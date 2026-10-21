# Python-First Architecture in Honba

> **Design Principle**: Python-First developer ergonomics with bare-metal Rust performance under the hood. Inspired by the architectural triumphs of **NautilusTrader**, **Jesse**, and modern **StockSharp**, Honba provides an intuitive, highly expressive Python strategy framework where all computationally intensive, concurrent, and safety-critical execution is delegated to an unyielding Rust actor core.

---

## 1. Why Python-First?

In quantitative finance, Python is the undisputed lingua franca for data science, machine learning, and strategy exploration. However, pure-Python trading frameworks inevitably hit severe architectural limitations:
* **Global Interpreter Lock (GIL) Contention**: High-frequency tick feeds overwhelm the Python event loop, causing severe latency spikes.
* **Floating-Point Inaccuracy**: Native Python floats produce fractional paisa/cent rounding errors across millions of transactions.
* **Concurrency Deadlocks**: Multi-threaded Python backtesting or live execution is complex and prone to race conditions.

Honba solves this through a **Python-First, Rust-Underneath Architecture**:

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      HONBA PYTHON-FIRST STACK                          │
 ├────────────────────────────────────────────────────────────────────────┤
 │  USER PYTHON LAYER (honba SDK / User Code)                            │
 │  - Declarative Strategy API (like Jesse & NautilusTrader)              │
 │  - PyTorch / Scikit-learn / LightGBM Machine Learning Models           │
 │  - Polars & PyArrow Vectorized Analysis                                │
 ├────────────────────────────────────────────────────────────────────────┤
 │  ZERO-COPY FFI BRIDGE (honba-tamashii)                                 │
 │  - PyO3 Native C-Extension compiling the `honba` package               │
 │  - PyArrow Zero-Copy Shared Memory for Ticks, Bars & Depth             │
 │  - Smart GIL Management: Workers run free, acquiring GIL only on hook  │
 ├────────────────────────────────────────────────────────────────────────┤
 │  HIGH-PERFORMANCE MULTI-THREADED RUST CORE                             │
 │  - Central Tokio Actor Pipeline (`honba-ya`)                           │
 │  - Zero-I/O Domain & Exact 128-bit Decimal Accounting (`honba-kokoro`)  │
 │  - Synchronous SEBI Regulatory Pre-Trade Risk Gate (`honba-mon`)       │
 │  - Sub-microsecond Multi-Broker WebSocket Ingestion (`honba-me`)       │
 └────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Python Strategy API & Ergonomics

Honba's strategy interface is clean, expressive, and type-annotated, combining Jesse's declarative simplicity with NautilusTrader's precision:

```python
from decimal import Decimal
from honba import Strategy, Parameter, Bar, Tick, OrderEvent, Position
from honba.indicators import EMA, RSI, ATR

class MeanReversionNiftyStrategy(Strategy):
    """
    A statistical mean-reversion strategy trading NIFTY intraday equities.
    Employs Jesse-like declarative parameters and Nautilus-like event hooks.
    """
    
    # Declarative Optimization DNA (Used by GA and Bayesian Optimizers)
    rsi_period = Parameter(default=14, min=7, max=28, step=1)
    rsi_oversold = Parameter(default=30, min=20, max=40, step=2)
    rsi_overbought = Parameter(default=70, min=60, max=80, step=2)
    atr_multiplier = Parameter(default=Decimal("1.5"), min=Decimal("1.0"), max=Decimal("3.0"), step=Decimal("0.5"))

    def on_start(self) -> None:
        """Called once when strategy initializes (live or backtest)."""
        self.symbol = "NSE:RELIANCE"
        self.subscribe_bars(self.symbol, timeframe="5m")
        
        # Incremental streaming indicators (computed in Rust without GIL)
        self.rsi = RSI(period=self.rsi_period)
        self.ema_trend = EMA(period=50)
        self.atr = ATR(period=14)

    def on_bar(self, bar: Bar) -> None:
        """Called upon completion of each canonical timeframe candle."""
        # Update streaming indicators
        self.rsi.update(bar.close)
        self.ema_trend.update(bar.close)
        self.atr.update(bar.high, bar.low, bar.close)
        
        if not self.rsi.is_ready or not self.ema_trend.is_ready:
            return

        current_pos: Position = self.portfolio.get_position(self.symbol)

        # Long Entry: Bullish trend with oversold momentum
        if current_pos.is_flat:
            if bar.close > self.ema_trend.value and self.rsi.value < self.rsi_oversold:
                # Dynamic ATR-based stop-loss
                stop_price = bar.close - (self.atr.value * self.atr_multiplier)
                
                # High-level execution helper with Indian product tagging
                self.buy(
                    symbol=self.symbol,
                    qty=50,
                    product="MIS",           # Intraday margin
                    algo_id="SEBI_ALGO_0491", # Registered SEBI Algo ID
                    stop_loss=stop_price,
                    take_profit=bar.close + (self.atr.value * self.atr_multiplier * Decimal("2.0"))
                )

        # Long Exit: Momentum reaches overbought extreme
        elif current_pos.is_long:
            if self.rsi.value > self.rsi_overbought:
                self.close_position(self.symbol, reason="RSI Overbought Target")

    def on_order_event(self, event: OrderEvent) -> None:
        """Real-time execution callbacks (Ack, Fill, Cancel, Reject)."""
        if event.is_filled:
            self.logger.info(f"Fill confirmed: {event.qty} @ ₹{event.price} (Fees: ₹{event.total_fees})")
        elif event.is_rejected:
            self.logger.error(f"Order rejected by risk engine: {event.rejection_reason}")
```

