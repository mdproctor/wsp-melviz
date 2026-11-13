# Split/Merge Visual Overlay Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #653 — move split/merge from tree context menu to visual overlay toolbar
**Issue group:** #653, #654

**Goal:** Replace tree-based split/merge with a visual overlay toolbar on grid items, and add a merge tool activation with cardinal drag-to-absorb and L-shape support.

**Architecture:** The ⋮ action panel in `selection-overlay.ts` gets a 2x3 icon grid for grid items (split-h, split-v, merge, swap, remove, add-empty). Split/swap/remove/add-empty dispatch directly to the existing `GridEditBridge`. Merge activates a transient state in `GridEditController` — CSS class toggles dim the grid, highlight the root, and show absorption targets during drag. The bridge and geometry layers (`grid-edit-bridge.ts`, `grid-split-merge.ts`) already handle the mutations.

**Tech Stack:** TypeScript, Lit Web Components, CSS custom properties

## Global Constraints

- All spatial operations use existing `GridEditBridge.applyGridEdit()` — no new mutation layer
- Merge geometry uses existing `mergeItems()` from `grid-split-merge.ts` — bounding-rect semantics
- `IdRect` interface: `{ id: string, x: number, y: number, w: number, h: number }` from `grid-compaction.ts`
- `GridEditDetail` union type in `grid-edit-bridge.ts` defines all edit shapes — no new variants needed
- `ButtonDef` tuple: `readonly [cls: string, label: string, event: string]` in `selection-overlay.ts`
- Events bubble from `_overlayRoot` via `CustomEvent` with `{ path, nodeType }` detail

---

## Batch 1: Overlay toolbar icon grid (#653)

### Task 1: Add spatial action buttons to selection overlay

**Files:**
- Modify: `packages/pages-builder/src/overlay/selection-overlay.ts:5-30` — add grid-item spatial buttons and update `getButtonsForNodeType()`
- Modify: `packages/pages-builder/src/shell/builder-shell.ts:512-559` — wire new overlay events
- Test: `packages/pages-builder/src/overlay/__tests__/selection-overlay.test.ts` (create)

**Interfaces:**
- Consumes: `ButtonDef` type, `getButtonsForNodeType()`, `GridEditBridge.applyGridEdit()`
- Produces: 6 new custom events — `selection-split-h`, `selection-split-v`, `selection-merge`, `selection-swap`, `selection-remove-grid`, `selection-add-empty`

- [ ] **Step 1: Write the failing test for grid-item button set**

```typescript
import { describe, it, expect } from 'vitest';

// Test the getButtonsForNodeType function directly
// We need to extract and export it, or test via DOM
describe('getButtonsForNodeType', () => {
  it('returns spatial action buttons for grid-item node type', () => {
    const buttons = getButtonsForNodeType('grid-item', false);
    const events = buttons.map(([, , evt]) => evt);
    expect(events).toContain('selection-split-h');
    expect(events).toContain('selection-split-v');
    expect(events).toContain('selection-merge');
    expect(events).toContain('selection-swap');
    expect(events).toContain('selection-delete');
    expect(events).toContain('selection-add-empty');
  });

  it('does not include spatial buttons for component node type', () => {
    const buttons = getButtonsForNodeType('component', false);
    const events = buttons.map(([, , evt]) => evt);
    expect(events).not.toContain('selection-split-h');
    expect(events).not.toContain('selection-merge');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-builder/src/overlay/__tests__/selection-overlay.test.ts`
Expected: FAIL — `getButtonsForNodeType` not exported or `selection-split-h` not in results

- [ ] **Step 3: Implement spatial buttons in selection-overlay.ts**

Export `getButtonsForNodeType` for testability. Add the grid-item spatial buttons:

```typescript
const GRID_ITEM_BUTTONS: ButtonDef[] = [
  ['sel-split-h-btn', '⬌', 'selection-split-h'],
  ['sel-split-v-btn', '⬍', 'selection-split-v'],
  ['sel-merge-btn', '⊞', 'selection-merge'],
  ['sel-swap-btn', '⇄', 'selection-swap'],
  ['sel-delete-btn', '✕', 'selection-delete'],
  ['sel-add-empty-btn', '+', 'selection-add-empty'],
];
```

