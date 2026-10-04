# Adding Broker Adapters

Adapter development happens in the `honba-adapters` repository, not in
the core `honba` repository. This document describes how to add a new
adapter, test it, and publish it.

## Before You Start

Adding an adapter is a substantial piece of work — typically 2–5 days
for a broker you have never integrated before. Before starting:

1. Check whether the broker is already supported. Adapters exist for
   Dhan, Zerodha, Angel One, Fyers, Upstox, Kotak, IIFL, and Motilal
   Oswal.
2. Read [Adapter Interface](../architecture/adapter_interface.md).
3. Get API access from the broker. Most Indian brokers require
   registration and a static IP for order placement.

## The Uniform Structure

Every adapter in `honba-adapters/` follows the same file layout. This
is not a stylistic preference — it means a developer familiar with one
adapter can read any other.

    honba-<broker>/
    ├── pyproject.toml
    ├── README.md
    ├── src/
    │   └── honba_<broker>/
    │       ├── __init__.py
    │       ├── config.py           # Pydantic config model
    │       ├── data.py             # MarketDataAdapter methods
    │       ├── execution.py        # ExecutionAdapter methods
    │       ├── instruments.py      # InstrumentProvider
    │       ├── websocket.py        # WebSocket connection
    │       ├── http.py             # REST client wrapper
    │       ├── parsing.py          # JSON → Honba types
    │       └── constants.py        # Endpoints, error codes
    └── tests/
        ├── conftest.py
        ├── fixtures/
        │   ├── instruments.json
        │   ├── quotes.json
        │   ├── orders.json
        │   └── ...
        ├── test_config.py
        ├── test_data.py
        ├── test_execution.py
        └── test_parsing.py

## Step 1: Create the Package Skeleton

    cd honba-adapters
    cp -r dhan/ mybroker/       # Copy an existing adapter as a template
    cd mybroker

    # Rename the package directory
    mv src/honba_dhan src/honba_mybroker

    # Rename the module references
    grep -rl "honba_dhan\|honba-dhan\|Dhan" . | xargs sed -i 's/honba_dhan/honba_mybroker/g; s/honba-dhan/honba-mybroker/g; s/Dhan/MyBroker/g'

The Dhan adapter is the recommended template because it has the
cleanest structure. The Zerodha adapter is a good second reference
for brokers with a daily-refresh OAuth flow.

## Step 2: Define the Configuration Model

`config.py`:

    from pydantic import BaseModel, Field, SecretStr


    class MyBrokerConfig(BaseModel):
        client_id: str = Field(..., description="API client ID")
        access_token: SecretStr = Field(..., description="Access token")
        base_url: str = "https://api.mybroker.com"
        ws_url: str = "wss://ws.mybroker.com"

        # Connection tuning
        reconnect_attempts: int = 5
        reconnect_backoff_seconds: float = 2.0
        heartbeat_seconds: int = 30

        # Rate limits
        rate_limit_per_second: int = 10
        order_rate_limit_per_second: int = 3

        class Config:
            extra = "forbid"

The `SecretStr` type prevents the token from appearing in logs.

## Step 3: Implement the REST Client

`http.py`:

    import httpx
    from tenacity import retry, stop_after_attempt, wait_exponential

    from .config import MyBrokerConfig


    class MyBrokerHTTP:
        def __init__(self, config: MyBrokerConfig):
            self.config = config
            self.client = httpx.AsyncClient(
                base_url=config.base_url,
                timeout=10.0,
                headers={"Authorization": f"Bearer {config.access_token.get_secret_value()}"},
            )

        async def authenticate(self) -> None:
            resp = await self.client.post("/auth/verify")
            resp.raise_for_status()

        @retry(
            stop=stop_after_attempt(3),
            wait=wait_exponential(multiplier=1, min=1, max=10),
        )
        async def get(self, path: str, **params) -> dict:
            resp = await self.client.get(path, params=params)
            resp.raise_for_status()
            return resp.json()

        async def post(self, path: str, json: dict) -> dict:
            resp = await self.client.post(path, json=json)
            resp.raise_for_status()
            return resp.json()

        async def close(self) -> None:
            await self.client.aclose()

