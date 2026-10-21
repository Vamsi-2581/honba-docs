# Migration Plan: Consolidating Pre-Trade Risk from `honba-kokoro-risk` to `honba-mon`

> **Document Status**: Active Migration Blueprint  
> **Author**: Google DeepMind Agentic Assistant (Antigravity)  
> **Target Subsystems**: `honba-mon`, `honba-ya`, `honba-tamashii`, `honba-nou`, `honba-me`, `honba-te`, `honba-kokoro`  
> **Target Architecture**: Domain-Driven Design (DDD), Python-First Strategy SDK, Exact Decimal Precision (`rust_decimal`), Zero Floats, Temporal Determinism.

---

## 1. Context & Architectural Root Cause Analysis

### 1.1 The Domain-Driven Design (DDD) Bounded Context Realignment
Under the foundational architecture redesign of the **Honba (本場)** platform, subsystem responsibilities are strictly demarcated across clear domain boundaries:

* **Kernel (`honba-core`)**: Shared Kernel value objects (`Price`, `Quantity`, `Money`, `InstrumentId`), wire contracts, and safe closed AI grammar (`Decision`). Zero dependencies, zero I/O.
* **Heart (`honba-kokoro`)**: Institutional Order Management System (`oms`), FIFO tax-lot tracking (`portfolio`), and immutable double-entry general ledger.
* **Gate (`honba-mon`)**: Pure, synchronous pre-trade regulatory risk engine. Enforces SEBI Retail Algo circular rules, Order-to-Trade Ratio (OTR) sliding-window circuit breakers, scrip-wise circuit price bands, single-order freeze limits, and global kill-switches.
* **House (`honba-ya`)**: Multi-threaded Tokio actor runtime, institutional backtester, and strategy optimizer.
* **Soul (`honba-tamashii`)**: PyO3 FFI bridge, Python-First Strategy SDK, and Arrow zero-copy memory transport.
* **Brain (`honba-nou`)**: Autonomous AI research loop, MCP tools, and Odysseus genetic rule synthesis.

### 1.2 Root Cause of the Broken Dependencies
Historically, pre-trade risk checks were implemented in `honba-kokoro/crates/honba-kokoro-risk` alongside legacy crates (`honba-kokoro-types`, `honba-kokoro-messages`, `honba-kokoro-entities`, `honba-kokoro-ports`). These legacy crates suffered from several severe architectural flaws:
1. **Raw Floating-Point Leaks**: Used `f64` for order quantities, positions, prices, and collars, directly violating the **Zero Floats Policy**.
2. **Scattered Risk Responsibility**: Split risk checks between Kokoro (`RiskStage`) and Mon (`RiskEngine`), creating duplicate and contradictory validation logic.
3. **Premature Deletion in Kokoro**: In commit `245784e`, `honba-kokoro` unified its internal OMS and portfolio engines on `honba-core` and excised all legacy crates (`crates/honba-kokoro-risk`, `types`, `messages`, `entities`, `ports`).

Because downstream crates in `honba-ya`, `honba-tamashii`, `honba-nou`, `honba-me`, and `honba-te` still declare path dependencies on the excised crates (e.g. `path = "../honba-kokoro/crates/honba-kokoro-risk"`), Cargo fails to resolve their manifests (`No such file or directory (os error 2)`).

This document establishes the **exhaustive, step-by-step migration plan** to purge all obsolete dependencies on `honba-kokoro-risk` and integrate `honba-mon` as the single authoritative pre-trade risk gate across the platform.

---

## 2. Dependency & Code Usage Inventory

The following table details every broken manifest reference and code usage that must be migrated:

