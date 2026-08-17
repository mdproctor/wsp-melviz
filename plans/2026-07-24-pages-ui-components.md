# pages-ui-components Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #233 — feat: @casehubio/pages-ui-components — standalone Lit form components with token styling
**Issue group:** #233, #93, #185

**Goal:** Create a new `@casehubio/pages-ui-components` package with 5 standalone Lit form components, then extract data pipeline logic from form inputs into the activation layer.

**Architecture:** One component per type with simple property APIs. The data pipeline sets those same properties from the outside — components don't know or care where their data comes from. Existing form input classes in pages-viz are deleted (4 of 6), their rendering logic moves to pages-ui-components, their data-binding logic moves to the activation layer.

**Tech Stack:** TypeScript 5, Lit 3, Vitest, `--pages-*` CSS custom properties

**Depends on:** #192 (PagesElement Lit migration) must complete first

## Global Constraints

- All custom elements use `pages-` prefix
- Tag names use guarded registration: `if (!customElements.get('pages-xxx')) customElements.define(...)`
- Styling via `--pages-*` CSS custom properties with fallback values only — no hardcoded colours
- Sub-path exports for side-effect isolation per web-component-strategy protocol
- `@property()` for public API, `@state()` for internal state
- Standard DOM events (`input`, `change`, `click`) — no custom event protocols
- No dependency on `pages-data`, `pages-component`, `pages-runtime`, or `echarts`
- `HTMLElement.dataset` is reserved — never use as a property name (GE-20260615-d356e6)

---

### Task 1: Package scaffolding

**Files:**
- Create: `packages/pages-ui-components/package.json`
- Create: `packages/pages-ui-components/tsconfig.json`
- Create: `packages/pages-ui-components/tsconfig.build.json`
- Create: `packages/pages-ui-components/vitest.config.ts`
- Create: `packages/pages-ui-components/src/index.ts`
- Create: `packages/pages-ui-components/src/types.ts`
- Modify: `package.json` (root — add to `build:packages` script)

**Interfaces:**
- Consumes: nothing
- Produces: `SelectOption` type (`{ value: string; label: string }`), package infrastructure for all subsequent tasks

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/pages-ui-components",
  "version": "0.2.3",
  "description": "CaseHub Pages standalone UI components — Lit web components styled with design tokens",
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
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./input": {
      "types": "./dist/input/index.d.ts",
      "default": "./dist/input/index.js"
    },
    "./select": {
      "types": "./dist/select/index.d.ts",
      "default": "./dist/select/index.js"
    },
    "./textarea": {
      "types": "./dist/textarea/index.d.ts",
      "default": "./dist/textarea/index.js"
    },
    "./checkbox": {
      "types": "./dist/checkbox/index.d.ts",
      "default": "./dist/checkbox/index.js"
    },
    "./button": {
      "types": "./dist/button/index.d.ts",
      "default": "./dist/button/index.js"
    },
    "./types": {
      "types": "./dist/types.d.ts",
      "default": "./dist/types.js"
    }
  },
  "sideEffects": [
    "./dist/index.js",
    "./dist/input/index.js",
    "./dist/select/index.js",
    "./dist/textarea/index.js",
    "./dist/checkbox/index.js",
    "./dist/button/index.js",
    "./src/index.ts",
    "./src/input/index.ts",
    "./src/select/index.ts",
    "./src/textarea/index.ts",
    "./src/checkbox/index.ts",
    "./src/button/index.ts"
  ],
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "lit": "^3.3.3"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "@casehubio/pages-ui-tokens": "workspace:*",
    "jsdom": "^26.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "@casehubio/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true,
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"],
  "references": []
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "emitDeclarationOnly": false,
    "composite": false
  },
  "exclude": ["**/*.test.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  esbuild: {
    target: 'es2022',
    tsconfigRaw: {
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    include: ['src/**/*.test.ts'],
  },
});
```

- [ ] **Step 5: Create src/types.ts**

```typescript
export interface SelectOption {
  readonly value: string;
  readonly label: string;
}
```

- [ ] **Step 6: Create src/index.ts (empty barrel — populated in later tasks)**

```typescript
export type { SelectOption } from './types.js';
```

- [ ] **Step 7: Update root package.json build:packages script**

Add `yarn workspace @casehubio/pages-ui-components run build &&` after `pages-ui-tokens` build and before `pages-data` build in the `build:packages` script.

- [ ] **Step 8: Run yarn install and verify build**

Run: `yarn install && yarn workspace @casehubio/pages-ui-components run build`
Expected: clean build, `dist/` directory created with `index.js`, `index.d.ts`, `types.js`, `types.d.ts`

- [ ] **Step 9: Commit**

```bash
git add packages/pages-ui-components/ package.json yarn.lock
git commit -m "feat(#233): scaffold @casehubio/pages-ui-components package

Refs #233"
```

---

### Task 2: PagesInput component

**Files:**
- Create: `packages/pages-ui-components/src/input/pages-input.ts`
- Create: `packages/pages-ui-components/src/input/index.ts`
- Create: `packages/pages-ui-components/src/input/pages-input.test.ts`
- Modify: `packages/pages-ui-components/src/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `PagesInput` class (tag `pages-input`), properties: `value: string`, `label: string | undefined`, `placeholder: string | undefined`, `maxlength: number | undefined`, `required: boolean`, `readonly: boolean`, `disabled: boolean`, `error: string | undefined`, `type: 'text' | 'email' | 'password' | 'url'`

- [ ] **Step 1: Write failing tests**

