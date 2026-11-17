# HANDOFF — casehub-pages

## Status

On main. Queue has 3 issues for next session. All published.

## What shipped this session

**#674 — Remove old rows/columns grid model.** Complete removal from TS runtime across 6 packages. ~1,050 lines deleted, 7 YAML files migrated.

**#680 — Restore 25 gallery samples.** The #640 YAML format migration had silently stripped nested content (tabs, splits, docks, schema-forms) from 25 samples. All restored from pre-migration git history with correct `grid:` format.

**#678 — Navigation Rebinding title.** Empty grid item at index page header — added title component.

**#679 — Grid edit handles bleeding into gallery.** Runtime activation auto-attached grid edit controllers to every positioned grid. Fixed: now gates on `data-grid-editing-enabled` attribute. Builder and standalone pages unaffected (they call `attachGridEditController` directly).

**#677 — Dock workbench visual verification.** Confirmed working — `rows()`/`columns()` → `type: "panel"` + inline styles renders correctly.

## Queue — next session

| Priority | # | Title | Scale | Complexity | Notes |
|----------|---|-------|-------|------------|-------|
| 1 | #681 | Dataset scoping for targetPage tabs | S | Med | Runtime bug — datasets at document root not found by virtual tab pages. Blocks multiple gallery samples. |
| 2 | #676 | Paste indicators for grid/items mode | S | Low | `yaml-paste-indicators.ts` only handles flat mode. Pre-existing gap. |
| 3 | #675 | Migrate examples repo .dash.yaml files | S | Low | 19 domain samples in external repo still on old format. Mechanical. |

## Other open bugs worth tackling

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #682 | Verify Carousel and Stack sample renders | XS | May be blocked by #681 |
| #663 | Selection overlay tests need DOM environment | S | 6 pre-existing jsdom failures |
| #661 | Tree selection doesn't show grid-item controls | S | Tree click doesn't trigger controller |
| #651 | Builder chrome poor contrast | S | Hardcoded colors vs design tokens |
| #670 | Clean up orphaned selection overlay event handlers | XS | Dead code from split/merge refactor |

## References

- Blog: `docs/blog/2026-11-17-mdp02-finishing-what-the-grid-started.md`
- Design spec: `docs/specs/issue-674-remove-old-grid-model/`
- Gallery dev server: `yarn --cwd examples dev` → http://localhost:8083
- Builder demo: `yarn --cwd packages/pages-builder dev` → http://localhost:5173
