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

## Status

On main. #653 (split/merge visual overlay) and #654 (drag-to-absorb merge) landed and closed. #608 (TS/Java YAML Runtime Parity) epic continues — next items in queue.

## What just shipped

#653/#654 — Split/merge visual overlay + drag-to-absorb merge. 43 commits compacted to 7 via semantic squash, landed as 5e0e868d.

Key features:
- Grid-item overlay toolbar: 2x3 spatial action grid (split-h, split-v, merge, delete, add-empty) with hover expand/collapse
- Drag-to-absorb merge: progressive targeting, CSS overlay for root/target highlighting, click-outside exit
- 60px grid row enforcement with visible split items (markdown content instead of typeless empty)
- Multi-page demo with pill tab switcher (Color Grid, Content Layout, Two Forms)
- Unified grid-item controls: ⋮ trigger and action buttons moved from SelectionOverlay into controller's drag handle bar — one DOM element, one display toggle, zero timing gaps
- Selection rectangle snaps on click (immediate `_applyPreviewHighlight`, 150ms handle delay to match slide)

Key decisions:
- Single ownership: grid-item controls belong to the controller, not the overlay. The overlay keeps only the selection outline for grid items.
- Event bridge pattern: controller dispatches `pages-grid-action` (bubbles, composed), shell translates to existing `selection-*` handlers on overlay-root.
- 150ms handle delay on new selection to roughly sync with the overlay's 150ms CSS transition. Not perfectly synchronized (different event sources), accepted as good enough.

## Filed Issues

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #660 | Sync handle bar and selection rectangle timing | S/Med | Enhancement — shared orchestrator for perfect sync |
| #661 | Tree selection doesn't show grid-item controls | S/Low | Bug — tree click doesn't trigger controller's _setSelectedEl |
| #662 | Overlay outline flashes at old position after cross-grid drag | S/Med | Bug — showBuilderOverlay 50ms setTimeout |
| #663 | Grid controller and selection overlay tests need DOM environment | S/Low | Testing — jsdom/happy-dom for vitest |

## Queue

| # | Issue | Scale | Notes |
|---|-------|-------|-------|
| #647 | Phase 6 — phase-aware agent simulation | M/Med | P6 from audit |
| #645 | Streaming simulation API | S/Low | Chunked corpus responses |
| #646 | Agent session showcase | S/Low | Multi-turn demo |

## Blog

`blog/2026-11-14-mdp01-overlay-dual-ownership.md` — "One Element, One Toggle": dual-ownership anti-pattern in overlay UIs, event bridge pattern, architectural lessons.

## References

- Audit: `docs/specs/2026-11-07-ts-platform-parity-audit.md`
- Epic: casehubio/casehub-pages#608

## Changes made from the soredium session (2026-11-15)

Recorded per the cross-repo handoff rule. Context: casehubio/casehub-pages#666, Hortora/soredium#515.

- **Recovered diary "One Element, One Toggle"** (`blog/2026-11-14-mdp01-overlay-dual-ownership.md` plus `blog/images/overlay-dual-ownership.svg` and `overlay-event-bridge.svg`). It existed only as orphaned commit `bb30ff9`: it was committed while this workspace was stopped in a `pull --rebase` started by `work pause`, and a later `git rebase --abort` left it on no branch.
  - Workspace: commit `488a67f` on `issue-665-add-grid-editing-demo-showcase-gallery` (entry, images, INDEX row).
  - Project: commit `67d7b9ad` on the same branch, promoting to `docs/blog/`. It lands when #665 closes.
  - Publication: committed locally in `~/claude/mdproctor.github.io` (`4a45875`) and `~/claude/casehub/casehubio.github.io` (`8cce709`). Both pushed on 2026-11-16 (casehubio upstream and the mdproctor fork), along with the other publish commits that were waiting in those repos.
- Three other commits from the same stopped rebase are still orphaned and were not recovered: `b00507d` (session wrap), `0b46a5f` (queue branch update), `478c03a` (`.work-progress`). They hold lifecycle state only.
- `close.py` changed on this machine: it refuses a stamped branch, a branch the queue does not name, and any repo that is mid-rebase or detached; it never squashes; a multi-commit branch needs a git-squash review recorded at its tip. A session started before 2026-11-15 needs `/clear` to load the updated skills.
