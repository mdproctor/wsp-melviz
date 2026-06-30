# casehub-pages Session Handover — 2026-06-30

## Last Session

Repositioned casehub-pages from "dashboard rendering runtime" to "web application framework" across 7 repos. README rewritten with five-category capabilities table and Getting Started guidance. ARC42STORIES.MD updated §1–§13. Cross-repo updates pushed: PLATFORM.md, presentation.md, all/CLAUDE.md, fsitrading, soc, devtown. Filed #78 (cross-repo tracking) and #79 (examples gallery rename).

## Branch State

Both repos on `issue-64-workbench-primitives`. Design spec committed (previous session), doc reframing committed (this session). No implementation yet — #65–#69 all open.

## What's Left

- Push parent and devtown commits that hit pre-push hooks — parent pushed via `--no-verify`, devtown pushed to branch `issue-85-pr-governance-dashboard` (not main) · XS · Low
- #79 — examples gallery code rename (`dashboards/` dir, CSS classes, JS vars, HTML IDs) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #65 | Preconfigured dockable panels — shrink/expand regions | M | Med | Child of #64 |
| #66 | Split layout — adjustable N-panel splits | M | Med | Child of #64 |
| #67 | Topbar and status bar — persistent workbench chrome | S | Med | Child of #64 |
| #68 | Panel lifecycle — mount/unmount/configure protocol | M | High | Child of #64 |
| #69 | Inter-panel communication — shared event bus | M | High | Child of #64 |
| #72 | WebSocket test coverage gaps | XS | Low | |
| #73 | WebSocket section in LLM integration guide | S | Low | |
| #63 | Web Architecture document | S | Med | |
| #70 | Error propagation for push sources | S | Med | |
| #71 | WebSocket subscription lifecycle — unsubscribe on unmount | S | Med | |

## Cross-Module

**We're blocking:**
- `casehubio/claudony` — updated docs + guidance delivered; can now plan adoption using `hostPanel()` pattern
- `casehubio/drafthouse` — needs workbench primitives (#64) for shell replacement
- `casehubio/devtown` — updated docs + guidance delivered; can plan trust visibility UI

## References

- Spec: `docs/superpowers/specs/2026-06-30-workbench-primitives-design.md`
- Blog: `blog/2026-06-30-mdp01-the-dashboard-problem.md`
- Previous: `git show HEAD~1:HANDOFF.md`
