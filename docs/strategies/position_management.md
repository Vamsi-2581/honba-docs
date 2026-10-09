# Position Management

Position sizing determines more of a strategy's outcome than signal
generation does. A strategy with a mediocre signal and good sizing
outperforms a strategy with a great signal and fixed-lot sizing.

Honba provides four sizing models and a portfolio construction layer.

## The Sizing Models

Every model answers the same question: given a signal, how many units
do I trade?

### 1. Fixed Size

    self.buy("RELIANCE", size=100)

Fixed size is simple and adequate for research but rarely appropriate
for production. It ignores price level and volatility entirely.

### 2. Fixed Fractional

Trade a fixed percentage of equity per position:

    size = self.size_fixed_fraction(
        instrument="RELIANCE",
        fraction=0.05,              # 5% of equity per position
    )
    self.buy("RELIANCE", size=size)

This is the most common model in practice. For an equal-weight
portfolio of N instruments, use `fraction = 1 / N`.

### 3. Volatility Scaled

Scale the position so that a fixed move in the underlying produces a
fixed loss:

    size = self.size_volatility_scaled(
        instrument="RELIANCE",
        risk_per_trade=0.01,         # 1% of equity at risk
        atr=self.atr[instr].value,
        stop_multiple=2.0,           # stop is 2 ATR away
    )

With a 2-ATR stop and 1% risk per trade, a losing trade costs 1% of
equity regardless of the instrument's volatility. This is the model
used by most systematic CTA-style strategies.

### 4. Kelly Criterion

For strategies with a known edge, size by Kelly:

    size = self.size_kelly(
        instrument="RELIANCE",
        win_rate=0.55,
        win_loss_ratio=1.8,
        fraction=0.5,                # half-Kelly (recommended)
    )

Full Kelly maximises long-run growth but produces drawdowns that are
intolerable in practice. Half-Kelly is the standard compromise: it
achieves 75% of the growth rate with roughly 50% of the drawdown.

## Stop Losses

A stop loss is a standing order that closes a position if the price
moves adversely. Honba supports four stop types:

### Fixed Stop

    self.sell_stop("RELIANCE", size=100, stop=2400.00)

### Percentage Stop

    self.sell_stop(
        "RELIANCE", size=100,
        stop=self.position("RELIANCE").avg_price * 0.95,   # 5% below entry
    )

### ATR Stop

    self.sell_stop(
        "RELIANCE", size=100,
        stop=bar.close - self.stop_atr_multiple * self.atr["RELIANCE"].value,
    )

### Trailing Stop

A trailing stop follows the market in your favour and never retreats. It is
the `TRAILING_STOP` order type, built with two `OrderIntent` factories:

    from honba.entities.instrument import InstrumentId
    from honba.domain.order import OrderIntent

    tcs = InstrumentId("TCS", "NSE")

    # Protect a LONG: SELL stop that trails below the high-water mark.
    long_stop = OrderIntent.trailing_stop_sell(tcs, 100, trail_percent=3.0)

    # Protect a SHORT: BUY stop that trails above the low-water mark.
    short_stop = OrderIntent.trailing_stop_buy(tcs, 100, trail_amount=15.0)

    # inside a Strategy callback:  self.submit(long_stop)

Semantics:

| | Long protection | Short protection |
|---|---|---|
| Factory | `trailing_stop_sell` | `trailing_stop_buy` |
| Side | SELL | BUY |
| Stop level | `trail_amount` (price units) or `trail_percent` below the highest price seen since placement | `trail_amount` or `trail_percent` above the lowest price seen since placement |
| Moves | up with the market, never down | down with the market, never up |
| Triggers | when price falls back to the stop; sells at market | when price rallies back to the stop; buys at market |

Give **exactly one** of `trail_amount` (finite, > 0) or `trail_percent`
(0 < p < 100); giving both or neither raises `ValueError`. A trailing stop
takes no `price` and no `trigger_price`, and every other order type rejects
the trail fields.

Trailing stops are **opt-in per adapter**. `AdapterCapabilities` does not
include `OrderType.TRAILING_STOP` in its default `order_types`; an adapter
must list it explicitly, and `place_order` raises `CapabilityError` on one
that does not.

!!! warning "Current status: Python interface only"
    The order type and factories exist in the Python domain model. There is no
    Rust or wire-format parity yet, the backtest engine and simulator do not
    execute trailing stops, and no broker adapter lists `TRAILING_STOP` in its
    capabilities. Today you can build and validate the intent, and an adapter
    can opt in, but you cannot backtest or trade one. Until then, emulate a
    trail by re-submitting a fixed `OrderIntent.stop_sell` at a new level.
    `DeclarativeStrategy` does not use trailing stops.

Stops are placed as real orders on the simulated exchange, not checked
in strategy code. This means a stop can trigger on an intrabar move,
which is how stops work in live trading.

**Important:** in a backtest, a stop order can only fill if the price
actually trades at the stop level. If your data is end-of-day bars and
the stop price falls between two bars, the stop will not trigger. This
is realistic; do not expect the backtest to fill stops that did not
occur.

## Target Orders

Target orders close a position at a profit:

    self.bracket(
        instrument="RELIANCE",
        side="buy",
        size=100,
        entry=2450.00,
        stop=2400.00,
        target=2550.00,
    )

The engine manages the OCO (one-cancels-other) logic: when either the
stop or the target fills, the other is cancelled.

