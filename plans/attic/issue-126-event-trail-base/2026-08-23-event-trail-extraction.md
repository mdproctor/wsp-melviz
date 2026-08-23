# Event Trail Extraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/blocks-ui#126 — Event trail base extraction
**Issue group:** casehubio/casehub-pages#354 (pages-filter-bar), casehubio/blocks-ui#126 (blocks-event-trail + audit-trail-viewer refactor)

**Goal:** Extract reusable filter bar and event trail components from the 695-line `blocks-audit-trail-viewer`, creating two new components at the correct platform layers.

**Architecture:** Two components at two layers: `pages-filter-bar` (casehub-pages, domain-agnostic UI primitive) and `blocks-event-trail` (blocks-ui, domain-aware composition of filter bar + pages-table). The existing `audit-trail-viewer` is refactored to compose `blocks-event-trail` internally, dropping from ~695 to ~200 lines.

**Tech Stack:** TypeScript, Lit 3, Vitest, Yarn workspaces, `@casehubio/pages-data` (TypedDataSet, fromRows, columnId, ColumnType), `@casehubio/pages-component` (DataSourceMixin, DataSourceAdapter), `@casehubio/pages-primitives` (LiveRegionMixin)

## Global Constraints

- All CSS uses `--pages-*` custom properties from `pages-ui-tokens`
- Chip styling: neutral background, accent when selected, 16px border-radius
- Entity dropdown: custom dropdown (NOT native `<select>`) per ARC42STORIES.MD §6 shadow root positioning bugs
- ARIA is mandatory on every `@customElement` — role, aria-label, state attributes
- Use `portal:` resolutions for cross-repo dev dependencies in blocks-ui
- Package naming: `@casehubio/pages-filter-bar` (pages), component directory `event-trail` (blocks-ui)
- pages-filter-bar depends on `@casehubio/pages-data` (TypedDataSet) and `lit`
- blocks-event-trail depends on `@casehubio/pages-component` (DataSourceMixin), `@casehubio/pages-data` (fromRows), `@casehubio/pages-table`, `@casehubio/pages-filter-bar`, `lit`

---

## Batch 1: pages-filter-bar (casehub-pages #354)

### Task 1: Package scaffold + FilterState types + chip filter

**Files:**
- Create: `packages/pages-filter-bar/package.json`
- Create: `packages/pages-filter-bar/tsconfig.json`
- Create: `packages/pages-filter-bar/tsconfig.build.json`
- Create: `packages/pages-filter-bar/vitest.config.ts`
- Create: `packages/pages-filter-bar/src/types.ts`
- Create: `packages/pages-filter-bar/src/pages-filter-bar.ts`
- Create: `packages/pages-filter-bar/src/index.ts`
- Test: `packages/pages-filter-bar/src/pages-filter-bar.test.ts`

**Interfaces:**
- Consumes: `TypedDataSet`, `ColumnId` from `@casehubio/pages-data`
- Produces: `FilterState` interface, `PagesFilterBar` custom element (`<pages-filter-bar>`), `filter-change` CustomEvent

- [ ] **Step 1: Create package scaffold files**

Create the four config files matching the pages-table pattern:

`packages/pages-filter-bar/package.json`:
```json
{
  "name": "@casehubio/pages-filter-bar",
  "version": "0.1.0",
  "description": "CaseHub Pages filter bar — type chips, entity dropdown, date range",
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
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/pages-data": "workspace:*",
    "@casehubio/pages-ui-tokens": "workspace:*",
    "lit": "^3.0.0"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "jsdom": "^25.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

`packages/pages-filter-bar/tsconfig.json`:
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

`packages/pages-filter-bar/tsconfig.build.json`:
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

`packages/pages-filter-bar/vitest.config.ts`:
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

- [ ] **Step 2: Create FilterState types**

`packages/pages-filter-bar/src/types.ts`:
```typescript
export interface FilterState {
  readonly selectedChips: readonly string[];
  readonly selectedEntity: string | null;
  readonly dateFrom: string;
  readonly dateTo: string;
}

export const EMPTY_FILTER_STATE: FilterState = Object.freeze({
  selectedChips: Object.freeze([] as string[]),
  selectedEntity: null,
  dateFrom: '',
  dateTo: '',
});
```

- [ ] **Step 3: Create index.ts with exports**

`packages/pages-filter-bar/src/index.ts`:
```typescript
export { PagesFilterBar } from './pages-filter-bar.js';
export type { FilterState } from './types.js';
export { EMPTY_FILTER_STATE } from './types.js';
```

- [ ] **Step 4: Write failing tests for chip filter**

`packages/pages-filter-bar/src/pages-filter-bar.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { columnId, ColumnType, fromRows } from '@casehubio/pages-data';
import type { TypedDataSet, ColumnId } from '@casehubio/pages-data';
import type { FilterState } from './types.js';
import './index.js';
import type { PagesFilterBar } from './pages-filter-bar.js';

function createTestDataSet(): TypedDataSet {
  const typeCol = columnId('type');
  const actorCol = columnId('actor');
  return fromRows(
    [
      { type: 'COMMAND', actor: 'alice' },
      { type: 'EVENT', actor: 'bob' },
      { type: 'COMMAND', actor: 'alice' },
      { type: 'ATTESTATION', actor: 'charlie' },
    ],
    [
      { id: typeCol, type: ColumnType.TEXT, getValue: (r: any) => r.type },
      { id: actorCol, type: ColumnType.TEXT, getValue: (r: any) => r.actor },
    ],
  );
}

async function createElement(attrs: Record<string, unknown> = {}): Promise<PagesFilterBar> {
  const el = document.createElement('pages-filter-bar') as PagesFilterBar;
  Object.entries(attrs).forEach(([key, value]) => {
    (el as any)[key] = value;
  });
  document.body.appendChild(el);
  await el.updateComplete;
  return el;
}

