# Adapter Interface & Market Microstructure Pipelines

> **Decoupled Architecture**: High-frequency institutional market data ingestion (`honba-me`) is strictly separated from outbound broker order execution (`honba-te`). A quantitative strategy in Honba can seamlessly observe live market depth on one broker feed (e.g. Zerodha Kite 20-depth) while executing orders across another broker (e.g. Upstox or Angel One) through a cost-optimized Smart Order Router.

---

## 1. Subsystem Topology: Eye (`honba-me`) vs Hand (`honba-te`)

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                     HONBA ADAPTER ARCHITECTURE                         │
 ├────────────────────────────────────────────────────────────────────────┤
 │                                                                        │
 │   INBOUND MARKET DATA EYE: honba-me                                    │
 │   - Binary WebSocket Decoders: Kite Ticker, Upstox Protobuf, Fyers     │
 │   - Coalescing Tick Buffer (65,536 capacity, backpressure safe)        │
 │   - Real-time Price Band & VWAP Enrichment                             │
 │   - Parquet Lakehouse Ingestion (Arrow 53, Decimal128)                 │
 │                                                                        │
 │                          │ Canonical Ticks / Bars                      │
 │                          ▼                                             │
 │   CENTRAL ACTOR RUNTIME & PRE-TRADE RISK (`honba-ya` & `honba-mon`)    │
 │   - Evaluates Strategy Signal (Python / Rust)                          │
 │   - Synchronous SEBI Algo ID, OTR Breaker & Circuit Band Check         │
 │                                                                        │
 │                          │ Approved OrderRequest                       │
 │                          ▼                                             │
 │   OUTBOUND EXECUTION HAND: honba-te                                    │
 │   - Headless Auth: RFC 6238 TOTP & CDSL eDIS TPIN Pipeline            │
 │   - Token-Bucket Rate Limiter (per-broker exchange quotas)             │
 │   - Freeze-Limit Slicer (splits parent orders > exchange cap)          │
 │   - Dynamic Smart Order Router (SOR v2)                                │
 │   - Immutable SQLite WAL Tamper-Evident Audit Journal                  │
 │                                                                        │
 └────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Inbound Market Data Ingestion (`honba-me`)

### 2.1 The `MarketDataFeed` Trait
Defined in `honba-me-core`, this trait represents the streaming market data contract:

```rust
use async_trait::async_trait;
use tokio::sync::mpsc;
use honba_core::{InstrumentId, Tick, Bar};

#[async_trait]
pub trait MarketDataFeed: Send + Sync {
    /// Connects to exchange stream and begins background ingestion
    async fn connect(&mut self) -> Result<(), MarketDataError>;
    
    /// Subscribes to live ticks for a list of instruments
    async fn subscribe(&mut self, symbols: &[InstrumentId]) -> Result<(), MarketDataError>;
    
    /// Subscribes to aggregated candles (e.g. 1m, 5m)
    async fn subscribe_bars(&mut self, symbols: &[InstrumentId], timeframe: &str) -> Result<(), MarketDataError>;
    
    /// Returns a bounded receiver yielding canonical ticks
    fn tick_stream(&self) -> mpsc::Receiver<Tick>;
}
```

### 2.2 Binary Wire Protocol Decoders
Retail broker JSON feeds are inadequate for high-throughput market microstructure. `honba-me` implements native, zero-allocation binary decoders:

| Broker Feed | Protocol Format | Supported Modes | Wire Characteristics |
| :--- | :--- | :--- | :--- |
| **Zerodha Kite Ticker** | Binary TCP stream / zlib | LTP (8B), Quote (28B), Full 5-Depth (44B), Full 20-Depth (184B) | Big-endian packed struct, sub-millisecond parsing |
| **Upstox v2 Feed** | Binary Protobuf (WebSocket) | LTP, Full Market Depth (5-depth), Greeks | Google Protobuf schema with LZ4 decompression |
| **Fyers v3 Data Socket** | Binary TCP packets | 72-byte Lite / 200-byte Full Depth | Custom binary header with little-endian fields |
| **Angel One SmartStream**| Binary WebSocket | Best 5 Bid/Ask, LTP, Open Interest | 83-byte binary token packet |

### 2.3 Tick Coalescing & Backpressure Handling
During extreme market volatility (such as Indian Union Budget days or RBI policy announcements), tick arrival rates can exceed downstream processing capacity:
* **Bounded Channels**: Channels are bounded to 65,536 elements.
* **Last-Value Coalescing**: If the queue fills, intermediate quotes for the same instrument are coalesced into a last-value cache, preserving the latest bid/ask spread and strictly monotonic timestamps without dropping execution fills.
* **Auto-Reconnect State Machine**: If the WebSocket drops, the feed immediately reconnects with exponential backoff and jitter, resubscribes all active instruments, and queries the broker's REST API for open order reconciliations.

---

## 3. Outbound Execution Engine (`honba-te`)

### 3.1 The `BrokerExecutor` Trait
Defined in `honba-te-core`, this trait governs outbound execution:

