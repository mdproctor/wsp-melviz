# Decisions — #613 Layout Spanning and DnD

## D1: Layout model approach

**Choice:** Enhance existing `grid` layout type with auto-flow and spanning
**Alternatives:**
- New `positioned-grid` type — cleaner separation but more types to maintain, duplicate rendering logic
- Enhance slot-based types (`columns`, `rows`) — too limited, no coordinate model
**Rationale:** `grid` already has `GridPlacement {x, y, w, h}` rendering. Adding auto-flow for items without explicit coordinates extends the model naturally without a new type.
**Trade-offs:** Existing `grid` behaviour must remain backward compatible when all items have full `{x, y, w, h}`.
**Sources:** `pages-component/src/model/types.ts:20-30`, `pages-component/src/renderer/render.ts:96-109`, grid layout spec `docs/specs/2026-06-15-casehub-component-grid-layout-design.md`
**Exploration:** quick
**Status:** captured

## D2: Auto-flow strategy

**Choice:** CSS `grid-auto-flow: row dense` for items without explicit coordinates
**Alternatives:**
- Custom sequential fill algorithm — duplicates CSS grid behaviour, more code to maintain
- Strict separation (all-explicit or all-auto, no mixing) — less flexible, forces full coordinate specification
**Rationale:** CSS grid natively handles mixed placement (explicit + auto-flow). `dense` packing fills gaps left by explicitly positioned items. Zero custom layout code.
**Trade-offs:** `dense` packing can reorder items visually vs source order, which may surprise users. Document this.
**Sources:** CSS Grid spec (`grid-auto-flow`), Gridstack.js, React-Grid-Layout prior art
**Exploration:** quick
**Depends on:** D1 (grid type enhancement)
**Status:** captured

## D3: Span syntax

**Choice:** Extend `GridPlacement` — make `x` and `y` optional. `{w: 2, h: 3}` means "span 2 cols, 3 rows, auto-placed".
**Alternatives:**
- Separate `span: {cols, rows}` property on `GridItem` — overlapping semantics with `placement`, confusing
- `span` on `Component` itself — leaks grid concerns into the component model
**Rationale:** `w` and `h` already mean span width/height in the existing model. Making `x`/`y` optional is the minimal change. One model, one property, two modes (positioned vs auto-placed).
**Trade-offs:** TypeScript type becomes `{ x?: number; y?: number; w: number; h: number }` — callers must handle the optional fields.
**Sources:** `pages-component/src/model/types.ts` (`GridPlacement` interface)
**Exploration:** quick
**Depends on:** D1 (grid type enhancement), D2 (auto-flow)
**Status:** captured

## D4: DnD interaction strategy

**Choice:** Visual manipulation during drag (CSS transforms, ghost elements, highlight overlays), model update on drop
**Alternatives:**
- Model-driven (update model on every pointermove, re-render) — janky, expensive re-renders during drag
- Hybrid with virtual grid (separate occupancy map during drag) — more complex, unnecessary if visual approach works
**Rationale:** Decouples interaction feedback from model updates. Pointer events manipulate CSS directly for 60fps feedback. Model only changes on drop — clean, single update, re-render once.
**Trade-offs:** Visual state can diverge from model during drag — must handle cancel/escape to restore visual state.
**Sources:** `pages-runtime/src/frame-sandbox/free-layout-dnd.ts` (pointer event pattern), dock-workbench DnD prior art
**Exploration:** quick
**Status:** captured

## D5: Edit mode

**Choice:** Explicit edit mode toggle — entering edit mode shows grid lines, resize handles, drop zones
**Alternatives:**
- Always-on with hover affordances — risk of accidental drags, event listener overhead on every panel
- Per-panel unlock — unusual UX, granular but unfamiliar pattern
**Rationale:** Clean separation of viewing and editing. Edit mode can add DOM elements (handles, overlays) that don't exist in view mode. No performance cost during normal viewing.
**Trade-offs:** Extra click to enter edit mode. Must be discoverable.
**Sources:** Grafana panel edit mode, Gridstack.js `enableMove()`/`enableResize()` toggles
**Exploration:** quick
**Status:** captured

## D6: Delivery phasing

**Choice:** Phased — Layer 1 (model + spanning) lands first, Layer 2 (DnD) builds on top
**Alternatives:**
- Co-developed — DnD requirements could warp the model before it's validated
- DnD-first prototype — doubles the work, throwaway code
**Rationale:** Model must be validated by real YAML usage before DnD adds visual editing. A clean model makes DnD implementation easier; a DnD-warped model creates long-term debt.
**Trade-offs:** DnD development starts later. May discover model gaps that require Layer 1 changes.
**Sources:** Issue #613 architecture section (two-layer approach)
**Exploration:** quick
**Status:** captured

## D7: Grid row sizing

**Choice:** Fixed configurable row height (e.g., `rowHeight: 80` in `GridProps`, default value TBD)
**Alternatives:**
- Content-driven (`min-content`) — vertical spanning becomes unpredictable, DnD resize preview harder
- Fractional (`1fr`) — bad for scrollable content, locks layout to viewport height
**Rationale:** Fixed rows give predictable panel sizes. DnD snap targets are uniform. Gridstack.js and React-Grid-Layout both use this model. `rowHeight` as a grid-level prop keeps it configurable per layout.
**Trade-offs:** Content taller than `h * rowHeight` will overflow or scroll within the panel. Need CSS overflow handling.
**Sources:** Gridstack.js (`cellHeight` option), React-Grid-Layout (`rowHeight` prop)
**Exploration:** quick
**Depends on:** D1 (grid type enhancement)
**Status:** captured

## D8: Collision handling

**Choice:** Push/displace — overlapped panels are pushed down to make room, with visual preview during drag
**Alternatives:**
- Block/reject (prevent dropping on occupied cells) — frustrating, forces manual space clearing
- Overlap allowed (z-index layering) — confusing for layout editing
**Rationale:** Push/displace is the expected UX from Gridstack.js and similar tools. The grid reorganises around the dropped panel. Visual preview during drag shows where displaced panels will end up.
**Trade-offs:** Most complex collision algorithm. Must handle cascading displacement (pushing A displaces B which displaces C). Needs a layout solver.
**Sources:** Gridstack.js displacement algorithm, CSS Grid specification
**Exploration:** quick
**Depends on:** D4 (visual DnD strategy)
**Status:** captured

## D9: DnD capability scope

**Choice:** All four capabilities ship together — reposition, resize, insert, YAML round-trip
**Alternatives:**
- Reposition + resize first, insert + round-trip second — reduces blast radius but delays completeness
- Reposition only — too narrow, doesn't prove the full architecture
**Rationale:** The four capabilities share infrastructure (pointer events, grid math, model update). Shipping together avoids rework and delivers a complete editing experience.
**Trade-offs:** Larger implementation scope. Insertion zones and YAML serialisation add complexity beyond basic DnD. If one capability blocks, the whole layer blocks.
**Sources:** Issue #613 sub-issue list
**Exploration:** quick
**Depends on:** D6 (phased delivery — this is Layer 2)
**Status:** captured
