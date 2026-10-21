# Actor Runtime & Event Loop Architecture

> **Concurrency Architecture**: Inspired by the low-latency actor architectures of **NautilusTrader** and the **LMAX Disruptor**, Honba organizes runtime execution into a responsive, multi-threaded Tokio actor pipeline. Subsystems communicate via asynchronous, typed message-passing over bounded, lock-free channels, completely decoupled from Python's Global Interpreter Lock (GIL).

---

## 1. Concurrency Model: Multi-Threaded Actors

In high-frequency algorithmic trading, traditional multithreading using mutexes and shared memory leads to lock contention, cache invalidations, and deadlocks. 

Honba strictly enforces an **Actor Model** implemented on top of the Tokio asynchronous runtime:
* **No Shared Mutable State**: Actors own their internal state exclusively.
* **Message Passing**: Actors interact solely via strongly typed message channels (`tokio::sync::mpsc::Sender<M>` / `Receiver<M>`).
* **Pinned Thread Pools**: Core actors run on dedicated OS threads pinned to specific CPU cores, isolating network I/O from computation.

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                     HONBA ACTOR PIPELINE RUNTIME                       │
 ├────────────────────────────────────────────────────────────────────────┤
 │                                                                        │
 │   ┌───────────────────────┐                                            │
 │   │   MarketDataActor     │◄── Ingests raw exchange WebSocket streams  │
 │   └──────────┬────────────┘    (Zerodha, Upstox, Fyers, Angel One)     │
 │              │ Bounded MPSC Broadcast Channel                          │
 │              ├───────────────────────────────┐                         │
 │              ▼                               ▼                         │
 │   ┌───────────────────────┐      ┌─────────────────────────┐           │
 │   │     StrategyActor     │      │     PortfolioActor      │           │
 │   │ (Python / Rust Algo)  │      │ - Real-time MTM marks   │           │
 │   └──────────┬────────────┘      │ - FIFO Tax-Lot tracker  │           │
 │              │ Candidate Decision│ - Margin utilization    │           │
 │              ▼                   └───────────▲─────────────┘           │
 │   ┌───────────────────────┐                  │                         │
 │   │       RiskActor       │                  │                         │
 │   │ (honba-mon Gatekeeper)│                  │                         │
 │   └──────────┬────────────┘                  │                         │
 │              │ Approved OrderRequest         │ Fill / Ack Events       │
 │              ▼                               │                         │
 │   ┌───────────────────────┐                  │                         │
 │   │    ExecutionActor     │                  │                         │
 │   │ (honba-te Dispatcher) │──────────────────┘                         │
 │   └──────────┬────────────┘                                            │
 │              │ Order / Fill / Cancel Wire Stream                       │
 │              ▼                                                         │
 │   ┌───────────────────────┐                                            │
 │   │ AuditPersistenceActor │──▶ Persists to SQLite WAL / PostgreSQL     │
 │   └───────────────────────┘    with SHA-256 tamper-evident hash        │
 │                                                                        │
 └────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Actor Responsibilities & Pipeline Workflow

### 2.1 `MarketDataActor` (Subsystem: `honba-me`)
* **Role**: Primary market data distributor.
* **Ingestion**: Subscribes to raw binary WebSocket streams (Zerodha Kite binary frames, Upstox Protobuf, Fyers binary packets).
* **Normalization**: Normalizes payloads into canonical `honba_core::Tick` and `honba_core::Bar` instances.
* **Backpressure & Coalescing**: Uses a `CoalescingTickBuffer` (capacity 65,536). When downstream consumers experience backpressure, intermediate quote ticks for the same instrument are coalesced into a last-value cache, preserving the latest bid/ask and strictly monotonic timestamps without dropping trade fills.

### 2.2 `StrategyActor` (Subsystem: `honba-ya` / `honba-tamashii`)
* **Role**: Executes strategy signal generation logic.
* **Isolation**: Strategies can run either as pure compiled Rust structs or as Python classes (`Strategy` subclass).
* **Non-Blocking Dispatch**: Ticks and bars are dispatched to strategy queues. For Python strategies, `honba-tamashii` acquires the GIL only for the exact duration of the `on_tick` or `on_bar` callback, returning immediately to unblock the thread.

