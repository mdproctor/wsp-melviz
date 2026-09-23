# Selection-Based Visual Toolbar

**Issue:** #433 (Tree/visual structural editing)
**Branch:** issue-433-structural-editing
**Date:** 2026-09-18

## Overview

Replace the hover-based per-component overlay toolbar with a selection-based toolbar attached to the scope outline. A single trigger icon (`⋮`) appears on the selected scope outline's top-right corner. Hovering the icon expands to 5 action buttons. The existing `ComponentOverlayManager` is deleted — its overlay events are unwired in production code and the class is effectively dead.

## Architecture

### SelectionOverlay class

A new class replaces both the inline scope overlay in `_applyPreviewHighlight` and `ComponentOverlayManager`:

```typescript
// overlay/selection-overlay.ts
export class SelectionOverlay {
  constructor(overlayRoot: HTMLElement)
  update(bounds: DOMRect, nodeType: TreeNodeType,
         path: readonly (string | number)[]): void
  hide(): void
  setInsertMode(fragmentType: string): void
  clearInsertMode(): void
  dispose(): void
}
```

**Responsibilities:**
- Render the scope outline (blue border div) at the given bounds
- Render the trigger icon at top-right of the outline
- Expand to 5-button toolbar on hover (CSS-driven, no JS mousemove)
- Dispatch events on `overlayRoot` when buttons are clicked
- Manage insert-mode visual state (paste-target highlighting)

**What it does NOT do:**
- Compute bounds — builder-shell does this via `collectComponentsInScope` + `getBoundingClientRect`
- Track selection state — builder-shell owns `_selectedPath` / `_selectedNodeType`
- Execute actions — builder-shell handles the dispatched events

### Lifecycle

```
User clicks preview component
  → builder-shell._wirePreviewClicks sets _selectedPath/_selectedNodeType
  → builder-shell._updatePropertySource calls _applyPreviewHighlight
  → _applyPreviewHighlight computes bounds via collectComponentsInScope
  → If elements found: selectionOverlay.update(bounds, nodeType, path)
  → If no elements (dataset, nav-item, section): selectionOverlay.hide()
  → User hovers ⋮ icon: CSS expands button row
  → User clicks button: SelectionOverlay dispatches event
  → builder-shell handles event (reuses existing _handleTreeCut, etc.)
```

### DOM structure

SelectionOverlay creates and manages this DOM inside the overlay-root:

```html
<div class="builder-scope-overlay" style="position:absolute; ...bounds...">
  <div class="selection-toolbar" style="position:absolute; top:-2px; right:-2px;">
    <div class="toolbar-buttons">
      <button class="sel-add-btn">+</button>
      <button class="sel-insert-btn">↓</button>
      <button class="sel-cut-btn">✂</button>
      <button class="sel-copy-btn">⎘</button>
      <button class="sel-delete-btn">✕</button>
    </div>
    <div class="toolbar-trigger">⋮</div>
  </div>
</div>
```

The trigger is last in DOM order so it's rightmost. Buttons expand leftward from the trigger on hover.

### CSS-driven hover expansion

```css
.selection-toolbar {
  display: flex;
  align-items: center;
  pointer-events: auto;
  z-index: 11;
}

.toolbar-buttons {
  display: flex;
  gap: 2px;
  max-width: 0;
  overflow: hidden;
  opacity: 0;
  transition: max-width 150ms ease, opacity 150ms ease;
}

.selection-toolbar:hover .toolbar-buttons {
  max-width: 200px;
  opacity: 1;
}
```

No JavaScript mousemove/mouseleave tracking. The `.selection-toolbar` container wraps both trigger and buttons — CSS `:hover` on the container keeps buttons visible while the mouse moves between them.

### Events

SelectionOverlay dispatches on `overlayRoot`:

| Event | Detail |
|-------|--------|
| `selection-add` | `{ path, nodeType }` |
| `selection-insert` | `{ path, nodeType }` |
| `selection-cut` | `{ path, nodeType }` |
| `selection-copy` | `{ path, nodeType }` |
| `selection-delete` | `{ path, nodeType }` |

builder-shell wires these to existing handlers — the tree handlers already work on `path + nodeType` and can be reused directly.

### Button relevance by node type

| Button | page | row | column | component |
|--------|------|-----|--------|-----------|
| Add (+) | yes | yes | yes | container only |
| Insert (↓) | no | yes | yes | yes |
| Cut (✂) | no | yes | yes | yes |
| Copy (⎘) | no | yes | yes | yes |
| Delete (✕) | yes | yes | yes | yes |

SelectionOverlay receives `nodeType` in `update()` and conditionally renders buttons. Non-applicable buttons are omitted from the DOM (not just hidden).

For components: `Add` is shown only when the component is a container. builder-shell passes this via an optional `isContainer` flag in the update call:

```typescript
update(bounds: DOMRect, nodeType: TreeNodeType,
       path: readonly (string | number)[],
       options?: { isContainer?: boolean }): void
```

### Insert mode integration

When `setInsertMode(fragmentType)` is called, the scope outline gets the `paste-target` CSS class (same as current behavior). The outline border changes to accent color to indicate valid paste target.

When `clearInsertMode()` is called, the class is removed.

## Files changed

| File | Action |
|------|--------|
| `overlay/selection-overlay.ts` | **New** — SelectionOverlay class |
| `overlay/selection-overlay.test.ts` | **New** — unit tests |
| `overlay/component-overlay.ts` | **Delete** |
| `overlay/component-overlay.test.ts` | **Delete** |
| `shell/builder-shell.ts` | Modify — replace ComponentOverlayManager with SelectionOverlay, move scope outline rendering from `_applyPreviewHighlight` to `SelectionOverlay.update()`, wire `selection-*` events |

## Testing strategy

- Unit tests for `SelectionOverlay`: DOM creation, button rendering per node type, event dispatch, insert mode CSS, hover expansion (programmatic hover), hide/dispose cleanup
- Existing builder-shell tests continue to verify selection → highlight flow
- Integration: selection in preview → toolbar visible → button click → document mutation

## References

- `overlay/component-overlay.ts` — current overlay (to be deleted; events unwired)
- `shell/builder-shell.ts:354-405` — `_applyPreviewHighlight` (scope overlay management moves to SelectionOverlay)
- `shell/scope-collector.ts` — `collectComponentsInScope` (bounds computation stays in builder-shell)
- `clipboard/builder-clipboard.ts` — insert mode state
- `tree/builder-tree.ts` — tree button set (toolbar matches)
- HANDOFF.md — design decision: selection-based, not hover-based
- decisions.md D1-D4 — all design choices documented
