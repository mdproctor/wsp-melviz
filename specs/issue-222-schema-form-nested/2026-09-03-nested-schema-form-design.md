# Nested Schema Form — Design Spec

**Issue:** casehubio/casehub-pages#222
**Date:** 2026-09-03
**Predecessor:** #159 (schema-form runtime integration — flat fields)
**Prerequisite:** #337 (formScope composable layout — must be complete before this spec's formScope integration can land; specifically: `pages-field-register` event handling in activation, `FormScopeState` instantiation, and `FormScopeRegistry` wiring)
**Related:** #392 (FieldSchema consolidation), #334 (schemaForm DSL builder)
**JSON Schema draft:** draft-07 (siblings alongside `$ref` are ignored per spec)

## Problem

`PagesSchemaForm` renders form inputs from a JSON Schema but handles only flat schemas — each property maps to a single leaf component. The `FieldSchema` type already supports nesting (`properties`, `items`, `required`, `oneOf`), but the rendering pipeline ignores these:

- `mapFieldToComponentType` has no case for `type: "object"` or `type: "array"` — they fall through to `"input"` (text)
- `PagesSchemaForm.renderContent` iterates `schema.properties` as flat fields only
- `TypedDataSet` is flat (rows × columns) — no concept of nested data
- `validateField` validates leaf types only — no recursive validation

This issue adds full nested schema support: objects, arrays (primitives and objects), recursive nesting, `oneOf` discriminated unions, `$ref` resolution, custom renderers, and array reordering. It also resolves the data model question: how nested schemas map to the flat TypedDataSet pipeline.

## Architecture

### Recursive Component Model

Every JSON Schema node maps to exactly one component type. PagesSchemaForm walks the schema tree and creates the right component at each node. Nesting is recursion; leaf inputs are the base case.

| Schema shape | Component | Value type |
|-------------|-----------|------------|
| `string` (no format) | `pages-input` | `string` |
| `string` + `enum` | `pages-select` | `string` |
| `string` + `format: date` | `pages-date-input` | `string` |
| `string` + `format: datetime-local` | `pages-datetime-input` | `string` |
| `string` + `format: textarea` | `pages-textarea` | `string` |
| `number` / `integer` | `pages-number-input` | `number` |
| `boolean` | `pages-checkbox` | `boolean` |
| `type: "object"` or has `properties` | **`pages-object-group`** | `Record<string, unknown>` |
| `type: "array"` or has `items` | **`pages-array-group`** | `unknown[]` |
| has `oneOf` | **`pages-variant-group`** | value of active variant |
| has `x-renderer` | named custom element | per renderer |

### Extended `mapFieldToComponentType`

```typescript
export function mapFieldToComponentType(fieldSchema: FieldSchema): string {
  if (fieldSchema["x-renderer"]) return String(fieldSchema["x-renderer"]);
  if (fieldSchema.oneOf) return "variant-group";

  // Normalize type — JSON Schema allows type arrays for nullable: ["string", "null"]
  const effectiveType = Array.isArray(fieldSchema.type)
    ? fieldSchema.type.find(t => t !== "null")
    : fieldSchema.type;

  if (effectiveType === "array" || fieldSchema.items) return "array-group";
  if (effectiveType === "object" || (fieldSchema.properties && effectiveType !== "string")) return "object-group";
  // existing leaf mappings unchanged
  if (effectiveType === "boolean") return "checkbox";
  if (effectiveType === "number") return "number-input";
  if (effectiveType === "integer") return "number-input";
  if (effectiveType === "string") {
    if (fieldSchema.enum && fieldSchema.enum.length > 0) return "select";
    if (fieldSchema.format === "date") return "date-input";
    if (fieldSchema.format === "datetime-local") return "datetime-input";
    if (fieldSchema.format === "textarea") return "textarea";
    return "input";
  }
  if (fieldSchema.enum && fieldSchema.enum.length > 0) return "select";
  return "input";
}
```

Priority order: `x-renderer` → `oneOf` → array → object → leaf. This ensures custom renderers always win, and structural types take precedence over leaf-type inference.

### Rendering Walk

PagesSchemaForm's `renderContent` currently iterates `schema.properties` and creates leaf inputs. The change: when `mapFieldToComponentType` returns a composite type (`object-group`, `array-group`, `variant-group`), PagesSchemaForm creates the composite component, passes the sub-schema and sub-value, and the composite recursively renders its children using the same mapping.

The recursion path:
1. PagesSchemaForm receives schema + dataset
2. For each property in `schema.properties`: call `mapFieldToComponentType`
3. If leaf → create leaf input (existing behavior)
4. If composite → create composite component, set `schema` (sub-schema) and `value` (sub-value)
5. Composite internally iterates its sub-schema and repeats from step 2

## FormValueProvider Protocol

### TypeScript Interface

All form components — leaf and composite — implement a unified protocol defined as a TypeScript interface in `pages-component`:

```typescript
export interface FormValueProvider {
  readonly currentValue: unknown;
  value: unknown;
  error: string | undefined;
  validate(): boolean;
}
```

