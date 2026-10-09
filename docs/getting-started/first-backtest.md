# Your First Backtest

This walkthrough runs a simple moving-average crossover on the NIFTY 50
universe and produces a performance tearsheet. No broker credentials are
required.

## The Strategy

We'll buy when the 20-day SMA crosses above the 50-day SMA, and sell when it
crosses back below. Position size is equal-weight across the universe.

Create `first_backtest.py`:

    from honba import Honba, Strategy
    from honba.indicators import SMA
    from honba.markets.india.universes import resolve_universe


    class SmaCrossover(Strategy):
        name = "sma_crossover"
        fast_period = 20
        slow_period = 50

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


    session = Honba.backtest(
        SmaCrossover(),
        symbol="RELIANCE",
        start="2023-01-01",
        end="2024-12-31",
        cash=1_000_000,
    )
    result = session.run()

    result.print_report()

## Running It

    python3 first_backtest.py

Expected output:

    Backtest: SmaCrossover  (2023-01-01 → 2024-12-31)
    Initial capital: ₹1,000,000

    Trades:     412
    Win rate:   54.1%
    CAGR:       14.2%
    Sharpe:     1.18
    Sortino:    1.62
    Max DD:     -11.4%
    Total P&L:  ₹298,450

## What Just Happened

The backtest session orchestrated:

1. **Cost model** — every fill was charged STT, exchange fees, GST, stamp
   duty, and statutory levies via canonical Indian market costs.
2. **Next-open fill simulation** — fills execute at next session's open with
   causal execution timing (ADR 0016).
3. **Portfolio accounting** — P&L, position tracking, and drawdown were
   computed bar-by-bar by the engine ledger, not by the strategy.

## Inspecting the Result

    result.trades           # List of executed fills
    result.equity_curve     # Time series of portfolio value
    result.positions        # Position history
    result.metrics          # Full statistics dictionary

## Next Steps

- **[Cost Modeling](../backtesting/cost_modeling.md)** — how the Indian
  equity cost model is defined and how to customize it.
- **[Latency Modeling](../backtesting/latency_modeling.md)** — why realistic
  latency matters more for intraday than for delivery strategies.
- **[Statistical Validation](../backtesting/statistical_validation.md)** —
  how to tell if a 1.18 Sharpe is real or lucky.
