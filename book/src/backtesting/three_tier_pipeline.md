# The Three-Tier Backtesting Pipeline

Honba does not have "a backtester." It has three, each appropriate for a
different stage of research. A strategy is expected to pass through all
three before being considered validated.

| Tier | Engine | Runtime | Purpose |
|---|---|---|---|
| 1 | Vectorized (pandas/Arrow) | Seconds to minutes | Rapid screen of many candidates |
| 2 | Event-driven (Rust) | Minutes | Realistic simulation of survivors |
| 3 | Concurrent async (Barter-style) | Minutes for thousands | Parameter sweeps, Monte Carlo |

The tiers are not alternatives. They are stages. Using only Tier 2 for
100 candidate strategies wastes hours. Using only Tier 1 for the final
result produces a number that is not trustworthy.

## Tier 1 — Vectorized Pre-Filter

### What It Does

Tier 1 computes signals and P&L across a universe using vectorized
operations on Arrow/Parquet data. It does not simulate order book
interaction, latency, or fills. It assumes you can trade at the next
bar's open or close, with a cost deduction.

### When To Use It

- You have 50+ strategy ideas and need to eliminate the obviously
  unprofitable ones.
- You are exploring parameter ranges before committing to a full
  backtest.
- You are running factor research where the signal construction is the
  subject of the experiment, not the execution mechanics.

### What It Does Not Do

- No latency model. A signal at bar close assumes an immediate fill.
- No partial fills or rejected orders.
- No intrabar stop triggering.
- No realistic option pricing.
- No portfolio-level risk checks.

Results from Tier 1 are **upper bounds**. A strategy that fails in
Tier 1 will fail in Tier 2. A strategy that succeeds in Tier 1 might
still fail in Tier 2.

### The API

    from honba.research.vectorized import VectorBacktest
    from honba.research.vectorized.signals import sma_crossover

    bt = VectorBacktest(
        universe="nifty50",
        start="2020-01-01",
        end="2024-12-31",
        cost_model="india_equity_delivery",
    )

    result = bt.run(
        signal=sma_crossover(fast=20, slow=50),
        position_sizing="equal_weight",
        rebalance="daily",
    )

    print(result.summary())
    # Vector backtest: sma_crossover(20, 50) on nifty50
    # Period: 2020-01-01 to 2024-12-31 (1237 trading days)
    # CAGR: 16.4%   Sharpe: 1.24   Max DD: -13.2%

### Batching Many Strategies

    from honba.research.vectorized import batch_backtest

    strategies = [
        sma_crossover(10, 30),
        sma_crossover(20, 50),
        sma_crossover(30, 100),
        rsi_oversold(period=14, threshold=30),
        rsi_oversold(period=14, threshold=25),
    ]

    results = batch_backtest(
        strategies=strategies,
        universe="nifty50",
        start="2020-01-01",
        end="2024-12-31",
    )

    results.to_dataframe().sort_values("sharpe", ascending=False)

A batch of 100 strategies on NIFTY 50 over 5 years runs in a few
seconds on a laptop.

## Tier 2 — Event-Driven Simulation

### What It Does

Tier 2 runs the full engine: bar aggregation, event dispatch, risk
checks, order lifecycle, latency simulation, and cost model application.
A strategy running in Tier 2 is **the same code** that will run live.

### When To Use It

- A strategy has survived Tier 1.
- You need a realistic P&L number, not an upper bound.
- You are testing options strategies where expiry mechanics and Greek
  dynamics matter.
- You are validating intraday strategies where latency and fill
  quality matter.

### The API

    import honba

    result = honba.backtest(
        SmaCrossover,
        universe="nifty50",
        start="2020-01-01",
        end="2024-12-31",
        initial_capital=1_000_000,
        cost_model="india_equity_delivery",
        latency_ms=50,
        calendar="NSE",
    )

    print(result.summary())
    result.tearsheet().save("sma_crossover.html")

Tier 2 is the canonical backtest. The number it produces is the number
you should use when comparing strategies to each other or to a
benchmark.

### What It Models

- **Bar aggregation** in the engine, so intrabar events are handled
  correctly for stop triggering.
- **Latency** from signal to order submission, configurable.
- **Cost model** with segment-specific rules (STT, GST, stamp duty,
  exchange fees, brokerage).
- **Order lifecycle** including partial fills, cancellations, and
  rejections.
- **Risk checks** at the engine level, not the strategy level.
- **Portfolio accounting** with mark-to-market P&L, position tracking,
  and margin.

### What It Does Not Model

- Order book depth. Fills assume you get the bar's price for a small
  order. For large orders relative to ADV, Tier 2 is optimistic.
- Market impact. A strategy trading ₹1 crore per order on an illiquid
  stock will move the market; Tier 2 ignores this.
- Exchange outages, broker downtime, or connection issues.

## Tier 3 — Concurrent Async Backtesting

### What It Does