### 2.3 `RiskActor` (Subsystem: `honba-mon`)
* **Role**: Pre-trade compliance and regulatory risk gatekeeper.
* **Synchronous Verification**: Operates synchronously before any order touches network buffers:
  1. Validates registered SEBI `algo_id`.
  2. Evaluates Order-to-Trade Ratio (OTR) against exchange sliding windows.
  3. Verifies limit prices against scrip-wise circuit bands (2%, 5%, 10%, 20%).
  4. Enforces account-level maximum notional, position caps, and daily loss limits.

### 2.4 `ExecutionActor` (Subsystem: `honba-te`)
* **Role**: Outbound wire dispatcher and order lifecycle tracker.
* **Token-Bucket Rate Limiter**: Enforces broker-specific rate limits (Zerodha: 3 req/sec for orders; Upstox: 25 req/sec; Angel One: 10 req/sec) with cancellation priority over new orders.
* **Order Slicing (`honba-te-slicer`)**: Transparently splits parent orders exceeding exchange freeze limits (e.g. NIFTY 1,800 quantity) into child slices with randomized microsecond jitter.
* **Smart Order Router (SOR)**: Directs order flow across active broker accounts based on margin availability, latency health checks, and OTR headroom.

### 2.5 `PortfolioActor` (Subsystem: `honba-kokoro`)
* **Role**: Real-time position, tax-lot, and equity accounting.
* **FIFO Lot Matching**: Updates open tax-lot queues on fills, computing realized PnL and segregating delivery (`CNC`) holdings from leveraged intraday (`MIS`) positions.
* **MTM Mark**: Computes net liquidation value and margin utilization on every market tick.

### 2.6 `AuditPersistenceActor` (Subsystem: `honba-ya-persistence`)
* **Role**: Regulatory audit logging and persistence.
* **Append-Only Journal**: Streams all order requests, broker acknowledgments, execution fills, cancellations, and pre-trade risk checks into an append-only SQLite WAL database.
* **Tamper-Evident Hashing**: Computes SHA-256 hash chains across sequential records to provide cryptographically verifiable audit trails for exchange compliance inspections.

---

## 3. Supervision Trees & Fault Tolerance

Honba implements an Erlang/OTP-inspired **Supervision Hierarchy**:

```text
                     [Root Supervisor]
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
   [MarketData Supervisor]         [Execution Supervisor]
   (One-For-One Strategy)          (One-For-All Strategy)
        │         │                     │         │
        ▼         ▼                     ▼         ▼
     [KiteFeed] [UpstoxFeed]       [BrokerExec] [OMS State]
```

### Supervision Policies:
* **One-For-One**: If a market data feed drops its WebSocket connection, only that specific adapter actor is restarted with exponential backoff and jitter. Sibling feeds continue operating uninterrupted.
* **One-For-All**: If a critical OMS state corruption or risk gate panic occurs, all child execution actors are immediately halted, outstanding open orders are cancelled, and the supervisor enters safe liquidation mode.
* **Deterministic Drain on Exit**: On receiving `SIGINT` or `SIGTERM`, the platform executes an orderly shutdown:
  1. Freezes strategy signal ingestion.
  2. Flushes in-flight broker orders and queries broker REST endpoints for open state reconciliation.
  3. Flushes general ledger entries and audit journals to disk.

---

## 4. Python Runtime Integration (`honba.event_loop`)

When running inside Python or interactive Jupyter environments, `honba-tamashii` embeds and manages the Tokio actor runtime:

```python
from honba import event_loop

# Context manager: Starts Tokio worker threads, drains on exit
with event_loop.running(worker_threads=4):
    # Strategy execution runs in multi-threaded Rust runtime
    ...

# Explicit control
info = event_loop.start(worker_threads=8)
print(f"Runtime active: {info.flavor}, Threads: {info.worker_threads}")
event_loop.stop()
```

### Concurrency Guarantees:
* **Zero GIL Blocking**: The Tokio runtime runs on dedicated native OS threads completely detached from Python's interpreter loop.
* **Sub-Microsecond Latency**: Events crossing from Rust to Python use lock-free ring-buffers, minimizing marshalling overhead.
