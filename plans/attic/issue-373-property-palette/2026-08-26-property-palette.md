# Property Palette Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #373 — feat(pages-property-palette): schema-driven property panel with rich editors
**Issue group:** #373

**Goal:** Build a schema-driven property inspector panel (`<pages-property-palette>`) that renders JSON Schema as editable form fields, with grouping, validation, and rich editors — then migrate blocks-ui's existing property form to use it.

**Architecture:** New `@casehubio/pages-property-palette` package containing a single Lit web component. Six new form controls added to `@casehubio/pages-ui-components`. Shared `FieldSchema` type and `validateField()` utility consolidated in `pages-ui-components`. blocks-ui `diagram-properties.ts` becomes a thin wrapper.

**Tech Stack:** TypeScript 5, Lit 3, Vitest, jsdom, `@casehubio/pages-ui-tokens` design tokens

## Global Constraints

- All web components use Lit with `pages-` prefix (PP-20260705-c7687d)
- CSS uses `--pages-` design tokens, OKLCH 12-step scales (PP-20260705-2ae91d)
- Sub-path exports for side-effect isolation in `package.json`
- Guard custom element registration: `if (!customElements.get('pages-<name>'))` pattern
- Tests use Vitest with `environment: 'jsdom'`, `experimentalDecorators: true`, `useDefineForClassFields: false`
- All form controls follow the existing contract: `value` (or `checked` for PagesCheckbox), `label`, `disabled`, `readonly`, `error` properties + `change` event
- **Tag collision guard:** pages-viz has `pages-number-input` and `pages-date-picker` registered as PagesFormInput subclasses. Task 3 must remove these after adding standalone replacements to pages-ui-components, and update PagesSchemaForm's `STANDALONE_TYPES` set accordingly.
- **Schema extensions:** `x-placeholder` and `x-help` from the issue spec must be wired through the palette to form controls. `x-placeholder` maps to the `placeholder` property on text-like controls. `x-help` renders as a tooltip icon (title attribute) next to the field label.

---

## Batch 1: Foundation — Canonical types and shared validation

### Task 1: Extract FieldSchema and validateField into pages-ui-components

**Files:**
- Modify: `packages/pages-ui-components/src/types.ts` — add canonical FieldSchema
- Create: `packages/pages-ui-components/src/validation/index.ts`
- Create: `packages/pages-ui-components/src/validation/validate-field.ts`
- Create: `packages/pages-ui-components/src/validation/validate-field.test.ts`
- Modify: `packages/pages-ui-components/package.json` — add `./validation` and `./types` exports
- Modify: `packages/pages-viz/src/form-inputs/schema-types.ts` — re-export from pages-ui-components

**Interfaces:**
- Produces: `FieldSchema` type at `@casehubio/pages-ui-components/types`
- Produces: `validateField(schema: FieldSchema, value: unknown, required: boolean): string | null` at `@casehubio/pages-ui-components/validation`

- [ ] **Step 1: Write failing tests for validateField**

Create `packages/pages-ui-components/src/validation/validate-field.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { validateField } from './validate-field.js';
import type { FieldSchema } from '../types.js';

describe('validateField', () => {
  it('returns "Required" for empty required string', () => {
    expect(validateField({ type: 'string' }, '', true)).toBe('Required');
  });

  it('returns null for non-required empty value', () => {
    expect(validateField({ type: 'string' }, '', false)).toBeNull();
  });

  it('validates minimum on number', () => {
    expect(validateField({ type: 'number', minimum: 5 }, 3, false)).toBe('Must be at least 5');
  });

  it('validates maximum on number', () => {
    expect(validateField({ type: 'number', maximum: 10 }, 15, false)).toBe('Must be at most 10');
  });

  it('validates exclusiveMinimum', () => {
    expect(validateField({ type: 'number', exclusiveMinimum: 5 }, 5, false)).toBe('Must be greater than 5');
  });

  it('validates exclusiveMaximum', () => {
    expect(validateField({ type: 'number', exclusiveMaximum: 10 }, 10, false)).toBe('Must be less than 10');
  });

  it('validates multipleOf', () => {
    expect(validateField({ type: 'number', multipleOf: 3 }, 7, false)).toBe('Must be a multiple of 3');
  });

  it('validates minLength on string', () => {
    expect(validateField({ type: 'string', minLength: 3 }, 'ab', false)).toBe('Must be at least 3 characters');
  });

  it('validates maxLength on string', () => {
    expect(validateField({ type: 'string', maxLength: 5 }, 'toolong', false)).toBe('Must be at most 5 characters');
  });

  it('validates pattern', () => {
    expect(validateField({ type: 'string', pattern: '^[a-z]+$' }, 'ABC', false)).toBe('Invalid format');
  });

  it('validates enum membership', () => {
    expect(validateField({ type: 'string', enum: ['a', 'b'] }, 'c', false)).toBe('Must be one of: a, b');
  });

  it('passes enum membership', () => {
    expect(validateField({ type: 'string', enum: ['a', 'b'] }, 'a', false)).toBeNull();
  });

  it('validates minItems on array', () => {
    expect(validateField({ type: 'array', minItems: 2 }, ['one'], false)).toBe('Must have at least 2 items');
  });

  it('validates maxItems on array', () => {
    expect(validateField({ type: 'array', maxItems: 2 }, ['a', 'b', 'c'], false)).toBe('Must have at most 2 items');
  });

  it('returns null for valid value', () => {
    expect(validateField({ type: 'string', minLength: 1, maxLength: 10 }, 'hello', false)).toBeNull();
  });

  it('returns null for undefined non-required value', () => {
    expect(validateField({ type: 'string' }, undefined, false)).toBeNull();
  });

  it('returns "Required" for undefined required value', () => {
    expect(validateField({ type: 'string' }, undefined, true)).toBe('Required');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/validation/validate-field.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Add canonical FieldSchema to types.ts**

Use `ide_edit_member` to add the FieldSchema interface to `packages/pages-ui-components/src/types.ts`:

```typescript
export interface SelectOption {
  readonly value: string;
  readonly label: string;
}

export interface FieldSchema {
  readonly type?: string | readonly string[];
  readonly format?: string;
  readonly title?: string;
  readonly description?: string;
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
  /** @deprecated Use x-placeholder instead. Kept for backward compat with PagesSchemaForm. */
  readonly placeholder?: string;
  readonly properties?: Readonly<Record<string, FieldSchema>>;
  readonly required?: readonly string[];
  readonly items?: FieldSchema;
  readonly oneOf?: readonly FieldSchema[];
  readonly [key: `x-${string}`]: unknown;
}
```

- [ ] **Step 4: Implement validateField**

Create `packages/pages-ui-components/src/validation/validate-field.ts`:

```typescript
import type { FieldSchema } from '../types.js';

