# Decisions — #613 Layout Spanning and DnD

## D1: Layout model approach

**Choice:** Enhance existing `grid` layout type with auto-flow and spanning
**Alternatives:**
- New `positioned-grid` type — cleaner separation but more types to maintain, duplicate rendering logic
- Enhance slot-based types (`columns`, `rows`) — too limited, no coordinate model
**Rationale:** `grid` already has `GridPlacement {x, y, w, h}` rendering. Adding auto-flow for items without explicit coordinates extends the model naturally without a new type.
**Trade-offs:** Existing `grid` behaviour must remain backward compatible when all items have full `{x, y, w, h}`. Implementation must resolve the existing inconsistency where the items rendering path (`render.ts:95-100`) hardcodes `gridTemplateColumns: "repeat(12, 1fr)"`, overriding `GridProps.columns` set by `applyLayoutCSS`. The items path must delegate grid setup to a single authoritative source that reads `GridProps.columns`.
**Sources:** `pages-component/src/model/types.ts:20-30`, `pages-component/src/renderer/render.ts:96-109`, grid layout spec `docs/specs/2026-06-15-casehub-component-grid-layout-design.md`
**Exploration:** quick
**Status:** revised
**Revision note:** Added acknowledgement of the items/layout path inconsistency identified during review. The pre-existing bug must be fixed as part of this work.

## D2: Auto-flow strategy

**Choice:** CSS `grid-auto-flow: row` for items without explicit coordinates
**Alternatives:**
- `grid-auto-flow: row dense` — fills gaps left by explicitly positioned items, but reorders items visually relative to DOM source order, violating WCAG 2.1 SC 1.3.2 (Meaningful Sequence)
- Custom sequential fill algorithm — duplicates CSS grid behaviour, more code to maintain
- Strict separation (all-explicit or all-auto, no mixing) — less flexible, forces full coordinate specification
**Rationale:** CSS Grid natively handles mixed placement (explicit + auto-flow). `row` (without `dense`) preserves DOM source order → visual order correspondence, which is critical for keyboard navigation and screen reader accessibility. Gaps may appear where explicitly placed items block auto-flow, but this is an acceptable trade-off for accessibility compliance. CSS Grid handles the rendering layout natively — no custom layout code is needed for the display layer. The DnD editing layer (D8) requires separate custom code for collision resolution.
**Trade-offs:** Without `dense`, gaps can appear in the grid when explicitly placed items block auto-flow. Auto-placed items follow YAML array order, which determines DOM order and therefore visual position — this coupling must be documented for authors.
**Sources:** CSS Grid spec (`grid-auto-flow`), WCAG 2.1 SC 1.3.2, CSS Grid spec §8.5 (reorder caution)
**Exploration:** quick
**Depends on:** D1 (grid type enhancement)
**Status:** revised
**Revision note:** Changed from `row dense` to `row` to comply with WCAG 2.1 SC 1.3.2. Added note about auto-placed item ordering coupling. Scoped "zero custom layout code" to the display layer only.

## D3: Span syntax

**Choice:** Extend `GridPlacement` as a discriminated union with explicit `mode` field:
```typescript
type GridPlacement =
  | { mode: 'positioned'; x: number; y: number; w: number; h: number }
  | { mode: 'auto'; w: number; h: number };
```
**Alternatives:**
- Optional `x`/`y` fields (`{ x?: number; y?: number; w: number; h: number }`) — absence of a field is a fragile signal; every consumer must check optionality; `applyGridPlacement` produces NaN from `undefined + 1`
- Separate `span: {cols, rows}` property on `GridItem` — overlapping semantics with `placement`, confusing
- `span` on `Component` itself — leaks grid concerns into the component model
**Rationale:** A discriminated union makes the two placement modes explicit at the type level. TypeScript exhaustive checking on `mode` ensures every consumer handles both cases — no "forgot to check for undefined" bugs. `applyGridPlacement` branches on `mode`: `'positioned'` sets grid-column/row explicitly, `'auto'` omits them and lets CSS auto-flow place the item. For auto-placed items, YAML array order determines DOM order, which determines visual position via CSS Grid's auto-placement algorithm.
**Trade-offs:** Adds a `mode` field to every placement, slightly more verbose in YAML. All existing placements become `mode: 'positioned'`. Callers must handle the union via switch — this is the point, it forces explicitness.
**Sources:** `pages-component/src/model/types.ts` (`GridPlacement` interface), `pages-component/src/renderer/grid.ts` (`applyGridPlacement`)
**Exploration:** quick
**Depends on:** D1 (grid type enhancement), D2 (auto-flow)
**Status:** revised
**Revision note:** Changed from optional `x`/`y` to a discriminated union to prevent NaN crashes in `applyGridPlacement` and force exhaustive handling at every consumer. Added note on auto-placed item ordering.

