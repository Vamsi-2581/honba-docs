# Examples

The `honba-examples` repository is a numbered learning path: every script is
offline and deterministic by default (the `fake` adapter plus synthetic bars) and
teaches one concept end to end. This page is the index — for each script, the
strategy it runs, the indicators or portfolio parts it builds on, and how to run
it. Each directory also carries a `README.md` with the full set of options.

Run them from a checkout of `honba-examples`:

```bash
pip install -e ../honba/python      # core
pip install -e ".[dev]"             # honba_examples + pytest/ruff
python backtesting/01_first_backtest.py
pytest -q
```

Scripts that need `honba_examples` add the repo root to `sys.path` when the
package is not installed, so they also run from a plain checkout. Scripts that
load catalog strategies look for `honba-strategies` at `--strategies-dir`, then
`$HONBA_STRATEGIES_DIR`, then `../honba-strategies`.

## Shared helpers — `honba_examples/`

The numbered scripts keep only the lesson logic; plumbing lives in the small
`honba_examples` package.

| Module | Role |
|---|---|
| `honba_examples.synthetic` | Deterministic synthetic daily bars (`bar`, `weekdays`, `trending_bars`, `get_bars`). It replaces the old `tests.synthetic` and is shared by the scripts and the unit tests. |
| `honba_examples.cli` | `make_parser`, `add_out_arg`, `emit_json` and `run_example`: the parser, `--out` and JSON-printing boilerplate. |
| `honba_examples.replay_report` | `replay_report(strategy, bars, params)` runs `honba.strategies.testing.replay` and packages the JSON report the strategy examples print. |
| `honba_examples.ewr` | Package (`model`, `sizing`, `schedule`, `simulation`, `reporting`) behind the hand-rolled equal-weight rebalancer. |
| `honba_examples.screening` | Package (`paths`, `momentum`, `value`, `graham`, plus `render`, `cli_args`, `dispatch` and `nl_support` helpers) behind the screeners and the chained screener/SIP script. |
| `honba_examples.pipeline` | Composable screen, filter, size and execute steps, each with a `describe()` manifest, and the TOML chain runner in `chain.py`. See [Step Manifests and Chaining](../strategies/step_manifests_and_chaining.md). |

If you have older notes that import `tests.synthetic`, switch them to
`honba_examples.synthetic`.

Strategies come in two flavours: **inline demo classes** defined in the example
itself, kept small so the lesson is visible in one file, and **catalog
strategies** loaded by registry name from `honba-strategies`, the same code that
ships to users.

## Broker adapters — `basic/`

Adapter mechanics only, no trading strategy.

| # | File | Shows |
|---|---|---|
| 01 | `basic/01_connect_dhan.py` | Connect through the registry; print session and capabilities |
| 02 | `basic/02_fetch_instruments.py` | Pull the instrument master |
| 03 | `basic/03_subscribe_quotes.py` | Stream quotes and compare with snapshots |
| 04 | `basic/04_place_order_paper.py` | Intent → report → fills, a cancel, a reject, the books |
| 05 | `basic/05_check_positions.py` | Positions, funds and trades after a round trip |

```bash
python basic/01_connect_dhan.py
python basic/02_fetch_instruments.py --query RELIANCE
python basic/03_subscribe_quotes.py --advances 5
python basic/04_place_order_paper.py --out orders.json
python basic/05_check_positions.py
```

## Bar construction — `candles/`

No strategy; how bars are fetched and folded.

| # | File | Shows |
|---|---|---|
| 01 | `candles/01_historical_candles.py` | `historical_bars` over a half-open `[start, end)` |
| 02 | `candles/02_realtime_candles.py` | Build OHLC frames from pushed `QuoteTick`s |
| 03 | `candles/03_custom_aggregation.py` | The OHLC fold |
| 04 | `candles/04_multi_timeframe.py` | One fetch, many resolutions |

```bash
python candles/01_historical_candles.py --timeframe 1m --instrument RELIANCE
python candles/02_realtime_candles.py --advances 5
python candles/03_custom_aggregation.py --timeframe 5m --out bars.json
python candles/04_multi_timeframe.py
```

## Persistence — `storage/`

| # | File | Strategy / purpose |
|---|---|---|
| 01 | `storage/01_parquet_catalog.py` | No strategy — `ParquetBarStore` layout |
| 02 | `storage/02_import_nse_bhavcopy.py` | No strategy — EOD CSV → bars |
| 03 | `storage/03_import_amfi_nav.py` | No strategy — AMFI NAV → store |
| 04 | `storage/04_export_tearsheet.py` | Inline `BuyFirstSellLast` — deterministic tearsheet |