Retry logic belongs here, not in the client implementation. A 5xx
response is retried; a 4xx response is raised immediately.

## Step 4: Implement the WebSocket Client

`websocket.py`:

    import asyncio
    import json
    import logging
    from typing import Callable

    import websockets

    from .config import MyBrokerConfig

    log = logging.getLogger(__name__)


    class MyBrokerWebSocket:
        def __init__(self, config: MyBrokerConfig):
            self.config = config
            self.ws = None
            self._handlers: dict[str, Callable[[dict], None]] = {}
            self._running = False

        async def connect(self) -> None:
            self.ws = await websockets.connect(self.config.ws_url)
            self._running = True
            asyncio.create_task(self._receive_loop())

        async def _receive_loop(self) -> None:
            while self._running:
                try:
                    raw = await self.ws.recv()
                    message = json.loads(raw)
                    self._dispatch(message)
                except websockets.ConnectionClosed:
                    log.warning("WebSocket closed, reconnecting")
                    await self._reconnect()
                except Exception:
                    log.exception("Error in receive loop")

        def _dispatch(self, message: dict) -> None:
            msg_type = message.get("type")
            handler = self._handlers.get(msg_type)
            if handler:
                handler(message)

        def on(self, msg_type: str, handler: Callable[[dict], None]) -> None:
            self._handlers[msg_type] = handler

        async def subscribe(self, symbols: list[str]) -> None:
            await self.ws.send(json.dumps({
                "action": "subscribe",
                "symbols": symbols,
            }))

        async def _reconnect(self) -> None:
            for attempt in range(self.config.reconnect_attempts):
                try:
                    await asyncio.sleep(self.config.reconnect_backoff_seconds * (2 ** attempt))
                    await self.connect()
                    return
                except Exception:
                    continue
            log.error("Failed to reconnect after %d attempts", self.config.reconnect_attempts)

        async def close(self) -> None:
            self._running = False
            if self.ws:
                await self.ws.close()

The reconnect loop is mandatory. Indian broker WebSockets drop
regularly; an adapter without reconnection logic is unusable for live
trading.

## Step 5: Implement the Market Data Methods

`data.py`. The role is a `runtime_checkable` Protocol, so this class
implements it structurally — it inherits nothing but the shared base:

    from datetime import datetime

    from honba.adapters.capabilities import Capability
    from honba.adapters.errors import AdapterError
    from honba.adapters.models import (
        MarketDepth,
        StreamCallback,
        StreamMode,
        Subscription,
    )
    from honba.domain.bar import Bar
    from honba.domain.instrument import Instrument, InstrumentId
    from honba.domain.tick import QuoteTick

    from .http import MyBrokerHTTP
    from .websocket import MyBrokerWebSocket
    from .parsing import (
        parse_instrument,
        parse_quote,
        parse_depth,
        parse_bars,
        to_broker_interval,
    )


    class MyBrokerMarketData:
        """The MarketDataAdapter half of the adapter."""

        def __init__(self, http: MyBrokerHTTP, ws: MyBrokerWebSocket):
            self.http = http
            self.ws = ws
            self._subscriptions: set[str] = set()

        async def instruments(self) -> list[Instrument]:
            self.require_connected()
            self.require_capabilities(Capability.INSTRUMENT_MASTER)
            data = await self.http.get("/instruments")
            return [parse_instrument(row) for row in data["instruments"]]

        async def search_instruments(self, query: str) -> list[Instrument]:
            self.require_connected()
            needle = query.lower()
            return [
                instrument
                for instrument in await self.instruments()
                if needle in instrument.instrument_id.symbol.lower()
            ]

        async def quote(self, instrument_id: InstrumentId) -> QuoteTick:
            self.require_connected()
            self.require_capabilities(Capability.QUOTES)
            data = await self.http.get(f"/quote/{instrument_id.symbol}")
            return parse_quote(data, instrument_id)

        async def depth(self, instrument_id: InstrumentId, levels: int = 5) -> MarketDepth:
            self.require_connected()
            self.require_capabilities(Capability.DEPTH)
            data = await self.http.get(f"/depth/{instrument_id.symbol}", levels=levels)
            return parse_depth(data, instrument_id)

        async def historical_bars(
            self,
            instrument_id: InstrumentId,
            *,
            timeframe: str,
            start: datetime,
            end: datetime,
        ) -> list[Bar]:
            self.require_connected()
            self.require_capabilities(Capability.HISTORICAL_BARS)
            data = await self.http.get(
                "/historical",
                symbol=instrument_id.symbol,
                interval=to_broker_interval(timeframe),
                from_ts=start.timestamp(),
                to_ts=end.timestamp(),
            )
            return parse_bars(data, instrument_id, timeframe)

        async def subscribe(
            self,
            instruments: tuple[InstrumentId, ...],
            *,
            mode: StreamMode,
            callback: StreamCallback,
        ) -> Subscription:
            self.require_connected()
            # subscribe needs no Capability: the mode is the argument, so it
            # is checked against stream_modes instead.
            self.capabilities().require_stream_mode(mode)
            handle = await self.ws.subscribe(
                tuple(instruments), mode, callback
            )
            self._subscriptions.add(handle)
            return Subscription(
                id=handle, instruments=tuple(instruments), mode=mode
            )

        async def unsubscribe(self, subscription_id: str) -> None:
            self.require_connected()
            if subscription_id not in self._subscriptions:
                raise AdapterError(
                    f"unknown subscription {subscription_id!r}"
                )
            self._subscriptions.discard(subscription_id)
            await self.ws.unsubscribe(subscription_id)

