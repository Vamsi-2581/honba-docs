# Index Universes

A "universe" in Honba is a named set of instruments with a defined
selection rule and rebalance schedule. The three most commonly used
universes for Indian equity strategies are NIFTY 50, BANK NIFTY, and the
Alpha 30 family.

The critical difference from a static ticker list is that **constituents
change over time**. A backtest that uses today's NIFTY 50 list for a
2020 start date will include stocks that were not in the index then,
and exclude stocks that were. This introduces look-ahead bias and
invalidates the result.

*Last verified: 2026-09-30 against NSE Indices methodology documents.*

!!! note "What the Python API provides today"
    The Python surface for universes is `honba.markets.india.universes`:
    `resolve_universe(name, exchange="NSE", as_of=None)`, the built-in names
    `nifty50` and `nifty200_alpha30` (alias `alpha30`), and the point-in-time
    helpers `UniverseHistory`, `Constituent`, `register_universe_history`,
    `save_universe_history` and `load_universe_history`. The `Universe` trait,
    `Alpha30::nifty100()` and `announcement_date()` sections below describe the
    intended design; they are not part of the current Python API. See
    [Using a Universe in a Strategy](#using-a-universe-in-a-strategy).

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
audited results.[reference:11]

## BANK NIFTY

The banking sector index, and the most-traded index options contract in
the world by volume.

- **Constituents:** 12 (largest and most liquid Indian banks)
- **Rebalance:** semi-annual
- **Weighting:** free-float market cap
- **Expiry:** weekly and monthly options

BANK NIFTY does not have the same liquidity filter as NIFTY 50 because
it is sector-constrained.

## The Alpha 30 Family

NSE publishes two Alpha 30 indices:

### NIFTY100 Alpha 30

Consists of **30 companies from the NIFTY 100**, selected based on
**Jensen's Alpha** computed using 1-year trailing prices (adjusted for
corporate actions).[reference:12]

**Selection criteria:**[reference:13]

1. Stock must be a constituent of NIFTY 100 at review time.
2. Minimum listing history of 1 year.
3. Companies undergoing demerger/capital restructuring are eligible
   after twelve calendar months of trading on an ex-basis.
4. Jensen's Alpha is computed using 1-year trailing prices.
5. Top 30 stocks by alpha score form the index.

**Weighting:** a combination of alpha score and free-float market
capitalisation ("tilt weighted").[reference:14] Each stock is capped at
the lower of 5% or its free-float weight.

**Rebalance:** **quarterly**, using data from the six-month period
ending the last trading day of February, May, August, and November.[reference:15]

### NIFTY200 Alpha 30

Same methodology, but the parent universe is NIFTY 200 instead of
NIFTY 100. Stocks move out of the NIFTY 200 also move out of this
index.[reference:16]

The key practical consequence for backtesting: **Alpha 30 rebalances
four times a year**, not two. A backtest spanning 2020–2025 must apply
20 distinct constituent lists, each effective from its announcement
date (not its effective date — NSE announces changes in advance).

## The `Universe` Trait

    use honba_india::universes::{Universe, UniverseId};

    pub trait Universe {
        /// Constituents effective on a given date.
        fn constituents_at(&self, date: NaiveDate) -> Vec<Instrument>;

        /// Weight for a constituent on a given date.
        fn weight_at(&self, date: NaiveDate, instr: &Instrument) -> Option<f64>;

        /// Dates on which the constituent list changes.
        fn rebalance_dates(&self, from: NaiveDate, to: NaiveDate) -> Vec<NaiveDate>;

        /// Announcement date for a given effective date.
        fn announcement_date(&self, effective: NaiveDate) -> Option<NaiveDate>;
    }

Built-in implementations:

    Nifty50::new()
    BankNifty::new()
    Alpha30::nifty100()
    Alpha30::nifty200()

## Look-Ahead Safety

The most important property of the `Universe` trait is that
`constituents_at(date)` returns the constituents **as they were known on
that date**. If NSE announced on 2023-02-20 that a stock would enter the
index on 2023-03-31, then:

- A backtest iterating over dates through 2023-03-30 sees the old list.
- The new list becomes active on 2023-03-31.

`announcement_date()` is provided separately for strategies that want to
trade the announcement (a common "index inclusion" strategy).

    let u = Alpha30::nifty100();
    let entry = NaiveDate::from_ymd_opt(2023, 3, 31).unwrap();
    let announced = u.announcement_date(entry).unwrap();
    // → 2023-02-20

## Historical Constituent Data

`resolve_universe(name)` with no date returns **today's** list. It is
survivorship-biased by construction: every name that was delisted or dropped
along the way is already missing. With `as_of=<date>` it returns the
constituents effective on that date, from a history registered in the process
or from the Parquet catalog at `<data root>/constituents/<universe>.parquet`
(columns `symbol`, `included_from`, `excluded_on`). It never projects today's
list into the past: with no history it raises `ValueError`.

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

An interval is half-open: the inclusion day counts, the exclusion day does not.

!!! warning "No Alpha 30 history is registered yet"
    Honba does not ship constituent history for `nifty200_alpha30` (or any
    other built-in universe) today. `resolve_universe("nifty200_alpha30",
    as_of=...)` raises `ValueError` until you register or save one. Backtests
    that use the plain name run on today's 30 constituents and carry
    survivorship bias; see [Portfolio Construction](../strategies/portfolio_construction.md#survivorship-bias).

## The Rebalance Event

Backtests handle rebalances through the engine's event stream:

    [universe]
    name = "alpha30_nifty100"
    rebalance_mode = "trade"     # or "signal_only"

With `rebalance_mode = "trade"`, the engine emits synthetic
`UniverseRebalance` events on each effective date. Strategies can
subscribe to these and adjust positions accordingly. With
`signal_only`, the universe changes but no orders are generated — the
strategy decides what to do.

## Equal Weight vs Market Weight

Most strategies in practice use an equal-weight portfolio regardless of
the index's actual weights, because market-cap weights concentrate risk
in the top few names. For Alpha 30 specifically, equal weight is common
because the alpha-score tilt is already captured by constituent selection.

Note the split of responsibilities: the Alpha 30 is a **universe** (who is
eligible); "equal weight, rebalance periodically" is the **strategy**. In
Honba the strategy is `PortfolioStrategy` with an `EqualWeight` weighting
scheme, and the Alpha 30 enters only as a name:

    from honba.strategies.portfolio import EqualWeight, EveryNDays, NamedUniverse, PortfolioStrategy

    strategy = PortfolioStrategy(
        NamedUniverse("nifty200_alpha30"), EqualWeight(), EveryNDays(15)
    )

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

    let my_universe = StaticUniverse::new(vec![
        instrument("RELIANCE"),
        instrument("TCS"),
        instrument("INFY"),
    ]);

    // Or a filtered universe
    let liquid = FilteredUniverse::new(
        Nifty500::new(),
        |instr| instr.avg_daily_volume_30d > 1_000_000,
    );

Custom universes are useful for factor research where the selection
rule itself is the subject of the experiment.

## Source of Truth

- NIFTY 50, BANK NIFTY: NSE Indices methodology document (published
  annually, amended by circular).
- Alpha 30: NSE Indices "Methodology Document" PDF, section 13.
- Constituent changes: NSE Indices press releases, typically 4–6 weeks
  before effective date.
