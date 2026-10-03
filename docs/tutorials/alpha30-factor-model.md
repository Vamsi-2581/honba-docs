# Tutorial: Alpha30 Factor Model

Build a cross-sectional momentum factor on the NSE Alpha 30 universe.
This tutorial covers universe rebalancing, look-ahead safety, and
portfolio construction.

**Time:** 40 minutes.
**Prerequisites:** [Installation](../getting-started/installation.md),
[Universes](../india/universes.md).

## What You Will Build

A strategy that:

1. Runs on the NIFTY100 Alpha 30 universe (30 stocks selected by
   Jensen's Alpha).
2. Each month, ranks constituents by trailing 3-month return.
3. Buys the top 10 by momentum, equal-weighted.
4. Holds until the next monthly rebalance.

## Step 1: Understand the Universe

Alpha 30 is not a static list. Constituents are selected quarterly
based on Jensen's Alpha over a trailing 1-year window. A backtest
that uses today's list for a 2020 start date will include stocks that
were not in the index then.

Honba's `Alpha30` implementation handles this automatically:

    from honba.india.universes import Alpha30
    from datetime import date

    u = Alpha30.nifty100()

    # Constituents on a specific date
    constituents = u.constituents_at(date(2023, 1, 1))

    # Rebalance dates in a range
    dates = u.rebalance_dates(date(2023, 1, 1), date(2024, 12, 31))

    # When was a rebalance announced?
    announced = u.announcement_date(date(2023, 3, 31))
    print(f"Effective 2023-03-31, announced {announced}")

The `announcement_date` is important for strategies that trade the
index inclusion event.

## Step 2: The Strategy

Create `alpha30_momentum.py`:

    from honba.domain import Bar
    from honba.india.universes import Alpha30
    from honba.strategies.base import Strategy
    from honba.strategies.portfolio import EqualWeight


    class Alpha30Momentum(Strategy):
        lookback_days: int = 63       # ~3 months
        top_n: int = 10
        min_names: int = 5

        def on_start(self):
            self.universe_obj = Alpha30.nifty100()
            self.subscribe_bars(self.universe, timeframe="1d")
            self.portfolio = EqualWeight(universe=self.universe)
            self.portfolio.rebalance(schedule="monthly", at="first_open")
            self.risk.add_max_positions(self.top_n)
            self.risk.add_single_stock_limit(0.15)

        def on_rebalance(self):
            # Called on the first trading day of each month
            scores = {}
            for instr in self.universe:
                ret = self.trailing_return(instr, self.lookback_days)
                if ret is not None:
                    scores[instr] = ret

            if len(scores) < self.min_names:
                self.log.warning("Universe too small, skipping rebalance")
                return

            top = sorted(scores, key=scores.get, reverse=True)[:self.top_n]
            weights = self.portfolio.weights({i: 1.0 for i in top})
            self.rebalance_to(weights)

The `on_rebalance` callback is triggered by the portfolio scheduler.
It fires on the first trading day of each month at the open.

## Step 3: Important — Look-Ahead Safety

The most common bug in cross-sectional strategies is using the
universe as it exists *now* rather than as it existed *then*. Honba
handles this automatically when you use `Alpha30.nifty100()`:

- `constituents_at(date)` returns constituents effective on that date.
- The engine re-reads the universe on every rebalance.
- Names that were added mid-year do not appear in earlier rebalances.

If you were to hardcode a ticker list, none of this would apply and
the backtest would be invalid.

## Step 4: Configure and Run

Append:

    from honba.backtest import BacktestNode, BacktestConfig

    config = BacktestConfig(
        universe=Alpha30.nifty100(),
        start="2018-01-01",
        end="2024-12-31",
        initial_capital=1_000_000,
        cost_model="india_equity_delivery",
        latency_ms=50,
        calendar="NSE",
    )

    if __name__ == "__main__":
        node = BacktestNode(config)
        result = node.run(Alpha30Momentum)
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
            "alpha30_equal_weight_monthly",   # hold all 30, rebalance monthly
            "alpha30_top10_random_monthly",   # random top 10
        ],
    )
    print(comparison.table())

Expected shape of the output:

    Strategy                        CAGR    Sharpe   Max DD
    ─────────────────────────────────────────────────────────
    Alpha30Momentum (top 10)        16.8%   1.14     -17.2%
    NIFTY 50 buy-hold               12.4%   0.82     -22.1%
    Alpha30 equal-weight monthly    14.1%   0.96     -19.8%
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
        strategy=Alpha30Momentum,
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
- [Position Management](../strategies/position_management.md)
- [Statistical Validation](../backtesting/statistical_validation.md)
