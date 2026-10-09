# Tutorial: Alpha30 Factor Model

Build a cross-sectional momentum factor on the NSE Alpha 30 universe.
This tutorial covers universe rebalancing, look-ahead safety, and
portfolio construction.

**Time:** 40 minutes.
**Prerequisites:** [Installation](../getting-started/installation.md),
[Universes](../india/universes.md).

## What You Will Build

A strategy that:

1. Runs on the NIFTY200 Alpha 30 universe (30 stocks selected by
   Jensen's Alpha).
2. Each month, ranks constituents by trailing 3-month return.
3. Buys the top 10 by momentum, equal-weighted.
4. Holds until the next monthly rebalance.

!!! note "Runnable core and design-level steps"
    The strategy in Step 2 is built from the portfolio layer
    (`honba.strategies.portfolio`) and runs against the current API. The
    `Alpha30` universe helper, `BacktestNode`/`BacktestConfig` wiring,
    `compare_to_benchmarks` and `validate` in later steps describe the intended
    research workflow and may differ from the current Python API; check the
    [API reference](../api/python/index.md) before copying them.

## Step 1: Understand the Universe

Alpha 30 is a **universe**, not a strategy. Constituents are selected
quarterly based on Jensen's Alpha over a trailing 1-year window. A backtest
that uses today's list for a 2020 start date will include stocks that
were not in the index then.

Honba names the universe `nifty200_alpha30` (alias `alpha30`):

    from honba.markets.india.universes import resolve_universe

    today = resolve_universe("nifty200_alpha30")   # 30 InstrumentIds, today's list

`resolve_universe(..., as_of=date)` returns the constituents effective on a
past date, but only from a registered history (`register_universe_history`) or
the Parquet catalog; with none it raises `ValueError`. **No Alpha 30 history
is registered yet**, so a backtest on the plain name uses today's
constituents and is survivorship-biased. Read
[Portfolio Construction](../strategies/portfolio_construction.md#survivorship-bias)
before trusting results.

## Step 2: The Strategy

"Each month, hold the 10 best trailing-return names of the universe, equal
weighted" is a composition of universe, selector, weighting and schedule.
Create `alpha30_momentum.py`:

    from honba.strategies.portfolio import (
        EqualWeight,
        MonthlyFirstSession,
        NamedUniverse,
        PortfolioStrategy,
        TopN,
        momentum,
    )


    def build_strategy() -> PortfolioStrategy:
        return PortfolioStrategy(
            NamedUniverse("nifty200_alpha30"),
            EqualWeight(),
            MonthlyFirstSession(),
            TopN(10, momentum(63)),      # ~3 months of trading days
            history_len=64,
            name="alpha30_momentum",
        )

There is no `on_bar` body: at each rebalance the framework re-reads the
universe, ranks the members by their return over the last 63 closes, keeps
the top 10, sells names that dropped out, and trims or tops up the rest to
equal weight in whole shares. `TopN` drops names without 63 closes of history,
so the first rebalance waits until some names can be scored.

The same strategy as plain params (what a `config.toml` `[params]` table
holds):

    PortfolioStrategy.from_params(
        {
            "universe": "nifty200_alpha30",
            "select": "top:10:momentum:63",
            "schedule": "monthly:first_session",
            "history_len": 64,
        },
        name="alpha30_momentum",
    )

## Step 3: Important - Look-Ahead Safety

The most common bug in cross-sectional strategies is using the
universe as it exists *now* rather than as it existed *then*.

- `NamedUniverse("nifty200_alpha30")` ignores the bar date by default and
  returns the current list. That is convenient and survivorship-biased.
- `NamedUniverse("nifty200_alpha30", point_in_time=True)` re-reads the
  universe as of each bar's date, so names added mid-year do not appear in
  earlier rebalances. It needs registered history and raises `ValueError`
  otherwise; it never falls back to today's list.

If you hardcode a ticker list (`StaticUniverse`), none of this applies and
the backtest is invalid for any period in which membership changed.

## Step 4: Configure and Run

Append:

    from honba.backtest import BacktestNode, BacktestConfig

    config = BacktestConfig(
        universe="nifty200_alpha30",
        start="2018-01-01",
        end="2024-12-31",
        initial_capital=1_000_000,
        cost_model="india_equity_delivery",
        latency_ms=50,
        calendar="NSE",
    )

    if __name__ == "__main__":
        node = BacktestNode(config)
        result = node.run(build_strategy())
        print(result.summary())
        result.tearsheet().save("alpha30_momentum.html")

Run it:

    python3 alpha30_momentum.py

## Step 5: Compare Against Benchmarks

Cross-sectional strategies must beat naive alternatives, or the
complexity is not justified. Compare against two benchmarks:

    from honba.algo_analytics import compare_to_benchmarks

    comparison = compare_to_benchmarks(
        result=result,
        benchmarks=[
            "nifty50_buy_hold",
            "equal_weight",                   # catalog strategy: hold all 30 equal-weighted
            "alpha30_top10_random_monthly",   # random top 10
        ],
    )
    print(comparison.table())

Expected shape of the output:

    Strategy                        CAGR    Sharpe   Max DD
    ─────────────────────────────────────────────────────────
    Alpha30Momentum (top 10)        16.8%   1.14     -17.2%
    NIFTY 50 buy-hold               12.4%   0.82     -22.1%
    Alpha30 equal-weight (catalog)  14.1%   0.96     -19.8%
    Alpha30 top-10 random monthly   11.2%   0.71     -24.6%

The third benchmark is the important one. If a random selection of 10
names performs as well as your momentum-selected 10, the momentum
signal is not adding value — you are just capturing the Alpha 30
premium.

## Step 6: The Rebalance Timing Question

Alpha 30 rebalances quarterly, and NSE announces changes 4–6 weeks
before the effective date. This creates a tradeable event:

- **Strategy A:** trade on the announcement (buy names being added).
- **Strategy B:** trade on the effective date (what the strategy
  above does).
- **Strategy C:** avoid the rebalance window entirely (close positions
  two weeks before effective date, reopen two weeks after).

Strategy A is a pure event strategy and may have different risk
characteristics. Honba's `announcement_date()` supports it:

    def on_announcement(self, effective_date, added, removed):
        for instr in added:
            self.buy(instr, size=self.size_for(instr))
        for instr in removed:
            self.close(instr)

Backtest both and compare. The result is often surprising — the
announcement premium has been arbitraged away in some years and not
others.

## Step 7: Validate

    from honba.algo_analytics import validate

    report = validate(
        node=node,
        strategy=build_strategy(),
        num_trials=50,     # you tested ~50 parameter combinations
        config=ValidationConfig(min_deflated_sharpe=0.5),
    )
    report.print()

The deflated Sharpe correction is especially important for factor
strategies because the search space (lookback windows, top-N, rebalance
frequency) is large. A strategy that looks good on 5 years of data
after testing 50 combinations may be entirely a selection artifact.

## What to Try Next

1. **Multi-factor.** Combine momentum with a low-volatility factor.
   Ranks from each factor are averaged.
2. **Different lookback.** Momentum is famously sensitive to the
   lookback window. Test 21, 42, 63, 126, 252 days.
3. **Volatility scaling.** Scale weights so that each position
   contributes equal volatility rather than equal capital.

## Related

- [Universes](../india/universes.md)
- [Portfolio Construction](../strategies/portfolio_construction.md)
- [Position Management](../strategies/position_management.md)
- [Statistical Validation](../backtesting/statistical_validation.md)
