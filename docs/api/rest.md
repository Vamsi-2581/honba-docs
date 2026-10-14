# REST Read API and `honba serve`

`honba serve` exposes a read-only HTTP API over a directory of Parquet bar
files. Source of truth: `honba/crates/honba-api-rest` and ADR 0013.

```bash
honba serve --data-dir ./bars                      # 127.0.0.1:8080
honba serve --data-dir ./bars --addr 127.0.0.1:0   # pick a free port
```

The data directory holds `SYMBOL.EXCHANGE.parquet` files and is loaded **once** at
start-up; a bad directory fails before anything listens. The first stdout line
is `listening on http://<addr>` with the address actually bound.

| Flag | Default | Meaning |
|---|---|---|
| `--data-dir` | required | Directory of Parquet bar files |
| `--addr` | `127.0.0.1:8080` | Listen address |
| `--cors-origin <origin>` | none | Allow this browser origin (repeatable) |

## Security posture

- **Loopback by default.** If `--addr` is not loopback, `honba serve` prints a
  warning on stderr: the API has no auth or TLS, so anyone who can reach it can
  read the data.
- **No auth, no TLS, no rate limiting.**
- **No CORS by default** (no CORS headers are sent). Opt in with
  `--cors-origin https://app.example` (repeatable). `*` and values that are not
  valid header values are rejected at start-up.

## Routes

All routes below are implemented and return real data (except `/depth/{id}` which returns 404 `market_data_unavailable` over bar datasets):

| Route | Notes |
|---|---|
| `GET /health` | `{"status": "ok"}` |
| `GET /capabilities` | Derived from the endpoint registry; `not_implemented` list is now empty |
| `GET /instruments` | Filters `exchange`, `symbol`; id order |
| `GET /instruments/{id}` | `{id}` is `SYMBOL.EXCHANGE`, split on the last dot |
| `GET /bars/{id}` | `tf`, `from`, `to`; half-open `[from, to)`, ascending `ts_event` |
| `GET /quotes` | `symbols` (comma list), `venue`, `as_of` (inclusive) |
| `GET /depth/{id}` | `depth` 1..=50, default 5 — over bars returns 404 `market_data_unavailable` |
| `POST /strategies/verify` | Verifies a manifest, returns the IR |
| `GET /strategies` | Lists compiled strategies of this process, ordered by id |
| `POST /strategies` | Compiles and stores a manifest; returns its id and IR |
| `GET /screener/scan` | Evaluates screener filters over the bars; see below |
| `GET /schema` | Schema info only: `{"openapi": "3.1.0", "version": <API version>}`. The full OpenAPI document is generated, see [Contracts from Rust](../architecture/contracts_from_rust.md) |
| `POST /backtests` | Submits a backtest run, returns `run_id` |
| `GET /backtests/{id}` | Returns the run result with metrics |
| `GET /backtests/{id}/journal` | Returns the trade journal for the run |
| `GET /backtests/{id}/events` | SSE stream of run progress (`text/event-stream`) |
| `POST /sweeps` | Submits a parameter sweep |
| `GET /sweeps/{id}` | Returns the sweep report |
| `POST /orders` | Places an order (risk-gated, ADR 0018) |
| `GET /orders` | Lists orders |
| `DELETE /orders/{id}` | Cancels an order |
| `POST /positions/close` | Closes a position |
| `GET /journals/{id}` | Returns the journal for a backtest id |

`POST /orders` is risk-gated (ADR 0018): a refusal is a 422 carrying the
`risk_*` code (category `risk`, not retryable); `DELETE /orders/{id}` cancels
without a gate and `POST /positions/close` closes without one.

A request body that does not parse on these routes is still a 422 first.

### `GET /capabilities`

The manifest also carries `crates`, `market_packs`, `toolsets`, `adapters` and `features`.
`endpoints` lists every registry row as `METHOD /path` (for example
`GET /bars/{id}`). The `not_implemented` list is now empty.

### `GET /screener/scan`

Query: `universe` (JSON array of `SYMBOL.EXCHANGE`, required), `filters` (JSON filter group;
omitted matches every instrument), `tf` (default `1d`) and `as_of` (inclusive). The answer is
`{rows: [{instrument_id, metrics}]}` in instrument-id order; `metrics` holds the latest value of
every metric the filter reads (`null` while warming up). Predicates are evaluated in Rust over the
bar dataset. Computable metrics: `open`, `high`, `low`, `close`, `volume`,
`price_52_week_high`/`_low` (need 252 bars), `SMA<N>` and `RSI`. Anything else is a 422
`reason = unsupported_metric`, never an empty result; a predicate `timeframe` must equal `tf`
(`unsupported_timeframe`). An unknown instrument is a 404. A Parquet data directory holds `1m`
bars, so pass `tf=1m` there; the default `1d` is a 404 `market_data_unavailable`.

