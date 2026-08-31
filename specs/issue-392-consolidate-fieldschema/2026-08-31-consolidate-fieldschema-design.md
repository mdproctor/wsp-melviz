# Consolidate dual FieldSchema types

**Issue:** casehubio/casehub-pages#392
**Date:** 2026-08-31

## Problem

Three `FieldSchema` interface definitions exist across two packages:

1. `pages-component/src/model/form-input-types.ts:55` — basic version
2. `pages-component/src/model/displayer-types.ts:233` — identical copy of #1
3. `pages-ui-components/src/types.ts:6` — superset with JSON Schema fields

Plus a re-export in `pages-viz/src/form-inputs/schema-types.ts`.

This confuses downstream LLMs that see the same type defined in multiple
places with different shapes.

## Design

### Single canonical definition in `pages-component`

Merge the superset fields from `pages-ui-components` into
`pages-component/src/model/form-input-types.ts`. This becomes the single
canonical `FieldSchema`. The merged interface:

```typescript
export interface FieldSchema {
  readonly type?: string | readonly string[];
  readonly format?: string;
  readonly title?: string;
  readonly description?: string;
  /** @deprecated Use x-placeholder instead */
  readonly placeholder?: string;
  readonly enum?: readonly string[];
  readonly pattern?: string;
  readonly minimum?: number;
  readonly maximum?: number;
  readonly exclusiveMinimum?: number;
  readonly exclusiveMaximum?: number;
  readonly minLength?: number;
  readonly maxLength?: number;
  readonly minItems?: number;
  readonly maxItems?: number;
  readonly uniqueItems?: boolean;
  readonly multipleOf?: number;
  readonly readOnly?: boolean;
  readonly properties?: Readonly<Record<string, FieldSchema>>;
  readonly required?: readonly string[];
  readonly items?: FieldSchema;
  readonly const?: string | number | boolean | null;
  readonly oneOf?: readonly FieldSchema[];
  readonly [key: `x-${string}`]: unknown;
}
```

### Deletions

1. **Delete** `FieldSchema` from `pages-component/src/model/displayer-types.ts`
   — it imported/used the one from `form-input-types.ts` internally anyway
2. **Delete** `FieldSchema` from `pages-ui-components/src/types.ts`
   — replace with import from `@casehubio/pages-component`
3. **Delete** re-export from `pages-viz/src/form-inputs/schema-types.ts`
   — consumers import directly from `@casehubio/pages-component`

### Consumer migration (no re-exports)

Every file that imports `FieldSchema` must import directly from
`@casehubio/pages-component` (or from the internal path within
pages-component itself for internal consumers).

| Consumer | Current import | New import |
|----------|---------------|-----------|
| `pages-ui-components/validation/validate-field.ts` | `../types.js` | `@casehubio/pages-component` |
| `pages-ui-components/validation/validate-field.test.ts` | `../types.js` | `@casehubio/pages-component` |
| `pages-property-palette/src/types.ts` | `@casehubio/pages-ui-components/types` | `@casehubio/pages-component` |
| `pages-property-palette/src/resolver.ts` | `@casehubio/pages-ui-components/types` | `@casehubio/pages-component` |
| `pages-property-palette/src/palette/pages-property-palette.ts` | `@casehubio/pages-ui-components/types` | `@casehubio/pages-component` |
| `pages-viz/src/form-inputs/PagesSchemaForm.ts` | `@casehubio/pages-component` | unchanged |
| `pages-viz/src/form-inputs/schema-types.ts` | re-exports from `@casehubio/pages-component` | delete re-export, keep `deriveSchemaFromDataSet` with direct import |
| `pages-runtime/src/form-scope.ts` | `@casehubio/pages-component` | unchanged |
| `pages-component` internal files | `./form-input-types.js` | unchanged |

### pages-ui-components types.ts after cleanup

```typescript
export interface SelectOption {
  readonly value: string;
  readonly label: string;
}
```

`FieldSchema` is removed entirely — no re-export, no local definition.

## Test plan

1. `yarn typecheck` passes with no new errors
2. All existing tests pass — no behavioral changes, only import paths
3. `grep -r "FieldSchema" packages/` confirms single definition in
   `form-input-types.ts`, no duplicates

## Files changed

| File | Change |
|------|--------|
| `packages/pages-component/src/model/form-input-types.ts` | Merge superset fields |
| `packages/pages-component/src/model/displayer-types.ts` | Remove duplicate, import from form-input-types |
| `packages/pages-ui-components/src/types.ts` | Remove FieldSchema |
| `packages/pages-ui-components/src/index.ts` | Remove FieldSchema export |
| `packages/pages-ui-components/src/validation/validate-field.ts` | Change import |
| `packages/pages-ui-components/src/validation/validate-field.test.ts` | Change import |
| `packages/pages-property-palette/src/types.ts` | Change import, remove re-export |
| `packages/pages-property-palette/src/resolver.ts` | Change import |
| `packages/pages-property-palette/src/palette/pages-property-palette.ts` | Change import |
| `packages/pages-viz/src/form-inputs/schema-types.ts` | Remove FieldSchema re-export |

## References

- `packages/pages-component/src/model/form-input-types.ts:55` — canonical location
- `packages/pages-ui-components/src/types.ts:6` — superset to merge
- `packages/pages-component/src/model/displayer-types.ts:233` — duplicate to delete
- casehubio/casehub-pages#392 — issue
