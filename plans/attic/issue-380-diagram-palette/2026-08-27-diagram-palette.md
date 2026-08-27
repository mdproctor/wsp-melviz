# Diagram Palette & Node Chooser Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #380 — feat(pages-diagram-palette): generic stencil palette component with drag-to-canvas
**Issue group:** #380

**Goal:** Create a generic `@casehubio/pages-diagram-palette` package with two Lit components — `<pages-diagram-palette>` (sidebar) and `<pages-node-chooser>` (popover) — sharing a grouped-item renderer, and update the Interactive Editing example to use them.

**Architecture:** New standalone package following the `pages-property-palette` pattern. Both components consume `PaletteItem[]` (structurally compatible with `StencilTypeInfo`), render grouped items with icons via an `iconRenderer` callback, and fire `pages-palette-select` events. The palette adds collapse persistence; the chooser adds dismissal logic with `FocusTrapMixin`. The example replaces its hand-built toolbar and `showTypePicker()` with these components.

**Tech Stack:** Lit 3, TypeScript 5, Vitest, `@casehubio/pages-primitives` (RovingTabindexMixin, FocusTrapMixin)

## Global Constraints

- All custom elements use `pages-` prefix with guarded registration
- Shadow DOM enabled on both components
- Design tokens from `@casehubio/pages-ui-tokens` for all styling
- ARIA roles per `aria-interaction-contract` (PP-20260817-a11y01)
- Custom events with `composed: true, bubbles: true` per `pages-event-contract`
- Sub-path exports per `web-component-strategy` for side-effect isolation

---

## Batch 1: Package scaffold and shared types

### Task 1: Create package scaffold with types and shared renderer

**Files:**
- Create: `packages/pages-diagram-palette/package.json`
- Create: `packages/pages-diagram-palette/tsconfig.json`
- Create: `packages/pages-diagram-palette/tsconfig.build.json`
- Create: `packages/pages-diagram-palette/vitest.config.ts`
- Create: `packages/pages-diagram-palette/src/types.ts`
- Create: `packages/pages-diagram-palette/src/internal/search-filter.ts`
- Create: `packages/pages-diagram-palette/src/internal/stencil-list-renderer.ts`
- Create: `packages/pages-diagram-palette/src/index.ts`
- Create: `packages/pages-diagram-palette/src/palette/index.ts`
- Create: `packages/pages-diagram-palette/src/chooser/index.ts`
- Test: `packages/pages-diagram-palette/src/internal/search-filter.test.ts`

