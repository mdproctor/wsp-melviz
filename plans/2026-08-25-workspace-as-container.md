# Workspace as Container Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #369 — Workspace centre as root Container
**Issue group:** #367, #368, #369

**Goal:** Replace the triple-state architecture (engine + backend + strategy) with a single root Container using `layout: "free"`. DnD, edge splits, and frame lifecycle move into the free-layout strategy. Mode transitions become `setLayout()`.

**Architecture:** Root Container with free-layout strategy owns all frame interaction. `layout-math.ts` provides pure zone-preset functions. `wireFloatingWorkspace` becomes an ~80-line initializer. Engine and backend DnD code are deleted.

**Tech Stack:** TypeScript 5, Vitest (JSDOM), Yarn workspaces, pages-runtime package

## Global Constraints

- All existing 1003 tests must pass at every batch boundary
- `ContainerState` is the sole persistence format — `FrameLayout[]` migrated at load time
- No new package dependencies
- `detachEntry` does NOT call `contentDispose` — content survives for transfer
- `pages-tab-drag-start` and `pages-tab-escaped` events use `stopPropagation()` for scoping
- Test command: `yarn workspace @casehubio/pages-runtime run test`
- Use ide-tooling for all code navigation and structural editing

---

## Batch 1: Foundation — layout-math + interface additions

After this batch: pure layout math extracted, `detachEntry`/`hideEntry`/`showEntry` on interfaces, all existing tests pass.

### Task 1: Extract `layout-math.ts` — pure zone preset functions

**Files:**
- Create: `packages/pages-runtime/src/layout-math.ts`
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-strategy.ts` — import and use `computeZonePreset`
- Test: `packages/pages-runtime/src/layout-math.test.ts` (new)

**Interfaces:**
- Produces: `computeZonePreset(preset, entryCount, canvasSize) → Array<{x,y,width,height}>`, `scaleProportionally(entries, oldSize, newSize) → entries`, `equalGrid(count, canvasSize) → Array<{x,y,width,height}>`

- [ ] **Step 1: Write failing tests for zone presets**

In `layout-math.test.ts`:
```typescript
import { describe, it, expect } from "vitest";
import { computeZonePreset, scaleProportionally } from "./layout-math.js";