```rust
#[async_trait]
pub trait BrokerExecutor: Send + Sync {
    /// Submits a new order request to the broker
    async fn submit_order(&self, req: OrderRequest) -> Result<OrderAck, ExecutionError>;
    
    /// Cancels an existing open order by cl_ord_id
    async fn cancel_order(&self, cl_ord_id: Uuid) -> Result<OrderAck, ExecutionError>;
    
    /// Modifies price or quantity of an active order
    async fn modify_order(&self, cl_ord_id: Uuid, new_price: Price, new_qty: Quantity) -> Result<OrderAck, ExecutionError>;
    
    /// Queries current open positions from broker
    async fn get_positions(&self) -> Result<Vec<Position>, ExecutionError>;
    
    /// Queries available ledger cash and margin
    async fn get_margin(&self) -> Result<MarginState, ExecutionError>;
}
```

### 3.2 Headless Authentication (`honba-te-auth`)
Retail brokers require daily manual browser logins to obtain access tokens. `honba-te-auth` automates this via an institutional headless auth daemon:
* **RFC 6238 TOTP Login**: Automatically generates two-factor authentication codes from stored TOTP secrets, logging in at 08:30 IST every morning without human intervention.
* **CDSL eDIS Pre-Trade Delivery Authorization**: Automatically dispatches CDSL TPIN and OTP authorizations for delivery sell orders (`CNC`) to unlock demat shares prior to market open.
* **Secure Keyring Storage**: Credentials are encrypted at rest using system keychains (Linux Secret Service / Apple Keychain).

### 3.3 Token-Bucket Rate Limiter (`honba-te-ratelimit`)
Exchanges ban API keys that violate HTTP rate limits. Honba enforces an adaptive token-bucket rate limiter per broker:
* **Zerodha Kite**: 3 orders/second, 10 modifications/second.
* **Upstox**: 25 requests/second.
* **Angel One**: 10 requests/second.
* **Queue Prioritization**: Cancellation requests (`CancelOrder`) bypass the queue and take precedence over new order requests (`PlaceOrder`), ensuring risk-reducing actions are dispatched instantly.

### 3.4 SEBI Freeze Limit Slicer (`honba-te-slicer`)
Indian exchanges enforce strict per-order quantity ceilings ("freeze limits") to prevent rogue fat-finger market crashes:
* **NIFTY Options**: 1,800 quantity per order.
* **BANKNIFTY Options**: 900 quantity per order.
* **Equity Scrips**: Scrip-wise limits published daily by NSE.

When a strategy submits a parent order exceeding the freeze limit (e.g. 5,000 NIFTY contracts), `honba-te-slicer` automatically slices the order into valid child slices, applying randomized microsecond jitter to avoid exchange algorithm detection.

### 3.5 Dynamic Smart Order Router (SOR v2)
For multi-broker operations, `honba-te-router` dynamically routes orders:
1. **Cost Optimization**: Directs delivery equity trades to zero-brokerage accounts and options to flat ₹20/order brokers.
2. **Health & Latency Routing**: Continuously monitors broker API latency; if a broker's latency spikes above 500ms or returns HTTP 5xx errors, traffic fails over automatically to backup brokers.
3. **OTR Balancing**: Distributes order cancellations across accounts to prevent any single broker account from breaching SEBI's Order-to-Trade Ratio limits.

---

## 4. Execution Decorator Pipeline

Execution adapters can be layered using the Decorator Pattern:

```text
 ┌────────────────────────────────────────────────────────┐
 │              ExecutionActor (honba-ya)                 │
 └───────────────────────────┬────────────────────────────┘
                             │
 ┌───────────────────────────▼────────────────────────────┐
 │  Logging & Immutable Audit Decorator (`honba-te-audit`)│
 │  - Hashes all payloads to SQLite WAL (SHA-256)         │
 └───────────────────────────┬────────────────────────────┘
                             │
 ┌───────────────────────────▼────────────────────────────┐
 │  Pre-Trade Freeze Limit Slicer (`honba-te-slicer`)     │
 │  - Slices quantities > 1,800 into child orders         │
 └───────────────────────────┬────────────────────────────┘
                             │
 ┌───────────────────────────▼────────────────────────────┐
 │  Token-Bucket Rate Limiter (`honba-te-ratelimit`)      │
 │  - Enforces per-broker order rate limits (3 req/sec)   │
 └───────────────────────────┬────────────────────────────┘
                             │
 ┌───────────────────────────▼────────────────────────────┐
 │  Latency Injection Decorator (Backtest Simulation)     │
 │  - Injects realistic 5ms–20ms network transit jitter   │
 └───────────────────────────┬────────────────────────────┘
                             │
 ┌───────────────────────────▼────────────────────────────┐
 │  Wire Network Transport (Reqwest / Tokio-Tungstenite)  │
 └────────────────────────────────────────────────────────┘
```

---

## 5. First-Party Rust vs User Python Adapters

Honba maintains an intentional boundary between core institutional adapters and user-space adapters:

| Attribute | First-Party Rust Adapters (`honba-te`) | User Python Adapters (`honba-adapters`) |
| :--- | :--- | :--- |
| **Language** | 100% Rust | 100% Python |
| **Supported Brokers** | Zerodha Kite, Upstox, Angel One, Fyers | Nuvama, Finvasia, Alice Blue, custom brokers |
| **Source Directory** | `honba-te/crates/honba-te-*` | `honba-adapters/src/honba/adapters/*` |
| **Performance** | Sub-millisecond, zero-allocation serialization | Standard Python asyncio |
| **Target Use Case** | Production institutional trading desks | Rapid prototyping, alternative brokers, community |
