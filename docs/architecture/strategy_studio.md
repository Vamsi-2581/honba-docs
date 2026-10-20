# Honba Strategy Studio — AI-Native Strategy Workbench & Auto-Researcher

**Document ID:** HBD-2026-DES-006
**Status:** Draft for Implementation
**Version:** 1.0
**Last updated:** 2026-10-20
**Target Systems:** `honba-kao` (Leptos 0.7 CSR — this is a CSR-only design), `honba-kao-client`, `honba-kao-api-wasm`, `honba-ya` (REST backend, mostly as-is; two small gap fixes proposed but not in scope), `honba-nou` (patterns consumed; no server work)
**Reference Lineage:** noFX / bfx-hf-ui-core (panel architecture, dockable workbench motifs — the lineage honba-kao's widget library is explicitly modernized from, Apache-2.0 per `packages/ui-core/NOTICE.md`); Jesse (strategy-as-declarative-spec, backtest metrics summary, optimizer loop, journal discipline); honba-docs doctrine (`book/src/research_loop.md`, three-tier pipeline, statistical validation, backtest-bias guardrails); ADR 0016/0017/0018/0021/0029.
**Sibling documents:** HBD-2026-DES-004 (React algodesigner plan — the contract source), HBD-2026-DES-005 (honba-kao simulator port — shares P0 foundations, results surface, and handoff), `honba-kao/plan.md` §4.2.

---

## 1. Purpose

Build the honba-kao **Strategy Studio**: an AI-native visual + declarative strategy workbench for **Indian equities** (NSE session, IST, tick sizes, price bands, T+1 settlement, STT/cost model, intraday square-off) that:

1. Lets a human author a strategy graph (`StrategyGraphShape` v1 — the same contract as DES-004/DES-005) on a node canvas, with typed ports, a derived catalogue, code round-trip, snapshots, and a library.
2. Verifies every candidate against the server (`POST /strategies/verify`) and shows honest diagnostics (quarantine until verified — **propose-never-execute**, ADR 0029 S5).
3. Runs backtests and parameter sweeps through honba-ya REST (Tier 2 / Tier 3 of the documented pipeline) and presents results with the E4-S6 trust badges.
4. Hosts an **AI co-pilot** (chat, context briefs, candidate diff cards, apply/discard) and the flagship feature: the **AI Researcher auto mode** — a closed-loop research agent that mutates the strategy graph, evaluates each candidate, learns from a hypothesis journal of failures, and runs **up to 1000 iterations or until the researcher's stated goal is met**, entirely from the browser. No new backend service is required.
5. Interworks with the rest of honba: hands off to the Simulator (DES-005), reuses the workspace idioms of the Workbench/Screener, consumes honba-me indicators, honors honba-mon's risk doctrine at the authoring and proposal layers, and matches nou's determinism/audit patterns so a future server-side researcher can replay every session.

### 1.1 Why CSR (decision, not preference)

The user directive is **prefer CSR**, and the environment makes that the honest choice:

- honba-kao is already pure CSR Leptos 0.7 (Trunk, `mount_to_body`), and honba-docs precedent (DES-005) sets the port convention.
- **Nothing on the server can host the researcher today.** `honba-nou-llm::LlmProvider` is an unimplemented stub (`OpenAiProvider::complete` returns `Err(Transport("not implemented"))`; no `reqwest` wired; plan M0/M2 unchecked). There is no `/assistant/propose`, no prompt-queue, no GPU budget. Any "AI orchestration service" would be net-new infrastructure — exactly what "prefer CSR" excludes.
- The reactor pattern honba prefers — *the server is a deterministic adjudicator, the client is a dumb terminal* — inverts for research: the loop needs an LLM key, memory (hypothesis journal), and a stateful long-running controller. All three are naturally browser-side here, and they are cheap to keep there: 1000 iterations ≈ 1M tokens ≈ **$3** (research_loop.md's own budget) — a self-contained, user-owned script rather than a deployed service.
- The one thing CSR must never do is **falsely execute**: ADR 0029 D16 says the frontend never executes strategy logic. The design below honors the *spirit* (the browser never issues a **canonical** verdict) while explicitly amending the *letter* with a clearly-labelled Tier-1 **provisional screening evaluator** (§5.3, decision D4). This is a documented doctrine exception and is flagged for a ADR delta.

## 2. Lineage: what is taken from where

| Source | Taken | Not taken |
|---|---|---|
| **noFX / bfx-hf-ui-core** | Panel discipline (`HfPanel`/`HfPanelHeader`), dockable workspace motifs (open/close/width/focus-mode layout persisted per feature), the invariant that widgets stay presentational while a single app-owned store holds state | Dockview-style drag-to-dock (net-new engine; out of scope — the DES-004 experience proves a split+drawer layout is enough at this scale) |
| **Jesse** | Strategy as a **declarative spec** (the graph/manifest is the strategy; code is a rendering), the backtest **metrics summary** framing (a fixed documented set + derived table), the **optimizer loop** discipline (parameter ranges declared in the spec itself; sweeps over declared space), journal-of-trades habit | Jesse's Python strategy classes (honba's node graph replaces them), its equity-tick engine |
| **honba-fe algodesigner** (DES-004) | The entire `StrategyGraphShape` contract, node catalogue, the studio layout types, co-pilot message kinds, sweeps panel shapes | Its **mocked** AiQuantStudio engine (`ai-optimizer-engine.ts` returns 3 hard-coded iterations with `Math.random()` Monte-Carlo — the loop below replaces it with real calls), Monaco dependency |
| **honba-nou** | The harness patterns: closed-schema parses (`deny_unknown_fields`), the `LlmRecord` JSONL audit record, `PromptReplayer` determinism contract, `ValidationBlock` gates | Its (currently nonexistent) LLM transport |
| **Jesse + Indian market** | — | Framework specifics adapted to India: single-symbol-per-run universes, NSE bar specs, `SESSION_OPEN 09:15 IST`, T+1, freeze-quantity participation caps drawn from honba-mon's doctrine, square-off scheduling nodes |

## 3. The research loop — the contract of AI Researcher auto mode

**The loop is: LLM proposes structure, honba-ya (and the provisional evaluator) measures, the journal remembers.** The design is a direct implementation of `honba-docs/book/src/research_loop.md`, hardened by the statistical-validation and bias-guardrail doctrine.

### 3.1 The Hypothesis record (single source of truth for the loop)

Every loop turn is one hypothesis. Mirror of research_loop.md's pydantic schema, as a Rust struct parsed with `deny_unknown_fields` (the nou `DecisionParser` discipline: "Language models are untrusted for direct execution" — the same treatment a `Decision` got):

```rust
/// One turn of the researcher loop. Parsed from the LLM's JSON with
/// `#[serde(deny_unknown_fields)]` — junk is rejected before it can
/// mutate anything (nou agent pattern, DecisionParser::parse_str).
struct Hypothesis {
    id: String,                 // e.g. "hyp_<ulid-shape>"
    parent_ids: Vec<String>,
    statement: String,          // one-sentence claim
    rationale: String,
    graph: StrategyGraphShape,  // the full candidate (graphs are small: 5–12 nodes)
    failure_modes_addressed: Vec<FailureMode>,
    expected_edge_bps: i32,     // declared up-front, judged out-of-sample later
    self_critique: String,
}

/// The fixed failure-mode taxonomy from research_loop.md. Closed enum.
enum FailureMode { InsufficientTrades, CostDominated, OosDegradation, RegimeDependent,
    ParameterCliff, OutlierDriven, OverlappingPositions, LookAheadBias, ImplementationError, }
```

**Full-candidate graphs rather than deltas:** a delta-JSON risks application errors and double-verification; full graphs are small, self-contained, verifiable by construction, and diffing is a studio-side view (`graph_diff`, port of fe's `computeGraphDiff`).

### 3.2 Loop state machine

```
idle ─▶ reasoning ─▶ generating ─▶ screening ─▶ backtesting ─▶ validating ─▶ reflecting ─┐
  ▲                                                                                    │
  └──────────────────────────────── goal_met / limit_reached / halted ◀── … loop … ◀────┘
```

- `reasoning` — the journal is rendered into the prompt: unsimilar past hypotheses, failure-mode tally (how many of each kind have been observed), parent chain, and the *instruction* to address at least one observed failure mode.
- `generating` — the LLM emits the `Hypothesis` JSON. Parse failure (reject-unknown-fields, closed `FailureMode`) ⇒ retry once with the error embedded, then skip iteration and count it.
- `screening` — Tier 1 provisional evaluator (§5.3): indicators via `honba-kao-api-wasm`, next-open execution, ~seconds in wasm.
- `backtesting` — Tier 2 canonical: `POST /backtests` (server), only when the candidate can actually execute server-side (§5.4), with seed = `loop_seed + iteration` (journals byte-identical per manifest; honest reproducibility).
- `validating` — Tier 3 gates: `ValidationBlock`-shaped checks (§5.5) + the goal evaluation (§3.4).
- `reflecting` — failure-mode classification and journal append; the only state the LLM sees next turn.

Iteration speed: an iteration is ~1 LLM call + ≤1 backtest + a bundle of cheap client evals; the loop is **serial by design** (research_loop.md: "the journal is the memory of the loop" — parallelism breaks the learning signal) with the only parallelism being the sweep delegation (§5.6).

### 3.3 It does NOT do (ported verbatim from the doctrine, enforced)

1. **No numeric optimization by the LLM.** Parameter tuning is delegated to `POST /sweeps` (Tier 3) or the declared `ParameterSpec` ranges; the LLM only proposes structure and declares ranges.
2. **No market data in the prompt.** The LLM reads the hypothesis journal and results, not bars or news.
3. **No profitable strategies on demand.** The loop states in its UI ("of 100 hypotheses, ~60 fail screening; roughly 1 in 100 survives") — the goal meter conjugates *trials*, not promises.
4. **No executing an unverified candidate.** Quarantine until `verify` passes; the AI's candidate card says `PROPOSED — NOT EXECUTED (ADR 0029 S5)` until applied and verified.

### 3.4 Goals & stop conditions

```rust
/// Shared with the loop UI's goal sliders (fe OptimizationGoal, honest this time).
struct ResearchGoal { objective: GoalObjective, symbol: Symbol, timeframe: String,
    min_sharpe: f64, max_drawdown: f64, min_win_rate_pct: f64, min_profit_factor: f64,
    min_trades: u32,     // doctrine: fragmentary-statistics defense
    max_iterations: u32, // default 1000 (user spec); 1..=1000
    token_budget: u32, wall_clock_budget_min: u32 }
```

Stop when **any** of: `goal_met` (all sliders + gates green on the best candidate, evaluated on validation-tier numbers, not provisional ones) | `max_iterations = 1000` reached | token/wall-clock budget exhausted | operator pause/kill-switch. Statuses mirror fe's honest set: `idle|running|paused|goal_achieved|limit_reached`.

### 3.5 The journal (memory)

Port of research_loop.md's journal, sized honestly for a browser:

- Append-only **JSONL** of `ResearchRecord { hypothesis, tier1, tier2_backtest_id, metrics, validation, failure_mode, embedding?, ts }` persisted in **IndexedDB** (via a tiny `web-sys` crate handle) with localStorage mirror of the *last used provider settings*; export button writes `research_journal_<ts>.jsonl` (CSV fallback for `research_log.md`-style humans).
- **Similarity memory** replaces FAISS honestly: deterministic hashed-shingle TF-IDF similarity over `statement + rationale + node-vocabulary` (pure Rust in the wasm crate), top-K unsimilar retrieval for the prompt. Constants and a test-fix percentage are pinned; no "AI recalls everything" claims.
- Replay parity: every record is written with `prompt_sha256`/`output_sha256`/`model_version` like nou's `LlmRecord`, so a future server-side researcher (`honba-nou` growing a real `LlmProvider`) can replay this journal through `PromptReplayer` semantics. That is the long-term interworking this design buys with zero server work today.

### 3.6 Provider abstraction (client-side)

```rust
#[async_trait]
trait ResearchProvider {            // browser fetch, no server in this path
    async fn complete(&self, req: ProviderRequest) -> Result<LlmTurn, LlmError>;
}
struct ProviderRequest { prompt: String, json_schema: &'static str, max_tokens: u32, temperature: f32 }
```

Built-ins (BYO key in localStorage `honba.studio.research.providers` — UX: masked input + explicit "stored locally, fake-visible never" warning):

1. **OpenAI-compatible** (`POST {base}/chat/completions` with `response_format: json_object` or tool-call JSON schema when offered) — covers OpenAI/DeepSeek/Ollama-with-OpenAI/other proxies; flags CORS risk in docs.
2. **Anthropic** (`/v1/messages`) — browser key CORS is provider-dependent; warn loudly.
3. **Ollama local** (`http://localhost:11434/api/chat`, `format: json`) — **the zero-risk default**: no key leaves the machine, latency badge proves it isn't running on thin air.
4. **Disabled/manual** — no provider: the loop falls back to a **deterministic local heuristic proposer** (port of fe's keyword-proposals, upgraded: mutates non-sweepable structural axes only) so the whole loop is exercisable offline in tests (this is also the test double).

Provider selection is per-loop-run; a provider error backs off (3 attempts × exponential), and 3 consecutive provider failures trip the loop to `paused` with a journal dump.

## 4. Legacy contract recap — the graph/manifest model (from DES-004, unchanged)

The studio authors **`StrategyGraphShape` v1** — identical to what DES-005's simulator consumes and to what honba-ya verifies:

- Node types (10): `universe, indicator, transform, condition, logic, sizing, risk, order, schedule, benchmark`; typed ports `series|bool|size|order` per `NODE_PORT_MAP`.
- `StrategyManifest` = graph minus canvas extras (`strategy_api_version`, `parameters`); server `POST /strategies/verify` answers `StrategyIr`; `POST /strategies` compiles to a content-addressed `sha256:` id in a session catalog (≤1000, idempotent — **this id is the loop's candidate cache key**).
- Values that matter for the loop, verified in `honba-ya` source: verify errors come back as 422 with `context.reason = <IrError code>` (`no_subscriptions`, `SubscriptionOutsideUniverse`, …) — **prompt-string material, by design agent-friendly**. `source`/`code` keys are refused (`source_unsupported`).

## 5. Architecture in honba-kao

### 5.1 Module map

```
crates/honba-kao-app/src/studio/
  mod.rs                     pub use; page composition
  graph/
    types.rs                 StrategyGraphShape/GraphNode/GraphEdge/PortKind (serde-shaped, wire-neutral)
    validation.rs            graph_issues() — ported 12 codes + the 4 new codes in §6.2
    catalogue.rs             node catalogue + whitelisted NSE indicator set (§5.2)
    hash_id.rs               graphHashSync port (FNV-1a ×8 formatted "sha256:<hex>") + real sha256 via wasm
    manifest.rs              to_manifest(); rendering as serde_json (deny_unknown_fields by server)
    codegen.rs               generate_code() — Python strategy source, deterministic
    recognize.rs             recognize source → {graph, spans, code_only} (parser port; used by the code pane)
    diff.rs                  graph_diff + LCS source diff rows
    snapshot.rs              versions[] snapshot/restore (localStorage honba.studio.versions)
  layout.rs                  StudioLayout (port of designer layout; localStorage honba.studio.layout)
  library.rs                 SavedStrategy/Manifest, upsert/fork, library sync (localStorage honba.strategies.library)
  handoff.rs                 stash/take to Simulator (keys shared with DES-005 §6)
  stores/mod.rs              StudioStore (RwSignal struct, idiom of stores/trading_mode.rs)
  canvas/                    node editor (§6.3)
  panels/                    verify.rs run.rs sweeps.rs results.rs ops.rs
  copilot/                   assistant.rs chats, context.rs brief renderer
  research/
    mod.rs                   use_research(); page mount
    types.rs                 Goal/Hypothesis/Iteration metrics/LoopState
    providers.rs             ResearchProvider impls (openai-compatible/anthropic/ollama/manual)
    propose.rs               prompt builder + reject-unknown-fields parse + retry-one-shot
    journal.rs               IndexedDB JSONL + TF-IDF similarity (`honba.studio.research`)
    engine.rs                the loop state machine (spawn_local_scoped + sleep, pause/kill)
    eval/
      tier1.rs               provisional screening interpreter (§5.3)
      gate.rs                client-side ValidationBlock gates (§5.5)
      parity.rs              server-vs-tier1 parity harness (§5.4)
  components/…               goal sliders, iteration table, telemetry tabs, journal drawer
```

Plus `honba-kao-client` methods (DES-005 P0 keeps `get_strategies/post_strategies/post_strategies_verify`; this doc adds `post_sweeps/get_sweeps_by_id`) and an **honest `honba-kao-api-wasm` addition**: `research.rs` — tier-1 interpreter + similarity index + FNV hash (pure, golden-testable, mirrors the existing `replay.rs`/`screener.rs` pattern in that crate).

### 5.2 Node catalogue — the NSE whitelist

`honba-me-indicators` implements exactly 9 (verified): `sma, ema, rsi, macd, bollinger, donchian, atr, stochastic, williams_r`. The palette shows **those 9 as *verified*** and any other kind as *unverified-provisional* (badged, evaluated in Tier 1 only, cannot enter a `goal_met` status until server-side coverage exists). No `rma`, no `psar` today (DES-005's registry map was already pinned to this list). Derived catalogue rule (ADR 0029): the studio mirrors the registry shape; the source of truth is the honba-me list held in one `const` table, marked for regeneration when indicators land upstream.

NSE authoring constraints encoded at the palette/validation layer (not as folklore): sessions 09:15–15:30 IST; bar specs reuse honba-ya's `bar_spec` grammar (`(\d+)(s|m|h|d|w|mo)`); sizes are lot/kill aware in the UI copy: "freeze_quantity / participation honored server-side by the fill engine; client-side the sizer clamps to a paise-rounded, lot-multiple value" — the honest version is that **the runtime guards live in honba-mon**, and the studio surfaces them as *constraints the server can refuse*, never as local guards.

### 5.3 Tier-1: the provisional screening evaluator (doctrine exception)

**Decision D4 (doctrine-letter exception — see §8 risk R1).** Porting research_loop.md's tiering: tier 1 is canonical in the docs but unreachable on this server for structural candidates. The studio therefore runs tier-1 **client-side**, and the design's rule is *provenance, not prohibition*:

1. Fetch bars for the candidate's `universe.explicit[0]` via `GET /bars` (falls back to DES-005's labelled `preview` bars — never presented as history).
2. Compute indicator node outputs via `honba-kao-api-wasm::indicators` (same wasm path the chart uses).
3. Interpret the graph: condition/logic/sizing/risk/order per port semantics, **fills at next bar open (ADR 0016 `NextOpenExecution`), sells before buys, T+1-aware**, cost via a port of `IndianCostModel` constants (STT/classic charges, slippage bps from goal config).
4. Metrics = the fixed 5 (`trades/net_pnl/sharpe/max_drawdown/total_return`) + `win_rate/profit_factor/expected_edge_bps` computed client-side.
5. Every surfaced number is stamped `provisional — non-authoritative` (a rendered chip on every card; impossible to colour it as server `reported`); **`goal_met` can only trigger from validation-tier/server numbers** (§3.4). The provenance vocabulary from DES-005 (`reported|derived|provisional|n/a`) is shared so consumers can't mislabel.
6. Speed contract (P9 budget): interpreter + indicators ≤ ~50ms per candidate on a 1-year 5-min series on dev hardware; the screening stage of 1000 iterations must never exceed the tier-2 queue by even 10×.

**Tier-2 mapping:** a candidate is *canonically runnable* iff `strategy_ir.universe` compiles to one `SYMBOL.EXCHANGE` and the server registry carries its name (as of this doc: `sma_crossover|rsi_reversal|buy_and_hold` + aliases). Structural candidates that fail the runnable map are honestly labelled `promotion: blocked — no_rust_implementation` and the loop keeps them §1-tier only, exactly per research_loop.md's tier gates (Tier-1 Sharpe > 0.3 → candidate advances when server-side coverage catches up; auto-check via `GET /strategies` each run). A **proposed server gap fix** (out of scope, recommended): a graph-interpreter Strategy for the catalog, which would flip this block permanently without changing this design (the studio would simply lose the `provisional` flag).

### 5.4 Tier-2 + tier-3 evaluation

- A tier-2 run is `POST /backtests` with `seed = loop_seed + i` (non-zero, required), `strategy = <canonical id>`, `universe = "SYMBOL.EXCHANGE"`, `bar_spec`, window fixed for the whole loop (the same dataset for every candidate; changes invalidate the loop's comparability), costs from the goal sliders. Queue-realism (verified): the server is a bounded queue (64, `429 run_queue_full`) with `available_parallelism()` workers; the loop backs off on 429 and resumes; every successful run's RLULID `run_id` is journaled.
- Tier-3 numeric work: when a candidate has `sweepable` parameters (manifest `ParameterSpec` ranges — the sweep space is declared in the manifest by design: `sweep_space_from_manifest`), the studio builds a call `POST /sweeps {strategy, params: {name: [low, high, step]…}, trials: <grid len>, …}`. Honest handling of today's server (verified): sweeps are grid-only, `trials` must equal the grid size, `ranked` rows are indicative-only, param mapping is per-strategy-name hard-coded, and `custom` factories are stubs — so a sweep is issued **only** for registry-carried names and the studio labels those results `server: slice of declared space`. Anything beyond the grid (adaptive/random search) stays out of the LLM by the doctrine and out of the client by design until honba-ya grows a sampler (flagged in §8 R6).
- **Equity series for candidates** comes from the run journal (fills → round trips → equity, DES-005 P3 shared code), not from `/reports/equity-curve` — that endpoint is portfolio-derived with a pseudo-random price walker (verified) and would lie in a loop context. Honesty pin for the whole document.

### 5.5 Gates: client-side `ValidationBlock` + goal check

honba-nou-analytics' block (`DsrResult/PboResult/WalkForwardResult/HoldoutResult/MonteCarloResult/CostStressResult` + defaults verified in source) computes today but the server path attaches `validation: None` (`to_backtest_response`, `sweep_report_to_response` — verified). This design therefore:

- Adds `honba-analytics (honba-nou-analytics)` as a `workspace.dependencies` path dep of `honba-kao` (same pattern honba-ya already uses) — it is a pure-Rust serde crate, wasm-safe; `gate.rs` feeds return series from journal-derived equity into `evaluate_backtest(...)`/`evaluate_sweep(...)` **client-side** to produce the real block per candidate.
- Enforces the doctrine thresholds as the **loop goal defaults**: `ValidationThresholds { min_oos_is_ratio: 0.5, min_deflated_sharpe: 0.5, min_trades: 100, max_top5_contribution: 0.30 }`. `goal_met` requires the full block's `passed == true` on a *server-tier* metrics series, never tier-1.
- Renders trust badges with the DES-005 vocabulary (`reported|gates: DSR / PBO / holdout / walk-forward / monte-carlo / cost-stress`), and the honest footnote when the server sends nothing ("The run payload carried no validation block; gates computed client-side from guest journal series (E4-S6 thresholds)").

### 5.6 Studio workbench (noFX lineage, honba-kao-idiomatic)

Ported from fe's algodesigner, adapted to honba-kao primitives:

- Layout: `console`-shell + split canvas/code library/assistant + 300px bottom drawer (`verify|run|sweeps|results|ops`), all `RwSignal`-driven, persisted `honba.studio.layout`; focus mode mirrors fe's.
- Node canvas: **custom Leptos SVG** (no xyflow — see §6.3); scale-inverse rendering with `Background gap=16`, snap-to-grid 16.
- Store: `StudioStore` single reactive struct, granular `RwSignal`s; every graph mutation also re-derives `source` (codegen) and `sourceLines` atomically (fe's invariant).
- Handoff: simulator handoff keys are **the same strings** DES-005 documents (`honba.simulator.handoff`, sessionStorage) — the studio stashes and routes; the simulator hydrates both tracks from it on arrival (DES-005 P8 reads this).

### 5.7 Interworking map (one line each)

- **honba-ya REST** — verify/compile (with content-addressed id cache), backtests (seeded) + SSE events, sweeps (grid), bars for proxies. Shares honba-kao-client with DES-005.
- **DES-005 Simulator** — `stash/take` handoff; the Simulator's SelectionOrigin/`SelectionInput` already accepts handoff-first resolution.
- **honba-me-indicators** — the palette whitelist and tier-1 compute path.
- **honba-nou** — `LlmRecord` JSONL shape + `PromptReplayer`-compatible audit (`prompt_sha256`/`output_sha256`/`model_version`); the manifest/`StrategyIr` cross-crate already re-used. Everything is written so that once nou has a real provider, `research/journal.rs` can hand the whole trail to a server loop without schema change.
- **honba-mon** — the *constraints*, not the entity: `MAX_COMMISSION`, `MAX_SLIPPAGE`, order-rate/participation language in validation copy; risk doctrines are claimed (never silently censored — see the DES-005 note that REST backtest runs currently do not route through `RiskStage`; this doc calls it out as R7, not "fixes it").
- **honba-docs** — this document cites its doctrine and doesn't reinterpret it.

## 6. Phases

Each phase ends with named verification commands; [AGENT HINT] bullets are unchecked-machine-safe guidance. Test style: `#[cfg(test)]` with golden fixtures in `honba-kao/schema/conformance/studio/`.

---

### Phase 0 — Wire contracts shared with DES-005 + studio additions

**Goal:** the studio's REST surface exists and shape-locks zero drift.

1. Extend `honba-kao-client`: `post_sweeps`, `get_sweeps_by_id`, plus `put_profile_layouts_by_name` (server-side library sync, fe's `saveStrategy` path). Re-export `SweepRequest/SweepResponse/SweepReportResponse/StrategiesRequest/StrategyIr/StrategyManifest/ParameterSpec` in `wire`.
2. Golden fixtures in `honba-kao/schema/conformance/studio/`: `manifest_sma_cross.json`, `strategy_ir_echo.json`, `sweep_response.json`, `verify_error_no_subscriptions.json` (422 with `context.reason`). Tests parse both enveloped and bare.
3. Verify the two honest server facts our design leans on, and write them into the module doc as a "verified against source" field: (a) verify errors carry `context.reason = IrError::code()`; (b) `POST /strategies` is idempotent per canonical manifest bytes.

[AGENT HINT] `StrategyManifest` on the wire is `deny_unknown_fields` — since we serialize the crate's own struct, never build manifests as raw `json!` maps. The catalog id (`sha256:<hex>`) is content-addressed: **the same candidate never re-compiles**; use it as the loop cache key from day one.

[AGENT HINT] Library sync fails hard? Honor fe's behavior: server sync is best-effort — on failure, persist locally and set `save_state: offline` rather than breaking Ctrl+S.

**Acceptance:** `cargo test -p honba-kao-client` green; fixtures decode; `cargo check --all-features` clean.

---

### Phase 1 — Studio graph kernel (`studio/graph/*`)

**Goal:** types, validation, catalogue, hashing, manifest, codegen, recognize, diff — pure and golden-tested.

1. Port: types (§4 field-for-field), `graph_issues()` (the **12 exact fe codes**, order preserved: `invalid_schema_version, missing_name, missing_universe, invalid_timeframe, invalid_warmup, duplicate_node_id, unknown_node_type, missing_indicator, missing_edge_id, unknown_from_node, unknown_to_node, port_mismatch, cycle_detected`), `graph_hash_sync` (FNV-1a×8 formatted `sha256:<64hex>` — same digest as fe so handoffs compare equal), `to_manifest`, `generate_code` + `source_lines`, `recognize(source)` (the mini Python-literal parser), `graph_diff`.
2. **Add the 4 missing fe checks** (fe only ran 12; §7 audit found放眼 the gaps): `dangling_node` (nodes unreached from `universe.bars`), `no_order` (graph emits no order), `param_out_of_range` (catalogue min/max), `tf_aggregation_coarse` (driving aggregation coarser than weekly?). Keep codes snake_case server-compatible so the verify roundtrip diffs cleanly.
3. Catalogue: 9 whitelisted indicators + `transform` family ops + port map table.

[AGENT HINT] `graph_issues` order is semantics (thin UIs iterate in order); port tests must preserve it. `recognize` must be **lossless** on `code_only` (never reconstruct unmarked spans silently — DES-004 S3 invariant).

**Acceptance:** `cargo test -p honba-kao-app studio::graph` green, fe's vitest case names reproduced in Rust test names.

---

### Phase 2 — Node editor canvas

**Goal:** the authoring surface (this phase is the biggest UI chunk; fe's is 929+ lines of canvas + nodes).

1. `canvas/editor.rs`: SVG viewport with pan (drag), wheel-zoom, click-select, drag-move (16px snap), `Escape`/Del handling guarded against `INPUT/TEXTAREA`.
2. `canvas/node_card.rs` (one generic component): 200–300px card, category top-border color (from `--c-*` tokens, stored per-category in CSS vars), issue badge, per-kind body chips, ports as named rows with typed color dots.
3. `canvas/edge.rs`: smoothstep path; **insertable edge**: midpoint “+” button → filter-by-port-kind menu (`series`→ indicators, `bool`→ gates/orders); splice = replace edge with `a → new → b` positioned midway.
4. `canvas/palette.rs`, `canvas/inspector.rs`: floats; palette drag with HTML5 MIME `application/honba-node` (port fe's);\_inspector shows param inputs from catalogue ranges.
5. Store wiring: every editor gesture calls a store action; canvas-only extras (`positions`) stay out of `to_manifest`.

[AGENT HINT] Leptos SVG: keep one `Signal<Vec<RenderNode>>` `Memo` over positions; do NOT re-render each port as its own effect (a 12-node graph must be one MEMO render pass). Pan/zoom is a plain SVG transform — do not hand-roll DPR handling; keep `viewBox` in user units. Multi-select (rubber band) is **out of scope** — fe never implemented it either; state so in copy.

[AGENT HINT] Keyboard rule from fe: Delete/Backspace must **not** intercept editable inputs (INPUT/TEXTAREA guard); `Ctrl/Cmd+S` = snapshot+save, Escape clears selection first, then closes whatever else.

**Acceptance:** builds/ports the 6 template graphs (DES-004 §1.7’s exact `STRATEGY_TEMPLATES`) and renders + splices on every edge without orphan handles; `cargo test` + a vitest-parity golden: template → JSON → template.

---

### Phase 3 — Layout, library, snapshots, persistence

**Goal:** everything survives a reload; the studio's memory is real and local.

1. `StudioStore`: granular signals; layout persistence `honba.studio.layout` (fe's `DesignerLayout` fields exactly: `libraryOpen, assistantOpen, libraryWidth 200..600, assistantWidth 280..600, codeRatio 0.15..0.85, drawerOpen, drawerTab, focus`); focus mode.
2. `library.rs`: `SavedStrategy` (fe `SavedStrategyManifest` shape), `upsert/fork(slugify + unique_id)`, library drawer with search/category/modes `all|global|user`; Ctrl+S semantics; server sync best-effort (P0 method).
3. `snapshot.rs`: SHA-256 versions, dedupe-on-snapshot, restore reparse; UI: actions menu (Import/Export JSON/versions) **with the gate**: import validates first with `graph_issues` (no importing garbage).
4. SSE/poll code reuse: the studio page mounts `SimulatorStore`'s own polling client? **No** — the studio is a separate feature but the *run* code path should stay one codebase. **Decision:** the studio's Run/Sweeps/Results panels are the **same components DES-005 P5 build** (metrics ribbon, journal table, equity chart with drawdown-trust badges, validation badges, run log), mounted with studio-store context instead. This keeps two features honest and halves the surface.

[AGENT HINT] One feature = one store; don't cross-import `SimulatorStore` from `StudioStore` — share the *panels* via props, not the state.

[AGENT HINT] `localStorage` keys never iterate into `sessionStorage` semantics; the versions snapshot keeps the last 10 with unique hashes only.

**Acceptance:** reload restores layout, library, open drawers, and selection; save/fork/handoff all work offline (server 404 tolerated).

---

### Phase 4 — Code pane + verify + handoff plumbing

**Goal:** code round-trip, server-verified truth, and the simulator handoff.

1. `studio/codegen.rs` in-canvas integration: the code pane (read-only v1 — one file, line-numbered, `spans` markers shown as gutter hints); visible all mutations.
2. Code↔graph: `recognize + createSyncScheduler` port — fe semantics: `idle → pending (300ms debounce) → synced | invalid`; `code_only` flag and `spans`.
3. `panels/verify.rs`: `verify_graph()` = local issues first, then `POST /strategies/verify` → `StrategyIr` echo in state; per-issue jump-to-node from the drawer; verified vs invalid pills; **Simulate button disabled+red-count until `issues.length == 0`** (fe behavior).
4. Editor → simulator: the "Run in Simulator" button stashes `SimulatorHandoff { graph, name, source: 'studio' }` in `sessionStorage honba.simulator.handoff` and routes `/simulator?strategy=<name>&source=studio` — DES-005 consumes it natively iff we also extend its origin enum with `'studio'` (a two-word change to DES-005 P8; noted there as a live follow-up).
5. **Decision — editor tech (CSR honest):** fe uses Monaco on python. On honba-kao, Monaco is a net-new 2.5MB wasm/JS dependency that fights CSR budgets. V1 = custom minimal code pane (textarea + line gutters + highlight *via a tiny regex highlighter in web_sys, ~150 lines*); Monaco/CodeMirror is a P9 option behind a feature flag (`honba.studio.rich_editor`) via the existing vendor-JS bridge pattern (same seam as the chart). The studio remains 100% usable without it.

[AGENT HINT] `generate_code`'s Python must be byte-stable given (graph, hash) — pin a golden test with all 6 templates so a graph change never silently renames emitted params (dedupe via the `${nodeId}_${param}` rule).

[AGENT HINT] The 422 verify errors are *strings meant for agents* — surface them 1:1 in the drawer (`reason: no_subscriptions`); never paraphrase them away.

**Acceptance:** round-trip template graph ⇄ generated code ⇄ recognize → identical graph hash; verify panel shows server IR echo on the happy path and the honest 422 reason otherwise.

---

### Phase 5 — Run + sweeps panels (canonical evaluation)

**Goal:** the studio can drive honba-ya honestly: one run, one sweep, truth only.

1. `panels/run.rs`: window/seed/costs controls (goal sliders default `seed 42`), run button gated as P4; visible `RunState`; inline `Blocked` reasons from `RunErrorPill` wiring like DES-005 store (`foldStatus` etc.); SSE events drawer from DES-005 `run-log` shared component.
2. `panels/sweeps.rs`: sweeps are only offered when `GET /strategies` shows the compiled id is registry-carried; otherwise the panel shows the honest "no_rust_implementation" explanation box. Sweep params come from `manifest.parameters` (declared sweep space). Fixed `seed + i` per trial (matches server behavior); grid bar-length cap 50 client-side to make accidental mega-grids honest.
3. Results flow through the shared results components (DES-005 P5); journal download, CSV export; every panel credits its source (`server (journalled)` / `provisional (client-side tier-1) — evaluation not canonical`).

[AGENT HINT] Today sweeps' `ranked` param rows are **indicative** (server-side hard-coded mapping by strategy name); render them as read-only table content; "Apply best" maps back via the fe `${nodeId}.param` prefix rules and shows a before/after diff pill before applying.

[AGENT HINT] Never poll sweeps with the backtest poll constants: sweeps are a single worker's blocking job — the honest UX is an indeterminate "In sweep (occupies one executor slot)…" with a `GET /sweeps/{id}` poll and no fake progress bar.

**Acceptance:** with a live server: run `sma_crossover` end-to-end (seed, journal shown, trust badges render, honesty chips correct); sweep on it returns a `ranked` table; sweep on a non-registry graph **states** the limitation rather than failing silently.

---

### Phase 6 — AI co-pilot (chat column)

**Goal:** the propose/apply chat, with real context; no mock-x theatre.

1. `copilot/context.rs` — port fe's `assistantContext()` brief: graph structure list, edges as `a.port → b.port`, code (truncated 80 lines), last run summary, last 6 messages (200-char clipped), goal sliders.
2. `copilot/assistant.rs` — message kinds `discussion|candidate|validation_error`; `POST /assistant/propose` does not exist server-side — fe ships a **local heuristic fallback** (keyword-keyed candidate graphs). Port it exactly (it is the offline engine and the test seam), but brand it `mode: heuristic` in the UI (fe's fake “AI” quotes are honesty pills); real providers arrive in P7 and the co-pilot auto-upgrades.
3. Candidate cards from `Hypothesis.graph`: added/modified/removed + param changes (fe `GraphDiffSummary`), the ADR-0029-S5 quarantine chip, `Apply` = snapshot → `ValidationGate{ set graph, valid }` check → set graph + reroute to canvas; `Discard` = drop.
4. Buttons wired: PROMPT_CHIPS equivalent (the six canned directives from fe's AiQuantStudio, as text and shipped via the same input path).

[AGENT HINT] Candidate-on-canvas must never overwrite an unsaved user graph silently — the pattern is `snapshot_version() → apply → rename` (fe does exactly this; port it verbatim).

[AGENT HINT] Chat column must not `create_effect` read its own last message (infinite-loop shape); read the store's *actioned* state.

**Acceptance:** chat round-trips offline (heuristic mode): a candidate renders a stub card, `Apply` snapshots + merges, `Discard` leaves history, ADR-0029-S5 chip is always present until verify succeeds.

---

### Phase 7 — Research loop engine (LLM providers, journal, controller)

**Goal:** the heart of this doc. The loop state machine, provider dialing, structured hypothesis parses, the IndexedDB JSONL journal with similarity retrieval, and the honest LLM audit trail.

1. `research/types.rs` — all structs of §3 (`ResearchGoal`, `Iteration`, `LoopState` with fe's honest status/phase enums, `FailureMode` closed enum), `deny_unknown_fields` on all LLM-input shapes.
2. `research/providers.rs` — §3.6 contracts; Ollama-default recommendation in UI copy; every provider call surfaces tokens in/out and a per-loop cumulative cost meter (token-truth from the response; if a provider's cost is NULL, cost renders `n/a` honestly).
3. `research/propose.rs` — prompt assembly (`journal.unsimilar`, failure tally, parent chain, catalogue, constraints, explicit "do not optimize numerics; declare sweepable ranges"), response parsed into `Hypothesis` with **one retry with the parse error appended**, then a skip (record + count).
4. `research/journal.rs` — IndexedDB (`honba.studio.research`), append `ResearchRecord` (the SHA-256 `LlmRecord` fields honored), export journal JSONL + research_log.md text export; similarity index (bounded, deterministic) for the prompt.
5. `research/engine.rs` — the state machine: serial loop (3.2) with **operator controls** (pause, kill-switch = halts after current iteration; never mid-, respecting the doctrine's no-proposal-without-measurement discipline), budget meters (tokens/wall-clock/iterations), `goal_met` only on validation-tier truth (5.4/5.5), provider failures backoff ×3, 3 consecutive provider errors ⇒ `paused` with an explanation (loop never spins a provider bill).
6. Telemetry tab (fe's, honest this time): per-iteration table `(k, hypothesis id, statement, tier1 metrics, tier2 run_id, gates, failure_mode, tokens)`; `bestIterationIndex` selector; goal meter bars; **Monte-Carlo/walk-forward sections only render what the block contains** (fe's fakes are gone).

[AGENT HINT] All sealing happens in `spawn_local_scoped` steps; never `create_effect` the loop's own output signal (Leptos infinite-update trap; DES-005 P4 hint applies).

[AGENT HINT] The loop must survive reload: persist loop state (current iteration, phase, journal offsets, budget spent) to local storage keyed `honba.studio.research` on every phase transition, and offer "Resume" on remount. A reload mid-`backtesting` just re-polls the server run id (journals last).

[AGENT HINT] Never call `POST /backtests` for a candidate that fails `graph_issues` — that's not "cheap"; it wastes a server slot and might queue up to 64 more; keep gating a pure function.

**Acceptance:** manual-provider mode + fixture graphs run the whole loop end-to-end offline with deterministic sorted failures; a mocked local provider counts tokens per iteration; `goal_met` interpretation is a pure function over a testable `Iteration` chain (no DOM). Loop kills/pauses/resumes across reload.

---

### Phase 8 — Tier-1 evaluator + parity + runtime envelope

**Goal:** the provisional screening path is real, honest, and parities where it can.

1. `research/eval/tier1.rs` — the §5.3 interpreter inside `honba-kao-api-wasm` (bars fetch is `honba-kao-app`'s job; interpreter is pure): indicators → graph → next-open-fill trades → the 8 metrics; every return value echoes `source: provisional`.
2. `parity.rs` — for `sma_crossover` (registry-carried), drive **both** tier-1 and a real server backtest on the same `(seed, dataset)` and compare (equity/approx-on-metrics tolerance, documented over a fixed window); the parity report is *a fixture test* in ` honba-kao/schema/conformance/studio/t1_t2_parity.json`. The purpose: catch tier-1 drift when either side changes (e.g., DES-005 P0 lands server fixes); the harness runs by hand against a server, not in CI that must pass offline.
3. `research/eval/envelope.rs` — the runtime-risk **preview** (fe's `RuntimeRiskEnvelope` mock, minus the theatre): honest constants (`max_risk_per_trade_pct`, kill-switch drawdown), computed from honba-mon's published limits as **preview copy only** — a "would refuse" chip (server truth, listed in §8 R7).

[AGENT HINT] Tier-1 cache: candidates hash-id via `graph_hash_sync` — a loop that regenerates the same candidate skips re-eval (the wrong kind of fast), but **never** lets that change tier-2 seeds or journal ids (server cache invalidation is honest: same content = same result).

[AGENT HINT] Bars for tier-1 are the same fetch + provenance rules as DES-005 P2 (`server | preview` labeled chip) — share `sim/chart_data.rs` via `pub` re-export (DES-005 was designed as shareable, not feature-internal).

**Acceptance:** 1000-iteration synthetic journal (fixture) screens ≤ a fixed wasm-time budget on dev hardware; the parity fixture documents matched/mismatched metrics for `sma_crossover` explicitly.

---

### Phase 9 — Doctrine, doc, polish

**Goal:** everything above is written down honestly.

1. Workbook docs: `honba-docs` book chapter extract (the research loop user guide), including the "what it will not do" §3.3 verbatim copy.
2. `honba-kao/plan.md` §4.2 update: link this doc; add Phase checkboxes mirroring P0–P9.
3. Perf budget test (mirroring DES-005 P9): node editing with 50 nodes ≤ 16ms input→paint (per gesture), tier-1 on 1-year 5-minute bars ≤ 50ms, journal TF-IDF retrieval ≤ 5ms for 1000 records.
4. `A11y` pass on the canvas (keyboard navigation to node/edged via tab order is **in scope** since fe never did it — call it out as a new addition); color contrast against all 8 honba-kao themes.
5. The three **flagged server asks** each get one paragraph of doc content for the future honba-ya worker: (a) `validation: Some(...)` compute-and-attach in `to_backtest_response`/sweep response; (b) a manifest/graph-interpreter strategy so `no_rust_implementation` stops existing; (c) a sweep sampler beyond grid (`random | sobol`). Each documented as: current behavior (verified, with file+function), why the studio works around it, what changes on the surface if honored.

[AGENT HINT] The three server asks are **out of scope** for this repo's implementation phases but are the doc's diplomatic hooks (R2/R3/R6 in §8). Do not change honba-ya code in this workstream.

**Acceptance:** docs pass lint; `cargo clippy --all-targets -- -D warnings` clean; `make build` prod profile; the "Definition of Done" of DES-005 §7 applies verbatim (provenance, honest empty states, no hex colors, module docs name their port source).

## 7. Honest-current-state ledger (what the server actually does — verified inline)

| Facility | Current reality (verified) | Studio behavior |
|---|---|---|
| LLM | `honba-nou-llm` stub (transport unimplemented) | Providers client-side (BYO key / Ollama); nou's audit shapes honored |
| Execute arbitrary graph | 422 `no_rust_implementation` unless a builtin name matches; manifest `parameters` ignored by the registry | Tier-1 provisional path for everything else; tier-2 only for registry-carried |
| `ValidationBlock` | Computed in the crate, **attached `None`** on all response paths | Client-side compute from journal return series (pure gates) |
| Sweeps | Grid-only, `trials == grid.len()`, per-name param mapping; `custom` factory stub | Sweeps only for registry-carried names; visible limitation copy |
| Sweeps rank source | `ranked` rows indicative, param mapping hard-coded | Render as read-only indication; Apply is diffed first |
| `/reports/equity-curve` | Portfolio-derived + pseudo-random price walker | **Never used in the loop**; journal-equity only |
| Backtest run queue | 64 cap, 429 `run_queue_full`, `available_parallelism()` workers; one sweep blocks a worker | Backoff, serial loop, honest concurrency copy |
| `/reports` caching | `CacheKey` sha256(strategy, params, dataset, cost model); L2 never cached | Candidate ids are `(content-id + seed_i)` so re-sampling identical work is free |
| REST risk stage | `with_risk()` not wired to the REST backtest executor today | Stated as R7, never claimed as solved |

## 8. Risk register

| # | Risk | Mitigation |
|---|---|---|
| R1 | Tier-1 interpreter “violates” ADR 0029 D16 letter | §5.3's provenance model (provisional chip, ever-green, cannot trigger goal_met); §8 flagged as a doctrine delta for the ADR owner; the alternative (strict D16) makes the user's auto-researcher requirement impossible — the design chooses honesty over fiction. |
| R2 | Server can't run structural candidates (`no_rust_implementation`) | Registry-first loop; blocked-promotion honesty; P9's server ask (a). |
| R3 | `ValidationBlock` server-unattached | Client-side pure gates wired from nou-analytics (wasm-safe dep); P9's server ask (b). |
| R4 | Browser held the LLM API key → leak | Ollama default; masked storage; never mirrored to server; a future "local researcher sidecar" is the documented upgrade path if a key never belongs in a browser. |
| R5 | 1000 iterations × LLM cost/burn | Budget meters, stop conditions, cumulative cost meter, journal-dump on 3-failure pause; offline heuristic provider for zero-cost dev runs. |
| R6 | Sweeps grids burst (big cartesian) | Cap 50 client-side; `trials` must be declared; registry-first honest copy; P9 asks for (c) samplers. |
| R7 | REST backtests don't route honba-mon RiskStage today | Stated in copy; the loop never claims executed-refused order metrics as server truth (`refused_orders` from the ya-backtest engine only). |
| R8 | Reload mid-loop loses state | Persist-on-phase-transition + resume path (P7 #5), server run ids survive restarts (server-side `run_id` persists on journals). |
| R9 | Node canvas perf in wasm | One-Memo render pass, no per-node effects; P9 budget gate; polish budget claims that the DES-004 canvas state was fine after ~50 nodes. |
| R10 | honba-ya restart loses the compiled-candidate catalog → history breaks | Content-addressed `sha256:` ids mean re-compose refreshes; the loop re-compiles before tier-2 (`POST /strategies` id for `goal_met` continuity), never trusting in-process names. |

## 9. Definition of Done

1. Every module doc names its fe/docs reference and any semantic deviation (honba-kao convention).
2. `cargo test --workspace` + `cargo clippy --all-targets -- -D warnings` green; `make build` prod profile works.
3. All eight hongkao themes crisp; zero literal hexes in studio CSS (`var(--*)` only).
4. Provenance vocabulary enforced everywhere: nothing renders `server`-looking numbers unless they came from honba-ya, or show a `provisional` chip if they came from Tier-1.
5. Propose-never-execute holds end-to-end: no unverified candidate can pipeline into a run; the draft ADR-0029-S5 chip appears on every non-verified proposal path.
6. The loop honors *every* research_loop.md "does NOT do" rule, executable assertions where possible (fail-loud loops over numeric prompting).
7. Honba's statistical-validation thresholds (§5.5) are the loop's default goal gates; a user may lower them, with a one-line warning quote in the UI.