- `currentValue` — getter returning the component's current value (primitive, object, or array)
- `value` — setter for initial/current value from parent
- `error` — getter/setter for the component's own validation error (displayed inline)
- `validate()` — triggers validation, sets `error`, returns `true` if valid. For composites, recursively calls `validate()` on children first.

### FormValueMixin

Shared implementation via mixin, following the platform's established pattern (`RovingTabindexMixin`, `LiveRegionMixin`, `DataSourceMixin`):

```typescript
type Constructor<T = LitElement> = new (...args: any[]) => T;

export function FormValueMixin<T extends Constructor>(Base: T) {
  abstract class FormValueHost extends Base implements FormValueProvider {
    @state() private _error: string | undefined;
    private _value: unknown;

    get currentValue(): unknown {
      return this.collectValue();
    }

    set value(v: unknown) {
      this._value = v;
      this.propagateValue(v);
    }

    get value(): unknown {
      return this._value;
    }

    get error(): string | undefined {
      return this._error;
    }

    set error(e: string | undefined) {
      this._error = e;
    }

    validate(): boolean {
      const childrenValid = this.validateChildren();
      const selfValid = this.validateSelf();
      return childrenValid && selfValid;
    }

    protected abstract collectValue(): unknown;
    protected abstract propagateValue(v: unknown): void;
    protected abstract validateSelf(): boolean;
    protected abstract validateChildren(): boolean;
  }

  return FormValueHost as unknown as Constructor<FormValueProvider & {
    collectValue(): unknown;
    propagateValue(v: unknown): void;
    validateSelf(): boolean;
    validateChildren(): boolean;
  }> & T;
}
```

- `collectValue()` — assemble the component's value from its children (object → record, array → list)
- `propagateValue(v)` — distribute an incoming value to child components
- `validateSelf()` — validate this component's own constraints (e.g., required sub-properties, minItems)
- `validateChildren()` — call `validate()` on each child, return `true` if all pass

### Package Location

`FormValueProvider` interface: `pages-component/src/model/form-value-provider.ts`
`FormValueMixin` function: `pages-component/src/model/form-value-mixin.ts`

Exported from `@casehubio/pages-component` barrel.

## Component Design

### pages-object-group

Renders a fieldset for an object sub-schema.

**Extends:** `FormValueMixin(LitElement)`

**Properties:**

```typescript
@property({ attribute: false }) schema: FieldSchema;
@property({ attribute: false }) label: string;
@property({ attribute: false }) fieldName: string;
@property({ type: Boolean }) editable = false;
@property({ type: Boolean }) required = false;
@property({ type: Boolean }) collapsible = false;
@property({ type: Boolean }) validateOnBlur = false;
@state() private _collapsed = false;
```

`fieldName` is the key under which this composite appears in its parent's `properties` map. Set by the parent (PagesSchemaForm or another composite) during child creation in the rendering loop.

**Rendering:**

```html
<fieldset class="object-group" role="group" aria-labelledby="legend-id">
  <legend id="legend-id">${this.label}</legend>
  <!-- for each property in schema.properties: -->
  <!--   leaf → create leaf input -->
  <!--   composite → create composite component (recursive) -->
</fieldset>
```

The rendering loop mirrors PagesSchemaForm's `renderContent` — iterate `schema.properties`, call `mapFieldToComponentType`, create/update children. A shared utility function extracts this loop to avoid duplication between PagesSchemaForm and pages-object-group.

**Value collection:**

```typescript
protected collectValue(): Record<string, unknown> {
  const record: Record<string, unknown> = {};
  for (const [field, child] of this._children) {
    record[field] = isFormValueProvider(child)
      ? child.currentValue
      : readFieldValue(child, this._childTypes.get(field) ?? "input");
  }
  return record;
}
```

**Validation:**

```typescript
protected validateSelf(): boolean {
  if (!this.schema.required) return true;
  const requiredSet = new Set(this.schema.required);
  const value = this.collectValue();
  let allValid = true;
  for (const field of requiredSet) {
    if (value[field] === null || value[field] === undefined || value[field] === "") {
      const child = this._children.get(field);
      if (child) setFieldError(child, this._childTypes.get(field) ?? "input", "Required");
      allValid = false;
    }
  }
  return allValid;
}

protected validateChildren(): boolean {
  let allValid = true;
  for (const [field, child] of this._children) {
    if (isFormValueProvider(child)) {
      if (!child.validate()) allValid = false;
    } else {
      const fieldSchema = this.schema.properties?.[field];
      if (fieldSchema) {
        const ct = this._childTypes.get(field) ?? "input";
        const val = readFieldValue(child, ct);
        const requiredSet = new Set(this.schema.required ?? []);
        const err = validateField(fieldSchema, val, requiredSet.has(field));
        setFieldError(child, ct, err ?? undefined);
        if (err) allValid = false;
      }
    }
  }
  return allValid;
}
```

**A11y:** `role="group"` with `aria-labelledby` pointing to the legend. When `collapsible`, the legend is a button with `aria-expanded`.

### pages-array-group

Renders a list of items with add/remove/reorder controls.

**Extends:** `FormValueMixin(LitElement)`

**Properties:**