describe('PagesFilterBar', () => {
  afterEach(() => {
    document.body.innerHTML = '';
  });

  it('registers as a custom element', () => {
    expect(customElements.get('pages-filter-bar')).toBeDefined();
  });

  describe('chip filter', () => {
    it('renders chips from chipValues', async () => {
      const el = await createElement({
        chipValues: ['COMMAND', 'EVENT', 'ATTESTATION'],
      });
      const chips = el.shadowRoot!.querySelectorAll('[role="checkbox"]');
      expect(chips.length).toBe(3);
      expect(chips[0]?.textContent?.trim()).toBe('COMMAND');
      expect(chips[1]?.textContent?.trim()).toBe('EVENT');
      expect(chips[2]?.textContent?.trim()).toBe('ATTESTATION');
    });

    it('hides chip section when chipField and chipValues are omitted', async () => {
      const el = await createElement();
      const chips = el.shadowRoot!.querySelectorAll('[role="checkbox"]');
      expect(chips.length).toBe(0);
    });

    it('extracts chip values from dataSet when chipField is set', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        chipField: columnId('type'),
      });
      const chips = el.shadowRoot!.querySelectorAll('[role="checkbox"]');
      expect(chips.length).toBe(3);
      const labels = Array.from(chips).map(c => c.textContent?.trim());
      expect(labels).toContain('COMMAND');
      expect(labels).toContain('EVENT');
      expect(labels).toContain('ATTESTATION');
    });

    it('emits filter-change on chip toggle', async () => {
      const el = await createElement({
        chipValues: ['COMMAND', 'EVENT'],
      });
      const handler = vi.fn();
      el.addEventListener('filter-change', handler);

      const chip = el.shadowRoot!.querySelector('[role="checkbox"]') as HTMLElement;
      chip.click();
      await el.updateComplete;

      expect(handler).toHaveBeenCalledOnce();
      const detail = handler.mock.calls[0][0].detail as FilterState;
      expect(detail.selectedChips.includes('COMMAND')).toBe(true);
    });

    it('toggles chip off on second click', async () => {
      const el = await createElement({
        chipValues: ['COMMAND', 'EVENT'],
      });
      const handler = vi.fn();
      el.addEventListener('filter-change', handler);

      const chip = el.shadowRoot!.querySelector('[role="checkbox"]') as HTMLElement;
      chip.click();
      await el.updateComplete;
      chip.click();
      await el.updateComplete;

      expect(handler).toHaveBeenCalledTimes(2);
      const detail = handler.mock.calls[1][0].detail as FilterState;
      expect(detail.selectedChips.includes('COMMAND')).toBe(false);
    });

    it('sets aria-checked on selected chips', async () => {
      const el = await createElement({
        chipValues: ['COMMAND', 'EVENT'],
      });
      const chip = el.shadowRoot!.querySelector('[role="checkbox"]') as HTMLElement;
      expect(chip.getAttribute('aria-checked')).toBe('false');

      chip.click();
      await el.updateComplete;
      expect(chip.getAttribute('aria-checked')).toBe('true');
    });
  });

  describe('ARIA', () => {
    it('has role="toolbar" on host container', async () => {
      const el = await createElement({ chipValues: ['A'] });
      const toolbar = el.shadowRoot!.querySelector('[role="toolbar"]');
      expect(toolbar).not.toBeNull();
    });

    it('has aria-label="Filters" on toolbar', async () => {
      const el = await createElement({ chipValues: ['A'] });
      const toolbar = el.shadowRoot!.querySelector('[role="toolbar"]');
      expect(toolbar?.getAttribute('aria-label')).toBe('Filters');
    });
  });
});
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-filter-bar run test`
Expected: FAIL — module `./pages-filter-bar.js` not found

- [ ] **Step 6: Implement pages-filter-bar with chip filter**

`packages/pages-filter-bar/src/pages-filter-bar.ts`:
```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import type { TypedDataSet, ColumnId } from '@casehubio/pages-data';
import type { FilterState } from './types.js';

@customElement('pages-filter-bar')
export class PagesFilterBar extends LitElement {
  @property({ type: Object }) dataSet?: TypedDataSet;
  @property({ type: String }) chipField?: ColumnId;
  @property({ type: Array }) chipValues?: string[];
  @property({ type: String }) entityField?: ColumnId;
  @property({ type: String }) entityLabel = '';
  @property({ type: Boolean }) showDateRange = false;
  @property({ type: String }) dateFromLabel = 'From';
  @property({ type: String }) dateToLabel = 'To';

  @state() private _selectedChips: readonly string[] = [];
  @state() private _selectedEntity: string | null = null;
  @state() private _dateFrom = '';
  @state() private _dateTo = '';
  @state() private _dropdownOpen = false;
  @state() private _focusedIndex = -1;

  static override styles = css`
    :host { display: block; }
    .filter-toolbar {
      display: flex;
      gap: var(--pages-space-4, 16px);
      padding: var(--pages-space-3, 12px);
      background: var(--pages-neutral-2, #f8f9fa);
      border-radius: var(--pages-radius-1, 4px);
      flex-wrap: wrap;
      align-items: center;
    }
    .filter-section {
      display: flex;
      align-items: center;
      gap: var(--pages-space-2, 8px);
    }
    .chip {
      padding: var(--pages-space-1, 4px) var(--pages-space-3, 12px);
      border: 1px solid var(--pages-neutral-6, #d1d5db);
      border-radius: 16px;
      background: var(--pages-neutral-1, #fff);
      color: var(--pages-neutral-12, #111);
      cursor: pointer;
      font-size: 13px;
      font-weight: 500;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      transition: all 0.2s;
    }
    .chip[aria-checked="true"] {
      background: var(--pages-accent-9, #2563eb);
      color: white;
      border-color: var(--pages-accent-9, #2563eb);
    }
    .chip:hover { border-color: var(--pages-accent-7, #3b82f6); }
    .chip[aria-checked="true"]:hover { background: var(--pages-accent-10, #1d4ed8); }
    .dropdown-wrapper { position: relative; }
    .dropdown-trigger {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border: 1px solid var(--pages-neutral-5, #d4d4d4);
      border-radius: var(--pages-radius-1, 4px);
      background: var(--pages-neutral-1, #fff);
      color: var(--pages-neutral-12, #1a1a1a);
      font-size: 14px;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      cursor: pointer;
      text-align: left;
      display: flex;
      align-items: center;
      gap: var(--pages-space-2, 8px);
      min-width: 120px;
    }
    .dropdown-trigger:hover { border-color: var(--pages-neutral-7, #a3a3a3); }
    .dropdown-arrow { font-size: 10px; color: var(--pages-neutral-8, #888); margin-left: auto; }
    .dropdown-panel {
      position: absolute;
      top: 100%;
      left: 0;
      right: 0;
      margin-top: 2px;
      background: var(--pages-neutral-1, #fff);
      border: 1px solid var(--pages-neutral-5, #d4d4d4);
      border-radius: var(--pages-radius-1, 4px);
      box-shadow: var(--pages-shadow-3, 0 4px 12px rgba(0,0,0,0.1));
      z-index: 10;
      max-height: 200px;
      overflow-y: auto;
      list-style: none;
      margin: 2px 0 0;
      padding: var(--pages-space-1, 4px);
    }
    .dropdown-option {
      padding: var(--pages-space-2, 8px);
      cursor: pointer;
      border-radius: var(--pages-radius-1, 4px);
      font-size: 14px;
    }
    .dropdown-option:hover { background: var(--pages-neutral-3, #f5f5f5); }
    .dropdown-option.selected { background: var(--pages-accent-3, #e0f2fe); }
    .dropdown-option.focused {
      outline: 2px solid var(--pages-accent-7, #818cf8);
      outline-offset: -2px;
    }
    .filter-label {
      font-weight: 500;
      font-size: 14px;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      color: var(--pages-neutral-11, #333);
    }
    input[type="date"] {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border: 1px solid var(--pages-neutral-5, #d1d5db);
      border-radius: var(--pages-radius-1, 4px);
      font-size: 14px;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      background: var(--pages-neutral-1, #fff);
      color: var(--pages-neutral-12, #111);
    }
  `;

