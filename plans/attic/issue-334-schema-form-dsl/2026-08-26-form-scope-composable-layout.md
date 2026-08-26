# formScope Composable Layout — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #337 — design: make schema-form composable with standard layout primitives
**Issue group:** #337

**Goal:** Separate form management from field generation so form fields compose with standard layout primitives (`columns()`, `rows()`, `grid()`).

**Architecture:** `formScope` is an activation-layer container type (no web component) that provides validation, value collection, and submit wiring to any children. `schemaFields()` produces PagesSchemaForm in `fieldsOnly` mode for auto-generated fields. `submitButton()` provides the form submit trigger. Field discovery uses `pages-field-register` events bubbling to formScope's wrapper `el`.

**Tech Stack:** TypeScript, Lit (PagesSubmitButton), Vitest, pages-component / pages-viz / pages-runtime / pages-ui / pages-ui-components

## Global Constraints

- Build order: pages-component → pages-ui-components → pages-viz → pages-runtime → pages-ui
- All packages must build clean individually after each batch
- Existing `schemaForm()` behavior must remain unchanged
- `STANDALONE_TYPES` canonical definition in pages-component — all consumers import from there
- `validateField()` moves to pages-component (pure function, no DOM/Lit deps)
- No runtime dependency from pages-runtime → pages-viz (type-only imports are OK)

---

## Batch 1: Foundation — shared utilities and types

After this batch: pages-component has all new types, shared field-access utilities, and `validateField`. All existing tests pass.

### Task 1: Move validateField + create field-access utilities

**Files:**
- Create: `packages/pages-component/src/model/field-validation.ts`
- Create: `packages/pages-component/src/model/field-access.ts`
- Modify: `packages/pages-component/src/model/index.ts`
- Modify: `packages/pages-viz/src/form-inputs/schema-types.ts`
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`
- Test: `packages/pages-component/src/model/field-validation.test.ts`
- Test: `packages/pages-component/src/model/field-access.test.ts`

**Interfaces:**
- Consumes: `FieldSchema` from `pages-component/src/model/form-input-types.ts`
- Produces:
  - `validateField(schema: FieldSchema, value: unknown, required: boolean): string | null`
  - `readFieldValue(element: HTMLElement, componentType: string): unknown`
  - `setFieldError(element: HTMLElement, componentType: string, error: string | undefined): void`
  - `STANDALONE_TYPES: Set<string>` (exported const)

- [ ] **Step 1: Write failing tests for validateField in pages-component**

Create `packages/pages-component/src/model/field-validation.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { validateField } from "./field-validation.js";
import type { FieldSchema } from "./form-input-types.js";

