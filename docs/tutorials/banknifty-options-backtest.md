# Tutorial: BANKNIFTY Options Backtest

Build and backtest a short straddle on BANKNIFTY monthly options. This
tutorial covers expiry cycles, Greeks, and the STT asymmetry between
premium and exercise that trips up most options backtests.

**Time:** 45 minutes.
**Prerequisites:** [Installation](../getting-started/installation.md),
[Options](../india/options.md).

## What You Will Build

A strategy that:

1. On each monthly expiry day, sells an at-the-money call and put.
2. Holds the position until the following expiry.
3. Closes positions before expiry if delta exceeds a threshold.

## Step 1: Load Options Data

BANKNIFTY options data is not part of the synthetic dataset. Import a
truncated real dataset that ships with the repository:

    from honba.algo_import.nse import import_nse_bhavcopy_fno

    import_nse_bhavcopy_fno(
        path="tests/data/FO_Bhavcopy_2023_2024.zip",
        catalog="./data/catalog",
        underlying="BANKNIFTY",
    )

This imports daily EOD option data for BANKNIFTY contracts covering
2023–2024. Check that it loaded:

    from honba.algo.data import ParquetCatalog
    catalog = ParquetCatalog("./data/catalog")
    print(catalog.list_options(underlying="BANKNIFTY"))

## Step 2: The Strategy

Create `banknifty_straddle.py`:

    from honba.strategies.base import Strategy
    from honba.india.options import OptionChain, Index, OptionType


    class BankNiftyStraddle(Strategy):
        name = "banknifty_straddle"
        lots: int = 1
        lot_size: int = 15
        delta_close_threshold: float = 0.35

        def on_start(self):
            self.subscribe_bars("BANKNIFTY", timeframe="1d")
            self.subscribe_expiries(Index.BankNifty, cycle="monthly")
            self.chain = OptionChain(Index.BankNifty)

        def on_expiry(self, expiry):
            # Runs at 09:20 on the monthly expiry day
            if self.chain.is_expiring_today(expiry):
                return

            atm = self.chain.atm_strike(self.last_price("BANKNIFTY"))
            call = self.chain.option(atm, OptionType.CE, expiry)
            put = self.chain.option(atm, OptionType.PE, expiry)

            self.sell(call, qty=self.lots * self.lot_size)
            self.sell(put, qty=self.lots * self.lot_size)
            self.log.info("Opened straddle", strike=atm, expiry=expiry)

        def on_bar(self, bar):
            # Check Greeks; close if either leg is too directional
            if not self.positions:
                return

            for pos in list(self.positions.values()):
                greeks = self.chain.greeks(pos.instrument)
                if abs(greeks.delta) > self.delta_close_threshold:
                    self.close(pos.instrument)
                    self.log.warning("Closed on delta breach", delta=greeks.delta)

        def on_stop(self):
            self.close_all()

## Step 3: Configure the Backtest

Append the configuration:

    from honba import Honba

    if __name__ == "__main__":
        session = Honba.backtest(
            BankNiftyStraddle(),
            symbol="BANKNIFTY",
            start="2023-01-01",
            end="2024-12-31",
            cash=2_000_000,
        )
        result = session.run()
        result.print_report()

## Step 4: Run It

    python3 banknifty_straddle.py

Expected shape of the output:

    Backtest: BankNiftyStraddle
    Period:   2023-01-01 to 2024-12-31
    Expiries: 24 monthly cycles

    Initial capital:  ₹2,000,000
    Final equity:     ₹2,184,600

    Cycles:           24
    Winning cycles:   14 (58.3%)
    Worst cycle:      -₹42,800  (2024-06 expiry)
    Best cycle:       +₹58,200  (2023-08 expiry)

    Sharpe:           0.72
    Max drawdown:     -18.4%

    Cost breakdown:
      STT (premium):    ₹48,200   (58.2%)
      STT (exercise):   ₹22,400   (27.0%)
      Brokerage:        ₹9,600    (11.6%)
      Exchange + SEBI:  ₹2,140    (2.6%)
      GST:              ₹460      (0.6%)
      ─────────────────────────────
      Total costs:      ₹82,800   (4.1% of capital over 2 years)

## Step 5: Understand the STT Asymmetry

Two STT charges appear in this backtest:

- **STT on premium** (0.15% of option premium, sell side). This is
  charged when you open the short position.
- **STT on exercise** (0.15% of intrinsic value, buy side). This is
  charged when a leg expires in-the-money.

The second charge is what catches people off guard. If a leg expires
deep in-the-money — say ₹3,000 of intrinsic value on a ₹200 premium —
the STT on exercise is 15x the STT on premium for that leg.

In this backtest, exercise STT is 27% of total costs. A naive cost
model that only accounts for premium STT would understate the cost by
roughly 37%.

## Step 6: Check the Greeks Behaviour

The strategy closes positions when delta exceeds 0.35. Look at how
often this triggered:

    result.log.query(level="WARNING").to_dataframe()

If delta breaches are frequent, the position is being closed early
and the strategy is really a delta-hedged vol trade rather than a
short straddle. If they are rare, the strategy is holding to expiry
most of the time.

Both are valid strategies, but they have different risk profiles. The
backtest should tell you which one you actually built.

## Step 7: Validate

Options strategies are more prone to overfitting than equity
strategies because the parameter space (strike selection, delta
threshold, exit timing) is larger relative to the signal.

    from honba.algo_analytics import walk_forward, monte_carlo

    wf = walk_forward(session, BankNiftyStraddle,
                     train_months=12, test_months=6, n_folds=4)
    mc = monte_carlo(session, BankNiftyStraddle, iterations=500)

    print(wf.summary())
    print(mc.summary())

Two things to look for:

1. **Cycle-level P&L concentration.** If the top 3 of 24 cycles
   produce more than 50% of P&L, the strategy is a small number of
   lucky months away from being unprofitable.
2. **Regime dependence.** Short straddles lose money during volatility
   spikes. Check how the strategy performed during the March 2023
   banking crisis and the June 2024 election volatility.

## What to Try Next

1. **Add a volatility filter.** Skip the trade if India VIX is above
   a threshold.
2. **Try iron condors.** Selling further-out strikes reduces premium
   but caps the loss.
3. **Vary the exit rule.** Instead of closing on delta breach, close
   on a fixed profit target or DTE threshold.

## Related

- [Options](../india/options.md)
- [Cost Modeling](../backtesting/cost_modeling.md)
- [Position Management](../strategies/position_management.md)