  private get _resolvedChipValues(): string[] {
    if (this.chipValues) return this.chipValues;
    if (this.chipField && this.dataSet) {
      const unique = new Set<string>();
      for (const row of this.dataSet.rows) {
        const val = row.text(this.chipField);
        if (val) unique.add(val);
      }
      return [...unique].sort();
    }
    return [];
  }

  private get _resolvedEntityValues(): string[] {
    if (this.entityField && this.dataSet) {
      const unique = new Set<string>();
      for (const row of this.dataSet.rows) {
        const val = row.text(this.entityField);
        if (val) unique.add(val);
      }
      return [...unique].sort();
    }
    return [];
  }

  private get _resolvedEntityLabel(): string {
    if (this.entityLabel) return this.entityLabel;
    if (this.entityField && this.dataSet) {
      const col = this.dataSet.columns.find(c => c.id === this.entityField);
      if (col) return col.name;
    }
    return 'Entity';
  }

  private _emitFilterChange(): void {
    this.dispatchEvent(new CustomEvent<FilterState>('filter-change', {
      bubbles: true,
      composed: true,
      detail: {
        selectedChips: [...this._selectedChips],
        selectedEntity: this._selectedEntity,
        dateFrom: this._dateFrom,
        dateTo: this._dateTo,
      },
    }));
  }

  private _handleChipClick(value: string): void {
    const idx = this._selectedChips.indexOf(value);
    this._selectedChips = idx >= 0
      ? this._selectedChips.filter(v => v !== value)
      : [...this._selectedChips, value];
    this._emitFilterChange();
  }

  private _toggleDropdown(): void {
    this._dropdownOpen = !this._dropdownOpen;
    if (this._dropdownOpen) {
      const entities = this._resolvedEntityValues;
      const currentIdx = this._selectedEntity
        ? entities.indexOf(this._selectedEntity) + 1
        : 0;
      this._focusedIndex = currentIdx;
      document.addEventListener('click', this._closeDropdown);
    } else {
      document.removeEventListener('click', this._closeDropdown);
    }
  }

  private _closeDropdown = (): void => {
    this._dropdownOpen = false;
    this._focusedIndex = -1;
    document.removeEventListener('click', this._closeDropdown);
  };

  private _selectEntity(value: string | null): void {
    this._selectedEntity = value;
    this._dropdownOpen = false;
    document.removeEventListener('click', this._closeDropdown);
    this._emitFilterChange();
  }

  private _handleDropdownKeyDown(event: KeyboardEvent): void {
    const options = [null, ...this._resolvedEntityValues];
    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        if (!this._dropdownOpen) { this._toggleDropdown(); return; }
        this._focusedIndex = Math.min(this._focusedIndex + 1, options.length - 1);
        break;
      case 'ArrowUp':
        event.preventDefault();
        this._focusedIndex = Math.max(this._focusedIndex - 1, 0);
        break;
      case 'Enter':
        event.preventDefault();
        if (this._dropdownOpen) {
          this._selectEntity(options[this._focusedIndex] ?? null);
        } else {
          this._toggleDropdown();
        }
        break;
      case 'Escape':
        if (this._dropdownOpen) {
          event.preventDefault();
          this._closeDropdown();
        }
        break;
    }
  }

  private _handleDateFromChange(e: Event): void {
    this._dateFrom = (e.target as HTMLInputElement).value;
    this._emitFilterChange();
  }

  private _handleDateToChange(e: Event): void {
    this._dateTo = (e.target as HTMLInputElement).value;
    this._emitFilterChange();
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    document.removeEventListener('click', this._closeDropdown);
  }

  private _renderChips() {
    const values = this._resolvedChipValues;
    if (values.length === 0) return nothing;
    return html`
      <div class="filter-section" role="group" aria-label="Type filter">
        ${values.map(val => html`
          <button class="chip"
            role="checkbox"
            aria-checked="${this._selectedChips.includes(val)}"
            @click=${() => this._handleChipClick(val)}
          >${val}</button>
        `)}
      </div>
    `;
  }

  private _renderEntityDropdown() {
    if (!this.entityField) return nothing;
    const entities = this._resolvedEntityValues;
    const label = this._resolvedEntityLabel;
    const allLabel = `All ${label}s`;
    const triggerText = this._selectedEntity ?? allLabel;
    const options = [null, ...entities];

    return html`
      <div class="filter-section">
        <span class="filter-label">${label}:</span>
        <div class="dropdown-wrapper" @click=${(e: Event) => e.stopPropagation()}>
          <button class="dropdown-trigger"
            role="combobox"
            aria-expanded="${this._dropdownOpen}"
            aria-haspopup="listbox"
            aria-label="${label} filter"
            aria-activedescendant="${this._dropdownOpen && this._focusedIndex >= 0 ? `entity-option-${this._focusedIndex}` : ''}"
            @click=${() => this._toggleDropdown()}
            @keydown=${this._handleDropdownKeyDown}>
            <span>${triggerText}</span>
            <span class="dropdown-arrow">${this._dropdownOpen ? '▲' : '▼'}</span>
          </button>
          ${this._dropdownOpen ? html`
            <ul class="dropdown-panel" role="listbox" aria-label="${label} options">
              ${options.map((entity, index) => html`
                <li class="dropdown-option ${entity === this._selectedEntity ? 'selected' : ''} ${index === this._focusedIndex ? 'focused' : ''}"
                  role="option"
                  aria-selected="${entity === this._selectedEntity}"
                  id="entity-option-${index}"
                  @click=${() => this._selectEntity(entity)}>
                  ${entity ?? allLabel}
                </li>
              `)}
            </ul>
          ` : nothing}
        </div>
      </div>
    `;
  }

  private _renderDateRange() {
    if (!this.showDateRange) return nothing;
    return html`
      <div class="filter-section">
        <label class="filter-label" for="filter-date-from">${this.dateFromLabel}:</label>
        <input id="filter-date-from" type="date" .value=${this._dateFrom}
          @change=${this._handleDateFromChange} />
        <label class="filter-label" for="filter-date-to">${this.dateToLabel}:</label>
        <input id="filter-date-to" type="date" .value=${this._dateTo}
          @change=${this._handleDateToChange} />
      </div>
    `;
  }

  override render() {
    const hasContent = this._resolvedChipValues.length > 0
      || this.entityField
      || this.showDateRange;
    if (!hasContent) return nothing;

    return html`
      <div class="filter-toolbar" role="toolbar" aria-label="Filters">
        ${this._renderChips()}
        ${this._renderEntityDropdown()}
        ${this._renderDateRange()}
      </div>
    `;
  }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-filter-bar run test`
Expected: All chip and ARIA tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/151/pages add packages/pages-filter-bar/
git -C /Users/mdproctor/claude/casehub/slots/151/pages commit -m "feat(pages-filter-bar): chip filter with ARIA checkbox pattern Refs casehubio/casehub-pages#354"
```