`timeframe` is the canonical aggregation name (`"1m"`, `"1d"`); translate it
to the broker's code in `parsing.py` or `constants.py`, never in the engine.
An unknown instrument id is an `AdapterError`, not a `KeyError`.

## Step 6: Implement the Execution Methods

`execution.py`, against the `ExecutionAdapter` Protocol:

    from honba.adapters.capabilities import Capability
    from honba.adapters.errors import AdapterError
    from honba.adapters.models import (
        Funds,
        Holding,
        MarginReport,
        OrderReport,
        Product,
    )
    from honba.domain.instrument import InstrumentId
    from honba.domain.order import OrderIntent
    from honba.domain.position import Position
    from honba.domain.trade import Trade

    from .http import MyBrokerHTTP
    from .parsing import (
        parse_order_report,
        parse_position,
        parse_holding,
        parse_trade,
        parse_funds,
        parse_margin,
        to_broker_product,
    )


    class MyBrokerExecution:
        """The ExecutionAdapter half of the adapter."""

        def __init__(self, http: MyBrokerHTTP):
            self.http = http

        async def place_order(
            self,
            intent: OrderIntent,
            *,
            product: Product,
            client_order_id: str | None = None,
        ) -> OrderReport:
            self.require_connected()
            self.require_capabilities(Capability.PLACE_ORDER)
            # Refuse what we cannot express before spending a request.
            caps = self.capabilities()
            caps.require_product(product)
            caps.require_order_type(intent.order_type)
            caps.require_time_in_force(intent.time_in_force)
            payload = {
                "symbol": intent.instrument_id.symbol,
                "side": intent.side.value.upper(),
                "quantity": intent.quantity,
                "order_type": intent.order_type.value.upper(),
                "price": intent.price,
                "client_order_id": client_order_id,
            }
            resp = await self.http.post("/orders", json=payload)
            # parse_order_report maps the broker's rejection code onto
            # status=REJECTED with a reject_reason, so a refusal is data here,
            # not an exception.
            return parse_order_report(resp, client_order_id=client_order_id)

        async def modify_order(
            self,
            order_id: str,
            *,
            quantity: float | None = None,
            price: float | None = None,
            trigger_price: float | None = None,
        ) -> None:
            self.require_connected()
            self.require_capabilities(Capability.MODIFY_ORDER)
            if quantity is None and price is None and trigger_price is None:
                raise AdapterError("modify_order() needs a field to change")
            await self.http.post(f"/orders/{order_id}/modify", json={
                "quantity": quantity,
                "price": price,
                "trigger_price": trigger_price,
            })

        async def cancel_order(self, order_id: str) -> None:
            self.require_connected()
            self.require_capabilities(Capability.CANCEL_ORDER)
            await self.http.post(f"/orders/{order_id}/cancel", json={})

        async def cancel_all(
            self,
            *,
            instrument_id: InstrumentId | None = None,
            product: Product | None = None,
        ) -> None:
            self.require_connected()
            self.require_capabilities(Capability.CANCEL_ALL)
            await self.http.post("/orders/cancel-all", json={
                "symbol": instrument_id.symbol if instrument_id else None,
                "product": to_broker_product(product) if product else None,
            })

        async def order_status(self, order_id: str) -> OrderReport:
            self.require_connected()
            self.require_capabilities(Capability.ORDER_BOOK)
            resp = await self.http.get(f"/orders/{order_id}")
            return parse_order_report(resp)

        async def orders(self) -> list[OrderReport]:
            self.require_connected()
            self.require_capabilities(Capability.ORDER_BOOK)
            resp = await self.http.get("/orders")
            return [parse_order_report(o) for o in resp["orders"]]

        async def trades(self) -> list[Trade]:
            self.require_connected()
            self.require_capabilities(Capability.TRADE_BOOK)
            resp = await self.http.get("/trades")
            return [parse_trade(t) for t in resp["trades"]]

        async def positions(self) -> list[Position]:
            self.require_connected()
            self.require_capabilities(Capability.POSITIONS)
            resp = await self.http.get("/positions")
            return [parse_position(p) for p in resp["positions"]]

        async def holdings(self) -> list[Holding]:
            self.require_connected()
            self.require_capabilities(Capability.HOLDINGS)
            resp = await self.http.get("/holdings")
            return [parse_holding(h) for h in resp["holdings"]]

        async def funds(self) -> Funds:
            self.require_connected()
            self.require_capabilities(Capability.FUNDS)
            return parse_funds(await self.http.get("/funds"))

        async def margin(
            self, instrument_id: InstrumentId | None = None
        ) -> MarginReport:
            self.require_connected()
            self.require_capabilities(Capability.MARGIN)
            resp = await self.http.get(
                "/margin", symbol=instrument_id.symbol if instrument_id else None
            )
            return parse_margin(resp, instrument_id)

