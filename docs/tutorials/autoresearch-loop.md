# Tutorial: The Autoresearch Loop

Run the LLM-driven research loop on the Alpha 30 universe for one
overnight session and inspect the surviving hypotheses. This tutorial
covers prompt construction, cost budgeting, and how to read the loop's
output.

**Time:** 30 minutes of setup, then overnight unattended.
**Prerequisites:** [Installation](../getting-started/installation.md),
[Research Loop](../ai/research_loop.md).

**Warning:** This tutorial uses the LLM API and incurs costs (typically
$3–10 for 100 iterations with a mid-sized journal). Confirm your API
key and quota before running.

## What You Will Build

A loop that:

1. Generates 100 hypotheses for medium-frequency strategies on the
   Alpha 30 universe.
2. Backtests each through the three-tier pipeline.
3. Validates survivors against the statistical suite.
4. Produces a journal of hypotheses, results, and failure modes.

## Step 1: Configure the LLM Provider

Set the API key in the environment:

    export ANTHROPIC_API_KEY=sk-ant-...
    # or
    export OPENAI_API_KEY=sk-...

Verify the provider works:

    from honba.ai.llm import LLMProvider
    llm = LLMProvider.anthropic(model="claude-sonnet-4-5")
    response = llm.complete("Say OK.")
    print(response)

## Step 2: Initialize the Journal

The journal is the loop's memory. Create an empty one:

    from honba.ai.autoresearch import Journal

    journal = Journal("./data/journals")
    journal.initialize()

    print(journal.stats())
    # Hypotheses: 0
    # Validated:  0
    # Rejected:   0

If you have prior research, import it. Otherwise the loop starts from
nothing.

## Step 3: Configure the Loop

Create `autoresearch.py`:

    from honba.ai.autoresearch import ResearchLoop, LoopConfig
    from honba.ai.llm import LLMProvider
    from honba.algo_analytics import ValidationThresholds

    config = LoopConfig(
        research_question=(
            "Find robust medium-frequency long-only equity strategies "
            "on the NSE Alpha 30 universe that survive realistic Indian "
            "trading costs and out-of-sample validation."
        ),
        universe="nifty200_alpha30",
        start="2018-01-01",
        end="2024-12-31",
        initial_capital=1_000_000,
        constraints=[
            "Long-only",
            "No leverage",
            "Maximum 20 positions",
            "Daily bars only",
            "Delivery costs",
            "Minimum 100 trades over the backtest",
        ],
        num_trials_budget=100,
        validation_thresholds=ValidationThresholds(
            min_oos_is_ratio=0.5,
            min_deflated_sharpe=0.5,
            min_trades=100,
            max_top5_contribution=0.30,
        ),
        journal_path="./data/journals",
        llm=LLMProvider.anthropic(model="claude-sonnet-4-5"),
        retrieval_k=30,        # retrieve 30 similar prior hypotheses
        include_failures=True,
    )

## Step 4: Run It

    from honba.ai.autoresearch import ResearchLoop

    loop = ResearchLoop(config)
    loop.run(iterations=100)

The loop prints progress:

    [001/100] Hypothesis h_2026_09_30_001: momentum with volatility filter
              → Tier 1: Sharpe 1.24, 47 trades → insufficient trades
              → REJECTED: insufficient_trades

    [002/100] Hypothesis h_2026_09_30_002: mean reversion on top-20 by cap
              → Tier 1: Sharpe 0.68
              → Tier 2: Sharpe 0.41, costs 4.8% → marginal
              → Tier 3: MC 5th Sharpe 0.12, WF 3/8 folds → REJECTED: oos_degradation

    [003/100] Hypothesis h_2026_09_30_003: low-vol ranking, quarterly rebalance
              → Tier 1: Sharpe 1.31
              → Tier 2: Sharpe 1.08
              → Tier 3: MC 5th Sharpe 0.74, WF 7/8 folds
              → VALIDATION: PASSED
              → Promoted to paper trading queue

    ...

