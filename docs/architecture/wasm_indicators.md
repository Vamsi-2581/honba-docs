# WASM Indicators

The `honba-api-wasm` crate compiles the Rust indicators to WebAssembly for the frontend and for
JS agents. Pure compute only: no tokio, filesystem or network. Values come straight from
`honba-indicators`; nothing is reimplemented.

## Exports

| Export | Signature |
|---|---|
| `indicator_series(name, params_json, closes)` | `(string, string, Float64Array) -> Float64Array`; throws a JS `Error` on failure |
| `list_indicators()` | `() -> string`: JSON catalog of names, params, outputs and warm-up |

The result has one value per input close. The leading warm-up values are `NaN` and are kept.
`params_json` is a JSON object (blank means `{}`); unknown fields are rejected.

## Supported indicators

| name | params (default) | warm-up `NaN`s |
|---|---|---|
| `sma`, `ema` | `period` (required) | `period - 1` |
| `rsi` (Wilder) | `period` (required) | `period` |
| `macd` | `fast` (12), `slow` (26), `signal` (9), `output`: `macd` / `signal` / `histogram` (default `macd`) | `slow + signal - 2` |
| `bollinger` | `period` (required), `k` (2.0), `output`: `middle` / `upper` / `lower` (default `middle`) | `period - 1` |

`macd` requires `fast < slow`; `bollinger` requires finite `k >= 0`. ATR needs OHLC bars and is
not exposed.

## Errors

A failure throws a JS `Error` with a message; the call never returns infinity. It throws on:

- unknown indicator name;
- `period` (or `fast`/`slow`/`signal`) of 0, above 1,000,000, fractional, or missing where
  required; bad `output`; unknown param fields;
- any `NaN` or infinite input value;
- finite input whose result overflows `f64` (non-finite output, for example an SMA of values
  near `1e308`).

Empty input returns an empty array. Params are validated before the data, so a bad request fails
the same way on any input.

## Example

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

console.log(JSON.parse(wasm.list_indicators()).indicators.map((i) => i.name));
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

`schema/conformance/indicator_series.json` holds 28 vectors (sma, ema, rsi, macd, bollinger over
the inputs `empty`, `single`, `short`, `ramp`, `flat`, `zigzag`). It is run by native Rust
(`crates/honba-api-wasm/tests/indicator_conformance.rs`), by Python
(`python/tests/integration/test_indicator_series_conformance.py`) and by the real wasm build
under node (`make test-wasm`). Tolerance: `null` means `NaN`; integer-valued expectations match
exactly; others within `1e-12` relative.
