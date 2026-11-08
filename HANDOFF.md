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
