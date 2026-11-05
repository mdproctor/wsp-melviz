# Layout Spanning and Drag-and-Drop Panel Placement

Covers #613. Enhances the `grid` layout type with auto-flow spanning and adds
visual drag-and-drop layout editing across all layout types.

---

## 1. Architecture Overview

The enhanced `grid` layout type works in two modes via a discriminated union on
`GridPlacement`:

```typescript
type GridPlacement =
  | { mode: 'positioned'; x: number; y: number; w: number; h: number }
  | { mode: 'auto'; w: number; h: number };
```

- **Positioned mode** — existing behaviour: explicit `{x, y, w, h}` coordinates
  on the 12-column grid.
- **Auto mode** — new: item declares span size only (`w` columns, `h` rows),
  CSS `grid-auto-flow: row` places it automatically.

`GridProps` gains an optional `rowHeight?: number`. When absent, rows use
`min-content` (current behaviour). When present,
`grid-auto-rows: ${rowHeight}px` gives uniform row sizing for DnD snap targets.

Three delivery layers, each building on the previous:

1. **Layer 1 — Model + spanning:** Discriminated union, auto-flow rendering,
   backward compat.
2. **Layer 2a — DnD reposition + resize:** Edit mode, pointer events, grid-cell
   snapping, push/displace collision, model update on drop.
3. **Layer 2b — DnD insert + YAML round-trip:** Insertion zones, PageDocument
   integration, bidirectional YAML mapping.

### Future extension

Per-row and per-column sizing is a natural CSS grid extension. `rowHeight` is
the auto-row default; a future `rowHeights: number[]` or
`columnWidths: number[]` on `GridProps` would override uniform defaults with
explicit per-track sizes. The current design does not close this door.

---

## 2. Layer 1 — Model + Spanning

### 2.1 Type changes

`GridPlacement` becomes a discriminated union (D3). All existing placements gain
`mode: 'positioned'`. New auto-placed items use `mode: 'auto'`.

```typescript
type GridPlacement =
  | { mode: 'positioned'; x: number; y: number; w: number; h: number }
  | { mode: 'auto'; w: number; h: number };
```

`GridProps` gains optional `rowHeight`:

```typescript
interface GridProps {
  readonly columns: number;
  readonly rowHeight?: number;
}
```

### 2.2 YAML surface

```yaml
# Explicit positioning (existing, unchanged except for mode field)
items:
  - placement: { mode: positioned, x: 0, y: 0, w: 6, h: 2 }
    component: { type: metric, ... }

# Auto-placed with spanning (new)
  - placement: { mode: auto, w: 4, h: 1 }
    component: { type: bar-chart, ... }

# Auto-placed, default 1x1 (new, minimal)
  - placement: { mode: auto, w: 1, h: 1 }
    component: { type: metric, ... }
```

### 2.3 Renderer changes

1. **Fix pre-existing bug (D1):** The items rendering path
   (`render.ts:95-100`) hardcodes `gridTemplateColumns: "repeat(12, 1fr)"`,
   overriding `GridProps.columns` set by `applyLayoutCSS`. Consolidate grid
   setup to a single authoritative source that reads `GridProps.columns`.

2. **Auto-flow:** Add `grid-auto-flow: row` to the grid container when any
   item has `mode: 'auto'` (D2). Using `row` (not `dense`) preserves DOM
   source order for WCAG 2.1 SC 1.3.2 compliance. Gaps may appear where
   explicitly placed items block auto-flow — this is an acceptable trade-off
   for accessibility.

3. **Placement branching:** `applyGridPlacement` branches on `mode`:
   - `'positioned'` — sets `grid-column: (x+1) / span w` and
     `grid-row: (y+1) / span h` (existing behaviour).
   - `'auto'` — omits position properties, sets only
     `grid-column: span w; grid-row: span h`. CSS auto-flow places the item.

4. **Row height:** When `GridProps.rowHeight` is present, set
   `grid-auto-rows: ${rowHeight}px`. When absent, keep current `min-content`.

### 2.4 Parser changes

`page-parser.ts` handles the `mode` field in YAML and produces the
discriminated union. Existing YAML without `mode` is treated as
`mode: 'positioned'` for backward compatibility.

### 2.5 Backward compatibility

Grids where every item has `mode: 'positioned'` render identically to today.
`grid-auto-flow` is only added when auto items exist. The `rowHeight` property
is optional with no default change.

### 2.6 Auto-placed item ordering

