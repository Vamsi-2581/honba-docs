# Honba-kao Simulator — Port Design Document

**Document ID:** HBD-2026-DES-005
**Status:** Draft for Implementation
**Version:** 1.0
**Last updated:** 2026-10-20
**Target Systems:** `honba-kao` (Leptos CSR frontend), `honba-kao-client`, `honba-kao-api-wasm`, `honba-ya` (backend, reference only — no backend work in scope)
**Source of Record:** the audited React implementation in `honba-fe/apps/web/src` (`core/simulator/`, `features/simulator/`)
**Governing decisions:** supersedes nothing; sibling to HBD-2026-DES-004 (the React simulator plan, P0–P9 complete). Honors ADR 0017 (backtest run lifecycle), ADR 0021 (run events SSE), ADR 0011/0032 (Decimal/minor-unit money), ADR 0029 (strategy graph).

**Companion documents:** `algo_designer_and_simulator.md` (HBD-2026-DES-004), `honba-kao/plan.md` (§ Phase 4.3 “Simulator”), `honba-kao/docs/widget-integration.md` (openalgo-charts bridge), `honba-workspace/top-level-design.md`.

---

## 1. Purpose

Port the **Simulator** page/feature of `honba-fe` (React + zustand + `openalgo-charts`) into `honba-kao` (Leptos 0.7 CSR), reusing honba-kao's own infrastructure instead of re-implementing what already exists there. The result is a `/simulator` page that can:

1. Select a strategy (query param → localStorage handoff → server catalog → built-in template).
2. Configure control parameters (symbol, dates, timeframe, capital, commission, slippage, fill model), validated against the same Rust-side rules as `BacktestRequest::resolve`.
3. Submit `POST /backtests`, poll/fold the run lifecycle, and (feature-detected) tail an SSE `GET /backtests/{id}/events` log.
4. Fetch `GET /bars/{id}` for the run window and render the candle/volume chart with entry/exit markers and the strategy's own indicator series.
5. Fetch `GET /backtests/{id}/journal` (enveloped `TradesResponse`), pair fills into round trips client-side, and present: metrics ribbon with provenance, equity + drawdown curve, monthly P&L heatmap, trade journal table with CSV export, validation badges, run log.
6. Optional later phase: a client-side animated replay engine (bar-by-bar tick simulation) as in React Track B.

## 2. Audit of the source (honba-fe) — what is being ported

The React feature has **two tracks layered on top of each other**; the port treats them differently:

### 2.1 Track A — server-driven backtest (the spine; port fully)

`features/simulator/store/simulator-store.ts` binds everything: controls → `plan()` →
`POST /backtests` (via `run.ts` polling loop, 1 s interval, 120-poll cap **folding to `failed`, never hanging**) → `GET /backtests/{id}/journal` → `trade-pairing.ts` → `analytics.ts` → `ResultsPanel`.

Key contract facts verified against `honba-ya/crates/honba-ya-api` (source of truth for the port):

- `BacktestRequest` — `deny_unknown_fields`; every field `Option`; `resolve` requires `strategy`, `universe`, `start`, `end`, and non-zero `seed`. Dates `YYYY-MM-DD`. `bar_spec` matches `^(\d+)(s|m|h|d|w|mo)$` (crate exports `FILL_MODELS`, `MAX_COMMISSION_PER_FILL = 1e9`, `MAX_SLIPPAGE_BPS = 10_000`).
- `BacktestResponse { run_id, status: RunStatus, metrics?: BacktestMetrics, assumptions?, validation?: ValidationBlock }`; `RunStatus` is `#[non_exhaustive]` `\`pending|running|completed|failed|cancelled\``.
- `BacktestMetrics` is exactly five fields: `trades, net_pnl, sharpe, max_drawdown, total_return`. Everything else the UI shows is **derived** from these + round trips, or honestly `n/a`.
- `GET /bars/:id?tf=&from=&to=` → enveloped `BarsResponse { bars: Vec<Bar> }`, ascending `ts_event`. Wire `Bar` carries `UnixNanos` timestamps (nanos as **string** to dodge float overflow).
- `GET /backtests/:id/journal` → enveloped `TradesResponse` (**plain JSON, not NDJSON** — `runs.rs` decodes the journal and maps `trades_from_journal`). `Trade` side: `ts_event/ts_init: UnixNanos`, `costs: Money` (integer minor units).
- `GET /backtests/:id/events?resume_from_seq=` → SSE `StreamFrame::{Status, Record}`; implemented in `streams.rs` with keep-alives.
- All responses (including errors) use `ResponseEnvelope` with API/schema version pinning — which is what `honba-kao-client` already assumes.

### 2.2 Track B — client replay engine (`core/simulator/engine/`) (port later, leaner)

`ReplayController` (framework-free class) steps each bar through 4 tick phases (`open → intra1 → intra2 → close`), evaluates the strategy graph only at close, synthesizes a forming candle per tick, executes trades client-side (`bar_fill` / `bar_fill_through` / `l2` / `none`), and drives a confluence snapshot panel. Speeds `{1,5,10,25,50}`, scrubber `jumpToBar`, `instantRunToEnd`.

### 2.3 Warts found in the audit — decisions for the port

