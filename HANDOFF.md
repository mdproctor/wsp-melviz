# HANDOFF — casehub-pages

## Last Session

Fixed visual selection, hover, drag handle anchoring, and cross-grid DnD for nested grids. 25 commits on `issue-639-nested-grid-dnd-cross-grid-drag`. Core infrastructure (coordinator, path-routed bridge, cross-grid lifecycle) was already built — this session wired it end-to-end and fixed every visual/interaction bug exposed by Playwright testing with real mouse events.

Key decisions: theme system wired via `pages-ui-tokens` (replaces hardcoded hex); panel title rendered as flex-column header above grid wrapper; selection overlay uses path-based DOM index (not type-counting); cross-grid displace shows a floating ghost rectangle (not a grid placeholder); cross-grid swap highlights the target item with same style as within-grid swap.

Critical lesson: Vite caches `pages-component` in `node_modules/.vite/deps/` — after editing controller/coordinator source, **must** `rm -rf packages/pages-builder/node_modules/.vite` or changes don't take effect. This burned significant debugging time.

## How to Run the Example

```bash
yarn --cwd packages/pages-builder dev
# Opens http://localhost:5176 — Visual mode shows nested grid layout
# If Vite serves stale code: rm -rf packages/pages-builder/node_modules/.vite
```

Demo YAML (`packages/pages-builder/demo/main.ts`): 12-col root grid with Revenue Overview, Markdown, a Panel containing a 6-col nested grid (Nested A, B, C), Panel D, and a full-width item. Theme injected via `injectTheme(DEFAULT_THEME)` + `applyThemeMode(document.documentElement, 'dark')`.

## Architecture — How Cross-Grid Drag Works

1. **GridEditController** (`grid-edit-controller.ts`) — attached to each grid element via `activation.ts`. Handles within-grid drag/resize/swap. On `handlePointerMove`, checks if pointer left grid bounds OR entered a deeper grid (`coord.deepestGridAt`). If so, sets `_crossGridActive` and delegates to coordinator.

2. **GridEditCoordinator** (`grid-edit-coordinator.ts`) — singleton. Tracks all controllers in a parent-child tree built from DOM containment. Cross-grid drag lifecycle: `startCrossGridDrag` → `handleCrossGridMove` (shows swap highlight or ghost) → `completeCrossGridDrag` (returns `CrossGridEditDetail` with source/target paths and optional `swapTargetIndex`).

3. **GridEditBridge** (`grid-edit-bridge.ts`) — `applyCrossGridEdit` mutates YAML. With `swapTargetIndex`: two-way exchange (delete both, append each to the other's grid). Without: one-way move (delete from source, append to target).

4. **Builder Shell** (`builder-shell.ts`) — listens for `pages-grid-cross-edit` (composed, bubbles out of shadow DOM). Uses `'toolbar'` origin so `_syncViews` triggers `_refreshPreview()` (unlike within-grid edits which skip preview refresh). Clears stale selection after cross-grid edits, then selects the moved item at its new path.

5. **Panel grid wrapper** — `render.ts` wraps panel grid items in a `[data-grid-wrapper]` div inside a flex-column panel. `activation.ts` uses `el.querySelector('[data-grid-wrapper]')` to attach the controller to the wrapper, not the flex container.

## Known Issues for Next Session

- **Tree path mismatch after swap** — tree sorts nodes by position but `_selectedPath` uses YAML array index. After cross-grid swap, the tree highlights the wrong node. `_resolveClickedPath` (visual→path) works; the tree's path→node mapping needs the same fix.
- **Cross-grid displace doesn't compact target items** — the ghost follows the cursor but items in the target grid don't move out of the way. Would need the target controller to run `compactAll` for a virtual foreign item.
- **5 pre-existing builder test failures** — in palette (inline-picker, position-picker, palette-filter), tree (builder-tree-grid, builder-tree insertion points). Not caused by this session's changes.
- **`pages-runtime/dist/activation.js` needs rebuild** — source fix for `[data-grid-wrapper]` lookup was manually patched in dist but dist is gitignored. A proper build step is needed.

## Immediate Next Step

Fix the tree path-to-node mismatch so tree selection syncs correctly after cross-grid swaps.

## References

- Issue: #639 (nested grid DnD), #652 (tree-visual linkage)
- Plan: `plans/2026-11-10-nested-grid-dnd-cross-grid-drag.md`
- Demo: `packages/pages-builder/demo/main.ts`
- Coordinator: `packages/pages-component/src/renderer/grid-edit-coordinator.ts`
- Controller: `packages/pages-component/src/renderer/grid-edit-controller.ts`
- Bridge: `packages/pages-builder/src/shell/grid-edit-bridge.ts`
- Scope collector: `packages/pages-builder/src/shell/scope-collector.ts`
- Render (panel wrapper): `packages/pages-component/src/renderer/render.ts`
