# Writing Strategies

A Honba strategy is a Python class that inherits from `Strategy` and
implements lifecycle callbacks. The engine calls the callbacks in
response to market events; the strategy responds by placing, modifying,
or cancelling orders.

The strategy code is identical in backtest and live. The engine
determines which mode it is in.

## The Minimal Strategy

    from honba.domain import Bar
    from honba.entities import Instrument
    from honba.strategies.base import Strategy


    class BuyAndHold(Strategy):
        def on_start(self):
            self.buy("RELIANCE", size=100)

        def on_stop(self):
            self.close_all()

That is a complete strategy. `on_start` runs once when the engine
begins; `on_stop` runs once when it ends. Everything else is optional.

## Lifecycle Callbacks

The engine calls these methods, in this order, at the appropriate time:

| Callback | When |
|---|---|
| `on_start()` | Once, before any market data |
| `on_bar(bar)` | On every bar for a subscribed instrument |
| `on_quote(quote)` | On every quote tick (if subscribed) |
| `on_trade(trade)` | On every trade tick (if subscribed) |
| `on_order(event)` | On every order state change |
| `on_fill(fill)` | On every fill |
| `on_timer(name)` | On every scheduled timer |
| `on_stop()` | Once, after all data has been processed |

All callbacks are synchronous. The engine is single-threaded and
deterministic; do not spawn threads or use `asyncio` inside a strategy.

## Subscribing to Data

A strategy does not receive data for instruments it has not subscribed
to. Subscriptions are declared in `on_start`:

    def on_start(self):
        # Subscribe to 1-minute bars for a single instrument
        self.subscribe_bars("RELIANCE", timeframe="1m")

        # Subscribe to daily bars for a universe
        for instr in self.universe:
            self.subscribe_bars(instr, timeframe="1d")

        # Subscribe to quotes (for strategies that need bid/ask)
        self.subscribe_quotes("NIFTY50")

        # Subscribe to trades (for volume-aware strategies)
        self.subscribe_trades("BANKNIFTY")

Bars are aggregated **in the engine**, not in the strategy. A strategy
that subscribes to 1-minute bars receives fully-formed bars; it does
not see ticks.

## Placing Orders

The strategy API offers both convenience methods and full control:

    # Market order — fills at next available price
    self.buy("RELIANCE", size=100)
    self.sell("RELIANCE", size=100)

    # Limit order
    self.buy_limit("RELIANCE", size=100, price=2450.00)

    # Stop order (stop-loss)
    self.sell_stop("RELIANCE", size=100, stop=2400.00)

    # Bracket order (entry + stop + target)
    self.bracket(
        instrument="RELIANCE",
        side="buy",
        size=100,
        entry=2450.00,
        stop=2400.00,
        target=2550.00,
    )

    # Close position
    self.close("RELIANCE")
    self.close_all()

Every method returns a `client_order_id` (a string) that can be used to
correlate the order with subsequent events.

## Accessing State

The strategy has read-only access to:

    self.position(instr)          # Position: size, avg_price, unrealized_pnl
    self.positions                # dict of all positions
    self.cash                     # available cash
    self.equity                   # cash + unrealized P&L
    self.last_price(instr)        # most recent price (from cache)
    self.last_bar(instr, "1m")    # most recent bar of a timeframe
    self.now                      # current simulated or real timestamp

All these read from the cache maintained by the engine. They are O(1).

## Timers

Strategies can schedule callbacks that are not triggered by market data:

    def on_start(self):
        # Fire at 09:20 every trading day
        self.schedule_daily(
            at="09:20",
            callback="morning_rebalance",
        )

        # Fire 15 minutes before close
        self.schedule_session_relative(
            offset="-15m",
            callback="eod_flatten",
        )

        # Fire once after 30 bars have been seen
        self.schedule_bars(30, callback="warm_up_done")

    def morning_rebalance(self):
        # Rebalance logic
        pass

    def eod_flatten(self):
        self.close_all()

Timer callbacks are named methods on the strategy. They receive no
arguments.

## Indicators

Indicators are stateful objects that consume a stream and produce
values. They are defined in `honba.strategies.indicators`:

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

        if fast > slow and rsi < 70 and not self.position(bar.instrument).is_long:
            self.buy(bar.instrument, size=self.target_size)

Indicators can be **vectorised** for research but are always
**incremental** in the strategy. The `update()` method is O(1) per
call, not O(n). This is important — an SMA(200) does not recompute a
200-element sum on every bar.

## Multi-Instrument Strategies

A strategy can trade multiple instruments. The `on_bar` callback
receives one bar at a time; use `bar.instrument` to determine which:

    def on_bar(self, bar):
        sma = self.sma[bar.instrument].update(bar.close)
        if sma is None:
            return
        if bar.close > sma and not self.position(bar.instrument).is_long:
            self.buy(bar.instrument, size=self.target_size(bar.instrument))