Auto-placed items follow YAML array order, which determines DOM order and
therefore visual position via CSS Grid's auto-placement algorithm. This
coupling must be documented for authors — reordering items in YAML changes
their visual position.

---

## 3. Layer 2a — DnD Reposition + Resize

### 3.1 Edit mode (D5)

A toggle activates grid edit mode on a specific grid instance. In edit mode:

- Grid lines become visible (CSS outline on cells or overlay).
- Drag handles appear on each panel (top bar for move, edges/corners for
  resize).
- Container tree DnD (tab drag) is suppressed for grids inside Container
  entries to prevent interference between grid panel drag and container tab
  drag.
- Nested grids: only the targeted grid level shows edit affordances. Edit mode
  is scoped to a single grid instance.

### 3.2 Pointer event architecture (D4)

Grid-aware pointer event handling with cell snapping:

- `pointerdown` on a drag handle captures the pointer.
- `pointermove` translates pixel delta to grid cell delta using `rowHeight` and
  column width (`containerWidth / columns`).
- Ghost element shows the panel at the snapped grid position during drag.
- `pointerup` commits: compute new `GridPlacement`, run collision resolution,
  update model, re-render.

This is distinct from the Container tree free-layout DnD
(`free-layout-dnd.ts`) which uses absolute positioning and edge-zone detection.
Grid DnD requires cell snapping, cell occupancy detection, and column/row
coordinate translation.

### 3.3 Collision/displacement (D8, D12)

Custom algorithm studying Gridstack.js and React-Grid-Layout as reference
(both MIT). Libraries conflict with model-driven rendering — Gridstack.js
expects to own the DOM, React-Grid-Layout is React-based.

Algorithm:

1. On proposed placement, detect overlapping items by comparing grid cell
   rectangles.
2. Overlapped items are pushed down (increase `y`) by the minimum needed to
   clear.
3. Cascading: displaced items that now overlap others trigger further
   displacement.
4. Displacement preview renders during drag so the user sees where panels will
   land.
5. All displaced items become `mode: 'positioned'` (they now have explicit
   coordinates from the displacement calculation).

### 3.4 Resize

Dragging edges/corners changes `w` and/or `h` while keeping the opposite edge
fixed. Same collision resolution applies — growing a panel can displace
neighbours.

### 3.5 Cancel/escape

Restores visual state to pre-drag positions. No model change.

---

## 4. Layer 2b — DnD Insert + YAML Round-trip

### 4.1 Insertion zones

In edit mode, drop zones appear between existing panels and at the grid's
trailing edge. Dragging a component from an external source (e.g., component
palette from the Visual YAML Builder) into a drop zone creates a new
`GridItem`:

- Zone detection: when pointer hovers between occupied cells, highlight the
  insertion gap.
- New items default to `mode: 'auto', w: 1, h: 1` unless the source provides
  span hints.
- Insertion triggers the same collision/displacement algorithm if the target
  area is occupied.

### 4.2 PageDocument integration (D10, D11)

