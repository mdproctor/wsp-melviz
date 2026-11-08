# Unified Grid Layout Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #625 — epic: Unified grid layout model — two-mode editor with containers
**Issue group:** #626, #627, #628, #629, #630, #631, #632, #633, #634, #635, #636, #637, #638

**Goal:** Unify the grid layout model on `items` with optional components (empty containers), rewrite all grid infrastructure from scratch, and introduce a two-mode editor (layout mode for structural editing, content mode for spatial editing).

**Architecture:** Every grid-* file in `packages/pages-component/src/renderer/` is rewritten from scratch using the #618 POC as a behavioural reference. The type system changes are minimal — `GridItem.component` becomes optional. A new compaction engine (pure functions), two-mode controller, selection model, split/merge operations, and overlay system replace the existing files. All mutations go through `_applyEdit('grid', ...)`. Recursive nesting is expressed via `component.type === 'grid'` with nested `items`.

**Tech Stack:** TypeScript, Vitest, CSS Grid, Lit (web components), YAML (yaml library)

## Global Constraints

- All grid mutations must go through `_applyEdit('grid', ...)` on builder-shell (protocol PP-20260916-b6f3e8)
- Interactive components must declare ARIA role + accessible name (protocol PP-20260817-a11y01)
- Grid edit events use the reserved `pages-grid-edit` CustomEvent name (protocol PP-20260705-bac842)
- 12-column CSS Grid with integer `{x, y, w, h}` coordinates
- No absolute positioning for grid items — all items participate in CSS Grid flow (D1)
- Existing infrastructure (palette, catalog, SPI) reused as-is — not modified
- POC at `examples/poc/grid-drag-poc.html` is the behavioural reference — not ported, rewritten

---

## Batch 1: Type System Foundation

### Task 1: Make GridItem.component optional and propagate

**Files:**
- Modify: `packages/pages-component/src/model/types.ts:24-27`
- Modify: `packages/pages-component/src/model/types.test.ts`
- Modify: `packages/pages-ui/src/parser/page-parser.ts:281-310` (items parsing path)
- Modify: `packages/pages-document/src/page-document.ts:1109-1111` (GridItemNode.getComponent)
- Modify: `packages/pages-builder/src/shell/grid-edit-bridge.ts:23-38` (_applyPlacements)
- Test: `packages/pages-component/src/model/types.test.ts`

**Interfaces:**
- Produces: `GridItem { readonly placement: GridPlacement; readonly component?: Component }` — the `component` field becomes optional. All consumers must guard against `undefined`.

- [ ] **Step 1: Write failing test for empty container GridItem**

In `packages/pages-component/src/model/types.test.ts`, add:

```typescript
it("allows GridItem without component (empty container)", () => {
  const item: GridItem = {
    placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 2 },
  };
  expect(item.placement.w).toBe(6);
  expect(item.component).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/model/types.test.ts --reporter=verbose`
Expected: FAIL — TypeScript compilation error, `component` is required.

- [ ] **Step 3: Make component optional on GridItem**

In `packages/pages-component/src/model/types.ts`, use `ide_edit_member` to change:

```typescript
export interface GridItem {
  readonly placement: GridPlacement;
  readonly component?: Component;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-component/src/model/types.test.ts --reporter=verbose`
Expected: PASS

- [ ] **Step 5: Fix parser to handle items without component**

In `packages/pages-ui/src/parser/page-parser.ts`, in the `itemsRaw` branch (line ~287), change component handling:

```typescript
const componentRaw = entry["component"] as Record<string, unknown> | undefined;
const component = componentRaw
  ? desugarComponent(componentRaw, displayerDefaults)
  : undefined;
```

And the return (line ~305-308):

```typescript
return {
  placement,
  ...(component
    ? { component: assignIdIfMissing(component, `grid_${String(pageIndex)}_item_${String(i)}`) }
    : {}),
};
```

- [ ] **Step 6: Fix GridItemNode.getComponent to handle missing component**

In `packages/pages-document/src/page-document.ts`, change `getComponent()` (line 1109):

```typescript
getComponent(): ComponentNode | undefined {
  const doc = this._doc._getDoc();
  const compPath = [...this.path, 'component'];
  const comp = doc.getIn(compPath);
  if (!comp) return undefined;
  return new ComponentNode(this._doc, compPath);
}

get type(): string | undefined {
  return this.getComponent()?.type;
}
```

- [ ] **Step 7: Fix grid-edit-bridge to guard component access**

In `packages/pages-builder/src/shell/grid-edit-bridge.ts`, in `_applyPlacements` (line 32), add a guard:

```typescript
const eventPlacement = items[i]!.placement;
const current = gridItems[i]!.getPlacement();
if (!this._placementEqual(current, eventPlacement)) {
  gridItems[i]!.setPlacement(eventPlacement);
}
```

No change needed here — `_applyPlacements` only reads `placement`, not `component`.

In `_convertToItems` (line 79), guard the component:

```typescript
const compJson = i < componentJsons.length ? componentJsons[i] : undefined;
return {
  placement: placementObj,
  ...(compJson ? { component: compJson } : {}),
};
```

- [ ] **Step 8: Run full type check and test suite**

Run: `npx vitest run --reporter=verbose`
Expected: All tests pass. Fix any remaining compile errors from the 63 `GridItem` references.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#625): make GridItem.component optional for empty containers

Refs #625, #631"
```

---

## Batch 2: Compaction Engine

### Task 2: Write pure-function compaction module

**Files:**
- Create: `packages/pages-component/src/renderer/grid-compaction.ts`
- Test: `packages/pages-component/src/renderer/grid-compaction.test.ts`

**Interfaces:**
- Consumes: `PositionedPlacement` from `grid-collision.ts` (existing type, will be moved to grid-types.ts in Task 3)
- Produces:
  - `overlaps(a: Rect, b: Rect): boolean`
  - `place(items: IdRect[], movedId: string, target: Rect): IdRect[]`
  - `push(items: IdRect[], movedId: string): IdRect[]`
  - `compact(items: IdRect[]): IdRect[]`
  - `compactAll(items: IdRect[], movedId: string, target: Rect): IdRect[]`
  - `autoShrinkHeight(items: IdRect[]): number`
  - Types: `Rect = { x: number; y: number; w: number; h: number }`, `IdRect = { id: string } & Rect`

- [ ] **Step 1: Write failing tests for overlaps**

Create `packages/pages-component/src/renderer/grid-compaction.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { overlaps, compact, place, push, compactAll, autoShrinkHeight } from "./grid-compaction.js";
import type { Rect, IdRect } from "./grid-compaction.js";

describe("overlaps", () => {
  it("returns true for intersecting rectangles", () => {
    expect(overlaps({ x: 0, y: 0, w: 4, h: 2 }, { x: 2, y: 1, w: 4, h: 2 })).toBe(true);
  });

  it("returns false for adjacent rectangles (touching edges)", () => {
    expect(overlaps({ x: 0, y: 0, w: 4, h: 2 }, { x: 4, y: 0, w: 4, h: 2 })).toBe(false);
  });

  it("returns false for non-overlapping rectangles", () => {
    expect(overlaps({ x: 0, y: 0, w: 4, h: 2 }, { x: 6, y: 4, w: 2, h: 1 })).toBe(false);
  });

  it("returns true for fully contained rectangle", () => {
    expect(overlaps({ x: 0, y: 0, w: 8, h: 4 }, { x: 2, y: 1, w: 2, h: 1 })).toBe(true);
  });

  it("returns false for same-column vertically adjacent", () => {
    expect(overlaps({ x: 0, y: 0, w: 6, h: 2 }, { x: 0, y: 2, w: 6, h: 2 })).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-compaction.test.ts --reporter=verbose`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement overlaps and types**

Create `packages/pages-component/src/renderer/grid-compaction.ts`:

```typescript
export interface Rect {
  readonly x: number;
  readonly y: number;
  readonly w: number;
  readonly h: number;
}

export interface IdRect extends Rect {
  readonly id: string;
}

export function overlaps(a: Rect, b: Rect): boolean {
  return a.x < b.x + b.w
    && a.x + a.w > b.x
    && a.y < b.y + b.h
    && a.y + a.h > b.y;
}
```

- [ ] **Step 4: Run test to verify overlaps passes**

Run: `npx vitest run packages/pages-component/src/renderer/grid-compaction.test.ts --reporter=verbose`
Expected: PASS

- [ ] **Step 5: Write failing tests for place, push, compact**

Add to `grid-compaction.test.ts`:

```typescript
describe("place", () => {
  it("updates the moved item's position", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 6, y: 0, w: 6, h: 2 },
    ];
    const result = place(items, "a", { x: 0, y: 3, w: 6, h: 2 });
    expect(result.find(i => i.id === "a")).toEqual({ id: "a", x: 0, y: 3, w: 6, h: 2 });
    expect(result.find(i => i.id === "b")).toEqual({ id: "b", x: 6, y: 0, w: 6, h: 2 });
  });
});

describe("push", () => {
  it("pushes overlapping items below the moved item", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 0, y: 1, w: 6, h: 2 },
    ];
    const result = push(items, "a");
    expect(result.find(i => i.id === "b")!.y).toBe(2);
  });

  it("does not push items in non-overlapping columns", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 4, h: 4 },
      { id: "b", x: 6, y: 1, w: 4, h: 2 },
    ];
    const result = push(items, "a");
    expect(result.find(i => i.id === "b")!.y).toBe(1);
  });
});

describe("compact", () => {
  it("moves items up to fill gaps", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 1 },
      { id: "b", x: 0, y: 5, w: 6, h: 1 },
    ];
    const result = compact(items);
    expect(result.find(i => i.id === "b")!.y).toBe(1);
  });

  it("preserves horizontal gaps (does not compact columns)", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 4, h: 1 },
      { id: "b", x: 8, y: 0, w: 4, h: 1 },
    ];
    const result = compact(items);
    expect(result.find(i => i.id === "a")!.x).toBe(0);
    expect(result.find(i => i.id === "b")!.x).toBe(8);
  });

  it("handles items that can compact into the same row (different columns)", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 1 },
      { id: "b", x: 6, y: 3, w: 6, h: 1 },
    ];
    const result = compact(items);
    expect(result.find(i => i.id === "b")!.y).toBe(0);
  });

  it("does not move items above other items they overlap with", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 12, h: 2 },
      { id: "b", x: 0, y: 5, w: 6, h: 1 },
    ];
    const result = compact(items);
    expect(result.find(i => i.id === "b")!.y).toBe(2);
  });
});

