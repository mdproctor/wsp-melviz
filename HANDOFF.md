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

Implemented #639 (nested grid DnD + cross-grid drag infrastructure), fixed #650 (component parsing), fixed #651 (builder chrome contrast), fixed tree view hierarchy rendering.

### Branch: issue-639-nested-grid-dnd-cross-grid-drag

12 commits ahead of main. Clean working tree.

### What was built

**Coordinator infrastructure** — new `GridEditCoordinator` module replaces the singleton controller pattern. Builds parent-child tree from DOM containment, resolves deepest grid under pointer, resolves DOM → YAML paths, manages cross-grid drag state. Controllers register/deregister automatically.

**Path-routed bridge** — `GridEditDetail` gains optional `gridPath`. Bridge routes mutations to the correct nested YAML node. New `applyCrossGridEdit` for atomic cross-grid moves (single undo frame).

**Component parsing fix (#650)** — `desugarComponent` now catches any component type (not just `"grid"`) with a `grid:` property. Panel, container, and other types with nested grid items now parse and render correctly.

**Builder chrome contrast (#651)** — dark-mode `--pages-*` token overrides on `pages-dock-workbench` so tree, properties, and palette panels are readable.

**Tree view hierarchy** — `ComponentNode.isContainer()` dynamically detects components with inline `grid:` keys. `buildGridItemNode` promotes nested grid children directly (no redundant intermediate nodes).

### What needs a thorough debugging session

The infrastructure is in place and unit-tested (56 new tests, all passing). But **visual testing in the builder demo revealed multiple unresolved issues** that need systematic debugging:

**Tree view problems:**
- Labels truncate aggressively — "Markdown" shows as "M..." even with space available
- Component labels are generic ("Markdown") — should show distinguishing info (content preview, title prop)
- No visual distinction between root grid items and nested grid items (same icon, same style)
- Tree selection and preview highlighting may not sync correctly for nested items
- Context menu actions (add, insert, cut, copy) untested for nested grid items

**Preview rendering problems:**
- Nested grid items have no visual boundaries — hard to tell where one item ends and another begins
- Panel title renders below its nested content (should be above or as a header)
- Grid item hover/selection outlines not tested for nested grids
- Drag handles and resize edges untested for nested grid items
- Cross-grid drag pointer handoff not wired — coordinator methods exist but controller's `handlePointerMove` doesn't call them during drag

**Pre-existing issues:**
- `isNestedGrid` test in grid-compaction.test.ts uses `items` instead of `grid` (post-#640 format mismatch)
- Toolbar (Source/Split/Visual tabs, +Page/+Dataset buttons) still uses light-mode fallbacks
- Demo `main.ts` updated with nested grid YAML but component labels need work

### Test counts

| Package | New tests | Status |
|---------|-----------|--------|
| pages-component (coordinator) | 10 | pass |
| pages-component (controller) | 3 | pass |
| pages-component (scale) | 6 | pass |
| pages-builder (bridge) | 3 | pass |
| pages-ui (desugar) | 1 updated | pass |

### How to test

```bash
yarn --cwd packages/pages-builder dev
# Opens http://localhost:5173 — Visual mode shows nested grid layout
```

## References

- Spec: workspace `specs/issue-639-nested-grid-dnd-cross-grid-drag/`
- Decisions: workspace `specs/issue-639-nested-grid-dnd-cross-grid-drag/decisions.md`
- Plan: workspace `plans/2026-11-10-nested-grid-dnd-cross-grid-drag.md`
- Demo: `packages/pages-builder/demo/main.ts`
- Issues: #639 (nested DnD), #650 (parsing, fixed), #651 (contrast, fixed)
