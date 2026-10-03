# Adapter Interface

Broker adapters connect Honba to the outside world. They live in a
**separate repository** (`honba-adapters`) and are installed as Python
packages. The core engine never imports a specific adapter.

## Why Adapters Are Separate

The core `honba` package defines what an adapter must do — a lifecycle
facade plus a market-data role and an execution role. It does not define
how any specific broker does it. This means:

- Adding a new broker does not require touching the core repository.
- A broken adapter cannot break the engine.
- Adapters can be versioned independently and released at their own pace.

This mirrors StockSharp's `Connectors` repository, which maintains dozens
of broker integrations without any of them living in the core.

## The Adapter Surface

The contract is **three declarations**, all in `honba.adapters.base`: one
facade plus two roles. The facade is identity, capability and lifecycle; the
roles are what a caller actually depends on, so a market-data consumer never
has to implement order placement, and a paper adapter never has to fake a
REST client it does not use.

    Adapter             capabilities(), connect(), disconnect(), session(), is_connected()
      |-- MarketDataAdapter   quotes, depth, history, instrument master, streaming
      |-- ExecutionAdapter    orders, books, positions, holdings, funds, margin

    from honba.adapters.base import Adapter, ExecutionAdapter, MarketDataAdapter
    from honba.adapters.capabilities import AdapterCapabilities, Capability
    from honba.adapters.models import (
        DepthLevel,
        Funds,
        Holding,
        MarginReport,
        MarketDepth,
        OrderReport,
        Product,
        RunMode,
        SessionInfo,
        StreamMode,
        Subscription,
    )

Everything is re-exported from the package root, so the short form is
`from honba.adapters import Adapter, MarketDataAdapter, ExecutionAdapter, resolve_adapter`.

`Adapter` is an ABC. A subclass declares `name` (the registry key, equal to
`capabilities().name`) and implements the five abstract members —
`capabilities()`, `connect()`, `disconnect()`, `is_connected()` and
`session()`. Two concrete helpers come with it: `require_connected()`, the
guard every I/O method starts with, and `require_capabilities(*caps)`.

`MarketDataAdapter` and `ExecutionAdapter` are `runtime_checkable`
**Protocols**, not ABCs. An adapter satisfies a role by having the methods —
it inherits nothing — so `isinstance(adapter, MarketDataAdapter)` is true for
any object with the right shape.

Every method is `async`. Broker I/O is concurrent and a stream must not block
the engine loop. Backtest and paper adapters implement the same methods over a
simulator, which is why a strategy cannot tell the modes apart.

### The market-data role

`instruments()`, `search_instruments(query)`, `quote(instrument_id)`,
`depth(instrument_id, levels=5)`, `historical_bars(instrument_id, *, timeframe,
start, end)`, `subscribe(instruments, *, mode, callback)` and
`unsubscribe(subscription_id)`.

`timeframe` is the canonical aggregation name (`"1m"`, `"1d"`), never a
broker-specific code. A broker with a shallower book returns fewer levels than
asked for. The subscription callback runs on the adapter's transport thread,
so push events onto the engine queue rather than acting on them inline.

### The execution role

`place_order(intent, *, product, client_order_id=None)`,
`modify_order(order_id, *, quantity, price, trigger_price)`,
`cancel_order(order_id)`, `cancel_all(*, instrument_id, product)`,
`order_status(order_id)`, `orders()`, `trades()`, `positions()`, `holdings()`,
`funds()` and `margin(instrument_id)`.

`client_order_id` is the caller's idempotency key: a repeat with the same id
must not create a second order.

### Value types

`honba.adapters.models` holds the canonical values an adapter may return. They
are frozen dataclasses validated at construction, timestamps are unix
nanoseconds, and prices and quantities are `float`, matching `honba.domain`.

