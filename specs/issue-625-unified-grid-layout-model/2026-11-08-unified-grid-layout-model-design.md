# Unified Grid Layout Model — Two-Mode Editor with Containers

**Issue:** #625 (epic)
**Branch:** `issue-625-unified-grid-layout-model`
**Date:** 2026-11-08
**Child issues:** #626-#638

## Problem

The grid layout system has two layout vocabularies (`items` and `rows`/`columns`), no concept of empty containers, no structural editing (split/merge), and a drag controller built on ghost overlays instead of structural placeholders. The #618 POC proved the core algorithms (compaction, displacement, swap, row breaks) in a standalone HTML file. This epic unifies the layout model, introduces a two-mode editor, and rewrites the grid infrastructure from scratch using the POC as a behavioural reference.

## Type System

### GridPlacement (redesigned)

```typescript
type GridPlacement =
  | { mode: 'positioned'; x: number; y: number; w: number; h: number }
  | { mode: 'auto'; w: number; h: number };
```

Unchanged structurally. YAML serialization changes: items with no `placement` key default to `{ mode: 'auto', w: 1, h: 1 }`. Positioned items serialize as `placement: { x, y, w, h }` (no `mode` key in YAML — presence of `x`/`y` implies positioned).

### GridItem (redesigned)

```typescript
interface GridItem {
  readonly placement: GridPlacement;
  readonly component?: Component;  // absent = empty container
}
```

`component` is now optional (D14). Three states:
- **Empty container:** `{ placement: { x: 0, y: 0, w: 6, h: 2 } }` — no component key
- **Filled container:** `{ placement: ..., component: { type: 'html', ... } }` — standard
- **Nested grid:** `{ placement: ..., component: { type: 'grid', items: [...] } }` — recursive

Nested grids are components of type `grid` whose `items` array contains the sub-grid's children (D17). Each nested grid has an independent column count (default 12).

### YAML Format

After migration (D9), all pages use the `items` format exclusively. The parser drops `rows`/`columns` support. A migration script rewrites existing files.

```yaml
# Auto-flow (no placement key)
items:
  - component: { type: html, props: { content: "full width" } }

# Positioned
items:
  - placement: { x: 0, y: 0, w: 6, h: 2 }
    component: { type: html, props: { content: "left" } }
  - placement: { x: 6, y: 0, w: 6, h: 2 }
    component: { type: BARCHART, props: { title: "right" } }

# Empty container (no component key)
items:
  - placement: { x: 0, y: 0, w: 6, h: 2 }

# Nested grid
items:
  - placement: { x: 0, y: 0, w: 6, h: 4 }
    component:
      type: grid
      items:
        - placement: { x: 0, y: 0, w: 6, h: 1 }
          component: { type: html, props: { content: "nested" } }
```

## Two-Mode Editor

### Mode Switching (D10)

A toolbar toggle switches between Layout mode and Content mode. The toggle is a segmented control in the grid editor toolbar: `[Layout | Content]`.

### Layout Mode (D12, D18)

Structural editing. The grid is a skeleton of containers.

