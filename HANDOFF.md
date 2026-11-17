# HANDOFF — casehub-pages

## Status

On main. Branch `issue-670-cleanup-orphaned-overlay-handlers` closed, 4 issues landed. Queue has 6 open issues + 3 newly filed.

## What shipped this session

**#670 — Remove orphaned overlay handlers.** Deleted 92 lines of dead `onSplitH`/`onSplitV`/`onMerge` handlers and 6 test assertions for buttons that `ALL_BUTTONS` never contained. No code dispatched `selection-split-h`, `selection-split-v`, or `selection-merge` after the #665 hover bar refactor.

**#663 — DOM environment for tests.** Added `// @vitest-environment jsdom` to `grid-edit-controller.test.ts` and `selection-overlay.test.ts`. Both had `environment: 'jsdom'` in their package vitest config but failed from the repo root (no root config). Now 100 tests pass from any context.

**#661 — Tree-to-controller coordination.** Tree selection of `grid-item` now shows the controller's handle bar (drag handles, ⋮ trigger, action buttons). Added `pages-grid-select` custom event: builder-shell dispatches it in `_applyPreviewHighlight`, controller listens in both `enterPassive()` and `enter()`.

**#651 — Builder chrome contrast.** Replaced hardcoded hex colors (`#fff`, `#5f6368`, `#dadce0`, `#1967d2`) with `--pages-*` design tokens in tree view, selection overlay, and overlay toolbar. Annotation badges use `neutral-8` at full opacity (was `neutral-11` at 0.7). Preview grid items now have `outline: 1px dashed var(--pages-neutral-5)` for visual item boundaries.

## Issues filed this session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #683 | Preview outline toggle | XS | Low | Make `[data-item-id]` outlines toggleable |
| #684 | Pre-existing type errors from rows/columns removal | S | Low | 20 `"column"`/`"row"` TreeNodeType refs in tests and yaml-path.ts |
| #685 | pages-grid-select selection model gap | XS | Low | External select doesn't update multi-selection state |

## Queue — next session

| Priority | # | Title | Scale | Complexity | Notes |
|----------|---|-------|-------|------------|-------|
| 1 | #681 | Dataset scoping for targetPage tabs | M | Med | Runtime bug — datasets at document root not found by virtual tab pages. Blocks #682. |
| 2 | #676 | Paste indicators for grid/items mode | M | Med | `yaml-paste-indicators.ts` only handles flat mode. |
| 3 | #675 | Migrate examples repo .dash.yaml files | S | Med | 19 domain samples in external repo. Cross-repo. |
| 4 | #682 | Verify Carousel/Stack sample renders | XS | Low | Blocked by #681. |
| 5 | #684 | Pre-existing type errors cleanup | S | Low | Blocks `tsc --noEmit` on pages-builder/pages-component. |
| 6 | #685 | pages-grid-select selection model | XS | Low | Multi-select after tree selection. |

## References

- Blog: `docs/blog/2026-11-17-mdp03-the-small-things.md`
- Previous blog: `docs/blog/2026-11-17-mdp02-finishing-what-the-grid-started.md`
- Builder demo: `yarn --cwd packages/pages-builder dev` → http://localhost:5173
- Gallery dev server: `yarn --cwd examples dev` → http://localhost:8083
