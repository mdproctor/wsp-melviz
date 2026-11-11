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

Implemented #639 (nested grid DnD + cross-grid drag) — full implementation across 5 tasks in 3 batches. Also fixed #650 (generic component parsing bug) during manual testing.

### What was built

1. **GridEditCoordinator** (`grid-edit-coordinator.ts`) — new module replacing the singleton controller pattern. Builds parent-child tree from DOM containment, resolves deepest grid under pointer, resolves DOM → YAML paths, manages cross-grid drag state.

2. **Controller changes** (`grid-edit-controller.ts`) — removed `activeController` singleton (multiple controllers active simultaneously), added `stopPropagation` on nested controller `pointerdown`, added `releaseItem`/`acceptForeignItem`/`clearForeignItem` for cross-grid handoff, `dispatchEdit` now includes coordinator-resolved `gridPath`.

3. **Path-routed bridge** (`grid-edit-bridge.ts`) — `GridEditDetail` gains optional `gridPath`, bridge routes mutations to the correct nested YAML node. New `applyCrossGridEdit` for atomic remove-from-source + insert-into-target in single undo frame.

4. **Proportional scaling** (`grid-scale.ts`) — `scaleWidth(sourceW, sourceCols, targetCols)` with `Math.round` + clamp to `[1, targetColumns]`.

5. **Builder-shell wiring** — listens for `pages-grid-cross-edit` event alongside existing `pages-grid-edit`.

6. **Parsing fix (#650)** — `desugarComponent` now catches any component type (not just `"grid"`) with a `grid:` property before the `"unknown"` fallback. Panel, container, and any other component type with nested grid items now render correctly.

### What needs debugging next session

The coordinator infrastructure and bridge routing are tested and working (56 new unit tests across 4 packages). But **visual testing revealed multiple issues** that need a debugging session:

- **#651 — Builder chrome contrast**: tree view, properties panel use hardcoded colors with poor contrast. Design tokens protocol exists but builder shell doesn't use them consistently.
- **Cross-grid drag wiring**: the coordinator methods exist and are tested, but the controller's `handlePointerMove` doesn't call the coordinator during drag to detect grid boundary crossings. The pointer handoff needs wiring.
- **Pre-existing `isNestedGrid` test failure**: uses `items` instead of `grid` (post-#640 format mismatch).
- **Demo YAML**: updated to use nested grid layout for manual testing (`packages/pages-builder/demo/main.ts`).

### Test counts

| Package | Tests | Status |
|---------|-------|--------|
| pages-component coordinator | 10 | all pass |
| pages-component controller | 23 | all pass |
| pages-component scale | 6 | all pass |
| pages-builder bridge | 17 | all pass |
| pages-ui desugar | 703 | all pass |

## Branch State

Branch: `issue-639-nested-grid-dnd-cross-grid-drag` — 8 commits ahead of main. Clean working tree.

## Open Issues

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #650 | Nested grid items render as type 'unknown' | S | **Fixed this session** |
| #651 | Builder chrome poor contrast | S | Filed, not started |

## References

- Spec: `specs/issue-639-nested-grid-dnd-cross-grid-drag/2026-11-10-nested-grid-dnd-cross-grid-drag-design.md` (in workspace)
- Decisions: `specs/issue-639-nested-grid-dnd-cross-grid-drag/decisions.md` (in workspace)
- Plan: `plans/2026-11-10-nested-grid-dnd-cross-grid-drag.md` (in workspace)
- Demo: `packages/pages-builder/demo/main.ts` (nested grid YAML)
- Next issue: #651 (builder chrome contrast)
