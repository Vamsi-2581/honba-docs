# Contracts & Schema Generation from Rust

> **Single Source of Truth**: Every machine-readable contract, JSON schema, OpenAPI specification, Pydantic model, and MCP tool definition in Honba is generated deterministically from Rust types via `schemars`. Hand-written schema files and manual API models are strictly prohibited.

---

## 1. The Code Generation Pipeline

Honba enforces a single unidirectional source of truth:

```text
 ┌────────────────────────────────────────────────────────┐
 │        PURE RUST SHARED KERNEL (`honba-core`)          │
 │  - Derive macros: #[derive(Serialize, Deserialize,     │
 │                           JsonSchema)]                 │
 └──────────────────────────┬─────────────────────────────┘
                            │ schemars introspection
                            ▼
 ┌────────────────────────────────────────────────────────┐
 │            CODE GENERATION ENGINE                      │
 └──────┬───────────────┬─────────────────┬───────────────┘
        │               │                 │
        ▼               ▼                 ▼
 ┌──────────────┐┌──────────────┐┌────────────────┐┌────────────────┐
 │ JSON SCHEMA  ││ OPENAPI 3.1  ││ MCP TOOL SPECS ││ PYTHON TYPES   │
 │ domain_schema││ REST & WS API││ Frontier Model ││ .pyi Stubs &   │
 │ .json        ││ Spec         ││ Tools Catalog  ││ Pydantic v2    │
 └──────────────┘└──────────────┘└────────────────┘└────────────────┘
```

### Generated Artifact Matrix

| Contract Type | Target Artifact | Consumer Subsystem | Verification Tool |
| :--- | :--- | :--- | :--- |
| **Domain JSON Schema** | `schema/domain/domain_schema.json` | All subsystems | `cargo test -p honba-codegen` |
| **OpenAPI 3.1 Spec** | `schema/openapi/openapi.json` | `honba-ya-api-rest`, Swagger | Automated OpenAPI CI validation |
| **MCP Tool Schemas** | `schema/mcp/mcp_tools.json` | `honba-nou-mcp` (Claude/Gemini) | Model Context Protocol test suite |
| **Python Type Stubs** | `honba/src/honba/wire/*.pyi` | `honba-tamashii`, IDE Autocomplete | `pyright` & `mypy` strict checks |
| **Pydantic v2 Models** | `honba/src/honba/models.py` | Python Strategy SDK & Adapters | Automated golden conformance tests |

---

## 2. Drift Detection & CI Enforcement

Generated artifacts are committed directly to version control. Continuous Integration (CI) enforces that no developer modifies wire types without regenerating and committing the artifacts:

```bash
# Regenerate all contract schemas across the entire workspace
make codegen

# Validate that no uncommitted schema drift exists
make check-codegen-ci
```

If a developer alters a Rust struct in `honba-core` without running `make codegen`, the CI build immediately fails, pointing out the exact diff.

---

## 3. Golden Vector Conformance Testing

To prevent subtle cross-language deserialization discrepancies between Rust (`serde_json`), Python (`pydantic`), and JavaScript (`JSON.parse`):

1. **Golden Test Vectors**: Hundreds of real-world test cases are stored in `schema/golden/` (e.g. order requests, tick feeds, bracket orders, SEBI algo ID strings, circuit band updates).
2. **Bi-Directional Serialization Verification**:
   - Rust deserializes the JSON vector into typed structs, verifies all domain invariants, and re-serializes.
   - Python deserializes the same JSON into Pydantic v2 models and asserts exact mathematical value matching (e.g. `Decimal("2450.75")`).
   - WebAssembly tests verify that Leptos CSR frontend parsers deserialize the vector identically.

---

## 4. Architectural Rules

* **Rule 1: Never edit a generated file by hand.** Any wire contract change must originate as a Rust change in `honba-core`, followed by automated codegen.
* **Rule 2: Rust is the arbiter of correctness.** Where a generated schema and an external model disagree, the Rust type definition is authoritative, and the external consumer must be updated.
