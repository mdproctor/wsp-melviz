# Duration Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #374 — feat(pages-ui-components): ISO 8601 duration editor (format: duration)
**Issue group:** #374

**Goal:** Add a `<pages-duration-input>` multi-field form control to `pages-ui-components` and wire it into the property palette's resolver.

**Architecture:** Single Lit web component in `pages-ui-components` with inline number inputs for configurable time units (default: H/M/S). Parses and serializes ISO 8601 duration strings. Resolver mapping in `pages-property-palette` connects `format: "duration"` schemas to the new control.

**Tech Stack:** Lit, TypeScript, Vitest, `--pages-*` design tokens

## Global Constraints

- All CSS custom properties use `--pages-` prefix (PP-20260705-2ae91d)
- All web components use Lit, `pages-` element prefix (PP-20260705-c7687d)
- Guarded element registration: `if (!customElements.get('pages-duration-input'))`
- Sub-inputs are plain `<input type="number">`, not custom elements (avoids side-effect registration — same pattern as `pages-slider`)
- Integer-only seconds, no fractional support
- Omit zero-valued units in serialized output; all-zero emits `PT0S`

---

## Batch 1: Duration Input Component

### Task 1: Create `pages-duration-input` with tests

**Files:**
- Create: `packages/pages-ui-components/src/duration-input/pages-duration-input.ts`
- Create: `packages/pages-ui-components/src/duration-input/index.ts`
- Create: `packages/pages-ui-components/src/duration-input/pages-duration-input.test.ts`
- Modify: `packages/pages-ui-components/src/index.ts`
- Modify: `packages/pages-ui-components/package.json`

**Interfaces:**
- Consumes: `lit` (LitElement, html, css, nothing, property decorator, ifDefined directive)
- Produces: `PagesDurationInput` class, `DurationField` type — exported from `@casehubio/pages-ui-components/duration-input`

- [ ] **Step 1: Write the test file**

