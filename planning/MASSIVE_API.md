# Massive API Reference

Massive (massive.com, formerly Polygon.io) provides real-time and historical U.S. stock market data via REST, WebSocket, and flat file APIs. This document covers the REST endpoints relevant to FinAlly.

## Authentication

All requests require a Bearer token in the `Authorization` header:

```
Authorization: Bearer YOUR_API_KEY
```

Base URL: `https://api.massive.com`

## Python Client Library

The official Python client is the `massive` package (v2.5.0+), rebranded from `polygon-api-client`.

```bash
pip install massive
# or: uv add massive
```

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_API_KEY")
```

Constructor options:
- `api_key` (str, required) — Massive API key
- `pagination` (bool, default `True`) — auto-paginate list endpoints
- `trace` (bool, default `False`) — debug logging
- `verbose` (bool, default `False`) — verbose trace output

Import path changed from `from polygon import RESTClient` to `from massive import RESTClient` in v2.0.1. The legacy `api.polygon.io` domain still works during the transition period.

---

## Rate Limits

Limits are per-plan, enforced per minute:

| Plan | Calls/min | Recommended Poll Interval |
|------|-----------|---------------------------|
| Free | 5 | 15s |
| Stocks Starter | 100 | 5–10s |
| Stocks Developer | Unlimited* | 2–5s |
| Stocks Advanced/Business | Unlimited* | 2s |

*Subject to fair use. The free tier is sufficient for FinAlly (one user, 10 tickers, 15-second polling).

---

## Endpoints Used by FinAlly

### 1. Unified Snapshot — Multiple Tickers in One Call

**This is the primary endpoint for FinAlly.** It returns current price, previous close, and session data for up to 250 tickers in a single request.

```
GET /v3/snapshot?ticker.any_of=AAPL,GOOGL,MSFT&type=stocks&limit=250
```

**Python client:**
```python
snapshots = client.list_universal_snapshots(
    params={
        "ticker.any_of": "AAPL,GOOGL,MSFT,AMZN,TSLA,NVDA,META,JPM,V,NFLX",
        "type": "stocks",
        "limit": 250,
    }
)
for snap in snapshots:
    print(snap)
```

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticker.any_of` | string | Comma-separated list of tickers (up to 250) |
| `type` | string | Asset class filter: `stocks`, `options`, `fx`, `crypto`, `indices` |
| `limit` | integer | Max results per page (default 10, max 250) |
| `ticker` | string | Single ticker or lexicographic range search |
| `ticker.gte`, `ticker.gt`, `ticker.lte`, `ticker.lt` | string | Range filters |
| `order` | string | Sort direction |
| `sort` | string | Sort field |

**Response schema (relevant fields for stocks):**

```json
{
  "request_id": "abc123",
  "status": "OK",
  "results": [
    {
      "ticker": "AAPL",
      "name": "Apple Inc.",
      "type": "stocks",
      "market_status": "open",
      "last_trade": {
        "price": 192.53,
        "size": 100,
        "exchange": 4,
        "conditions": [37],
        "sip_timestamp": 1617901342969834000,
        "last_updated": 1617901342969834000,
        "id": "118749",
        "timeframe": "REAL-TIME"
      },
      "last_quote": {
        "bid": 192.52,
        "bid_size": 200,
        "ask": 192.54,
        "ask_size": 300,
        "last_updated": 1617901342969834000,
        "midpoint": 192.53,
        "timeframe": "REAL-TIME"
      },
      "session": {
        "open": 191.20,
        "high": 193.10,
        "low": 190.85,
        "close": 192.53,
        "previous_close": 190.42,
        "volume": 52341200,
        "change": 2.11,
        "change_percent": 1.108,
        "early_trading_change": 0.50,
        "early_trading_change_percent": 0.26,
        "regular_trading_change": 1.61,
        "regular_trading_change_percent": 0.85,
        "late_trading_change": 0.0,
        "late_trading_change_percent": 0.0
      },
      "error": null,
      "message": null
    }
  ],
  "next_url": null
}
```

Key fields for FinAlly:
- `last_trade.price` — current price
- `session.previous_close` — previous day's closing price (for daily change %)
- `session.change` / `session.change_percent` — pre-computed daily change
- `error` — non-null if the ticker is invalid (e.g., `"NOT_FOUND"`)

---

### 2. Full Market Snapshot (v2) — Alternative for Multiple Tickers

