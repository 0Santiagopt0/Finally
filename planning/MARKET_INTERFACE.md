# Market Data Interface Design

This document defines the unified Python interface for market data in FinAlly. The system has two implementations — a real-time Massive API client and a built-in simulator — selected at startup based on the `MASSIVE_API_KEY` environment variable.

## Design Goals

- Downstream code (SSE streaming, API routes, trade execution) never knows which source is active
- A single `PriceCache` holds all current prices; consumers read from it, never from the source directly
- Each source runs as an async background task that writes to the cache
- Adding/removing tickers at runtime is supported (watchlist changes)
- Startup blocks until the initial price cache is fully populated

## Core Data Types

### PriceUpdate

An immutable snapshot of the latest price for one ticker.

```python
@dataclass(frozen=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float  # price from the prior tick (for flash direction)
    prev_close: float      # previous trading day's close (for daily change %)
    timestamp: float        # Unix timestamp

    @property
    def change(self) -> float:
        """Tick-to-tick change (for flash animation)."""
        return self.price - self.previous_price

    @property
    def change_percent(self) -> float:
        """Tick-to-tick change as percentage."""
        if self.previous_price == 0:
            return 0.0
        return (self.change / self.previous_price) * 100

    @property
    def daily_change(self) -> float:
        """Change from previous close (for daily column)."""
        return self.price - self.prev_close

    @property
    def daily_change_percent(self) -> float:
        """Daily change as percentage."""
        if self.prev_close == 0:
            return 0.0
        return (self.daily_change / self.prev_close) * 100

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat' based on tick-to-tick change."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_sse_dict(self) -> dict:
        """Serialize for SSE event data."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "prev_close": self.prev_close,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": round(self.change_percent, 4),
            "daily_change": round(self.daily_change, 4),
            "daily_change_percent": round(self.daily_change_percent, 4),
            "direction": self.direction,
        }
```

### PriceCache

Thread-safe in-memory store. The single source of truth for current prices.

```python
class PriceCache:
    def update(self, ticker: str, price: float, prev_close: float, timestamp: float | None = None) -> PriceUpdate:
        """Update price for a ticker. Returns the new PriceUpdate.
        
        If the ticker already exists, previous_price is set to the old price.
        If the ticker is new, previous_price equals price (no flash on first tick).
        prev_close is stored on first update and retained on subsequent updates
        unless explicitly changed.
        """

    def get(self, ticker: str) -> PriceUpdate | None:
        """Get the latest PriceUpdate for a ticker, or None."""

    def get_price(self, ticker: str) -> float | None:
        """Get just the current price for a ticker, or None."""

    def get_all(self) -> dict[str, PriceUpdate]:
        """Get all current PriceUpdates."""

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache."""

    @property
    def version(self) -> int:
        """Monotonically increasing counter. Increments on every update.
        Used by SSE streaming to detect changes without polling each ticker."""
```

## Abstract Interface

```python
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    """Abstract interface for market data providers."""

    def __init__(self, price_cache: PriceCache):
        self._cache = price_cache

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Start producing prices for the given tickers.
        
        Must not return until the price cache is populated with initial
        prices for all requested tickers. This guarantees the first
        REST response and SSE event have real data.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and clean up resources."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Start tracking a new ticker.
        
        For Massive: validates the ticker on the next poll. If invalid
        (no data returned), removes it from the cache and raises ValueError.
        For Simulator: assigns a random seed price and starts GBM immediately.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Stop tracking a ticker and remove it from the cache."""
```

## Factory

```python
import os

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Create the appropriate market data source based on environment.
    
    If MASSIVE_API_KEY is set and non-empty, returns MassiveDataSource.
    Otherwise returns SimulatorDataSource.
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        from app.market.massive_client import MassiveDataSource
        return MassiveDataSource(price_cache=price_cache, api_key=api_key)
    else:
        from app.market.simulator import SimulatorDataSource
        return SimulatorDataSource(price_cache=price_cache)
```

## Module Layout

```
backend/app/market/
├── __init__.py          # Public exports: PriceUpdate, PriceCache, MarketDataSource, create_market_data_source
├── models.py            # PriceUpdate dataclass
├── cache.py             # PriceCache implementation
├── interface.py         # MarketDataSource ABC
├── factory.py           # create_market_data_source()
├── simulator.py         # SimulatorDataSource
├── massive_client.py    # MassiveDataSource
├── seed_prices.py       # Default ticker seed prices and GBM parameters
└── stream.py            # SSE streaming router (create_stream_router)
```

## Implementation: MassiveDataSource

