# Step Manifests and Chaining

A step manifest is a small TOML file that describes a strategy or pipeline
step as **data**: its parameters, what it looks at, what it needs from the
previous step and what it produces. A tool (a future GUI algo designer, or a
validator) can render a step's form and check that steps fit together without
importing any Python.

Two repositories carry the pieces:

- `honba-strategies` defines the schema and the validator, and ships a
  `step.toml` beside the `config.toml` of every registered strategy.
- `honba-examples` has a pipeline of screen, filter, size and execute steps
  that implement the same manifest shape, plus a runner that loads a chain from
  a TOML file.

The authoritative references are `honba-strategies/schema/README.md`,
`honba-strategies/docs/step-manifest-examples.md` and
`honba-examples/docs/roadmap-chaining.md` (Phase 5). This page is the short
version.

!!! note "Status"
    The schema, the validator, the registered `step.toml` files and the
    `honba-examples` pipeline steps exist today. A GUI that consumes them does
    not. Several step ids used as illustrations below are marked *future*.

## The stage model

A portfolio chain moves data through fixed stages:

```
universe --screen--> candidates --filter--> candidates --size--> weights --execute--> orders / result
```

| Kind | Consumes | Emits | Meaning |
|---|---|---|---|
| `screen` | `universe` or `none` | `candidates` | Pick instruments from a universe. |
| `filter` | `candidates` | `candidates` | Narrow or annotate the candidates. |
| `signal` | `universe` | `orders` | Per-symbol timing for a single-instrument bar strategy. Standalone. |
| `size` | `candidates` | `weights` | Turn candidates into target weights. It does not trade. |
| `risk` | `orders` | `orders` | Adjust orders. |
| `execute` | `weights` **or** `universe` | `orders` or `result` | Trade. Given `weights` it executes them; given a `universe` it is a self-contained portfolio strategy that selects, weights and schedules itself. |

Points that are easy to get wrong:

- `signal` and `size` are each optional, but they belong to two different kinds
  of chain. A portfolio chain is `screen -> filter -> size -> execute`. A bar
  strategy is a lone `signal`, or a lone self-contained `execute`. A `signal`
  cannot feed a `size`: it already emits orders, and a sizer needs candidates.
  The validator rejects the combination and says why.
- The six portfolio strategies in the catalog (`equal_weight` and the five
  `alpha30_*` strategies) are `execute.*` steps, not sizers. They consume a
  bare `universe`.
- `consumes.shape` is either one shape or a list of accepted shapes. For
  example `execute.equal_weight` accepts `["weights", "universe"]`.
- Kinds must run in the order screen, filter, signal, size, risk, execute.
  Repeats are allowed; going backwards is not.

## What is in a `step.toml`

| Field | Meaning |
|---|---|
| `schema` | Always `1`. |
| `id` | `<kind>.<registry name>`, for example `signal.sma_crossover`. |
| `kind` | `screen`, `filter`, `signal`, `size`, `risk` or `execute`. |
| `title`, `description`, `tags` | Display text and free-form semantic tags. These are not the indicator-family tags in `registry.json`. |
| `[params.<name>]` | One table per parameter: `type` (`int`, `float`, `bool`, `str`, `enum`, `date`, `symbols`, `window`), `default`, `min`/`max`/`step`, `choices`, `pattern`, `description`, `sweepable`. `optional = true` means no default and the param may be omitted. |
| `pitfalls` | Optional array of short one-line warnings (overfitting, lookahead, cost sensitivity) that a GUI can show when the step is added to a chain. See below. |
| `[requires]` | What the step **looks at**: `bars`, `fundamentals`, `indicators`, `history_bars`, `pit` (needs point-in-time data). An indicator argument or `history_bars` may be `"$param"` to bind it to a step param. |
| `[consumes]` | `shape` (string or list) and `columns` needed from the previous step. |
| `[emits]` | `shape`, `adds` (columns appended), `report` (always shown), `reportable` (extra columns a user may ask for). |
| `[chain]` | `after` / `before` palette hints. The enforced rule is the kind order above. |

The schema is `honba-strategies/schema/step.schema.json` (JSON Schema, draft
2020-12). It is separate from `honba.strategies.manifest.StrategyManifest`,
which is the runtime manifest (subscriptions, warm-up) used by the engine.

## Pitfalls in a manifest

`pitfalls` is the short form of a strategy README's `## Pitfalls` section. The
schema only requires non-empty, unique strings; `test_readme_sections.py` goes
further for registered strategies and requires 2 to 4 strings of 10 to 160
characters each. The README keeps the long form and must link the generic bias
page.

```toml
pitfalls = [
  "Tuned fast/slow pairs overfit; validate walk-forward or out-of-sample.",
  "Decides on the bar close; fill at the next open, not the same close.",
  "Whipsaws in sideways markets; lags trend turns.",
]
```

The field is optional for a manifest in general, but every registered
catalog strategy carries it. See
[Pitfalls in the Examples and the Catalog](../backtesting/pitfalls_in_examples.md).

## Looks at versus reports

A step declares what it **looks at** (`requires`) separately from what it
**reports** (`emits.report` and `emits.reportable`). The distinction matters:

