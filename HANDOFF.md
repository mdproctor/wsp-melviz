# HANDOFF — casehub-pages

## Last Session

Completed all 14 implementation tasks for epic #625 — unified grid layout model. Then ran a comprehensive code review that caught a migration bug, a renderer gap (missing `data-item-id`), and runtime null guards for empty containers. All fixes applied, all tests pass, edit controller activates in the gallery.

**Critical remaining issue:** The builder tree UX is broken. Removing rows/columns from the model broke all the tree context menus, insertion flows, and interaction patterns without replacing them with items-aware equivalents. This is a UX regression that needs dedicated work.

## What's Broken — Builder Tree UX

The tree view, context menus, and insertion flows were built around the rows/columns model. That model is gone but the UI wasn't rebuilt:

1. **Context menus reference dead operations:** "Add row" on page, "Wrap in Row/Column" on component, "Add column" on row — these actions call removed PageNode methods (`addRow`, `insertRowAt`, `wrapInRow`)
2. **Insert flow doesn't work for items:** Clicking the down-arrow on a grid item shows "Before/After" options but the insertion code path calls `insertRowAt` / column-based insertion that no longer exists. Nothing gets added.
3. **Component picker categories stale:** "Layout" category still shows "Grid", "Columns", "Rows" — "Rows" type no longer makes sense as a structural insertion target
4. **Tree node types mismatch:** `TreeNodeType` still includes `'row' | 'column'` even though these node types only appear in unmigrated legacy documents
5. **`_handleTreeAction`** still has `add-column` handler; `_findRowAtPath` and `_findColumnAtPath` remain but their mutation callers are gone
6. **No grid-item-specific actions:** Items in the tree have no meaningful context menu — should have: move, resize, split, merge, delete, add empty container

## What Needs to Happen

Step through each builder interaction end-to-end and make it work for the items model:

1. **Tree context menu for `page` node:** Replace "Add row" → "Add item" (creates empty container at next available position)
2. **Tree context menu for `grid-item` node:** Add move up/down, split horizontal/vertical, delete. These should dispatch through `GridEditBridge`
3. **Insert before/after on grid items:** Use `PageNode.addEmptyItem()` at the correct placement position, then let the user drag a component into it
4. **Component picker integration:** When adding a component to a page, create a positioned item with the component (not a row/column wrapper)
5. **Remove "Wrap in Row/Column"** from component context menu. Keep "Wrap in Tabs"
6. **Remove row/column tree node types** and their context menus
7. **Edit layout button in builder preview:** Verify the `renderPreview` → `loadSite` path activates the grid edit controller inside the builder's Visual mode

## Branch State

All implementation and review-fix commits are on `issue-625-unified-grid-layout-model`. Clean working tree, no uncommitted changes. All test suites pass.

## References

- Design spec: `specs/issue-625-unified-grid-layout-model/2026-11-08-unified-grid-layout-model-design.md`
- Decisions: `specs/issue-625-unified-grid-layout-model/decisions.md`
- Plan: `plans/2026-11-08-unified-grid-layout-model.md`
- Epic: casehubio/casehub-pages#625
