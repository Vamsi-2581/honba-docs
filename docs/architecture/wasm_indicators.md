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

### 3.1 Trend & Moving Averages (StockSharp Port)
* **Moving Averages**: SMA, EMA, WMA, HMA (Hull), DEMA, TEMA, KAMA (Kaufman), ZLEMA, VWMA, ALMA (Arnaud Legoux), FRAMA (Fractal Adaptive), T3, McGinley Dynamic.
* **Trend Followers**: Supertrend (dynamic ATR bands), Parabolic SAR (PSAR), Ichimoku Kinko Hyo (Cloud, Tenkan, Kijun, Senkou A/B, Chikou).

### 3.2 Momentum & Oscillators (StockSharp Port)
* **Oscillators**: RSI (Wilder), Stochastic (%K, %D, Slow/Fast), Stochastic RSI, MACD, ROC (Rate of Change), Momentum, CCI, Williams %R.
* **Complex Oscillators**: Ultimate Oscillator, Awesome Oscillator (AO), Accelerator Oscillator (AC), CMO (Chande Momentum), DPO, Fisher Transform, QQE, TSI (True Strength Index).

### 3.3 Volatility, Range & Channels (StockSharp Port)
* **Channels**: Bollinger Bands (%B, Bandwidth), Keltner Channel, Donchian Channel (Highest High, Lowest Low).
* **Volatility Metrics**: Average True Range (ATR), Standard Deviation, Historical Volatility (Parkinson, Garman-Klass, Yang-Zhang), Chaikin Volatility, Mass Index, Relative Volatility Index (RVI), Ulcer Index.

### 3.4 Volume & Microstructure (StockSharp Port)
* **Volume & Benchmarks**: On-Balance Volume (OBV), Volume Oscillator, Volume Rate of Change (VROC), Volume-Weighted Average Price (VWAP: session, weekly, monthly + bands), Accumulation/Distribution (A/D), Chaikin Money Flow (CMF), Money Flow Index (MFI).
* **Order Book Depth**: Order Book Imbalance (bid/ask volume ratio), Weighted Mid-Price, Micro-Price, Spread (BPS), Liquidity Index.
* **Candlestick Patterns & Stats**: Doji, Hammer, Engulfing, Morning/Evening Star; Linear Regression (Slope, Intercept, R2), Pearson Correlation, Covariance.

---

## 4. Conformance & Parity Testing

Two conformance suites ensure mathematical parity across native Rust, Python, and WebAssembly:
* `schema/conformance/indicator_series.json`: Validates outputs over synthetic waveforms (`empty`, `single`, `ramp`, `flat`, `zigzag`, `sine`).
* `schema/conformance/ohlc_series.json`: Validates candle-dependent indicators against known golden vectors.
* Tolerance: NaN handling matches Wilder conventions; finite values match within $1 \times 10^{-12}$ relative precision.