Tier 3 runs thousands of Tier 2 backtests in parallel using Tokio's
async runtime, mirroring Barter-rs's concurrent backtest utilities. It
is used for:

- Parameter sweeps (grid search over a strategy's hyperparameters)
- Monte Carlo simulations (shuffled returns, bootstrapped trades)
- Walk-forward validation (rolling train/test windows)
- Genetic or Bayesian optimization

### The API

    from honba.backtest.runner import ConcurrentRunner

    runner = ConcurrentRunner(max_concurrency=16)

    # Parameter sweep
    results = runner.sweep(
        strategy=SmaCrossover,
        params={
            "fast_period": [5, 10, 15, 20, 30],
            "slow_period": [30, 50, 75, 100, 150],
            "stop_atr_multiple": [1.5, 2.0, 2.5, 3.0],
        },
        config=config,
    )

    # 100 combinations, all run in parallel
    results.heatmap(x="fast_period", y="slow_period", z="sharpe")

### Monte Carlo

    from honba.algo_analytics import monte_carlo

    mc = monte_carlo(
        strategy=SmaCrossover,
        iterations=1000,
        method="bootstrap",     # "bootstrap", "shuffle", "block_bootstrap"
    )

    print(mc.summary())
    # Monte Carlo: 1000 iterations
    # Sharpe: mean=1.18, std=0.24, 5th=0.78, 95th=1.58
    # Max DD: mean=-12.4%, 5th=-21.1%
    # P(Sharpe > 0): 99.8%

    mc.plot_distribution().save("mc.html")

### Walk-Forward

    from honba.algo_analytics import walk_forward

    wf = walk_forward(
        strategy=SmaCrossover,
        train_months=24,
        test_months=6,
        n_folds=6,
    )

    print(wf.summary())
    # Walk-forward: 6 folds
    # Fold 1: train 2020-2021, test 2022-H1, Sharpe 1.34
    # Fold 2: train 2020-2022, test 2022-H2, Sharpe 0.91
    # ...
    # Aggregate: mean Sharpe 1.02, std 0.31

    # Key output: out-of-sample Sharpe distribution
    wf.oos_sharpe_distribution()      # the number that matters

### Distributed Execution

For very large sweeps, `ConcurrentRunner` can distribute across a Ray
cluster:

    runner = ConcurrentRunner(
        backend="ray",
        cluster="ray://head-node:10001",
        max_concurrency=256,
    )

## The Pipeline in Practice

The recommended workflow for a new strategy idea:

1. **Write the strategy in Python** using the `Strategy` base class.
2. **Run Tier 1** with 50+ parameter combinations. Eliminate anything
   with a negative Sharpe or unprofitable after costs.
3. **Run Tier 2** on the survivors. Get realistic P&L numbers.
4. **Run Tier 3** on the top 3–5 by Tier 2 Sharpe:
   - Full parameter sweep to check for parameter cliffs.
   - Monte Carlo to check for overfitting to specific trade
     sequences.
   - Walk-forward to check for out-of-sample degradation.
5. **Reject** any strategy whose:
   - Out-of-sample Sharpe is less than 50% of in-sample Sharpe.
   - Monte Carlo 5th percentile Sharpe is negative.
   - Parameter surface is a needle (very narrow ridge).
   - Results depend on a small number of outlier trades.
6. **Promote** survivors to paper trading.

## Why Three Tiers

The alternative — one engine, one number — hides two failure modes:

- **Overfitting to a specific simulation.** If the backtest simulator
  is buggy or biased, a single result will not reveal it. Three
  independent engines cross-check.
- **Overfitting to a specific data path.** A single backtest produces
  one realisation. Monte Carlo produces a distribution. Only the
  distribution tells you if the result is robust.

The tiers are cheap to run and expensive to skip. A strategy that looks
good in Tier 2 but fails Tier 3 is a strategy that would have lost
money.

## Configuration Files

A complete pipeline can be described in a single TOML:

    # configs/backtest/nifty50_momentum.toml
    [universe]
    name = "nifty50"
    start = "2020-01-01"
    end = "2024-12-31"

    [capital]
    initial = 1_000_000

    [cost]
    model = "india_equity_delivery"
    brokerage = { type = "per_order", value = 20.0 }

    [latency]
    model = "fixed"
    ms = 50

    [tier1]
    enabled = true
    batch_size = 100

    [tier2]
    enabled = true
    save_tearsheet = true

    [tier3]
    enabled = true
    monte_carlo = { iterations = 1000, method = "bootstrap" }
    walk_forward = { train_months = 24, test_months = 6, n_folds = 6 }
    sweep = { params_file = "sweep_params.toml" }

    [risk]
    max_positions = 20
    max_single_stock = 0.10
    max_drawdown = { pct = 0.20, on_breach = "reduce_leverage" }

Then:

    honba backtest --config configs/backtest/nifty50_momentum.toml

The pipeline runs all three tiers and produces a report.
