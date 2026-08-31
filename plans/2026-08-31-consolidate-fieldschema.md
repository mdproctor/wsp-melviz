# Consolidate FieldSchema Types — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #392 — chore: consolidate dual FieldSchema types across pages-component and pages-ui-components
**Issue group:** #392

**Goal:** Consolidate three `FieldSchema` definitions into a single canonical interface in `pages-component/src/model/form-input-types.ts`, with no cross-package re-exports.

**Architecture:** Merge the superset fields from `pages-ui-components/src/types.ts` into `pages-component`'s `FieldSchema`. Delete the duplicate from `displayer-types.ts` (import from `form-input-types.ts` instead). Delete from `pages-ui-components`. Update all consumer imports to point directly to `@casehubio/pages-component`. Remove re-exports from `pages-viz/schema-types.ts` and `pages-property-palette/types.ts`.

**Tech Stack:** TypeScript

## Global Constraints

- **No cross-package re-exports** — every consumer imports `FieldSchema` directly from `@casehubio/pages-component` or from the internal path within pages-component
- Intra-package barrels (`index.ts`) are fine
- All changes must land atomically — partial migration breaks typecheck

---

## Batch 1: Consolidate and migrate

### Task 1: Merge superset, delete duplicates, update all imports

**Files:**
- Modify: `packages/pages-component/src/model/form-input-types.ts:55-69` — merge superset fields
- Modify: `packages/pages-component/src/model/displayer-types.ts:233-247` — delete duplicate, add import
- Modify: `packages/pages-ui-components/src/types.ts` — remove FieldSchema
- Modify: `packages/pages-ui-components/src/index.ts:1` — remove FieldSchema from export
- Modify: `packages/pages-ui-components/src/validation/validate-field.ts:1` — change import
- Modify: `packages/pages-ui-components/src/validation/validate-field.test.ts:3` — change import
- Modify: `packages/pages-property-palette/src/types.ts:1,4` — change import, remove re-export
- Modify: `packages/pages-property-palette/src/resolver.ts:1` — change import
- Modify: `packages/pages-property-palette/src/palette/pages-property-palette.ts:4` — change import
- Modify: `packages/pages-viz/src/form-inputs/schema-types.ts:3` — remove re-export line

**Interfaces:**
- Consumes: existing `FieldSchema` consumers across 5 packages
- Produces: single canonical `FieldSchema` in `@casehubio/pages-component`

- [ ] **Step 1: Merge superset fields into pages-component's FieldSchema**

In `packages/pages-component/src/model/form-input-types.ts`, replace the current `FieldSchema` interface (lines 55-69) with the merged superset:

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

Use `ide_replace_text_in_file` to replace the old interface with the new one.

- [ ] **Step 2: Delete duplicate from displayer-types.ts and add import**

In `packages/pages-component/src/model/displayer-types.ts`:

1. Add import at the top (after line 2):
```typescript
import type { FieldSchema } from "./form-input-types.js";
```

2. Delete the `FieldSchema` interface block (lines 233-247) entirely. `SchemaFormProps` and `FormScopeProps` will now reference the imported `FieldSchema`.

- [ ] **Step 3: Remove FieldSchema from pages-ui-components/types.ts**

Replace `packages/pages-ui-components/src/types.ts` content — remove the entire `FieldSchema` interface, keeping only `SelectOption`:

```typescript
export interface SelectOption {
  readonly value: string;
  readonly label: string;
}
```

- [ ] **Step 4: Remove FieldSchema from pages-ui-components barrel export**

In `packages/pages-ui-components/src/index.ts`, change line 1:

From: `export type { SelectOption, FieldSchema } from './types.js';`
To: `export type { SelectOption } from './types.js';`

- [ ] **Step 5: Update pages-ui-components validation imports**

In `packages/pages-ui-components/src/validation/validate-field.ts`, change line 1:

From: `import type { FieldSchema } from '../types.js';`
To: `import type { FieldSchema } from '@casehubio/pages-component';`

In `packages/pages-ui-components/src/validation/validate-field.test.ts`, change line 3:

From: `import type { FieldSchema } from '../types.js';`
To: `import type { FieldSchema } from '@casehubio/pages-component';`

- [ ] **Step 6: Update pages-property-palette imports**

In `packages/pages-property-palette/src/types.ts`:
- Change line 1: `import type { FieldSchema } from '@casehubio/pages-component';`
- Remove line 4: `export type { FieldSchema };` (this was a re-export — delete it)

In `packages/pages-property-palette/src/resolver.ts`:
- Change line 1: `import type { FieldSchema } from '@casehubio/pages-component';`

In `packages/pages-property-palette/src/palette/pages-property-palette.ts`:
- Change line 4: `import type { FieldSchema } from '@casehubio/pages-component';`

- [ ] **Step 7: Remove FieldSchema re-export from pages-viz/schema-types.ts**

In `packages/pages-viz/src/form-inputs/schema-types.ts`, remove line 3 (the re-export):

Delete: `export type { FieldSchema, SchemaFormProps } from "@casehubio/pages-component";`

Lines 4-6 already import `FieldSchema` directly from `@casehubio/pages-component` for use in the functions — keep those.

- [ ] **Step 8: Build packages and run typecheck**

Run: `yarn build:packages`
Then: `yarn typecheck`
Expected: No new type errors. The `type` field widened from `string` to `string | readonly string[]` — existing consumers that only pass `string` remain compatible.

- [ ] **Step 9: Run all tests**

Run: `yarn workspace @casehubio/pages-component run test -- --run`
Run: `yarn workspace @casehubio/pages-viz run test -- --run`
Expected: All tests pass — no behavioral changes

- [ ] **Step 10: Verify no duplicate definitions remain**

Run: `grep -rn "interface FieldSchema" packages/ --include="*.ts" | grep -v node_modules | grep -v .typecheck | grep -v target`
Expected: Exactly one result: `packages/pages-component/src/model/form-input-types.ts`

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/form-input-types.ts packages/pages-component/src/model/displayer-types.ts packages/pages-ui-components/src/types.ts packages/pages-ui-components/src/index.ts packages/pages-ui-components/src/validation/validate-field.ts packages/pages-ui-components/src/validation/validate-field.test.ts packages/pages-property-palette/src/types.ts packages/pages-property-palette/src/resolver.ts packages/pages-property-palette/src/palette/pages-property-palette.ts packages/pages-viz/src/form-inputs/schema-types.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "chore(#392): consolidate FieldSchema to single canonical definition in pages-component

Merge superset fields from pages-ui-components into pages-component's
form-input-types.ts. Delete duplicates from displayer-types.ts and
pages-ui-components/types.ts. Update all consumer imports to point
directly to @casehubio/pages-component. No cross-package re-exports.

Closes #392"
```

## References

- [2026-08-31-consolidate-fieldschema-design.md] — design spec
- `packages/pages-component/src/model/form-input-types.ts:55-69` — canonical location (subset)
- `packages/pages-ui-components/src/types.ts:6-32` — superset to merge
- `packages/pages-component/src/model/displayer-types.ts:233-247` — duplicate to delete
- `packages/pages-viz/src/form-inputs/schema-types.ts:3` — re-export to remove
- `packages/pages-property-palette/src/types.ts:4` — re-export to remove
- [GitHub #392] — issue