| Repository / Subsystem | File Location | Broken Dependency / Symbol | Action Required |
| :--- | :--- | :--- | :--- |
| **`honba-me`** | `honba-me/Cargo.toml` | `honba-kokoro-types` | Replace with `honba-core = { path = "../honba-core" }`. |
| **`honba-te`** | `honba-te/Cargo.toml` | `honba-kokoro-types` | Replace with `honba-core = { path = "../honba-core" }`. |
| **`honba-mon`** | `src/engine.rs:274-285` | `Decision::Buy`, `Decision::Sell` | Update pattern match to `Decision::PlaceOrder { symbol, side, qty, price_limit, algo_id, .. }`. |
| **`honba-mon`** | `src/engine.rs:310` | `order.validate()` | Use `order.order_type.validate()` or implement structural validation in `honba-core::OrderRequest`. |
| **`honba-ya`** | `honba-ya/Cargo.toml:84-92` | `honba-kokoro-risk`, `honba-risk`, `honba-ports`, `honba-messages`, `honba-entities` | Prune legacy paths. Add `honba-mon = { path = "../honba-mon" }` and `honba-core = { path = "../honba-core" }`. |
| **`honba-ya`** | `crates/honba-ya-config/src/lib.rs` | `honba_risk::{OrderRateLimit, RiskLimits}` | Replace with `honba_mon::{OrderRateLimit, RiskLimits}`. |
| **`honba-ya`** | `crates/honba-ya-cli/src/serve.rs` | `honba_risk::RiskLimits` | Migrate to `honba_mon::RiskLimits`. |
| **`honba-ya`** | `crates/honba-ya-actors/src/risk.rs` | `RiskActor` | Connect `RiskActor` synchronously to `honba_mon::RiskEngine`. |
| **`honba-ya`** | `crates/honba-ya-engine/` | `honba_risk::{RiskStage, RiskCheck, ...}` | Deprecate `honba-ya-engine` in favor of `honba-ya-actors` per Master Plan §0.1. |
| **`honba-ya`** | `crates/honba-ya-api-rest/src/orders.rs` | `honba_risk::{RiskStage, RiskRefusal}` | Migrate pre-trade order refusal to `honba_mon::RiskRefusal`. |
| **`honba-tamashii`**| `honba-tamashii/Cargo.toml:52-59` | `honba-kokoro-risk`, `honba-ports`, `honba-messages` | Prune legacy paths. Add `honba-mon = { path = "../honba-mon" }` and `honba-core`. |
| **`honba-tamashii`**| `crates/honba-tamashii-py/src/pyclasses/risk.rs` | `honba_risk::{OrderRateLimit, RiskDecision}` | Wrap `honba_mon::RiskEngine`, `RiskLimits`, `RiskRefusal` via PyO3. |
| **`honba-tamashii`**| `python/honba/risk.py` | Legacy `_POSITION_EPS = 1e-9` and float checks | Replace with decimal validation invoking native `honba._honba.RiskEngine`. |
| **`honba-nou`** | `honba-nou/Cargo.toml:55-61` | `honba-kokoro-risk`, `honba-messages`, `honba-entities` | Prune legacy paths. Add `honba-mon = { path = "../honba-mon" }` and `honba-core`. |
| **`honba-nou`** | `crates/honba-nou-strategy/src/runner.rs` | `honba_risk::{RiskCheck, RiskStage}` | Evaluate `honba_core::Decision` against `honba_mon::RiskEngine`. |
| **`honba-nou`** | `crates/honba-nou-trader-loop/src/lib.rs` | `honba_risk::RiskLimits` | Replace with `honba_mon::RiskLimits`. |

---

## 3. Domain Model & API Transformation Matrix

### 3.1 `RiskLimits`: Floats to Exact Decimals & SEBI Compliance

