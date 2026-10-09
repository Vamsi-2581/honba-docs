# Portfolio Construction

A rebalanced portfolio answers four separate questions. Honba's portfolio
layer (`honba.strategies.portfolio`) makes each one a small, swappable part
and wires them together in `PortfolioStrategy`:

    Universe -> Selector -> WeightingScheme -> RebalanceSchedule -> PortfolioStrategy
    (who)       (which)     (how much)         (when)               (orders)

At each rebalance, `universe.members(as_of)` is filtered by the selector,
the weighting scheme assigns weights to what is left, and the base class
(`TargetWeightStrategy`, see [Strategy Styles](strategy_styles.md)) turns
target weights into orders in whole shares: exits, then trims, then buys. The
schedule decides *when* that happens. Everything is deterministic and
long-only, and time comes from bar event time (UTC), never the wall clock.

## Vocabulary

| Term | Meaning | Honba part |
|---|---|---|
| **Universe** | The set of instruments eligible to be held. Its *constituents* (members) can change over time. | `NamedUniverse("nifty50")`, `StaticUniverse([...])` |
| **Selection** | Narrowing the universe to the names to hold, e.g. the 10 with the best trailing return. | `SelectAll()` (default), `TopN(n, score)` |
| **Weighting scheme** | How capital is split across the selected names. | `EqualWeight()` (default), `InverseVolatility(lookback)` |
| **Rebalance policy** | When to trade back to the target weights. A *calendar* policy fires on dates; a *drift* policy fires when the portfolio has wandered too far from target. | `EveryNDays(n)`, `MonthlyFirstSession()`, `DriftBand(tolerance)`, `AnyOf(...)` |
| **Allocation** | Fraction of portfolio value to deploy; the rest is a cash buffer. | `allocation=0.98` (default) |

Note that the Alpha 30 is a **universe**. The *strategy* is "equal weight,
rebalance periodically".

## The Parts

**Universes.** `NamedUniverse(name, exchange="NSE", point_in_time=False)`
resolves a name known to the engine (`nifty50`, `nifty200_alpha30`).
`StaticUniverse(instruments)` is a fixed list (duplicates dropped, order
kept). The universe is re-read at every rebalance, so joiners are bought and
leavers are sold.

**Selectors.** `SelectAll()` keeps every member. `TopN(n, score, descending=True)`
keeps the `n` best-scoring members; names whose score is `None` (not enough
history) are dropped, so fewer than `n` may be held, and ties break by symbol.
Ready-made scores: `momentum(lookback)` (return over the last `lookback`
closes) and `low_volatility(lookback)` (calmer ranks higher). Until the
selector keeps something, the first rebalance is deferred rather than
rebalancing into nothing.

**Weighting.** `EqualWeight()` gives 1/N. `InverseVolatility(lookback=20)`
weights each name in proportion to 1/sigma, where sigma is the sample standard
deviation of simple returns over the last `lookback` closes; names without a
valid volatility get the mean of the valid inverse-vols, and if none are valid
the result is equal weight.

**Schedules.** All schedules make the first rebalance as soon as every member
has a price (or on the next day).

- `EveryNDays(n)`: then every `n` trading days (default 15).
- `MonthlyFirstSession()`: then on the first bar of each new UTC calendar month.
- `DriftBand(tolerance)`: then whenever the largest gap between a name's
  current weight and its target (`allocation * target`) reaches `tolerance`
  (e.g. `0.05`), but never on the same day as the previous rebalance.
- `AnyOf(a, b, ...)`: due when any child is due, e.g. a monthly rebalance that
  also fires on drift.

**History.** `PortfolioStrategy` keeps the last `history_len` closes per
instrument (default 64) for selectors and weighting. It must be at least the
longest lookback in use; `build_portfolio_strategy` sizes it automatically as
`max(64, longest lookback)`.

## Params Grammar

`PortfolioStrategy.from_params(params, name=None)` (an alias of
`build_portfolio_strategy`) builds the same strategy from plain,
TOML-friendly values. Unknown keys and malformed values raise `ValueError`
naming the key and the value.

| Key | Values | Default |
|---|---|---|
| `universe` (required) | A name such as `"nifty50"`, or a list of `"SYMBOL"` / `"SYMBOL:EXCHANGE"` | none |
| `exchange` | Exchange filled in for names and symbols without one | `"NSE"` |
| `point_in_time` | `true` to use historical snapshots (named universes only) | `false` |
| `weighting` | `"equal"`, `"inverse_vol"` (lookback 20), `"inverse_vol:30"` | `"equal"` |
| `schedule` | `"every:15d"`, `"monthly:first_session"`, `"drift:0.05"`; join with `+` for any-of | `"every:15d"` |
| `select` | `"top:10:momentum:126"`, `"top:10:low_vol:60"`; omit to keep everything | keep all |
| `allocation` | Number in (0, 1] | `0.98` |
| `history_len` | Integer, at least the longest lookback | auto-sized |

## Worked Examples

### 1. Equal-weight Alpha 30

