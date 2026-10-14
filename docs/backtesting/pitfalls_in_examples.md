# Pitfalls in the Examples and the Catalog

Every example script in `honba-examples` and every registered strategy in
`honba-strategies` is written to be read together with its pitfalls. The
convention is deliberate: a backtest number that comes without its failure modes
is easy to over-trust, and the code that produced it is the best place to say
what it does not prove.

This page explains the convention and indexes the main classes of trap against a
concrete script or strategy README that demonstrates it or defends against it.
It does not repeat the general treatment; for that, read
[Avoiding Backtest Biases](https://honba-labs.github.io/honba/book/backtesting/avoiding_backtest_biases.html)
in the mdBook (source: `book/src/backtesting/avoiding_backtest_biases.md`).

## The convention

**Example scripts (`honba-examples`).** Every numbered script
(`<dir>/NN_*.py`) has a `Pitfalls::` section in its module docstring with 3 to 6
bullets that are specific to what that script does. A unit test,
`tests/unit/test_pitfalls_sections.py`, discovers the scripts and fails when one
has no section or a bullet count outside that range. A `PENDING_PITFALLS`
allowlist exists to stage a new script; it is empty today. Each directory's
`README.md` carries a short "Pitfalls checklist" pointing at the same idea.

**Catalog strategies (`honba-strategies`).** Every strategy in `registry.json`
has a `README.md` with three sections, enforced by `test_readme_sections.py`:

| Section | Holds |
|---|---|
| `## Usage` | A snippet that runs from the repo root: load by registry name, run through `honba.strategies.testing.replay`. |
| `## Possible output` | The real output of that snippet. `backtest_result.json` metrics are quoted only when its `status` is `completed`, and the README says where they came from. |
| `## Pitfalls` | At least 3 bullets specific to that strategy's code and config, plus a link to the generic bias page. |

Each registered `step.toml` also carries a short `pitfalls` array (2 to 4
one-line strings) that a GUI can show when the step is added to a chain; see
[Step Manifests and Chaining](../strategies/step_manifests_and_chaining.md).
Unregistered scaffolds keep the same three headings with an honest "Status:
scaffold" stub.

!!! note "What the tests check, and what they do not"
    The tests check that the sections exist and have a minimum shape. They do
    not check that a bullet is correct. Treat a pitfall as a claim made by the
    author of that script, and verify it against the code when it matters.

## Trap classes and where to see them

| Trap | What goes wrong | See |
|---|---|---|
| Lookahead and fill timing | A decision made on a bar's close is filled at that same close, a price you only know after the session ends. | `backtesting/03_latency_modeling.py` compares `bar_close`, `next_open` and a two-bar delay on fill prices. `backtesting/01_first_backtest.py` fills at the next open. The `momentum/trend/sma_crossover` README shows how `fill_delay=1` moves the first fills. |
| As-of dates and fundamentals dating | A screen at date D uses data that was not yet public, or trades D's close. | `screening/02_value_52w_low_screener.py` (fiscal period end plus `--fiscal-lag-days`, no filing dates, buy at the next open). `backtesting/10_chained_screener_and_sip.py` repeats it for the chained run. |
| Survivorship | Today's members, or only stocks that still exist, are tested over the past. | `universes/03_alpha30_constituents.py` and `universes/08_alpha30_union_ewr_backtest.py` (today's names with a 2005-04-01 stamp). `backtesting/10_chained_screener_and_sip.py` (unpriced and delisted names held flat). The `universe/alpha/factor` README (`point_in_time = true` does not remove the bias). |
| Overfitting and multiple testing | The best of many tried parameter sets is reported as if it were the only one tried. | `backtesting/06_concurrent_backtest.py` prints the trial count in a `multiple_testing` block and does not rank pairs with too few trades. `backtesting/04_walk_forward.py` is the out-of-sample step. `ai_research/02_autoresearch_loop.py` (every hypothesis is a trial; keep a final holdout). The `sma_crossover` README counts 8,523 valid `fast < slow` pairs. |
| Costs and slippage | Gross results shown as net; one flat fee instead of the statutory legs; no spread or impact. | `backtesting/02_cost_modeling.py` (per-leg breakdown; its STT leg needs checking, see [Known Issues](../examples/known_issues.md)). `backtesting/11_portfolio_rebalancer.py` (a flat `--fee` only). The `momentum/oscillator/kdj_cross` README (138 fills in 600 bars on the synthetic run). |
| Data adjustment | Splits, bonuses and demergers appear as false gaps and false returns. | `storage/02_import_nse_bhavcopy.py` (bhavcopy is the raw traded price). `candles/04_multi_timeframe.py` (resampling invariants). The `momentum/trend/turtle_20` README (an unadjusted split can fire a false signal). |
| LLM leakage and trust | A model that has seen the backtest window "predicts" it; model text is trusted or executed. | `ai_research/01_llm_research_analyst.py` (translation only, no as-of date). `ai_research/03_nl_screener.py` (strict predicate, whitelist `safe_eval_predicate`). `ai_research/09_typed_llm_decision.py` (fails closed, reduce-only, test vetoes after the model's cutoff). |
| Paper versus live | A paper run is read as a rehearsal of live risk control and order handling. | `production/01_paper_trading.py` (risk-limit arguments echoed, not enforced). `basic/04_place_order_paper.py` (only the `fake` adapter is paper). `production/02_live_trading_dhan.py` (a template: its risk-limit code is commented out). `basic/01_connect_dhan.py` (check `session.mode` before sending any order). |

Several of these traps have known open items in the example or catalog code
itself. They are collected, marked open, on
[Known Issues](../examples/known_issues.md).

## Reading a pitfalls section

- **Read it before the number.** The bullets say which direction the bias runs,
  so a result that looks good can be discounted correctly.
- **A defence is not a guarantee.** Some bullets are labelled "Defence shown":
  they describe what the script does right, for example collecting sweep results
  in grid order or reading only data up to `--asof`. They do not make the
  surrounding setup safe.
- **Synthetic data proves plumbing, not performance.** Most numbered scripts run
  on deterministic synthetic bars so they are reproducible. The pitfalls say so
  wherever it matters.

## See also

- [Statistical Validation](statistical_validation.md) for the gates that count
  trials and deflate Sharpe.
- [Cost Modeling](cost_modeling.md) and [Latency Modeling](latency_modeling.md)
  for the engine-side models these examples approximate.
- [Examples](../examples/index.md) for the script index.
- [Strategies](../strategies/index.md) for the catalog and its caveats.