Limits, each a 422 `too_many_rows` with `context.limit`: 1,000 instruments in the universe, 500
matching rows, 2,000,000 bars read per scan, and 100,000 bars per instrument. Filters are bounded
to 8 levels of nesting and 64 predicates (`invalid_filter`). There is no pagination.

### Strategy compilation: `POST /strategies` and `GET /strategies`

**`POST /strategies`**: compiles and stores a manifest. Request body is `{manifest}` only
(unknown fields are rejected). Compilation is the same verification step as `POST /strategies/verify`;
if it succeeds, the strategy is stored in an in-process catalog and returned as
`{id, ir}` (the IR is a dict). The `id` is the SHA-256 of the canonical manifest JSON,
so the same manifest gets the same id across processes and is stored once. A manifest
already held is re-posted successfully without duplication.

The catalog is per-process, in-memory, and lost on restart; capacity is bounded at
**1,000 strategies**. A new strategy past the limit is a 422 `validation_invalid_request`
with `context.reason = catalog_full` and `context.limit`, but re-posting a manifest
already held always succeeds.

A request body containing `code` or `source` is rejected with 422
`context.reason = source_unsupported`. An invalid manifest gets the same 422 envelope
as `/strategies/verify` with `context.reason` naming the issue (e.g., `no_subscriptions`).

**`GET /strategies`**: lists all compiled strategies of this process, ordered by id.
Returns `{strategies: [CompiledStrategy, ...]}` where each has `id` and `ir`.

## Behaviour notes

- **Bars cap:** a selection of more than **100,000** bars is rejected with 422
  `validation_invalid_request`, `context.reason = "too_many_rows"` and
  `context.limit`. Narrow `from`/`to`. There is no pagination.
- **Quotes** are derived from bars: bid = ask = close of the latest bar at or
  before `as_of`, sizes `0` ("last price, no book"). A request is
  all-or-nothing: one instrument without a quote makes the whole request 404.
- **Depth** over a bar dataset is 404 `market_data_unavailable`; no levels are
  fabricated.
- Parquet files carry no reference data: instruments are derived as INR
  equities, lot 1, tick 0.05, and files are read as 1-minute bars.
- Prices are `f64` observations; timestamps are `{iso, unix_nanos}`.

| Case | Status | Code |
|---|---|---|
| Unknown instrument | 404 | `instrument_not_found` |
| No bar at or before `as_of`; timeframe not held; bar-only depth | 404 | `market_data_unavailable` |
| Bad `tf`/`from`/`to`, empty range, unknown query key, malformed id, `depth` out of range | 422 | `validation_invalid_request` |
| More than 100,000 bars | 422 | `validation_invalid_request` (`too_many_rows`) |
| Unknown route | 404 | `not_found` (an envelope, not an empty body) |
| Known path, unsupported method | 405 | `unsupported` (an envelope) |
| Route not built | 501 | `not_implemented` |
| Port unavailable / timeout / transport | 503 / 504 / 502 | `market_data_unavailable` / `timeout` / `transport_error` |

## Envelope

Every response, success or failure, is an envelope:

```json
{"api_version": "...", "schema_version": 3, "data": {"status": "ok"}}
```

```json
{
  "api_version": "...",
  "schema_version": 3,
  "error": {
    "code": "instrument_not_found",
    "message": "...",
    "retryable": false,
    "context": {"field": "id"}
  }
}
```

Exactly one of `data` and `error` is present. `category` is not on the wire; it
is derived from `code`. Use `retryable`, not the status, to decide on retry.
(The `schema_version` value shown is illustrative; read it from a response.)

## CLI

- `honba serve --data-dir DIR [--addr ADDR] [--cors-origin ORIGIN]...` as above.
- `honba verify MANIFEST.json` verifies a strategy manifest and prints the compiled IR as JSON
  (the CLI form of `POST /strategies/verify`).
- `honba schema export [--output DIR] [--typescript DIR] [--client DIR] [--openapi DIR] [--pyi DIR] [--mcp DIR] [--docs DIR]`
  writes the generated contract artifacts; `honba codegen` is an alias that emits every one
  (see [Contracts from Rust](../architecture/contracts_from_rust.md)).

Use the typed [Python client](python-sdk.md) rather than parsing this by hand.
