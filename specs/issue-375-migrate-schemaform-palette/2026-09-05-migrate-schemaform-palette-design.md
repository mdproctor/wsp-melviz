# Design: Migrate PagesSchemaForm to embed pages-property-palette

**Issue:** casehubio/casehub-pages#375
**Branch:** `issue-375-migrate-schemaform-palette`
**Date:** 2026-09-05

## Overview

PagesSchemaForm in `pages-viz` currently implements its own field rendering loop — creating DOM elements imperatively, mapping schema types to component tags via `mapFieldToComponentType()`, and managing element caching, property setting, and stale element cleanup. This duplicates the palette's `resolveEditor()` for leaf types and diverges as the palette gains richer resolution.

This migration makes PagesSchemaForm a thin wrapper: it builds a `PropertyPaletteSource` from its TypedDataSet and renders `<pages-property-palette>` internally. The public API (`validate()`, `submit()`, `currentValue`, events) is unchanged. Consumers don't need to know the palette exists.

## Prerequisites

### Palette element caching (D4)

The palette's `_renderTagEditor` currently calls `document.createElement(tag)` on every render cycle with no element reuse. This causes text inputs to lose focus, dropdowns to close, and partially entered values to be lost.

**Fix:** Add an element cache keyed by field path to `PagesPropertyPalette`. On re-render, reuse the existing element if its tag name hasn't changed; create a new one only if the tag changes or the field is new. Clean up stale entries when fields are removed from the schema.

```typescript
private _elementCache: Map<string, HTMLElement> = new Map();
```

In `_renderTagEditor`, replace:
```typescript
const el = document.createElement(descriptor.tag);
```
With:
```typescript
const cacheKey = path.concat(key).join('.');
let el = this._elementCache.get(cacheKey);
if (!el || el.tagName.toLowerCase() !== descriptor.tag) {
  el = document.createElement(descriptor.tag);
  this._elementCache.set(cacheKey, el);
}
```

After rendering all fields, prune stale cache entries (fields no longer in the schema).

This fix is in `pages-property-palette`, not `pages-viz`. It benefits all palette consumers.

## Architecture

### Data Flow

```
TypedDataSet → PagesSchemaForm → PropertyPaletteSource → <pages-property-palette>
                    ↓                     ↓
              data mirror ←──── source.onChange
                    ↓
         validate() / submit() / currentValue
```

1. **Schema resolution:** PagesSchemaForm resolves the schema (from props or derived from dataset via `deriveSchemaFromDataSet`). `$ref` is resolved via `resolveSchemaRefs`.
2. **Schema enrichment (D5):** For select fields with no `enum`, PagesSchemaForm scans all dataset rows to extract distinct values and populates a cloned schema's `enum` array.
3. **Data extraction (D2):** PagesSchemaForm reads the first dataset row's cells and builds a flat `Record<string, unknown>` for `source.data`.
4. **Data mirror (D6):** The extracted data is also stored as `this._dataMirror` — the single source of truth for `currentValue`.
5. **Source construction:** PagesSchemaForm builds a `PropertyPaletteSource` with the enriched schema, extracted data, and an `onChange` callback that updates the data mirror.
6. **Rendering:** PagesSchemaForm's `renderContent` returns a Lit template containing `<pages-property-palette .source=${this._source} .resolver=${this._resolver}>`.

### Custom Resolver for Composite Types (D1)

PagesSchemaForm provides a custom `EditorResolver` that handles composite types before the palette's default resolver:

```typescript
private _resolver: EditorResolver = (schema: FieldSchema) => {
  if (schema['x-renderer']) {
    return { kind: 'tag', tag: `pages-${String(schema['x-renderer'])}` };
  }
  if (schema.oneOf) {
    return { kind: 'render', render: (ctx) => this._renderVariantGroup(ctx) };
  }
  const effectiveType = Array.isArray(schema.type)
    ? schema.type.find(t => t !== 'null') : schema.type;
  if (effectiveType === 'array' || schema.items) {
    return { kind: 'render', render: (ctx) => this._renderArrayGroup(ctx) };
  }
  if (effectiveType === 'object' || (schema.properties && effectiveType !== 'string')) {
    return { kind: 'render', render: (ctx) => this._renderObjectGroup(ctx) };
  }
  return undefined; // fall through to palette default resolver
};
```

The render functions create/reuse PagesObjectGroup, PagesArrayGroup, or PagesVariantGroup elements, passing schema, value, editable state, and wiring onChange to flow through `source.onChange`.

Note: the composites are pipeline-agnostic (they extend `FormValueMixin(LitElement)`, not `PagesElement`). Moving them into the palette is a viable follow-up that would eliminate the custom resolver entirely and give all palette consumers native array/variant support.

### Data Mirror and Public API (D3, D6)

PagesSchemaForm maintains a `_dataMirror: Record<string, unknown>` that is:
- **Pre-seeded** from the initial dataset extraction (same data passed to `source.data`)
- **Updated** via `source.onChange` callbacks during user interaction

This ensures `currentValue` returns correct values at all times — including before any user interaction.

```typescript
private _dataMirror: Record<string, unknown> = {};

get currentValue(): Record<string, unknown> {
  return { ...this._dataMirror };
}
```