Update `getButtonsForNodeType()`:
```typescript
export function getButtonsForNodeType(
  nodeType: TreeNodeType,
  isContainer: boolean,
): ButtonDef[] {
  if (nodeType === 'grid-item') {
    return GRID_ITEM_BUTTONS;
  }
  // ... existing cases unchanged
}
```

Update `BTN_STYLE` and the toolbar rendering in `update()` to use a 2x3 CSS grid layout when the buttons are `GRID_ITEM_BUTTONS`:

```typescript
// In update(), after creating buttonsContainer:
const isGridItemToolbar = nodeType === 'grid-item';
buttonsContainer.style.cssText = isGridItemToolbar
  ? `display: grid; grid-template-columns: repeat(3, 20px); gap: 2px;
     max-height: 0; overflow: hidden; opacity: 0;
     transition: max-height 150ms ease, opacity 150ms ease;`
  : `display: flex; gap: 2px;
     max-width: 0; overflow: hidden; opacity: 0;
     transition: max-width 150ms ease, opacity 150ms ease;`;
```

Update the mouseenter/mouseleave to toggle `max-height` instead of `max-width` when `isGridItemToolbar`:

```typescript
toolbar.addEventListener('mouseenter', () => {
  if (isGridItemToolbar) {
    buttonsContainer.style.maxHeight = '50px';
  } else {
    buttonsContainer.style.maxWidth = '200px';
  }
  buttonsContainer.style.opacity = '1';
});
toolbar.addEventListener('mouseleave', () => {
  if (this._toolbarLocked) return;
  if (isGridItemToolbar) {
    buttonsContainer.style.maxHeight = '0';
  } else {
    buttonsContainer.style.maxWidth = '0';
  }
  buttonsContainer.style.opacity = '0';
});
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-builder/src/overlay/__tests__/selection-overlay.test.ts`
Expected: PASS

- [ ] **Step 5: Wire new events in builder-shell.ts**

In `_wireSelectionOverlayEvents()`, add handlers for the new spatial events:

```typescript
const onSplitH = (e: Event) => {
  const { path, nodeType } = (e as CustomEvent).detail;
  if ((path[2] === 'grid' || path[2] === 'items') && typeof path[3] === 'number') {
    this._applyEdit('toolbar', () => {
      this._gridBridge.applyGridEdit(this._document, {
        type: 'split', targetIndex: path[3] as number, direction: 'horizontal',
      });
    });
  }
};
const onSplitV = (e: Event) => {
  const { path, nodeType } = (e as CustomEvent).detail;
  if ((path[2] === 'grid' || path[2] === 'items') && typeof path[3] === 'number') {
    this._applyEdit('toolbar', () => {
      this._gridBridge.applyGridEdit(this._document, {
        type: 'split', targetIndex: path[3] as number, direction: 'vertical',
      });
    });
  }
};
const onSwap = (e: Event) => {
  // Swap requires 2 selected items — the selection state comes from the grid controller
  // For now, dispatch as a custom event on the grid element for the controller to handle
  const { path } = (e as CustomEvent).detail;
  this._overlayRoot?.dispatchEvent(new CustomEvent('grid-swap-activate', {
    bubbles: true, detail: { path },
  }));
};
const onMerge = (e: Event) => {
  const { path } = (e as CustomEvent).detail;
  this._selectionOverlay?.hide();
  this._overlayRoot?.dispatchEvent(new CustomEvent('grid-merge-activate', {
    bubbles: true, detail: { path },
  }));
};
const onAddEmpty = (e: Event) => {
  const { path } = (e as CustomEvent).detail;
  const page = this._findPageAtPath(['pages', path[1] as number]);
  if (page) {
    this._applyEdit('toolbar', () => {
      const items = page.getItems();
      const maxY = items.reduce((m, it) => {
        const p = it.getPlacement();
        return p.mode === 'positioned' ? Math.max(m, p.y + p.h) : m;
      }, 0);
      page.addEmptyItem({ mode: 'positioned', x: 0, y: maxY, w: 12, h: 1 });
    });
  }
};

overlayRoot.addEventListener('selection-split-h', onSplitH);
overlayRoot.addEventListener('selection-split-v', onSplitV);
overlayRoot.addEventListener('selection-swap', onSwap);
overlayRoot.addEventListener('selection-merge', onMerge);
overlayRoot.addEventListener('selection-add-empty', onAddEmpty);
```