Grid DnD operates as a new editing surface within the Visual YAML Builder
(#434). All mutations flow through `PageDocument`:

- Reposition → `PageDocument` updates the `placement` node in the YAML CST.
- Resize → `PageDocument` updates `w`/`h` in the `placement` node.
- Insert → `PageDocument` appends a new `GridItem` node to the `items` array.
- Delete → `PageDocument` removes the node.

This gives undo/redo for free — every DnD operation is a `PageDocument`
transaction. Change notifications propagate to all surfaces (tree view, YAML
editor, preview).

### 4.3 YAML round-trip (D10)

When DnD changes a `GridPlacement`:

1. `PageDocument` receives the new placement values.
2. It locates the corresponding node in the YAML CST by component ID.
3. It mutates the `placement` properties in-place, preserving formatting and
   comments.
4. `PageDocument.toString()` emits the updated YAML.

The mapping stays within the `GridPlacement` model — DnD operates on
`{mode, x, y, w, h}` and writes back the same structure. The lossy conversion
problem (YAML `rows/columns` → runtime `GridPlacement`) only applies if we
tried to convert positioned items back to slot-based layout syntax, which we
don't — grids stay as `items` with `GridPlacement`.

---

## 5. Layout-Agnostic DnD Architecture

The DnD infrastructure is designed to work across all layout types, not just
`grid`. Each layout type provides a constraint adapter that tells the DnD
system what operations are permitted:

| Layout | Span directions | Snap axis | Resize handles |
|--------|----------------|-----------|----------------|
| `grid` | cols + rows (2D) | both | all edges + corners |
| `columns` | cols only | horizontal | left/right edges |
| `rows` | rows only | vertical | top/bottom edges |
| `tabs`/`stack`/etc | none | reorder only | none |

The constraint adapter interface:

- **Snap targets:** What grid cells or positions exist for snapping.
- **Allowed resize handles:** Which edges/corners to show based on permitted
  span directions.
- **Displacement axis:** Whether displaced items push horizontally, vertically,
  or both.
- **Reorder-only mode:** For 1D layouts like tabs/stack, DnD becomes reordering
  (change array index) rather than spanning.

The core pointer event machinery, edit mode toggle, ghost preview, and
PageDocument integration are shared across all layout types. Only the
layout-specific constraint math differs.

Build for `grid` first (2D, full spanning). Other layout types add DnD by
implementing their constraint adapter — the infrastructure is ready.

---

## 6. Testing Strategy

### 6.1 Layer 1 tests (model + rendering)

- `GridPlacement` discriminated union: type guards, exhaustive switch coverage.
- Auto-placed items: verify no `grid-column`/`grid-row` position properties
  set, only `span` values.
- Positioned items: verify `grid-column: (x+1) / span w` (unchanged).
- Mixed mode: grid with both positioned and auto items, `grid-auto-flow: row`
  present.
- `rowHeight`: when set, `grid-auto-rows` matches; when absent, `min-content`.
- Bug fix: `GridProps.columns` respected by items path (not hardcoded to 12).
- Parser: YAML without `mode` parses as `mode: 'positioned'`; YAML with
  `mode: 'auto'` produces correct union variant.
- Backward compat: existing grid YAML renders identically.

### 6.2 Layer 2a tests (DnD reposition + resize)

- Edit mode toggle: entering shows handles/gridlines, exiting removes them.
- Drag reposition: panel moves to correct grid cell on drop.
- Drag resize: panel span changes correctly, opposite edge stays fixed.
- Collision displacement: dropping on occupied cell pushes occupant down.
- Cascading displacement: chain of displaced panels resolves correctly.
- Cancel/escape: visual state restored, no model change.
- Nested grids: edit mode on outer grid doesn't activate inner grid handles.
- Container tree coexistence: tab drag suppressed when grid edit mode active.

### 6.3 Layer 2b tests (insert + round-trip)

- Insertion zone detection: zones appear between panels and at trailing edge.
- Insert creates new `GridItem` in model.
- PageDocument integration: each DnD operation creates an undoable transaction.
- Undo/redo: move a panel, undo restores original position.
- YAML round-trip: DnD change → `PageDocument.toString()` → re-parse → same
  runtime state.

### 6.4 Layout-agnostic DnD tests

- Constraint adapter: `columns` layout restricts resize to horizontal only.
- Constraint adapter: `rows` layout restricts resize to vertical only.
- Constraint adapter: `tabs`/`stack` allows reorder only, no spanning.
- Shared infrastructure: edit mode toggle works for `columns` and `rows` (not
  just `grid`).

---

## References

- `pages-component/src/model/types.ts:20-30` — `GridPlacement`, `GridItem`, `Component` types
- `pages-component/src/renderer/render.ts:96-109` — items rendering path (hardcoded 12-column bug)
- `pages-component/src/renderer/grid.ts` — `applyGridPlacement`
- `pages-component/src/renderer/layout.ts:21-27` — `applyLayoutCSS` for grid type
- `pages-runtime/src/frame-sandbox/free-layout-dnd.ts` — Container tree free-layout DnD (pointer event pattern reference, not grid-specific)
- `docs/specs/2026-06-15-casehub-component-grid-layout-design.md` — original grid layout spec
- `docs/specs/issue-434-visual-yaml-builder-phase1b/2026-09-14-visual-yaml-builder-phase1b-design.md` — Visual YAML Builder Phase 1b (PageDocument, SelectionOverlay)
- `pages-document/src/page-document.ts` — PageDocument CST facade
- `pages-builder/src/overlay/selection-overlay.ts` — builder overlay mechanics
- CSS Grid spec — `grid-auto-flow`, auto-placement algorithm
- WCAG 2.1 SC 1.3.2 — Meaningful Sequence (DOM order = visual order)
- Gridstack.js (MIT) — collision/displacement algorithm reference
- React-Grid-Layout (MIT) — collision algorithm reference
- Issue #434 — Visual YAML Builder
- Issue #613 — this epic
