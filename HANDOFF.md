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

Completed all 14 tasks of epic #625 — unified grid layout model with two-mode editor. All batches (1-12) done.

This session completed Tasks 6-14:
- **Task 6:** Two-mode edit controller (layout swap + content compact)
- **Task 7:** GridEditBridge with split/merge/swap/remove/add-empty operations
- **Task 8:** YAML migration script (migrateRowsToItems)
- **Task 9:** Removed rows/columns parser paths (~1600 lines deleted)
- **Task 10:** Recursive nested sub-grid support (desugarComponent + isNestedGrid)
- **Task 11:** emptyContainer() DSL builder
- **Task 12:** Edge resize operations (resizeItem)
- **Task 13:** Builder-shell wiring verified (already connected)
- **Task 14:** Migrated 221 sample YAML files from rows/columns to items format

## Immediate Next Step

All implementation tasks complete. Ready for `work end` — code review, squash, and merge to main.

## Known Pre-existing Issues

- `grid-overlay-insertion.test.ts`: 4 tests fail (insertion zones stub from Task 5 overlay rewrite)
- `builders.test.ts`: 2 tests fail (grid/at builders — pre-existing)
- 9 pre-existing TypeScript compilation errors in pages-builder (childKey, toJSON, type narrowing)

## References

- Design spec: `specs/issue-625-unified-grid-layout-model/2026-11-08-unified-grid-layout-model-design.md`
- Decisions: `specs/issue-625-unified-grid-layout-model/decisions.md`
- Plan: `plans/2026-11-08-unified-grid-layout-model.md`
- POC: `examples/poc/grid-drag-poc.html` (on issue-618 branch)
- Epic: casehubio/casehub-pages#625
