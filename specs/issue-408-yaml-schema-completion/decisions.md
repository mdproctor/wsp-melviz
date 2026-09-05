# Decisions — #408 Schema-Driven YAML Completion

## D1: Schema package placement

**Choice:** New `@casehubio/pages-schema` package
**Alternatives:**
- In `pages-component` — co-locates schemas with interfaces but adds `zod` runtime weight to a lightweight package
- In `pages-data` — `zod` already there but component prop schemas don't belong in the data layer
**Rationale:** Dedicated package gives clean import boundaries: code editor, LSP (#407), and validation consumers all import from one schema package. Follows the `pages-table` precedent — heavyweight concerns with distinct dependency profiles get their own package.
**Trade-offs:** One more package to maintain. Schema-to-interface sync requires cross-package discipline (mitigated by tests).
**Sources:** packages/pages-code-editor/package.json, packages/pages-component/package.json, packages/pages-data/package.json, issue #372 design spec (package placement rationale)
**Exploration:** quick
**Status:** captured
