# Market Data Simulator Design

The simulator generates realistic stock prices using Geometric Brownian Motion (GBM) with correlated moves, per-ticker volatility, and occasional random events. It is the default market data source when `MASSIVE_API_KEY` is not set.

## GBM Model

Each price update applies a single GBM step:

```
S(t+dt) = S(t) * exp((mu - sigma^2/2)*dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` is the current price
- `mu` is the annualized drift (expected return)
- `sigma` is the annualized volatility
- `dt` is the time step (update interval / seconds per trading year)
- `Z` is a standard normal random variable (possibly correlated across tickers)

For a 0.5-second update interval: `dt = 0.5 / (252 * 6.5 * 3600) ≈ 8.5e-8`

At this tiny `dt`, individual ticks produce small, realistic-looking price changes (fractions of a cent to a few cents).

## Seed Prices and Parameters

Each of the 10 default tickers has a seed price, annualized volatility, and drift:

```python
SEED_PRICES: dict[str, SeedData] = {
    "AAPL":  SeedData(price=189.84, volatility=0.25, drift=0.08),
    "GOOGL": SeedData(price=175.98, volatility=0.28, drift=0.10),
    "MSFT":  SeedData(price=420.72, volatility=0.24, drift=0.09),
    "AMZN":  SeedData(price=186.13, volatility=0.30, drift=0.12),
    "TSLA":  SeedData(price=248.42, volatility=0.55, drift=0.05),
    "NVDA":  SeedData(price=878.37, volatility=0.50, drift=0.15),
    "META":  SeedData(price=501.80, volatility=0.32, drift=0.11),
    "JPM":   SeedData(price=198.47, volatility=0.20, drift=0.06),
    "V":     SeedData(price=279.98, volatility=0.18, drift=0.07),
    "NFLX":  SeedData(price=609.27, volatility=0.35, drift=0.10),
}
```

Volatility values reflect each stock's real-world character: TSLA and NVDA are highly volatile, V and JPM are more stable.

### Dynamic Ticker Addition

When a user adds a ticker not in the seed set, the simulator assigns:
- Random seed price between $20 and $300
- Default volatility of 0.30
- Default drift of 0.08

GBM generation starts immediately.

## Previous Close (prev_close)

Each ticker gets a synthetic `prev_close` computed at initialization:

```python
prev_close = seed_price * (1 + random.uniform(-0.02, 0.02))
```

This gives each ticker a small initial daily change so the frontend's daily change column has non-zero values from the start. The `prev_close` is set once per ticker and does not change during a session.

## Correlated Moves

Real stocks don't move independently — tech stocks tend to move together, and market-wide sentiment affects everything. The simulator models this with a correlation matrix applied via Cholesky decomposition.

### Correlation Structure

Tickers are grouped by sector:

| Group | Tickers | Intra-group correlation |
|-------|---------|------------------------|
| Big Tech | AAPL, GOOGL, MSFT, AMZN, META | 0.6 |
| High Vol Tech | TSLA, NVDA | 0.5 |
| Finance | JPM, V | 0.5 |
| Media/Entertainment | NFLX | — |

Cross-group correlation: 0.2 (mild market-wide coupling).

### Implementation

1. Build a correlation matrix `C` for all active tickers
2. Compute the Cholesky decomposition `L = cholesky(C)`
3. Each update step:
   - Generate independent standard normal vector `Z_indep` (one per ticker)
   - Multiply: `Z_correlated = L @ Z_indep`
   - Apply GBM step to each ticker using its correlated `Z` value

```python
import numpy as np

class CorrelatedGBM:
    def __init__(self, tickers: list[str], correlation_matrix: np.ndarray):
        self._tickers = tickers
        self._cholesky = np.linalg.cholesky(correlation_matrix)

    def step(self, prices: dict[str, float], params: dict[str, SeedData], dt: float) -> dict[str, float]:
        n = len(self._tickers)
        z_indep = np.random.standard_normal(n)
        z_corr = self._cholesky @ z_indep

        new_prices = {}
        for i, ticker in enumerate(self._tickers):
            p = params[ticker]
            s = prices[ticker]
            drift_term = (p.drift - 0.5 * p.volatility**2) * dt
            diffusion_term = p.volatility * np.sqrt(dt) * z_corr[i]
            new_prices[ticker] = s * np.exp(drift_term + diffusion_term)
        return new_prices
```

When tickers are added or removed dynamically, the correlation matrix and Cholesky decomposition are recomputed. New tickers not in any defined group get the cross-group correlation (0.2) with all existing tickers.

## Random Events

Every update step, there is a small probability of a "market event" — a sudden 2–5% move on a single ticker. This adds drama and makes the demo more visually interesting.

```python
EVENT_PROBABILITY = 0.002  # ~0.2% chance per tick per ticker
EVENT_MIN_MAGNITUDE = 0.02  # 2%
EVENT_MAX_MAGNITUDE = 0.05  # 5%

def maybe_apply_event(ticker: str, price: float) -> float:
    if random.random() < EVENT_PROBABILITY:
        magnitude = random.uniform(EVENT_MIN_MAGNITUDE, EVENT_MAX_MAGNITUDE)
        direction = random.choice([-1, 1])
        return price * (1 + direction * magnitude)
    return price
```