```bash
python storage/01_parquet_catalog.py --data-dir /tmp/store
python storage/02_import_nse_bhavcopy.py --data-dir /tmp/bhavcopy
python storage/03_import_amfi_nav.py --data-dir /tmp/nav
python storage/04_export_tearsheet.py --out-dir /tmp/tearsheet
```

## Universes — `universes/`

Alpha 30 is a **universe** (a parameter); the strategy is "equal-weight periodic
rebalance". The catalog entry is `portfolio/rebalancing/equal_weight`.

| # | File | Strategy / purpose |
|---|---|---|
| 01 | `universes/01_nifty50_constituents.py` | No strategy — `resolve_universe("nifty50")` |
| 02 | `universes/02_banknifty_constituents.py` | No strategy — thematic universe |
| 03 | `universes/03_alpha30_constituents.py` | No strategy — `load_alpha30()` seed |
| 04 | `universes/04_alpha30_generic_rebalancer.py` | Inline `UniverseEqualWeightRebalance` |
| 05 | `universes/05_alpha30_custom_universe.py` | Custom universe registration |
| 06 | `universes/06_alpha30_equal_weight_demo.py` | Inline `Alpha30EqualWeightRebalance` |
| 07 | `universes/07_alpha30_backtest.py` | Catalog `equal_weight` on `nifty200_alpha_30`, next-open fills, T+N settlement |
| 08 | `universes/08_alpha30_union_ewr_backtest.py` | Hand-rolled equal-weight rebalancer |

```bash
python universes/01_nifty50_constituents.py
python universes/04_alpha30_generic_rebalancer.py
python universes/06_alpha30_equal_weight_demo.py
python universes/07_alpha30_backtest.py --test-start 2026-06-01 --out-dir /tmp/alpha30
python universes/08_alpha30_union_ewr_backtest.py --start 2026-01-01 --end 2026-09-23
```

## Reference strategies — `strategies/`

Scripts 01 to 07 run on synthetic data via `honba.strategies.testing.replay`
(bars from `honba_examples.synthetic`, report from `replay_report`); 08 to 11
are pipeline recipes.

| # | File | Strategy class | Indicators / parts |
|---|---|---|---|
| 01 | `strategies/01_first_strategy.py` | `SmaCrossover` | SMA |
| 02 | `strategies/02_with_indicators.py` | `CompositeSmaRsi` | SMA + RSI + ATR + trailing stop |
| 03 | `strategies/03_position_sizing.py` | `SmaSizing` | SMA + ATR; fixed / volatility / Kelly |
| 04 | `strategies/04_risk_management.py` | `RiskManagedSma` | SMA + drawdown/loss/position guards |
| 05 | `strategies/05_multi_instrument.py` | `BasketSma` | SMA per symbol + equal-weight rebalance |
| 06 | `strategies/06_equal_weight_rebalance.py` | `PortfolioStrategy` | Universe → Selector → Weighting → Schedule; matches catalog `equal_weight` |
| 07 | `strategies/07_declarative_long_short.py` | `LongShortCrossover` | `DeclarativeStrategy`; long/short SMA + stop/target |
| 08 | `strategies/08_pipeline_intro.py` | — | Minimal `A >> B >> C` composition with the `Pipeline` DSL |
| 09 | `strategies/09_multibagger_momentum.py` | — | Pipeline recipe: multibagger screener, quality filter, fixed SIP, backtest |
| 10 | `strategies/10_dogs_of_nifty.py` | — | Pipeline recipe: high dividend yielders at the 52-week low |
| 11 | `strategies/11_contrarian_value.py` | — | Pipeline recipe: deep value at the 52-week low with P/E and P/B screens |

```bash
python strategies/01_first_strategy.py
python strategies/02_with_indicators.py --rsi-period 14
python strategies/03_position_sizing.py --sizing volatility
python strategies/04_risk_management.py --max-drawdown-pct 0.15
python strategies/05_multi_instrument.py --rebalance-every 15
python strategies/06_equal_weight_rebalance.py --rebalance-days 10
python strategies/07_declarative_long_short.py --stop-pct 0.02
python strategies/08_pipeline_intro.py
python strategies/09_multibagger_momentum.py --asof 2025-03-01 --top 10
```

Scripts 09 to 11 read the Parquet catalog and `fundamentals.duckdb`; see
the Screening section below for the data requirement.

## Backtesting — `backtesting/`

Scripts 01 to 06 use an SMA crossover demo class on the event-driven harness
with the India cost model; 07 to 11 cover the client transports, SIP and
portfolio runs.

