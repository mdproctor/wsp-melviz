# casehub-pages Session Handover — 2026-07-02

## Last Session

Closed paused branch `issue-85-cleanup-batch` (#85 terminal tests, #79 gallery rename — landed on main). Then added `distinctJoin` aggregation to pages-data (#84) and updated stale "dashboard rendering" references in historical docs (#78 — in-repo scope only). Both issues closed, 0.3.0 now includes distinctJoin. Versioning strategy discussed: Maven `0.3` = npm `0.3.0`, SNAPSHOT maps to npm alpha prerelease tags.

## Branch State

Both repos on `main`. Pause stack empty.

## What's Left

- PLATFORM.md update — approved wording for layout serialization, needs applying in casehub-parent repo · XS · Low
- #78 cross-repo scope — casehub-parent, casehub-all, fsitrading, soc, devtown still have stale "dashboard rendering" refs · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #86 | Publish 0.3.0 with workbench primitives + terminal | S | Low | CI green, distinctJoin included — unblocked |
| #21 | Optional Quarkus backend MVP | M | Med | REST LayoutStore adapter plugs into #76 contract |
| #75 | Drag-and-drop panel rearrangement | L | High | future epic |
| #77 | Floating/popout panels | M | High | detach into separate windows |

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Previous: `git show HEAD~1:HANDOFF.md`
