# HANDOFF — casehub-pages

Branch: `issue-367-container-toolbar-unification`
Issue: #367, #368, #369 — Container toolbar unification + workspace as root Container

## Last Session

Completed the engine elimination — Tasks 8-9 of the workspace-as-container plan plus full backend deletion. Rewrote `wireFloatingWorkspace` (362→140 lines), migrated all engine consumers (frame-keyboard, frame-detach-handler, frame-zone-picker, activation.ts, site.ts) to Container API, deleted engine (333 lines), backend (1094 lines), frame-organisers, workspace-content-lifecycle, floating-frame-backend interface, frame-state. Net: 4036 lines deleted, 433 added. 918 tests passing, typecheck clean, lint clean. Fixed pre-existing accordion `detachEntry` bug (`hostElement`→`containerEl`).

## Immediate Next Step

Task 10 (Batch 4): Integration tests + final verification. Write DnD integration tests (cross-entry tab transfer via DOM event dispatch), workspace mode switch tests (`setLayout("tabbed")`/`setLayout("free")` preserves entries), and migration edge case tests (nested containers, split containers). Run `/work continue` to resume.

## References

- Plan: `plans/2026-08-25-workspace-as-container.md`
- Blog: `blog/2026-08-26-mdp02-killing-the-engine.md`
