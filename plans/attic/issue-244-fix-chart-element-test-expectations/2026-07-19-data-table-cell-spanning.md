# Data Table Cell Spanning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #26 — pages-data-table: row and column spanning support
**Issue group:** #26

**Goal:** Add row and column spanning to pages-table by replacing the
per-row CSS Grid model with a single body grid, enabling native CSS Grid
`grid-row: span N` and `grid-column: span N`.

**Architecture:** Single CSS Grid on `.body-content` with `display: contents`
row wrappers. SpanMap pre-computed in `willUpdate` from per-column `cellSpan`
callbacks and `mergeRows` shorthand. Virtual scroll extended with boundary-scan
for span origins above viewport. All cells explicitly placed via `grid-row` /
`grid-column` inline styles.

**Tech Stack:** Lit 3, TypeScript, CSS Grid, Vitest, Playwright

**Implementation repo:** `casehub-pages` (`packages/pages-table/`,
`packages/pages-component/`)

**Spec:** `casehub-blocks-ui/docs/specs/2026-07-19-data-table-cell-spanning-design.md`

## Global Constraints

- pages-table is `@casehubio/pages-table` — the authoritative table component
- `TableColumnConfig` base type lives in `packages/pages-component/src/model/displayer-types.ts`
- pages-table local extension in `packages/pages-table/src/types.ts`
- Virtual scroll engine is a pure function in `virtual-scroll-engine.ts`
- Tests: Vitest for unit/DOM, Playwright for visual (existing infra in `examples/`)
- Fixed row heights required for virtual scroll — `rowHeight` property (default 48px)
- Tree rows and `groupBy` are mutually exclusive with spanning (throw in willUpdate)

---

### Task 1: SpanMap types and computation module

**Files:**
- Create: `packages/pages-table/src/span-map.ts`
- Test: `packages/pages-table/src/span-map.test.ts`

**Interfaces:**
- Consumes: `TypedRow`, `CellValue`, `Column`, `ColumnId` from `@casehubio/pages-data`; `TableColumnConfig` from `./types.js`
- Produces:
  - `CellSpan { colSpan: number; rowSpan: number }`
  - `SuppressedCell { originRow: number; originCol: string }`
  - `SpanMap` = `Map<number, Map<string, CellSpan | SuppressedCell>>`
  - `computeSpanMap(rows: readonly TypedRow[], columns: readonly Column[], config: readonly TableColumnConfig[], visibleColIds: Set<string>): SpanMap`
  - `isSuppressed(entry: CellSpan | SuppressedCell): entry is SuppressedCell`
  - `isOrigin(entry: CellSpan | SuppressedCell): entry is CellSpan`

- [ ] **Step 1: Write failing tests for mergeRows: true**