`validate()` delegates to the palette for leaf fields (reading error state) and to composite components via `FormValueProvider.validate()` for composites. `submit()` reads `currentValue`, fires `pages-record-create`, and announces via the live region — unchanged from current behavior.

### Schema Enrichment for Select Options (D5)

When building the PropertyPaletteSource, PagesSchemaForm clones schema properties for fields where:
- The field type resolves to `select` (string with no enum, but dataset has distinct values)
- `extractDistinctValues()` returns a non-empty set

```typescript
private _enrichSchema(schema: FieldSchema, dataset: TypedDataSet): FieldSchema {
  const props = schema.properties;
  if (!props) return schema;
  const enriched: Record<string, FieldSchema> = {};
  let changed = false;
  for (const [key, fieldSchema] of Object.entries(props)) {
    if (fieldSchema.type === 'string' && !fieldSchema.enum) {
      const distinct = this.extractDistinctValues(key, dataset);
      if (distinct.length > 0) {
        enriched[key] = { ...fieldSchema, enum: distinct };
        changed = true;
        continue;
      }
    }
    enriched[key] = fieldSchema;
  }
  return changed ? { ...schema, properties: enriched } : schema;
}
```

### Editable / Display Mode

The palette's `PropertyPaletteSource.readonly` maps to PagesSchemaForm's display/editable state:
- `props.mode === 'display' || !this._editable` → `source.readonly = true`

### fieldsOnly Mode

When `fieldsOnly` is true, PagesSchemaForm currently dispatches `pages-field-register` events for each child element. After migration, the palette owns the child elements. PagesSchemaForm accesses them via the custom resolver's render functions (for composites) and via the palette's element cache (for leaf fields — requires the palette to expose a `getFieldElement(key)` method or PagesSchemaForm to query the palette's shadow DOM).

Alternative: PagesSchemaForm can continue to emit field-register events by querying the palette after render. The palette's `updateComplete` promise signals when rendering is done.

## Files Changed

| File | Change |
|------|--------|
| `packages/pages-property-palette/src/palette/pages-property-palette.ts` | Add element cache (D4), prune stale entries |
| `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` | Rewrite to thin wrapper: build source, render palette, maintain data mirror |
| `packages/pages-viz/src/form-inputs/schema-types.ts` | Keep `deriveSchemaFromDataSet` and `validateField` re-export; `mapFieldToComponentType` retained for composite components (they import it directly) |
| `packages/pages-viz/package.json` | Add `@casehubio/pages-property-palette` dependency |
| `packages/pages-viz/src/form-inputs/schema-form.test.ts` | Update tests — same public API, same behavior |
| `packages/pages-viz/src/form-inputs/schema-types.test.ts` | Keep `mapFieldToComponentType` tests (composites still use it) |

## Not in Scope

- **Moving composites to the palette** (R1-01 follow-up) — viable since they're pipeline-agnostic, but expands scope. File as follow-up issue.
- **PagesSchemaForm no longer extending PagesElement** (R1-05) — architectural improvement but changes the public contract. Separate issue.
- **Relocating PagesSchemaForm to pages-property-palette** (R1-06) — dependency direction concern. Separate architectural decision.

## Testing Strategy

- **Unit tests (schema-types.ts):** `mapFieldToComponentType` tests unchanged — composites still use it
- **Unit tests (PagesSchemaForm):** Existing test suite exercises the public API (`currentValue`, `validate`, `submit`, field rendering, events). After migration, all tests must pass unchanged — the internal switch to the palette is invisible to consumers.
- **Palette element caching tests:** New tests in `pages-property-palette` — verify element reuse across re-renders, stale entry cleanup, tag change creates new element
- **Integration:** Schema enrichment (select options from dataset), data mirror initialisation, composite types via custom resolver, validateOnBlur, fieldsOnly mode

## References

- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — migration source (375 lines)
- `packages/pages-viz/src/form-inputs/schema-types.ts` — `mapFieldToComponentType`, `deriveSchemaFromDataSet`
- `packages/pages-property-palette/src/palette/pages-property-palette.ts` — rendering target
- `packages/pages-property-palette/src/resolver.ts` — `resolveEditor` default resolver
- `packages/pages-property-palette/src/types.ts` — `PropertyPaletteSource`, `EditorResolver`, `EditorDescriptor`
- `packages/pages-viz/src/form-inputs/PagesObjectGroup.ts` — composite (pipeline-agnostic)
- `packages/pages-viz/src/form-inputs/PagesArrayGroup.ts` — composite (pipeline-agnostic)
- `packages/pages-viz/src/form-inputs/PagesVariantGroup.ts` — composite (pipeline-agnostic)
- `docs/specs/issue-373-property-palette/2026-08-26-property-palette-design.md` — palette architecture
- `docs/specs/issue-373-property-palette/decisions.md` — D5 (custom resolver design)
- `docs/specs/issue-392-consolidate-fieldschema/2026-08-31-consolidate-fieldschema-design.md` — FieldSchema consolidation
- Decision review R1-01 through R1-08 — `reviews/casehub-pages/issue-375-decision-20260905-034524/`
- casehubio/casehub-pages#373 — property palette (CLOSED, prerequisite)
- casehubio/casehub-pages#392 — FieldSchema consolidation (CLOSED, prerequisite)
