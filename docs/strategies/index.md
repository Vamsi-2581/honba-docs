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
- **[Step Manifests and Chaining](step_manifests_and_chaining.md)** — the
  `step.toml` manifest, the screen/filter/size/execute stage model and the
  chain validator.
- **[Pitfalls in the Examples and the Catalog](../backtesting/pitfalls_in_examples.md)**
  — the convention that every example and strategy documents its failure modes.

## The Strategy Catalog

The community catalog lives in the `honba-strategies` repository, grouped by
category and sub-category without numeric prefixes
(`momentum/trend/sma_crossover`, `mean_reversion/oscillator/rsi_oversold`,
`universe/alpha/momentum`). Each strategy directory has `strategy.py`,
`config.toml`, `README.md`, `tests/` and `backtest_result.json`.

`registry.json` indexes the catalog. Every entry has exactly the keys `name`,
`category`, `path`, `description` and `tags`; `tags` are indicator families
(`moving_average`, `trend`, `momentum`, `volatility`, `volume`,
`support_resistance`, `breadth`, `statistical`). The `name` is what
`load_named` resolves, and it can differ from the strategy's runtime id: the
registry entry `equal_weight` runs as `equal_weight_rebalance`.

Today the registry holds 17 strategies. A further 24 directories are
placeholders with a docstring-only `strategy.py`; they are not registered and
carry `status = "scaffold"` in their `config.toml`. Do not treat a scaffold as
an implemented strategy. Every registered strategy also has a `step.toml`; see
[Step Manifests and Chaining](step_manifests_and_chaining.md).

!!! warning "sma_crossover defaults"
    `sma_crossover` has two sets of defaults. The code (`Parameter()`
    declarations) defaults to fast 10 and slow 30; its `config.toml` sets 20
    and 50. Pass the periods explicitly if the difference matters to you.

## README convention: Usage, Possible output, Pitfalls

Every registered strategy README has the same three sections, and
`test_readme_sections.py` in `honba-strategies` fails when one is missing or too
short:

- **Usage**: a snippet that runs from the repo root with `honba` installed
  (load by registry name, run through `honba.strategies.testing.replay`).
- **Possible output**: what that snippet actually prints. Metrics from
  `backtest_result.json` are quoted only when its `status` is `completed`.
- **Pitfalls**: at least 3 bullets specific to the strategy's code and config,
  linking the generic
  [avoiding backtest biases](https://honba-labs.github.io/honba/book/backtesting/avoiding_backtest_biases.html)
  page.

Each `step.toml` also carries an optional `pitfalls` array of 2 to 4 short
strings; see [Step Manifests and Chaining](step_manifests_and_chaining.md).
Scaffolds are not registered, but their README keeps the same headings and says
`Status: scaffold`. The convention and an index of trap classes are on
[Pitfalls in the Examples and the Catalog](../backtesting/pitfalls_in_examples.md).

## Honest caveats in the catalog

These come from the strategy READMEs' own Pitfalls sections. Read the full
section before relying on a strategy.

| Strategy | Caveat |
|---|---|
| `vwap_reversion` | Intraday only. There is no minimum-bars-into-session guard, so the VWAP band is near zero at the open and a first-bar close slightly below hlc3 can trade (a BUY at 09:15 in the README's check). There is no stop, and no cap on same-session re-entries. |
| `turtle_20`, `kdj_cross` | The ATR stop is not a resting order. The strategy checks `bar.low <= stop` after the bar completes and sells at the close (or the next open with `fill_delay=1`), so a gap makes the realised loss larger than `atr_mult * ATR`. `turtle_20` is also a simplified Turtle (no ATR-unit sizing, no pyramiding). |
| `rsi2_connors`, `ifr2` | No stop and no time exit. A position is held until the exit condition fires, so one trade can lose a lot while the win rate stays high; look at the worst trade and the drawdown. `ifr2` also drops the Hilbert trend-mode filter of its source. |
| `alpha30_factor`, `alpha30_quality` | The names are misleading. Neither ranks or reads factor or fundamental data: both hold all 30 members at equal weight, so they are `equal_weight` with a different allocation or cadence. Do not read their results as evidence for a factor. |
| All `alpha30_*` and `equal_weight` on `nifty200_alpha30` | `point_in_time = true` does not remove survivorship bias yet. The only history the engine registers (`default_alpha30_history`) stamps today's 30 names as included from 2005-04-01 with no exits, so `resolve_universe("nifty200_alpha30", as_of=...)` returns all 30 for any date after that, including names listed years later. Register a real history with `register_universe_history` or provide a `constituents/` catalog file. |
| `sma_crossover` | Code defaults are fast 10 / slow 30; `config.toml` uses 20 / 50 (see the warning above). |
| Every single-instrument strategy | `replay(fill_delay=0)` fills at the signal bar's close and charges no costs. Use `fill_delay=1` or the engine's next-open execution, and run with the cost pack on. |

!!! warning "Catalog backtest results"
    `backtest_result.json` files do not always record the data, universe
    snapshot or cost model behind them. A strategy README says so when that is
    the case. Treat the numbers as a record that the code ran, not as evidence of
    an edge.

## The Three-Line Version

If you have written a Honba strategy before, this is the entire API:

```python
from honba.entities.instrument import InstrumentId
from honba.strategies.base import Strategy


class MyStrategy(Strategy):
    name = "my_strategy"

    def on_bar(self, bar):
        if self.signal(bar):
            self.buy(bar.instrument_id, self.size(bar))
```

Everything else in these documents is elaboration. Subscriptions are declared in the run config (e.g., `config.toml`), not in `on_start`.

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
- Caching all state in the runner's ledger, so `self.ctx.position()` and `self.ctx.cash()` are O(1).
- Exposing indicators as Rust objects behind Python wrappers.

The remaining Python overhead is the callback body itself. Keep it
small: compute a signal, decide, place an order, return.
