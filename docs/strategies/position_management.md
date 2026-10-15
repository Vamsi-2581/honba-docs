# Position Management

Position sizing determines more of a strategy's outcome than signal generation does. A strategy with a mediocre signal and good sizing outperforms a strategy with a great signal and fixed-lot sizing.

Honba provides sizing helpers and a portfolio construction layer for multi-instrument strategies.

## The Sizing Models

Every model answers the same question: given a signal, how many units do I trade?

### 1. Fixed Size

```python
self.buy(InstrumentId("RELIANCE", "NSE"), 100)
```

Fixed size is simple and adequate for research but rarely appropriate for production. It ignores price level and volatility entirely.

### 2. Fixed Fractional

Trade a fixed percentage of equity per position:

```python
from honba.strategies.sizing import whole_shares

cash = self.ctx.cash().to_major()
size = whole_shares(cash, 0.05, bar.close)  # 5% of equity per position
self.buy(InstrumentId("RELIANCE", "NSE"), size)
```

This is the most common model in practice. For an equal-weight portfolio of N instruments, use `fraction = 1 / N`.

### 3. Volatility Scaled

Scale the position so that a fixed move in the underlying produces a fixed loss:

```python
from honba.strategies.sizing import whole_shares

cash = self.ctx.cash().to_major()
atr_value = self.atr[iid].value
if atr_value is not None:
    risk_amount = cash * 0.01  # 1% of equity at risk
    stop_distance = 2.0 * atr_value  # stop is 2 ATR away
    size = whole_shares(risk_amount, 1.0, stop_distance)
    self.buy(iid, size)
```

With a 2-ATR stop and 1% risk per trade, a losing trade costs 1% of equity regardless of the instrument's volatility. This is the model used by most systematic CTA-style strategies.

### 4. Kelly Criterion

For strategies with a known edge, size by Kelly:

```python
from honba.strategies.sizing import whole_shares

# Kelly fraction: (win_rate * win_loss_ratio - (1 - win_rate)) / win_loss_ratio
win_rate = 0.55
win_loss_ratio = 1.8
kelly = (win_rate * win_loss_ratio - (1 - win_rate)) / win_loss_ratio
half_kelly = kelly * 0.5

cash = self.ctx.cash().to_major()
size = whole_shares(cash, half_kelly, bar.close)
self.buy(InstrumentId("RELIANCE", "NSE"), size)
```

Full Kelly maximises long-run growth but produces drawdowns that are intolerable in practice. Half-Kelly is the standard compromise: it achieves 75% of the growth rate with roughly 50% of the drawdown.

## Stop Losses

A stop loss is a standing order that closes a position if the price moves adversely. Honba supports several stop types through `OrderIntent` factories:

### Fixed Stop (Stop-Market)

```python
from honba.entities.order import OrderIntent
from honba.entities.instrument import InstrumentId

iid = InstrumentId("RELIANCE", "NSE")
self.ctx.submit(OrderIntent.stop_sell(iid, 100, 2400.00))
```

### Percentage Stop

```python
pos = self.ctx.position(iid)
if pos > 0:
    entry_price = pos.avg_price  # Note: position() returns float, use ctx for details
    stop_price = entry_price * 0.95  # 5% below entry
    self.ctx.submit(OrderIntent.stop_sell(iid, pos, stop_price))
```

### ATR Stop

```python
atr_val = self.atr[iid].value
if atr_val is not None:
    stop_price = bar.close - 2.0 * atr_val
    self.ctx.submit(OrderIntent.stop_sell(iid, 100, stop_price))
```

### Trailing Stop

A trailing stop follows the market in your favour and never retreats. It is the `TRAILING_STOP` order type, built with two `OrderIntent` factories:

```python
from honba.entities.instrument import InstrumentId
from honba.entities.order import OrderIntent

iid = InstrumentId("TCS", "NSE")

# Protect a LONG: SELL stop that trails below the high-water mark.
long_stop = OrderIntent.trailing_stop_sell(iid, 100, trail_percent=3.0)

# Protect a SHORT: BUY stop that trails above the low-water mark.
short_stop = OrderIntent.trailing_stop_buy(iid, 100, trail_amount=15.0)

# Inside a Strategy callback:
self.ctx.submit(long_stop)
```

Semantics:

| | Long protection | Short protection |
|---|---|---|
| Factory | `trailing_stop_sell` | `trailing_stop_buy` |
| Side | SELL | BUY |
| Stop level | `trail_amount` (price units) or `trail_percent` below the highest price seen since placement | `trail_amount` or `trail_percent` above the lowest price seen since placement |
| Moves | up with the market, never down | down with the market, never up |
| Triggers | when price falls back to the stop; sells at market | when price rallies back to the stop; buys at market |

Give **exactly one** of `trail_amount` (finite, > 0) or `trail_percent` (0 < p < 100); giving both or neither raises `ValueError`. A trailing stop takes no `price` and no `trigger_price`, and every other order type rejects the trail fields.

