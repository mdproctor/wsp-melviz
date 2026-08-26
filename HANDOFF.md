# HANDOFF — casehub-pages

Branch: `main` (direct-to-main fixes)
Issue: #367, #368, #369 (closed), #376 (filed)

## Last Session

Visual testing of the engine elimination (branch `issue-367-container-toolbar-unification`, closed previous session). Launched the examples gallery in Playwright and found 19 regressions + missing features. All fixed via TDD directly on main: nested workspace height collapse, double toolbars, cross-workspace toolbar leaking, toolbar depth suppression, nest button restoration, edge split creating peer frames instead of split containers, tab drag gap preview, within-strip reorder triggering cross-frame DnD, cross-frame drop position/activation, stale preview cleanup, pane-level split scoping, layout cycle including split modes, root toolbar made inline (recursively regular), and auto-re-arrange on addEntry. Also cleaned stale `dist/` artifacts from deleted source files.

## Immediate Next Step

ARC42STORIES.MD is stale — test counts (shows 183, actual 957), missing packages (pages-aria, pages-table), container model not documented. File an issue or update directly.

## References

- Filed: #376 — dual-zone edge split scoping (pane vs frame level)
- Spec: `specs/issue-367-container-toolbar-unification/2026-08-25-workspace-as-container-design.md`
- Plan: `plans/2026-08-25-workspace-as-container.md`
