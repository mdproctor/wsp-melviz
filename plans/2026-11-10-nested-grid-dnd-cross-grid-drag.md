# Nested Grid DnD + Cross-Grid Drag Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #639 — Nested grid DnD doesn't work — controller only targets root items
**Issue group:** #639

**Goal:** Enable independent DnD/resize/split/merge in nested grids, and cross-grid drag between any two grids in the hierarchy with proportional scaling.

**Architecture:** A new `GridEditCoordinator` module replaces the singleton controller pattern. The coordinator builds a tree of registered controllers via DOM containment, resolves the deepest grid under the pointer, handles cross-grid drag handoff between controllers, and resolves DOM → YAML paths for the bridge. Individual controllers retain all single-grid behavior and gain three new methods (`releaseItem`, `acceptForeignItem`, `clearForeignItem`) for cross-grid support.

**Tech Stack:** TypeScript, Vitest, JSDOM, Lit (pages-builder shell)

## Global Constraints

- All grid items use the `grid:` YAML key (post-#640 unification)
- `data-item-id` stores `originalIndex` — the YAML array index, not DOM order
- Proportional scaling formula: `Math.max(1, Math.round(sourceW * targetCols / sourceCols))`, clamped to `[1, targetColumns]`
- Cross-grid mutations use a single undo frame (`_pushUndoInternal` once, `_notifyInternal` once)
- Nested controllers call `e.stopPropagation()` on `pointerdown` to prevent parent controller interference

---

## Batch 1: Coordinator Foundation

After this batch: coordinator module exists, controllers register/deregister, tree is built from DOM, deepest-grid resolution works, and DOM → YAML path reconstruction is tested. No behavioral changes to existing DnD yet.

### Task 1: GridEditCoordinator — tree, registration, path resolution

**Files:**
- Create: `packages/pages-component/src/renderer/grid-edit-coordinator.ts`
- Test: `packages/pages-component/src/renderer/grid-edit-coordinator.test.ts`

**Interfaces:**
- Consumes: `GridEditController` interface from `grid-edit-controller.ts`
- Produces:
  - `getCoordinator(): GridEditCoordinator` — module-level accessor
  - `GridEditCoordinator.register(controller: GridEditController): void`
  - `GridEditCoordinator.deregister(controller: GridEditController): void`
  - `GridEditCoordinator.deepestGridAt(clientX: number, clientY: number): GridEditController | null`
  - `GridEditCoordinator.resolveGridPath(controller: GridEditController): (string | number)[]`
  - `GridEditCoordinator.getParent(controller: GridEditController): GridEditController | null`
  - `GridEditCoordinator.getControllerForEl(gridEl: HTMLElement): GridEditController | null`

- [ ] **Step 1: Write failing tests for tree building**

```typescript
// grid-edit-coordinator.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { getCoordinator } from "./grid-edit-coordinator.js";
import { attachGridEditController } from "./grid-edit-controller.js";
import type { GridItem } from "../model/types.js";

function makePositionedItems(count: number): GridItem[] {
  return Array.from({ length: count }, (_, i) => ({
    at: { mode: "positioned" as const, x: 0, y: i, w: 12, h: 1 },
    component: { type: "html", id: `item-${i}` },
  }));
}

function setupNestedGrids(): { root: HTMLElement; nested: HTMLElement } {
  const root = document.createElement("div");
  root.style.display = "grid";
  root.style.gridTemplateColumns = "repeat(12, 1fr)";
  document.body.appendChild(root);

  // Item 0 in root grid — contains a nested grid
  const item0 = document.createElement("div");
  item0.dataset.itemId = "0";
  item0.dataset.componentType = "panel";
  root.appendChild(item0);

  // Nested grid inside item 0
  const nested = document.createElement("div");
  nested.style.display = "grid";
  nested.style.gridTemplateColumns = "repeat(6, 1fr)";
  nested.dataset.componentType = "grid";
  item0.appendChild(nested);

  // Items inside nested grid
  const nestedItem = document.createElement("div");
  nestedItem.dataset.itemId = "0";
  nested.appendChild(nestedItem);

  return { root, nested };
}

describe("GridEditCoordinator", () => {
  beforeEach(() => {
    document.body.innerHTML = "";
    const coord = getCoordinator();
    // Clear all registrations
    for (const c of [...coord.controllers]) coord.deregister(c);
  });

  it("registers a controller and tracks it", () => {
    const root = document.createElement("div");
    root.style.display = "grid";
    root.style.gridTemplateColumns = "repeat(12, 1fr)";
    document.body.appendChild(root);
    const items = makePositionedItems(2);
    for (let i = 0; i < items.length; i++) {
      const el = document.createElement("div");
      el.dataset.itemId = String(i);
      root.appendChild(el);
    }
    const ctrl = attachGridEditController(root, items, 12, 60);
    const coord = getCoordinator();
    expect(coord.getControllerForEl(root)).toBe(ctrl);
    ctrl.dispose();
  });

  it("builds parent-child tree from DOM containment", () => {
    const { root, nested } = setupNestedGrids();
    const rootItems = makePositionedItems(1);
    const nestedItems = makePositionedItems(1);
    const rootCtrl = attachGridEditController(root, rootItems, 12, 60);
    const nestedCtrl = attachGridEditController(nested, nestedItems, 6, 60);
    const coord = getCoordinator();
    expect(coord.getParent(nestedCtrl)).toBe(rootCtrl);
    expect(coord.getParent(rootCtrl)).toBeNull();
    rootCtrl.dispose();
    nestedCtrl.dispose();
  });

  it("deregisters on dispose", () => {
    const root = document.createElement("div");
    root.style.display = "grid";
    root.style.gridTemplateColumns = "repeat(12, 1fr)";
    document.body.appendChild(root);
    const items = makePositionedItems(1);
    const el = document.createElement("div");
    el.dataset.itemId = "0";
    root.appendChild(el);
    const ctrl = attachGridEditController(root, items, 12, 60);
    const coord = getCoordinator();
    expect(coord.getControllerForEl(root)).toBe(ctrl);
    ctrl.dispose();
    expect(coord.getControllerForEl(root)).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose grid-edit-coordinator`
Expected: FAIL — module not found

- [ ] **Step 3: Implement coordinator — tree, register, deregister, parent resolution**

```typescript
// grid-edit-coordinator.ts
import type { GridEditController } from "./grid-edit-controller.js";

export class GridEditCoordinator {
  private _controllers = new Set<GridEditController>();
  private _parents = new Map<GridEditController, GridEditController | null>();
  private _dirty = true;

  get controllers(): ReadonlySet<GridEditController> {
    return this._controllers;
  }

  register(controller: GridEditController): void {
    this._controllers.add(controller);
    this._dirty = true;
  }

  deregister(controller: GridEditController): void {
    this._controllers.delete(controller);
    this._parents.delete(controller);
    this._dirty = true;
  }

  getControllerForEl(gridEl: HTMLElement): GridEditController | null {
    for (const c of this._controllers) {
      if (c.gridEl === gridEl) return c;
    }
    return null;
  }

  getParent(controller: GridEditController): GridEditController | null {
    this._rebuildIfDirty();
    return this._parents.get(controller) ?? null;
  }

  deepestGridAt(clientX: number, clientY: number): GridEditController | null {
    this._rebuildIfDirty();
    let deepest: GridEditController | null = null;
    let deepestDepth = -1;
    for (const c of this._controllers) {
      if (!c.active) continue;
      const rect = c.gridEl.getBoundingClientRect();
      if (clientX >= rect.left && clientX <= rect.right &&
          clientY >= rect.top && clientY <= rect.bottom) {
        const depth = this._depth(c);
        if (depth > deepestDepth) {
          deepestDepth = depth;
          deepest = c;
        }
      }
    }
    return deepest;
  }

  resolveGridPath(controller: GridEditController): (string | number)[] {
    const segments: (string | number)[] = [];
    let el: HTMLElement | null = controller.gridEl;
    while (el && el !== document.body) {
      if (el.dataset.itemId != null) {
        segments.unshift("grid", Number(el.dataset.itemId));
      }
      el = el.parentElement;
    }
    if (segments.length === 0) {
      return ["pages", 0, "grid"];
    }
    return ["pages", 0, ...segments, "grid"];
  }

  private _rebuildIfDirty(): void {
    if (!this._dirty) return;
    this._dirty = false;
    this._parents.clear();
    const controllers = [...this._controllers];
    for (const child of controllers) {
      let parent: GridEditController | null = null;
      for (const candidate of controllers) {
        if (candidate === child) continue;
        if (candidate.gridEl.contains(child.gridEl)) {
          if (!parent || parent.gridEl.contains(candidate.gridEl)) {
            parent = candidate;
          }
        }
      }
      this._parents.set(child, parent);
    }
  }

  private _depth(controller: GridEditController): number {
    let d = 0;
    let c: GridEditController | null = controller;
    while (c) {
      c = this._parents.get(c) ?? null;
      if (c) d++;
    }
    return d;
  }
}

let coordinator: GridEditCoordinator | null = null;

export function getCoordinator(): GridEditCoordinator {
  if (!coordinator) coordinator = new GridEditCoordinator();
  return coordinator;
}
```

- [ ] **Step 4: Write tests for path resolution**

```typescript
// Add to grid-edit-coordinator.test.ts
describe("resolveGridPath", () => {
  it("returns root path for top-level grid", () => {
    const root = document.createElement("div");
    root.style.display = "grid";
    root.style.gridTemplateColumns = "repeat(12, 1fr)";
    document.body.appendChild(root);
    const items = makePositionedItems(1);
    const el = document.createElement("div");
    el.dataset.itemId = "0";
    root.appendChild(el);
    const ctrl = attachGridEditController(root, items, 12, 60);
    const coord = getCoordinator();
    expect(coord.resolveGridPath(ctrl)).toEqual(["pages", 0, "grid"]);
    ctrl.dispose();
  });

  it("returns nested path for grid inside item 2", () => {
    const { root, nested } = setupNestedGrids();
    // Adjust: item0 has data-item-id="0" in setupNestedGrids
    const rootItems = makePositionedItems(1);
    const nestedItems = makePositionedItems(1);
    const rootCtrl = attachGridEditController(root, rootItems, 12, 60);
    const nestedCtrl = attachGridEditController(nested, nestedItems, 6, 60);
    const coord = getCoordinator();
    expect(coord.resolveGridPath(nestedCtrl)).toEqual(["pages", 0, "grid", 0, "grid"]);
    rootCtrl.dispose();
    nestedCtrl.dispose();
  });

  it("handles three-level nesting", () => {
    const { root, nested } = setupNestedGrids();
    // Add a third level inside nested grid
    const nestedItem = nested.children[0] as HTMLElement;
    const deep = document.createElement("div");
    deep.style.display = "grid";
    deep.style.gridTemplateColumns = "repeat(3, 1fr)";
    deep.dataset.componentType = "grid";
    nestedItem.appendChild(deep);
    const deepItem = document.createElement("div");
    deepItem.dataset.itemId = "0";
    deep.appendChild(deepItem);

    const rootCtrl = attachGridEditController(root, makePositionedItems(1), 12, 60);
    const nestedCtrl = attachGridEditController(nested, makePositionedItems(1), 6, 60);
    const deepCtrl = attachGridEditController(deep, makePositionedItems(1), 3, 60);
    const coord = getCoordinator();
    expect(coord.resolveGridPath(deepCtrl)).toEqual(["pages", 0, "grid", 0, "grid", 0, "grid"]);
    rootCtrl.dispose();
    nestedCtrl.dispose();
    deepCtrl.dispose();
  });
});
```

- [ ] **Step 5: Run all coordinator tests**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose grid-edit-coordinator`
Expected: ALL PASS

- [ ] **Step 6: Wire registration into attachGridEditController**

Modify `grid-edit-controller.ts`:
- In `attachGridEditController`, after creating the controller, call `getCoordinator().register(controller)`
- In `GridEditControllerImpl.dispose()`, call `getCoordinator().deregister(this)`
- Remove the module-level `let activeController` singleton
- In `enter()`, remove the `if (activeController && activeController !== this) activeController.exit()` line
- In `enterPassive()`, remove `activeController = this`
- In `exit()`, remove `if (activeController === this) activeController = null`

- [ ] **Step 7: Export coordinator from index.ts**

Add to `packages/pages-component/src/renderer/index.ts`:
```typescript
export { getCoordinator } from "./grid-edit-coordinator.js";
export type { GridEditCoordinator } from "./grid-edit-coordinator.js";
```

- [ ] **Step 8: Run existing controller tests to verify no regressions**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose grid-edit-controller`
Expected: ALL PASS — existing single-grid behavior unchanged

- [ ] **Step 9: Commit**

```bash
git add packages/pages-component/src/renderer/grid-edit-coordinator.ts packages/pages-component/src/renderer/grid-edit-coordinator.test.ts packages/pages-component/src/renderer/grid-edit-controller.ts packages/pages-component/src/renderer/index.ts
git commit -m "feat(#639): GridEditCoordinator — tree registration, path resolution, singleton removal"
```

---

## Batch 2: Path-Routed Bridge + stopPropagation

After this batch: nested grids have independent controllers that don't interfere with each other. The bridge routes mutations to the correct YAML path. Within-grid DnD works at any nesting depth.

### Task 2: stopPropagation on nested controller pointerdown

**Files:**
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts`
- Test: `packages/pages-component/src/renderer/grid-edit-controller.test.ts`

**Interfaces:**
- Consumes: `getCoordinator()` from Task 1
- Produces: nested controllers that don't forward pointer events to parent

- [ ] **Step 1: Write failing test for event isolation**

```typescript
// Add to grid-edit-controller.test.ts
it("nested controller stopPropagation prevents parent pointerdown", () => {
  const root = document.createElement("div");
  root.style.display = "grid";
  root.style.gridTemplateColumns = "repeat(12, 1fr)";
  document.body.appendChild(root);
  const rootItem = document.createElement("div");
  rootItem.dataset.itemId = "0";
  root.appendChild(rootItem);

  const nested = document.createElement("div");
  nested.style.display = "grid";
  nested.style.gridTemplateColumns = "repeat(6, 1fr)";
  rootItem.appendChild(nested);
  const nestedItem = document.createElement("div");
  nestedItem.dataset.itemId = "0";
  nested.appendChild(nestedItem);

  const rootItems = makeItems();
  const nestedItems: GridItem[] = [
    { at: { mode: "positioned", x: 0, y: 0, w: 6, h: 1 }, component: { type: "html", id: "n0" } },
  ];
  const rootCtrl = attachGridEditController(root, rootItems, 12, 60);
  const nestedCtrl = attachGridEditController(nested, nestedItems, 6, 60);

  const parentSpy = vi.fn();
  root.addEventListener("pointerdown", parentSpy);

  nested.dispatchEvent(new PointerEvent("pointerdown", { bubbles: true, clientX: 10, clientY: 10 }));

  // pointerdown should NOT reach root because nested calls stopPropagation
  expect(parentSpy).not.toHaveBeenCalled();

  rootCtrl.dispose();
  nestedCtrl.dispose();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose -t "stopPropagation"`
Expected: FAIL — parent spy IS called (no stopPropagation yet)

- [ ] **Step 3: Add stopPropagation to nested controller's pointerdown handler**

In `GridEditControllerImpl.handlePointerDown`, add at the start:
```typescript
private handlePointerDown(e: PointerEvent): void {
  if (this.dragging || !this.geometry) return;
  if ((e.target as HTMLElement).dataset.gridEditButton !== undefined) return;
  if ((e.target as HTMLElement).closest("[data-mode-toggle]")) return;

  // Prevent parent grid controllers from seeing this event
  const coord = getCoordinator();
  if (coord.getParent(this)) {
    e.stopPropagation();
  }

  // ... rest of existing handler
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose -t "stopPropagation"`
Expected: PASS

- [ ] **Step 5: Run full controller test suite**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose grid-edit-controller`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/src/renderer/grid-edit-controller.ts packages/pages-component/src/renderer/grid-edit-controller.test.ts
git commit -m "feat(#639): stopPropagation on nested controller pointerdown"
```

### Task 3: Path-routed bridge mutations

**Files:**
- Modify: `packages/pages-builder/src/shell/grid-edit-bridge.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts` (add gridPath to dispatchEdit)
- Test: `packages/pages-builder/src/shell/grid-edit-bridge.test.ts`

**Interfaces:**
- Consumes: `getCoordinator().resolveGridPath()` from Task 1
- Produces:
  - `GridEditDetail` gains optional `gridPath: (string | number)[]`
  - `GridEditBridge.applyGridEdit` routes by path when present

- [ ] **Step 1: Write failing test for path-routed placement**

```typescript
// Add to grid-edit-bridge.test.ts
const NESTED_GRID_YAML = `pages:
  - title: Test
    grid:
      - at: { x: 0, y: 0, w: 12, h: 4 }
        type: panel
        grid:
          - at: { x: 0, y: 0, w: 3, h: 2 }
            type: chart
          - at: { x: 3, y: 0, w: 3, h: 2 }
            type: table
`;

describe("path-routed mutations", () => {
  it("routes move to nested grid when gridPath is provided", () => {
    const doc = PageDocument.parse(NESTED_GRID_YAML);
    const bridge = new GridEditBridge();

    bridge.applyGridEdit(doc, {
      type: "move",
      gridPath: ["pages", 0, "grid", 0, "grid"],
      items: [
        { at: { mode: "positioned", x: 1, y: 1, w: 3, h: 2 }, component: { type: "chart" } },
        { at: { mode: "positioned", x: 3, y: 0, w: 3, h: 2 }, component: { type: "table" } },
      ],
    });

    const yaml = doc.toString();
    const reparsed = PageDocument.parse(yaml);
    const rootItems = reparsed.getPages()[0]!.getItems();
    // Root item unchanged
    expect(rootItems[0]!.getPlacement()).toEqual({ mode: "positioned", x: 0, y: 0, w: 12, h: 4 });

    // Parse nested grid items via YAML path
    const nestedSeq = reparsed._getDoc().getIn(["pages", 0, "grid", 0, "grid"]);
    const nestedAt0 = (nestedSeq as any).get(0)?.get("at")?.toJSON();
    expect(nestedAt0).toEqual({ x: 1, y: 1, w: 3, h: 2 });
  });

  it("falls back to root when gridPath is absent", () => {
    const doc = PageDocument.parse(NESTED_GRID_YAML);
    const bridge = new GridEditBridge();

    bridge.applyGridEdit(doc, {
      type: "move",
      items: [
        { at: { mode: "positioned", x: 2, y: 1, w: 12, h: 4 }, component: { type: "panel" } },
      ],
    });

    const rootItems = doc.getPages()[0]!.getItems();
    expect(rootItems[0]!.getPlacement()).toEqual({ mode: "positioned", x: 2, y: 1, w: 12, h: 4 });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/pages-builder test -- --reporter=verbose -t "path-routed"`
Expected: FAIL — `gridPath` property not recognized

- [ ] **Step 3: Add gridPath to GridEditDetail type**

In `grid-edit-bridge.ts`, update the type:
```typescript
export type GridEditDetail =
  | { type: 'move'; items: GridItem[]; gridPath?: (string | number)[] }
  | { type: 'resize'; items: GridItem[]; gridPath?: (string | number)[] }
  | { type: 'insert'; placement: GridPlacement; componentType: string; gridPath?: (string | number)[] }
  | { type: 'split'; targetIndex: number; direction: 'horizontal' | 'vertical'; gridPath?: (string | number)[] }
  | { type: 'merge'; indices: number[]; gridPath?: (string | number)[] }
  | { type: 'swap'; indexA: number; indexB: number; gridPath?: (string | number)[] }
  | { type: 'remove'; index: number; gridPath?: (string | number)[] }
  | { type: 'add-empty'; placement: GridPlacement; gridPath?: (string | number)[] };
```

- [ ] **Step 4: Implement path-routed _applyPlacements**

Refactor `GridEditBridge._applyPlacements` to accept an optional gridPath. When present, navigate the YAML document directly via `doc._getDoc().getIn(gridPath)` instead of `page.getItems()`. Extract a helper `_getGridItems(doc, gridPath?)` that returns the items from either path.

All bridge methods (`_splitItem`, `_mergeItems`, `_swapItems`, `_removeItem`, `_addEmptyContainer`, `_insertItem`) need the same treatment — extract the grid path routing into the top-level `applyGridEdit` method and pass the resolved items array to each handler.

- [ ] **Step 5: Add gridPath to controller's dispatchEdit**

In `grid-edit-controller.ts`, modify `dispatchEdit` to include the coordinator's resolved path:
```typescript
private dispatchEdit(type: string, items: IdRect[]): void {
  // ... existing debounce and sorting ...
  const coord = getCoordinator();
  const gridPath = coord.resolveGridPath(this);
  this.gridEl.dispatchEvent(new CustomEvent("pages-grid-edit", {
    bubbles: true,
    composed: true,
    detail: { type, items: resolved, gridPath },
  }));
}
```

- [ ] **Step 6: Run path-routed tests**

Run: `yarn --cwd packages/pages-builder test -- --reporter=verbose -t "path-routed"`
Expected: ALL PASS

- [ ] **Step 7: Run full bridge test suite**

Run: `yarn --cwd packages/pages-builder test -- --reporter=verbose grid-edit-bridge`
Expected: ALL PASS — existing root-level tests still pass (backward compat)

- [ ] **Step 8: Run full controller test suite**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose grid-edit-controller`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-builder/src/shell/grid-edit-bridge.ts packages/pages-builder/src/shell/grid-edit-bridge.test.ts packages/pages-component/src/renderer/grid-edit-controller.ts
git commit -m "feat(#639): path-routed bridge — nested grid edits target correct YAML path"
```

---

## Batch 3: Cross-Grid Drag

After this batch: users can drag items between grids at any nesting depth. Proportional scaling, atomic undo, and the full cross-grid drag lifecycle work end-to-end.

### Task 4: Controller release/accept API + proportional scaling

**Files:**
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts`
- Create: `packages/pages-component/src/renderer/grid-scale.ts`
- Test: `packages/pages-component/src/renderer/grid-edit-controller.test.ts`
- Test: `packages/pages-component/src/renderer/grid-scale.test.ts`

**Interfaces:**
- Consumes: `GridItem`, `GridPlacement` from `types.ts`
- Produces:
  - `GridEditController.releaseItem(index: number): void`
  - `GridEditController.acceptForeignItem(item: GridItem, placement: GridPlacement): void`
  - `GridEditController.clearForeignItem(): void`
  - `scaleWidth(sourceWidth: number, sourceColumns: number, targetColumns: number): number`

- [ ] **Step 1: Write failing tests for proportional scaling**

```typescript
// grid-scale.test.ts
import { describe, it, expect } from "vitest";
import { scaleWidth } from "./grid-scale.js";

describe("scaleWidth", () => {
  it("scales proportionally: 6 in 12-col → 3 in 6-col", () => {
    expect(scaleWidth(6, 12, 6)).toBe(3);
  });

  it("rounds: 5 in 12-col → 3 in 6-col (2.5 rounds to 3)", () => {
    expect(scaleWidth(5, 12, 6)).toBe(3);
  });

  it("scales up: 1 in 6-col → 2 in 12-col", () => {
    expect(scaleWidth(1, 6, 12)).toBe(2);
  });

  it("clamps minimum to 1: 1 in 12-col → 1 in 6-col (0.5 clamps to 1)", () => {
    expect(scaleWidth(1, 12, 6)).toBe(1);
  });

  it("clamps maximum to targetColumns", () => {
    expect(scaleWidth(12, 12, 6)).toBe(6);
  });

  it("same column count returns same width", () => {
    expect(scaleWidth(4, 12, 12)).toBe(4);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose grid-scale`
Expected: FAIL — module not found

- [ ] **Step 3: Implement scaleWidth**

```typescript
// grid-scale.ts
export function scaleWidth(sourceWidth: number, sourceColumns: number, targetColumns: number): number {
  const scaled = Math.round(sourceWidth * targetColumns / sourceColumns);
  return Math.min(Math.max(1, scaled), targetColumns);
}
```

- [ ] **Step 4: Run scaling tests**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose grid-scale`
Expected: ALL PASS

- [ ] **Step 5: Write failing tests for releaseItem / acceptForeignItem / clearForeignItem**

```typescript
// Add to grid-edit-controller.test.ts
describe("cross-grid API", () => {
  it("releaseItem hides the element", () => {
    const items = makeItems();
    const grid = setupGrid(items);
    const ctrl = attachGridEditController(grid, items, 12, 60);
    ctrl.releaseItem(0);
    const el = grid.querySelector('[data-item-id="0"]') as HTMLElement;
    expect(el.style.display).toBe("none");
    ctrl.dispose();
  });

  it("acceptForeignItem shows a placeholder", () => {
    const items = makeItems();
    const grid = setupGrid(items);
    const ctrl = attachGridEditController(grid, items, 12, 60);
    const foreign: GridItem = {
      at: { mode: "positioned", x: 0, y: 4, w: 6, h: 1 },
      component: { type: "html", id: "foreign" },
    };
    ctrl.acceptForeignItem(foreign, { mode: "positioned", x: 0, y: 4, w: 6, h: 1 });
    const placeholder = grid.querySelector(".pages-grid-placeholder");
    expect(placeholder).not.toBeNull();
    ctrl.dispose();
  });

  it("clearForeignItem removes the placeholder", () => {
    const items = makeItems();
    const grid = setupGrid(items);
    const ctrl = attachGridEditController(grid, items, 12, 60);
    const foreign: GridItem = {
      at: { mode: "positioned", x: 0, y: 4, w: 6, h: 1 },
      component: { type: "html", id: "foreign" },
    };
    ctrl.acceptForeignItem(foreign, { mode: "positioned", x: 0, y: 4, w: 6, h: 1 });
    ctrl.clearForeignItem();
    const placeholder = grid.querySelector(".pages-grid-placeholder");
    expect(placeholder).toBeNull();
    ctrl.dispose();
  });
});
```

- [ ] **Step 6: Run to verify failure**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose -t "cross-grid API"`
Expected: FAIL — methods not defined

- [ ] **Step 7: Implement releaseItem, acceptForeignItem, clearForeignItem**

Add to `GridEditController` interface:
```typescript
export interface GridEditController {
  readonly gridEl: HTMLElement;
  readonly items: readonly GridItem[];
  readonly columns: number;
  readonly active: boolean;
  readonly mode: EditorMode;
  enter(): void;
  exit(): void;
  dispose(): void;
  releaseItem(index: number): void;
  acceptForeignItem(item: GridItem, placement: GridPlacement): void;
  clearForeignItem(): void;
}
```

Implement in `GridEditControllerImpl`:
```typescript
releaseItem(index: number): void {
  const el = this.gridEl.querySelector(`[data-item-id="${index}"]`) as HTMLElement | null;
  if (el) el.style.display = "none";
}

acceptForeignItem(_item: GridItem, placement: GridPlacement): void {
  this.clearForeignItem();
  showPlaceholder(this.gridEl, placement);
}

clearForeignItem(): void {
  hidePlaceholder(this.gridEl);
}
```

- [ ] **Step 8: Run cross-grid API tests**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose -t "cross-grid API"`
Expected: ALL PASS

- [ ] **Step 9: Export scaleWidth from index.ts**

Add to `packages/pages-component/src/renderer/index.ts`:
```typescript
export { scaleWidth } from "./grid-scale.js";
```

- [ ] **Step 10: Commit**

```bash
git add packages/pages-component/src/renderer/grid-edit-controller.ts packages/pages-component/src/renderer/grid-edit-controller.test.ts packages/pages-component/src/renderer/grid-scale.ts packages/pages-component/src/renderer/grid-scale.test.ts packages/pages-component/src/renderer/index.ts
git commit -m "feat(#639): controller release/accept API + proportional scaling"
```

### Task 5: Coordinator cross-grid drag lifecycle

**Files:**
- Modify: `packages/pages-component/src/renderer/grid-edit-coordinator.ts`
- Modify: `packages/pages-builder/src/shell/grid-edit-bridge.ts`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts`
- Test: `packages/pages-component/src/renderer/grid-edit-coordinator.test.ts`
- Test: `packages/pages-builder/src/shell/grid-edit-bridge.test.ts`

**Interfaces:**
- Consumes: `releaseItem`, `acceptForeignItem`, `clearForeignItem` from Task 4; `scaleWidth` from Task 4; `resolveGridPath` from Task 1
- Produces:
  - `GridEditCoordinator.startCrossGridDrag(source: GridEditController, itemIndex: number, item: GridItem): void`
  - `GridEditCoordinator.handleCrossGridMove(clientX: number, clientY: number): void`
  - `GridEditCoordinator.completeCrossGridDrag(clientX: number, clientY: number): CrossGridEditDetail | null`
  - `GridEditCoordinator.cancelCrossGridDrag(): void`
  - `CrossGridEditDetail` type on bridge

- [ ] **Step 1: Write failing tests for cross-grid drag lifecycle on coordinator**

```typescript
// Add to grid-edit-coordinator.test.ts
import type { GridItem } from "../model/types.js";

describe("cross-grid drag", () => {
  function setupTwoGrids() {
    const root = document.createElement("div");
    root.style.display = "grid";
    root.style.gridTemplateColumns = "repeat(12, 1fr)";
    document.body.appendChild(root);
    // Mock getBoundingClientRect for root
    root.getBoundingClientRect = () => ({ left: 0, top: 0, right: 1200, bottom: 400, width: 1200, height: 400, x: 0, y: 0, toJSON: () => ({}) });

    const item0 = document.createElement("div");
    item0.dataset.itemId = "0";
    root.appendChild(item0);

    const nested = document.createElement("div");
    nested.style.display = "grid";
    nested.style.gridTemplateColumns = "repeat(6, 1fr)";
    item0.appendChild(nested);
    nested.getBoundingClientRect = () => ({ left: 0, top: 0, right: 600, bottom: 200, width: 600, height: 200, x: 0, y: 0, toJSON: () => ({}) });

    const nestedItem = document.createElement("div");
    nestedItem.dataset.itemId = "0";
    nested.appendChild(nestedItem);

    const rootItems: GridItem[] = [
      { at: { mode: "positioned", x: 0, y: 0, w: 12, h: 4 }, component: { type: "panel", grid: [] } },
    ];
    const nestedItems: GridItem[] = [
      { at: { mode: "positioned", x: 0, y: 0, w: 3, h: 2 }, component: { type: "chart", id: "c1" } },
    ];

    const rootCtrl = attachGridEditController(root, rootItems, 12, 60);
    const nestedCtrl = attachGridEditController(nested, nestedItems, 6, 60);
    return { root, nested, rootCtrl, nestedCtrl, rootItems, nestedItems };
  }

  it("completeCrossGridDrag returns detail with source and target paths", () => {
    const { rootCtrl, nestedCtrl, nestedItems } = setupTwoGrids();
    const coord = getCoordinator();
    coord.startCrossGridDrag(nestedCtrl, 0, nestedItems[0]!);
    // Simulate pointer at root grid area (outside nested bounds)
    const detail = coord.completeCrossGridDrag(800, 300);
    expect(detail).not.toBeNull();
    expect(detail!.sourceGridPath).toEqual(coord.resolveGridPath(nestedCtrl));
    expect(detail!.targetGridPath).toEqual(coord.resolveGridPath(rootCtrl));
    expect(detail!.sourceIndex).toBe(0);
    rootCtrl.dispose();
    nestedCtrl.dispose();
  });

  it("cancelCrossGridDrag restores source item visibility", () => {
    const { nested, nestedCtrl, nestedItems } = setupTwoGrids();
    const coord = getCoordinator();
    coord.startCrossGridDrag(nestedCtrl, 0, nestedItems[0]!);
    nestedCtrl.releaseItem(0);
    coord.cancelCrossGridDrag();
    const el = nested.querySelector('[data-item-id="0"]') as HTMLElement;
    expect(el.style.display).not.toBe("none");
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose -t "cross-grid drag"`
Expected: FAIL — methods not defined

- [ ] **Step 3: Implement cross-grid drag lifecycle on coordinator**

Add to `GridEditCoordinator`:
```typescript
private _crossGridState: {
  source: GridEditController;
  itemIndex: number;
  item: GridItem;
  currentTarget: GridEditController | null;
} | null = null;

startCrossGridDrag(source: GridEditController, itemIndex: number, item: GridItem): void {
  this._crossGridState = { source, itemIndex, item, currentTarget: null };
  source.releaseItem(itemIndex);
}

handleCrossGridMove(clientX: number, clientY: number): void {
  if (!this._crossGridState) return;
  const target = this.deepestGridAt(clientX, clientY);
  if (target === this._crossGridState.source) return;

  if (target !== this._crossGridState.currentTarget) {
    this._crossGridState.currentTarget?.clearForeignItem();
    this._crossGridState.currentTarget = target;
  }

  if (target && target !== this._crossGridState.source) {
    const sourceItem = this._crossGridState.item;
    const sourceAt = sourceItem.at;
    if (sourceAt.mode !== "positioned") return;
    const scaledW = scaleWidth(sourceAt.w, this._crossGridState.source.columns, target.columns);
    const geometry = computeGridGeometry(
      target.gridEl.getBoundingClientRect(), target.columns, 60, 12
    );
    const cell = snapToCell(clientX, clientY, geometry);
    const clamped = clampPlacement(cell.x, cell.y, scaledW, sourceAt.h, target.columns);
    target.acceptForeignItem(sourceItem, {
      mode: "positioned", x: clamped.x, y: clamped.y, w: scaledW, h: sourceAt.h,
    });
  }
}

completeCrossGridDrag(clientX: number, clientY: number): CrossGridEditDetail | null {
  if (!this._crossGridState) return null;
  const target = this.deepestGridAt(clientX, clientY);
  if (!target || target === this._crossGridState.source) {
    this.cancelCrossGridDrag();
    return null;
  }
  const { source, itemIndex, item } = this._crossGridState;
  const sourceAt = item.at;
  if (sourceAt.mode !== "positioned") { this.cancelCrossGridDrag(); return null; }
  const scaledW = scaleWidth(sourceAt.w, source.columns, target.columns);
  const geometry = computeGridGeometry(
    target.gridEl.getBoundingClientRect(), target.columns, 60, 12
  );
  const cell = snapToCell(clientX, clientY, geometry);
  const clamped = clampPlacement(cell.x, cell.y, scaledW, sourceAt.h, target.columns);
  const detail: CrossGridEditDetail = {
    type: "cross-move",
    sourceGridPath: this.resolveGridPath(source),
    targetGridPath: this.resolveGridPath(target),
    sourceIndex: itemIndex,
    targetPlacement: { mode: "positioned", x: clamped.x, y: clamped.y, w: scaledW, h: sourceAt.h },
    component: item.component!,
  };
  target.clearForeignItem();
  this._crossGridState = null;
  return detail;
}

cancelCrossGridDrag(): void {
  if (!this._crossGridState) return;
  this._crossGridState.currentTarget?.clearForeignItem();
  // Restore source item visibility
  const el = this._crossGridState.source.gridEl.querySelector(
    `[data-item-id="${this._crossGridState.itemIndex}"]`
  ) as HTMLElement | null;
  if (el) { el.style.display = ""; }
  this._crossGridState = null;
}
```

Add the `CrossGridEditDetail` type and required imports.

- [ ] **Step 4: Run cross-grid coordinator tests**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose -t "cross-grid drag"`
Expected: ALL PASS

- [ ] **Step 5: Write failing test for cross-grid bridge mutation**

```typescript
// Add to grid-edit-bridge.test.ts
describe("cross-grid mutation", () => {
  it("removes item from source and inserts into target atomically", () => {
    const doc = PageDocument.parse(NESTED_GRID_YAML);
    const bridge = new GridEditBridge();

    bridge.applyCrossGridEdit(doc, {
      type: "cross-move",
      sourceGridPath: ["pages", 0, "grid", 0, "grid"],
      targetGridPath: ["pages", 0, "grid"],
      sourceIndex: 0,
      targetPlacement: { mode: "positioned", x: 0, y: 4, w: 6, h: 2 },
      component: { type: "chart" },
    });

    // Source nested grid should have 1 item (was 2, removed index 0)
    const nestedSeq = doc._getDoc().getIn(["pages", 0, "grid", 0, "grid"]);
    expect((nestedSeq as any).items.length).toBe(1);

    // Target root grid should have 2 items (was 1, added chart)
    const rootSeq = doc._getDoc().getIn(["pages", 0, "grid"]);
    expect((rootSeq as any).items.length).toBe(2);
  });
});
```

- [ ] **Step 6: Run to verify failure**

Run: `yarn --cwd packages/pages-builder test -- --reporter=verbose -t "cross-grid mutation"`
Expected: FAIL — `applyCrossGridEdit` not defined

- [ ] **Step 7: Implement applyCrossGridEdit on bridge**

Add to `GridEditBridge`:
```typescript
applyCrossGridEdit(doc: PageDocument, detail: CrossGridEditDetail): void {
  doc._pushUndoInternal();
  const yamlDoc = doc._getDoc();

  // 1. Read source item before removing
  const sourceSeq = yamlDoc.getIn(detail.sourceGridPath) as YAMLSeq;
  const sourceNode = sourceSeq.get(detail.sourceIndex);

  // 2. Remove from source
  sourceSeq.delete(detail.sourceIndex);

  // 3. Build target entry with scaled placement
  const atObj: Record<string, number> = {
    x: detail.targetPlacement.x,
    y: detail.targetPlacement.y,
    w: detail.targetPlacement.w,
    h: detail.targetPlacement.h,
  };
  const entry: Record<string, unknown> = { at: atObj };
  if (detail.component) {
    entry["type"] = detail.component.type;
    if (detail.component.props) entry["properties"] = detail.component.props;
  }
  const targetNode = yamlDoc.createNode(entry);

  // 4. Insert into target
  const targetSeq = yamlDoc.getIn(detail.targetGridPath) as YAMLSeq;
  targetSeq.add(targetNode);

  doc._notifyInternal();
}
```

Add `CrossGridEditDetail` type import and the `pages-grid-cross-edit` event listener in `builder-shell.ts`.

- [ ] **Step 8: Wire cross-grid event in builder-shell**

In `builder-shell.ts`, add alongside the existing `pages-grid-edit` listener:
```typescript
this.addEventListener('pages-grid-cross-edit', this._crossGridEditHandler);
```

With handler:
```typescript
private _crossGridEditHandler = (e: Event) => {
  const detail = (e as CustomEvent).detail as CrossGridEditDetail;
  this._applyEdit('grid', () => this._gridBridge.applyCrossGridEdit(this._document, detail));
};
```

- [ ] **Step 9: Run cross-grid bridge tests**

Run: `yarn --cwd packages/pages-builder test -- --reporter=verbose -t "cross-grid mutation"`
Expected: ALL PASS

- [ ] **Step 10: Run full test suites**

Run: `yarn --cwd packages/pages-component test -- --reporter=verbose`
Run: `yarn --cwd packages/pages-builder test -- --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add packages/pages-component/src/renderer/grid-edit-coordinator.ts packages/pages-component/src/renderer/grid-edit-coordinator.test.ts packages/pages-builder/src/shell/grid-edit-bridge.ts packages/pages-builder/src/shell/grid-edit-bridge.test.ts packages/pages-builder/src/shell/builder-shell.ts
git commit -m "feat(#639): cross-grid drag lifecycle — coordinator handoff + atomic bridge mutations"
```

---

## References

- [2026-11-10-nested-grid-dnd-cross-grid-drag-design.md] — design spec this plan implements
- [decisions.md] — D1-D5 architectural decisions
- grid-edit-controller.ts — existing controller (877 lines)
- grid-edit-bridge.ts — bridge routing
- grid-edit-controller.test.ts — existing test patterns (Vitest, JSDOM, setupGrid helper)
- grid-edit-bridge.test.ts — existing bridge test patterns (PageDocument.parse, YAML fixtures)
- activation.ts:948-951 — controller attachment callsite
- render.ts:104-152 — recursive grid rendering, data-item-id assignment
- grid-compaction.ts:141 — isNestedGrid() utility
- grid-drag.ts — computeGridGeometry, snapToCell, clampPlacement
- grid-overlay.ts — showPlaceholder, hidePlaceholder
- page-document.ts:372-381 — path-based getItems()
- GitHub #639 — focal issue