describe("layout-math", () => {
  describe("computeZonePreset", () => {
    it("grid preset tiles entries evenly", () => {
      const result = computeZonePreset("grid", 4, { width: 800, height: 600 });
      expect(result).toHaveLength(4);
      expect(result[0]!.x).toBe(0);
      expect(result[0]!.width).toBeCloseTo(400, -1);
      expect(result[2]!.y).toBeGreaterThan(0);
    });

    it("side-by-side splits horizontally", () => {
      const result = computeZonePreset("side-by-side", 2, { width: 800, height: 600 });
      expect(result).toHaveLength(2);
      expect(result[0]!.width).toBeCloseTo(400, -1);
      expect(result[1]!.x).toBeGreaterThan(0);
    });

    it("stacked splits vertically", () => {
      const result = computeZonePreset("stacked", 2, { width: 800, height: 600 });
      expect(result).toHaveLength(2);
      expect(result[0]!.height).toBeCloseTo(300, -1);
      expect(result[1]!.y).toBeGreaterThan(0);
    });

    it("focus gives first entry 70% width", () => {
      const result = computeZonePreset("focus", 3, { width: 800, height: 600 });
      expect(result[0]!.width).toBeGreaterThan(500);
    });

    it("main-sidebar gives first entry 65% width", () => {
      const result = computeZonePreset("main-sidebar", 2, { width: 800, height: 600 });
      expect(result[0]!.width).toBeGreaterThan(result[1]!.width);
    });
  });

  describe("scaleProportionally", () => {
    it("scales entry positions and sizes to new canvas", () => {
      const entries = [{ x: 100, y: 50, width: 200, height: 150 }];
      const result = scaleProportionally(entries, { width: 800, height: 600 }, { width: 400, height: 300 });
      expect(result[0]!.x).toBe(50);
      expect(result[0]!.width).toBe(100);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify failure**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose layout-math.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement `layout-math.ts`**

Extract zone preset logic from `frame-organisers.ts` (`applyPreset` function) and resize proportional logic from both `group-organiser-backend.ts` and `free-layout-strategy.ts`. All pure functions operating on `{x, y, width, height}` records.

- [ ] **Step 4: Update free-layout strategy `arrange()` to use `computeZonePreset`**

Replace the inline `tileArrange()` with a call to `computeZonePreset(preset, count, canvasSize)`, applying the resulting positions/sizes to all entries.

- [ ] **Step 5: Run tests, verify pass**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 6: Commit**

```
feat: layout-math.ts — pure zone preset and scaling functions Refs #369
```

### Task 2: Add `detachEntry` to LayoutStrategy and Container

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/types.ts` — add `detachEntry` to `LayoutStrategy` and `Container`
- Modify: `packages/pages-runtime/src/frame-sandbox/container.ts` — implement `detachEntry`
- Modify: `packages/pages-runtime/src/frame-sandbox/tabbed-strategy.ts` — implement `detachEntry`
- Modify: `packages/pages-runtime/src/frame-sandbox/accordion-strategy.ts` — implement `detachEntry`
- Modify: `packages/pages-runtime/src/frame-sandbox/split-strategy.ts` — implement `detachEntry`
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-strategy.ts` — implement `detachEntry`
- Test: `packages/pages-runtime/src/frame-sandbox/container.test.ts`

**Interfaces:**
- Produces: `LayoutStrategy.detachEntry(key: string): Entry | null`
- Produces: `Container.detachEntry(key: string): Entry | null`

- [ ] **Step 1: Write failing tests**

```typescript
describe("detachEntry", () => {
  it("removes entry from container without disposing content", () => {
    const factory = testFactory();
    const entry: Entry = { key: "a", label: "A" };
    entry.component = { type: "html", props: {} };
    const parent = createContainer({
      entries: [entry, { key: "b", label: "B" }],
      layout: "tabbed",
      contentFactory: factory,
    });
    parent.mount(container);

    const detached = parent.detachEntry("a");
    expect(detached).not.toBeNull();
    expect(detached!.key).toBe("a");
    expect(detached!.component).toEqual({ type: "html", props: {} });
    expect(parent.entries).toHaveLength(1);
  });

  it("does not fire onCollapse", () => {
    const onCollapse = vi.fn();
    const parent = createContainer({
      entries: [{ key: "a", label: "A" }, { key: "b", label: "B" }],
      layout: "tabbed",
      contentFactory: testFactory(),
      onCollapse,
    });
    parent.mount(container);

    parent.detachEntry("a");
    expect(onCollapse).not.toHaveBeenCalled();
  });

  it("preserves childContainer without disposing", () => {
    const inner = createContainer({
      entries: [{ key: "leaf", label: "Leaf" }],
      layout: "tabbed",
      contentFactory: testFactory(),
      depth: 2,
    });
    const entry: Entry = { key: "host", label: "Host", childContainer: inner };
    const parent = createContainer({
      entries: [entry, { key: "b", label: "B" }],
      layout: "tabbed",
      contentFactory: testFactory(),
    });
    parent.mount(container);

    const detached = parent.detachEntry("host");
    expect(detached!.childContainer).toBe(inner);
  });
});
```

- [ ] **Step 2: Run tests to verify failure**

- [ ] **Step 3: Add `detachEntry` to `LayoutStrategy` interface in types.ts**

```typescript
detachEntry(key: string): Entry | null;
```

- [ ] **Step 4: Implement in all four strategies**

Each strategy: find entry, remove from DOM and tracking arrays, do NOT call contentDispose or fire callbacks, return the entry.

- [ ] **Step 5: Add `detachEntry` to Container in container.ts**

```typescript
detachEntry(key) {
  const idx = entries.findIndex(e => e.key === key);
  if (idx === -1) return null;
  const entry = entries[idx]!;
  currentOrganiser.detachEntry(key);
  entries.splice(idx, 1);
  return entry;
},
```

- [ ] **Step 6: Run tests, commit**

```
feat: detachEntry — remove entry without disposing content Refs #369
```

### Task 3: Add `hideEntry`/`showEntry` optional methods to LayoutStrategy

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/types.ts` — add optional methods
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-strategy.ts` — implement
- Test: `packages/pages-runtime/src/frame-sandbox/free-layout-strategy.test.ts`

**Interfaces:**
- Produces: `LayoutStrategy.hideEntry?(key: string): void`
- Produces: `LayoutStrategy.showEntry?(key: string): void`

- [ ] **Step 1: Write failing test**

```typescript
it("hideEntry removes frame from DOM but preserves state", () => {
  // create free-layout strategy with entries
  // call hideEntry
  // verify frame element removed from DOM
  // verify entryState preserved
  // call showEntry
  // verify frame element re-rendered with same position/size
});
```

- [ ] **Step 2: Add to interface, implement in free-layout**

- [ ] **Step 3: Run tests, commit**

```
feat: hideEntry/showEntry on LayoutStrategy for detach support Refs #369
```

---

## Batch 2: DnD — event protocol + cross-entry interaction

After this batch: tabs can be dragged between entries in a free-layout container, edge splits work, depth escape fires correctly.

### Task 4: Tab drag event protocol — `pages-tab-drag-start` firing

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/tabbed-strategy.ts` — fire DOM event on tab drag start
- Test: `packages/pages-runtime/src/frame-sandbox/tabbed-strategy.test.ts`

**Interfaces:**
- Produces: `pages-tab-drag-start` CustomEvent with `{ tabKey, ghost, sourceContainer }` detail

- [ ] **Step 1: Write test verifying event fires and bubbles**

```typescript
it("fires pages-tab-drag-start on tab drag", () => {
  const host = document.createElement("div");
  document.body.appendChild(host);
  let received: CustomEvent | null = null;
  host.addEventListener("pages-tab-drag-start", (e) => { received = e as CustomEvent; });
  // mount tabbed strategy, simulate pointerdown on tab
  // verify event received with correct detail
});
```

- [ ] **Step 2: Implement in tabbed strategy**

In the existing `onTabDragStart` callback path, after creating the ghost element, fire:
```typescript
hostElement?.dispatchEvent(new CustomEvent("pages-tab-drag-start", {
  bubbles: true, detail: { tabKey, ghost, sourceContainer: currentContainer },
}));
```

- [ ] **Step 3: Run tests, commit**

```
feat: pages-tab-drag-start event protocol Refs #369
```

### Task 5: Free-layout DnD listener — cross-entry tab drop

**Files:**
- Create: `packages/pages-runtime/src/frame-sandbox/free-layout-dnd.ts` — DnD state machine
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-strategy.ts` — wire DnD handler
- Test: `packages/pages-runtime/src/frame-sandbox/free-layout-dnd.test.ts` (new)

**Interfaces:**
- Consumes: `pages-tab-drag-start` event, `entryState`, `frameElements`
- Produces: cross-entry tab transfer (detachEntry from source, addEntry to target)

- [ ] **Step 1: Write tests for DnD target detection**

Test that the DnD handler:
- Listens for `pages-tab-drag-start` on the host
- Detects which entry the pointer is over based on `entryState` positions
- Calls `detachEntry` on source, `addEntry` on target

- [ ] **Step 2: Create `free-layout-dnd.ts`**

Extracted DnD state machine (~200 lines):
- `createFreeLayoutDnd(hostElement, entryState, frameElements, callbacks)`
- Returns `{ dispose(): void }`
- Handles: pointermove target detection, cross-entry drop, new entry creation, ghost positioning

- [ ] **Step 3: Wire into free-layout strategy**

In `mount()`, create the DnD handler. In `unmount()`/`dispose()`, dispose it.

- [ ] **Step 4: Run tests, commit**

```
feat: cross-entry DnD in free-layout strategy Refs #369
```

### Task 6: Edge splits + depth escape

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-dnd.ts` — add edge detection + split
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-dnd.ts` — add depth escape (`pages-tab-escaped`)
- Test: `packages/pages-runtime/src/frame-sandbox/free-layout-dnd.test.ts`

**Interfaces:**
- Consumes: `refreshEntry` for surgical split, `childContainerFactory` for new containers
- Produces: `pages-tab-escaped` CustomEvent for depth escape

- [ ] **Step 1: Write tests for edge split**

Test that dragging to a frame edge creates a split container via `refreshEntry`.

- [ ] **Step 2: Write tests for depth escape**

Test that dragging outside the host fires `pages-tab-escaped` with the detached entry.

- [ ] **Step 3: Implement edge detection + split creation**

Reuse edge zone detection logic from `frame-boundaries.ts`. When a drop lands on an edge zone, call `detachEntry` on source, create a split container with the existing child and new leaf, then `refreshEntry` on the parent.

- [ ] **Step 4: Implement depth escape**

When pointer exits host bounds, call `detachEntry`, fire `pages-tab-escaped`, exit drag mode. Parent free-layout strategy listens for `pages-tab-escaped` and enters drag mode for the escaped entry.

- [ ] **Step 5: Run tests, commit**

```
feat: edge splits + depth escape in free-layout DnD Refs #369
```

---

## Batch 3: Workspace Transition — wire + persistence + activation

After this batch: wireFloatingWorkspace uses root Container, persistence migrated, activation.ts updated. Engine and backend DnD code deleted.

### Task 7: Persistence migration function

**Files:**
- Create: `packages/pages-runtime/src/layout-migration.ts`
- Test: `packages/pages-runtime/src/layout-migration.test.ts` (new)

**Interfaces:**
- Produces: `migrateFrameLayout(frames: FrameLayout[]): ContainerState`

- [ ] **Step 1: Write round-trip tests**

```typescript
it("converts FrameLayout[] to ContainerState with free layout", () => {
  const frames: FrameLayout[] = [
    { key: "f1", tabs: [{ key: "t1", label: "T1", content: { type: "html", props: {} } }],
      position: { x: 50, y: 50 }, size: { width: 300, height: 200 },
      order: 0, zIndex: 1, pinned: false, hidden: false, activeTabKey: "t1" },
  ];
  const state = migrateFrameLayout(frames);
  expect(state.layout).toBe("free");
  expect(state.tabs).toHaveLength(1);
  expect(state.tabs[0]!.key).toBe("f1");
  expect(state.tabs[0]!.children!.layout).toBe("tabbed");
  expect(state.layoutState.entries.f1.position).toEqual({ x: 50, y: 50 });
});
```

- [ ] **Step 2: Implement migration function**

- [ ] **Step 3: Run tests, commit**

```
feat: migrateFrameLayout — FrameLayout[] to ContainerState Refs #369
```

### Task 8: Rewrite wireFloatingWorkspace as thin initializer

**Files:**
- Modify: `packages/pages-runtime/src/wire-floating-workspace.ts` — rewrite to ~80 lines
- Modify: `packages/pages-runtime/src/wire-floating-workspace.test.ts` — update tests
- Delete: `packages/pages-runtime/src/floating-frame-engine.ts`
- Delete: `packages/pages-runtime/src/floating-frame-engine.test.ts`
- Delete: `packages/pages-runtime/src/frame-organisers.ts`

**Interfaces:**
- Consumes: `createContainer`, `migrateFrameLayout`, `restoreContainerFromState`
- Produces: `WireHandle { rootContainer, dispose }`

- [ ] **Step 1: Write test for new wireFloatingWorkspace**

Test that it creates a root Container with layout "free" from a ContainerState.

- [ ] **Step 2: Rewrite wireFloatingWorkspace**

~80 lines: migrate saved layout, create root Container, mount, wire external events, wire detach handler, return handle.

- [ ] **Step 3: Delete engine, frame-organisers**

Remove `floating-frame-engine.ts`, `floating-frame-engine.test.ts`, `frame-organisers.ts`.

- [ ] **Step 4: Run full test suite**

Fix any broken imports or test references.

- [ ] **Step 5: Commit**

```
refactor: wireFloatingWorkspace as thin initializer — delete engine Refs #369
```

### Task 9: Migrate activation.ts + detach handler + delete backend DnD

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts` — use Container API
- Modify: `packages/pages-runtime/src/frame-detach-handler.ts` — use `rootContainer.organiser.hideEntry/showEntry`
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts` — delete DnD code (~220 lines)
- Delete: `packages/pages-runtime/src/workspace-content-lifecycle.ts` (if exists)
- Test: `packages/pages-runtime/src/activation.test.ts`

**Interfaces:**
- Consumes: `rootContainer.entries`, `findContainerWithTab`, `hideEntry`, `showEntry`

- [ ] **Step 1: Migrate activation.ts**

Replace `handle.engine` references with `handle.rootContainer` operations per the spec's migration table.

- [ ] **Step 2: Migrate detach handler**

Replace engine methods with Container/strategy equivalents per the spec.

- [ ] **Step 3: Delete backend DnD code**

Remove `handleCrossFrameDragMove`, `handleCrossFrameDragEnd`, `showEdgeSplitOverlay`, `edgeSplitPreview` state, and all related event wiring from `group-organiser-backend.ts`.

- [ ] **Step 4: Run full test suite**

- [ ] **Step 5: Commit**

```
refactor: activation + detach use Container API, delete backend DnD Closes #369
```

---

## Batch 4: Integration Tests + Final Verification

After this batch: comprehensive integration tests verify DnD, persistence round-trip, and workspace mode switching. Typecheck clean, lint clean.

### Task 10: Integration tests + final verification

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/combinatorial.test.ts` — add DnD tests
- Modify: `packages/pages-runtime/src/wire-floating-workspace.test.ts` — add mode switch tests
- Test: `packages/pages-runtime/src/layout-migration.test.ts` — add edge case tests

- [ ] **Step 1: Add DnD integration tests**

Test cross-entry tab transfer via DOM event dispatch in a free-layout container.

- [ ] **Step 2: Add workspace mode switch tests**

Test that `rootContainer.setLayout("tabbed")` / `setLayout("free")` preserves entries.

- [ ] **Step 3: Add persistence migration edge cases**

Test migration with nested containers, split containers, hidden frames.

- [ ] **Step 4: Run full verification**

```bash
yarn workspace @casehubio/pages-runtime run test
yarn typecheck
yarn lint
```

All must pass.

- [ ] **Step 5: Commit**

```
test: integration tests for DnD, mode switch, persistence migration Refs #369
```

---

## Final verification

After all batches:

```bash
yarn workspace @casehubio/pages-runtime run test
yarn typecheck
yarn lint
```

All must pass. The test count should increase (new tests for layout-math, detachEntry, DnD, migration) while the deleted engine tests are replaced.

## References

- [2026-08-25-workspace-as-container-design.md] — design spec
- [decisions.md] — D1-D8 decisions
- #345 recursive container model (Entry.childContainer, refreshEntry, surgical replant)
- #366 split divider resize
- #367 arrange button at all depths
- #368 toolbar position
- `container-tree-ops.ts` — tree traversal (findParentOf, findContainerWithTab)
- `frame-shell.ts` — shared frame rendering primitives
- `frame-boundaries.ts` — edge zone detection
- `free-layout-strategy.ts` — current free-layout implementation
- `group-organiser-backend.ts` — current backend (DnD code to absorb)
- `wire-floating-workspace.ts` — current orchestrator (to rewrite)
- `floating-frame-engine.ts` — current engine (to delete)
- `frame-organisers.ts` — current preset functions (to extract to layout-math)
