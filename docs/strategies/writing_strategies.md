# Writing Strategies

A Honba strategy is a Python class that inherits from `Strategy` and implements lifecycle callbacks. The engine calls the callbacks in response to market events; the strategy responds by submitting order intents through its context.

The strategy code is identical in backtest and live. The engine determines which mode it is in.

## The Minimal Strategy

```python
from honba.entities.bar import Bar
from honba.entities.instrument import InstrumentId
from honba.strategies.base import Strategy


class BuyAndHold(Strategy):
    name = "buy_and_hold"

    def on_start(self):
        self.ctx.submit(
            OrderIntent.market_buy(InstrumentId("RELIANCE", "NSE"), 100)
        )

    def on_stop(self):
        pass  # intents submitted here are discarded (run is over)
```

That is a complete strategy. `on_start` runs once when the engine begins; `on_stop` runs once when it ends. Everything else is optional.

!!! warning "`on_stop` intents are discarded"
    The `on_stop` hook is called, but any order intents submitted during it are **discarded** (the run is already over). This is a known limitation; see the runner implementation. If you need to flatten positions at the end, do it in `on_bar` on the last bar or via a timer scheduled near session close.

## Lifecycle Callbacks

The engine calls these methods, in this order, at the appropriate time:

| Callback | When |
|---|---|
| `on_start()` | Once, before any market data |
| `on_bar(bar)` | On every bar for a subscribed instrument |
| `on_quote(quote)` | On every quote tick (if subscribed) |
| `on_trade(trade)` | On every trade tick (if subscribed) |
| `on_fill(fill)` | On every fill |
| `on_stop()` | Once, after all data has been processed |

All callbacks are synchronous. The engine is single-threaded and deterministic; do not spawn threads or use `asyncio` inside a strategy.

## Subscribing to Data

A strategy does not subscribe to data directly. Subscriptions are declared in the **run configuration** (e.g., `config.toml`), not in strategy code. The engine delivers events for the instruments and timeframes specified in the config.

```toml
# config.toml
[engine]
subscriptions = [
  { symbol = "RELIANCE", exchange = "NSE", timeframe = "1d" },
  { symbol = "NIFTY50", exchange = "NSE", timeframe = "1m" },
]
```

Bars are aggregated **in the engine**, not in the strategy. A strategy that receives 1-minute bars receives fully-formed bars; it does not see ticks.

## Placing Orders

The strategy places orders by submitting `OrderIntent` objects through its context (`self.ctx`). The convenience methods `buy()` and `sell()` submit market orders:

```python
# Market order — fills at next available price
self.buy(InstrumentId("RELIANCE", "NSE"), 100)
self.sell(InstrumentId("RELIANCE", "NSE"), 100)

# Limit order
self.ctx.submit(OrderIntent.limit_buy(InstrumentId("RELIANCE", "NSE"), 100, 2450.00))

# Stop order (stop-loss)
self.ctx.submit(OrderIntent.stop_sell(InstrumentId("RELIANCE", "NSE"), 100, 2400.00))

# Stop-limit order
self.ctx.submit(OrderIntent.stop_limit_sell(InstrumentId("RELIANCE", "NSE"), 100, 2400.00, 2395.00))

# Trailing stop (Python interface only — no Rust/backtest execution yet)
self.ctx.submit(
    OrderIntent.trailing_stop_sell(InstrumentId("RELIANCE", "NSE"), 100, trail_percent=3.0)
)
```

Every intent submitted through `self.ctx.submit()` is queued and turned into an order by the runner after the current hook returns.

## Accessing State

The strategy has read-only access to state through its context (`self.ctx`):

```python
self.ctx.position(instrument_id)      # Net signed quantity held (float)
self.ctx.positions()                  # dict of all non-flat positions
self.ctx.cash()                       # Available cash as Money (integer minor units)
self.ctx.cash().to_major()            # Cash as float for display/sizing
self.ctx.busy(instrument_id)          # True while an order for this instrument is unfilled
self.ctx.instrument(instrument_id)    # Instrument metadata (lot size, tick size) or None
self.ctx.now()                        # Current simulated or real timestamp (unix ns)
self.ctx.now().to_major()             # Cash as float for display/sizing
```

All these read from the ledger maintained by the runner. They are O(1).

## Convenience Methods on Strategy

The `Strategy` base class provides these convenience methods (delegating to the context):

```python
self.position(instrument_id)    # Same as self.ctx.position(instrument_id)
self.busy(instrument_id)        # Same as self.ctx.busy(instrument_id)
self.log_event(event_type, message="", **data)  # Structured event log
self.buy(instrument_id, quantity, **data)       # Market buy + log EVENT_BUY
self.sell(instrument_id, quantity, **data)      # Market sell + log EVENT_SELL
self.submit(intent)             # Same as self.ctx.submit(intent)
```

## Indicators

Indicators are stateful objects that consume a stream and produce values. They are defined in `honba.strategies.indicators`:

```python
from honba.strategies.indicators import SMA, RSI, ATR, Bollinger

def on_start(self):
    self.sma_fast = SMA(period=20)
    self.sma_slow = SMA(period=50)
    self.rsi = RSI(period=14)
    self.atr = ATR(period=14)

def on_bar(self, bar):
    fast = self.sma_fast.update(bar.close)
    slow = self.sma_slow.update(bar.close)
    rsi = self.rsi.update(bar.close)
    atr = self.atr.update(bar.high, bar.low, bar.close)

    # Indicators return None until enough data has been seen
    if fast is None or slow is None:
        return

    if fast > slow and rsi < 70 and self.position(bar.instrument_id) == 0:
        self.buy(bar.instrument_id, size=100)
```

Indicators can be **vectorised** for research but are always **incremental** in the strategy. The `update()` method is O(1) per call, not O(n). This is important — an SMA(200) does not recompute a 200-element sum on every bar.

## Multi-Instrument Strategies

A strategy can trade multiple instruments. The `on_bar` callback receives one bar at a time; use `bar.instrument_id` to determine which:

```python
def on_bar(self, bar):
    sma = self.sma[bar.instrument_id].update(bar.close)
    if sma is None:
        return
    if bar.close > sma and self.position(bar.instrument_id) == 0:
        self.buy(bar.instrument_id, size=100)
```

## Configuration Parameters

Strategy parameters should be class attributes using the `Parameter` descriptor:

```python
from honba.strategies.base import Strategy, Parameter


class SmaCrossover(Strategy):
    name = "sma_crossover"
    fast_period = Parameter(default=20, min=5, max=50, step=5, description="Fast SMA")
    slow_period = Parameter(default=50, min=10, max=200, step=10, description="Slow SMA")
    target_position_pct = Parameter(default=0.02, min=0.005, max=0.1, step=0.005)

    def on_bar(self, bar):
        # Access parameters via self.fast_period, etc.
        ...
```

This makes strategies sweepable by the optimizer without code changes.

## Logging

```python
self.log_event("EVENT_BUY", "Signal triggered", symbol=bar.instrument_id.symbol, price=bar.close)
self.log_event("EVENT_WARNING", "Position exceeds threshold")
self.log_event("EVENT_ERROR", "Unexpected state", positions=dict(self.ctx.positions()))
```

Logs are captured by the engine and included in the backtest report. In live mode, logs are written to the configured sink (stdout, file, or an external system).

## A Complete Example

```python
from honba.entities.bar import Bar
from honba.entities.instrument import InstrumentId
from honba.strategies.base import Strategy, Parameter
from honba.strategies.indicators import SMA, ATR
from honba.entities.order import OrderIntent


class SmaAtrStrategy(Strategy):
    name = "sma_atr"
    fast_period = Parameter(default=20, min=5, max=50, step=5)
    slow_period = Parameter(default=50, min=10, max=200, step=10)
    atr_period = Parameter(default=14, min=5, max=30, step=1)
    risk_per_trade = Parameter(default=0.01, min=0.005, max=0.05, step=0.005)
    stop_atr_multiple = Parameter(default=2.0, min=1.0, max=4.0, step=0.5)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.sma_fast: dict[InstrumentId, SMA] = {}
        self.sma_slow: dict[InstrumentId, SMA] = {}
        self.atr: dict[InstrumentId, ATR] = {}

    def on_start(self):
        # Indicators are created in on_start so they start fresh each run
        for iid in self.universe:
            self.sma_fast[iid] = SMA(self.fast_period)
            self.sma_slow[iid] = SMA(self.slow_period)
            self.atr[iid] = ATR(self.atr_period)

    def on_bar(self, bar: Bar):
        iid = bar.instrument_id
        f = self.sma_fast[iid].update(bar.close)
        s = self.sma_slow[iid].update(bar.close)
        a = self.atr[iid].update(bar.high, bar.low, bar.close)

        if f is None or s is None or a is None:
            return

        pos = self.position(iid)

        if f > s and pos == 0:
            size = int(self.ctx.cash().to_major() * self.risk_per_trade / (a * 2))
            if size > 0:
                self.buy(iid, size)
                self.ctx.submit(
                    OrderIntent.stop_sell(iid, size, bar.close - self.stop_atr_multiple * a)
                )
        elif f < s and pos > 0:
            self.sell(iid, pos)

    def on_stop(self):
        self.log_event("INFO", "Backtest complete", final_equity=self.ctx.cash().to_major())
```

This strategy:
- Trades only when both SMAs and ATR have warmed up
- Sizes positions so that a 2-ATR stop loss equals 1% of equity
- Places a stop-loss order immediately after entry
- Uses `Parameter` descriptors for sweepable hyperparameters

## What Not to Do

- **Do not compute indicators in `__init__`.** Indicators are stateful and must be created in `on_start` so they start fresh each run.
- **Do not cache state across callbacks that the engine already caches.** `self.ctx.position()` is O(1); maintaining your own position dict is a bug waiting to happen.
- **Do not call `self.buy()` in `on_start`.** There is no market data yet, and the order will be rejected. Wait for the first bar.
- **Do not use `datetime.now()`.** Use `self.ctx.now()`, which reflects the simulated clock in backtest.
- **Do not use threads.** The engine is single-threaded for a reason.

## Next Steps

- **[Indicators](indicators.md)** — the full indicator catalogue.
- **[Strategy Styles](strategy_styles.md)** — event-driven, declarative and portfolio strategies.
- **[Position Management](position_management.md)** — sizing, stop-loss, and portfolio construction.