**Operations:**
- **Click** selects a cell (solid outline)
- **Shift+click** extends selection to rectangular region (bounding rect of first and shift-clicked cells)
- **Drag across empty cells** creates a spanning container (#628)
- **Split** subdivides the selected cell into equal halves — horizontal (stack vertically) or vertical (side by side). Valid directions constrained by cell geometry; 1×1 cannot split (D5)
- **Merge** combines selected rectangular region into one item spanning their bounding rectangle. Rectangular only (CSS Grid constraint). Contents of merged cells are preserved in the resulting item; displaced items compact upward (D7)
- **Resize edges** — drag a cell edge to change its `w` or `h`. Adjacent cells adjust to fill or are displaced
- **Add/remove rows** — insert row break, remove empty row
- **Drag-to-swap** — drag a cell by its handle to swap positions with the drop target. Swap only, no displacement (layout mode preserves structure)

**Visual feedback:**
- Selected cells: solid amber outline (2px)
- Hover: solid blue outline (2px)
- Drag (swap): dragged cell at 40% opacity, swap target highlighted with blue outline
- Empty containers: styled drop zones (dashed border, subtle background, "drop component" label)

### Content Mode (D13)

Spatial editing. Components move between containers.

**Operations:**
- **Drag from palette** onto empty container fills it (existing palette infrastructure reused)
- **Drag onto empty container** fills it
- **Drag onto occupied container** uses collision mode:
  - `collision: displace` — automatic displacement, push down, compact (D3, D8)
  - `collision: prompt` — popup with Displace/Split/Replace/Swap options (D4)
- **Dual drag handles** — ⠿ for displace, ⇄ for swap (D6, D13)

**Visual feedback:**
- Placeholder: structural CSS Grid item at snap position (dashed border, semi-transparent) (D2)
- Dragged item: 40% opacity at original position
- Other items: animate via CSS transitions as compaction runs

## Compaction Algorithm (D7, D16)

### Per-Level Independent Compaction

Each grid level (root or nested) runs compaction independently. The algorithm is identical at every level:

1. **Place** — set the dragged/moved item at target `{x, y, w, h}`
2. **Push** — for each non-moved item that overlaps: set `y = moved.y + moved.h`
3. **Compact** — for each item (top to bottom by y): move to lowest `y` with no overlap above

All operations are integer arithmetic on `{x, y, w, h}` tuples. Overlap detection:
```
columnsOverlap = (a.x < b.x + b.w) && (b.x < a.x + a.w)
rowsOverlap    = (a.y < b.y + b.h) && (b.y < a.y + a.h)
overlap        = columnsOverlap && rowsOverlap
```

Horizontal gaps are not compacted — column positions are intentional.

### Nested Grid Auto-Shrink

After compacting a nested grid's items, the container's `h` optionally shrinks to the compacted content height (max `y + h` across all nested items). The parent grid does not re-compact — the container just takes less space vertically, and the parent's next compaction pass (if any) closes the gap.

## Editor Architecture (D15)

### Single Controller with Mode Flag

```typescript
class GridEditController {
  mode: 'layout' | 'content' = 'layout';
  // ... pointer tracking, snap-to-cell, geometry, overlay, selection
  
  private handleDrop(target: CellCoord): void {
    if (this.mode === 'layout') {
      this.swapItems(this.dragItemId, targetItemId);
    } else {
      this.resolveCollision(this.dragItemId, target);
    }
  }
}
```

The controller owns:
- Pointer event handling (down/move/up)
- Snap-to-cell computation
- Grid geometry calculation
- Overlay rendering (placeholders, selection indicators, edit controls)
- Mode-specific dispatch on drop

Structural operations (split, merge, resize) are methods on the controller, guarded by `mode === 'layout'`.

### Mutation via _applyEdit (D19)

Every grid operation calls `_applyEdit('grid', () => { ... })` on the builder shell. No grid-local undo stack. Undo/redo is document-level via PageDocument.

### Event Contract

Grid edit completion dispatches `pages-grid-edit` (already reserved in the event contract protocol). The event detail carries the operation type and affected item IDs.

## Recursive Nesting (D14, D16, D17)

### Model

A nested grid is an item whose component has `type: 'grid'` and an `items` array. Each nested grid has its own column count (default 12, independent of parent).

### Rendering

The renderer detects `component.type === 'grid'` and creates a nested `display: grid` container with its own `grid-template-columns: repeat(N, 1fr)`. Nested items are rendered inside this container using `applyGridPlacement`.

### Editing

When editing a nested grid, the controller scopes its operations to that level. Clicking into a nested grid "focuses" it — operations affect the nested grid's items. Clicking outside returns focus to the parent grid. This is analogous to double-clicking a group in Figma to enter it.

### Compaction

Each grid level compacts independently (D16). Parent sees the nested grid as an opaque box.

## Migration (D9)

### Script

A migration script scans all `.page.yaml` files and rewrites `rows`/`columns` format to `items` format:

- Each row becomes items placed at `y = rowIndex`
- Each column within a row becomes an item at `x = columnOffset, w = span`
- `components` shorthand becomes items at `y = index, x: 0, w: 12, h: 1`

### Parser Changes

After migration, the parser's `rows`/`columns` code paths are removed. Only `items` is parsed. The `components` shorthand may be retained as a convenience (it's just `items` with implicit full-width auto-flow placement).

### PageDocument Changes

`PageNode.getRows()`, `getColumns()`, `addRow()`, `addRowAt()` are removed. Grid operations go through `GridItemNode` exclusively.

## File Structure (All Rewritten)

All grid-related files are rewritten from scratch. The POC and existing code serve as behavioural reference.