## D4: DnD interaction strategy

**Choice:** Visual manipulation during drag (CSS transforms, ghost elements, highlight overlays), model update on drop
**Alternatives:**
- Model-driven (update model on every pointermove, re-render) — janky, expensive re-renders during drag
- Hybrid with virtual grid (separate occupancy map during drag) — more complex, unnecessary if visual approach works
**Rationale:** Decouples interaction feedback from model updates. Pointer events manipulate CSS directly for 60fps feedback. Model only changes on drop — clean, single update, re-render once. Grid DnD requires grid-cell snapping, cell occupancy detection, and column/row coordinate translation — fundamentally different from the free-layout absolute-positioning DnD used by the Container tree.
**Trade-offs:** Visual state can diverge from model during drag — must handle cancel/escape to restore visual state.
**Sources:** dock-workbench DnD prior art, Gridstack.js drag interaction model
**Exploration:** quick
**Status:** revised
**Revision note:** Removed misleading `free-layout-dnd.ts` citation — that file implements Container tree free-layout DnD with absolute positioning and edge-zone detection, not CSS Grid DnD. Clarified that grid DnD requires distinct interaction logic (cell snapping, occupancy detection).

## D5: Edit mode

**Choice:** Explicit edit mode toggle — entering edit mode shows grid lines, resize handles, drop zones
**Alternatives:**
- Always-on with hover affordances — risk of accidental drags, event listener overhead on every panel
- Per-panel unlock — unusual UX, granular but unfamiliar pattern
**Rationale:** Clean separation of viewing and editing. Edit mode can add DOM elements (handles, overlays) that don't exist in view mode. No performance cost during normal viewing.
**Trade-offs:** Extra click to enter edit mode. Must be discoverable. Implementation must address coexistence with existing editing surfaces:
  1. **Container tree interaction:** Grid edit mode must suppress container-level DnD (tab drag) when the grid is inside a Container entry, to prevent interference between grid panel drag and container tab drag.
  2. **Nested grids:** The component model allows grids inside grids. Edit mode for the outer grid must not activate inner grid handles — edit mode must be scoped to a single grid level.
  3. **PageDocument integration:** Edit mode state changes and all grid mutations (reposition, resize, insert) must flow through `PageDocument` for undo/redo support, consistent with the Visual YAML Builder's structural editing pattern where all operations go through the document facade.
**Sources:** Grafana panel edit mode, Gridstack.js `enableMove()`/`enableResize()` toggles
**Exploration:** quick
**Depends on:** D11 (Visual YAML Builder relationship)
**Status:** revised
**Revision note:** Added coexistence requirements for Container tree, nested grids, and PageDocument integration.

## D6: Delivery phasing

**Choice:** Phased — Layer 1 (model + spanning) lands first, Layer 2 (DnD) builds on top
**Alternatives:**
- Co-developed — DnD requirements could warp the model before it's validated
- DnD-first prototype — doubles the work, throwaway code
**Rationale:** Model must be validated by real YAML usage before DnD adds visual editing. A clean model makes DnD implementation easier; a DnD-warped model creates long-term debt. This constrains Layer 2: it cannot make assumptions about Layer 1's model shape until Layer 1 is validated through real usage.
**Trade-offs:** DnD development starts later. May discover model gaps that require Layer 1 changes.
**Sources:** Issue #613 architecture section (two-layer approach)
**Exploration:** quick
**Status:** captured

## D7: Grid row sizing

**Choice:** Optional configurable row height via `rowHeight` on `GridProps`. When absent, keep current `min-content` behavior. When present, use `grid-auto-rows: ${rowHeight}px`.
```typescript
interface GridProps {
  readonly columns: number;
  readonly rowHeight?: number;
}
```
**Alternatives:**
- Mandatory fixed row height (breaking change) — every existing grid that relies on content-driven row sizing breaks
- Fractional (`1fr`) — bad for scrollable content, locks layout to viewport height
- Content-driven only (`min-content`) — makes DnD resize preview unpredictable, no uniform snap targets
**Rationale:** Opt-in preserves backward compatibility — existing grids with no `rowHeight` continue to use `min-content`. Layer 2 (DnD) sets `rowHeight` when it needs uniform snap targets. DnD snap targets require predictable cell dimensions; content-driven rows cannot provide this.
**Trade-offs:** Content taller than `h * rowHeight` will overflow or scroll within the panel when `rowHeight` is set. Need CSS overflow handling. Layer 1 (model + spanning) works without `rowHeight`; only Layer 2 (DnD) needs it.
**Sources:** Gridstack.js (`cellHeight` option), React-Grid-Layout (`rowHeight` prop)
**Exploration:** quick
**Depends on:** D1 (grid type enhancement), D6 (phased delivery — rowHeight is primarily a Layer 2 concern)
**Status:** revised
**Revision note:** Changed from mandatory fixed row height to optional `rowHeight` on `GridProps`. Default remains `min-content` (current behavior). Added D6 dependency — rowHeight is primarily a Layer 2 concern for DnD snap targets.

