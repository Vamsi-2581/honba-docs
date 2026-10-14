# Adapter Configuration

How to configure the broker adapters in `honba-adapters`: environment
variables, paper and live modes, credentials, the cache directory, declared
capabilities, certification and adding a broker. For broker-by-broker
differences see [Brokers](brokers.md); for the contract itself see
[Adapter Interface](../architecture/adapter_interface.md). The authoritative
text is `honba-adapters/README.md` and each adapter's own README.

## What is implemented

| Package | Status |
|---|---|
| `honba-dhan` | Implemented: execution, market data, streaming, contract-certified |
| `honba-zerodha` | Implemented: execution, market data, KiteTicker, contract-certified |
| `honba-angelone`, `honba-fyers`, `honba-upstox`, `honba-kotak`, `honba-iifl`, `honba-motilal` | Scaffold placeholders only: uniform layout, empty `__init__.py`, no entry point, no implementation |

Only `dhan` and `zerodha` are registered through the `honba.adapters` entry
point. Naming a scaffold broker in a run config fails with `AdapterNotFound`
listing the installed adapters; that is intended.

## Environment variables

Credentials come from the environment, one prefix per adapter. `<PREFIX>MODE`
is `live` (the default) or `paper`.

| Adapter | Variables |
|---|---|
| `dhan` (`DHAN_`) | `DHAN_CLIENT_ID`, `DHAN_ACCESS_TOKEN`, `DHAN_MODE` |
| `zerodha` (`ZERODHA_`) | `ZERODHA_API_KEY`, `ZERODHA_ACCESS_TOKEN`, `ZERODHA_API_SECRET` (optional), `ZERODHA_USER_ID` (optional), `ZERODHA_MODE` |
| both | `HONBA_CACHE_DIR` (see below) |

`DhanConfig.from_env()` and `ZerodhaConfig.from_env()` read those variables.
Pass another prefix, for example `from_env(prefix="ALT_")`, to read `ALT_*`
instead; that is how you run several accounts side by side. The Zerodha access
token is a daily token, so it has to be refreshed each trading day. Credentials
belong in the environment, not in config files.

## Modes and credentials

Credentials are **required** for live. A bare `DhanAdapter()` or
`ZerodhaAdapter()` with no config and no credentials in the environment raises
`AuthenticationError` naming the missing variables (`DHAN_CLIENT_ID` and
`DHAN_ACCESS_TOKEN`, or `ZERODHA_API_KEY` and `ZERODHA_ACCESS_TOKEN`). It never
falls back to paper silently.

Ask for simulation explicitly, any of three ways:

```python
from honba_dhan import DhanAdapter, DhanConfig

DhanAdapter(mode="paper")            # no credentials needed
DhanAdapter(DhanConfig.paper())      # same, via the config
# or set DHAN_MODE=paper in the environment and call DhanAdapter()
```

Paper mode needs no credentials and uses an in-memory transport. The same
applies to `ZerodhaAdapter` and `ZerodhaConfig.paper()`.

!!! warning "Behaviour change"
    Code that built `DhanAdapter()` or `ZerodhaAdapter()` bare and relied on
    getting a paper adapter now raises `AuthenticationError`. Add
    `mode="paper"`.

A run config selects the adapter by name and the engine calls
`resolve_adapter(name, **config)`:

```toml
[adapter]
name = "dhan"
mode = "paper"
client_id = "${DHAN_CLIENT_ID}"
access_token = "${DHAN_ACCESS_TOKEN}"
```

## Cache directory

Each adapter keeps its cache (for example the instrument master) under
`$HONBA_CACHE_DIR/<broker>`. When `HONBA_CACHE_DIR` is unset the default is
`~/.cache/honba/<broker>`, so Dhan uses `~/.cache/honba/dhan` and Zerodha uses
`~/.cache/honba/zerodha`.

## Capabilities

Each adapter declares exactly what it supports in `AdapterCapabilities`. An
unsupported request raises `CapabilityError` **before** any network call.