describe("validateField", () => {
  it("returns Required for empty required field", () => {
    const schema: FieldSchema = { type: "string" };
    expect(validateField(schema, "", true)).toBe("Required");
  });

  it("returns null for empty optional field", () => {
    const schema: FieldSchema = { type: "string" };
    expect(validateField(schema, "", false)).toBeNull();
  });

  it("validates minLength", () => {
    const schema: FieldSchema = { type: "string", minLength: 3 };
    expect(validateField(schema, "ab", false)).toBe("Must be at least 3 characters");
    expect(validateField(schema, "abc", false)).toBeNull();
  });

  it("validates maximum", () => {
    const schema: FieldSchema = { type: "number", maximum: 100 };
    expect(validateField(schema, 101, false)).toBe("Must be at most 100");
    expect(validateField(schema, 100, false)).toBeNull();
  });

  it("validates pattern", () => {
    const schema: FieldSchema = { type: "string", pattern: "^[A-Z]+$" };
    expect(validateField(schema, "abc", false)).toBe("Invalid format");
    expect(validateField(schema, "ABC", false)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- src/model/field-validation.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Create field-validation.ts**

Create `packages/pages-component/src/model/field-validation.ts` — copy `validateField` from `pages-viz/src/form-inputs/schema-types.ts` (lines 54-84):

```typescript
import type { FieldSchema } from "./form-input-types.js";

export function validateField(
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): string | null {
  if (required && (value === null || value === undefined || value === "")) {
    return "Required";
  }
  if (value === null || value === undefined || value === "") return null;
  if (typeof value === "string") {
    if (schema.pattern != null) {
      const re = new RegExp(schema.pattern);
      if (!re.test(value)) return "Invalid format";
    }
    if (schema.minLength != null && value.length < schema.minLength) {
      return `Must be at least ${schema.minLength} characters`;
    }
    if (schema.maxLength != null && value.length > schema.maxLength) {
      return `Must be at most ${schema.maxLength} characters`;
    }
  }
  if (typeof value === "number") {
    if (schema.minimum != null && value < schema.minimum) {
      return `Must be at least ${schema.minimum}`;
    }
    if (schema.maximum != null && value > schema.maximum) {
      return `Must be at most ${schema.maximum}`;
    }
  }
  return null;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-component run test -- src/model/field-validation.test.ts`
Expected: PASS

- [ ] **Step 5: Write failing tests for field-access utilities**

Create `packages/pages-component/src/model/field-access.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { readFieldValue, setFieldError, STANDALONE_TYPES } from "./field-access.js";

describe("STANDALONE_TYPES", () => {
  it("contains input, select, textarea, checkbox", () => {
    expect(STANDALONE_TYPES.has("input")).toBe(true);
    expect(STANDALONE_TYPES.has("select")).toBe(true);
    expect(STANDALONE_TYPES.has("textarea")).toBe(true);
    expect(STANDALONE_TYPES.has("checkbox")).toBe(true);
    expect(STANDALONE_TYPES.size).toBe(4);
  });
});

describe("readFieldValue", () => {
  it("reads .checked for checkbox type", () => {
    const el = { checked: true } as unknown as HTMLElement;
    expect(readFieldValue(el, "checkbox")).toBe(true);
  });

  it("reads .value for standalone input types", () => {
    const el = { value: "hello" } as unknown as HTMLElement;
    expect(readFieldValue(el, "input")).toBe("hello");
  });

  it("reads .currentValue for non-standalone types", () => {
    const el = { currentValue: 42 } as unknown as HTMLElement;
    expect(readFieldValue(el, "number-input")).toBe(42);
  });

  it("falls back to .value when .currentValue absent", () => {
    const el = { value: "fallback" } as unknown as HTMLElement;
    expect(readFieldValue(el, "number-input")).toBe("fallback");
  });
});

describe("setFieldError", () => {
  it("sets .error for standalone types", () => {
    const el = {} as any;
    setFieldError(el, "input", "Required");
    expect(el.error).toBe("Required");
  });

  it("sets .errorMessage for non-standalone types with errorMessage", () => {
    const el = { errorMessage: "" } as any;
    setFieldError(el, "number-input", "Too low");
    expect(el.errorMessage).toBe("Too low");
  });

  it("clears error with undefined", () => {
    const el = { error: "old" } as any;
    setFieldError(el, "input", undefined);
    expect(el.error).toBeUndefined();
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- src/model/field-access.test.ts`
Expected: FAIL — module not found

- [ ] **Step 7: Create field-access.ts**

Create `packages/pages-component/src/model/field-access.ts`:

```typescript
export const STANDALONE_TYPES = new Set(["input", "select", "textarea", "checkbox"]);

export function readFieldValue(element: HTMLElement, componentType: string): unknown {
  if (componentType === "checkbox") return (element as any).checked;
  if (STANDALONE_TYPES.has(componentType)) return (element as any).value;
  return "currentValue" in element ? (element as any).currentValue : (element as any).value;
}

export function setFieldError(element: HTMLElement, componentType: string, error: string | undefined): void {
  if (STANDALONE_TYPES.has(componentType)) {
    (element as any).error = error;
  } else if ("errorMessage" in element) {
    (element as any).errorMessage = error;
  } else {
    (element as any).error = error;
  }
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-component run test -- src/model/field-access.test.ts`
Expected: PASS

- [ ] **Step 9: Export from pages-component model index**

Add to `packages/pages-component/src/model/index.ts`:

```typescript
export { validateField } from "./field-validation.js";
export { readFieldValue, setFieldError, STANDALONE_TYPES } from "./field-access.js";
```

- [ ] **Step 10: Update PagesSchemaForm to import from pages-component**

In `packages/pages-viz/src/form-inputs/schema-types.ts`, replace the `validateField` export with a re-export:

```typescript
export { validateField } from "@casehubio/pages-component";
```

Remove the `validateField` function body (lines 54-84). Keep `deriveSchemaFromDataSet` and `mapFieldToComponentType` — they depend on `@casehubio/pages-data` types that pages-component doesn't have.

In `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`, replace the local `STANDALONE_TYPES` const (line 21) with an import:

```typescript
import { STANDALONE_TYPES } from "@casehubio/pages-component";
```

- [ ] **Step 11: Run all tests to verify no regressions**

Run: `yarn workspace @casehubio/pages-component run test && yarn workspace @casehubio/pages-viz run test`
Expected: All tests pass

- [ ] **Step 12: Build all affected packages**

Run: `yarn workspace @casehubio/pages-component run build && yarn workspace @casehubio/pages-viz run build`
Expected: Clean builds

- [ ] **Step 13: Commit**

```bash
git add packages/pages-component/src/model/field-validation.ts packages/pages-component/src/model/field-validation.test.ts packages/pages-component/src/model/field-access.ts packages/pages-component/src/model/field-access.test.ts packages/pages-component/src/model/index.ts packages/pages-viz/src/form-inputs/schema-types.ts packages/pages-viz/src/form-inputs/PagesSchemaForm.ts
git commit -m "refactor: move validateField to pages-component, create shared field-access utilities

Refs #337"
```

### Task 2: FormScopeProps, SubmitButtonProps, SchemaFormProps.fields, ComponentTypeRegistry

**Files:**
- Create: `packages/pages-component/src/model/form-scope-types.ts`
- Create: `packages/pages-component/src/model/submit-button-types.ts`
- Modify: `packages/pages-component/src/model/form-input-types.ts`
- Modify: `packages/pages-component/src/model/type-guards.ts`
- Modify: `packages/pages-component/src/model/index.ts`

**Interfaces:**
- Consumes: `FieldSchema` from `form-input-types.ts`
- Produces:
  - `FormScopeProps { schema?: FieldSchema; validateOnBlur?: boolean; mode?: "display" | "edit" }`
  - `SubmitButtonProps { label: string; style?: "primary" | "danger" | "secondary" | "ghost" | "outline"; disabled?: boolean }`
  - `SchemaFormProps.fields?: string[]` (new optional prop)
  - `"form-scope"` and `"submit-button"` in `ComponentTypeRegistry`
  - `isFormScope()` and `isSubmitButton()` type guards

- [ ] **Step 1: Create form-scope-types.ts**

Create `packages/pages-component/src/model/form-scope-types.ts`:

```typescript
import type { FieldSchema } from "./form-input-types.js";

export interface FormScopeProps {
  readonly schema?: FieldSchema;
  readonly validateOnBlur?: boolean;
  readonly mode?: "display" | "edit";
}
```

- [ ] **Step 2: Create submit-button-types.ts**

Create `packages/pages-component/src/model/submit-button-types.ts`:

```typescript
export interface SubmitButtonProps {
  readonly label: string;
  readonly style?: "primary" | "danger" | "secondary" | "ghost" | "outline";
  readonly disabled?: boolean;
}
```

- [ ] **Step 3: Add `fields` to SchemaFormProps**

In `packages/pages-component/src/model/form-input-types.ts`, add to `SchemaFormProps`:

```typescript
readonly fields?: string[];
```

- [ ] **Step 4: Add to ComponentTypeRegistry**

In `packages/pages-component/src/model/type-guards.ts`:

1. Add imports:
```typescript
import type { FormScopeProps } from "./form-scope-types.js";
import type { SubmitButtonProps } from "./submit-button-types.js";
```

2. Add to `ComponentTypeRegistry`:
```typescript
  "form-scope": FormScopeProps;
  "submit-button": SubmitButtonProps;
```

3. Add type guards:
```typescript
export function isFormScope(c: Component): c is TypedComponent<"form-scope"> {
  return c.type === "form-scope";
}

export function isSubmitButton(c: Component): c is TypedComponent<"submit-button"> {
  return c.type === "submit-button";
}
```

- [ ] **Step 5: Export from model index**

In `packages/pages-component/src/model/index.ts`, add:

```typescript
export type { FormScopeProps } from "./form-scope-types.js";
export type { SubmitButtonProps } from "./submit-button-types.js";
```

And add `isFormScope`, `isSubmitButton` to the type-guards re-export.

- [ ] **Step 6: Build and verify**

Run: `yarn workspace @casehubio/pages-component run build && yarn workspace @casehubio/pages-component run test`
Expected: Clean build, all tests pass

- [ ] **Step 7: Commit**

```bash
git add packages/pages-component/src/model/form-scope-types.ts packages/pages-component/src/model/submit-button-types.ts packages/pages-component/src/model/form-input-types.ts packages/pages-component/src/model/type-guards.ts packages/pages-component/src/model/index.ts
git commit -m "feat: add FormScopeProps, SubmitButtonProps, SchemaFormProps.fields, registry entries

Refs #337"
```

---

## Batch 2: Components — PagesSubmitButton + PagesSchemaForm modifications

After this batch: PagesSubmitButton web component works standalone, PagesSchemaForm supports `fieldsOnly` and `fields` props with tests.

### Task 3: PagesSubmitButton web component

**Files:**
- Create: `packages/pages-ui-components/src/submit-button/pages-submit-button.ts`
- Create: `packages/pages-ui-components/src/submit-button/pages-submit-button.test.ts`
- Create: `packages/pages-ui-components/src/submit-button/index.ts`
- Modify: `packages/pages-ui-components/src/index.ts`

**Interfaces:**
- Consumes: `SubmitButtonProps` from `@casehubio/pages-component`
- Produces: `PagesSubmitButton` web component (`pages-submit-button` tag), dispatches `pages-form-submit` with `{ resolve: (result) => void }`

- [ ] **Step 1: Write failing tests**

Create `packages/pages-ui-components/src/submit-button/pages-submit-button.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import "./pages-submit-button.js";

describe("PagesSubmitButton", () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement("pages-submit-button");
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it("registers as custom element", () => {
    expect(customElements.get("pages-submit-button")).toBeDefined();
  });

  it("dispatches pages-form-submit with resolve on click", async () => {
    (el as any).label = "Submit";
    await (el as any).updateComplete;

    const handler = vi.fn();
    el.addEventListener("pages-form-submit", handler);

    const btn = el.shadowRoot?.querySelector("button");
    btn?.click();

    expect(handler).toHaveBeenCalledTimes(1);
    const detail = handler.mock.calls[0][0].detail;
    expect(typeof detail.resolve).toBe("function");
  });

  it("enters loading state on click and exits on resolve", async () => {
    (el as any).label = "Submit";
    await (el as any).updateComplete;

    let resolveRef: ((r: any) => void) | undefined;
    el.addEventListener("pages-form-submit", (e: Event) => {
      resolveRef = (e as CustomEvent).detail.resolve;
    });

    el.shadowRoot?.querySelector("button")?.click();
    await (el as any).updateComplete;

    expect(el.shadowRoot?.querySelector("button")?.getAttribute("aria-busy")).toBe("true");

    resolveRef?.({ success: true });
    await (el as any).updateComplete;

    expect(el.shadowRoot?.querySelector("button")?.getAttribute("aria-busy")).toBe("false");
  });

  it("disables click during loading", async () => {
    (el as any).label = "Submit";
    await (el as any).updateComplete;

    const handler = vi.fn();
    el.addEventListener("pages-form-submit", handler);

    el.shadowRoot?.querySelector("button")?.click();
    el.shadowRoot?.querySelector("button")?.click();

    expect(handler).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/submit-button/pages-submit-button.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement PagesSubmitButton**

Create `packages/pages-ui-components/src/submit-button/pages-submit-button.ts`:

```typescript
import { LitElement, html, css } from "lit";
import { customElement, property, state } from "lit/decorators.js";
import { classMap } from "lit/directives/class-map.js";

interface SubmitResult {
  readonly success: boolean;
  readonly error?: string;
}

@customElement("pages-submit-button")
export class PagesSubmitButton extends LitElement {
  static override styles = css`
    :host { display: inline-block; }
    button {
      display: inline-flex; align-items: center; gap: var(--pages-space-1, 4px);
      padding: var(--pages-space-1-5, 6px) var(--pages-space-4, 16px);
      border-radius: var(--pages-radius-sm, 4px);
      font-size: var(--pages-font-size-base, 14px);
      font-family: var(--pages-font-family, system-ui, sans-serif);
      font-weight: var(--pages-font-weight-medium, 500);
      cursor: pointer; border: 1px solid transparent;
      transition: background var(--pages-duration-fast, 120ms) var(--pages-ease-out);
    }
    button:disabled { cursor: not-allowed; opacity: 0.6; }
    .primary { background: var(--pages-accent-9, #5470c6); color: white; border-color: var(--pages-accent-9, #5470c6); }
    .primary:hover:not(:disabled) { background: var(--pages-accent-10, #4060b6); }
    .danger { background: var(--pages-danger-9, #dc2626); color: white; border-color: var(--pages-danger-9, #dc2626); }
    .danger:hover:not(:disabled) { background: var(--pages-danger-10, #b91c1c); }
    .secondary { background: var(--pages-neutral-8, #6c757d); color: white; }
    .secondary:hover:not(:disabled) { background: var(--pages-neutral-9, #5a6268); }
    .ghost { background: transparent; color: var(--pages-accent-9, #5470c6); }
    .ghost:hover:not(:disabled) { background: var(--pages-neutral-3, #f5f5f5); }
    .outline { background: transparent; color: var(--pages-accent-9, #5470c6); border: 1px solid var(--pages-accent-7, #99c2e6); }
    .outline:hover:not(:disabled) { background: var(--pages-accent-3, #e6f0fa); }
    @keyframes spin { to { transform: rotate(360deg); } }
    .spinner { width: 14px; height: 14px; border: 2px solid currentColor; border-top-color: transparent; border-radius: 50%; animation: spin 0.6s linear infinite; }
    .result { padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px); border-radius: var(--pages-radius-sm, 4px); font-size: var(--pages-font-size-sm, 13px); margin-top: var(--pages-space-1, 4px); }
    .result-success { background: var(--pages-success-3, #d4edda); color: var(--pages-success-11, #155724); }
    .result-error { background: var(--pages-danger-3, #f8d7da); color: var(--pages-danger-11, #721c24); }
  `;

  @property() label = "Submit";
  @property() style: "primary" | "danger" | "secondary" | "ghost" | "outline" = "primary";
  @property({ type: Boolean }) disabled = false;

  @state() private _isLoading = false;
  @state() private _resultMessage: string | null = null;
  @state() private _resultType: "success" | "error" | null = null;
  private _timeoutId: ReturnType<typeof setTimeout> | null = null;

  override render() {
    const isDisabled = this.disabled || this._isLoading;
    return html`
      <div>
        <button class=${classMap({ [this.style]: true })}
                ?disabled=${isDisabled}
                aria-busy=${String(this._isLoading)}
                aria-disabled=${isDisabled ? "true" : "false"}
                @click=${this._handleClick}>
          ${this._isLoading ? html`<span class="spinner" aria-hidden="true"></span>` : ""}
          ${this.label}
        </button>
        ${this._resultMessage ? html`
          <div class="result ${this._resultType === "success" ? "result-success" : "result-error"}">
            ${this._resultMessage}
          </div>
        ` : ""}
      </div>
    `;
  }

  private _handleClick(): void {
    if (this._isLoading || this.disabled) return;
    this._isLoading = true;
    this._resultMessage = null;
    this._resultType = null;

    this._timeoutId = setTimeout(() => {
      this._handleResult({ success: false, error: "Form submit timed out" });
    }, 5000);

    this.dispatchEvent(new CustomEvent("pages-form-submit", {
      bubbles: true, composed: true,
      detail: { resolve: (result: SubmitResult) => { this._handleResult(result); } },
    }));
  }

  private _handleResult(result: SubmitResult): void {
    if (this._timeoutId !== null) {
      clearTimeout(this._timeoutId);
      this._timeoutId = null;
    }
    this._isLoading = false;
    if (result.success) {
      this._resultMessage = "Submitted";
      this._resultType = "success";
      setTimeout(() => { this._resultMessage = null; this._resultType = null; }, 3000);
    } else if (result.error) {
      this._resultMessage = result.error;
      this._resultType = "error";
    }
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    if (this._timeoutId !== null) {
      clearTimeout(this._timeoutId);
      this._timeoutId = null;
    }
  }
}
```

Create `packages/pages-ui-components/src/submit-button/index.ts`:

```typescript
export { PagesSubmitButton } from "./pages-submit-button.js";
```

Add to `packages/pages-ui-components/src/index.ts`:

```typescript
export { PagesSubmitButton } from './submit-button/index.js';
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/submit-button/pages-submit-button.test.ts`
Expected: PASS

- [ ] **Step 5: Build**

Run: `yarn workspace @casehubio/pages-ui-components run build`
Expected: Clean build

- [ ] **Step 6: Commit**

```bash
git add packages/pages-ui-components/src/submit-button/ packages/pages-ui-components/src/index.ts
git commit -m "feat: add PagesSubmitButton web component — dispatches pages-form-submit with resolve

Refs #337"
```

### Task 4: PagesSchemaForm — fieldsOnly mode, fields prop, shared imports

**Files:**
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`
- Test: `packages/pages-viz/src/form-inputs/schema-form.test.ts`

**Interfaces:**
- Consumes: `STANDALONE_TYPES`, `readFieldValue`, `setFieldError` from `@casehubio/pages-component`
- Produces: PagesSchemaForm with `fieldsOnly` prop (skips submit/validation, dispatches `pages-field-register`), `fields` prop (include-filter whitelist)

- [ ] **Step 1: Write failing tests for fieldsOnly mode**

Add to `packages/pages-viz/src/form-inputs/schema-form.test.ts`:

```typescript
describe("PagesSchemaForm — fieldsOnly mode", () => {
  it("skips submit button when fieldsOnly is true", async () => {
    const form = document.createElement("pages-schema-form") as PagesSchemaForm;
    form.props = { forceCreate: true, fieldsOnly: true };
    form.editable = true;
    form.dataSet = emptyDataSet;
    document.body.appendChild(form);
    await form.updateComplete;

    const submitBtn = form.shadowRoot?.querySelector(".submit-btn");
    expect(submitBtn).toBeNull();
    form.remove();
  });

  it("dispatches pages-field-register for each field in fieldsOnly mode", async () => {
    const form = document.createElement("pages-schema-form") as PagesSchemaForm;
    form.props = {
      fieldsOnly: true,
      schema: { properties: { name: { type: "string" }, age: { type: "number" } } },
    };
    form.editable = true;

    const events: CustomEvent[] = [];
    form.addEventListener("pages-field-register", (e) => events.push(e as CustomEvent));

    form.dataSet = singleRowDataSet;
    document.body.appendChild(form);
    await form.updateComplete;

    expect(events.length).toBe(2);
    expect(events[0].detail.field).toBe("name");
    expect(events[1].detail.field).toBe("age");
    form.remove();
  });
});

describe("PagesSchemaForm — fields prop", () => {
  it("renders only listed fields in specified order", async () => {
    const form = document.createElement("pages-schema-form") as PagesSchemaForm;
    form.props = {
      schema: {
        properties: {
          name: { type: "string" },
          age: { type: "number" },
          email: { type: "string" },
        },
      },
      fields: ["email", "name"],
    };
    form.editable = true;
    form.dataSet = singleRowDataSet;
    document.body.appendChild(form);
    await form.updateComplete;

    const labels = Array.from(
      form.shadowRoot?.querySelectorAll("[label]") ?? []
    ).map((el) => (el as any).label);

    expect(labels).toEqual(["Email", "Name"]);
    form.remove();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-viz run test -- src/form-inputs/schema-form.test.ts`
Expected: FAIL — fieldsOnly and fields not recognized

- [ ] **Step 3: Implement fieldsOnly and fields in PagesSchemaForm**

In `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`:

1. Add `fieldsOnly` property:
```typescript
private _fieldsOnly = false;

set fieldsOnly(value: boolean) { this._fieldsOnly = value; }
get fieldsOnly(): boolean { return this._fieldsOnly; }
```

2. Modify `renderContent` to use `fields` prop for filtering (around line 108-109):
```typescript
const visibleFields = props.fields
  ? props.fields.filter((f) => f in schemaProps)
  : fieldOrder.filter((f) => !excludeSet.has(f) && f in schemaProps);
```

Replace `fields` variable usage with `visibleFields`.

3. Skip submit button when `fieldsOnly` is true (around line 187-193):
```typescript
${isCreateMode && !isDisplay && !this._fieldsOnly ? html`
  <div class="submit-bar">
    <button class="submit-btn" @click=${() => this.submit()}>Submit</button>
  </div>
` : ""}
```

4. Dispatch `pages-field-register` in `fieldsOnly` mode — after creating/updating each child element in the for loop (around line 173):
```typescript
if (this._fieldsOnly) {
  child.dispatchEvent(new CustomEvent("pages-field-register", {
    bubbles: true, composed: true,
    detail: { field, element: child, componentType },
  }));
}
```

5. Skip internal validation handler when `fieldsOnly`:
```typescript
@pages-field-change=${props.validateOnBlur && !this._fieldsOnly ? this._handleFieldChange : undefined}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-viz run test -- src/form-inputs/schema-form.test.ts`
Expected: PASS (including existing tests)

- [ ] **Step 5: Build**

Run: `yarn workspace @casehubio/pages-viz run build`
Expected: Clean build

- [ ] **Step 6: Commit**

```bash
git add packages/pages-viz/src/form-inputs/PagesSchemaForm.ts packages/pages-viz/src/form-inputs/schema-form.test.ts
git commit -m "feat: PagesSchemaForm fieldsOnly mode + fields include-filter

In fieldsOnly mode, skips submit button and internal validation,
dispatches pages-field-register for each generated field.

Refs #337"
```

---

## Batch 3: Runtime wiring — FormScopeState, activation handlers, resolve chain

After this batch: full form-scope runtime works — field registration, validation, submit, resolve chain through site.ts.

### Task 5: FormScopeState/Registry + form-scope activation handler

**Files:**
- Create: `packages/pages-runtime/src/form-scope.ts`
- Create: `packages/pages-runtime/src/form-scope.test.ts`
- Modify: `packages/pages-runtime/src/activation.ts`

**Interfaces:**
- Consumes: `validateField` from `@casehubio/pages-component`, `readFieldValue`, `setFieldError`, `STANDALONE_TYPES` from `@casehubio/pages-component`
- Produces: `FormScopeState`, `FormScopeRegistry` (WeakMap), activation handler for `"form-scope"` type

- [ ] **Step 1: Write failing tests for FormScopeState**

Create `packages/pages-runtime/src/form-scope.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { FormScopeState } from "./form-scope.js";

describe("FormScopeState", () => {
  it("registers a field", () => {
    const state = new FormScopeState(undefined, false);
    const el = document.createElement("input");
    state.registerField("name", el, "input");
    expect(state.hasField("name")).toBe(true);
  });

  it("collects values from registered fields", () => {
    const state = new FormScopeState(undefined, false);
    const el = { value: "Alice" } as unknown as HTMLElement;
    state.registerField("name", el, "input");
    const values = state.collectValues();
    expect(values).toEqual({ name: "Alice" });
  });

  it("prunes disconnected fields on collectValues", () => {
    const state = new FormScopeState(undefined, false);
    const el = document.createElement("input");
    state.registerField("name", el, "input");
    expect(state.hasField("name")).toBe(true);

    const values = state.collectValues();
    expect(values).toEqual({});
    expect(state.hasField("name")).toBe(false);
  });

  it("validates all fields against schema", () => {
    const schema = {
      properties: { name: { type: "string" as const, minLength: 1 } },
      required: ["name"],
    };
    const state = new FormScopeState(schema, false);
    const el = document.createElement("div");
    document.body.appendChild(el);
    (el as any).value = "";
    state.registerField("name", el, "input");
    const errors = state.validateAll();
    expect(errors).toEqual({ name: "Required" });
    el.remove();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- src/form-scope.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement FormScopeState**

Create `packages/pages-runtime/src/form-scope.ts`:

```typescript
import { validateField, readFieldValue, setFieldError } from "@casehubio/pages-component";
import type { FieldSchema } from "@casehubio/pages-component";

interface RegisteredField {
  element: HTMLElement;
  field: string;
  componentType: string;
}

export class FormScopeState {
  readonly fields = new Map<string, RegisteredField>();

  constructor(
    readonly schema: FieldSchema | undefined,
    readonly validateOnBlur: boolean,
  ) {}

  registerField(field: string, element: HTMLElement, componentType: string): void {
    this.fields.set(field, { element, field, componentType });
  }

  hasField(field: string): boolean {
    return this.fields.has(field);
  }

  private pruneDisconnected(): void {
    for (const [key, entry] of this.fields) {
      if (!entry.element.isConnected) {
        this.fields.delete(key);
      }
    }
  }

  collectValues(): Record<string, unknown> {
    this.pruneDisconnected();
    const values: Record<string, unknown> = {};
    for (const [field, entry] of this.fields) {
      values[field] = readFieldValue(entry.element, entry.componentType);
    }
    return values;
  }

  validateAll(): Record<string, string> {
    this.pruneDisconnected();
    if (!this.schema?.properties) return {};
    const requiredSet = new Set(this.schema.required ?? []);
    const errors: Record<string, string> = {};

    for (const [field, entry] of this.fields) {
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
    return errors;
  }

  validateField(field: string, value: unknown): void {
    if (!this.schema?.properties) return;
    const fieldSchema = this.schema.properties[field];
    if (!fieldSchema) return;
    const entry = this.fields.get(field);
    if (!entry) return;
    const requiredSet = new Set(this.schema.required ?? []);
    const error = validateField(fieldSchema, value, requiredSet.has(field));
    setFieldError(entry.element, entry.componentType, error ?? undefined);
  }
}

export const FormScopeRegistry = new WeakMap<HTMLElement, FormScopeState>();
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- src/form-scope.test.ts`
Expected: PASS

- [ ] **Step 5: Add form-scope activation handler to activation.ts**

In `packages/pages-runtime/src/activation.ts`, add the handler block for `"form-scope"` after the `"action-button"` handler (around line 371):

```typescript
if (component.type === "form-scope") {
  const nestingParent = el.parentElement?.closest('[data-component-type="form-scope"]');
  if (nestingParent) {
    console.error("[pages] Nested form-scope is not supported. Inner formScope will not function.");
    return;
  }

  const fsProps = component.props as FormScopeProps | undefined;
  const state = new FormScopeState(fsProps?.schema, fsProps?.validateOnBlur ?? false);
  FormScopeRegistry.set(el, state);

  const isDisplay = fsProps?.mode === "display";
  el.setAttribute("role", isDisplay ? "group" : "form");

  registry.set(componentId, {
    element: el, component, pagePath,
    hasExplicitId: component.id !== undefined,
  });

  el.addEventListener("pages-field-register", ((e: Event) => {
    const detail = (e as CustomEvent).detail as {
      field: string; element: HTMLElement; componentType: string;
    };
    state.registerField(detail.field, detail.element, detail.componentType);
  }));

  if (state.validateOnBlur) {
    el.addEventListener("pages-field-change", ((e: Event) => {
      const detail = (e as CustomEvent).detail as {
        field: string; value: unknown; committed: boolean;
      };
      if (!detail.committed) return;
      state.validateField(detail.field, detail.value);
    }));
  }

  if (!isDisplay) {
    el.addEventListener("pages-form-submit", ((e: Event) => {
      e.stopPropagation();
      const submitDetail = (e as CustomEvent).detail as {
        resolve?: (result: { success: boolean; error?: string }) => void;
      };

      const errors = state.validateAll();
      if (Object.keys(errors).length > 0) {
        submitDetail.resolve?.({ success: false, error: "Validation failed" });
        return;
      }

      const values = state.collectValues();
      el.dispatchEvent(new CustomEvent("pages-record-create", {
        bubbles: true, composed: true,
        detail: { record: values, resolve: submitDetail.resolve },
      }));
    }));
  }

  if (component.visibleWhen && contextManager) {
    registerVisibleWhenConsumer(el, null, component.visibleWhen, contextManager);
  }
  return;
}
```

Add necessary imports at the top of activation.ts:

```typescript
import { FormScopeState, FormScopeRegistry } from "./form-scope.js";
import type { FormScopeProps } from "@casehubio/pages-component";
```

- [ ] **Step 6: Build**

Run: `yarn workspace @casehubio/pages-runtime run build`
Expected: Clean build

- [ ] **Step 7: Commit**

```bash
git add packages/pages-runtime/src/form-scope.ts packages/pages-runtime/src/form-scope.test.ts packages/pages-runtime/src/activation.ts
git commit -m "feat: FormScopeState + form-scope activation handler

Field registration, validation, submit, nesting guard, ComponentRegistry
entry for pages-record-create resolution.

Refs #337"
```

### Task 6: submit-button activation + field-register dispatch + site.ts resolve

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/site.ts`

**Interfaces:**
- Consumes: `PagesSubmitButton` from `@casehubio/pages-ui-components`, `RecordCreateDetail` from site.ts
- Produces: submit-button activation handler, `pages-field-register` dispatch from standalone inputs, `RecordCreateDetail.resolve` on all exit paths

- [ ] **Step 1: Add submit-button activation handler**

In `packages/pages-runtime/src/activation.ts`, add after the form-scope handler:

```typescript
if (component.type === "submit-button" && component.props) {
  const btn = document.createElement("pages-submit-button");
  (btn as any).props = component.props;
  el.appendChild(btn);
  if (component.visibleWhen && contextManager) {
    registerVisibleWhenConsumer(el, null, component.visibleWhen, contextManager);
  }
  return;
}
```

Add the `pages-submit-button` import at the top:
```typescript
import "@casehubio/pages-ui-components/submit-button";
```

- [ ] **Step 2: Add pages-field-register dispatch to standalone input handler**

In `packages/pages-runtime/src/activation.ts`, in the `STANDALONE_FORM_TYPES` handler block, after the `el.appendChild(formEl)` line (around line 202), add:

```typescript
if (field) {
  formEl.dispatchEvent(new CustomEvent("pages-field-register", {
    bubbles: true, composed: true,
    detail: { field, element: formEl, componentType: component.type },
  }));
}
```

- [ ] **Step 3: Add resolve to RecordCreateDetail in site.ts**

In `packages/pages-runtime/src/site.ts`, update the `RecordCreateDetail` interface (line 112):

```typescript
interface RecordCreateDetail {
  readonly record?: Record<string, unknown>;
  readonly resolve?: (result: { success: boolean; error?: string }) => void;
}
```

- [ ] **Step 4: Add resolve calls to all exit paths in pages-record-create handler**

In site.ts, update the `pages-record-create` handler (starting line 771). Extract `resolve` from detail:

```typescript
const { record: eventRecord, resolve } = (e as CustomEvent<RecordCreateDetail>).detail;
```

Add `resolve?.({ success: false, error: "..." })` before each early return, and `resolve?.({ success: true })` / `resolve?.({ success: false, error: ... })` for success/failure paths. Add `resolve?.({ success: false, error: msg })` in the catch block.

- [ ] **Step 5: Build and run all tests**

Run: `yarn workspace @casehubio/pages-runtime run build && yarn workspace @casehubio/pages-runtime run test`
Expected: Clean build, all tests pass

- [ ] **Step 6: Commit**

```bash
git add packages/pages-runtime/src/activation.ts packages/pages-runtime/src/site.ts
git commit -m "feat: submit-button activation, field-register dispatch, record-create resolve chain

site.ts calls resolve on all 8 exit paths in pages-record-create handler.
Standalone form inputs dispatch pages-field-register when field prop present.

Refs #337"
```

---

## Batch 4: DSL builders + YAML

After this batch: `formScope()`, `schemaFields()`, `submitButton()` builders work from TypeScript DSL. YAML `form-scope:` and `submit-button:` container types parse correctly.

### Task 7: DSL builders, re-exports, and YAML support

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts`
- Modify: `packages/pages-ui/src/dsl/index.ts`
- Modify: `packages/pages-ui/src/parser/component-desugar.ts`
- Test: `packages/pages-ui/src/parser/desugar-new-types.test.ts` (or existing test file)

**Interfaces:**
- Consumes: `FormScopeProps`, `SubmitButtonProps`, `SchemaFormProps` from `@casehubio/pages-component`, `TypedComponent` from `@casehubio/pages-component`
- Produces: `formScope()`, `schemaFields()`, `submitButton()` DSL builders, YAML parsing for `form-scope:` and `submit-button:`

- [ ] **Step 1: Add DSL builders**

In `packages/pages-ui/src/dsl/builders.ts`:

1. Add imports:
```typescript
import type { FormScopeProps, SubmitButtonProps } from "@casehubio/pages-component";
```

2. Add `formScope` builder (after `schemaForm`):
```typescript
export function formScope(
  props: FormScopeProps,
  ...children: Component[]
): TypedComponent<"form-scope"> {
  return freeze({
    type: "form-scope" as const,
    props: { ...props },
    slots: freeze({ default: Object.freeze(children) }),
  });
}
```

3. Add `schemaFields` builder:
```typescript
export type SchemaFieldsProps = Omit<SchemaFormProps, "validateOnBlur">;

export function schemaFields(
  props: SchemaFieldsProps
): TypedComponent<"schema-form"> {
  return freeze({
    type: "schema-form" as const,
    props: { ...props, fieldsOnly: true },
  });
}
```

4. Add `submitButton` builder:
```typescript
export function submitButton(
  props: SubmitButtonProps
): TypedComponent<"submit-button"> {
  return freeze({
    type: "submit-button" as const,
    props: freeze({ ...props }),
  });
}
```

- [ ] **Step 2: Add re-exports to DSL index**

In `packages/pages-ui/src/dsl/index.ts`:

```typescript
export type { FormScopeProps, SubmitButtonProps } from "@casehubio/pages-component";
```

- [ ] **Step 3: Add YAML support in component-desugar.ts**

Check the existing desugar patterns for container types (like `tabs:`, `sidebar:`) and add `form-scope:` and `submit-button:` following the same pattern. The `form-scope:` handler should convert `components:` key to `slots: { default: [...] }`. The `submit-button:` handler is a leaf component — props directly.

- [ ] **Step 4: Write tests for YAML parsing**

Add tests to verify `form-scope:` YAML parses to the correct component tree with `type: "form-scope"`, `props`, and `slots.default`.

- [ ] **Step 5: Run all tests**

Run: `yarn workspace @casehubio/pages-ui run test && yarn workspace @casehubio/pages-ui run build`
Expected: All tests pass, clean build

- [ ] **Step 6: Full integration build**

Run: `yarn build:packages`
Expected: All packages build successfully

- [ ] **Step 7: Commit**

```bash
git add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/index.ts packages/pages-ui/src/parser/component-desugar.ts packages/pages-ui/src/parser/desugar-new-types.test.ts
git commit -m "feat: formScope/schemaFields/submitButton DSL builders + YAML support

Closes #337"
```

---

## References

- [2026-08-26-form-scope-composable-layout-design.md] — design spec this plan implements
- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — schema-form internals
- `packages/pages-viz/src/form-inputs/schema-types.ts:54-84` — validateField source
- `packages/pages-runtime/src/activation.ts:48-82` — STANDALONE_FORM_TYPES, DATA_COMPONENT_TYPES
- `packages/pages-runtime/src/activation.ts:362-371` — action-button handler pattern
- `packages/pages-runtime/src/site.ts:112-114` — RecordCreateDetail
- `packages/pages-runtime/src/site.ts:771-805` — pages-record-create handler
- `packages/pages-component/src/model/type-guards.ts:61-124` — ComponentTypeRegistry
- `packages/pages-component/src/model/form-input-types.ts:71` — SchemaFormProps
- `packages/pages-ui/src/dsl/builders.ts:499-501` — existing schemaForm builder
- `packages/pages-ui-components/src/button/pages-button.ts` — PagesButton pattern
- GitHub #337 — focal issue
- GitHub #336 — actionButton precedent (activation handler pattern)
