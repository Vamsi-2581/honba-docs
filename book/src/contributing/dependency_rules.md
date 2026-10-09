# Dependency Rules

The crate hierarchy is enforced by `scripts/dependency_graph.py`. Every
Rust crate in the workspace declares its `honba-*` dependencies in its
`Cargo.toml`, and the script verifies those dependencies match the
allowed table.

A pull request that violates the hierarchy fails CI. This is
intentional. The hierarchy is what keeps the codebase maintainable as
it grows.

## The Allowed Table

| Crate | May depend on |
|---|---|
| `honba-messages` | *(nothing)* |
| `honba-entities` | `honba-messages` |
| `honba-algo` | `honba-messages`, `honba-entities` |
| `honba-algo-strategies` | `honba-algo` |
| `honba-algo-indicators` | `honba-messages`, `honba-entities` |
| `honba-algo-testing` | `honba-algo`, `honba-algo-strategies` |
| `honba-algo-analytics` | `honba-messages`, `honba-entities` |
| `honba-algo-import` | `honba-messages`, `honba-entities` |
| `honba-algo-export` | `honba-messages`, `honba-entities` |
| `honba-market` | `honba-entities` |

Non-`honba-*` dependencies (serde, tokio, arrow, etc.) are not
restricted by the rule. They are audited separately by `cargo deny`.

## Why Each Rule Exists

### `honba-messages` depends on nothing

Messages are the vocabulary of the engine. If `honba-messages` depended
on `honba-entities`, the two crates would be coupled and neither could
be tested in isolation. Messages must be self-contained types.

### `honba-entities` depends only on `honba-messages`

Entities are domain models. They use message types (a `Bar` might be
referenced by an `Instrument`), but they do not know how to be traded.
A `Position` does not know how to place an order.

### `honba-algo` depends on messages and entities

The engine operates on messages and tracks entities. It defines the
connector traits that adapters implement. It does not depend on any
specific strategy or indicator.

### `honba-algo-strategies` depends only on `honba-algo`

Strategies use the engine API. They do not import indicators (those
are separate) or testing infrastructure (that is higher).

### `honba-algo-indicators` depends on messages and entities

Indicators consume bars (messages) and can be attached to instruments
(entities). They do not need the engine.

### `honba-algo-testing` depends on the engine and strategies

The backtester runs strategies in the engine. It is at the top of the
Rust hierarchy.

### `honba-algo-analytics` depends on messages and entities

Analytics operate on backtest results, which are messages and entities.
They do not need the engine.

### `honba-algo-import` and `honba-algo-export` depend on messages and entities

Data I/O produces and consumes messages and entities. It has no need
for the engine.

### `honba-market` depends only on `honba-entities`

This is the most important restriction. `honba-market` is the
market specialization module. It should be testable without
spinning up an engine. The market calendars, cost models, and universe
definitions are all pure data transformations that take a date and
return a value.

Adding `honba-algo` as a dependency of `honba-market` would mean the
calendar cannot be tested without an event loop. This is exactly the
kind of coupling the hierarchy is designed to prevent.

## When You Need a Forbidden Dependency

If a piece of code in `honba-market` needs something from `honba-algo`,
you have four options, in order of preference:

### 1. Move the code to the lower crate

The most common case. If `honba-market` needs to know about a
`Price` type from `honba-algo`, the `Price` type probably belongs in
`honba-messages`, not `honba-algo`.

### 2. Introduce a trait in the lower crate

If `honba-market` needs to call a function from `honba-algo`, define a
trait in `honba-market` and implement it in `honba-algo`. This inverts
the dependency.

    // In honba-market
    pub trait Clock {
        fn now(&self) -> UnixNanos;
    }

    // In honba-algo
    impl Clock for EngineClock {
        fn now(&self) -> UnixNanos { ... }
    }

### 3. Duplicate the small bit of code

If it is genuinely small — a five-line helper — duplicating it in the
lower crate is sometimes cheaper than restructuring. This is a last
resort, and the duplication must be noted in a comment.

### 4. Discuss in an issue

If none of the above work, open an issue. The response may be that the
hierarchy needs to change, which is a bigger decision than a single
pull request should make.

## What the Script Checks

`scripts/dependency_graph.py` reads each crate's `Cargo.toml` and
extracts the `honba-*` dependencies. For each crate, it compares the
actual dependencies against the allowed set.

    # From the script
    ALLOWED = {
        "honba-messages": set(),
        "honba-entities": {"honba-messages"},
        "honba-algo": {"honba-messages", "honba-entities"},
        "honba-algo-strategies": {"honba-algo"},
        "honba-algo-indicators": {"honba-messages", "honba-entities"},
        "honba-algo-testing": {"honba-algo", "honba-algo-strategies"},
        "honba-algo-analytics": {"honba-messages", "honba-entities"},
        "honba-algo-import": {"honba-messages", "honba-entities"},
        "honba-algo-export": {"honba-messages", "honba-entities"},
        "honba-market": {"honba-entities"},
    }

Running it locally:

    python3 scripts/dependency_graph.py
    # → Dependency hierarchy OK.

If a crate violates the hierarchy:

    python3 scripts/dependency_graph.py
    # VIOLATION: honba-market illegally depends on ['honba-algo']
    #
    # 1 violation(s) found.

The error message is precise. It names the crate and the offending
dependency.

## Dev Dependencies

Dev dependencies (under `[dev-dependencies]` in `Cargo.toml`) are
**not** checked by the hierarchy script. This is deliberate: tests
often need to set up scenarios that cross layers. For example, testing
`honba-market` might use `honba-algo-testing` to build a small fixture
engine.

The rule is: a **runtime** dependency must follow the hierarchy. A
**test-only** dependency may not.

This means the following is legal:

    # honba-market/Cargo.toml
    [dependencies]
    honba-entities = { workspace = true }

    [dev-dependencies]
    honba-algo-testing = { workspace = true }   # only for tests

But this is not:

    # honba-market/Cargo.toml
    [dependencies]
    honba-entities = { workspace = true }
    honba-algo = { workspace = true }            # runtime violation

## Feature Flags

Feature flags are checked the same way as regular dependencies. A
crate that adds an optional dependency on a forbidden crate must be
modified or the feature moved.

## The Python Layer

The dependency hierarchy applies to Rust crates. The Python control
plane in `python/honba/` has a looser structure: it can import from
any crate that has been exposed via PyO3. However, the same principle
applies — the Python `core` module should not import from
`python.honba.ai`, and so on.

This is enforced by convention, not by a script. If it becomes a
problem, a Python-side version of `dependency_graph.py` can be added.

## Removing a Dependency

Removing a dependency is always allowed. If you find that a crate no
longer needs a `honba-*` dependency it has, remove it from
`Cargo.toml`. The script will pass either way, but removing unused
dependencies reduces build time and clarifies intent.

## Updating the Rule Table

If you genuinely need to add a new edge to the hierarchy — for
example, because `honba-market` needs to depend on `honba-algo` for a
new feature — you must:

1. Justify the change in an issue or pull request description.
2. Update `scripts/dependency_graph.py` with the new edge.
3. Update `docs/architecture/crate_hierarchy.md` with the new edge.
4. Update this document if the reasoning changes.

The rule table is not a bureaucratic constraint. It is a description
of the intended architecture. If the intended architecture changes,
the table changes with it, in the same pull request.