Events are applied after the GBM step. They're rare enough (roughly one event per ticker every ~4 minutes at 0.5s intervals) to be surprising but not disruptive.

## SimulatorDataSource Implementation

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5):
        super().__init__(price_cache)
        self._interval = update_interval
        self._ticker_states: dict[str, TickerState] = {}
        self._gbm: CorrelatedGBM | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        # Initialize all tickers synchronously
        for ticker in tickers:
            self._init_ticker(ticker)
        self._rebuild_correlation()
        # Start background updates
        self._task = asyncio.create_task(self._update_loop())

    def _init_ticker(self, ticker: str) -> None:
        """Set up a ticker with seed price and write to cache."""
        seed = SEED_PRICES.get(ticker)
        if seed:
            price = seed.price
            vol = seed.volatility
            drift = seed.drift
        else:
            price = random.uniform(20.0, 300.0)
            vol = 0.30
            drift = 0.08

        prev_close = price * (1 + random.uniform(-0.02, 0.02))
        self._ticker_states[ticker] = TickerState(
            price=price, volatility=vol, drift=drift, prev_close=prev_close
        )
        self._cache.update(ticker, price, prev_close)

    def _rebuild_correlation(self) -> None:
        """Rebuild the correlation matrix and Cholesky decomposition."""
        tickers = list(self._ticker_states.keys())
        if len(tickers) < 2:
            self._gbm = None
            return
        matrix = build_correlation_matrix(tickers)
        self._gbm = CorrelatedGBM(tickers, matrix)

    async def _update_loop(self) -> None:
        """Background loop: apply GBM steps at the configured interval."""
        dt = self._interval / (252 * 6.5 * 3600)
        while True:
            await asyncio.sleep(self._interval)
            self._step(dt)

    def _step(self, dt: float) -> None:
        """Single GBM step for all tickers."""
        tickers = list(self._ticker_states.keys())
        current_prices = {t: self._ticker_states[t].price for t in tickers}
        params = self._ticker_states

        if self._gbm and len(tickers) >= 2:
            new_prices = self._gbm.step(current_prices, params, dt)
        else:
            # Single ticker: uncorrelated GBM
            new_prices = {}
            for t in tickers:
                p = params[t]
                z = np.random.standard_normal()
                drift_term = (p.drift - 0.5 * p.volatility**2) * dt
                diffusion_term = p.volatility * np.sqrt(dt) * z
                new_prices[t] = current_prices[t] * np.exp(drift_term + diffusion_term)

        # Apply random events and update cache
        for ticker in tickers:
            price = maybe_apply_event(ticker, new_prices[ticker])
            self._ticker_states[ticker].price = price
            self._cache.update(ticker, price, self._ticker_states[ticker].prev_close)

    async def add_ticker(self, ticker: str) -> None:
        self._init_ticker(ticker)
        self._rebuild_correlation()

    async def remove_ticker(self, ticker: str) -> None:
        self._ticker_states.pop(ticker, None)
        self._cache.remove(ticker)
        self._rebuild_correlation()

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()
            self._task = None
```

## Internal State

```python
@dataclass
class TickerState:
    price: float       # current price
    volatility: float  # annualized sigma
    drift: float       # annualized mu
    prev_close: float  # synthetic previous day close

@dataclass(frozen=True)
class SeedData:
    price: float
    volatility: float
    drift: float
```

## Correlation Matrix Builder

```python
SECTOR_GROUPS = {
    "big_tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META"},
    "high_vol_tech": {"TSLA", "NVDA"},
    "finance": {"JPM", "V"},
}

INTRA_GROUP_CORR = {
    "big_tech": 0.6,
    "high_vol_tech": 0.5,
    "finance": 0.5,
}

CROSS_GROUP_CORR = 0.2

def build_correlation_matrix(tickers: list[str]) -> np.ndarray:
    n = len(tickers)
    matrix = np.eye(n)

    def get_group(ticker: str) -> str | None:
        for group, members in SECTOR_GROUPS.items():
            if ticker in members:
                return group
        return None

    for i in range(n):
        for j in range(i + 1, n):
            g_i = get_group(tickers[i])
            g_j = get_group(tickers[j])
            if g_i is not None and g_i == g_j:
                corr = INTRA_GROUP_CORR[g_i]
            else:
                corr = CROSS_GROUP_CORR
            matrix[i, j] = corr
            matrix[j, i] = corr

    return matrix
```

## Timing and Performance

- Update interval: 500ms (configurable, passed to `SimulatorDataSource`)
- At 10 tickers: each step involves generating 10 random numbers, a 10x10 matrix multiply, and 10 cache updates — trivially fast
- At 50 tickers (edge case): still sub-millisecond
- NumPy handles the Cholesky decomposition and matrix operations efficiently
- The Cholesky decomposition is only recomputed when tickers are added or removed, not on every step
