# Strategy Styles: Event-Driven, Declarative and Portfolio

Honba offers three ways to write a strategy. They are layers over the same
event flow, so a strategy written in any of them runs unchanged in backtest,
paper and live. Pick the highest layer that expresses your idea; drop down a
layer only when it gets in the way.

| Layer | You write | The framework does | Use it when |
|---|---|---|---|
| `Strategy` (event-driven) | `on_start`, `on_bar`, `on_fill`, ... and your own orders | Delivers events, tracks positions | You need full control: custom order flow, multi-leg logic, state machines |
| `DeclarativeStrategy` | Per-instrument rules: `should_long`, `should_short`, `go_long`, `go_short`, `should_exit` | Sends the entry, places the protective stop and target after the fill | One signal per instrument, entry plus stop/target, long and/or short |
| `TargetWeightStrategy` | `universe()` and optionally `target_weights()` | Values the portfolio and sizes, trims, exits and tops up | You know what the portfolio should look like, not how to get there |
| `PortfolioStrategy` | Four parts: universe, selector, weighting, schedule | Everything `TargetWeightStrategy` does, plus selection and rebalance timing | A standard rebalanced portfolio; configurable from `config.toml` |

`DeclarativeStrategy` and `TargetWeightStrategy` both extend `Strategy`, and
`PortfolioStrategy` extends `TargetWeightStrategy`. Overriding `on_bar` in
either facade bypasses it entirely.

## Layer 1: Event-Driven `Strategy`

The base class. You receive events and decide:

    from honba.entities.bar import Bar
    from honba.strategies.base import Strategy


    class Momentum(Strategy):
        name = "momentum"

        def on_bar(self, bar: Bar) -> None:
            if not self.busy(bar.instrument_id) and self.position(bar.instrument_id) == 0:
                self.buy(bar.instrument_id, 10)

`buy`, `sell` and `submit(OrderIntent)` queue orders; `position(id)` is the signed
quantity and `busy(id)` is true while an order for that instrument is unfilled.

## Layer 2: `DeclarativeStrategy`

Per-instrument rules instead of an `on_bar` body:

    from honba.entities.bar import Bar
    from honba.strategies import DeclarativeStrategy, Entry


    class Breakout(DeclarativeStrategy):
        name = "breakout"

        def should_long(self, bar: Bar) -> bool:
            return bar.close > 100.0

        def go_long(self, bar: Bar) -> Entry:
            return Entry(quantity=10, stop_loss=bar.close * 0.95, take_profit=bar.close * 1.10)

        def should_exit(self, bar: Bar) -> bool:
            return bar.close < 90.0

Evaluation order for each bar, for that bar's instrument only (no state is
shared between instruments):

1. **busy** (an unfilled order exists): nothing happens.
2. **flat**: `should_long`, then `should_short` (long wins ties). On `True` the
   matching `go_long` / `go_short` returns an `Entry`, and a market order is sent.
3. **in a position**: `should_exit` returning `True` sends a market order that
   flattens it.

After the *entry fill*, protective orders are placed, sized to the filled
quantity (so partial fills are protected as they arrive): for a long, a
stop-market sell at `stop_loss` and a limit sell at `take_profit`; for a short,
a stop-market buy and a limit buy. `Entry` validates its fields (`quantity`
finite and > 0; levels finite and > 0 when given), and levels are checked
against the bar close when the entry is decided: for a long `stop_loss < close
< take_profit`, for a short the reverse. Otherwise `ValueError` is raised.

If you override `on_fill`, call `super().on_fill(fill)` or no protective orders
are placed. Warmup is the runner's job (`Strategy.warmup_bars`).

### Jesse comparison

| Jesse | Honba | Notes |
|---|---|---|
| `should_long()` | `should_long(bar)` | Default `False`; receives the bar instead of reading `self.price` |
| `should_short()` | `should_short(bar)` | Default `False` |
| `go_long()` (sets `self.buy`, `self.stop_loss`, `self.take_profit`) | `go_long(bar) -> Entry(quantity, stop_loss, take_profit)` | Returns a value object; nothing is assigned on `self` |
| `go_short()` | `go_short(bar) -> Entry(...)` | Same |
| `update_position()` | `should_exit(bar)` | Returning `True` flattens at market; no in-place stop edits |
| `should_cancel_entry()` | not provided | Gate on `self.busy(instrument_id)` |
| `self.price`, `self.close` | the `bar` argument | |
| `self.stop_loss`, `self.take_profit` | `Entry.stop_loss`, `Entry.take_profit` | Placed after the entry fill, not with it |