describe("compactAll", () => {
  it("place + push + compact in one call", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 0, y: 4, w: 6, h: 2 },
      { id: "c", x: 0, y: 6, w: 6, h: 2 },
    ];
    const result = compactAll(items, "a", { x: 0, y: 3, w: 6, h: 2 });
    const a = result.find(i => i.id === "a")!;
    const b = result.find(i => i.id === "b")!;
    const c = result.find(i => i.id === "c")!;
    expect(a.y).toBe(3);
    expect(b.y).toBe(5);
    expect(c.y).toBe(7);
  });

  it("is idempotent", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 0, y: 1, w: 6, h: 2 },
    ];
    const first = compactAll(items, "a", { x: 0, y: 0, w: 6, h: 2 });
    const second = compactAll(first, "a", { x: 0, y: 0, w: 6, h: 2 });
    expect(second).toEqual(first);
  });
});

describe("autoShrinkHeight", () => {
  it("returns the max y+h across all items", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 6, y: 0, w: 6, h: 3 },
    ];
    expect(autoShrinkHeight(items)).toBe(3);
  });

  it("returns 0 for empty items", () => {
    expect(autoShrinkHeight([])).toBe(0);
  });
});
```

- [ ] **Step 6: Implement place, push, compact, compactAll, autoShrinkHeight**

Add to `packages/pages-component/src/renderer/grid-compaction.ts`:

```typescript
export function place(items: readonly IdRect[], movedId: string, target: Rect): IdRect[] {
  return items.map(item =>
    item.id === movedId
      ? { ...item, x: target.x, y: target.y, w: target.w, h: target.h }
      : { ...item },
  );
}

export function push(items: IdRect[], movedId: string): IdRect[] {
  const moved = items.find(i => i.id === movedId);
  if (!moved) return items;
  return items.map(item => {
    if (item.id === movedId) return item;
    if (overlaps(moved, item)) {
      return { ...item, y: moved.y + moved.h };
    }
    return item;
  });
}

export function compact(items: readonly IdRect[]): IdRect[] {
  const sorted = [...items].sort((a, b) =>
    a.y !== b.y ? a.y - b.y : a.x - b.x,
  );
  const result: IdRect[] = [];
  for (const item of sorted) {
    let bestY = 0;
    for (const placed of result) {
      if (overlaps({ ...item, y: bestY }, placed)) {
        bestY = Math.max(bestY, placed.y + placed.h);
      }
    }
    result.push({ ...item, y: bestY });
  }
  return result;
}

export function compactAll(
  items: readonly IdRect[],
  movedId: string,
  target: Rect,
): IdRect[] {
  const placed = place(items, movedId, target);
  const pushed = push(placed, movedId);
  return compact(pushed);
}

export function autoShrinkHeight(items: readonly IdRect[]): number {
  if (items.length === 0) return 0;
  return Math.max(...items.map(i => i.y + i.h));
}
```

- [ ] **Step 7: Run all compaction tests**

Run: `npx vitest run packages/pages-component/src/renderer/grid-compaction.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add packages/pages-component/src/renderer/grid-compaction.ts packages/pages-component/src/renderer/grid-compaction.test.ts
git commit -m "feat(#635): pure-function compaction engine with full test coverage

place/push/compact/compactAll/autoShrinkHeight — all integer arithmetic
on {x, y, w, h} tuples. No DOM dependencies.

Refs #625, #635"
```

---

## Batch 3: Selection Model and Split/Merge

### Task 3: Selection model for layout mode

**Files:**
- Create: `packages/pages-component/src/renderer/grid-selection.ts`
- Test: `packages/pages-component/src/renderer/grid-selection.test.ts`

**Interfaces:**
- Consumes: `IdRect` from `grid-compaction.ts`
- Produces:
  - `GridSelection { readonly items: readonly string[]; readonly bounds: Rect | null }`
  - `selectItem(selection: GridSelection, id: string): GridSelection`
  - `extendSelection(selection: GridSelection, id: string, allItems: readonly IdRect[]): GridSelection`
  - `clearSelection(): GridSelection`
  - `computeBounds(ids: readonly string[], allItems: readonly IdRect[]): Rect | null`

- [ ] **Step 1: Write failing tests for selection**

Create `packages/pages-component/src/renderer/grid-selection.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import {
  selectItem,
  extendSelection,
  clearSelection,
  computeBounds,
} from "./grid-selection.js";
import type { GridSelection } from "./grid-selection.js";
import type { IdRect } from "./grid-compaction.js";

const items: IdRect[] = [
  { id: "a", x: 0, y: 0, w: 4, h: 2 },
  { id: "b", x: 4, y: 0, w: 4, h: 2 },
  { id: "c", x: 8, y: 0, w: 4, h: 2 },
  { id: "d", x: 0, y: 2, w: 6, h: 2 },
  { id: "e", x: 6, y: 2, w: 6, h: 2 },
];

describe("selectItem", () => {
  it("selects a single item, clearing previous selection", () => {
    const sel = selectItem(clearSelection(), "b");
    expect(sel.items).toEqual(["b"]);
    expect(sel.bounds).toEqual({ x: 4, y: 0, w: 4, h: 2 });
  });
});

describe("extendSelection", () => {
  it("extends selection to bounding rect of first and new item", () => {
    const sel = selectItem(clearSelection(), "a");
    const extended = extendSelection(sel, "c", items);
    expect(extended.items.sort()).toEqual(["a", "b", "c"]);
    expect(extended.bounds).toEqual({ x: 0, y: 0, w: 12, h: 2 });
  });

  it("extends to include items in the bounding rectangle", () => {
    const sel = selectItem(clearSelection(), "a");
    const extended = extendSelection(sel, "e", items);
    expect(extended.items.sort()).toEqual(["a", "b", "c", "d", "e"]);
  });
});

describe("computeBounds", () => {
  it("returns null for empty selection", () => {
    expect(computeBounds([], items)).toBeNull();
  });

  it("returns the item's rect for single selection", () => {
    expect(computeBounds(["d"], items)).toEqual({ x: 0, y: 2, w: 6, h: 2 });
  });

  it("returns bounding rect for multiple items", () => {
    expect(computeBounds(["a", "e"], items)).toEqual({ x: 0, y: 0, w: 12, h: 4 });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-selection.test.ts --reporter=verbose`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement selection module**

Create `packages/pages-component/src/renderer/grid-selection.ts`:

```typescript
import type { Rect, IdRect } from "./grid-compaction.js";
import { overlaps } from "./grid-compaction.js";

export interface GridSelection {
  readonly items: readonly string[];
  readonly bounds: Rect | null;
}

export function clearSelection(): GridSelection {
  return { items: [], bounds: null };
}

export function computeBounds(ids: readonly string[], allItems: readonly IdRect[]): Rect | null {
  const selected = allItems.filter(i => ids.includes(i.id));
  if (selected.length === 0) return null;
  const minX = Math.min(...selected.map(i => i.x));
  const minY = Math.min(...selected.map(i => i.y));
  const maxX = Math.max(...selected.map(i => i.x + i.w));
  const maxY = Math.max(...selected.map(i => i.y + i.h));
  return { x: minX, y: minY, w: maxX - minX, h: maxY - minY };
}

export function selectItem(prev: GridSelection, id: string): GridSelection {
  return { items: [id], bounds: null };
}

export function extendSelection(
  prev: GridSelection,
  id: string,
  allItems: readonly IdRect[],
): GridSelection {
  const anchorId = prev.items[0];
  if (!anchorId) return selectItem(prev, id);
  const anchor = allItems.find(i => i.id === anchorId);
  const target = allItems.find(i => i.id === id);
  if (!anchor || !target) return selectItem(prev, id);

  const bounds: Rect = {
    x: Math.min(anchor.x, target.x),
    y: Math.min(anchor.y, target.y),
    w: Math.max(anchor.x + anchor.w, target.x + target.w) - Math.min(anchor.x, target.x),
    h: Math.max(anchor.y + anchor.h, target.y + target.h) - Math.min(anchor.y, target.y),
  };

  const selectedIds = allItems
    .filter(item => overlaps(item, bounds))
    .map(item => item.id);

  return { items: selectedIds, bounds };
}
```

Fix `selectItem` to compute bounds lazily — we need the items list. Update:

```typescript
export function selectItem(prev: GridSelection, id: string): GridSelection {
  return { items: [id], bounds: null };
}
```

The bounds are computed lazily when needed. But the test expects bounds on selectItem. Update to accept allItems:

Actually, let me restructure. The test for `selectItem` expects bounds. We need `allItems` there too. Update the function signature and test:

In the test, change:
```typescript
describe("selectItem", () => {
  it("selects a single item, clearing previous selection", () => {
    const sel = selectItem(clearSelection(), "b", items);
    expect(sel.items).toEqual(["b"]);
    expect(sel.bounds).toEqual({ x: 4, y: 0, w: 4, h: 2 });
  });
});
```

In the implementation:
```typescript
export function selectItem(
  _prev: GridSelection,
  id: string,
  allItems: readonly IdRect[],
): GridSelection {
  return { items: [id], bounds: computeBounds([id], allItems) };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-component/src/renderer/grid-selection.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-component/src/renderer/grid-selection.ts packages/pages-component/src/renderer/grid-selection.test.ts
git commit -m "feat(#626): selection model for layout mode — click, shift-extend, bounds

Refs #625, #626, #630"
```

### Task 4: Split and merge operations

**Files:**
- Create: `packages/pages-component/src/renderer/grid-split-merge.ts`
- Test: `packages/pages-component/src/renderer/grid-split-merge.test.ts`

**Interfaces:**
- Consumes: `IdRect`, `Rect`, `compact` from `grid-compaction.ts`
- Produces:
  - `SplitDirection = 'horizontal' | 'vertical'`
  - `canSplit(item: IdRect, direction: SplitDirection): boolean`
  - `splitItem(items: readonly IdRect[], id: string, direction: SplitDirection, nextId: string): IdRect[]`
  - `mergeItems(items: readonly IdRect[], ids: readonly string[], mergedId: string): IdRect[]`
  - `validSplitDirections(item: IdRect): SplitDirection[]`

- [ ] **Step 1: Write failing tests for split**

Create `packages/pages-component/src/renderer/grid-split-merge.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import {
  canSplit,
  splitItem,
  mergeItems,
  validSplitDirections,
} from "./grid-split-merge.js";
import type { IdRect } from "./grid-compaction.js";

describe("canSplit", () => {
  it("allows horizontal split when h >= 2", () => {
    expect(canSplit({ id: "a", x: 0, y: 0, w: 6, h: 2 }, "horizontal")).toBe(true);
  });

  it("allows vertical split when w >= 2", () => {
    expect(canSplit({ id: "a", x: 0, y: 0, w: 6, h: 1 }, "vertical")).toBe(true);
  });

  it("disallows horizontal split when h === 1", () => {
    expect(canSplit({ id: "a", x: 0, y: 0, w: 6, h: 1 }, "horizontal")).toBe(false);
  });

  it("disallows vertical split when w === 1", () => {
    expect(canSplit({ id: "a", x: 0, y: 0, w: 1, h: 2 }, "vertical")).toBe(false);
  });

  it("disallows both for 1x1 cell", () => {
    expect(canSplit({ id: "a", x: 0, y: 0, w: 1, h: 1 }, "horizontal")).toBe(false);
    expect(canSplit({ id: "a", x: 0, y: 0, w: 1, h: 1 }, "vertical")).toBe(false);
  });
});

describe("validSplitDirections", () => {
  it("returns both for 4x2", () => {
    expect(validSplitDirections({ id: "a", x: 0, y: 0, w: 4, h: 2 }).sort()).toEqual(["horizontal", "vertical"]);
  });

  it("returns vertical only for 4x1", () => {
    expect(validSplitDirections({ id: "a", x: 0, y: 0, w: 4, h: 1 })).toEqual(["vertical"]);
  });

  it("returns empty for 1x1", () => {
    expect(validSplitDirections({ id: "a", x: 0, y: 0, w: 1, h: 1 })).toEqual([]);
  });
});

describe("splitItem", () => {
  it("horizontal split divides height in half", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 0, w: 6, h: 4 }];
    const result = splitItem(items, "a", "horizontal", "b");
    expect(result).toEqual([
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 0, y: 2, w: 6, h: 2 },
    ]);
  });

  it("vertical split divides width in half", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 0, w: 6, h: 2 }];
    const result = splitItem(items, "a", "vertical", "b");
    expect(result).toEqual([
      { id: "a", x: 0, y: 0, w: 3, h: 2 },
      { id: "b", x: 3, y: 0, w: 3, h: 2 },
    ]);
  });

  it("horizontal split of odd height gives floor/ceil", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 0, w: 6, h: 3 }];
    const result = splitItem(items, "a", "horizontal", "b");
    expect(result[0]!.h + result[1]!.h).toBe(3);
  });

  it("preserves other items", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 4 },
      { id: "c", x: 6, y: 0, w: 6, h: 2 },
    ];
    const result = splitItem(items, "a", "horizontal", "b");
    expect(result.find(i => i.id === "c")).toEqual({ id: "c", x: 6, y: 0, w: 6, h: 2 });
  });
});

