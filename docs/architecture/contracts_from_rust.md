# Contracts Are Generated From Rust

Every machine-readable contract of Honba is generated from the Rust types. Source of truth:
ADR 0014 (supersedes ADR 0006 decision 1) and `honba/crates/honba-codegen`.

## What `honba-codegen` emits

One registry of Rust wire types (`registry.rs`) is walked once and rendered five ways, using
`schemars` derives on the domain crates:

| Output | Generated file(s) | Committed in |
|---|---|---|
| JSON Schema | `domain_schema.json` | `honba/schema/domain` |
| OpenAPI 3.1, with real `paths` from the endpoint registry | `openapi.json` | `honba/schema/openapi` |
| MCP tool schemas | `mcp_tools.json` | `honba/schema/mcp` |
| Python `.pyi` stubs | `__init__.pyi` | `honba/python/src/honba/wire/generated` |
| TypeScript | frontend types | `honba-frontend/src/core/types/generated` (sibling checkout) |

They are driven by `honba schema export` (`make codegen` regenerates all of them). Python is a
consumer: the pydantic wire models are checked against the Rust contract but never produce a
committed schema artifact, and the generated Python is `.pyi` stubs only.

## Drift checks

Generated files are committed, and CI fails on drift rather than relying on review.

- `make check-codegen-ci` runs `check-schema`, `check-openapi`, `check-pyi` and `check-mcp`.
  Each regenerates its directory and fails on any modified, deleted or untracked file there
  (a new file nobody committed counts). This is the step CI runs.
- `make check-codegen` adds `check-schema-ts`, which writes into the sibling `honba-frontend`
  checkout and so runs locally only. CI does not guard the TypeScript artifact.

## Golden-vector validation

`crates/honba-codegen/tests/golden_vectors_conform.rs` validates the wire-shaped golden vectors
(`schema/golden` cases and tolerated inputs, accepted strategy manifests, screener predicates and
groups) against the generated `domain_schema.json`, so the schema must describe the serde form
faithfully. It uses a small in-crate validator (`honba_codegen::validate_instance`) that covers
only the keywords `schemars` emits and fails on any other validating keyword.

Not checked: `invalid*` vectors (rejected by Rust invariants the schema cannot express, such as
`low <= high`), engine scenario scripts and indicator series (not wire instances), and string
formats (`format` is an annotation).

## Rules

- **Never edit a generated file by hand.** A wire change is a Rust change plus regenerated,
  committed artifacts in the same commit.
- Where the generated schema and a pydantic model disagree, the generated schema is right and the
  model is the bug.
