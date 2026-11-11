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

Implemented #639 (nested grid DnD + cross-grid drag), fixed #650 (generic component parsing), and fixed #651 (builder chrome contrast).

### Commits on branch

1. `feat(#639)`: GridEditCoordinator — tree registration, path resolution, singleton removal
2. `feat(#639)`: stopPropagation on nested controller pointerdown
3. `feat(#639)`: path-routed bridge — nested grid edits target correct YAML path
4. `feat(#639)`: controller release/accept API + proportional scaling
5. `feat(#639)`: cross-grid drag lifecycle — coordinator handoff + atomic bridge mutations
6. `fix(#650)`: generic component handler — any type with grid: gets nested items parsed
7. `fix(#651)`: dark-mode CSS tokens on dock workbench — tree and panels readable

### What needs debugging next session

The coordinator infrastructure, bridge routing, and parsing are implemented and tested. Visual testing confirmed nested grids render and the tree is readable. But **the following need work**:

- **Cross-grid drag wiring**: the coordinator methods exist and are tested, but the controller's `handlePointerMove` doesn't call the coordinator during drag to detect grid boundary crossings. The pointer handoff from controller to coordinator needs wiring.
- **Pre-existing `isNestedGrid` test failure**: uses `items` instead of `grid` (post-#640 format mismatch in grid-compaction.test.ts).
- **Toolbar contrast**: the Source/Split/Visual tab bar and +Page/+Dataset buttons still use light-mode fallbacks. These sit above the dock workbench and aren't covered by the token override.
- **Grid item visual boundaries**: items in the preview have no visible borders/shadows — hard to tell where one item ends and another begins without hovering.

### Test counts

| Package | Tests | Status |
|---------|-------|--------|
| pages-component (coordinator, controller, scale) | 39 new | all pass |
| pages-builder (bridge) | 17 total (3 new) | all pass |
| pages-ui (desugar, parser) | 703 total | all pass |

## Branch State

Branch: `issue-639-nested-grid-dnd-cross-grid-drag` — 10 commits ahead of main. Clean working tree.

## References

- Spec: `specs/issue-639-nested-grid-dnd-cross-grid-drag/` (in workspace)
- Plan: `plans/2026-11-10-nested-grid-dnd-cross-grid-drag.md` (in workspace)
- Demo: `packages/pages-builder/demo/main.ts` (nested grid YAML, `yarn --cwd packages/pages-builder dev`)