Trailing stops are **opt-in per adapter**. `AdapterCapabilities` does not include `OrderType.TRAILING_STOP` in its default `order_types`; an adapter must list it explicitly, and `place_order` raises `CapabilityError` on one that does not.

!!! warning "Current status: Python interface only"
    The order type and factories exist in the Python domain model. There is no Rust or wire-format parity yet, the backtest engine and simulator do not execute trailing stops, and no broker adapter lists `TRAILING_STOP` in its capabilities. Today you can build and validate the intent, and an adapter can opt in, but you cannot backtest or trade one. Until then, emulate a trail by re-submitting a fixed `OrderIntent.stop_sell` at a new level. `DeclarativeStrategy` does not use trailing stops.

Stops are placed as real orders on the simulated exchange, not checked in strategy code. This means a stop can trigger on an intrabar move, which is how stops work in live trading.

**Important:** in a backtest, a stop order can only fill if the price actually trades at the stop level. If your data is end-of-day bars and the stop price falls between two bars, the stop will not trigger. This is realistic; do not expect the backtest to fill stops that did not occur.

## Target Orders

Target orders close a position at a profit. There is no `bracket()` convenience method; submit the entry, stop and target as separate intents:

```python
iid = InstrumentId("RELIANCE", "NSE")

# Entry
self.ctx.submit(OrderIntent.market_buy(iid, 100))

# Stop-loss (after entry fill, in on_fill)
self.ctx.submit(OrderIntent.stop_sell(iid, 100, 2400.00))

# Take-profit (after entry fill, in on_fill)
self.ctx.submit(OrderIntent.limit_sell(iid, 100, 2550.00))
```

The engine does **not** manage OCO (one-cancels-other) logic automatically for hand-written strategies. When either the stop or the target fills, the other remains active unless you cancel it, which you can do with `self.cancel(intent)`.

