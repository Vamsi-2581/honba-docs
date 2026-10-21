# Honba Architecture Guide

Welcome to the architectural specifications and design documentation for **Honba (本場)** — an institutional-grade, AI-native algorithmic trading platform and quantitative research operating system designed for Indian financial markets (NSE, BSE, NFO, MCX) and global multi-asset execution.

---

## 1. Architectural Philosophy & Core Tenets

Honba combines five foundational design paradigms into a unified, coherent trading system:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      HONBA ARCHITECTURAL PILLARS                       │
 ├───────────────────┬───────────────────┬────────────────────────────────┤
 │   PYTHON FIRST    │    DDD & ZERO-IO  │     MULTITHREADED ACTORS       │
 │ Ergonomic Python  │ Ubiquitous Lang   │ Supervised Tokio actors        │
 │ API like Nautilus │ Domain Aggregates │ Sub-microsecond dispatch       │
 │ & Jesse (PyO3)    │ Decimal purity    │ Lock-free channels (Nautilus)  │
 ├───────────────────┴───────────────────┴────────────────────────────────┤
 │   AI-NATIVE ENGINE                    │ STRATEGY TESTER & ENHANCER     │
 │ Closed decision grammar, MCP tools,   │ Genetic & Bayesian optimizer,  │
 │ deterministic prompt & trade replay   │ walk-forward, Monte Carlo, DSR │
 └───────────────────────────────────────┴────────────────────────────────┘