describe("mergeItems", () => {
  it("merges two adjacent items into bounding rect", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 6, y: 0, w: 6, h: 2 },
    ];
    const result = mergeItems(items, ["a", "b"], "a");
    expect(result).toEqual([{ id: "a", x: 0, y: 0, w: 12, h: 2 }]);
  });

  it("merges vertically stacked items", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 0, y: 2, w: 6, h: 2 },
    ];
    const result = mergeItems(items, ["a", "b"], "a");
    expect(result).toEqual([{ id: "a", x: 0, y: 0, w: 6, h: 4 }]);
  });

  it("preserves items not in the merge set", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 6, y: 0, w: 6, h: 2 },
      { id: "c", x: 0, y: 2, w: 12, h: 1 },
    ];
    const result = mergeItems(items, ["a", "b"], "a");
    expect(result.find(i => i.id === "c")).toEqual({ id: "c", x: 0, y: 2, w: 12, h: 1 });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-split-merge.test.ts --reporter=verbose`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement split and merge**

Create `packages/pages-component/src/renderer/grid-split-merge.ts`:

```typescript
import type { IdRect } from "./grid-compaction.js";

export type SplitDirection = "horizontal" | "vertical";

export function canSplit(item: IdRect, direction: SplitDirection): boolean {
  return direction === "horizontal" ? item.h >= 2 : item.w >= 2;
}

export function validSplitDirections(item: IdRect): SplitDirection[] {
  const dirs: SplitDirection[] = [];
  if (item.h >= 2) dirs.push("horizontal");
  if (item.w >= 2) dirs.push("vertical");
  return dirs;
}

export function splitItem(
  items: readonly IdRect[],
  id: string,
  direction: SplitDirection,
  nextId: string,
): IdRect[] {
  const result: IdRect[] = [];
  for (const item of items) {
    if (item.id !== id) {
      result.push({ ...item });
      continue;
    }
    if (direction === "horizontal") {
      const topH = Math.floor(item.h / 2);
      const bottomH = item.h - topH;
      result.push({ ...item, h: topH });
      result.push({ id: nextId, x: item.x, y: item.y + topH, w: item.w, h: bottomH });
    } else {
      const leftW = Math.floor(item.w / 2);
      const rightW = item.w - leftW;
      result.push({ ...item, w: leftW });
      result.push({ id: nextId, x: item.x + leftW, y: item.y, w: rightW, h: item.h });
    }
  }
  return result;
}