What Honba does differently:

- **A `busy()` guard.** While any order for the instrument is unfilled, the
  rules are not evaluated. This prevents stacking duplicate entries, but see
  the limitation below.
- **Stops via `Entry`.** Protective levels are declared up front and validated;
  the framework places them once the entry has filled, sized to what filled.
- **Pure facade.** It submits ordinary `OrderIntent`s through the strategy
  context, so there is no separate execution path for declarative strategies.
- **Per-instrument evaluation.** A multi-instrument strategy gets one call per
  instrument bar.

### OCO protective orders

The stop and the take-profit placed for one entry fill form a One-Cancels-Other
group (`oco = True`, the default; set `oco = False` on your class for the old
behaviour). `DeclarativeStrategy` manages the group itself through
`self.cancel(intent)`, which the runner relays to the execution port:

- when one leg fills in full, the other is cancelled;
- when the position returns to flat, any leftover protective order is cancelled;
- `should_exit` is consulted while only protective orders rest: it cancels them,
  then sends the market exit. Any other unfilled order (an entry or exit in
  flight) still blocks the rules, so evaluation step 1 becomes "busy with
  something other than resting protectives".

A partial fill of one leg does not resize its sibling (there is no order-modify
support yet): the sibling stays at full size until its leg fills completely or
the position is flat. Each entry fill, including each partial entry fill, is
its own group. Cancel is a request: if both legs fill in the same simulator
step, the second fill is not undone. Execution models that fill every order
immediately (`BarCloseFills`) or reject non-market orders (`NextOpenExecution`)
never leave a pair resting, so OCO matters with `L2Execution` and in live trading.

Trailing stops (`OrderType.TRAILING_STOP`) are not used by this class; see
[Position Management](position_management.md#trailing-stop).

## Layer 3: `TargetWeightStrategy`

Declare what the portfolio should look like:

    from honba.entities.instrument import InstrumentId
    from honba.strategies import TargetWeightStrategy


    class TwoNames(TargetWeightStrategy):
        name = "two_names"
        rebalance_days = 21          # trading days between rebalances

        def universe(self):
            return [InstrumentId("TCS", "NSE"), InstrumentId("INFY", "NSE")]

`target_weights()` defaults to equal weight; override it to return a
`{InstrumentId: weight}` mapping (non-negative, sum at most 1; a negative
weight, a non-finite one or a total above 1 raises `ValueError`).

On a rebalance bar the framework values the portfolio (cash plus open
positions at the last close), sizes each name to `value * allocation * weight`
in whole shares, and submits market orders in this order: exits (held names
with weight 0), then trims (sells), then top-ups and entries (buys), each group
in symbol order. Busy or unpriced names are skipped and differences under one
share are ignored. `universe()` is re-read at every rebalance, so membership
may change; additions and removals are logged. Days come from bar event time
(UTC), never the wall clock. Long-only for now.

The default `should_rebalance` fires on the first bar once the whole universe
has prices (or on the next calendar day), then every `rebalance_days` trading
days (default 15). `allocation` (default 0.98) is the fraction of portfolio
value deployed.

## Layer 4: `PortfolioStrategy`

The same engine, with universe, selection, weighting and timing as swappable
parts, or as a TOML-friendly `params` dict. See
[Portfolio Construction](portfolio_construction.md).

## Choosing a Layer

- One instrument or one signal per instrument, with a stop and a target:
  `DeclarativeStrategy`.
- "Hold these names at these weights": `TargetWeightStrategy`.
- "Hold the top N of a universe, weighted a certain way, rebalanced on a
  schedule" (momentum, low-vol, equal weight, inverse volatility):
  `PortfolioStrategy`.
- Anything else, or when you need to cancel, modify or sequence orders
  yourself: `Strategy`.

## Related

- [Writing Strategies](writing_strategies.md)
- [Portfolio Construction](portfolio_construction.md)
- [Position Management](position_management.md)
- Examples: `honba-examples/strategies/07_declarative_long_short.py` and
  `06_equal_weight_rebalance.py`.
