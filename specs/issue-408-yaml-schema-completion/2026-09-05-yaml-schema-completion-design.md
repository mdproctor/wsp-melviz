# Design: Schema-Driven YAML Completion for pages-code-editor

**Issue:** casehubio/casehub-pages#408
**Date:** 2026-09-05
**Branch:** issue-408-yaml-schema-completion

## Overview

Replace the hardcoded completion data in `pages-code-editor` with a schema-driven system built on Zod. Three deliverables:

1. **`@casehubio/pages-schema`** — new package containing Zod schemas for all 45 component types, the composed dashboard document schema, and a schema registry
2. **Schema-driven completion engine** — generic Zod schema walker in `pages-code-editor` that produces CodeMirror autocompletion from any Zod schema
3. **Public schema exports from `pages-data`** — the existing lookup and dataset-def Zod schemas, currently module-private, exported for reuse

The schemas serve both completion (this issue) and validation/diagnostics (future LSP, #407). The completion engine is schema-agnostic — it works with any Zod schema, enabling pluggable domain-specific YAML formats.

## 1. Package: `@casehubio/pages-schema`

### Placement rationale

Dedicated package following the `pages-table` precedent — heavyweight concerns with distinct dependency profiles get their own package. Consumers (code editor, LSP, validation tools) import schemas without pulling in the CodeMirror editor or the full component runtime.

### Package structure

```
packages/pages-schema/
  package.json
  tsconfig.json
  tsconfig.build.json
  src/
    index.ts                     # barrel exports
    component-schemas.ts         # per-component Zod schemas
    component-schemas.test.ts    # type-compatibility tests
    document-schema.ts           # composed dashboard document schema
    document-schema.test.ts      # document validation tests
    schema-registry.ts           # component type → schema lookup
```

### Dependencies

```json
{
  "name": "@casehubio/pages-schema",
  "dependencies": {
    "@casehubio/pages-data": "workspace:*",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@casehubio/pages-component": "workspace:*",
    "@casehubio/pages-tsconfig": "workspace:*",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1",
    "rimraf": "^6.1.0"
  }
}
```

`pages-component` is a devDependency only — needed for type-compatibility tests that verify Zod schemas produce types assignable to the TypeScript interfaces, but not a runtime dependency. The schemas are self-contained Zod definitions.

### Per-component Zod schemas

Each component type in `ComponentTypeRegistry` (45 types) gets a corresponding Zod schema. The schemas mirror the existing TypeScript interfaces in `displayer-types.ts`, `component-props.ts`, `form-input-types.ts`, `action-types.ts`, `form-scope-types.ts`, and `submit-button-types.ts`.

Shared base schemas factor out common shapes:

```typescript
const dataComponentCommonSchema = z.object({
  title: z.string().optional(),
  visible: z.boolean().optional(),
  width: z.string().optional(),
  height: z.string().optional(),
  csvExport: z.boolean().optional(),
  lookup: lookupSchema,           // from pages-data
  rowCount: z.number().optional(),
  rowOffset: z.number().optional(),
  columns: z.array(columnSettingsSchema).optional(),
  filter: filterSettingsSchema.optional(),
  refresh: refreshSettingsSchema.optional(),
});

const chartSettingsSchema = z.object({
  resizable: z.boolean().optional(),
  zoom: z.boolean().optional(),
  maxWidth: z.number().optional(),
  maxHeight: z.number().optional(),
  legend: z.object({
    show: z.boolean().optional(),
    position: z.enum(["top", "bottom", "left", "right"]).optional(),
  }).optional(),
  margin: z.object({
    top: z.number().optional(),
    right: z.number().optional(),
    bottom: z.number().optional(),
    left: z.number().optional(),
  }).optional(),
  xAxis: z.object({
    title: z.string().optional(),
    showLabels: z.boolean().optional(),
    labelAngle: z.number().optional(),
  }).optional(),
  yAxis: z.object({
    title: z.string().optional(),
    showLabels: z.boolean().optional(),
    labelAngle: z.number().optional(),
  }).optional(),
  grid: z.object({
    x: z.boolean().optional(),
    y: z.boolean().optional(),
  }).optional(),
  extra: z.record(z.unknown()).optional(),
});
```

Then per-component schemas compose these:

```typescript
export const barChartPropsSchema = dataComponentCommonSchema
  .merge(chartSettingsSchema)
  .extend({
    subtype: z.enum(["column", "column-stacked", "bar", "bar-stacked"]).optional(),
  });

export const dataTablePropsSchema = dataComponentCommonSchema.extend({
  pageSize: z.number().optional(),
  sortable: z.boolean().optional(),
  resizable: z.boolean().optional(),
  selection: z.enum(["none", "single", "multi"]).optional(),
  selectionKey: z.string().optional(),
});
```

Layout, content, and form component schemas follow the same pattern — mechanical translations of the TypeScript interfaces with appropriate Zod types.

### Composed document schema

The full CaseHub dashboard YAML structure uses `z.discriminatedUnion` to encode the type→properties relationship. Common component fields are factored into a base schema that each branch extends:

```typescript
const componentBase = z.object({
  id: z.string().optional(),
  style: z.record(z.string()).optional(),
  visibleWhen: z.string().optional(),
});

const componentSchema = z.discriminatedUnion('type', [
  componentBase.extend({ type: z.literal('bar-chart'), properties: barChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal('data-table'), properties: dataTablePropsSchema.optional() }),
  componentBase.extend({ type: z.literal('metric'), properties: metricPropsSchema.optional() }),
  // ... all 45 component types
]);

const pageSchema = z.object({
  name: z.string().optional(),
  components: z.array(componentSchema).optional(),
  rows: z.array(rowSchema).optional(),
  columns: z.array(columnSchema).optional(),
  properties: z.record(z.string()).optional(),
});

export const dashboardSchema = z.object({
  pages: z.array(pageSchema).optional(),
  datasets: z.array(datasetDefSchema).optional(),    // from pages-data
  navTree: navTreeSchema.optional(),
  properties: z.record(z.string()).optional(),
});
```

### Schema registry

A flat `Map` for direct component-type → schema lookup, derived from the discriminatedUnion for convenience:

```typescript
export const componentSchemaRegistry: ReadonlyMap<string, z.ZodType> = new Map([
  ['bar-chart', barChartPropsSchema],
  ['data-table', dataTablePropsSchema],
  // ... all 45
]);
```

This is a convenience export. The discriminatedUnion is the canonical source; the registry provides O(1) lookup without navigating `_def.optionsMap`.

### Type-compatibility tests

Each component schema has a compile-time test verifying its output type is assignable to the corresponding TypeScript interface:

```typescript
import type { BarChartProps } from '@casehubio/pages-component';

// Compile-time check: schema output must be assignable to interface
const _typeCheck: BarChartProps = {} as z.output<typeof barChartPropsSchema>;
```

If the schema drifts from the interface, this fails at compile time. These tests use `pages-component` as a devDependency.

## 2. Schema-Driven Completion Engine

### Location

`packages/pages-code-editor/src/schema-completion.ts` (new file). The existing `yaml-completion.ts` is removed.

### API

```typescript
export function createSchemaCompletion(schema: z.ZodType): Extension
```

Returns a CodeMirror `autocompletion` extension configured with a completion source that walks the provided Zod schema. The existing `yamlCompletion` export is removed from the barrel; `createSchemaCompletion` replaces it.

### Schema walker

The walker resolves completions by navigating the Zod schema tree to match the current YAML cursor position:

1. **`buildAncestorPath(doc, pos)`** — preserved from current implementation. Walks YAML indentation to produce an ancestor key path (e.g., `['pages', 'components', 'properties']`).

2. **`navigateSchema(schema, path)`** — new function. Given a Zod schema and an ancestor path, descends through the schema tree:
   - `ZodObject` → looks up key in `.shape`, recurses into the value schema
   - `ZodArray` → recurses into `.element`
   - `ZodOptional` / `ZodDefault` → unwraps and recurses
   - `ZodDiscriminatedUnion` → reads the discriminator value from the YAML document, selects the matching branch via `optionsMap`
   - `ZodUnion` → returns completions merged from all branches
   - `ZodIntersection` → merges completions from both sides
   - `ZodLazy` → unwraps via `._def.getter()`

3. **`schemaToCompletions(schema, context)`** — new function. Produces `CompletionResult` from the resolved schema at the cursor position:
   - `ZodObject` → key completions from `Object.keys(schema.shape)`, each with `type: 'property'` and `apply: key + ': '`
   - `ZodEnum` → value completions from `schema.options`, each with `type: 'enum'`
   - `ZodNativeEnum` → value completions from enum values
   - `ZodLiteral` → single value completion
   - `ZodBoolean` → `true` / `false` completions
   - `ZodNumber` → no completions (user types a number)
   - `ZodString` with `.description` → shows description as hint

### Discriminated union dispatch

When the walker encounters a `ZodDiscriminatedUnion` during schema navigation, it needs the discriminator value from the YAML document. Extended `buildAncestorPath` returns sibling key-value pairs at the current indent level:

```typescript
interface YamlContext {
  path: string[];
  siblings: Record<string, string>;  // key-value pairs at current indent level
}
```

The walker reads `siblings.type` to select the matching branch from the discriminated union's `optionsMap`. If `type` is not yet written (user hasn't typed it), the walker falls back to completing the `type` key itself with all valid type names from the union.