| Type | What it carries |
|---|---|
| `Product` | `INTRADAY`, `DELIVERY`, `CARRY` |
| `RunMode` | `BACKTEST`, `PAPER`, `LIVE` |
| `StreamMode` | `LTP`, `QUOTE`, `DEPTH` |
| `SessionInfo` | `user_id`, `mode`, `expires_at`, `accounts` |
| `MarketDepth` | `instrument_id`, `ts`, `bids`, `asks` |
| `Funds` | `available_cash`, `opening_balance`, `margin_used` |
| `Holding` | `instrument_id`, `quantity`, `average_price` |
| `OrderReport` | `status`, `filled_quantity`, `reject_reason` |
| `MarginReport` | `initial`, `maintenance`, `instrument_id` |
| `Subscription` | `id`, `instruments`, `mode` |

`MarketDepth.bids` and `.asks` are tuples of `DepthLevel`, with `best_bid`,
`best_ask` and `mid_price` helpers; `MarginReport.instrument_id` is `None` for
the account-level report. `OrderReport` also carries `ts_event`, and `Funds`
carries `currency`, `collateral`, `payin` and `payout`.

Domain types cross the boundary too: `Instrument` and `InstrumentId`,
`QuoteTick`, `TradeTick`, `Bar`, `OrderIntent`, `OrderSide` / `OrderStatus` /
`OrderType` / `TimeInForce`, `Trade` and `Position` all come from
`honba.domain`. A streaming callback receives a `StreamEvent`, which is
`QuoteTick | TradeTick | Bar`; a `QuoteTick` carries `instrument_id`, `ts`,
`bid_price`, `ask_price`, `bid_size` and `ask_size`.

## What an Adapter Supports

An adapter publishes what it can do as data, before it is called, so the engine
can refuse an unsupported request with a typed `CapabilityError` instead of
discovering it as an `AttributeError` or, worse, a failed HTTP call.

`AdapterCapabilities` carries the adapter `name` plus seven non-empty sets:
`venues`, `products`, `order_types`, `time_in_force`, `price_types`,
`stream_modes` and `features`. `supports()` and `require()` answer for
`features`; the same pair exists per dimension (`supports_product` /
`require_product`, `supports_order_type`, `supports_time_in_force`,
`supports_price_type`, `supports_stream_mode`).

Every adapter method maps to the capability it needs:

| Method | Capability |
|---|---|
| `instruments`, `search_instruments` | `INSTRUMENT_MASTER` |
| `quote` | `QUOTES` |
| `depth` | `DEPTH` |
| `historical_bars` | `HISTORICAL_BARS` |
| `subscribe`, `unsubscribe` | none — checked against `stream_modes` |
| `place_order` | `PLACE_ORDER` |
| `modify_order` | `MODIFY_ORDER` |
| `cancel_order` | `CANCEL_ORDER` |
| `cancel_all` | `CANCEL_ALL` |
| `order_status`, `orders` | `ORDER_BOOK` |
| `trades` | `TRADE_BOOK` |
| `positions` | `POSITIONS` |
| `holdings` | `HOLDINGS` |
| `funds` | `FUNDS` |
| `margin` | `MARGIN` |
| `capabilities`, `connect`, `disconnect`, `session`, `is_connected` | none |
| `require_connected`, `require_capabilities` | none |

`subscribe` and `unsubscribe` need no `Capability` because the stream mode is an
argument: the caller picks one and the adapter checks it against `stream_modes`.

### Rules every implementation follows

The shared suite in `honba.adapters.contract` enforces these — they are what
"implements the adapter contract" means:

1. Call `require_connected()` before any I/O, and refuse an unsupported
   capability with `CapabilityError` **before** making a request.
2. A broker refusal is an `OrderReport` with `status=REJECTED` and a
   `reject_reason`, never an exception.
3. An unknown instrument or order id raises `AdapterError`, never `KeyError` or
   `LookupError`.
4. Broker wire types never appear in a return value, and nothing but the
   canonical types above comes back out.

## The Adapter Contract

An adapter does three things and nothing else:

1. **Translate.** Convert broker-specific JSON into the canonical types in
   `honba.adapters.models` and `honba.domain`, and canonical commands into
   broker-specific HTTP requests.
