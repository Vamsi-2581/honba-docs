# Message Flow

This document traces a single market tick from a broker WebSocket to a
strategy decision and back out as an order. Understanding this flow is
necessary for debugging any behaviour that crosses layer boundaries.

## The Path of a Tick

    ┌──────────────┐
    │ Broker WS    │  1. Raw JSON arrives
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ Adapter      │  2. Parse → QuoteTick
    │ (Python)     │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ MessageBus   │  3. Publish to subscribers
    │ (Rust)       │
    └──────┬───────┘
           │
           ├────► Cache        4a. Update last price
           │
           ├────► BarAggregator 4b. Roll into a 1-min bar
           │
           └────► Strategy      4c. Deliver to on_quote()
                     │
                     ▼
              ┌──────────────┐
              │ Decision     │  5. Strategy decides to buy
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ RiskEngine   │  6. Pre-trade checks
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ Execution    │  7. SubmitOrder command
              │ Client       │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ Adapter      │  8. HTTP POST to broker
              │ (Python)     │
              └──────────────┘

## The Event Loop

The engine is a single-threaded event loop. Events are processed in the
order they are queued. This is the determinism guarantee: given the same
sequence of input events, the engine produces the same sequence of output
events, every time.

    loop {
        match event_bus.next() {
            Event::Quote(q)   => dispatcher.on_quote(q),
            Event::Trade(t)   => dispatcher.on_trade(t),
            Event::Bar(b)     => dispatcher.on_bar(b),
            Event::Order(o)   => dispatcher.on_order(o),
            Event::Fill(f)    => dispatcher.on_fill(f),
            Event::Timer(t)   => dispatcher.on_timer(t),
            Event::Stop       => break,
        }
    }

In backtest mode, the clock is simulated: the event loop advances a
`UnixNanos` cursor and pulls the next event from the data catalog. In live
mode, the clock is real: the event loop blocks on the next incoming
message from any subscribed source.

The strategy code is identical in both modes. This is the entire point of
the deterministic design.

## Message Types

Every message implements a common envelope:

    pub struct Message {
        pub header: MessageHeader,
        pub payload: MessagePayload,
    }

    pub struct MessageHeader {
        pub correlation_id: Option<Uuid>,
        pub timestamp: UnixNanos,
        pub source: VenueId,
    }

    pub enum MessagePayload {
        Quote(QuoteTick),
        Trade(TradeTick),
        Bar(Bar),
        OrderBookDelta(OrderBookDeltas),
        Order(OrderEvent),
        Fill(FillEvent),
        Custom(Vec<u8>),
    }

The `correlation_id` ties a command (submit order) to its subsequent
events (order accepted, partially filled, filled). This is how a strategy
knows which of its own orders an event refers to, without the engine
having to maintain per-strategy routing tables.

## Bar Aggregation

Quotes and trades arrive at tick granularity. Strategies usually care about
bars. The engine has a `BarAggregator` that subscribes to the raw streams
and emits `Bar` events at the requested intervals.

    // Strategy requests 1-minute bars for a set of instruments
    self.subscribe_bars(BarType::new(
        BarSpecification::minute(1),
        AggregationSource::Trade,
        PriceType::Last,
    ), &instruments);

Aggregation happens **in the engine**, not in the strategy. A strategy that
subscribes to 1-minute bars receives them fully-formed; it does not see
ticks and it does not aggregate. This keeps aggregation logic consistent
across every strategy and avoids a common source of subtle bugs.

## The Cache

Each strategy has a **context-aware cache** — an in-memory store of the
data it has subscribed to. When a strategy calls `self.last_price(instr)`,
it reads from the cache, not from the event stream.

    // Inside a strategy
    fn on_bar(&mut self, bar: &Bar) {
        let last = self.cache.last_price(&bar.instrument);
        // ...
    }

The cache is updated by the engine as events flow through. This design
comes from WonderTrader, and its purpose is to let strategy code be
written as if all the data it needs is always available, without the
strategy having to manually maintain state.

The cache is scoped per strategy. Two strategies in the same backtest do
not share cache state, which means a bug in one strategy cannot corrupt
another.

## Order Lifecycle

An order emits a sequence of events, all tied by `correlation_id`:

    SubmitOrder(correlation_id=abc) ───►
                                        │
    OrderAccepted(correlation_id=abc) ◄─┤  (broker acknowledges)
                                        │
    OrderPartiallyFilled(abc, 100)    ◄─┤
                                        │
    OrderPartiallyFilled(abc, 250)    ◄─┤
                                        │
    OrderFilled(abc, 500)             ◄─┘  (complete)

If the strategy also set a `client_order_id`, that ID is preserved through
every event. This is useful when the broker assigns its own order ID (as
most Indian brokers do) and you need to correlate back to your own record.

## Timers

Strategies can schedule timers for events that are not triggered by market
data — for example, "close all positions at 15:15 IST".

    self.schedule_at(
        session_close_minus(Duration::minutes(15)),
        TimerCallback::CloseAll,
    );

In backtest, timers fire on the simulated clock. In live, they fire on the
real clock. The strategy does not need to know which mode it is in.

## Backtest vs Live Differences

The engine is designed so that **the only difference is the clock source
and the data source**. Everything else — dispatch order, cache updates,
bar aggregation, risk checks, order lifecycle — is identical.

| Concern | Backtest | Live |
|---|---|---|
| Clock | Simulated, advanced by data | Real wall clock |
| Data source | Parquet catalog | Broker WebSocket |
| Order routing | Simulated exchange | Broker HTTP API |
| Fill modelling | Configurable (latency, slippage) | Real broker fills |
| Determinism | Full | None (network jitter) |

If a strategy behaves differently between backtest and live in any way
other than fill quality, that is a bug in the engine, not in the strategy.

## Debugging

The engine emits a structured log of every event. In backtest mode, this
log is retained and can be replayed:

    Engine::new(config)
        .with_event_log("events.parquet")
        .run(strategy);

Then:

    let replay = EventLog::load("events.parquet")?;
    let engine = Engine::from_log(replay);
    engine.run_with_breakpoint(|event| {
        if event.is_order_submitted() {
            inspect(&event);
        }
    });

This is the mechanism for reproducing bugs that only appear in specific
market conditions.
