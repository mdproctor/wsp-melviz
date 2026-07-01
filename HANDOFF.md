# casehub-pages Session Handover — 2026-07-01

## Last Session

Cleared the backlog: #85 (terminal test quality — fake timers, backoff progression, ordering tests), #79 (gallery rename dashboard→sample — directory, CSS, JS, HTML, tests, build scripts), #78 (workbench spec cross-repo reference update). Then designed and implemented #76 (layout serialization — observable split resize via pages-split-resize, LayoutStore contract with localStorage adapter, auto-persistence wiring in site.ts, lazy container restore, hidden dock panel ratio correction). Design review: 4 rounds, 16 issues, all resolved. Also wrote getting-started guide and built packages for local consumption.

## Branch State

Both repos on `main`. Fork and blessed current. Pause stack has `issue-85-cleanup-batch` (paused — the 3 small issues from earlier are committed there but not yet merged to main via work-end; they were committed directly, not through the full close flow).

## What's Left

- Paused branch `issue-85-cleanup-batch` — contains terminal test improvements (#85), gallery rename (#79), spec update (#78). Needs work-end to close properly and land on main.
- PLATFORM.md update — approved wording for layout serialization capability, needs applying in casehub-parent repo.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #21 | Optional Quarkus backend MVP | M | Med | REST LayoutStore adapter would plug into the contract from #76 |
| #75 | Drag-and-drop panel rearrangement | L | High | future epic |
| #77 | Floating/popout panels | M | High | detach into separate windows |

## Cross-Module

**We're blocking:**
- `casehubio/claudony` — can now embed terminal panels + persist workspace layouts; needs claudony#166 (close code 4001)
- `casehubio/drafthouse` — can migrate shell to pages primitives with persistent layout
- `casehubio/connectors` — getting-started guide written for chat client setup

## References

- Spec: `docs/superpowers/specs/2026-07-01-layout-serialization-design.md`
- Getting started: `docs/GETTING-STARTED.md`
- Blog: `blog/2026-07-01-mdp02-layout-as-first-class.md`
- Design review: `~/adr/casehub-pages/layout-serialization-20260701-094609/`
- Previous: `git show HEAD~1:HANDOFF.md`