Create `packages/pages-ui-components/src/duration-input/pages-duration-input.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesDurationInput', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-duration-input');
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-duration-input')).toBeDefined();
  });

  it('renders 3 number inputs for default fields (h/m/s)', async () => {
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    expect(inputs.length).toBe(3);
  });

  it('renders unit labels for default fields', async () => {
    await (el as any).updateComplete;
    const labels = el.shadowRoot!.querySelectorAll('.unit-label');
    expect(labels.length).toBe(3);
    expect(labels[0].textContent).toBe('h');
    expect(labels[1].textContent).toBe('m');
    expect(labels[2].textContent).toBe('s');
  });

  it('renders custom fields', async () => {
    (el as any).fields = ['days', 'hours', 'minutes'];
    await (el as any).updateComplete;
    const labels = el.shadowRoot!.querySelectorAll('.unit-label');
    expect(labels.length).toBe(3);
    expect(labels[0].textContent).toBe('d');
    expect(labels[1].textContent).toBe('h');
    expect(labels[2].textContent).toBe('m');
  });

  it('parses ISO 8601 duration string', async () => {
    (el as any).value = 'PT1H30M15S';
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    expect((inputs[0] as HTMLInputElement).value).toBe('1');
    expect((inputs[1] as HTMLInputElement).value).toBe('30');
    expect((inputs[2] as HTMLInputElement).value).toBe('15');
  });

  it('parses duration with days when field is visible', async () => {
    (el as any).fields = ['days', 'hours', 'minutes', 'seconds'];
    (el as any).value = 'P2DT4H';
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    expect((inputs[0] as HTMLInputElement).value).toBe('2');
    expect((inputs[1] as HTMLInputElement).value).toBe('4');
    expect((inputs[2] as HTMLInputElement).value).toBe('0');
    expect((inputs[3] as HTMLInputElement).value).toBe('0');
  });

  it('defaults to all zeros for invalid string', async () => {
    (el as any).value = 'not-a-duration';
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    expect((inputs[0] as HTMLInputElement).value).toBe('0');
    expect((inputs[1] as HTMLInputElement).value).toBe('0');
    expect((inputs[2] as HTMLInputElement).value).toBe('0');
  });

  it('defaults to all zeros for empty string', async () => {
    (el as any).value = '';
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    expect((inputs[0] as HTMLInputElement).value).toBe('0');
    expect((inputs[1] as HTMLInputElement).value).toBe('0');
    expect((inputs[2] as HTMLInputElement).value).toBe('0');
  });

  it('drops hidden units from parsed value', async () => {
    (el as any).fields = ['hours'];
    (el as any).value = 'P1YT2H30M';
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    expect(inputs.length).toBe(1);
    expect((inputs[0] as HTMLInputElement).value).toBe('2');
  });

  it('serializes with zeros omitted', async () => {
    (el as any).value = 'PT1H0M0S';
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    // Simulate changing minutes to 30
    (inputs[1] as HTMLInputElement).value = '30';
    inputs[1].dispatchEvent(new Event('change', { bubbles: true }));
    await (el as any).updateComplete;
    expect((el as any).value).toBe('PT1H30M');
  });

  it('serializes all-zero as PT0S', async () => {
    (el as any).value = 'PT0S';
    await (el as any).updateComplete;
    expect((el as any).value).toBe('PT0S');
  });

  it('fires change event on sub-input change', async () => {
    await (el as any).updateComplete;
    let fired = false;
    el.addEventListener('change', () => { fired = true; });
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    (inputs[0] as HTMLInputElement).value = '5';
    inputs[0].dispatchEvent(new Event('change', { bubbles: true }));
    expect(fired).toBe(true);
  });

  it('sets readonly on all inputs', async () => {
    (el as any).readonly = true;
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    inputs.forEach(input => {
      expect((input as HTMLInputElement).readOnly).toBe(true);
    });
  });

  it('sets disabled on all inputs', async () => {
    (el as any).disabled = true;
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    inputs.forEach(input => {
      expect((input as HTMLInputElement).disabled).toBe(true);
    });
  });

  it('renders label when provided', async () => {
    (el as any).label = 'Timeout';
    await (el as any).updateComplete;
    const label = el.shadowRoot!.querySelector('label');
    expect(label).not.toBeNull();
    expect(label!.textContent).toBe('Timeout');
  });

  it('shows error message', async () => {
    (el as any).error = 'Required';
    await (el as any).updateComplete;
    const err = el.shadowRoot!.querySelector('[role="alert"]');
    expect(err).not.toBeNull();
    expect(err!.textContent).toBe('Required');
  });

  it('sets aria attributes on group', async () => {
    (el as any).label = 'Duration';
    (el as any).required = true;
    (el as any).error = 'Required';
    await (el as any).updateComplete;
    const group = el.shadowRoot!.querySelector('[role="group"]');
    expect(group).not.toBeNull();
    expect(group!.getAttribute('aria-label')).toBe('Duration');
    expect(group!.getAttribute('aria-required')).toBe('true');
    expect(group!.getAttribute('aria-invalid')).toBe('true');
  });

  it('sets aria-label on individual inputs', async () => {
    (el as any).label = 'Duration';
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    expect((inputs[0] as HTMLInputElement).getAttribute('aria-label')).toBe('Duration hours');
    expect((inputs[1] as HTMLInputElement).getAttribute('aria-label')).toBe('Duration minutes');
    expect((inputs[2] as HTMLInputElement).getAttribute('aria-label')).toBe('Duration seconds');
  });

  it('enforces min=0 on all inputs', async () => {
    await (el as any).updateComplete;
    const inputs = el.shadowRoot!.querySelectorAll('input[type="number"]');
    inputs.forEach(input => {
      expect((input as HTMLInputElement).min).toBe('0');
    });
  });
});
```

- [ ] **Step 2: Create the barrel export**

Create `packages/pages-ui-components/src/duration-input/index.ts`:

```typescript
export { PagesDurationInput, type DurationField } from './pages-duration-input.js';
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/duration-input/pages-duration-input.test.ts`
Expected: FAIL — module not found

- [ ] **Step 4: Write the component implementation**