### Task 2: Entity dropdown with keyboard navigation tests

**Files:**
- Modify: `packages/pages-filter-bar/src/pages-filter-bar.test.ts` (add dropdown tests)

**Interfaces:**
- Consumes: `PagesFilterBar` from Task 1, `FilterState` from Task 1
- Produces: Validated entity dropdown behavior (keyboard navigation, outside-click dismiss, aria-expanded)

- [ ] **Step 1: Write failing tests for entity dropdown**

Add to `pages-filter-bar.test.ts`:
```typescript
  describe('entity dropdown', () => {
    it('renders entity dropdown when entityField is set', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
        entityLabel: 'Actor',
      });
      const trigger = el.shadowRoot!.querySelector('[role="combobox"]');
      expect(trigger).not.toBeNull();
      expect(trigger?.textContent).toContain('All Actors');
    });

    it('hides dropdown when entityField is omitted', async () => {
      const el = await createElement({ chipValues: ['A'] });
      const trigger = el.shadowRoot!.querySelector('[role="combobox"]');
      expect(trigger).toBeNull();
    });

    it('opens dropdown on click', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
        entityLabel: 'Actor',
      });
      const trigger = el.shadowRoot!.querySelector('[role="combobox"]') as HTMLElement;
      trigger.click();
      await el.updateComplete;

      const panel = el.shadowRoot!.querySelector('[role="listbox"]');
      expect(panel).not.toBeNull();
      expect(trigger.getAttribute('aria-expanded')).toBe('true');
    });

    it('shows unique entity values as options', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
        entityLabel: 'Actor',
      });
      const trigger = el.shadowRoot!.querySelector('[role="combobox"]') as HTMLElement;
      trigger.click();
      await el.updateComplete;

      const options = el.shadowRoot!.querySelectorAll('[role="option"]');
      expect(options.length).toBe(4); // "All Actors" + alice, bob, charlie
    });

    it('emits filter-change on entity selection', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
        entityLabel: 'Actor',
      });
      const handler = vi.fn();
      el.addEventListener('filter-change', handler);

      const trigger = el.shadowRoot!.querySelector('[role="combobox"]') as HTMLElement;
      trigger.click();
      await el.updateComplete;

      const options = el.shadowRoot!.querySelectorAll('[role="option"]');
      (options[1] as HTMLElement).click(); // select "alice"
      await el.updateComplete;

      expect(handler).toHaveBeenCalledOnce();
      const detail = handler.mock.calls[0][0].detail as FilterState;
      expect(detail.selectedEntity).toBe('alice');
    });

    it('navigates with ArrowDown/ArrowUp keys', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
        entityLabel: 'Actor',
      });
      const trigger = el.shadowRoot!.querySelector('[role="combobox"]') as HTMLElement;
      trigger.click();
      await el.updateComplete;

      trigger.dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowDown', bubbles: true }));
      await el.updateComplete;

      const focused = el.shadowRoot!.querySelector('.dropdown-option.focused');
      expect(focused).not.toBeNull();
    });

    it('selects with Enter key', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
        entityLabel: 'Actor',
      });
      const handler = vi.fn();
      el.addEventListener('filter-change', handler);

      const trigger = el.shadowRoot!.querySelector('[role="combobox"]') as HTMLElement;
      trigger.click();
      await el.updateComplete;

      trigger.dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowDown', bubbles: true }));
      await el.updateComplete;
      trigger.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', bubbles: true }));
      await el.updateComplete;

      expect(handler).toHaveBeenCalled();
    });

    it('closes with Escape key', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
        entityLabel: 'Actor',
      });
      const trigger = el.shadowRoot!.querySelector('[role="combobox"]') as HTMLElement;
      trigger.click();
      await el.updateComplete;
      expect(el.shadowRoot!.querySelector('[role="listbox"]')).not.toBeNull();

      trigger.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));
      await el.updateComplete;
      expect(el.shadowRoot!.querySelector('[role="listbox"]')).toBeNull();
    });

    it('uses column name as label when entityLabel omitted', async () => {
      const ds = createTestDataSet();
      const el = await createElement({
        dataSet: ds,
        entityField: columnId('actor'),
      });
      const label = el.shadowRoot!.querySelector('.filter-label');
      expect(label?.textContent?.trim()).toBe('actor:');
    });
  });
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-filter-bar run test`
Expected: All tests PASS (implementation already written in Task 1)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/151/pages add packages/pages-filter-bar/src/pages-filter-bar.test.ts
git -C /Users/mdproctor/claude/casehub/slots/151/pages commit -m "test(pages-filter-bar): entity dropdown keyboard navigation tests Refs casehubio/casehub-pages#354"
```

### Task 3: Date range tests + build integration

**Files:**
- Modify: `packages/pages-filter-bar/src/pages-filter-bar.test.ts` (add date range tests)
- Modify: `package.json` (root — add pages-filter-bar to `build:packages` script)

**Interfaces:**
- Consumes: `PagesFilterBar` from Task 1
- Produces: Complete, tested `@casehubio/pages-filter-bar` package ready for consumption

- [ ] **Step 1: Write date range and visibility tests**

Add to `pages-filter-bar.test.ts`:
```typescript
  describe('date range', () => {
    it('renders date inputs when showDateRange=true', async () => {
      const el = await createElement({ showDateRange: true });
      const inputs = el.shadowRoot!.querySelectorAll('input[type="date"]');
      expect(inputs.length).toBe(2);
    });

    it('hides date inputs when showDateRange=false', async () => {
      const el = await createElement({ chipValues: ['A'] });
      const inputs = el.shadowRoot!.querySelectorAll('input[type="date"]');
      expect(inputs.length).toBe(0);
    });

    it('emits filter-change on date from change', async () => {
      const el = await createElement({ showDateRange: true });
      const handler = vi.fn();
      el.addEventListener('filter-change', handler);

      const input = el.shadowRoot!.querySelector('#filter-date-from') as HTMLInputElement;
      input.value = '2026-01-01';
      input.dispatchEvent(new Event('change'));
      await el.updateComplete;

      expect(handler).toHaveBeenCalledOnce();
      const detail = handler.mock.calls[0][0].detail as FilterState;
      expect(detail.dateFrom).toBe('2026-01-01');
    });

    it('emits filter-change on date to change', async () => {
      const el = await createElement({ showDateRange: true });
      const handler = vi.fn();
      el.addEventListener('filter-change', handler);

      const input = el.shadowRoot!.querySelector('#filter-date-to') as HTMLInputElement;
      input.value = '2026-12-31';
      input.dispatchEvent(new Event('change'));
      await el.updateComplete;

      expect(handler).toHaveBeenCalledOnce();
      const detail = handler.mock.calls[0][0].detail as FilterState;
      expect(detail.dateTo).toBe('2026-12-31');
    });

    it('uses custom date labels', async () => {
      const el = await createElement({
        showDateRange: true,
        dateFromLabel: 'Start',
        dateToLabel: 'End',
      });
      const labels = el.shadowRoot!.querySelectorAll('.filter-label');
      const texts = Array.from(labels).map(l => l.textContent?.trim());
      expect(texts).toContain('Start:');
      expect(texts).toContain('End:');
    });
  });

  describe('visibility', () => {
    it('renders nothing when no filter properties are set', async () => {
      const el = await createElement();
      const toolbar = el.shadowRoot!.querySelector('[role="toolbar"]');
      expect(toolbar).toBeNull();
    });

    it('renders toolbar when only chipValues set', async () => {
      const el = await createElement({ chipValues: ['A'] });
      const toolbar = el.shadowRoot!.querySelector('[role="toolbar"]');
      expect(toolbar).not.toBeNull();
    });

    it('renders toolbar when only showDateRange is true', async () => {
      const el = await createElement({ showDateRange: true });
      const toolbar = el.shadowRoot!.querySelector('[role="toolbar"]');
      expect(toolbar).not.toBeNull();
    });
  });
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-filter-bar run test`
Expected: All tests PASS

- [ ] **Step 3: Add pages-filter-bar to root build:packages script**

In root `package.json`, add `yarn workspace @casehubio/pages-filter-bar run build` to the `build:packages` script. Insert it after `pages-table` (since it has similar dependencies):

```
... && yarn workspace @casehubio/pages-table run build && yarn workspace @casehubio/pages-filter-bar run build && yarn workspace @casehubio/pages-runtime run build
```

- [ ] **Step 4: Verify full build**

Run: `yarn workspace @casehubio/pages-filter-bar run build`
Expected: TypeScript compiles to `dist/` without errors

Run: `yarn workspace @casehubio/pages-filter-bar run typecheck`
Expected: No type errors

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/151/pages add packages/pages-filter-bar/ package.json
git -C /Users/mdproctor/claude/casehub/slots/151/pages commit -m "feat(pages-filter-bar): complete filter bar with date range and build integration Closes casehubio/casehub-pages#354"
```