---

## 3. The `honba-tamashii` FFI Bridge

The bridge between Python and Rust is housed exclusively inside `honba-tamashii`, compiling into a single native C-extension via **PyO3**:

```text
Python Interpreter (CPython 3.12+)
      │
      ▼ PyO3 C-Extension
┌────────────────────────────────────────────────────────┐
│               honba-tamashii (Rust)                    │
├────────────────────────────────────────────────────────┤
│ - GIL Management (py.allow_threads)                    │
│ - Zero-Copy Arrow Tables (Arrow PyCapsule Protocol)    │
│ - Type Marshalling (Decimal <-> rust_decimal)          │
│ - Tokio Actor Runtime Lifecycle Management             │
└────────────────────────────────────────────────────────┘
      │
      ▼ Pure Rust Engine
Rust Core (honba-core, honba-ya, honba-kokoro, honba-mon)
```

### 3.1 Smart GIL Management (`py.allow_threads`)
Under Honba, the Python GIL is never held during:
* Ingesting high-throughput binary WebSocket ticks from exchanges.
* Updating order book depth and calculating scrip price bands.
* Evaluating pre-trade risk compliance rules (`honba-mon`).
* Reconciling double-entry general ledger accounts and FIFO tax lots.
* Running event-driven backtesting loops and optimization iterations.

The GIL is acquired **only for the microseconds required to invoke Python callbacks** (`on_tick`, `on_bar`, `on_order_event`). This allows external background threads to operate at hardware line-rate.

### 3.2 Zero-Copy PyArrow Memory Bridge
When dealing with years of historical tick or 1-minute bar data for quantitative research, serializing millions of objects between Python and Rust introduces prohibitive memory overhead.

Honba leverages the **PyArrow PyCapsule Protocol**:
* Market data is stored in Apache Arrow format inside Rust.
* Rust hands memory pointers directly to Python without serialization or data copying.
* Python strategies can query Polars or Pandas DataFrames instantaneously:

```python
import polars as pl
from honba.data import load_catalog

# Returns a Polars DataFrame backed by zero-copy Arrow memory
df = load_catalog(symbol="NSE:NIFTY50", timeframe="1m", start="2025-01-01", end="2026-01-01")
print(df.head())
```

---

## 4. Feature Comparison: Honba vs NautilusTrader vs Jesse vs StockSharp

| Capability | Honba | NautilusTrader | Jesse | StockSharp |
| :--- | :---: | :---: | :---: | :---: |
| **Primary Language** | **Python First (Rust Core)** | Python First (Rust Core) | Python Only | C# / Python Wrapper |
| **Monetary Arithmetic** | **Exact Decimal (128-bit)** | Float / Fixed Int | Float64 | Decimal (C#) |
| **Pre-Trade Risk Gate** | **SEBI Algo ID, OTR, Circuit Bands** | General Risk Limits | Basic Stop-loss | Comprehensive |
| **Double-Entry Ledger** | **Full Audit Journal** | Cash Account | Cash Balance | Cash & Portfolio |
| **Indian Market Microstructure** | **Native (NSE, T+1, STT, ICCL)** | Generic Global | Crypto Only | Global / Broker-specific |
| **AI-Native & MCP Server** | **Native (`honba-nou`)** | No | No | No |
| **Automatic Strategy Enhancer** | **GA, Bayesian, WFO, DSR** | Basic Optimization | GA Optimization | Genetic & Brute-force |
| **UI Studio & Simulator** | **Leptos CSR Canvas (`honba-kao`)** | No Official UI | Terminal / Basic Web | WPF Desktop / Web |

---

## 5. Development Lifecycle & Tooling

To ensure world-class developer ergonomics in Python:
* **Strict Typing & Stubs**: Full PEP 484/561 type stub files (`.pyi`) shipped with every binary wheel, providing rich autocompletion and type checking in VS Code, PyCharm, and Cursor.
* **Pydantic v2 Contracts**: Wire payloads and strategy manifests are validated using Pydantic v2 schemas generated directly from Rust `schemars` specifications, ensuring 100% cross-language schema parity.
* **Interactive Notebook Support**: First-class support for JupyterLab, Google Colab, and Marimo notebooks, allowing quants to plot backtest tear-sheets, equity curves, and trade distributions interactively.
