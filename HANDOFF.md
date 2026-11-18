# HANDOFF — casehub-pages

## Status

On main. Four issues closed this session (#697–#700). Three new issues filed (#701–#703). Queue has 3 open issues. 4 commits unpublished on main.

## What shipped this session

**#697 — Pre-existing typecheck errors.** Added `'row' | 'column'` to `TreeNodeType` (yaml-path returns them, type didn't declare them). Exported `rows`/`columns` from builders DSL and `getButtonsForNodeType` from selection-overlay. Typecheck clean: 488 stale errors → 32 real → 0.

**#698 — Grid edit undo broken by compaction.** Wrapped move/resize in `beginTransaction`/`commitTransaction` so compaction + placement are a single atomic undo entry. Added try/catch with `abortTransaction` for error safety. Updated 3 test expectations to reflect compaction pulling items to y:0. Blog: "When Improvements Break Undo".

**#699 — Carousel data-table slide test.** Added Playwright test: click next twice to reach slide 3, verify `pages-data-table` renders with table rows.

**#700 — YAML formatting differences.** Closed as already resolved — audited 119 .page.yaml files, none of the described formatting issues exist.

## Code review findings (filed as issues)

**#701 — Merge/remove transaction wrapping.** `_mergeItems` and `_removeItem` both call `_compactGrid` — same multi-undo problem as move/resize. No undo tests exist for either. XS/Low.

**#702 — Builder tree CSS test fix.** Pre-existing failure: test expects `opacity: 0` on `.add-btn` but CSS applies it on parent `.tree-actions`. XS/Low.

**#703 — Nested grid compaction target.** When `gridPath` is set, `_applyPlacements` manipulates the nested YAML but `_compactGrid` runs on root page. Semantically wrong, currently harmless. S/Low.

## Queue — next session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #701 | Merge/remove transaction wrapping | XS | Low | Same pattern as #698 fix |
| #702 | Builder tree CSS test fix | XS | Low | .add-btn → .tree-actions opacity |
| #703 | Nested grid compaction target | S | Low | gridPath routes to wrong grid |

## Notes

- 4 commits on main ahead of origin — not yet published
- `yarn typecheck --build --clean && yarn typecheck` for a clean typecheck (0 errors)
- `yarn workspace @casehubio/pages-ui build` required before typecheck if `dist/` is stale (examples import from dist path)
- Pre-existing test failure: `add-btn stays in layout flow` in builder-tree.test.ts — tracked as #702

## References

- Blog: `blog/2026-11-18-mdp05-when-improvements-break-undo.md`
- Blog: `blog/2026-11-18-mdp03-the-ghosts-in-typecheck.md`
- Blog: `blog/2026-11-18-mdp04-present-but-not-proven.md`
- Builder demo: `yarn --cwd packages/pages-builder dev` → http://localhost:5173
- Gallery dev server: `yarn --cwd examples dev` → http://localhost:8081
