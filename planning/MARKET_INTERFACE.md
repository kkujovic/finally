# Market Data Interface — Python API Design

This document describes the Python API for the market data subsystem in `backend/app/market/`. All code lives there and is complete.

---

## Architecture

```
MASSIVE_API_KEY set?
        │
   yes  │  no
        │
        ▼           ▼
MassiveDataSource  SimulatorDataSource
        │                   │
        └────────┬───────────┘
                 │
           PriceCache  (thread-safe, in-memory)
                 │
     ┌───────────┼──────────────┐
     ▼           ▼              ▼
 SSE stream  Portfolio      Trade execution
 /api/stream  valuation
 /prices
```

Both data sources implement `MarketDataSource` (ABC). All downstream code reads from `PriceCache` — never from the source directly. The factory function `create_market_data_source()` picks the right implementation based on the environment.

---

## Module Map

| File                   | Purpose                                                     |
|------------------------|-------------------------------------------------------------|
| `models.py`            | `PriceUpdate` — immutable dataclass for a single price tick |
| `interface.py`         | `MarketDataSource` — abstract base class (the contract)     |
| `cache.py`             | `PriceCache` — thread-safe in-memory price store            |
| `seed_prices.py`       | Seed prices, GBM params, sector correlation constants       |
| `simulator.py`         | `GBMSimulator` + `SimulatorDataSource`                      |
| `massive_client.py`    | `MassiveDataSource` — REST polling client                   |
| `factory.py`           | `create_market_data_source(cache)` — env-driven factory     |
| `stream.py`            | `create_stream_router(cache)` — FastAPI SSE router factory  |

---

## Core Types

### `PriceUpdate` (`models.py`)

Immutable frozen dataclass. One instance per price tick.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds

    # Computed properties (no storage cost):
    change: float             # price - previous_price
    change_percent: float     # change / previous_price * 100
    direction: str            # "up" | "down" | "flat"

    def to_dict(self) -> dict: ...   # JSON-serializable
```

### `MarketDataSource` (`interface.py`)

Abstract base class. Both implementations must conform.

```python
class MarketDataSource(ABC):
    async def start(self, tickers: list[str]) -> None: ...
    async def stop(self) -> None: ...
    async def add_ticker(self, ticker: str) -> None: ...
    async def remove_ticker(self, ticker: str) -> None: ...
    def get_tickers(self) -> list[str]: ...
```

### `PriceCache` (`cache.py`)

Thread-safe. The single source of truth for current prices at runtime.

```python
cache = PriceCache()

# Write (called by data source only)
update: PriceUpdate = cache.update("AAPL", 190.50, timestamp=1707580800.0)

# Read (called by SSE, portfolio, trade execution)
update: PriceUpdate | None = cache.get("AAPL")
price: float | None         = cache.get_price("AAPL")
all_prices: dict[str, PriceUpdate] = cache.get_all()

# Remove (called on watchlist removal)
cache.remove("AAPL")

# SSE change detection
version: int = cache.version   # increments on every update
```

---

## Factory

```python
# factory.py
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)
# Returns MassiveDataSource if MASSIVE_API_KEY is set, else SimulatorDataSource
```

---

## Lifecycle (FastAPI app startup / shutdown)

```python
# In main.py lifespan handler:

@asynccontextmanager
async def lifespan(app: FastAPI):
    cache = PriceCache()
    source = create_market_data_source(cache)
    await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                        "NVDA", "META", "JPM", "V", "NFLX"])
    app.state.price_cache = cache
    app.state.market_source = source
    yield
    await source.stop()
```

---

## Dynamic Watchlist Management

When the user adds or removes a ticker via the watchlist API:

```python
# Add ticker (watchlist POST handler)
await source.add_ticker("PYPL")
# SimulatorDataSource: seeds cache immediately with a price
# MassiveDataSource: adds to _tickers list; appears on next poll

# Remove ticker (watchlist DELETE handler)
await source.remove_ticker("PYPL")
# Both implementations remove from cache immediately
```

---

## SSE Streaming

```python
# In main.py, after cache is created:
from app.market import create_stream_router

stream_router = create_stream_router(cache)
app.include_router(stream_router)

# Endpoint: GET /api/stream/prices
# Media type: text/event-stream
# Each event:
#   data: {"AAPL": {"ticker":"AAPL","price":190.5,...}, "GOOGL": {...}, ...}
```

Change detection: the generator polls `cache.version` every 500 ms and only emits an event when the version has incremented. No redundant pushes.

---

## Environment Variables

| Variable         | Effect                                                    |
|------------------|-----------------------------------------------------------|
| `MASSIVE_API_KEY` | If set and non-empty, `MassiveDataSource` is used         |
| (not set)         | `SimulatorDataSource` is used (default, no API key needed)|

---

## Update Cadence

| Source              | Update frequency   | Notes                              |
|---------------------|--------------------|------------------------------------|
| SimulatorDataSource | every 500 ms       | asyncio background task            |
| MassiveDataSource   | every 15 s (default) | free-tier safe; configurable via `poll_interval` |

---

## Extending the Interface

To add a new data source (e.g., a WebSocket feed):

1. Create a new file, e.g., `websocket_client.py`
2. Subclass `MarketDataSource` and implement all 5 abstract methods
3. Write to `PriceCache` via `cache.update(ticker, price, timestamp)`
4. Update `factory.py` to select the new source based on a new env var

No other code changes needed — SSE, portfolio valuation, and trade execution all read from the cache.