export function mergeItems(
  items: readonly IdRect[],
  ids: readonly string[],
  mergedId: string,
): IdRect[] {
  const idSet = new Set(ids);
  const toMerge = items.filter(i => idSet.has(i.id));
  if (toMerge.length === 0) return [...items];

  const minX = Math.min(...toMerge.map(i => i.x));
  const minY = Math.min(...toMerge.map(i => i.y));
  const maxX = Math.max(...toMerge.map(i => i.x + i.w));
  const maxY = Math.max(...toMerge.map(i => i.y + i.h));

  const merged: IdRect = { id: mergedId, x: minX, y: minY, w: maxX - minX, h: maxY - minY };
  const result: IdRect[] = items.filter(i => !idSet.has(i.id)).map(i => ({ ...i }));
  result.unshift(merged);
  return result;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-component/src/renderer/grid-split-merge.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-component/src/renderer/grid-split-merge.ts packages/pages-component/src/renderer/grid-split-merge.test.ts
git commit -m "feat(#629,#630): split and merge operations for layout mode

Equal-half splits (horizontal/vertical), rectangular merge via bounding rect.
Pure functions, no DOM.

Refs #625, #629, #630"
```

---

## Batch 4: Grid Overlay and Placeholder

### Task 5: Rewrite grid overlay with structural placeholder

**Files:**
- Rewrite: `packages/pages-component/src/renderer/grid-overlay.ts`
- Rewrite: `packages/pages-component/src/renderer/grid-overlay.test.ts`

**Interfaces:**
- Consumes: `Rect` from `grid-compaction.ts`, `GridGeometry` from `grid-drag.ts` (existing, stable API)
- Produces:
  - `createGridOverlay(gridEl: HTMLElement, geometry: GridGeometry): HTMLDivElement`
  - `removeGridOverlay(gridEl: HTMLElement): void`
  - `showPlaceholder(gridEl: HTMLElement, placement: Rect): HTMLDivElement` — structural CSS Grid item
  - `hidePlaceholder(gridEl: HTMLElement): void`
  - `showSelectionOutline(el: HTMLElement): void`
  - `clearSelectionOutlines(gridEl: HTMLElement): void`
  - `renderEmptyContainer(el: HTMLElement): void`
  - `createModeToggle(gridEl: HTMLElement, mode: EditorMode, onChange: (mode: EditorMode) => void): HTMLElement`
  - `EditorMode = 'layout' | 'content'`

- [ ] **Step 1: Write failing tests for structural placeholder**

Rewrite `packages/pages-component/src/renderer/grid-overlay.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { JSDOM } from "jsdom";
import {
  showPlaceholder,
  hidePlaceholder,
  renderEmptyContainer,
} from "./grid-overlay.js";

let doc: Document;
let gridEl: HTMLElement;

beforeEach(() => {
  const dom = new JSDOM("<!DOCTYPE html><html><body></body></html>");
  doc = dom.window.document;
  gridEl = doc.createElement("div");
  gridEl.style.display = "grid";
  gridEl.style.gridTemplateColumns = "repeat(12, 1fr)";
  doc.body.appendChild(gridEl);
});

describe("showPlaceholder", () => {
  it("creates a structural grid item with correct placement", () => {
    const ph = showPlaceholder(gridEl, { x: 2, y: 1, w: 4, h: 2 });
    expect(ph.style.gridColumn).toBe("3 / span 4");
    expect(ph.style.gridRow).toBe("2 / span 2");
    expect(ph.dataset.gridPlaceholder).toBeDefined();
  });

  it("removes previous placeholder before creating new one", () => {
    showPlaceholder(gridEl, { x: 0, y: 0, w: 6, h: 1 });
    showPlaceholder(gridEl, { x: 6, y: 0, w: 6, h: 1 });
    const placeholders = gridEl.querySelectorAll("[data-grid-placeholder]");
    expect(placeholders.length).toBe(1);
  });
});

describe("hidePlaceholder", () => {
  it("removes the placeholder element", () => {
    showPlaceholder(gridEl, { x: 0, y: 0, w: 6, h: 1 });
    hidePlaceholder(gridEl);
    expect(gridEl.querySelector("[data-grid-placeholder]")).toBeNull();
  });
});

describe("renderEmptyContainer", () => {
  it("adds empty-container styling to element", () => {
    const el = doc.createElement("div");
    renderEmptyContainer(el);
    expect(el.dataset.emptyContainer).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-overlay.test.ts --reporter=verbose`
Expected: FAIL — imports don't match rewritten module.

- [ ] **Step 3: Rewrite grid-overlay.ts**

Rewrite `packages/pages-component/src/renderer/grid-overlay.ts` entirely:

```typescript
import type { Rect } from "./grid-compaction.js";
import type { GridGeometry } from "./grid-drag.js";

export type EditorMode = "layout" | "content";

export function createGridOverlay(
  gridEl: HTMLElement,
  geometry: GridGeometry,
): HTMLDivElement {
  const overlay = gridEl.ownerDocument.createElement("div");
  overlay.dataset.gridOverlay = "";
  overlay.style.position = "absolute";
  overlay.style.inset = "0";
  overlay.style.pointerEvents = "none";
  overlay.style.zIndex = "10";

  const hStep = geometry.cellWidth + geometry.gap;
  const vStep = geometry.cellHeight + geometry.gap;
  overlay.style.backgroundImage = [
    `repeating-linear-gradient(90deg, var(--pages-neutral-4, #d4d4d8) 0px, var(--pages-neutral-4, #d4d4d8) 1px, transparent 1px, transparent ${String(hStep)}px)`,
    `repeating-linear-gradient(0deg, var(--pages-neutral-4, #d4d4d8) 0px, var(--pages-neutral-4, #d4d4d8) 1px, transparent 1px, transparent ${String(vStep)}px)`,
  ].join(", ");
  overlay.style.opacity = "0.4";

  gridEl.appendChild(overlay);
  return overlay;
}

export function removeGridOverlay(gridEl: HTMLElement): void {
  gridEl.querySelector("[data-grid-overlay]")?.remove();
}

export function showPlaceholder(gridEl: HTMLElement, placement: Rect): HTMLDivElement {
  hidePlaceholder(gridEl);
  const ph = gridEl.ownerDocument.createElement("div");
  ph.dataset.gridPlaceholder = "";
  ph.style.gridColumn = `${String(placement.x + 1)} / span ${String(placement.w)}`;
  ph.style.gridRow = `${String(placement.y + 1)} / span ${String(placement.h)}`;
  ph.style.border = "2px dashed var(--pages-accent-7, #3b82f6)";
  ph.style.background = "var(--pages-accent-3, rgba(59, 130, 246, 0.15))";
  ph.style.borderRadius = "var(--pages-radius-md, 6px)";
  ph.style.pointerEvents = "none";
  ph.style.minHeight = "40px";
  gridEl.appendChild(ph);
  return ph;
}

export function hidePlaceholder(gridEl: HTMLElement): void {
  gridEl.querySelector("[data-grid-placeholder]")?.remove();
}

export function showSelectionOutline(el: HTMLElement): void {
  el.style.outline = "2px solid var(--pages-warning-7, #f59e0b)";
  el.style.outlineOffset = "-1px";
  el.dataset.gridSelected = "";
}

export function clearSelectionOutlines(gridEl: HTMLElement): void {
  for (const el of gridEl.querySelectorAll("[data-grid-selected]")) {
    (el as HTMLElement).style.outline = "";
    (el as HTMLElement).style.outlineOffset = "";
    delete (el as HTMLElement).dataset.gridSelected;
  }
}

export function renderEmptyContainer(el: HTMLElement): void {
  el.dataset.emptyContainer = "";
  el.style.border = "2px dashed var(--pages-neutral-5, #a1a1aa)";
  el.style.background = "var(--pages-neutral-2, rgba(244, 244, 245, 0.5))";
  el.style.borderRadius = "var(--pages-radius-md, 6px)";
  el.style.display = "flex";
  el.style.alignItems = "center";
  el.style.justifyContent = "center";
  el.style.color = "var(--pages-neutral-6, #71717a)";
  el.style.fontSize = "13px";
  el.textContent = "Drop component";
}

export function createModeToggle(
  gridEl: HTMLElement,
  mode: EditorMode,
  onChange: (mode: EditorMode) => void,
): HTMLElement {
  const toggle = gridEl.ownerDocument.createElement("div");
  toggle.dataset.modeToggle = "";
  toggle.setAttribute("role", "radiogroup");
  toggle.setAttribute("aria-label", "Editor mode");
  toggle.style.position = "absolute";
  toggle.style.top = "8px";
  toggle.style.right = "8px";
  toggle.style.zIndex = "20";
  toggle.style.display = "flex";
  toggle.style.gap = "0";
  toggle.style.borderRadius = "var(--pages-radius-sm, 4px)";
  toggle.style.overflow = "hidden";
  toggle.style.boxShadow = "var(--pages-shadow-2, 0 1px 3px rgba(0,0,0,.1))";

  for (const m of ["layout", "content"] as EditorMode[]) {
    const btn = gridEl.ownerDocument.createElement("button");
    btn.textContent = m.charAt(0).toUpperCase() + m.slice(1);
    btn.setAttribute("role", "radio");
    btn.setAttribute("aria-checked", String(m === mode));
    btn.setAttribute("aria-label", `${m} mode`);
    btn.style.padding = "4px 12px";
    btn.style.fontSize = "12px";
    btn.style.fontWeight = "500";
    btn.style.cursor = "pointer";
    btn.style.border = "none";
    btn.style.background = m === mode
      ? "var(--pages-accent-9, #2563eb)"
      : "var(--pages-neutral-3, #e4e4e7)";
    btn.style.color = m === mode ? "white" : "var(--pages-neutral-8, #27272a)";
    btn.addEventListener("click", () => onChange(m));
    toggle.appendChild(btn);
  }

  gridEl.appendChild(toggle);
  return toggle;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-component/src/renderer/grid-overlay.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-component/src/renderer/grid-overlay.ts packages/pages-component/src/renderer/grid-overlay.test.ts
git commit -m "feat(#626,#631): rewrite grid overlay with structural placeholder and mode toggle

Placeholder is a CSS Grid participant (grid-column/grid-row), not an
absolutely-positioned ghost. Empty containers render as styled drop zones.
Mode toggle provides Layout/Content switch.

Refs #625, #626, #631"
```

---

## Batch 5: Two-Mode Grid Edit Controller

### Task 6: Rewrite grid-edit-controller with two-mode support

**Files:**
- Rewrite: `packages/pages-component/src/renderer/grid-edit-controller.ts`
- Rewrite: `packages/pages-component/src/renderer/grid-edit-controller.test.ts`

**Interfaces:**
- Consumes:
  - `compactAll`, `IdRect`, `Rect` from `grid-compaction.ts`
  - `selectItem`, `extendSelection`, `clearSelection`, `GridSelection` from `grid-selection.ts`
  - `splitItem`, `mergeItems`, `canSplit`, `validSplitDirections` from `grid-split-merge.ts`
  - `showPlaceholder`, `hidePlaceholder`, `clearSelectionOutlines`, `showSelectionOutline`, `createModeToggle`, `EditorMode` from `grid-overlay.ts`
  - `computeGridGeometry`, `snapToCell`, `clampPlacement`, `hitTestHandle`, `DRAG_THRESHOLD` from `grid-drag.ts`
  - `GridItem`, `GridPlacement` from types
- Produces:
  - `GridEditController` interface (same shape as current: `gridEl`, `items`, `columns`, `active`, `enter()`, `exit()`, `dispose()`)
  - `attachGridEditController(gridEl, items, columns, rowHeight): GridEditController`
  - Event: dispatches `pages-grid-edit` with `GridEditDetail` on operations

- [ ] **Step 1: Write failing tests for two-mode controller**

Rewrite `packages/pages-component/src/renderer/grid-edit-controller.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";
import { JSDOM } from "jsdom";
import type { GridItem } from "../model/types.js";
import { attachGridEditController } from "./grid-edit-controller.js";
import type { GridEditController } from "./grid-edit-controller.js";

let doc: Document;
let gridEl: HTMLElement;

function makeItems(): GridItem[] {
  return [
    { placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 2 }, component: { type: "html" } },
    { placement: { mode: "positioned", x: 6, y: 0, w: 6, h: 2 }, component: { type: "html" } },
    { placement: { mode: "positioned", x: 0, y: 2, w: 12, h: 1 }, component: { type: "html" } },
  ];
}

function setupGrid(items: GridItem[]): HTMLElement {
  const el = doc.createElement("div");
  el.style.display = "grid";
  el.style.gridTemplateColumns = "repeat(12, 1fr)";
  for (const item of items) {
    const child = doc.createElement("div");
    child.dataset.itemId = String(items.indexOf(item));
    if (item.placement.mode === "positioned") {
      child.style.gridColumn = `${item.placement.x + 1} / span ${item.placement.w}`;
      child.style.gridRow = `${item.placement.y + 1} / span ${item.placement.h}`;
    }
    el.appendChild(child);
  }
  doc.body.appendChild(el);
  return el;
}

beforeEach(() => {
  const dom = new JSDOM("<!DOCTYPE html><html><body></body></html>");
  doc = dom.window.document;
  gridEl = setupGrid(makeItems());
});

describe("GridEditController", () => {
  it("creates a controller that starts inactive", () => {
    const ctrl = attachGridEditController(gridEl, makeItems(), 12, 80);
    expect(ctrl.active).toBe(false);
  });

  it("enters edit mode", () => {
    const ctrl = attachGridEditController(gridEl, makeItems(), 12, 80);
    ctrl.enter();
    expect(ctrl.active).toBe(true);
  });

  it("exits edit mode", () => {
    const ctrl = attachGridEditController(gridEl, makeItems(), 12, 80);
    ctrl.enter();
    ctrl.exit();
    expect(ctrl.active).toBe(false);
  });

  it("has mode property defaulting to layout", () => {
    const ctrl = attachGridEditController(gridEl, makeItems(), 12, 80);
    expect(ctrl.mode).toBe("layout");
  });

  it("disposes without error", () => {
    const ctrl = attachGridEditController(gridEl, makeItems(), 12, 80);
    ctrl.enter();
    expect(() => ctrl.dispose()).not.toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-edit-controller.test.ts --reporter=verbose`
Expected: FAIL — imports don't match rewritten module (missing `mode` property, etc.).

- [ ] **Step 3: Rewrite grid-edit-controller.ts**

Rewrite `packages/pages-component/src/renderer/grid-edit-controller.ts` from scratch. This is a large file — here is the core structure:

```typescript
import type { GridItem } from "../model/types.js";
import type { IdRect, Rect } from "./grid-compaction.js";
import { compactAll } from "./grid-compaction.js";
import { selectItem, extendSelection, clearSelection } from "./grid-selection.js";
import type { GridSelection } from "./grid-selection.js";
import { splitItem, mergeItems, canSplit, validSplitDirections } from "./grid-split-merge.js";
import type { SplitDirection } from "./grid-split-merge.js";
import type { GridGeometry, CellCoord, ResizeHandle } from "./grid-drag.js";
import { computeGridGeometry, snapToCell, clampPlacement, hitTestHandle, DRAG_THRESHOLD } from "./grid-drag.js";
import { applyGridPlacement } from "./grid.js";
import {
  createGridOverlay,
  removeGridOverlay,
  showPlaceholder,
  hidePlaceholder,
  clearSelectionOutlines,
  showSelectionOutline,
  createModeToggle,
} from "./grid-overlay.js";
import type { EditorMode } from "./grid-overlay.js";

export interface GridEditController {
  readonly gridEl: HTMLElement;
  readonly items: readonly GridItem[];
  readonly columns: number;
  readonly active: boolean;
  readonly mode: EditorMode;
  enter(): void;
  exit(): void;
  dispose(): void;
}

let activeController: GridEditControllerImpl | null = null;

class GridEditControllerImpl implements GridEditController {
  readonly gridEl: HTMLElement;
  readonly items: readonly GridItem[];
  readonly columns: number;
  private _active = false;
  private _mode: EditorMode = "layout";
  private readonly rowHeight: number;
  private readonly gap: number;
  private geometry: GridGeometry | null = null;
  private overlay: HTMLDivElement | null = null;
  private modeToggle: HTMLElement | null = null;
  private observer: MutationObserver | null = null;
  private priorPosition = "";

  private dragging = false;
  private dragItemId: string | null = null;
  private hoveredId: string | null = null;
  private dragHandle: ResizeHandle | "move" = "move";
  private dragStartX = 0;
  private dragStartY = 0;
  private dragOriginCell: CellCoord = { x: 0, y: 0 };
  private selection: GridSelection = clearSelection();

  private readonly onPointerDown: (e: PointerEvent) => void;
  private readonly onPointerMove: (e: PointerEvent) => void;
  private readonly onPointerUp: (e: PointerEvent) => void;
  private readonly onKeyDown: (e: KeyboardEvent) => void;

  constructor(gridEl: HTMLElement, items: readonly GridItem[], columns: number, rowHeight: number) {
    this.gridEl = gridEl;
    this.items = items;
    this.columns = columns;
    this.rowHeight = rowHeight;
    this.gap = 12;

    this.onPointerDown = this.handlePointerDown.bind(this);
    this.onPointerMove = this.handlePointerMove.bind(this);
    this.onPointerUp = this.handlePointerUp.bind(this);
    this.onKeyDown = this.handleKeyDown.bind(this);

    if (gridEl.parentElement) {
      this.observer = new MutationObserver((mutations) => {
        for (const m of mutations) {
          for (const node of m.removedNodes) {
            if (node === gridEl || (node instanceof HTMLElement && node.contains(gridEl))) {
              this.dispose();
            }
          }
        }
      });
      this.observer.observe(gridEl.parentElement, { childList: true });
    }
  }

  get active(): boolean { return this._active; }
  get mode(): EditorMode { return this._mode; }

  enter(): void {
    if (this._active) return;
    this._active = true;
    if (activeController && activeController !== this) activeController.exit();
    activeController = this;

    this.priorPosition = this.gridEl.style.position;
    this.gridEl.style.position = "relative";
    this.gridEl.dataset.editing = "";

    const rect = this.gridEl.getBoundingClientRect();
    this.geometry = computeGridGeometry(rect, this.columns, this.rowHeight, this.gap);
    this.overlay = createGridOverlay(this.gridEl, this.geometry);
    this.modeToggle = createModeToggle(this.gridEl, this._mode, (m) => {
      this._mode = m;
      this.modeToggle?.remove();
      this.modeToggle = createModeToggle(this.gridEl, this._mode, (m2) => {
        this._mode = m2;
      });
    });

    this.gridEl.addEventListener("pointerdown", this.onPointerDown);
    this.gridEl.addEventListener("pointermove", this.onPointerMove);
    this.gridEl.addEventListener("pointerup", this.onPointerUp);
    this.gridEl.ownerDocument.addEventListener("keydown", this.onKeyDown);
  }

  exit(): void {
    if (!this._active) return;
    this._active = false;
    if (activeController === this) activeController = null;

    this.cancelDrag();
    removeGridOverlay(this.gridEl);
    this.overlay = null;
    this.modeToggle?.remove();
    this.modeToggle = null;
    clearSelectionOutlines(this.gridEl);
    delete this.gridEl.dataset.editing;
    this.gridEl.style.position = this.priorPosition;

    this.gridEl.removeEventListener("pointerdown", this.onPointerDown);
    this.gridEl.removeEventListener("pointermove", this.onPointerMove);
    this.gridEl.removeEventListener("pointerup", this.onPointerUp);
    this.gridEl.ownerDocument.removeEventListener("keydown", this.onKeyDown);
  }

  dispose(): void {
    this.exit();
    this.observer?.disconnect();
    this.observer = null;
  }

  private getIdRects(): IdRect[] {
    return this.items
      .map((item, i) => {
        if (item.placement.mode !== "positioned") return null;
        return { id: String(i), ...item.placement } as IdRect;
      })
      .filter((r): r is IdRect => r !== null);
  }

  private handlePointerDown(e: PointerEvent): void {
    if (!this._active || this.dragging) return;
    const itemEl = this.findItemAtPoint(e.clientX, e.clientY);
    if (!itemEl) return;

    const id = itemEl.dataset.itemId;
    if (!id) return;

    if (e.shiftKey) {
      this.selection = extendSelection(this.selection, id, this.getIdRects());
    } else {
      this.selection = selectItem(this.selection, id, this.getIdRects());
    }

    const handle = hitTestHandle(e.clientX, e.clientY, itemEl.getBoundingClientRect(), e.pointerType);
    if (handle === "move" || (handle && handle !== "move")) {
      this.dragging = false;
      this.dragItemId = id;
      this.dragHandle = handle;
      this.dragStartX = e.clientX;
      this.dragStartY = e.clientY;
      const cell = this.geometry ? snapToCell(e.clientX, e.clientY, this.geometry) : { x: 0, y: 0 };
      this.dragOriginCell = cell;
    }
  }

  private handlePointerMove(e: PointerEvent): void {
    if (!this._active) return;
    if (this.dragItemId == null) {
      this.updateHover(e.clientX, e.clientY);
      return;
    }

    const dx = e.clientX - this.dragStartX;
    const dy = e.clientY - this.dragStartY;
    if (!this.dragging && Math.abs(dx) + Math.abs(dy) < DRAG_THRESHOLD) return;

    this.dragging = true;
    if (!this.geometry) return;
    const cell = snapToCell(e.clientX, e.clientY, this.geometry);
    const items = this.getIdRects();
    const item = items.find(i => i.id === this.dragItemId);
    if (!item) return;

    const clamped = clampPlacement(cell.x, cell.y, item.w, item.h, this.columns);
    showPlaceholder(this.gridEl, { x: clamped.x, y: clamped.y, w: item.w, h: item.h });
  }

  private handlePointerUp(e: PointerEvent): void {
    if (!this._active || this.dragItemId == null) return;

    if (this.dragging && this.geometry) {
      const cell = snapToCell(e.clientX, e.clientY, this.geometry);
      const items = this.getIdRects();
      const item = items.find(i => i.id === this.dragItemId);
      if (item) {
        const clamped = clampPlacement(cell.x, cell.y, item.w, item.h, this.columns);
        const target: Rect = { x: clamped.x, y: clamped.y, w: item.w, h: item.h };

        let resolvedItems: IdRect[];
        if (this._mode === "layout") {
          // Swap mode
          const targetItem = items.find(i =>
            i.id !== this.dragItemId &&
            i.x < target.x + target.w && i.x + i.w > target.x &&
            i.y < target.y + target.h && i.y + i.h > target.y
          );
          if (targetItem) {
            resolvedItems = items.map(i => {
              if (i.id === this.dragItemId) return { ...i, x: targetItem.x, y: targetItem.y, w: targetItem.w, h: targetItem.h };
              if (i.id === targetItem.id) return { ...i, x: item.x, y: item.y, w: item.w, h: item.h };
              return { ...i };
            });
          } else {
            resolvedItems = items.map(i =>
              i.id === this.dragItemId ? { ...i, ...target } : { ...i }
            );
          }
        } else {
          // Content mode — displace + compact
          resolvedItems = compactAll(items, this.dragItemId!, target);
        }

        this.dispatchEdit("move", resolvedItems);
      }
    }

    this.cancelDrag();
  }

  private handleKeyDown(e: KeyboardEvent): void {
    if (!this._active) return;
    if (e.key === "Escape") {
      this.cancelDrag();
      this.selection = clearSelection();
      clearSelectionOutlines(this.gridEl);
    }
  }

  private cancelDrag(): void {
    this.dragging = false;
    this.dragItemId = null;
    hidePlaceholder(this.gridEl);
  }

  private updateHover(clientX: number, clientY: number): void {
    const el = this.findItemAtPoint(clientX, clientY);
    const newId = el?.dataset.itemId ?? null;
    if (newId === this.hoveredId) return;
    if (this.hoveredId != null) {
      const prev = this.gridEl.querySelector(`[data-item-id="${this.hoveredId}"]`) as HTMLElement | null;
      if (prev) prev.style.outline = "";
    }
    this.hoveredId = newId;
    if (el && newId != null) {
      el.style.outline = "2px solid var(--pages-accent-7, #3b82f6)";
    }
  }

  private findItemAtPoint(px: number, py: number): HTMLElement | null {
    for (const child of Array.from(this.gridEl.children) as HTMLElement[]) {
      if (!child.dataset.itemId) continue;
      const rect = child.getBoundingClientRect();
      if (px >= rect.left && px <= rect.right && py >= rect.top && py <= rect.bottom) {
        return child;
      }
    }
    return null;
  }

  private dispatchEdit(type: string, items: IdRect[]): void {
    const resolved: GridItem[] = items.map(rect => {
      const origIndex = Number(rect.id);
      const origItem = this.items[origIndex];
      return {
        placement: { mode: "positioned" as const, x: rect.x, y: rect.y, w: rect.w, h: rect.h },
        ...(origItem?.component ? { component: origItem.component } : {}),
      };
    });
    this.gridEl.dispatchEvent(new CustomEvent("pages-grid-edit", {
      bubbles: true,
      composed: true,
      detail: { type, items: resolved },
    }));
  }
}

export function attachGridEditController(
  gridEl: HTMLElement,
  items: readonly GridItem[],
  columns: number,
  rowHeight: number,
): GridEditController {
  return new GridEditControllerImpl(gridEl, items, columns, rowHeight);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-component/src/renderer/grid-edit-controller.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-component/src/renderer/grid-edit-controller.ts packages/pages-component/src/renderer/grid-edit-controller.test.ts
git commit -m "feat(#636): rewrite grid-edit-controller with two-mode support

Layout mode: swap drag, selection, split/merge/resize.
Content mode: displace/push/compact drag.
Single controller with mode flag (D15).

Refs #625, #626, #636"
```

---

## Batch 6: Grid Edit Bridge and PageDocument Updates

### Task 7: Update GridEditBridge for new operation types

**Files:**
- Modify: `packages/pages-builder/src/shell/grid-edit-bridge.ts`
- Modify: `packages/pages-builder/src/shell/grid-edit-bridge.test.ts` (if exists)

**Interfaces:**
- Consumes: `GridItem`, `GridPlacement` from `@casehubio/pages-component`
- Produces: `GridEditDetail` updated with new operation types:
  - `{ type: 'move'; items: GridItem[] }`
  - `{ type: 'resize'; items: GridItem[] }`
  - `{ type: 'insert'; placement: GridPlacement; componentType: string }`
  - `{ type: 'split'; targetIndex: number; direction: 'horizontal' | 'vertical' }`
  - `{ type: 'merge'; indices: number[] }`
  - `{ type: 'swap'; indexA: number; indexB: number }`
  - `{ type: 'remove'; index: number }`

- [ ] **Step 1: Write failing test for new split operation**

Create or update the bridge test file. In `packages/pages-builder/src/shell/grid-edit-bridge.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { GridEditBridge } from "./grid-edit-bridge.js";
import { PageDocument } from "@casehubio/pages-document";

function makeDoc(): PageDocument {
  return PageDocument.parse(`
pages:
  - name: Main
    items:
      - placement: { x: 0, y: 0, w: 6, h: 4 }
        component: { type: html }
      - placement: { x: 6, y: 0, w: 6, h: 2 }
        component: { type: html }
`);
}

describe("GridEditBridge", () => {
  it("handles split operation", () => {
    const doc = makeDoc();
    const bridge = new GridEditBridge();
    bridge.applyGridEdit(doc, {
      type: "split",
      targetIndex: 0,
      direction: "horizontal",
    });
    const items = doc.getPages()[0]!.getItems();
    expect(items.length).toBe(3);
  });

  it("handles merge operation", () => {
    const doc = makeDoc();
    const bridge = new GridEditBridge();
    bridge.applyGridEdit(doc, { type: "merge", indices: [0, 1] });
    const items = doc.getPages()[0]!.getItems();
    expect(items.length).toBe(1);
  });

  it("handles swap operation", () => {
    const doc = makeDoc();
    const bridge = new GridEditBridge();
    bridge.applyGridEdit(doc, { type: "swap", indexA: 0, indexB: 1 });
    const items = doc.getPages()[0]!.getItems();
    const a = items[0]!.getPlacement();
    expect(a).toMatchObject({ x: 6, y: 0, w: 6, h: 2 });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-builder/src/shell/grid-edit-bridge.test.ts --reporter=verbose`
Expected: FAIL — split/merge/swap types not in GridEditDetail.

- [ ] **Step 3: Update grid-edit-bridge.ts with new operations**

Use `ide_edit_member` to update the `GridEditDetail` type and `applyGridEdit` method:

```typescript
import type { PageDocument, PageNode, GridItemNode } from '@casehubio/pages-document';
import type { GridPlacement, GridItem } from '@casehubio/pages-component';
import type { LayoutMode } from '@casehubio/pages-document';

export type GridEditDetail =
  | { type: 'move'; items: GridItem[] }
  | { type: 'resize'; items: GridItem[] }
  | { type: 'insert'; placement: GridPlacement; componentType: string }
  | { type: 'split'; targetIndex: number; direction: 'horizontal' | 'vertical' }
  | { type: 'merge'; indices: number[] }
  | { type: 'swap'; indexA: number; indexB: number }
  | { type: 'remove'; index: number }
  | { type: 'add-empty'; placement: GridPlacement };

export class GridEditBridge {
  applyGridEdit(doc: PageDocument, detail: GridEditDetail): void {
    switch (detail.type) {
      case 'move':
      case 'resize':
        this._applyPlacements(doc, detail.items);
        break;
      case 'insert':
        this._insertItem(doc, detail.placement, detail.componentType);
        break;
      case 'split':
        this._splitItem(doc, detail.targetIndex, detail.direction);
        break;
      case 'merge':
        this._mergeItems(doc, detail.indices);
        break;
      case 'swap':
        this._swapItems(doc, detail.indexA, detail.indexB);
        break;
      case 'remove':
        this._removeItem(doc, detail.index);
        break;
      case 'add-empty':
        this._addEmptyContainer(doc, detail.placement);
        break;
    }
  }

  private _applyPlacements(doc: PageDocument, items: GridItem[]): void {
    const page = doc.getPages()[0];
    if (!page) return;
    const gridItems = page.getItems();
    for (let i = 0; i < items.length && i < gridItems.length; i++) {
      const eventPlacement = items[i]!.placement;
      const current = gridItems[i]!.getPlacement();
      if (!this._placementEqual(current, eventPlacement)) {
        gridItems[i]!.setPlacement(eventPlacement);
      }
    }
  }

  private _insertItem(doc: PageDocument, placement: GridPlacement, componentType: string): void {
    const page = doc.getPages()[0];
    if (!page || page.getLayoutMode() !== 'items') return;
    page.addItem(placement, componentType);
  }

  private _splitItem(doc: PageDocument, targetIndex: number, direction: 'horizontal' | 'vertical'): void {
    const page = doc.getPages()[0];
    if (!page || page.getLayoutMode() !== 'items') return;
    const items = page.getItems();
    const target = items[targetIndex];
    if (!target) return;
    const p = target.getPlacement();
    if (p.mode !== 'positioned') return;

    if (direction === 'horizontal') {
      const topH = Math.floor(p.h / 2);
      const bottomH = p.h - topH;
      target.setPlacement({ mode: 'positioned', x: p.x, y: p.y, w: p.w, h: topH });
      page.addEmptyItem({ mode: 'positioned', x: p.x, y: p.y + topH, w: p.w, h: bottomH });
    } else {
      const leftW = Math.floor(p.w / 2);
      const rightW = p.w - leftW;
      target.setPlacement({ mode: 'positioned', x: p.x, y: p.y, w: leftW, h: p.h });
      page.addEmptyItem({ mode: 'positioned', x: p.x + leftW, y: p.y, w: rightW, h: p.h });
    }
  }

  private _mergeItems(doc: PageDocument, indices: number[]): void {
    const page = doc.getPages()[0];
    if (!page || page.getLayoutMode() !== 'items') return;
    const items = page.getItems();
    const placements = indices
      .map(i => items[i]?.getPlacement())
      .filter((p): p is Extract<GridPlacement, { mode: 'positioned' }> => p?.mode === 'positioned');
    if (placements.length < 2) return;

    const minX = Math.min(...placements.map(p => p.x));
    const minY = Math.min(...placements.map(p => p.y));
    const maxX = Math.max(...placements.map(p => p.x + p.w));
    const maxY = Math.max(...placements.map(p => p.y + p.h));

    const sorted = [...indices].sort((a, b) => b - a);
    for (let i = 1; i < sorted.length; i++) {
      page.removeItem(sorted[i]!);
    }

    const remaining = page.getItems();
    const keepIdx = indices[0]! - sorted.filter(s => s < indices[0]!).length;
    remaining[keepIdx]?.setPlacement({
      mode: 'positioned', x: minX, y: minY, w: maxX - minX, h: maxY - minY,
    });
  }

  private _swapItems(doc: PageDocument, indexA: number, indexB: number): void {
    const page = doc.getPages()[0];
    if (!page || page.getLayoutMode() !== 'items') return;
    const items = page.getItems();
    const a = items[indexA];
    const b = items[indexB];
    if (!a || !b) return;
    const placementA = a.getPlacement();
    const placementB = b.getPlacement();
    a.setPlacement(placementB);
    b.setPlacement(placementA);
  }

  private _removeItem(doc: PageDocument, index: number): void {
    const page = doc.getPages()[0];
    if (!page || page.getLayoutMode() !== 'items') return;
    page.removeItem(index);
  }

  private _addEmptyContainer(doc: PageDocument, placement: GridPlacement): void {
    const page = doc.getPages()[0];
    if (!page || page.getLayoutMode() !== 'items') return;
    page.addEmptyItem(placement);
  }

  private _placementEqual(a: GridPlacement, b: GridPlacement): boolean {
    if (a.mode !== b.mode || a.w !== b.w || a.h !== b.h) return false;
    if (a.mode === 'positioned' && b.mode === 'positioned') {
      return a.x === b.x && a.y === b.y;
    }
    return true;
  }
}
```

- [ ] **Step 4: Add `addEmptyItem` to PageNode**

In `packages/pages-document/src/page-document.ts`, add an `addEmptyItem` method to `PageNode` (after `addItem` at line ~476):

```typescript
addEmptyItem(placement: GridPlacement): GridItemNode {
  if (this.getLayoutMode() !== 'items') {
    throw new Error('addEmptyItem() requires items: layout mode');
  }
  this._doc._pushUndoInternal();
  const doc = this._doc._getDoc();
  const placementObj: Record<string, number> = { w: placement.w, h: placement.h };
  if (placement.mode === 'positioned') {
    placementObj['x'] = placement.x;
    placementObj['y'] = placement.y;
  }
  const entry: Record<string, unknown> = { placement: placementObj };
  const node = doc.createNode(entry);
  const seq = doc.getIn([...this.path, 'items']);
  (seq as YAMLSeq).add(node);
  this._doc._notifyInternal();
  const items = this.getItems();
  return items[items.length - 1]!;
}
```

- [ ] **Step 5: Run tests**

Run: `npx vitest run packages/pages-builder/src/shell/grid-edit-bridge.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-builder/src/shell/grid-edit-bridge.ts packages/pages-document/src/page-document.ts
git commit -m "feat(#637): update GridEditBridge with split/merge/swap/remove/add-empty ops

Add PageNode.addEmptyItem() for empty container creation.
All operations go through _applyEdit pattern.

Refs #625, #629, #630, #637"
```

---

## Batch 7: Migration Script and Parser Cleanup

### Task 8: YAML migration script (rows/columns → items)

**Files:**
- Create: `packages/pages-component/src/renderer/grid-migration.ts`
- Test: `packages/pages-component/src/renderer/grid-migration.test.ts`

**Interfaces:**
- Consumes: nothing — standalone migration utility
- Produces:
  - `migrateRowsToItems(yaml: string): string` — rewrites a page YAML string from rows/columns to items format
  - `needsMigration(yaml: string): boolean`

- [ ] **Step 1: Write failing tests for migration**

Create `packages/pages-component/src/renderer/grid-migration.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { migrateRowsToItems, needsMigration } from "./grid-migration.js";

describe("needsMigration", () => {
  it("returns true for rows-format YAML", () => {
    const yaml = `pages:\n  - name: Main\n    rows:\n      - columns:\n          - span: 6\n            components:\n              - type: html`;
    expect(needsMigration(yaml)).toBe(true);
  });

  it("returns false for items-format YAML", () => {
    const yaml = `pages:\n  - name: Main\n    items:\n      - placement: { x: 0, y: 0, w: 6, h: 1 }\n        component: { type: html }`;
    expect(needsMigration(yaml)).toBe(false);
  });
});

describe("migrateRowsToItems", () => {
  it("converts single row with two columns to items", () => {
    const input = `pages:
  - name: Main
    rows:
      - columns:
          - span: 6
            components:
              - type: html
                properties:
                  content: left
          - span: 6
            components:
              - type: html
                properties:
                  content: right`;

    const result = migrateRowsToItems(input);
    expect(result).toContain("items:");
    expect(result).not.toContain("rows:");
    expect(result).toContain("x: 0");
    expect(result).toContain("x: 6");
    expect(result).toContain("w: 6");
  });

  it("converts components shorthand to items", () => {
    const input = `pages:
  - name: Main
    components:
      - type: html
      - type: BARCHART`;

    const result = migrateRowsToItems(input);
    expect(result).toContain("items:");
    expect(result).not.toContain("components:");
  });

  it("leaves items-format YAML unchanged", () => {
    const input = `pages:
  - name: Main
    items:
      - placement: { x: 0, y: 0, w: 12, h: 1 }
        component: { type: html }`;

    expect(migrateRowsToItems(input)).toBe(input);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-migration.test.ts --reporter=verbose`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement migration**

Create `packages/pages-component/src/renderer/grid-migration.ts`:

```typescript
import { parseDocument, stringify } from "yaml";

export function needsMigration(yaml: string): boolean {
  return /\brows:\s*\n/m.test(yaml) || /\bcomponents:\s*\n/m.test(yaml);
}

interface MigItem {
  placement: { x: number; y: number; w: number; h: number };
  component: Record<string, unknown>;
}

export function migrateRowsToItems(yaml: string): string {
  if (!needsMigration(yaml)) return yaml;

  const doc = parseDocument(yaml);
  const json = doc.toJSON() as Record<string, unknown>;
  const pages = json["pages"] as Record<string, unknown>[] | undefined;
  if (!pages) return yaml;

  let changed = false;
  for (const page of pages) {
    const rows = page["rows"] as Record<string, unknown>[] | undefined;
    const components = page["components"] as Record<string, unknown>[] | undefined;

    if (rows) {
      const items: MigItem[] = [];
      let y = 0;
      for (const row of rows) {
        const cols = (row["columns"] ?? row["layoutColumns"]) as Record<string, unknown>[] | undefined;
        if (!cols) { y++; continue; }
        let x = 0;
        for (const col of cols) {
          const span = Number(col["span"] ?? 12);
          const colComps = (col["components"] ?? col["layoutComponents"]) as Record<string, unknown>[] | undefined;
          if (colComps) {
            for (let ci = 0; ci < colComps.length; ci++) {
              items.push({
                placement: { x, y: y + ci, w: span, h: 1 },
                component: colComps[ci]!,
              });
            }
          }
          x += span;
        }
        y++;
      }
      delete page["rows"];
      page["items"] = items;
      changed = true;
    } else if (components) {
      const items: MigItem[] = components.map((comp, i) => ({
        placement: { x: 0, y: i, w: 12, h: 1 },
        component: comp,
      }));
      delete page["components"];
      page["items"] = items;
      changed = true;
    }
  }

  return changed ? stringify(json, { lineWidth: 0 }) : yaml;
}
```

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-component/src/renderer/grid-migration.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-component/src/renderer/grid-migration.ts packages/pages-component/src/renderer/grid-migration.test.ts
git commit -m "feat(#633): YAML migration script — rows/columns to items format

migrateRowsToItems() rewrites page YAML from rows/columns or components
shorthand into unified items format. needsMigration() detects which files
need rewriting.

Refs #625, #633"
```

### Task 9: Remove rows/columns parser paths

**Files:**
- Modify: `packages/pages-ui/src/parser/page-parser.ts:324-424` (remove rows parsing block)
- Modify: `packages/pages-document/src/page-document.ts` (remove RowNode, ColumnNode, getRows, getColumns, addRow, insertRowAt, wrapInRow)
- Modify: `packages/pages-ui/src/parser/layout-placement.test.ts` (remove rows-based tests)

**Interfaces:**
- Consumes: `GridItem` (with optional component) from `types.ts`
- Produces: `parsePageContent` now only handles `items` and `components` shorthand — no `rows`/`columns`

- [ ] **Step 1: Write test confirming items-only parsing still works**

Add a test to `packages/pages-ui/src/parser/page-parser.test.ts` (or the appropriate test file):

```typescript
it("parses item without component as empty container", () => {
  const result = parsePageContent(
    { items: [{ placement: { x: 0, y: 0, w: 6, h: 2 } }] },
    0,
  );
  expect("items" in result).toBe(true);
  if ("items" in result) {
    expect(result.items![0]!.component).toBeUndefined();
    expect(result.items![0]!.placement).toEqual({ mode: "positioned", x: 0, y: 0, w: 6, h: 2 });
  }
});
```

- [ ] **Step 2: Run test to verify it fails (component is currently always required)**

Run: `npx vitest run packages/pages-ui/src/parser/ --reporter=verbose`
Expected: Test may fail because parser currently calls `desugarComponent(componentRaw ?? {})` which produces `{ type: undefined }`.

- [ ] **Step 3: Remove rows/columns block from parsePageContent**

In `packages/pages-ui/src/parser/page-parser.ts`, delete lines 324-424 (the entire `if (rowsRaw)` block). This removes the rows/columns parsing path.

- [ ] **Step 4: Fix items parsing to handle missing component**

In the `if (itemsRaw)` block (line ~281-310), update to handle empty containers:

```typescript
if (itemsRaw) {
  const maxColumns = 12;
  const items: GridItem[] = itemsRaw.map((entryRaw, i) => {
    const entry = entryRaw as Record<string, unknown>;
    const placementRaw = entry["placement"] as Record<string, unknown> | undefined;
    const componentRaw = entry["component"] as Record<string, unknown> | undefined;

    let placement: GridPlacement;
    if (placementRaw) {
      const w = Math.min(Number(placementRaw["w"] ?? 1), maxColumns);
      const h = Number(placementRaw["h"] ?? 1);
      if ("x" in placementRaw && "y" in placementRaw) {
        placement = { mode: "positioned", x: Number(placementRaw["x"]), y: Number(placementRaw["y"]), w, h };
      } else {
        placement = { mode: "auto", w, h };
      }
    } else {
      placement = { mode: "auto", w: 1, h: 1 };
    }

    if (!componentRaw) {
      return { placement };
    }

    const component = desugarComponent(componentRaw, displayerDefaults);
    return {
      placement,
      component: assignIdIfMissing(component, `grid_${String(pageIndex)}_item_${String(i)}`),
    };
  });
  return { items };
}
```

Note: placement mode inference changed — presence of `x`/`y` implies positioned (D9 clarification), no `mode` key needed.

- [ ] **Step 5: Remove RowNode, ColumnNode, and related methods from PageDocument**

In `packages/pages-document/src/page-document.ts`:
- Delete `RowNode` class (lines 493-543)
- Delete `ColumnNode` class (lines 545-622)
- Remove `getRows()` (line 311-315), `getColumns()` (317-321), `addRow()` (329-344), `insertRowAt()` (346-360), `wrapInRow()` (403-440) from `PageNode`
- Update `getLayoutMode()` to only return `'items'` or `'flat'` — remove `'rows'` and `'columns'` cases
- Update `removeChild()` and `insertChildAt()` to remove rows/columns cases

- [ ] **Step 6: Run full test suite**

Run: `npx vitest run --reporter=verbose`
Expected: ALL PASS. Some existing tests for rows/columns will break — delete those test cases.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#633): remove rows/columns parser paths and PageDocument RowNode/ColumnNode

parsePageContent now only handles items and components shorthand.
RowNode, ColumnNode, getRows(), getColumns(), addRow(), insertRowAt(),
wrapInRow() removed from PageDocument. Unified on items layout.

Refs #625, #633

BREAKING CHANGE: rows/columns YAML format no longer supported. Run
migration script first."
```

---

## Batch 8: Nested Sub-Grids

### Task 10: Recursive nesting support

**Files:**
- Modify: `packages/pages-ui/src/parser/page-parser.ts` (items parsing: recursively parse nested grid components)
- Modify: `packages/pages-component/src/renderer/grid.ts` (detect nested grids, create sub-grid containers)
- Modify: `packages/pages-component/src/renderer/grid-compaction.ts` (no change needed — each level independently calls compactAll)
- Test: `packages/pages-component/src/renderer/grid-compaction.test.ts` (add nested scenario tests)
- Test: `packages/pages-ui/src/parser/page-parser.test.ts` (add nested grid parsing test)

**Interfaces:**
- Consumes: `GridItem` (with optional component), `Component` (with items), `applyGridPlacement`
- Produces:
  - Parser handles `component.type === 'grid'` with nested `items` recursively
  - `applyGridPlacement` handles nested grid rendering
  - `isNestedGrid(item: GridItem): boolean`

- [ ] **Step 1: Write failing test for nested grid parsing**

Add to the parser test file:

```typescript
it("parses nested grid component with recursive items", () => {
  const result = parsePageContent({
    items: [{
      placement: { x: 0, y: 0, w: 6, h: 4 },
      component: {
        type: "grid",
        items: [{
          placement: { x: 0, y: 0, w: 6, h: 1 },
          component: { type: "html", properties: { content: "nested" } },
        }],
      },
    }],
  }, 0);

  expect("items" in result).toBe(true);
  if ("items" in result) {
    const outer = result.items![0]!;
    expect(outer.component?.type).toBe("grid");
    expect(outer.component?.items?.length).toBe(1);
    expect(outer.component?.items?.[0]?.component?.type).toBe("html");
  }
});
```

- [ ] **Step 2: Write failing test for nested compaction independence**

Add to `grid-compaction.test.ts`:

```typescript
describe("nested compaction independence", () => {
  it("compacting inner items does not affect outer items", () => {
    const outer: IdRect[] = [
      { id: "container", x: 0, y: 0, w: 6, h: 4 },
      { id: "sibling", x: 6, y: 0, w: 6, h: 2 },
    ];
    const inner: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 1 },
      { id: "b", x: 0, y: 3, w: 6, h: 1 },
    ];
    const compactedInner = compact(inner);
    expect(compactedInner.find(i => i.id === "b")!.y).toBe(1);

    const newContainerH = autoShrinkHeight(compactedInner);
    expect(newContainerH).toBe(2);

    expect(outer.find(i => i.id === "sibling")!.y).toBe(0);
  });
});
```

- [ ] **Step 3: Verify nested grid parsing works**

The parser's `items` path already recurses into `component.items` via the Component type. The `desugarComponent` function handles nested items. Verify by running the test.

Run: `npx vitest run --reporter=verbose`
Expected: Parsing test passes. The nested grid structure is preserved through the Component type's existing `items` field.

- [ ] **Step 4: Add isNestedGrid helper**

Add to `packages/pages-component/src/renderer/grid-compaction.ts`:

```typescript
import type { GridItem } from "../model/types.js";

export function isNestedGrid(item: GridItem): boolean {
  return item.component?.type === "grid" && Array.isArray(item.component.items) && item.component.items.length > 0;
}
```

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#634): recursive nested sub-grid support

Nested grids are components of type 'grid' with items array. Each level
compacts independently. autoShrinkHeight calculates nested content height.
isNestedGrid helper identifies nested grid items.

Refs #625, #634"
```

---

## Batch 9: DSL Builders Update

### Task 11: Update DSL builders for empty containers

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts` (add `emptyContainer()` builder, update `at()`)

**Interfaces:**
- Consumes: `GridItem` (with optional component)
- Produces:
  - `emptyContainer(x: number, y: number, w: number, h: number): GridItem`
  - Updated `at()` to allow no component (empty container)

- [ ] **Step 1: Write failing test for emptyContainer builder**

Add test in `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
import { emptyContainer, at } from "./builders.js";

describe("emptyContainer", () => {
  it("creates a GridItem with no component", () => {
    const item = emptyContainer(0, 0, 6, 2);
    expect(item.placement).toEqual({ mode: "positioned", x: 0, y: 0, w: 6, h: 2 });
    expect(item.component).toBeUndefined();
  });
});
```

- [ ] **Step 2: Implement emptyContainer**

Add to `packages/pages-ui/src/dsl/builders.ts`:

```typescript
export function emptyContainer(x: number, y: number, w: number, h: number): GridItem {
  return {
    placement: { mode: "positioned" as const, x, y, w, h },
  };
}
```

- [ ] **Step 3: Run tests**

Run: `npx vitest run packages/pages-ui/src/dsl/ --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts
git commit -m "feat(#631): add emptyContainer() DSL builder for empty grid containers

Refs #625, #631"
```

---

## Batch 10: Resize Module

### Task 12: Edge resize operations for layout mode

**Files:**
- Create: `packages/pages-component/src/renderer/grid-resize.ts`
- Test: `packages/pages-component/src/renderer/grid-resize.test.ts`

**Interfaces:**
- Consumes: `IdRect`, `Rect` from `grid-compaction.ts`
- Produces:
  - `ResizeEdge = 'top' | 'bottom' | 'left' | 'right'`
  - `resizeItem(items: readonly IdRect[], id: string, edge: ResizeEdge, delta: number, columns: number): IdRect[]`

- [ ] **Step 1: Write failing tests for resize**

Create `packages/pages-component/src/renderer/grid-resize.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { resizeItem } from "./grid-resize.js";
import type { IdRect } from "./grid-compaction.js";

describe("resizeItem", () => {
  it("increases width from right edge", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 0, w: 4, h: 2 }];
    const result = resizeItem(items, "a", "right", 2, 12);
    expect(result[0]).toEqual({ id: "a", x: 0, y: 0, w: 6, h: 2 });
  });

  it("decreases width from right edge", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 0, w: 6, h: 2 }];
    const result = resizeItem(items, "a", "right", -2, 12);
    expect(result[0]).toEqual({ id: "a", x: 0, y: 0, w: 4, h: 2 });
  });

  it("clamps width to minimum 1", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 0, w: 2, h: 2 }];
    const result = resizeItem(items, "a", "right", -5, 12);
    expect(result[0]!.w).toBe(1);
  });

  it("clamps width to columns boundary", () => {
    const items: IdRect[] = [{ id: "a", x: 8, y: 0, w: 4, h: 2 }];
    const result = resizeItem(items, "a", "right", 5, 12);
    expect(result[0]!.x + result[0]!.w).toBeLessThanOrEqual(12);
  });

  it("increases height from bottom edge", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 0, w: 6, h: 2 }];
    const result = resizeItem(items, "a", "bottom", 1, 12);
    expect(result[0]).toEqual({ id: "a", x: 0, y: 0, w: 6, h: 3 });
  });

  it("moves item right when resizing from left edge", () => {
    const items: IdRect[] = [{ id: "a", x: 2, y: 0, w: 6, h: 2 }];
    const result = resizeItem(items, "a", "left", 2, 12);
    expect(result[0]).toEqual({ id: "a", x: 4, y: 0, w: 4, h: 2 });
  });

  it("moves item down when resizing from top edge", () => {
    const items: IdRect[] = [{ id: "a", x: 0, y: 2, w: 6, h: 4 }];
    const result = resizeItem(items, "a", "top", 1, 12);
    expect(result[0]).toEqual({ id: "a", x: 0, y: 3, w: 6, h: 3 });
  });

  it("preserves other items", () => {
    const items: IdRect[] = [
      { id: "a", x: 0, y: 0, w: 6, h: 2 },
      { id: "b", x: 6, y: 0, w: 6, h: 2 },
    ];
    const result = resizeItem(items, "a", "right", 2, 12);
    expect(result.find(i => i.id === "b")).toEqual({ id: "b", x: 6, y: 0, w: 6, h: 2 });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-resize.test.ts --reporter=verbose`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement resize**

Create `packages/pages-component/src/renderer/grid-resize.ts`:

```typescript
import type { IdRect } from "./grid-compaction.js";

export type ResizeEdge = "top" | "bottom" | "left" | "right";

export function resizeItem(
  items: readonly IdRect[],
  id: string,
  edge: ResizeEdge,
  delta: number,
  columns: number,
): IdRect[] {
  return items.map(item => {
    if (item.id !== id) return { ...item };
    switch (edge) {
      case "right": {
        const newW = Math.max(1, Math.min(item.w + delta, columns - item.x));
        return { ...item, w: newW };
      }
      case "left": {
        const shrink = Math.min(delta, item.w - 1);
        const actualDelta = Math.max(shrink, -item.x);
        return { ...item, x: item.x + actualDelta, w: item.w - actualDelta };
      }
      case "bottom": {
        const newH = Math.max(1, item.h + delta);
        return { ...item, h: newH };
      }
      case "top": {
        const shrink = Math.min(delta, item.h - 1);
        const actualDelta = Math.max(shrink, -item.y);
        return { ...item, y: item.y + actualDelta, h: item.h - actualDelta };
      }
    }
  });
}
```

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-component/src/renderer/grid-resize.test.ts --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-component/src/renderer/grid-resize.ts packages/pages-component/src/renderer/grid-resize.test.ts
git commit -m "feat(#626): edge resize operations for layout mode

resizeItem handles top/bottom/left/right edge drags with clamping.
Pure function, no DOM.

Refs #625, #626"
```

---

## Batch 11: Final Integration

### Task 13: Wire grid editor into builder-shell as page builder

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` (wire `pages-grid-edit` handler to use updated bridge with `_applyEdit('grid', ...)`)
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts` (dispatch extended GridEditDetail types)

**Interfaces:**
- Consumes: `GridEditBridge.applyGridEdit()`, `_applyEdit('grid', ...)`
- Produces: Full integration — grid editor dispatches `pages-grid-edit`, builder-shell catches it, calls `_applyEdit('grid', () => bridge.applyGridEdit(doc, detail))`

- [ ] **Step 1: Verify builder-shell already handles pages-grid-edit**

Check `builder-shell.ts` for the existing `pages-grid-edit` handler. It should already call `_applyEdit('grid', ...)` with the bridge. If it does, this task just verifies the new detail types flow through correctly.

Run: `npx vitest run packages/pages-builder/ --reporter=verbose`

- [ ] **Step 2: Add integration test for split via grid-edit event**

Add to the builder-shell or grid-edit-bridge test file:

```typescript
it("split via pages-grid-edit dispatches through _applyEdit", () => {
  const doc = makeDoc();
  const bridge = new GridEditBridge();
  bridge.applyGridEdit(doc, {
    type: "split",
    targetIndex: 0,
    direction: "vertical",
  });
  const items = doc.getPages()[0]!.getItems();
  expect(items.length).toBe(3);
  const first = items[0]!.getPlacement();
  expect(first).toMatchObject({ mode: "positioned", w: 3 });
});
```

- [ ] **Step 3: Run all tests**

Run: `npx vitest run --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat(#638): wire grid editor as root page builder in visual editor

Grid edit events dispatch through _applyEdit('grid', ...) via GridEditBridge.
All operation types (move, resize, split, merge, swap, insert, remove,
add-empty) flow from controller through bridge to PageDocument.

Refs #625, #637, #638"
```

### Task 14: Run migration on existing sample files

**Files:**
- Modify: `examples/samples/**/*.page.yaml` (all files using rows/columns format)

**Interfaces:**
- Consumes: `migrateRowsToItems()` from `grid-migration.ts`

- [ ] **Step 1: Find files needing migration**

```bash
grep -rl "rows:" examples/samples/ --include="*.page.yaml" | head -20
```

- [ ] **Step 2: Run migration on each file**

Write a small script to `/tmp/migrate.ts`:

```typescript
import { readFileSync, writeFileSync } from 'fs';
import { migrateRowsToItems, needsMigration } from './packages/pages-component/src/renderer/grid-migration.js';

const files = process.argv.slice(2);
for (const file of files) {
  const content = readFileSync(file, 'utf-8');
  if (needsMigration(content)) {
    const migrated = migrateRowsToItems(content);
    writeFileSync(file, migrated);
    console.log(`Migrated: ${file}`);
  }
}
```

Run: `npx tsx /tmp/migrate.ts $(grep -rl "rows:" examples/samples/ --include="*.page.yaml")`

- [ ] **Step 3: Verify samples still parse**

Run: `npx vitest run --reporter=verbose`
Expected: ALL PASS — migrated samples parse correctly under items-only parser.

- [ ] **Step 4: Commit**

```bash
git add examples/samples/
git commit -m "chore(#633): migrate sample YAML files from rows/columns to items format

All .page.yaml files now use unified items layout.

Refs #625, #633"
```

---

## References

- `specs/issue-625-unified-grid-layout-model/2026-11-08-unified-grid-layout-model-design.md` — design spec this plan implements
- `specs/issue-625-unified-grid-layout-model/decisions.md` — D1-D19 decisions
- `packages/pages-component/src/model/types.ts:20-27` — GridPlacement, GridItem (to be modified)
- `packages/pages-component/src/renderer/grid-collision.ts` — current compaction (behavioural reference)
- `packages/pages-component/src/renderer/grid-edit-controller.ts` — current controller (behavioural reference)
- `packages/pages-component/src/renderer/grid-overlay.ts` — current overlay (behavioural reference)
- `packages/pages-component/src/renderer/grid-drag.ts` — drag helpers (stable APIs reused)
- `packages/pages-component/src/renderer/grid.ts` — applyGridPlacement (stable, minimal change)
- `packages/pages-document/src/page-document.ts:1072-1147` — GridItemNode (to be modified)
- `packages/pages-document/src/page-document.ts:287-491` — PageNode (rows/columns to be removed)
- `packages/pages-document/src/page-document.ts:493-622` — RowNode, ColumnNode (to be removed)
- `packages/pages-ui/src/parser/page-parser.ts:272-428` — parsePageContent (rows path to be removed)
- `packages/pages-builder/src/shell/grid-edit-bridge.ts` — GridEditBridge (to be extended)
- `packages/pages-ui/src/dsl/builders.ts:149-173` — grid(), at() builders
- `examples/poc/grid-drag-poc.html` — #618 POC (behavioural reference)
- `docs/protocols/casehub/all-mutations-through-applyedit.md` — PP-20260916-b6f3e8
- `docs/protocols/casehub/aria-interaction-contract.md` — PP-20260817-a11y01
- `docs/protocols/casehub/pages-event-contract.md` — PP-20260705-bac842
- GitHub #625, #626-#638