2. **Transport.** Manage WebSocket connections, reconnection, heartbeat and
   authentication. Rate limiting is shared plumbing owned by the core
   (`honba.adapters` shared helpers, E1-S5) rather than per adapter.
3. **Report.** Emit every state change as an event — connection status,
   order acknowledgement, fill, error.

An adapter **never** makes trading decisions, computes indicators, or
maintains strategy state. If an adapter is doing any of those things, it
is doing too much.

## Uniform Internal Structure

Every adapter follows the same file layout, so a developer moving from one
broker to another recognises the code instantly:

    honba_dhan/
    ├── __init__.py
    ├── config.py        — DhanConfig (pydantic model)
    ├── data.py          — the MarketDataAdapter side of DhanAdapter
    ├── execution.py     — the ExecutionAdapter side of DhanAdapter
    ├── instruments.py   — the instrument master
    ├── websocket.py     — WS connection, reconnection, heartbeat
    ├── http.py          — REST client wrapper
    ├── parsing.py       — JSON → Honba canonical types
    └── constants.py     — endpoint URLs, error codes

The names are identical in every adapter. `config.py` is always the
configuration model. `parsing.py` is always where JSON becomes Honba
types.

The three declarations are satisfied by **one object**: `DhanAdapter(Adapter)`
implements both roles, and `data.py` / `execution.py` hold one role's methods
each.

## A Minimal Adapter

Here is a skeleton for a hypothetical broker:

    from honba.adapters.base import Adapter
    from honba.adapters.capabilities import AdapterCapabilities, Capability
    from honba.adapters.models import (
        RunMode,
        SessionInfo,
        StreamCallback,
        StreamMode,
        Subscription,
    )
    from honba.domain.instrument import InstrumentId
    from honba.domain.tick import QuoteTick

    from .config import DhanConfig
    from .http import DhanHTTP
    from .websocket import DhanWebSocket
    from .parsing import parse_quote


    class DhanAdapter(Adapter):
        name = "dhan"

        def __init__(self, config: DhanConfig):
            self.config = config
            self.http = DhanHTTP(config)
            self.ws = DhanWebSocket(config)
            self._caps = AdapterCapabilities(name="dhan", venues=frozenset({"NSE"}))

        def capabilities(self) -> AdapterCapabilities:
            return self._caps

        async def connect(self) -> SessionInfo:
            await self.http.authenticate()
            await self.ws.connect()
            return SessionInfo(user_id=self.config.client_id, mode=RunMode.LIVE)

        async def disconnect(self) -> None:
            await self.ws.close()
            await self.http.close()

        def is_connected(self) -> bool:
            return self.ws.is_connected

        async def session(self) -> SessionInfo:
            return await self.http.session_info()

        async def quote(self, instrument_id: InstrumentId) -> QuoteTick:
            self.require_connected()
            self.require_capabilities(Capability.QUOTES)
            raw = await self.http.get(f"/quote/{instrument_id}")
            return parse_quote(raw, instrument_id)

        async def subscribe(
            self,
            instruments: tuple[InstrumentId, ...],
            *,
            mode: StreamMode,
            callback: StreamCallback,
        ) -> Subscription:
            self.require_connected()
            self.capabilities().require_stream_mode(mode)
            handle = await self.ws.subscribe(tuple(instruments), mode, callback)
            return Subscription(id=handle, instruments=tuple(instruments), mode=mode)

Every I/O method starts with `require_connected()`, and an unsupported request
is refused with `CapabilityError` before the request is made. `subscribe` takes
the callback as an argument, so the engine wires the feed at subscription time
and never touches the adapter again.

## Registration

There is exactly one registry, `honba.adapters.registry`. Register a factory
under a name:

    from honba.adapters.registry import register_adapter
    import honba_dhan

    register_adapter("dhan", honba_dhan.DhanAdapter)

or let an installed package advertise itself through the `honba.adapters`
entry-point group in its own `pyproject.toml`:

    [project.entry-points."honba.adapters"]
    dhan = "honba_dhan:DhanAdapter"