| Old `honba-kokoro-risk::RiskLimits` | New `honba-mon::RiskLimits` | Rationale & Behavioral Difference |
| :--- | :--- | :--- |
| `max_notional_per_order: Option<f64>` | `max_notional_per_order: Option<Decimal>` | Zero Floats Policy. Exact paisa-level precision without float drift. |
| `max_notional_per_day: Option<f64>` | `max_daily_notional: Option<Decimal>` | Aggregated cumulative daily turnover threshold. |
| `max_quantity_per_order: Option<f64>` | `single_order_freeze_limit: Option<Quantity>` | Strongly typed `Quantity`. Enforces exchange freeze limits (e.g. NIFTY 1,800, BANKNIFTY 900). |
| `price_collar_fraction: Option<f64>` | `scrip_circuit_limit_pct: Option<Decimal>` | Scrip operational band (2%, 5%, 10%, 20%). Replaces arbitrary float collars with exchange bands. |
| `order_rate_limit: Option<OrderRateLimit>` | `order_rate_limit: Option<OrderRateLimit>` | Max order events within a sliding window (e.g. 50 orders / 1,000 ms). |
| *(Missing in legacy)* | `max_otr: Option<Decimal>` | **SEBI Retail Algo Mandatory**: Order-to-Trade Ratio calculation ($OTR = \frac{Total Orders}{Executed Trades}$). Breaker halts algo if $OTR > 50$. |
| *(Missing in legacy)* | `require_sebi_algo_id: bool` | Strict rejection of orders lacking registered algo identifier. |

### 3.2 `RiskRefusal`: Strongly Typed, Serializable Taxonomy

The legacy `RiskRefusal` enum used raw floats and lacked JSON Schema serialization. In `honba-mon`, `RiskRefusal` is a comprehensive, typed, serializable enumeration deriving `schemars::JsonSchema`:

```rust
// honba-mon/src/refusal.rs
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, JsonSchema)]
#[serde(tag = "reason", content = "details", rename_all = "snake_case")]
pub enum RiskRefusal {
    GlobalHalted { reason: String },
    AlgoHalted { algo_id: String, reason: String },
    MarketCircuitHalt { index_symbol: String, stage_pct: u8, resume_at: Option<DateTime<Utc>> },
    OtrBreached { algo_id: String, current_otr: Decimal, max_otr: Decimal },
    PriceBandViolation { price: Price, band: PriceBand },
    MaxNotionalExceeded { requested: Decimal, limit: Decimal },
    SingleOrderFreezeExceeded { requested: Quantity, limit: Quantity },
    OrderRateLimitExceeded { count: u32, window_ms: u64 },
    MissingAlgoId,
    InvalidOrder { error: String },
}
```

### 3.3 Engine Invocation & Interface Shift
The legacy design required constructing a synthetic `RiskRequest` object:
```rust
// LEGACY (DEPRECATED):
let request = RiskRequest::new(&order, &market_state);
let decision = risk_stage.check(&request);
```

The new interface operates directly on canonical domain contracts with zero intermediate allocation:
```rust
// MODERN (honba-mon):
let mut risk_engine = RiskEngine::new(risk_limits, otr_config);

// 1. Direct validation of outgoing broker orders:
risk_engine.check_order(&order_request, Utc::now())?;

// 2. Direct validation of untrusted AI/LLM decisions:
risk_engine.check_decision(&decision, Utc::now())?;
```

---

## 4. Subsystem-by-Subsystem Migration Plan

