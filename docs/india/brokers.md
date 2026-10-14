# Broker Adapters

Honba connects to Indian brokers through adapters. Each adapter
implements the same contract — the `Adapter` facade plus the
`MarketDataAdapter` and `ExecutionAdapter` role protocols — but the
underlying broker APIs differ in authentication, symbol format,
data granularity, rate limits, and regulatory constraints.

This document summarises the practical differences.

*Last verified: 2026-09-30. Broker APIs change frequently; verify against
the broker's own documentation before relying on any detail below.*

## Adapter Status

Two adapters are implemented and contract-certified: **Dhan** and **Zerodha**.
Angel One, Fyers, Upstox, Kotak, IIFL and Motilal Oswal exist as package
scaffolds with the uniform layout only: no implementation, no entry point,
and naming one in a run config fails with `AdapterNotFound`. The table below
describes the brokers' own APIs so you can plan; it does not mean Honba can
trade through the scaffolded ones yet. Status and setup are in
[Adapter Configuration](adapter_configuration.md).

| Broker | Honba adapter |
|---|---|
| Zerodha | Implemented |
| Dhan | Implemented |
| Angel One, Fyers, Upstox, Kotak, IIFL, Motilal Oswal | Scaffold only |

## The Primary Brokers

The brokers most commonly used by algorithmic traders in India:

| Broker | API | Auth | Sandbox | Market data |
|---|---|---|---|---|
| Zerodha | Kite Connect | OAuth (daily) | No | Paid (₹500/mo) |
| Dhan | DhanHQ | Token | **Yes** | Paid |
| Angel One | SmartAPI | TOTP + API key | No | Included |
| Fyers | API v3 | OAuth | No | Included |
| Upstox | Uplink | OAuth (daily) | **Yes** | Included |
| Kotak | Kotak Neo | OAuth | No | Included |

## Dhan Adapter

Dhan (`honba-dhan`, `DhanAdapter`, registered as `dhan`) declares NSE, BSE and
MCX; delivery, intraday and carry products; market, limit, stop-limit and
stop-market orders; and `DAY` and `IOC` time in force. `GTC`, `FOK`, `GTD` and
trailing stops are not declared and raise `CapabilityError` before any network
call, which now includes a time in force with no wire mapping.

