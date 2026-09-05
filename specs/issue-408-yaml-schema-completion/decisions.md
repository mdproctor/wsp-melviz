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

## D2: Type-dependent property dispatch

**Choice:** Zod `discriminatedUnion` on the `type` field
**Alternatives:**
- Context-aware walker with flat `Map<string, ZodType>` — simpler walker code but type→properties relationship lives outside the schema; LSP validation would need separate dispatch logic
**Rationale:** The discriminatedUnion encodes the type→properties relationship in Zod itself. A single `componentSchema.parse()` validates that bar-chart components only have BarChartProps. Both completion (walker extracts matching branch via `_def.optionsMap`) and validation (LSP calls `.parse()`) derive from one schema. Individual component schemas still exist independently for modularity — they're composed into the union.
**Trade-offs:** Walker must navigate Zod's `_def.optionsMap` internal structure. Building the union with 45 branches is verbose (but generated from per-component schemas).
**Sources:** packages/pages-component/src/model/type-guards.ts (ComponentTypeRegistry), packages/pages-code-editor/src/yaml-completion.ts (existing walker), issue #408 ("schemas become the LSP's source of truth for completion, validation, and diagnostics")
**Exploration:** deep-analysis
**Depends on:** D1 (schemas live in pages-schema package)
**Status:** captured