```mermaid
flowchart TD
    subgraph Phase 1: Leaf Crate Manifest Unification
        M1["honba-me: Point to honba-core"]
        M2["honba-te: Point to honba-core"]
    end

    subgraph Phase 2: honba-mon Engine Polish
        MON1["Fix Decision::PlaceOrder matching in RiskEngine"]
        MON2["Ensure 100% tests pass in honba-mon/tests/"]
    end

    subgraph Phase 3: honba-ya Actor & Runtime Migration
        YA1["Purge honba-kokoro-* from honba-ya/Cargo.toml"]
        YA2["Wire honba-ya-actors::RiskActor to honba-mon::RiskEngine"]
        YA3["Update honba-ya-config & honba-ya-api-rest to honba_mon types"]
        YA4["Retire legacy honba-ya-engine"]
    end

    subgraph Phase 4: honba-tamashii Python SDK & PyO3
        TAM1["Purge legacy kokoro crates in Cargo.toml"]
        TAM2["Wrap honba-mon::RiskEngine in PyO3 (honba._honba)"]
        TAM3["Refactor python/honba/risk.py to native FFI"]
    end

    subgraph Phase 5: honba-nou AI Brain & Runner
        NOU1["Purge legacy kokoro crates in Cargo.toml"]
        NOU2["Connect StrategyRunner to honba-mon via Decision grammar"]
    end

    subgraph Phase 6: System Verification
        V1["cargo check --all across entire workspace"]
        V2["Run parallel test suites (tests/*)"]
        V3["Commit all repositories on branch main"]
    end

    Phase 1 --> Phase 2
    Phase 2 --> Phase 3
    Phase 3 --> Phase 4
    Phase 4 --> Phase 5
    Phase 5 --> Phase 6
```

### 4.1 Phase 1: Leaf Crate Manifest Unification (`honba-me` & `honba-te`)
* **Goal**: Unblock Cargo manifest parsing in market data and execution subsystems.
* **Tasks**:
  1. In `honba-me/Cargo.toml`, remove `honba-kokoro-types` and replace with `honba-core = { path = "../honba-core" }`.
  2. In `honba-te/Cargo.toml`, remove `honba-kokoro-types` and replace with `honba-core = { path = "../honba-core" }`.
  3. Verify `cargo check --manifest-path honba-me/Cargo.toml` and `cargo check --manifest-path honba-te/Cargo.toml` pass cleanly.

### 4.2 Phase 2: `honba-mon` Engine Polish
* **Goal**: Ensure `honba-mon` compiles cleanly and passes all unit and integration tests.
* **Tasks**:
  1. In `honba-mon/src/engine.rs`, update `check_decision()` to match `Decision::PlaceOrder { symbol, side, qty, price_limit, algo_id, .. }`.
  2. Implement structural order validation in `honba-core::OrderRequest` or replace `order.validate()` with `order.order_type.validate()` in `honba-mon/src/engine.rs`.
  3. Execute `cargo test --manifest-path honba-mon/Cargo.toml` to guarantee 100% test pass rate in external `tests/` directory.

### 4.3 Phase 3: `honba-ya` Actor & Runtime Migration
* **Goal**: Modernize `honba-ya` to utilize `honba-mon` for all pre-trade risk and retire legacy `honba-ya-engine`.
* **Tasks**:
  1. In `honba-ya/Cargo.toml`:
     - Delete `honba-kokoro-types`, `honba-kokoro-messages`, `honba-kokoro-entities`, `honba-kokoro-ports`, `honba-kokoro-risk`.
     - Add `honba-mon = { path = "../honba-mon" }`.
     - Point `honba-core` directly to `../honba-core`.
  2. In `honba-ya/crates/honba-ya-config/Cargo.toml`:
     - Replace `honba-risk` dependency with `honba-mon`.
     - In `src/lib.rs`, re-export `pub use honba_mon::{OrderRateLimit, RiskLimits};`.
  3. In `honba-ya/crates/honba-ya-actors`:
     - Verify `RiskActor` holds `honba_mon::RiskEngine`.
     - Ensure synchronous evaluation of order requests before passing to `ExecutionActor`.
  4. In `honba-ya/crates/honba-ya-api-rest`:
     - Update order placement endpoint in `src/orders.rs` to validate with `AppState.risk_engine` (`honba-mon`).
     - Map `honba_mon::RiskRefusal` to HTTP 422 Unprocessable Entity with structured JSON refusal payload.
  5. In `honba-ya/crates/honba-ya-cli`:
     - Update `serve.rs` to parse `RiskLimits` from `honba-mon`.
  6. Retire or remove `honba-ya-engine` and `honba-ya-sim` references in favor of `honba-ya-actors` and `honba-ya-backtest`.