Dhan sends the order validity that matches the intent: `DAY` as `"DAY"` and
`IOC` as `"IOC"`. Earlier versions sent `DAY` for every order, so an IOC intent
was silently placed as a day order. The `"IOC"` wire value has not been
verified against the live DhanHQ API; see
[Adapter Configuration](adapter_configuration.md#dhan-time-in-force).

## Zerodha Adapter

Zerodha is a **Python adapter only**: the `honba-zerodha` package in
`honba-adapters/zerodha` (`honba_zerodha`). There is no Rust Zerodha crate in
the core `honba` workspace any more, and the Kite Connect Rust client
(`kiteconnect-rs`) is a protocol reference, not a dependency. Like every other
adapter it implements the `Adapter` facade plus the market-data and execution
role protocols and is discovered through the adapter registry.

What it covers:

- **Order vocabulary.** Products `CNC`, `MIS` and `NRML` (delivery, intraday,
  carry); order types market, limit, stop-limit (`SL`) and stop-market
  (`SL-M`). Trailing stops are not listed.
- **Time in force: `DAY` and `IOC` only.** Kite has no other validity for
  regular orders; any other value raises `CapabilityError` before the network
  is touched.
- **Whole quantities only.** A fractional quantity is rejected with an
  `AdapterError` locally, as are non-positive prices and trigger prices.
- **Order tags.** A `client_order_id` is sent as the Kite `tag`: ASCII
  alphanumerics only, at most 20 characters (`HONBA` if nothing is left).
- **Reconciliation of ambiguous submits.** If an order submission fails with a
  transport error or a rate-limit error, the outcome is unknown, so the
  `client_order_id` is marked ambiguous. The next `place_order` with the same
  id first looks for an order carrying that tag (`GET /orders`) and adopts it
  instead of sending a duplicate. A repeated `client_order_id` otherwise
  returns the cached report. A Kite refusal comes back as a `REJECTED`
  `OrderReport`, not an exception.
- **`poll_fills()`.** Reads the broker's trades and returns the fills of orders
  this adapter placed or adopted that have not been reported yet,
  de-duplicated by Kite trade id; phantom rows (non-positive size or price,
  unparseable timestamp) are skipped. This is the pull path for fills.
- **KiteTicker binary decoder.** The websocket feed client decodes Kite's
  binary frames (LTP, quote and full-depth packets; one-byte heartbeats yield
  nothing; malformed or unknown-length packets raise a typed ticker error) and
  turns them into Honba quote and trade ticks. A stateful filter reports a
  trade only when volume strictly increased, because Kite re-sends the same
  snapshot whenever anything in the packet changes. Stream modes are LTP, quote
  and depth.
- **Account reads.** Orders, trades, positions, holdings, funds and margin.

## The Static IP Requirement

SEBI requires a registered static IP for API-based order placement. This
is the single biggest practical constraint on live algorithmic trading
in India.

- **Upstox and Dhan** offer sandbox environments that sidestep the
  static IP requirement for testing.
- **Angel One and Zerodha** do not, which means their write paths can
  be tested only after registering a static IP with the broker.[reference:24]

For a research-first platform like Honba, this means: **do backtesting
and paper trading with any adapter, but only go live after registering a
static IP with your broker**. The Sandbox adapter, which simulates
execution locally over real market data, does not require a static IP.

## Authentication Differences

Each broker handles auth differently, and this is the most common source
of adapter bugs:

- **Zerodha Kite Connect (Python adapter):** OAuth flow with a daily access token
  generated from a login redirect. The token expires every morning,
  which is inconvenient for headless systems but is what Kite
  requires.
- **Dhan:** Token-based. The access token can be long-lived, which
  makes Dhan the simplest adapter to run unattended.
- **Angel One SmartAPI:** TOTP-based two-factor authentication. The
  adapter generates TOTP codes from a shared secret.
- **Fyers:** OAuth v3 with token refresh.
- **Upstox:** OAuth with a daily browser login and no headless
  refresh — this is Upstox's main limitation for automated systems.

In every case, credentials are read from the environment (`DHAN_*`,
`ZERODHA_*`); see [Adapter Configuration](adapter_configuration.md).

Honba's adapter facade abstracts the auth flow behind a single
`connect()` coroutine returning a `SessionInfo`, but the token refresh
interval is broker-specific and configured per adapter.

## Data Granularity

| Broker | Live depth | Historical (intraday) | Historical (daily) |
|---|---|---|---|
| Zerodha | 5 levels | 1-min (paid) | From inception |
| Dhan | 5, 20, 200 levels | 1-min, 5-year window | From inception |
| Angel One | 5 levels | 1-min | From inception |
| Fyers | 5, 50 on separate feed | 1-min from mid-2017 | From inception |
| Upstox | 5, 30 on paid tier | 1-min | From 2000 |

[reference:25]

For options strategies, none of these brokers provide sufficient
historical option data for serious backtesting. You must supplement
with NSE bhavcopy archives or a paid vendor.

## Order Update Notifications

Brokers deliver order status updates through two mechanisms: **postback
webhooks** and **WebSocket streams**. Postback is more reliable but
requires a publicly reachable endpoint; WebSocket is faster but can drop
messages.

| Broker | Postback | WebSocket |
|---|---|---|
| AliceBlue | Yes | Yes |
| Upstox | Yes | Yes |
| Fyers | Yes | Yes |
| Dhan | Yes | Yes |
| Zerodha | — | Yes |
| Angel One | — | Yes |

[reference:26]

Honba's adapters prefer postback when available and fall back to
WebSocket. If neither delivers an update within a configurable timeout,
the adapter polls the broker's order status endpoint.

## Brokerage Defaults

Honba's cost model ships with per-broker brokerage defaults, current as
of the last verification date:

| Broker | Equity delivery | Equity intraday | F&O |
|---|---|---|---|
| Zerodha | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Dhan | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Angel One | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Fyers | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Upstox | ₹0 | ₹20 or 0.05% | ₹20 per order |

These defaults are configurable. Verify against the broker's current
pricing page before relying on them — brokerage schedules change.

## Adapter Configuration

    # configs/live/dhan_paper.toml
    [adapter]
    name = "dhan"
    mode = "paper"                # or "live"
    client_id = "${DHAN_CLIENT_ID}"
    access_token = "${DHAN_ACCESS_TOKEN}"

    [adapter.data]
    feed = "market"               # or "full" for depth
    reconnect_attempts = 5
    heartbeat_seconds = 30

    [adapter.execution]
    order_timeout_seconds = 10
    max_retries = 3

Credentials are read from environment variables, never stored in the
config file. A live adapter built without them raises `AuthenticationError`
naming the missing variables; it does not fall back to paper. The variable
names, the `<PREFIX>MODE` switch and the cache directory (`HONBA_CACHE_DIR`)
are described in [Adapter Configuration](adapter_configuration.md).

## Paper vs Live

There are two ways to run without a live broker. For adapter-level paper mode,
build the adapter explicitly in paper mode (`DhanAdapter(mode="paper")`,
`ZerodhaAdapter(mode="paper")`, `XConfig.paper()` or `<PREFIX>MODE=paper`); it
needs no credentials and uses an in-memory transport. Bare `DhanAdapter()` and
`ZerodhaAdapter()` no longer default to paper.

Paper trading in the engine sense is the **Sandbox adapter** (E3-S7). It implements the same
adapter contract over the real simulator, so a strategy cannot tell paper
from live, and it is the recommended way to validate a strategy before
going live. Configure it like any other adapter, by name under `[adapter]`,
with market data coming from a broker adapter and fills simulated locally.

Paper trading uses real market data but simulates fills locally. This
is identical to the event-driven backtest simulator, so a strategy that
performs as expected in backtest will perform as expected in paper.
The only difference between paper and live is the fill quality.

## Rate Limits

All Indian broker APIs impose rate limits. The limits differ:

- Zerodha: 3 requests/second for order placement, 10/second for market
  data (varies by plan).
- Dhan: 25 requests/second.
- Angel One: 3 requests/second for historical data.

Rate limiting is shared plumbing owned by the core rather than per-adapter
code. The adapter contract publishes no rate-limit accessor today (E1-S5),
so an adapter throttles its own transport until it does.

## Adding a New Broker

If your broker is not implemented (that includes the scaffolded ones):

1. Start from `honba-adapters/dhan/` and keep the uniform structure
   (`config.py`, `data.py`, `execution.py`, `instruments.py`, `websocket.py`,
   `http.py`, `parsing.py`, `constants.py`).
2. Implement the `Adapter` facade plus the `MarketDataAdapter` and
   `ExecutionAdapter` role protocols, declaring exactly what the broker supports.
3. Build the honba-to-wire tables with `WireMap` and check them with
   `assert_covers`.
4. Record fixtures and add `tests/fixtures/<name>/routes.toml`; the contract
   suite discovers the adapter from it.
5. Advertise the package through the `honba.adapters` entry-point group.

The checklist, with details, is in
[Adapter Configuration](adapter_configuration.md#adding-a-broker). The adapter
interface is documented in [Adapter Interface](../architecture/adapter_interface.md).

## Source of Truth

- Broker API documentation (linked from each broker's developer portal).
- SEBI circular on static IP requirements for API trading.
- Broker pricing pages for brokerage schedules.
