# HANDOFF — casehub-pages

## Status

On main. Three issues closed this session (#701–#703). Three new issues filed (#704–#706). Queue has 2 open issues. All work published.

## What shipped this session

**#701 — Merge/remove transaction wrapping.** Wrapped `_mergeItems` and `_removeItem` in `beginTransaction`/`commitTransaction` with try/catch abort, same pattern as the move/resize fix in #698. Added undo tests for both operations. Added `// @vitest-environment jsdom` to `grid-edit-bridge.test.ts` and `builder-shell-grid.test.ts` (pre-existing breakage from `shared-timer-controller.ts` calling `document.addEventListener` at module load time).

**#702 — Builder tree CSS test fix.** Test asserted `opacity: 0` on `.add-btn`, but the CSS applies it to the parent `.tree-actions` container. Updated selector in test. Added `// @vitest-environment jsdom` to `builder-tree.test.ts`.

**#703 — Nested grid compaction target.** When `gridPath` targets a nested grid, `_applyPlacements` correctly manipulates the nested YAML but `_compactGrid` ran on the root page. Fixed by skipping root compaction when `gridPath` is set. Added test proving root grid gaps are preserved during nested grid edits.

## Code review findings (filed as issues)

**#704 — Split/swap transaction wrapping.** `_splitItem` (setPlacement + addItem) and `_swapItems` (setPlacement × 2) both have the same multi-undo-entry problem. XS/Low — same pattern as #698/#701.

**#705 — gridPath routing for non-move operations.** All operations except move/resize accept `gridPath` but ignore it (prefixed with `_`). They always target the root grid regardless of which nested grid the user is editing. M/Med — requires YAML-level reimplementation or explicit API boundary.

**#706 — Builder test jsdom annotations.** `shared-timer-controller.ts` calls `document.addEventListener` at module load time. Any test importing from `@casehubio/pages-component` needs `// @vitest-environment jsdom`. Root cause fix: guard with `typeof document !== 'undefined'`. XS/Low.

## Queue — next session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #705 | gridPath routing for non-move operations | M | Med | YAML-level operation routing or explicit API boundary |
| #706 | Builder test jsdom annotations | XS | Low | Guard module-scope document call |

#704 (split/swap transactions) was checked off by the ceremony but not implemented — it needs to be reopened or the work done.

## Notes

- Blog revised: "When Improvements Break Undo" — extended with the full transaction audit, new SVG diagram showing all grid operations' transaction status
- All commits published to origin
- `yarn typecheck --build --clean && yarn typecheck` for a clean typecheck (0 errors)
- Pre-existing test failure in builder-tree.test.ts now fixed (#702)

## References

- Blog: `docs/blog/2026-11-18-mdp05-when-improvements-break-undo.md`
- Builder demo: `yarn --cwd packages/pages-builder dev` → http://localhost:5173
- Gallery dev server: `yarn --cwd examples dev` → http://localhost:8081
