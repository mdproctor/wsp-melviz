# Nested Grid DnD with Cross-Grid Drag

**Issue:** casehubio/casehub-pages#639
**Branch:** issue-639-nested-grid-dnd-cross-grid-drag
**Date:** 2026-11-10

## Problem

`GridEditController` operates on a single grid element and its items array. A module-level `activeController` singleton allows only one controller at a time. `GridEditBridge` always routes mutations to `doc.getPages()[0].getItems()` — the root page's items. When a component of type `grid` is nested inside a grid item (a panel containing a sub-grid), the nested grid has no functional drag-and-drop, resize, split, or merge operations.

The requirement extends beyond independent controllers: users need **cross-grid drag** — moving items between any two grids in the hierarchy, with proportional size adaptation.

## Requirements

1. Each nested grid gets its own independent edit controller (drag, resize, split, merge)
2. Cross-grid drag: move items between any two grids in the hierarchy
3. Arbitrary nesting depth (tree of grids, not just parent-child)
4. Proportional width scaling when crossing grid boundaries (50% width in source → 50% width in target)
5. Full move semantics — component travels with placement (removed from source, inserted into target)
6. Target grid resolution: deepest grid under pointer
7. Empty nested grids persist as empty containers until manually removed

## Architecture

### Controller Tree with Coordinator (D1)

Replace the module-level `activeController` singleton with a `GridEditCoordinator`. The coordinator is a new module in `pages-component/src/renderer/grid-edit-coordinator.ts`.

```
GridEditCoordinator (singleton, module-level)
├── registers/deregisters controllers
├── builds parent-child tree from DOM containment
├── resolves deepest grid under pointer
├── owns cross-grid drag state
└── resolves DOM → YAML path for bridge routing

GridEditController (per grid, unchanged for single-grid ops)
├── handles drag, resize, split, merge within one grid
├── registers with coordinator on construction
├── deregisters on dispose
└── gains releaseItem() and acceptForeignItem() for cross-grid
```

Individual controllers retain all existing single-grid behavior. The coordinator adds cross-grid awareness as a separate concern.

### Controller Registration (D2)

Controllers register with the coordinator on construction (in `attachGridEditController`). The coordinator discovers parent-child relationships by DOM containment: if controller B's `gridEl` is a descendant of controller A's `gridEl`, B is a child of A.

The tree is rebuilt lazily — only when controllers are added or removed (on render/dispose), never during drag.

No changes to the render pipeline or activation.ts are needed. The existing `attachGridEditController` call in activation.ts (line 948-951) continues to work — the coordinator intercepts registration.

### DOM → YAML Path Reconstruction

The coordinator must map each grid's DOM element to its YAML path for bridge routing (D3). The algorithm:

1. Start at the grid element
2. Walk up the DOM, collecting path segments:
   - If the current element has `data-item-id`, record `[gridKey, Number(itemId)]` where `gridKey` is `'grid'` or `'items'` (determined by the parent grid's layout mode)
   - If the current element has `data-component-type`, skip (it's the component wrapper, path segment comes from the item-id above)
3. Prepend the page path (e.g., `['pages', 0]`)
4. The final path points to the grid array: e.g., `['pages', 0, 'grid']` for the root grid, or `['pages', 0, 'grid', 2, 'grid']` for a nested grid inside item 2

The `data-item-id` attribute (render.ts:146) stores the `originalIndex` which correctly maps to the YAML array index even after positional sort (render.ts:127-131).

### Bridge Routing (D3)

The `pages-grid-edit` event gains a `gridPath: (string | number)[]` property. The coordinator resolves the path (using the DOM → YAML algorithm above) and attaches it to the event before dispatch.

The bridge changes from:

```typescript
// Before: always root
const page = doc.getPages()[0];
const gridItems = page.getItems();
```

To:

```typescript
// After: path-routed
const gridPath = detail.gridPath; // e.g., ['pages', 0, 'grid', 2, 'grid']
const yamlDoc = doc._getDoc();
const seq = yamlDoc.getIn(gridPath);
// operate on seq items
```

Backward compatibility: if `gridPath` is absent (older events), fall back to root page items.

### Pointer Event Handling

**Non-drag interactions (selection, hover):** Each controller handles its own pointer events on its `gridEl` as today. Nested controllers call `e.stopPropagation()` on `pointerdown` to prevent parent controllers from reacting to clicks inside nested grids.

**During drag (D4):** The coordinator adds a document-level `pointermove` listener when a drag starts. On each move:

1. Resolve the deepest grid under the pointer (walk the coordinator's tree, hit-test each grid's bounding rect, prefer the deepest match)
2. If the target grid is the same as the current drag grid → delegate to the current controller's normal drag handling
3. If the target grid changes (cross-grid transition):
   a. Tell the source controller to `releaseItem(index)` — hides the element, removes from local flow
   b. Compute proportional scaling: `targetWidth = Math.max(1, Math.round(sourceWidth * targetColumns / sourceColumns))`
   c. Tell the target controller to `acceptForeignItem(item, scaledPlacement)` — shows placeholder at snapped position
   d. Update coordinator's cross-grid drag state (current grid, current item, original source)

On `pointerup`, the coordinator dispatches a `pages-grid-cross-edit` event with:
```typescript
{
  type: 'cross-move',
  sourceGridPath: (string | number)[],
  targetGridPath: (string | number)[],
  sourceIndex: number,
  targetPlacement: GridPlacement,
  component: Component,
}
```

On `Escape`, the coordinator cancels the cross-grid drag, restores the item in the source grid.

### Cross-Grid Bridge Mutations

The bridge handles `pages-grid-cross-edit` as an atomic operation:

1. Call `doc._pushUndoInternal()` once (single undo frame)
2. Remove item from source grid array at `sourceGridPath`
3. Insert item into target grid array at `targetGridPath` with scaled placement
4. Call `doc._notifyInternal()` once

This ensures a single undo reverses the entire cross-grid move.

### Proportional Scaling

When an item crosses grid boundaries:

```
scaledWidth = Math.max(1, Math.round(sourceWidth * targetColumns / sourceColumns))
```

Height is preserved unchanged (row height is independent of column count).

Clamping: `scaledWidth = Math.min(scaledWidth, targetColumns)`.

Examples:
- 6-wide in 12-col → 6-col: `Math.round(6 * 6/12) = 3` (50% → 50%)
- 5-wide in 12-col → 6-col: `Math.round(5 * 6/12) = Math.round(2.5) = 3`
- 1-wide in 6-col → 12-col: `Math.round(1 * 12/6) = 2`
- 1-wide in 12-col → 6-col: `Math.round(1 * 6/12) = Math.round(0.5) = 1` (clamped by max(1,...))

### Controller API Changes

`GridEditController` gains two new methods for cross-grid support:

```typescript
interface GridEditController {
  // Existing...
  releaseItem(index: number): void;
  acceptForeignItem(item: GridItem, placement: GridPlacement): void;
  clearForeignItem(): void;
}
```

- `releaseItem`: hides the item element, removes it from local displacement flow
- `acceptForeignItem`: shows a placeholder at the given placement position, applies displacement to existing items
- `clearForeignItem`: removes the foreign placeholder (on cancel or grid change)

### Singleton Removal

The module-level `let activeController` (grid-edit-controller.ts:40) is replaced by coordinator registration. The `enter()` method no longer exits the previous controller — multiple controllers can be active simultaneously.

The `enterPassive()` method (called on construction for grids with positioned items) continues to set up pointer listeners on the grid element. With `stopPropagation()` on nested controllers, parent controllers won't receive nested events.

### Empty Grid Cleanup (D5)

When all items are dragged out of a nested grid, the parent grid item remains as an empty grid container (`grid: []`). Users can drag items back or delete it manually via the context menu. Consistent with existing empty-item behavior.

## Files Changed

| File | Change |
|------|--------|
| `pages-component/src/renderer/grid-edit-coordinator.ts` | **New** — coordinator module |
| `pages-component/src/renderer/grid-edit-controller.ts` | Remove singleton, add releaseItem/acceptForeignItem/clearForeignItem, add stopPropagation on nested pointerdown |
| `pages-component/src/renderer/index.ts` | Export coordinator |
| `pages-builder/src/shell/grid-edit-bridge.ts` | Path-routed mutations, cross-grid-edit handler |
| `pages-builder/src/shell/builder-shell.ts` | Listen for `pages-grid-cross-edit` event |
| `pages-component/src/renderer/grid-edit-controller.test.ts` | Tests for nested controllers, release/accept |
| `pages-component/src/renderer/grid-edit-coordinator.test.ts` | **New** — coordinator tree, path resolution, cross-grid drag |
| `pages-builder/src/shell/grid-edit-bridge.test.ts` | Tests for path-routed and cross-grid mutations |

## Testing Strategy

1. **Unit — coordinator tree:** Register controllers at various depths, verify tree structure, verify deepest-grid resolution
2. **Unit — path reconstruction:** DOM elements with data-item-id attributes, verify YAML paths
3. **Unit — proportional scaling:** All rounding edge cases
4. **Unit — release/accept:** Controller items array manipulation during foreign item operations
5. **Integration — single nested grid:** Drag within a nested grid, verify bridge routes to correct YAML path
6. **Integration — cross-grid drag:** Drag from parent to nested and vice versa, verify both YAML arrays mutated correctly
7. **Integration — undo:** Cross-grid drag undoes in a single step
8. **Integration — arbitrary depth:** Three-level nesting, drag between level 1 and level 3

## Non-Goals

- Direct sibling-to-sibling grid drag shortcuts. Dragging from nested grid A (in parent item 1) to nested grid B (in parent item 2) works naturally — the pointer path goes A → parent → B through the coordinator tree. What's excluded is any special "direct transfer" logic that bypasses the intermediate parent grid.
- Creating new nested grids by dragging items. Users create nested grids via the component picker.
- Visual preview of the dragged item during cross-grid transit (ghost/clone). The source hides the item, the target shows a placeholder. A floating preview could be added later.

## References

- grid-edit-controller.ts — existing controller (877 lines, all single-grid operations)
- grid-edit-bridge.ts — bridge routing (root-only)
- activation.ts:948-951 — controller attachment callsite
- render.ts:104-152 — recursive grid rendering
- grid-compaction.ts:141 — `isNestedGrid()` utility
- page-document.ts:372-381 — path-based `getItems()`
- container-descriptors.ts:27 — `grid` type registered with `grid-items` slot kind
- types.ts:12 — `Component.grid?: readonly GridItem[]` enables nesting
- D16 (from #625) — independent compaction per grid level
- D17 (from #625) — nested grids are components of type `grid` with `items`