Add cleanup for all new listeners in the cleanup function.

- [ ] **Step 6: Commit**

```bash
git add packages/pages-builder/src/overlay/selection-overlay.ts packages/pages-builder/src/overlay/__tests__/selection-overlay.test.ts packages/pages-builder/src/shell/builder-shell.ts
git commit -m "feat(#653): add 2x3 spatial action grid to overlay toolbar for grid items"
```

### Task 2: Remove split from tree context menu

**Files:**
- Modify: `packages/pages-builder/src/tree/tree-context-menu.ts:19-25` — remove split menu items
- Modify: `packages/pages-builder/src/shell/builder-shell.ts:1286-1294` — remove split handlers
- Test: `packages/pages-builder/src/tree/__tests__/tree-context-menu.test.ts` (create)

**Interfaces:**
- Consumes: `computeMenuItems()`, `MenuItem` type from `pages-primitives`
- Produces: updated `computeMenuItems()` without split entries for `grid-item`

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, it, expect } from 'vitest';
import { computeMenuItems } from '../tree-context-menu.js';

describe('computeMenuItems', () => {
  it('grid-item menu does not include split actions', () => {
    const items = computeMenuItems('grid-item', false);
    const actions = items.map(i => i.action).filter(Boolean);
    expect(actions).not.toContain('split-horizontal');
    expect(actions).not.toContain('split-vertical');
  });

  it('grid-item menu still includes move and delete', () => {
    const items = computeMenuItems('grid-item', false);
    const actions = items.map(i => i.action).filter(Boolean);
    expect(actions).toContain('move-up');
    expect(actions).toContain('move-down');
    expect(actions).toContain('delete');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-builder/src/tree/__tests__/tree-context-menu.test.ts`
Expected: FAIL — `split-horizontal` is still in the menu

- [ ] **Step 3: Remove split items from tree-context-menu.ts**

In `computeMenuItems()`, update the `'grid-item'` case:

```typescript
case 'grid-item':
  return [
    { label: 'Move up', action: 'move-up' },
    { label: 'Move down', action: 'move-down' },
    { separator: true, label: '' },
    { label: 'Delete', action: 'delete', shortcut: '⌫' },
  ];
```

- [ ] **Step 4: Remove split handlers from builder-shell.ts**

In `_handleTreeAction()`, remove the `case 'split-horizontal':` and `case 'split-vertical':` block (lines 1286-1294).

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run packages/pages-builder/src/tree/__tests__/tree-context-menu.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-builder/src/tree/tree-context-menu.ts packages/pages-builder/src/tree/__tests__/tree-context-menu.test.ts packages/pages-builder/src/shell/builder-shell.ts
git commit -m "feat(#653): remove split actions from tree context menu  Refs #653"
```

## Batch 2: Merge tool activation (#654)

### Task 3: Add merge state and CSS overlay to GridEditController

**Files:**
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts:50-90` — add merge state fields
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts:30-46` — add `enterMerge()` and `exitMerge()` to interface
- Test: `packages/pages-component/src/renderer/__tests__/grid-merge.test.ts` (create)

**Interfaces:**
- Consumes: `GridEditController` interface, `IdRect` from `grid-compaction.ts`, `mergeItems()` from `grid-split-merge.ts`
- Produces: `enterMerge(rootId: string): void`, `exitMerge(): void` on `GridEditController`; CSS data attributes `data-merge-active`, `data-merge-root`, `data-merge-target`

- [ ] **Step 1: Write the failing test for merge state entry/exit**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';

describe('GridEditController merge state', () => {
  let gridEl: HTMLElement;

  beforeEach(() => {
    gridEl = document.createElement('div');
    gridEl.innerHTML = `
      <div data-item-id="0" style="grid-column: 1 / span 4; grid-row: 1 / span 1;">A</div>
      <div data-item-id="1" style="grid-column: 5 / span 4; grid-row: 1 / span 1;">B</div>
      <div data-item-id="2" style="grid-column: 9 / span 4; grid-row: 1 / span 1;">C</div>
    `;
    document.body.appendChild(gridEl);
  });

  it('enterMerge sets data-merge-active on grid and data-merge-root on root item', () => {
    const controller = attachGridEditController(gridEl, items, 12);
    controller.enter();
    controller.enterMerge('0');

    expect(gridEl.dataset.mergeActive).toBeDefined();
    const rootEl = gridEl.querySelector('[data-item-id="0"]') as HTMLElement;
    expect(rootEl.dataset.mergeRoot).toBeDefined();
  });

  it('exitMerge removes all merge data attributes', () => {
    const controller = attachGridEditController(gridEl, items, 12);
    controller.enter();
    controller.enterMerge('0');
    controller.exitMerge();

    expect(gridEl.dataset.mergeActive).toBeUndefined();
    expect(gridEl.querySelector('[data-merge-root]')).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/__tests__/grid-merge.test.ts`
Expected: FAIL — `enterMerge` is not a function

- [ ] **Step 3: Add merge state fields and methods to controller**

Add to `GridEditController` interface:
```typescript
enterMerge(rootId: string): void;
exitMerge(): void;
```

Add private fields to `GridEditControllerImpl`:
```typescript
private _mergeActive = false;
private _mergeRootId: string | null = null;
private _mergeTargetIds: string[] = [];
private _mergeDragStart: { x: number; y: number } | null = null;
```

Implement `enterMerge()`:
```typescript
enterMerge(rootId: string): void {
  if (!this._active) return;
  this._mergeActive = true;
  this._mergeRootId = rootId;
  this._mergeTargetIds = [];
  this.gridEl.dataset.mergeActive = '';
  const rootEl = this.gridEl.querySelector(`[data-item-id="${rootId}"]`) as HTMLElement | null;
  if (rootEl) rootEl.dataset.mergeRoot = '';
}
```

Implement `exitMerge()`:
```typescript
exitMerge(): void {
  this._mergeActive = false;
  this._mergeRootId = null;
  this._mergeTargetIds = [];
  this._mergeDragStart = null;
  delete this.gridEl.dataset.mergeActive;
  for (const el of this.gridEl.querySelectorAll('[data-merge-root]')) {
    delete (el as HTMLElement).dataset.mergeRoot;
  }
  for (const el of this.gridEl.querySelectorAll('[data-merge-target]')) {
    delete (el as HTMLElement).dataset.mergeTarget;
  }
}
```

Update `handleKeyDown()` to exit merge on Escape:
```typescript
private handleKeyDown(e: KeyboardEvent): void {
  if (!this._active) return;
  if (e.key === 'Escape') {
    if (this._mergeActive) {
      this.exitMerge();
    } else if (this.dragging) {
      // ... existing drag cancel
    } else {
      this.exit();
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-component/src/renderer/__tests__/grid-merge.test.ts`
Expected: PASS

- [ ] **Step 5: Add merge overlay CSS**

Add CSS rules for merge state. These go in the component's styles or as inline styles applied by the controller. Since `grid-overlay.ts` uses inline styles, follow the same pattern — add a function `applyMergeStyles()` in `grid-overlay.ts`:

```typescript
export function applyMergeStyles(gridEl: HTMLElement): void {
  const style = gridEl.ownerDocument.createElement('style');
  style.dataset.mergeStyles = '';
  style.textContent = `
    [data-merge-active] > [data-item-id] {
      opacity: 0.4;
      pointer-events: none;
      transition: opacity 150ms ease;
    }
    [data-merge-active] > [data-merge-root] {
      opacity: 1;
      outline: 2px solid var(--pages-accent-9, #2563eb);
      outline-offset: -1px;
      pointer-events: auto;
      cursor: grab;
    }
    [data-merge-active] > [data-merge-target] {
      opacity: 0.8;
      outline: 2px dashed var(--pages-warning-7, #f59e0b);
      outline-offset: -1px;
      pointer-events: none;
    }
  `;
  gridEl.appendChild(style);
}

export function removeMergeStyles(gridEl: HTMLElement): void {
  gridEl.querySelector('[data-merge-styles]')?.remove();
}
```

Call `applyMergeStyles()` in `enterMerge()` and `removeMergeStyles()` in `exitMerge()`.

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/src/renderer/grid-edit-controller.ts packages/pages-component/src/renderer/grid-overlay.ts packages/pages-component/src/renderer/__tests__/grid-merge.test.ts
git commit -m "feat(#654): add merge state entry/exit with CSS overlay to GridEditController  Refs #654"
```

### Task 4: Implement drag-to-absorb pointer handling

**Files:**
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts:337-401` — merge-aware pointer-down
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts:403-575` — merge drag tracking
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts:577-671` — merge commit on pointer-up
- Test: `packages/pages-component/src/renderer/__tests__/grid-merge.test.ts` (extend)

**Interfaces:**
- Consumes: `enterMerge()`, `exitMerge()`, `getIdRects()`, `mergeItems()` from `grid-split-merge.ts`, `dispatchEdit()`, `findItemAtPoint()`
- Produces: merge drag interaction — pointer events update `_mergeTargetIds` and toggle `data-merge-target` attributes; pointer-up dispatches `pages-grid-edit` with type `merge`

- [ ] **Step 1: Write the failing test for cardinal direction detection**

Add a pure helper function `computeMergeTargets()` that can be tested independently:

```typescript
import { describe, it, expect } from 'vitest';
import { computeMergeTargets } from '../grid-merge-targets.js';
import type { IdRect } from '../grid-compaction.js';

describe('computeMergeTargets', () => {
  const items: IdRect[] = [
    { id: '0', x: 0, y: 0, w: 4, h: 1 },
    { id: '1', x: 4, y: 0, w: 4, h: 1 },
    { id: '2', x: 8, y: 0, w: 4, h: 1 },
  ];

  it('dragging right from item 0 targets item 1', () => {
    const targets = computeMergeTargets(items, '0', 'right', 1);
    expect(targets).toEqual(['1']);
  });

  it('dragging right from item 0 with distance 2 targets items 1 and 2', () => {
    const targets = computeMergeTargets(items, '0', 'right', 2);
    expect(targets).toEqual(['1', '2']);
  });

  it('dragging left from item 2 targets item 1', () => {
    const targets = computeMergeTargets(items, '2', 'left', 1);
    expect(targets).toEqual(['1']);
  });

  it('does not jump over gaps', () => {
    const gapped: IdRect[] = [
      { id: '0', x: 0, y: 0, w: 4, h: 1 },
      { id: '2', x: 8, y: 0, w: 4, h: 1 },
    ];
    const targets = computeMergeTargets(gapped, '0', 'right', 2);
    expect(targets).toEqual([]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/__tests__/grid-merge.test.ts`
Expected: FAIL — `grid-merge-targets.ts` does not exist

- [ ] **Step 3: Create grid-merge-targets.ts with computeMergeTargets()**

Create `packages/pages-component/src/renderer/grid-merge-targets.ts`:

```typescript
import type { IdRect } from './grid-compaction.js';

export type MergeDirection = 'left' | 'right' | 'up' | 'down';

export function computeMergeDirection(
  dx: number,
  dy: number,
): MergeDirection | null {
  if (Math.abs(dx) < 5 && Math.abs(dy) < 5) return null;
  return Math.abs(dx) > Math.abs(dy)
    ? (dx > 0 ? 'right' : 'left')
    : (dy > 0 ? 'down' : 'up');
}

export function computeMergeTargets(
  items: readonly IdRect[],
  rootId: string,
  direction: MergeDirection,
  distance: number,
): string[] {
  const root = items.find(i => i.id === rootId);
  if (!root || distance <= 0) return [];

  const targets: string[] = [];
  let edge: number;

  switch (direction) {
    case 'right': {
      edge = root.x + root.w;
      for (let d = 0; d < distance; d++) {
        const next = items.find(i =>
          i.id !== rootId && !targets.includes(i.id) &&
          i.x === edge && i.y <= root.y && i.y + i.h >= root.y + root.h
        );
        if (!next) break;
        targets.push(next.id);
        edge = next.x + next.w;
      }
      break;
    }
    case 'left': {
      edge = root.x;
      for (let d = 0; d < distance; d++) {
        const next = items.find(i =>
          i.id !== rootId && !targets.includes(i.id) &&
          i.x + i.w === edge && i.y <= root.y && i.y + i.h >= root.y + root.h
        );
        if (!next) break;
        targets.push(next.id);
        edge = next.x;
      }
      break;
    }
    case 'down': {
      edge = root.y + root.h;
      for (let d = 0; d < distance; d++) {
        const next = items.find(i =>
          i.id !== rootId && !targets.includes(i.id) &&
          i.y === edge && i.x <= root.x && i.x + i.w >= root.x + root.w
        );
        if (!next) break;
        targets.push(next.id);
        edge = next.y + next.h;
      }
      break;
    }
    case 'up': {
      edge = root.y;
      for (let d = 0; d < distance; d++) {
        const next = items.find(i =>
          i.id !== rootId && !targets.includes(i.id) &&
          i.y + i.h === edge && i.x <= root.x && i.x + i.w >= root.x + root.w
        );
        if (!next) break;
        targets.push(next.id);
        edge = next.y;
      }
      break;
    }
  }

  return targets;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-component/src/renderer/__tests__/grid-merge.test.ts`
Expected: PASS

- [ ] **Step 5: Wire merge pointer handling into GridEditController**

In `handlePointerDown()`, add merge-aware early return at the top:

```typescript
private handlePointerDown(e: PointerEvent): void {
  if (this._mergeActive) {
    const target = this.findItemAtPoint(e.clientX, e.clientY);
    if (target && target.id === this._mergeRootId) {
      this._mergeDragStart = { x: e.clientX, y: e.clientY };
      this.gridEl.setPointerCapture(e.pointerId);
    }
    return;
  }
  // ... existing logic
}
```

In `handlePointerMove()`, add merge drag tracking:

```typescript
private handlePointerMove(e: PointerEvent): void {
  if (!this._active) return;
  if (this._mergeActive && this._mergeDragStart) {
    const dx = e.clientX - this._mergeDragStart.x;
    const dy = e.clientY - this._mergeDragStart.y;
    const dir = computeMergeDirection(dx, dy);
    if (!dir) return;
    const cellSize = this.geometry ? this.geometry.cellWidth + this.geometry.gap : 80;
    const dist = Math.max(1, Math.floor(Math.max(Math.abs(dx), Math.abs(dy)) / cellSize));
    const idRects = this.getIdRects();
    const newTargets = computeMergeTargets(idRects, this._mergeRootId!, dir, dist);

    // Clear old targets
    for (const el of this.gridEl.querySelectorAll('[data-merge-target]')) {
      delete (el as HTMLElement).dataset.mergeTarget;
    }
    this._mergeTargetIds = newTargets;
    // Apply new targets
    for (const id of newTargets) {
      const el = this.gridEl.querySelector(`[data-item-id="${id}"]`) as HTMLElement | null;
      if (el) el.dataset.mergeTarget = '';
    }
    return;
  }
  // ... existing logic
}
```

In `handlePointerUp()`, add merge commit:

```typescript
private handlePointerUp(e: PointerEvent): void {
  if (this._mergeActive && this._mergeDragStart) {
    this._mergeDragStart = null;
    if (this._mergeTargetIds.length > 0) {
      const allIds = [this._mergeRootId!, ...this._mergeTargetIds];
      const idRects = this.getIdRects();
      const merged = mergeItems(idRects, allIds, this._mergeRootId!);
      this.dispatchEdit('merge', merged);

      // Clear target highlights
      for (const el of this.gridEl.querySelectorAll('[data-merge-target]')) {
        delete (el as HTMLElement).dataset.mergeTarget;
      }
      this._mergeTargetIds = [];
      // Stay in merge state — root is now larger, ready for L-shape extension
    }
    return;
  }
  // ... existing logic
}
```

Add import at top of file:
```typescript
import { computeMergeDirection, computeMergeTargets } from './grid-merge-targets.js';
import { mergeItems } from './grid-split-merge.js';
```

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/src/renderer/grid-merge-targets.ts packages/pages-component/src/renderer/grid-edit-controller.ts packages/pages-component/src/renderer/__tests__/grid-merge.test.ts
git commit -m "feat(#654): implement drag-to-absorb merge with cardinal direction and L-shape support  Refs #654"
```

### Task 5: Wire merge activation from overlay to controller

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts:208-248` — handle `grid-merge-activate` event
- Test: manual verification — merge flow end-to-end

**Interfaces:**
- Consumes: `grid-merge-activate` custom event from Task 1, `GridEditController.enterMerge()` from Task 3
- Produces: end-to-end wiring — click merge icon in ⋮ → controller enters merge state

- [ ] **Step 1: Write the wiring in builder-shell.ts**

The `grid-merge-activate` event was dispatched by the overlay handler in Task 1. Now wire it to the controller. In `_wirePreviewClicks()` or alongside the `_gridEditHandler`, add:

```typescript
const onMergeActivate = (e: Event) => {
  const { path } = (e as CustomEvent).detail;
  if ((path[2] === 'grid' || path[2] === 'items') && typeof path[3] === 'number') {
    const gridEl = container.querySelector(`[data-grid-path="${path.slice(0, 2).join('/')}"]`) as HTMLElement | null;
    if (gridEl) {
      const controller = getCoordinator().getControllerForEl(gridEl);
      if (controller) {
        controller.enterMerge(String(path[3]));
      }
    }
  }
};
```

Register on the preview container and clean up on disconnect.

- [ ] **Step 2: Verify the complete flow manually**

Start the dev server. Open a page with a grid layout. Click a grid item, click ⋮, verify the 2x3 icon grid appears. Click merge — verify grid items dim, root highlights. Drag from root — verify adjacent cells highlight. Release — verify merge commits. Drag again in perpendicular direction — verify L-shape. Press Escape — verify exit.

- [ ] **Step 3: Commit**

```bash
git add packages/pages-builder/src/shell/builder-shell.ts
git commit -m "feat(#654): wire merge activation from overlay toolbar to grid controller  Refs #654"
```

## References

- [2026-11-13-split-merge-visual-overlay-design.md] — design spec this plan implements
- [packages/pages-builder/src/overlay/selection-overlay.ts] — ⋮ panel ButtonDef, getButtonsForNodeType, toolbar rendering
- [packages/pages-builder/src/tree/tree-context-menu.ts] — computeMenuItems with split entries to remove
- [packages/pages-builder/src/shell/builder-shell.ts:512-559] — _wireSelectionOverlayEvents, event wiring pattern
- [packages/pages-builder/src/shell/builder-shell.ts:1258-1324] — _handleTreeAction with split handlers to remove
- [packages/pages-builder/src/shell/grid-edit-bridge.ts:1-14] — GridEditDetail union, applyGridEdit
- [packages/pages-component/src/renderer/grid-edit-controller.ts:30-46] — GridEditController interface
- [packages/pages-component/src/renderer/grid-edit-controller.ts:337-401] — handlePointerDown
- [packages/pages-component/src/renderer/grid-edit-controller.ts:673-685] — handleKeyDown Escape handling
- [packages/pages-component/src/renderer/grid-split-merge.ts:43-61] — mergeItems bounding-rect geometry
- [packages/pages-component/src/renderer/grid-compaction.ts:1-10] — IdRect interface
- [packages/pages-component/src/renderer/grid-overlay.ts] — existing overlay patterns (inline styles, data attributes)
- [GitHub #653] — move split/merge from tree context menu to visual overlay toolbar
- [GitHub #654] — merge sub-mode with drag-to-merge
