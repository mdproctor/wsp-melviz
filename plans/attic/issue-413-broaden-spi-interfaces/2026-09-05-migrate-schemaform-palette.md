# PagesSchemaForm Palette Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #375 — refactor(pages-viz): migrate PagesSchemaForm to embed pages-property-palette
**Issue group:** #375

**Goal:** Make PagesSchemaForm a thin wrapper that renders `<pages-property-palette>` internally, eliminating the duplicate field-rendering loop while preserving the public API.

**Architecture:** Two-phase migration. Phase 1 adds element caching and accessor methods to the palette (prerequisite fix for all consumers). Phase 2 rewrites PagesSchemaForm to build a `PropertyPaletteSource` from its TypedDataSet, provide a custom `EditorResolver` for composite types (object-group, array-group, variant-group), and maintain a data mirror for validate/submit/currentValue.

**Tech Stack:** TypeScript, Lit 3, vitest, @casehubio/pages-property-palette, @casehubio/pages-viz

## Global Constraints

- Public API of `<pages-schema-form>` must not change — consumers are unaware of internal migration
- `pages-property-palette` must not gain data pipeline dependencies (`@casehubio/pages-data`)
- All existing tests must continue to pass unchanged
- Composite types (object-group, array-group, variant-group) stay in `pages-viz` — moving to palette is a follow-up

---

## Batch 1: Palette element caching and accessor methods

### Task 1: Add element cache to PagesPropertyPalette

**Files:**
- Modify: `packages/pages-property-palette/src/palette/pages-property-palette.ts`
- Modify: `packages/pages-property-palette/src/palette/pages-property-palette.test.ts`

**Interfaces:**
- Produces: `PagesPropertyPalette._elementCache: Map<string, HTMLElement>` (private), `PagesPropertyPalette.getFieldElement(key: string): HTMLElement | undefined` (public), `PagesPropertyPalette.setFieldErrors(errors: Map<string, string | undefined>): void` (public)

- [ ] **Step 1: Write failing test — element reuse across re-renders**

```typescript
it('reuses DOM elements across re-renders', async () => {
  const onChange = () => {};
  (el as any).source = {
    schema: { properties: { name: { type: 'string', title: 'Name' } } },
    data: { name: 'hello' },
    onChange,
  } satisfies PropertyPaletteSource;
  await (el as any).updateComplete;

  const firstInput = el.shadowRoot!.querySelector('pages-input');
  expect(firstInput).not.toBeNull();

  // Trigger re-render by changing data
  (el as any).source = {
    schema: { properties: { name: { type: 'string', title: 'Name' } } },
    data: { name: 'world' },
    onChange,
  };
  await (el as any).updateComplete;

  const secondInput = el.shadowRoot!.querySelector('pages-input');
  expect(secondInput).toBe(firstInput); // same DOM element
  expect((secondInput as any).value).toBe('world');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/pages-property-palette test -- --reporter verbose -t "reuses DOM elements"`
Expected: FAIL — palette creates fresh elements on each render

- [ ] **Step 3: Add `_elementCache` field and modify `_renderTagEditor` to cache elements**

In `pages-property-palette.ts`, add the cache field after `_errors`:

```typescript
private _elementCache: Map<string, HTMLElement> = new Map();
```

In `_renderTagEditor`, replace `const el = document.createElement(tag) as any;` (line 220) with element-reuse logic:

```typescript
const cacheKey = [...path, key].join('.');
let el = this._elementCache.get(cacheKey) as any;
if (!el || el.tagName.toLowerCase() !== tag) {
  el = document.createElement(tag);
  this._elementCache.set(cacheKey, el);
}
```

After the `render()` method's field rendering loop, add stale cache pruning. In `render()`, after building all field entries and rendering them, collect the active keys and prune:

```typescript
const activeKeys = new Set<string>();
// ... collect keys during rendering ...
for (const key of this._elementCache.keys()) {
  if (!activeKeys.has(key)) {
    this._elementCache.delete(key);
  }
}
```