At the end:

    Loop complete: 100 iterations
    Hypotheses generated:  100
    Tier 1 survivors:      42
    Tier 2 survivors:       8
    Validation passed:      1
    Validation failed:      7
    Promoted:               1

    Journal: ./data/journals
    Report:  ./data/journals/report_2026_09_30.html

## Step 5: Read the Journal

After the loop completes:

    from honba.ai.autoresearch import Journal
    journal = Journal("./data/journals")

    # All validated hypotheses
    for h in journal.validated():
        print(h.id, h.statement, h.result_summary["sharpe"])

    # Failures grouped by mode
    for mode, count in journal.failures_by_mode().items():
        print(f"{mode}: {count}")

    # The full tree of descendants of the surviving hypothesis
    for h in journal.descendants("h_2026_09_30_003"):
        print(h.id, h.status, h.statement)

The `failures_by_mode()` summary is often the most informative output.
If 40 of 100 hypotheses failed with `cost_dominated`, that is a strong
signal that the research question is asking for strategies that trade
too frequently for the cost structure.

## Step 6: Inspect the Survivor

The one hypothesis that passed validation is worth reading carefully:

    survivor = journal.get("h_2026_09_30_003")
    print(survivor.statement)
    print(survivor.rationale)
    print(survivor.failure_modes)
    print(survivor.strategy_code)

Three things to check before trusting the result:

1. **Does the strategy code actually implement the stated hypothesis?**
   The LLM occasionally writes code that drifts from the rationale.
2. **Is there look-ahead bias?** Scan the code for uses of future
   information. The validation suite catches most, but not all.
3. **Is the parameter surface a plateau or a spike?** Run a small
   sweep around the winning parameters:

       from honba.algo_analytics import parameter_surface

       surface = parameter_surface(
           strategy=survivor.strategy_class,
           params={
               "lookback": range(40, 90, 10),
               "top_n": [5, 10, 15, 20],
           },
       )
       surface.heatmap().save("survivor_params.html")

If the winning parameters are a lone peak surrounded by valleys, the
result is likely overfitting to the specific 2018–2024 period.

## Step 7: Cost Awareness

The cost of a 100-iteration run depends on the LLM provider and the
size of the retrieved context. As a rough guide:

    LLM tokens (100 iters × ~10K tokens): ~1M tokens
    Anthropic Claude Sonnet 4.5:           ~$3
    OpenAI GPT-4o:                         ~$5
    Compute (100 × Tier 1 + 10 × Tier 2):  ~30 min CPU

Set a hard budget in the config to prevent runaway costs:

    config = LoopConfig(
        ...,
        max_llm_cost_usd=10.0,
    )

The loop stops when the budget is exhausted, even if iterations remain.

## Step 8: Honest Expectations

One survivor out of 100 is the expected outcome. If the loop produces
10 survivors, something is wrong — either the validation thresholds
are too loose, or the LLM has found a shortcut.

Common causes of inflated survivor rates:

- **Validation thresholds set too low.** Requiring `min_deflated_sharpe=0.5`
  is meaningful; requiring `> 0.0` is not.
- **Overlapping hypotheses.** If the LLM generates variations of the
  same idea, the trials count is inflated and the deflated Sharpe
  correction under-penalizes.
- **A bug in the backtest.** Check that the surviving strategy's code
  actually runs the trades you expect.

Before promoting any survivor to paper trading, have a human read the
strategy code end-to-end. The validation suite catches statistical
problems; it does not catch implementation bugs.

## What to Try Next

1. **Different research question.** The loop's output quality depends
   heavily on the question. "Find strategies that work" produces
   generic results. "Find low-turnover strategies that capture the
   small-cap premium after costs" produces focused ones.
2. **Multi-universe.** Run separate loops on NIFTY 50, Alpha 30, and
   NIFTY MIDCAP 150 and compare the survivor rates.
3. **Iterate on failures.** Feed the failure modes back as constraints
   in the next loop run.

## Related

- [Research Loop](../ai/research_loop.md)
- [Statistical Validation](../backtesting/statistical_validation.md)
- [MCP Gateway](../ai/mcp_gateway.md) — running the loop from an
  external AI agent