## D8: Collision handling

**Choice:** Push/displace — overlapped panels are pushed down to make room, with visual preview during drag
**Alternatives:**
- Block/reject (prevent dropping on occupied cells) — frustrating, forces manual space clearing
- Overlap allowed (z-index layering) — confusing for layout editing
**Rationale:** Push/displace is the expected UX from Gridstack.js and similar tools. The grid reorganises around the dropped panel. Visual preview during drag shows where displaced panels will end up. This requires a custom displacement algorithm in JavaScript — CSS Grid does not natively handle collision detection or displacement. The algorithm must: detect collisions, calculate displacement vectors, handle cascading displacement, update `GridPlacement` coordinates, and preview the displaced layout.
**Trade-offs:** Most complex collision algorithm. Must handle cascading displacement (pushing A displaces B which displaces C). Needs a layout solver. Implementation approach (library vs custom) is decided in D12.
**Sources:** Gridstack.js displacement algorithm, React-Grid-Layout collision algorithm
**Exploration:** quick
**Depends on:** D2 (auto-flow strategy), D3 (placement model), D4 (visual DnD strategy), D12 (library vs reimplementation)
**Status:** revised
**Revision note:** Removed incorrect CSS Grid specification source — CSS Grid specifies nothing about displacement. Acknowledged that displacement is a custom JavaScript algorithm, not CSS Grid native. Added D2, D3 dependencies. Added D12 dependency for implementation approach.

## D9: DnD capability scope

**Choice:** Phased Layer 2 delivery — reposition and resize ship first; insert and YAML round-trip ship second.
**Alternatives:**
- All four capabilities ship together — larger blast radius; YAML round-trip complexity blocks all DnD capabilities
- Reposition only — too narrow, doesn't prove resize/collision handling
**Rationale:** Reposition and resize share the same infrastructure (pointer events, grid math, collision handling, model update) and can work with in-memory model changes. Insert adds insertion zone detection. YAML round-trip requires the full D10 mapping strategy and integration with PageDocument (D11) — a substantially harder problem because the YAML model (`rows/columns/span`) and runtime model (`GridPlacement {x, y, w, h}`) use different representations, and the conversion is lossy (row/column grouping is discarded during parsing). Bundling YAML round-trip with basic DnD means a block in round-trip blocks all DnD. No `yamlDump`, `toYaml`, or `serializeComponent` exists in the codebase — YAML round-trip is an entirely new subsystem.
**Trade-offs:** Users get DnD before YAML persistence. Changes are in-memory until round-trip lands. Reposition/resize prove the collision algorithm and visual feedback before tackling the harder YAML mapping problem.
**Sources:** Issue #613 sub-issue list, `pages-document/src/page-document.ts` (PageDocument CST facade)
**Exploration:** quick
**Depends on:** D3 (placement model), D6 (phased delivery — this is Layer 2), D8 (collision handling), D10 (YAML-to-runtime mapping — for the second phase), D11 (Visual YAML Builder — for PageDocument integration)
**Status:** revised
**Revision note:** Split Layer 2 into two sub-phases: reposition+resize first, insert+YAML-round-trip second. Referenced PageDocument. Acknowledged YAML round-trip as a new subsystem with no existing serialization infrastructure. Added dependency chain.

## D10: YAML-to-runtime model mapping strategy

