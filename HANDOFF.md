# HANDOFF — casehub-pages (slot 218)

## Status: CLOSED — ready for archive

All work in this slot is complete and landed on main.

## Completed Issues

### #390 — Scenario format refinements (TS side)
Extracted dispatch protocol types to `playbook/types.ts`. Renamed CommandPayload.target to element, added DispatchStep.delay, made speed opt-in, added REST action definition. Landed on main.

### #531 — Inferred method-call dispatch for playbook steps
Added secure inferred dispatch so unrecognised step names resolve as method calls on target elements (TS) or beans (Java). Key deliverables:
- `PLAYBOOK_ACTIONS` Symbol SPI in pages-primitives
- `Catalog.resolveInferred()` in yaml-core (Walker accepts unknown step keys)
- Secure `invokePlaybookMethod` replacing raw `invokeMethod` in command-executor.ts
- `InferredMethodHandler` + catalog factory wiring in pages-aria
- `playbook-handler.ts` default case extended for server-dispatched path
- `pages-document-diff` as first adopter (9 methods declared)
- `@PlaybookInterface` annotation + `ActionRegistry` fallback on Java side
- Post-audit fix: `ElementMethodBinding` export, `executeAriaAction` made async
Landed on main, issue closed.

## Design Artifacts (workspace)

- `specs/issue-531-inferred-method-dispatch/` — design spec, decisions, pipeline state
- `plans/2026-10-20-inferred-method-dispatch.md` — implementation plan
- `specs/issue-571-decorator-structural-keys/` — prior work spec
- `plans/2026-10-20-scenario-format-refinements.md` — prior work plan

## References

- `packages/pages-primitives/src/playbook/` — PLAYBOOK_ACTIONS SPI
- `packages/pages-aria/src/executor/playbook-resolve.ts` — shared dispatch utilities
- `packages/pages-aria/src/playbook/invoke/inferred-method-handler.ts` — InferredMethodHandler
- `packages/pages-aria/src/playbook/catalog-factory.ts` — resolveInferred on catalog
- `packages/pages-document-diff/src/pages-document-diff.ts` — first PLAYBOOK_ACTIONS adopter
- `backend/playbook-client/src/main/java/.../PlaybookInterface.java` — Java annotation
- `backend/playbook-client/src/main/java/.../ActionRegistry.java` — interface scanning fallback

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