| File | Purpose |
|------|---------|
| `grid.ts` | `applyGridPlacement` — CSS Grid style application (redesigned for new types) |
| `grid-types.ts` | `GridItem`, `GridPlacement`, `PositionedPlacement`, `IdentifiedPlacement` |
| `grid-compaction.ts` | Pure-function compaction: `place()`, `push()`, `compact()`, `autoShrink()` |
| `grid-collision.ts` | Overlap detection, collision resolution dispatch |
| `grid-edit-controller.ts` | Two-mode controller (layout/content), pointer handling, snap, overlay |
| `grid-selection.ts` | Selection model (click, shift-extend, rectangular region) |
| `grid-split-merge.ts` | Split and merge operations (layout mode) |
| `grid-overlay.ts` | Visual overlays (placeholder, selection, drag indicators, edit controls) |
| `grid-resize.ts` | Edge resize operations (layout mode) |
| `grid-migration.ts` | rows/columns → items migration script |

### Test Files

Each module gets a corresponding `.test.ts` with unit tests covering:
- Compaction: overlap scenarios, spanning items, edge cases, nested grids
- Collision: all span combinations, mode-specific resolution
- Split/merge: valid directions, resulting placements, merge with contents
- Selection: click, shift-extend, rectangular bounding
- Resize: edge detection, adjacent cell adjustment
- Migration: rows/columns → items conversion accuracy

## Phase Mapping

### Phase 1 — Layout Mode (#626-#630, #635 row breaks)
- #626: Layout mode toggle in toolbar
- #627: Click empty cell → create container
- #628: Drag across empty cells → create spanning container
- #629: Split cell (horizontal/vertical, equal halves)
- #630: Merge adjacent cells (rectangular selection)
- #635: Port POC row breaks to production (via compaction rewrite)

### Phase 2 — Containers (#631-#632)
- #631: Empty containers render as styled drop zones
- #632: Component palette drag-to-fill (wire existing palette infrastructure)

### Phase 3 — Unified Model (#633-#634)
- #633: Simplified YAML output with auto/positioned inference
- #634: Recursive nested sub-grids

### Phase 4 — Framework Integration (#635-#638)
- #635: Rewrite compaction (grid-compaction.ts)
- #636: Rewrite drag UX (grid-edit-controller.ts)
- #637: Wire grid editor to PageDocument model via _applyEdit
- #638: Integrate grid editor as root page builder in YAML visual editor

**Note:** Phase 4 issues overlap with Phase 1 — the rewrite is the vehicle for delivering layout mode. The phase ordering in the epic is conceptual; implementation interleaves: rewrite the modules (Phase 4), then build layout mode features on top (Phase 1), then containers (Phase 2), then nesting/migration (Phase 3).

## References

- `packages/pages-component/src/renderer/grid.ts` — current `applyGridPlacement`
- `packages/pages-component/src/renderer/grid-edit-controller.ts` — current controller (to be rewritten)
- `packages/pages-component/src/renderer/grid-collision.ts` — current collision (to be rewritten)
- `packages/pages-component/src/renderer/grid-overlay.ts` — current overlay (to be rewritten)
- `packages/pages-component/src/renderer/grid-drag.ts` — current drag helpers (to be rewritten)
- `packages/pages-component/src/model/types.ts` — `GridItem`, `GridPlacement` (to be redesigned)
- `packages/pages-document/src/page-document.ts` — `GridItemNode`, `PageNode` grid operations
- `packages/pages-ui/src/parser/page-parser.ts` — `parsePageContent` (rows/columns paths to be removed)
- `packages/pages-builder/src/palette/builder-palette.ts` — existing palette (reused)
- `packages/pages-builder/src/catalog/palette-context.ts` — existing catalog context (reused)
- `packages/pages-builder/src/shell/builder-shell.ts` — `_applyEdit` mutation protocol
- `docs/protocols/casehub/all-mutations-through-applyedit.md` — mutation protocol (PP-20260916-b6f3e8)
- `docs/protocols/casehub/aria-interaction-contract.md` — ARIA requirements (PP-20260817-a11y01)
- `docs/protocols/casehub/pages-event-contract.md` — `pages-grid-edit` reserved event
- `examples/poc/grid-drag-poc.html` — #618 POC (behavioural reference)
- `specs/issue-618-grid-edit-placeholder-drag/decisions.md` — D1-D13
- `specs/issue-618-grid-edit-placeholder-drag/2026-11-06-grid-edit-placeholder-drag-design.md` — POC design spec
- Issue #625 — epic definition with phase breakdown
