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

Continued implementation of epic #625 — unified grid layout model with two-mode editor. Completed Task 6: rewrote `grid-edit-controller.ts` with two-mode support. Total progress: 6 of 14 tasks complete (Batches 1-6 done).

Task 6 changes: Rewrote the controller to support layout mode (swap drag, selection with click/shift-extend) and content mode (displace/push/compact drag via compactAll). Items now identified by index (`data-item-id`) instead of component ID to support empty containers. Mode toggle added to overlay. Keydown listener moved to document for escape handling.

## Immediate Next Step

Task 7: Update GridEditBridge for new operation types — add split/merge/swap/remove/add-empty operations. Then Task 8: YAML migration script, Task 9: parser cleanup.

## References

- Design spec: `specs/issue-625-unified-grid-layout-model/2026-11-08-unified-grid-layout-model-design.md`
- Decisions: `specs/issue-625-unified-grid-layout-model/decisions.md`
- Plan: `plans/2026-11-08-unified-grid-layout-model.md`
- POC: `examples/poc/grid-drag-poc.html` (on issue-618 branch)
- Epic: casehubio/casehub-pages#625
