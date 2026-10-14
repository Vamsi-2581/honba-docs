# Tutorial: SIP Backtest

Backtest a Systematic Investment Plan on a mutual fund scheme. This
tutorial covers AMFI NAV data, direct-vs-regular plans, and XIRR
computation.

**Time:** 15 minutes.
**Prerequisites:** [Installation](../getting-started/installation.md),
[Mutual Funds](../india/mutual_funds.md).

## What You Will Build

A SIP that invests ₹10,000 on the 5th of every month into a single
scheme, from 2015 through 2024, and reports the XIRR, CAGR, and total
invested.

## Step 1: Load Scheme Data

AMFI publishes daily NAVs for every scheme in India. Import a decade
of NAV history:

    from honba.india.mutual_funds import AmfiNavImporter

    importer = AmfiNavImporter(catalog="./data/catalog")
    importer.import_history(
        start="2015-01-01",
        end="2024-12-31",
        schemes=[120503],      # Axis Bluechip - Direct
    )

`120503` is the AMFI scheme code for Axis Bluechip Fund - Direct
Growth. To find another scheme's code:

    from honba.india.mutual_funds import search_schemes

    results = search_schemes("bluechip", plan="direct")
    for s in results[:5]:
        print(s.code, s.name, s.amc)

## Step 2: The SIP Backtest

Create `sip_backtest.py`:

    from honba.india.mutual_funds import SipBacktest, Frequency

    bt = SipBacktest(
        scheme=120503,
        amount=10_000,
        frequency=Frequency.Monthly,
        day_of_month=5,
        start="2015-01-01",
        end="2024-12-31",
    )

    if __name__ == "__main__":
        result = bt.run()
        print(result.summary())
        result.tearsheet().save("sip_axis_bluechip.html")

The `day_of_month=5` is the target investment date. If the 5th is a
non-trading day (weekend or holiday), the installment executes on the
next trading day.

## Step 3: Run It

    python3 sip_backtest.py

Expected output:

    SIP Backtest: Axis Bluechip Fund - Direct Growth
    Scheme code:  120503
    Period:       2015-01-01 to 2024-12-31

    Installments:       120
    Total invested:     ₹12,00,000
    Final value:        ₹22,84,600

    XIRR (annualized):  12.4%
    CAGR (NAV):         11.8%
    Volatility:         14.2%
    Max drawdown:       -24.6%

    Cost breakdown:
      Expense ratio:    ₹48,600    (annualized ~1.2%)
      Stamp duty:       ₹600       (0.005% per installment)
      ─────────────────────────────
      Total costs:      ₹49,200

## Step 4: Understand XIRR vs CAGR

Two numbers appear in the output and they are often confused:

- **CAGR (11.8%)** measures the fund's NAV growth over the period.
  It ignores your investment schedule entirely.
- **XIRR (12.4%)** measures the return on *your money*, taking into
  account that each installment was invested at a different time and
  price.

For a SIP, XIRR is the number that matters. It is higher here because
the fund performed better in the later period, when more of your
money was invested. If the fund had done better early, XIRR would be
lower than CAGR.

## Step 5: Direct vs Regular

The scheme code `120503` is a direct plan. Regular plans carry an
additional distribution expense of 0.5–1.5% annually. Over 10 years,
this compounds substantially:

    from honba.india.mutual_funds import Scheme

    direct = Scheme.by_code(120503)     # Direct
    regular = Scheme.by_code(120465)    # Regular plan of same fund

    direct_bt = SipBacktest(direct, 10_000, ...).run()
    regular_bt = SipBacktest(regular, 10_000, ...).run()

    print(f"Direct final:  ₹{direct_bt.final_value:,}")
    print(f"Regular final: ₹{regular_bt.final_value:,}")
    print(f"Difference:    ₹{direct_bt.final_value - regular_bt.final_value:,}")

Typical result: the regular plan ends ₹1.5–2.5 lakh behind over 10
years on a ₹10,000/month SIP. This is the entire cost of a
distributor's commission.

## Step 6: Compare Against a Benchmark

A fund's return is meaningless without a benchmark. Compare against
the category average and the index:

    from honba.india.mutual_funds import compare_to_category, compare_to_index

    category = compare_to_category(
        scheme=120503,
        start="2015-01-01",
        end="2024-12-31",
    )
    print(category.rank())       # where the fund ranked in its category
    print(category.category_xirr())  # average XIRR of the category

    index = compare_to_index(
        scheme=120503,
        index="NIFTY50_TRI",     # total return index
        start="2015-01-01",
        end="2024-12-31",
    )
    print(index.alpha())          # excess return over the index

Most large-cap equity funds underperform the NIFTY 50 TRI after fees
over 10-year windows. If this fund did, it may not justify the active
management cost over an index fund.

## Step 7: The Behavior Question

SIP backtests assume perfect discipline — the investor contributes on
schedule, every month, regardless of market conditions. This is the
hardest part of a SIP in practice.

A more honest comparison:

    from honba.india.mutual_funds import SipBacktest, BehavioralModel

    # Model an investor who skips installments during drawdowns
    bt = SipBacktest(
        scheme=120503,
        amount=10_000,
        frequency=Frequency.Monthly,
        day_of_month=5,
        start="2015-01-01",
        end="2024-12-31",
        behavior=BehavioralModel(
            skip_when_drawdown_gt=0.15,    # skip if fund down > 15% from peak
            skip_when_up_gt=0.20,          # skip if fund up > 20% in 3 months
        ),
    )

The behavioral model typically produces a 1–2% lower XIRR than the
disciplined SIP, because investors who skip installments systematically
miss the best entry points.

A NAV dated D is not known during D. In the `honba-examples` mutual-fund SIP
(`mutual_funds/02_sip_backtest.py`), an instalment dated on a day with no
published NAV buys the next published NAV, never a past one, and an instalment
with no NAV within a week is skipped. Check how your own loader treats a
non-publication day.

## What to Try Next

1. **Step-up SIP.** Increase the monthly amount by 10% each year.
2. **Multi-scheme SIP.** Split the ₹10,000 across a large-cap,
   mid-cap, and small-cap fund.
3. **Rebalanced SIP.** Annually rebalance across schemes to a target
   allocation.

## Related

- [Mutual Funds](../india/mutual_funds.md)
- [Cost Model](../india/cost_model.md)
- [Examples](../examples/index.md): `honba-examples` also has a SIP on a listed
  instrument (`backtesting/08_sip_buy_and_hold.py`) and a screener-to-SIP chain
  (`backtesting/10_chained_screener_and_sip.py`).
- [Step Manifests and Chaining](../strategies/step_manifests_and_chaining.md)