The cleanest approach: track active keys as a Set built during `_renderTagEditor` calls, then prune after the full render. Add a field `private _activeKeys: Set<string> = new Set()` that is cleared at the start of `render()` and populated in `_renderTagEditor`. After rendering, prune stale cache entries.

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd packages/pages-property-palette test -- --reporter verbose -t "reuses DOM elements"`
Expected: PASS

- [ ] **Step 5: Write failing test — `getFieldElement`**

```typescript
it('exposes field elements via getFieldElement', async () => {
  (el as any).source = {
    schema: {
      properties: {
        name: { type: 'string', title: 'Name' },
        age: { type: 'number', title: 'Age' },
      },
    },
    data: { name: 'test', age: 25 },
    onChange: () => {},
  } satisfies PropertyPaletteSource;
  await (el as any).updateComplete;

  const nameEl = (el as any).getFieldElement('name');
  expect(nameEl).not.toBeNull();
  expect(nameEl.tagName.toLowerCase()).toBe('pages-input');

  const ageEl = (el as any).getFieldElement('age');
  expect(ageEl).not.toBeNull();
  expect(ageEl.tagName.toLowerCase()).toBe('pages-number-input');

  const missing = (el as any).getFieldElement('nonexistent');
  expect(missing).toBeUndefined();
});
```

- [ ] **Step 6: Implement `getFieldElement`**

```typescript
getFieldElement(key: string): HTMLElement | undefined {
  return this._elementCache.get(key);
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `yarn --cwd packages/pages-property-palette test -- --reporter verbose -t "exposes field elements"`
Expected: PASS

- [ ] **Step 8: Write failing test — `setFieldErrors`**

```typescript
it('sets errors on field elements via setFieldErrors', async () => {
  (el as any).source = {
    schema: {
      properties: {
        name: { type: 'string', title: 'Name' },
        age: { type: 'number', title: 'Age' },
      },
    },
    data: { name: '', age: 25 },
    onChange: () => {},
  } satisfies PropertyPaletteSource;
  await (el as any).updateComplete;

  const errors = new Map<string, string | undefined>();
  errors.set('name', 'Required field');
  errors.set('age', undefined);
  (el as any).setFieldErrors(errors);
  await (el as any).updateComplete;

  const nameEl = (el as any).getFieldElement('name');
  expect(nameEl.error).toBe('Required field');

  const ageEl = (el as any).getFieldElement('age');
  expect(ageEl.error).toBeUndefined();
});
```

- [ ] **Step 9: Implement `setFieldErrors`**

```typescript
setFieldErrors(errors: Map<string, string | undefined>): void {
  for (const [key, error] of errors) {
    const el = this._elementCache.get(key) as any;
    if (el) {
      el.error = error;
    }
  }
  this._errors = new Map(errors);
  this.requestUpdate();
}
```

- [ ] **Step 10: Run all palette tests**

Run: `yarn --cwd packages/pages-property-palette test -- --reporter verbose`
Expected: ALL PASS — existing tests unaffected, 3 new tests pass

- [ ] **Step 11: Write failing test — stale cache cleanup**

```typescript
it('prunes cached elements when fields are removed from schema', async () => {
  (el as any).source = {
    schema: {
      properties: {
        name: { type: 'string', title: 'Name' },
        age: { type: 'number', title: 'Age' },
      },
    },
    data: { name: 'test', age: 25 },
    onChange: () => {},
  } satisfies PropertyPaletteSource;
  await (el as any).updateComplete;
  expect((el as any).getFieldElement('age')).toBeDefined();

  // Remove 'age' from schema
  (el as any).source = {
    schema: { properties: { name: { type: 'string', title: 'Name' } } },
    data: { name: 'test' },
    onChange: () => {},
  };
  await (el as any).updateComplete;

  expect((el as any).getFieldElement('age')).toBeUndefined();
  expect((el as any).getFieldElement('name')).toBeDefined();
});
```

- [ ] **Step 12: Run test to verify it passes (should pass from Step 3 implementation)**

Run: `yarn --cwd packages/pages-property-palette test -- --reporter verbose -t "prunes cached"`
Expected: PASS

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-property-palette/src/palette/pages-property-palette.ts packages/pages-property-palette/src/palette/pages-property-palette.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-property-palette): element caching, getFieldElement, setFieldErrors Refs #375"
```

---

## Batch 2: PagesSchemaForm migration

### Task 2: Rewrite PagesSchemaForm as palette thin wrapper

**Files:**
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`
- Modify: `packages/pages-viz/package.json` (add `@casehubio/pages-property-palette` dependency)
- Test: `packages/pages-viz/src/form-inputs/schema-form.test.ts` (existing — must continue to pass)

