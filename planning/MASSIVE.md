# Massive (Polygon.io) API — Reference for FinAlly

Massive is the Python client for the [Polygon.io](https://polygon.io) REST API. The `massive` package exposes a `RESTClient` that is **synchronous** — all calls are blocking. In an async FastAPI app, run them in a thread via `asyncio.to_thread()`.

---

## Authentication

```python
from massive import RESTClient

client = RESTClient(api_key="your_key_here")
# Or set env var MASSIVE_API_KEY and call RESTClient() with no arguments
```

---

## Rate Limits

| Tier       | Requests / min | Recommended poll interval |
|------------|---------------|--------------------------|
| Free       | 5             | ≥ 15 s                   |
| Starter    | 60            | ~5 s                     |
| Developer  | 300           | ~2 s                     |
| Advanced   | unlimited     | 1 s                      |

The FinAlly backend defaults to 15 s polling (free-tier safe).

---

## Key Methods

### 1. Snapshot — all tickers at once (primary method used in FinAlly)

Fetches the latest trade, day OHLCV, and minute bar for multiple tickers in one request.

```python
from massive.rest.models import SnapshotMarketType

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT"],  # omit for all tickers
    include_otc=False,
)

for snap in snapshots:
    print(snap.ticker)                          # "AAPL"
    print(snap.last_trade.price)                # 190.50  (float)
    print(snap.last_trade.timestamp)            # 1707580800000  (Unix ms)
    print(snap.day.open)                        # 188.00
    print(snap.day.close)                       # 190.20
    print(snap.day.high)                        # 191.50
    print(snap.day.low)                         # 187.80
    print(snap.day.volume)                      # 54321000.0
    print(snap.todays_change_perc)              # 1.23  (% change from prior close)
    print(snap.prev_day.close)                  # 188.10
    print(snap.min.open)                        # most recent minute bar open
```

**Timestamp conversion**: `last_trade.timestamp` is Unix **milliseconds**. Convert to seconds:
```python
timestamp_s = snap.last_trade.timestamp / 1000.0
```

### 2. Snapshot — single ticker

```python
snap = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
# Same fields as above
```

### 3. Last trade

```python
trade = client.get_last_trade("AAPL")
print(trade.results.price)          # float
print(trade.results.sip_timestamp)  # Unix ns — divide by 1e9 for seconds
print(trade.results.size)           # shares in trade
```

### 4. Aggregate bars (OHLCV history)

Returns historical price bars at any resolution. Useful for charts or seeding sparklines.

```python
bars = client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="minute",       # second | minute | hour | day | week | month
    from_="2024-02-10",
    to="2024-02-10",
    adjusted=True,
    sort="asc",
    limit=120,
)

for bar in bars:
    print(bar.open, bar.high, bar.low, bar.close)
    print(bar.volume, bar.vwap)
    print(bar.timestamp)   # Unix ms for the bar's start
```

### 5. Top movers

```python
from massive.rest.models import Direction

gainers = client.get_snapshot_direction(
    market_type=SnapshotMarketType.STOCKS,
    direction=Direction.GAINERS,
)
losers = client.get_snapshot_direction(
    market_type=SnapshotMarketType.STOCKS,
    direction=Direction.LOSERS,
)
```

---

## Error Handling

| HTTP Status | Meaning                  | Action                        |
|-------------|--------------------------|-------------------------------|
| 401         | Bad API key              | Log error, stop polling       |
| 403         | Insufficient plan        | Endpoint not available on tier |
| 429         | Rate limit exceeded      | Increase poll interval        |
| 5xx         | Server error             | Log, retry on next interval   |

The `MassiveDataSource` catches all exceptions and logs them — the polling loop continues regardless.

---

## Running in Async FastAPI

The `RESTClient` is **synchronous**. Always offload to a thread:

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="...")

async def fetch_prices(tickers: list[str]) -> list:
    return await asyncio.to_thread(
        client.get_snapshot_all,
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
```

---

## Fields Available on `PriceUpdate` (FinAlly's abstraction)

After the `MassiveDataSource` processes a snapshot, the `PriceCache` stores:

| Field            | Source                         |
|------------------|-------------------------------|
| `ticker`         | `snap.ticker`                 |
| `price`          | `snap.last_trade.price`       |
| `previous_price` | prior cache value (tracked internally) |
| `timestamp`      | `snap.last_trade.timestamp / 1000.0` |
| `direction`      | computed: up / down / flat    |
| `change`         | computed: price − previous    |
| `change_percent` | computed: change / previous × 100 |

The `day.open` / daily change percent fields from the snapshot are not yet stored in `PriceCache` — if daily change display is needed, `PriceUpdate` can be extended with an `open_price` field and the `MassiveDataSource._poll_once()` updated to pass `snap.day.open`.
