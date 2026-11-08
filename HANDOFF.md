# HANDOFF — casehub-pages

## Last Session

Fixed the builder tree UX for the items model (#625). All 7 checklist items from the previous session are now complete and verified in Playwright:

1. **Page context menu:** "Add item" replaces "Add row" — creates empty container at next available y position
2. **Grid-item context menu:** Move up/down, Split horizontal/vertical, Delete — wired through `GridEditBridge`
3. **Insert before/after:** Displacement semantics — "before" places at target position and pushes target down; "after" places below target
4. **Component picker integration:** Selecting a component on an items-mode page creates a positioned grid item (not a flat component)
5. **"Wrap in Row/Column" removed:** Only "Wrap in Tabs" remains on component menu
6. **Row/column context menus removed:** No stale menus for dead node types
7. **Edit layout button:** Activates grid edit controller in builder's Visual mode

Additional fixes applied during the work:
- `TreeNodeType` union now includes `'grid-item'` — previously used as string literal without being in the type
- Filled grid items in the tree use nodeType `'grid-item'` (was incorrectly `'component'`)
- `classifyPath` handles `items` paths — returns `'grid-item'` for YAML cursor navigation
- `_resolvePropertySource` handles `grid-item` — shows component properties for filled items, placement for empty
- `_findGridItemAtPath` helper added for path resolution
- `scope-collector.ts` handles `grid-item` nodeType for preview highlighting
- Filed #639 for nested grid DnD (controller only targets root items)

## What's Left on #625

The core grid model and builder tree UX are complete. Remaining work:

- **#639 — Nested grid DnD:** `GridEditController` only operates on root-level items. Nested grids inside containers have no drag/drop support.
- **Split undo batching:** Split creates two mutations (resize + add empty) requiring two undos. Should batch into a single undo step.

## Branch State

All implementation and UX-fix commits are on `issue-625-unified-grid-layout-model`. Clean working tree, all test suites pass.

## References

- Design spec: `docs/specs/issue-625-unified-grid-layout-model/2026-11-08-unified-grid-layout-model-design.md`
- Decisions: `docs/specs/issue-625-unified-grid-layout-model/decisions.md`
- Plan: `plans/2026-11-08-unified-grid-layout-model.md`
- Test page: `packages/pages-builder/builder-tree-test.html`
- Epic: casehubio/casehub-pages#625
- Nested grid DnD: casehubio/casehub-pages#639
