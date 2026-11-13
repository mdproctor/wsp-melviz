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

Implemented cross-grid displace with displacement preview, pinned nested panels, and extensive UI polish. Branch `issue-639-nested-grid-dnd-cross-grid-drag` is functionally complete — 16 commits (11 from semantic squash + 5 post-squash fixes). Needs one more squash pass before landing.

Key achievements: cross-grid displace works both directions (nested→parent and parent→nested). Pinned panels solve the "chasing problem" where compactAll pushed nested panels away from the cursor. Pin toggle (📌/📍) on panels with nested grids, state on coordinator singleton. Displacement preview with compactDown + hysteresis eliminates visual flickering at cell boundaries.

Key design decisions: panels with nested grids are pinned by default during displace drags. Dwell-to-unpin was explored and rejected (cursor-based signal doesn't match item-position-based displacement). compactAll switched from push+compact to compactDown to prevent gravity-compact-induced oscillation. FLIP animation removed during drag (200ms transition fought 16ms pointer events). Hysteresis tracks displaced positions to prevent snap-back at cell boundaries.

## How to Run

```bash
yarn --cwd packages/pages-builder dev
# Or: cd packages/pages-builder && npx vite --config demo/vite.config.ts --port 5176
# If Vite serves stale code: rm -rf packages/pages-builder/node_modules/.vite
```

Demo at http://localhost:5176/ — nested grid layout with cross-grid drag.

## Architecture

1. **GridEditController** — per-grid instance. Within-grid drag/resize/swap/displace. Pointer events, snap-to-cell, compactDown with hysteresis. Pinned panel detection via `_getPinnedIds()`. `pointerleave` clears hover. `clearInteractionState()` for parent-child coordination.

2. **GridEditCoordinator** — singleton. Cross-grid lifecycle: `startCrossGridDrag` → `handleCrossGridMove` (displacement preview with compactAll + FLIP) → `completeCrossGridDrag` (returns `CrossGridEditDetail` with `targetItemPlacements`). Pin state: `isPinned()`/`togglePin()` keyed by grid path, survives controller recreation. `_buildTargetPinnedIds()` for cross-grid preview.

3. **GridEditBridge** — `applyCrossGridEdit` mutates YAML. Applies `targetItemPlacements` to compact existing items, then appends moved item.

4. **Builder Shell** — `_crossGridEditHandler` applies edit, sets selection, dispatches synthetic pointerdown/pointermove after 400ms for auto-select. `_highlightPreviewNode` with pad=0 aligns selection overlay with item outlines.

## Before Landing

- Re-squash: 5 post-squash commits need folding into the 11 semantic commits. Run `/git-squash` with range covering the new commits.
- Remove console.log in grid-edit-controller.ts (search for `[drag]`)
- Artifact sweep: 9 workspace artifacts not yet promoted to project

## Next Work

- Split/merge issue — user has a plan
- Migration scripts for dependent repos in ~/claude/casehub — find repos that depend on pages and migrate their grids/tables from old approach. Existing script: `scripts/migrate-to-grid-flat.mjs`
