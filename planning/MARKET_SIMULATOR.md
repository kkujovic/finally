# Market Simulator — Approach and Code Structure

The simulator is the default market data source. It requires no API key and produces realistic, visually engaging price movements suitable for a demo trading terminal.

---

## Overview

Two classes cooperate:

- **`GBMSimulator`** — pure math engine. Holds prices and parameters, advances all tickers by one time step, returns new prices.
- **`SimulatorDataSource`** — lifecycle wrapper. Implements `MarketDataSource`. Runs an asyncio background task that calls `GBMSimulator.step()` every 500 ms and writes results to `PriceCache`.

---

## Price Model: Geometric Brownian Motion (GBM)

Each tick, every ticker's price evolves as:

```
S(t+dt) = S(t) * exp((mu - sigma²/2) * dt + sigma * sqrt(dt) * Z)
```

| Symbol  | Meaning                                              |
|---------|------------------------------------------------------|
| `S(t)`  | Current price                                        |
| `mu`    | Annualized drift (expected return, e.g. 0.05 = 5%)   |
| `sigma` | Annualized volatility (e.g. 0.25 = 25%)              |
| `dt`    | Time step as fraction of a trading year              |
| `Z`     | Correlated standard normal random variable           |

### Time step

500 ms expressed as a fraction of a trading year (252 days × 6.5 h × 3600 s):

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600   # 5,896,800 s
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR    # ~8.48e-8
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally into realistic intraday swings.

---

## Correlated Moves

Independent random draws would make all tickers move completely uncorrelated — unrealistic. Real stocks in the same sector tend to move together. The simulator uses **Cholesky decomposition** to introduce sector correlation.

### Correlation matrix

Built once when `GBMSimulator` is constructed (and rebuilt whenever tickers are added/removed):

```python
# Build n×n correlation matrix
corr = np.eye(n)
for i, j in pairs:
    corr[i, j] = corr[j, i] = pairwise_correlation(ticker_i, ticker_j)

# Decompose: corr = L @ L.T
cholesky = np.linalg.cholesky(corr)
```

### Correlation rules

```python
# Pairwise correlation between any two tickers:
if either ticker is TSLA:    rho = 0.3   # TSLA does its own thing
elif both in tech sector:    rho = 0.6   # AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX
elif both in finance sector: rho = 0.5   # JPM, V
else (cross-sector):         rho = 0.3
```

### Applying correlation

```python
# n independent standard normals
z_independent = np.random.standard_normal(n)

# Apply Cholesky: L @ z gives correlated normals
z_correlated = cholesky @ z_independent
```

Each `z_correlated[i]` is used in the GBM formula for ticker `i`. Tickers in the same sector get positively correlated draws, producing synchronized moves.

---

## Random Shock Events

On every tick, each ticker has a small chance (~0.1%) of a sudden 2–5% jump or drop:

```python
if random.random() < 0.001:   # event_probability
    shock = random.uniform(0.02, 0.05)
    sign  = random.choice([-1, 1])
    price *= 1 + shock * sign
```

With 10 tickers at 2 ticks/sec, expect a shock event roughly every 50 seconds. This provides visual drama in the UI (sudden price flashes) without making the simulation unrealistic.

---

## Seed Prices and Parameters (`seed_prices.py`)

```python
SEED_PRICES = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00,
    "AMZN": 185.00, "TSLA": 250.00, "NVDA": 800.00,
    "META": 500.00, "JPM":  195.00, "V":    280.00,
    "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # High vol
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # High vol + strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # Low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # Low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # For dynamically added tickers
```

Tickers added dynamically (not in `SEED_PRICES`) start at a random price between $50–$300 and use `DEFAULT_PARAMS`.

---

## `GBMSimulator` Class Interface

```python
class GBMSimulator:
    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None: ...

    def step(self) -> dict[str, float]:
        """Advance all tickers by one dt. Returns {ticker: new_price}.
        Hot path — called every 500 ms."""

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Rebuilds the Cholesky matrix."""

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds the Cholesky matrix."""

    def get_price(self, ticker: str) -> float | None: ...
    def get_tickers(self) -> list[str]: ...
```

---

## `SimulatorDataSource` Class Interface

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,     # seconds between ticks
        event_probability: float = 0.001,
    ) -> None: ...

    # Lifecycle
    async def start(self, tickers: list[str]) -> None:
        # Creates GBMSimulator, seeds cache, starts asyncio background task

    async def stop(self) -> None:
        # Cancels the background task

    # Dynamic watchlist
    async def add_ticker(self, ticker: str) -> None:
        # Adds to GBMSimulator, immediately seeds cache with initial price

    async def remove_ticker(self, ticker: str) -> None:
        # Removes from GBMSimulator and from cache

    def get_tickers(self) -> list[str]: ...
```

### Background loop

```python
async def _run_loop(self) -> None:
    while True:
        prices = self._sim.step()           # dict[str, float]
        for ticker, price in prices.items():
            self._cache.update(ticker, price)
        await asyncio.sleep(self._interval)
```

On every tick: step the simulator → write all prices to cache → sleep 500 ms. The cache version increments on every `update()` call, triggering SSE emission.

---

## Performance Notes

- `step()` is O(n) in ticker count after the Cholesky is applied.
- `_rebuild_cholesky()` is O(n²) but n < 50 in practice. Rebuilds happen only on `add_ticker` / `remove_ticker`, not on every tick.
- NumPy operations (random draws, matrix multiply) run in C — negligible CPU overhead for n ≤ 50.
- The asyncio `sleep(0.5)` means the event loop is free for 499.9 ms of every 500 ms tick.

---

## Adding a New Ticker at Runtime

```python
await source.add_ticker("PYPL")
# 1. GBMSimulator._add_ticker_internal("PYPL"):
#      price = SEED_PRICES.get("PYPL", random.uniform(50, 300))
#      params = TICKER_PARAMS.get("PYPL", DEFAULT_PARAMS)
# 2. GBMSimulator._rebuild_cholesky()  ← includes PYPL in correlation matrix
# 3. SimulatorDataSource seeds cache.update("PYPL", price)
#    → cache has PYPL immediately, SSE emits it on next tick
```

---

## Demo

```bash
cd backend
uv run market_data_demo.py
```

Runs a Rich terminal dashboard showing all 10 tickers with live sparklines, color-coded direction arrows, and a shock event log. Runs for 60 seconds or until Ctrl+C.
