# Order Reject and Cancel Contract

How an execution port reports an order, or part of one, that will never fill. The contract is
the same in Rust and Python. Source of truth: ADR 0008 (strategy contract parity), decision 11
(Python) and decision 13 (Rust), in the `honba` repository.

## Value object: `OrderRejection`

| Field | Rust (`honba_engine::OrderRejection`) | Python (`honba.strategies.execution.OrderRejection`) |
|---|---|---|
| order | `order_id: OrderId` | `order_id: str` (must not be empty) |
| what will not fill | `instrument_id`, `side`, `quantity` | `intent: OrderIntent` (carries the quantity) |
| why | `reason: String` | `reason: str` |
| port time | `ts: UnixNanos` | `ts: int` (unix ns, default 0) |
| kind | `is_cancelled()`, derived from `reason == "cancelled"` (no field) | `cancelled: bool` (default `False`) |

- The quantity is the **unfilled remainder**, the amount the strategy context releases. A partly
  filled order reports only what is left.
- Rust carries instrument, side and quantity instead of an `OrderIntent` because `OrderIntent`
  lives in `honba-strategy`, above the port crate (`honba-engine`).
- Constructors in Rust: `OrderRejection::rejected(..)` and `OrderRejection::cancelled(..)`
  (reason is the constant `OrderRejection::CANCELLED`, `"cancelled"`).
- `cancelled` cannot contradict `reason`. Rust has no `cancelled` field: `is_cancelled()` is derived
  from `reason == OrderRejection::CANCELLED`. Python keeps the `cancelled` field and raises
  `ValueError` when it disagrees with `reason == CANCELLED_REASON` (`"cancelled"`): a cancellation
  must pass `cancelled=True`, and a venue rejection must not use the reason `"cancelled"`.
- Reason strings shared by both languages: `insufficient_funds`, `no_position`, `cancelled`.
- Wire event types when logged: `order_rejected`, `order_cancelled`.

## Port surface

| | Rust `ExecutionEngine` | Python `ExecutionPort` |
|---|---|---|
| submit | `submit(order)` | `submit(order_id, intent, ts)` |
| fills | `drain_fills()` | `drain_fills()` |
| rejections (optional) | `drain_rejections() -> Result<Vec<OrderRejection>>`, default returns nothing | `drain_rejections() -> list[OrderRejection]` |
| cancel (optional in Python) | `cancel(order_id, now)` | `cancel(order_id)` |

Python ports without the optional methods keep working through the shims
`drain_port_rejections` and `cancel_order`; `BaseExecutionPort` supplies inert defaults.

### Cancel semantics

- An engine holding the order reports the unfilled remainder as a rejection with
  `is_cancelled()` true (Python: `cancelled == True`). Cancelling an unknown or finished order is a no-op.
- The cancellation is stamped with the **time of the cancel**, not the order's original
  `ts_event` (ADR 0008, decision 13 addendum). Rust `ExecutionEngine::cancel(order_id, now)` takes
  that time: `Engine` passes its clock and `StrategyRunner::cancel` passes the `ts_init` of the
  latest event it saw (0 before any event). Python `NextOpenExecution` uses the session time.
  Rejections raised by the venue at submit or fill still carry the order's `ts_event`.
- `cancel` need not drain fills first. The remainder excludes anything already filled, so the
  context's pending quantity only decreases by `filled + released`, which sums to the ordered
  quantity in either booking order. The hard rule: **a remainder must never overlap a fill**
  (the same quantity may not appear in both).
- Once every working order is cancelled, `filled + released == ordered` for each order.

## Runner behavior

For each event the runner drains fills first, then rejections (runner step 5). Each rejection is
released in the strategy context and recorded: Rust `StrategyRunner::order_rejections()`, Python
`RunResult.order_rejections`. `StrategyRunner.cancel(id)` (Rust: `cancel(id)`) asks the port to
cancel and books what it reports immediately.

When the port fails a submit mid-drain, the runner releases the failed intent and every drained
intent not yet sent (their instruments no longer stay busy). When `on_fill` or fill booking fails,
the other drained fills are still booked and recorded and rejections are still released. The first
error is returned afterwards, and an `Err` from `on_event` is terminal for the run.

There is no `on_order_rejected` hook. A strategy sees a rejection only indirectly: the released
quantity makes `ctx.busy(instrument)` false again (a held order stays busy until cancelled).
`on_intent_rejected` (Rust only) concerns invalid intents, not port rejections.

## Reference engine: `ScriptedExecution`

`honba_sim::ScriptedExecution::new(price)` fills at one fixed price and stamps fills and
venue rejections with the order's `ts_event` (a cancel is stamped with the cancel time). Per order id, `.with(order_id, Behavior)` scripts:

| `Behavior` | Effect |
|---|---|
| `Fill` (default for an unscripted id) | fill in full |
| `Reject { reason }` | refuse the whole order |
| `Partial { filled, reason }` | fill `filled`, reject the remainder |
| `Hold` | keep the order working until cancelled |

Validation:

- At submit, an order without a side (`NoOrderSide`) and an order id that is already working
  (held, not yet filled, rejected or cancelled) are refused: Rust `ScriptedExecution::submit`
  returns an error, Python `NextOpenExecution.submit` raises `ValueError`. An id may be reused once
  it is no longer working.

- `Partial.filled` must be positive and finite; `.with(..)` panics otherwise (the script is a
  test fixture).
- `filled >= quantity` is only known at submit and returns an error there.
- A later `.with` for the same id replaces the earlier entry.

## Conformance

- Vectors: `schema/conformance/order_rejections.json` (8 scenarios, including
  `reject_whole_order`, `partial_fill_rejects_the_remainder`, `sell_without_position`,
  `cancel_working_order`, `late_cancel_is_stamped_with_the_cancel_time`, `held_order_stays_busy`, `cancel_finished_or_unknown_is_a_no_op`,
  `release_is_per_instrument`).
- Run by `crates/honba-strategy/tests/order_rejections.rs` (Rust) and
  `python/tests/integration/test_order_rejections_conformance.py` (Python).
- Every core port also runs a contract test: Rust `crates/honba-sim/tests/execution_contract.rs`,
  Python `python/tests/integration/test_execution_port_contract.py`.
- Not exposed through `honba._honba`: Python already has `OrderRejection` and
  `RunResult.order_rejections`, and the `run_strategy` reference engine never rejects.