export function validateField(
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): string | null {
  const isEmpty =
    value === null || value === undefined || value === '';

  if (required && isEmpty) return 'Required';
  if (isEmpty) return null;

  if (typeof value === 'string') {
    if (schema.minLength != null && value.length < schema.minLength) {
      return `Must be at least ${schema.minLength} characters`;
    }
    if (schema.maxLength != null && value.length > schema.maxLength) {
      return `Must be at most ${schema.maxLength} characters`;
    }
    if (schema.pattern != null) {
      const re = new RegExp(schema.pattern);
      if (!re.test(value)) return 'Invalid format';
    }
    if (schema.enum != null && !schema.enum.includes(value)) {
      return `Must be one of: ${schema.enum.join(', ')}`;
    }
  }

  if (typeof value === 'number') {
    if (schema.minimum != null && value < schema.minimum) {
      return `Must be at least ${schema.minimum}`;
    }
    if (schema.maximum != null && value > schema.maximum) {
      return `Must be at most ${schema.maximum}`;
    }
    if (schema.exclusiveMinimum != null && value <= schema.exclusiveMinimum) {
      return `Must be greater than ${schema.exclusiveMinimum}`;
    }
    if (schema.exclusiveMaximum != null && value >= schema.exclusiveMaximum) {
      return `Must be less than ${schema.exclusiveMaximum}`;
    }
    if (schema.multipleOf != null && value % schema.multipleOf !== 0) {
      return `Must be a multiple of ${schema.multipleOf}`;
    }
  }

  if (Array.isArray(value)) {
    if (schema.minItems != null && value.length < schema.minItems) {
      return `Must have at least ${schema.minItems} items`;
    }
    if (schema.maxItems != null && value.length > schema.maxItems) {
      return `Must have at most ${schema.maxItems} items`;
    }
  }

  return null;
}
```

Create `packages/pages-ui-components/src/validation/index.ts`:

```typescript
export { validateField } from './validate-field.js';
```

- [ ] **Step 5: Update package.json with new exports**

Verify `./types` export already exists in `packages/pages-ui-components/package.json` (it should — check first). Add `./validation` export:

```json
"./validation": {
  "types": "./dist/validation/index.d.ts",
  "default": "./dist/validation/index.js"
}
```

No sideEffects entry needed — validation is side-effect-free.

- [ ] **Step 6: Update barrel export**

Add to `packages/pages-ui-components/src/index.ts`:

```typescript
export type { FieldSchema } from './types.js';
export { validateField } from './validation/index.js';
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/validation/validate-field.test.ts`
Expected: all 17 tests PASS

- [ ] **Step 8: Update pages-viz to re-export from shared utility**

Modify `packages/pages-viz/src/form-inputs/schema-types.ts`:
- Replace the local `FieldSchema` interface with a re-export: `export type { FieldSchema } from '@casehubio/pages-ui-components/types';`
- Replace the local `validateField` function with a re-export: `export { validateField } from '@casehubio/pages-ui-components/validation';`
- Keep `SchemaFormProps`, `deriveSchemaFromDataSet`, `mapFieldToComponentType` unchanged (they're pages-viz-specific)

Add `@casehubio/pages-ui-components` to `packages/pages-viz/package.json` dependencies (it's likely already there — verify first).

- [ ] **Step 9: Run pages-viz tests to verify nothing broke**

Run: `yarn workspace @casehubio/pages-viz run test`
Expected: all existing tests PASS

- [ ] **Step 10: Run full typecheck**

Run: `yarn typecheck`
Expected: no new errors

- [ ] **Step 11: Commit**

```bash
git add packages/pages-ui-components/src/types.ts packages/pages-ui-components/src/validation/ packages/pages-ui-components/src/index.ts packages/pages-ui-components/package.json packages/pages-viz/src/form-inputs/schema-types.ts
git commit -m "feat(#373): extract canonical FieldSchema and validateField to pages-ui-components

Consolidates three duplicate validation implementations. FieldSchema is
the canonical JSON Schema field type with x-* extension support.

