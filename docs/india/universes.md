# Index Universes

A "universe" in Honba is a named set of instruments with a defined
selection rule and rebalance schedule. The two most commonly used
built-in universes for Indian equity strategies are NIFTY 50 and the
NIFTY 200 Alpha 30.

The critical difference from a static ticker list is that **constituents
change over time**. A backtest that uses today's NIFTY 50 list for a
2020 start date will include stocks that were not in the index then,
and exclude stocks that were. This introduces look-ahead / survivorship bias
and invalidates the result.

*Last verified: 2026-09-30 against NSE Indices methodology documents.*

## The Python Universe API

The Python surface for universes lives in `honba.markets.india.universes`:

- `resolve_universe(name, exchange="NSE", as_of=None)`
- Built-in universe names: `"nifty50"`, `"nifty200_alpha30"` (and alias `"alpha30"`)
- Point-in-time constituent history: `UniverseHistory`, `Constituent`,
  `register_universe_history`, `save_universe_history`, `load_universe_history`,
  and `default_alpha30_history`.

## NIFTY 50

The flagship large-cap index. 50 stocks selected from the NIFTY 100
based on free-float market capitalisation, liquidity, and trading
frequency.

- **Parent universe:** NIFTY 100
- **Constituents:** 50
- **Rebalance:** semi-annual (March and September)
- **Weighting:** free-float market cap

Selection criteria require a minimum listing history of one year, 100%
trading frequency over the last year, and positive net worth as per
audited results.

## The Alpha 30 Family

NSE publishes the **NIFTY 200 Alpha 30** index, consisting of **30 companies
from the NIFTY 200**, selected based on **Jensen's Alpha** computed using
1-year trailing prices (adjusted for corporate actions).

**Selection criteria:**

1. Stock must be a constituent of NIFTY 200 at review time.
2. Minimum listing history of 1 year.
3. Companies undergoing demerger/capital restructuring are eligible
   after twelve calendar months of trading on an ex-basis.
4. Jensen's Alpha is computed using 1-year trailing prices.
5. Top 30 stocks by alpha score form the index.

**Rebalance:** **quarterly**, using data from the six-month period
ending the last trading day of February, May, August, and November.

A backtest spanning 2020–2025 must apply distinct constituent lists,
each effective from its historical snapshot.

## Historical Constituent Data & Point-in-Time Resolution

`resolve_universe(name)` with no date returns **today's** list. It is
convenient for live trading but survivorship-biased if applied backwards.
With `as_of=<date>` it returns the constituents effective on that date,
from registered history or from the Parquet catalog at
`<data root>/constituents/<universe>.parquet` (columns `symbol`,
`included_from`, `excluded_on`). It refuses to project today's list into
the past: with no history it raises `ValueError`.

```python
import datetime as dt
from honba.markets.india.universes import (
    Constituent,
    UniverseHistory,
    register_universe_history,
    resolve_universe,
)

register_universe_history(
    UniverseHistory(
        "nifty50",
        (
            Constituent("TCS", dt.date(2015, 1, 1)),
            Constituent("YESBANK", dt.date(2015, 1, 1), dt.date(2020, 3, 19)),
        ),
    )
)
resolve_universe("nifty50", as_of=dt.date(2019, 1, 3))   # TCS and YESBANK
```

An interval is half-open: the inclusion day counts, the exclusion day does not.

For Alpha 30, baseline historical constituents are registered via `default_alpha30_history()`
and can be overridden or extended with custom snapshots.

## Equal Weight vs Market Weight

Most strategies in practice use an equal-weight portfolio regardless of
the index's actual weights, because market-cap weights concentrate risk
in the top few names. For Alpha 30 specifically, equal weight is common
because the alpha-score tilt is already captured by constituent selection.

Note the split of responsibilities: the Alpha 30 is a **universe** (who is
eligible); "equal weight, rebalance periodically" is the **strategy**. In
Honba the strategy is `PortfolioStrategy` with an `EqualWeight` weighting
scheme, and the Alpha 30 enters only as a name:

```python
from honba.strategies.portfolio import EqualWeight, EveryNDays, NamedUniverse, PortfolioStrategy

strategy = PortfolioStrategy(
    NamedUniverse("nifty200_alpha30"), EqualWeight(), EveryNDays(15)
)
```

The catalog strategy `equal_weight`
(`honba-strategies/portfolio/rebalancing/equal_weight`) is exactly this,
configured from `config.toml`. See
[Portfolio Construction](../strategies/portfolio_construction.md).

## Using a Universe in a Strategy

`NamedUniverse(name, exchange="NSE", point_in_time=False)` wraps
`resolve_universe` for the portfolio layer and `StaticUniverse([...])` wraps a
fixed list. By default `NamedUniverse` ignores the bar date and returns the
engine's current list; with `point_in_time=True` it asks for the snapshot
effective on the bar's UTC date and raises `ValueError` if no history is
registered.

## Custom Universes

Any user-defined set of instruments can be wrapped as a universe:

```python
from honba.entities.instrument import InstrumentId
from honba.strategies.portfolio import StaticUniverse

my_universe = StaticUniverse([
    InstrumentId("RELIANCE", "NSE"),
    InstrumentId("TCS", "NSE"),
    InstrumentId("INFY", "NSE"),
])
```

Custom universes are useful for factor research where the selection
rule itself is the subject of the experiment.

## Source of Truth

- NIFTY 50: NSE Indices methodology document (published annually, amended by circular).
- Alpha 30: NSE Indices "Methodology Document" PDF, section 13.
- Constituent changes: NSE Indices press releases, typically 4–6 weeks before effective date.
