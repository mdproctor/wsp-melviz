# HANDOFF — casehub-pages

Branch: `issue-367-container-toolbar-unification`
Issue: #367, #368, #369 — Container toolbar unification + workspace as root Container

## Last Session

Completed Batch 2 (DnD event protocol, cross-entry tab transfer, edge splits + depth escape) and Task 7 of Batch 3 (migrateFrameLayout persistence function). 7 of 10 tasks done on the workspace-as-container plan. Assessed Tasks 8-9 (engine-to-Container migration) — coupled refactor across ~10 files and ~70 tests, deferred to next session. 1038 tests passing.

## Immediate Next Step

Continue plan at **Batch 3, Tasks 8-9 (engine-to-Container migration)**. This is an atomic refactor: rewrite `wireFloatingWorkspace` (~80 lines), migrate `activation.ts` (7 `handle.engine` call sites), rewrite `workspace-content-lifecycle.ts`, migrate `frame-detach-handler.ts` and `frame-keyboard.ts`, delete `floating-frame-engine.ts` (58 tests), `frame-organisers.ts`. Run `/work continue` to resume.

## References

- Plan: `plans/2026-08-25-workspace-as-container.md`
- Blog: `blog/2026-08-26-mdp01-dnd-protocol-and-the-engine-boundary.md`