- A *filter* tests a value and drops candidates that fail. ROE above 15 and a
  close above the 200-day EMA are filters.
- A *report* column is shown in the results but never tested. Market cap and
  dividend yield can ride along without excluding anyone.

An illustrative chain that filters on ROE and EMA200 but only *shows* market cap
and dividend yield:

```toml
[[steps]]
id = "screen.index_members"          # future: starting universe
with = { index = "nifty200" }

[[steps]]
id = "filter.fundamental_trend"      # future
with = { min_roe = 15.0, ema_period = 200 }     # looked at: filtered on
report = ["market_cap", "dividend_yield_pct"]   # reported: shown, not tested

[[steps]]
id = "size.equal_weight"             # future in the catalog
with = { allocation = 0.95 }

[[steps]]
id = "execute.equal_weight"          # exists: portfolio/rebalancing/equal_weight
```

Only `execute.equal_weight` exists as a catalog step today. The screen, filter
and size ids above are hypothetical and appear only to show how the pieces
compose. The validator accepts a `report` list only when every column is in
that step's `emits.reportable`; asking for anything else is an error.

## What is implemented

| Piece | Where | State |
|---|---|---|
| Schema and `step.toml` per registered strategy (17) | `honba-strategies` | Exists. `test_step_manifests.py` keeps each manifest in step with its `config.toml` and code. |
| Chain validator | `honba-strategies/tools/validate_chain.py` | Exists. `validate_chain(steps)` returns a list of error strings; empty means valid. |
| Validator rules | same | Shape compatibility, column availability, kind order, `with` values checked against each param's type, bounds, choices and pattern. |
| `describe()` on pipeline steps | `honba-examples/honba_examples/pipeline/` | Exists for every step except the signal stubs; returns a dict in the `step.toml` shape. |
| `report=[...]` | screeners and the fundamental-quality filter in `honba-examples` | Exists. Columns: `market_cap`, `dividend_yield_pct`, `pe_ratio`, `pb_ratio`, `roce_pct`, `roe_pct`, `debt_to_equity`, `operating_margin_pct`, `net_margin_pct`, `free_cash_flow`, `eps`. |
| TOML chain runner | `honba_examples/pipeline/chain.py` and `screening/06_run_chain.py` | Exists. Worked chain: `screening/chains/quality_value.toml`. |
| GUI algo designer | not in any repo | Future. |
| Sizer, screen and filter steps in the strategy catalog | `honba-strategies` | Future. No catalog strategy is a sizer; today a catalog chain is a single trading step. |

Report columns are derived point-in-time from the latest fiscal-year
fundamentals on or before the as-of date. `market_cap` is the as-of close times
diluted average shares, not the live snapshot. A symbol with no data gets
`null`.

## Validated versus runnable

Validation and execution are separate. The chain runner in `honba-examples`
checks a chain with the `honba-strategies` validator before running it, and
the validator knows catalog ids (`signal.*`, `execute.*`) as well as the
pipeline's own `screen.*`, `filter.*`, `size.*` and `execute.sip_backtest`.

So a chain ending in `execute.equal_weight` **validates**, but
`06_run_chain.py` refuses to run it with "validated but not runnable in this
pipeline yet". Running catalog strategies from a chain would need a bridge to
the backtest engine, which is future work. Treat a passing validation as "these
steps fit together", not as "this chain will run here".

```bash
# from a honba-examples checkout
python screening/06_run_chain.py --chain screening/chains/quality_value.toml --validate-only
python screening/06_run_chain.py --chain screening/chains/quality_value.toml --format csv
python screening/06_run_chain.py --describe        # every step's manifest as JSON
```

The validator is found through `$HONBA_STRATEGIES_DIR` or the sibling
`honba-strategies` checkout.

## Known caveats

- `sma_crossover`: the `Parameter()` defaults in `strategy.py` are fast 10 and
  slow 30, while `config.toml` and `step.toml` say 20 and 50. The manifest
  follows `config.toml`, and `test_step_manifests.py` records the mismatch
  explicitly.
- `alpha30_factor` and `alpha30_quality` do no ranking: they keep every member
  of the universe and weight equally. Use `alpha30_momentum`, `alpha30_low_vol`
  or `alpha30_mean_reversion` for ranked selection.
- `point_in_time = true` on the Alpha 30 strategies does not remove survivorship
  bias: the shipped history is today's 30 names stamped from 2005-04-01.
- Other per-strategy caveats (`vwap_reversion` trading the first bar and having
  no stop, `turtle_20` and `kdj_cross` stops checked after the bar,
  `rsi2_connors` and `ifr2` having no stop) are listed in the
  [Strategies index](index.md#honest-caveats-in-the-catalog).
- The Graham and net-net screeners in `honba-examples` are not pipeline steps
  yet.

## See also

- [Strategies index](index.md) for the catalog layout and registry.
- [Portfolio Construction](portfolio_construction.md) for the universe,
  selection, weighting and schedule grammar behind the `execute.*` steps.
- [Examples](../examples/index.md) for the `screening/` and `backtesting/`
  scripts that use the pipeline.
