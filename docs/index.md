# Honba

**AI-native trading and research platform for Indian markets.**

Honba is a Rust-powered engine with a Python control plane, purpose-built for
Indian equities, indices, index universes (NIFTY 50, BANK NIFTY, Alpha30),
mutual funds, and ETFs. It combines deterministic event-driven backtesting with
a three-tier optimization pipeline and an AI feedback loop that reads its own
research journal.

## Why Honba

Existing open-source trading platforms solve infrastructure problems well, but
none of them integrate coherently for the Indian market. Honba composes their
proven ideas into a single stack:

- **Deterministic engine** — same event flow in backtest and live, so strategies
  never need to be reimplemented between research and production.
- **Three-tier backtesting** — a fast vectorized pre-filter, a realistic
  event-driven simulator, and a concurrent parameter sweep for optimization.
- **Indian market first-class** — NSE/BSE calendar, STT/GST/stamp-duty cost
  model, dynamic index constituents, weekly options expiry cycles.
- **AI feedback loop** — a research journal that an LLM reads and refines,
  Monte Carlo and walk-forward validation automated on every strategy, and an
  MCP gateway for external agents.

## Architecture at a Glance

    ┌────────────────────────────────────────────────────┐
    │ AI feedback layer  (LLM research, RL training, MCP) │
    ├────────────────────────────────────────────────────┤
    │ Strategy + optimization  (vectorized → event → async) │
    ├────────────────────────────────────────────────────┤
    │ Data + execution  (adapters, catalog, India module)  │
    ├────────────────────────────────────────────────────┤
    │ Rust core  (Nautilus-style determinism, Barter async) │
    └────────────────────────────────────────────────────┘

## Where to Start

1. [Installation](getting-started/installation.md) — set up Rust, Python, and
   the broker adapters.
2. [Your First Backtest](getting-started/first-backtest.md) — run a moving
   average strategy on NIFTY 50.
3. [Architecture Overview](architecture/overview.md) — understand why the
   crate hierarchy is what it is.
4. [Indian Market Calendar](india/market_calendar.md) — the market-specific
   details every strategy must respect.

## Status

Honba is under active development. The crate skeleton is complete and the
dependency hierarchy is enforced in CI. The `honba-messages`, `honba-entities`,
and `honba-market` crates are the current focus.