---

## Batch 2: blocks-event-trail (blocks-ui #126)

### Task 4: Cross-repo wiring + blocks-event-trail component

**Files:**
- Create: `components/event-trail/package.json`
- Create: `components/event-trail/tsconfig.json`
- Create: `components/event-trail/tsconfig.build.json`
- Create: `components/event-trail/vitest.config.ts`
- Create: `components/event-trail/src/event-trail.ts`
- Create: `components/event-trail/src/index.ts`
- Modify: `package.json` (root — add portal resolution for `@casehubio/pages-filter-bar`)
- Test: `components/event-trail/src/event-trail.test.ts`

**Interfaces:**
- Consumes: `DataSourceMixin` from `@casehubio/pages-component`, `fromRows`/`TypedDataSet`/`ColumnId`/`ColumnType` from `@casehubio/pages-data`, `PagesFilterBar`/`FilterState` from `@casehubio/pages-filter-bar`, `pages-table` from `@casehubio/pages-table`
- Produces: `BlocksEventTrail` custom element (`<blocks-event-trail>`) with `endpoint`, `data`, `columnDefs`, `columnConfig`, `columnRenderers`, `chipField`, `chipValues`, `entityField`, `entityLabel`, `showDateRange`, `getRowDetail`, `getRowKey` properties. Emits `detail-change` event.

- [ ] **Step 1: Publish pages SNAPSHOT and wire cross-repo dependency**

Build pages-filter-bar and publish to local Maven repo:
```bash
yarn --cwd /Users/mdproctor/claude/casehub/slots/151/pages workspace @casehubio/pages-filter-bar run build
```

Copy built pages-filter-bar to blocks-ui `.casehub-packages`:
```bash
cp -r /Users/mdproctor/claude/casehub/slots/151/pages/packages/pages-filter-bar /Users/mdproctor/claude/casehub/slots/151/blocks-ui/.casehub-packages/packages/pages-filter-bar
```

Add portal resolution to blocks-ui root `package.json` in the `resolutions` block:
```json
"@casehubio/pages-filter-bar": "portal:./.casehub-packages/packages/pages-filter-bar"
```

Run: `yarn --cwd /Users/mdproctor/claude/casehub/slots/151/blocks-ui install`

- [ ] **Step 2: Create package scaffold**

`components/event-trail/package.json`:
```json
{
  "name": "@casehubio/blocks-ui-event-trail",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc --project tsconfig.build.json",
    "test": "vitest run"
  },
  "dependencies": {
    "@casehubio/pages-component": "*",
    "@casehubio/pages-data": "*",
    "@casehubio/pages-filter-bar": "*",
    "@casehubio/pages-primitives": "*",
    "@casehubio/pages-table": "*",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "jsdom": "^26.0.0",
    "typescript": "~5.7.2",
    "vitest": "^2.1.8"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

`components/event-trail/tsconfig.json`, `tsconfig.build.json`, `vitest.config.ts` — same pattern as audit-trail-viewer:

`components/event-trail/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "declaration": true,
    "strict": true,
    "experimentalDecorators": true,
    "useDefineForClassFields": false,
    "rootDir": "src",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

`components/event-trail/tsconfig.build.json`:
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

`components/event-trail/vitest.config.ts`:
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

`components/event-trail/src/index.ts`:
```typescript
export { BlocksEventTrail } from './event-trail.js';
```

- [ ] **Step 3: Write failing tests**