**Interfaces:**
- Consumes: `PagesPropertyPalette.getFieldElement(key: string): HTMLElement | undefined`, `PagesPropertyPalette.setFieldErrors(errors: Map<string, string | undefined>): void`, `PropertyPaletteSource` (from `@casehubio/pages-property-palette/types`), `EditorResolver` (from `@casehubio/pages-property-palette/types`), `FieldRenderContext` (from `@casehubio/pages-property-palette/types`)
- Produces: Same public API — `PagesSchemaForm.currentValue`, `.validate()`, `.submit()`, `.editable`, `.fieldsOnly`, events `pages-record-create`, `pages-field-change`, `pages-field-register`

- [ ] **Step 1: Add `@casehubio/pages-property-palette` dependency to pages-viz**

```bash
yarn --cwd packages/pages-viz add @casehubio/pages-property-palette
```

If the package is workspace-local, add it directly to package.json:

```json
"@casehubio/pages-property-palette": "workspace:*"
```

Then run `yarn install`.

- [ ] **Step 2: Run existing schema-form tests to establish baseline**

Run: `yarn --cwd packages/pages-viz test -- --reporter verbose -t "PagesSchemaForm"`
Expected: ALL PASS — this is the baseline all subsequent changes must preserve

- [ ] **Step 3: Rewrite PagesSchemaForm internals**

Replace the body of `PagesSchemaForm` with the thin-wrapper implementation. The class still extends `PagesElement`, still has the same public API. The internal change:

1. Remove: `_children`, `_childTypes` Maps, imperative element creation loop, `buildChildProps`, `extractDistinctValues` (move to `_enrichSchema`), direct DOM manipulation in `renderContent`
2. Add: `_dataMirror`, `_source`, `_resolver`, `_compositeRefs`, `_enrichSchema`
3. `renderContent` now builds a `PropertyPaletteSource` and returns `html\`<pages-property-palette .source=\${source} .resolver=\${resolver}>\``

Key implementation:

```typescript
import '@casehubio/pages-property-palette';
import type { PropertyPaletteSource, EditorResolver, FieldRenderContext } from '@casehubio/pages-property-palette/types';
import type { PagesPropertyPalette } from '@casehubio/pages-property-palette/palette';

export class PagesSchemaForm extends PagesElement<SchemaFormProps & { lookup?: DataSetLookup }> {
  private _dataMirror: Record<string, unknown> = {};
  private _resolvedSchema: FieldSchema | null = null;
  private _editable = false;
  private _fieldsOnly = false;
  private _liveRegion: HTMLElement | null = null;
  private _compositeRefs: Map<string, HTMLElement> = new Map();
  private _palette: PagesPropertyPalette | null = null;

  // ... editable, fieldsOnly, disconnectedCallback, announce unchanged ...

  private _resolver: EditorResolver = (schema: FieldSchema) => {
    if (schema['x-renderer']) {
      return { kind: 'tag', tag: `pages-${String(schema['x-renderer'])}` };
    }
    if (schema.oneOf) {
      return { kind: 'render', render: (ctx) => this._renderComposite('variant-group', ctx) };
    }
    const effectiveType = Array.isArray(schema.type)
      ? (schema.type as readonly string[]).find(t => t !== 'null') : schema.type;
    if (effectiveType === 'array' || schema.items) {
      return { kind: 'render', render: (ctx) => this._renderComposite('array-group', ctx) };
    }
    if (effectiveType === 'object' || (schema.properties && effectiveType !== 'string')) {
      return { kind: 'render', render: (ctx) => this._renderComposite('object-group', ctx) };
    }
    return undefined;
  };

  get currentValue(): Record<string, unknown> {
    const record: Record<string, unknown> = { ...this._dataMirror };
    for (const [field, child] of this._compositeRefs) {
      if (isFormValueProvider(child)) {
        record[field] = child.currentValue;
      }
    }
    return record;
  }

  protected override renderContent(
    props: SchemaFormProps & { lookup?: DataSetLookup },
    dataset: TypedDataSet,
  ): TemplateResult {
    const schema = props.schema
      ? resolveSchemaRefs(props.schema)
      : deriveSchemaFromDataSet(dataset);
    this._resolvedSchema = schema;

    const enrichedSchema = this._enrichSchema(schema, dataset);
    const sourceData = this._extractData(schema, dataset);
    this._dataMirror = { ...sourceData };

    const isDisplay = props.mode === 'display' || !this._editable;

    const source: PropertyPaletteSource = {
      schema: enrichedSchema,
      data: sourceData,
      readonly: isDisplay,
      onChange: (fieldPath, value) => {
        const key = typeof fieldPath[0] === 'string' ? fieldPath[0] : String(fieldPath[0]);
        this._dataMirror[key] = value;
        this.dispatchEvent(new CustomEvent('pages-field-change', {
          bubbles: true, composed: true,
          detail: { field: key, value, committed: true },
        }));
      },
    };

    return html`
      <div class="schema-form-fields" role="${isDisplay ? 'group' : 'form'}">
        <pages-property-palette
          .source=${source}
          .resolver=${this._resolver}
          @pages-property-palette-rendered=${() => { this._palette = this.shadowRoot?.querySelector('pages-property-palette') as PagesPropertyPalette ?? null; }}
        ></pages-property-palette>
        ${this._renderSubmitButton(props, dataset, isDisplay)}
      </div>
    `;
  }

  private _renderSubmitButton(props: SchemaFormProps, dataset: TypedDataSet, isDisplay: boolean): TemplateResult | typeof nothing {
    const isCreateMode = props.forceCreate === true || dataset.rows.length === 0;
    if (isCreateMode && !isDisplay && !this._fieldsOnly) {
      return html`<div class="submit-bar"><button class="submit-btn" @click=${() => this.submit()}>Submit</button></div>`;
    }
    return nothing;
  }

  private _enrichSchema(schema: FieldSchema, dataset: TypedDataSet): FieldSchema {
    const props = schema.properties;
    if (!props) return schema;
    const enriched: Record<string, FieldSchema> = {};
    let changed = false;
    for (const [key, fieldSchema] of Object.entries(props)) {
      if (fieldSchema.type === 'string' && !fieldSchema.enum) {
        const distinct = this._extractDistinctValues(key, dataset);
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

  private _extractData(schema: FieldSchema, dataset: TypedDataSet): Record<string, unknown> {
    if (dataset.rows.length === 0) return {};
    const row = dataset.rows[0]!;
    const data: Record<string, unknown> = {};
    const fields = Object.keys(schema.properties ?? {});
    for (const field of fields) {
      try {
        const cell = row.cell(field as ColumnId);
        if (cell.type !== 'NULL') {
          const fieldSchema = schema.properties?.[field];
          const effectiveType = Array.isArray(fieldSchema?.type)
            ? (fieldSchema!.type as readonly string[]).find(t => t !== 'null')
            : fieldSchema?.type;
          if (effectiveType === 'boolean') {
            data[field] = typeof cell.value === 'boolean' ? cell.value : String(cell.value).toLowerCase() === 'true';
          } else if (effectiveType === 'number' || effectiveType === 'integer') {
            const num = typeof cell.value === 'number' ? cell.value : parseFloat(String(cell.value));
            data[field] = isNaN(num) ? null : num;
          } else {
            data[field] = String(cell.value);
          }
        }
      } catch { /* column not found */ }
    }
    return data;
  }

  private _extractDistinctValues(field: string, dataset: TypedDataSet): string[] {
    const seen = new Set<string>();
    for (const row of dataset.rows) {
      try {
        const cell = row.cell(field as ColumnId);
        const raw = cellToRaw(cell);
        if (raw !== null) seen.add(String(raw));
      } catch { /* skip */ }
    }
    return [...seen].sort();
  }

  private _renderComposite(componentType: string, ctx: FieldRenderContext): TemplateResult {
    const tagName = `pages-${componentType}`;
    let child = this._compositeRefs.get(ctx.key);
    if (!child || child.tagName.toLowerCase() !== tagName) {
      child = document.createElement(tagName);
      this._compositeRefs.set(ctx.key, child);
    }
    (child as any).schema = ctx.schema;
    (child as any).label = ctx.schema.title ?? ctx.key.replace(/([A-Z])/g, ' $1').replace(/^./, s => s.toUpperCase());
    (child as any).fieldName = ctx.key;
    (child as any).editable = !ctx.readonly;
    (child as any).required = ctx.required;
    if (ctx.value != null) {
      try { (child as any).value = typeof ctx.value === 'string' ? JSON.parse(ctx.value) : ctx.value; } catch { /* not JSON */ }
    }
    return html`${child}`;
  }

  validate(): boolean {
    if (!this._resolvedSchema?.properties) return true;
    const requiredSet = new Set(this._resolvedSchema.required ?? []);
    let allValid = true;
    const fieldErrors = new Map<string, string | undefined>();

    for (const [field, fieldSchema] of Object.entries(this._resolvedSchema.properties)) {
      const compositeChild = this._compositeRefs.get(field);
      if (compositeChild && isFormValueProvider(compositeChild)) {
        if (!compositeChild.validate()) allValid = false;
      } else {
        const value = this._dataMirror[field];
        const error = validateField(fieldSchema, value, requiredSet.has(field));
        if (error) {
          fieldErrors.set(field, error);
          allValid = false;
        } else {
          fieldErrors.set(field, undefined);
        }
      }
    }

    if (this._palette && fieldErrors.size > 0) {
      this._palette.setFieldErrors(fieldErrors);
    }
    return allValid;
  }

  // submit() and announce() unchanged
}
```

