# HANDOFF — casehub-pages

## Status

On branch `issue-674-remove-old-rows-columns-grid`. Queue position 5/10, active issue #675. Three issues completed this session (#665 was pre-done, #674 was 3-line fix, #676 was 8-line feature).

## What shipped this session

**#674 — Remove old rows/columns grid model.** Nearly all removal was already done in prior sessions. Only remaining source artifact was a stale `ColumnsProps` test import in `component-props.test.ts`. Removed the import and test block. Cleaned stale `.typecheck` directories.

**#676 — Paste indicators for grid/items layout mode.** `computeValidPasteLines()` only generated sibling paste indicators for `mode === 'flat'` pages. Added `else if (mode === 'grid' || mode === 'items')` branch that iterates `page.getItems()` and generates before/after entries for each grid item. 4 new tests covering both `items:` and `grid:` formats.

**#675 — Migrate examples repo dash.yaml files.** Mechanical conversion of 23 `.dash.yaml`/`.dash.yml` files in `casehubio/examples` from old `rows:/columns:/span:` layout to `grid:` with `at: {x,y,w,h}` placement. Displayer components converted to new type/properties format. Committed to examples repo main as `e1e141c6`.

## Cross-repo changes

| Repo | Branch | Commit | What |
|------|--------|--------|------|
| `casehubio/examples` | main | `e1e141c6` | 23 dash.yaml files migrated from rows/columns to grid format |

## Queue — next session

| Priority | # | Title | Scale | Complexity | Notes |
|----------|---|-------|-------|------------|-------|
| 1 | #682 | Verify Carousel/Stack sample renders | XS | Low | Was blocked by #681 — now unblocked |
| 2 | #663 | Selection overlay tests need DOM environment | S | Low | — |
| 3 | #661 | Tree selection doesn't show grid-item controls | — | — | — |
| 4 | #651 | Builder chrome poor contrast | — | — | — |
| 5 | #670 | Clean up orphaned selection overlay event handlers | — | — | — |

## References

- Builder demo: `yarn --cwd packages/pages-builder dev` → http://localhost:5173
- Gallery dev server: `yarn --cwd examples dev` → http://localhost:8081
