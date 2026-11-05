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

The page-level parser (`parsePageContent`) currently accepts two mutually
exclusive YAML formats: `components:` (flat list) and `rows:` (structured
layout with columns). Both produce `GridItem[]` internally — `components:`
assigns `{x:0, y:i, w:12, h:1}` per item, `rows:` assigns placement from
the column/span structure. Neither format exposes `placement` objects or
the `mode` field to YAML authors.

This spec adds a **third format**: `items:` with explicit `placement` objects.
The parser selects format based on which key is present (mutually exclusive,
priority: `items:` > `rows:` > `components:`).

#### New `items:` format

```yaml
items:
  - placement: { mode: positioned, x: 0, y: 0, w: 6, h: 2 }
    component: { type: metric, ... }

  - placement: { mode: auto, w: 4, h: 1 }
    component: { type: bar-chart, ... }

  - placement: { mode: auto, w: 1, h: 1 }
    component: { type: metric, ... }
```

#### Existing format mapping

- **`components:` format** — each component gets
  `{ mode: 'positioned', x: 0, y: i, w: 12, h: 1 }`. No change to existing
  parser logic; `mode: 'positioned'` is set on the produced `GridPlacement`.
- **`rows:` format** — placement derived from column spans and row positions
  as today, with `mode: 'positioned'` added to each produced placement.

#### DnD and format conversion

Pages authored with `components:` or `rows:` format produce positioned items
at parse time. If a user edits placement via DnD (Layer 2b), the YAML must be
in `items:` format for round-trip to preserve placement coordinates. The first
DnD operation on a `components:`/`rows:` page triggers a one-time format
conversion: `PageDocument` rewrites the page content to `items:` format with
the current computed placements. This is a lossy conversion (the original
`rows:`/`columns:` structure is not recoverable), but it is explicit — the
user chose to enter DnD edit mode, which implies grid-based authoring.

#### Nested grid components

Components with `type: grid` and an `items:` array in their YAML are parsed
directly — each item's `placement` is read as a `GridPlacement` discriminated
union. The `component-desugar.ts` path for `type: columns` already produces
`{ type: "grid", items: [...] }` with `{ x, y, w, h }` placements; these
gain `mode: 'positioned'`.

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
   Content taller than `h * rowHeight` overflows — the **renderer** sets
   `overflow: hidden` on each grid item container when `rowHeight` is
   present. Authors can override to `overflow: auto` or `overflow: visible`
   via `Component.style` if needed.

5. **DOM order invariant (WCAG 2.1 SC 1.3.2):** The renderer orders DOM
   children to preserve reading order. The ordering rule depends on item
   mode:

   - **All-positioned grids:** Items sorted by `(y ascending, x ascending)`.
     DOM order matches visual grid position exactly.
   - **All-auto grids:** Items rendered in array order. CSS
     `grid-auto-flow: row` places them in reading order. DOM order = array
     order = visual order.
   - **Mixed grids (positioned + auto):** Positioned items rendered first,
     sorted by `(y ascending, x ascending)`. Auto items rendered after, in
     array order. CSS Grid places positioned items at explicit cells and
     auto items fill remaining cells in DOM order.

   In mixed grids, auto items maintain correct relative order among
   themselves but their visual position relative to positioned items depends
   on CSS auto-flow. If auto items fill gaps between positioned items, the
   absolute DOM-to-visual mapping may diverge for those auto items. This is
   an inherent property of mixing explicit and auto CSS Grid placement —
   the same divergence occurs in any CSS Grid with both placed and
   auto-placed children. For fully positioned or fully auto grids (the
   common cases), the WCAG guarantee is exact.

6. **Auto item ID generation:** For `mode: 'positioned'` items, the
   renderer passes `(placement.x, placement.y)` to `generateId`,
   producing IDs like `{parentId}::{x}::{y}`. For `mode: 'auto'` items,
   the renderer passes `("auto", arrayIndex)`, producing IDs like
   `{parentId}::auto::{index}`. This follows the existing slot child
   pattern (`{parentId}::{slotName}::{index}`) and is distinguishable
   from positioned item IDs. Array index stability: IDs shift when items
   are inserted or removed — the same behaviour as slot child IDs.
   Components needing stable identity should use explicit `component.id`.

### 2.4 Parser changes

**`page-parser.ts`** — `parsePageContent` gains a third branch for the
`items:` key. When present, each array entry is parsed as
`{ placement: GridPlacement, component: Component }`. The `placement` object
is validated: `mode` must be `'positioned'` or `'auto'`; positioned items
require `x`, `y`, `w`, `h`; auto items require `w`, `h`. Width is clamped
to `GridProps.columns` (default 12). Existing YAML without `mode` in a
placement is treated as `mode: 'positioned'` for backward compatibility.