Two rules carry most of the weight here:

- **Unsupported is an exception.** A capability, product, order type or time
  in force the adapter does not declare raises `CapabilityError` before the
  request is sent. That is a configuration bug, so it must be loud.
- **Refusal is not an exception.** When the *broker* refuses an order, return
  an `OrderReport` with `status=REJECTED` and a `reject_reason`. The engine
  journals and reconciles that; raising would lose the refusal.

`client_order_id` is the caller's idempotency key — store it and return the
same order for a repeat instead of creating a second one.

Compose the two halves with the facade in `__init__.py`, and the single
object satisfies all three declarations:

    from honba.adapters.base import Adapter
    from honba.adapters.capabilities import AdapterCapabilities, Capability
    from honba.adapters.models import RunMode, SessionInfo

    from .config import MyBrokerConfig
    from .http import MyBrokerHTTP
    from .websocket import MyBrokerWebSocket
    from .data import MyBrokerMarketData
    from .execution import MyBrokerExecution


    class MyBrokerAdapter(Adapter, MyBrokerMarketData, MyBrokerExecution):
        name = "mybroker"

        def __init__(self, config: MyBrokerConfig):
            self.config = config
            self.http = MyBrokerHTTP(config)
            self.ws = MyBrokerWebSocket(config)
            MyBrokerMarketData.__init__(self, self.http, self.ws)
            MyBrokerExecution.__init__(self, self.http)
            self._connected = False
            self._caps = AdapterCapabilities(
                name="mybroker",
                exchanges=frozenset({"NSE"}),
                features=frozenset({Capability.QUOTES, Capability.ORDER_BOOK}),
            )

        def capabilities(self) -> AdapterCapabilities:
            return self._caps

        async def connect(self) -> SessionInfo:
            await self.http.authenticate()
            await self.ws.connect()
            self._connected = True
            return SessionInfo(user_id=self.config.client_id, mode=RunMode.LIVE)

        async def disconnect(self) -> None:
            await self.ws.close()
            await self.http.close()
            self._connected = False

        def is_connected(self) -> bool:
            return self._connected

        async def session(self) -> SessionInfo:
            self.require_connected()
            return SessionInfo(user_id=self.config.client_id, mode=RunMode.LIVE)