!!! note "Declarative strategies"
    `DeclarativeStrategy` cancels the surviving protective leg automatically when the other fills or the position goes flat. See [Strategy Styles](strategy_styles.md#oco-protective-orders).

## Position Tracking

```python
pos_qty = self.ctx.position(iid)           # Signed quantity (positive = long)
positions = self.ctx.positions()           # dict of all non-flat positions
cash = self.ctx.cash()                     # Available cash as Money
cash_major = cash.to_major()               # Cash as float
```

The `Position` object with `avg_price`, `unrealized_pnl`, `is_long`, etc. is not directly exposed on `StrategyContext`. Use `ctx.position()` for the signed quantity and track entry prices yourself if needed.

For portfolio-level queries:

```python
value = cash_major
for iid, qty in self.ctx.positions().items():
    price = self._last_prices.get(iid)  # Track last price in on_bar
    if price:
        value += qty * price

total_exposure = sum(abs(qty) * price for iid, (qty, price) in ...)
net_exposure = sum(qty * price for ...)
gross_exposure = sum(abs(qty) * price for ...)
leverage = gross_exposure / value if value > 0 else 0
```

## Portfolio Construction

For multi-instrument strategies, the portfolio layer handles allocation across positions. See [Portfolio Construction](portfolio_construction.md) for the full `PortfolioStrategy` API.

```python
from honba.strategies.portfolio import (
    EqualWeight, InverseVolatility, EveryNDays, MonthlyFirstSession,
    NamedUniverse, PortfolioStrategy, TopN, momentum
)

# Equal weight across N instruments
strategy = PortfolioStrategy(
    NamedUniverse("nifty50"),
    EqualWeight(),
    EveryNDays(15),
    allocation=0.98,
    name="equal_weight",
)

# Risk parity: each instrument contributes equal volatility
strategy = PortfolioStrategy(
    NamedUniverse("nifty200_alpha30"),
    InverseVolatility(30),
    MonthlyFirstSession(),
    history_len=64,
    name="inverse_vol",
)

# Top-10 momentum with a drift band
from honba.strategies.portfolio import AnyOf, DriftBand

strategy = PortfolioStrategy(
    NamedUniverse("nifty50"),
    EqualWeight(),
    AnyOf(MonthlyFirstSession(), DriftBand(0.05)),
    TopN(10, momentum(126)),
    history_len=126,
    name="nifty50_top10_momentum",
)
```

These strategies use `TargetWeightStrategy` under the hood, which:
- Values the portfolio (cash + open positions at the last close)
- Sizes each name to `value * allocation * weight` in whole shares
- Submits market orders: first exits (held names with weight 0), then trims (sells), then top-ups / entries (buys), each group in symbol order
- Skips busy or unpriced names and ignores differences under one share

## Rebalancing Frequency

```python
# Calendar rebalancing
EveryNDays(15)              # Every 15 trading days
MonthlyFirstSession()       # First bar of each UTC calendar month

# Drift-based rebalancing
DriftBand(0.05)             # When max weight gap reaches 5%

# Combined
AnyOf(MonthlyFirstSession(), DriftBand(0.05))
```

Threshold rebalancing trades only when target weights drift more than `threshold` from current weights. This reduces turnover and cost, and is usually superior to calendar rebalancing.

## Risk Controls

Risk is enforced by the **runner** (via `RiskStage`), not by strategy code. Pass a risk stage when constructing the runner:

```python
from honba.risk import RiskLimits, RiskStage
from honba.strategies.runner import StrategyRunner

limits = RiskLimits()
limits.max_notional = 10_000_000
limits.max_positions = 20
limits.max_leverage = 2.0
# ... other limits

stage = RiskStage(limits)
runner = StrategyRunner(strategy, execution_port, risk=stage, risk_limits=limits)
```

The runner checks every order against the risk limits before submitting to the execution port. A refusal never reaches the port and is audited.

Available limits (see `honba.risk.RiskLimits`):
- `max_notional` — maximum order notional
- `max_positions` — maximum concurrent positions
- `max_leverage` — maximum portfolio leverage
- `max_daily_loss` — maximum daily loss (on breach: reject orders)
- `max_drawdown` — maximum portfolio drawdown (on breach: reduce leverage or reject)
- `max_position_value` — per-instrument maximum position value
- `sector_limits` — per-sector maximum allocation
- `single_stock_limit` — per-stock maximum allocation
- `correlated_group_limits` — correlated group maximum allocation

These are enforced by the runner during order submission, not by strategy callbacks.

## Drawdown Control

Drawdown is measured on total portfolio equity (cash + marked positions). Configure it in `RiskLimits` and the runner will enforce it.

## Multi-Account Execution

Honba supports multi-account execution where the same strategy runs across multiple broker accounts. The engine merges target positions before submitting to avoid self-trading:

```toml
[execution]
accounts = ["primary", "huf"]
merge_targets = true
```

This is a WonderTrader-inspired feature. Without merging, two accounts running the same strategy would place opposing orders that cancel each other out on the exchange, incurring cost for no net position change.

## A Complete Example

```python
from honba.entities.bar import Bar
from honba.entities.instrument import InstrumentId
from honba.strategies.base import Strategy, Parameter
from honba.strategies.indicators import SMA, ATR
from honba.entities.order import OrderIntent
from honba.strategies.sizing import whole_shares


class MomentumPortfolio(Strategy):
    name = "momentum_portfolio"
    lookback = Parameter(default=60, min=20, max=252, step=10)
    top_n = Parameter(default=10, min=5, max=30, step=5)
    rebalance_days = Parameter(default=15, min=5, max=60, step=5)
    risk_per_trade = Parameter(default=0.01, min=0.005, max=0.03, step=0.005)
    stop_atr_multiple = Parameter(default=2.0, min=1.0, max=4.0, step=0.5)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.sma: dict[InstrumentId, SMA] = {}
        self.atr: dict[InstrumentId, ATR] = {}
        self._rebalance_counter = 0

    def on_start(self):
        for iid in self.universe:
            self.sma[iid] = SMA(self.lookback)
            self.atr[iid] = ATR(14)

    def on_bar(self, bar: Bar):
        iid = bar.instrument_id
        sma_val = self.sma[iid].update(bar.close)
        atr_val = self.atr[iid].update(bar.high, bar.low, bar.close)

        if sma_val is None or atr_val is None:
            return

        pos = self.position(iid)

        # Simple momentum: price above SMA = long
        if bar.close > sma_val and pos == 0 and not self.busy(iid):
            cash = self.ctx.cash().to_major()
            size = whole_shares(cash * self.risk_per_trade, 1.0, atr_val * self.stop_atr_multiple)
            if size > 0:
                self.buy(iid, size)
                self.ctx.submit(
                    OrderIntent.stop_sell(iid, size, bar.close - self.stop_atr_multiple * atr_val)
                )
        elif bar.close < sma_val and pos > 0:
            self.sell(iid, pos)

    def on_stop(self):
        self.log_event("INFO", "Final equity", equity=self.ctx.cash().to_major())
        # Note: intents submitted here are discarded (run is over).
```

This strategy:
- Ranks the universe by trailing return (simplified: price > SMA)
- Selects top names implicitly via per-instrument signals
- Sizes positions using volatility-scaled sizing (ATR-based)
- Places stop-loss orders after entry
- Uses `Parameter` descriptors for sweepable hyperparameters

## Next Steps

- **[Strategy Styles](strategy_styles.md)** — event-driven, declarative and portfolio strategies.
- **[Portfolio Construction](portfolio_construction.md)** — universe, selection, weighting and rebalance schedule.
- **[Three-Tier Pipeline](../backtesting/three_tier_pipeline.md)** — how to validate these strategies rigorously.
- **[Cost Modeling](../backtesting/cost_modeling.md)** — why rebalancing frequency is a cost decision, not just a signal decision.