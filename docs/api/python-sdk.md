# Python SDK: `honba.client`

Typed access to the [REST read API](rest.md). The same `Client` methods work
over HTTP (against `honba serve`) and in process (no server).

```python
from honba.client import Client, NotFoundApiError

# Against a running `honba serve --data-dir ./bars`
with Client.http("http://127.0.0.1:8080", timeout=5) as api:
    print(api.health().status)
    for bar in api.bars("TCS.NSE", tf="1m", from_="2024-01-01", to="2024-01-02"):
        print(bar.ts_event.to_ns(), bar.close)

# No server: the same Rust router in process (needs the native extension)
api = Client.inproc("./bars")
quotes = api.quotes(["TCS"], venue="NSE")
try:
    api.instrument("NOPE.NSE")
except NotFoundApiError as err:
    print(err.code, err.category, err.retryable)  # instrument_not_found not_found False
```

## `Client`

| Method | Route | Returns |
|---|---|---|
| `health()` | `GET /health` | `Health` |
| `instruments(*, exchange=None, symbol=None)` | `GET /instruments` | `list[InstrumentInfo]` |
| `instrument(instrument_id)` | `GET /instruments/{id}` | `InstrumentInfo` |
| `bars(instrument_id, *, tf=None, from_=None, to=None)` | `GET /bars/{id}` | `list[Bar]` |
| `quotes(symbols, *, venue=None, as_of=None)` | `GET /quotes` | `list[QuoteTick]` |
| `depth(instrument_id, *, levels=None)` | `GET /depth/{id}` | `Depth` |
| `verify_strategy(manifest)` | `POST /strategies/verify` | `dict` (the IR) |
| `strategies()` | `GET /strategies` | `list[CompiledStrategy]` |
| `compile_strategy(manifest)` | `POST /strategies` | `CompiledStrategy` |

Only these routes have methods. Every other route answers 501 and has no client
method. `instrument_id` is `SYMBOL.EXCHANGE` (a string or an `InstrumentId`).
`tf` is `<n><s|m|h|d|w|mo>`, default `1m` on the server. `levels` is 1..=50.
`Client` is a context manager; `close()` releases the transport.

`CompiledStrategy` has `id` (string, `sha256:<hex>`) and `ir` (dict, the verified IR).

Constructors: `Client.http(base_url, **options)` (options go to `HttpTransport`),
`Client.inproc(data_dir)`, or `Client(transport)` for any `Transport`.

## Strategy compilation

Use `compile_strategy(manifest)` to compile and store a strategy manifest in the server's
in-process catalog. The returned `CompiledStrategy` holds a stable `id` (SHA-256 of the
canonical manifest JSON) and the verified IR. The same manifest always gets the same id
and is stored once. The catalog holds at most 1,000 strategies; a new one past the limit
raises `ValidationApiError` with `context["reason"] == "catalog_full"`. Source code in the
manifest (`code` or `source` keys) is rejected with `reason = "source_unsupported"`. Use
`strategies()` to list all compiled strategies of this process, ordered by id.

```python
api = Client.inproc("./bars")

# Compile a manifest
manifest = {...}  # strategy manifest as a dict
compiled = api.compile_strategy(manifest)
print(compiled.id)   # sha256:...
print(compiled.ir)   # {"warmup_bars": 20, "universe": {...}, ...}

# List all compiled strategies
all_strategies = api.strategies()
```

## Transports

- `HttpTransport(base_url, *, timeout=10.0, retry=None, headers=None, ...)`:
  `base_url` must start with `http://` or `https://`; `timeout` is a positive
  number of seconds; redirects are not followed.
- `InprocTransport(data_dir)`: runs the Rust router in this process over the
  `SYMBOL.EXCHANGE.parquet` files in `data_dir` (`NotADirectoryError` if it is
  not a directory). The directory is loaded once per process and cached, so
  files added later are not seen.

Both return every HTTP-style answer, error statuses included; the client turns
non-2xx answers into exceptions.

## Time inputs

`from_`, `to` and `as_of` accept a `TimeLike`:

- `UnixNanos`;
- an aware `datetime` (naive raises `naive_datetime`);
- a `date` or `YYYY-MM-DD` string (midnight UTC);
- an RFC 3339 string such as `2024-01-01T09:15:00+05:30` (validated and sent
  verbatim).

Times before 1970-01-01 (`before_epoch`) or after the largest nanosecond `i64`
instant, **2262-04-11T23:47:16.854775807Z** (`invalid_time`), are rejected. A
range with `from_ >= to` is `empty_range`.

## Errors

Bad arguments raise `RequestValidationError` (a `ValueError` with `field` and
`reason`) **before anything is sent**. A failed call raises an `ApiError`
subclass chosen by the envelope `code`. Attributes: `code`, `message`,
`status` (`None` if no response arrived), `retryable` (the server's flag),
`context`, `category`, `api_version`.

| Class | Codes |
|---|---|
| `ValidationApiError` | `validation_invalid_request` (422; includes `too_many_rows`) |
| `NotFoundApiError` | `not_found`, `instrument_not_found`, `order_not_found` |
| `AuthApiError` | `unauthorized`, `forbidden` |
| `RateLimitedApiError` | `rate_limited` |
| `RiskApiError` | `risk_max_*` |
| `OrderRejectedApiError` | `order_rejected` |
| `MarketDataUnavailableApiError` | `market_data_unavailable` (for example no depth on bars) |
| `TransportApiError` | `timeout`, `transport_error` |
| `InternalApiError` | `internal_error` |
| `UnsupportedApiError` | `unsupported` |
| `NotImplementedApiError` (subclass of `UnsupportedApiError`) | `not_implemented` (501) |
| `InvalidResponseError` | `invalid_response`: not an envelope, a payload that does not parse, or a `schema_version` mismatch |

An unknown code from a newer server stays a plain `ApiError` with the raw code.

`TransportApiError` is also raised by the transports when no answer arrived:
over HTTP for timeouts and connection failures (`retryable=True`), and in
process when the native call raises `OSError` or `ValueError`, for example a
data directory that vanished (`code="transport_error"`, `retryable=False`).

```python
from honba.client import (
    Client, NotImplementedApiError, ValidationApiError, RequestValidationError,
)

api = Client.inproc("./bars")
try:
    api.bars("TCS.NSE", tf="1m", from_="2020-01-01", to="2024-01-01")
except ValidationApiError as err:
    if err.context and err.context.get("reason") == "too_many_rows":
        print("narrow the range; limit =", err.context.get("limit"))
except RequestValidationError as err:
    print(err.field, err.reason)   # raised locally, nothing was sent
```

## Retry policy

Retries exist only on `HttpTransport` and are **off by default**.
`RetryPolicy(delays=(...))` has one delay (seconds) per retry, at most 10, each
finite and >= 0. Only an envelope with `retryable: true` or a connection
failure is retried; anything else returns at once.

```python
from honba.client import Client, RetryPolicy

api = Client.http("http://127.0.0.1:8080", retry=RetryPolicy.fixed(3, 0.5))
```

`RetryPolicy.none()` is the default; `RetryPolicy.fixed(retries, delay)` repeats
one delay.