**Interfaces:**
- Produces: `PaletteItem { type: string; label: string; icon: string; group?: string }`, `PaletteSelectDetail { item: PaletteItem }`, `IconRenderer = (icon: string) => TemplateResult`, `filterItems(items, query): PaletteItem[]`, `groupItems(items): Map<string, PaletteItem[]>`, `renderStencilList(items, options): TemplateResult`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/pages-diagram-palette",
  "version": "0.1.0",
  "description": "Generic diagram palette and node chooser components",
  "repository": { "type": "git", "url": "https://github.com/casehubio/casehub-pages.git" },
  "publishConfig": { "registry": "https://npm.pkg.github.com" },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./palette": { "types": "./dist/palette/index.d.ts", "default": "./dist/palette/index.js" },
    "./chooser": { "types": "./dist/chooser/index.d.ts", "default": "./dist/chooser/index.js" },
    "./types": { "types": "./dist/types.d.ts", "default": "./dist/types.js" }
  },
  "sideEffects": ["./dist/index.js", "./dist/palette/index.js", "./dist/chooser/index.js", "./src/index.ts", "./src/palette/index.ts", "./src/chooser/index.ts"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/pages-primitives": "workspace:*",
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

- [ ] **Step 2: Create tsconfig files**

`tsconfig.json`:
```json
{
  "extends": "@casehubio/pages-tsconfig",
  "compilerOptions": {
    "composite": true,
    "outDir": ".typecheck",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

`tsconfig.build.json`:
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "composite": false
  },
  "exclude": ["src/**/*.test.ts"]
}
```

`vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: { environment: 'jsdom' },
});
```

- [ ] **Step 3: Write types.ts**

```typescript
import type { TemplateResult } from 'lit';

export interface PaletteItem {
  readonly type: string;
  readonly label: string;
  readonly icon: string;
  readonly group?: string;
}

export interface PaletteSelectDetail {
  readonly item: PaletteItem;
}

export type IconRenderer = (icon: string) => TemplateResult;
```

- [ ] **Step 4: Write failing test for search-filter**

```typescript
import { describe, it, expect } from 'vitest';
import { filterItems, groupItems } from './search-filter.js';
import type { PaletteItem } from '../types.js';

const items: PaletteItem[] = [
  { type: 'source', label: 'Source', icon: '⬇', group: 'Input' },
  { type: 'transform', label: 'Transform', icon: '⚙', group: 'Processing' },
  { type: 'filter', label: 'Filter', icon: '⧖', group: 'Processing' },
  { type: 'sink', label: 'Sink', icon: '⬆', group: 'Output' },
  { type: 'join', label: 'Join', icon: '⨝' },
];

describe('filterItems', () => {
  it('returns all items for empty query', () => {
    expect(filterItems(items, '')).toEqual(items);
  });

  it('filters by label substring case-insensitive', () => {
    const result = filterItems(items, 'trans');
    expect(result).toHaveLength(1);
    expect(result[0].type).toBe('transform');
  });

  it('filters by type', () => {
    const result = filterItems(items, 'sink');
    expect(result).toHaveLength(1);
    expect(result[0].type).toBe('sink');
  });

  it('filters by group', () => {
    const result = filterItems(items, 'processing');
    expect(result).toHaveLength(2);
  });

  it('returns empty for no matches', () => {
    expect(filterItems(items, 'zzz')).toHaveLength(0);
  });
});

describe('groupItems', () => {
  it('groups by group field', () => {
    const groups = groupItems(items);
    expect(groups.get('Input')).toHaveLength(1);
    expect(groups.get('Processing')).toHaveLength(2);
    expect(groups.get('Output')).toHaveLength(1);
  });

  it('puts ungrouped items under empty string key', () => {
    const groups = groupItems(items);
    expect(groups.get('')).toHaveLength(1);
    expect(groups.get('')![0].type).toBe('join');
  });

  it('returns empty map for empty input', () => {
    expect(groupItems([])).toEqual(new Map());
  });
});
```

- [ ] **Step 5: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-diagram-palette run test`
Expected: FAIL — modules not found

- [ ] **Step 6: Implement search-filter.ts**

```typescript
import type { PaletteItem } from '../types.js';

export function filterItems(
  items: readonly PaletteItem[],
  query: string,
): PaletteItem[] {
  if (!query) return [...items];
  const lower = query.toLowerCase();
  return items.filter(
    item =>
      item.label.toLowerCase().includes(lower) ||
      item.type.toLowerCase().includes(lower) ||
      (item.group?.toLowerCase().includes(lower) ?? false),
  );
}

export function groupItems(
  items: readonly PaletteItem[],
): Map<string, PaletteItem[]> {
  const groups = new Map<string, PaletteItem[]>();
  for (const item of items) {
    const key = item.group ?? '';
    const list = groups.get(key);
    if (list) {
      list.push(item);
    } else {
      groups.set(key, [item]);
    }
  }
  return groups;
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-diagram-palette run test`
Expected: PASS

- [ ] **Step 8: Write stencil-list-renderer.ts**

```typescript
import { html, nothing, type TemplateResult } from 'lit';
import type { PaletteItem, IconRenderer } from '../types.js';
import { filterItems, groupItems } from './search-filter.js';

export interface RenderOptions {
  collapsible: boolean;
  isGroupOpen?: (name: string) => boolean;
  onGroupToggle?: (name: string, open: boolean) => void;
  onSelect: (item: PaletteItem) => void;
  searchQuery: string;
  itemRole: 'button' | 'option';
  iconRenderer?: IconRenderer;
}

function renderIcon(icon: string, renderer?: IconRenderer): TemplateResult {
  if (renderer) return renderer(icon);
  return html`<span class="palette-item-icon">${icon}</span>`;
}

function renderItem(
  item: PaletteItem,
  role: 'button' | 'option',
  onSelect: (item: PaletteItem) => void,
  iconRenderer?: IconRenderer,
): TemplateResult {
  const handleClick = () => onSelect(item);
  const handleKeydown = (e: KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      onSelect(item);
    }
  };
  return html`
    <div class="palette-item"
      role=${role}
      aria-label=${item.label}
      tabindex="-1"
      @click=${handleClick}
      @keydown=${handleKeydown}>
      ${renderIcon(item.icon, iconRenderer)}
      <span class="palette-item-label">${item.label}</span>
    </div>`;
}

export function renderStencilList(
  items: readonly PaletteItem[],
  options: RenderOptions,
): TemplateResult {
  const filtered = filterItems(items, options.searchQuery);
  const groups = groupItems(filtered);
  const searchActive = options.searchQuery.length > 0;

  const ungrouped = groups.get('');
  groups.delete('');

  const groupEntries = Array.from(groups.entries());

  return html`
    ${ungrouped && ungrouped.length > 0
      ? html`<div class="ungrouped-items">
          ${ungrouped.map(item => renderItem(item, options.itemRole, options.onSelect, options.iconRenderer))}
        </div>`
      : nothing}
    ${groupEntries.map(([name, groupItems]) =>
      options.collapsible && !searchActive
        ? html`
            <details class="palette-group"
              ?open=${options.isGroupOpen?.(name) ?? true}
              @toggle=${(e: Event) => options.onGroupToggle?.(name, (e.target as HTMLDetailsElement).open)}>
              <summary>${name}</summary>
              <div class="palette-group-items">
                ${groupItems.map(item => renderItem(item, options.itemRole, options.onSelect, options.iconRenderer))}
              </div>
            </details>`
        : html`
            <div class="palette-group" role="group" aria-label=${name}>
              <div class="palette-group-header">${name}</div>
              <div class="palette-group-items">
                ${groupItems.map(item => renderItem(item, options.itemRole, options.onSelect, options.iconRenderer))}
              </div>
            </div>`,
    )}`;
}
```

- [ ] **Step 9: Write barrel index files**

`src/palette/index.ts`:
```typescript
export { PagesDiagramPalette } from './pages-diagram-palette.js';
```

`src/chooser/index.ts`:
```typescript
export { PagesNodeChooser } from './pages-node-chooser.js';
```

`src/index.ts`:
```typescript
export type { PaletteItem, PaletteSelectDetail, IconRenderer } from './types.js';
export { PagesDiagramPalette } from './palette/index.js';
export { PagesNodeChooser } from './chooser/index.js';
```

(These will fail until Tasks 2 and 3 create the components — that's expected.)

- [ ] **Step 10: Install dependencies**

Run: `yarn install`

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-diagram-palette/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-diagram-palette): package scaffold, types, search-filter, shared renderer Refs #380"
```

---

## Batch 2: Palette and chooser components

### Task 2: Implement `<pages-diagram-palette>` sidebar component

**Files:**
- Create: `packages/pages-diagram-palette/src/palette/pages-diagram-palette.ts`
- Test: `packages/pages-diagram-palette/src/palette/pages-diagram-palette.test.ts`

**Interfaces:**
- Consumes: `PaletteItem`, `PaletteSelectDetail`, `IconRenderer` from `../types.js`; `renderStencilList()` from `../internal/stencil-list-renderer.js`; `RovingTabindexMixin` from `@casehubio/pages-primitives/a11y`
- Produces: `PagesDiagramPalette` — Lit element with properties `items: PaletteItem[]`, `paletteId: string`, `searchThreshold: number`, `iconRenderer: IconRenderer`; fires `pages-palette-select` event

- [ ] **Step 1: Write failing test**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import type { PaletteItem } from '../types.js';

import './pages-diagram-palette.js';

const items: PaletteItem[] = [
  { type: 'source', label: 'Source', icon: '⬇', group: 'Input' },
  { type: 'transform', label: 'Transform', icon: '⚙', group: 'Processing' },
  { type: 'filter', label: 'Filter', icon: '⧖', group: 'Processing' },
  { type: 'sink', label: 'Sink', icon: '⬆', group: 'Output' },
];

function createElement(props: Partial<{ items: PaletteItem[]; paletteId: string; searchThreshold: number }> = {}) {
  const el = document.createElement('pages-diagram-palette') as any;
  el.items = props.items ?? items;
  if (props.paletteId) el.paletteId = props.paletteId;
  if (props.searchThreshold !== undefined) el.searchThreshold = props.searchThreshold;
  document.body.appendChild(el);
  return el;
}

describe('pages-diagram-palette', () => {
  beforeEach(() => {
    document.body.innerHTML = '';
    localStorage.clear();
  });

  it('registers as custom element', () => {
    expect(customElements.get('pages-diagram-palette')).toBeDefined();
  });

  it('renders items grouped by group field', async () => {
    const el = createElement();
    await el.updateComplete;
    const groups = el.shadowRoot.querySelectorAll('details.palette-group');
    expect(groups.length).toBe(3);
  });

  it('fires pages-palette-select on item click', async () => {
    const el = createElement();
    await el.updateComplete;
    const fired: any[] = [];
    el.addEventListener('pages-palette-select', (e: CustomEvent) => fired.push(e.detail));
    const item = el.shadowRoot.querySelector('.palette-item');
    item?.click();
    expect(fired).toHaveLength(1);
    expect(fired[0].item.type).toBeDefined();
  });

  it('shows search input when items exceed threshold', async () => {
    const el = createElement({ searchThreshold: 3 });
    await el.updateComplete;
    const search = el.shadowRoot.querySelector('[role="searchbox"]');
    expect(search).not.toBeNull();
  });

  it('hides search input when items below threshold', async () => {
    const el = createElement({ searchThreshold: 10 });
    await el.updateComplete;
    const search = el.shadowRoot.querySelector('[role="searchbox"]');
    expect(search).toBeNull();
  });

  it('has role="region" with aria-label', async () => {
    const el = createElement();
    await el.updateComplete;
    const root = el.shadowRoot.querySelector('[role="region"]');
    expect(root).not.toBeNull();
    expect(root?.getAttribute('aria-label')).toBe('Node palette');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-diagram-palette run test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement pages-diagram-palette.ts**

```typescript
import { LitElement, html, css, nothing, type TemplateResult } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { PaletteItem, PaletteSelectDetail, IconRenderer } from '../types.js';
import { renderStencilList } from '../internal/stencil-list-renderer.js';

export class PagesDiagramPalette extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .palette-search {
      display: block; width: 100%; padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border: 1px solid var(--pages-neutral-4, #e5e7eb); border-radius: var(--pages-radius-sm, 4px);
      background: var(--pages-neutral-2, #fafafa); color: var(--pages-neutral-12, #333);
      font-size: var(--pages-font-size-base, 14px); font-family: inherit;
      margin-bottom: var(--pages-space-2, 8px); box-sizing: border-box;
    }
    .palette-search:focus { outline: 2px solid var(--pages-accent-9, #5470c6); outline-offset: -2px; }
    .palette-item {
      display: flex; align-items: center; gap: var(--pages-space-2, 8px);
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border-radius: var(--pages-radius-sm, 4px); cursor: pointer;
      border: none; background: transparent; color: var(--pages-neutral-12, #333);
      font-size: var(--pages-font-size-base, 14px); width: 100%; text-align: left;
    }
    .palette-item:hover { background: var(--pages-neutral-3, #f3f4f6); }
    .palette-item:focus-visible { outline: 2px solid var(--pages-accent-9, #5470c6); outline-offset: -2px; }
    .palette-item-icon { width: 20px; height: 20px; flex-shrink: 0; display: inline-flex; align-items: center; justify-content: center; }
    .palette-item-label { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    details.palette-group { border: 1px solid var(--pages-neutral-4, #e5e7eb); border-radius: var(--pages-radius-sm, 4px); margin-bottom: var(--pages-space-1, 4px); }
    details.palette-group summary {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      font-size: var(--pages-font-size-base, 14px); font-weight: var(--pages-font-weight-semibold, 600);
      color: var(--pages-neutral-11, #374151); cursor: pointer; user-select: none;
    }
    .palette-group-items, .ungrouped-items {
      display: flex; flex-direction: column; padding: var(--pages-space-1, 4px);
    }
    .palette-group-header {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      font-size: var(--pages-font-size-base, 14px); font-weight: var(--pages-font-weight-semibold, 600);
      color: var(--pages-neutral-11, #374151);
    }
  `;

  @property({ attribute: false }) items: readonly PaletteItem[] = [];
  @property() paletteId: string | undefined;
  @property({ type: Number }) searchThreshold = 8;
  @property({ attribute: false }) iconRenderer: IconRenderer | undefined;

  @state() private _searchQuery = '';

  override render(): TemplateResult {
    const showSearch = this.items.length > this.searchThreshold;
    return html`
      <div role="region" aria-label="Node palette">
        ${showSearch
          ? html`<input class="palette-search" role="searchbox"
              aria-label="Filter palette items"
              placeholder="Search..."
              .value=${this._searchQuery}
              @input=${(e: Event) => { this._searchQuery = (e.target as HTMLInputElement).value; }}
            />`
          : nothing}
        ${renderStencilList(this.items, {
          collapsible: true,
          isGroupOpen: (name) => this._isGroupOpen(name),
          onGroupToggle: (name, open) => this._onGroupToggle(name, open),
          onSelect: (item) => this._onSelect(item),
          searchQuery: this._searchQuery,
          itemRole: 'button',
          iconRenderer: this.iconRenderer,
        })}
      </div>`;
  }

  private _onSelect(item: PaletteItem): void {
    this.dispatchEvent(new CustomEvent<PaletteSelectDetail>('pages-palette-select', {
      detail: { item },
      bubbles: true,
      composed: true,
    }));
  }

  private _storageKey(groupName: string): string {
    return `pages-palette-${this.paletteId ?? 'default'}-${groupName}`;
  }

  private _isGroupOpen(name: string): boolean {
    const stored = localStorage.getItem(this._storageKey(name));
    return stored === null ? true : stored === 'true';
  }

  private _onGroupToggle(name: string, open: boolean): void {
    localStorage.setItem(this._storageKey(name), String(open));
  }
}

if (!customElements.get('pages-diagram-palette')) {
  customElements.define('pages-diagram-palette', PagesDiagramPalette);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-diagram-palette run test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-diagram-palette/src/palette/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-diagram-palette): implement <pages-diagram-palette> sidebar component Refs #380"
```

### Task 3: Implement `<pages-node-chooser>` popover component

**Files:**
- Create: `packages/pages-diagram-palette/src/chooser/pages-node-chooser.ts`
- Test: `packages/pages-diagram-palette/src/chooser/pages-node-chooser.test.ts`

**Interfaces:**
- Consumes: `PaletteItem`, `PaletteSelectDetail`, `IconRenderer` from `../types.js`; `renderStencilList()` from `../internal/stencil-list-renderer.js`; `FocusTrapMixin` from `@casehubio/pages-primitives/a11y`
- Produces: `PagesNodeChooser` — Lit element with properties `items: PaletteItem[]`, `searchThreshold: number`, `iconRenderer: IconRenderer`, `abortSignal: AbortSignal`; fires `pages-palette-select` and `pages-chooser-dismiss` events

- [ ] **Step 1: Write failing test**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import type { PaletteItem } from '../types.js';

import './pages-node-chooser.js';

const items: PaletteItem[] = [
  { type: 'source', label: 'Source', icon: '⬇', group: 'Input' },
  { type: 'transform', label: 'Transform', icon: '⚙', group: 'Processing' },
  { type: 'filter', label: 'Filter', icon: '⧖', group: 'Processing' },
];

function createElement(props: Partial<{ items: PaletteItem[]; searchThreshold: number; abortSignal: AbortSignal }> = {}) {
  const el = document.createElement('pages-node-chooser') as any;
  el.items = props.items ?? items;
  if (props.searchThreshold !== undefined) el.searchThreshold = props.searchThreshold;
  if (props.abortSignal) el.abortSignal = props.abortSignal;
  document.body.appendChild(el);
  return el;
}

describe('pages-node-chooser', () => {
  beforeEach(() => { document.body.innerHTML = ''; });

  it('registers as custom element', () => {
    expect(customElements.get('pages-node-chooser')).toBeDefined();
  });

  it('has role="dialog" with aria-label and aria-modal', async () => {
    const el = createElement();
    await el.updateComplete;
    const dialog = el.shadowRoot.querySelector('[role="dialog"]');
    expect(dialog).not.toBeNull();
    expect(dialog?.getAttribute('aria-label')).toBe('Choose node type');
    expect(dialog?.getAttribute('aria-modal')).toBe('true');
  });

  it('contains a listbox with options', async () => {
    const el = createElement();
    await el.updateComplete;
    const listbox = el.shadowRoot.querySelector('[role="listbox"]');
    expect(listbox).not.toBeNull();
    const options = el.shadowRoot.querySelectorAll('[role="option"]');
    expect(options.length).toBe(3);
  });

  it('fires pages-palette-select then pages-chooser-dismiss on item click', async () => {
    const el = createElement();
    await el.updateComplete;
    const selected: any[] = [];
    let dismissed = false;
    el.addEventListener('pages-palette-select', (e: CustomEvent) => selected.push(e.detail));
    el.addEventListener('pages-chooser-dismiss', () => { dismissed = true; });
    const item = el.shadowRoot.querySelector('[role="option"]');
    item?.click();
    expect(selected).toHaveLength(1);
    expect(dismissed).toBe(true);
  });

  it('fires pages-chooser-dismiss on Escape', async () => {
    const el = createElement();
    await el.updateComplete;
    let dismissed = false;
    el.addEventListener('pages-chooser-dismiss', () => { dismissed = true; });
    el.shadowRoot.querySelector('[role="dialog"]')?.dispatchEvent(
      new KeyboardEvent('keydown', { key: 'Escape', bubbles: true, composed: true }),
    );
    expect(dismissed).toBe(true);
  });

  it('fires pages-chooser-dismiss on abortSignal', async () => {
    const ac = new AbortController();
    const el = createElement({ abortSignal: ac.signal });
    await el.updateComplete;
    let dismissed = false;
    el.addEventListener('pages-chooser-dismiss', () => { dismissed = true; });
    ac.abort();
    expect(dismissed).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-diagram-palette run test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement pages-node-chooser.ts**

```typescript
import { LitElement, html, css, nothing, type TemplateResult } from 'lit';
import { property, state } from 'lit/decorators.js';
import { FocusTrapMixin } from '@casehubio/pages-primitives/a11y';
import type { PaletteItem, PaletteSelectDetail, IconRenderer } from '../types.js';
import { renderStencilList } from '../internal/stencil-list-renderer.js';

export class PagesNodeChooser extends FocusTrapMixin(LitElement) {
  static override styles = css`
    :host {
      display: block;
      background: var(--pages-neutral-1, #fff);
      border: 1px solid var(--pages-neutral-4, #e5e7eb);
      border-radius: var(--pages-radius-md, 8px);
      box-shadow: var(--pages-shadow-md, 0 4px 12px rgba(0,0,0,0.1));
      padding: var(--pages-space-2, 8px);
      min-width: 200px;
      max-height: 320px;
      overflow-y: auto;
      font-family: var(--pages-font-family, system-ui, sans-serif);
    }
    .chooser-search {
      display: block; width: 100%; padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border: 1px solid var(--pages-neutral-4, #e5e7eb); border-radius: var(--pages-radius-sm, 4px);
      background: var(--pages-neutral-2, #fafafa); color: var(--pages-neutral-12, #333);
      font-size: var(--pages-font-size-base, 14px); font-family: inherit;
      margin-bottom: var(--pages-space-2, 8px); box-sizing: border-box;
    }
    .chooser-search:focus { outline: 2px solid var(--pages-accent-9, #5470c6); outline-offset: -2px; }
    .palette-item {
      display: flex; align-items: center; gap: var(--pages-space-2, 8px);
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      border-radius: var(--pages-radius-sm, 4px); cursor: pointer;
      border: none; background: transparent; color: var(--pages-neutral-12, #333);
      font-size: var(--pages-font-size-base, 14px); width: 100%; text-align: left;
    }
    .palette-item:hover { background: var(--pages-neutral-3, #f3f4f6); }
    .palette-item:focus-visible { outline: 2px solid var(--pages-accent-9, #5470c6); outline-offset: -2px; }
    .palette-item-icon { width: 20px; height: 20px; flex-shrink: 0; display: inline-flex; align-items: center; justify-content: center; }
    .palette-item-label { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    .palette-group { margin-bottom: var(--pages-space-1, 4px); }
    .palette-group-header {
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      font-size: 11px; font-weight: var(--pages-font-weight-semibold, 600);
      color: var(--pages-neutral-8, #9ca3af); text-transform: uppercase; letter-spacing: 0.05em;
    }
    .palette-group-items, .ungrouped-items {
      display: flex; flex-direction: column;
    }
  `;

  @property({ attribute: false }) items: readonly PaletteItem[] = [];
  @property({ type: Number }) searchThreshold = 8;
  @property({ attribute: false }) abortSignal: AbortSignal | undefined;
  @property({ attribute: false }) iconRenderer: IconRenderer | undefined;

  @state() private _searchQuery = '';

  override connectedCallback(): void {
    super.connectedCallback();
    if (this.abortSignal) {
      this.abortSignal.addEventListener('abort', this._onAbort);
    }
    requestAnimationFrame(() => {
      document.addEventListener('pointerdown', this._onClickOutside, true);
    });
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    document.removeEventListener('pointerdown', this._onClickOutside, true);
    this.abortSignal?.removeEventListener('abort', this._onAbort);
  }

  override render(): TemplateResult {
    const showSearch = this.items.length > this.searchThreshold;
    const listboxId = 'node-list';
    return html`
      <div role="dialog" aria-label="Choose node type" aria-modal="true"
        @keydown=${this._onKeydown}>
        ${showSearch
          ? html`<input class="chooser-search" role="searchbox"
              aria-label="Filter node types"
              aria-controls=${listboxId}
              placeholder="Search..."
              .value=${this._searchQuery}
              @input=${(e: Event) => { this._searchQuery = (e.target as HTMLInputElement).value; }}
            />`
          : nothing}
        <div role="listbox" id=${listboxId} aria-label="Node types">
          ${renderStencilList(this.items, {
            collapsible: false,
            onSelect: (item) => this._onSelect(item),
            searchQuery: this._searchQuery,
            itemRole: 'option',
            iconRenderer: this.iconRenderer,
          })}
        </div>
      </div>`;
  }

  private _onSelect(item: PaletteItem): void {
    this.dispatchEvent(new CustomEvent<PaletteSelectDetail>('pages-palette-select', {
      detail: { item },
      bubbles: true,
      composed: true,
    }));
    this._dismiss();
  }

  private _dismiss(): void {
    this.dispatchEvent(new CustomEvent('pages-chooser-dismiss', {
      bubbles: true,
      composed: true,
    }));
  }

  private _onKeydown = (e: KeyboardEvent): void => {
    if (e.key === 'Escape') {
      e.preventDefault();
      this._dismiss();
    }
  };

  private _onClickOutside = (e: PointerEvent): void => {
    if (!this.contains(e.target as Node)) {
      this._dismiss();
    }
  };

  private _onAbort = (): void => {
    this._dismiss();
  };
}

if (!customElements.get('pages-node-chooser')) {
  customElements.define('pages-node-chooser', PagesNodeChooser);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-diagram-palette run test`
Expected: PASS (all tests for both components)

- [ ] **Step 5: Run typecheck**

Run: `yarn workspace @casehubio/pages-diagram-palette run typecheck`
Expected: PASS — no type errors

- [ ] **Step 6: Build the package**

Run: `yarn workspace @casehubio/pages-diagram-palette run build`
Expected: PASS — dist/ produced

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-diagram-palette/src/chooser/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-diagram-palette): implement <pages-node-chooser> popover component Refs #380"
```

---

## Batch 3: Example integration

### Task 4: Update Interactive Editing example to use palette and chooser

**Files:**
- Modify: `examples/package.json` — add `@casehubio/pages-diagram-palette` dependency
- Modify: `examples/src/casehub-entry.ts` — add palette import
- Modify: `examples/samples/Graph Editing/Interactive Editing.dash.yaml` — replace toolbar HTML with `<pages-diagram-palette>`
- Modify: `examples/samples/Graph Editing/Interactive Editing.ts` — replace hand-built toolbar/picker with palette events and `<pages-node-chooser>`

**Interfaces:**
- Consumes: `<pages-diagram-palette>` (fires `pages-palette-select`), `<pages-node-chooser>` (fires `pages-palette-select`, `pages-chooser-dismiss`), `getAllStencils()` from `@casehubio/graph-renderer`, `defaultEditPolicy()`, `applyGraphEdit()`

- [ ] **Step 1: Add dependency to examples/package.json**

Add `"@casehubio/pages-diagram-palette": "workspace:*"` to `dependencies` in `examples/package.json`.

- [ ] **Step 2: Add import to casehub-entry.ts**

Add after the `pages-property-palette` import:
```typescript
import "@casehubio/pages-diagram-palette";
```

- [ ] **Step 3: Update Interactive Editing YAML**

Replace the hand-built toolbar `<div id="node-toolbar">` with `<pages-diagram-palette>`:

```yaml
pages:
  - name: Interactive Editing
    components:
      - type: markdown
        properties:
          content: |
            Interactive data pipeline editor with constraint validation.

            **Try:** Click a palette item to add a node &bull; Drag from a handle to connect nodes &bull; Click an edge to insert a node &bull; Click empty canvas to create a node &bull; Select a node and press Delete to remove it

            Connection rules enforce domain constraints &mdash; Source can only connect to Transform/Filter/Join, Sink has no outbound.

      - type: html
        properties:
          content: |
            <div style="display: flex; gap: 12px; height: 520px;">
              <pages-diagram-palette id="edit-palette" paletteId="pipeline-editor" style="width: 160px; border: 1px solid var(--pages-neutral-4, #d1d5db); border-radius: 8px; padding: 8px; overflow-y: auto;"></pages-diagram-palette>
              <div style="flex: 1; display: flex; flex-direction: column;">
                <pages-graph-canvas id="edit-canvas" style="flex: 1; display: block; border: 1px solid var(--pages-neutral-4, #d1d5db); border-radius: 8px; overflow: hidden;"></pages-graph-canvas>
                <div style="margin-top: 8px; font-family: var(--pages-font-family, system-ui);">
                  <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 4px;">
                    <strong style="font-size: 13px; color: var(--pages-neutral-11, #374151);">Mutation Log</strong>
                    <button id="clear-log" style="font-size: 11px; padding: 2px 8px; border: 1px solid var(--pages-neutral-5); border-radius: 4px; background: var(--pages-neutral-3); color: var(--pages-neutral-11); cursor: pointer;">Clear</button>
                  </div>
                  <pre id="mutation-log" style="font-size: 11px; max-height: 120px; overflow-y: auto; padding: 8px; background: var(--pages-neutral-2, #1e1e1e); border-radius: 4px; color: var(--pages-neutral-11, #d4d4d4); margin: 0;"></pre>
                </div>
              </div>
            </div>
```

- [ ] **Step 4: Rewrite Interactive Editing.ts**

Replace the entire file — remove the hand-built toolbar button listeners and `showTypePicker()` function, use the palette component and `<pages-node-chooser>` instead:

```typescript
var editCanvas = document.getElementById('edit-canvas');
var editPalette = document.getElementById('edit-palette');
var mutationLog = document.getElementById('mutation-log');
var clearBtn = document.getElementById('clear-log');
var selectedNodeIds = [];
var lastClickX = 0;
var lastClickY = 0;
var icons = { source: '⬇', transform: '⚙', filter: '⧖', join: '⨝', sink: '⬆' };
var labels = { source: 'New Source', transform: 'New Transform', filter: 'New Filter', join: 'New Join', sink: 'New Sink' };
var stencilItems = (window as any).casehubPages.getAllStencils
  ? (window as any).casehubPages.getAllStencils().map(function(s) {
      return { type: s.type, label: s.label, icon: s.icon, group: undefined };
    })
  : [
      { type: 'source', label: 'Source', icon: '⬇', group: 'Input' },
      { type: 'transform', label: 'Transform', icon: '⚙', group: 'Processing' },
      { type: 'filter', label: 'Filter', icon: '⧖', group: 'Processing' },
      { type: 'join', label: 'Join', icon: '⨝', group: 'Processing' },
      { type: 'sink', label: 'Sink', icon: '⬆', group: 'Output' },
    ];

function logMutation(label, detail) {
  if (mutationLog) {
    var entry = new Date().toLocaleTimeString() + '  ' + label + ': ' + JSON.stringify(detail) + '\n';
    mutationLog.textContent = entry + (mutationLog.textContent || '');
  }
}

function dismissChooser() {
  var existing = document.querySelector('pages-node-chooser');
  if (existing) existing.remove();
}

function showNodeChooser(x, y, types, onSelect) {
  dismissChooser();
  var chooser = document.createElement('pages-node-chooser') as any;
  chooser.items = types;
  chooser.style.cssText = 'position:fixed;left:' + x + 'px;top:' + y + 'px;z-index:9999';
  document.body.appendChild(chooser);

  chooser.addEventListener('pages-palette-select', function(e) {
    onSelect(e.detail.item.type);
  });
  chooser.addEventListener('pages-chooser-dismiss', function() {
    chooser.remove();
  });
}

if (editPalette) {
  (editPalette as any).items = stencilItems;
}

if (editCanvas) {
  (editCanvas as any).model = (window as any).casehubPages.createBasicPipelineModel();
  (editCanvas as any).editPolicy = (window as any).casehubPages.defaultEditPolicy();

  editCanvas.addEventListener('click', function(evt) {
    lastClickX = (evt as MouseEvent).clientX;
    lastClickY = (evt as MouseEvent).clientY;
  }, true);

  (editCanvas as any).onMutation = function(edit) {
    var result = (window as any).casehubPages.applyGraphEdit((editCanvas as any).model, edit);
    (editCanvas as any).model = result.model;
    logMutation(edit.type, edit);
  };

  if (editPalette) {
    editPalette.addEventListener('pages-palette-select', function(e) {
      var nodeType = (e as CustomEvent).detail.item.type;
      (editCanvas as any).onMutation({ type: 'addNode', nodeType: nodeType, properties: { name: labels[nodeType] || nodeType } });
    });
  }

  editCanvas.addEventListener('pages-event', function(e) {
    var detail = (e as CustomEvent).detail;
    if (detail.topic === 'graph:selection:change') {
      selectedNodeIds = detail.payload.nodeIds || [];
    }
    if (detail.topic === 'graph:edge:click') {
      var edgeId = detail.payload.edgeId;
      var model = (editCanvas as any).model;
      var policy = (editCanvas as any).editPolicy;
      var edge = model.edges.find(function(ed) { return ed.id === edgeId; });
      if (!edge || !policy) return;
      var types = policy.getInsertableTypes(edge, model);
      if (types.length === 0) return;
      if (types.length === 1) {
        (editCanvas as any).onMutation({ type: 'splitEdge', edgeId: edgeId, insertNodeType: types[0].type });
        return;
      }
      showNodeChooser(lastClickX, lastClickY, types, function(nodeType) {
        (editCanvas as any).onMutation({ type: 'splitEdge', edgeId: edgeId, insertNodeType: nodeType });
      });
    }
    if (detail.topic === 'graph:pane:click') {
      var panePolicy = (editCanvas as any).editPolicy;
      var paneModel = (editCanvas as any).model;
      if (!panePolicy) return;
      var creatableTypes = panePolicy.getCreatableTypes(null, paneModel);
      if (creatableTypes.length === 0) return;
      showNodeChooser(detail.payload.x, detail.payload.y, creatableTypes, function(nodeType) {
        (editCanvas as any).onMutation({ type: 'addNode', nodeType: nodeType, properties: { name: labels[nodeType] || nodeType } });
      });
    }
    if (detail.topic === 'graph:connect:end-on-empty') {
      var sourceId = detail.payload.sourceNodeId;
      var connectPolicy = (editCanvas as any).editPolicy;
      var connectModel = (editCanvas as any).model;
      if (!connectPolicy || !sourceId) return;
      var sourceNode = connectModel.nodes.find(function(n) { return n.id === sourceId; });
      var connectTypes = connectPolicy.getCreatableTypes(sourceNode || null, connectModel);
      if (connectTypes.length === 0) return;
      showNodeChooser(detail.payload.x, detail.payload.y, connectTypes, function(nodeType) {
        var newId = 'node-' + Date.now();
        (editCanvas as any).onMutation({
          type: 'compound',
          edits: [
            { type: 'addNode', id: newId, nodeType: nodeType, properties: { name: labels[nodeType] || nodeType } },
            { type: 'addEdge', sourceId: sourceId, targetId: newId },
          ],
        });
      });
    }
  });

  document.addEventListener('keydown', function(e) {
    if (e.key === 'Escape') { dismissChooser(); return; }
    if (e.key !== 'Delete' && e.key !== 'Backspace') return;
    if (!selectedNodeIds.length) return;
    var target = e.target;
    if (target && ((target as HTMLElement).tagName === 'INPUT' || (target as HTMLElement).tagName === 'TEXTAREA')) return;

    e.preventDefault();
    var policy = (editCanvas as any).editPolicy;
    var model = (editCanvas as any).model;
    if (!policy || !model) return;

    for (var i = 0; i < selectedNodeIds.length; i++) {
      var nodeId = selectedNodeIds[i];
      var node = model.nodes.find(function(n) { return n.id === nodeId; });
      if (!node) continue;
      var strategy = policy.getDeleteStrategy(node, model);
      var result = (window as any).casehubPages.applyGraphEdit(model, { type: 'removeNode', nodeId: nodeId, strategy: strategy });
      model = result.model;
      logMutation('removeNode', { nodeId: nodeId, strategy: strategy.type });
    }
    (editCanvas as any).model = model;
    selectedNodeIds = [];
  });
}

if (clearBtn && mutationLog) {
  clearBtn.addEventListener('click', function() {
    mutationLog.textContent = '';
  });
}
```

- [ ] **Step 5: Export getAllStencils from casehub-entry.ts**

Add to the exports in `casehub-entry.ts`:
```typescript
export { getAllStencils } from "@casehubio/graph-renderer";
```

- [ ] **Step 6: Build and verify**

Run: `yarn build`
Expected: Full build passes — packages, then examples

- [ ] **Step 7: Verify visually**

Run: `yarn workspace @casehubio/pages-examples run dev`
Navigate to the Interactive Editing sample. Verify:
- Palette sidebar renders with grouped stencil items
- Click a palette item → node added to the graph
- Click an edge → node chooser popover appears at click position
- Select from popover → node inserted, popover dismissed
- Escape → popover dismissed
- Click outside popover → popover dismissed
- Delete selected nodes still works

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/ packages/pages-diagram-palette/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(examples): integrate pages-diagram-palette into Interactive Editing sample Refs #380"
```

---

## References

- `specs/issue-380-diagram-palette/2026-08-27-diagram-palette-design.md` — design spec
- `packages/pages-property-palette/` — established pattern (package structure, Lit conventions)
- `packages/pages-primitives/src/a11y/` — `RovingTabindexMixin`, `FocusTrapMixin`
- `packages/graph-renderer/src/editing/types.ts` — `StencilTypeInfo`, `EditPolicy`
- `packages/graph-renderer/src/registry/stencil-registry.ts` — `getAllStencils()`
- `examples/src/pipeline-stencils.ts` — pipeline stencil registrations
- `examples/samples/Graph Editing/Interactive Editing.ts` — current hand-built picker
- `docs/protocols/casehub/web-component-strategy.md` (PP-20260705-c7687d)
- `docs/protocols/casehub/aria-interaction-contract.md` (PP-20260817-a11y01)
- GitHub #380
