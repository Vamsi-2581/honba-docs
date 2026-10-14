# Known Issues in the Examples

This page lists problems in `honba-examples`, and in the core and data code the
examples rely on, that are **open**: they are documented in the scripts'
`Pitfalls::` sections or were found while checking them, and they have not been
fixed. It is not a changelog. Fixed behaviour is described in
[Examples](index.md).

Each item gives the owner (the repository where a fix belongs) and a status:

- **Open** means the behaviour was confirmed in the code or the script's own
  docstring.
- **Unverified** means it has been reported or suspected and has not been
  confirmed here. Do not treat an unverified item as a fact in either direction.

## Examples (`honba-examples`)

| Item | Status | What happens | Workaround |
|---|---|---|---|
| `production/05_deployment.py` runs a module that does not exist | Open | The generated systemd unit and Dockerfile run `python -m honba.session.run --config ...`. `honba.session` is a single module and has no `run`. With `Restart=on-failure` the unit would restart every 10 seconds without ever trading. | Treat the generated files as a template and set a real entry point. |
| Bhavcopy loader keeps one row per symbol | Open | `storage/02_import_nse_bhavcopy.py` parses into a dict keyed by symbol, so a symbol present in two series in the same file (EQ and BE) keeps only the last row. Series other than EQ, BE and BZ are dropped and reported in `dropped_series`. | Check `dropped_series` and compare counts per series before trusting a symbol's bar. |
| Coverage ledger `row_count` inflates on re-append | Open | `storage/01_parquet_catalog.py`: the ledger counts rows appended, not distinct bars, so appending the same data twice doubles `row_count`. The bars themselves are idempotent (last write wins by `ts`). | Do not read `row_count` as a bar count. Keep one importer per store. |
| Hourly (and 30m) bars are bucketed in UTC, not IST | Open | `honba_examples.bars.aggregate` buckets by UTC. 5m and 15m line up with IST, but 30m and 1h do not (IST is UTC+5:30): a 09:15 IST open falls in the 08:30 to 09:30 IST bucket, so the first 1h bar holds only 15 minutes. A daily bucket is a UTC day. | Resample with an IST-aware rule before using 30m or 1h bars, and check constituent counts per bar. |
| Mutual-fund category loading is a stub | Open | `AmfiNavLoader.load_category` returns `{}` today, so `mutual_funds/03_category_analysis.py` has no schemes to rank from the loader alone. | Load NAVs by scheme code with `AmfiNavLoader.load` instead. |
| Production risk-limit flags are not enforced | Open | `production/01_paper_trading.py` echoes the risk-limit arguments into its config but does not apply them. `production/02_live_trading_dhan.py` is a template whose adapter, session and risk-limit code is commented out. | Do not read a paper run as a rehearsal of live risk control. Enforce limits in the strategy or the engine. |
| Metrics server binds `0.0.0.0` with no authentication | Open | `production/04_monitoring.py` exposes P&L over HTTP on all interfaces. It also stops after `--duration` seconds. | Bind a loopback or internal address, or firewall the port. |
| Unpriced symbols are held flat as idle cash | Open | In the forward SIP (`honba_examples.sip`, used by `backtesting/10_chained_screener_and_sip.py`), a symbol with no bar after `--asof` is now reported (`unpriced_symbols`, `delisted_symbols`) but its capital sits as idle cash. A collapse or delisting therefore shows 0 percent, not a loss, which understates delisting losses. | Read `unpriced_symbols` and `delisted_symbols` in the summary and treat the return as an upper bound. |
| Alpha 30 shipped history is current names from 2005 | Open | The history that backs `--alpha30-list engine` (and the seed list) is today's members. Neither list is point-in-time, so any past-dated run has survivorship and selection bias. | See the next section. |

## Core (`honba`) and catalog (`honba-strategies`)

| Item | Owner | Status | What happens |
|---|---|---|---|
| Alpha 30 history stamps today's names from 2005-04-01 | `honba` (`default_alpha30_history` in `honba.markets.india.universes`) | Open | Every one of the 30 names has an inclusion date of 2005-04-01 and no exit. `resolve_universe("nifty200_alpha30", as_of=2010-01-04)` returns all 30, including names listed years later. `point_in_time=True` only stops the engine raising; it does not remove the bias. Register a real history with `register_universe_history` or provide a `constituents/` catalog file. |
| Delivery STT charged on the sell leg only | `honba` (`honba/markets/india/costs.py`) | **Unverified** | The delivery cost function sets STT to 0.0 on a buy and 0.10 percent of notional on a sell (confirmed in the code). Whether that matches the statutory schedule has not been checked against the current NSE/SEBI circular. If both legs carry STT, every round trip is under-costed. `backtesting/02_cost_modeling.py` flags this itself. |
| Lakehouse split and bonus adjustment | `honba-data` / lakehouse (owner not confirmed) | **Unverified** | It is not known whether the stored daily bars are corporate-action adjusted. A RELIANCE drop of about 48 percent over 2024-01 to 2025-03 was observed and may be a bonus-issue artefact rather than a real move. This was not confirmed in this review and the figure comes from that observation, not from a reproducible check here. Treat any return across a split or bonus as suspect until adjustment is checked. |
| `sma_crossover` code defaults differ from config | `honba-strategies` | Open | `Parameter()` defaults are fast 10 and slow 30; `config.toml` and `step.toml` use 20 and 50, and the constructor reads `config.toml`. `test_step_manifests.py` records the mismatch. |
| `alpha30_factor` and `alpha30_quality` do not rank | `honba-strategies` | Open | Both hold every member at equal weight. The registry description of `alpha30_quality` no longer claims quality screening, but the names remain misleading. |
| Intraday and stop caveats | `honba-strategies` | Open | `vwap_reversion` can trade the first bar of a session and has no stop; `turtle_20` and `kdj_cross` check the ATR stop after the bar; `rsi2_connors` and `ifr2` have no stop. See [Strategies](../strategies/index.md#honest-caveats-in-the-catalog). |

## Not defects, but easy to trip over

- The Graham and net-net screeners (`screening/04`, `05`) are not pipeline steps
  yet, and chains that name catalog ids validate but do not run
  ([Step Manifests and Chaining](../strategies/step_manifests_and_chaining.md)).
- Fundamentals carry period-end dates, not filing dates. `--fiscal-lag-days`
  (default 90) is a proxy, so a late filer or restatement can still leak
  information into a screen.
- Several scripts run on synthetic bars. Their output demonstrates plumbing, not
  performance. See [Pitfalls in the Examples and the Catalog](../backtesting/pitfalls_in_examples.md).