### Description annotations

Zod schemas support `.describe()` for adding human-readable descriptions. The component schemas use this to provide completion detail text:

```typescript
export const barChartPropsSchema = dataComponentCommonSchema
  .merge(chartSettingsSchema)
  .extend({
    subtype: z.enum(["column", "column-stacked", "bar", "bar-stacked"])
      .describe("bar/column chart variant")
      .optional(),
  });
```

The walker extracts descriptions via `schema.description` and maps them to CodeMirror's `detail` field on completions.

### Backward compatibility

The `yamlCompletion` export is removed. The examples gallery entry point (`casehub-entry.ts`) switches to:

```typescript
import { createSchemaCompletion } from '@casehubio/pages-code-editor';
import { dashboardSchema } from '@casehubio/pages-schema';

export const yamlCompletion = createSchemaCompletion(dashboardSchema);
```

This preserves the same external API for `casehub-bundle` consumers while using the schema-driven engine internally.

## 3. Public Schema Exports from `pages-data`

### Lookup schema

Export `lookupSchema` from `lookup-parser.ts`. Currently only `parseLookup` (the function) is public; the Zod schema that powers it is module-private.

```typescript
// pages-data/src/dataset/lookup-parser.ts — add export
export { lookupSchema };
```

Also export the sub-schemas used in composition: `filterLeafSchema`, `columnGroupSchema`, `sortColumnSchema`, `groupEntrySchema`.

