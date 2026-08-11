# Selection-Driven Dataset Builders + YAML Desugaring

**Issue:** casehubio/casehub-pages#299 (part of epic #289)
**Date:** 2026-08-11
**Scope:** #299 only — selection context bridge (#298, done) and host-panel forwarding (#300) are separate issues

## Problem

The selection context bridge (#298) wired `PagesDataTable` selection events into `RuntimeContext.selection` and enabled template resolution for `#{selection.xxx.yyy}` in dataset URLs. But there's no ergonomic API for declaring selection-driven datasets — authors must manually construct `ExternalDataSetDef` objects with template URLs and understand the runtime plumbing.

Five casehub-clinical master-detail patterns need a clean builder API and YAML surface for declaring detail datasets that auto-fetch when a master table row is selected.

## Design

### 1. Type Change — ExternalDataSetDef

Add `selectionSource` to `ExternalDataSetDef` in `packages/pages-data/src/dataset/external/types.ts`:

```typescript
export interface ExternalDataSetDef extends ExtractionDef {
  // ... existing fields ...
  readonly selectionSource?: string;
}
```

`selectionSource` is the dataset ID whose selection drives this source's URL template resolution. It is **pure metadata** for v1 — the URL template `#{selection.xxx.yyy}` does all the runtime work via the existing `ContextConsumer` pipeline on the `handleDefRequest` path. `selectionSource` serves as documentation, `detailDataset()` enabler, YAML ergonomics, and a future hook for tooling/validation.

No changes to `RestSourceOptions`, `DataSourceBinding`, or `bind()`. The field lives exclusively on `ExternalDataSetDef` because template resolution only exists on the `handleDefRequest` path in `data-pipeline.ts`. The `handleBindingRequest` path (used by `DataSourceBinding`) connects sources directly with no template awareness.

### 2. Builder API — detailDataset()

Add to `packages/pages-ui/src/dsl/builders.ts`:

```typescript
export function detailDataset(
  id: string,
  selectionSource: string,
  url: string,
  options?: Omit<ExternalDataSetDef, 'uuid' | 'url' | 'selectionSource'>,
): ExternalDataSetDef {
  return {
    uuid: dataSetId(id),
    url,
    selectionSource,
    ...options,
  };
}
```

Export from `packages/pages-ui/src/dsl/index.ts` alongside the existing DSL builders.

**Usage:**

```typescript
import { detailDataset } from "@casehubio/pages-ui";

const gradeHistory = detailDataset(
  "grade-history",
  "adverse-events",
  "/api/ae/#{selection.adverse-events.id}/history",
);
```

The optional `options` bag accepts all other `ExternalDataSetDef` fields (`dataPath`, `columns`, `expression`, `refreshTime`, etc.) for cases where the detail dataset needs extraction or polling configuration.

**Return type is `ExternalDataSetDef`**, not `DataSourceBinding`. This is load-bearing: `ExternalDataSetDef` enters the `handleDefRequest` path in `data-pipeline.ts`, which registers a `ContextConsumer`, resolves `#{...}` templates, and gates fetches via `allTemplateVarsResolved()`. A `DataSourceBinding` would enter `handleBindingRequest`, which connects the source directly with no template resolution — template placeholders would be fetched literally.

### 3. YAML Surface

No parser changes needed. `selectionSource` passes through the existing YAML pipeline without transformation:

```yaml
datasets:
  - uuid: adverse-events
    url: "/api/adverse-events"

  - uuid: grade-history
    url: "/api/ae/#{selection.adverse-events.id}/history"
    selectionSource: adverse-events
```

The raw YAML property lands on `ExternalDataSetDef` directly — `page-parser.ts` passes datasets as raw objects into root component props, and the runtime interprets them as `ExternalDataSetDef`. No changes to `page-parser.ts`, `component-desugar.ts`, `page-schema.ts`, or `defToBinding()`.

### 4. Runtime Behaviour (No Changes)

The existing runtime pipeline handles selection-driven datasets without modification:

1. `handleDefRequest` registers a `ContextConsumer` for the dataset's URL
2. `ContextConsumer` resolves `#{selection.adverse-events.id}` via the dot-path walker against `RuntimeContext.selection`
3. `allTemplateVarsResolved()` returns `false` when no selection exists → fetch is deferred
4. When `updateSelection("adverse-events", row)` is called (from the selection-change listener added by #298), `evaluateAll()` re-runs all consumers
5. The template resolves to a real value → `allTemplateVarsResolved()` returns `true` → the consumer fetches the detail dataset with the resolved URL
6. When selection is cleared, the template becomes unresolved again → detail dataset stops fetching

`selectionSource` is not read by the runtime at all in v1. The template in the URL fully encodes the dependency.

## Scope Boundaries

**In scope (#299):**
- `selectionSource?: string` on `ExternalDataSetDef`
- `detailDataset()` builder function in pages-ui DSL
- Export from pages-ui DSL index
- Unit tests for `detailDataset()` builder
- Integration test for YAML selectionSource passthrough
- Integration test for end-to-end template resolution with selection

**Out of scope:**
- Runtime behaviour changes — existing pipeline handles everything
- `DataSourceBinding` or `bind()` changes — not needed
- `RestSourceOptions` changes — not needed
- host-panel `pages-selection-changed` event forwarding → #300
- `:field` shorthand syntax (auto-expanding to `#{selection.source.field}`) — future enhancement to eliminate redundancy between `selectionSource` param and URL template
- Parse-time validation that `selectionSource` references an existing dataset — future enhancement
- Runtime enforcement that `selectionSource` matches template references — future enhancement

## Files Modified

| File | Change |
|------|--------|
| `packages/pages-data/src/dataset/external/types.ts` | Add `selectionSource?: string` to `ExternalDataSetDef` |
| `packages/pages-ui/src/dsl/builders.ts` | Add `detailDataset()` builder function |
| `packages/pages-ui/src/dsl/index.ts` | Export `detailDataset` |
| `packages/pages-ui/src/dsl/builders.test.ts` | Unit tests for `detailDataset()` |
| `packages/pages-ui/src/parser/page-parser.test.ts` | Integration test: YAML selectionSource passthrough |

## Testing Strategy

- **Unit:** `detailDataset()` returns correct `ExternalDataSetDef` shape — `uuid`, `url`, `selectionSource` set, optional fields forwarded
- **Unit:** `detailDataset()` with options bag (`dataPath`, `columns`) merges correctly
- **Integration:** YAML with `selectionSource` parses into a root component whose dataset has the field intact
- **Integration:** End-to-end template resolution — dataset with `url: "/api/#{selection.master.id}"` defers fetch until selection exists, fetches with resolved URL when `updateSelection` is called, stops fetching when selection is cleared
- **Edge cases:** Missing selection (template unresolved → no fetch), selection cleared (detail stops), multiple detail datasets bound to same master, `selectionSource` declared but no templates in URL (harmless)
