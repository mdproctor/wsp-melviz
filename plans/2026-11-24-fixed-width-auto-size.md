# Fixed-Width Panel Auto-Size Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #619 — Fixed-width panels should auto-size column allocation to reduce visual gaps
**Issue group:** #619

**Goal:** Add `fixedWidth` to `GridPlacement` and compute the minimum column span at render time, so fixed-width panels don't waste grid columns.

**Architecture:** Add an optional `fixedWidth?: number` field to the `GridPlacement` type. At render time, measure the grid container's width, compute `cellWidth`, and derive `effectiveW` — the minimum columns needed to contain the fixed pixel width. Apply `effectiveW` as the CSS grid span, with `justify-self: start` and explicit `width` on the element. Preserve `fixedWidth` through YAML parse/serialize and all edit bridge operations.

**Tech Stack:** TypeScript, Vitest, CSS Grid, `yaml` library (YAML document manipulation)

## Global Constraints

- `effectiveW = Math.min(Math.ceil((fixedWidth + gap) / (cellWidth + gap)), w)` — gap-aware formula, clamped to declared `w`
- Default gap: `var(--pages-space-3, 12px)` — hardcoded as 12 in geometry calculations
- Default columns: 12
- No changes to `grid-compaction.ts`, `grid-edit-controller.ts`, or `grid-scale.ts`

---

## Batch 1: Type model and formula

### Task 1: effectiveW computation function and tests

**Files:**
- Create: `packages/pages-component/src/renderer/grid-fixed-width.ts`
- Create: `packages/pages-component/src/renderer/grid-fixed-width.test.ts`
- Modify: `packages/pages-component/src/renderer/index.ts` (add export)

**Interfaces:**
- Consumes: nothing
- Produces: `computeEffectiveWidth(fixedWidth: number, cellWidth: number, gap: number, declaredW: number): number` — returns clamped integer column span

- [ ] **Step 1: Write failing tests for the effectiveW formula**

Create the test file:

```ts
// packages/pages-component/src/renderer/grid-fixed-width.test.ts
import { describe, it, expect } from "vitest";
import { computeEffectiveWidth } from "./grid-fixed-width.js";

describe("computeEffectiveWidth", () => {
  it("returns 1 when fixedWidth fits in a single column", () => {
    // cellWidth=100, gap=12 → 1 col = 100px, fixedWidth=80 fits
    expect(computeEffectiveWidth(80, 100, 12, 4)).toBe(1);
  });

  it("returns 2 when fixedWidth exceeds one column", () => {
    // cellWidth=100, gap=12 → 1 col = 100px, 2 cols = 212px, fixedWidth=160
    expect(computeEffectiveWidth(160, 100, 12, 4)).toBe(2);
  });

  it("accounts for inter-column gap in span calculation", () => {
    // cellWidth=100, gap=12 → 2 cols = 100+12+100 = 212px
    // fixedWidth=210 fits in 2 cols (210 < 212), not 3
    expect(computeEffectiveWidth(210, 100, 12, 4)).toBe(2);
  });

  it("clamps to declared w as upper bound", () => {
    // fixedWidth=500 needs ceil((500+12)/(100+12))=ceil(4.57)=5 cols
    // but declaredW=3, so clamp to 3
    expect(computeEffectiveWidth(500, 100, 12, 3)).toBe(3);
  });

  it("returns 1 when fixedWidth equals cellWidth exactly", () => {
    expect(computeEffectiveWidth(100, 100, 12, 4)).toBe(1);
  });

  it("returns declaredW when fixedWidth is 0", () => {
    // 0 fixedWidth → ceil((0+12)/(100+12))=ceil(0.107)=1
    // but this is an edge case — 0 means no constraint
    expect(computeEffectiveWidth(0, 100, 12, 4)).toBe(1);
  });

  it("clamps to 1 minimum", () => {
    expect(computeEffectiveWidth(10, 200, 12, 4)).toBe(1);
  });

  it("returns declaredW when cellWidth is very small", () => {
    // cellWidth=5, gap=12 → ceil((160+12)/(5+12))=ceil(10.1)=11
    // declaredW=4, clamp to 4
    expect(computeEffectiveWidth(160, 5, 12, 4)).toBe(4);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-component/src/renderer/grid-fixed-width.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement computeEffectiveWidth**

Create the module:

```ts
// packages/pages-component/src/renderer/grid-fixed-width.ts
export function computeEffectiveWidth(
  fixedWidth: number,
  cellWidth: number,
  gap: number,
  declaredW: number,
): number {
  if (fixedWidth <= 0 || cellWidth <= 0) return Math.max(1, Math.min(declaredW, 1));
  const needed = Math.ceil((fixedWidth + gap) / (cellWidth + gap));
  return Math.max(1, Math.min(needed, declaredW));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-component/src/renderer/grid-fixed-width.test.ts`
Expected: all 8 tests PASS

- [ ] **Step 5: Add export to renderer index**

Add to `packages/pages-component/src/renderer/index.ts`:

```ts
export { computeEffectiveWidth } from "./grid-fixed-width.js";
```

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/src/renderer/grid-fixed-width.ts packages/pages-component/src/renderer/grid-fixed-width.test.ts packages/pages-component/src/renderer/index.ts
git commit -m "feat(#619): effectiveW computation with gap-aware formula and clamping"
```

### Task 2: Add fixedWidth to GridPlacement type

**Files:**
- Modify: `packages/pages-component/src/model/types.ts:20-22`

**Interfaces:**
- Consumes: nothing
- Produces: `GridPlacement` with optional `readonly fixedWidth?: number` on both union variants

- [ ] **Step 1: Add fixedWidth to both GridPlacement variants**

Change lines 20-22 in `types.ts` from:

```ts
export type GridPlacement =
  | { readonly mode: "positioned"; readonly x: number; readonly y: number; readonly w: number; readonly h: number }
  | { readonly mode: "auto"; readonly w: number; readonly h: number };
```

to:

```ts
export type GridPlacement =
  | { readonly mode: "positioned"; readonly x: number; readonly y: number; readonly w: number; readonly h: number; readonly fixedWidth?: number }
  | { readonly mode: "auto"; readonly w: number; readonly h: number; readonly fixedWidth?: number };
```

- [ ] **Step 2: Verify no type errors**

Run: `yarn tsc --noEmit -p packages/pages-component/tsconfig.json`
Expected: no errors (optional field is backward compatible)

- [ ] **Step 3: Commit**

```bash
git add packages/pages-component/src/model/types.ts
git commit -m "feat(#619): add fixedWidth to GridPlacement type"
```

## Batch 2: YAML round-trip

### Task 3: Parse and persist fixedWidth in GridItemNode

**Files:**
- Modify: `packages/pages-document/src/page-document.ts:940-972` (`getPlacement` and `setPlacement`)
- Modify: `packages/pages-document/src/grid-item-node.test.ts` (add tests)

**Interfaces:**
- Consumes: `GridPlacement` with `fixedWidth?: number` (from Task 2)
- Produces: `GridItemNode.getPlacement()` returns `fixedWidth` when present in YAML; `GridItemNode.setPlacement()` persists it

- [ ] **Step 1: Write failing test for getPlacement with fixedWidth**

Add to `grid-item-node.test.ts` inside the `getPlacement` describe:

```ts
it('reads fixedWidth from positioned placement', () => {
  const yaml = `pages:
  - title: Test
    items:
      - placement: { x: 8, y: 0, w: 2, h: 1, fixedWidth: 160 }
        component:
          type: html
`;
  const doc = parse(yaml);
  const p = doc.getPages()[0]!.getItems()[0]!.getPlacement();
  expect(p).toEqual({ mode: 'positioned', x: 8, y: 0, w: 2, h: 1, fixedWidth: 160 });
});

it('omits fixedWidth when not in YAML', () => {
  const doc = parse(GRID_PAGE_YAML);
  const p = doc.getPages()[0]!.getItems()[0]!.getPlacement();
  expect(p.fixedWidth).toBeUndefined();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-document/src/grid-item-node.test.ts`
Expected: first test FAIL — `fixedWidth` not in returned object

- [ ] **Step 3: Update getPlacement to read fixedWidth**

In `page-document.ts`, update `getPlacement()` (around line 940). After reading `x`, `y`, `w`, `h`, also read `fixedWidth`:

Change the positioned return (line 948-954) from:

```ts
    if ('x' in obj && 'y' in obj) {
      return {
        mode: 'positioned',
        x: obj.x as number,
        y: obj.y as number,
        w: (obj.w as number) ?? 1,
        h: (obj.h as number) ?? 1,
      };
    }
```

to:

```ts
    if ('x' in obj && 'y' in obj) {
      const p: GridPlacement = {
        mode: 'positioned',
        x: obj.x as number,
        y: obj.y as number,
        w: (obj.w as number) ?? 1,
        h: (obj.h as number) ?? 1,
      };
      const fw = obj.fixedWidth as number | undefined;
      if (fw != null) return { ...p, fixedWidth: fw };
      return p;
    }
```

And the auto return (line 956) from:

```ts
    return { mode: 'auto', w: (obj.w as number) ?? 1, h: (obj.h as number) ?? 1 };
```

to:

```ts
    const autoP: GridPlacement = { mode: 'auto', w: (obj.w as number) ?? 1, h: (obj.h as number) ?? 1 };
    const autoFw = obj.fixedWidth as number | undefined;
    if (autoFw != null) return { ...autoP, fixedWidth: autoFw };
    return autoP;
```

- [ ] **Step 4: Run tests to verify getPlacement passes**

Run: `yarn vitest run packages/pages-document/src/grid-item-node.test.ts`
Expected: new getPlacement tests PASS

- [ ] **Step 5: Write failing test for setPlacement with fixedWidth**

Add to `grid-item-node.test.ts` inside the `setPlacement` describe:

```ts
it('persists fixedWidth in YAML round-trip', () => {
  const yaml = `pages:
  - title: Test
    items:
      - placement: { x: 0, y: 0, w: 6, h: 2 }
        component:
          type: chart
`;
  const doc = parse(yaml);
  const item = doc.getPages()[0]!.getItems()[0]!;
  item.setPlacement({ mode: 'positioned', x: 0, y: 0, w: 3, h: 1, fixedWidth: 160 });
  const result = doc.toString();
  expect(result).toContain('fixedWidth: 160');
  const updated = parse(result).getPages()[0]!.getItems()[0]!.getPlacement();
  expect(updated).toEqual({ mode: 'positioned', x: 0, y: 0, w: 3, h: 1, fixedWidth: 160 });
});

it('removes fixedWidth when not in new placement', () => {
  const yaml = `pages:
  - title: Test
    items:
      - placement: { x: 0, y: 0, w: 2, h: 1, fixedWidth: 160 }
        component:
          type: chart
`;
  const doc = parse(yaml);
  const item = doc.getPages()[0]!.getItems()[0]!;
  item.setPlacement({ mode: 'positioned', x: 0, y: 0, w: 6, h: 2 });
  const result = doc.toString();
  expect(result).not.toContain('fixedWidth');
});
```

- [ ] **Step 6: Run tests to verify setPlacement tests fail**

Run: `yarn vitest run packages/pages-document/src/grid-item-node.test.ts`
Expected: FAIL — fixedWidth not persisted

- [ ] **Step 7: Update setPlacement to write fixedWidth**

In `page-document.ts`, update `setPlacement()` (around line 959). After writing `w`, `h`, `x`, `y`, also handle `fixedWidth`:

Change `setPlacement` from:

```ts
  setPlacement(p: GridPlacement): void {
    this._doc._pushUndoInternal();
    const doc = this._doc._getDoc();
    const hasAt = doc.getIn([...this.path, 'at']) != null;
    const atKey = hasAt ? 'at' : (doc.getIn([...this.path, 'placement']) != null ? 'placement' : 'at');
    const base = [...this.path, atKey];
    doc.setIn([...base, 'w'], p.w);
    doc.setIn([...base, 'h'], p.h);
    if (p.mode === 'positioned') {
      doc.setIn([...base, 'x'], p.x);
      doc.setIn([...base, 'y'], p.y);
    }
    this._doc._notifyInternal();
  }
```

to:

```ts
  setPlacement(p: GridPlacement): void {
    this._doc._pushUndoInternal();
    const doc = this._doc._getDoc();
    const hasAt = doc.getIn([...this.path, 'at']) != null;
    const atKey = hasAt ? 'at' : (doc.getIn([...this.path, 'placement']) != null ? 'placement' : 'at');
    const base = [...this.path, atKey];
    doc.setIn([...base, 'w'], p.w);
    doc.setIn([...base, 'h'], p.h);
    if (p.mode === 'positioned') {
      doc.setIn([...base, 'x'], p.x);
      doc.setIn([...base, 'y'], p.y);
    }
    if (p.fixedWidth != null) {
      doc.setIn([...base, 'fixedWidth'], p.fixedWidth);
    } else {
      doc.deleteIn([...base, 'fixedWidth']);
    }
    this._doc._notifyInternal();
  }
```

- [ ] **Step 8: Run all grid-item-node tests**

Run: `yarn vitest run packages/pages-document/src/grid-item-node.test.ts`
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-document/src/page-document.ts packages/pages-document/src/grid-item-node.test.ts
git commit -m "feat(#619): parse and persist fixedWidth in GridItemNode"
```

## Batch 3: Render-time auto-sizing

### Task 4: Apply fixedWidth at render time

**Files:**
- Modify: `packages/pages-component/src/renderer/render.ts:116-165` (grid render loop)
- Modify: `packages/pages-component/src/renderer/render.test.ts` (add tests)

**Interfaces:**
- Consumes: `computeEffectiveWidth(fixedWidth, cellWidth, gap, declaredW)` from Task 1; `GridPlacement.fixedWidth` from Task 2
- Produces: grid items with `fixedWidth` render with reduced column span, `justify-self: start`, and explicit `width`

- [ ] **Step 1: Write failing test for fixed-width rendering**

Add to `render.test.ts`. The test needs `jsdom` environment (already annotated in the file). Note: in jsdom, `getBoundingClientRect()` returns zero-width rects. We'll need to mock it or test at a higher level. Instead, test the CSS properties directly by setting up a mock scenario:

```ts
describe("fixed-width panel rendering", () => {
  it("applies justify-self and width when fixedWidth is present", () => {
    const component: Component = {
      type: "page",
      grid: [
        {
          at: { mode: "positioned", x: 0, y: 0, w: 4, h: 1, fixedWidth: 160 },
          component: { type: "html", props: { content: "fixed" } },
        },
        {
          at: { mode: "positioned", x: 4, y: 0, w: 8, h: 1 },
          component: { type: "html", props: { content: "normal" } },
        },
      ],
    };
    const target = document.createElement("div");
    renderComponent(target, component);

    const items = target.querySelectorAll("[data-item-id]");
    const fixedItem = items[0] as HTMLElement;
    const normalItem = items[1] as HTMLElement;

    // In jsdom, containerWidth = 0, so effectiveW = min(ceil((160+12)/(0+12)), 4) = min(15, 4) = 4
    // With zero-width container, effectiveW clamps to w, so no shrink occurs
    // But justify-self and width should still be applied for fixedWidth panels
    expect(fixedItem.style.width).toBe("160px");
    expect(fixedItem.style.justifySelf).toBe("start");
    expect(normalItem.style.width).toBe("");
    expect(normalItem.style.justifySelf).toBe("");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-component/src/renderer/render.test.ts`
Expected: FAIL — no width/justifySelf applied

- [ ] **Step 3: Implement render-time fixedWidth handling**

In `render.ts`, after the import of `applyGridPlacement`, add import:

```ts
import { computeEffectiveWidth } from "./grid-fixed-width.js";
```

Then in the grid rendering section (around line 116-125), after setting up `gridTarget.style.gap`, compute grid geometry. Update the section that applies grid placement on each item (around line 156-164).

Replace the grid item rendering block. After `gridTarget.style.gap = "var(--pages-space-3, 12px)";` (line 125), add container measurement:

```ts
    const gapPx = 12;
    const containerWidth = gridTarget.getBoundingClientRect().width;
    const cellWidth = columns > 0 && containerWidth > 0
      ? (containerWidth - gapPx * (columns - 1)) / columns
      : 0;
```

Then in the per-item loop, after `applyGridPlacement(child, item.at);` (line 160), add:

```ts
        if (item.at.fixedWidth != null && item.at.fixedWidth > 0) {
          if (cellWidth > 0) {
            const effectiveW = computeEffectiveWidth(item.at.fixedWidth, cellWidth, gapPx, item.at.w);
            if (effectiveW < item.at.w) {
              if (item.at.mode === "positioned") {
                child.style.gridColumn = `${String(item.at.x + 1)} / span ${String(effectiveW)}`;
              } else {
                child.style.gridColumn = `span ${String(effectiveW)}`;
              }
            }
          }
          child.style.width = `${String(item.at.fixedWidth)}px`;
          child.style.justifySelf = "start";
        }
```

- [ ] **Step 4: Run tests**

Run: `yarn vitest run packages/pages-component/src/renderer/render.test.ts`
Expected: all tests PASS (including existing tests)

- [ ] **Step 5: Run full pages-component test suite**

Run: `yarn vitest run --project pages-component`
Expected: all pass — no regressions

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/src/renderer/render.ts packages/pages-component/src/renderer/render.test.ts
git commit -m "feat(#619): apply fixedWidth auto-sizing at render time"
```

## Batch 4: Edit bridge preservation

### Task 5: Preserve fixedWidth in grid edit operations

**Files:**
- Modify: `packages/pages-builder/src/shell/grid-edit-bridge.ts` (multiple methods)
- Modify: `packages/pages-builder/src/shell/grid-edit-bridge.test.ts` (add tests)

**Interfaces:**
- Consumes: `GridPlacement.fixedWidth` (from Task 2)
- Produces: all edit operations preserve `fixedWidth` when present on the source item

The edit bridge constructs `at` objects with `{ x, y, w, h }` in multiple places. Each place that constructs an `at` object from an existing item must also carry `fixedWidth` if the source has it.

- [ ] **Step 1: Write failing test for move preserving fixedWidth**

Add to `grid-edit-bridge.test.ts`:

```ts
const GRID_FIXED_WIDTH_YAML = `pages:
  - title: Test
    grid:
      - at: { x: 0, y: 0, w: 4, h: 1, fixedWidth: 160 }
        type: html
      - at: { x: 4, y: 0, w: 8, h: 1 }
        type: chart
`;

describe('fixedWidth preservation', () => {
  it('preserves fixedWidth through move via setPlacement', () => {
    const doc = PageDocument.parse(GRID_FIXED_WIDTH_YAML);
    const bridge = new GridEditBridge();

    bridge.applyGridEdit(doc, {
      type: 'move',
      items: [
        { at: { mode: 'positioned', x: 2, y: 1, w: 4, h: 1, fixedWidth: 160 }, component: { type: 'html' } },
        { at: { mode: 'positioned', x: 4, y: 0, w: 8, h: 1 }, component: { type: 'chart' } },
      ],
    });

    const p = doc.getPages()[0]!.getItems()[0]!.getPlacement();
    expect(p).toEqual({ mode: 'positioned', x: 2, y: 1, w: 4, h: 1, fixedWidth: 160 });
  });

  it('preserves fixedWidth through nested grid move', () => {
    const yaml = `pages:
  - title: Test
    grid:
      - at: { x: 0, y: 0, w: 12, h: 2 }
        type: grid
        grid:
          - at: { x: 0, y: 0, w: 3, h: 1, fixedWidth: 100 }
            type: html
`;
    const doc = PageDocument.parse(yaml);
    const bridge = new GridEditBridge();
    const gridPath = ['pages', 0, 'grid', 0, 'grid'];

    bridge.applyGridEdit(doc, {
      type: 'move',
      items: [
        { at: { mode: 'positioned', x: 2, y: 0, w: 3, h: 1, fixedWidth: 100 }, component: { type: 'html' } },
      ],
      gridPath,
    });

    const result = doc.toString();
    expect(result).toContain('fixedWidth: 100');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-builder/src/shell/grid-edit-bridge.test.ts`
Expected: nested grid test FAIL — fixedWidth dropped from `at` node construction

- [ ] **Step 3: Update _applyPlacements for nested grids to preserve fixedWidth**

In `grid-edit-bridge.ts`, the nested grid path (line 74) constructs:

```ts
const atNode = yamlDoc.createNode({ x: eventAt.x, y: eventAt.y, w: eventAt.w, h: eventAt.h });
```

Change to:

```ts
const atObj: Record<string, number> = { x: eventAt.x, y: eventAt.y, w: eventAt.w, h: eventAt.h };
if (eventAt.fixedWidth != null) atObj['fixedWidth'] = eventAt.fixedWidth;
const atNode = yamlDoc.createNode(atObj);
```

Also update the comparison on line 72-73 to include fixedWidth:

```ts
if (currentAt && currentAt.x === eventAt.x && currentAt.y === eventAt.y &&
    currentAt.w === eventAt.w && currentAt.h === eventAt.h &&
    (currentAt.fixedWidth ?? undefined) === (eventAt.fixedWidth ?? undefined)) continue;
```

- [ ] **Step 4: Update _insertItem to preserve fixedWidth**

In `_insertItem` (line 136-138), change:

```ts
const atObj = placement.mode === 'positioned'
    ? { x: placement.x, y: placement.y, w: placement.w, h: placement.h }
    : { w: placement.w, h: placement.h };
```

to:

```ts
const atBase = placement.mode === 'positioned'
    ? { x: placement.x, y: placement.y, w: placement.w, h: placement.h }
    : { w: placement.w, h: placement.h };
const atObj = placement.fixedWidth != null ? { ...atBase, fixedWidth: placement.fixedWidth } : atBase;
```

- [ ] **Step 5: Update _splitItem to preserve fixedWidth on the original item**

In `_splitItem` for nested grids (line 169), after constructing new `at` nodes, check if the original entry had `fixedWidth` and carry it forward on the original (not the new split piece). The original entry's `at` is rebuilt — include `fixedWidth` from the original `p`:

After line 162 where `p` is extracted:

```ts
const p = (at as YAMLMap).toJSON() as { x: number; y: number; w: number; h: number; fixedWidth?: number };
```

Then at line 169 for horizontal split, if the original had `fixedWidth`, include it:

```ts
const topAt: Record<string, number> = { x: p.x, y: p.y, w: p.w, h: topH };
if (p.fixedWidth != null) topAt['fixedWidth'] = p.fixedWidth;
(entry as YAMLMap).set('at', yamlDoc.createNode(topAt));
```

And for vertical split (line 174):

```ts
const leftAt: Record<string, number> = { x: p.x, y: p.y, w: leftW, h: p.h };
if (p.fixedWidth != null) leftAt['fixedWidth'] = p.fixedWidth;
(entry as YAMLMap).set('at', yamlDoc.createNode(leftAt));
```

For the top-level `setPlacement` path (lines 197-203), `fixedWidth` is already handled by `setPlacement` from Task 3.

- [ ] **Step 6: Update _addEmptyContainer to preserve fixedWidth**

In `_addEmptyContainer` (line 362-364), change:

```ts
const atObj = placement.mode === 'positioned'
    ? { x: placement.x, y: placement.y, w: placement.w, h: placement.h }
    : { w: placement.w, h: placement.h };
```

to:

```ts
const atBase = placement.mode === 'positioned'
    ? { x: placement.x, y: placement.y, w: placement.w, h: placement.h }
    : { w: placement.w, h: placement.h };
const atObj = placement.fixedWidth != null ? { ...atBase, fixedWidth: placement.fixedWidth } : atBase;
```

- [ ] **Step 7: Update applyCrossGridEdit to preserve fixedWidth**

In `applyCrossGridEdit`, line 404 and line 426-428 construct `at` objects. Update:

Line 404:

```ts
const newSourceAt = yamlDoc.createNode(tp.mode === 'positioned' ? { x: tp.x, y: tp.y, w: tp.w, h: tp.h } : { w: tp.w, h: tp.h });
```

to:

```ts
const sourceAtBase = tp.mode === 'positioned' ? { x: tp.x, y: tp.y, w: tp.w, h: tp.h } : { w: tp.w, h: tp.h };
const newSourceAt = yamlDoc.createNode(tp.fixedWidth != null ? { ...sourceAtBase, fixedWidth: tp.fixedWidth } : sourceAtBase);
```

Lines 426-428:

```ts
const atObj: Record<string, number> = tp.mode === 'positioned'
    ? { x: tp.x, y: tp.y, w: tp.w, h: tp.h }
    : { w: tp.w, h: tp.h };
```

to:

```ts
const atObj: Record<string, number> = tp.mode === 'positioned'
    ? { x: tp.x, y: tp.y, w: tp.w, h: tp.h }
    : { w: tp.w, h: tp.h };
if (tp.fixedWidth != null) atObj['fixedWidth'] = tp.fixedWidth;
```

Also update line 420 similarly — the target item placements loop:

```ts
const atNode = yamlDoc.createNode({ x: tp.x, y: tp.y, w: tp.w, h: tp.h });
```

to:

```ts
const tpAt: Record<string, number> = { x: tp.x, y: tp.y, w: tp.w, h: tp.h };
if ((tp as GridPlacement & { fixedWidth?: number }).fixedWidth != null) tpAt['fixedWidth'] = (tp as GridPlacement & { fixedWidth?: number }).fixedWidth!;
const atNode = yamlDoc.createNode(tpAt);
```

Note: `targetItemPlacements` uses a different type (`{ index: number; x: number; y: number; w: number; h: number }`), not `GridPlacement`. This path doesn't carry `fixedWidth` — it's just repositioning displaced items. Leave it as-is since displaced items get their `fixedWidth` from the YAML node directly (which is preserved by the swap/clone operations on lines 301-302 and 407-412).

- [ ] **Step 8: Update _compactGrid to preserve fixedWidth**

In `_compactGrid` (line 386), when writing back compacted positions:

```ts
item.setPlacement({ mode: 'positioned', x: p.x, y: rect.y, w: p.w, h: p.h });
```

Change to include `fixedWidth`:

```ts
const compactedP: GridPlacement = { mode: 'positioned', x: p.x, y: rect.y, w: p.w, h: p.h };
item.setPlacement(p.fixedWidth != null ? { ...compactedP, fixedWidth: p.fixedWidth } : compactedP);
```

- [ ] **Step 9: Run all edit bridge tests**

Run: `yarn vitest run packages/pages-builder/src/shell/grid-edit-bridge.test.ts`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add packages/pages-builder/src/shell/grid-edit-bridge.ts packages/pages-builder/src/shell/grid-edit-bridge.test.ts
git commit -m "feat(#619): preserve fixedWidth in all grid edit bridge operations"
```

## Batch 5: Integration verification

### Task 6: Full integration test and typecheck

**Files:**
- No new files

**Interfaces:**
- Consumes: all prior tasks
- Produces: verified green build

- [ ] **Step 1: Run full typecheck across all packages**

Run: `yarn tsc --noEmit` (or the project's typecheck command)
Expected: no errors

- [ ] **Step 2: Run all tests across affected packages**

Run: `yarn vitest run --project pages-component && yarn vitest run --project pages-document && yarn vitest run --project pages-builder`
Expected: all pass

- [ ] **Step 3: Commit if any fixes were needed**

If any adjustments were made, commit them.

## References

- [2026-11-24-fixed-width-auto-size-design.md] — design spec this plan implements
- [packages/pages-component/src/model/types.ts:20-22] — GridPlacement type
- [packages/pages-component/src/renderer/render.ts:116-165] — grid render loop
- [packages/pages-component/src/renderer/grid.ts:1-14] — applyGridPlacement
- [packages/pages-component/src/renderer/grid-drag.ts:21-36] — computeGridGeometry
- [packages/pages-document/src/page-document.ts:940-972] — getPlacement, setPlacement
- [packages/pages-builder/src/shell/grid-edit-bridge.ts:60-94] — _applyPlacements
- [packages/pages-component/src/renderer/grid.test.ts] — existing grid placement tests
- [packages/pages-document/src/grid-item-node.test.ts] — existing document tests
- [packages/pages-builder/src/shell/grid-edit-bridge.test.ts] — existing bridge tests
- [GitHub #619] — focal issue
- [GitHub #743] — full sizing model (out of scope)