Hold all 30 names of the NIFTY 200 Alpha 30 at equal weight and rebalance
every 15 trading days.

Code:

    from honba.strategies.portfolio import EqualWeight, EveryNDays, NamedUniverse, PortfolioStrategy

    strategy = PortfolioStrategy(
        NamedUniverse("nifty200_alpha30"),
        EqualWeight(),
        EveryNDays(15),
        allocation=0.995,
        name="equal_weight_rebalance",
    )

Equivalent `config.toml`:

    name     = "equal_weight_rebalance"
    symbol   = "MULTI"
    exchange = "NSE"

    [params]
    universe   = "nifty200_alpha30"
    weighting  = "equal"
    schedule   = "every:15d"
    allocation = 0.995

This is the catalog strategy `equal_weight`
(`honba-strategies/portfolio/rebalancing/equal_weight`, class
`EqualWeightRebalance`), a thin `PortfolioStrategy` subclass that builds its
parts from `[params]`. The runnable example is
`honba-examples/strategies/06_equal_weight_rebalance.py`.

### 2. Inverse-volatility, monthly

Same universe, but calmer names get more weight and the portfolio is rebalanced
on the first session of each month.

Code:

    from honba.strategies.portfolio import (
        InverseVolatility,
        MonthlyFirstSession,
        NamedUniverse,
        PortfolioStrategy,
    )

    strategy = PortfolioStrategy(
        NamedUniverse("nifty200_alpha30"),
        InverseVolatility(30),
        MonthlyFirstSession(),
        history_len=64,
        name="alpha30_inverse_vol",
    )

Equivalent `config.toml`:

    name     = "alpha30_inverse_vol"
    symbol   = "MULTI"
    exchange = "NSE"

    [params]
    universe  = "nifty200_alpha30"
    weighting = "inverse_vol:30"
    schedule  = "monthly:first_session"

Or directly from the params dict:

    from honba.strategies.portfolio import PortfolioStrategy

    strategy = PortfolioStrategy.from_params(
        {
            "universe": "nifty200_alpha30",
            "weighting": "inverse_vol:30",
            "schedule": "monthly:first_session",
        },
        name="alpha30_inverse_vol",
    )

### 3. Top-10 momentum with a drift band

From the NIFTY 50, hold the 10 names with the best return over the last 126
closes, equal weighted. Rebalance on the first session of each month, and also
whenever any position has drifted 5 percentage points from its target.

Code:

    from honba.strategies.portfolio import (
        AnyOf,
        DriftBand,
        EqualWeight,
        MonthlyFirstSession,
        NamedUniverse,
        PortfolioStrategy,
        TopN,
        momentum,
    )

    strategy = PortfolioStrategy(
        NamedUniverse("nifty50"),
        EqualWeight(),
        AnyOf(MonthlyFirstSession(), DriftBand(0.05)),
        TopN(10, momentum(126)),
        history_len=126,            # must cover the longest lookback
        name="nifty50_top10_momentum",
    )

Equivalent `config.toml`:

    name     = "nifty50_top10_momentum"
    symbol   = "MULTI"
    exchange = "NSE"

    [params]
    universe  = "nifty50"
    weighting = "equal"
    select    = "top:10:momentum:126"
    schedule  = "monthly:first_session+drift:0.05"
    # history_len is auto-sized to max(64, 126) = 126 when omitted

With a selector that needs history, nothing is held until at least one name has
126 closes; the strategy waits instead of rebalancing into an empty selection.

## Survivorship Bias

A universe built from *today's* constituents silently drops every name that was
delisted, merged or removed during your backtest, which flatters the results.

- `NamedUniverse` **ignores the bar date by default** and returns the engine's
  current list (`point_in_time=False`).
- `NamedUniverse(name, point_in_time=True)`, or `point_in_time = true` in
  params, asks for the snapshot effective on each bar's UTC date. This needs a
  registered history (`register_universe_history`, or a catalog Parquet file
  from `save_universe_history`) and **raises `ValueError`** if none exists. It
  never falls back to today's list.
- `point_in_time` applies to named universes only; combining it with an
  explicit symbol list is an error.
- **No Alpha 30 history is registered yet.** `nifty200_alpha30` with
  `point_in_time=True` fails until you provide one, and without it a backtest
  uses today's 30 names. Treat such results as survivorship-biased.

Registering a history:

    import datetime as dt
    from honba.markets.india.universes import (
        Constituent,
        UniverseHistory,
        register_universe_history,
    )

    register_universe_history(
        UniverseHistory(
            "nifty200_alpha30",
            (
                Constituent("TCS", dt.date(2020, 1, 1)),
                Constituent("INFY", dt.date(2020, 1, 1), dt.date(2022, 6, 30)),
            ),
        )
    )

Intervals are half-open: the inclusion day counts, the exclusion day does not.
See [Universes](../india/universes.md).

## Related

- [Strategy Styles](strategy_styles.md)
- [Universes](../india/universes.md)
- [Alpha30 Factor Model tutorial](../tutorials/alpha30-factor-model.md)
- [Position Management](position_management.md)
