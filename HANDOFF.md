# casehub-pages Session Handover — 2026-06-30

## Last Session

Designed and implemented epic #64 — workbench primitives. Three composable types (split, dockBar, hostPanel) plus unified event bus extending the existing WebSocket data pipeline with an `event` op. Design review (5 rounds, 17 issues, all resolved). TDD implementation (7 tasks). app-grid removed. All six sub-issues (#64–#69) closed, squashed to 1 commit, pushed to fork + blessed.

## Branch State

Both repos on `main`. Fork and blessed current.

## What's Left

- #81 — Wire eventTarget from loadSite to WebSocketSource for event op dispatch · XS · Low
- #82 — Add runtime pages-event listener in site.ts for interception/logging · XS · Low
- #83 — Update ARC42STORIES.MD and CASEHUB-PAGES.MD for workbench primitives · S · Low
- #332 (casehubio/parent) — Update PLATFORM.md capability entry for workbench primitives · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #81 | Wire eventTarget from loadSite to WebSocketSource | XS | Low | event op end-to-end gap |
| #82 | Add runtime pages-event listener for interception | XS | Low | middleware hook |
| #83 | Update ARC42STORIES.MD + CASEHUB-PAGES.MD for workbench | S | Low | doc sync |
| #72 | WebSocket test coverage gaps | XS | Low | |
| #73 | WebSocket section in LLM integration guide | S | Low | |
| #74 | SSE source type | S | Low | same op vocabulary |
| #63 | Web Architecture document | S | Med | |
| #70 | Error propagation for push sources | S | Med | |
| #71 | WebSocket subscription lifecycle — unsubscribe on unmount | S | Med | |
| #75 | Drag-and-drop panel rearrangement | L | High | future epic |

## Cross-Module

**We're blocking:**
- `casehubio/drafthouse` — can now migrate shell to pages primitives (split, dockBar, hostPanel)
- `casehubio/claudony` — can build workbench UIs with the same primitives
- `casehubio/devtown` — can plan trust visibility dashboard using hostPanel

## References

- Spec: `docs/superpowers/specs/2026-06-30-workbench-primitives-design.md`
- Blog: `blog/2026-06-30-mdp02-three-primitives-one-pipeline.md`
- Design review: `~/adr/casehub-pages/workbench-primitives-20260630-063705/`
- Garden: `GE-20260630-b8e2d8` (CSS Grid fr tracks don't collapse on display:none)
- Previous: `git show HEAD~1:HANDOFF.md`