```typescript
@property({ attribute: false }) schema: FieldSchema; // the array schema (has .items)
@property({ attribute: false }) label: string;
@property({ attribute: false }) fieldName: string;
@property({ type: Boolean }) editable = false;
@property({ type: Boolean }) required = false;
@property({ type: Boolean }) validateOnBlur = false;
@state() private _items: ArrayItem[] = [];
private _nextKey = 0;

interface ArrayItem {
  key: number;        // synthetic monotonic key (D13)
  element: HTMLElement;
  componentType: string;
}
```

**Rendering:**

```html
<div class="array-group" role="list" aria-label="${this.label}">
  <div class="array-header">
    <span class="array-label">${this.label}</span>
    <span class="array-count">${this._items.length} items</span>
  </div>
  ${repeat(this._items, item => item.key, item => html`
    <div class="array-item" role="listitem">
      ${item.element}
      <div class="array-item-controls">
        <button @click=${() => this._moveUp(item.key)} ?disabled=${isFirst}
          aria-label="Move up">↑</button>
        <button @click=${() => this._moveDown(item.key)} ?disabled=${isLast}
          aria-label="Move down">↓</button>
        <button @click=${() => this._removeItem(item.key)}
          ?disabled=${atMinItems} aria-label="Remove item">×</button>
      </div>
    </div>
  `)}
  <button class="array-add" @click=${this._addItem}
    ?disabled=${atMaxItems} aria-label="Add ${this.label}">
    + Add
  </button>
</div>
```

**Item identity (D13):** Each item receives a synthetic monotonic key (`this._nextKey++`) on creation. Keys never appear in `currentValue` output — they are purely for Lit's `repeat()` directive to enable efficient DOM reconciliation during reorder.

**Items schema handling:**

| `items` schema type | Item rendering |
|--------------------|----------------|
| Leaf (`string`, `number`, etc.) | Single leaf input per item |
| Object | `pages-object-group` per item |
| Array | `pages-array-group` per item (nested arrays) |
| `oneOf` | `pages-variant-group` per item |

**Value collection:**

```typescript
protected collectValue(): unknown[] {
  return this._items.map(item =>
    isFormValueProvider(item.element)
      ? item.element.currentValue
      : readFieldValue(item.element, item.componentType)
  );
}
```

Items are returned in visual (display) order — reordering changes the output order.

**Validation:**

```typescript
protected validateSelf(): boolean {
  const count = this._items.length;
  if (this.schema.minItems != null && count < this.schema.minItems) {
    this.error = `At least ${this.schema.minItems} items required`;
    return false;
  }
  if (this.schema.maxItems != null && count > this.schema.maxItems) {
    this.error = `At most ${this.schema.maxItems} items allowed`;
    return false;
  }
  if (this.schema.uniqueItems) {
    const values = this.collectValue();
    const serialized = values.map(v => JSON.stringify(v));
    if (new Set(serialized).size !== serialized.length) {
      this.error = "Items must be unique";
      return false;
    }
  }
  this.error = undefined;
  return true;
}
```

**Add/remove constraints:** The add button is disabled when `_items.length >= schema.maxItems`. Remove buttons are disabled when `_items.length <= schema.minItems`. When adding, a new item is created with default values from the items schema (empty string for strings, 0 for numbers, false for booleans, empty object for objects).

**A11y:** `role="list"` on the container, `role="listitem"` on each item. Reorder buttons include `aria-label` with position context. When items are added or removed, a live region announces the change ("Item added, 3 of 5" / "Item removed, 2 of 5").

### pages-variant-group

Renders a discriminated union for `oneOf` schemas.

**Extends:** `FormValueMixin(LitElement)`

**Properties:**

```typescript
@property({ attribute: false }) schema: FieldSchema; // has .oneOf
@property({ attribute: false }) label: string;
@property({ attribute: false }) fieldName: string;
@property({ type: Boolean }) editable = false;
@state() private _activeVariantIndex = 0;
@state() private _discriminatorField: string | null = null;
```

**Discriminator detection (D12):**

At `connectedCallback` or when `schema` changes, scan all `oneOf` sub-schemas for a shared property with a `const` value:

```typescript
private detectDiscriminator(): void {
  const variants = this.schema.oneOf;
  if (!variants || variants.length === 0) return;

  // Collect candidate properties from ALL variants, not just the first
  const candidateProps = new Set<string>();
  for (const variant of variants) {
    for (const prop of Object.keys(variant.properties ?? {})) {
      candidateProps.add(prop);
    }
  }

  for (const prop of candidateProps) {
    const allHaveConst = variants.every(v =>
      v.properties?.[prop]?.const !== undefined
    );
    if (allHaveConst) {
      this._discriminatorField = prop;
      return;
    }
  }
  console.error(
    "pages-variant-group: oneOf schema has no discriminator property (no shared property with const values across all variants). " +
    "Use x-renderer for undiscriminated oneOf."
  );
}
```

**Undiscriminated non-object unions** (e.g., `[{type: "string"}, {type: "number"}]`) are not supported — use `x-renderer` to provide a custom component for these cases.

**Rendering:**

