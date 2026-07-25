*Updated: #230 closed — removed from backlog.*

# casehub-pages Session Handover — 2026-07-25

## Last Session

Shipped 8 fixes/features across two sessions (23–25 Jul). Major items: `mutableRestSource` production write path (#144 — brainstorm → spec → plan → TDD implementation → work-end), GridTable `transpose` prop (#235), tree-table hierarchy-preserving client-filter (#240), GroupedView toggle fix (#239), `<pages-table>` alias (#242), dark theme defaults and token fixes (#243, #238, #226, #234). Fixed gallery CSS crash from missing webpack aliases. Stamped and closed 6 stale branches.

## Immediate Next Step

Gallery is running on casehub-dark. The 2 pre-existing `PagesChartElement` test failures (backgroundColor: "transparent" not expected in setOption mock) should be fixed — they're from #230's transparent chart background change.

## What's Left

- PagesChartElement test failures from #230 transparent background · XS · Low
- Garden push failing (auth/remote issue) — 1 entry committed locally · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #237 | Propagate form tag renames to downstream repos | M | Low | Cross-repo coordination |
| #236 | Rename blocks-ui components to blocks- prefix | M | Med | Naming convention sweep |
| #222 | Nested object/array schema support for schema-form | L | High | Recursive rendering |
| #180 | Remove // @ts-nocheck from example files | L | Low | 40 files, mechanical |