The two roles are Protocols, so there is nothing to register or inherit for
them: `isinstance(MyBrokerAdapter(config), MarketDataAdapter)` and
`isinstance(..., ExecutionAdapter)` are simply `True` because the methods
exist.

## Step 7: Parsing

`parsing.py` is where broker-specific JSON becomes the canonical types from
`honba.adapters.models` and `honba.domain`. This is where most bugs live, so
keep it small and test it thoroughly:

    from honba.domain.bar import Bar
    from honba.domain.instrument import Instrument, InstrumentId
    from honba.domain.tick import QuoteTick

    from .constants import MILLIS_TO_NANOS
    from .errors import ParseError


    def parse_quote(raw: dict, instrument_id: InstrumentId) -> QuoteTick:
        return QuoteTick(
            instrument_id=instrument_id,
            ts=raw["timestamp_ms"] * MILLIS_TO_NANOS,
            bid_price=float(raw["bid"]),
            ask_price=float(raw["ask"]),
            bid_size=float(raw.get("bid_qty", 0.0)),
            ask_size=float(raw.get("ask_qty", 0.0)),
        )


    def parse_bars(
        raw: list[dict], instrument_id: InstrumentId, timeframe: str
    ) -> list[Bar]:
        return [
            Bar(
                instrument_id=instrument_id,
                ts=row["timestamp_ms"] * MILLIS_TO_NANOS,
                open=float(row["open"]),
                high=float(row["high"]),
                low=float(row["low"]),
                close=float(row["close"]),
                volume=float(row["volume"]),
            )
            for row in raw
        ]

Three rules for parsing code:

1. **Emit the canonical types exactly.** Prices and quantities are `float`
   and timestamps are unix nanoseconds, because that is what the domain uses;
   the money-representation decision is tracked in the core. Field names
   must match (`bid_price`, not `bid`; `instrument_id`, not `instrument`).
2. **Always validate.** If a required field is missing, raise a descriptive
   error, do not return a partially-populated object.
3. **Always test.** Every parsing function gets a test with a real broker
   fixture.

## Step 8: Tests

`tests/test_parsing.py`:

    import pytest

    from honba.domain.instrument import InstrumentId
    from honba_mybroker.errors import ParseError
    from honba_mybroker.parsing import parse_quote

    from honba_adapters_shared.fixtures import load_fixture


    def test_parse_quote():
        raw = load_fixture("quote_reliance.json")
        quote = parse_quote(raw, InstrumentId("RELIANCE", "NSE"))
        assert quote.instrument_id.symbol == "RELIANCE"
        assert quote.bid_price == 2450.50
        assert quote.ask_price == 2450.75


    def test_parse_quote_missing_bid_raises():
        raw = load_fixture("quote_missing_bid.json")
        with pytest.raises(ParseError):
            parse_quote(raw, InstrumentId("RELIANCE", "NSE"))

The fixtures directory contains recorded broker responses. Save them
from your own API calls or from the broker's documentation examples.

Then run the shared contract suite, which is the check that actually makes
the adapter an adapter:

    from honba.adapters.contract import verify_adapter_contract

    from honba_mybroker import MyBrokerAdapter
    from honba_mybroker.config import MyBrokerConfig


    @pytest.mark.asyncio
    async def test_mybroker_satisfies_the_contract():
        await verify_adapter_contract(MyBrokerAdapter(MyBrokerConfig.paper()))

It is deterministic and pytest-free (it raises `ContractViolation` on the
first broken rule), and it leaves the adapter disconnected when it finishes.
The contract suite deliberately leaves stream delivery, reconnect and
data-gap markers to their own story, so cover those in the adapter's tests.

## Step 9: Registration

The registry lives in the core, in `honba.adapters.registry`. An installed
package advertises itself through the `honba.adapters` entry-point group in
its own `pyproject.toml`:

    [project.entry-points."honba.adapters"]
    mybroker = "honba_mybroker:MyBrokerAdapter"

Discovery is lazy, so importing `honba` never touches installed
distributions, and explicit registration always wins — a config file or a
test can override an installed adapter without uninstalling it. For a local
run without an install, register the factory directly:

    from honba.adapters.registry import register_adapter

    from honba_mybroker import MyBrokerAdapter

    register_adapter("mybroker", MyBrokerAdapter)

