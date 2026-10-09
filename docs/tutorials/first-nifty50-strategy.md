# Tutorial: Your First NIFTY 50 Strategy

Build, backtest, and validate a moving-average crossover on the NIFTY
50 universe. This tutorial requires no broker credentials and runs
entirely against the small synthetic dataset that ships with the
repository.

**Time:** 20 minutes.
**Prerequisites:** [Installation](../getting-started/installation.md).

## What You Will Build

A strategy that:

1. Subscribes to daily bars for NIFTY 50 constituents.
2. Buys when a 20-day SMA crosses above a 50-day SMA.
3. Sells when it crosses back below.
4. Sizes positions per name.

## Step 1: Create the Strategy File

Create `first_strategy.py` in the workspace root:

    from honba import Honba, Strategy
    from honba.indicators import SMA
    from honba.markets.india.universes import resolve_universe


    class SmaCrossover(Strategy):
        name = "sma_crossover"
        fast_period: int = 20
        slow_period: int = 50

        def on_start(self):
            self.fast = SMA(self.fast_period)
            self.slow = SMA(self.slow_period)

        def on_bar(self, bar):
            f = self.fast.update(bar.close)
            s = self.slow.update(bar.close)
            if f is None or s is None:
                return

            pos = self.position(bar.instrument_id)
            if f > s and pos <= 0:
                self.buy(bar.instrument_id, quantity=10.0)
            elif f < s and pos > 0:
                self.sell(bar.instrument_id, quantity=pos)

## Step 2: Configure the Backtest

Append to `first_strategy.py`:

    if __name__ == "__main__":
        session = Honba.backtest(
            SmaCrossover(),
            symbol="INFY",
            start="2023-01-01",
            end="2024-12-31",
            cash=1_000_000,
        )
        result = session.run()
        result.print_report()

## Step 3: Run It

    python3 first_strategy.py

Expected output (numbers will differ slightly on synthetic data):

    Strategy:  SmaCrossover
    Period:    2023-01-01 → 2024-12-31 (500 bars)
    Trades:    412  (Win rate: 54.1%)
    Sharpe:    1.18
    CAGR:      14.2%
    Max DD:    -11.4%
    Final:     ₹1,298,450 (+29.8%)

## Step 4: Validate

Before believing that Sharpe of 1.18, run the statistical validation:

    honba validate first_strategy.py --trials 1

Output:

    Trials tested:          1
    Deflated Sharpe Ratio:  0.89  [PASS >= 0.85]
    Max DD duration:        62 days
    Verdict:                ACCEPT for paper trading
