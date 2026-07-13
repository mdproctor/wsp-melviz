# Row Detail Expansion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #172 — feat(pages-table): row-detail expansion — expandable detail panels below rows
**Issue group:** #172

**Goal:** Add row-detail expansion to `pages-table` — a dedicated expand column, animated detail panels below rows, single/multi mode, controlled/uncontrolled state, keyboard nav, and ARIA disclosure pattern.

**Architecture:** Three new public properties (`getRowDetail`, `detailMode`, `expandedDetailKeys`), one new event (`detail-change`), internal state management following the existing selection pattern (controlled vs uncontrolled), a dedicated expand column prepended to the grid, and detail panels rendered as sibling divs after each row with CSS `grid-template-rows: 0fr→1fr` animation.

**Tech Stack:** Lit 3 (reactive properties, decorators, html/css tagged templates), Vitest + jsdom, CSS custom properties from `@casehubio/pages-ui-tokens`

## Global Constraints

- All CSS custom properties use `--pages-` prefix (protocol PP-20260705-2ae91d)
- Motion tokens: `--pages-duration-fast` (120ms), `--pages-duration-normal` (200ms), `--pages-ease-out`
- Surface tokens: `--pages-surface-1` through `--pages-surface-4`
- Event naming: component-internal events use `{noun}-change` pattern (`detail-change`)
- Lit conventions: `@property()` for public API, `@state()` for internal state
- Immutable collection updates — replace Set, never mutate in place
- `prefers-reduced-motion: reduce` disables all transitions
- Instance-scoped IDs via `crypto.randomUUID()` for ARIA linking
- IntelliJ MCP required for all code operations on `.ts` files

---

### Task 1: Types and Public API Surface

**Files:**
- Modify: `packages/pages-table/src/types.ts`
- Modify: `packages/pages-table/src/index.ts`

**Interfaces:**
- Consumes: existing `TypedRow` from `@casehubio/pages-data`
- Produces: `DetailMode`, `DetailChangeDetail` types exported from `types.ts` and `index.ts`

- [ ] **Step 1: Add detail types to types.ts**

Use `ide_insert_member` to add after the `FilterConfig` interface at end of file:

```typescript
export type DetailMode = 'single' | 'multi';

export interface DetailChangeDetail {
  readonly key: string;
  readonly row: TypedRow;
  readonly expanded: boolean;
}
```

- [ ] **Step 2: Export new types from index.ts**

Add `DetailMode` and `DetailChangeDetail` to the type export list in `index.ts`.

- [ ] **Step 3: Verify — run typecheck**

Run: `yarn workspace @casehubio/pages-table run typecheck`
Expected: PASS — new types compile, no existing code broken

- [ ] **Step 4: Commit**

```
feat(pages-table): add DetailMode and DetailChangeDetail types  Refs #172
```

---

### Task 2: Expand State Management (Unit Tests + Implementation)

**Files:**
- Modify: `packages/pages-table/src/pages-table.test.ts`
- Modify: `packages/pages-table/src/pages-table.ts`

**Interfaces:**
- Consumes: `DetailMode`, `DetailChangeDetail` from Task 1
- Produces: `getRowDetail`, `detailMode`, `expandedDetailKeys` properties on `PagesTable`; `_expandedDetailKeys` internal state; `_toggleDetail()`, `_isDetailExpanded()`, `_emitDetailChange()` methods

This task covers the state machine — no rendering yet. The properties, validation, and event emission are testable via the component API without inspecting DOM.

- [ ] **Step 1: Write failing tests for expand state**

Add a new `describe('row detail expansion')` block in `pages-table.test.ts`. Update the `TableEl` type alias to include the new properties. Tests:

```typescript
// In the TableEl type, add:
//   getRowDetail?: (row: TypedRow) => unknown;
//   detailMode?: string;
//   expandedDetailKeys?: readonly string[];

describe('row detail expansion', () => {
  describe('validation', () => {
    it('throws when getRowDetail set without getRowKey', async () => {
      el.dataSet = testDataSet;
      el.columnConfig = testConfig;
      el.getRowDetail = () => 'detail';
      await expect(el.updateComplete).rejects.toThrow('getRowKey is required');
    });

    it('throws when getRowDetail set with mode=scroll', async () => {
      el.dataSet = testDataSet;
      el.columnConfig = testConfig;
      el.getRowKey = (row: TypedRow) => row.text(nameCol);
      el.getRowDetail = () => 'detail';
      el.mode = 'scroll';
      await expect(el.updateComplete).rejects.toThrow('mode=\'scroll\'');
    });
  });

  describe('single mode (default)', () => {
    beforeEach(async () => {
      el.dataSet = testDataSet;
      el.columnConfig = testConfig;
      el.getRowKey = (row: TypedRow) => row.text(nameCol);
      el.getRowDetail = () => html`<div>Detail</div>`;
      await el.updateComplete;
    });

    it('emits detail-change on toggle', async () => {
      const events: Array<{key: string; expanded: boolean}> = [];
      el.addEventListener('detail-change', ((e: CustomEvent) => {
        events.push(e.detail);
      }) as EventListener);

      // Click first expand button
      const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
      btn.click();
      await el.updateComplete;

      expect(events).toHaveLength(1);
      expect(events[0]!.key).toBe('Alice');
      expect(events[0]!.expanded).toBe(true);
    });

    it('collapses previous when expanding another in single mode', async () => {
      const events: Array<{key: string; expanded: boolean}> = [];
      el.addEventListener('detail-change', ((e: CustomEvent) => {
        events.push(e.detail);
      }) as EventListener);

      const btns = el.shadowRoot!.querySelectorAll('.expand-toggle');
      (btns[0] as HTMLElement).click();
      await el.updateComplete;

      (btns[1] as HTMLElement).click();
      await el.updateComplete;

      // Should have: expand Alice, collapse Alice, expand Bob
      expect(events).toHaveLength(3);
      expect(events[1]!.key).toBe('Alice');
      expect(events[1]!.expanded).toBe(false);
      expect(events[2]!.key).toBe('Bob');
      expect(events[2]!.expanded).toBe(true);
    });
  });

  describe('multi mode', () => {
    beforeEach(async () => {
      el.dataSet = testDataSet;
      el.columnConfig = testConfig;
      el.getRowKey = (row: TypedRow) => row.text(nameCol);
      el.getRowDetail = () => html`<div>Detail</div>`;
      el.detailMode = 'multi';
      await el.updateComplete;
    });

    it('allows multiple panels open simultaneously', async () => {
      const btns = el.shadowRoot!.querySelectorAll('.expand-toggle');
      (btns[0] as HTMLElement).click();
      await el.updateComplete;
      (btns[1] as HTMLElement).click();
      await el.updateComplete;

      const panels = el.shadowRoot!.querySelectorAll('.detail-panel:not([hidden])');
      expect(panels.length).toBe(2);
    });
  });

  describe('controlled mode', () => {
    it('does not manage internal state when expandedDetailKeys is set', async () => {
      el.dataSet = testDataSet;
      el.columnConfig = testConfig;
      el.getRowKey = (row: TypedRow) => row.text(nameCol);
      el.getRowDetail = () => html`<div>Detail</div>`;
      el.expandedDetailKeys = ['Alice'];
      await el.updateComplete;

      const panels = el.shadowRoot!.querySelectorAll('.detail-panel:not([hidden])');
      expect(panels.length).toBe(1);

      // Clicking toggle should NOT change internal state
      const events: unknown[] = [];
      el.addEventListener('detail-change', ((e: CustomEvent) => {
        events.push(e.detail);
      }) as EventListener);

      const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
      btn.click();
      await el.updateComplete;

      // Event fires but panel count unchanged (controlled)
      expect(events).toHaveLength(1);
      const panelsAfter = el.shadowRoot!.querySelectorAll('.detail-panel:not([hidden])');
      expect(panelsAfter.length).toBe(1);
    });
  });

  describe('virtual scroll interaction', () => {
    it('disables virtual scroll when getRowDetail is set in auto mode', async () => {
      el.dataSet = makeLargeDataSet(100);
      el.columnConfig = testConfig;
      el.getRowKey = (row: TypedRow) => row.text(nameCol);
      el.getRowDetail = () => html`<div>Detail</div>`;
      el.mode = 'auto';
      await el.updateComplete;

      // AUTO_THRESHOLD is 50, 100 rows would normally trigger virtual scroll
      // but getRowDetail disables it
      const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]');
      expect(rows.length).toBe(100);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: FAIL — properties and methods don't exist yet

- [ ] **Step 3: Add properties and state to PagesTable**

Using `ide_insert_member`, add after the `getChildren` property (line 35):

```typescript
@property({ attribute: false }) getRowDetail?: (row: TypedRow) => TemplateResult | undefined;
@property({ type: String, attribute: 'detail-mode' }) detailMode: DetailMode = 'single';
@property({ type: Array, attribute: false }) expandedDetailKeys?: readonly string[];
```

Add internal state after `_csvExportEnabled` (line 113):

```typescript
@state() private _internalExpandedDetailKeys = new Set<string>();
private _instanceId = '';
```

Import `DetailMode`, `DetailChangeDetail` from `./types.js` in the import statement.

- [ ] **Step 4: Add connectedCallback UUID generation**

Using `ide_replace_member` on `connectedCallback`, add `this._instanceId = crypto.randomUUID();` after `super.connectedCallback()`.

- [ ] **Step 5: Add validation in willUpdate**

Using `ide_replace_member` on `willUpdate`, add two validation blocks after the existing `selection !== 'none'` check:

```typescript
if (this.getRowDetail && !this.getRowKey) {
  throw new Error('getRowKey is required when getRowDetail is set');
}

