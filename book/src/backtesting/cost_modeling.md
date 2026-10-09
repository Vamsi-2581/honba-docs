# Cost Modeling in Backtests

The Indian cost model is documented in
[India → Cost Model](https://honba-labs.github.io/honba-docs/india/cost_model.md). This document explains how
it integrates with the backtest engine and how to measure a strategy's
sensitivity to it.

## Why Costs Are Not Optional

Consider a strategy that trades once per day on a ₹10 lakh portfolio:

    250 trades/year × ₹20 brokerage × 2 sides          = ₹10,000
    STT (delivery, 0.1% per side × 250 × ₹5L)          = ₹2,50,000
    Exchange + SEBI + stamp duty + GST                 ≈ ₹15,000
    ─────────────────────────────────────────────────────────────
    Total annual cost                                  ≈ ₹2,75,000

That is **27.5% of the initial capital** in costs alone. A backtest
that ignores costs would show this strategy as profitable with any
signal that has a positive gross edge. In reality, the strategy needs a
gross return above 27.5% just to break even.

Every backtest in Honba applies the full cost model by default. There
is no flag to turn it off. If you want to see gross P&L for research
purposes, use `result.gross_pnl` — the net P&L is always the primary
number.

## Where Costs Are Applied

The cost model sits between the strategy's order and the fill event:

    Strategy          → SubmitOrder
    RiskEngine        → (approved)
    CostModel         → (compute cost preview)
    SimulatedExchange → (match against book)
    CostModel         → (deduct cost from fill)
    Fill event        → delivered to strategy

Costs are deducted at **fill time**, not at order time. If an order is
partially filled, cost is deducted proportionally. If an order is
rejected, no cost is applied.

## The `CostModel` Trait

    pub trait CostModel {
        fn cost_of(&self, fill: &Fill, context: &MarketContext) -> Cost;
        fn name(&self) -> &str;
    }

    pub struct Cost {
        pub stt: Decimal,
        pub exchange_fee: Decimal,
        pub sebi_fee: Decimal,
        pub stamp_duty: Decimal,
        pub gst: Decimal,
        pub brokerage: Decimal,
        pub total: Decimal,
    }

The engine stores the per-fill cost and aggregates it into the
backtest report. `result.costs.breakdown()` shows the contribution of
each component.

## Presets

    from honba.backtest.cost import (
        IndiaEquityDelivery,
        IndiaEquityIntraday,
        IndiaFuturesIndex,
        IndiaOptionsIndex,
    )

    # Delivery equity
    model = IndiaEquityDelivery(
        brokerage=Brokerage.per_order(20.0),
        state="Maharashtra",
    )

    # Intraday equity (MIS)
    model = IndiaEquityIntraday(
        brokerage=Brokerage.percent(0.03, min=20.0),
    )

    # Index futures (NIFTY)
    model = IndiaFuturesIndex(
        brokerage=Brokerage.per_order(20.0),
    )

    # Index options (NIFTY/BANK NIFTY)
    model = IndiaOptionsIndex(
        brokerage=Brokerage.per_order(20.0),
        stt_on_exercise=True,
    )

## Sensitivity Analysis

Because cost assumptions can dominate a result, Honba provides a
sensitivity sweep:

    from honba.backtest.cost import sensitivity

    sweep = sensitivity(
        strategy=SmaCrossover,
        brokerage=[0, 10, 20, 30, 50],
        latency_ms=[0, 25, 50, 100, 200],
    )

    sweep.plot().save("cost_latency_sensitivity.html")

The plot shows Sharpe ratio as a function of both brokerage and latency.
A robust strategy has a broad region of viability; a fragile one has a
narrow peak.

## Common Mistakes

### Mistake 1: Applying GST to the full cost stack

GST is 18% on brokerage + exchange fees + SEBI fee, not on STT or
stamp duty. Honba's model gets this right; hand-rolled models often
do not.

### Mistake 2: Using delivery STT for intraday

Delivery equity pays 0.1% STT on **both** sides. Intraday equity pays
0.025% on the **sell** side only. A 4x difference. Using the wrong one
silently biases the result by hundreds of basis points.

### Mistake 3: Forgetting stamp duty is buy-only

Stamp duty is charged only on the buy side. Applying it to both sides
roughly doubles the cost contribution (though it is a small component).

### Mistake 4: Ignoring options exercise STT

If an option position expires in-the-money, STT is charged on the
intrinsic value, not the premium. For a deep ITM option, this can be
larger than the premium STT.

### Mistake 5: Zero brokerage

Some discount brokers offer zero brokerage on equity delivery. This
does not mean costs are zero — STT, exchange fees, and stamp duty
still apply, and they are usually larger than brokerage would have
been.

## Brokerage Models

    from honba.backtest.cost import Brokerage

    # Flat per order
    Brokerage.per_order(20.0)

    # Percentage of turnover, with a minimum
    Brokerage.percent(0.03, min=20.0)

    # The lower of per-order and percentage (Zerodha-style)
    Brokerage.lower_of(per_order=20.0, percent=0.03)

    # Zero brokerage
    Brokerage.zero()

    # Tiered by turnover
    Brokerage.tiered([
        (0, 0.03),
        (1_000_000, 0.02),
        (5_000_000, 0.01),
    ])

## Segment-Specific Rules

The cost model enforces segment consistency. If a strategy places an
intraday order but the cost model is `IndiaEquityDelivery`, the engine
raises a configuration error. This prevents the common bug of
inadvertently using the wrong cost model.

    # This will fail at backtest start:
    config = BacktestConfig(
        strategy_mode="intraday",
        cost_model="india_equity_delivery",
    )
    # → ConfigurationError: intraday strategy cannot use delivery cost model

    # Use the matching model:
    config = BacktestConfig(
        strategy_mode="intraday",
        cost_model="india_equity_intraday",
    )

## Historical Rate Pinning

STT and exchange fees change over time. A backtest spanning 2020–2025
crosses multiple rate regimes. Honba pins rates to named versions so a
backtest reproduces exactly:

    [cost]
    model = "india_equity_delivery"
    rates_version = "finance_act_2024"     # or "finance_act_2026"

The `rates_version` selects a snapshot of all applicable rates. When
the Finance Act changes STT, a new version is added without modifying
the old one. Historical backtests continue to use the correct rate for
their period.

    # Rate history in honba-market/src/india/costs/ or honba.markets.india.costs
    finance_act_2020.rs
    finance_act_2024.rs
    finance_act_2026.rs

If a backtest spans a rate change, set `rates_version = "auto"` and the
engine applies the correct rate per day:

    [cost]
    rates_version = "auto"

## Reporting

Every backtest report includes a cost breakdown:

    result.costs.summary()
    # Total costs:        ₹2,74,850  (27.5% of initial capital)
    #   STT:              ₹2,50,000  (91.0%)
    #   Brokerage:        ₹10,000    ( 3.6%)
    #   Exchange fees:    ₹8,000     ( 2.9%)
    #   GST:              ₹5,400     ( 2.0%)
    #   Stamp duty:       ₹1,200     ( 0.4%)
    #   SEBI fee:         ₹250       ( 0.1%)
    #
    # Gross return:       22.3%
    # Net return:         -5.2%
    # Cost drag:          27.5 pp

This report is the single most important output of a backtest. A
strategy whose gross return is 22% and net return is -5% is not a
strategy; it is a cost generator.

## Next Steps

- **[Latency Modeling](latency_modeling.md)** — the other invisible
  cost.
- **[Statistical Validation](statistical_validation.md)** — how to
  decide whether a net-positive result is real.
