# HANDOFF — casehub-pages

Branch: `issue-367-container-toolbar-unification`
Issue: #367, #368, #369 — Container toolbar unification + workspace as root Container

## Last Session

Completed the engine elimination — Tasks 8-9 of the workspace-as-container plan plus full backend deletion. Rewrote `wireFloatingWorkspace` (362→140 lines), migrated all engine consumers (frame-keyboard, frame-detach-handler, frame-zone-picker, activation.ts, site.ts) to Container API, deleted engine (333 lines), backend (1094 lines), frame-organisers, workspace-content-lifecycle, floating-frame-backend interface, frame-state. Net: 4036 lines deleted, 433 added across 28 files. 918 tests passing (down from 1038 — delta is deleted engine/backend tests), typecheck clean, lint clean.

Fixed pre-existing accordion `detachEntry` bug (`hostElement`→`containerEl` — silent no-op via optional chaining on undefined variable).

## Immediate Next Step

Task 10 (Batch 4, final task): Integration tests + final verification. Three test files to modify:

1. `packages/pages-runtime/src/frame-sandbox/combinatorial.test.ts` — add DnD integration tests: cross-entry tab transfer via DOM event dispatch in a free-layout container
2. `packages/pages-runtime/src/wire-floating-workspace.test.ts` — add workspace mode switch tests: `rootContainer.setLayout("tabbed")` / `setLayout("free")` preserves entries and child containers
3. `packages/pages-runtime/src/layout-migration.test.ts` — add migration edge cases: nested containers, split containers, hidden frames, round-trip (migrate → capture → compare)

After tests pass, run full verification: `yarn workspace @casehubio/pages-runtime run test`, `yarn typecheck`, `yarn lint`. Then `work end` to close the branch.

## References

- Plan: `plans/2026-08-25-workspace-as-container.md`
- Blog: `blog/2026-08-26-mdp02-killing-the-engine.md`