The `components:` and `rows:` branches remain unchanged except that the
`GridPlacement` objects they produce now include `mode: 'positioned'`.

**`component-desugar.ts`** — the DashBuilder `type: columns` desugaring
(lines 396–424) produces `{ type: "grid", items: [...] }` with
`{ x, y, w, h }` placements. These placements gain `mode: 'positioned'`
since they have explicit coordinates computed from the column spans.

**Migration of all call sites:** `GridPlacement` becomes a discriminated
union. TypeScript's exhaustive `switch`/`if` checking enforces that every
site reading `placement.x` or `placement.y` first narrows on
`mode === 'positioned'`. Known call sites: `grid.ts:7-8`,
`render.ts:104`, `grid.test.ts:7,14`, `component-desugar.ts:414`,
`page-parser.ts` placement construction (lines 284, 324, 358, 379).
TypeScript compilation failure is the migration mechanism — no runtime
risk of missed sites.

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
- Container tree DnD is suppressed (see §3.1.2).
- Nested grids: only the targeted grid level shows edit affordances. Edit mode
  is scoped to a single grid instance.
- Discoverability: the toggle must be visible without prior knowledge — a
  toolbar button or prominent affordance on the grid container, not buried in
  a context menu.

#### 3.1.1 State management

Edit mode is **runtime UI state** — not persisted in the model, not saved to
YAML, not restored on navigation. It is transient interaction state, like a
dropdown being open.

- **Storage:** A `data-grid-editing` attribute on the grid container element.
  The attribute's presence indicates edit mode is active. All edit mode
  behaviour (handle visibility, grid line overlay, event capture) is driven
  by this attribute via CSS selectors and event delegation.
- **Mutual exclusion:** A singleton `GridEditController` ensures only one grid
  is in edit mode at a time. Activating edit mode on a second grid
  deactivates the first. The controller is scoped to the page runtime — it
  coordinates across all grids on the page.
- **Toggle location:** An overlay button positioned at the top-right corner
  of the grid container, visible on hover. In edit mode, the button
  changes to a "done editing" state.
- **Event isolation:** In edit mode, a transparent overlay `<div>` covers each
  grid item's content area. This overlay captures `pointerdown` events,
  preventing clicks from reaching panel content (which may have its own click
  handlers). Drag handles sit above the overlay. The overlay is removed when
  edit mode is deactivated.
- **Interaction with SelectionOverlay:** The Visual YAML Builder's
  `SelectionOverlay` provides click-to-select for the component tree. In grid
  edit mode, the `SelectionOverlay` defers to grid DnD — grid edit mode takes
  priority. When grid edit mode is deactivated, `SelectionOverlay` resumes.

#### 3.1.2 Container tree DnD suppression

Grids inside Container entries (tab panels, accordion sections, etc.) must
prevent interference between grid panel drag and container tab drag.

**Mechanism:** When grid edit mode activates, the grid container registers a
**capturing** event listener for `pages-tab-drag-start` (the custom event
that initiates Container tree tab drag, dispatched from `container.ts:123`).
The capturing listener calls `stopPropagation()`, preventing the event from
reaching the `free-layout-dnd.ts` handler on the host element.

**Scope:** All Container tree DnD events are suppressed — both tab drag and
edge splits. The Container's toolbar (layout cycle button, arrange dropdown)
remains visible and functional — only DnD pointer interactions are blocked.

**Cleanup:** When grid edit mode deactivates, the capturing listener is
removed, restoring Container tree DnD.

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

#### Scope: positioned items only

The collision/displacement algorithm operates **exclusively on
`mode: 'positioned'` items**. Auto-placed items (`mode: 'auto'`) are exempt
from displacement — CSS `grid-auto-flow: row` reflows them automatically
around positioned items. This eliminates the auto-to-positioned ratchet
(where displacement would permanently convert auto items to positioned) and
removes the need to read auto items' computed DOM positions.

**Dragging an auto item:** When a user drags an auto item, the drag
operation reads the item's current DOM position once (`getBoundingClientRect`)
to initialise the drag ghost, then converts the item to
`mode: 'positioned'` with the drop coordinates. This is an intentional user
action — the user chose to place this item at specific coordinates.

#### Algorithm (sorted-by-y greedy, single downward pass)

Adapted from React-Grid-Layout's compaction model. The algorithm is a
single downward pass over positioned items sorted by position, which
guarantees termination.