Create `src/input/pages-input.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesInput', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-input');
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-input')).toBeDefined();
  });

  it('renders an input element in shadow DOM', async () => {
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input');
    expect(input).not.toBeNull();
    expect(input!.type).toBe('text');
  });

  it('renders label when provided', async () => {
    (el as any).label = 'Name';
    await (el as any).updateComplete;
    const label = el.shadowRoot!.querySelector('label');
    expect(label).not.toBeNull();
    expect(label!.textContent).toBe('Name');
  });

  it('does not render label when undefined', async () => {
    await (el as any).updateComplete;
    const label = el.shadowRoot!.querySelector('label');
    expect(label).toBeNull();
  });

  it('sets placeholder on input', async () => {
    (el as any).placeholder = 'Enter name';
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.placeholder).toBe('Enter name');
  });

  it('sets maxlength on input', async () => {
    (el as any).maxlength = 50;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.maxLength).toBe(50);
  });

  it('reflects value property', async () => {
    (el as any).value = 'hello';
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.value).toBe('hello');
  });

  it('supports type variants', async () => {
    (el as any).type = 'email';
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.type).toBe('email');
  });

  it('sets aria-required when required', async () => {
    (el as any).required = true;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.getAttribute('aria-required')).toBe('true');
    expect(input.required).toBe(true);
  });

  it('sets readonly attribute', async () => {
    (el as any).readonly = true;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.readOnly).toBe(true);
  });

  it('sets disabled attribute', async () => {
    (el as any).disabled = true;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.disabled).toBe(true);
  });

  it('renders error message with role=alert', async () => {
    (el as any).error = 'Required field';
    await (el as any).updateComplete;
    const errorEl = el.shadowRoot!.querySelector('[role="alert"]');
    expect(errorEl).not.toBeNull();
    expect(errorEl!.textContent).toBe('Required field');
  });

  it('sets aria-invalid when error is present', async () => {
    (el as any).error = 'Invalid';
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    expect(input.getAttribute('aria-invalid')).toBe('true');
  });

  it('does not render error when undefined', async () => {
    await (el as any).updateComplete;
    const errorEl = el.shadowRoot!.querySelector('[role="alert"]');
    expect(errorEl).toBeNull();
  });

  it('fires input event on keystroke', async () => {
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    let fired = false;
    el.addEventListener('input', () => { fired = true; });
    input.value = 'a';
    input.dispatchEvent(new Event('input', { bubbles: true, composed: true }));
    expect(fired).toBe(true);
  });

  it('fires change event on blur', async () => {
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')!;
    let fired = false;
    el.addEventListener('change', () => { fired = true; });
    input.dispatchEvent(new Event('change', { bubbles: true, composed: true }));
    expect(fired).toBe(true);
  });

  it('does not render duplicate on re-registration attempt', () => {
    expect(() => {
      document.createElement('pages-input');
    }).not.toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: FAIL — module `./index.js` not found

- [ ] **Step 3: Implement PagesInput**

Create `src/input/pages-input.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';
import { ifDefined } from 'lit/directives/if-defined.js';