| Dimension | Dhan | Zerodha |
|---|---|---|
| Exchanges | NSE, BSE, MCX | NSE, BSE, NFO, BFO, MCX, CDS |
| Products | delivery, intraday, carry | delivery, intraday, carry |
| Order types | market, limit, stop_limit, stop_market | market, limit, stop_limit, stop_market |
| Time in force | day, ioc | day, ioc |
| Stream modes | ltp, quote, depth | ltp, quote, depth |

Neither broker declares `trailing_stop`, `gtc`, `fok` or `gtd`.
`OrderType.TRAILING_STOP` exists in the core (see
[Position Management](../strategies/position_management.md)), but no adapter
lists it, so a trailing-stop intent raises `CapabilityError` on both.

Honba-to-wire mappings are `WireMap` objects (`DHAN_*_WIRE`, `ZERODHA_*_WIRE`
in each package's `constants.py`): one dictionary per mapping, with the reverse
derived and duplicate values rejected. `assert_covers` runs at import and fails
if a declared product, order type or time in force has no wire mapping, so a
gap shows up immediately instead of at order time.

### Dhan time in force

Dhan now sends the validity that matches the intent's time in force: `DAY`
orders go as `"DAY"` and `IOC` orders as `"IOC"`. Earlier versions always sent
`DAY`, so an IOC intent was silently placed as a day order. A time in force that
is not mapped is rejected with `CapabilityError` before any network call.

!!! warning "Not verified against live Dhan"
    The `"IOC"` wire value is covered by the adapter's offline unit tests, not
    by a recorded or live Dhan response. Treat it as unverified until it has
    been confirmed against the live DhanHQ API, and do not rely on IOC
    semantics with real money before then.

## Certification

An adapter is certified when its entry passes the shared contract suite:

```sh
python -m pytest tests/test_contract_suite.py -v
```

The suite runs `honba.adapters.contract.verify_adapter_contract` over every
package that has `tests/fixtures/<name>/routes.toml`. Read endpoints are
replayed from recorded broker responses in that directory; the stateful order
lifecycle uses the adapter's in-memory mock handler. Packages without a
`routes.toml`, which today means the six scaffold brokers, are discovered
automatically and reported as skipped placeholders. The suite needs Python
3.11 or newer (`tomllib`).

`routes.toml` holds three importable `module:attr` hooks (`adapter`, `config`
with a `.paper()` constructor, and `fallback`, the in-memory handler) and a
`[routes]` table mapping a URL path fragment to a recorded file; the longest
matching fragment wins.

Dhan also has live, read-only smoke tests marked `live`. They skip unless the
credentials are set, and a conftest guard refuses any request that is not a
documented read endpoint. `scripts/smoke_test.py` defaults to paper and refuses
live mode unless `HONBA_LIVE_SMOKE_TEST=1` is set.

## CI

The adapters' CI builds `honba` from source rather than installing it, because
core is not on PyPI and needs its Rust extension built with maturin. The
workflow checks out `honba-labs/honba` at the ref in `HONBA_REF` (currently
`main`), runs `maturin develop`, installs `shared`, `dhan` and `zerodha`
editable, then runs ruff and the full pytest suite. If the core contract changes
incompatibly, pin `HONBA_REF` to a tag or SHA until the adapters catch up.

## Adding a broker

Start from `dhan/` as the template, then:

1. Implement the facade and both roles in `<name>/src/honba_<name>/`, declaring
   exactly what the broker supports in `AdapterCapabilities`.
2. Build the honba-to-wire tables with `WireMap` and call `assert_covers(...)`
   at import.
3. Declare the entry point in `<name>/pyproject.toml` under
   `[project.entry-points."honba.adapters"]`.
4. Record read responses under `tests/fixtures/<name>/` and add `routes.toml`
   there. The contract suite picks the package up with no edit to
   `tests/test_contract_suite.py`.
5. Add the package to `pytest.ini`, the CI workflow's `pip install -e` line and
   the install instructions.

The full walkthrough is the mdBook chapter "Adding Broker Adapters"
(`book/src/contributing/adding_adapters.md` in this repository) and
`honba-adapters/MIGRATION.md`.