```
resolveCollisions(allPositioned: GridPlacement[], moved: GridPlacement):
  1. Place `moved` at its proposed {x, y, w, h}.
  2. Collect all positioned items (excluding `moved`) into a list.
  3. Sort the list by (y ascending, x ascending).
  4. blockers = [moved]
  5. For each item in sorted order:
     a. newY = item.y
     b. For each blocker in blockers (sorted by y ascending):
        If columnsOverlap(item, blocker) AND
           blocker.y < newY + item.h AND
           blocker.y + blocker.h > newY:
          newY = blocker.y + blocker.h
     c. If newY > item.y:
        item.y = newY
        Append item to blockers
  6. Return the new positions for all items.
```

**Column overlap test:** Two items share columns when
`a.x < b.x + b.w && a.x + a.w > b.x`.

**Full overlap test:** Two items overlap when their grid cell rectangles
intersect: column overlap AND `a.y < b.y + b.h && a.y + a.h > b.y`.

The inner loop (step 5b) scans all blockers for each item. Because
`newY` only increases and blockers are checked in y-ascending order, an
item is pushed below every blocker it overlaps at its proposed position.
This prevents the multi-overlap bug where two items could be pushed to
the same row: item B checks displaced item A as a blocker and is pushed
below it.

**Termination guarantee:** Items are processed in y-ascending order and
only move downward (increasing `y`). `newY` only increases within the
inner loop. No item can be pushed above its current position, so no
cycle is possible. The algorithm terminates in one pass of O(N) items,
with each item checking at most N blockers — worst case O(N²), which is
acceptable for typical grid sizes (N < 100).

**Width validation:** `w` is clamped to `GridProps.columns` at parse time
and on every placement update. An item with `w > columns` is clamped to
`columns` and positioned at `x = 0`. An item placed at `x` where
`x + w > columns` is clamped: `x = columns - w`.

**Push direction:** Always downward (increasing `y`). Horizontal
displacement is not performed — items are never pushed left or right.
Resize that increases `w` may cause overlap; the overlapped items are
pushed down, not sideways. This matches Gridstack.js and React-Grid-Layout
behaviour.

**Unbounded grid height:** The grid grows vertically as needed. There is
no maximum row count — displacement can push items to any `y` value.
The grid container uses `grid-auto-rows` (not `grid-template-rows`), so
new rows are created implicitly.

#### Displacement preview

During drag, the algorithm runs on every `pointermove` that crosses a cell
boundary, producing a preview of where all items will land. The preview
renders displaced items at their projected positions using CSS transforms
(not model updates) for performance. On `pointerup`, the final positions
are committed to the model.

**Auto item reflow trade-off:** Auto items are exempt from the displacement
preview. During drag, auto items remain at their current CSS auto-flow
positions. On drop, the model updates and re-render causes auto items to
reflow to new positions — this reflow is not previewed. For grids with many
auto items, the visual jump on drop may be noticeable. This is an accepted
trade-off: previewing auto item reflow would require computing the CSS
auto-placement algorithm in JavaScript on every `pointermove` frame, which
is the same DOM-coupling problem that the positioned-only scope (§3.3) was
designed to avoid. In practice, grids that undergo heavy DnD editing tend
toward all-positioned (dragging auto items converts them), reducing the
prevalence of mixed-mode grids where this matters.

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

Until Layer 2b lands, DnD changes in Layer 2a are in-memory only — the
runtime model updates but YAML is not persisted. Users see visual changes
immediately but must wait for YAML round-trip to save them.

#### 4.2.1 Required PageDocument extensions

`PageDocument` currently has no grid-item capability — no container
descriptor for `grid`, no methods for manipulating `GridItem` placement
values or the `items` array. The following extensions are required:

**Container descriptor for `grid`:** Add to `CONTAINER_DESCRIPTORS`:

```typescript
{ type: 'grid', defaultContentSlot: 'items',
  slots: [{ kind: 'grid-items'; yamlKey: 'items' }] }
```

This requires a new `SlotDescriptor` kind (`'grid-items'`) because grid
items have a `{ placement, component }` structure that differs from all
existing slot kinds (which expect either named maps, arrays, or nested
arrays of components).

**New `GridItemNode` class** (parallel to `RowNode`, `ColumnNode`):

```typescript
class GridItemNode {
  constructor(private _doc: PageDocument, readonly path: (string | number)[]);

  getPlacement(): GridPlacement;
  setPlacement(placement: GridPlacement): void;
  getComponent(): ComponentNode;
}
```

**New methods on `ComponentNode`** for grid containers:

```typescript
getGridItems(): GridItemNode[];
addGridItem(placement: GridPlacement, type: string,
            props?: Record<string, unknown>): GridItemNode;
insertGridItemAt(index: number, placement: GridPlacement, type: string,
                 props?: Record<string, unknown>): GridItemNode;
removeGridItem(index: number): void;
```

**CST path navigation:** Grid items are navigated via the CST path
`[...componentPath, 'items', index]`. The `placement` sub-object is at
`[...componentPath, 'items', index, 'placement']`. `setPlacement` mutates
individual properties (`mode`, `x`, `y`, `w`, `h`) on the `placement` map
node in place, preserving YAML formatting and comments.

**Undo/redo integration:** Each grid DnD operation (reposition, resize,
insert, delete) is a `PageDocument` transaction. Compound operations
(displacement of multiple items during a drop) use
`beginTransaction()`/`commitTransaction()` so that the entire drop +
displacement is a single undo step.

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

The DnD infrastructure is designed for extension beyond `grid`, but **this
spec fully specifies only `grid` DnD**. Other layout types have fundamentally
different placement models and require their own DnD specifications.

### 5.1 Shared infrastructure (specified)

The following components are layout-agnostic and shared:

- **Edit mode toggle** — the `GridEditController` pattern (singleton mutual
  exclusion, overlay button, event isolation) applies to any layout type.
- **Pointer event machinery** — `pointerdown`/`pointermove`/`pointerup` with
  `setPointerCapture`.
- **Ghost preview** — visual feedback during drag.
- **PageDocument integration** — undo/redo transactions, CST-preserving
  mutations.
- **Container tree DnD suppression** — capturing listener for
  `pages-tab-drag-start`.

### 5.2 Layout-specific constraint adapters

Each layout type provides a constraint adapter. The models differ
significantly:

| Layout | Placement model | DnD operation | Status |
|--------|----------------|---------------|--------|
| `grid` | `{x, y, w, h}` coordinate grid | 2D reposition + resize + insert | **Specified (this spec)** |
| `columns` | `distribution: number[]` ratios | Ratio adjustment at column boundaries | Future spec |
| `rows` | Flex ordering (DOM order) | Vertical reorder | Future spec |
| `tabs`/`stack`/etc | Array index | Reorder (change index) | Future spec |

The `grid` adapter is fully specified in §3. The `columns` adapter would
manipulate `ColumnsProps.distribution` ratios — not grid coordinates. The
`rows` adapter would reorder flex children — not grid-row spans. These
are genuinely different DnD mechanisms sharing only the pointer event and
edit mode infrastructure, not the collision or snapping logic.

The constraint adapter interface abstracts over these differences:

- **`getSnapTargets()`** — grid cells (grid), column boundaries (columns),
  row boundaries (rows), or array indices (tabs/stack).
- **`getAllowedHandles()`** — which edges/corners to show.
- **`applyDrop(from, to)`** — apply the layout-specific model mutation.
- **`getDisplacementAxis()`** — vertical (grid), horizontal (columns),
  or none (reorder-only).

Build for `grid` first. Other layout types add DnD by implementing their
constraint adapter against the shared infrastructure.

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

### 6.4 Shared DnD infrastructure tests

- Edit mode mutual exclusion: activating edit on grid B deactivates grid A.
- Event isolation: pointer events on grid item content are captured by overlay.
- Container DnD suppression: `pages-tab-drag-start` is stopped during grid
  edit mode.
- Ghost preview: ghost element follows pointer at snapped grid coordinates.
- DOM reorder: after DnD reposition, DOM children are sorted by (y, x).

---

## References

- `pages-component/src/model/types.ts:20-30` — `GridPlacement`, `GridItem`, `Component` types
- `pages-component/src/renderer/render.ts:96-109` — items rendering path (hardcoded 12-column bug)
- `pages-component/src/renderer/grid.ts` — `applyGridPlacement`
- `pages-component/src/renderer/layout.ts:21-27` — `applyLayoutCSS` for grid type
- `pages-ui/src/parser/page-parser.ts:272-395` — `parsePageContent` (three YAML format branches)
- `pages-ui/src/parser/component-desugar.ts:396-424` — DashBuilder `type: columns` → `type: grid` desugaring (constructs `GridPlacement` objects, needs `mode: 'positioned'`)
- `pages-runtime/src/frame-sandbox/free-layout-dnd.ts` — Container tree free-layout DnD (pointer event pattern reference, not grid-specific)
- `pages-runtime/src/frame-sandbox/container.ts:123` — `pages-tab-drag-start` custom event dispatch
- `pages-document/src/container-descriptors.ts` — `CONTAINER_DESCRIPTORS` registry (needs `grid` entry)
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
