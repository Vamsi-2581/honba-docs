# WASM & High-Performance Indicators Engine

> **Single Mathematical Source of Truth**: The `honba-api-wasm` and `honba-me-indicators` crates compile verified Rust mathematical indicators to WebAssembly (WASM) for the Leptos frontend (`honba-kao`), browser simulators, and AI evaluation loops. Compute is pure, zero-allocation, and byte-for-byte identical across native Rust, Python (`honba-tamashii`), and WASM.

---

## 1. Architectural Vision

Indicator calculations must never be reimplemented in different languages. In typical trading platforms, technical indicators calculated in Python (using Pandas/TA-Lib) diverge from indicators calculated in the browser (using JavaScript) or execution servers (in C++/Rust), creating catastrophic phantom signal discrepancies.

Honba guarantees mathematical parity by compiling the exact same Rust indicator algorithms to WebAssembly:

```text
 ┌────────────────────────────────────────────────────────┐
 │        PURE RUST INDICATOR KERNEL (`honba-me`)         │
 │  - Zero I/O, pure mathematical functions               │
 └──────────────────────────┬─────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
 ┌─────────────────────────┐ ┌─────────────────────────┐
 │   NATIVE RUST & PYTHON  │ │   WEBASSEMBLY TARGET    │
 │ - honba-ya backtester   │ │ - honba-kao Leptos CSR  │
 │ - honba-tamashii PyO3   │ │ - Interactive Canvas    │
 │ - Sub-microsecond ticks │ │ - In-browser Simulator  │
 └─────────────────────────┘ └─────────────────────────┘
```

---

## 2. WebAssembly Export Surface

Compiled via `wasm-bindgen` with zero external runtime dependencies:

| WASM Export Function | Signature | Description |
| :--- | :--- | :--- |
| `indicator_series(name, params_json, closes)` | `(string, string, Float64Array) -> Float64Array` | Computes series for close-only indicators (`SMA`, `EMA`, `RSI`, `MACD`, `Bollinger`). |
| `ohlc_indicator_series(name, params_json, high, low, close)` | `(string, string, Float64Array, Float64Array, Float64Array) -> Float64Array` | Computes series for candle-dependent indicators (`ATR`, `Supertrend`, `VWAP`, `ADX`). |
| `cpr_series(high, low, close)` | `(Float64Array, Float64Array, Float64Array) -> Float64Array` | Computes Indian Central Pivot Range (Pivot, Bottom Central, Top Central). |
| `list_indicators()` | `() -> string` | Returns JSON catalog describing inputs, parameter types, bounds, and warm-up requirements. |

---

## 3. Supported Indicator Catalog

### 3.1 Close-Based Indicators
* **SMA / EMA**: Exponential and Simple Moving Averages.
* **RSI**: Wilder-smoothed Relative Strength Index on range $[0, 100]$.
* **MACD**: Moving Average Convergence Divergence (`macd`, `signal`, `histogram`).
* **Bollinger Bands**: Moving average with configurable standard deviation multiplier $k$.

### 3.2 Candle & Indian Microstructure Indicators
* **Average True Range (ATR)**: Wilder true range for volatility normalization and stop-loss tagging.
* **Supertrend**: Dynamic ATR trailing-stop trend indicator with direction flip tagging.
* **Central Pivot Range (CPR)**: Indian intraday benchmark (Pivot, TC, BC) with virgin CPR test detection.
* **Volume-Weighted Average Price (VWAP)**: Session-anchored intraday benchmark with standard deviation bands ($+1\sigma, +2\sigma, -1\sigma, -2\sigma$).
* **Average Directional Index (ADX)**: Trend strength oscillator filtering rangebound sideways chop.

---

## 4. Conformance & Parity Testing

Two conformance suites ensure mathematical parity across native Rust, Python, and WebAssembly:
* `schema/conformance/indicator_series.json`: Validates outputs over synthetic waveforms (`empty`, `single`, `ramp`, `flat`, `zigzag`, `sine`).
* `schema/conformance/ohlc_series.json`: Validates candle-dependent indicators against known golden vectors.
* Tolerance: NaN handling matches Wilder conventions; finite values match within $1 \times 10^{-12}$ relative precision.
