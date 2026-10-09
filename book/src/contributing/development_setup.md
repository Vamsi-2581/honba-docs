# Development Setup

This document covers setting up Honba for local development: the
toolchain, the test loop, the pre-commit hooks, and the common
workflows.

If you only want to *use* Honba, see
[Installation](https://honba-labs.github.io/honba-docs/getting-started/installation.md). This document is
for people changing the code.

## Prerequisites

Same as the user installation — Rust, Python 3.10+, optionally Node.js
for the frontend. Plus:

- `git` (obviously)
- `just` or `make` for task running
- `cargo-deny` for license auditing
- `pre-commit` for git hooks

Install the extra tools:

    cargo install cargo-deny cargo-watch
    pip install pre-commit

## Clone and Bootstrap

    git clone https://github.com/honba-labs/honba.git
    cd honba

    # Install git hooks
    pre-commit install

    # Build everything
    make build

    # Install the Python extension in editable mode
    make python

    # Run the test suite
    make test

The `Makefile` at the workspace root defines the common tasks:

    make build      # cargo build --workspace
    make test       # cargo test --workspace && pytest python/tests/
    make lint       # cargo clippy + ruff
    make fmt        # cargo fmt + ruff format
    make python     # cd python && maturin develop
    make clean      # cargo clean + remove Python build artifacts
    make docs       # build mkdocs site

## The Inner Loop

The development cycle depends on which layer you are working on.

### Rust core (`crates/*`)

    # Watch and rebuild on file change
    cargo watch -x "check --workspace"

    # Run just the crate you're working on
    cargo test -p honba-messages

    # Run a specific test
    cargo test -p honba-market -- calendar

Rust incremental builds are fast for a single crate. Full workspace
builds take 30–90 seconds on the first pass and a few seconds
afterwards.

### Python control plane (`python/`)

    cd python

    # Editable install (re-run after changing pyproject.toml)
    maturin develop

    # Run tests
    pytest tests/ -x

    # Run a single test file
    pytest tests/unit/test_calendar.py -v

When you change Rust code that is exposed to Python, re-run
`maturin develop`. When you change only Python code, no rebuild is
needed.

### Documentation (`docs/`)

    mkdocs serve
    # → http://localhost:8000

Live reload works for markdown changes. Changes to `mkdocs.yml` require
a restart.

## Pre-Commit Hooks

The hooks run on every commit:

- `cargo fmt --check` — Rust formatting
- `cargo clippy` — Rust lints
- `ruff check` — Python lints
- `ruff format --check` — Python formatting
- `dependency_graph.py` — crate hierarchy enforcement
- Trailing whitespace, end-of-file newline

If a hook fails, fix the issue and re-commit. To bypass for a
work-in-progress commit:

    git commit --no-verify -m "wip"

Do not bypass on commits that will be pushed.

## Testing

### Rust tests

Tests live in three places:

- Inline unit tests (`#[cfg(test)] mod tests`) inside source files.
- Integration tests in `crates/<crate>/tests/`.
- Doc tests in `///` comments.

    # All tests in the workspace
    cargo test --workspace

    # Just one crate, with output
    cargo test -p honba-market -- --nocapture

    # Doc tests only
    cargo test --workspace --doc

### Python tests

    cd python
    pytest tests/unit/          # fast tests
    pytest tests/integration/   # slow tests, need catalog
    pytest tests/strategies/    # strategy regression tests

Test markers:

    @pytest.mark.slow         # excluded from the default run
    @pytest.mark.integration  # requires a data catalog
    @pytest.mark.llm          # requires an LLM API key

Run slow tests only when needed:

    pytest -m "slow"

### The Dependency Test

The most important test is the dependency hierarchy check:

    python3 scripts/dependency_graph.py

This runs in CI and fails the build if any crate imports from a layer
above it. If you are adding a dependency and this test fails, read
[Dependency Rules](dependency_rules.md) before adding an `#[allow]`.

## Data for Testing

Most tests do not require real market data — they use synthetic bars
or fixtures. Integration tests that need a catalog will skip if
`data/catalog/` does not exist.

To populate a small test catalog:

    python3 scripts/fetch_nse_data.py --universe nifty50 --start 2023-01-01 --end 2023-12-31
    python3 scripts/fetch_amfi_nav.py --start 2023-01-01 --end 2023-12-31

This downloads one year of data for NIFTY 50 constituents and a small
set of mutual fund NAVs. It is enough for most integration tests.

## Debugging

### Rust

    # Verbose logging
    RUST_LOG=debug cargo test -p honba-algo -- --nocapture

    # A specific module
    RUST_LOG=honba_algo::cache=trace cargo run --example cache_demo

### Python

    # Enable debug logging
    HONBA_LOG_LEVEL=DEBUG pytest tests/unit/test_engine.py -v

    # Drop into pdb on failure
    pytest tests/unit/test_engine.py --pdb

### Event Log Replay

The backtest engine can write a structured event log to Parquet and
replay it deterministically. This is the primary debugging tool for
bugs that only appear under specific market conditions:

    honba.backtest(strategy, event_log="./debug/events.parquet")

Then:

    honba debug replay ./debug/events.parquet --break-on order_submitted

## Working with Brokers

Broker adapters live in `honba-adapters`, not in the core repository.
If you are testing adapter code, clone that repository as a sibling:

    parent/
    ├── honba/
    └── honba-adapters/

The adapter test suite uses recorded fixtures, so live broker
credentials are not required. Adapters that need credentials (for the
paper-trading integration tests) read them from environment variables
and skip if not present.

## Common Development Tasks

### Add a new indicator

See [Adding Indicators](adding_indicators.md).

### Add a new broker adapter

See [Adding Adapters](adding_adapters.md).

### Add a new crate

1. Create `crates/honba-<name>/` with `Cargo.toml`, `src/lib.rs`,
   `README.md`, and `tests/`.
2. Add it to the workspace `members` list in the root `Cargo.toml`.
3. Add its dependencies to `[workspace.dependencies]`.
4. Add the dependency rules to `scripts/dependency_graph.py`.
5. Update `docs/architecture/crate_hierarchy.md` with the new crate
   and its allowed dependencies.
6. Add the crate to the workspace `Cargo.toml` in any crate that
   needs it.

Adding a crate is a significant change. Open an issue first to
discuss whether the new crate is the right home for the code, or
whether it belongs in an existing crate.

### Update the market calendar

The calendar lives in
`crates/honba-market/src/india/calendar/` and `honba.markets.india.calendar`. NSE publishes the
following year's calendar in December. When the new calendar is
released:

1. Add calendar data for the new year.
2. Add the year to the `Calendar::for_year()` match.
3. Add tests for the new year.
4. Bump the patch version of `honba-market`.

When NSE issues an amendment, update the same file and bump the patch
version again. Historical calendars are never modified — a 2025
backtest uses the 2025 calendar even if 2026 has different holidays.

## CI

The GitHub Actions workflow runs on every push and pull request:

| Job | What it checks |
|---|---|
| `rust` | fmt, clippy, tests, dependency graph |
| `python` | ruff, mypy, pytest |
| `license-audit` | `cargo deny` |
| `docs` | mkdocs build |