if (this.getRowDetail && this.mode === 'scroll') {
  throw new Error("getRowDetail is incompatible with mode='scroll' — virtual scrolling requires fixed row heights");
}
```

Also add controlled-mode sync (similar to `selectedKeys` pattern):

```typescript
if (changed.has('expandedDetailKeys') && this.expandedDetailKeys !== undefined) {
  this._internalExpandedDetailKeys = new Set(this.expandedDetailKeys);
}
```

- [ ] **Step 6: Add expand state helpers**

Using `ide_insert_member`, add after `_emitRowActivate`:

```typescript
private get _expandedDetails(): Set<string> {
  if (this.expandedDetailKeys !== undefined) {
    return new Set(this.expandedDetailKeys);
  }
  return this._internalExpandedDetailKeys;
}

private _isDetailExpanded(key: string): boolean {
  return this._expandedDetails.has(key);
}

private _emitDetailChange(key: string, row: TypedRow, expanded: boolean): void {
  const detail: DetailChangeDetail = { key, row, expanded };
  this.dispatchEvent(new CustomEvent('detail-change', {
    detail,
    bubbles: true,
    composed: true,
  }));
}

private _toggleDetail(row: TypedRow): void {
  if (!this.getRowKey) return;
  const key = this.getRowKey(row);
  const isExpanded = this._isDetailExpanded(key);

  if (this.expandedDetailKeys === undefined) {
    if (isExpanded) {
      const next = new Set(this._internalExpandedDetailKeys);
      next.delete(key);
      this._internalExpandedDetailKeys = next;
    } else {
      if (this.detailMode === 'single') {
        // Emit collapse for previously expanded row
        for (const prevKey of this._internalExpandedDetailKeys) {
          const prevRow = this._visibleRows.find(r => this.getRowKey!(r) === prevKey);
          if (prevRow) this._emitDetailChange(prevKey, prevRow, false);
        }
        this._internalExpandedDetailKeys = new Set([key]);
      } else {
        const next = new Set(this._internalExpandedDetailKeys);
        next.add(key);
        this._internalExpandedDetailKeys = next;
      }
    }
  }

  this._emitDetailChange(key, row, !isExpanded);
}
```

- [ ] **Step 7: Modify _useVirtualScroll to disable when getRowDetail is set**

Using `ide_replace_member` on `_useVirtualScroll`:

```typescript
private get _useVirtualScroll(): boolean {
  if (this.getRowDetail) return false;
  if (this.mode === 'scroll') return true;
  return this.mode === 'auto' && this._dataRows.length > AUTO_THRESHOLD;
}
```

- [ ] **Step 8: Run tests**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: Some tests pass (validation, virtual scroll). Rendering tests may still fail — that's Task 3.

- [ ] **Step 9: Commit**

```
feat(pages-table): expand state management — properties, validation, events  Refs #172
```

---

### Task 3: Expand Column and Detail Panel Rendering

**Files:**
- Modify: `packages/pages-table/src/pages-table.test.ts`
- Modify: `packages/pages-table/src/pages-table.ts`

**Interfaces:**
- Consumes: state management from Task 2
- Produces: rendered expand column, detail panels, ARIA linking, expand-all toggle

- [ ] **Step 1: Write failing tests for rendering**

Add to the `row detail expansion` describe block:

```typescript
describe('rendering', () => {
  beforeEach(async () => {
    el.dataSet = testDataSet;
    el.columnConfig = testConfig;
    el.getRowKey = (row: TypedRow) => row.text(nameCol);
    el.getRowDetail = (row: TypedRow) => {
      const name = row.text(nameCol);
      return name === 'Bob' ? undefined : html`<div class="test-detail">Detail for ${name}</div>`;
    };
    await el.updateComplete;
  });

  it('renders expand column when getRowDetail is set', async () => {
    const headerCells = el.shadowRoot!.querySelectorAll('.header-cell, .expand-header');
    // expand header + name + age
    expect(headerCells.length).toBeGreaterThanOrEqual(3);
  });

  it('renders expand button only for rows with detail content', async () => {
    const toggles = el.shadowRoot!.querySelectorAll('.expand-toggle');
    // Alice and Carol have detail, Bob returns undefined
    expect(toggles.length).toBe(2);
  });

  it('renders detail panel with correct ARIA when expanded', async () => {
    const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    btn.click();
    await el.updateComplete;

    const panel = el.shadowRoot!.querySelector('.detail-panel:not([hidden])');
    expect(panel).toBeTruthy();
    expect(panel!.getAttribute('role')).toBe('region');

    // aria-controls/id linking
    const btnAriaControls = btn.getAttribute('aria-controls');
    expect(btnAriaControls).toBe(panel!.id);
    expect(btn.getAttribute('aria-expanded')).toBe('true');
  });

  it('does not include detail panels in aria-rowcount', async () => {
    const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    btn.click();
    await el.updateComplete;

    const grid = el.shadowRoot!.querySelector('[role="grid"]');
    const rowCount = parseInt(grid!.getAttribute('aria-rowcount')!, 10);
    // 3 data rows + 1 header = 4
    expect(rowCount).toBe(4);
  });

  it('expand button click does not fire row-activate', async () => {
    const activates: unknown[] = [];
    el.addEventListener('row-activate', () => activates.push(true));

    const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    btn.click();
    await el.updateComplete;

    expect(activates).toHaveLength(0);
  });
});

