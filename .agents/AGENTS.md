# `honba-docs` Agent Guidelines & Architecture Invariants

**Subsystem**: Documentation (MkDocs & mdBook)
**Language**: Markdown
**Primary Responsibility**: Comprehensive user manuals, tutorials, architectural specifications, and API documentation.

---

## 1. Role & Architectural Hints
* `honba-docs` is the official knowledge base:
  1. Built using MkDocs Material and mdBook.
  2. Single source of truth for Indian equity regulations, cost models (STT, GST, stamp duty), SEBI algo circulars, and strategy tutorials.
  3. Code snippets in docs must correspond to tested code in `honba-examples` or `honba-strategies`.

---

## 2. Maintenance & Quality
* **Strict Markdown Linting**: Ensure tables, math fences, and links are valid.
* **Strict Build Verification**: `mkdocs build --strict` must pass without broken links or unresolved anchors.
* **Check .agents/HANDOFF.md**: Always verify handoff notes before editing docs.

---

## 3. Development & CI Workflow
* **Build Docs**:
  ```bash
  mkdocs build --strict
  mdbook build book
  ```