Create `packages/pages-ui-components/src/duration-input/pages-duration-input.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';
import { ifDefined } from 'lit/directives/if-defined.js';

export type DurationField = 'years' | 'months' | 'days' | 'hours' | 'minutes' | 'seconds';

const UNIT_LABELS: Record<DurationField, string> = {
  years: 'y', months: 'mo', days: 'd', hours: 'h', minutes: 'm', seconds: 's',
};

const UNIT_NAMES: Record<DurationField, string> = {
  years: 'years', months: 'months', days: 'days', hours: 'hours', minutes: 'minutes', seconds: 'seconds',
};

const DURATION_RE = /^P(?:(\d+)Y)?(?:(\d+)M)?(?:(\d+)D)?(?:T(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?)?$/;

const FIELD_PARSE_INDEX: Record<DurationField, number> = {
  years: 1, months: 2, days: 3, hours: 4, minutes: 5, seconds: 6,
};

function parseDuration(value: string, fields: readonly DurationField[]): Record<DurationField, number> {
  const result = {} as Record<DurationField, number>;
  for (const f of fields) result[f] = 0;
  if (!value) return result;
  const match = value.match(DURATION_RE);
  if (!match) return result;
  for (const f of fields) {
    const v = match[FIELD_PARSE_INDEX[f]];
    result[f] = v ? parseInt(v, 10) : 0;
  }
  return result;
}

function serializeDuration(values: Record<DurationField, number>): string {
  const y = values.years || 0;
  const mo = values.months || 0;
  const d = values.days || 0;
  const h = values.hours || 0;
  const m = values.minutes || 0;
  const s = values.seconds || 0;

  let datePart = '';
  if (y) datePart += `${y}Y`;
  if (mo) datePart += `${mo}M`;
  if (d) datePart += `${d}D`;

  let timePart = '';
  if (h) timePart += `${h}H`;
  if (m) timePart += `${m}M`;
  if (s) timePart += `${s}S`;

  if (!datePart && !timePart) return 'PT0S';
  return `P${datePart}${timePart ? `T${timePart}` : ''}`;
}

export class PagesDurationInput extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .field { display: flex; flex-direction: column; gap: 6px; }
    label {
      font-size: var(--pages-font-size-base, 14px);
      font-weight: var(--pages-font-weight-medium, 500);
      color: var(--pages-neutral-12, #333);
    }
    .duration-fields { display: inline-flex; gap: var(--pages-space-2, 8px); align-items: flex-start; }
    .unit { display: flex; flex-direction: column; align-items: center; gap: 2px; }
    .unit input {
      width: 3.5em;
      text-align: center;
      padding: var(--pages-space-1, 4px) var(--pages-space-1, 4px);
      border: 1px solid var(--pages-neutral-6, #e0e0e0);
      border-radius: var(--pages-radius-sm, 4px);
      font-size: var(--pages-font-size-base, 14px);
      font-family: inherit;
      background: var(--pages-neutral-1, #fff);
      color: var(--pages-neutral-12, #333);
      transition: border-color var(--pages-duration-fast, 150ms) var(--pages-ease-out, ease-out);
    }
    .unit input:focus { outline: none; border-color: var(--pages-accent-9, #5470c6); }
    .unit input:read-only { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; }
    .unit input:disabled { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; opacity: 0.6; }
    .unit-label {
      font-size: var(--pages-font-size-xs, 11px);
      color: var(--pages-neutral-9, #6b7280);
    }
    .error {
      color: var(--pages-danger-9, #dc2626);
      font-size: var(--pages-font-size-xs, 11px);
      margin-top: var(--pages-space-0-5, 2px);
    }
  `;

  @property() value = '';
  @property({ attribute: false }) fields: DurationField[] = ['hours', 'minutes', 'seconds'];
  @property() label: string | undefined;
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) readonly = false;
  @property({ type: Boolean }) disabled = false;
  @property() error: string | undefined;

  private _values: Record<DurationField, number> = {} as any;

  override willUpdate(changed: Map<string, unknown>) {
    if (changed.has('value') || changed.has('fields')) {
      this._values = parseDuration(this.value, this.fields);
    }
  }

  private _onFieldChange(field: DurationField, e: Event) {
    const input = e.target as HTMLInputElement;
    const v = input.value === '' ? 0 : Math.max(0, parseInt(input.value, 10) || 0);
    this._values = { ...this._values, [field]: v };
    this.value = serializeDuration(this._values);
    this.dispatchEvent(new Event('change', { bubbles: true, composed: true }));
  }

  override render() {
    const groupLabel = this.label ?? 'Duration';
    return html`
      <div class="field">
        ${this.label ? html`<label>${this.label}</label>` : nothing}
        <div class="duration-fields" role="group"
          aria-label=${groupLabel}
          aria-required=${ifDefined(this.required ? 'true' : undefined)}
          aria-invalid=${ifDefined(this.error ? 'true' : undefined)}
        >
          ${this.fields.map(f => html`
            <div class="unit">
              <input type="number"
                min="0" step="1"
                .value=${String(this._values[f] ?? 0)}
                ?readonly=${this.readonly}
                ?disabled=${this.disabled}
                aria-label="${groupLabel} ${UNIT_NAMES[f]}"
                @change=${(e: Event) => this._onFieldChange(f, e)}
              />
              <span class="unit-label">${UNIT_LABELS[f]}</span>
            </div>
          `)}
        </div>
        ${this.error ? html`<span class="error" role="alert">${this.error}</span>` : nothing}
      </div>
    `;
  }
}