Refs #373"
```

### Task 2: Add readonly support to PagesSelect and PagesCheckbox

**Files:**
- Modify: `packages/pages-ui-components/src/select/pages-select.ts`
- Modify: `packages/pages-ui-components/src/select/pages-select.test.ts`
- Modify: `packages/pages-ui-components/src/checkbox/pages-checkbox.ts`
- Modify: `packages/pages-ui-components/src/checkbox/pages-checkbox.test.ts`

**Interfaces:**
- Produces: `PagesSelect.readonly: boolean` property — renders selected value as static text
- Produces: `PagesCheckbox.readonly: boolean` property — renders with `pointer-events: none`

- [ ] **Step 1: Write failing tests for PagesSelect readonly**

Add to `packages/pages-ui-components/src/select/pages-select.test.ts`:

```typescript
it('renders static text when readonly', async () => {
  (el as any).readonly = true;
  (el as any).value = 'foo';
  (el as any).options = [{ value: 'foo', label: 'Foo' }];
  await (el as any).updateComplete;
  const select = el.shadowRoot!.querySelector('select');
  expect(select).toBeNull();
  const text = el.shadowRoot!.querySelector('.readonly-value');
  expect(text).not.toBeNull();
  expect(text!.textContent!.trim()).toBe('Foo');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/select/pages-select.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement readonly on PagesSelect**

Add `@property({ type: Boolean }) readonly = false;` to PagesSelect.

Update `render()` to show static text when readonly — find the matching option's label and display it instead of the `<select>` element. Add CSS for `.readonly-value`:

```css
.readonly-value {
  padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
  font-size: var(--pages-font-size-base, 14px);
  color: var(--pages-neutral-11, #374151);
  background: var(--pages-neutral-3, #f5f5f5);
  border: 1px solid var(--pages-neutral-4, #e5e7eb);
  border-radius: var(--pages-radius-sm, 4px);
}
```

In render, conditionally show the static text:

```typescript
${this.readonly
  ? html`<span class="readonly-value" tabindex="0" aria-label=${ifDefined(this.label)}>
      ${this.options.find(o => o.value === this.value)?.label ?? this.value}
    </span>`
  : html`<select ...>...</select>`
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/select/pages-select.test.ts`
Expected: PASS

- [ ] **Step 5: Write failing tests for PagesCheckbox readonly**

Add to `packages/pages-ui-components/src/checkbox/pages-checkbox.test.ts`:

```typescript
it('preserves checked state when readonly', async () => {
  (el as any).readonly = true;
  (el as any).checked = true;
  await (el as any).updateComplete;
  const input = el.shadowRoot!.querySelector('input[type="checkbox"]')! as HTMLInputElement;
  expect(input.checked).toBe(true);
});

it('adds readonly class to field container', async () => {
  (el as any).readonly = true;
  await (el as any).updateComplete;
  const field = el.shadowRoot!.querySelector('.field')!;
  expect(field.classList.contains('readonly')).toBe(true);
});
```

- [ ] **Step 6: Implement readonly on PagesCheckbox**

Add `@property({ type: Boolean }) readonly = false;` to PagesCheckbox.

Add CSS:
```css
:host([readonly]) .field { pointer-events: none; }
```

Reflect the attribute in render: add `?readonly` to `:host`.

Alternatively, use a class-based approach: add `${this.readonly ? 'readonly' : ''}` class to `.field` and style `.field.readonly { pointer-events: none; }`.

- [ ] **Step 7: Run all pages-ui-components tests**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add packages/pages-ui-components/src/select/ packages/pages-ui-components/src/checkbox/
git commit -m "feat(#373): add readonly support to PagesSelect and PagesCheckbox

Select renders static text when readonly. Checkbox uses pointer-events:
none. Both remain focusable/tabbable (distinct from disabled).

Refs #373"
```

---

## Batch 2: New form controls

### Task 3: pages-number-input, pages-date-input, pages-datetime-input

**Files:**
- Create: `packages/pages-ui-components/src/number-input/index.ts`
- Create: `packages/pages-ui-components/src/number-input/pages-number-input.ts`
- Create: `packages/pages-ui-components/src/number-input/pages-number-input.test.ts`
- Create: `packages/pages-ui-components/src/date-input/index.ts`
- Create: `packages/pages-ui-components/src/date-input/pages-date-input.ts`
- Create: `packages/pages-ui-components/src/date-input/pages-date-input.test.ts`
- Create: `packages/pages-ui-components/src/datetime-input/index.ts`
- Create: `packages/pages-ui-components/src/datetime-input/pages-datetime-input.ts`
- Create: `packages/pages-ui-components/src/datetime-input/pages-datetime-input.test.ts`
- Modify: `packages/pages-ui-components/package.json` — add sub-path exports
- Modify: `packages/pages-ui-components/src/index.ts` — add barrel exports

**Interfaces:**
- Produces: `<pages-number-input>` — `value: number | null`, `min`, `max`, `step`, `label`, `placeholder`, `required`, `readonly`, `disabled`, `error`
- Produces: `<pages-date-input>` — `value: string` (YYYY-MM-DD), `min`, `max`, `label`, `required`, `readonly`, `disabled`, `error`
- Produces: `<pages-datetime-input>` — `value: string` (ISO 8601), `min`, `max`, `label`, `required`, `readonly`, `disabled`, `error`

- [ ] **Step 1: Write failing tests for pages-number-input**

Create `packages/pages-ui-components/src/number-input/pages-number-input.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesNumberInput', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-number-input');
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-number-input')).toBeDefined();
  });

  it('renders a number input in shadow DOM', async () => {
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input');
    expect(input).not.toBeNull();
    expect(input!.type).toBe('number');
  });

  it('sets min/max/step attributes', async () => {
    (el as any).min = 0;
    (el as any).max = 100;
    (el as any).step = 5;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.min).toBe('0');
    expect(input.max).toBe('100');
    expect(input.step).toBe('5');
  });

  it('reflects value property', async () => {
    (el as any).value = 42;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.value).toBe('42');
  });

  it('renders null value as empty', async () => {
    (el as any).value = null;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.value).toBe('');
  });

  it('shows error message', async () => {
    (el as any).error = 'Too low';
    await (el as any).updateComplete;
    const err = el.shadowRoot!.querySelector('[role="alert"]');
    expect(err).not.toBeNull();
    expect(err!.textContent).toBe('Too low');
  });

  it('sets readonly attribute on input', async () => {
    (el as any).readonly = true;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.readOnly).toBe(true);
  });
});
```

- [ ] **Step 2: Implement pages-number-input**

Create `packages/pages-ui-components/src/number-input/pages-number-input.ts` following the PagesInput pattern. Uses `<input type="number">` with `min`, `max`, `step` attributes. Value is `number | null`. Fires `change` event on blur, emitting parsed number or `null` for empty.

Create `packages/pages-ui-components/src/number-input/index.ts`:
```typescript
export { PagesNumberInput } from './pages-number-input.js';
```

- [ ] **Step 3: Write tests and implement pages-date-input**

Same pattern as PagesInput but with `<input type="date">`. Value is ISO 8601 date string. Test: renders date input, reflects value, shows error, sets readonly.

Create `packages/pages-ui-components/src/date-input/pages-date-input.ts` and test file.

- [ ] **Step 4: Write tests and implement pages-datetime-input**

Same pattern but with `<input type="datetime-local">`. Value is ISO 8601 datetime string.

Create `packages/pages-ui-components/src/datetime-input/pages-datetime-input.ts` and test file.

- [ ] **Step 5: Add sub-path exports to package.json**

Add to `packages/pages-ui-components/package.json` exports:

```json
"./number-input": { "types": "./dist/number-input/index.d.ts", "default": "./dist/number-input/index.js" },
"./date-input": { "types": "./dist/date-input/index.d.ts", "default": "./dist/date-input/index.js" },
"./datetime-input": { "types": "./dist/datetime-input/index.d.ts", "default": "./dist/datetime-input/index.js" }
```

Add corresponding sideEffects entries.

Update barrel `index.ts`:
```typescript
export { PagesNumberInput } from './number-input/index.js';
export { PagesDateInput } from './date-input/index.js';
export { PagesDatetimeInput } from './datetime-input/index.js';
```

- [ ] **Step 6: Resolve tag collision — update PagesSchemaForm and remove pages-viz duplicates**

pages-viz has `PagesNumberInput` (`pages-number-input`) and `PagesDatePicker` (`pages-date-picker`) as PagesFormInput subclasses. The new standalone controls use the same tag names. To avoid collision:

1. In `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`:
   - Add `"number-input"`, `"date-input"` to the `STANDALONE_TYPES` set (replacing `"date-picker"` with `"date-input"` in `mapFieldToComponentType` results)
   - Replace `import "./PagesNumberInput.js"` with `import "@casehubio/pages-ui-components/number-input"`
   - Replace `import "./PagesDatePicker.js"` with `import "@casehubio/pages-ui-components/date-input"` and `import "@casehubio/pages-ui-components/datetime-input"`
2. In `packages/pages-viz/src/form-inputs/schema-types.ts`:
   - Update `mapFieldToComponentType()`: change `"number-input"` return to `"number-input"` (same tag), `"date-picker"` return to `"date-input"` for `format: "date"`, add `"datetime-input"` for `format: "datetime-local"`
3. Remove (use `ide_refactor_safe_delete`):
   - `packages/pages-viz/src/form-inputs/PagesNumberInput.ts`
   - `packages/pages-viz/src/form-inputs/PagesDatePicker.ts`

- [ ] **Step 7: Run all tests (both packages)**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Run: `yarn workspace @casehubio/pages-viz run test`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add packages/pages-ui-components/src/number-input/ packages/pages-ui-components/src/date-input/ packages/pages-ui-components/src/datetime-input/ packages/pages-ui-components/src/index.ts packages/pages-ui-components/package.json packages/pages-viz/src/form-inputs/
git commit -m "feat(#373): add pages-number-input, pages-date-input, pages-datetime-input

Standalone form controls wrapping native HTML5 inputs. Replaces
pages-viz PagesNumberInput and PagesDatePicker (PagesFormInput subclasses)
with standalone versions. PagesSchemaForm updated to use standalone controls.

Refs #373"
```

### Task 4: pages-color-swatch and pages-slider