For cross-sectional strategies (like Alpha30 factor rotation), the
engine provides a `on_bar_universe` callback that fires once per bar
interval with all instruments:

    def on_bar_universe(self, bars: dict[Instrument, Bar]):
        scores = {}
        for instr, bar in bars.items():
            scores[instr] = self.factor_score(instr, bar)
        top = sorted(scores, key=scores.get, reverse=True)[:10]
        self.rebalance_to(top)

## Risk Hooks

The engine runs pre-trade risk checks before every order. Strategies
can register additional checks:

    def on_start(self):
        self.risk.add_max_position_value(
            instrument="RELIANCE",
            max_value=200_000,
        )
        self.risk.add_max_leverage(2.0)
        self.risk.add_max_daily_loss(
            amount=50_000,
            on_breach="close_all",
        )

Risk rules are enforced by the engine, not by strategy code. A
strategy cannot place an order that violates a registered risk rule.

## P&L and Metrics

During the backtest, the strategy can query its own performance:

    def on_bar(self, bar):
        if self.equity < self.initial_capital * 0.9:
            # Down 10% from start
            self.close_all()
            self.stop()

The `self.stats` property exposes live metrics:

    self.stats.sharpe
    self.stats.max_drawdown
    self.stats.win_rate
    self.stats.profit_factor

These are updated as the backtest progresses.

## Configuration Parameters

Strategy parameters should be class attributes with type annotations:

    class SmaCrossover(Strategy):
        fast_period: int = 20
        slow_period: int = 50
        target_position_pct: float = 0.02
        stop_loss_atr_multiple: float = 2.0

    # Override at run time
    config = BacktestConfig(
        strategy_params={
            "fast_period": 10,
            "slow_period": 30,
        },
        ...
    )

This makes strategies sweepable by the optimizer without code changes.

## Logging

    self.log.info("Signal triggered", instrument=bar.instrument, price=bar.close)
    self.log.warning("Position exceeds threshold")
    self.log.error("Unexpected state", positions=self.positions)

Logs are captured by the engine and included in the backtest report.
In live mode, logs are written to the configured sink (stdout, file,
or an external system).

## A Complete Example

    from honba.strategies.base import Strategy
    from honba.strategies.indicators import SMA, ATR


    class SmaAtrStrategy(Strategy):
        fast_period: int = 20
        slow_period: int = 50
        atr_period: int = 14
        risk_per_trade: float = 0.01      # 1% of equity per trade
        stop_atr_multiple: float = 2.0

        def on_start(self):
            self.subscribe_bars(self.universe, timeframe="1d")
            self.fast = {i: SMA(self.fast_period) for i in self.universe}
            self.slow = {i: SMA(self.slow_period) for i in self.universe}
            self.atr = {i: ATR(self.atr_period) for i in self.universe}
            self.risk.add_max_leverage(1.0)
            self.risk.add_max_daily_loss(
                amount=self.initial_capital * 0.03,
                on_breach="close_all",
            )

        def on_bar(self, bar):
            instr = bar.instrument
            f = self.fast[instr].update(bar.close)
            s = self.slow[instr].update(bar.close)
            a = self.atr[instr].update(bar.high, bar.low, bar.close)

            if f is None or s is None or a is None:
                return

            pos = self.position(instr)

            if f > s and not pos.is_long:
                size = int(self.equity * self.risk_per_trade / (a * 2))
                if size > 0:
                    self.buy(instr, size=size)
                    self.sell_stop(
                        instr,
                        size=size,
                        stop=bar.close - self.stop_atr_multiple * a,
                    )
            elif f < s and pos.is_long:
                self.close(instr)

        def on_stop(self):
            self.log.info("Backtest complete", final_equity=self.equity)

This strategy:

- Trades only when both SMAs and ATR have warmed up
- Sizes positions so that a 2-ATR stop loss equals 1% of equity
- Places a stop-loss order immediately after entry
- Enforces a 3% daily loss limit at the engine level

## What Not to Do

- **Do not compute indicators in `__init__`.** Indicators are stateful
  and must be created in `on_start` so they start fresh each run.
- **Do not cache state across callbacks that the engine already caches.**
  `self.last_price()` is O(1); maintaining your own price dict is a bug
  waiting to happen.
- **Do not call `self.buy()` in `on_start`.** There is no market data
  yet, and the order will be rejected. Wait for the first bar.
- **Do not use `datetime.now()`.** Use `self.now`, which reflects the
  simulated clock in backtest.
- **Do not use threads.** The engine is single-threaded for a reason.

## Next Steps

- **[Indicators](indicators.md)** — the full indicator catalogue.
- **[Position Management](position_management.md)** — sizing,
  stop-loss, and portfolio construction.