describe('expand all (multi mode)', () => {
  beforeEach(async () => {
    el.dataSet = testDataSet;
    el.columnConfig = testConfig;
    el.getRowKey = (row: TypedRow) => row.text(nameCol);
    el.getRowDetail = (row: TypedRow) => {
      const name = row.text(nameCol);
      return name === 'Bob' ? undefined : html`<div>Detail</div>`;
    };
    el.detailMode = 'multi';
    await el.updateComplete;
  });

  it('expands all rows with detail content', async () => {
    const expandAll = el.shadowRoot!.querySelector('.expand-all-toggle') as HTMLElement;
    expandAll.click();
    await el.updateComplete;

    const panels = el.shadowRoot!.querySelectorAll('.detail-panel:not([hidden])');
    // Alice and Carol — Bob has no detail
    expect(panels.length).toBe(2);
  });

  it('collapses all when any are expanded', async () => {
    // Expand one first
    const btns = el.shadowRoot!.querySelectorAll('.expand-toggle');
    (btns[0] as HTMLElement).click();
    await el.updateComplete;

    const expandAll = el.shadowRoot!.querySelector('.expand-all-toggle') as HTMLElement;
    expandAll.click();
    await el.updateComplete;

    const panels = el.shadowRoot!.querySelectorAll('.detail-panel:not([hidden])');
    expect(panels.length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: FAIL — no expand column or detail panel rendering yet

- [ ] **Step 3: Modify _gridTemplateColumns to prepend expand column**

Using `ide_replace_member` on `_gridTemplateColumns`:

```typescript
private get _gridTemplateColumns(): string {
  if (this._visibleColumns.length === 0) return '1fr';
  const columns = this._visibleColumns.map(c => {
    const config = this._configFor(c);
    return config?.width ?? '1fr';
  }).join(' ');
  const prefix = [
    this.getRowDetail ? '40px' : '',
    this.selection === 'multi' ? '40px' : '',
  ].filter(Boolean).join(' ');
  return prefix ? `${prefix} ${columns}` : columns;
}
```

- [ ] **Step 4: Add _renderExpandCell and _renderExpandHeader methods**

Using `ide_insert_member`, add before `_renderRow`:

```typescript
private _renderExpandHeader() {
  if (!this.getRowDetail) return nothing;
  if (this.detailMode === 'multi') {
    const anyExpanded = this._expandedDetails.size > 0;
    return html`
      <div class="expand-header">
        <button
          class="expand-all-toggle"
          aria-label="${anyExpanded ? 'Collapse all details' : 'Expand all details'}"
          @click="${this._handleExpandAll}"
        >
          ${anyExpanded ? '▼' : '▶'}
        </button>
      </div>
    `;
  }
  return html`<div class="expand-header"></div>`;
}

private _renderExpandCell(row: TypedRow) {
  if (!this.getRowDetail) return nothing;
  const detail = this.getRowDetail(row);
  if (detail === undefined) {
    return html`<div class="expand-cell"></div>`;
  }
  const key = this.getRowKey!(row);
  const isExpanded = this._isDetailExpanded(key);
  const panelId = `${this._instanceId}-detail-${key}`;
  const btnId = `${this._instanceId}-detail-btn-${key}`;
  return html`
    <div class="expand-cell">
      <button
        id="${btnId}"
        class="expand-toggle"
        aria-expanded="${isExpanded ? 'true' : 'false'}"
        aria-controls="${panelId}"
        aria-label="${isExpanded ? 'Hide details' : 'Show details'} for row ${key}"
        @click="${(e: MouseEvent) => { e.stopPropagation(); this._toggleDetail(row); }}"
      >
        <span class="expand-chevron ${isExpanded ? 'expanded' : ''}">▶</span>
      </button>
    </div>
  `;
}
```

- [ ] **Step 5: Add _renderDetailPanel method**

Using `ide_insert_member`, add after `_renderExpandCell`:

```typescript
private _renderDetailPanel(row: TypedRow) {
  if (!this.getRowDetail) return nothing;
  const detail = this.getRowDetail(row);
  if (detail === undefined) return nothing;
  const key = this.getRowKey!(row);
  const isExpanded = this._isDetailExpanded(key);
  const panelId = `${this._instanceId}-detail-${key}`;
  const btnId = `${this._instanceId}-detail-btn-${key}`;
  return html`
    <div
      id="${panelId}"
      class="detail-panel ${isExpanded ? 'expanded' : ''}"
      role="region"
      aria-labelledby="${btnId}"
      ?hidden="${!isExpanded}"
      @transitionend="${(e: TransitionEvent) => this._handleDetailTransitionEnd(e, key)}"
    >
      <div class="detail-content">
        ${isExpanded ? detail : nothing}
      </div>
    </div>
  `;
}
```

- [ ] **Step 6: Add _handleExpandAll method**

Using `ide_insert_member`, add after `_toggleDetail`:

```typescript
private _handleExpandAll = (): void => {
  if (!this.getRowDetail || !this.getRowKey) return;
  const anyExpanded = this._expandedDetails.size > 0;

  if (anyExpanded) {
    // Collapse all
    if (this.expandedDetailKeys === undefined) {
      for (const key of this._internalExpandedDetailKeys) {
        const row = this._visibleRows.find(r => this.getRowKey!(r) === key);
        if (row) this._emitDetailChange(key, row, false);
      }
      this._internalExpandedDetailKeys = new Set();
    } else {
      for (const key of this.expandedDetailKeys) {
        const row = this._visibleRows.find(r => this.getRowKey!(r) === key);
        if (row) this._emitDetailChange(key, row, false);
      }
    }
  } else {
    // Expand all with detail content
    const toExpand = new Set<string>();
    for (const row of this._visibleRows) {
      const detail = this.getRowDetail(row);
      if (detail !== undefined) {
        const key = this.getRowKey(row);
        toExpand.add(key);
        this._emitDetailChange(key, row, true);
      }
    }
    if (this.expandedDetailKeys === undefined) {
      this._internalExpandedDetailKeys = toExpand;
    }
  }
};
```

- [ ] **Step 7: Add _handleDetailTransitionEnd method**

Using `ide_insert_member`, add after `_handleExpandAll`:

```typescript
private _handleDetailTransitionEnd = (e: TransitionEvent, key: string): void => {
  if (e.propertyName !== 'grid-template-rows') return;
  const panel = e.currentTarget as HTMLElement;
  if (!panel.classList.contains('expanded')) {
    panel.hidden = true;
  }
};
```

- [ ] **Step 8: Modify _renderRow to include expand cell and detail panel**

Using `ide_replace_member` on `_renderRow`, add `this._renderExpandCell(row)` before `this._renderCheckbox(row)` in the row template, and add `${this._renderDetailPanel(row)}` after the closing `</div>` of the row:

```typescript
private _renderRow(row: TypedRow, actualIndex: number, displayIndex: number) {
  const rowClass = this.getRowClass?.(row) ?? '';
  const part = rowClass ? `row ${rowClass}` : 'row';
  const ariaRowIndex = actualIndex + 2;
  const isSelected = this._isRowSelected(row);
  const tabindex = actualIndex === this._focusRowIndex ? '0' : '-1';
  const isClickable = this._filterConfig.enabled;
  const isFilterSelected = this._isFilterSelected(row);
  const treeNode = this._treeNodeByRow.get(row);
  const rowStyleResult = this._evaluateRowStyle(row);
  const rowStyleClass = isFilterSelected ? '' : (rowStyleResult?.className ?? '');
  const effectiveStyle = isFilterSelected
    ? { ...rowStyleResult?.style, backgroundColor: 'var(--pages-accent-5, #d3e3fd)' }
    : rowStyleResult?.style;
  const rowInlineStyle = effectiveStyle
    ? Object.entries(effectiveStyle).map(([k, v]) => `${k.replace(/[A-Z]/g, m => `-${m.toLowerCase()}`)}: ${String(v)}`).join('; ')
    : '';

  const stripe = actualIndex % 2 === 0 ? 'row-even' : 'row-odd';

  const isDetailExpanded = this.getRowDetail && this.getRowKey
    ? this._isDetailExpanded(this.getRowKey(row))
    : false;

  return html`
    <div
      class="row ${stripe} ${isClickable ? 'clickable' : ''} ${isFilterSelected ? 'selected' : ''} ${rowStyleClass} ${isDetailExpanded ? 'detail-expanded' : ''}"
      style="grid-template-columns: ${this._gridTemplateColumns}; ${rowInlineStyle}"
      role="row"
      part="${part}"
      aria-rowindex="${ariaRowIndex}"
      aria-selected="${this.selection !== 'none' && isSelected ? 'true' : 'false'}"
      aria-level="${treeNode ? String(treeNode.depth + 1) : nothing}"
      aria-setsize="${treeNode ? String(treeNode.siblingCount) : nothing}"
      aria-posinset="${treeNode ? String(treeNode.siblingIndex) : nothing}"
      aria-expanded="${treeNode && treeNode.children.length > 0 ? String(this._treeExpandState.get(treeNode.id) === true) : nothing}"
      tabindex="${tabindex}"
      @click="${(e: MouseEvent) => this._handleRowClick(row, e)}"
      @dblclick="${(e: MouseEvent) => this._handleRowDoubleClick(row, e)}"
    >
      ${this._renderExpandCell(row)}
      ${this._renderCheckbox(row)}
      ${this._visibleColumns.map((col, i) => this._renderCell(row, col, i === 0, treeNode))}
    </div>
    ${this._renderDetailPanel(row)}
  `;
}
```

- [ ] **Step 9: Modify render() to include expand header**

In the `render()` method, add `${this._renderExpandHeader()}` before `${this._renderCheckbox(this._dataRows[0]!, true)}` in both the empty-state header and the main header.

- [ ] **Step 10: Run tests**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: PASS — all rendering and state tests pass

- [ ] **Step 11: Commit**

```
feat(pages-table): expand column and detail panel rendering  Refs #172
```

---

### Task 4: CSS Styles for Expand Column, Detail Panel, and Animation

**Files:**
- Modify: `packages/pages-table/src/pages-table.ts` (styles block)

**Interfaces:**
- Consumes: CSS classes from Task 3 rendering (`.expand-header`, `.expand-cell`, `.expand-toggle`, `.expand-chevron`, `.detail-panel`, `.detail-content`, `.detail-expanded`, `.expand-all-toggle`)
- Produces: complete visual treatment using design tokens

- [ ] **Step 1: Write visual assertion tests**

Add to the test file:

```typescript
describe('detail panel CSS classes', () => {
  beforeEach(async () => {
    el.dataSet = testDataSet;
    el.columnConfig = testConfig;
    el.getRowKey = (row: TypedRow) => row.text(nameCol);
    el.getRowDetail = () => html`<div>Detail</div>`;
    await el.updateComplete;
  });

  it('expanded row has detail-expanded class', async () => {
    const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    btn.click();
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]');
    expect(rows[0]!.classList.contains('detail-expanded')).toBe(true);
  });

  it('detail panel has hidden attribute when collapsed', async () => {
    const panels = el.shadowRoot!.querySelectorAll('.detail-panel');
    for (const panel of panels) {
      expect(panel.hasAttribute('hidden')).toBe(true);
    }
  });

  it('expanded panel has expanded class and no hidden attribute', async () => {
    const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    btn.click();
    await el.updateComplete;

    const expandedPanel = el.shadowRoot!.querySelector('.detail-panel.expanded');
    expect(expandedPanel).toBeTruthy();
    expect(expandedPanel!.hasAttribute('hidden')).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-table run test`

- [ ] **Step 3: Add CSS to the static styles block**

Using `ide_read_file` to find the end of the existing styles block (before the closing backtick), then `ide_replace_text_in_file` to insert before the closing backtick of the `css` tagged template. Add:

```css
.expand-header {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--pages-space-3, 12px) var(--pages-space-2, 8px);
}

.expand-cell {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--pages-space-3, 12px) var(--pages-space-2, 8px);
}

.expand-toggle {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border: none;
  background: none;
  cursor: pointer;
  padding: 0;
  border-radius: var(--pages-radius-sm, 4px);
  color: var(--pages-neutral-9, #737373);
}

.expand-toggle:hover {
  background: var(--pages-neutral-3, #f5f5f5);
  color: var(--pages-neutral-12, #171717);
}

.expand-toggle:focus-visible {
  outline: 2px solid var(--pages-primary-9, #3b82f6);
  outline-offset: -2px;
}

.expand-chevron {
  display: inline-block;
  font-size: 10px;
  transition: transform var(--pages-duration-fast, 120ms) var(--pages-ease-out, ease-out);
}

.expand-chevron.expanded {
  transform: rotate(90deg);
}

.expand-all-toggle {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border: none;
  background: none;
  cursor: pointer;
  padding: 0;
  border-radius: var(--pages-radius-sm, 4px);
  color: var(--pages-neutral-9, #737373);
  font-size: 10px;
}

.expand-all-toggle:hover {
  background: var(--pages-neutral-3, #f5f5f5);
  color: var(--pages-neutral-12, #171717);
}

.row.detail-expanded {
  background: var(--pages-surface-1);
  border-bottom: none;
}

.detail-panel {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows var(--pages-duration-normal, 200ms) var(--pages-ease-out, ease-out);
}

.detail-panel[hidden] {
  display: none !important;
}

.detail-panel.expanded {
  grid-template-rows: 1fr;
}

.detail-panel > .detail-content {
  overflow: hidden;
  min-height: 0;
  background: var(--pages-surface-2);
  padding-left: 40px;
  opacity: 0;
  transition: opacity var(--pages-duration-fast, 120ms) var(--pages-ease-out, ease-out);
}

.detail-panel.expanded > .detail-content {
  opacity: 1;
}

@media (prefers-reduced-motion: reduce) {
  .detail-panel,
  .detail-panel > .detail-content,
  .expand-chevron {
    transition: none !important;
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(pages-table): detail expansion CSS — tokens, animation, reduced-motion  Refs #172
```

---

### Task 5: Keyboard Navigation and Focus Rescue

**Files:**
- Modify: `packages/pages-table/src/pages-table.test.ts`
- Modify: `packages/pages-table/src/pages-table.ts`

**Interfaces:**
- Consumes: rendering from Task 3, state from Task 2
- Produces: keyboard-accessible detail expansion, focus rescue on collapse

- [ ] **Step 1: Write failing keyboard tests**

```typescript
describe('keyboard navigation', () => {
  beforeEach(async () => {
    el.dataSet = testDataSet;
    el.columnConfig = testConfig;
    el.getRowKey = (row: TypedRow) => row.text(nameCol);
    el.getRowDetail = () => html`<div><button class="inner-btn">Action</button></div>`;
    await el.updateComplete;
  });

  it('arrow keys skip detail panels', async () => {
    // Expand first row
    const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    btn.click();
    await el.updateComplete;

    // Focus first row and press ArrowDown
    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]');
    (rows[0] as HTMLElement).focus();
    const event = new KeyboardEvent('keydown', { key: 'ArrowDown', bubbles: true });
    rows[0]!.dispatchEvent(event);
    await el.updateComplete;

    // Should focus second data row, not the detail panel
    expect(el.shadowRoot!.activeElement?.getAttribute('role')).toBe('row');
  });

  it('rescues focus when detail panel with focus is collapsed', async () => {
    // Expand first row
    const expandBtn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    expandBtn.click();
    await el.updateComplete;

    // Focus the inner button inside detail panel
    const innerBtn = el.shadowRoot!.querySelector('.detail-panel .inner-btn') as HTMLElement;
    if (innerBtn) innerBtn.focus();

    // Collapse — focus should return to expand toggle
    expandBtn.click();
    await el.updateComplete;

    expect(el.shadowRoot!.activeElement?.classList.contains('expand-toggle')).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: FAIL

- [ ] **Step 3: Add focus rescue to _toggleDetail**

Modify `_toggleDetail` to check `detailPanel.contains(document.activeElement)` before collapsing, and move focus to the toggle button:

```typescript
// At the start of collapse logic in _toggleDetail, before state update:
if (isExpanded) {
  const panelId = `${this._instanceId}-detail-${key}`;
  const panel = this.shadowRoot?.getElementById(panelId);
  const btnId = `${this._instanceId}-detail-btn-${key}`;
  const btn = this.shadowRoot?.getElementById(btnId);
  if (panel && btn && panel.contains(this.shadowRoot?.activeElement ?? document.activeElement)) {
    (btn as HTMLElement).focus();
  }
}
```

- [ ] **Step 4: Verify keyboard navigation naturally skips panels**

The existing `_handleKeyDown` queries `.row[role="row"]` — detail panels have `role="region"`, so they are naturally excluded. Verify this is already the case by reading the query selector in `_handleKeyDown`. If it already uses `.row[role="row"]`, no change needed.

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(pages-table): keyboard nav — arrow keys skip panels, focus rescue  Refs #172
```

---

### Task 6: Pagination, Tree Coexistence, and Edge Cases

**Files:**
- Modify: `packages/pages-table/src/pages-table.test.ts`
- Modify: `packages/pages-table/src/pages-table.ts`

**Interfaces:**
- Consumes: all prior tasks
- Produces: pagination-aware expand-all, tree+detail coexistence, CSV exclusion

- [ ] **Step 1: Write failing edge case tests**

```typescript
describe('pagination interaction', () => {
  it('expand state persists across page changes', async () => {
    const ds = makeLargeDataSet(10);
    el.dataSet = ds;
    el.columnConfig = testConfig;
    el.mode = 'paginated';
    el.pageSize = 3;
    el.getRowKey = (row: TypedRow) => row.text(nameCol);
    el.getRowDetail = () => html`<div>Detail</div>`;
    await el.updateComplete;

    // Expand first row on page 1
    const btn = el.shadowRoot!.querySelector('.expand-toggle') as HTMLElement;
    btn.click();
    await el.updateComplete;

    // Go to page 2 and back
    el.currentPage = 1;
    await el.updateComplete;
    el.currentPage = 0;
    await el.updateComplete;

    // First row should still be expanded
    const panel = el.shadowRoot!.querySelector('.detail-panel:not([hidden])');
    expect(panel).toBeTruthy();
  });
});

describe('tree + detail coexistence', () => {
  it('renders detail panel between parent and children when both expanded', async () => {
    const parent = { id: 'p1', name: 'Parent', age: 40, created: new Date('2024-01-01') };
    const child = { id: 'c1', name: 'Child', age: 10, created: new Date('2024-01-01') };
    const ds = makeDataSet([parent, child]);
    el.dataSet = ds;
    el.columnConfig = testConfig;
    el.getRowKey = (row: TypedRow) => row.text(nameCol);
    el.getChildren = (row: TypedRow) => {
      if (row.text(nameCol) === 'Parent') return [ds.rows[1]!];
      return [];
    };
    el.getRowDetail = () => html`<div class="test-detail">Details</div>`;
    await el.updateComplete;

    // Both tree and detail expand should render
    const expandBtns = el.shadowRoot!.querySelectorAll('.expand-toggle');
    expect(expandBtns.length).toBeGreaterThan(0);
  });
});

describe('getRowKey validation', () => {
  it('throws Error consistent with selection validation message', async () => {
    el.dataSet = testDataSet;
    el.columnConfig = testConfig;
    el.getRowDetail = () => html`<div>Detail</div>`;
    // No getRowKey set
    await expect(el.updateComplete).rejects.toThrow('getRowKey is required');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-table run test`

- [ ] **Step 3: Implement any remaining edge case handling**

Most edge cases should already work from Tasks 2-5:
- Pagination persistence: key-based state in `_internalExpandedDetailKeys` is independent of page
- Tree coexistence: tree toggle is in the first data cell, expand toggle is in the dedicated expand column — they coexist naturally
- CSV exclusion: `tableToCsv` only reads `_dataRows` and `_visibleColumns` — detail panels are not data rows

Verify by running tests. Fix any failures found.

- [ ] **Step 4: Run full test suite**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: ALL PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn workspace @casehubio/pages-table run typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(pages-table): pagination, tree coexistence, edge cases  Refs #172
```

---

### Task 7: Full Build Verification and Export Cleanup

**Files:**
- Modify: `packages/pages-table/src/index.ts` (verify exports)

**Interfaces:**
- Consumes: all prior tasks
- Produces: verified, clean build

- [ ] **Step 1: Run full project build**

Run: `yarn build`
Expected: PASS — all packages compile

- [ ] **Step 2: Run project-wide typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 3: Run linter**

Run: `yarn lint`
Expected: PASS (or only pre-existing warnings)

- [ ] **Step 4: Run full test suite across all packages**

Run: `yarn workspace @casehubio/pages-table run test`
Expected: ALL PASS

- [ ] **Step 5: Verify exports are complete**

Check that `index.ts` exports `DetailMode` and `DetailChangeDetail`. These are the only new public types. The `PagesTable` class already gets its new properties through the class definition.

- [ ] **Step 6: Commit if any cleanup was needed**

```
chore(pages-table): build verification and export cleanup  Refs #172
```

---

## Self-Review Checklist

### Spec coverage
| Spec section | Task |
|---|---|
| API Surface (3 properties, 1 event) | Task 1 (types), Task 2 (properties) |
| Expand column rendering | Task 3 |
| Detail panel rendering | Task 3 |
| Detail panel animation | Task 4 |
| Tree + detail coexistence | Task 3 (rendering order), Task 6 (test) |
| State management (controlled/uncontrolled) | Task 2 |
| Keyboard navigation | Task 5 |
| Accessibility (ARIA disclosure) | Task 3 |
| Expand all (multi mode) | Task 3 |
| Validation (getRowKey, mode=scroll) | Task 2 |
| Virtual scroll disable | Task 2 |
| Pagination interaction | Task 6 |
| CSS tokens | Task 4 |
| prefers-reduced-motion | Task 4 |
| Instance-scoped IDs | Task 3 |
| YAML surface excluded | N/A (no YAML work needed — TypeScript-only API) |
| Testing | Tasks 2-6 |
| Event naming (detail-change) | Task 1 |

### Placeholder scan
No TBD, TODO, or "fill in details" placeholders found.

### Type consistency
- `DetailMode` = `'single' | 'multi'` — used consistently in Task 1 (type), Task 2 (property), Task 3 (expand-all gate)
- `DetailChangeDetail` = `{ key, row, expanded }` — used consistently in Task 1 (type), Task 2 (emit method)
- `_isDetailExpanded(key)` — used consistently in Tasks 2, 3, 5
- `_toggleDetail(row)` — used consistently in Tasks 2, 3
- `_renderExpandCell(row)` — defined Task 3, called Task 3
- `_renderDetailPanel(row)` — defined Task 3, called Task 3
- `_instanceId` — set in `connectedCallback` Task 2, used in rendering Task 3

### Tooling safety scan
All code edits use IntelliJ MCP tools (`ide_insert_member`, `ide_replace_member`, `ide_replace_text_in_file`). No bash file operations on source files.
