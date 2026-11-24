# Fixed-Width Panel Auto-Size Column Allocation

**Issue:** casehubio/casehub-pages#619
**Scale:** S | **Complexity:** Med
**Date:** 2026-11-24

## Problem

Panels with a fixed pixel width (e.g., 160px) occupy their full column span in the CSS Grid but render at their declared pixel width. This creates dead space between the panel's right edge and the next panel — the gap is inside the panel's grid allocation, not between grid tracks.

Example: Panel D at `(8, 0, 2, 1)` with `fixedWidth: 160` renders 160px inside a ~214px slot (2 columns at typical viewport), leaving ~54px of visual gap. Adjacent panels cannot expand into this space because the CSS Grid allocation (`grid-column: 9 / span 2`) claims both columns.

## Solution

Add an optional `fixedWidth` property to `GridPlacement`. At render time, measure the grid container's width, compute the minimum column span needed for the fixed pixel width, and use that for the CSS `grid-column` span.

### Formula

```
cellWidth = (containerWidth - gap × (columns - 1)) / columns
effectiveW = min(ceil((fixedWidth + gap) / (cellWidth + gap)), w)
```

Where:
- `containerWidth` is the grid container's rendered width (`getBoundingClientRect().width`)
- `gap` is the grid gap (CSS custom property `--pages-space-3`, default 12px)
- `columns` is the grid's column count (default 12)
- `w` is the declared column span from the YAML (acts as upper bound)

The formula accounts for inter-column gaps: a panel spanning `n` columns has `n * cellWidth + (n-1) * gap` available pixels. Adding `gap` to both numerator and denominator correctly solves for the minimum `n` such that `n * cellWidth + (n-1) * gap >= fixedWidth`. The `ceil` ensures integer column spans. The `min` clamps to the declared `w` so a fixed-width panel never grows beyond its YAML-declared span.

### CSS Applied

When `effectiveW < w`:
- `grid-column: (x+1) / span effectiveW` (instead of `span w`)
- `width: fixedWidth + 'px'`
- `justify-self: start`

When `effectiveW === w`:
- Normal rendering — no special CSS

## Changes by Layer

### 1. Type model — `types.ts`

Add `fixedWidth?: number` to both variants of the `GridPlacement` union:

```ts
export type GridPlacement =
  | { readonly mode: "positioned"; readonly x: number; readonly y: number;
      readonly w: number; readonly h: number; readonly fixedWidth?: number }
  | { readonly mode: "auto"; readonly w: number; readonly h: number;
      readonly fixedWidth?: number };
```

### 2. YAML parsing — `page-document.ts`

Update `GridItemNode.getPlacement()` to read `fixedWidth` from the `at` object:

```yaml
grid:
  - at: { x: 8, y: 0, w: 2, h: 1, fixedWidth: 160 }
    type: html
```

Update `GridItemNode.setPlacement()` to persist `fixedWidth` when present.

### 3. Renderer — `render.ts`

Before the item loop (after setting `gridTemplateColumns`), measure the container:

```ts
const containerWidth = gridTarget.getBoundingClientRect().width;
const cellWidth = (containerWidth - gapPx * (columns - 1)) / columns;
```

Pass `cellWidth` to a new helper that wraps `applyGridPlacement`:

```ts
function applyFixedWidthPlacement(
  element: HTMLElement,
  placement: GridPlacement,
  cellWidth: number,
): void
```

This helper calls `applyGridPlacement` with an adjusted `w` when `fixedWidth` is present, and sets `width` and `justify-self` on the element.

### 4. Grid placement — `grid.ts`

No change to `applyGridPlacement` itself — it stays a pure function that maps placement to CSS. The `effectiveW` computation and CSS overrides happen in the new helper in `render.ts`.

### 5. Edit bridge — `grid-edit-bridge.ts`

Preserve `fixedWidth` during all edit operations (move, resize, split, merge, swap, cross-grid move). Currently, the bridge constructs `at` objects with only `{ x, y, w, h }` — extend to include `fixedWidth` when the source item has it.

### 6. Compaction — `grid-compaction.ts`

No changes. Compaction operates on `Rect { x, y, w, h }` and only adjusts `y` (vertical compaction). The declared `w` is used for overlap detection, which is correct — it represents the maximum space the panel can occupy.

### 7. Edit controller — `grid-edit-controller.ts`

Out of scope for this issue. The edit controller already has `GridGeometry` with `cellWidth`, so a future enhancement could compute `effectiveW` during drag for visual accuracy. For now, dragging a fixed-width panel uses the declared `w`.

## Testing

1. **Unit: effectiveW computation** — test the clamping formula with various `fixedWidth`, `cellWidth`, and `w` values including edge cases (fixedWidth = 0, fixedWidth > container, cellWidth = 0)
2. **Unit: YAML round-trip** — parse `at: { x: 0, y: 0, w: 3, h: 1, fixedWidth: 160 }`, verify `fixedWidth` is preserved through get/set placement
3. **Unit: edit bridge preservation** — move/resize/split operations preserve `fixedWidth` on items that have it
4. **Integration: render output** — render a grid with a fixed-width panel, verify the CSS `grid-column` span reflects `effectiveW`, `width` is set, and `justify-self: start` is applied
5. **Visual: gallery sample** — add a fixed-width panel example to the grid editing gallery for manual verification

## Out of Scope

- Full sizing model (`widthMode`, `heightMode`, `fixedHeight`) — tracked as #743
- Edit controller integration (drag/resize with effectiveW) — future enhancement
- Responsive re-computation via ResizeObserver — initial implementation uses the measurement at render time; dynamic resize is a separate concern
- Grid scale (`grid-scale.ts`) interaction — `scaleWidth` operates on `w`, not `fixedWidth`; cross-grid scaling of fixed-width panels needs design

## References

- `packages/pages-component/src/renderer/render.ts:116-165` — grid render loop
- `packages/pages-component/src/renderer/grid.ts:1-14` — `applyGridPlacement`
- `packages/pages-component/src/renderer/grid-drag.ts:21-36` — `computeGridGeometry`
- `packages/pages-component/src/model/types.ts:20-27` — `GridPlacement`, `GridItem`
- `packages/pages-document/src/page-document.ts:940-972` — `getPlacement`, `setPlacement`
- `packages/pages-builder/src/shell/grid-edit-bridge.ts:60-94` — `_applyPlacements`
- POC: branch `issue-618`, `examples/poc/grid-drag-poc.html:496` — panel D fixedWidth prototype
- Decision D8: sizing modes
