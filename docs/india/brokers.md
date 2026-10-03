# Broker Adapters

Honba connects to Indian brokers through adapters. Each adapter
implements the same contract — the `Adapter` facade plus the
`MarketDataAdapter` and `ExecutionAdapter` role protocols — but the
underlying broker APIs differ in authentication, symbol format,
data granularity, rate limits, and regulatory constraints.

This document summarises the practical differences.

*Last verified: 2026-09-30. Broker APIs change frequently; verify against
the broker's own documentation before relying on any detail below.*

## The Six Primary Adapters

Honba ships adapters for the brokers most commonly used by algorithmic
traders in India:

| Broker | API | Auth | Sandbox | Market data |
|---|---|---|---|---|
| Zerodha | Kite Connect | OAuth (daily) | No | Paid (₹500/mo) |
| Dhan | DhanHQ | Token | **Yes** | Paid |
| Angel One | SmartAPI | TOTP + API key | No | Included |
| Fyers | API v3 | OAuth | No | Included |
| Upstox | Uplink | OAuth (daily) | **Yes** | Included |
| Kotak | Kotak Neo | OAuth | No | Included |

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

- **Zerodha Kite Connect:** OAuth flow with a daily access token
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
config file. The `.env.example` in the scaffold lists the variables
each adapter expects.

## Paper vs Live

Paper trading is the **Sandbox adapter** (E3-S7). It implements the same
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

If your broker is not in the list above:

1. Create a new directory in `honba-adapters/` following the uniform
   structure (`config.py`, `data.py`, `execution.py`, `instruments.py`,
   `websocket.py`, `http.py`, `parsing.py`).
2. Implement the `Adapter` facade plus the `MarketDataAdapter` and
   `ExecutionAdapter` role protocols.
3. Add fixtures and tests, including the shared contract suite.
4. Advertise the package through the `honba.adapters` entry-point group.

The adapter interface is documented in
[Adapter Interface](../architecture/adapter_interface.md).

## Source of Truth

- Broker API documentation (linked from each broker's developer portal).
- SEBI circular on static IP requirements for API trading.
- Broker pricing pages for brokerage schedules.