```python
class MassiveDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, api_key: str, poll_interval: float = 15.0):
        super().__init__(price_cache)
        self._client = RESTClient(api_key=api_key)
        self._poll_interval = poll_interval
        self._tickers: set[str] = set()
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._tickers = set(tickers)
        # Blocking bootstrap: fetch all tickers before returning
        await self._fetch_snapshots()
        # Start background polling
        self._task = asyncio.create_task(self._poll_loop())

    async def _fetch_snapshots(self) -> None:
        """Fetch unified snapshots for all tracked tickers."""
        ticker_str = ",".join(sorted(self._tickers))
        # Run the synchronous Massive client in a thread
        snapshots = await asyncio.to_thread(
            lambda: list(self._client.list_universal_snapshots(
                params={
                    "ticker.any_of": ticker_str,
                    "type": "stocks",
                    "limit": 250,
                }
            ))
        )
        found_tickers = set()
        for snap in snapshots:
            ticker = snap.ticker
            if hasattr(snap, 'error') and snap.error:
                continue  # skip invalid tickers
            price = snap.last_trade.price if snap.last_trade else 0
            prev_close = snap.session.previous_close if snap.session else price
            self._cache.update(ticker, price, prev_close)
            found_tickers.add(ticker)

        # Remove tickers that Massive didn't return data for
        missing = self._tickers - found_tickers
        for ticker in missing:
            self._tickers.discard(ticker)

    async def _poll_loop(self) -> None:
        """Background loop: re-fetch snapshots at the configured interval."""
        while True:
            await asyncio.sleep(self._poll_interval)
            await self._fetch_snapshots()

    async def add_ticker(self, ticker: str) -> None:
        self._tickers.add(ticker)
        # Validate immediately with a single-ticker snapshot
        snapshot = await asyncio.to_thread(
            lambda: self._client.get_snapshot_ticker("stocks", ticker)
        )
        if snapshot is None:
            self._tickers.discard(ticker)
            raise ValueError(f"Ticker '{ticker}' not found on Massive")
        price = snapshot.last_trade.price if snapshot.last_trade else 0
        prev_close = snapshot.prev_day.close if snapshot.prev_day else price
        self._cache.update(ticker, price, prev_close)

    async def remove_ticker(self, ticker: str) -> None:
        self._tickers.discard(ticker)
        self._cache.remove(ticker)

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()
            self._task = None
```

## Implementation: SimulatorDataSource

See [MARKET_SIMULATOR.md](MARKET_SIMULATOR.md) for the full simulator design.

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5):
        super().__init__(price_cache)
        self._interval = update_interval
        self._tickers: dict[str, TickerState] = {}
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        # Seed all tickers synchronously (cache populated before returning)
        for ticker in tickers:
            self._init_ticker(ticker)
        # Start background GBM updates
        self._task = asyncio.create_task(self._update_loop())

    async def add_ticker(self, ticker: str) -> None:
        self._init_ticker(ticker)

    async def remove_ticker(self, ticker: str) -> None:
        self._tickers.pop(ticker, None)
        self._cache.remove(ticker)

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()
            self._task = None
```

## SSE Streaming

The SSE stream reads from the PriceCache and pushes updates to connected clients.

```python
def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create a FastAPI router with the SSE price stream endpoint."""
    router = APIRouter()

    @router.get("/api/stream/prices")
    async def stream_prices():
        async def event_generator():
            last_version = 0
            while True:
                await asyncio.sleep(0.5)
                if price_cache.version == last_version:
                    continue
                last_version = price_cache.version
                all_prices = price_cache.get_all()
                data = [update.to_sse_dict() for update in all_prices.values()]
                yield {"event": "prices", "data": json.dumps(data)}

        return EventSourceResponse(event_generator())

    return router
```

## Startup Integration

In the FastAPI lifespan, the market data source is created and started before the app accepts requests:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 1. Initialize database (schema + seed data)
    init_db()

    # 2. Create price cache and market data source
    price_cache = PriceCache()
    source = create_market_data_source(price_cache)

    # 3. Start market data (blocks until cache is populated)
    default_tickers = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA", "NVDA", "META", "JPM", "V", "NFLX"]
    await source.start(default_tickers)

    # 4. Record initial portfolio snapshot
    record_portfolio_snapshot()

    # 5. Make cache and source available to routes
    app.state.price_cache = price_cache
    app.state.market_source = source

    yield

    await source.stop()
```

## REST Watchlist Endpoint Contract

`GET /api/watchlist` returns tickers with current prices and `prev_close` so the frontend can compute daily change % on first load without waiting for the SSE stream:

```json
{
  "tickers": [
    {
      "ticker": "AAPL",
      "price": 192.53,
      "prev_close": 190.42,
      "daily_change": 2.11,
      "daily_change_percent": 1.108,
      "added_at": "2025-04-09T10:00:00Z"
    }
  ]
}
```