A v2 endpoint that also supports querying multiple tickers at once.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT
```

**Python client:**
```python
snapshots = client.get_snapshot_all("stocks", params={"tickers": "AAPL,GOOGL,MSFT"})
```

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tickers` | string | Comma-separated ticker list (omit for all) |
| `include_otc` | boolean | Include OTC securities (default `false`) |

**Response schema:**
```json
{
  "status": "OK",
  "count": 3,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 2.11,
      "todaysChangePerc": 1.108,
      "updated": 1617901342969834000,
      "day": {
        "o": 191.20,
        "h": 193.10,
        "l": 190.85,
        "c": 192.53,
        "v": 52341200,
        "vw": 191.85
      },
      "prevDay": {
        "o": 189.50,
        "h": 191.00,
        "l": 189.20,
        "c": 190.42,
        "v": 48231000,
        "vw": 190.15
      },
      "min": {
        "o": 192.40,
        "h": 192.55,
        "l": 192.38,
        "c": 192.53,
        "v": 12340,
        "vw": 192.45,
        "t": 1617901320000,
        "n": 85
      },
      "lastQuote": {
        "p": 192.52,
        "P": 192.54,
        "s": 200,
        "S": 300,
        "t": 1617901342969834000
      },
      "lastTrade": {
        "p": 192.53,
        "s": 100,
        "x": 4,
        "t": 1617901342969834000,
        "c": [37],
        "i": "118749"
      },
      "fmv": 192.53
    }
  ]
}
```

Key fields for FinAlly:
- `lastTrade.p` — current trade price
- `prevDay.c` — previous day's close
- `todaysChange` / `todaysChangePerc` — daily change (pre-computed)

---

### 3. Single Ticker Snapshot

For looking up individual tickers (e.g., when a user adds a new ticker to the watchlist).

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}
```

**Python client:**
```python
snapshot = client.get_snapshot_ticker("stocks", "AAPL")
```

Same response shape as one element of the v2 full market snapshot above.

---

### 4. Previous Day Bar (OHLC)

Returns the previous trading day's OHLC data for a single ticker.

```
GET /v2/aggs/ticker/{stocksTicker}/prev
```

**Python client:**
```python
aggs = client.get_previous_close_agg("AAPL")
```

**Response:**
```json
{
  "ticker": "AAPL",
  "adjusted": true,
  "queryCount": 1,
  "resultsCount": 1,
  "status": "OK",
  "request_id": "6a7e466379af0a71039d60cc78e72282",
  "results": [
    {
      "T": "AAPL",
      "o": 115.55,
      "h": 117.59,
      "l": 114.13,
      "c": 115.97,
      "v": 131704427,
      "vw": 116.3058,
      "t": 1605042000000
    }
  ]
}
```

Key field: `results[0].c` is the previous close.

---

### 5. Last Trade

Returns the most recent trade for a ticker.

```
GET /v2/last/trade/{stocksTicker}
```

**Python client:**
```python
trade = client.get_last_trade(ticker="AAPL")
print(trade.price)  # e.g., 192.53
```

**Response:**
```json
{
  "request_id": "f05562305bd26ced64b98ed68b3c5d96",
  "status": "OK",
  "results": {
    "T": "AAPL",
    "p": 129.8473,
    "s": 25,
    "x": 4,
    "t": 1617901342969834000,
    "c": [37],
    "i": "118749"
  }
}
```

---

## Error Handling

- **Invalid ticker**: The Unified Snapshot (`/v3/snapshot`) returns the ticker in `results` with a non-null `error` field (e.g., `"NOT_FOUND"`). The v2 snapshot simply omits tickers that have no data.
- **Rate limit exceeded**: Returns HTTP 429. The Python client does not auto-retry; the caller must implement backoff.
- **Authentication error**: Returns HTTP 401/403 if the API key is missing or invalid.
- **Market closed**: Endpoints still return data (last available prices). The `market_status` field in the Unified Snapshot indicates whether the market is currently open.

## Recommended Strategy for FinAlly

1. **Bootstrap on startup**: Call the Unified Snapshot (`/v3/snapshot`) with `ticker.any_of` for all default watchlist tickers. This populates the price cache with current prices and `previous_close` in a single API call.
2. **Poll periodically**: Re-call the same endpoint every 15 seconds (free tier) to get updated prices for all watched tickers in one request.
3. **Add ticker**: When a user adds a ticker, call the Single Ticker Snapshot to validate and get its initial price. If the response contains an error, reject the ticker.
4. **Minimize API calls**: Always batch tickers into a single Unified Snapshot call rather than making per-ticker requests. This is critical on the free tier (5 calls/min).