### Dataset definition schema

Export `externalDataSetDefSchema` from `external/schema.ts`. Currently only `parseExternalDataSetDef` and `ParsedExternalDataSetDef` are public.

```typescript
// pages-data/src/dataset/external/schema.ts — add export
export { externalDataSetDefSchema };
```

### Barrel updates

Add the schema exports to `pages-data/src/index.ts`:

```typescript
export { lookupSchema } from './dataset/lookup-parser.js';
export { externalDataSetDefSchema } from './dataset/external/schema.js';
```

## 4. Domain-Specific Schema Support

The `createSchemaCompletion(schema)` factory accepts any Zod schema, not just the CaseHub dashboard schema. Domain-specific YAML formats plug in by passing a different root schema:

```typescript
// Serverless Workflow completion
import { createSchemaCompletion } from '@casehubio/pages-code-editor';
import { serverlessWorkflowSchema } from './sw-schema.js';

editor.extensions = [createSchemaCompletion(serverlessWorkflowSchema)];
```

This is not a separate deliverable — it falls out of the factory function design. No domain-specific schemas are created in this issue; the architecture supports them.

## Testing Strategy

### Unit tests (Vitest)

**pages-schema:**
- Each component schema parses valid props without error
- Each component schema rejects invalid props (wrong types, unknown keys with `.strict()`)
- Composed document schema validates a complete dashboard YAML
- Type-compatibility tests (compile-time): schema output assignable to TypeScript interface
- Schema registry has an entry for every key in `ComponentTypeRegistry`