```html
<fieldset class="variant-group" role="group" aria-labelledby="variant-legend">
  <legend id="variant-legend">${this.label}</legend>
  <pages-select
    .label=${this._discriminatorField}
    .options=${variantOptions}
    .value=${activeDiscriminatorValue}
    @pages-field-change=${this._onVariantSwitch}
  ></pages-select>
  <!-- Render active variant's fields (excluding discriminator) -->
  ${this._renderActiveVariant()}
</fieldset>
```

The dropdown options are derived from each variant's discriminator `const` value and `title` (if present).

**Value collection:**

```typescript
protected collectValue(): Record<string, unknown> {
  const activeVariant = this.schema.oneOf![this._activeVariantIndex];
  const record: Record<string, unknown> = {};
  if (this._discriminatorField) {
    record[this._discriminatorField] = activeVariant.properties?.[this._discriminatorField]?.const;
  }
  for (const [field, child] of this._activeChildren) {
    if (field === this._discriminatorField) continue;
    record[field] = isFormValueProvider(child)
      ? child.currentValue
      : readFieldValue(child, this._activeChildTypes.get(field) ?? "input");
  }
  return record;
}
```

**Variant switching (D12):** When the user selects a different variant, all entered data is cleared. No preservation or merging across variants. The new variant's fields render with default/empty values.

**Validation:** Validates only the active variant's sub-schema. Inactive variants are not validated.

**A11y:** `role="group"` with `aria-labelledby`. When variant switches, a live region announces "Switched to {variant name}".

## Data Model

### Structured Records (D2)

Forms always produce structured records matching the JSON Schema shape:

```typescript
// Schema
{
  properties: {
    name: { type: "string" },
    address: {
      type: "object",
      properties: {
        street: { type: "string" },
        city: { type: "string" }
      }
    },
    tags: { type: "array", items: { type: "string" } }
  }
}

// currentValue output
{
  name: "Jane",
  address: { street: "123 Main", city: "NYC" },
  tags: ["developer", "admin"]
}
```

### Pipeline Adapter Layer (D5)

Forms work with structured records. The adapter layer translates between structured records and `TypedDataSet`. Two strategies are in scope for #222:

| Strategy | Detection | Read | Write |
|----------|-----------|------|-------|
| **Standalone** | No dataset (create mode, `forceCreate`) | Empty/default values from schema | Emit structured record via `pages-record-create` |
| **Flat projection** | Dataset has individual columns matching top-level property names | Read leaf values from columns directly; parse JSON columns for nested sub-trees | Leaf changes emit per-column `pages-field-change`; nested changes serialize to JSON for their parent column |

