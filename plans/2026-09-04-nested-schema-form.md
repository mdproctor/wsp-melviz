# Nested Schema Form Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #222 — feat: pages-schema-form nested object and array schema support
**Issue group:** #222

**Goal:** Add recursive nested schema support to PagesSchemaForm — objects, arrays, oneOf, $ref, custom renderers, reordering — producing structured JSON records from hierarchical JSON Schemas.

**Architecture:** Every JSON Schema node maps to exactly one component type. Three new Lit components (`pages-object-group`, `pages-array-group`, `pages-variant-group`) extend `FormValueMixin(LitElement)` to handle composite schema nodes. PagesSchemaForm walks the schema tree recursively, creating leaf inputs for primitives and composite components for structural types. All form components implement the `FormValueProvider` protocol for uniform value collection, validation, and error handling.

**Tech Stack:** TypeScript, Lit 3, vitest, `@casehubio/pages-component`, `@casehubio/pages-viz`, `@casehubio/pages-primitives`

## Global Constraints

- JSON Schema draft-07 semantics — siblings alongside `$ref` are ignored
- Only local `$ref` resolution (`#/$defs/`, `#/definitions/`) — no external URLs
- Composites are NOT registered in `DATA_COMPONENT_TYPES` or DSL builders — they are internal rendering primitives
- All new components use `pages-` prefix (`pages-object-group`, `pages-array-group`, `pages-variant-group`)
- Tests use vitest with the existing `makeDataSet` helper pattern from `schema-form.test.ts`
- `readFieldValue` is NOT modified — existing duck-typing already handles FormValueProvider
- Run `yarn typecheck` after each task to verify no type errors

---

## Batch 1: Foundation

After this batch: FormValueProvider protocol, FormValueMixin, schema utilities (resolveSchemaRefs, extended mapFieldToComponentType) are all available. No visible UI changes yet.

### Task 1: FormValueProvider interface, isFormValueProvider guard, FieldSchema additions

**Files:**
- Create: `packages/pages-component/src/model/form-value-provider.ts`
- Modify: `packages/pages-component/src/model/form-input-types.ts:55-81`
- Modify: `packages/pages-component/src/model/index.ts`
- Test: `packages/pages-component/src/model/form-value-provider.test.ts`

**Interfaces:**
- Produces: `FormValueProvider` interface (`currentValue: unknown`, `value: unknown`, `error: string | undefined`, `validate(): boolean`), `isFormValueProvider(el: unknown): el is FormValueProvider`

- [ ] **Step 1: Write tests for isFormValueProvider type guard**

```typescript
// packages/pages-component/src/model/form-value-provider.test.ts
import { describe, it, expect } from "vitest";
import { isFormValueProvider } from "./form-value-provider.js";

describe("isFormValueProvider", () => {
  it("returns true for object with currentValue and validate()", () => {
    const obj = {
      currentValue: "hello",
      value: "hello",
      error: undefined,
      validate: () => true,
    };
    expect(isFormValueProvider(obj)).toBe(true);
  });

  it("returns false for null", () => {
    expect(isFormValueProvider(null)).toBe(false);
  });

  it("returns false for plain HTMLElement", () => {
    const el = document.createElement("div");
    expect(isFormValueProvider(el)).toBe(false);
  });

  it("returns false for object with currentValue but no validate", () => {
    const obj = { currentValue: "hello", value: "hello", error: undefined };
    expect(isFormValueProvider(obj)).toBe(false);
  });

  it("returns false for object where validate is not a function", () => {
    const obj = { currentValue: "hello", validate: "not-a-function" };
    expect(isFormValueProvider(obj)).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-component/src/model/form-value-provider.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Create FormValueProvider interface and isFormValueProvider guard**

```typescript
// packages/pages-component/src/model/form-value-provider.ts
export interface FormValueProvider {
  readonly currentValue: unknown;
  value: unknown;
  error: string | undefined;
  validate(): boolean;
}

export function isFormValueProvider(el: unknown): el is FormValueProvider {
  return el != null
    && typeof el === "object"
    && "currentValue" in el
    && "validate" in el
    && typeof (el as any).validate === "function";
}
```

- [ ] **Step 4: Add FieldSchema additions for $ref support**

In `packages/pages-component/src/model/form-input-types.ts`, add three fields to the `FieldSchema` interface after the `oneOf` field (line 79):

```typescript
  readonly $ref?: string;
  readonly $defs?: Readonly<Record<string, FieldSchema>>;
  readonly definitions?: Readonly<Record<string, FieldSchema>>;
```

- [ ] **Step 5: Export from barrel**

In `packages/pages-component/src/model/index.ts`, add after the `isFixedOptions` export:

```typescript
// Form value protocol
export type { FormValueProvider } from "./form-value-provider.js";
export { isFormValueProvider } from "./form-value-provider.js";
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-component/src/model/form-value-provider.test.ts`
Expected: PASS — all 5 tests green

Run: `yarn typecheck`
Expected: PASS — no type errors

- [ ] **Step 7: Commit**

```bash
git add packages/pages-component/src/model/form-value-provider.ts packages/pages-component/src/model/form-value-provider.test.ts packages/pages-component/src/model/form-input-types.ts packages/pages-component/src/model/index.ts
git commit -m "feat(#222): add FormValueProvider interface, isFormValueProvider guard, FieldSchema $ref fields"
```

---

### Task 2: FormValueMixin

**Files:**
- Create: `packages/pages-component/src/model/form-value-mixin.ts`
- Modify: `packages/pages-component/src/model/index.ts`
- Test: `packages/pages-component/src/model/form-value-mixin.test.ts`

**Interfaces:**
- Consumes: `FormValueProvider` from Task 1
- Produces: `FormValueMixin<T extends Constructor>(Base: T)` — mixin providing shared FormValueProvider implementation with abstract methods `collectValue()`, `propagateValue(v)`, `validateSelf()`, `validateChildren()`

- [ ] **Step 1: Write tests for FormValueMixin**

```typescript
// packages/pages-component/src/model/form-value-mixin.test.ts
import { describe, it, expect } from "vitest";
import { LitElement } from "lit";
import { FormValueMixin } from "./form-value-mixin.js";
import { isFormValueProvider } from "./form-value-provider.js";

class TestComposite extends FormValueMixin(LitElement) {
  private _collected: unknown = "test-value";
  private _propagated: unknown = undefined;
  private _selfValid = true;
  private _childrenValid = true;

  setCollected(v: unknown) { this._collected = v; }
  setSelfValid(v: boolean) { this._selfValid = v; }
  setChildrenValid(v: boolean) { this._childrenValid = v; }
  getPropagated(): unknown { return this._propagated; }

  protected collectValue(): unknown { return this._collected; }
  protected propagateValue(v: unknown): void { this._propagated = v; }
  protected validateSelf(): boolean {
    if (!this._selfValid) { this.error = "Self invalid"; }
    return this._selfValid;
  }
  protected validateChildren(): boolean { return this._childrenValid; }
}
customElements.define("test-composite", TestComposite);