`components/event-trail/src/event-trail.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { columnId, ColumnType, fromRows } from '@casehubio/pages-data';
import type { TypedDataSet, ColumnId, TypedRow } from '@casehubio/pages-data';
import type { TableColumnConfig, ColumnRenderer } from '@casehubio/pages-table';
import type { TemplateResult } from 'lit';
import { html } from 'lit';
import './index.js';
import type { BlocksEventTrail } from './event-trail.js';

const TYPE_COL = columnId('type');
const ACTOR_COL = columnId('actor');
const MSG_COL = columnId('message');
const ID_COL = columnId('id');

const TEST_COL_DEFS = [
  { id: ID_COL, type: ColumnType.TEXT, getValue: (r: any) => r.id },
  { id: TYPE_COL, name: 'Type', type: ColumnType.TEXT, getValue: (r: any) => r.type },
  { id: ACTOR_COL, name: 'Actor', type: ColumnType.TEXT, getValue: (r: any) => r.actor },
  { id: MSG_COL, name: 'Message', type: ColumnType.TEXT, getValue: (r: any) => r.message },
] as const;

const TEST_COL_CONFIG: TableColumnConfig[] = [
  { id: ID_COL, visible: false },
  { id: TYPE_COL, sortable: true },
  { id: ACTOR_COL, sortable: true },
  { id: MSG_COL, sortable: false },
];

const TEST_DATA = [
  { id: '1', type: 'COMMAND', actor: 'alice', message: 'Created case' },
  { id: '2', type: 'EVENT', actor: 'bob', message: 'Status changed' },
  { id: '3', type: 'COMMAND', actor: 'alice', message: 'Updated field' },
];

async function createElement(attrs: Record<string, unknown> = {}): Promise<BlocksEventTrail> {
  const el = document.createElement('blocks-event-trail') as BlocksEventTrail;
  Object.entries(attrs).forEach(([key, value]) => {
    (el as any)[key] = value;
  });
  document.body.appendChild(el);
  await el.updateComplete;
  return el;
}

describe('BlocksEventTrail', () => {
  afterEach(() => {
    document.body.innerHTML = '';
  });

  it('registers as a custom element', () => {
    expect(customElements.get('blocks-event-trail')).toBeDefined();
  });

  describe('dual data mode', () => {
    it('renders table from data property', async () => {
      const el = await createElement({
        data: TEST_DATA,
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
      });
      const table = el.shadowRoot!.querySelector('pages-table');
      expect(table).not.toBeNull();
      expect((table as any).dataSet).toBeDefined();
      expect((table as any).dataSet.rows.length).toBe(3);
    });
  });

  describe('filter integration', () => {
    it('renders pages-filter-bar when chipField is set', async () => {
      const el = await createElement({
        data: TEST_DATA,
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
        chipField: TYPE_COL,
      });
      const filterBar = el.shadowRoot!.querySelector('pages-filter-bar');
      expect(filterBar).not.toBeNull();
    });

    it('applies chip filter to table data', async () => {
      const el = await createElement({
        data: TEST_DATA,
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
        chipField: TYPE_COL,
      });
      const filterBar = el.shadowRoot!.querySelector('pages-filter-bar') as HTMLElement;
      filterBar.dispatchEvent(new CustomEvent('filter-change', {
        bubbles: true,
        composed: true,
        detail: {
          selectedChips: ['EVENT'],
          selectedEntity: null,
          dateFrom: '',
          dateTo: '',
        },
      }));
      await el.updateComplete;

      const table = el.shadowRoot!.querySelector('pages-table') as any;
      expect(table.dataSet.rows.length).toBe(1);
    });

    it('applies entity filter to table data', async () => {
      const el = await createElement({
        data: TEST_DATA,
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
        entityField: ACTOR_COL,
      });
      const filterBar = el.shadowRoot!.querySelector('pages-filter-bar') as HTMLElement;
      filterBar.dispatchEvent(new CustomEvent('filter-change', {
        bubbles: true,
        composed: true,
        detail: {
          selectedChips: [],
          selectedEntity: 'alice',
          dateFrom: '',
          dateTo: '',
        },
      }));
      await el.updateComplete;

      const table = el.shadowRoot!.querySelector('pages-table') as any;
      expect(table.dataSet.rows.length).toBe(2);
    });
  });

  describe('detail expansion', () => {
    it('forwards getRowDetail to pages-table', async () => {
      const getRowDetail = (row: TypedRow): TemplateResult | undefined => {
        return html`<div class="test-detail">${row.text(MSG_COL)}</div>`;
      };
      const el = await createElement({
        data: TEST_DATA,
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
        getRowDetail,
        getRowKey: (row: TypedRow) => row.text(ID_COL),
      });
      const table = el.shadowRoot!.querySelector('pages-table') as any;
      expect(table.getRowDetail).toBe(getRowDetail);
      expect(table.getRowKey).toBeDefined();
    });
  });

  describe('configure()', () => {
    it('sets properties and triggers refresh', async () => {
      const el = await createElement({
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
      });
      el.configure({
        data: TEST_DATA,
      });
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;

      const table = el.shadowRoot!.querySelector('pages-table') as any;
      expect(table.dataSet).toBeDefined();
    });
  });

  describe('ARIA', () => {
    it('has role="region" on host', async () => {
      const el = await createElement({
        data: TEST_DATA,
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
      });
      expect(el.getAttribute('role')).toBe('region');
    });

    it('has aria-label="Event trail"', async () => {
      const el = await createElement({
        data: TEST_DATA,
        columnDefs: TEST_COL_DEFS,
        columnConfig: TEST_COL_CONFIG,
      });
      expect(el.getAttribute('aria-label')).toBe('Event trail');
    });
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-event-trail run test`
Expected: FAIL — module `./event-trail.js` not found

- [ ] **Step 5: Implement blocks-event-trail component**