| # | File | Strategy class | Adds |
|---|---|---|---|
| 01 | `backtesting/01_first_backtest.py` | `SmaBacktest` | Next-open fills, NSE equity costs |
| 02 | `backtesting/02_cost_modeling.py` | — | Per-leg cost breakdown |
| 03 | `backtesting/03_latency_modeling.py` | `SmaLatency` | Fill models |
| 04 | `backtesting/04_walk_forward.py` | `SmaWalk` | Rolling in/out-of-sample |
| 05 | `backtesting/05_monte_carlo.py` | `SmaMonteCarlo` | Bootstrap equity curves |
| 06 | `backtesting/06_concurrent_backtest.py` | `SmaSweep` | Parameter sweep, Pareto frontier |
| 07 | `backtesting/07_run_via_client.py` | — | Same strategy over in-process and HTTP transports; run id and assumptions |
| 08 | `backtesting/08_sip_buy_and_hold.py` | — | Systematic investment plan (SIP), monthly or weekly buy-and-hold, XIRR, drawdown |
| 09 | `backtesting/09_forward_sip_backtest.py` | — | Forward SIP backtester (`--symbols` or a stdin pipe); alias of 10 |
| 10 | `backtesting/10_chained_screener_and_sip.py` | — | Chained screener (momentum, value, dividend) and forward SIP out-of-sample backtest |
| 11 | `backtesting/11_portfolio_rebalancer.py` | — | Equal-weight periodic rebalance, `--symbols` or stdin aware |

```bash
python backtesting/01_first_backtest.py --out first_backtest.json
python backtesting/02_cost_modeling.py --price 2500 --qty 10
python backtesting/03_latency_modeling.py
python backtesting/04_walk_forward.py --bars 120 --window 40 --step 20
python backtesting/05_monte_carlo.py --paths 1000
python backtesting/06_concurrent_backtest.py --workers 4
python backtesting/08_sip_buy_and_hold.py --symbol ALPHAETF --frequency monthly --sip-amount 5000
python backtesting/10_chained_screener_and_sip.py --strategy multibagger --asof 2023-01-01 --top 5
```

## Screening — `screening/`

Point-in-time screeners that read the 1D Parquet catalog and
`fundamentals.duckdb` as of `--asof` and print a table, JSON, CSV or a symbol
stream that pipes into the forward SIP backtests. Valuation (P/E, P/B, dividend
yield) is derived at the as-of date, never read from a live snapshot. The
bundled point-in-time fundamentals are annual vintages from 2023-03-31, so
earlier `--asof` dates return few or no fundamental candidates.

| # | File | Adds |
|---|---|---|
| 01 | `screening/01_multibagger_screener.py` | Stage 2 momentum breakout candidates |
| 02 | `screening/02_value_52w_low_screener.py` | Deep value near the 52-week low |
| 03 | `screening/03_dividend_yield_screener.py` | High dividend yielders near the 52-week low |
| 04 | `screening/04_intelligent_investor_screener.py` | Graham value and margin of safety |
| 05 | `screening/05_net_net_screener.py` | Graham net-net (NCAV) deep asset value |
| 06 | `screening/06_run_chain.py` | Run a screen, filter, size, execute chain from a TOML file |

```bash
python screening/03_dividend_yield_screener.py --asof 2025-03-01 --top 5 --format symbols
python screening/03_dividend_yield_screener.py --asof 2025-03-01 --format symbols \
  | python backtesting/10_chained_screener_and_sip.py --asof 2025-03-01 --sip-amount 10000
python screening/06_run_chain.py --chain screening/chains/quality_value.toml --validate-only
python screening/06_run_chain.py --chain screening/chains/quality_value.toml --format csv
python screening/06_run_chain.py --describe
```

`06_run_chain.py` runs steps from `honba_examples.pipeline`. Each `[[steps]]`
entry names a step (`screen.value`, `filter.fundamental_quality`,
`size.equal_weight`, `execute.sip_backtest`, and others), may override params
with `with`, and may add display-only columns with `report`:

```toml
[[steps]]
uses = "filter.fundamental_quality"
with = { min_roce = 12.0, max_debt_to_equity = 1.0 }
report = ["market_cap", "dividend_yield_pct", "roe_pct"]   # shown, not tested
```

The Graham and net-net screeners (04, 05) are not pipeline steps yet. Chains
that name `honba-strategies` catalog ids validate but are not runnable here. See
[Step Manifests and Chaining](../strategies/step_manifests_and_chaining.md).

## Mutual funds — `mutual_funds/`

No `Strategy` subclasses; direct NAV/SIP math over `honba_examples.amfi`.