**Files:**
- Create: `packages/pages-ui-components/src/color-swatch/index.ts`
- Create: `packages/pages-ui-components/src/color-swatch/pages-color-swatch.ts`
- Create: `packages/pages-ui-components/src/color-swatch/pages-color-swatch.test.ts`
- Create: `packages/pages-ui-components/src/slider/index.ts`
- Create: `packages/pages-ui-components/src/slider/pages-slider.ts`
- Create: `packages/pages-ui-components/src/slider/pages-slider.test.ts`
- Modify: `packages/pages-ui-components/package.json` — add sub-path exports
- Modify: `packages/pages-ui-components/src/index.ts` — add barrel exports

**Interfaces:**
- Produces: `<pages-color-swatch>` — `value: string` (hex), `label`, `readonly`, `disabled`, `error`
- Produces: `<pages-slider>` — `value: number`, `min`, `max`, `step`, `label`, `readonly`, `disabled`, `error`

- [ ] **Step 1: Write failing tests for pages-color-swatch**

Create `packages/pages-ui-components/src/color-swatch/pages-color-swatch.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesColorSwatch', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-color-swatch');
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-color-swatch')).toBeDefined();
  });

  it('renders a color input and hex text input', async () => {
    await (el as any).updateComplete;
    const colorInput = el.shadowRoot!.querySelector('input[type="color"]');
    const textInput = el.shadowRoot!.querySelector('input[type="text"]');
    expect(colorInput).not.toBeNull();
    expect(textInput).not.toBeNull();
  });

  it('reflects value as hex color on swatch', async () => {
    (el as any).value = '#ff0000';
    await (el as any).updateComplete;
    const colorInput = el.shadowRoot!.querySelector('input[type="color"]') as HTMLInputElement;
    expect(colorInput.value).toBe('#ff0000');
  });

  it('updates hex input when color changes', async () => {
    (el as any).value = '#00ff00';
    await (el as any).updateComplete;
    const textInput = el.shadowRoot!.querySelector('input[type="text"]') as HTMLInputElement;
    expect(textInput.value).toBe('#00ff00');
  });

  it('hides inputs when readonly', async () => {
    (el as any).readonly = true;
    (el as any).value = '#0000ff';
    await (el as any).updateComplete;
    const colorInput = el.shadowRoot!.querySelector('input[type="color"]') as HTMLInputElement;
    expect(colorInput.disabled).toBe(true);
  });
});
```

- [ ] **Step 2: Implement pages-color-swatch**

Create `packages/pages-ui-components/src/color-swatch/pages-color-swatch.ts`:
- Renders a clickable swatch (colored `<button>` showing the color) + hex `<input type="text">` + hidden `<input type="color">`
- Clicking the swatch opens the native color picker
- Typing in hex input updates the swatch on blur (validates hex format)
- Fires `change` event with hex string value
- `readonly`: disables color picker and hex input, shows swatch + hex value as static display
- Swatch uses `background-color` set from value, border uses `--pages-neutral-6`

- [ ] **Step 3: Write failing tests for pages-slider**

Create `packages/pages-ui-components/src/slider/pages-slider.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesSlider', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-slider');
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-slider')).toBeDefined();
  });

  it('renders range and number inputs', async () => {
    await (el as any).updateComplete;
    const range = el.shadowRoot!.querySelector('input[type="range"]');
    const number = el.shadowRoot!.querySelector('input[type="number"]');
    expect(range).not.toBeNull();
    expect(number).not.toBeNull();
  });

  it('syncs range and number values', async () => {
    (el as any).value = 50;
    (el as any).min = 0;
    (el as any).max = 100;
    await (el as any).updateComplete;
    const range = el.shadowRoot!.querySelector('input[type="range"]') as HTMLInputElement;
    const number = el.shadowRoot!.querySelector('input[type="number"]') as HTMLInputElement;
    expect(range.value).toBe('50');
    expect(number.value).toBe('50');
  });

  it('sets min/max/step on both inputs', async () => {
    (el as any).min = 10;
    (el as any).max = 90;
    (el as any).step = 5;
    await (el as any).updateComplete;
    const range = el.shadowRoot!.querySelector('input[type="range"]') as HTMLInputElement;
    expect(range.min).toBe('10');
    expect(range.max).toBe('90');
    expect(range.step).toBe('5');
  });
});
```

- [ ] **Step 4: Implement pages-slider**

Create `packages/pages-ui-components/src/slider/pages-slider.ts`:
- Renders `<input type="range">` + internal `<input type="number">` side by side
- Both inputs share the same min/max/step/value — changing either updates the other
- The number input is a plain HTML element, NOT a `<pages-number-input>` (avoids transitive side-effect registration)
- Fires `change` event with numeric value
- `readonly`: both inputs non-interactive but visible, focusable

- [ ] **Step 5: Add sub-path exports and barrel**

Update `packages/pages-ui-components/package.json` and `index.ts` for `./color-swatch` and `./slider`.

- [ ] **Step 6: Run all tests**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-ui-components/src/color-swatch/ packages/pages-ui-components/src/slider/ packages/pages-ui-components/src/index.ts packages/pages-ui-components/package.json
git commit -m "feat(#373): add pages-color-swatch and pages-slider controls

Color swatch: hex input + native color picker popup.
Slider: range + number paired inputs with synchronized values.

Refs #373"
```

### Task 5: pages-tag-editor

**Files:**
- Create: `packages/pages-ui-components/src/tag-editor/index.ts`
- Create: `packages/pages-ui-components/src/tag-editor/pages-tag-editor.ts`
- Create: `packages/pages-ui-components/src/tag-editor/pages-tag-editor.test.ts`
- Modify: `packages/pages-ui-components/package.json` — add sub-path export
- Modify: `packages/pages-ui-components/src/index.ts` — add barrel export

**Interfaces:**
- Consumes: `LiveRegionMixin` from `@casehubio/pages-primitives/a11y` (for announcing tag add/remove)
- Produces: `<pages-tag-editor>` — `value: string[]`, `label`, `placeholder`, `maxItems`, `uniqueItems`, `readonly`, `disabled`, `error`

- [ ] **Step 1: Write failing tests**

Create `packages/pages-ui-components/src/tag-editor/pages-tag-editor.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesTagEditor', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-tag-editor');
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-tag-editor')).toBeDefined();
  });

  it('renders existing tags as chips', async () => {
    (el as any).value = ['foo', 'bar'];
    await (el as any).updateComplete;
    const chips = el.shadowRoot!.querySelectorAll('[role="listitem"]');
    expect(chips.length).toBe(2);
    expect(chips[0]!.textContent).toContain('foo');
    expect(chips[1]!.textContent).toContain('bar');
  });

  it('renders a text input for adding tags', async () => {
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input[type="text"]');
    expect(input).not.toBeNull();
  });

  it('each chip has a remove button with aria-label', async () => {
    (el as any).value = ['tag1'];
    await (el as any).updateComplete;
    const btn = el.shadowRoot!.querySelector('button');
    expect(btn).not.toBeNull();
    expect(btn!.getAttribute('aria-label')).toBe("Remove 'tag1'");
  });

  it('hides input and remove buttons when readonly', async () => {
    (el as any).readonly = true;
    (el as any).value = ['a', 'b'];
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input');
    expect(input).toBeNull();
    const buttons = el.shadowRoot!.querySelectorAll('button');
    expect(buttons.length).toBe(0);
  });
});
```

- [ ] **Step 2: Implement pages-tag-editor**

Create `packages/pages-ui-components/src/tag-editor/pages-tag-editor.ts`:
- Renders chips for each string in `value` array, each with a remove `<button>`
- Text `<input>` at the end — Enter adds a new tag, fires `change` event with updated array
- `uniqueItems`: if true, duplicate tags are rejected (show brief validation message)
- `maxItems`: if set, input is disabled when array length reaches max
- Remove button: click removes the tag, fires `change` event
- Add `@casehubio/pages-primitives` as dependency for `LiveRegionMixin` (announce add/remove)
- Chip list uses `role="list"`, each chip `role="listitem"`
- `readonly`: hide input and remove buttons, show tags as read-only chips
- CSS: chips styled with `--pages-accent-3` background, `--pages-accent-11` text, `--pages-radius-sm` border-radius

Create `packages/pages-ui-components/src/tag-editor/index.ts`:
```typescript
export { PagesTagEditor } from './pages-tag-editor.js';
```

- [ ] **Step 3: Add sub-path export and barrel**

Update `packages/pages-ui-components/package.json` and `index.ts` for `./tag-editor`.

Add `@casehubio/pages-primitives` to `packages/pages-ui-components/package.json` dependencies.

- [ ] **Step 4: Run all tests**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: all tests PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: no new errors

- [ ] **Step 6: Commit**

```bash
git add packages/pages-ui-components/src/tag-editor/ packages/pages-ui-components/src/index.ts packages/pages-ui-components/package.json
git commit -m "feat(#373): add pages-tag-editor control