if (!customElements.get('pages-duration-input')) {
  customElements.define('pages-duration-input', PagesDurationInput);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-components run test -- src/duration-input/pages-duration-input.test.ts`
Expected: All 18 tests PASS

- [ ] **Step 6: Wire up exports**

Add to `packages/pages-ui-components/src/index.ts`:

```typescript
export { PagesDurationInput, type DurationField } from './duration-input/index.js';
```

Add to `packages/pages-ui-components/package.json` `"exports"` section:

```json
"./duration-input": {
  "types": "./dist/duration-input/index.d.ts",
  "default": "./dist/duration-input/index.js"
}
```

- [ ] **Step 7: Build and verify**

Run: `yarn workspace @casehubio/pages-ui-components run build`
Expected: Build succeeds, `dist/duration-input/` directory created

- [ ] **Step 8: Commit**

```bash
git -C <PROJECT> add packages/pages-ui-components/src/duration-input/ packages/pages-ui-components/src/index.ts packages/pages-ui-components/package.json
git -C <PROJECT> commit -m "feat(#374): add pages-duration-input ISO 8601 duration editor

Refs #374"
```

---

### Task 2: Wire resolver and palette import

**Files:**
- Modify: `packages/pages-property-palette/src/resolver.ts`
- Modify: `packages/pages-property-palette/src/resolver.test.ts`
- Modify: `packages/pages-property-palette/src/index.ts`

**Interfaces:**
- Consumes: `PagesDurationInput` from `@casehubio/pages-ui-components/duration-input` (Task 1)
- Produces: Resolver maps `{ type: 'string', format: 'duration' }` → `{ kind: 'tag', tag: 'pages-duration-input' }`

- [ ] **Step 1: Write the failing resolver test**

Add to `packages/pages-property-palette/src/resolver.test.ts`:

```typescript
it('resolves string format:duration to pages-duration-input', () => {
  expect(resolveEditor({ type: 'string', format: 'duration' })).toEqual({ kind: 'tag', tag: 'pages-duration-input' });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-property-palette run test -- src/resolver.test.ts`
Expected: FAIL — received `pages-input` instead of `pages-duration-input`

- [ ] **Step 3: Add the resolver mapping**

In `packages/pages-property-palette/src/resolver.ts`, inside the `type === 'string'` branch, add after the `format: 'date-time'` line:

```typescript
if (schema.format === 'duration') return { kind: 'tag', tag: 'pages-duration-input' };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-property-palette run test -- src/resolver.test.ts`
Expected: All tests PASS (including the new duration test)

- [ ] **Step 5: Add the side-effect import**

In `packages/pages-property-palette/src/index.ts`, add alongside the existing form control imports:

```typescript
import '@casehubio/pages-ui-components/duration-input';
```

- [ ] **Step 6: Build and run full test suite**

Run: `yarn build:packages && yarn workspace @casehubio/pages-property-palette run test`
Expected: Build succeeds, all tests pass

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add packages/pages-property-palette/src/resolver.ts packages/pages-property-palette/src/resolver.test.ts packages/pages-property-palette/src/index.ts
git -C <PROJECT> commit -m "feat(#374): wire duration format to pages-duration-input in resolver

Refs #374"
```

---

## References

- `specs/issue-374-duration-editor/2026-08-26-duration-editor-design.md` — design spec
- `packages/pages-ui-components/src/date-input/pages-date-input.ts` — form control pattern
- `packages/pages-ui-components/src/number-input/pages-number-input.ts` — number input pattern
- `packages/pages-ui-components/src/slider/pages-slider.ts` — inline input precedent
- `packages/pages-ui-components/src/index.ts` — barrel export pattern
- `packages/pages-ui-components/package.json` — sub-path export pattern
- `packages/pages-property-palette/src/resolver.ts` — resolver mapping
- `packages/pages-property-palette/src/resolver.test.ts` — resolver test pattern
- `packages/pages-property-palette/src/index.ts` — side-effect import pattern
- `docs/protocols/casehub/web-component-strategy.md` — PP-20260705-c7687d
- `docs/protocols/casehub/css-design-tokens.md` — PP-20260705-2ae91d
- casehubio/casehub-pages#374 — focal issue
- casehubio/casehub-pages#373 — parent issue