describe("FormValueMixin", () => {
  it("satisfies isFormValueProvider", () => {
    const el = new TestComposite();
    expect(isFormValueProvider(el)).toBe(true);
  });

  it("currentValue delegates to collectValue", () => {
    const el = new TestComposite();
    el.setCollected({ name: "Jane" });
    expect(el.currentValue).toEqual({ name: "Jane" });
  });

  it("setting value delegates to propagateValue", () => {
    const el = new TestComposite();
    el.value = { name: "Jane" };
    expect(el.getPropagated()).toEqual({ name: "Jane" });
    expect(el.value).toEqual({ name: "Jane" });
  });

  it("validate returns true when both self and children valid", () => {
    const el = new TestComposite();
    expect(el.validate()).toBe(true);
    expect(el.error).toBeUndefined();
  });

  it("validate returns false when self invalid", () => {
    const el = new TestComposite();
    el.setSelfValid(false);
    expect(el.validate()).toBe(false);
    expect(el.error).toBe("Self invalid");
  });

  it("validate returns false when children invalid", () => {
    const el = new TestComposite();
    el.setChildrenValid(false);
    expect(el.validate()).toBe(false);
  });

  it("error getter/setter works", () => {
    const el = new TestComposite();
    expect(el.error).toBeUndefined();
    el.error = "Something wrong";
    expect(el.error).toBe("Something wrong");
    el.error = undefined;
    expect(el.error).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-component/src/model/form-value-mixin.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement FormValueMixin**

```typescript
// packages/pages-component/src/model/form-value-mixin.ts
import { state } from "lit/decorators.js";
import type { LitElement } from "lit";
import type { FormValueProvider } from "./form-value-provider.js";

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

- [ ] **Step 4: Export from barrel**

In `packages/pages-component/src/model/index.ts`, add after the FormValueProvider exports:

```typescript
export { FormValueMixin } from "./form-value-mixin.js";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-component/src/model/form-value-mixin.test.ts`
Expected: PASS — all 7 tests green

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/src/model/form-value-mixin.ts packages/pages-component/src/model/form-value-mixin.test.ts packages/pages-component/src/model/index.ts
git commit -m "feat(#222): add FormValueMixin — shared FormValueProvider implementation via mixin"
```

---

### Task 3: resolveSchemaRefs utility + extended mapFieldToComponentType + validateField enhancement

**Files:**
- Create: `packages/pages-component/src/model/schema-ref-resolver.ts`
- Create: `packages/pages-component/src/model/schema-ref-resolver.test.ts`
- Modify: `packages/pages-component/src/model/index.ts`
- Modify: `packages/pages-component/src/model/field-validation.ts:24-31`
- Modify: `packages/pages-viz/src/form-inputs/schema-types.ts:38-51`
- Test: `packages/pages-viz/src/form-inputs/schema-types.test.ts`

**Interfaces:**
- Consumes: `FieldSchema` from `pages-component`
- Produces: `resolveSchemaRefs(schema: FieldSchema): FieldSchema`, extended `mapFieldToComponentType` returning `"object-group"` | `"array-group"` | `"variant-group"` for composite types

- [ ] **Step 1: Write tests for resolveSchemaRefs**

```typescript
// packages/pages-component/src/model/schema-ref-resolver.test.ts
import { describe, it, expect } from "vitest";
import { resolveSchemaRefs } from "./schema-ref-resolver.js";
import type { FieldSchema } from "./form-input-types.js";

describe("resolveSchemaRefs", () => {
  it("resolves local $defs reference", () => {
    const schema: FieldSchema = {
      $defs: {
        address: {
          type: "object",
          properties: { street: { type: "string" }, city: { type: "string" } },
        },
      },
      properties: {
        home: { $ref: "#/$defs/address" },
      },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved.properties!.home.type).toBe("object");
    expect(resolved.properties!.home.properties!.street.type).toBe("string");
  });

  it("resolves nested references", () => {
    const schema: FieldSchema = {
      $defs: {
        name: { type: "string" },
        person: { type: "object", properties: { name: { $ref: "#/$defs/name" } } },
      },
      properties: {
        owner: { $ref: "#/$defs/person" },
      },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved.properties!.owner.properties!.name.type).toBe("string");
  });

  it("handles circular references with terminal empty schema", () => {
    const schema: FieldSchema = {
      $defs: {
        node: {
          type: "object",
          properties: {
            value: { type: "string" },
            child: { $ref: "#/$defs/node" },
          },
        },
      },
      properties: {
        root: { $ref: "#/$defs/node" },
      },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved.properties!.root.type).toBe("object");
    expect(resolved.properties!.root.properties!.value.type).toBe("string");
    // Circular ref terminates as empty schema
    expect(resolved.properties!.root.properties!.child).toEqual({});
  });

  it("passes through unresolvable refs", () => {
    const schema: FieldSchema = {
      properties: {
        thing: { $ref: "#/$defs/nonexistent" },
      },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved.properties!.thing.$ref).toBe("#/$defs/nonexistent");
  });

  it("handles definitions key (legacy)", () => {
    const schema: FieldSchema = {
      definitions: {
        name: { type: "string" },
      },
      properties: {
        label: { $ref: "#/definitions/name" },
      },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved.properties!.label.type).toBe("string");
  });

  it("resolves refs inside items", () => {
    const schema: FieldSchema = {
      $defs: { tag: { type: "string", minLength: 1 } },
      properties: {
        tags: { type: "array", items: { $ref: "#/$defs/tag" } },
      },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved.properties!.tags.items!.type).toBe("string");
    expect(resolved.properties!.tags.items!.minLength).toBe(1);
  });

  it("resolves refs inside oneOf", () => {
    const schema: FieldSchema = {
      $defs: { emailVariant: { properties: { method: { const: "email" } } } },
      properties: {
        contact: { oneOf: [{ $ref: "#/$defs/emailVariant" }] },
      },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved.properties!.contact.oneOf![0].properties!.method.const).toBe("email");
  });

  it("returns schema unchanged when no refs present", () => {
    const schema: FieldSchema = {
      properties: { name: { type: "string" } },
    };
    const resolved = resolveSchemaRefs(schema);
    expect(resolved).toEqual(schema);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-component/src/model/schema-ref-resolver.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement resolveSchemaRefs**

```typescript
// packages/pages-component/src/model/schema-ref-resolver.ts
import type { FieldSchema } from "./form-input-types.js";

export function resolveSchemaRefs(schema: FieldSchema): FieldSchema {
  const defs: Readonly<Record<string, FieldSchema>> = schema.$defs ?? schema.definitions ?? {};
  return resolveNode(schema, defs, new Set());
}

function resolveNode(
  node: FieldSchema,
  defs: Readonly<Record<string, FieldSchema>>,
  visiting: Set<string>,
): FieldSchema {
  const ref = node.$ref;
  if (ref) {
    const defName = ref.replace(/^#\/\$defs\/|^#\/definitions\//, "");
    if (visiting.has(defName)) return {};
    visiting.add(defName);
    const resolved = defs[defName];
    if (!resolved) { visiting.delete(defName); return node; }
    const result = resolveNode(resolved, defs, visiting);
    visiting.delete(defName);
    return result;
  }

  const resolved: Record<string, unknown> = { ...node };
  if (node.properties) {
    const props: Record<string, FieldSchema> = {};
    for (const [key, prop] of Object.entries(node.properties)) {
      props[key] = resolveNode(prop, defs, visiting);
    }
    resolved.properties = props;
  }
  if (node.items) {
    resolved.items = resolveNode(node.items, defs, visiting);
  }
  if (node.oneOf) {
    resolved.oneOf = node.oneOf.map(v => resolveNode(v, defs, visiting));
  }
  return resolved as FieldSchema;
}
```

- [ ] **Step 4: Export resolveSchemaRefs from barrel**

In `packages/pages-component/src/model/index.ts`, add after the FormValueMixin export:

```typescript
export { resolveSchemaRefs } from "./schema-ref-resolver.js";
```

- [ ] **Step 5: Run resolveSchemaRefs tests**

Run: `yarn vitest run packages/pages-component/src/model/schema-ref-resolver.test.ts`
Expected: PASS — all 8 tests green

- [ ] **Step 6: Add exclusiveMinimum/exclusiveMaximum to validateField**

In `packages/pages-component/src/model/field-validation.ts`, add after the `maximum` check (after line 29):

```typescript
    if (schema.exclusiveMinimum != null && value <= schema.exclusiveMinimum) {
      return `Must be greater than ${schema.exclusiveMinimum}`;
    }
    if (schema.exclusiveMaximum != null && value >= schema.exclusiveMaximum) {
      return `Must be less than ${schema.exclusiveMaximum}`;
    }
```

- [ ] **Step 7: Write tests for extended mapFieldToComponentType**

```typescript
// packages/pages-viz/src/form-inputs/schema-types.test.ts
import { describe, it, expect } from "vitest";
import { mapFieldToComponentType } from "./schema-types.js";

describe("mapFieldToComponentType — composite types", () => {
  it("x-renderer takes highest priority", () => {
    expect(mapFieldToComponentType({ type: "string", "x-renderer": "my-widget" } as any)).toBe("my-widget");
  });

  it("oneOf maps to variant-group", () => {
    expect(mapFieldToComponentType({
      oneOf: [
        { properties: { method: { const: "email" } } },
        { properties: { method: { const: "phone" } } },
      ],
    })).toBe("variant-group");
  });

  it("type array maps to array-group", () => {
    expect(mapFieldToComponentType({ type: "array", items: { type: "string" } })).toBe("array-group");
  });

  it("items without type maps to array-group", () => {
    expect(mapFieldToComponentType({ items: { type: "string" } })).toBe("array-group");
  });

  it("type object maps to object-group", () => {
    expect(mapFieldToComponentType({
      type: "object",
      properties: { name: { type: "string" } },
    })).toBe("object-group");
  });

  it("properties without type maps to object-group", () => {
    expect(mapFieldToComponentType({
      properties: { name: { type: "string" } },
    })).toBe("object-group");
  });

  it("type array ['string', 'null'] normalizes to string", () => {
    expect(mapFieldToComponentType({ type: ["string", "null"] as any })).toBe("input");
  });

  it("type array ['object', 'null'] normalizes to object-group", () => {
    expect(mapFieldToComponentType({
      type: ["object", "null"] as any,
      properties: { a: { type: "string" } },
    })).toBe("object-group");
  });

  it("existing leaf mappings unchanged", () => {
    expect(mapFieldToComponentType({ type: "string" })).toBe("input");
    expect(mapFieldToComponentType({ type: "number" })).toBe("number-input");
    expect(mapFieldToComponentType({ type: "integer" })).toBe("number-input");
    expect(mapFieldToComponentType({ type: "boolean" })).toBe("checkbox");
    expect(mapFieldToComponentType({ type: "string", enum: ["a", "b"] })).toBe("select");
    expect(mapFieldToComponentType({ type: "string", format: "date" })).toBe("date-input");
    expect(mapFieldToComponentType({ type: "string", format: "datetime-local" })).toBe("datetime-input");
    expect(mapFieldToComponentType({ type: "string", format: "textarea" })).toBe("textarea");
  });
});
```

- [ ] **Step 8: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/schema-types.test.ts`
Expected: FAIL — composite type tests fail (current mapping returns "input" for objects/arrays)

- [ ] **Step 9: Extend mapFieldToComponentType**

Replace the function in `packages/pages-viz/src/form-inputs/schema-types.ts:38-51` with:

```typescript
export function mapFieldToComponentType(fieldSchema: FieldSchema): string {
  if (fieldSchema["x-renderer"]) return String(fieldSchema["x-renderer"]);
  if (fieldSchema.oneOf) return "variant-group";

  const effectiveType = Array.isArray(fieldSchema.type)
    ? (fieldSchema.type as readonly string[]).find(t => t !== "null")
    : fieldSchema.type;

  if (effectiveType === "array" || fieldSchema.items) return "array-group";
  if (effectiveType === "object" || (fieldSchema.properties && effectiveType !== "string")) return "object-group";
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

- [ ] **Step 10: Run all tests to verify they pass**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/schema-types.test.ts`
Expected: PASS — all composite and leaf tests green

Run: `yarn vitest run packages/pages-component/src/model/`
Expected: PASS — all foundation tests green

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add packages/pages-component/src/model/schema-ref-resolver.ts packages/pages-component/src/model/schema-ref-resolver.test.ts packages/pages-component/src/model/index.ts packages/pages-component/src/model/field-validation.ts packages/pages-viz/src/form-inputs/schema-types.ts packages/pages-viz/src/form-inputs/schema-types.test.ts
git commit -m "feat(#222): add resolveSchemaRefs, extend mapFieldToComponentType for composite types, add exclusiveMin/Max validation"
```

---

## Batch 2: Object Nesting

After this batch: PagesSchemaForm renders nested objects. A schema with `type: "object"` properties produces `pages-object-group` fieldsets with recursive sub-field rendering. Values are collected as structured JSON records.

### Task 4: pages-object-group component

**Files:**
- Create: `packages/pages-viz/src/form-inputs/PagesObjectGroup.ts`
- Create: `packages/pages-viz/src/form-inputs/object-group.test.ts`
- Modify: `packages/pages-viz/src/form-inputs/index.ts` (if barrel exists, otherwise skip)

**Interfaces:**
- Consumes: `FormValueMixin` from Task 2, `mapFieldToComponentType` from Task 3, `isFormValueProvider` from Task 1, `readFieldValue`/`setFieldError`/`validateField` from `@casehubio/pages-component`
- Produces: `PagesObjectGroup` — `pages-object-group` custom element. Props: `schema: FieldSchema`, `label: string`, `fieldName: string`, `editable: boolean`, `required: boolean`, `collapsible: boolean`, `validateOnBlur: boolean`. Implements `FormValueProvider`.

- [ ] **Step 1: Write tests for pages-object-group**

```typescript
// packages/pages-viz/src/form-inputs/object-group.test.ts
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import type { FieldSchema } from "@casehubio/pages-component";
import type { PagesObjectGroup } from "./PagesObjectGroup.js";
import "./PagesObjectGroup.js";

describe("PagesObjectGroup", () => {
  let container: HTMLDivElement;

  beforeEach(() => {
    container = document.createElement("div");
    document.body.appendChild(container);
  });

  afterEach(() => {
    container.remove();
  });

  it("renders fieldset with legend from label", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = { type: "object", properties: { name: { type: "string" } } };
    el.label = "Address";
    el.fieldName = "address";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const fieldset = el.shadowRoot!.querySelector("fieldset");
    expect(fieldset).not.toBeNull();
    const legend = el.shadowRoot!.querySelector("legend");
    expect(legend!.textContent).toContain("Address");
  });

  it("creates leaf inputs for flat properties", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = {
      type: "object",
      properties: {
        street: { type: "string" },
        zip: { type: "string" },
      },
    };
    el.label = "Address";
    el.fieldName = "address";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const inputs = el.shadowRoot!.querySelectorAll("pages-input");
    expect(inputs.length).toBe(2);
  });

  it("currentValue returns nested record", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = {
      type: "object",
      properties: {
        street: { type: "string" },
        city: { type: "string" },
      },
    };
    el.label = "Address";
    el.fieldName = "address";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const inputs = el.shadowRoot!.querySelectorAll("pages-input");
    (inputs[0] as any).value = "123 Main";
    (inputs[1] as any).value = "NYC";

    const value = el.currentValue as Record<string, unknown>;
    expect(value.street).toBe("123 Main");
    expect(value.city).toBe("NYC");
  });

  it("propagateValue distributes to children", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = {
      type: "object",
      properties: {
        street: { type: "string" },
        city: { type: "string" },
      },
    };
    el.label = "Address";
    el.fieldName = "address";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    el.value = { street: "456 Oak", city: "LA" };
    await el.updateComplete;

    const inputs = el.shadowRoot!.querySelectorAll("pages-input");
    expect((inputs[0] as any).value).toBe("456 Oak");
    expect((inputs[1] as any).value).toBe("LA");
  });

  it("validate checks required sub-properties", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = {
      type: "object",
      properties: {
        street: { type: "string" },
        city: { type: "string" },
      },
      required: ["street", "city"],
    };
    el.label = "Address";
    el.fieldName = "address";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    // Fields are empty — validation should fail
    expect(el.validate()).toBe(false);
  });

  it("re-emits committed pages-field-change with composite field name", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = {
      type: "object",
      properties: { name: { type: "string" } },
    };
    el.label = "Person";
    el.fieldName = "person";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener("pages-field-change", (e) => events.push(e as CustomEvent));

    const input = el.shadowRoot!.querySelector("pages-input")!;
    (input as any).value = "Jane";
    input.dispatchEvent(new CustomEvent("pages-field-change", {
      bubbles: true, composed: true,
      detail: { field: "name", value: "Jane", committed: true },
    }));

    expect(events.length).toBe(1);
    expect(events[0]!.detail.field).toBe("person");
    expect(events[0]!.detail.committed).toBe(true);
    expect(events[0]!.detail.value).toEqual({ name: "Jane" });
  });

  it("does not re-emit uncommitted events", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = {
      type: "object",
      properties: { name: { type: "string" } },
    };
    el.label = "Person";
    el.fieldName = "person";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener("pages-field-change", (e) => events.push(e as CustomEvent));

    const input = el.shadowRoot!.querySelector("pages-input")!;
    input.dispatchEvent(new CustomEvent("pages-field-change", {
      bubbles: true, composed: true,
      detail: { field: "name", value: "J", committed: false },
    }));

    expect(events.length).toBe(0);
  });

  it("recursively renders nested object-groups", async () => {
    const el = document.createElement("pages-object-group") as PagesObjectGroup;
    el.schema = {
      type: "object",
      properties: {
        name: { type: "string" },
        coordinates: {
          type: "object",
          properties: {
            lat: { type: "number" },
            lng: { type: "number" },
          },
        },
      },
    };
    el.label = "Location";
    el.fieldName = "location";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const nestedGroup = el.shadowRoot!.querySelector("pages-object-group");
    expect(nestedGroup).not.toBeNull();
    expect((nestedGroup as any).label).toBe("Coordinates");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/object-group.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PagesObjectGroup**

```typescript
// packages/pages-viz/src/form-inputs/PagesObjectGroup.ts
import { html, css, type TemplateResult } from "lit";
import { property, state } from "lit/decorators.js";
import { LitElement } from "lit";
import type { FieldSchema } from "@casehubio/pages-component";
import { isFormValueProvider, readFieldValue, setFieldError, validateField, FormValueMixin } from "@casehubio/pages-component";
import { mapFieldToComponentType } from "./schema-types.js";

import "@casehubio/pages-ui-components/input";
import "@casehubio/pages-ui-components/select";
import "@casehubio/pages-ui-components/checkbox";
import "@casehubio/pages-ui-components/textarea";
import "@casehubio/pages-ui-components/number-input";
import "@casehubio/pages-ui-components/date-input";
import "@casehubio/pages-ui-components/datetime-input";

const STANDALONE_LEAF_TYPES = new Set(["input", "select", "textarea", "checkbox", "number-input", "date-input", "datetime-input"]);
const COMPOSITE_TYPES = new Set(["object-group", "array-group", "variant-group"]);

export class PagesObjectGroup extends FormValueMixin(LitElement) {
  @property({ attribute: false }) schema!: FieldSchema;
  @property({ attribute: false }) label = "";
  @property({ attribute: false }) fieldName = "";
  @property({ type: Boolean }) editable = false;
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) collapsible = false;
  @property({ type: Boolean }) validateOnBlur = false;
  @state() private _collapsed = false;

  private _children = new Map<string, HTMLElement>();
  private _childTypes = new Map<string, string>();

  static override styles = css`
    :host { display: block; }
    fieldset {
      border: 1px solid var(--pages-border-color, #e0e0e0);
      border-radius: var(--pages-radius-sm, 4px);
      padding: var(--pages-space-3, 12px);
      margin: var(--pages-space-2, 8px) 0;
    }
    legend {
      font-weight: 600;
      font-size: var(--pages-font-size-sm, 13px);
      color: var(--pages-text-secondary, #666);
      padding: 0 var(--pages-space-1, 4px);
    }
    .object-fields { display: flex; flex-direction: column; gap: var(--pages-space-2, 8px); }
    .collapsed .object-fields { display: none; }
    .collapse-btn { cursor: pointer; background: none; border: none; padding: 0; font: inherit; color: inherit; }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.renderRoot.addEventListener("pages-field-change", this._onChildFieldChange);
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this.renderRoot.removeEventListener("pages-field-change", this._onChildFieldChange);
  }

  protected collectValue(): Record<string, unknown> {
    const record: Record<string, unknown> = {};
    for (const [field, child] of this._children) {
      record[field] = isFormValueProvider(child)
        ? child.currentValue
        : readFieldValue(child, this._childTypes.get(field) ?? "input");
    }
    return record;
  }

  protected propagateValue(v: unknown): void {
    if (!v || typeof v !== "object") return;
    const obj = v as Record<string, unknown>;
    for (const [field, child] of this._children) {
      const fieldValue = obj[field];
      if (fieldValue === undefined) continue;
      if (isFormValueProvider(child)) {
        child.value = fieldValue;
      } else {
        const ct = this._childTypes.get(field) ?? "input";
        if (ct === "checkbox") {
          (child as any).checked = Boolean(fieldValue);
        } else {
          (child as any).value = fieldValue;
        }
      }
    }
  }

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
    if (allValid) this.error = undefined;
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

  private _onChildFieldChange = (e: Event): void => {
    const detail = (e as CustomEvent).detail;
    if (!detail.committed) return;
    e.stopPropagation();

    if (this.validateOnBlur) {
      const childField = detail.field as string;
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
      detail: { field: this.fieldName, value: this.currentValue, committed: true },
    }));
  };

  private _deriveLabel(field: string, fieldSchema: FieldSchema): string {
    return fieldSchema.title ?? field.replace(/([A-Z])/g, " $1").replace(/^./, (s) => s.toUpperCase());
  }

  override render(): TemplateResult {
    const schemaProps = this.schema.properties ?? {};
    const fields = Object.keys(schemaProps);
    const staleKeys = new Set(this._children.keys());

    for (const field of fields) {
      staleKeys.delete(field);
      const fieldSchema = schemaProps[field]!;
      const componentType = mapFieldToComponentType(fieldSchema);
      const tagName = `pages-${componentType}`;
      const label = this._deriveLabel(field, fieldSchema);

      let child = this._children.get(field);
      if (!child || child.tagName.toLowerCase() !== tagName) {
        child = document.createElement(tagName);
        this._children.set(field, child);
      }
      this._childTypes.set(field, componentType);

      if (COMPOSITE_TYPES.has(componentType)) {
        (child as any).schema = fieldSchema;
        (child as any).label = label;
        (child as any).fieldName = field;
        (child as any).editable = this.editable;
        (child as any).validateOnBlur = this.validateOnBlur;
        const requiredSet = new Set(this.schema.required ?? []);
        (child as any).required = requiredSet.has(field);
      } else if (STANDALONE_LEAF_TYPES.has(componentType)) {
        (child as any).label = label;
        (child as any).disabled = !this.editable;
        const requiredSet = new Set(this.schema.required ?? []);
        (child as any).required = requiredSet.has(field);
        if (componentType === "select" && fieldSchema.enum) {
          (child as any).options = fieldSchema.enum.map((v: string) => ({ value: v, label: v }));
        }
        if (componentType === "number-input") {
          if (fieldSchema.minimum !== undefined) (child as any).min = fieldSchema.minimum;
          if (fieldSchema.maximum !== undefined) (child as any).max = fieldSchema.maximum;
          if (fieldSchema.type === "integer") (child as any).step = 1;
        }
      }
    }

    for (const key of staleKeys) {
      this._children.get(key)?.remove();
      this._children.delete(key);
      this._childTypes.delete(key);
    }

    const legendId = `legend-${this.fieldName}`;
    return html`
      <fieldset class="${this._collapsed ? "collapsed" : ""}" role="group" aria-labelledby="${legendId}">
        ${this.collapsible
          ? html`<legend id="${legendId}"><button class="collapse-btn" @click=${() => { this._collapsed = !this._collapsed; }} aria-expanded="${!this._collapsed}">${this.label}</button></legend>`
          : html`<legend id="${legendId}">${this.label}</legend>`
        }
        <div class="object-fields">
          ${fields.map((field) => this._children.get(field)!)}
        </div>
      </fieldset>
    `;
  }
}

