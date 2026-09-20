# Session Handover

**Branch:** `issue-456-pointer-pipeline` (2 commits ahead of main)
**Issue:** #456 — Unified pointer event pipeline and drill-down stack
**Date:** 2026-09-20
**Paused:** #433 — issue-433-structural-editing (pause stack depth 1)

## What Was Built

### This session was design-only — no implementation code for #456 yet.

### Prior work on #433 (paused)
- SelectionOverlay class — selection-based toolbar replacing hover-based ComponentOverlayManager
- Examples gallery fixes — theme crash (builder import crashing UMD bundle), .page label suffix, compact theme picker

### Design work for #456
- Root-caused the drag bug: pointer capture fight between React Flow's full-node Handle and custom NodeMoveCoordinator
- Explored blocks-ui drill-down (case → swf) via 3-phase protocol
- Brainstormed unified pointer event pipeline — 5 design decisions captured
- Wrote spec: NodeGestureCoordinator (capture-phase interceptor) + DrillDownStack (visual cascade with collapsible bars)
- Standard design review — 3 dimensions (coherence/structure/robustness), 3 rounds each, 49 findings all resolved
- Wrote implementation plan — 4 batches, 7 tasks, all TDD
- Created GitHub issue #456 and branch

## Key Design Decisions

1. **Full-node drag-to-connect stays** — visible port dots would imply sticky ports (model is node-to-node)
2. **Immediate drag = connect, hold 300ms = move** — existing gesture model is correct, implementation was broken
3. **Capture-phase interceptor** — blocks React Flow's Handle during 300ms classification, replays event for CONNECT
4. **Drill-down moves to graph-renderer** — emit/navigate + stack generalized, resolution via consumer callback
5. **Visual cascade** — collapsed vertical bars stack left-to-right, click bar to navigate back
6. **ARIA compliant** — keyboard (Enter/Escape/Home), focus management, aria-live announcements

## What's Next

| Priority | Item | Scale | Notes |
|----------|------|-------|-------|
| 1 | Batch 1: NodeGestureCoordinator + wire into GraphCanvas | M | Fixes the drag bug — TDD |
| 2 | Batch 2: DrillDownState + DrillDownBars with ARIA | S | Pure logic + DOM, independent |
| 3 | Batch 3: isDrillable injection + wire drill-down into GraphCanvas | M | Full integration |
| 4 | Batch 4: Nested Pipeline example (3-level) | S | Showcase |

## Files

- Spec: `specs/issue-456-pointer-pipeline/2026-09-20-pointer-pipeline-design.md`
- Decisions: `specs/issue-456-pointer-pipeline/2026-09-20-pointer-pipeline-decisions.md`
- Plan: `plans/2026-09-20-pointer-pipeline.md`
- Review workspaces: `~/reviews/casehub-pages/pointer-pipeline-spec-{coherence,structure,robustness}-*`

## Cross-Repo

- **blocks-ui** is on branch `issue-158-lsp-schema-refinements` — if changes needed: switch to main first, make changes, switch back and rebase
- blocks-ui migration is a follow-up after #456 lands — file a cross-repo issue in blocks-ui repo first