**pages-code-editor (schema-completion.ts):**
- `navigateSchema` descends through object keys, arrays, optionals, unions, intersections
- `navigateSchema` handles discriminated union dispatch with a mock sibling reader
- `schemaToCompletions` produces correct key completions from `ZodObject`
- `schemaToCompletions` produces correct value completions from `ZodEnum`
- `schemaToCompletions` produces `true`/`false` from `ZodBoolean`
- Description annotations appear as completion detail text
- End-to-end: `createSchemaCompletion(dashboardSchema)` produces correct completions at various YAML positions

**pages-data:**
- Existing `parseLookup` and `parseExternalDataSetDef` tests continue to pass (no behavior change)
- New: `lookupSchema` and `externalDataSetDefSchema` are importable from the public API

### Integration tests (Playwright)

- Code editor with schema completion shows type-aware property suggestions
- Typing `type: bar-chart` then moving to properties shows BarChartProps keys
- Typing `type: data-table` shows DataTableProps keys (different from bar-chart)
- Type value completion shows all 45 component type names

## Build integration

Add `@casehubio/pages-schema` to the root `package.json` `build:packages` script. It must build after `pages-data` (provides `zod` and lookup/dataset schemas) and before `pages-code-editor` (if the examples gallery imports from it at build time — but since the factory function keeps pages-code-editor schema-agnostic, there's no build-time dependency between them).

## Future: LSP integration (#407)

The schemas defined here become the LSP's source of truth:
- **Completion:** LSP calls `navigateSchema()` + `schemaToCompletions()` (same walker, different transport)
- **Validation:** LSP calls `dashboardSchema.parse()` and maps Zod errors to LSP diagnostics
- **Hover:** LSP reads `.description` from the schema at the cursor position

The LSP wraps the schemas in LSP protocol; this issue creates the schemas themselves.

## References

- [packages/pages-code-editor/src/yaml-completion.ts] — existing hardcoded completion (replaced)
- [packages/pages-component/src/model/type-guards.ts:64-134] — ComponentTypeRegistry (45 component types)
- [packages/pages-component/src/model/displayer-types.ts] — data component TypeScript interfaces
- [packages/pages-component/src/model/component-props.ts] — layout/content component TypeScript interfaces
- [packages/pages-component/src/model/form-input-types.ts] — form input TypeScript interfaces
- [packages/pages-component/src/model/action-types.ts] — action button TypeScript interfaces
- [packages/pages-data/src/dataset/lookup-parser.ts] — existing Zod lookup schema (module-private)
- [packages/pages-data/src/dataset/external/schema.ts] — existing Zod dataset-def schema (module-private)
- [GE-20260813-674be0] — YAML desugarer drops unknown component props silently (motivates single source of truth)
- [GE-20260823-590f19] — Three-point registration pain (motivates schema registry)
- [GE-20260615-8cd96f] — TypeScript generic re-export cannot widen constraint (informs schema typing approach)
- [casehubio/casehub-pages#372] — pages-code-editor component (parent work)
- [casehubio/casehub-pages#407] — LSP server (downstream consumer of these schemas)
- [docs/specs/issue-372-pages-code-editor/2026-09-04-pages-code-editor-design.md] — code editor design spec
