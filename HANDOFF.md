# casehub-pages Session Handover — 2026-07-01

## Last Session

Fixed CI: typecheck errors in `data-pipeline-lifecycle.test.ts` and `data-pipeline.test.ts` (test types drifted from current interfaces during layout serialization work), plus CodeQL workflow referencing removed `./core` Java directory. #87 closed. CI should now be green on main.

## Branch State

Both repos on `main`. Pause stack has `issue-85-cleanup-batch` (paused — terminal test improvements #85, gallery rename #79, spec update #78 committed but not closed via work-end).

## What's Left

- Paused branch `issue-85-cleanup-batch` — needs work-end to close properly and land on main · S · Low
- PLATFORM.md update — approved wording for layout serialization, needs applying in casehub-parent repo · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #86 | Publish 0.3.0 with workbench primitives + terminal | S | Low | CI now green — unblocked |
| #21 | Optional Quarkus backend MVP | M | Med | REST LayoutStore adapter plugs into #76 contract |
| #75 | Drag-and-drop panel rearrangement | L | High | future epic |
| #77 | Floating/popout panels | M | High | detach into separate windows |

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Previous: `git show HEAD~1:HANDOFF.md`