```typescript
// span-map.test.ts
import { describe, it, expect } from 'vitest';
import { computeSpanMap, isSuppressed, isOrigin } from './span-map.js';
import { fromRows } from '@casehubio/pages-data';
import { ColumnType } from '@casehubio/pages-data';
import type { ColumnId } from '@casehubio/pages-data';
import type { TableColumnConfig } from './types.js';

const countryCol = 'country' as ColumnId;
const nameCol = 'name' as ColumnId;

const columns = [
  { id: countryCol, name: 'Country', type: ColumnType.TEXT, getValue: (r: any) => r.country },
  { id: nameCol, name: 'Name', type: ColumnType.TEXT, getValue: (r: any) => r.name },
];

describe('computeSpanMap', () => {
  describe('mergeRows: true', () => {
    it('merges adjacent rows with equal values', () => {
      const data = [
        { country: 'USA', name: 'Alice' },
        { country: 'USA', name: 'Bob' },
        { country: 'USA', name: 'Carol' },
        { country: 'UK', name: 'Dave' },
      ];
      const ds = fromRows(data, columns);
      const config: TableColumnConfig[] = [
        { id: countryCol, mergeRows: true },
        { id: nameCol },
      ];
      const visible = new Set([String(countryCol), String(nameCol)]);
      const map = computeSpanMap(ds.rows, ds.columns, config, visible);

      // Row 0 country: origin with rowSpan 3
      const r0 = map.get(0)?.get(String(countryCol));
      expect(r0).toBeTruthy();
      expect(isOrigin(r0!)).toBe(true);
      expect((r0 as any).rowSpan).toBe(3);
      expect((r0 as any).colSpan).toBe(1);

      // Row 1 country: suppressed, origin at row 0
      const r1 = map.get(1)?.get(String(countryCol));
      expect(r1).toBeTruthy();
      expect(isSuppressed(r1!)).toBe(true);
      expect((r1 as any).originRow).toBe(0);

      // Row 2 country: suppressed
      const r2 = map.get(2)?.get(String(countryCol));
      expect(isSuppressed(r2!)).toBe(true);

      // Row 3 country: not in map (single row, no span)
      const r3 = map.get(3)?.get(String(countryCol));
      expect(r3).toBeUndefined();

      // Name column: no spans at all
      expect(map.get(0)?.get(String(nameCol))).toBeUndefined();
    });

    it('handles all-same-value column (single large span)', () => {
      const data = Array.from({ length: 5 }, () => ({ country: 'USA', name: 'X' }));
      const ds = fromRows(data, columns);
      const config: TableColumnConfig[] = [{ id: countryCol, mergeRows: true }, { id: nameCol }];
      const map = computeSpanMap(ds.rows, ds.columns, config, new Set([String(countryCol), String(nameCol)]));

      const origin = map.get(0)?.get(String(countryCol));
      expect(isOrigin(origin!)).toBe(true);
      expect((origin as any).rowSpan).toBe(5);

      for (let i = 1; i < 5; i++) {
        expect(isSuppressed(map.get(i)!.get(String(countryCol))!)).toBe(true);
      }
    });

    it('handles all-unique values (no spans)', () => {
      const data = [
        { country: 'A', name: 'a' },
        { country: 'B', name: 'b' },
        { country: 'C', name: 'c' },
      ];
      const ds = fromRows(data, columns);
      const config: TableColumnConfig[] = [{ id: countryCol, mergeRows: true }, { id: nameCol }];
      const map = computeSpanMap(ds.rows, ds.columns, config, new Set([String(countryCol), String(nameCol)]));
      expect(map.size).toBe(0);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/span-map.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement SpanMap types and mergeRows computation**

Create `packages/pages-table/src/span-map.ts`:

```typescript
import type { TypedRow, Column, ColumnId, CellValue } from '@casehubio/pages-data';
import type { TableColumnConfig } from './types.js';

export interface CellSpan {
  readonly colSpan: number;
  readonly rowSpan: number;
}

export interface SuppressedCell {
  readonly originRow: number;
  readonly originCol: string;
}

export type SpanEntry = CellSpan | SuppressedCell;
export type SpanMap = Map<number, Map<string, SpanEntry>>;

export function isSuppressed(entry: SpanEntry): entry is SuppressedCell {
  return 'originRow' in entry;
}

export function isOrigin(entry: SpanEntry): entry is CellSpan {
  return 'colSpan' in entry;
}

function getOrCreateRow(map: SpanMap, rowIndex: number): Map<string, SpanEntry> {
  let row = map.get(rowIndex);
  if (!row) {
    row = new Map();
    map.set(rowIndex, row);
  }
  return row;
}