Then configuration references the adapter by name:

    # configs/live/dhan_paper.toml
    [adapter]
    name = "dhan"
    mode = "paper"
    client_id = "${DHAN_CLIENT_ID}"
    access_token = "${DHAN_ACCESS_TOKEN}"

The engine resolves `"dhan"` at startup with `resolve_adapter("dhan", **config)`;
if the adapter is not installed, that raises `AdapterNotFound` listing what is
installed. Entry points are discovered lazily the first time a lookup needs them,
not at import time, so importing `honba` never touches installed distributions.
Explicit registration always wins over discovery, so a config file or a test
can override an installed adapter without uninstalling it. A broken entry point
is recorded in `discovery_errors()` instead of hiding the adapters that do load.
`AdapterRegistry` also exposes `register` / `unregister` / `get` / `create` /
`available`, and `available_adapters()` lists what this installation can use.

## Error Handling

Errors live in `honba.adapters.errors`: `AdapterError` (recoverable),
`AdapterFatalError`, `AdapterNotFound`, `CapabilityError`, `SessionError` and
`SessionExpired`. Adapters must distinguish between **recoverable** and
**fatal** errors:

- **Recoverable:** WebSocket disconnect, transient HTTP 5xx, rate limit
  hit. The adapter reconnects or backs off automatically.
- **Fatal:** Invalid credentials, account suspended, malformed response
  from a documented endpoint. The adapter raises `AdapterFatalError` and
  the engine halts cleanly.

A recoverable error must **never** lose a message silently. If the adapter
cannot deliver a fill event because the connection dropped mid-order, it
must reconcile with the broker's order status endpoint on reconnect.

## Testing Adapters

Every adapter's suite runs the shared contract suite from
`honba.adapters.contract`. It is pytest-free and deterministic — no network, no
wall clock, no randomness — and raises `ContractViolation` on the first rule
broken, without leaving the adapter connected:

    from honba.adapters.contract import verify_adapter_contract

    @pytest.mark.asyncio
    async def test_dhan_satisfies_the_contract():
        await verify_adapter_contract(DhanAdapter(DhanConfig.paper()))

`probe_quantity` sizes the probe order (keep it small: a broker with a real
account rejects an oversized probe, and a rejection is a valid outcome);
`disconnect=False` leaves the adapter connected for a test that needs it.

`honba.adapters.testing.FakeAdapter` is the in-memory reference implementation
to build against before touching a broker. It is deterministic and declares a
*subset* of the capability set on purpose, so it shows what refusing an
unsupported capability looks like. It is a test double, not a paper trading
adapter: fills are instant at the top of book, with no slippage, latency,
partial fills or margin.

Recorded broker responses live in each adapter's `tests/fixtures/`, are checked
into git, and are read with the adapters-repo helper:

    from honba_adapters_shared.fixtures import load_fixture

    def test_dhan_parses_quote():
        quote = parse_quote(load_fixture("dhan_quote_response.json"))
        assert quote.instrument_id.symbol == "RELIANCE"
        assert quote.bid_price == 2450.45

## Paper Trading and Synthetic Data

There is no separate paper or sandbox package. Paper trading is the **Sandbox
adapter** (E3-S7): an adapter that implements this same contract over the real
simulator, so a strategy cannot tell paper from live.

- Market data comes from a real broker adapter.
- Order execution is simulated locally, using the same fill models as
  backtesting.
- Positions and P&L are tracked in-memory.

This is the recommended way to validate a strategy before risking capital, and
the only difference between paper and live is fill quality.

## Adding a New Adapter

1. Create a new directory in `honba-adapters/` mirroring the structure
   above.
2. Implement the facade plus both role protocols.
3. Add fixtures and tests, including
   `await verify_adapter_contract(MyAdapter(config))`.
4. Advertise the adapter through the `honba.adapters` entry-point group.
5. Submit a pull request.

The adapter repository's CI runs each adapter's tests independently, so a
failure in one adapter never blocks a pull request touching another.