| # | File | Purpose |
|---|---|---|
| 01 | `mutual_funds/01_fetch_nav.py` | AMFI NAV history |
| 02 | `mutual_funds/02_sip_backtest.py` | Monthly SIP, units at NAV, XIRR |
| 03 | `mutual_funds/03_category_analysis.py` | Category stats and ranking |
| 04 | `mutual_funds/04_portfolio_optimization.py` | Mean-variance allocation |

```bash
python mutual_funds/01_fetch_nav.py --scheme 120503 --start 2023-01-01 --end 2024-12-31
python mutual_funds/02_sip_backtest.py --scheme 120503 --monthly 10000 --start 2023-01-01
python mutual_funds/03_category_analysis.py --category "Equity:Large Cap" --start 2020-01-01
python mutual_funds/04_portfolio_optimization.py --schemes 120503,120504,120505 --start 2022-01-01
```

## Options — `options/`

Hand-rolled legs; the packaged versions live in the catalog
(`options/nifty_short_straddle`, `options/banknifty_iron_condor`,
`options/expiry_day_scalp`).

| # | File | Strategy |
|---|---|---|
| 01 | `options/01_option_chain.py` | No strategy — chain snapshot |
| 02 | `options/02_greeks_calculation.py` | No strategy — Black-Scholes Greeks |
| 03 | `options/03_backtest_straddle.py` | Long ATM straddle to expiry |
| 04 | `options/04_backtest_iron_condor.py` | Iron condor (4-leg credit spread) |
| 05 | `options/05_expiry_day_strategy.py` | Expiry-day gamma scalping / theta capture |

```bash
python options/01_option_chain.py --underlying NIFTY --expiry 2024-12-26
python options/02_greeks_calculation.py --spot 24000 --strike 24000 --days-to-expiry 7
python options/03_backtest_straddle.py --underlying NIFTY --expiry 2024-12-26
python options/04_backtest_iron_condor.py --underlying NIFTY --expiry 2024-12-26
python options/05_expiry_day_strategy.py --underlying BANKNIFTY --expiry 2024-12-25
```

## AI research — `ai_research/`

No trading strategy. 01, 03, 06 and 09 are runnable offline (a scripted fake
LLM, no network, no API keys); the rest are design placeholders whose `honba.ai`
modules are still empty stubs. Slot 08 is intentionally unused; numbers are not
reused or shifted.

| # | File | Status | Purpose |
|---|---|---|---|
| 01 | `ai_research/01_llm_research_analyst.py` | Runnable | NL question → validated screener filter |
| 02 | `ai_research/02_autoresearch_loop.py` | Design only | Hypothesis → backtest → journal |
| 03 | `ai_research/03_nl_screener.py` | Runnable | NL query → screener filter → historical backtest |
| 04 | `ai_research/04_nl_to_strategy.py` | Design only | NL → strategy manifest → `verify_manifest` |
| 05 | `ai_research/05_rl_training.py` | Design only | Gym environment + RL policy |
| 06 | `ai_research/06_mcp_integration.py` | Runnable | Enumerate the static MCP tool schema |
| 07 | `ai_research/07_natural_language_critique.py` | Design only | Backtest result → LLM critique |
| 09 | `ai_research/09_typed_llm_decision.py` | Runnable | Typed `LlmDecision` (veto or size down) applied to an order intent; size increases are rejected; Journal v1 audit records |

```bash
python ai_research/01_llm_research_analyst.py --query "mid cap with rsi below 30"
python ai_research/06_mcp_integration.py
```

## Production — `production/`

All run the `sma_crossover` strategy; swap `--adapter dhan` for real
connectivity.

| # | File | Strategy |
|---|---|---|
| 01 | `production/01_paper_trading.py` | Inline `SmaPaper` — simulated fills, P&L, risk guards |
| 02 | `production/02_live_trading_dhan.py` | Inline `SmaLive` — Dhan connect, orders, position sync |
| 03 | `production/03_multi_account.py` | Inline `SmaMulti` — isolated capital, aggregated P&L |
| 04 | `production/04_monitoring.py` | No strategy — health checks |
| 05 | `production/05_deployment.py` | No strategy — deployment config |

```bash
python production/01_paper_trading.py --strategy sma_crossover --capital 100000
python production/02_live_trading_dhan.py --client-id XXX --access-token YYY  # needs Dhan creds
python production/03_multi_account.py --accounts config/accounts.yaml
python production/04_monitoring.py --port 9090
python production/05_deployment.py --output /etc/honba/
```

For the strategy catalog itself, see the `honba-strategies` repository; the
[BANKNIFTY Options Backtest](../tutorials/banknifty-options-backtest.md),
[Alpha30 Factor Model](../tutorials/alpha30-factor-model.md) and
[SIP Backtest](../tutorials/sip-backtest.md) tutorials walk through end-to-end
variants.