| # | Wart in honba-fe | Decision |
|---|---|---|
| W1 | Hard-coded Sharpe fallback `\|\| '2.14'` in the overview KPI card | **Do not port.** Violates the feature's own R5 honesty rule. Show `n/a` when the wire is silent. |
| W2 | `strategyToRun = compiled.id === 'sma_cross' ? 'sma_crossover' : compiled.id` spelling hack | Do not port. Use the id from `POST /strategies` verbatim; if such an alias turns out to exist, fix it in honba-ya, not the client. |
| W3 | `DEFAULT_TIMEFRAME ('1d')` vs `DEFAULT_CONTROLS.timeframe ('5m')` inconsistency | Single `DEFAULTS` module, one default (`5m`), documented. |
| W4 | Dead/duplicated `workbench-tabs.tsx` + `replay-toolbar.tsx` (457+280 lines) duplicating `SimulatorView`'s inline dock | Do not port; the port builds `SimulatorView` modularly from the first phase (the monolith is 1495 lines and is the biggest maintenance wart). |
| W5 | Hand-rolled equity SVG + monthly heatmap + CSV export in `SimulatorView` **duplicated** the Phase-8 analytics implementations (and got them subtly wrong) | Port only the `analytics.ts` implementations; the view consumes them once. |
| W6 | Replay drawings drawn in pixel space (don't track pan/zoom), not persisted | Deferred phase (P8); when ported, must draw in **chart space** via the widget's time/price coordinates, or not at all. |
| W7 | Sandbox portfolio adapter hard-codes ₹10 fill costs and mutates a separate portfolio store | Do not port as-is; replay TF execution lives entirely inside the simulator's own state (no cross-store mutation). |
| W8 | Benchmark bars refetch effect depends on whole `equity`/`input` objects (churn) | Port with primitive dependencies (run-id + window + bar_spec). |
| W9 | `useSimulationEngine` takes 10 positional args | Port as a config struct. |

### 2.4 Honesty invariants to preserve verbatim (non-negotiable)

1. **Never invent a metric.** Every displayed metric carries provenance: `reported` (wire), `derived` (computed from wire + journal), or `n/a` (shown, never fabricated). Sortino is `n/a` by design in honba-fe because the payload carries no periodic return series — same in the port.
2. **Never synthesize a candle.** Resampling folds bars that exist into buckets; an empty bucket is not emitted.
3. **Markers land on bar timestamps.** Snap markers to bar times within `markerTolerance(barSpec)`.
4. **Slippage column stays blank.** The journal is fills with no signal price; it is not guessed.
5. **Preview bars are labelled.** The deterministic offline preview is always surfaced as `source: 'preview'`, never passed off as served history.

## 3. Target architecture (honba-kao)

### 3.1 Where code lives

```
honba-kao/crates/honba-kao-client/src/lib.rs      P0: + bars/backtest/journal/events methods
honba-kao/crates/honba-kao-app/src/
  sim/                                            P1+ core logic (pure, unit-testable)
    mod.rs           pub use
    defaults.rs      DEFAULTS, bar-spec units, fill-model descriptors (P1)
    controls.rs      SimulatorControls, resolve_range, ControlPlan, plan_request (P1)
    selection.rs     StrategySelection, resolve_selection (P1, P8 wires UI)
    persistence.rs   localStorage honba.simulator.selection (P2)
    chart_data.rs    UnixNanos→UTC s, to_chart_bars, resample, snap_to_bars, preview (P2)
    pairing.rs       PairingResult / RoundTrip / markers (P3)
    analytics.rs     RoundTripStats, equity_series, benchmark_equity, monthly_heatmap,
                     RibbonMetric, journal_rows, to_csv (P3)
    log.rs           SSE fold: RunLogState, frame_to_line (P4)
  stores/
    simulator.rs     SimStore struct + RwSignals, provided at the page (P4)
  components/simulator/
    results_panel.rs, metrics_ribbon.rs, journal_table.rs, monthly_heatmap.rs,
    validation_badges.rs, run_log.rs, simulator_chart.rs, simulation_controls.rs,
    replay_controls.rs (P5–P7)
  pages/simulator_page.rs                         existing placeholder, replaced
  types/chart.rs                                  existing Bar/SeriesKind (reused as-is)
```

The `sim/` modules are **pure** (no gloo, no DOM, no leptos): they are direct ports of the React `core/simulator` pure modules and keep their tests as Rust `#[cfg(test)]` golden tests, mirroring the vitest suites (`controls.test.ts`, `trade-pairing.test.ts`, `analytics.test.ts`, `chart-data.test.ts`, `bars.test.ts`).

### 3.2 State translation table (React → Leptos)

| React concept | honba-fe implementation | honba-kao implementation |
|---|---|---|
| zustand store singleton | `createSimulatorStore()` module singleton | `SimulatorStore` struct of `RwSignal`/`Memo` fields, `provide_context` on the page, `use_simulator()` accessor (idiom: `stores/trading_mode.rs`) |
| store actions | `state.setControls{...}` methods | store impl methods taking `&self` and touching the inner `RwSignal`s |
| derived values | selector functions | `Memo` (`Signal::derive`) |
| run polling loop | injected-sleep async in `run.ts` | `leptos::task::spawn_local_scoped` + `gloo_timers::future::TimeoutFuture` (the `use_interval.rs` idiom), `on_cleanup` abort |
| localStorage hand-rolled persistence | `persistence.ts` | `web_sys::window().local_storage()` best-effort, same key `honba.simulator.selection` |
| feature-detected client seams (`hasJournalClient`, `hasBarsClient`, `hasEventStream`) | duck-typing around codegen client | type system: methods on `ApiClient`; SSE feeature-detect not needed (client is always present) — instead a `bool` flag/mode until P0 confirms the SSE transport |
| `useSimulationEngine` hook + callbacks |_class + effect re-binding | `ReplayController` struct owned in a `RwSignal<Option<…>>`, `Effect::new` re-init keyed by the composite init key (same field list as fe, minus the wart W9 => one struct arg) |
| CSS | `SimulatorView.css` (948 lines) + ui-core tokens | new `style/simulator.css` linked in `index.html`, using **only** `--surface/--hair/--ink/--pnl-gain/--pnl-loss/--shade-*` semantic tokens (never literal hexes; heatmap tints use `color-mix(in srgb, var(--pnl-gain) α%, transparent)` like `screener/heatmap.rs`) |
| `openalgo-charts` direct `createChart` tier | low-level API w/ markers + addIndicator | honba-kao **must extend the bridge**: `vendor/openalgo-charts/honba-bridge.js` currently exposes widget-tier only (mount/addIndicator/snapshot). Add (P6): `setMarkers(series, markers)`, multi-pane volumes with the `createChart` tier or widget indicator panes. See §3.4. |

### 3.3 Types to port (exact shapes)

Port into `sim/` as `serde`-free plain Rust; wire types come from crates, not mirrors:

- Wire (re-export via `honba_kao_client::wire::` additions, P0): `honba_api::{BacktestRequest, BacktestResponse, BacktestMetrics, BarsResponse, BarsQuery, CompiledStrategy, StrategiesResponse, TradesResponse, RunStatus, FILL_MODELS, MAX_SLIPPAGE_BPS, MAX_COMMISSION_PER_FILL}`, `honba_analytics::ValidationBlock`, `honba_entities::Trade` (journal fill), `honba_kokoro_entities`/`honba_kokoro_types` for `InstrumentId`/`UnixNanos`/`Money` as needed.
- Domain: `ControlPlan { request: BacktestRequest, range: (Date, Date), issues: Vec<String>, not_modelled: Vec<String> }`, `SimRunState { status, run_id, error, logs: Vec<String> }`, `JournalState { status, run_id, fills: Vec<Trade>, error }`, `PairingResult { fills, trips, open_positions, dropped }`, `RoundTrip …` (same 16 fields as fe), `RibbonMetric { key, label, value, kind: pct|dd|ratio|count|ccy, provenance, note }`, `HeatmapCell/MonthlyHeatmap`, `RunLogState { lines: Vec<String>, last_seq, terminal }`, `ChartBar { time: i64, open..close: f64, volume }` (reuse `types/chart.rs::Bar`; add a domain alias if desired).

`RunStatus` is `#[non_exhaustive]` — inside this workspace we consume the crate enum directly (no JS re-encoding issues), so exhaustive matching is safe; still write `match` arms without `_` so a new server state is a compile error here.

### 3.4 Chart stack decision

honba-kao already has:
- `components/charts/openalgo_chart.rs` — Rust⇄JS bridge to the **vendored widget** (`vendor/openalgo-charts/honba-bridge.js`; topbar railed off, drawing rail on, theme from `--chart-*` tokens).
- `equity_chart.rs` — demo-data area series (to be rewired to real `equity_series`).
- `honba-kao-api-wasm::indicators` — real indicator computation from `honba-me-indicators` (sma/ema/rsi/macd/bollinger/… exist there).

Decision: the port uses the **existing bridge** for candles/volume/area and the existing wasm indicators for indicator series, mapped through a ported `chart_series.rs` (`registry_id_for(kind)`, `settings_for`, `describe_kind`, param aliases). The React app's separate low-level `createChart` marker machinery maps to two bridge extensions (both small, both in P6):
1. `honba-bridge.js`: a `setMarkers(instanceId, markers)` that calls the widget's marker/series API (whatever the bundle exposes; the React app gestures at `createMarkers`/`setMarkers` layers of the same bundle).
2. A pane for volume (widget already supports indicator panes; a histogram series may be acceptable as a synthetic "volume indicator" if the widget API does not expose raw histogram series — probe in P6, choose accordingly, document in `widget-integration.md`).

If the widget truly cannot host markers, fall back to drawing markers as an SVG overlay in **chart space** (converting time→x via `chart.timeScale()`, exposed through the bridge snapshot/convert API), never pixel space (W6).

## 4. Phases

Each phase is individually shippable and ends with `cargo check -p honba-kao-app` (or the named verification) green. Phases have explicit **Agent hints** — the accumulated gotchas for unattended implementation.

---

### Phase 0 — Wire contract verification + client extension

**Goal:** honba-kao-client can talk to every simulator endpoint with zero shape guesswork.

**Work items**

1. Extend `crates/honba-kao-client/src/lib.rs` with methods on `ApiClient` mirroring existing style:
   - `get_bars(id, tf, from, to) -> Result<BarsResponse, ClientError>`
   - `get_strategies() -> Result<StrategiesResponse, ClientError>`
   - `post_strategies(manifest: &serde_json::Value) -> Result<CompiledStrategy, ClientError>` (and `post_strategies_verify` only if the studio needs it)
   - `post_backtests(req: &BacktestRequest) -> Result<BacktestResponse, ClientError>`
   - `get_backtest(id) -> Result<BacktestResponse, ClientError>`
   - `get_backtest_journal(id) -> Result<TradesResponse, ClientError>`
   Re-export all of these + `ValidationBlock` in `pub mod wire`.
2. SSE client: an `EventSource`-based generator via `web-sys` (`EventSource::new(url)`, `MessageEvent` payloads → `StreamFrame`), OR fetch-stream of the NDJSON-ish SSE body; pick one and pin the decision in the module doc. The URL includes `?resume_from_seq=`. Since `EventSource` cannot set auth headers, confirm honba-ya's SSE route tolerates cookie-less dev (Trunk proxy on `127.0.0.1:8080 → :8000`).
3. Version pin: all decodes go through the envelope helper already in the client (`API_VERSION`/`SCHEMA_VERSION` checks). Do not hand-roll a second decode path.
4. Contract probes as **wasmtests or curl scripts** recorded in `honba-kao/scripts/` (e.g. `simulator-smoke.sh`): hit a live `honba-ya-cli serve` and assert `/backtests` reject a seed-free request with the documented error, accept `{"strategy":"sma_cross","universe":"TCS.NSE","start":"2025-01-01","end":"2025-06-01","bar_spec":"1d","seed":42}` etc. If no live server, rely on golden JSON (next item).
5. Golden decode fixtures: copy/truncate real responses into `honba-kao/schema/conformance/` (or `crates/honba-kao-client/tests/fixtures/`): `bars_response.json`, `backtest_response_completed.json`, `backtest_response_failed.json`, `trades_response.json`, `stream_frame.json?`. Unit tests decode them both raw and enveloped.

**Agent hints**

- The server is enveloped — **`{"data": {...}, "status": ...}`** style `ResponseEnvelope`; every route in `honba-ya-api-rest/src/lib.rs` wraps (`ApiResponse::success`), including errors. Decode through the same helper `POST /orders` uses; do not write bare-JSON decodes.
- `UnixNanos` is nanos-as-string; **never** `as f64`, never JS `Number()`. In Rust, `honba_api`'s types handle it; when you eventually convert to chart seconds, slice/parse the string (fe does `unixNanosToSeconds` on the string) to avoid precision issues with huge u64s.
- `BacktestRequest` is `deny_unknown_fields` — since we serialize the crate's own struct from Rust, this is automatic; **do not build the request as a `serde_json::json!`** map (that reintroduces typos the resolver would reject at distance).
- Journal endpoint returns JSON `TradesResponse` (it is **not** NDJSON despite the journal being NDJSON on disk — the server maps it). Audit claim confirmed in `runs.rs`.
- `RunStatus` on the wire is `pending|running|completed|failed|cancelled`; fold `pending|running → running` in UI state, exactly like fe's `foldStatus`.
- honba-ya's strategy side names and enum spelling come from `/strategies`; never hard-code aliases.

**Acceptance:** `cargo test -p honba-kao-client` green incl. golden fixtures; `cargo check --all-features` workspace-green; SSE choice documented in the method doc comment.

---

### Phase 1 — Controls and request planning kernel (`sim/controls.rs`)

**Goal:** pure, unit-tested control model matching `BacktestRequest::resolve` rules.

**Work items**

1. Port `controls.ts` into `sim/controls.rs`: `PeriodPreset`, `PERIOD_PRESETS`, `TIMEFRAMES`, `FILL_MODELS` (from crate), `FillModelDescriptor` table, `DEFAULTS` (single source, resolving W3 — default timeframe `'5m'`), `SimulatorControls` struct, `resolve_range` (preset arithmetic: day-clamped `minus_months`, YTD, `'YYYY-MM-DD'` formatting, **IST calendar-free** — it is pure date math), `universe_for` (`SYMBOL.EXCHANGE`), `control_issues` (symbol empty, capital ≤ 0, commission > MAX_COMMISSION_PER_FILL, slippage > MAX_SLIPPAGE_BPS, seed 0, custom range start ≥ end), `plan_request(controls, strategy) -> ControlPlan`.
2. Mirror the resolver rules in tests copied from `controls.test.ts` (period presets, request mapping, validation).
3. Seed default 42, capital 100_000, benchmark `NIFTY 50`, fill model `bar_fill`.
4. `bar_spec` grammar helper: `parse_bar_spec("5m") -> Some((5, Minute))` (months handled as unit without fixed seconds) — used by chart-data and marker tolerance in P2.

**Agent hints**

- Rules must stay in **sync by construction** with `honba-ya-api`: prefer re-reading `BacktestRequest::resolve`'s doc comment; if discrepancy is found, honor the server (it is the source of truth) and file a note in the doc.
- Do not port `toBacktestRequest`'s `notModelled` mechanism yet — today it is always `[]`; keep the field in `ControlPlan` for parity but no content beyond a pending-HB-field note.
- Dates: keep strings (`"YYYY-MM-DD"`) as the currency of truth until request build (avoids timezone drift); local date math must be pure string/integer-based (no `chrono` in wasm unless already in the tree; if adding chrono, fix `default-features = false, features = ["clock", "alloc"]`-style minimalism or prefer hand-rolled helpers like fe does).

**Acceptance:** `cargo test -p honba-kao-app sim::controls` green; test names mirror the vitest `describe` blocks.

---

### Phase 2 — Bars: fetch, conversion, resampling, preview (`sim/chart_data.rs`, `sim/persistence.rs`)

**Goal:** a `ChartBar` series for the chart and the replay engine, from server or labelled preview.

**Work items**

1. `chart_data.rs`: `unix_nanos_to_seconds(&str) -> Option<i64>` (string parse), `to_chart_bars(&[WireBar]) -> Vec<Bar>` (dedup by time, later wins, ascending), `parse_bar_spec` from P1, `bucket_start` (IST day/week/month buckets — constants `IST_OFFSET = 5.5h`, session 09:15–15:30 IST), `resample_bars(bars, spec)` (**never emit empty buckets**, fold OHLCV into first-bar-name buckets preserving fe's rule: bucket key = first source bar's timestamp — copy the rationale comment), `marker_tolerance_sec(spec)`, `snap_to_bars`.
2. `preview.rs` (in `sim/`): the deterministic hash-driven preview generator (`preview_bars(PreviewBarsRequest)`) — weekend-skip + IST-session + `MAX_PREVIEW_BARS = 4000` cap, labelled `source: 'preview'`. It is the offline dev fallback and the only reason the page works before data exists. Port its **hash constants exactly** so visuals match honba-fe during review.
3. Store wiring part 1: `sim/persistence.rs` with `LayoutStorage` trait → `LocalStorage` impl (best-effort, try/catch) and the `honba.simulator.selection` key.
4. Unit tests from `chart-data.test.ts` and `bars.test.ts` (golden cases incl. bucket boundary).

**Agent hints**

- `Bar` in `types/chart.rs` is already the chart shape (time i64 UTC s). Keep it; the wire→domain conversion is the only place that knows about nanos.
- fe's session/window constants are UTC-second-of-day derived from IST; keep the derivation comment (why: NSE session 09:15–15:30, no DST).
- Do not compute anything from the preview inside `analytics` (P3): the preview is a chart *input* only, never a metrics source.
- Keep bar dedup rule (dedupe by time, later wins) even if it looks odd — the journal/bars endpoints may duplicate `ts_event`s on session opens.

**Acceptance:** `cargo test -p honba-kao-app sim::chart_data` green; preview determinism test passes (same seed ⇒ same output twice).

---

### Phase 3 — Analytics kernel (`sim/pairing.rs`, `sim/analytics.rs`)

**Goal:** fills → round trips → honest metrics; CSV export.

**Work items**

1. `pairing.rs` — full port of `trade-pairing.ts` (403 lines): `fills_from_trades`, `pair_fills` (per-instrument, `ts_event` order; episode = net position leaves zero → back to zero or flip; scale-in joins, scale-out FIFO consumes; pro-rata entry-cost allocation; flip splits residue into a new episode with `exit_reason: flipped`; open-at-end → `status: open`, `exit_reason: open_at_end`), `markers_from_pairing`, `trip_for_marker`. This 500-line module is the most delicate port in the doc; carry over the module doc ("everything a round trip reports is derived… no exit reason ever guessed").
2. `analytics.rs` — port of `analytics.ts` (552 lines): `round_trip_stats`, `merge_intervals`/`exposure_fraction`, `equity_series` (daily grid when window known; per-closed-trip otherwise), `benchmark_equity` (buy&hold normalized; `None` unless daily sampling), `monthly_heatmap` (IST bucketing; `return_pct` on running realized equity; `HeatmapCell`), `ist_year_month`, `journal_rows`, `to_csv` (RFC-4180 quoting, CRLF, hidden slippage column empty by rule 4), `format_timestamp`, `benchmark_instrument_id` (`'NIFTY 50' → 'NIFTY50.NSE'`), `derive_ribbon_metrics` (exact emitted key order: `total_return` wire, `cagr` derived, `sharpe` wire, `sortino` **n/a**, `calmar` derived, `max_drawdown` wire, `win_rate`, `profit_factor`, `trades`, `exposure`), `years_between`/`annualised_return` (365.25 d/y, `(1+r)^(1/y)−1`, `≤ −1 ⇒ −1`), `format_metric_value`.
3. Golden tests translated from `analytics.test.ts`, `trade-pairing.test.ts` (long ones — keep names).

**Agent hints**

- Money is integer minor units on the wire (paise). All *presentation* of currency divides by 100 at render time only; analytics keeps raw floats like fe (`pnl` is float from server `net_pnl` too). If a `Money` value sneaks into analytics, stop and convert to f64 (document where).
- `exposure_fraction` needs the window (`window.start/end`); if absent → `n/a`.
- The heatmap's footer reconciliation ("realised total may differ from payload mark-to-market total") is a *copy* requirement, port the sentence.
- CSV: the tabular CSV must contain exactly the visible journal rows (build CSV from the same `journal_rows(trips)` the table uses — do not copy fe's divergent duplicate (W5)).

**Acceptance:** `cargo test -p honba-kao-app sim::pairing sim::analytics` green, including the flip/scale-in/open-at-end golden cases.

---

### Phase 4 — Simulator store + run lifecycle (`stores/simulator.rs`, `sim/log.rs`)

**Goal:** the reactive heart: select → plan → run → journal → analytics memoized; SSE log.

**Work items**

1. `SimulatorStore` (idiom of `stores/trading_mode.rs`): signals for `strategy: RwSignal<StrategyGraphShape>` *(placeholder import from wherever the strategy-studio store lands — see hint)*, `strategy_name`, `origin: RwSignal<SelectionOrigin>`, `controls: RwSignal<SimulatorControls>`, `run: RwSignal<SimRunState>`, `journal: RwSignal<JournalState>`; `Memo` for `plan`, `trips` (pairing over journal fills), `analytics_bundle` (ribbon/equity/heatmap/rows). Methods: `hydrate`, `set_controls(patch)`, `select(graph, name, origin)`, `execute(client, opts)`, `reset_run`, `last_plan()`.
   Warnings live in the same store as fe (wiping journal on new run, superseded-run guard via generation counter instead of runId compare, poll cap → `failed`).
2. `execute` lifecycle, ported exactly: wipe journal + set plan → if `plan.issues` non-empty, failed state with `Blocked: …` logs (no request sent) → status `running`, "Compiling the strategy…" log → `post_strategies(manifest)` → on failure, failed → `post_backtests(request)` → poll `get_backtest(r_id)` every 1 s up to 120 → on `completed` + `run_id`: load journal (guard), persist selection.
3. `sim/log.rs`: `StreamFrame` fold (from `events.ts`): `frame_to_line` (`#12 status: running`), `fold_run_log` capped at 500 lines, `RunLogState`, plus the SSE consumer (`EventSource` per P0 decision) started when the drawer is open, `resume_from_seq` on reconnect, `on_cleanup` close.
4. `hooks/use_interval.rs` or explicit `spawn_local` loop for polling; `on_cleanup` must abort.

**Agent hints**

- Leptos gotcha: do not hold a `RwSignal<impl Clone>` of the whole machine state as one struct — granular signals let the chart, ribbon, and journal re-render independently (that's what fe's zustand selectors did subtly; the monolith store read whole-store in some components — a perf antipattern, don't replicate).
- Never spawn run loops in `create_effect` on the same signal it writes — use explicit actions (`execute`) and `spawn_local_scoped`; otherwise infinite update loops.
- The strategy graph type: honba-kao's `/strategy` page is a placeholder today. For unattended agents: define a minimal local `StrategyGraphShape { nodes, edges, universe, driving_timeframe }` in `sim/selection.rs` now with the same JSON field names as fe's handoff (`sessionStorage` item from the algodesigner — in kao, the handoff key is whatever strategy-studio-on-kao writes; until that exists, read `honba.strategy.handoff` from `sessionStorage` with try/catch and treat absence as "template mode").
- Marry compile + run like fe: `strategy` in the request = server's `compiled.id` (no spelling hacks — W2).
- Poll loop: cap of 120 polls × 1 s. On cap, **fold to failed with a "timed out" log** — never leave the UI running forever.

**Acceptance:** store unit tests with a fake `Client` trait mock (`#[async_trait]`-style test double) covering: blocked-request path, failed compile, successful run, poll-cap, superseded-run journal guard. `cargo test -p honba-kao-app stores::simulator` green.

---

### Phase 5 — Results surface (no chart yet)

**Goal:** present the completed run honestly, before charts: ribbon / journal / heatmap / validation / log tabs, plus the raw request payload.

**Work items**

1. Replace `pages/simulator_page.rs` scaffold with: `console` + `PageHeader` + arena area (empty for now) + bottom dock with `HfTabs`-style tab strip (`DetailTab`-enum pattern like screener store): `overview | journal | heatmap | validation | logs | payload` (fe also shows `signals/portfolio` — defer to P7).
2. `metrics_ribbon.rs`: provenance chips (`reported` muted, `derived` gold, `n/a`), passthrough of `format_metric_value`.
3. `journal_table.rs`: sortable columns (same 8 keys), text filter, net-total footer, CSV export (copy idiom from `stores/screener.rs::export_csv`, but data from `journal_rows`).
4. `monthly_heatmap.rs`: grid styled solely from semantic tokens + `color-mix` alpha function `0.12 + min(1,|pct|/maxAbs)×0.68`; year = current, current−1 default per fe (or from data years if data has older — port fe's exact two-year rule to avoid scope creep, note for future).
5. `validation_badges.rs`: PASS/FAIL chip, trust badges via a ported `sim::ops::trust_badges` (DSR/PBO/holdout/walk-forward/monte-carlo/cost-stress keys), `assumptions.not_modelled` chips, and the "no validation block ⇒ nothing is inferred" copy.
6. `run_log.rs` (drawer): poll lines + SSE lines, "streaming…/poll only" tag.
7. `style/simulator.css`: all new classes BEM `sim-*`; zero literal colors.

**Agent hints**

- KPI Sharpe: render wire value only; if `metrics` absent → whole overview area shows an empty state, not the fake 2.14 (W1).
- `Payload` tab: `<pre>{request}</pre>` via `serde_json::to_string_pretty` of the stored `BacktestRequest` plus the benchmark explanation paragraph and not-modelled note from fe's copy.
- Heatmap year deduction: port `ist_year_month` algebra; do not depend on `Intl`.
- No `use_effect` refetch implications here (charts come in P6); keep this render purely from memos so the audit-free path exists standalone.

**Acceptance:** page renders with a completed mocked run (fixture-driven via the store's test double in a `#[wasm_bindgen_test]` or manual fixture button behind `cfg(debug_assertions)`); no hard-coded hex colors; check `cargo clippy` clean for the new modules.

---

### Phase 6 — Chart integration

**Goal:** the candle chart with the strategy's indicators and trade markers; the equity curve.

**Work items**

1. `sim/chart_series.rs` — port `chart-series.ts`: `registry_id_for(kind)` (mutation observation: `rma` and `psar` were unsupported in fe's registry — re-probe against the vendored bundle and update the map), `settings_for` (param aliases `period→length`, `fast→fastPeriod`, …), `describe_kind`, `graph_specs(graph)` (basis/band coloring), and the resolve-vs-registry step via the bridge's `indicatorCatalog` (kao already exposes `window.honbaCharts.indicatorCatalog`).
2. `simulator_chart.rs` (store-wired container): load bars via client (effect keyed on `instrumentId/start/end/tf`), wrap state `BarsState` (status/source/labels from fe), fall back to `preview` if the fetch errors *and* include the `source: 'preview'` chip; hand bars to the store for engine use (P7); measure container via the existing `NodeRef` + `Effect::new` + retry mount loop idiom of `openalgo_chart.rs`.
3. Bridge extension in `vendor/openalgo-charts/honba-bridge.js` (and `docs/widget-integration.md`): `setMarkers(instance, markers)` and a volume pane path; probe the bundle API first (list what changelog exposes — `3.0` bundles include markers API; the vendored `2.6.6` reportedly includes `createMarkers`, per fe's use).
4. Markers: `snap_to_bars(markers_from_pairing(trips), bars, marker_tolerance_sec(tf))`; glyphs: entry `arrowUp` belowBar (buys) / `arrowDown` aboveBar (sells); `open` dot circle; tooltip on click showing Price/Size/Instrument/trip PnL/exit reason + the R5 footnote copy.
5. Equity curve: rewire existing `equity_chart.rs` demo to `analytics.equity_series` + `benchmark_equity` overlay (labelled `benchmark` in `shades`); also `drawdown_series` (from `sim/ops.rs`, port `drawdownSeries`, `equity_from_curve`, `max_drawdown_pct`).
6. Conventions: colors from CSS vars documented in `widget-integration.md` (`--chart-*`, `--pnl-gain/loss`); timezone `Asia/Kolkata` in the widget theme object.

**Agent hints**

- Add/remove indicators per `spec.key(node id)` — stable across edits so a param tweak doesn't blink the series.
- Never `await` bridge calls inline in render; all via `Effect::new` + `spawn_local`.
- volume histogram if pane-unsupported: emit `high = volume, open/low/close = 0` colored `rgba` from `--chart-volume` var if we have to add one — pick the token route, no hex.
- Dead-code check: fe's `PENDING_NOTE` const was unused — don't port ASCII relics.
- Re-anchor behavior: replay keeps a 40–100 bar leading window; static view anchors to last 180 bars on `seriesKey` change only. Port both constants; document them as consts in `simulator_chart.rs`.

**Acceptance:** with fixture journal + live/preview bars: candles + volume + ≥2 indicator series + markers with tooltips + equity area with benchmark; theme switch recolors without remount; `cargo test` green and manual `make serve` checklist noted in the module docs.

---

### Phase 7 — Replay engine (Track B, sized down)

**Goal:** animated bar-by-bar replay with client-side execution and confluence display, reusing existing infrastructure.

**Work items**

1. `sim/engine/mod.rs` config struct + status machine port (`ReplayStatus idle|playing|paused|completed`, speeds `{1,5,10,25,50}`, `interval_ms = max(12, 180/min(speed,20))`, ticks-per-frame map).
2. Indicators: **do not port `indicator-calculator.ts`** — reuse `honba-kao-api-wasm::indicators` (already built on `honba-me-indicators` since it is in the dependency graph). Where a formula diff exists (EMA warm-up semantics, RSI smoothing), match fe's semantics and pin decisions in the module doc.
3. `engine/evaluator.rs`: port `strategy-evaluator.ts` (`compute_graph_indicators`-equivalent via wasm calls, `evaluate_bar_signals`: condition graphs → `crosses_above/lt/gt/and`, threshold heuristics 35/70, confluence ratio, order quantity defaults `params.quantity ?? 10`).
4. `engine/controller.rs`: `ReplayController` struct: 4-tick phases, forming candle synthesis for `on_bar_change`, execution models (strict `bar_fill_through`, `l2` adverse impact formula `max(2e-4, ((high−low)/close)×0.05) + slippage_pct` rounded to paise, default `bar_fill` at close, `none` signals-only), fees (`commission>0 ? commission : 10`, ×2 per round trip), one-open-position constraint, `jump_to_bar`, `instant_run_to_end`. Execution state (round trips, position, P&L) lives in **the replay's own store slice**, not the app portfolio store (W7).
5. Visible-bars driving: the store replays `visible_bars` into the chart (mutable prefix + forming bar); markers for engine trips where replay-active; keep the "both tracks simultaneous on Run" behavior from fe only after P6 is accepted — **initially**: Run button = server run only; replay cluster = bars/journal-driven scrub; later merge.
6. `pages/simulator_page.rs` adds the `signals` + `portfolio`(sandbox-local) dock tabs, replay toolbar (reset / play-pause / step / instant / speed pills / scrubber) as `components/simulator/replay_controls.rs` with the `engine_state` memo; confluence panel: bull/bear pill in the header + `signals` table in the dock.

**Agent hints**

- Speed maps: `speed>=50 ⇒ 8 ticks/frame, >=25 ⇒ 4, else 1` — port constants verbatim.
- Forming candle volume: `volume × (phase+1)/4` — the fractional-volume rule is a visible-behavior nugget, port it.
- Do not let the controller own Leptos signals — plain struct + a `RwSignal<u64>` "generation" bump the store subscribes to; replicate fe's callbacks as method-call returns the effect writes.
- `requestAnimationFrame` via `web_sys::window().requestAnimationFrame`; `on_cleanup` cancels the id — cheap and important.
- Position/`ActivePosition` shape is fe's; port fields exactly (`symbol, exchange, side, quantity, entry_price, current_price, unrealized_pnl, unrealized_return_pct, entry_timestamp`).

**Acceptance:** fixture bars replay at all speeds; speed=1 completes ≤ ~2 min on a 6-month 5m series; confluence pill toggles; engine shares bars with chart without extra fetches; `instant` completes synchronously-ish (needs ~ms-level per-bar eval on 27k bars → perf check §6).

---


**Goal:** markers on canvas, volume pane, and the remaining chart-bridge hooks deferred to the integration team.

**Work items**

1. Markers API: expose `setMarkers` through the vendored `openalgo-charts` bridge (call the widget's `createMarkers`/`setMarkers` layers); the bridge extension lives in `vendor/openalgo-charts/honba-bridge.js`. Use markers from `markers_from_pairing(trips)` (see Phase 7) — snap to bar timestamps within `marker_tolerance_sec(spec)`; glyphs: `arrowUp` belowBar (buys) / `arrowDown` aboveBar (sells); `dot` for open; click-tooltip shows Price/Size/Instrument/trip PnL/exit reason + W5 copy ("Slippage column stays blank").
2. Volume pane: if the widget API supports it, add a `SeriesOptions`-style histogram series; otherwise, add a synthetic "volume indicator" as a vertical bar span using `color-mix(in srgb, var(--chart-volume) 18%, transparent)` from CSS tokens, rendered in chart space never pixel space (W6).
3. `indicator-calculator.ts` is **not** ported wholesale; instead each needed series (SMA/EMA/MACD/Bollinger) is mapped through `chart_series.rs` registry lookups. The registry map already has SMA/EMA/MACD from the wasm indicators crate; add Bollinger only if the 5‑parameter windowed variance formula is needed and justified; otherwise note "omitted to P9" in the module doc.
4. Bridge export script: the chart crate will generate a `types.d.ts` for the bridge so downstream teams can call `chart.setIndicator(id, params)` without looking at the JS source each time.
5. If the widget cannot host markers, fall back to drawing markers as an SVG overlay in chart space (convert time→x via `chart.timeScale()`, exposed through the bridge snapshot/convert API), never pixel space.

**Agent hints**

- Markers always derive from the pairing engine output — never from fe's chart-click position.
- Volume tint respects the `--chart-volume` CSS var defined in `widget-integration.md`; never a literal hex.
- Do not `await` bridge calls inline in render; all via `Effect::new` + `spawn_local`.
- If the chart engine cannot expose `setMarkers`, record that as a P9 blocker in the module doc; the hook is a future gate.

**Acceptance:** markers appear on candle close phase ticks when replay-active; volume band respects skin tokens; series registry maps compile; no hard-coded hex anywhere; the P8/blocker checklist is documented in `widget-integration.md`.


### Phase 8 — Control panel + selection UI (P8 optional granularity)

**Goal:** full controls drawer + selection flow parity.

**Work items**

1. `sim/selection.rs` resolution order: `?strategy=` query → sessionStorage handoff → first saved/template (from `/strategies`) → built-in template (define one tiny SMA-cross template JSON in constants). `controls_for_graph` (timeframe from `driving_timeframe`, interval→`AGGREGATION_UNIT` mapping table, `params`→aliases).
2. `simulation_controls.rs`: sections Symbol (select from loaded instrument universe; fe used a modal with hardcoded anchors (W8) — instead use honba-kao workbench's symbol list if it loads from live `/quotes`-adjacent APIs, else a simple text field + known-symbols dropdown; no fake anchor table), Benchmark text input, Date Range (preset chips + native date inputs; chip-click sets `period:X`; manual edits set `period: Custom`), Timeframe (chips + grouped select), Initial capital / Commission / Slippage `NumberField`s with live clamp validation, Sandbox size note (replay fixed qty), Fill model select with badge + description.
3. The Run bar: Run (disabled when `plan.issues` non-empty or running); live issue bullets under the button; strategy selector (`active workspace (handoff) | global catalog (/strategies) | template`) with origin badge.
4. Multi-chart: defer (max 3 secondaries grid, shared bar set, `resample_bars` per pane) — **out of scope for v1 of the port**, listed as future work; saves ~300 lines (fe `multi-chart-view.tsx`) plus the sync toggle plumbing.
5. Drawing rail/canvas: deferred (W6) — listed as future work, needs chart-space coordinates.

**Agent hints**

- A missing strategy in `/strategies` ⇒ template with `origin: 'template'`. Never block the page; every empty state has copy.
- Controls validation runs on every derive (cheap pure fn) — do not memo-cap it in a way that hides edits;
- Instrument universe: honba-ya has no list-all-instruments endpoint in the audit; `/quotes?symbols=` is targeted. So symbol entry is a text input with validation against `^[A-Z0-9\\- .]+$\\.NSE` style; keep the fe rehearsal of `universe_for(controls)`.

**Acceptance:** can configure and fire a real run only through this panel (mock path included); selection hydrates from localStorage on reload; the disabled reasons are printed, not silent.

---

### Phase 9 — Multi-pane / Markers / bridge hooks (P8; owned by chart-bridge team)

**Goal:** markers on canvas, volume pane, and the remaining chart-bridge hooks deferred to the integration team.

**Work items**

1. Markers API: expose `setMarkers` through the vendored `openalgo-charts` bridge (call the widget's `createMarkers`/`setMarkers` layers); the bridge extension lives in `vendor/openalgo-charts/honba-bridge.js`. Use markers from `markers_from_pairing(trips)` (see Phase 7) — snap to bar timestamps within `marker_tolerance_sec(spec)`; glyphs: `arrowUp` belowBar (buys) / `arrowDown` aboveBar (sells); `dot` for open; click-tooltip shows Price/Size/Instrument/trip PnL/exit reason + W5 copy ("Slippage column stays blank").
2. Volume pane: if the widget API supports it, add a `SeriesOptions`-style histogram series; otherwise, add a synthetic "volume indicator" as a vertical bar span using `color-mix(in srgb, var(--chart-volume) 18%, transparent)` from CSS tokens, rendered in chart space never pixel space (W6).
3. `indicator-calculator.ts` is **not** ported wholesale; instead each needed series (SMA/EMA/MACD/Bollinger) is mapped through `chart_series.rs` registry lookups. The registry map already has SMA/EMA/MACD from the wasm indicators crate; add Bollinger only if the 5‑parameter windowed variance formula is needed and justified; otherwise note "omitted to P9" in the module doc.
4. Bridge export script: the chart crate will generate a `types.d.ts` for the bridge so downstream teams can call `chart.setIndicator(id, params)` without looking at the JS source each time.
5. If the widget cannot host markers, fall back to drawing markers as an SVG overlay in chart space (convert time→x via `chart.timeScale()`, exposed through the bridge snapshot/convert API), never pixel space.

**Agent hints**

- Markers always derive from the pairing engine output — never from fe's chart-click position.
- Volume tint respects the `--chart-volume` CSS var defined in `widget-integration.md`; never a literal hex.
- Do not `await` bridge calls inline in render; all via `Effect::new` + `spawn_local`.
- If the chart engine cannot expose `setMarkers`, record that as a P9 blocker in the module doc; the hook is a future gate.

**Acceptance:** markers appear on candle close phase ticks when replay-active; volume band respects skin tokens; series registry maps compile; no hard-coded hex anywhere; the P8/blocker checklist is documented in `widget-integration.md`.


## 8. Risk Register with Mitigations

| # | Risk | Mitigation |
|---|---|---|
| R1 | Tier-1 interpreter "violates" ADR 0029 D16 letter | The design explicitly adopts a *provenance model* — every Tier-1 number carries a `provisional` chip rendering as `[provisional]` in the UI. `goal_met` **cannot** trigger from Tier-1 numbers; it requires validation-tier server numbers. A doctored ADR-0029 delta is documented in the risk register (R1) and flagged for the ADR owner. |
| R2 | Server can't run structural candidates (`no_rust_implementation`) | Registry-first loop — only registry-carried names (`sma_crossover|rsi_reversal|buy_and_hold` + aliases) advance to Tier-2. All other candidates remain honest `promotion: blocked — no_rust_implementation` and stay §1-tier only. The loop keeps running; badged `promotion` chips surface the limitation. |
| R3 | `ValidationBlock` server-unattached | `honba-nou-analytics` crate (pure Rust, wasm-safe) computes the block client-side from journal return series. The design gates `goal_met` on `ValidationBlock.passed == true` **only** when server numbers exist; otherwise the honest footnote renders: *"The run payload carried no validation block; gates computed client-side from guest journal series (E4-S6 thresholds)"*. |
| R4 | Browser held the LLM API key → leak | Ollama is the default (no key leaves the machine). BYO-key options are masked-input fields with explicit warning: *"Stored locally, fake-visible never"*. A future "local researcher sidecar" is the documented upgrade path if a key never belongs in a browser. |
| R5 | 1000 iterations × LLM cost/burn | Budget meters (tokens/wall-clock/iterations) with cumulative cost meter. Journal dump on 3-fold provider failure. Offline heuristic provider for zero-cost dev runs. The loop kills/pauses/resumes across reload with state persisted to `honba.studio.research` localStorage. |
| R6 | Sweeps grids burst (big cartesian) | Cap 50 client-side; `trials` must equal grid length; registry-first honest copy. For non-registry graphs, the studio states the limitation rather than failing silently. |
| R7 | REST backtests don't route honba-mon RiskStage today | Stated as R7 in copy; the loop never claims executed-refused order metrics as server truth. The `refused_orders` from the ya-backtest engine only appear as documentation copy. |
| R8 | Reload mid-loop loses state | Persist-on-phase-transition (localStorage `honba.studio.research`) + resume path (P7 #5). Server `run_id` persists on journals; reload just re-polls the server run id. |
| R9 | Node canvas perf in wasm | One-Memo render pass (no per-node effects). P9 budget gate claims 50 nodes ≤ 16ms input→paint — verified against DES-004 canvas state. Polished budget claims that the DES-004 canvas state was fine after ~50 nodes. |
| R10 | honba-ya restart loses the compiled-candidate catalog → history breaks | Content-addressed `sha256:` ids mean re-compose refreshes; the loop re-compiles before tier-2 (`POST /strategies` id for `goal_met` continuity), never trusting in-process names. |

R9: Node canvas perf in wasm
---
R10: honba-ya restart loses the compiled-candidate catalog → history breaks
---
## 7. Definition of Done

1. Every module doc names its fe/docs reference and any semantic deviation.
2. `cargo test --workspace` + `cargo clippy --all-targets -- -D warnings` green; `make build` prod profile works.
3. All eight honbao-ka themes crisp; zero literal hexes.
4. Provenance vocabulary enforced everywhere: nothing renders `server`-looking numbers unless they came from honba-ya, or show a `provisional` chip if they came from Tier-1.
5. Propose-never-execute holds end-to-end: no unverified candidate can pipeline into a run; the draft ADR-0029-S5 chip appears on every non-verified proposal path.
6. The loop honors *every* research_loop.md "does NOT do" rule, executable assertions where possible (fail-loud loops over numeric prompting).
7. hongbao-kao statistical-validation thresholds (§5.5) are the loop's default goal gates; a user may lower them, with a one-line warning quote in the UI.
R9: Node canvas perf in wasm
---
R10: honba-ya restart loses the compiled-candidate catalog → history breaks
R9: Node canvas perf in wasm
R10: honba-ya restart loses the compiled-candidate catalog → history breaks
R9: Node canvas perf in wasm
R10: honba-ya restart loses the compiled-candidate catalog → history breaks
R9: Node canvas perf in wasm
R10: honba-ya restart loses the compiled-candidate catalog → history breaks
PHASE9EOF
wc -l honba-docs/docs/architecture/kao_simulator_port.md

## 8. Risk Register with Mitigations

| # | Risk | Mitigation |
|---|---|---|
| R1 | Tier-1 interpreter "violates" ADR 0029 D16 letter | The design explicitly adopts a *provenance model* — every Tier-1 number carries a `provisional` chip rendering as `[provisional]` in the UI. `goal_met` **cannot** trigger from Tier-1 numbers; it requires validation-tier server numbers. A doctored ADR-0029 delta is documented in the risk register (R1) and flagged for the ADR owner. |
| R2 | Server can't run structural candidates (`no_rust_implementation`) | Registry-first loop — only registry-carried names (`sma_crossover|rsi_reversal|buy_and_hold` + aliases) advance to Tier-2. All other candidates remain honest `promotion: blocked — no_rust_implementation` and stay §1-tier only. The loop keeps running; badged `promotion` chips surface the limitation. |
| R3 | `ValidationBlock` server-unattached | `honba-nou-analytics` crate (pure Rust, wasm-safe) computes the block client-side from journal return series. The design gates `goal_met` on `ValidationBlock.passed == true` **only** when server numbers exist; otherwise the honest footnote renders: *"The run payload carried no validation block; gates computed client-side from guest journal series (E4-S6 thresholds)"*. |
| R4 | Browser held the LLM API key → leak | Ollama is the default (no key leaves the machine). BYO-key options are masked-input fields with explicit warning: *"Stored locally, fake-visible never"*. A future "local researcher sidecar" is the documented upgrade path if a key never belongs in a browser. |
| R5 | 1000 iterations × LLM cost/burn | Budget meters (tokens/wall-clock/iterations) with cumulative cost meter. Journal dump on 3-fold provider failure. Offline heuristic provider for zero-cost dev runs. The loop kills/pauses/resumes across reload with state persisted to `honba.studio.research` localStorage. |
| R6 | Sweeps grids burst (big cartesian) | Cap 50 client-side; `trials` must equal grid length; registry-first honest copy. For non-registry graphs, the studio states the limitation rather than failing silently. |
| R7 | REST backtests don't route honba-mon RiskStage today | Stated as R7 in copy; the loop never claims executed-refused order metrics as server truth. The `refused_orders` from the ya-backtest engine only appear as documentation copy. |
| R8 | Reload mid-loop loses state | Persist-on-phase-transition (localStorage `honba.studio.research`) + resume path (P7 #5). Server `run_id` persists on journals; reload just re-polls the server run id. |
| R9 | Node canvas perf in wasm | One-Memo render pass (no per-node effects). P9 budget gate claims 50 nodes ≤ 16ms input→paint — verified against DES-004 canvas state. Polished budget claims that the DES-004 canvas state was fine after ~50 nodes. |
| R10 | honba-ya restart loses the compiled-candidate catalog → history breaks | Content-addressed `sha256:` ids mean re-compose refreshes; the loop re-compiles before tier-2 (`POST /strategies` id for `goal_met` continuity), never trusting in-process names. |

### Phase 7 — Research loop engine (LLM providers, journal, controller)

**Goal:** the heart of this doc. The loop state machine, provider dialing, structured hypothesis parses, the IndexedDB JSONL journal with similarity retrieval, and the honest LLM audit trail.

1. `research/types.rs` — all structs of §3 (`ResearchGoal`, `Iteration`, `LoopState` with fe's honest status/phase enums, `FailureMode` closed enum, `deny_unknown_fields` on all LLM-input shapes).
2. `research/providers.rs` — §3.6 contracts; Ollama (local, zero-risk default); OpenAI/Anthropic (BYO key, CORS warn); Disabled/manual (heuristic fallback). Every provider call surfaces tokens in/out and a per-loop cumulative cost meter (token-truth from the response; if a provider's cost is NULL, cost renders `n/a` honestly).
3. `research/propose.rs` — prompt assembly (`journal.unsimilar`, failure tally, parent chain, catalogue, constraints, explicit "do not optimize numerics; declare sweepable ranges"); response parsed into `Hypothesis` with **one retry with the parse error appended**, then a skip (record + count).
4. `research/journal.rs` — IndexedDB (`honba.studio.research`), append `ResearchRecord` (the SHA-256 `LlmRecord` fields honored), export journal JSONL + `research_log.md` text export; similarity index (bounded, deterministic) for the prompt.
5. `research/engine.rs` — the state machine: serial loop (3.2) with **operator controls** (pause, kill-switch = halts after current iteration; never mid-, respecting the doctrine's no-proposal-without-measurement discipline), budget meters (tokens/wall-clock/iterations), `goal_met` only on validation-tier truth (5.4/5.5), provider failures backoff ×3, 3 consecutive provider errors ⇒ `paused` with an explanation (loop never spins a provider bill).
6. Telemetry tab (fe's, honest this time): per-iteration table `(k, hypothesis id, statement, tier1 metrics, tier2 run_id, gates, failure_mode, tokens)`; `bestIterationIndex` selector; goal meter bars; **Monte-Carlo/walk-forward sections only render what the block contains** (fe's fakes are gone).

[AGENT HINT] All sealing happens in `spawn_local_scoped` steps; never `create_effect` the loop's own output signal (Leptos infinite-update trap; DES-005 P4 hint applies).

[AGENT HINT] The loop must survive reload: persist loop state (current iteration, phase, journal offsets, budget spent) to local storage keyed `honba.studio.research` on every phase transition, and offer "Resume" on remount. A reload mid-`backtesting` just re-polls the server run id (journals last).

[AGENT HINT] Never call `POST /backtests` for a candidate that fails `graph_issues` — that's not "cheap"; it wastes a server slot and might queue up to 64 more; keep gating a pure function.

**Acceptance:** manual-provider mode + fixture graphs run the whole loop end-to-end offline with deterministic sorted failures; a mocked local provider counts tokens per iteration; `goal_met` interpretation is a pure function over a testable `Iteration` chain (no DOM). Loop kills/pauses/resumes across reload.

