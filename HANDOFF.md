# HANDOFF — casehub-pages

## Status

On main. #674 (remove old rows/columns grid model) landed and closed. #665 (grid editing demo) was already closed in a prior session.

## What just shipped

**#674 — Remove old rows/columns grid model.** Complete removal of the old `rows:` / `columns:` page-level layout and `type: "columns"` / `type: "rows"` layout components from the TypeScript runtime. 8 commits compacted to 4 via semantic squash.

Key changes:
- 7 YAML files migrated from rows/columns to `grid:` with `at: {x, y, w, h}` placement
- `RowNode`, `ColumnNode` classes deleted from pages-document
- `LayoutMode` narrowed from 5 values to 3 (`'grid' | 'flat' | 'items'`)
- Schema validators (`rowSchema`, `columnSchema`) and type guards (`isColumns`, `isRows`) removed
- Columns desugaring branch removed from parser
- `grid-migration.ts` migration script deleted (job done)
- Builder: row/column tree nodes, scope collection, paste indicators, path resolution, property panels all removed
- Dock-workbench DSL helpers (`rows()`, `columns()`) converted from exported layout-type functions to local panel+inline-style helpers
- Component catalog cleaned: `LEGACY_LAYOUT_TYPES`, `layoutRelevance`, `LAYOUT_BLOCKED_PARENTS` dead code removed

## Filed Issues

| # | Title | Notes |
|---|-------|-------|
| #675 | Migrate examples repo .dash.yaml files from rows/columns to grid format | 19 domain samples in external repo |
| #676 | Paste indicators for grid/items layout mode in code editor | Pre-existing limitation surfaced during review |
| #677 | Visual verification: dock-workbench after rows/columns removal | Inline-style helpers untested visually |

## Pre-existing test failures

6 tests in `selection-overlay.test.ts` fail due to jsdom not rendering custom element shadow DOM. Not caused by #674. Tracked in #663.

## References

- Design spec: `docs/specs/issue-674-remove-old-grid-model/2026-11-17-remove-old-grid-model-design.md`
- Decisions: `docs/specs/issue-674-remove-old-grid-model/decisions.md`
- Blog: `docs/blog/2026-11-17-mdp02-finishing-what-the-grid-started.md`
- Prior: #625 (unified grid layout model — the design decision this implements)
