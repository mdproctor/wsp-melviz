# HANDOFF — casehub-pages (slot 218)

## Status: CLOSED — ready for archive

All work in this slot is complete and landed on main.

## Completed Issues

### #390 — Scenario format refinements (TS side)
Extracted dispatch protocol types to `playbook/types.ts`. Renamed CommandPayload.target to element, added DispatchStep.delay, made speed opt-in, added REST action definition. Landed on main.

### #531 — Inferred method-call dispatch for playbook steps
Added secure inferred dispatch so unrecognised step names resolve as method calls on target elements (TS) or beans (Java). Key deliverables:
- `PLAYBOOK_ACTIONS` Symbol SPI in pages-primitives
- `Catalog.resolveInferred()` in yaml-core (Walker accepts unknown step keys)
- Secure `invokePlaybookMethod` replacing raw `invokeMethod` in command-executor.ts
- `InferredMethodHandler` + catalog factory wiring in pages-aria
- `playbook-handler.ts` default case extended for server-dispatched path
- `pages-document-diff` as first adopter (9 methods declared)
- `@PlaybookInterface` annotation + `ActionRegistry` fallback on Java side
- Post-audit fix: `ElementMethodBinding` export, `executeAriaAction` made async
Landed on main, issue closed.

## Design Artifacts (workspace)

- `specs/issue-531-inferred-method-dispatch/` — design spec, decisions, pipeline state
- `plans/2026-10-20-inferred-method-dispatch.md` — implementation plan
- `specs/issue-571-decorator-structural-keys/` — prior work spec
- `plans/2026-10-20-scenario-format-refinements.md` — prior work plan

## References

- `packages/pages-primitives/src/playbook/` — PLAYBOOK_ACTIONS SPI
- `packages/pages-aria/src/executor/playbook-resolve.ts` — shared dispatch utilities
- `packages/pages-aria/src/playbook/invoke/inferred-method-handler.ts` — InferredMethodHandler
- `packages/pages-aria/src/playbook/catalog-factory.ts` — resolveInferred on catalog
- `packages/pages-document-diff/src/pages-document-diff.ts` — first PLAYBOOK_ACTIONS adopter
- `backend/playbook-client/src/main/java/.../PlaybookInterface.java` — Java annotation
- `backend/playbook-client/src/main/java/.../ActionRegistry.java` — interface scanning fallback

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