Also add the adapter to the CI workflow in `.github/workflows/ci.yml`
so its tests run:

    - run: pip install -e ./mybroker
    - run: pytest mybroker/

## Step 10: Documentation

Create `mybroker/README.md` with:

- Installation: `pip install -e ./mybroker`
- Configuration: required credentials and optional settings
- Authentication flow: how tokens are obtained and refreshed
- Known limitations: what the adapter does not support
- Example: a minimal script that connects, subscribes, and places an
  order

The README is the primary documentation users see. It should be
complete enough that someone can use the adapter without reading the
source.

## Testing Against a Live Broker

Live tests are gated by environment variables. In `conftest.py`:

    import os
    import pytest


    @pytest.fixture
    def live_credentials():
        client_id = os.environ.get("MYBROKER_CLIENT_ID")
        access_token = os.environ.get("MYBROKER_ACCESS_TOKEN")
        if not client_id or not access_token:
            pytest.skip("Live credentials not configured")
        return MyBrokerConfig(client_id=client_id, access_token=access_token)

Tests that need live credentials are marked `@pytest.mark.live` and
skipped by default. CI does not run them.

## Publishing

Once the adapter passes its tests and has been used for a week of
paper trading without issues:

1. Bump the version in `pyproject.toml`.
2. Update the adapter list in `honba-adapters/README.md`.
3. Open a pull request. The adapter repository maintains its own
   release cycle and does not depend on the core.

## Common Mistakes

### Inventing your own price type

The canonical types use `float` for prices and quantities and unix-nanosecond
integers for timestamps. Do not introduce `Decimal` or a wrapper type at the
boundary: the domain validates these values on construction, and a mismatch
fails the contract suite.

### Blocking the event loop

All I/O must be `async`. A synchronous HTTP call inside an `async`
method blocks the entire engine.

### Forgetting to close connections

Every `connect()` needs a matching `disconnect()`. The engine calls
`disconnect()` on shutdown, but a bug in `connect()` can leave a
dangling WebSocket.

### Not handling broker-side rate limits

Indian brokers rate-limit aggressively. The token-bucket helper is shared
plumbing owned by the core (`honba.adapters` shared helpers, E1-S5), so it
is not per-adapter code — do not reimplement it in your package.

### Assuming the order ID is stable

Most brokers return a broker-assigned order ID that differs from your
`client_order_id`. Preserve both. `client_order_id` is the caller's
idempotency key: a repeat with the same id must not create a second order.

### Raising on a rejected order

A broker refusal is an `OrderReport` with `status=REJECTED` and a
`reject_reason`, not an exception. Only `CapabilityError` (something the
adapter does not declare) and `AdapterError` (unknown id, transport failure)
are raised.

### Testing only the happy path

Most production bugs come from error paths — a rejected order, a
disconnected WebSocket, a malformed response. Test these explicitly
with fixtures.

## Checklist

Before opening a pull request:

- [ ] Package skeleton created with uniform structure
- [ ] `config.py` with Pydantic model and `SecretStr` for tokens
- [ ] `http.py` with retry logic for transient failures
- [ ] `websocket.py` with reconnection
- [ ] `data.py` implementing `instruments`, `search_instruments`, `quote`,
      `depth`, `historical_bars`, `subscribe`, `unsubscribe`
- [ ] `execution.py` implementing `place_order`, `modify_order`,
      `cancel_order`, `cancel_all`, `order_status`, `orders`, `trades`,
      `positions`, `holdings`, `funds`, `margin`
- [ ] Facade: `capabilities()`, `connect()`, `disconnect()`, `is_connected()`,
      `session()`
- [ ] `parsing.py` emitting the canonical field names (`float` prices,
      unix-nanosecond timestamps)
- [ ] Fixtures recorded for all message types
- [ ] `await verify_adapter_contract(MyBrokerAdapter(config))` passes
- [ ] Tests for parsing, data, execution, and error paths
- [ ] Entry point declared under `[project.entry-points."honba.adapters"]`
- [ ] Added to CI workflow
- [ ] README with installation, configuration, and example
- [ ] All tests pass
- [ ] `ruff check` and `mypy` pass