!!! note "Declarative strategies"
    `DeclarativeStrategy` places its protective stop and target as two
    independent orders and does **not** cancel the survivor when the other
    fills (a TODO in the code). See
    [Strategy Styles](strategy_styles.md#known-limitation-no-automatic-oco).

## Position Tracking

    pos = self.position("RELIANCE")
    pos.size             # signed quantity (positive = long)
    pos.avg_price        # average entry price
    pos.unrealized_pnl   # mark-to-market P&L
    pos.realized_pnl     # P&L from closed portions
    pos.is_long
    pos.is_short
    pos.is_flat

For portfolio-level queries:

    self.total_exposure()             # sum of |position value|
    self.net_exposure()               # long value - short value
    self.gross_exposure()             # long + short value
    self.leverage()                   # gross exposure / equity

## Portfolio Construction

For multi-instrument strategies, the portfolio layer handles
allocation across positions:

    from honba.strategies.portfolio import EqualWeight, RiskParity, MeanVariance

    # Equal weight across N instruments
    self.portfolio = EqualWeight(universe=self.universe)

    # Risk parity: each instrument contributes equal volatility
    self.portfolio = RiskParity(
        universe=self.universe,
        lookback_days=60,
    )

    # Mean-variance (Markowitz) with shrinkage
    self.portfolio = MeanVariance(
        universe=self.universe,
        lookback_days=252,
        shrinkage=0.1,
        max_weight=0.1,
    )

Then in `on_bar`:

    def on_bar_universe(self, bars):
        signals = {i: self.signal(i, bars[i]) for i in bars}
        target_weights = self.portfolio.weights(signals)
        self.rebalance_to(target_weights)

`rebalance_to` computes the orders needed to move from the current
positions to the target weights and submits them, respecting the
configured cost model and minimum trade size.

## Rebalancing Frequency

    self.portfolio.rebalance(schedule="weekly")   # Monday open
    self.portfolio.rebalance(schedule="monthly")  # 1st trading day
    self.portfolio.rebalance(schedule="quarterly")
    self.portfolio.rebalance(schedule="threshold", threshold=0.05)

Threshold rebalancing trades only when target weights drift more than
`threshold` from current weights. This reduces turnover and cost, and is
usually superior to calendar rebalancing.

## Maximum Positions

    self.risk.add_max_positions(20)

The engine rejects new orders when the position limit is reached. This
is preferable to a strategy-level check because it applies uniformly
across all orders, including stops and targets.

## Diversification Constraints

    self.risk.add_sector_limit(
        max_pct=0.30,
        sector_lookup="nse_sector_map",
    )

    self.risk.add_single_stock_limit(max_pct=0.10)

    self.risk.add_correlated_group_limit(
        groups={"banks": ["HDFCBANK", "ICICIBANK", "SBIN", "AXISBANK"]},
        max_pct=0.25,
    )

These are enforced by the engine during order placement.

## Drawdown Control

    self.risk.add_max_drawdown(
        pct=0.20,
        on_breach="reduce_leverage",   # or "close_all" or "stop"
    )

The engine tracks running equity and reduces leverage (or closes all
positions) when the drawdown threshold is breached. This is a
portfolio-level control, not a per-strategy one — the drawdown is
measured on total equity.

## Position Merging Across Accounts

Honba supports multi-account execution, where the same strategy runs
across multiple broker accounts (for example, a joint account and an
HUF account). The engine merges target positions before submitting to
avoid self-trading:

    [execution]
    accounts = ["primary", "huf"]
    merge_targets = true

This is a WonderTrader-inspired feature. Without merging, two accounts
running the same strategy would place opposing orders that cancel each
other out on the exchange, incurring cost for no net position change.

## A Complete Example

    from honba.strategies.base import Strategy
    from honba.strategies.indicators import SMA, ATR
    from honba.strategies.portfolio import RiskParity


    class MomentumPortfolio(Strategy):
        lookback: int = 60
        top_n: int = 10
        rebalance_freq: str = "weekly"

        def on_start(self):
            self.subscribe_bars(self.universe, timeframe="1d")
            self.portfolio = RiskParity(
                universe=self.universe,
                lookback_days=self.lookback,
            )
            self.atr = {i: ATR(14) for i in self.universe}
            self.risk.add_max_positions(self.top_n)
            self.risk.add_single_stock_limit(0.15)
            self.risk.add_max_drawdown(
                pct=0.20, on_breach="reduce_leverage",
            )
            self.portfolio.rebalance(schedule=self.rebalance_freq)

        def on_bar_universe(self, bars):
            signals = {}
            for instr, bar in bars.items():
                ret = self.trailing_return(instr, self.lookback)
                signals[instr] = ret if ret is not None else 0.0

            top = sorted(signals, key=signals.get, reverse=True)[:self.top_n]
            target_weights = self.portfolio.weights({i: 1.0 for i in top})
            self.rebalance_to(target_weights)

        def on_stop(self):
            self.log.info("Final equity", equity=self.equity)

This strategy:

- Ranks the universe by trailing return
- Selects the top 10 momentum names
- Allocates risk-parity weights across them
- Rebalances weekly
- Enforces a 15% single-stock cap and 20% max drawdown
- Reduces leverage (rather than closing all) when drawdown triggers

## Next Steps

- **[Three-Tier Pipeline](../backtesting/three_tier_pipeline.md)** —
  how to validate these strategies rigorously.
- **[Cost Modeling](../backtesting/cost_modeling.md)** — why
  rebalancing frequency is a cost decision, not just a signal decision.