**Future work (out of scope for #222):** A JSON column strategy (entire form value stored as JSON in a single TEXT column) may be added later if use cases emerge. This would require strategy detection heuristics and JSON serialization/deserialization — deferred to avoid speculative complexity.

The adapter is a utility function in PagesSchemaForm, called at the start of `renderContent` to extract a structured record from the dataset:

```typescript
function extractRecord(
  schema: FieldSchema,
  dataset: TypedDataSet,
): Record<string, unknown> {
  const record: Record<string, unknown> = {};
  const schemaProps = schema.properties ?? {};

  for (const field of Object.keys(schemaProps)) {
    const fieldSchema = schemaProps[field];
    const col = dataset.columns.find(c => c.id === field);
    if (!col) continue;

    const row = dataset.rows[0];
    if (!row) continue;

    try {
      const cell = row.cell(col.id);
      if (cell.type === "NULL") continue;

      const componentType = mapFieldToComponentType(fieldSchema!);
      if (componentType === "object-group" || componentType === "array-group" || componentType === "variant-group") {
        // Nested types stored as JSON in a text column
        const raw = String(cell.value);
        try { record[field] = JSON.parse(raw); } catch { record[field] = null; }
      } else {
        record[field] = cell.value;
      }
    } catch { /* column not found */ }
  }
  return record;
}
```

## Event Propagation (D14)

Composites use re-emission: they intercept `pages-field-change` events from their children, stop propagation, and re-emit a new event with the composite's own field name and full value.

```typescript
// In each composite component (object-group, array-group, variant-group):
connectedCallback() {
  super.connectedCallback();
  // Listen on renderRoot (shadow root), NOT on `this` — dispatching a new event
  // on `this` while `this` has a same-event listener creates an infinite loop
  // at the AT_TARGET phase.
  this.renderRoot.addEventListener("pages-field-change", this._onChildFieldChange);
}

disconnectedCallback() {
  super.disconnectedCallback();
  this.renderRoot.removeEventListener("pages-field-change", this._onChildFieldChange);
}

private _onChildFieldChange = (e: Event): void => {
  const detail = (e as CustomEvent).detail;
  if (!detail.committed) return; // only re-emit committed changes

  e.stopPropagation();

  // Blur-time validation: validate the changed child before re-emitting
  if (this.validateOnBlur) {
    const childField = detail.field;
    const child = this._children.get(childField);
    if (child) {
      if (isFormValueProvider(child)) {
        child.validate();
      } else {
        const fieldSchema = this.schema.properties?.[childField];
        if (fieldSchema) {
          const requiredSet = new Set(this.schema.required ?? []);
          const error = validateField(fieldSchema, detail.value, requiredSet.has(childField));
          setFieldError(child, this._childTypes.get(childField) ?? "input", error ?? undefined);
        }
      }
    }
  }

  this.dispatchEvent(new CustomEvent("pages-field-change", {
    bubbles: true, composed: true,
    detail: {
      field: this.fieldName,
      value: this.currentValue,
      committed: true,
    },
  }));
};
```

**Only committed events are re-emitted.** Uncommitted (keystroke-level) events are not re-emitted — this prevents full-object serialization on every keystroke. The existing auto-save pipeline already batches committed changes via `pages-form-commit` in the activation handler.

**Consistency with D6:** formScope and the auto-save pipeline see the composite as a single opaque field changing its value. They never see internal child events.

## formScope Integration (D6)

**Prerequisite:** This section assumes #337 (formScope composable layout) is complete — specifically that the activation handler listens for `pages-field-register` events, instantiates `FormScopeState` via `FormScopeRegistry`, and uses `FormScopeState.validateAll()` for validation dispatch. The current codebase uses ad-hoc property storage (`__formScopeSchema`) and a local `validateFormField()` copy — #337 replaces these with the proper `FormScopeState` machinery. This spec (#222) adds FormValueProvider awareness on top of the completed #337 implementation.

formScope treats composite components as single opaque fields:

1. **Registration:** Composites register with formScope via `pages-field-register` like any other field. The `componentType` is `"object-group"`, `"array-group"`, or `"variant-group"`.

2. **Value collection:** `FormScopeState.collectValues()` calls `readFieldValue()` on each registered element. The existing duck-typing in `readFieldValue` (`"currentValue" in element ? element.currentValue : element.value`) already handles FormValueProvider-conformant elements. No modification to `readFieldValue` is needed — the duck-typing path is sufficient.

3. **Validation dispatch:** `FormScopeState.validateAll()` gains FormValueProvider awareness:

```typescript
validateAll(): Record<string, string> {
  this.pruneDisconnected();
  const requiredSet = new Set(this.schema?.required ?? []);
  const errors: Record<string, string> = {};

  for (const [field, entry] of this.fields) {
    if (isFormValueProvider(entry.element)) {
      // Composite: delegate validation to the component
      if (!entry.element.validate()) {
        errors[field] = entry.element.error ?? "Validation failed";
      }
    } else {
      // Leaf: existing validateField path
      if (!this.schema?.properties) continue;
      const fieldSchema = this.schema.properties[field];
      if (!fieldSchema) continue;
      const value = readFieldValue(entry.element, entry.componentType);
      const error = validateField(fieldSchema, value, requiredSet.has(field));
      if (error) {
        errors[field] = error;
        setFieldError(entry.element, entry.componentType, error);
      } else {
        setFieldError(entry.element, entry.componentType, undefined);
      }
    }
  }
  return errors;
}
```

**Type guard:**

```typescript
export function isFormValueProvider(el: unknown): el is FormValueProvider {
  return el != null
    && typeof el === "object"
    && "currentValue" in el
    && "validate" in el
    && typeof (el as any).validate === "function";
}
```

Defined in `pages-component/src/model/form-value-provider.ts` alongside the interface.

## Schema Pre-Processing (D8)

### $ref Resolution

**FieldSchema additions:** `FieldSchema` gains `$defs` and `definitions` fields to support `$ref` resolution with type safety:

```typescript
readonly $ref?: string;
readonly $defs?: Readonly<Record<string, FieldSchema>>;
readonly definitions?: Readonly<Record<string, FieldSchema>>;
```

These are added to `pages-component/src/model/form-input-types.ts`. The FieldSchema consolidation spec (#392) should also include them.

A pure function that resolves local `#/$defs/...` references before rendering:

```typescript
export function resolveSchemaRefs(schema: FieldSchema): FieldSchema {
  const defs = schema.$defs ?? schema.definitions ?? {};
  return resolveNode(schema, defs, new Set());
}

function resolveNode(
  node: FieldSchema,
  defs: Record<string, FieldSchema>,
  visiting: Set<string>,
): FieldSchema {
  const ref = node.$ref;
  if (ref) {
    const defName = ref.replace(/^#\/\$defs\/|^#\/definitions\//, "");
    if (visiting.has(defName)) return {}; // cycle detected — terminal empty schema
    visiting.add(defName);
    const resolved = defs[defName];
    if (!resolved) return node; // unresolvable ref, pass through
    const result = resolveNode(resolved, defs, visiting);
    visiting.delete(defName);
    return result;
  }

  // Recursively resolve sub-schemas
  const resolved: any = { ...node };
  if (node.properties) {
    resolved.properties = {};
    for (const [key, prop] of Object.entries(node.properties)) {
      resolved.properties[key] = resolveNode(prop, defs, visiting);
    }
  }
  if (node.items) {
    resolved.items = resolveNode(node.items, defs, visiting);
  }
  if (node.oneOf) {
    resolved.oneOf = node.oneOf.map(v => resolveNode(v, defs, visiting));
  }
  return resolved;
}
```

**Scope:** Only local references (`#/$defs/...`, `#/definitions/...`). External references (URLs, file paths) are not supported — schemas using external refs must be pre-resolved before passing to PagesSchemaForm.

**Draft-07 `$ref` semantics:** When a node has `$ref` alongside sibling properties (e.g., `{ "$ref": "#/$defs/address", "title": "Home Address" }`), the resolver returns only the resolved target and discards siblings. This follows JSON Schema draft-07 where siblings alongside `$ref` are ignored. The spec header declares `draft-07` as the target draft.

**Cycle handling:** The resolver tracks the set of `$ref` paths in the current resolution stack. When a cycle is detected, it inserts a terminal empty schema at the cycle point. This avoids the exponential expansion risk of depth-limited unrolling.

**Location:** `pages-component/src/model/schema-ref-resolver.ts`

### PagesSchemaForm Conformance

PagesSchemaForm gains FormValueProvider conformance and LiveRegionMixin consolidation as part of this spec:

1. **FormValueProvider:** PagesSchemaForm already has `currentValue` (getter returning a record of all field values). This spec adds a `validate(): boolean` method (extracted from `submit()` — validate-only, no event dispatch). With both `currentValue` and `validate()`, `isFormValueProvider()` returns `true` for PagesSchemaForm, making it usable as a composite child in formScope and unifying value collection.

2. **LiveRegionMixin:** PagesSchemaForm currently has an inline `announce()` method (lines 68-83) that is functionally identical to `LiveRegionMixin` from `pages-primitives`. This spec refactors PagesSchemaForm to use `LiveRegionMixin` instead of the inline implementation. Since PagesElement is abstract and cannot be directly passed to the mixin, a standalone `createLiveRegionHelper()` utility is extracted from `LiveRegionMixin` into `pages-primitives/src/a11y/live-region.ts` — both `LiveRegionMixin` and PagesSchemaForm use it internally. This consolidates the implementation without requiring mixin composition through abstract base classes.

### mapFieldToComponentType Extension

In `pages-viz/src/form-inputs/schema-types.ts`, the existing `mapFieldToComponentType` is extended with the priority order specified in the Architecture section. The `deriveSchemaFromDataSet` function is unchanged — it produces flat schemas from TypedDataSet columns. Nested schemas come from explicit `schema` props, not auto-derivation.

## Custom Renderers (D9)

Any schema node can specify `x-renderer`:

```yaml
schema:
  properties:
    color:
      type: string
      x-renderer: my-color-picker
    location:
      type: object
      x-renderer: my-map-widget
```

**Runtime conformance check:** When a custom renderer element is created, verify it implements FormValueProvider before using it:

```typescript
if (fieldSchema["x-renderer"]) {
  const tagName = String(fieldSchema["x-renderer"]);
  const el = document.createElement(tagName);
  if (!isFormValueProvider(el)) {
    console.warn(
      `pages-schema-form: custom renderer <${tagName}> does not implement FormValueProvider ` +
      `(missing currentValue getter or validate() method). Values may be undefined and validation skipped.`
    );
  }
  return tagName;
}
```

The form still renders the element — the warning prevents silent failures where broken custom renderers produce `undefined` values.

## Validation (D7)

### Recursive Model

Each component validates its own sub-schema via `validate(): boolean`. The existing `validateField()` utility stays leaf-only — it is used internally by leaf components, not extended for composite types.

### `validateField` Enhancement

The existing `validateField()` in `pages-component/src/model/field-validation.ts` checks `minimum` and `maximum` but not `exclusiveMinimum` and `exclusiveMaximum`, even though `FieldSchema` declares both. This spec adds the missing checks:

```typescript
if (typeof value === "number") {
  if (schema.minimum != null && value < schema.minimum) return `Must be at least ${schema.minimum}`;
  if (schema.maximum != null && value > schema.maximum) return `Must be at most ${schema.maximum}`;
  if (schema.exclusiveMinimum != null && value <= schema.exclusiveMinimum) return `Must be greater than ${schema.exclusiveMinimum}`;
  if (schema.exclusiveMaximum != null && value >= schema.exclusiveMaximum) return `Must be less than ${schema.exclusiveMaximum}`;
}
```

### Submit-Time Flow

1. PagesSchemaForm's `submit()` iterates all top-level children
2. For each child: if FormValueProvider → call `validate()`; else → call `validateField()` (existing path)
3. Each composite's `validate()` recursively validates children, then validates self
4. Errors are set on each component's `error` property — displayed inline at the deepest relevant level
5. If any top-level field returns `false`, submit is blocked; a live region announces the error count

### Blur-Time Flow (validateOnBlur)

1. Composite intercepts committed `pages-field-change` from its children
2. Composite validates the changed child's value against its sub-schema
3. Error is set on the child element
4. Composite re-emits the event (D14) — formScope/auto-save receive the committed change

## YAML DSL

### Nested Object

```yaml
- schema-form:
    schema:
      properties:
        name: { type: string, minLength: 1 }
        address:
          type: object
          properties:
            street: { type: string }
            city: { type: string }
            zip: { type: string, pattern: "^\\d{5}$" }
          required: [street, city]
      required: [name]
    lookup:
      uuid: contacts
```

### Array of Primitives

```yaml
- schema-form:
    schema:
      properties:
        title: { type: string }
        tags:
          type: array
          items: { type: string }
          minItems: 1
          maxItems: 10
      required: [title]
```

### Array of Objects

```yaml
- schema-form:
    schema:
      properties:
        name: { type: string }
        addresses:
          type: array
          items:
            type: object
            properties:
              street: { type: string }
              city: { type: string }
              primary: { type: boolean }
            required: [street, city]
          minItems: 1
```

### oneOf Discriminated Union

```yaml
- schema-form:
    schema:
      properties:
        name: { type: string }
        contact:
          oneOf:
            - properties:
                method: { const: email }
                address: { type: string, format: email }
              required: [method, address]
            - properties:
                method: { const: phone }
                number: { type: string, pattern: "^\\+?[\\d\\s-]+$" }
              required: [method, number]
```

### $ref with Definitions

```yaml
- schema-form:
    schema:
      $defs:
        address:
          type: object
          properties:
            street: { type: string }
            city: { type: string }
          required: [street, city]
      properties:
        home: { $ref: "#/$defs/address" }
        work: { $ref: "#/$defs/address" }
```

### formScope with Nested Fields

```yaml
- form-scope:
    schema:
      properties:
        personal:
          type: object
          properties:
            name: { type: string }
            email: { type: string }
        address:
          type: object
          properties:
            street: { type: string }
            city: { type: string }
    validateOnBlur: true
    components:
      - columns:
          widths: [6, 6]
          components:
            - schema-form:
                fieldsOnly: true
                fields: [personal]
            - schema-form:
                fieldsOnly: true
                fields: [address]
      - submit-button:
          label: Save
```

## Package Changes

### New Files

| Package | File | Contents |
|---------|------|----------|
| `pages-component` | `src/model/form-value-provider.ts` | `FormValueProvider` interface, `isFormValueProvider()` guard |
| `pages-component` | `src/model/form-value-mixin.ts` | `FormValueMixin` function |
| `pages-component` | `src/model/schema-ref-resolver.ts` | `resolveSchemaRefs()` pure function |
| `pages-viz` | `src/form-inputs/PagesObjectGroup.ts` | `pages-object-group` component |
| `pages-viz` | `src/form-inputs/PagesArrayGroup.ts` | `pages-array-group` component |
| `pages-viz` | `src/form-inputs/PagesVariantGroup.ts` | `pages-variant-group` component |
| `pages-viz` | `src/form-inputs/form-field-renderer.ts` | Shared rendering loop (extracted from PagesSchemaForm) |
| `pages-viz` | `src/form-inputs/object-group.test.ts` | Unit tests |
| `pages-viz` | `src/form-inputs/array-group.test.ts` | Unit tests |
| `pages-viz` | `src/form-inputs/variant-group.test.ts` | Unit tests |
| `pages-viz` | `src/form-inputs/schema-ref-resolver.test.ts` | $ref resolution tests |
| `pages-runtime` | `src/nested-form-integration.test.ts` | Integration tests |

### Modified Files

| Package | File | Change |
|---------|------|--------|
| `pages-component` | `src/model/index.ts` | Export `FormValueProvider`, `isFormValueProvider`, `FormValueMixin`, `resolveSchemaRefs` |
| `pages-component` | `src/model/form-input-types.ts` | Add `$ref`, `$defs`, and `definitions` fields to `FieldSchema` |
| `pages-component` | `src/model/field-validation.ts` | Add `exclusiveMinimum`/`exclusiveMaximum` checks to `validateField()` |
| `pages-viz` | `src/form-inputs/schema-types.ts` | Extend `mapFieldToComponentType` with composite types and type-array normalization |
| `pages-viz` | `src/form-inputs/PagesSchemaForm.ts` | Call `resolveSchemaRefs` on schema; add `validate()` method (FormValueProvider conformance); replace inline `announce()` with `createLiveRegionHelper()`; use shared rendering loop; extract record from dataset via adapter; pass sub-values to composite children |
| `pages-viz` | `src/form-inputs/index.ts` | Export new components |
| `pages-primitives` | `src/a11y/live-region.ts` | Extract `createLiveRegionHelper()` utility; refactor `LiveRegionMixin` to use it |
| `pages-runtime` | `src/form-scope.ts` | `FormScopeState.validateAll()` — detect FormValueProvider, call `validate()` for composites |
| `pages-runtime` | `src/activation.ts` | Replace local `validateFormField()` with import of `validateField` from `@casehubio/pages-component` |

**Not modified (intentional):**
- `pages-component/src/model/field-access.ts` — `readFieldValue`'s existing duck-typing (`"currentValue" in element ? element.currentValue : element.value`) already handles FormValueProvider-conformant elements. No modification needed.
- `pages-runtime/src/activation.ts` `DATA_COMPONENT_TYPES` — composites are NOT registered here. They are internal rendering primitives created programmatically by PagesSchemaForm and by each other during recursive traversal, not standalone DSL/YAML component types. They don't go through the activation callback and would fail if cast to `PagesElement<VizComponentProps>`.
- `pages-ui/src/dsl/builders.ts` — no builder functions for composites. These components are never authored in DSL or YAML.
- `pages-ui/src/component-desugar.ts` — no desugar handlers for composites.

### Dependencies

- `pages-viz` depends on `pages-component` (existing) — gains `FormValueMixin`, `resolveSchemaRefs`
- `pages-viz` depends on `pages-primitives` (existing) — `LiveRegionMixin` for array group announcements
- No new package dependencies

## Testing

### Unit Tests (pages-viz)

**pages-object-group:**
- Renders fieldset with legend from label
- Creates leaf inputs for flat properties
- Recursively renders nested object-groups
- `currentValue` returns nested record
- `propagateValue` distributes to children
- `validate()` checks required sub-properties
- `validate()` recursively validates children
- Collapsible mode toggles visibility
- `pages-field-change` re-emitted with composite field name and full value
- Uncommitted events are not re-emitted

**pages-array-group:**
- Renders items from value array
- Add button creates new item with defaults
- Remove button deletes item
- Reorder buttons swap items
- `currentValue` returns items in display order
- Synthetic keys enable efficient DOM reconciliation (no DOM destruction on reorder)
- `validate()` checks minItems/maxItems/uniqueItems
- `validate()` recursively validates each item
- Add disabled at maxItems, remove disabled at minItems
- Live region announces add/remove
- Array of primitives (string items)
- Array of objects (object-group items)

**pages-variant-group:**
- Auto-detects discriminator from const values
- Renders dropdown for variant selection
- Renders active variant's fields
- `currentValue` includes discriminator value
- Variant switch clears all data
- `validate()` validates only active variant
- Logs error for undiscriminated oneOf
- Live region announces variant switch

**schema-ref-resolver:**
- Resolves local `#/$defs/` references
- Resolves nested references
- Detects and terminates circular references
- Passes through unresolvable refs
- Handles `$defs` and `definitions` keys

**mapFieldToComponentType (extended):**
- `x-renderer` takes highest priority
- `oneOf` maps to `variant-group`
- `type: "array"` maps to `array-group`
- `type: "object"` maps to `object-group`
- Existing leaf mappings unchanged

### Integration Tests (pages-runtime)

- YAML with nested object schema → object-group renders, values collected correctly
- YAML with array schema → array-group renders, add/remove works
- YAML with oneOf schema → variant-group renders, switching works
- Nested form in formScope → composite registers as opaque field, formScope collects nested value
- formScope `validateAll()` calls composite `validate()` → errors surface at nested level
- Pipeline adapter: flat projection reads leaf columns, parses JSON for nested sub-trees
- Standalone (no pipeline): create mode with nested schema, submit emits structured record
- `$ref` schema with definitions → resolved before rendering, form works
- `x-renderer` with conformant element → renders and collects values
- `x-renderer` with non-conformant element → console.warn logged, element still renders
- Nested event propagation: child committed change → composite re-emits → formScope/auto-save receives

### Example Validation

- Schema Form example: add new tabs demonstrating nested objects, arrays, and oneOf
- Contact Manager example: update with address sub-form (nested object)

## References

- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — current flat rendering implementation
- `packages/pages-viz/src/form-inputs/schema-types.ts` — `mapFieldToComponentType`, `deriveSchemaFromDataSet`, `validateField` re-export
- `packages/pages-component/src/model/form-input-types.ts:55-81` — `FieldSchema` (already supports `properties`, `items`, `oneOf`)
- `packages/pages-component/src/model/form-input-types.ts:83-93` — `SchemaFormProps`
- `packages/pages-component/src/model/field-access.ts` — `STANDALONE_TYPES`, `readFieldValue`, `setFieldError`
- `packages/pages-component/src/model/field-validation.ts` — `validateField` (leaf-only)
- `packages/pages-runtime/src/form-scope.ts` — `FormScopeState`, `FormScopeRegistry`
- `packages/pages-runtime/src/activation.ts:406-430` — formScope activation handler
- `packages/pages-primitives/src/a11y/live-region.ts` — `LiveRegionMixin` pattern
- `packages/pages-component/src/controller/data-source-mixin.ts` — `DataSourceMixin` mixin pattern
- `docs/protocols/casehub/web-component-strategy.md` — Lit conventions, mixin convention, base class hierarchy
- `docs/protocols/casehub/dataset-contract.md` — DatasetContract, shape convention
- `docs/specs/2026-07-21-schema-form-runtime-integration-design.md` — predecessor spec (#159)
- `docs/specs/issue-392-consolidate-fieldschema/2026-08-31-consolidate-fieldschema-design.md` — FieldSchema consolidation
- `docs/specs/issue-334-schema-form-dsl/2026-08-26-form-scope-composable-layout-design.md` — formScope composable layout
- casehubio/casehub-pages#222 — issue
- casehubio/casehub-pages#159 — predecessor (flat schema-form integration)
- casehubio/casehub-pages#392 — FieldSchema consolidation
- casehubio/casehub-pages#334 — schemaForm DSL builder
- casehubio/casehub-pages#337 — formScope composable layout
