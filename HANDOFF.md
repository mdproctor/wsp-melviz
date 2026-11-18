# HANDOFF — casehub-pages

## Status

On main. Two issues closed this session (#704, #706). Queue has 1 open issue (#705). All work published.

## What shipped this session

**#704 — Split/swap transaction wrapping.** Wrapped `_splitItem` (setPlacement + addItem) and `_swapItems` (setPlacement × 2) in `beginTransaction`/`commitTransaction` with try/catch abort. Added undo tests for both. This completes the transaction audit — every compound operation in `GridEditBridge` now has atomic undo.

**#706 — Guard module-scope document access.** `shared-timer-controller.ts` called `document.addEventListener` at module load time, causing `ReferenceError` in non-DOM test environments. All three `document` references now guarded with `typeof document !== 'undefined'`. Root cause fix — the `// @vitest-environment jsdom` annotations added in #701/#702 are no longer needed for this import chain (but remain since those test files also use `document` directly).

## Transaction audit — complete

All compound operations in `GridEditBridge.applyGridEdit` are now wrapped:

| Operation | Issue | Mutations |
|-----------|-------|-----------|
| move/resize | #698 | N × setPlacement + compaction |
| merge | #701 | N × removeItem + setPlacement + compaction |
| remove | #701 | removeItem + compaction |
| split | #704 | setPlacement + addItem |
| swap | #704 | setPlacement × 2 |

Single-mutation operations (insert, add-empty) need no wrapping.

## Queue — next session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #705 | gridPath routing for non-move operations | M | Med | All ops except move/resize ignore gridPath — design decision needed |

#705 requires a design decision: implement full YAML-level routing for all operations (merge, remove, split, swap, insert target nested grid), or make the API boundary explicit (remove `gridPath` param from operations that don't support it, throw if unexpectedly provided). Option 2 is safer for now.

## Notes

- Blog revised: "When Improvements Break Undo" — all operations now green in the SVG audit diagram
- All commits published to origin
- `yarn typecheck --build --clean && yarn typecheck` for a clean typecheck (0 errors)

## References

- Blog: `docs/blog/2026-11-18-mdp05-when-improvements-break-undo.md`
- Builder demo: `yarn --cwd packages/pages-builder dev` → http://localhost:5173
- Gallery dev server: `yarn --cwd examples dev` → http://localhost:8081
