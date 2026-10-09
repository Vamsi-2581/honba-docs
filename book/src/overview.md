# Architecture Overview

Honba is organized as a layered system with strict, enforced dependency
direction. Each layer can be developed, tested, and released independently.
The layering is not a suggestion — it is checked in CI.

## The Four Layers

    ┌─────────────────────────────────────────────────────────────┐
    │ Layer 4 — AI Feedback                                        │
    │   LLM research analyst · autoresearch loop · RL training     │
    │   MCP gateway · natural-language verification                │
    ├─────────────────────────────────────────────────────────────┤
    │ Layer 3 — Strategy & Optimization                            │
    │   Vectorized pre-filter · Nautilus-style event-driven sim    │
    │   Barter-style concurrent parameter sweep                    │
    ├─────────────────────────────────────────────────────────────┤
    │ Layer 2 — Data & Execution                                   │
    │   Broker adapters · Parquet catalog · Indian market module   │
    ├─────────────────────────────────────────────────────────────┤
    │ Layer 1 — Rust Core                                          │
    │   Deterministic event kernel · Async concurrent execution    │
    └─────────────────────────────────────────────────────────────┘

Dependencies flow **downward only**. Nothing in Layer 2 imports from Layer 3.
Nothing in Layer 1 imports from anywhere. This is enforced by
`scripts/dependency_graph.py`.

## What Each Layer Owns

### Layer 1 — Rust Core

The engine. This layer defines what a message is, what a bar is, what an
order is, and how events are dispatched. It has no knowledge of Indian
markets, brokers, or strategies. It is the piece that would be identical if
Honba were built for any other market.

Crates: `honba-messages`, `honba-entities`, `honba-algo`.

### Layer 2 — Data & Execution

The Indian specialization and the bridge to external systems. This layer
knows about NSE trading hours, STT rates, index constituents, and broker
WebSocket protocols. It translates between the external world and the
messages defined in Layer 1.

Crates: `honba-market`, `honba-algo-import`, `honba-algo-export`.

The broker adapters themselves live in a **separate repository** so that
adding a new broker does not require touching the core.

### Layer 3 — Strategy & Optimization

The strategy framework, indicators, backtesting infrastructure, and
statistical analytics. This layer answers "did the strategy work?" — it
does not answer "should the strategy exist?"

Crates: `honba-algo-strategies`, `honba-algo-indicators`,
`honba-algo-testing`, `honba-algo-analytics`.

### Layer 4 — AI Feedback

The differentiating layer. An LLM reads backtest results and generates
new hypotheses. An RL agent trains on the simulator. An MCP gateway exposes
the whole system to external AI agents.

This lives in Python because the LLM ecosystem is Python. It talks to
Layer 3 through the Python control plane.

## The Three-Tier Backtesting Pipeline

Honba does not have "a backtester." It has three, each appropriate for a
different stage of research:

| Tier | Engine | Cost | Use case |
|---|---|---|---|
| 1 | Vectorized (pandas) | Seconds | Rapid screen of 500+ strategies |
| 2 | Event-driven (Rust) | Minutes | Realistic simulation of survivors |
| 3 | Concurrent async (Barter) | Minutes, thousands in parallel | Parameter sweeps, Monte Carlo |

A strategy is expected to pass through all three. Tier 1 eliminates the
obviously broken; Tier 2 gives a realistic number; Tier 3 tells you whether
that number is robust.

## Why Rust + Python

**Rust** for the engine because nanosecond-resolution event processing with
deterministic behaviour across backtest and live is not achievable in Python
at acceptable latency. Options backtesting on NIFTY requires tick-level data
and realistic fill modelling; Python's GIL makes this impractical.

**Python** for the control plane because that is where the research,
strategy authoring, ML, and LLM ecosystem lives. A user should never have to
write Rust to test an idea.

The bridge is PyO3. Strategy logic is Python; the engine that runs it is
Rust. This is the same split Nautilus Trader uses and it works.

## What Honba Deliberately Does Not Do

- **Not a broker.** Honba routes orders through adapters, but it does not
  hold positions, custody assets, or interact with exchanges directly.
- **Not a data vendor.** Honba reads data from sources you provide
  (broker APIs, NSE bhavcopy, AMFI) and caches it in Parquet.
- **Not a charting platform.** The frontend shows backtest results and live
  P&L. Chart analysis is not the goal.
- **Not a signal service.** Honba produces research, not recommendations.

## Reading Order

If you are new, read in this order:

1. **Crate Hierarchy** — the physical organization of the code.
2. **Message Flow** — how a market tick becomes a fill.
3. **Adapter Interface** — how brokers plug in.
4. **Three-Tier Pipeline** — how backtesting actually runs.