export class PagesInput extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .field { display: flex; flex-direction: column; gap: 6px; }
    label {
      font-size: var(--pages-font-size-base, 14px);
      font-weight: var(--pages-font-weight-medium, 500);
      color: var(--pages-neutral-12, #333);
    }
    input {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border: 1px solid var(--pages-neutral-6, #e0e0e0);
      border-radius: var(--pages-radius-sm, 4px);
      font-size: var(--pages-font-size-base, 14px);
      font-family: inherit;
      background: var(--pages-neutral-1, #fff);
      color: var(--pages-neutral-12, #333);
      transition: border-color var(--pages-duration-fast, 150ms) var(--pages-ease-out, ease-out);
    }
    input:focus { outline: none; border-color: var(--pages-accent-9, #5470c6); }
    input:read-only { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; }
    input:disabled { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; opacity: 0.6; }
    .error {
      color: var(--pages-danger-9, #dc2626);
      font-size: var(--pages-font-size-xs, 11px);
      margin-top: var(--pages-space-0-5, 2px);
    }
  `;

  @property() value = '';
  @property() label: string | undefined;
  @property() placeholder: string | undefined;
  @property({ type: Number }) maxlength: number | undefined;
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) readonly = false;
  @property({ type: Boolean }) disabled = false;
  @property() error: string | undefined;
  @property() type: 'text' | 'email' | 'password' | 'url' = 'text';

  override render() {
    return html`
      <div class="field">
        ${this.label ? html`<label>${this.label}</label>` : nothing}
        <input
          .type=${this.type}
          .value=${this.value}
          placeholder=${ifDefined(this.placeholder)}
          maxlength=${ifDefined(this.maxlength)}
          ?required=${this.required}
          ?readonly=${this.readonly}
          ?disabled=${this.disabled}
          aria-required=${ifDefined(this.required ? 'true' : undefined)}
          aria-invalid=${ifDefined(this.error ? 'true' : undefined)}
        />
        ${this.error ? html`<span class="error" role="alert">${this.error}</span>` : nothing}
      </div>
    `;
  }
}

if (!customElements.get('pages-input')) {
  customElements.define('pages-input', PagesInput);
}
```

Create `src/input/index.ts`:

```typescript
export { PagesInput } from './pages-input.js';
```

Update `src/index.ts`:

```typescript
export type { SelectOption } from './types.js';
export { PagesInput } from './input/index.js';
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui-components/src/input/ packages/pages-ui-components/src/index.ts
git commit -m "feat(#233): add PagesInput standalone component

Refs #233"
```

---

### Task 3: PagesSelect component

**Files:**
- Create: `packages/pages-ui-components/src/select/pages-select.ts`
- Create: `packages/pages-ui-components/src/select/index.ts`
- Create: `packages/pages-ui-components/src/select/pages-select.test.ts`
- Modify: `packages/pages-ui-components/src/index.ts`

**Interfaces:**
- Consumes: `SelectOption` from `types.ts`
- Produces: `PagesSelect` class (tag `pages-select`), properties: `value: string`, `label: string | undefined`, `options: SelectOption[]`, `required: boolean`, `disabled: boolean`, `error: string | undefined`

- [ ] **Step 1: Write failing tests**

Create `src/select/pages-select.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesSelect', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-select');
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-select')).toBeDefined();
  });

  it('renders a select element in shadow DOM', async () => {
    await (el as any).updateComplete;
    const select = el.shadowRoot!.querySelector('select');
    expect(select).not.toBeNull();
  });

  it('renders options from property', async () => {
    (el as any).options = [
      { value: 'a', label: 'Alpha' },
      { value: 'b', label: 'Beta' },
    ];
    await (el as any).updateComplete;
    const options = el.shadowRoot!.querySelectorAll('option');
    expect(options.length).toBe(2);
    expect(options[0]!.value).toBe('a');
    expect(options[0]!.textContent).toBe('Alpha');
    expect(options[1]!.value).toBe('b');
  });

  it('selects option matching value property', async () => {
    (el as any).options = [
      { value: 'a', label: 'Alpha' },
      { value: 'b', label: 'Beta' },
    ];
    (el as any).value = 'b';
    await (el as any).updateComplete;
    const select = el.shadowRoot!.querySelector('select')!;
    expect(select.value).toBe('b');
  });

  it('renders label when provided', async () => {
    (el as any).label = 'Country';
    await (el as any).updateComplete;
    const label = el.shadowRoot!.querySelector('label');
    expect(label).not.toBeNull();
    expect(label!.textContent).toBe('Country');
  });

  it('sets disabled on select', async () => {
    (el as any).disabled = true;
    await (el as any).updateComplete;
    const select = el.shadowRoot!.querySelector('select')!;
    expect(select.disabled).toBe(true);
  });

  it('renders error message', async () => {
    (el as any).error = 'Pick one';
    await (el as any).updateComplete;
    const errorEl = el.shadowRoot!.querySelector('[role="alert"]');
    expect(errorEl).not.toBeNull();
    expect(errorEl!.textContent).toBe('Pick one');
  });

  it('fires change event on selection', async () => {
    (el as any).options = [
      { value: 'a', label: 'Alpha' },
      { value: 'b', label: 'Beta' },
    ];
    await (el as any).updateComplete;
    const select = el.shadowRoot!.querySelector('select')!;
    let fired = false;
    el.addEventListener('change', () => { fired = true; });
    select.dispatchEvent(new Event('change', { bubbles: true, composed: true }));
    expect(fired).toBe(true);
  });

  it('renders empty when no options', async () => {
    await (el as any).updateComplete;
    const options = el.shadowRoot!.querySelectorAll('option');
    expect(options.length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: FAIL for PagesSelect tests

- [ ] **Step 3: Implement PagesSelect**

Create `src/select/pages-select.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';
import { ifDefined } from 'lit/directives/if-defined.js';
import type { SelectOption } from '../types.js';

export class PagesSelect extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .field { display: flex; flex-direction: column; gap: 6px; }
    label {
      font-size: var(--pages-font-size-base, 14px);
      font-weight: var(--pages-font-weight-medium, 500);
      color: var(--pages-neutral-12, #333);
    }
    select {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border: 1px solid var(--pages-neutral-6, #e0e0e0);
      border-radius: var(--pages-radius-sm, 4px);
      font-size: var(--pages-font-size-base, 14px);
      font-family: inherit;
      background: var(--pages-neutral-1, #fff);
      color: var(--pages-neutral-12, #333);
      transition: border-color var(--pages-duration-fast, 150ms) var(--pages-ease-out, ease-out);
    }
    select:focus { outline: none; border-color: var(--pages-accent-9, #5470c6); }
    select:disabled { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; opacity: 0.6; }
    .error {
      color: var(--pages-danger-9, #dc2626);
      font-size: var(--pages-font-size-xs, 11px);
      margin-top: var(--pages-space-0-5, 2px);
    }
  `;

  @property() value = '';
  @property() label: string | undefined;
  @property({ attribute: false }) options: SelectOption[] = [];
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) disabled = false;
  @property() error: string | undefined;

  override render() {
    return html`
      <div class="field">
        ${this.label ? html`<label>${this.label}</label>` : nothing}
        <select
          ?required=${this.required}
          ?disabled=${this.disabled}
          aria-required=${ifDefined(this.required ? 'true' : undefined)}
          aria-invalid=${ifDefined(this.error ? 'true' : undefined)}
          @change=${(e: Event) => {
            this.value = (e.target as HTMLSelectElement).value;
          }}
        >
          ${this.options.map((opt) => html`
            <option value=${opt.value} ?selected=${this.value === opt.value}>
              ${opt.label}
            </option>
          `)}
        </select>
        ${this.error ? html`<span class="error" role="alert">${this.error}</span>` : nothing}
      </div>
    `;
  }
}

if (!customElements.get('pages-select')) {
  customElements.define('pages-select', PagesSelect);
}
```

Create `src/select/index.ts`:

```typescript
export { PagesSelect } from './pages-select.js';
```

Update `src/index.ts` — add:

```typescript
export { PagesSelect } from './select/index.js';
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui-components/src/select/ packages/pages-ui-components/src/index.ts
git commit -m "feat(#233): add PagesSelect standalone component

Refs #233"
```

---

### Task 4: PagesTextarea component

**Files:**
- Create: `packages/pages-ui-components/src/textarea/pages-textarea.ts`
- Create: `packages/pages-ui-components/src/textarea/index.ts`
- Create: `packages/pages-ui-components/src/textarea/pages-textarea.test.ts`
- Modify: `packages/pages-ui-components/src/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `PagesTextarea` class (tag `pages-textarea`), properties: `value: string`, `label: string | undefined`, `placeholder: string | undefined`, `rows: number | undefined`, `maxlength: number | undefined`, `required: boolean`, `readonly: boolean`, `disabled: boolean`, `error: string | undefined`

- [ ] **Step 1: Write failing tests**

Create `src/textarea/pages-textarea.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesTextarea', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-textarea');
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-textarea')).toBeDefined();
  });

  it('renders a textarea element', async () => {
    await (el as any).updateComplete;
    const textarea = el.shadowRoot!.querySelector('textarea');
    expect(textarea).not.toBeNull();
  });

  it('reflects value property', async () => {
    (el as any).value = 'multi\nline';
    await (el as any).updateComplete;
    const textarea = el.shadowRoot!.querySelector('textarea')!;
    expect(textarea.value).toBe('multi\nline');
  });

  it('sets rows attribute', async () => {
    (el as any).rows = 5;
    await (el as any).updateComplete;
    const textarea = el.shadowRoot!.querySelector('textarea')!;
    expect(textarea.rows).toBe(5);
  });

  it('renders label', async () => {
    (el as any).label = 'Notes';
    await (el as any).updateComplete;
    expect(el.shadowRoot!.querySelector('label')!.textContent).toBe('Notes');
  });

  it('sets readonly', async () => {
    (el as any).readonly = true;
    await (el as any).updateComplete;
    expect(el.shadowRoot!.querySelector('textarea')!.readOnly).toBe(true);
  });

  it('renders error with aria-invalid', async () => {
    (el as any).error = 'Too long';
    await (el as any).updateComplete;
    expect(el.shadowRoot!.querySelector('[role="alert"]')!.textContent).toBe('Too long');
    expect(el.shadowRoot!.querySelector('textarea')!.getAttribute('aria-invalid')).toBe('true');
  });

  it('fires input event', async () => {
    await (el as any).updateComplete;
    let fired = false;
    el.addEventListener('input', () => { fired = true; });
    el.shadowRoot!.querySelector('textarea')!.dispatchEvent(
      new Event('input', { bubbles: true, composed: true }),
    );
    expect(fired).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement PagesTextarea**

Create `src/textarea/pages-textarea.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';
import { ifDefined } from 'lit/directives/if-defined.js';

export class PagesTextarea extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .field { display: flex; flex-direction: column; gap: 6px; }
    label {
      font-size: var(--pages-font-size-base, 14px);
      font-weight: var(--pages-font-weight-medium, 500);
      color: var(--pages-neutral-12, #333);
    }
    textarea {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border: 1px solid var(--pages-neutral-6, #e0e0e0);
      border-radius: var(--pages-radius-sm, 4px);
      font-size: var(--pages-font-size-base, 14px);
      font-family: inherit;
      background: var(--pages-neutral-1, #fff);
      color: var(--pages-neutral-12, #333);
      resize: vertical;
      transition: border-color var(--pages-duration-fast, 150ms) var(--pages-ease-out, ease-out);
    }
    textarea:focus { outline: none; border-color: var(--pages-accent-9, #5470c6); }
    textarea:read-only { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; }
    textarea:disabled { background: var(--pages-neutral-3, #f5f5f5); cursor: not-allowed; opacity: 0.6; }
    .error {
      color: var(--pages-danger-9, #dc2626);
      font-size: var(--pages-font-size-xs, 11px);
      margin-top: var(--pages-space-0-5, 2px);
    }
  `;

  @property() value = '';
  @property() label: string | undefined;
  @property() placeholder: string | undefined;
  @property({ type: Number }) rows: number | undefined;
  @property({ type: Number }) maxlength: number | undefined;
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) readonly = false;
  @property({ type: Boolean }) disabled = false;
  @property() error: string | undefined;

  override render() {
    return html`
      <div class="field">
        ${this.label ? html`<label>${this.label}</label>` : nothing}
        <textarea
          .value=${this.value}
          rows=${ifDefined(this.rows)}
          maxlength=${ifDefined(this.maxlength)}
          placeholder=${ifDefined(this.placeholder)}
          ?required=${this.required}
          ?readonly=${this.readonly}
          ?disabled=${this.disabled}
          aria-required=${ifDefined(this.required ? 'true' : undefined)}
          aria-invalid=${ifDefined(this.error ? 'true' : undefined)}
        ></textarea>
        ${this.error ? html`<span class="error" role="alert">${this.error}</span>` : nothing}
      </div>
    `;
  }
}

if (!customElements.get('pages-textarea')) {
  customElements.define('pages-textarea', PagesTextarea);
}
```

Create `src/textarea/index.ts`:

```typescript
export { PagesTextarea } from './pages-textarea.js';
```

Update `src/index.ts` — add:

```typescript
export { PagesTextarea } from './textarea/index.js';
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui-components/src/textarea/ packages/pages-ui-components/src/index.ts
git commit -m "feat(#233): add PagesTextarea standalone component

Refs #233"
```

---

### Task 5: PagesCheckbox component

**Files:**
- Create: `packages/pages-ui-components/src/checkbox/pages-checkbox.ts`
- Create: `packages/pages-ui-components/src/checkbox/index.ts`
- Create: `packages/pages-ui-components/src/checkbox/pages-checkbox.test.ts`
- Modify: `packages/pages-ui-components/src/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `PagesCheckbox` class (tag `pages-checkbox`), properties: `checked: boolean`, `label: string | undefined`, `required: boolean`, `disabled: boolean`, `error: string | undefined`

- [ ] **Step 1: Write failing tests**

Create `src/checkbox/pages-checkbox.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesCheckbox', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-checkbox');
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-checkbox')).toBeDefined();
  });

  it('renders a checkbox input', async () => {
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input[type="checkbox"]');
    expect(input).not.toBeNull();
  });

  it('reflects checked property', async () => {
    (el as any).checked = true;
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector('input')! as HTMLInputElement;
    expect(input.checked).toBe(true);
  });

  it('renders label', async () => {
    (el as any).label = 'Accept terms';
    await (el as any).updateComplete;
    const label = el.shadowRoot!.querySelector('label');
    expect(label).not.toBeNull();
    expect(label!.textContent).toBe('Accept terms');
  });

  it('sets disabled', async () => {
    (el as any).disabled = true;
    await (el as any).updateComplete;
    expect(el.shadowRoot!.querySelector('input')!.disabled).toBe(true);
  });

  it('renders error', async () => {
    (el as any).error = 'Must accept';
    await (el as any).updateComplete;
    expect(el.shadowRoot!.querySelector('[role="alert"]')!.textContent).toBe('Must accept');
  });

  it('fires change event on toggle', async () => {
    await (el as any).updateComplete;
    let fired = false;
    el.addEventListener('change', () => { fired = true; });
    el.shadowRoot!.querySelector('input')!.dispatchEvent(
      new Event('change', { bubbles: true, composed: true }),
    );
    expect(fired).toBe(true);
  });

  it('defaults to unchecked', async () => {
    await (el as any).updateComplete;
    expect((el.shadowRoot!.querySelector('input')! as HTMLInputElement).checked).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement PagesCheckbox**

Create `src/checkbox/pages-checkbox.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';
import { ifDefined } from 'lit/directives/if-defined.js';

export class PagesCheckbox extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .field { display: flex; align-items: center; gap: var(--pages-space-1, 4px); }
    label {
      font-size: var(--pages-font-size-base, 14px);
      font-weight: var(--pages-font-weight-medium, 500);
      color: var(--pages-neutral-12, #333);
      cursor: pointer;
    }
    input[type="checkbox"] {
      width: 18px; height: 18px; cursor: pointer;
      accent-color: var(--pages-accent-9, #5470c6);
    }
    input[type="checkbox"]:disabled { cursor: not-allowed; }
    .error {
      color: var(--pages-danger-9, #dc2626);
      font-size: var(--pages-font-size-xs, 11px);
      margin-top: var(--pages-space-0-5, 2px);
    }
  `;

  @property({ type: Boolean }) checked = false;
  @property() label: string | undefined;
  @property({ type: Boolean }) required = false;
  @property({ type: Boolean }) disabled = false;
  @property() error: string | undefined;

  override render() {
    return html`
      <div>
        <div class="field">
          <input
            type="checkbox"
            id="cb"
            .checked=${this.checked}
            ?required=${this.required}
            ?disabled=${this.disabled}
            aria-required=${ifDefined(this.required ? 'true' : undefined)}
            aria-invalid=${ifDefined(this.error ? 'true' : undefined)}
            @change=${(e: Event) => {
              this.checked = (e.target as HTMLInputElement).checked;
            }}
          />
          ${this.label ? html`<label for="cb">${this.label}</label>` : nothing}
        </div>
        ${this.error ? html`<span class="error" role="alert">${this.error}</span>` : nothing}
      </div>
    `;
  }
}

if (!customElements.get('pages-checkbox')) {
  customElements.define('pages-checkbox', PagesCheckbox);
}
```

Create `src/checkbox/index.ts`:

```typescript
export { PagesCheckbox } from './pages-checkbox.js';
```

Update `src/index.ts` — add:

```typescript
export { PagesCheckbox } from './checkbox/index.js';
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui-components/src/checkbox/ packages/pages-ui-components/src/index.ts
git commit -m "feat(#233): add PagesCheckbox standalone component

Refs #233"
```

---

### Task 6: PagesButton component

**Files:**
- Create: `packages/pages-ui-components/src/button/pages-button.ts`
- Create: `packages/pages-ui-components/src/button/index.ts`
- Create: `packages/pages-ui-components/src/button/pages-button.test.ts`
- Modify: `packages/pages-ui-components/src/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `PagesButton` class (tag `pages-button`), properties: `label: string`, `variant: 'primary' | 'secondary' | 'ghost' | 'danger'`, `disabled: boolean`, `loading: boolean`, `size: 'sm' | 'md' | 'lg'`

- [ ] **Step 1: Write failing tests**

Create `src/button/pages-button.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './index.js';

describe('PagesButton', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-button');
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-button')).toBeDefined();
  });

  it('renders a button element', async () => {
    await (el as any).updateComplete;
    const button = el.shadowRoot!.querySelector('button');
    expect(button).not.toBeNull();
  });

  it('renders label text', async () => {
    (el as any).label = 'Submit';
    await (el as any).updateComplete;
    const button = el.shadowRoot!.querySelector('button')!;
    expect(button.textContent!.trim()).toBe('Submit');
  });

  it('renders slot content when no label', async () => {
    el.textContent = 'Slot Content';
    await (el as any).updateComplete;
    const slot = el.shadowRoot!.querySelector('slot');
    expect(slot).not.toBeNull();
  });

  it('applies variant class', async () => {
    (el as any).variant = 'primary';
    await (el as any).updateComplete;
    const button = el.shadowRoot!.querySelector('button')!;
    expect(button.classList.contains('primary')).toBe(true);
  });

  it('defaults to secondary variant', async () => {
    await (el as any).updateComplete;
    const button = el.shadowRoot!.querySelector('button')!;
    expect(button.classList.contains('secondary')).toBe(true);
  });

  it('applies danger variant', async () => {
    (el as any).variant = 'danger';
    await (el as any).updateComplete;
    const button = el.shadowRoot!.querySelector('button')!;
    expect(button.classList.contains('danger')).toBe(true);
  });

  it('sets disabled', async () => {
    (el as any).disabled = true;
    await (el as any).updateComplete;
    expect(el.shadowRoot!.querySelector('button')!.disabled).toBe(true);
  });

  it('disables button when loading', async () => {
    (el as any).loading = true;
    await (el as any).updateComplete;
    expect(el.shadowRoot!.querySelector('button')!.disabled).toBe(true);
  });

  it('shows spinner when loading', async () => {
    (el as any).loading = true;
    await (el as any).updateComplete;
    const spinner = el.shadowRoot!.querySelector('.spinner');
    expect(spinner).not.toBeNull();
  });

  it('applies size class', async () => {
    (el as any).size = 'sm';
    await (el as any).updateComplete;
    const button = el.shadowRoot!.querySelector('button')!;
    expect(button.classList.contains('sm')).toBe(true);
  });

  it('fires click event', async () => {
    await (el as any).updateComplete;
    let fired = false;
    el.addEventListener('click', () => { fired = true; });
    el.shadowRoot!.querySelector('button')!.click();
    expect(fired).toBe(true);
  });

  it('does not fire click when disabled', async () => {
    (el as any).disabled = true;
    await (el as any).updateComplete;
    let fired = false;
    el.addEventListener('click', () => { fired = true; });
    el.shadowRoot!.querySelector('button')!.click();
    expect(fired).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement PagesButton**

Create `src/button/pages-button.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { property } from 'lit/decorators.js';
import { classMap } from 'lit/directives/class-map.js';

export class PagesButton extends LitElement {
  static override styles = css`
    :host { display: inline-block; }
    button {
      display: inline-flex; align-items: center; gap: var(--pages-space-1, 4px);
      padding: var(--pages-space-1, 4px) var(--pages-space-3, 12px);
      border-radius: var(--pages-radius-sm, 4px);
      font-size: var(--pages-font-size-base, 14px);
      font-family: var(--pages-font-family, system-ui, sans-serif);
      font-weight: var(--pages-font-weight-medium, 500);
      cursor: pointer;
      border: 1px solid transparent;
      transition: background var(--pages-duration-fast, 150ms) var(--pages-ease-out, ease-out),
                  border-color var(--pages-duration-fast, 150ms) var(--pages-ease-out, ease-out);
    }
    button:disabled { cursor: not-allowed; opacity: 0.6; }
    button.sm { padding: var(--pages-space-0-5, 2px) var(--pages-space-2, 8px); font-size: var(--pages-font-size-sm, 12px); }
    button.lg { padding: var(--pages-space-2, 8px) var(--pages-space-4, 16px); font-size: var(--pages-font-size-lg, 16px); }
    button.primary {
      background: var(--pages-accent-9, #5470c6); color: white; border-color: var(--pages-accent-9, #5470c6);
    }
    button.primary:hover:not(:disabled) { background: var(--pages-accent-10, #4060b6); }
    button.secondary {
      background: transparent; color: var(--pages-accent-9, #5470c6); border-color: var(--pages-accent-9, #5470c6);
    }
    button.secondary:hover:not(:disabled) { background: var(--pages-accent-3, #e8eaf6); }
    button.ghost {
      background: transparent; color: var(--pages-neutral-12, #333); border-color: transparent;
    }
    button.ghost:hover:not(:disabled) { background: var(--pages-neutral-3, #f5f5f5); }
    button.danger {
      background: var(--pages-danger-9, #dc2626); color: white; border-color: var(--pages-danger-9, #dc2626);
    }
    button.danger:hover:not(:disabled) { background: var(--pages-danger-10, #b91c1c); }
    @keyframes spin { to { transform: rotate(360deg); } }
    .spinner {
      width: 14px; height: 14px; border: 2px solid currentColor;
      border-top-color: transparent; border-radius: 50%; animation: spin 0.6s linear infinite;
    }
  `;

  @property() label = '';
  @property() variant: 'primary' | 'secondary' | 'ghost' | 'danger' = 'secondary';
  @property({ type: Boolean }) disabled = false;
  @property({ type: Boolean }) loading = false;
  @property() size: 'sm' | 'md' | 'lg' = 'md';

  override render() {
    const classes = {
      [this.variant]: true,
      [this.size]: this.size !== 'md',
    };

    return html`
      <button
        class=${classMap(classes)}
        ?disabled=${this.disabled || this.loading}
      >
        ${this.loading ? html`<span class="spinner"></span>` : ''}
        ${this.label || html`<slot></slot>`}
      </button>
    `;
  }
}

if (!customElements.get('pages-button')) {
  customElements.define('pages-button', PagesButton);
}
```

Create `src/button/index.ts`:

```typescript
export { PagesButton } from './pages-button.js';
```

Update `src/index.ts` — add:

```typescript
export { PagesButton } from './button/index.js';
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Verify full build**

Run: `yarn workspace @casehubio/pages-ui-components run build`
Expected: clean build with all sub-path entry points in `dist/`

- [ ] **Step 6: Commit**

```bash
git add packages/pages-ui-components/src/button/ packages/pages-ui-components/src/index.ts
git commit -m "feat(#233): add PagesButton standalone component

Refs #233"
```

---

### Task 7: Type system renames — ComponentTypeRegistry and type guards

**Files:**
- Modify: `packages/pages-component/src/model/type-guards.ts`
- Modify: `packages/pages-component/src/model/index.ts`
- Modify: `packages/pages-ui/src/model/type-guards.ts`
- Modify: `packages/pages-ui/src/model/index.ts`
- Modify: `packages/pages-ui/src/model/type-guards.test.ts`

**Interfaces:**
- Consumes: existing `ComponentTypeRegistry`, `isTextInput`, `isDropdown` etc.
- Produces: updated registry with `input` (replacing `text-input`), `select` (replacing `dropdown`), and new type guards `isInput()`, `isSelect()` (replacing `isTextInput()`, `isDropdown()`). Old names removed.

- [ ] **Step 1: Update tests in pages-ui**

In `packages/pages-ui/src/model/type-guards.test.ts`, rename test cases:
- `isTextInput matches text-input type` → `isInput matches input type`
- `isDropdown matches dropdown type` → `isSelect matches select type`
- Update component type strings: `"text-input"` → `"input"`, `"dropdown"` → `"select"`
- Update function calls: `isTextInput(c)` → `isInput(c)`, `isDropdown(c)` → `isSelect(c)`
- Update FORM_INPUT_TYPES test expectations: `"text-input"` → `"input"`, `"dropdown"` → `"select"`

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui run test`
Expected: FAIL — `isInput` and `isSelect` not found

- [ ] **Step 3: Update ComponentTypeRegistry in pages-component**

In `packages/pages-component/src/model/type-guards.ts`:
- Change `"text-input": TextInputProps` → `input: TextInputProps`
- Change `dropdown: DropdownProps` → `select: DropdownProps`
- Rename `isTextInput()` → `isInput()`, update body to check `c.type === "input"`
- Rename `isDropdown()` → `isSelect()`, update body to check `c.type === "select"`

- [ ] **Step 4: Update pages-component exports**

In `packages/pages-component/src/model/index.ts`:
- Replace `isTextInput` with `isInput` in the export list
- Replace `isDropdown` with `isSelect` in the export list

- [ ] **Step 5: Update pages-ui re-exports**

In `packages/pages-ui/src/model/type-guards.ts`:
- Replace `isTextInput` with `isInput` in the re-export from `@casehubio/pages-component`
- Replace `isDropdown` with `isSelect` in the re-export
- Update `FORM_INPUT_TYPES`: `"text-input"` → `"input"`, `"dropdown"` → `"select"`

In `packages/pages-ui/src/model/index.ts`:
- Replace `isTextInput` with `isInput` in the export list
- Replace `isDropdown` with `isSelect` in the export list

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test`
Expected: PASS

- [ ] **Step 7: Run typecheck across affected packages**

Run: `yarn typecheck`
Expected: TypeScript will surface all stale references to old type names as compile errors. Fix any remaining references.

- [ ] **Step 8: Commit**

```bash
git add packages/pages-component/src/model/ packages/pages-ui/src/model/
git commit -m "refactor(#233): rename text-input→input, dropdown→select in type system

ComponentTypeRegistry keys, type guards, and re-exports updated.
FORM_INPUT_TYPES set updated.

Refs #233"
```

---

### Task 8: Activation layer — FormFieldProxy and pipeline extraction

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/activation.test.ts`
- Modify: `packages/pages-runtime/package.json`

**Interfaces:**
- Consumes: `PagesInput`, `PagesSelect`, `PagesTextarea`, `PagesCheckbox` from `@casehubio/pages-ui-components`, `VizTarget` from `@casehubio/pages-component`, `TypedDataSet`/`ColumnId` from `@casehubio/pages-data`
- Produces: `createFormFieldProxy()` function, updated activation callback that creates standalone form components + wires pipeline externally

- [ ] **Step 1: Add pages-ui-components dependency to pages-runtime**

In `packages/pages-runtime/package.json`, add to `dependencies`:
```json
"@casehubio/pages-ui-components": "workspace:*"
```

Run: `yarn install`

- [ ] **Step 2: Write failing tests for FormFieldProxy**

Add tests to `packages/pages-runtime/src/activation.test.ts` (or create a new `form-field-proxy.test.ts`):

```typescript
import { describe, it, expect } from 'vitest';
// Test that createFormFieldProxy creates a VizTarget that:
// 1. Extracts field value from TypedDataSet and sets it on the component
// 2. Clears error when dataSet is set (mutual-clearing invariant)
// 3. Clears dataSet reference when error is set
// 4. Clears error when loading is set to true
```

- [ ] **Step 3: Implement createFormFieldProxy**

Add to `activation.ts`:

```typescript
import '@casehubio/pages-ui-components/input';
import '@casehubio/pages-ui-components/select';
import '@casehubio/pages-ui-components/textarea';
import '@casehubio/pages-ui-components/checkbox';

function createFormFieldProxy(
  component: HTMLElement,
  fieldName: string,
): VizTarget {
  let _dataSet: TypedDataSet | undefined;
  return {
    get loading() { return false; },
    set loading(v: boolean) {
      if (v) (component as any).error = undefined;
    },
    get dataSet() { return _dataSet; },
    set dataSet(ds: TypedDataSet | undefined) {
      _dataSet = ds;
      (component as any).error = undefined;
      if (ds) {
        const value = extractFormFieldValue(ds, fieldName);
        setFormComponentValue(component, value);
      }
    },
    get error() { return (component as any).error ?? ''; },
    set error(msg: string) {
      _dataSet = undefined;
      (component as any).error = msg || undefined;
    },
    get totalRows() { return 0; },
    set totalRows(_: number) {},
    get activeSort() { return undefined; },
    set activeSort(_: SortColumn | undefined) {},
    get activePage() { return undefined; },
    set activePage(_: number | undefined) {},
  };
}

function extractFormFieldValue(dataset: TypedDataSet, field: string): unknown {
  if (!dataset.rows.length) return undefined;
  const row = dataset.rows[0];
  if (!row) return undefined;
  try {
    const cell = row.cell(field as ColumnId);
    if (cell.type === 'NULL') return undefined;
    return cell.value;
  } catch {
    return undefined;
  }
}

function setFormComponentValue(component: HTMLElement, value: unknown): void {
  const tag = component.tagName.toLowerCase();
  if (tag === 'pages-checkbox') {
    let checked = false;
    if (typeof value === 'boolean') checked = value;
    else if (typeof value === 'string') checked = value.toLowerCase() === 'true';
    (component as any).checked = checked;
  } else {
    (component as any).value = value !== undefined && value !== null ? String(value) : '';
  }
}
```

- [ ] **Step 4: Update FORM_INPUT_TYPES and activation callback**

Update `FORM_INPUT_TYPES`:
```typescript
const STANDALONE_FORM_TYPES = new Set(["input", "select", "textarea", "checkbox"]);
const LEGACY_FORM_TYPES = new Set(["number-input", "date-picker"]);
const FORM_INPUT_TYPES = new Set([...STANDALONE_FORM_TYPES, ...LEGACY_FORM_TYPES]);
```

In the activation callback, add a branch for `STANDALONE_FORM_TYPES` before the generic `DATA_COMPONENT_TYPES` branch:

```typescript
if (STANDALONE_FORM_TYPES.has(component.type)) {
  const tagName = `pages-${component.type}`;
  const formEl = document.createElement(tagName);
  const field = (component.props as Record<string, unknown>)?.field as string | undefined;

  // Set direct props (label, placeholder, etc.)
  if (component.props) {
    const { label, placeholder, maxLength, required, readonly: ro, rows, options: optsProp } = component.props as any;
    if (label) (formEl as any).label = label;
    if (placeholder) (formEl as any).placeholder = placeholder;
    if (maxLength) (formEl as any).maxlength = maxLength;
    if (required) (formEl as any).required = required;
    if (ro) (formEl as any).readonly = ro;
    if (rows) (formEl as any).rows = rows;
    // Handle FixedOptions for select
    if (optsProp && component.type === 'select') {
      if (optsProp.values) {
        (formEl as any).options = optsProp.values.map((v: string) => ({ value: v, label: v }));
      }
    }
  }

  // Create proxy and register
  const proxy = field ? createFormFieldProxy(formEl, field) : undefined;
  let lookup = (component.props as Record<string, unknown>)?.lookup as DataSetLookup | undefined;

  // Form input implicit lookup injection from dataScope
  if (options) {
    const pageDataScope = options.dataScopeRegistry.get(pagePath);
    if (pageDataScope) {
      lookup = { dataSetId: pageDataScope.dataset, operations: [] };
      // editable → not disabled
    } else if (field) {
      (formEl as any).error = 'Form input requires page dataScope';
    }
  }

  const entry = {
    element: el,
    ...(proxy && { vizElement: proxy }),
    component,
    pagePath,
    hasExplicitId: component.id !== undefined,
    ...(lookup !== undefined && { originalLookup: lookup }),
  };
  registry.set(componentId, entry);
  el.appendChild(formEl);

  // Wire native events → pipeline events
  if (field) {
    const emitFieldChange = (value: unknown, committed: boolean) => {
      formEl.dispatchEvent(new CustomEvent('pages-field-change', {
        bubbles: true, composed: true,
        detail: { field, value, committed },
      }));
    };
    formEl.addEventListener('input', (e: Event) => {
      emitFieldChange((e.target as any).value, false);
    });
    formEl.addEventListener('change', (e: Event) => {
      const target = e.target as any;
      const val = component.type === 'checkbox' ? target.checked : target.value;
      emitFieldChange(val, true);
    });
  }

  // Dispatch data request
  if (proxy && lookup) {
    formEl.dispatchEvent(new CustomEvent('pages-data-request', {
      bubbles: true, composed: true,
      detail: { element: proxy, lookup },
    }));
  }

  // Cleanup controller
  if ('addController' in formEl) {
    (formEl as any).addController({
      hostConnected() {},
      hostDisconnected() {
        registry.delete(componentId);
      },
    });
  }

  return;
}
```

- [ ] **Step 5: Update DATA_COMPONENT_TYPES**

Add the new type names and ensure old names are removed:
```typescript
const DATA_COMPONENT_TYPES = new Set([
  // ... existing chart/table/display types ...
  ...FORM_INPUT_TYPES,
]);
```

- [ ] **Step 6: Run activation tests**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: existing tests may need updating for new type names. Fix as needed.

- [ ] **Step 7: Run full typecheck**

Run: `yarn typecheck`
Expected: PASS — all type references updated

- [ ] **Step 8: Commit**

```bash
git add packages/pages-runtime/
git commit -m "feat(#233): add FormFieldProxy and standalone form activation

Activation layer creates standalone components for input, select,
textarea, checkbox. Pipeline wired externally via FormFieldProxy.

Refs #233"
```

---

### Task 9: Delete extracted form input classes from pages-viz

**Files:**
- Delete: `packages/pages-viz/src/form-inputs/PagesTextInput.ts`
- Delete: `packages/pages-viz/src/form-inputs/PagesDropdown.ts`
- Delete: `packages/pages-viz/src/form-inputs/PagesTextarea.ts`
- Delete: `packages/pages-viz/src/form-inputs/PagesCheckbox.ts`
- Delete: `packages/pages-viz/src/form-inputs/form-inputs.test.ts`
- Delete: `packages/pages-viz/src/form-inputs/form-submit.test.ts`
- Modify: `packages/pages-viz/src/index.ts` — remove deleted exports
- Modify: `packages/pages-viz/src/custom-elements.ts` — remove deleted entries
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — remove deleted side-effect imports

**Interfaces:**
- Consumes: Tasks 2-6 (standalone components exist), Task 8 (activation layer handles these types)
- Produces: cleaned-up pages-viz without the 4 extracted form input classes

- [ ] **Step 1: Remove deleted class exports from index.ts**

In `packages/pages-viz/src/index.ts`, remove these lines:
```typescript
export { PagesTextInput } from "./form-inputs/PagesTextInput.js";
export { PagesDropdown } from "./form-inputs/PagesDropdown.js";
export { PagesCheckbox } from "./form-inputs/PagesCheckbox.js";
export { PagesTextarea } from "./form-inputs/PagesTextarea.js";
```

- [ ] **Step 2: Remove deleted entries from custom-elements.ts**

In `packages/pages-viz/src/custom-elements.ts`, remove the import and declaration for:
- `PagesCheckbox` / `"pages-checkbox"`
- `PagesDropdown` / `"pages-dropdown"`
- `PagesTextInput` / `"pages-text-input"`
- `PagesTextarea` / `"pages-textarea"`

- [ ] **Step 3: Remove side-effect imports from PagesSchemaForm.ts**

In `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`, remove:
```typescript
import "./PagesTextInput.js";
import "./PagesCheckbox.js";
import "./PagesDropdown.js";
import "./PagesTextarea.js";
```

Keep:
```typescript
import "./PagesNumberInput.js";
import "./PagesDatePicker.js";
```

- [ ] **Step 4: Update mapFieldToComponentType in schema-types.ts**

In `packages/pages-viz/src/form-inputs/schema-types.ts`:
- Change `return "text-input"` → `return "input"` (two occurrences)
- Change `return "dropdown"` → `return "select"` (two occurrences)

- [ ] **Step 5: Delete the 4 source files and their test files**

Use `ide_refactor_safe_delete` for each:
- `packages/pages-viz/src/form-inputs/PagesTextInput.ts`
- `packages/pages-viz/src/form-inputs/PagesDropdown.ts`
- `packages/pages-viz/src/form-inputs/PagesTextarea.ts`
- `packages/pages-viz/src/form-inputs/PagesCheckbox.ts`
- `packages/pages-viz/src/form-inputs/form-inputs.test.ts`
- `packages/pages-viz/src/form-inputs/form-submit.test.ts`

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/pages-viz run test`
Expected: remaining tests pass (PagesFormInput base, PagesNumberInput, PagesDatePicker, PagesSchemaForm)

- [ ] **Step 7: Run full typecheck**

Run: `yarn typecheck`
Expected: PASS — no dangling references to deleted classes

- [ ] **Step 8: Commit**

```bash
git add packages/pages-viz/
git commit -m "refactor(#233): delete extracted form inputs from pages-viz

PagesTextInput, PagesDropdown, PagesTextarea, PagesCheckbox removed.
PagesFormInput base retained for PagesNumberInput and PagesDatePicker.
Schema form updated for new type names.

Refs #233"
```

---

### Task 10: PagesSchemaForm child adapter

**Files:**
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts`
- Modify: `packages/pages-viz/src/form-inputs/schema-form.test.ts`

**Interfaces:**
- Consumes: standalone components from `@casehubio/pages-ui-components`, `PagesFormInput` API for legacy children
- Produces: PagesSchemaForm that creates standalone components for `input`/`select`/`checkbox`/`textarea` and PagesFormInput-based components for `number-input`/`date-picker`, with a thin adapter normalizing the API

- [ ] **Step 1: Update schema form tests**

Update `packages/pages-viz/src/form-inputs/schema-form.test.ts`:
- Change expected tag names: `"pages-text-input"` → `"pages-input"`, `"pages-dropdown"` → `"pages-select"`
- Update querySelector calls to use new tag names
- Add tests for the adapter: verify standalone children receive `.value`, `.label`, `.error` etc. correctly

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-viz run test -- schema-form`
Expected: FAIL — old tag names no longer exist

- [ ] **Step 3: Update PagesSchemaForm child interaction**

In `PagesSchemaForm.ts`, add a helper to normalize interaction with both standalone and PagesFormInput children:

```typescript
private setChildProps(child: HTMLElement, componentType: string, props: Record<string, unknown>, dataset: TypedDataSet, editable: boolean, required: boolean): void {
  const isStandalone = ['input', 'select', 'checkbox', 'textarea'].includes(componentType);
  if (isStandalone) {
    if (props.label) (child as any).label = props.label;
    if (props.placeholder) (child as any).placeholder = props.placeholder;
    if (props.maxLength) (child as any).maxlength = props.maxLength;
    (child as any).required = required;
    (child as any).disabled = !editable;
    // Extract and set value
    const field = props.field as string;
    if (field && dataset.rows.length > 0) {
      const row = dataset.rows[0]!;
      try {
        const cell = row.cell(field as ColumnId);
        if (cell.type !== 'NULL') {
          if (componentType === 'checkbox') {
            const v = typeof cell.value === 'boolean' ? cell.value : String(cell.value).toLowerCase() === 'true';
            (child as any).checked = v;
          } else {
            (child as any).value = String(cell.value);
          }
        }
      } catch { /* column not found */ }
    }
    // Handle select options
    if (componentType === 'select' && props.options) {
      const opts = props.options as { values: string[] };
      if (opts.values) {
        (child as any).options = opts.values.map((v: string) => ({ value: v, label: v }));
      }
    }
  } else {
    const formInput = child as unknown as PagesFormInput<any>;
    formInput.props = props;
    formInput.dataSet = dataset;
    formInput.editable = editable;
    formInput.required = required;
  }
}

private getChildValue(child: HTMLElement, componentType: string): unknown {
  const isStandalone = ['input', 'select', 'checkbox', 'textarea'].includes(componentType);
  if (isStandalone) {
    return componentType === 'checkbox' ? (child as any).checked : (child as any).value;
  }
  return (child as unknown as PagesFormInput<any>).currentValue;
}

private setChildError(child: HTMLElement, componentType: string, error: string | undefined): void {
  const isStandalone = ['input', 'select', 'checkbox', 'textarea'].includes(componentType);
  if (isStandalone) {
    (child as any).error = error;
  } else {
    (child as unknown as PagesFormInput<any>).errorMessage = error;
  }
}
```

Update `renderContent()` and `submit()` to use these helpers instead of direct PagesFormInput casts.

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-viz run test -- schema-form`
Expected: PASS

- [ ] **Step 5: Run full build**

Run: `yarn build`
Expected: clean build

- [ ] **Step 6: Commit**

```bash
git add packages/pages-viz/src/form-inputs/
git commit -m "refactor(#233): update PagesSchemaForm for standalone + legacy children

Thin adapter normalizes API between standalone pages-ui-components
(input, select, checkbox, textarea) and legacy PagesFormInput children
(number-input, date-picker).

Refs #233"
```

---

### Task 11: Update pages-runtime form tests

**Files:**
- Modify: `packages/pages-runtime/src/form-activation.test.ts`
- Modify: `packages/pages-runtime/src/form-integration.test.ts`
- Modify: `packages/pages-runtime/src/form-equivalence.test.ts`
- Modify: `packages/pages-runtime/src/form-schema-integration.test.ts`

**Interfaces:**
- Consumes: Tasks 7-9 (type renames and activation changes)
- Produces: Updated test suites using new type and tag names

- [ ] **Step 1: Update tag name references in all form test files**

Across all 4 test files:
- Replace `"pages-text-input"` → `"pages-input"`
- Replace `"pages-dropdown"` → `"pages-select"`
- Replace `"text-input"` (as YAML type) → `"input"`
- Replace `"dropdown"` (as YAML type) → `"select"`

- [ ] **Step 2: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: PASS (or failures to investigate — some tests may need deeper changes due to activation callback restructuring)

- [ ] **Step 3: Fix any remaining test failures**

Adapt tests that relied on PagesElement API (`.props`, `.dataSet`) to use the new standalone API (`.value`, `.label`, `.error`).

- [ ] **Step 4: Commit**

```bash
git add packages/pages-runtime/src/
git commit -m "test(#233): update form tests for type renames and standalone activation

Refs #233"
```

---

### Task 12: Full build verification and examples

**Files:**
- Modify: `examples/` — update any YAML dashboards using old type names

**Interfaces:**
- Consumes: all prior tasks
- Produces: verified end-to-end build

- [ ] **Step 1: Run full build**

Run: `yarn build`
Expected: clean build with no errors

- [ ] **Step 2: Run all tests**

Run: `yarn workspaces foreach -Apt run test`
Expected: all tests pass

- [ ] **Step 3: Run typecheck**

Run: `yarn typecheck`
Expected: no type errors

- [ ] **Step 4: Run lint**

Run: `yarn lint`
Expected: no new lint errors

- [ ] **Step 5: Update example YAML files**

Search examples directory for `type: text-input` and `type: dropdown`, update to `type: input` and `type: select`.

- [ ] **Step 6: Commit**

```bash
git add examples/
git commit -m "chore(#233): update example dashboards for type renames

Refs #233"
```