`components/event-trail/src/event-trail.ts`:
```typescript
import { LitElement, html, css, nothing, type PropertyValues, type TemplateResult } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { DataSourceMixin } from '@casehubio/pages-component';
import { fromRows } from '@casehubio/pages-data';
import type { TypedDataSet, TypedRow, ColumnId, ColumnType } from '@casehubio/pages-data';
import type { TableColumnConfig, ColumnRenderer } from '@casehubio/pages-table';
import { EMPTY_FILTER_STATE, type FilterState } from '@casehubio/pages-filter-bar';
import type { DataSource, DataSink } from '@casehubio/pages-data';
import '@casehubio/pages-table';
import '@casehubio/pages-filter-bar';

type ColDef = Parameters<typeof fromRows>[1][number];

@customElement('blocks-event-trail')
export class BlocksEventTrail extends DataSourceMixin(LitElement) {
  @property({ type: Array }) data?: unknown[];
  @property({ type: Array }) columnDefs: readonly ColDef[] = [];
  @property({ type: Array }) columnConfig?: TableColumnConfig[];
  @property({ type: Object }) columnRenderers?: ReadonlyMap<ColumnId, ColumnRenderer>;
  @property({ type: String }) chipField?: ColumnId;
  @property({ type: Array }) chipValues?: string[];
  @property({ type: String }) entityField?: ColumnId;
  @property({ type: String }) entityLabel?: string;
  @property({ type: Boolean }) showDateRange = false;
  @property({ type: Object }) getRowDetail?: (row: TypedRow) => TemplateResult | undefined;
  @property({ type: Object }) getRowKey?: (row: TypedRow) => string;

  @state() private _rawEntries: unknown[] = [];
  @state() private _filterState: FilterState = EMPTY_FILTER_STATE;
  @state() private _filteredDataSet?: TypedDataSet;
  @state() private _expandedKey: string | null = null;

  override connectedCallback(): void {
    super.connectedCallback();
    this.setAttribute('role', 'region');
    this.setAttribute('aria-label', 'Event trail');
  }

  override createSourceFactory() {
    return (url: string) => {
      let abort: AbortController | undefined;
      return {
        connect: (sink: DataSink) => {
          abort = new AbortController();
          const signal = abort.signal;
          globalThis.fetch(url, { signal })
            .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
            .then((entries: unknown[]) => {
              if (signal.aborted) return;
              this._rawEntries = entries;
              const dataset = fromRows(entries, this.columnDefs);
              sink.apply({ type: 'snapshot', dataset });
              this._applyFilters();
              this.dispatchEvent(new CustomEvent('data-loaded', {
                bubbles: true, composed: true,
                detail: { entries },
              }));
            })
            .catch(err => {
              if (signal.aborted || err.name === 'AbortError') return;
              sink.error({ message: err instanceof Error ? err.message : String(err), permanent: true });
            });
        },
        disconnect: () => { abort?.abort(); abort = undefined; },
      } as DataSource;
    };
  }

  override resolveEndpoint(): string | undefined {
    if (this.data) return undefined;
    if (!this.endpoint) return undefined;
    const url = new URL(this.endpoint, globalThis.location?.origin ?? 'http://localhost');
    if (this._filterState.dateFrom) url.searchParams.set('from', this._filterState.dateFrom);
    if (this._filterState.dateTo) url.searchParams.set('to', this._filterState.dateTo);
    return url.toString();
  }

  override willUpdate(changed: PropertyValues): void {
    super.willUpdate(changed);
    if (changed.has('data') && this.data) {
      this._rawEntries = this.data;
      this.dataSet = fromRows(this.data, this.columnDefs);
      this._applyFilters();
      this.dispatchEvent(new CustomEvent('data-loaded', {
        bubbles: true, composed: true,
        detail: { entries: this.data },
      }));
    }
  }

  override configure(props: Record<string, unknown>): void {
    if (props.data !== undefined) this.data = props.data as unknown[];
    if (props.columnDefs !== undefined) this.columnDefs = props.columnDefs as ColDef[];
    if (props.columnConfig !== undefined) this.columnConfig = props.columnConfig as TableColumnConfig[];
    if (props.endpoint !== undefined) this.endpoint = props.endpoint as string;
    if (props.chipField !== undefined) this.chipField = props.chipField as ColumnId;
    if (props.chipValues !== undefined) this.chipValues = props.chipValues as string[];
    if (props.entityField !== undefined) this.entityField = props.entityField as ColumnId;
    if (props.entityLabel !== undefined) this.entityLabel = props.entityLabel as string;
    if (props.showDateRange !== undefined) this.showDateRange = props.showDateRange as boolean;
    super.configure(props);
  }

  private _handleFilterChange(e: CustomEvent<FilterState>): void {
    const prev = this._filterState;
    this._filterState = e.detail;
    if (prev.dateFrom !== e.detail.dateFrom || prev.dateTo !== e.detail.dateTo) {
      this.syncEndpoint();
    }
    this._applyFilters();
  }

  private _applyFilters(): void {
    const chipGetter = this.chipField
      ? this.columnDefs.find(c => c.id === this.chipField)?.getValue
      : undefined;
    const entityGetter = this.entityField
      ? this.columnDefs.find(c => c.id === this.entityField)?.getValue
      : undefined;

    const filtered = this._rawEntries.filter(entry => {
      if (this._filterState.selectedChips.length > 0 && chipGetter) {
        if (!this._filterState.selectedChips.includes(String(chipGetter(entry)))) return false;
      }
      if (this._filterState.selectedEntity && entityGetter) {
        if (String(entityGetter(entry)) !== this._filterState.selectedEntity) return false;
      }
      return true;
    });
    this._filteredDataSet = fromRows(filtered, this.columnDefs);
  }

  private _handleDetailChange(e: CustomEvent): void {
    const { key, expanded } = e.detail as { key: string; expanded: boolean };
    this._expandedKey = expanded ? key : null;
    this.dispatchEvent(new CustomEvent('detail-change', {
      bubbles: true,
      composed: true,
      detail: e.detail,
    }));
  }

  static override styles = css`
    :host { display: block; }
    .loading {
      padding: var(--pages-space-4, 16px);
      color: var(--pages-neutral-9, #999);
      font-family: var(--pages-font-family, system-ui);
    }
    .error {
      padding: var(--pages-space-4, 16px);
      color: var(--pages-danger-11, #b91c1c);
      font-family: var(--pages-font-family, system-ui);
    }
    pages-filter-bar { margin-bottom: var(--pages-space-3, 12px); }
  `;

  override render() {
    if (this.loading) return html`<div class="loading" aria-busy="true">Loading...</div>`;
    if (this.error) return html`<div class="error" role="alert">
      <p>${this.error}</p>
      <button @click=${() => this.syncEndpoint()}>Retry</button>
    </div>`;

    const hasFilters = this.chipField || this.chipValues || this.entityField || this.showDateRange;
    const activeDataSet = this._filteredDataSet ?? this.dataSet;

    return html`
      ${hasFilters ? html`
        <pages-filter-bar
          .dataSet=${this.dataSet}
          .chipField=${this.chipField}
          .chipValues=${this.chipValues}
          .entityField=${this.entityField}
          .entityLabel=${this.entityLabel ?? ''}
          ?showDateRange=${this.showDateRange}
          @filter-change=${this._handleFilterChange}
        ></pages-filter-bar>
      ` : nothing}
      <pages-table
        .dataSet=${activeDataSet}
        .columnConfig=${this.columnConfig}
        .columnRenderers=${this.columnRenderers}
        .getRowKey=${this.getRowKey}
        .getRowDetail=${this.getRowDetail}
        detailMode="single"
        .expandedDetailKeys=${this._expandedKey ? [this._expandedKey] : []}
        client-sort
        client-filter
        @detail-change=${this._handleDetailChange}
      ></pages-table>
    `;
  }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehubio/blocks-ui-event-trail run test`
Expected: All tests PASS

- [ ] **Step 7: Verify build**

Run: `yarn workspace @casehubio/blocks-ui-event-trail run build`
Expected: TypeScript compiles without errors

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/151/blocks-ui add components/event-trail/ package.json
git -C /Users/mdproctor/claude/casehub/slots/151/blocks-ui commit -m "feat(event-trail): blocks-event-trail composing pages-filter-bar + pages-table Refs #126"
```

---

## Batch 3: audit-trail-viewer refactor (blocks-ui #126)

### Task 5: Refactor audit-trail-viewer to compose blocks-event-trail

**Files:**
- Modify: `components/audit-trail-viewer/src/audit-trail-viewer.ts` (major refactor)
- Modify: `components/audit-trail-viewer/package.json` (add event-trail dependency)
- Modify: `components/audit-trail-viewer/src/audit-trail-viewer.test.ts` (regression tests)

**Interfaces:**
- Consumes: `BlocksEventTrail` from Task 4, `DataSourceAdapter` from `@casehubio/pages-component`
- Produces: Refactored `AuditTrailViewer` that composes `<blocks-event-trail>` internally. Same external API (endpoint, identity, subjectId, renderEntryPayload).

- [ ] **Step 1: Add event-trail dependency to audit-trail-viewer package.json**

Add to `dependencies` in `components/audit-trail-viewer/package.json`:
```json
"@casehubio/blocks-ui-event-trail": "workspace:*"
```

- [ ] **Step 2: Rewrite audit-trail-viewer to compose blocks-event-trail**

Replace the component body (lines 70-689) with the refactored version. Keep:
- Verification banner (own DataSourceAdapter for `/api/v1/ledger/verify`)
- Attestation detail rendering (`_getRowDetail`, `_fetchAttestations`)
- Ledger column definitions (`ENTRY_COL_DEFS`, `ENTRY_COL_CONFIG`, `ENTRY_RENDERERS`)
- Domain types import
- Endpoint construction (builds full ledger URL with `subjectId`, `tenancyId`)

Remove:
- Filter state management (`_selectedActorId`, `_selectedTypes`, `_dateFrom`, `_dateTo`)
- Filter bar rendering (`_renderFilterControls`)
- DataSourceAdapter for entries (now handled by blocks-event-trail)
- `_filteredEntries` getter
- `_uniqueActors` getter
- Filter handler methods
- Filter CSS (`.filter-controls`, `.chip`, `.filter-section`, `.filter-label`)

The refactored `render()` method:
```typescript
override render(): TemplateResult {
  const verifyBanner = this.verify.error
    ? html`<div class="verification-banner failed" role="status" aria-live="polite">
        <span class="status-icon">⚠</span>
        <span>Verification unavailable: ${this.verify.error}</span>
      </div>`
    : this.verify.loading
      ? html`<div class="verification-banner" role="status" aria-live="polite">Verifying chain integrity...</div>`
      : this._renderVerificationBanner();

  return html`
    ${verifyBanner}
    <blocks-event-trail
      .endpoint=${this._buildLedgerEndpoint()}
      .columnDefs=${ENTRY_COL_DEFS}
      .columnConfig=${ENTRY_COL_CONFIG}
      .columnRenderers=${ENTRY_RENDERERS}
      .chipField=${ENTRY_TYPE_COL}
      .chipValues=${['COMMAND', 'EVENT', 'ATTESTATION']}
      .entityField=${ACTOR_ID_COL}
      entityLabel="Actor"
      showDateRange
      .getRowDetail=${this._getRowDetail}
      .getRowKey=${(row: TypedRow) => row.text(ID_COL)}
      @detail-change=${this._handleDetailChange}
      @data-loaded=${this._handleDataLoaded}
    ></blocks-event-trail>
  `;
}
```

New `_buildLedgerEndpoint()` method replaces `_updateEndpoints()`:
```typescript
private _buildLedgerEndpoint(): string | undefined {
  if (!this.endpoint || !this.subjectId || !this.identity) return undefined;
  const base = typeof window !== 'undefined' ? window.location.origin : 'http://localhost';
  const url = new URL(`${this.endpoint}/api/v1/ledger/entries`, base);
  url.searchParams.set('subjectId', this.subjectId);
  if (this.identity.tenancyId) url.searchParams.set('tenancyId', this.identity.tenancyId);
  return url.toString();
}
```

Remove `entries` DataSourceAdapter — the component no longer manages entry fetching.

The refactored component listens for the `data-loaded` custom event from `blocks-event-trail` to receive raw `LedgerEntry[]`. This is needed because `_getRowDetail` accesses raw entry objects for attestation rendering and `renderEntryPayload`:

```typescript
private _handleDataLoaded(e: CustomEvent<{ entries: unknown[] }>): void {
  this._entries = e.detail.entries as LedgerEntry[];
}
```

Add `@data-loaded=${this._handleDataLoaded}` to the `<blocks-event-trail>` element in the render method.

- [ ] **Step 3: Run existing regression tests**

Run: `yarn workspace @casehubio/blocks-ui-audit-trail-viewer run test`
Expected: Tests pass (verify filter, verification banner, attestation tests still work after refactor)

- [ ] **Step 4: Fix broken tests — shadow DOM query migration**

Tests that query for `.filter-controls`, `.chip`, or `select` elements will break — these now live inside nested shadow DOMs. Migration pattern:

```typescript
// Before (direct shadow DOM query):
const chip = el.shadowRoot!.querySelector('.chip');

// After (nested shadow DOM traversal):
const eventTrail = el.shadowRoot!.querySelector('blocks-event-trail');
const filterBar = eventTrail?.shadowRoot?.querySelector('pages-filter-bar');
const chip = filterBar?.shadowRoot?.querySelector('[role="checkbox"]');
```

Key tests to update:
- Filter chip toggle tests → query through `blocks-event-trail > pages-filter-bar`
- Actor dropdown tests → same nested query path
- Date range input tests → same nested query path
- Table rendering tests → query through `blocks-event-trail > pages-table`
- Verification banner tests → still direct (stays in audit-trail-viewer)
- Attestation tests → still direct (stays in audit-trail-viewer)
- `data-loaded` event test → new test: verify `_entries` populated after event

- [ ] **Step 5: Verify build**

Run: `yarn workspace @casehubio/blocks-ui-audit-trail-viewer run build`
Expected: TypeScript compiles without errors

Run full test suite:
```bash
yarn --cwd /Users/mdproctor/claude/casehub/slots/151/blocks-ui test
```
Expected: All component tests pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/151/blocks-ui add components/audit-trail-viewer/ components/event-trail/
git -C /Users/mdproctor/claude/casehub/slots/151/blocks-ui commit -m "refactor(audit-trail-viewer): compose blocks-event-trail internally Closes #126"
```

---

## References

- [2026-08-23-event-trail-extraction-design.md] — design spec this plan implements
- [decisions.md] — D1-D5 architectural decisions
- [components/audit-trail-viewer/src/audit-trail-viewer.ts] — source being refactored (695 lines)
- [components/channel-activity/src/channel-nav.ts] — custom dropdown keyboard navigation pattern
- [packages/pages-viz/src/components/event-timeline/renderers/filter-bar.ts] — chip filter pattern
- [packages/pages-ui-components/src/badge/pages-badge.ts] — Lit component + CSS custom properties pattern
- [packages/pages-table/] — package scaffold pattern (package.json, tsconfig, vitest.config)
- [packages/pages-component/src/controller/data-source-mixin.ts] — DataSourceMixin API
- [packages/pages-data/src/dataset/types.ts] — TypedDataSet, TypedRow, ColumnId
- [packages/pages-data/src/dataset/conversion.ts] — fromRows API
- [casehubio/blocks-ui#126] — primary issue
- [casehubio/casehub-pages#354] — pages-filter-bar issue