**Choice:** Bidirectional mapping via PageDocument — runtime `GridPlacement` changes map back to YAML `rows/columns/span` mutations through the PageDocument CST facade
**Alternatives:**
- Separate YAML serializer (new `fromGridPlacement`-to-YAML converter) — duplicates PageDocument's CST capability, loses undo/redo integration
- Store only runtime model, regenerate YAML from scratch — loses YAML formatting, comments, ordering
- Dual-model synchronization (runtime model as source of truth, YAML derived) — inverts the current architecture where YAML is the source of truth
**Rationale:** PageDocument already provides CST-backed undo/redo, transactions, and `toString()` for YAML round-trip. DnD mutations should be PageDocument operations that modify the YAML CST directly, preserving formatting and enabling undo. The mapping challenge is that `page-parser.ts` converts the YAML `rows/columns/span` structure into `GridItem[]` with calculated `{x, y, w, h}` placements — this conversion is lossy (row/column grouping is discarded). The reverse mapping must reconstruct which row and column a panel belongs to from its grid coordinates.
**Trade-offs:** The reverse mapping (runtime → YAML) is context-dependent: a panel at `{x:6, y:2}` could map to a different column, a different row, or a new row depending on the existing YAML structure. The algorithm must consult the current YAML structure to find the best mapping.
**Sources:** `pages-ui/src/parser/page-parser.ts` (YAML → runtime), `pages-document/src/page-document.ts` (CST facade)
**Exploration:** quick
**Depends on:** D3 (placement model), D11 (Visual YAML Builder relationship)
**Status:** captured

## D11: Grid DnD relationship to Visual YAML Builder

**Choice:** Grid DnD is a new editing surface within the Visual YAML Builder, invoking `PageDocument` operations alongside tree and preview surfaces
**Alternatives:**
- Independent editing surface — duplicates undo/redo, change notification, document state management
- Direct DOM manipulation without document model — loses YAML round-trip entirely
- Full Gridstack.js widget management — conflicts with model-driven rendering, Gridstack expects to own the DOM
**Rationale:** The Visual YAML Builder's Phase 1b spec (#434) establishes that the `PageDocument` facade API is surface-agnostic — tree, preview, and YAML editor invoke the same operations (D12 in Phase 1b decisions). Grid DnD should be a new surface invoking the same `PageDocument` mutations, sharing undo/redo and change notifications. The builder already has `SelectionOverlay`, insertion points, and a document facade — grid DnD integrates with these rather than duplicating them.
**Trade-offs:** Grid DnD is coupled to the builder's infrastructure. Standalone grid editing (outside the builder) requires the PageDocument dependency. The builder's overlay mechanics (CSS-positioned overlays on arbitrary DOM elements) differ from grid editing overlays (grid-cell-aligned handles and zones) — the overlay implementations are distinct even though they share the document model.
**Sources:** `docs/specs/issue-434-visual-yaml-builder-phase1b/2026-09-14-visual-yaml-builder-phase1b-design.md`, `pages-builder/src/overlay/selection-overlay.ts`, `pages-document/src/page-document.ts`
**Exploration:** quick
**Depends on:** D5 (edit mode)
**Status:** captured

## D12: Library vs reimplementation for collision/displacement

**Choice:** Custom reimplementation studying Gridstack.js and React-Grid-Layout displacement algorithms as reference
**Alternatives:**
- Use Gridstack.js as a dependency — immediate collision/displacement/DnD, but Gridstack expects to own the DOM and manage the grid directly, which conflicts with our model-driven rendering through `renderComponent()`. Gridstack's widget model is a separate state from `GridPlacement`/`PageDocument`, creating a synchronization problem.
- Use React-Grid-Layout — React-based; our grid rendering is Lit/vanilla TypeScript. Integration requires a React island for the editing surface, adding framework coupling.
- Minimal algorithm extraction — port only the collision/displacement algorithm from Gridstack (MIT licensed) into a standalone function, without the full library
**Rationale:** The platform uses model-driven rendering (`Component` tree → `renderComponent()` → DOM) with CST-backed YAML editing via `PageDocument`. Gridstack.js assumes ownership of the DOM — it creates, moves, and resizes grid widgets directly. This architectural mismatch means either (a) maintaining two models (Gridstack widgets + PageDocument) in sync, or (b) abandoning model-driven rendering during editing. Neither is acceptable. A custom implementation that studies Gridstack.js's well-documented algorithms (collision detection, displacement vectors, cascading push) provides full control over DOM interaction, direct integration with `PageDocument`, and compatibility with the edit mode scoping (D5).
**Trade-offs:** More implementation effort than a library dependency. The collision/displacement algorithm is well-understood from Gridstack.js and React-Grid-Layout source code (both MIT). Risk: underestimating algorithmic complexity — mitigate by extracting and adapting the specific algorithm rather than designing from scratch.
**Sources:** Gridstack.js source (MIT), React-Grid-Layout source (MIT)
**Exploration:** quick
**Depends on:** D8 (collision handling strategy), D11 (Visual YAML Builder relationship)
**Status:** captured
