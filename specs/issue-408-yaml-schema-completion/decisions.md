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

## D3: Completion walker API

**Choice:** Factory function `createSchemaCompletion(schema: ZodType): Extension`
**Alternatives:**
- Built-in + override — ships CaseHub schema as default, override via extensions prop. Simpler for CaseHub users but couples pages-code-editor to pages-schema
- Registration API — mutable registry `registerCompletionSchema()`. More flexible for multi-schema scenarios but introduces global mutable state
**Rationale:** A factory function keeps pages-code-editor schema-agnostic. Any consumer passes any Zod schema and gets completion. The existing `yamlCompletion` export is replaced by this generic factory. CaseHub-specific wiring happens at the integration point (examples gallery, app shell), not in the editor package. Domain schemas (Serverless Workflow, Case Diagrams) plug in through the same factory.
**Trade-offs:** Integration code must explicitly compose the schema and pass it. Slightly more boilerplate at the call site vs built-in default.
**Sources:** packages/pages-code-editor/src/yaml-completion.ts (existing yamlCompletion export), packages/pages-code-editor/src/index.ts (barrel exports)
**Exploration:** quick
**Depends on:** D1 (schemas in pages-schema), D2 (discriminatedUnion structure)
**Status:** captured

## D4: Initial component scope

**Choice:** All 45 component types
**Alternatives:**
- High-value subset (~15) — lower risk per PR but partial regression (some types lose even hardcoded completions)
- Data components only (~20) — prioritises complex types but leaves layout/form types uncovered
**Rationale:** The schemas are mechanical translations of existing TypeScript interfaces, not design work. Shipping all at once means completion is immediately better than the hardcoded list for every component type. Partial delivery risks regressing types that had some coverage in the hardcoded list.
**Trade-offs:** Larger PR. More testing surface. Mitigated by the fact that each schema is independent and testable.
**Sources:** packages/pages-component/src/model/type-guards.ts (ComponentTypeRegistry — 45 entries)
**Exploration:** quick
**Depends on:** D1 (schemas in pages-schema)
**Status:** captured