```

1. **Python First (like NautilusTrader, StockSharp, and Jesse)**:
   - Clean, idiomatic Python 3.12+ strategy interface (`Strategy` base class, reactive decorators, type annotations).
   - High-performance Rust engine beneath: PyO3 zero-overhead bindings, GIL-released background worker threads, and zero-copy PyArrow/Polars data bridges.
2. **Domain-Driven Design (DDD) & Zero-I/O Domain Purity (like StockSharp)**:
   - Clear bounded contexts separating the pure Shared Kernel (`honba-core`) from stateful domain aggregates (`honba-kokoro`), pre-trade risk controls (`honba-mon`), adapters (`honba-me`, `honba-te`), and runtime actors (`honba-ya`).
   - Ubiquitous domain language: `InstrumentId`, `Price`, `Quantity`, `Money`, `Order`, `Position`, `Portfolio`, `DoubleEntryLedger`.
   - Strict 128-bit decimal arithmetic (`rust_decimal::Decimal`) across all monetary calculations. Zero floats for prices, shares, or cash.
3. **Responsive, Multithreaded Concurrency (like NautilusTrader)**:
   - Actor-oriented runtime built on Tokio with dedicated, pinned thread pools and bounded MPSC channels.
   - Independent actors for Market Data ingestion, Strategy signal generation, Synchronous Pre-Trade Risk gating, Order Execution, Portfolio MTM valuation, and Relational Persistence.
   - Non-blocking execution paths, real-time backpressure handling via last-value quote coalescing, and lock-free event distribution.
4. **AI-Native Quantitative OS (`honba-nou`)**:
   - Autonomous quantitative research loop: hypothesis formulation, closed-grammar code generation, automated backtesting, and out-of-sample statistical verification.
   - Closed decision grammar (`honba_core::Decision`): prevents LLM hallucinations, requires mandatory SEBI `algo_id`, and enforces strict JSON schema adherence (`deny_unknown_fields`).
   - Deterministic prompt and decision replayer: complete cryptographic audit trail (SHA-256) enabling byte-for-byte reproducible LLM backtests.
   - Native Model Context Protocol (MCP) server exposing tools to frontier models (Claude, Gemini).
5. **Strategy Tester & Automatic Strategy Enhancer (like Jesse)**:
   - High-throughput, event-driven backtesting engine with microsecond fidelity and research-live parity.
   - Multi-stage optimization engine: Genetic Algorithms (GA) with tournament selection, Bayesian Hyperparameter Optimization (Tree-structured Parzen Estimators / TPE), and grid/random sweeps.
   - Rigorous Overfitting Guardrails: Deflated Sharpe Ratio (DSR), Probability of Backtest Overfitting (PBO), and Anchored/Rolling Walk-Forward Analysis (WFO).
   - Monte Carlo Permutation & Stress Testing: Trade sequence shuffling, slippage/spread shock simulation, and drawdown confidence intervals.
   - AI Strategy Auto-Enhancer: Closed-loop diagnosis where AI models analyze drawdown streaks and generate mutated strategy variants with expanded parameter spaces.

---

## 2. Platform Subsystem Topology

Honba is architected as modular Japanese-named subsystems, each fulfilling a single bounded context:

| Kanji | Subsystem | Language | Role & Responsibility |
| :---: | :--- | :--- | :--- |
| **核** | `honba-core` | Rust | **Shared Kernel**: Zero-IO canonical value objects (`Price`, `Quantity`, `Money`), wire messages, and `Decision` contracts. Zero dependencies. |
| **心** | `honba-kokoro` | Rust | **Domain Heart**: Pure domain aggregates — deterministic Order Management System (`honba-kokoro-oms`), tax-lot FIFO portfolio (`honba-kokoro-portfolio`), and double-entry accounting ledger (`honba-kokoro-ledger`). |
| **門** | `honba-mon` | Rust | **Regulatory Gate**: Authoritative synchronous pre-trade risk engine. SEBI algo ID verification, Order-to-Trade Ratio (OTR) circuit breakers, price bands, and capital freeze limits. |
| **目** | `honba-me` | Rust | **Market Eye**: Real-time market data ingestion (Zerodha, Upstox, Fyers, Angel One), binary WebSocket decoders, tick coalescing buffers, Parquet lakehouse catalog, and incremental indicators. |
| **手** | `honba-te` | Rust | **Execution Hand**: Outbound broker execution, headless RFC 6238 TOTP / OAuth2 login, token-bucket rate limiters, freeze-limit order slicing, smart order routing (SOR), and immutable audit journal. |
| **家** | `honba-ya` | Rust (Tokio) | **Runtime House**: Multi-threaded actor pipeline, institutional event-driven backtester (`honba-ya-backtest`), genetic/Bayesian strategy optimizer, REST/WebSocket gateway, and SQL persistence. |
| **脳** | `honba-nou` | Rust + Python | **AI Brain**: Quantitative research loop, closed decision validation, deterministic prompt & trade replay harness, and native MCP research server. |
| **魂** | `honba-tamashii` | Rust (PyO3) | **Python Soul**: Native C-extension compiling the `honba` PyPI package. Houses zero-copy memory bridges, PyArrow data transfers, and Tokio actor runtime lifecycle management in Python. |
| **顔** | `honba-kao` | Rust (Leptos CSR / WASM) | **User Face**: High-performance reactive web dashboard, visual Strategy Studio node canvas, interactive backtest analysis workbench, and openalgo-charts integration. |
| **適** | `honba-adapters` | Python | **User Adapters**: Python-first community broker adapters, alternative data feeds, and rapid prototyping plugins conforming to the core adapter interface. |
| **例** | `honba-examples` | Python | **Canonical Examples**: Production-ready reference strategies: NIFTY/BankNIFTY options selling, equity momentum, statistical arbitrage, and factor-based screening. |

---

## 3. Architecture Guide Roadmap

Explore the comprehensive architectural documentation:

### Core Architecture & Design
* **[System Overview](overview.md)**: Master architectural blueprint, end-to-end event lifecycle, and subsystem communication topology.
* **[Domain-Driven Design (DDD)](domain_driven_design.md)**: Deep dive into StockSharp-inspired bounded contexts, aggregates, value objects, domain events, and state invariants.
* **[Python-First Architecture](python_first.md)**: Python strategy authoring ergonomics (like Nautilus and Jesse), PyO3 zero-copy FFI bridge, and GIL-free asynchronous execution.
* **[Strategy Tester & Automatic Strategy Enhancer](strategy_tester_and_enhancer.md)**: Jesse-like backtester, Genetic/Bayesian parameter optimization, Monte Carlo stress testing, and AI-driven automated strategy enhancement.

### Intelligence, Concurrency & Microstructure
* **[AI Engine & Quant Researcher](ai_engine.md)**: Autonomous quant research loop (`honba-nou`), closed decision grammar, deterministic prompt replay, and MCP server tools.
* **[Actor Runtime & Event Loop](event_loop.md)**: Multi-threaded actor architecture (like NautilusTrader), Tokio pipelines, lock-free channels, and supervision trees.
* **[Adapter Interface & Market Feeds](adapter_interface.md)**: Inbound WebSocket feeds (`honba-me`), outbound broker execution (`honba-te`), rate limiters, and order slicers.
* **[Money, Currency & Indian Cost Modeling](money.md)**: 128-bit decimal arithmetic, currency minor units, and full Indian statutory cost formulas (STT, Stamp Duty, GST, T+1 settlement).
* **[Order Management & Rejections](order_reject_cancel.md)**: Formal OMS state transitions, idempotency via client order IDs, late-fill reconciliation, and cancel-replace handling.

### Visual Studio, WASM & Contracts
* **[Algo Designer & Simulator](algo_designer_and_simulator.md)**: Declarative node graph IR, compilation pipeline, and browser-to-backend parity.
* **[honba-kao Strategy Studio](strategy_studio.md)**: Reactive Leptos 0.7 CSR visual canvas, AI Co-Pilot, and local hypothesis journal.
* **[honba-kao Simulator Port](kao_simulator_port.md)**: WebAssembly-powered backtest simulator and analytical reporting workbench.
* **[Using the Simulator](simulator_usage.md)**: User operational guide for running backtests, inspecting charts, and analyzing trade journals.
* **[WASM Indicators Catalog](wasm_indicators.md)**: Vectorized indicators compiled to WebAssembly for sub-millisecond frontend calculation.
* **[Rust-Python Contracts & Codegen](contracts_from_rust.md)**: Automated schema generation with `schemars`, Pydantic models, and cross-language golden conformance tests.
