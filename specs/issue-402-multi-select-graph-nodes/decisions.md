# Decisions — Multi-Select Graph Nodes (#402)

## D1: Selection constraint model

**Choice:** Two selection modes — constrained (drag-select, enforces 1-in/1-out boundary edges) and unconstrained (shift-click, any nodes)
**Alternatives:**
- Single mode with context-sensitive operations — simpler model but users can't tell if operations are available until after selecting
- Constrained-only for both gestures — too restrictive for shift-click use cases (bulk property edit, visual comparison)
**Rationale:** Two distinct gestures with distinct semantics is clearer UX. Drag-select = "select a pipeline segment for structural operations." Shift-click = "select arbitrary nodes for non-structural operations."
**Trade-offs:** Two modes to understand vs one; mode transitions need careful handling
**Sources:** GraphCanvas.ts (current single-select), EditPolicy (grammar validation)
**Exploration:** quick
**Status:** captured

## D2: Invalid selection handling during rubber-band drag

**Choice:** Reject entirely — if the rubber-band contents don't satisfy 1-in/1-out, no valid selection is created. During drag, valid nodes highlight in selection colour, invalid nodes in warning colour, providing real-time feedback before release.
**Alternatives:**
- Trim to valid — automatically exclude invalid nodes (unpredictable: which nodes get excluded?)
- Show invalid state — allow the selection but visually mark it invalid and disable operations (confusing persistent state)
**Rationale:** Rejection is simplest to implement and teaches the user what "valid segment" means. Live feedback during drag means the user knows exactly what they'll get before releasing.
**Trade-offs:** User must reposition the rectangle to get a valid selection; no partial salvage
**Sources:** Brainstorming discussion
**Exploration:** quick
**Status:** captured

## D3: ReactFlow integration for rubber-band

**Choice:** Custom rubber-band implementation using pointer events, with ReactFlow's `screenToFlowPosition()` for coordinate transforms
**Alternatives:**
- ReactFlow's built-in `selectionOnDrag` with validation layer — gets 70% of the way but creates two-source-of-truth problem (ReactFlow thinks node X is selected, validator says invalid). Limited control over per-node live feedback during drag.
- Hybrid (ReactFlow rectangle + custom overlay) — fighting the framework's all-or-nothing selection model
**Rationale:** The core requirement (live per-node valid/invalid highlighting during drag) cannot be achieved cleanly with ReactFlow's built-in selection. Custom rubber-band is ~50 extra lines for pointer handling and rectangle overlay, but gives clean ownership of the entire interaction with no framework tension.
**Trade-offs:** Must implement coordinate transforms and hit detection manually (mitigated by ReactFlow exposing `screenToFlowPosition()`)
**Sources:** ReactFlow API (selectionOnDrag, onSelectionChange), ViewportBridge (§4.8 of diagram-editing-infrastructure spec)
**Exploration:** deep-analysis
**Status:** captured

## D4: Shift-click extending a constrained selection

**Choice:** Shift-click on a constrained selection (initiated by drag) enforces the 1-in/1-out constraint — add/remove is only allowed if the resulting selection maintains integrity
**Alternatives:**
- Shift-click always unconstrained — would break structural operation guarantees on mixed selections
- Shift-click on constrained selection switches to unconstrained mode — loses the structural semantics the user established with the drag
**Rationale:** The user initiated a constrained selection with intent. Subsequent shift-clicks should refine it, not break it. "No other lines sliced and left dangling."
**Trade-offs:** Some shift-clicks may be rejected (node flashes warning); user must understand why
**Sources:** Brainstorming discussion
**Exploration:** quick
**Status:** captured

## D5: Hold-to-drag gesture for multi-select segment

**Choice:** Hold-click any node in the selection to start dragging the entire segment — same 300ms hold timer as existing single-node drag
**Alternatives:**
- Dedicated grab handle on the selection group — new visual element to learn, clutters the UI
- Different modifier key (e.g., Alt+drag) — inconsistent with existing single-node hold-to-drag
**Rationale:** Consistent with existing single-node hold-to-drag. No new gesture to learn — the only difference is that instead of ghosting one node, the entire selection ghosts.
**Trade-offs:** Hold-click on unselected node while selection exists must clear selection and drag only that node (clear precedence rule needed)
**Sources:** node-move-coordinator.ts (existing hold-to-drag), edge-splice spec
**Exploration:** quick
**Status:** captured

## D6: Ghost visual for multi-node drag

**Choice:** Bounding box of the selection with a label showing node count (e.g., "3 nodes") rather than cloning all node DOMs
**Alternatives:**
- Clone all nodes as floating elements — expensive, messy for large selections
- Miniature representation — complex to implement, hard to read
**Rationale:** Single-node drag clones one node's DOM which is cheap. Cloning N nodes is expensive and visually noisy. A bounding-box ghost communicates "you're moving a group" clearly without the complexity.
**Trade-offs:** Less visual fidelity than seeing actual nodes during drag
**Sources:** node-move-coordinator.ts (single-node clone approach)
**Exploration:** quick
**Status:** captured

## D7: Bulk operations on unconstrained selection

**Choice:** Support bulk delete (disconnect strategy) and bulk property edit (common properties intersection)
**Alternatives:**
- No operations on unconstrained selection — too restrictive, makes shift-click selection pointless
- Full operations including structural (auto-join) — violates the constraint model
**Rationale:** Unconstrained selection needs a purpose. Bulk delete with disconnect is safe (no structural assumptions). Bulk property edit on common properties is natural when same-type nodes are selected.
**Trade-offs:** Mixed-type selections show no common properties (empty panel); bulk property edit is deferred if scope is too large
**Sources:** pages-property-palette (JSON Schema-driven), displayer-types.ts (SelectionMode)
**Exploration:** quick
**Status:** captured
