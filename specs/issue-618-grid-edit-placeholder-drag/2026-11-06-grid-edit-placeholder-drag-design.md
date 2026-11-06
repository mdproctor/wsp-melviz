# Grid Edit UX: Placeholder-Based Drag with Vertical Compaction

**Issue:** #618
**Branch:** `issue-618-grid-edit-placeholder-drag`
**Date:** 2026-11-06

## Problem

The grid edit controller (#615) uses a pixel-positioned ghost overlay during drag operations. The ghost is absolutely positioned (`position: absolute` with `top`/`left`/`width`/`height` in pixels) and floats above the CSS Grid layout. This creates four UX failures:

1. No placeholder showing where the panel will land relative to other panels
2. Unpredictable collision resolution — panels push in arbitrary directions
3. No drag handle — entire panel is the drag target, causing accidental drags
4. Ghost overlay has no relationship to the visible panel boundaries

## Design

### Layout Model (Unchanged)

The existing placement model stays: `{ mode: positioned, x, y, w, h }` on a 12-column CSS Grid, rendered by `applyGridPlacement` in `grid.ts`:

```typescript
element.style.gridColumn = `${x + 1} / span ${w}`;
element.style.gridRow    = `${y + 1} / span ${h}`;
```

All coordinates are integers. The Grid Spanning showcase demonstrates that this model handles all spanning combinations natively. No absolute positioning is used for grid items.

### Placeholder — A Structural Grid Item

During drag, a **placeholder div** is inserted into the CSS Grid at the target `{x, y, w, h}`. The placeholder is a real grid participant — it has `grid-column` and `grid-row` set via `applyGridPlacement`, so it displaces other items naturally through CSS Grid flow.

As the cursor moves and snaps to new grid cells, the placeholder's `grid-column`/`grid-row` are updated. The browser's grid engine handles the visual reflow. No pixel math is needed.

The absolutely-positioned ghost overlay (`grid-overlay.ts:45-49`) is removed entirely. The overlay layer (`grid-overlay.ts`) is retained only for the edit button, insertion zones, and drag handle indicators.

**Placeholder styling:** dashed border, semi-transparent background, matching the target item's dimensions. The dragged item itself stays in its original position at reduced opacity (0.4) until drop.

### Drag Handle

Only the top ~24px of a panel triggers drag (the title bar area). The rest of the panel is for content interaction.

- Cursor changes to `grab` only when hovering the title bar strip
- A subtle drag handle icon (⠿) appears on hover in edit mode
- Panels without a visible title bar get a thin drag indicator strip at the top

### Compaction Algorithm

Vertical compaction with upward gravity. Three steps on every snap position change:

1. **Place** — set the dragged item's placement to the cursor's snapped `{x, y, w, h}`
2. **Push** — for each non-dragged item that overlaps the dragged item, set its `y = dragged.y + dragged.h` (push it below)
3. **Compact** — for each item (top to bottom by y), move it to the lowest `y` where it doesn't overlap anything above it

All operations are integer arithmetic on `{x, y, w, h}` tuples. Overlap detection: two items overlap when their column ranges intersect AND their row ranges intersect.

```
columnsOverlap = (a.x < b.x + b.w) && (b.x < a.x + a.w)
rowsOverlap    = (a.y < b.y + b.h) && (b.y < a.y + a.h)
overlap        = columnsOverlap && rowsOverlap
```

Horizontal gaps within a row are not compacted — column positions are intentional.

### Collision Resolution

A new `collision` property on the grid layout YAML controls behavior when the placeholder lands on an occupied cell:

#### `collision: displace` (dashboard-style)

Automatic displacement — the occupied item is pushed to the next valid row, compaction runs, no user prompt. This is the expected behavior for uniform dashboard grids.

#### `collision: prompt` (page-builder-style, default)

A popup appears at the drop position offering four options:

- **Displace** — push the occupied item down (always valid, even with spans)
- **Split** — subdivide the target cell to fit both items. Direction options:
  - Horizontal (stack vertically): valid when target `h > 1` or grid can grow rows
  - Vertical (side by side): valid when target `w > 1`
  - A 1×1 cell cannot split (option hidden)
- **Replace** — dragged item takes target's `{x, y, w, h}`, target is removed
- **Swap** — exchange positions: dragged item gets target's placement, target gets dragged item's original placement

### Visual Feedback Progression

| State | Visual |
|-------|--------|
| Edit mode entered | All panels get dashed outlines; grab cursor on title bar |
| Hover | Hovered panel gets solid blue outline (2px) |
| Drag started | Dragged panel fades to 40% opacity in place; placeholder appears at snap position |
| During drag | Placeholder moves as cursor snaps; other panels animate smoothly via CSS transitions |
| Drop (empty cell) | Panel slides to placeholder position; outlines return to dashed |
| Drop (occupied, displace mode) | Displaced items animate to new positions |
| Drop (occupied, prompt mode) | Popup appears; action executes on selection |

### Files Changed

| File | Change |
|------|--------|
| `grid-edit-controller.ts` | Replace ghost-based drag with placeholder insertion/update |
| `grid-collision.ts` | Replace `resolveCollisions` with compaction algorithm (place/push/compact) |
| `grid-overlay.ts` | Remove ghost creation (`createGhostElement`); keep edit button and insertion zones |
| `grid-drag.ts` | Remove `cellRectToPixel`; keep `snapToCell`, `computeGridGeometry` |
| `types.ts` | Add `collision?: 'displace' \| 'prompt'` to grid layout type |
| `render.ts` | Read `collision` property from YAML |
| New: `grid-compaction.ts` | Pure-function compaction algorithm (place, push, compact) |
| New: `grid-collision-popup.ts` | Popup UI for split/replace/swap/displace |

### What Is Preserved

From #614 and #615, these are solid and unchanged:

- `GridItemNode` document model (`page-document.ts`)
- `GridEditBridge` event handler (`grid-edit-bridge.ts`)
- `pages-grid-edit` event contract
- Tree view with placement annotations
- Builder-shell wiring (`_applyEdit('grid', ...)` + preview skip)
- `collectComponentsInScope` items support
- Format conversion (components/rows/columns → items)
- `applyGridPlacement` in `grid.ts`
- `snapToCell` and `computeGridGeometry` in `grid-drag.ts`

### Testing Strategy

- **Unit tests:** Compaction algorithm — place/push/compact with various overlap scenarios, spanning items, edge cases (item pushed off bottom, 1×1 items, full-width items)
- **Unit tests:** Collision detection — overlap function with all span combinations
- **Unit tests:** Split mechanics — valid directions per cell geometry, resulting placements
- **Integration tests:** Drag sequence → verify final `{x, y, w, h}` for all items
- **Gallery sample:** Update Grid Builder sample to demonstrate both `collision: displace` and `collision: prompt` modes
- **Visual test:** Headed Playwright — drag a panel, verify placeholder appears at correct position, verify other panels reflow

## References

- `packages/pages-component/src/renderer/grid.ts` — `applyGridPlacement` (the CSS Grid translation)
- `packages/pages-component/src/renderer/grid-edit-controller.ts` — current drag controller (to be replaced)
- `packages/pages-component/src/renderer/grid-overlay.ts:45-49` — current ghost (to be removed)
- `packages/pages-component/src/renderer/grid-drag.ts:77-91` — `cellRectToPixel` (to be removed)
- `packages/pages-component/src/renderer/grid-collision.ts` — current `resolveCollisions` (to be replaced)
- `examples/samples/Layout/Grid Spanning.page.yaml` — showcase demonstrating the placement model
- Issue #618 — problem statement and user feedback
- DashBuilder/Melviz GWT layout editor — prior art for table-based drag with column/row spans
