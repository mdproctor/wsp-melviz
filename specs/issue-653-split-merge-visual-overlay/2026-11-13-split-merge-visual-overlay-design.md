# Split/Merge Visual Overlay Design

**Issues:** #653, #654
**Branch:** issue-653-split-merge-visual-overlay
**Date:** 2026-11-13

## Overview

Move split and merge from the tree context menu to the visual grid editor's ⋮ action panel. Add a merge tool activation that uses CSS-based overlay colorization and cardinal-direction drag-to-absorb.

## Part 1: Overlay Toolbar Icon Grid (#653)

### Remove from tree

Delete `split-horizontal` and `split-vertical` from:
- `tree-context-menu.ts` — remove menu items from `computeMenuItems()` for `grid-item` nodes
- `builder-shell.ts` — remove `case 'split-horizontal'` / `case 'split-vertical'` from `_handleTreeAction()`

### Add to ⋮ action panel

Add a compact 2x3 icon grid to `selection-overlay.ts`, shown when a grid item is selected in editable mode:

```
 ┌──────┬──────┬──────┐
 │ ⬌    │ ⬍    │ ⊞    │
 │split-h│split-v│merge │
 ├──────┼──────┼──────┤
 │ ⇄    │ ✕    │ +    │
 │ swap │remove│ add  │
 └──────┴──────┴──────┘
```

### Enabled/disabled states

All 6 icons always rendered (static layout). Disabled states:
- **split-h**: disabled when selected item `w < 2`
- **split-v**: disabled when selected item `h < 2`
- **merge**: disabled when grid has only 1 item
- **swap**: disabled when fewer than 2 items selected
- **remove**: always enabled when an item is selected
- **add-empty**: always enabled in editable mode

### Action dispatch

Each icon dispatches to existing `GridEditBridge.applyGridEdit()`:
- split-h → `{ type: 'split', direction: 'horizontal' }`
- split-v → `{ type: 'split', direction: 'vertical' }`
- merge → activates merge tool (see Part 2)
- swap → `{ type: 'swap' }` (requires 2 selected items)
- remove → `{ type: 'remove' }`
- add-empty → `{ type: 'add-empty' }`

## Part 2: Merge Tool Activation (#654)

### Interaction flow

1. **Activate** — user clicks merge icon in ⋮ panel. The ⋮ panel closes. The grid enters merge state (transient, not a mode).
2. **Visual overlay** — CSS classes applied:
   - `[data-merge-active]` on grid container — dims all items via `opacity` or semi-transparent overlay
   - `[data-merge-root]` on the selected item — distinct highlight color (the merge root)
3. **Drag to absorb** — user presses on the merge root and drags in a cardinal direction:
   - Drag direction from root center determines axis (horizontal or vertical)
   - Contiguous cells along that axis from the root edge become candidates (adjacency = shared edge on the drag axis, no jumping over gaps)
   - As pointer moves further from root, more cells highlight with `[data-merge-target]` class
   - Live preview: highlighted cells show the absorption boundary
4. **Release to commit** — pointer-up triggers merge:
   - Compute bounding rect of root + all targeted cells via `mergeItems()` from `grid-split-merge.ts`
   - Call `GridEditBridge.applyGridEdit({ type: 'merge', ... })`
   - YAML updates immediately
   - **Stay in merge state** — the now-larger root item remains highlighted, ready for another drag (L-shape support)
5. **Exit** — Escape key or click outside the grid. Remove all `data-merge-*` attributes.

### L-shape merges

Two sequential drags from the same root:
1. First drag merges horizontally (e.g. 3 cells right) — committed immediately
2. Root is now wider. Second drag merges vertically — committed immediately
3. Result: L-shaped merged region (actually a bounding rectangle of both operations)

Each drag is an independent `applyGridEdit` call. The merge state persists between drags. No special L-shape geometry — sequential bounding-rect merges handle it naturally.

### State machine in GridEditController

Add to `GridEditControllerImpl`:

```typescript
private _mergeActive = false;
private _mergeRootId: string | null = null;
private _mergeTargetIds: string[] = [];
```

Pointer event flow when `_mergeActive`:
- `_onPointerDown` — if on merge root, begin drag tracking. Record start position.
- `_onPointerMove` — determine cardinal direction from drag start. Compute candidate cells along that axis. Update `_mergeTargetIds`, toggle `[data-merge-target]` classes.
- `_onPointerUp` — if targets exist, call bridge merge. Update root to merged item. Clear targets but keep `_mergeActive`.
- Escape listener — call `_exitMerge()` to clean up all state and classes.

### CSS classes

```css
[data-merge-active] .grid-item {
  opacity: 0.4;
  pointer-events: none;
}
[data-merge-active] [data-merge-root] {
  opacity: 1;
  outline: 2px solid var(--merge-root-color, #4a9eff);
  pointer-events: auto;
  cursor: grab;
}
[data-merge-active] [data-merge-target] {
  opacity: 0.8;
  outline: 2px dashed var(--merge-target-color, #ff9f43);
}
```

### Entry point wiring

SelectionOverlay dispatches a custom event (e.g. `grid-merge-activate`) when merge icon clicked. The grid component's event handler calls `controller.enterMerge(selectedItemId)`. The controller sets `_mergeActive = true`, applies CSS classes, and takes over pointer semantics.

## Scope boundaries

**In scope:**
- Remove split/merge from tree context menu
- 2x3 icon grid in ⋮ panel for grid items
- Merge tool activation with CSS overlay
- Cardinal drag-to-absorb with L-shape support
- Split dispatches directly to bridge (no new interaction — just click)

**Out of scope:**
- Undo/redo for merge operations (existing undo infrastructure handles this)
- Merge across nested grids (same-grid only)
- Animated transitions during merge (CSS transitions on the class changes are sufficient)

## References

- `packages/pages-builder/src/tree/tree-context-menu.ts` — split menu items to remove
- `packages/pages-builder/src/shell/builder-shell.ts` — split handlers to remove
- `packages/pages-builder/src/overlay/selection-overlay.ts` — ⋮ panel to extend
- `packages/pages-component/src/renderer/grid-edit-controller.ts` — merge state machine host
- `packages/pages-component/src/renderer/grid-overlay.ts` — existing overlay patterns
- `packages/pages-builder/src/shell/grid-edit-bridge.ts` — existing split/merge YAML mutations
- `packages/pages-component/src/renderer/grid-split-merge.ts` — merge geometry (mergeItems, canSplit)
