# Strategies

A Honba strategy is a Python class that inherits from `Strategy` and
implements lifecycle callbacks. The engine calls the callbacks in
response to market events; the strategy responds by placing orders.

## The Documents

- **[Writing Strategies](writing_strategies.md)** — the `Strategy` base
  class, lifecycle callbacks, data subscriptions, and order placement.
- **[Indicators](indicators.md)** — the full indicator catalogue: trend,
  momentum, volatility, volume, and India-specific indicators.
- **[Strategy Styles](strategy_styles.md)** — event-driven, declarative
  and portfolio strategies, with a Jesse comparison and when to use each.
- **[Portfolio Construction](portfolio_construction.md)** — universe,
  selection, weighting and rebalance schedule, with the `params` grammar.
- **[Position Management](position_management.md)** — sizing models,
  stop losses, and portfolio construction.

## The Three-Line Version

If you have written a Honba strategy before, this is the entire API:

    class MyStrategy(Strategy):
        def on_start(self):
            self.subscribe_bars(self.universe, timeframe="1d")

        def on_bar(self, bar):
            if self.signal(bar):
                self.buy(bar.instrument, size=self.size(bar))

Everything else in these documents is elaboration.

## What Makes a Good Strategy

A strategy in Honba is expected to be:

- **Explicit about its edge.** A comment or docstring stating what
  market inefficiency the strategy is exploiting.
- **Explicit about its failure modes.** Under what conditions does it
  lose money, and how much?
- **Robust to parameters.** Small changes to the parameters should not
  collapse the strategy's performance.
- **Cost-aware.** The strategy should have a view on how much turnover
  it can afford.

These are not enforced by the engine. They are enforced by the
[statistical validation](../backtesting/statistical_validation.md)
suite, which will fail a strategy that violates them.

## A Note on Python Performance

Strategies run in Python, which means a per-bar callback has Python
overhead. For strategies that trade a handful of instruments on daily
bars, this is negligible. For strategies that trade hundreds of
instruments on minute bars, it is not.

The engine mitigates this by:

- Aggregating bars in Rust before the Python callback.
- Caching all state in Rust, so `self.last_price(instr)` is O(1).
- Exposing indicators as Rust objects behind Python wrappers.

The remaining Python overhead is the callback body itself. Keep it
small: compute a signal, decide, place an order, return.
