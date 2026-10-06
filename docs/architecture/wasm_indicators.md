# WASM Indicators

The `honba-api-wasm` crate compiles the Rust indicators to WebAssembly for the frontend and for
JS agents. Pure compute only: no tokio, filesystem or network. Values come straight from
`honba-indicators`; nothing is reimplemented.

## Exports

| Export | Signature |
|---|---|
| `indicator_series(name, params_json, closes)` | `(string, string, Float64Array) -> Float64Array`; throws a JS `Error` on failure |
| `ohlc_indicator_series(name, params_json, high, low, close)` | `(string, string, Float64Array, Float64Array, Float64Array) -> Float64Array`; throws a JS `Error` on failure |
| `list_indicators()` | `() -> string`: JSON catalog listing `input` ("close" or "ohlc"), `export` function name, params, outputs and warm-up |

The result has one value per input close (or bar). The leading warm-up values are `NaN` and are kept.
`params_json` is a JSON object (blank means `{}`); unknown fields are rejected. The `high`, `low`, and `close` arrays in `ohlc_indicator_series` must have equal length.

## Supported indicators (close-only)

| name | params (default) | warm-up `NaN`s |
|---|---|---|
| `sma`, `ema` | `period` (required) | `period - 1` |
| `rsi` (Wilder) | `period` (required) | `period` |
| `macd` | `fast` (12), `slow` (26), `signal` (9), `output`: `macd` / `signal` / `histogram` (default `macd`) | `slow + signal - 2` |
| `bollinger` | `period` (required), `k` (2.0), `output`: `middle` / `upper` / `lower` (default `middle`) | `period - 1` |

`macd` requires `fast < slow`; `bollinger` requires finite `k >= 0`.

## Supported indicators (OHLC)

Available only via `ohlc_indicator_series`:

| name | params | warm-up `NaN`s |
|---|---|---|
| `atr` (Wilder) | `period` (required, integer in `1..=1000000`) | `period - 1` |

ATR uses the first bar's `high - low` as its first true range, then `max(high - low, |high - prev_close|, |low - prev_close|)`; the first value is the mean of the first `period` true ranges, then Wilder smoothing. This matches the Python `Atr(include_first_bar=True)` but differs from the Python default (`include_first_bar=False`, which warms up `period + 1` bars).

## Errors

A failure throws a JS `Error` with a message; the call never returns infinity. It throws on:

- unknown indicator name (or a close-only name like `sma` passed to `ohlc_indicator_series`, or an
  OHLC-only name like `atr` passed to `indicator_series`);
- `period` (or `fast`/`slow`/`signal`) of 0, above 1,000,000, fractional, or missing where
  required; bad `output`; unknown param fields;
- any `NaN` or infinite input value (the message names the first bad index, for example
  `non-finite input at index 2`);
- `ohlc_indicator_series`: the `high`, `low`, and `close` arrays having different lengths;
- `ohlc_indicator_series`: `high` below `low` at any index (`IndicatorError::InvertedRange`), with
  the message `high is below low at index N` where `N` is the first such bar. `high == low` is
  valid (a flat bar);
- finite input whose result overflows `f64` (non-finite output, for example an SMA of values
  near `1e308`).

Empty input returns an empty array. Params are validated before the data, so a bad request fails
the same way on any input. `ohlc_indicator_series` then checks, in order: equal lengths, finite
values, and `high >= low`; so a non-finite value is reported before an inverted range.

## Edge cases

Pinned by `crates/honba-api-wasm/src/tests/edge_cases.rs`:

- **Flat series** (all values equal): never overflows or returns infinity. `sma`, `ema` and
  `bollinger` bands equal the constant, the `macd` histogram is `0`, and `atr` is `0`. RSI stays
  within `0..=100`.
- **`period` 1** is accepted by every period indicator; `sma` and `ema` with `period: 1` return
  the input unchanged.
- **Input shorter than the warm-up is not an error**: the result is all `NaN`, still one value per
  input (for example two closes with `period: 3`).
- **Length equal to `period`** gives exactly one value, at the end, for `sma`, `ema`, `bollinger`
  and `atr` (a 3-bar `atr` with `period: 3` yields one value). `rsi` needs `period + 1` closes
  (it needs `period` changes), so with `period` closes it is all `NaN`.

## Examples

### Close-only indicators

```js
const wasm = require("./pkg"); // output of wasm-pack build --target nodejs

const closes = Float64Array.from([10, 11, 12, 11, 13, 14, 13, 15]);
const sma = wasm.indicator_series("sma", JSON.stringify({ period: 3 }), closes);
// Float64Array [NaN, NaN, 11, 11.33.., 12, 12.66.., 13.33.., 14]

const hist = wasm.indicator_series("macd", '{"output":"histogram"}', closes);

try {
  wasm.indicator_series("sma", '{"period":0}', closes);
} catch (e) {
  console.error(e.message); // invalid params: `period` must be in 1..=1000000, got 0
}
```

### OHLC indicators

```js
const high = Float64Array.from([11, 12, 13, 12, 14, 15, 14, 16]);
const low = Float64Array.from([9, 10, 11, 10, 12, 13, 12, 14]);
const close = Float64Array.from([10, 11, 12, 11, 13, 14, 13, 15]);

const atr = wasm.ohlc_indicator_series("atr", JSON.stringify({ period: 3 }), high, low, close);
// Float64Array [NaN, NaN, 1.33.., 1.55.., ...]

try {
  // high and low have different lengths
  wasm.ohlc_indicator_series("atr", '{"period":3}', high, low.slice(0, 5), close);
} catch (e) {
  console.error(e.message); // series lengths differ: high=8, low=5, close=8
}

try {
  // bar 1 has high (2) below low (2.5)
  wasm.ohlc_indicator_series("atr", '{"period":2}',
    Float64Array.from([3, 2, 4]), Float64Array.from([1, 2.5, 1]), Float64Array.from([2, 2.2, 2]));
} catch (e) {
  console.error(e.message); // high is below low at index 1
}
```

### Catalog

```js
console.log(JSON.parse(wasm.list_indicators()).indicators.map((i) => i.name));
// ["sma", "ema", "rsi", "macd", "bollinger", "atr"]
```

Note: RSI on a flat-price series returns 100 once warmed up (no losses), the same in Rust and
Python.

## Build

Run from the `honba` repository:

```bash
wasm-pack build crates/honba-api-wasm --target nodejs   # Node
wasm-pack build crates/honba-api-wasm --target web      # browsers / bundlers
make check-wasm   # cargo check for wasm32-unknown-unknown
make test-wasm    # builds for node and runs the conformance script
```

## Parity vectors

Two conformance suites ensure parity across Rust, Python and WebAssembly:

- `schema/conformance/indicator_series.json`: 28 vectors (sma, ema, rsi, macd, bollinger over
  `empty`, `single`, `short`, `ramp`, `flat`, `zigzag`). Tested by:
  - Native Rust: `crates/honba-api-wasm/tests/indicator_conformance.rs`
  - Python: `python/tests/integration/test_indicator_series_conformance.py`
  - WebAssembly: `make test-wasm` (runs `tests/js/indicator_conformance.mjs` under node)

- `schema/conformance/ohlc_series.json`: 10 vectors (ATR with `period` 1 and 3 over various bar
  inputs). Tested by:
  - Native Rust: `crates/honba-api-wasm/tests/ohlc_conformance.rs`
  - Python: `python/tests/integration/test_ohlc_series_conformance.py`
  - WebAssembly: `make test-wasm`

Tolerance: `null` means `NaN`; integer-valued expectations match exactly; others within `1e-12` relative.