Tag/chip input for string arrays with add/remove, maxItems, uniqueItems,
LiveRegion announcements for a11y.

Refs #373"
```

---

## Batch 3: Property palette package

### Task 6: Package scaffold, types, and default resolver

**Files:**
- Create: `packages/pages-property-palette/package.json`
- Create: `packages/pages-property-palette/tsconfig.json`
- Create: `packages/pages-property-palette/tsconfig.build.json`
- Create: `packages/pages-property-palette/vitest.config.ts`
- Create: `packages/pages-property-palette/src/types.ts`
- Create: `packages/pages-property-palette/src/resolver.ts`
- Create: `packages/pages-property-palette/src/resolver.test.ts`
- Create: `packages/pages-property-palette/src/index.ts`
- Modify: root `package.json` — add workspace entry if needed

**Interfaces:**
- Consumes: `FieldSchema` from `@casehubio/pages-ui-components/types`
- Produces: `PropertyPaletteSource` interface at `@casehubio/pages-property-palette/types`
- Produces: `EditorDescriptor` type at `@casehubio/pages-property-palette/types`
- Produces: `EditorResolver` type at `@casehubio/pages-property-palette/types`
- Produces: `FieldRenderContext` interface at `@casehubio/pages-property-palette/types`
- Produces: `resolveEditor(schema: FieldSchema): EditorDescriptor` at `@casehubio/pages-property-palette`

- [ ] **Step 1: Create package scaffold**

Create `packages/pages-property-palette/package.json`:

```json
{
  "name": "@casehubio/pages-property-palette",
  "version": "0.1.0",
  "description": "Schema-driven property inspector panel",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/casehub-pages.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./palette": { "types": "./dist/palette/index.d.ts", "default": "./dist/palette/index.js" },
    "./types": { "types": "./dist/types.d.ts", "default": "./dist/types.js" }
  },
  "sideEffects": ["./dist/index.js", "./src/index.ts"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/pages-ui-components": "workspace:*",
    "lit": "^3.3.3"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "@casehubio/pages-ui-tokens": "workspace:*",
    "jsdom": "^26.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

Create `packages/pages-property-palette/vitest.config.ts` (copy from pages-ui-components pattern).

Create `packages/pages-property-palette/tsconfig.json` and `tsconfig.build.json` (copy from pages-ui-components pattern, adjusting `references` to include `pages-ui-components`).

- [ ] **Step 2: Write failing tests for resolver**

Create `packages/pages-property-palette/src/resolver.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { resolveEditor } from './resolver.js';
import type { FieldSchema } from '@casehubio/pages-ui-components/types';

describe('resolveEditor', () => {
  it('resolves string to pages-input', () => {
    const result = resolveEditor({ type: 'string' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-input' });
  });

  it('resolves string with enum to pages-select', () => {
    const result = resolveEditor({ type: 'string', enum: ['a', 'b'] });
    expect(result.kind).toBe('tag');
    expect((result as any).tag).toBe('pages-select');
  });

  it('resolves string format:color to pages-color-swatch', () => {
    const result = resolveEditor({ type: 'string', format: 'color' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-color-swatch' });
  });

  it('resolves string format:date to pages-date-input', () => {
    const result = resolveEditor({ type: 'string', format: 'date' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-date-input' });
  });

  it('resolves string format:date-time to pages-datetime-input', () => {
    const result = resolveEditor({ type: 'string', format: 'date-time' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-datetime-input' });
  });

  it('resolves string format:uri to pages-input type url', () => {
    const result = resolveEditor({ type: 'string', format: 'uri' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-input', config: { type: 'url' } });
  });

  it('resolves string x-display-hint:textarea to pages-textarea', () => {
    const result = resolveEditor({ type: 'string', 'x-display-hint': 'textarea' } as FieldSchema);
    expect(result).toEqual({ kind: 'tag', tag: 'pages-textarea' });
  });

  it('resolves number to pages-number-input', () => {
    const result = resolveEditor({ type: 'number' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-number-input' });
  });

  it('resolves integer to pages-number-input', () => {
    const result = resolveEditor({ type: 'integer' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-number-input' });
  });

  it('resolves number x-display-hint:slider to pages-slider', () => {
    const result = resolveEditor({ type: 'number', 'x-display-hint': 'slider' } as FieldSchema);
    expect(result).toEqual({ kind: 'tag', tag: 'pages-slider' });
  });

  it('resolves boolean to pages-checkbox', () => {
    const result = resolveEditor({ type: 'boolean' });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-checkbox' });
  });

  it('resolves array of strings to pages-tag-editor', () => {
    const result = resolveEditor({ type: 'array', items: { type: 'string' } });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-tag-editor' });
  });

  it('resolves object with properties to nested render', () => {
    const result = resolveEditor({ type: 'object', properties: { name: { type: 'string' } } });
    expect(result.kind).toBe('render');
  });

  it('resolves object without properties to JSON display', () => {
    const result = resolveEditor({ type: 'object' });
    expect(result.kind).toBe('render');
  });

  it('handles nullable type ["string", "null"]', () => {
    const result = resolveEditor({ type: ['string', 'null'] });
    expect(result).toEqual({ kind: 'tag', tag: 'pages-input' });
  });

  it('returns JSON display fallback for unknown type', () => {
    const result = resolveEditor({});
    expect(result.kind).toBe('render');
  });
});
```

- [ ] **Step 3: Implement types**

Create `packages/pages-property-palette/src/types.ts`:

```typescript
import type { FieldSchema } from '@casehubio/pages-ui-components/types';
import type { TemplateResult } from 'lit';

export type { FieldSchema };

export interface PropertyPaletteSource {
  readonly schema: FieldSchema;
  readonly data: Record<string, unknown>;
  readonly readonly?: boolean;
  onChange(field: (string | number)[], value: unknown): void;
}

export interface FieldRenderContext {
  key: string;
  schema: FieldSchema;
  value: unknown;
  required: boolean;
  readonly: boolean;
  error: string | undefined;
  onChange: (value: unknown) => void;
}

export type EditorDescriptor =
  | { kind: 'tag'; tag: string; config?: Record<string, unknown> }
  | { kind: 'render'; render: (ctx: FieldRenderContext) => TemplateResult };

export type EditorResolver = (schema: FieldSchema) => EditorDescriptor | undefined;
```

- [ ] **Step 4: Implement resolver**

Create `packages/pages-property-palette/src/resolver.ts`:

```typescript
import type { FieldSchema } from '@casehubio/pages-ui-components/types';
import type { EditorDescriptor } from './types.js';
import { html } from 'lit';

export function resolveEditor(schema: FieldSchema): EditorDescriptor {
  const type = normalizeType(schema.type);
  const displayHint = schema['x-display-hint'] as string | undefined;

  if (type === 'boolean') {
    return { kind: 'tag', tag: 'pages-checkbox' };
  }

  if (type === 'number' || type === 'integer') {
    if (displayHint === 'slider') return { kind: 'tag', tag: 'pages-slider' };
    return { kind: 'tag', tag: 'pages-number-input' };
  }

  if (type === 'string') {
    if (schema.enum && schema.enum.length > 0) return { kind: 'tag', tag: 'pages-select' };
    if (displayHint === 'textarea') return { kind: 'tag', tag: 'pages-textarea' };
    if (schema.format === 'color') return { kind: 'tag', tag: 'pages-color-swatch' };
    if (schema.format === 'date') return { kind: 'tag', tag: 'pages-date-input' };
    if (schema.format === 'date-time') return { kind: 'tag', tag: 'pages-datetime-input' };
    if (schema.format === 'uri') return { kind: 'tag', tag: 'pages-input', config: { type: 'url' } };
    return { kind: 'tag', tag: 'pages-input' };
  }

  if (type === 'array') {
    if (schema.items?.type === 'string') return { kind: 'tag', tag: 'pages-tag-editor' };
    if (schema.items?.enum) {
      return { kind: 'render', render: (ctx) => renderMultiSelectCheckboxes(ctx) };
    }
    return jsonDisplayDescriptor();
  }

  if (type === 'object') {
    if (schema.properties && Object.keys(schema.properties).length > 0) {
      return { kind: 'render', render: () => html`` }; // placeholder — palette handles nested rendering
    }
    return jsonDisplayDescriptor();
  }

  return jsonDisplayDescriptor();
}

function normalizeType(type: string | readonly string[] | undefined): string | undefined {
  if (!type) return undefined;
  if (typeof type === 'string') return type;
  const nonNull = type.filter(t => t !== 'null');
  return nonNull.length === 1 ? nonNull[0] : undefined;
}

function jsonDisplayDescriptor(): EditorDescriptor {
  return {
    kind: 'render',
    render: (ctx) => html`<pre style="font-size: 11px; margin: 0; white-space: pre-wrap;">${JSON.stringify(ctx.value, null, 2) ?? '—'}</pre>`,
  };
}

function renderMultiSelectCheckboxes(ctx: import('./types.js').FieldRenderContext) {
  const options = ctx.schema.items?.enum ?? [];
  const selected = new Set(Array.isArray(ctx.value) ? ctx.value as string[] : []);
  return html`
    <div role="group" aria-label=${ctx.key} style="display: flex; flex-direction: column; gap: 4px;">
      ${options.map(opt => {
        const cb = document.createElement('pages-checkbox') as any;
        cb.label = opt;
        cb.checked = selected.has(opt);
        cb.readonly = ctx.readonly;
        cb.addEventListener('change', () => {
          const next = cb.checked
            ? [...selected, opt]
            : [...selected].filter((v: string) => v !== opt);
          ctx.onChange(next);
        });
        return cb;
      })}
    </div>
  `;
}
```

- [ ] **Step 5: Create barrel index**

Create `packages/pages-property-palette/src/index.ts` (minimal for now — palette element registration comes in Task 7):

```typescript
export { resolveEditor } from './resolver.js';
export type { PropertyPaletteSource, EditorDescriptor, EditorResolver, FieldRenderContext } from './types.js';
```

- [ ] **Step 6: Run yarn install and tests**

Run: `yarn install` (to register new workspace package)
Run: `yarn workspace @casehubio/pages-property-palette run test -- src/resolver.test.ts`
Expected: all resolver tests PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-property-palette/
git commit -m "feat(#373): scaffold pages-property-palette package with types and resolver

EditorResolver maps JSON Schema type+format+x-display-hint to editor
descriptors. Handles nullable types, nested objects, multi-select arrays,
and JSON display fallback.

Refs #373"
```

### Task 7: PagesPropertyPalette component

**Files:**
- Create: `packages/pages-property-palette/src/palette/index.ts`
- Create: `packages/pages-property-palette/src/palette/pages-property-palette.ts`
- Create: `packages/pages-property-palette/src/palette/pages-property-palette.test.ts`
- Modify: `packages/pages-property-palette/src/index.ts` — register element

**Interfaces:**
- Consumes: `PropertyPaletteSource`, `EditorDescriptor`, `EditorResolver` from `./types.ts`
- Consumes: `resolveEditor` from `./resolver.ts`
- Consumes: `validateField` from `@casehubio/pages-ui-components/validation`
- Consumes: `FieldSchema` from `@casehubio/pages-ui-components/types`
- Produces: `<pages-property-palette>` element with `source`, `resolver`, `paletteId` properties

- [ ] **Step 1: Write failing tests**

Create `packages/pages-property-palette/src/palette/pages-property-palette.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import '../index.js';
import type { PropertyPaletteSource } from '../types.js';

// Side-effect imports for form controls used by the palette
import '@casehubio/pages-ui-components/input';
import '@casehubio/pages-ui-components/select';
import '@casehubio/pages-ui-components/checkbox';
import '@casehubio/pages-ui-components/textarea';
import '@casehubio/pages-ui-components/number-input';

describe('PagesPropertyPalette', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-property-palette');
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-property-palette')).toBeDefined();
  });

  it('renders empty when source is undefined', async () => {
    await (el as any).updateComplete;
    const palette = el.shadowRoot!.querySelector('.palette');
    expect(palette).not.toBeNull();
    expect(palette!.children.length).toBe(0);
  });

  it('renders fields from schema', async () => {
    const changes: Array<{ field: (string | number)[]; value: unknown }> = [];
    (el as any).source = {
      schema: {
        properties: {
          name: { type: 'string', title: 'Name' },
          count: { type: 'number', title: 'Count' },
        },
      },
      data: { name: 'hello', count: 5 },
      onChange: (f: (string | number)[], v: unknown) => changes.push({ field: f, value: v }),
    } satisfies PropertyPaletteSource;
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('pages-input, pages-number-input');
    expect(inputs.length).toBe(2);
  });

  it('renders groups from x-group', async () => {
    (el as any).source = {
      schema: {
        properties: {
          color: { type: 'string', 'x-group': 'Appearance', title: 'Color' },
          size: { type: 'number', 'x-group': 'Appearance', title: 'Size' },
          name: { type: 'string', title: 'Name' },
        },
      },
      data: {},
      onChange: () => {},
    };
    await (el as any).updateComplete;
    const details = el.shadowRoot!.querySelectorAll('details');
    expect(details.length).toBe(1);
    expect(details[0]!.querySelector('summary')!.textContent).toContain('Appearance');
  });

  it('hides advanced fields by default', async () => {
    (el as any).source = {
      schema: {
        properties: {
          name: { type: 'string', title: 'Name' },
          debug: { type: 'boolean', title: 'Debug', 'x-visibility': 'advanced' },
        },
      },
      data: {},
      onChange: () => {},
    };
    await (el as any).updateComplete;
    const checkboxes = el.shadowRoot!.querySelectorAll('pages-checkbox');
    expect(checkboxes.length).toBe(0);
    const toggle = el.shadowRoot!.querySelector('.advanced-toggle');
    expect(toggle).not.toBeNull();
  });

  it('shows advanced fields when toggled', async () => {
    (el as any).source = {
      schema: {
        properties: {
          debug: { type: 'boolean', title: 'Debug', 'x-visibility': 'advanced' },
        },
      },
      data: {},
      onChange: () => {},
    };
    await (el as any).updateComplete;
    const toggle = el.shadowRoot!.querySelector('.advanced-toggle input') as HTMLInputElement;
    toggle.click();
    await (el as any).updateComplete;
    const checkboxes = el.shadowRoot!.querySelectorAll('pages-checkbox');
    expect(checkboxes.length).toBe(1);
  });

  it('calls source.onChange on field change', async () => {
    const changes: Array<{ field: (string | number)[]; value: unknown }> = [];
    (el as any).source = {
      schema: { properties: { name: { type: 'string' } } },
      data: { name: 'old' },
      onChange: (f: (string | number)[], v: unknown) => changes.push({ field: f, value: v }),
    };
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('pages-input') as any;
    input.value = 'new';
    input.dispatchEvent(new Event('change', { bubbles: true, composed: true }));
    expect(changes).toEqual([{ field: ['name'], value: 'new' }]);
  });

  it('maps boolean value to pages-checkbox checked property', async () => {
    (el as any).source = {
      schema: { properties: { enabled: { type: 'boolean', title: 'Enabled' } } },
      data: { enabled: true },
      onChange: () => {},
    };
    await (el as any).updateComplete;
    const cb = el.shadowRoot!.querySelector('pages-checkbox') as any;
    expect(cb).not.toBeNull();
    expect(cb.checked).toBe(true);
  });

  it('renders nested object with field paths', async () => {
    const changes: Array<{ field: (string | number)[]; value: unknown }> = [];
    (el as any).source = {
      schema: {
        properties: {
          address: {
            type: 'object',
            title: 'Address',
            properties: {
              city: { type: 'string', title: 'City' },
            },
          },
        },
      },
      data: { address: { city: 'London' } },
      onChange: (f: (string | number)[], v: unknown) => changes.push({ field: f, value: v }),
    };
    await (el as any).updateComplete;
    const details = el.shadowRoot!.querySelector('details');
    expect(details).not.toBeNull();
    const input = el.shadowRoot!.querySelector('pages-input') as any;
    expect(input).not.toBeNull();
    input.value = 'Paris';
    input.dispatchEvent(new Event('change', { bubbles: true, composed: true }));
    expect(changes).toEqual([{ field: ['address', 'city'], value: 'Paris' }]);
  });

  it('sets all fields readonly when source.readonly is true', async () => {
    (el as any).source = {
      schema: { properties: { name: { type: 'string' } } },
      data: { name: 'test' },
      readonly: true,
      onChange: () => {},
    };
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('pages-input') as any;
    expect(input.readonly).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-property-palette run test -- src/palette/pages-property-palette.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PagesPropertyPalette**

Create `packages/pages-property-palette/src/palette/pages-property-palette.ts`:

The component:
1. Takes `source: PropertyPaletteSource | undefined`, `resolver: EditorResolver | undefined`, `paletteId: string | undefined`
2. When source changes, parses schema properties:
   - Groups them by `x-group` (ungrouped first)
   - Orders by `x-order` then schema property order
   - Filters `x-visibility: "advanced"` based on toggle state
3. Renders groups as `<details>` elements with `<summary>`
4. Renders each field by calling `resolveEditor()` (custom resolver first, then default)
5. For `tag` descriptors: creates the element, sets value/label/error/readonly/disabled props, listens for `change` events
6. For `render` descriptors: calls the render function with a `FieldRenderContext`
7. Validates on blur using `validateField()`
8. Persists group collapse state to localStorage when `paletteId` is set
9. For nested objects (`type: object` with `properties`): recursively renders with updated path, capped at 5 levels

Special property mapping:
- `pages-checkbox` uses `checked: boolean`, not `value` — the palette must map `FieldRenderContext.value` to `.checked` for boolean fields
- `x-placeholder` from schema maps to the `placeholder` property on `pages-input`, `pages-textarea`, `pages-number-input`, `pages-tag-editor`
- `x-help` from schema renders as a `<span class="help-icon" title="${helpText}">?</span>` next to the field label
- Required fields render an asterisk `*` after the label text
- `x-order` sorting: within each group, fields are sorted by `x-order` (numeric ascending). Fields without `x-order` are placed after those with it, in schema property order.

Key internal state:
- `@state() private _showAdvanced = false` — toggles advanced fields
- `@state() private _errors: Map<string, string>` — validation error state per field path

Create `packages/pages-property-palette/src/palette/index.ts`:
```typescript
export { PagesPropertyPalette } from './pages-property-palette.js';
```

- [ ] **Step 4: Update barrel index with element registration**

Update `packages/pages-property-palette/src/index.ts`:

```typescript
export { resolveEditor } from './resolver.js';
export { PagesPropertyPalette } from './palette/index.js';
export type { PropertyPaletteSource, EditorDescriptor, EditorResolver, FieldRenderContext } from './types.js';

// Side-effect: import form controls used by the palette
import '@casehubio/pages-ui-components/input';
import '@casehubio/pages-ui-components/select';
import '@casehubio/pages-ui-components/checkbox';
import '@casehubio/pages-ui-components/textarea';
import '@casehubio/pages-ui-components/number-input';
import '@casehubio/pages-ui-components/date-input';
import '@casehubio/pages-ui-components/datetime-input';
import '@casehubio/pages-ui-components/color-swatch';
import '@casehubio/pages-ui-components/slider';
import '@casehubio/pages-ui-components/tag-editor';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-property-palette run test`
Expected: all tests PASS

- [ ] **Step 6: Run typecheck across packages**

Run: `yarn typecheck`
Expected: no new errors

- [ ] **Step 7: Commit**

```bash
git add packages/pages-property-palette/
git commit -m "feat(#373): implement PagesPropertyPalette component

Schema-driven property panel with grouping (x-group), ordering (x-order),
advanced visibility toggle (x-visibility), per-field readonly (readOnly),
inline validation on blur, group collapse persistence (localStorage),
recursive nested objects (5-level cap), and custom resolver support.

Refs #373"
```

---

## Batch 4: blocks-ui migration

### Task 8: Migrate diagram-properties.ts to use pages-property-palette

**Files:**
- Modify: `blocks-ui/packages/diagram-core/src/diagram-properties.ts` — rewrite as thin wrapper
- Delete: `blocks-ui/packages/diagram-core/src/form/field-renderer.ts` (use `ide_refactor_safe_delete`)
- Delete: `blocks-ui/packages/diagram-core/src/form/validation.ts` (use `ide_refactor_safe_delete`)
- Delete: `blocks-ui/packages/diagram-core/src/form/nested-group.ts` (use `ide_refactor_safe_delete`)
- Modify: `blocks-ui/packages/diagram-core/src/form/property-form.ts` — remove (keep trigger-editor.ts)
- Modify: `blocks-ui/packages/diagram-core/package.json` — add `@casehubio/pages-property-palette` dependency

**Interfaces:**
- Consumes: `<pages-property-palette>` element
- Consumes: `PropertyPaletteSource`, `EditorResolver` from `@casehubio/pages-property-palette/types`
- Consumes: `FieldSchema` from `@casehubio/pages-ui-components/types`

- [ ] **Step 1: Add pages-property-palette dependency to blocks-ui**

Update `blocks-ui/packages/diagram-core/package.json`:
Add `"@casehubio/pages-property-palette": "workspace:*"` to dependencies.
Add `"@casehubio/pages-ui-components": "workspace:*"` to dependencies (if not already present).

Run: `yarn install` (from blocks-ui root)

- [ ] **Step 2: Rewrite diagram-properties.ts**

Replace `blocks-ui/packages/diagram-core/src/diagram-properties.ts` with the thin wrapper from the spec:

```typescript
import { LitElement, html } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import type { PropertyValues } from 'lit';
import '@casehubio/pages-property-palette';
import type { PropertyPaletteSource, EditorResolver } from '@casehubio/pages-property-palette/types';
import type { FieldSchema } from '@casehubio/pages-ui-components/types';
import { renderTriggerEditor } from './form/trigger-editor.js';

export function emitPropertyChange(
  field: (string | number)[],
  value: unknown,
): CustomEvent<{ field: (string | number)[]; value: unknown }> {
  return new CustomEvent('property-change', {
    bubbles: true, composed: true,
    detail: { field, value },
  });
}

@customElement('diagram-properties')
export class DiagramProperties extends LitElement {
  @property({ attribute: false }) schema: Record<string, unknown> = {};
  @property({ attribute: false }) data: Record<string, unknown> = {};
  @property({ type: Boolean }) readonly = false;

  private _resolver: EditorResolver = (schema) =>
    schema.oneOf ? { kind: 'render', render: renderTriggerEditor } : undefined;

  @state() private _source: PropertyPaletteSource | undefined;

  override willUpdate(changed: PropertyValues): void {
    if (changed.has('schema') || changed.has('data') || changed.has('readonly')) {
      this._source = {
        schema: this.schema as FieldSchema,
        data: this.data,
        readonly: this.readonly,
        onChange: (field, value) => {
          this.dispatchEvent(emitPropertyChange(field, value));
        },
      };
    }
  }

  override render() {
    return html`
      <pages-property-palette
        .source=${this._source}
        .resolver=${this._resolver}
      ></pages-property-palette>
    `;
  }
}
```

- [ ] **Step 3: Update trigger-editor.ts import**

The `trigger-editor.ts` file may need adjustment — it currently imports from `./field-renderer.js` and `./validation.js`. Update its imports to use `@casehubio/pages-ui-components/types` for FieldSchema and the render function signature from `@casehubio/pages-property-palette/types` for FieldRenderContext. Check `trigger-editor.ts` for exact imports needed.

- [ ] **Step 4: Remove replaced form files**

Use `ide_refactor_safe_delete` to remove:
- `blocks-ui/packages/diagram-core/src/form/field-renderer.ts`
- `blocks-ui/packages/diagram-core/src/form/validation.ts`
- `blocks-ui/packages/diagram-core/src/form/nested-group.ts`
- `blocks-ui/packages/diagram-core/src/form/property-form.ts`

Keep `trigger-editor.ts` — it's domain-specific.

- [ ] **Step 5: Run blocks-ui tests**

Run: `yarn workspace @casehubio/blocks-ui-diagram-core run test` (or equivalent test command for diagram-core)
Expected: existing tests PASS (test any property-form tests that exist)

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck` from blocks-ui root
Expected: no new errors

- [ ] **Step 7: Commit**

```bash
git -C /path/to/blocks-ui add packages/diagram-core/
git -C /path/to/blocks-ui commit -m "feat(#373): migrate diagram-properties to pages-property-palette

DiagramProperties is now a thin wrapper over <pages-property-palette>.
Removed form/field-renderer.ts, form/validation.ts, form/nested-group.ts,
form/property-form.ts — logic replaced by palette's built-in resolver
and shared validator. trigger-editor.ts preserved as domain-specific.

Refs casehubio/casehub-pages#373"
```

---

## References

- [wsp-melviz/specs/issue-373-property-palette/2026-08-26-property-palette-design.md] — design spec this plan implements
- [packages/pages-ui-components/src/input/pages-input.ts] — existing PagesInput component (pattern reference)
- [packages/pages-ui-components/src/types.ts] — current SelectOption type
- [packages/pages-ui-components/package.json] — existing exports map
- [packages/pages-viz/src/form-inputs/schema-types.ts] — existing FieldSchema/validateField to extract
- [packages/pages-viz/src/form-inputs/PagesNumberInput.ts] — data-pipeline-coupled number input (reference)
- [packages/pages-viz/src/form-inputs/PagesDatePicker.ts] — data-pipeline-coupled date picker (reference)
- [blocks-ui/packages/diagram-core/src/form/property-form.ts] — migration source
- [blocks-ui/packages/diagram-core/src/form/field-renderer.ts] — migration source
- [blocks-ui/packages/diagram-core/src/form/validation.ts] — migration source
- [PP-20260705-c7687d] — web-component-strategy protocol (Lit conventions)
- [PP-20260705-2ae91d] — css-design-tokens protocol (token naming)
- [casehubio/casehub-pages#373] — focal issue
- [casehubio/blocks-ui#136] — companion issue
- [casehubio/casehub-pages#374] — deferred duration editor
- [casehubio/casehub-pages#375] — deferred PagesSchemaForm migration