### 4.4 Phase 4: `honba-tamashii` Python SDK & PyO3 Native Bridge
* **Goal**: Expose `honba-mon` to Python quantitative research and replace float-based checks.
* **Tasks**:
  1. In `honba-tamashii/Cargo.toml`:
     - Remove dead Kokoro crate references.
     - Add `honba-mon = { path = "../honba-mon" }` and `honba-core = { path = "../honba-core" }`.
  2. In `crates/honba-tamashii-py/src/pyclasses/risk.rs`:
     - Implement PyO3 wrappers `PyRiskEngine`, `PyRiskLimits`, and `PyRiskRefusal`.
     - Expose `.check_order(order: PyOrderRequest)` and `.check_decision(decision: PyDecision)`.
     - Release GIL (`py.allow_threads`) during risk evaluation.
  3. In `python/honba/risk.py`:
     - Prune legacy float epsilon checks (`_POSITION_EPS`).
     - Route all risk checks to `honba._honba.RiskEngine`.

### 4.5 Phase 5: `honba-nou` AI Brain & Strategy Runner Migration
* **Goal**: Connect AI strategy loops directly to `honba-mon`.
* **Tasks**:
  1. In `honba-nou/Cargo.toml`:
     - Remove `honba-kokoro-risk`, `honba-messages`, `honba-entities`.
     - Add `honba-mon = { path = "../honba-mon" }` and `honba-core = { path = "../honba-core" }`.
  2. In `crates/honba-nou-strategy/src/runner.rs`:
     - Update `StrategyRunner` to validate generated `Decision`s against `honba_mon::RiskEngine`.
     - Persist SHA-256 prompt hashes and pre-trade refusal diagnostics in `honba-ya-persistence` audit logs.
  3. In `crates/honba-nou-trader-loop/src/lib.rs`:
     - Update `RiskLimits` import to `honba_mon::RiskLimits`.

### 4.6 Phase 6: System-Wide Verification & CI Integration
* **Goal**: Validate zero compilation errors, full test suite execution, and documentation parity.
* **Tasks**:
  1. Run `cargo check` across all Rust workspaces:
     ```bash
     for repo in honba-core honba-mon honba-kokoro honba-me honba-te honba-ya honba-nou honba-tamashii; do
       echo "Checking $repo..."
       cargo check --manifest-path "$repo/Cargo.toml" --all-targets
     done
     ```
  2. Run all unit and integration test suites in `tests/`:
     ```bash
     cargo test --all
     ```
  3. Run Python linting and type checks:
     ```bash
     ruff check . && ruff format --check . && pyright
     ```
  4. Build and verify documentation:
     ```bash
     cd honba-docs && ../.venv/bin/mkdocs build
     ```

---

## 5. Architectural Invariants & Agent Guidelines

During the execution of this migration plan, all agents and contributors must strictly enforce:
1. **Parallel Test Directory Rule (`AGENTS.md §1.1`)**:
   Under no circumstances should `#[test]` or `#[cfg(test)] mod tests` be introduced into production files (`src/*.rs`). All tests must be placed in `tests/` parallel to `src/`.
2. **Interface-First Coding (`AGENTS.md §2.1`)**:
   Define traits and abstract contracts (`RiskGate`, `OrderValidator`) before implementing concrete broker or simulation adapters.
3. **Zero Floats Policy (`AGENTS.md §2.4`)**:
   All monetary amounts, execution prices, quantities, and statutory fees must utilize `rust_decimal::Decimal` (or `Price` / `Quantity` / `Money`). Floating-point types (`f32`, `f64`, Python `float`) are forbidden.
4. **Deterministic Temporal Purity (`AGENTS.md §2.5`)**:
   `honba-mon` must remain strictly zero-I/O with no clock calls (`SystemTime::now()` is prohibited; state transitions take explicit `DateTime<Utc>`).