- [ ] **Step 4: Run existing schema-form tests**

Run: `yarn --cwd packages/pages-viz test -- --reporter verbose -t "PagesSchemaForm"`
Expected: ALL PASS — public API behavior preserved

- [ ] **Step 5: Fix any test failures**

If tests fail due to timing (palette renders asynchronously), ensure tests await `el.updateComplete` and then `palette.updateComplete`. If the palette reference isn't available on first render, use `queueMicrotask` or `requestAnimationFrame` to allow the palette to mount.

- [ ] **Step 6: Run full pages-viz test suite**

Run: `yarn --cwd packages/pages-viz test -- --reporter verbose`
Expected: ALL PASS

- [ ] **Step 7: Run full palette test suite to confirm no regressions**

Run: `yarn --cwd packages/pages-property-palette test -- --reporter verbose`
Expected: ALL PASS

- [ ] **Step 8: Run typecheck**

Run: `yarn typecheck`
Expected: PASS — no type errors from the new dependency or API changes

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/form-inputs/PagesSchemaForm.ts packages/pages-viz/package.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-viz): migrate PagesSchemaForm to embed pages-property-palette Refs #375"
```

---

## Batch 3: Cleanup and fieldsOnly support

### Task 3: Wire fieldsOnly mode and remove dead code

**Files:**
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`
- Modify: `packages/pages-viz/src/form-inputs/schema-types.ts` (keep `deriveSchemaFromDataSet`, `mapFieldToComponentType` for composites, `validateField` re-export)
- Test: `packages/pages-viz/src/form-inputs/schema-form.test.ts` (fieldsOnly tests)

