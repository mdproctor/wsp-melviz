# HANDOFF — casehub-pages

## Status

On main. Branch `issue-674-remove-old-rows-columns-grid` closed. Five issues landed this session. Queue has 8 open issues (5 pre-existing + 3 newly filed).

## What shipped this session

**#674 — Remove old rows/columns grid model (final remnant).** Only a stale `ColumnsProps` test import remained in `component-props.test.ts`. Removed. Stale `.typecheck` dirs cleaned.

**#676 — Paste indicators for grid/items layout mode.** `computeValidPasteLines()` now generates sibling before/after entries for grid items in both `items:` and `grid:` page formats. 4 new tests.

**#675 — Migrate examples repo dash.yaml files.** 23 `.dash.yaml` files in `casehubio/examples` converted from `rows:/columns:/span:` to `grid:` with `at: {x,y,w,h}`. Cross-repo commit `e1e141c6` on examples main.

**#682 — Verify Carousel and Stack sample renders.** Confirmed the sample works after #681's targetPage fix. 2 Playwright tests added to gallery.spec.ts.

**#663 — DOM environment for renderer tests.** Added `// @vitest-environment jsdom` to 9 test files in `pages-component/src/renderer/`. 560 tests now pass (was 288 failing).

## Cross-repo changes

| Repo | Branch | Commit | What |
|------|--------|--------|------|
| `casehubio/examples` | main | `e1e141c6` | 23 dash.yaml files migrated from rows/columns to grid format |

## Issues filed this session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #688 | Stale .typecheck directories contain removed types | XS | Low | IntelliJ indexes ghost results |
| #689 | Gallery: Carousel test doesn't verify data in slides | XS | Low | Test coverage gap |
| #690 | examples repo: yaml library reformats string scalars | XS | Low | Cosmetic — semantically correct |

## Queue — next session

| Priority | # | Title | Scale | Complexity | Notes |
|----------|---|-------|-------|------------|-------|
| 1 | #661 | Tree selection doesn't show grid-item controls | S | Med | Overlay/controller coordination |
| 2 | #651 | Builder chrome poor contrast | S | Low | Design token fix |
| 3 | #670 | Clean up orphaned selection overlay event handlers | S | Low | Dead code removal |
| 4 | #688 | Stale .typecheck directories | XS | Low | New this session |
| 5 | #689 | Gallery Carousel test gaps | XS | Low | New this session |
| 6 | #690 | YAML roundtrip formatting | XS | Low | New this session |

## References

- Blog: `blog/2026-11-17-mdp02-finishing-what-the-grid-started.md` (revised this session)
- Builder demo: `yarn --cwd packages/pages-builder dev` → http://localhost:5173
- Gallery dev server: `yarn --cwd examples dev` → http://localhost:8081
