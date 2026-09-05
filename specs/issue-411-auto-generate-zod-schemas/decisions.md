# Decisions — #411 Auto-Generate Zod Schemas

## D1: Generator technology

**Choice:** ts-morph
**Alternatives:**
- ts-to-zod — purpose-built but less control over edge cases (branded types, function filtering, intersection handling)
- Custom TS compiler API — maximum control but verbose; ts-morph wraps this with a better API
**Rationale:** ts-morph provides a high-level API for reading interfaces, types, and inheritance chains. We control exactly how each TypeScript construct maps to Zod — handling intersection types (extends DataComponentCommon, ChartSettings), optional fields, string literal unions, and filtering function-typed properties.
**Trade-offs:** New dev dependency (ts-morph). Generator script must be maintained when new TypeScript patterns are introduced.
**Sources:** packages/pages-component/src/model/displayer-types.ts, packages/pages-component/src/model/component-props.ts
**Exploration:** quick
**Status:** captured

## D2: Build timing

**Choice:** Pre-build script with committed output
**Alternatives:**
- Test-time validation — catches drift but doesn't prevent manual maintenance
- Build-time plugin — always derived but complex setup, slower builds
**Rationale:** A generate-schemas.ts script runs before tsc, reads interfaces from pages-component source, writes component-schemas.ts to pages-schema/src/. The generated file is committed to git so consumers don't need ts-morph. Re-run on interface changes. Simple, transparent, reviewable diffs.
**Trade-offs:** Generated file in git can cause merge conflicts. Requires remembering to re-run after interface changes (mitigated by CI check).
**Sources:** packages/pages-schema/src/component-schemas.ts (current hand-written file to be replaced)
**Exploration:** quick
**Depends on:** D1 (ts-morph as the generator engine)
**Status:** captured

## D3: Desugarer integration

**Choice:** Schema.parse() on extracted props, delete passthrough loop
**Alternatives:**
- Schema.strip() then passthrough — gentler migration but keeps the passthrough
- Warn-only mode first — log warnings for undeclared properties before removing passthrough
**Rationale:** After the desugarer builds the props object (from general/chart/table sections), validate with componentSchemaRegistry.get(type).parse(props). Unknown keys are stripped by Zod's default behavior. The passthrough loop (displayer-desugar.ts lines 374-385) is deleted entirely. No dynamic properties survive.
**Trade-offs:** Breaking change for any YAML that relies on undeclared properties passing through. Pre-release platform — breaking changes cost nothing. Interfaces must be complete BEFORE this change lands.
**Sources:** packages/pages-ui/src/parser/displayer-desugar.ts:374-385 (passthrough loop), packages/pages-schema/src/schema-registry.ts
**Exploration:** quick
**Depends on:** D1 (generator), D2 (generated schemas exist)
**Status:** captured