**Interfaces:**
- Consumes: `PagesPropertyPalette.getFieldElement(key: string)` from Task 1, `PagesPropertyPalette.updateComplete` from Lit lifecycle

- [ ] **Step 1: Run fieldsOnly-specific tests to establish baseline**

Run: `yarn --cwd packages/pages-viz test -- --reporter verbose -t "fieldsOnly"`
Expected: ALL PASS (baseline)

- [ ] **Step 2: Implement fieldsOnly mode using palette accessor**

In `renderContent`, after rendering the palette template, add fieldsOnly wiring. After the palette renders, query field elements and dispatch registration events:

```typescript
if (this._fieldsOnly) {
  const fields = Object.keys(enrichedSchema.properties ?? {});
  this.updateComplete.then(async () => {
    const palette = this.shadowRoot?.querySelector('pages-property-palette') as PagesPropertyPalette | null;
    if (!palette) return;
    await palette.updateComplete;
    for (const field of fields) {
      const element = this._compositeRefs.get(field) ?? palette.getFieldElement(field);
      if (element) {
        this.dispatchEvent(new CustomEvent('pages-field-register', {
          bubbles: true, composed: true,
          detail: { field, element, componentType: mapFieldToComponentType(enrichedSchema.properties![field]!) },
        }));
      }
    }
  });
}
```

- [ ] **Step 3: Run fieldsOnly tests**

Run: `yarn --cwd packages/pages-viz test -- --reporter verbose -t "fieldsOnly"`
Expected: ALL PASS

- [ ] **Step 4: Remove dead imports from PagesSchemaForm**

Remove imports that are no longer used after the migration:
- `PagesFormInput` type import (if no longer used)
- `STANDALONE_TYPES`, `readFieldValue`, `setFieldError` (if only used for old rendering loop)
- Side-effect imports for `@casehubio/pages-ui-components/*` (palette handles its own imports)

Keep: `resolveSchemaRefs`, `isFormValueProvider`, `deriveSchemaFromDataSet`, `mapFieldToComponentType` (used by composites and fieldsOnly), `validateField`, `cellToRaw`

- [ ] **Step 5: Run full test suite**

Run: `yarn --cwd packages/pages-viz test -- --reporter verbose`
Expected: ALL PASS

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/form-inputs/PagesSchemaForm.ts packages/pages-viz/src/form-inputs/schema-types.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-viz): fieldsOnly mode via palette accessor, remove dead imports Refs #375"
```

## References

- `specs/issue-375-migrate-schemaform-palette/2026-09-05-migrate-schemaform-palette-design.md` — design spec this plan implements
- `specs/issue-375-migrate-schemaform-palette/decisions.md` — D1-D6 decisions
- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — migration source (375 lines)
- `packages/pages-viz/src/form-inputs/schema-types.ts:38-60` — `mapFieldToComponentType` (retained for composites)
- `packages/pages-property-palette/src/palette/pages-property-palette.ts:203-280` — `_renderTagEditor` (element caching target)
- `packages/pages-property-palette/src/types.ts` — PropertyPaletteSource, EditorResolver
- `packages/pages-viz/src/form-inputs/PagesObjectGroup.ts` — composite component (pipeline-agnostic)
- `packages/pages-viz/src/form-inputs/PagesArrayGroup.ts` — composite component (pipeline-agnostic)
- `packages/pages-viz/src/form-inputs/PagesVariantGroup.ts` — composite component (pipeline-agnostic)
- `docs/specs/issue-373-property-palette/2026-08-26-property-palette-design.md` — palette architecture
- `reviews/casehub-pages/issue-375-decision-20260905-034524/` — decision review findings R1-01 through R1-08
- casehubio/casehub-pages#375
- casehubio/casehub-pages#373 (prerequisite — CLOSED)
- casehubio/casehub-pages#392 (prerequisite — CLOSED)