if (!customElements.get("pages-object-group")) {
  customElements.define("pages-object-group", PagesObjectGroup);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/object-group.test.ts`
Expected: PASS — all 8 tests green

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-viz/src/form-inputs/PagesObjectGroup.ts packages/pages-viz/src/form-inputs/object-group.test.ts
git commit -m "feat(#222): add pages-object-group — recursive fieldset for nested object schemas"
```

---

### Task 5: PagesSchemaForm recursive rendering + $ref + extractRecord adapter

**Files:**
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`
- Test: `packages/pages-viz/src/form-inputs/schema-form.test.ts` (add new tests)

**Interfaces:**
- Consumes: `PagesObjectGroup` from Task 4, `resolveSchemaRefs` from Task 3, `mapFieldToComponentType` (extended) from Task 3, `isFormValueProvider` from Task 1
- Produces: Updated `PagesSchemaForm` that recursively renders composite children, resolves `$ref`, and extracts structured records from datasets

- [ ] **Step 1: Write tests for nested object rendering in PagesSchemaForm**

Append to `packages/pages-viz/src/form-inputs/schema-form.test.ts`:

```typescript
import "./PagesObjectGroup.js";

describe("PagesSchemaForm — nested objects", () => {
  let container: HTMLDivElement;

  beforeEach(() => {
    container = document.createElement("div");
    document.body.appendChild(container);
  });

  afterEach(() => {
    container.remove();
  });

  it("renders pages-object-group for type: object property", async () => {
    const ds = makeDataSet([["name", "TEXT"]], [["Alice"]]);
    const form = document.createElement("pages-schema-form") as PagesSchemaForm;
    form.props = {
      schema: {
        properties: {
          name: { type: "string" },
          address: {
            type: "object",
            properties: {
              street: { type: "string" },
              city: { type: "string" },
            },
          },
        },
      },
    };
    form.editable = true;
    container.appendChild(form);
    await form.updateComplete;
    form.dataSet = ds;
    await form.updateComplete;

    expect(form.shadowRoot!.querySelector("pages-input")).not.toBeNull();
    expect(form.shadowRoot!.querySelector("pages-object-group")).not.toBeNull();
  });

  it("currentValue includes nested object values", async () => {
    const ds = makeDataSet([], []);
    const form = document.createElement("pages-schema-form") as PagesSchemaForm;
    form.props = {
      schema: {
        properties: {
          name: { type: "string" },
          address: {
            type: "object",
            properties: {
              street: { type: "string" },
              city: { type: "string" },
            },
          },
        },
      },
      forceCreate: true,
    };
    form.editable = true;
    container.appendChild(form);
    await form.updateComplete;
    form.dataSet = ds;
    await form.updateComplete;

    const textInput = form.shadowRoot!.querySelector("pages-input") as any;
    textInput.value = "Jane";

    const objectGroup = form.shadowRoot!.querySelector("pages-object-group") as any;
    const innerInputs = objectGroup.shadowRoot!.querySelectorAll("pages-input");
    (innerInputs[0] as any).value = "123 Main";
    (innerInputs[1] as any).value = "NYC";

    const value = form.currentValue;
    expect(value.name).toBe("Jane");
    expect((value.address as any).street).toBe("123 Main");
    expect((value.address as any).city).toBe("NYC");
  });

  it("resolves $ref before rendering", async () => {
    const ds = makeDataSet([], []);
    const form = document.createElement("pages-schema-form") as PagesSchemaForm;
    form.props = {
      schema: {
        $defs: {
          address: {
            type: "object",
            properties: { street: { type: "string" } },
          },
        },
        properties: {
          home: { $ref: "#/$defs/address" },
        },
      },
      forceCreate: true,
    };
    form.editable = true;
    container.appendChild(form);
    await form.updateComplete;
    form.dataSet = ds;
    await form.updateComplete;

    const objectGroup = form.shadowRoot!.querySelector("pages-object-group");
    expect(objectGroup).not.toBeNull();
  });

  it("extractRecord parses JSON columns for nested types", async () => {
    const ds = makeDataSet(
      [["name", "TEXT"], ["address", "TEXT"]],
      [["Alice", '{"street":"123 Main","city":"NYC"}']],
    );
    const form = document.createElement("pages-schema-form") as PagesSchemaForm;
    form.props = {
      schema: {
        properties: {
          name: { type: "string" },
          address: {
            type: "object",
            properties: { street: { type: "string" }, city: { type: "string" } },
          },
        },
      },
    };
    form.editable = true;
    container.appendChild(form);
    await form.updateComplete;
    form.dataSet = ds;
    await form.updateComplete;

    const objectGroup = form.shadowRoot!.querySelector("pages-object-group") as any;
    expect(objectGroup).not.toBeNull();
    // The object-group should have received the parsed JSON as its value
    const groupValue = objectGroup.currentValue;
    expect(groupValue.street).toBe("123 Main");
    expect(groupValue.city).toBe("NYC");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/schema-form.test.ts`
Expected: FAIL — nested object tests fail (no pages-object-group rendered)

- [ ] **Step 3: Modify PagesSchemaForm for recursive rendering**

In `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`:

1. Add imports at top:
```typescript
import { resolveSchemaRefs, isFormValueProvider } from "@casehubio/pages-component";
import "./PagesObjectGroup.js";
```

2. Add composite type constants after imports:
```typescript
const COMPOSITE_TYPES = new Set(["object-group", "array-group", "variant-group"]);
```

3. In `renderContent`, resolve $ref on the schema before processing:
```typescript
const schema = props.schema
  ? resolveSchemaRefs(props.schema)
  : deriveSchemaFromDataSet(dataset);
```

4. In the field loop inside `renderContent`, add composite handling after the standalone branch (after line 182):
```typescript
if (COMPOSITE_TYPES.has(componentType)) {
  (child as any).schema = fieldSchema;
  (child as any).label = label;
  (child as any).fieldName = field;
  (child as any).editable = !isDisplay && this._editable;
  (child as any).validateOnBlur = props.validateOnBlur ?? false;
  const requiredSet = new Set(schema.required ?? []);
  (child as any).required = requiredSet.has(field);

  // Extract nested value from dataset (JSON column)
  if (dataset.rows.length > 0) {
    const row = dataset.rows[0]!;
    try {
      const cell = row.cell(field as ColumnId);
      if (cell.type !== "NULL") {
        const raw = String(cell.value);
        try { (child as any).value = JSON.parse(raw); } catch { /* not JSON */ }
      }
    } catch { /* column not found */ }
  }
}
```

5. Update `currentValue` getter to use `isFormValueProvider` for composites:
```typescript
get currentValue(): Record<string, unknown> {
  const record: Record<string, unknown> = {};
  for (const [field, child] of this._children) {
    const ct = this._childTypes.get(field) ?? "input";
    record[field] = isFormValueProvider(child)
      ? child.currentValue
      : readFieldValue(child, ct);
  }
  return record;
}
```

6. Add `validate()` method for FormValueProvider conformance:
```typescript
validate(): boolean {
  if (!this._resolvedSchema?.properties) return true;
  const requiredSet = new Set(this._resolvedSchema.required ?? []);
  let allValid = true;
  for (const [field, child] of this._children) {
    if (isFormValueProvider(child)) {
      if (!child.validate()) allValid = false;
    } else {
      const fieldSchema = this._resolvedSchema.properties[field];
      if (!fieldSchema) continue;
      const ct = this._childTypes.get(field) ?? "input";
      const value = readFieldValue(child, ct);
      const error = validateField(fieldSchema, value, requiredSet.has(field));
      if (error) {
        this.setChildError(child, ct, error);
        allValid = false;
      } else {
        this.setChildError(child, ct, undefined);
      }
    }
  }
  return allValid;
}
```

7. Update `submit()` to use `validate()`:
```typescript
submit(): Record<string, unknown> | null {
  if (!this._resolvedSchema?.properties) return null;

  const allValid = this.validate();
  if (!allValid) {
    const errorCount = [...this._children.entries()].filter(([, child]) => {
      if (isFormValueProvider(child)) return child.error !== undefined;
      return false;
    }).length;
    this.announce(
      `${String(errorCount || 1)} validation error${errorCount !== 1 ? "s" : ""} — please correct before submitting`,
      "assertive",
    );
    return null;
  }

  const record = this.currentValue;
  this.dispatchEvent(
    new CustomEvent("pages-record-create", {
      bubbles: true, composed: true,
      detail: { record },
    }),
  );
  this.announce("Record submitted successfully");
  return record;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/schema-form.test.ts`
Expected: PASS — all existing + new nested object tests green

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-viz/src/form-inputs/PagesSchemaForm.ts packages/pages-viz/src/form-inputs/schema-form.test.ts
git commit -m "feat(#222): PagesSchemaForm recursive rendering — nested objects, $ref resolution, extractRecord adapter"
```

---

## Batch 3: Arrays

After this batch: Array schemas render as lists with add/remove/reorder controls. Arrays of primitives and arrays of objects both work.

### Task 6: pages-array-group component

**Files:**
- Create: `packages/pages-viz/src/form-inputs/PagesArrayGroup.ts`
- Create: `packages/pages-viz/src/form-inputs/array-group.test.ts`
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` (add import)

**Interfaces:**
- Consumes: `FormValueMixin` from Task 2, `mapFieldToComponentType` from Task 3, `isFormValueProvider`/`readFieldValue`/`setFieldError`/`validateField` from `@casehubio/pages-component`
- Produces: `PagesArrayGroup` — `pages-array-group` custom element. Props: `schema: FieldSchema`, `label: string`, `fieldName: string`, `editable: boolean`, `validateOnBlur: boolean`. Implements `FormValueProvider`. Manages synthetic-key items with add/remove/reorder.

- [ ] **Step 1: Write tests for pages-array-group**

```typescript
// packages/pages-viz/src/form-inputs/array-group.test.ts
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import type { PagesArrayGroup } from "./PagesArrayGroup.js";
import "./PagesArrayGroup.js";
import "./PagesObjectGroup.js";

describe("PagesArrayGroup", () => {
  let container: HTMLDivElement;

  beforeEach(() => {
    container = document.createElement("div");
    document.body.appendChild(container);
  });

  afterEach(() => {
    container.remove();
  });

  it("renders items from value array (primitives)", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" } };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["alpha", "beta"];
    await el.updateComplete;

    const inputs = el.shadowRoot!.querySelectorAll("pages-input");
    expect(inputs.length).toBe(2);
    expect((inputs[0] as any).value).toBe("alpha");
    expect((inputs[1] as any).value).toBe("beta");
  });

  it("add button creates new item", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" } };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["alpha"];
    await el.updateComplete;

    const addBtn = el.shadowRoot!.querySelector(".array-add") as HTMLButtonElement;
    addBtn.click();
    await el.updateComplete;

    const inputs = el.shadowRoot!.querySelectorAll("pages-input");
    expect(inputs.length).toBe(2);
  });

  it("remove button deletes item", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" } };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["alpha", "beta"];
    await el.updateComplete;

    const removeBtns = el.shadowRoot!.querySelectorAll("[aria-label='Remove item']") as NodeListOf<HTMLButtonElement>;
    removeBtns[0]!.click();
    await el.updateComplete;

    const inputs = el.shadowRoot!.querySelectorAll("pages-input");
    expect(inputs.length).toBe(1);
    expect((inputs[0] as any).value).toBe("beta");
  });

  it("currentValue returns items in display order", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" } };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["alpha", "beta", "gamma"];
    await el.updateComplete;

    expect(el.currentValue).toEqual(["alpha", "beta", "gamma"]);
  });

  it("validates minItems", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" }, minItems: 2 };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["alpha"];
    await el.updateComplete;

    expect(el.validate()).toBe(false);
    expect(el.error).toContain("2");
  });

  it("validates maxItems", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" }, maxItems: 2 };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["a", "b", "c"];
    await el.updateComplete;

    expect(el.validate()).toBe(false);
    expect(el.error).toContain("2");
  });

  it("add disabled at maxItems", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" }, maxItems: 2 };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["a", "b"];
    await el.updateComplete;

    const addBtn = el.shadowRoot!.querySelector(".array-add") as HTMLButtonElement;
    expect(addBtn.disabled).toBe(true);
  });

  it("remove disabled at minItems", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" }, minItems: 1 };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["alpha"];
    await el.updateComplete;

    const removeBtns = el.shadowRoot!.querySelectorAll("[aria-label='Remove item']") as NodeListOf<HTMLButtonElement>;
    expect(removeBtns[0]!.disabled).toBe(true);
  });

  it("renders array of objects with pages-object-group per item", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = {
      type: "array",
      items: {
        type: "object",
        properties: {
          street: { type: "string" },
          city: { type: "string" },
        },
      },
    };
    el.label = "Addresses";
    el.fieldName = "addresses";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = [{ street: "123 Main", city: "NYC" }];
    await el.updateComplete;

    const objectGroups = el.shadowRoot!.querySelectorAll("pages-object-group");
    expect(objectGroups.length).toBe(1);
  });

  it("reorder moves item up", async () => {
    const el = document.createElement("pages-array-group") as PagesArrayGroup;
    el.schema = { type: "array", items: { type: "string" } };
    el.label = "Tags";
    el.fieldName = "tags";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;
    el.value = ["alpha", "beta", "gamma"];
    await el.updateComplete;

    const upBtns = el.shadowRoot!.querySelectorAll("[aria-label='Move up']") as NodeListOf<HTMLButtonElement>;
    upBtns[1]!.click(); // move "beta" up
    await el.updateComplete;

    expect(el.currentValue).toEqual(["beta", "alpha", "gamma"]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/array-group.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PagesArrayGroup**

```typescript
// packages/pages-viz/src/form-inputs/PagesArrayGroup.ts
import { html, css, type TemplateResult } from "lit";
import { property, state } from "lit/decorators.js";
import { repeat } from "lit/directives/repeat.js";
import { LitElement } from "lit";
import type { FieldSchema } from "@casehubio/pages-component";
import { isFormValueProvider, readFieldValue, setFieldError, validateField, FormValueMixin } from "@casehubio/pages-component";
import { mapFieldToComponentType } from "./schema-types.js";

import "@casehubio/pages-ui-components/input";
import "@casehubio/pages-ui-components/select";
import "@casehubio/pages-ui-components/checkbox";
import "@casehubio/pages-ui-components/number-input";
import "@casehubio/pages-ui-components/date-input";
import "@casehubio/pages-ui-components/datetime-input";
import "@casehubio/pages-ui-components/textarea";
import "./PagesObjectGroup.js";

const COMPOSITE_TYPES = new Set(["object-group", "array-group", "variant-group"]);

interface ArrayItem {
  key: number;
  element: HTMLElement;
  componentType: string;
}

export class PagesArrayGroup extends FormValueMixin(LitElement) {
  @property({ attribute: false }) schema!: FieldSchema;
  @property({ attribute: false }) label = "";
  @property({ attribute: false }) fieldName = "";
  @property({ type: Boolean }) editable = false;
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) validateOnBlur = false;
  @state() private _items: ArrayItem[] = [];
  private _nextKey = 0;

  static override styles = css`
    :host { display: block; }
    .array-group { display: flex; flex-direction: column; gap: var(--pages-space-2, 8px); }
    .array-header { display: flex; align-items: center; gap: var(--pages-space-2, 8px); font-weight: 600; font-size: var(--pages-font-size-sm, 13px); color: var(--pages-text-secondary, #666); }
    .array-count { font-weight: 400; color: var(--pages-text-tertiary, #999); }
    .array-item { display: flex; align-items: flex-start; gap: var(--pages-space-2, 8px); padding: var(--pages-space-2, 8px); border: 1px solid var(--pages-border-color, #e0e0e0); border-radius: var(--pages-radius-sm, 4px); }
    .array-item > :first-child { flex: 1; }
    .array-item-controls { display: flex; flex-direction: column; gap: 2px; }
    .array-item-controls button { padding: 2px 6px; border: 1px solid var(--pages-border-color, #e0e0e0); border-radius: 3px; background: var(--pages-surface-1, #fff); cursor: pointer; font-size: 12px; line-height: 1; }
    .array-item-controls button:disabled { opacity: 0.3; cursor: not-allowed; }
    .array-add { align-self: flex-start; padding: 4px 12px; border: 1px dashed var(--pages-border-color, #e0e0e0); border-radius: var(--pages-radius-sm, 4px); background: none; cursor: pointer; color: var(--pages-accent-9, #5470c6); font-size: var(--pages-font-size-sm, 13px); }
    .array-add:disabled { opacity: 0.3; cursor: not-allowed; }
    .error-msg { color: var(--pages-error, #e53e3e); font-size: var(--pages-font-size-xs, 12px); }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.renderRoot.addEventListener("pages-field-change", this._onChildFieldChange);
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this.renderRoot.removeEventListener("pages-field-change", this._onChildFieldChange);
  }

  private _createItem(itemValue?: unknown): ArrayItem {
    const itemSchema = this.schema.items ?? { type: "string" };
    const componentType = mapFieldToComponentType(itemSchema);
    const tagName = `pages-${componentType}`;
    const element = document.createElement(tagName);
    const key = this._nextKey++;

    if (COMPOSITE_TYPES.has(componentType)) {
      (element as any).schema = itemSchema;
      (element as any).label = `${this.label} ${key + 1}`;
      (element as any).fieldName = String(key);
      (element as any).editable = this.editable;
      (element as any).validateOnBlur = this.validateOnBlur;
      if (itemValue !== undefined) (element as any).value = itemValue;
    } else {
      (element as any).label = "";
      (element as any).disabled = !this.editable;
      if (itemValue !== undefined) {
        if (componentType === "checkbox") {
          (element as any).checked = Boolean(itemValue);
        } else {
          (element as any).value = itemValue;
        }
      }
    }

    return { key, element, componentType };
  }

  private _getDefaultValue(): unknown {
    const itemSchema = this.schema.items ?? { type: "string" };
    const ct = mapFieldToComponentType(itemSchema);
    if (ct === "checkbox") return false;
    if (ct === "number-input") return 0;
    if (ct === "object-group") return {};
    if (ct === "array-group") return [];
    return "";
  }

  protected collectValue(): unknown[] {
    return this._items.map(item =>
      isFormValueProvider(item.element)
        ? item.element.currentValue
        : readFieldValue(item.element, item.componentType)
    );
  }

  protected propagateValue(v: unknown): void {
    if (!Array.isArray(v)) { this._items = []; return; }
    this._items = v.map(itemValue => this._createItem(itemValue));
    this.requestUpdate();
  }

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

  protected validateChildren(): boolean {
    let allValid = true;
    const itemSchema = this.schema.items ?? { type: "string" };
    for (const item of this._items) {
      if (isFormValueProvider(item.element)) {
        if (!item.element.validate()) allValid = false;
      } else {
        const val = readFieldValue(item.element, item.componentType);
        const err = validateField(itemSchema, val, false);
        setFieldError(item.element, item.componentType, err ?? undefined);
        if (err) allValid = false;
      }
    }
    return allValid;
  }

  private _onChildFieldChange = (e: Event): void => {
    const detail = (e as CustomEvent).detail;
    if (!detail.committed) return;
    e.stopPropagation();

    this.dispatchEvent(new CustomEvent("pages-field-change", {
      bubbles: true, composed: true,
      detail: { field: this.fieldName, value: this.currentValue, committed: true },
    }));
  };

  private _addItem(): void {
    if (this.schema.maxItems != null && this._items.length >= this.schema.maxItems) return;
    const item = this._createItem(this._getDefaultValue());
    this._items = [...this._items, item];
  }

  private _removeItem(key: number): void {
    if (this.schema.minItems != null && this._items.length <= this.schema.minItems) return;
    this._items = this._items.filter(i => i.key !== key);
  }

  private _moveUp(key: number): void {
    const idx = this._items.findIndex(i => i.key === key);
    if (idx <= 0) return;
    const newItems = [...this._items];
    [newItems[idx - 1], newItems[idx]] = [newItems[idx]!, newItems[idx - 1]!];
    this._items = newItems;
  }

  private _moveDown(key: number): void {
    const idx = this._items.findIndex(i => i.key === key);
    if (idx < 0 || idx >= this._items.length - 1) return;
    const newItems = [...this._items];
    [newItems[idx], newItems[idx + 1]] = [newItems[idx + 1]!, newItems[idx]!];
    this._items = newItems;
  }

  override render(): TemplateResult {
    const atMaxItems = this.schema.maxItems != null && this._items.length >= this.schema.maxItems;
    const atMinItems = this.schema.minItems != null && this._items.length <= this.schema.minItems;

    return html`
      <div class="array-group" role="list" aria-label="${this.label}">
        <div class="array-header">
          <span class="array-label">${this.label}</span>
          <span class="array-count">${this._items.length} item${this._items.length !== 1 ? "s" : ""}</span>
        </div>
        ${this.error ? html`<div class="error-msg">${this.error}</div>` : ""}
        ${repeat(this._items, item => item.key, (item, index) => html`
          <div class="array-item" role="listitem">
            ${item.element}
            ${this.editable ? html`
              <div class="array-item-controls">
                <button @click=${() => this._moveUp(item.key)} ?disabled=${index === 0}
                  aria-label="Move up">↑</button>
                <button @click=${() => this._moveDown(item.key)} ?disabled=${index === this._items.length - 1}
                  aria-label="Move down">↓</button>
                <button @click=${() => this._removeItem(item.key)}
                  ?disabled=${atMinItems} aria-label="Remove item">×</button>
              </div>
            ` : ""}
          </div>
        `)}
        ${this.editable ? html`
          <button class="array-add" @click=${this._addItem}
            ?disabled=${atMaxItems} aria-label="Add ${this.label}">
            + Add
          </button>
        ` : ""}
      </div>
    `;
  }
}

if (!customElements.get("pages-array-group")) {
  customElements.define("pages-array-group", PagesArrayGroup);
}
```

- [ ] **Step 4: Add array-group import to PagesSchemaForm**

In `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`, add import:

```typescript
import "./PagesArrayGroup.js";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/array-group.test.ts`
Expected: PASS — all 10 tests green

Run: `yarn vitest run packages/pages-viz/src/form-inputs/`
Expected: PASS — all form tests green

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-viz/src/form-inputs/PagesArrayGroup.ts packages/pages-viz/src/form-inputs/array-group.test.ts packages/pages-viz/src/form-inputs/PagesSchemaForm.ts
git commit -m "feat(#222): add pages-array-group — list with add/remove/reorder for array schemas"
```

---

## Batch 4: Variants + Integration

After this batch: Full nested schema support — oneOf discriminated unions, custom renderers, formScope integration with composite validation, pipeline adapter. All features complete.

### Task 7: pages-variant-group component

**Files:**
- Create: `packages/pages-viz/src/form-inputs/PagesVariantGroup.ts`
- Create: `packages/pages-viz/src/form-inputs/variant-group.test.ts`
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` (add import)

**Interfaces:**
- Consumes: `FormValueMixin` from Task 2, `mapFieldToComponentType` from Task 3, `isFormValueProvider`/`readFieldValue` from `@casehubio/pages-component`
- Produces: `PagesVariantGroup` — `pages-variant-group` custom element. Props: `schema: FieldSchema`, `label: string`, `fieldName: string`, `editable: boolean`. Implements `FormValueProvider`. Auto-detects discriminator from `const` values.

- [ ] **Step 1: Write tests for pages-variant-group**

```typescript
// packages/pages-viz/src/form-inputs/variant-group.test.ts
import { describe, it, expect, beforeEach, afterEach, vi } from "vitest";
import type { PagesVariantGroup } from "./PagesVariantGroup.js";
import "./PagesVariantGroup.js";

describe("PagesVariantGroup", () => {
  let container: HTMLDivElement;

  beforeEach(() => {
    container = document.createElement("div");
    document.body.appendChild(container);
  });

  afterEach(() => {
    container.remove();
  });

  const contactSchema = {
    oneOf: [
      {
        properties: {
          method: { const: "email" },
          address: { type: "string" as const },
        },
        required: ["method", "address"] as readonly string[],
      },
      {
        properties: {
          method: { const: "phone" },
          number: { type: "string" as const },
        },
        required: ["method", "number"] as readonly string[],
      },
    ],
  };

  it("auto-detects discriminator from const values", async () => {
    const el = document.createElement("pages-variant-group") as PagesVariantGroup;
    el.schema = contactSchema;
    el.label = "Contact";
    el.fieldName = "contact";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const select = el.shadowRoot!.querySelector("pages-select");
    expect(select).not.toBeNull();
  });

  it("renders active variant fields (excluding discriminator)", async () => {
    const el = document.createElement("pages-variant-group") as PagesVariantGroup;
    el.schema = contactSchema;
    el.label = "Contact";
    el.fieldName = "contact";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    // Default is first variant (email) — should show "address" input, no "number"
    const inputs = el.shadowRoot!.querySelectorAll("pages-input");
    expect(inputs.length).toBe(1);
  });

  it("currentValue includes discriminator value", async () => {
    const el = document.createElement("pages-variant-group") as PagesVariantGroup;
    el.schema = contactSchema;
    el.label = "Contact";
    el.fieldName = "contact";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    const input = el.shadowRoot!.querySelector("pages-input") as any;
    input.value = "jane@example.com";

    const value = el.currentValue as Record<string, unknown>;
    expect(value.method).toBe("email");
    expect(value.address).toBe("jane@example.com");
  });

  it("validates only active variant", async () => {
    const el = document.createElement("pages-variant-group") as PagesVariantGroup;
    el.schema = contactSchema;
    el.label = "Contact";
    el.fieldName = "contact";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    // Email variant active, address is required but empty
    expect(el.validate()).toBe(false);
  });

  it("logs error for undiscriminated oneOf", async () => {
    const consoleSpy = vi.spyOn(console, "error").mockImplementation(() => {});
    const el = document.createElement("pages-variant-group") as PagesVariantGroup;
    el.schema = {
      oneOf: [
        { properties: { a: { type: "string" } } },
        { properties: { b: { type: "number" } } },
      ],
    };
    el.label = "Thing";
    el.fieldName = "thing";
    el.editable = true;
    container.appendChild(el);
    await el.updateComplete;

    expect(consoleSpy).toHaveBeenCalledWith(expect.stringContaining("no discriminator"));
    consoleSpy.mockRestore();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/variant-group.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PagesVariantGroup**

```typescript
// packages/pages-viz/src/form-inputs/PagesVariantGroup.ts
import { html, css, type TemplateResult } from "lit";
import { property, state } from "lit/decorators.js";
import { LitElement } from "lit";
import type { FieldSchema } from "@casehubio/pages-component";
import { isFormValueProvider, readFieldValue, setFieldError, validateField, FormValueMixin } from "@casehubio/pages-component";
import { mapFieldToComponentType } from "./schema-types.js";

import "@casehubio/pages-ui-components/input";
import "@casehubio/pages-ui-components/select";
import "@casehubio/pages-ui-components/checkbox";
import "@casehubio/pages-ui-components/number-input";
import "@casehubio/pages-ui-components/date-input";
import "@casehubio/pages-ui-components/datetime-input";
import "@casehubio/pages-ui-components/textarea";
import "./PagesObjectGroup.js";
import "./PagesArrayGroup.js";

const COMPOSITE_TYPES = new Set(["object-group", "array-group", "variant-group"]);

export class PagesVariantGroup extends FormValueMixin(LitElement) {
  @property({ attribute: false }) schema!: FieldSchema;
  @property({ attribute: false }) label = "";
  @property({ attribute: false }) fieldName = "";
  @property({ type: Boolean }) editable = false;
  @state() private _activeVariantIndex = 0;
  @state() private _discriminatorField: string | null = null;

  private _activeChildren = new Map<string, HTMLElement>();
  private _activeChildTypes = new Map<string, string>();

  static override styles = css`
    :host { display: block; }
    fieldset {
      border: 1px solid var(--pages-border-color, #e0e0e0);
      border-radius: var(--pages-radius-sm, 4px);
      padding: var(--pages-space-3, 12px);
      margin: var(--pages-space-2, 8px) 0;
    }
    legend { font-weight: 600; font-size: var(--pages-font-size-sm, 13px); color: var(--pages-text-secondary, #666); padding: 0 var(--pages-space-1, 4px); }
    .variant-fields { display: flex; flex-direction: column; gap: var(--pages-space-2, 8px); margin-top: var(--pages-space-2, 8px); }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.renderRoot.addEventListener("pages-field-change", this._onChildFieldChange);
    this._detectDiscriminator();
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this.renderRoot.removeEventListener("pages-field-change", this._onChildFieldChange);
  }

  override updated(changed: Map<string, unknown>): void {
    super.updated(changed);
    if (changed.has("schema")) this._detectDiscriminator();
  }

  private _detectDiscriminator(): void {
    const variants = this.schema.oneOf;
    if (!variants || variants.length === 0) return;

    const candidateProps = new Set<string>();
    for (const variant of variants) {
      for (const prop of Object.keys(variant.properties ?? {})) {
        candidateProps.add(prop);
      }
    }

    for (const prop of candidateProps) {
      const allHaveConst = variants.every(v => v.properties?.[prop]?.const !== undefined);
      if (allHaveConst) {
        this._discriminatorField = prop;
        return;
      }
    }

    console.error(
      "pages-variant-group: oneOf schema has no discriminator property (no shared property with const values across all variants). Use x-renderer for undiscriminated oneOf."
    );
  }

  private _getVariantOptions(): Array<{ value: string; label: string }> {
    const variants = this.schema.oneOf ?? [];
    if (!this._discriminatorField) return [];
    return variants.map((v, i) => {
      const constVal = String(v.properties?.[this._discriminatorField!]?.const ?? i);
      const label = v.title ?? constVal;
      return { value: constVal, label };
    });
  }

  private _onVariantSwitch = (e: Event): void => {
    const detail = (e as CustomEvent).detail;
    if (!detail.committed) return;
    e.stopPropagation();

    const newValue = detail.value as string;
    const variants = this.schema.oneOf ?? [];
    const newIndex = variants.findIndex(v =>
      String(v.properties?.[this._discriminatorField!]?.const) === newValue
    );
    if (newIndex >= 0 && newIndex !== this._activeVariantIndex) {
      this._activeVariantIndex = newIndex;
      this._activeChildren.clear();
      this._activeChildTypes.clear();
    }
  };

  private _onChildFieldChange = (e: Event): void => {
    const detail = (e as CustomEvent).detail;
    if (!detail.committed) return;
    e.stopPropagation();

    this.dispatchEvent(new CustomEvent("pages-field-change", {
      bubbles: true, composed: true,
      detail: { field: this.fieldName, value: this.currentValue, committed: true },
    }));
  };

  protected collectValue(): Record<string, unknown> {
    const variants = this.schema.oneOf;
    if (!variants || variants.length === 0) return {};
    const activeVariant = variants[this._activeVariantIndex]!;
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

  protected propagateValue(v: unknown): void {
    if (!v || typeof v !== "object") return;
    const obj = v as Record<string, unknown>;

    if (this._discriminatorField && obj[this._discriminatorField] !== undefined) {
      const variants = this.schema.oneOf ?? [];
      const idx = variants.findIndex(vr =>
        vr.properties?.[this._discriminatorField!]?.const === obj[this._discriminatorField!]
      );
      if (idx >= 0) this._activeVariantIndex = idx;
    }

    for (const [field, child] of this._activeChildren) {
      if (field === this._discriminatorField) continue;
      const fieldValue = obj[field];
      if (fieldValue === undefined) continue;
      if (isFormValueProvider(child)) {
        child.value = fieldValue;
      } else {
        (child as any).value = fieldValue;
      }
    }
  }

  protected validateSelf(): boolean {
    this.error = undefined;
    return true;
  }

  protected validateChildren(): boolean {
    const variants = this.schema.oneOf;
    if (!variants) return true;
    const activeVariant = variants[this._activeVariantIndex];
    if (!activeVariant) return true;

    const requiredSet = new Set(activeVariant.required ?? []);
    let allValid = true;

    for (const [field, child] of this._activeChildren) {
      if (field === this._discriminatorField) continue;
      if (isFormValueProvider(child)) {
        if (!child.validate()) allValid = false;
      } else {
        const fieldSchema = activeVariant.properties?.[field];
        if (fieldSchema) {
          const val = readFieldValue(child, this._activeChildTypes.get(field) ?? "input");
          const err = validateField(fieldSchema, val, requiredSet.has(field));
          setFieldError(child, this._activeChildTypes.get(field) ?? "input", err ?? undefined);
          if (err) allValid = false;
        }
      }
    }
    return allValid;
  }

  override render(): TemplateResult {
    const variants = this.schema.oneOf ?? [];
    if (variants.length === 0) return html``;

    const activeVariant = variants[this._activeVariantIndex];
    if (!activeVariant) return html``;

    const activeProps = activeVariant.properties ?? {};
    const fields = Object.keys(activeProps).filter(f => f !== this._discriminatorField);
    const staleKeys = new Set(this._activeChildren.keys());

    for (const field of fields) {
      staleKeys.delete(field);
      const fieldSchema = activeProps[field]!;
      const componentType = mapFieldToComponentType(fieldSchema);
      const tagName = `pages-${componentType}`;
      const label = fieldSchema.title ?? field.replace(/([A-Z])/g, " $1").replace(/^./, s => s.toUpperCase());

      let child = this._activeChildren.get(field);
      if (!child || child.tagName.toLowerCase() !== tagName) {
        child = document.createElement(tagName);
        this._activeChildren.set(field, child);
      }
      this._activeChildTypes.set(field, componentType);

      if (COMPOSITE_TYPES.has(componentType)) {
        (child as any).schema = fieldSchema;
        (child as any).label = label;
        (child as any).fieldName = field;
        (child as any).editable = this.editable;
      } else {
        (child as any).label = label;
        (child as any).disabled = !this.editable;
        const requiredSet = new Set(activeVariant.required ?? []);
        (child as any).required = requiredSet.has(field);
        if (componentType === "select" && fieldSchema.enum) {
          (child as any).options = fieldSchema.enum.map((v: string) => ({ value: v, label: v }));
        }
      }
    }

    for (const key of staleKeys) {
      this._activeChildren.get(key)?.remove();
      this._activeChildren.delete(key);
      this._activeChildTypes.delete(key);
    }

    const legendId = `variant-legend-${this.fieldName}`;
    const variantOptions = this._getVariantOptions();
    const activeDiscriminatorValue = this._discriminatorField
      ? String(activeVariant.properties?.[this._discriminatorField]?.const ?? "")
      : "";

    return html`
      <fieldset class="variant-group" role="group" aria-labelledby="${legendId}">
        <legend id="${legendId}">${this.label}</legend>
        ${this._discriminatorField ? html`
          <pages-select
            .label=${this._discriminatorField}
            .options=${variantOptions}
            .value=${activeDiscriminatorValue}
            ?disabled=${!this.editable}
            @pages-field-change=${this._onVariantSwitch}
          ></pages-select>
        ` : ""}
        <div class="variant-fields">
          ${fields.map(field => this._activeChildren.get(field)!)}
        </div>
      </fieldset>
    `;
  }
}

if (!customElements.get("pages-variant-group")) {
  customElements.define("pages-variant-group", PagesVariantGroup);
}
```

- [ ] **Step 4: Add variant-group import to PagesSchemaForm**

In `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`, add:

```typescript
import "./PagesVariantGroup.js";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-viz/src/form-inputs/variant-group.test.ts`
Expected: PASS — all 5 tests green

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-viz/src/form-inputs/PagesVariantGroup.ts packages/pages-viz/src/form-inputs/variant-group.test.ts packages/pages-viz/src/form-inputs/PagesSchemaForm.ts
git commit -m "feat(#222): add pages-variant-group — discriminated union for oneOf schemas"
```

---

### Task 8: Custom renderers (x-renderer) + formScope integration + integration tests

**Files:**
- Modify: `packages/pages-runtime/src/form-scope.ts:43-62`
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`
- Create: `packages/pages-runtime/src/nested-form-integration.test.ts`
- Test: `packages/pages-runtime/src/form-scope.test.ts` (add composite tests if file exists)

**Interfaces:**
- Consumes: `isFormValueProvider` from Task 1, `FormValueProvider` from Task 1, all composite components from Tasks 4-7
- Produces: Updated `FormScopeState.validateAll()` with FormValueProvider dispatch, custom renderer conformance check in PagesSchemaForm, integration tests validating the full pipeline

- [ ] **Step 1: Write test for formScope composite validation**

```typescript
// Append to packages/pages-runtime/src/form-scope.test.ts (or create if it doesn't exist)
import { describe, it, expect } from "vitest";
import { FormScopeState } from "./form-scope.js";
import type { FieldSchema } from "@casehubio/pages-component";

describe("FormScopeState — composite validation", () => {
  it("calls validate() on FormValueProvider-conformant elements", () => {
    const schema: FieldSchema = {
      properties: {
        address: {
          type: "object",
          properties: { street: { type: "string" } },
          required: ["street"],
        },
      },
    };
    const state = new FormScopeState(schema, false);

    let validateCalled = false;
    const mockComposite = {
      currentValue: { street: "" },
      value: {},
      error: "Required" as string | undefined,
      validate: () => { validateCalled = true; return false; },
      isConnected: true,
    } as unknown as HTMLElement;

    state.registerField("address", mockComposite, "object-group");
    const errors = state.validateAll();

    expect(validateCalled).toBe(true);
    expect(errors.address).toBeDefined();
  });

  it("uses validateField for non-FormValueProvider elements", () => {
    const schema: FieldSchema = {
      properties: { name: { type: "string" } },
      required: ["name"],
    };
    const state = new FormScopeState(schema, false);

    const mockInput = {
      value: "",
      isConnected: true,
    } as unknown as HTMLElement;

    state.registerField("name", mockInput, "input");
    const errors = state.validateAll();

    expect(errors.name).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-runtime/src/form-scope.test.ts`
Expected: FAIL — validateAll doesn't check FormValueProvider

- [ ] **Step 3: Update FormScopeState.validateAll for composite validation**

In `packages/pages-runtime/src/form-scope.ts`, add import:

```typescript
import { validateField, readFieldValue, setFieldError, isFormValueProvider } from "@casehubio/pages-component";
```

(Remove the existing separate imports for `validateField`, `readFieldValue`, `setFieldError` if present, consolidating into one line.)

Replace the `validateAll()` method:

```typescript
validateAll(): Record<string, string> {
  this.pruneDisconnected();
  const requiredSet = new Set(this.schema?.required ?? []);
  const errors: Record<string, string> = {};

  for (const [field, entry] of this.fields) {
    if (isFormValueProvider(entry.element)) {
      if (!entry.element.validate()) {
        errors[field] = entry.element.error ?? "Validation failed";
      }
    } else {
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

- [ ] **Step 4: Add x-renderer conformance check to PagesSchemaForm**

In `PagesSchemaForm.ts`, in the `renderContent` method, when creating a child element for a composite or custom type, add after element creation:

```typescript
if (fieldSchema["x-renderer"]) {
  if (!isFormValueProvider(child)) {
    console.warn(
      `pages-schema-form: custom renderer <${child.tagName.toLowerCase()}> does not implement FormValueProvider ` +
      `(missing currentValue getter or validate() method). Values may be undefined and validation skipped.`
    );
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-runtime/src/form-scope.test.ts`
Expected: PASS

Run: `yarn vitest run packages/pages-viz/src/form-inputs/`
Expected: PASS — all form tests green

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-runtime/src/form-scope.ts packages/pages-runtime/src/form-scope.test.ts packages/pages-viz/src/form-inputs/PagesSchemaForm.ts
git commit -m "feat(#222): formScope composite validation, x-renderer conformance check, integration tests"
```

---

## References

- `specs/issue-222-schema-form-nested/2026-09-03-nested-schema-form-design.md` — design spec this plan implements
- `specs/issue-222-schema-form-nested/decisions.md` — 14 design decisions (D1-D14)
- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — current flat rendering
- `packages/pages-viz/src/form-inputs/schema-types.ts` — mapFieldToComponentType
- `packages/pages-component/src/model/form-input-types.ts:55-81` — FieldSchema
- `packages/pages-component/src/model/field-access.ts` — readFieldValue, setFieldError
- `packages/pages-component/src/model/field-validation.ts` — validateField
- `packages/pages-runtime/src/form-scope.ts` — FormScopeState
- `packages/pages-primitives/src/a11y/live-region.ts` — LiveRegionMixin
- `docs/protocols/casehub/web-component-strategy.md` — mixin convention
- casehubio/casehub-pages#222 — issue
- casehubio/casehub-pages#159 — predecessor
- casehubio/casehub-pages#337 — formScope prerequisite
