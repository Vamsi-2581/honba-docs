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

Served (answer with real data):

| Route | Notes |
|---|---|
| `GET /health` | `{"status": "ok"}` |
| `GET /capabilities` | Derived from the endpoint registry; see below |
| `GET /instruments` | Filters `exchange`, `symbol`; id order |
| `GET /instruments/{id}` | `{id}` is `SYMBOL.EXCHANGE`, split on the last dot |
| `GET /bars/{id}` | `tf`, `from`, `to`; half-open `[from, to)`, ascending `ts_event` |
| `GET /quotes` | `symbols` (comma list), `venue`, `as_of` (inclusive) |
| `GET /depth/{id}` | `depth` 1..=50, default 5 |
| `POST /strategies/verify` | Verifies a manifest, returns the IR |
| `GET /strategies` | Lists compiled strategies of this process, ordered by id |
| `POST /strategies` | Compiles and stores a manifest; returns its id and IR |

`GET /schema` answers but is a stub (`openapi` and `version` only).

**501 `not_implemented`** (in the contract, not built; category `unsupported`):

```text
POST /backtests           GET /backtests/{id}       GET /backtests/{id}/journal
POST /sweeps              GET /sweeps/{id}
POST /orders              GET /orders               DELETE /orders/{id}
POST /positions/close
GET /screener/scan        GET /journals/{id}
```

A request body that does not parse on these routes is still a 422 first.

### `GET /capabilities`

`endpoints` lists every registry row as `METHOD /path` (for example
`GET /bars/{id}`). The additive `not_implemented` list names the rows above that
answer 501. Agents should read it instead of probing.

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

Use the typed [Python client](python-sdk.md) rather than parsing this by hand.