export function computeSpanMap(
  rows: readonly TypedRow[],
  columns: readonly Column[],
  config: readonly TableColumnConfig[],
  visibleColIds: Set<string>,
): SpanMap {
  const map: SpanMap = new Map();

  for (const col of columns) {
    const colId = String(col.id);
    if (!visibleColIds.has(colId)) continue;
    const cfg = config.find(c => String(c.id) === colId);
    if (!cfg) continue;

    // Phase 1: mergeRows
    if (cfg.mergeRows) {
      const comparator = typeof cfg.mergeRows === 'function'
        ? cfg.mergeRows
        : (a: CellValue, b: CellValue) => {
            if (a.type === 'NULL' && b.type === 'NULL') return true;
            if (a.type === 'NULL' || b.type === 'NULL') return false;
            return a.value === b.value;
          };

      let runStart = 0;
      for (let i = 1; i <= rows.length; i++) {
        const shouldMerge = i < rows.length &&
          comparator(rows[i]!.cell(col.id), rows[runStart]!.cell(col.id));

        if (!shouldMerge) {
          const runLength = i - runStart;
          if (runLength > 1) {
            getOrCreateRow(map, runStart).set(colId, { colSpan: 1, rowSpan: runLength });
            for (let j = runStart + 1; j < i; j++) {
              getOrCreateRow(map, j).set(colId, { originRow: runStart, originCol: colId });
            }
          }
          runStart = i;
        }
      }
    }

    // Phase 2: cellSpan overrides
    if (cfg.cellSpan) {
      for (let i = 0; i < rows.length; i++) {
        const span = cfg.cellSpan(rows[i]!, i);
        if (span === undefined) continue;

        const colSpan = span.colSpan ?? 1;
        const rowSpan = span.rowSpan ?? 1;
        if (colSpan <= 1 && rowSpan <= 1) continue;

        const clampedRowSpan = Math.min(rowSpan, rows.length - i);
        const colIds = [...visibleColIds];
        const colIndex = colIds.indexOf(colId);
        const clampedColSpan = Math.min(colSpan, colIds.length - colIndex);

        getOrCreateRow(map, i).set(colId, { colSpan: clampedColSpan, rowSpan: clampedRowSpan });

        // Mark suppressed cells
        for (let r = i; r < i + clampedRowSpan; r++) {
          for (let c = colIndex; c < colIndex + clampedColSpan; c++) {
            if (r === i && c === colIndex) continue; // skip origin
            const suppColId = colIds[c]!;
            const row = getOrCreateRow(map, r);
            if (!row.has(suppColId)) {
              row.set(suppColId, { originRow: i, originCol: colId });
            }
          }
        }
      }
    }
  }

  return map;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/span-map.test.ts`
Expected: PASS

- [ ] **Step 5: Add tests for cellSpan callback, overrides, and clamping**

Add to `span-map.test.ts`:

```typescript
  describe('cellSpan callback', () => {
    it('applies explicit colSpan', () => {
      const data = [
        { country: 'USA', name: 'Alice' },
        { country: 'UK', name: 'Bob' },
      ];
      const ds = fromRows(data, columns);
      const config: TableColumnConfig[] = [
        { id: countryCol, cellSpan: (_row, i) => i === 0 ? { colSpan: 2 } : undefined },
        { id: nameCol },
      ];
      const visible = new Set([String(countryCol), String(nameCol)]);
      const map = computeSpanMap(ds.rows, ds.columns, config, visible);

      const origin = map.get(0)?.get(String(countryCol));
      expect(isOrigin(origin!)).toBe(true);
      expect((origin as any).colSpan).toBe(2);

      // name column at row 0 is suppressed
      const suppressed = map.get(0)?.get(String(nameCol));
      expect(isSuppressed(suppressed!)).toBe(true);
    });

    it('cellSpan overrides mergeRows for specific rows', () => {
      const data = [
        { country: 'USA', name: 'Alice' },
        { country: 'USA', name: 'Bob' },
        { country: 'USA', name: 'Carol' },
      ];
      const ds = fromRows(data, columns);
      const config: TableColumnConfig[] = [
        { id: countryCol, mergeRows: true,
          cellSpan: (_row, i) => i === 0 ? { colSpan: 2 } : undefined },
        { id: nameCol },
      ];
      const visible = new Set([String(countryCol), String(nameCol)]);
      const map = computeSpanMap(ds.rows, ds.columns, config, visible);

      // Row 0: cellSpan wins — colSpan 2 (overrides mergeRows rowSpan)
      const r0 = map.get(0)?.get(String(countryCol));
      expect(isOrigin(r0!)).toBe(true);
      expect((r0 as any).colSpan).toBe(2);
    });

    it('clamps spans past last row', () => {
      const data = [{ country: 'A', name: 'a' }, { country: 'B', name: 'b' }];
      const ds = fromRows(data, columns);
      const config: TableColumnConfig[] = [
        { id: countryCol, cellSpan: (_row, i) => i === 1 ? { rowSpan: 5 } : undefined },
        { id: nameCol },
      ];
      const visible = new Set([String(countryCol), String(nameCol)]);
      const map = computeSpanMap(ds.rows, ds.columns, config, visible);

      const origin = map.get(1)?.get(String(countryCol));
      expect(isOrigin(origin!)).toBe(true);
      expect((origin as any).rowSpan).toBe(1); // clamped: only 1 row left
    });
  });

  describe('mergeRows callback', () => {
    it('uses custom comparator', () => {
      const data = [
        { country: 'usa', name: 'Alice' },
        { country: 'USA', name: 'Bob' },
        { country: 'UK', name: 'Carol' },
      ];
      const ds = fromRows(data, columns);
      const config: TableColumnConfig[] = [
        { id: countryCol, mergeRows: (a, b) =>
          a.type !== 'NULL' && b.type !== 'NULL' &&
          String(a.value).toLowerCase() === String(b.value).toLowerCase() },
        { id: nameCol },
      ];
      const visible = new Set([String(countryCol), String(nameCol)]);
      const map = computeSpanMap(ds.rows, ds.columns, config, visible);

      const origin = map.get(0)?.get(String(countryCol));
      expect(isOrigin(origin!)).toBe(true);
      expect((origin as any).rowSpan).toBe(2); // usa + USA merged
    });
  });
```

- [ ] **Step 6: Run tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/span-map.test.ts`
Expected: PASS

- [ ] **Step 7: Export from index.ts and commit**

Add to `packages/pages-table/src/index.ts`:
```typescript
export { computeSpanMap, isSuppressed, isOrigin, type CellSpan, type SuppressedCell, type SpanMap, type SpanEntry } from './span-map.js';
```

Commit:
```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-table/src/span-map.ts packages/pages-table/src/span-map.test.ts packages/pages-table/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#26): SpanMap types and computation — mergeRows + cellSpan"
```

---

### Task 2: Span-aware virtual scroll extension

**Files:**
- Modify: `packages/pages-table/src/virtual-scroll-engine.ts`
- Modify: `packages/pages-table/src/virtual-scroll-engine.test.ts`

**Interfaces:**
- Consumes: `SpanMap`, `SuppressedCell`, `CellSpan`, `isSuppressed`, `isOrigin` from `./span-map.js`; `ScrollWindow` from `./virtual-scroll-engine.js`
- Produces:
  - `extendWindowForSpans(window: ScrollWindow, spanMap: SpanMap, spanColumns: ReadonlySet<string>): ScrollWindow`

- [ ] **Step 1: Write failing tests**

Add to `virtual-scroll-engine.test.ts`:

```typescript
import { extendWindowForSpans } from './virtual-scroll-engine.js';
import type { SpanMap } from './span-map.js';

describe('extendWindowForSpans', () => {
  it('extends startIndex when a suppressed cell points to an earlier origin', () => {
    const spanMap: SpanMap = new Map([
      [5, new Map([['country', { colSpan: 1, rowSpan: 4 }]])],
      [6, new Map([['country', { originRow: 5, originCol: 'country' }]])],
      [7, new Map([['country', { originRow: 5, originCol: 'country' }]])],
      [8, new Map([['country', { originRow: 5, originCol: 'country' }]])],
    ]);
    const window = { startIndex: 7, endIndex: 20, offsetY: 336, totalHeight: 4800 };
    const result = extendWindowForSpans(window, spanMap, new Set(['country']));
    expect(result.startIndex).toBe(5);
  });

  it('returns unchanged window when no spans at boundaries', () => {
    const spanMap: SpanMap = new Map();
    const window = { startIndex: 10, endIndex: 25, offsetY: 480, totalHeight: 4800 };
    const result = extendWindowForSpans(window, spanMap, new Set(['country']));
    expect(result.startIndex).toBe(10);
    expect(result.endIndex).toBe(25);
  });

  it('extends endIndex when an origin span exceeds the window', () => {
    const spanMap: SpanMap = new Map([
      [23, new Map([['country', { colSpan: 1, rowSpan: 5 }]])],
    ]);
    const window = { startIndex: 10, endIndex: 25, offsetY: 480, totalHeight: 4800 };
    const result = extendWindowForSpans(window, spanMap, new Set(['country']));
    expect(result.endIndex).toBe(28); // 23 + 5
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/virtual-scroll-engine.test.ts`
Expected: FAIL — extendWindowForSpans not found

- [ ] **Step 3: Implement extendWindowForSpans**

Add to `packages/pages-table/src/virtual-scroll-engine.ts`:

```typescript
import type { SpanMap } from './span-map.js';
import { isSuppressed, isOrigin } from './span-map.js';

export function extendWindowForSpans(
  window: ScrollWindow,
  spanMap: SpanMap,
  spanColumns: ReadonlySet<string>,
): ScrollWindow {
  if (spanMap.size === 0 || spanColumns.size === 0) return window;

  let { startIndex, endIndex } = window;

  // Top boundary: check if cells at startIndex are suppressed
  for (const colId of spanColumns) {
    const entry = spanMap.get(startIndex)?.get(colId);
    if (entry && isSuppressed(entry)) {
      startIndex = Math.min(startIndex, entry.originRow);
    }
  }

  // Bottom boundary: check if origins near endIndex extend beyond it
  for (let r = Math.max(startIndex, endIndex - 10); r < endIndex; r++) {
    const rowEntries = spanMap.get(r);
    if (!rowEntries) continue;
    for (const [colId, entry] of rowEntries) {
      if (!spanColumns.has(colId)) continue;
      if (isOrigin(entry) && r + entry.rowSpan > endIndex) {
        endIndex = Math.max(endIndex, r + entry.rowSpan);
      }
    }
  }

  return { ...window, startIndex, endIndex };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/virtual-scroll-engine.test.ts`
Expected: PASS

- [ ] **Step 5: Export and commit**

Add to `packages/pages-table/src/index.ts`:
```typescript
export { extendWindowForSpans } from './virtual-scroll-engine.js';
```

Commit:
```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-table/src/virtual-scroll-engine.ts packages/pages-table/src/virtual-scroll-engine.test.ts packages/pages-table/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#26): span-aware virtual scroll — extendWindowForSpans"
```

---

### Task 3: Add cellSpan and mergeRows to TableColumnConfig

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts`
- Modify: `packages/pages-table/src/types.ts` (re-export)

**Interfaces:**
- Consumes: `CellValue`, `TypedRow` from `@casehubio/pages-data`
- Produces: Extended `TableColumnConfig` with:
  - `cellSpan?: (row: TypedRow, rowIndex: number) => { colSpan?: number; rowSpan?: number } | undefined`
  - `mergeRows?: boolean | ((valueA: CellValue, valueB: CellValue) => boolean)`

- [ ] **Step 1: Add properties to BaseTableColumnConfig**

In `packages/pages-component/src/model/displayer-types.ts`, add to `TableColumnConfig`:

```typescript
  readonly cellSpan?: (row: TypedRow, rowIndex: number) =>
    { colSpan?: number; rowSpan?: number } | undefined;
  readonly mergeRows?: boolean | ((valueA: CellValue, valueB: CellValue) => boolean);
```

- [ ] **Step 2: Verify build**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages typecheck`
Expected: PASS (new optional properties are backward-compatible)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/displayer-types.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#26): add cellSpan and mergeRows to TableColumnConfig"
```

---

### Task 4: Single-grid rendering model

This is the core refactor. Replace per-row grid containers with a single
CSS Grid on `.body-content`. All existing tests must pass after this change.

**Files:**
- Modify: `packages/pages-table/src/pages-table.ts`
- Modify: `packages/pages-table/src/pages-table.test.ts` (adapt assertions)

**Interfaces:**
- Consumes: All existing properties and types
- Produces: Same public API, different internal rendering

The change is internal to `_renderRow`, `render()`, and associated CSS. No
public property or event changes.

**Key changes:**

1. `.body-content` gets `display: grid; grid-template-columns; grid-template-rows`
2. `.row` divs get `display: contents`
3. Each cell gets explicit `grid-row: N; grid-column: M` inline styles
4. Virtual scroll: remove `translateY` wrapper; cells placed at actual grid-row
5. Row hover: add `_hoverRowIndex` reactive state, `mouseenter`/`mouseleave` handlers
6. Row focus ring: per-cell `.focus-row` class instead of `.row:focus { outline }`
7. `::part(row)` → `::part(cell ...)` with row class forwarded to cells
8. CSS: remove `.row { display: grid }`, add `.body-content { display: grid }`

This task is large. Break the implementation into sub-steps:

- [ ] **Step 1: Update CSS — remove per-row grid, add body grid**

In static styles, change `.row` from `display: grid` to `display: contents`.
Add grid layout to `.body-content`. Update hover/focus/selection CSS to be
cell-based.

- [ ] **Step 2: Update `_renderRow` — cells get explicit grid placement**

Each cell receives `grid-row: ${actualIndex + 1}; grid-column: ${colIndex + 1}`
style. Row wrapper gets `display: contents` and `data-row="${actualIndex}"`.

- [ ] **Step 3: Update `render()` — body-content becomes a grid**

Replace the three rendering branches (groupBy, virtualScroll, normal) to
render `.body-content` as `display: grid` with `grid-template-columns` and
`grid-template-rows: repeat(N, ${rowHeight}px)`.

- [ ] **Step 4: Hover — add `_hoverRowIndex` state and cell handlers**

Add `@state() private _hoverRowIndex = -1`. On cell `mouseenter`, set to
row index. On `.body-content` `mouseleave`, reset to -1. Apply hover class
per-cell during render.

- [ ] **Step 5: Focus ring — per-cell instead of row**

Move `tabindex` handling to work with `display: contents` rows (tabindex
on the row div still works for DOM focus; visual ring via per-cell class).

- [ ] **Step 6: Run existing test suite — all tests must pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/pages-table.test.ts`
Expected: PASS — behavior identical, rendering model different

Some tests may need adjustment if they assert on `.row { display: grid }` or
query `.row` styles. Fix assertions to match the new model.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-table/src/pages-table.ts packages/pages-table/src/pages-table.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor(#26): single CSS Grid body — replace per-row grid model

Row wrappers use display: contents. Cells explicitly placed via grid-row/
grid-column. Hover and focus ring are cell-based. Virtual scroll uses grid
track placement instead of translateY."
```

---

### Task 5: Integrate SpanMap into rendering

Connect the SpanMap to the rendering pipeline. Cells with spans get
`grid-row: span N` / `grid-column: span N`. Suppressed cells are not
rendered.

**Files:**
- Modify: `packages/pages-table/src/pages-table.ts`
- Test: `packages/pages-table/src/pages-table.test.ts`

**Interfaces:**
- Consumes: `computeSpanMap`, `SpanMap`, `isSuppressed`, `isOrigin`, `extendWindowForSpans` from span-map and virtual-scroll-engine
- Produces: Rendered spans in the DOM

- [ ] **Step 1: Write failing tests for span rendering**

Add to `pages-table.test.ts`:

```typescript
describe('cell spanning', () => {
  it('renders rowspan via mergeRows: true', async () => {
    const items = [
      { id: '1', name: 'Alice', age: 30, created: new Date('2024-01-01') },
      { id: '2', name: 'Alice', age: 25, created: new Date('2024-01-01') },
      { id: '3', name: 'Bob', age: 35, created: new Date('2024-01-01') },
    ];
    el.dataSet = makeDataSet(items);
    el.columnConfig = [
      { id: nameCol, width: '1fr', mergeRows: true },
      { id: ageCol, width: '80px' },
    ];
    await el.updateComplete;

    const cells = el.shadowRoot!.querySelectorAll('.cell');
    const nameCells = [...cells].filter(c =>
      c.getAttribute('style')?.includes('grid-column: 1'));
    // Only 2 name cells rendered (Alice span + Bob), not 3
    expect(nameCells.length).toBe(2);

    const aliceCell = nameCells[0]!;
    expect(aliceCell.getAttribute('style')).toContain('span 2');
    expect(aliceCell.getAttribute('aria-rowspan')).toBe('2');
  });

  it('renders colspan via cellSpan', async () => {
    el.dataSet = makeDataSet(testItems);
    el.columnConfig = [
      { id: nameCol, width: '1fr',
        cellSpan: (_row: any, i: number) => i === 0 ? { colSpan: 2 } : undefined },
      { id: ageCol, width: '80px' },
    ];
    await el.updateComplete;

    const row0cells = [...el.shadowRoot!.querySelectorAll('.cell')]
      .filter(c => c.getAttribute('style')?.includes('grid-row: 1'));
    // Row 0: only 1 cell (name spans both columns)
    expect(row0cells.length).toBe(1);
    expect(row0cells[0]!.getAttribute('style')).toContain('grid-column: 1 / span 2');
  });

  it('throws when spanning combined with tree rows', async () => {
    el.dataSet = makeDataSet(testItems);
    el.columnConfig = [{ id: nameCol, mergeRows: true }, { id: ageCol }];
    el.getChildren = () => [];
    el.getRowKey = (r: any) => r.cell(nameCol).value;
    expect(() => (el as any).willUpdate(new Map([['dataSet', undefined]]))).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/pages-table.test.ts -- -t "cell spanning"`
Expected: FAIL

- [ ] **Step 3: Integrate SpanMap computation in willUpdate**

Add to `willUpdate`:
- Compute `this._spanMap` when `dataSet`, `columnConfig`, sort, filter, or visible columns change
- Add mutual exclusion throws for tree + spanning and groupBy + spanning
- Track `this._spanColumns` (Set of column IDs that have span config)

- [ ] **Step 4: Update cell rendering to use SpanMap**

In `_renderRow` / `_renderCell`:
- Check SpanMap for each cell
- Skip suppressed cells
- Add `grid-row: span N` / `grid-column: span N` for origin cells
- Add `aria-rowspan` / `aria-colspan` for origin cells

- [ ] **Step 5: Update virtual scroll to use extendWindowForSpans**

In the render method's virtual scroll branch, call `extendWindowForSpans`
after `computeScrollWindow` and render extra origin cells.

- [ ] **Step 6: Run full test suite**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-table/src/pages-table.ts packages/pages-table/src/pages-table.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#26): integrate SpanMap into rendering — cell spanning works"
```

---

### Task 6: Span-aware interactions

**Files:**
- Modify: `packages/pages-table/src/pages-table.ts`
- Test: `packages/pages-table/src/pages-table.test.ts`

- [ ] **Step 1: Write failing tests for span interactions**

```typescript
describe('span interactions', () => {
  const spanItems = [
    { id: '1', name: 'Alice', age: 30, created: new Date('2024-01-01') },
    { id: '2', name: 'Alice', age: 25, created: new Date('2024-01-01') },
    { id: '3', name: 'Alice', age: 35, created: new Date('2024-01-01') },
    { id: '4', name: 'Bob', age: 28, created: new Date('2024-01-01') },
  ];

  it('hover highlights all rows in span', async () => {
    el.dataSet = makeDataSet(spanItems);
    el.columnConfig = [{ id: nameCol, width: '1fr', mergeRows: true }, { id: ageCol, width: '80px' }];
    await el.updateComplete;
    const spanCell = el.shadowRoot!.querySelector('.cell[style*="span 3"]')!;
    spanCell.dispatchEvent(new MouseEvent('mouseenter', { bubbles: true }));
    await el.updateComplete;
    const hoverCells = el.shadowRoot!.querySelectorAll('.cell.hover');
    // All cells in rows 0-2 should have hover class
    expect(hoverCells.length).toBeGreaterThanOrEqual(4); // 3 age cells + 1 span cell
  });

  it('keyboard ArrowDown skips spanned rows', async () => {
    el.dataSet = makeDataSet(spanItems);
    el.columnConfig = [{ id: nameCol, width: '1fr', mergeRows: true }, { id: ageCol, width: '80px' }];
    el.selection = 'single';
    el.getRowKey = (r: any) => r.cell(nameCol).value + r.cell(ageCol).value;
    await el.updateComplete;
    // Focus on the span cell (row 0, col 0, spans 3 rows)
    // ArrowDown should skip to row 3 (Bob)
    // Implementation tested via rovingIndex state
  });

  it('selection on spanned cell selects origin row', async () => {
    el.dataSet = makeDataSet(spanItems);
    el.columnConfig = [{ id: nameCol, width: '1fr', mergeRows: true }, { id: ageCol, width: '80px' }];
    el.selection = 'single';
    el.getRowKey = (r: any) => String(r.cell(nameCol).value) + String(r.cell(ageCol).value);
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('selection-change', handler);
    const spanCell = el.shadowRoot!.querySelector('.cell[style*="span 3"]')! as HTMLElement;
    spanCell.click();
    expect(handler).toHaveBeenCalledTimes(1);
    expect(handler.mock.calls[0][0].detail.selectedKeys.length).toBe(1);
  });
});
```

- [ ] **Step 2: Implement span-aware hover**

Update `_hoverRowIndex` to track a range when hovering a spanned cell.
Apply hover class to all cells where `rowIndex >= hoverStart && rowIndex < hoverEnd`.

- [ ] **Step 3: Implement span-aware keyboard navigation**

In `_handleKeyDown`, when ArrowDown from a cell with rowSpan N, advance
`rovingIndex` by N instead of 1. Same for ArrowUp (go back by the previous
cell's rowSpan). ArrowRight/Left skip by colSpan.

- [ ] **Step 4: Implement span-aware selection styling**

Spanned cell shows selected when ALL covered rows are in the selection set.

- [ ] **Step 5: Run tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test packages/pages-table/src/pages-table.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-table/src/pages-table.ts packages/pages-table/src/pages-table.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#26): span-aware hover, keyboard, and selection"
```

---

### Task 7: Playwright visual tests

**Files:**
- Create: `examples/tests/table-spanning.spec.ts`
- Create: `examples/samples/Tables/Spanning.dash.yaml` (test fixture)
- Create: `examples/samples/Tables/Spanning.ts` (test data)

**Interfaces:**
- Consumes: pages-table via example gallery, Playwright test runner

This task creates the Playwright visual tests specified in the design spec.
Tests use screenshot comparison against baseline snapshots.

- [ ] **Step 1: Create spanning test fixture (YAML + TS data)**

The test fixture renders multiple pages-table instances with different span
configurations in a single page: rowspan, colspan, both, and no-span baseline.

- [ ] **Step 2: Write Playwright tests**

Create `examples/tests/table-spanning.spec.ts` with tests for:
- colspan renders correctly
- rowspan renders correctly
- both directions
- scroll into rowspan from above
- scroll past rowspan
- fast scroll through multiple spans
- rowspan at page boundary
- hover on spanned cell
- no spans regression baseline

Each test navigates to the spanning sample, interacts (scroll, hover), and
takes a screenshot for comparison.

- [ ] **Step 3: Generate baseline screenshots**

Run: `npx --cwd /Users/mdproctor/claude/casehub/pages/examples playwright test tests/table-spanning.spec.ts --update-snapshots`

- [ ] **Step 4: Verify tests pass against baselines**

Run: `npx --cwd /Users/mdproctor/claude/casehub/pages/examples playwright test tests/table-spanning.spec.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/tests/table-spanning.spec.ts examples/samples/Tables/
git -C /Users/mdproctor/claude/casehub/pages commit -m "test(#26): Playwright visual tests for cell spanning"
```

---

### Task 8: Build verification and full test run

**Files:**
- No new files — verification only

- [ ] **Step 1: Run full typecheck**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages typecheck`
Expected: PASS

- [ ] **Step 2: Run full test suite**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages test`
Expected: PASS

- [ ] **Step 3: Run Playwright tests**

Run: `npx --cwd /Users/mdproctor/claude/casehub/pages/examples playwright test`
Expected: PASS (no regressions in existing tests)

- [ ] **Step 4: Final commit if any fixes needed**

---

## Dependency graph

```
Task 1 (SpanMap) ──→ Task 2 (scroll ext) ──→ Task 5 (integrate)
                                                  ↑
Task 3 (types) ────────────────────────────────────┘
                                                  ↓
Task 4 (rendering model) ──→ Task 5 ──→ Task 6 (interactions) ──→ Task 7 (Playwright) ──→ Task 8 (verify)
```

Tasks 1, 2, 3 can be done in sequence first (pure additions, no refactoring).
Task 4 is the big refactor. Task 5 connects everything. Tasks 6-8 build on top.
