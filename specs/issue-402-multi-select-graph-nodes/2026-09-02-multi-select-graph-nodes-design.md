# Design: Multi-Select Graph Nodes (#402)

**Date:** 2026-09-02
**Branch:** issue-402-multi-select-graph-nodes
**Extends:** Diagram editing infrastructure spec (interaction #11), edge-splice spec

---

## 1. Problem & Scope

The graph editor currently supports single-select only — `GraphCanvas` sets
exactly one node as `selected` on click. Users need to select multiple nodes
for structural operations (delete a pipeline segment and auto-join, move a
segment to a different edge) and non-structural operations (bulk delete with
disconnect, bulk property edit).

**In scope:**
- Rubber-band drag-select with 1-in/1-out constraint and live feedback
- Shift-click multi-select (unconstrained from scratch, constrained when
  extending a drag-select)
- Segment delete with auto-join (constrained selection)
- Segment hold-to-drag-and-splice (constrained selection)
- Bulk delete with disconnect (unconstrained selection)
- Bulk property edit showing common properties (both modes)

**Out of scope:**
- Free-form multi-node drag positioning (auto-layout owns positions)
- Copy/paste of selections
- Group/ungroup into containers
- Selection persistence across re-layout

## 2. Decisions

See `decisions.md` in this directory for the full decision log (D1–D7).

Key decisions summarised:

| # | Decision | Choice |
|---|---|---|
| D1 | Selection model | Two modes: constrained (drag) + unconstrained (shift-click) |
| D2 | Invalid selection | Reject on release; live valid/invalid highlighting during drag |
| D3 | Rubber-band implementation | Custom pointer events, not ReactFlow's built-in |
| D4 | Shift-click on constrained | Enforces 1-in/1-out integrity |
| D5 | Segment drag gesture | Hold-click any node in selection (300ms, same as single) |
| D6 | Segment drag ghost | Bounding box with node count label |
| D7 | Unconstrained operations | Bulk delete (disconnect) + bulk property edit (common props) |

## 3. Selection Model

### 3.1 State

```typescript
interface MultiSelectState {
  readonly selectedNodeIds: ReadonlySet<string>;
  readonly mode: 'none' | 'constrained' | 'unconstrained';
  readonly boundaryInput: GraphEdge | null;
  readonly boundaryOutput: GraphEdge | null;
}
```

- `constrained` mode: initiated by drag-select, enforces 1-in/1-out
- `unconstrained` mode: initiated by shift-click on empty selection
- `none`: no multi-selection (single-select or nothing selected)

### 3.2 Mode transitions

| From | Action | To |
|---|---|---|
| none | Rubber-band drag completes with valid set | constrained |
| none | Shift-click a node | unconstrained |
| constrained | Shift-click adds/removes node (integrity holds) | constrained |
| constrained | Shift-click would break integrity | rejected (stays constrained) |
| unconstrained | Shift-click toggles nodes | unconstrained |
| any | Click empty canvas | none |
| any | Click single node without shift | none (single-select) |
| any | New rubber-band drag starts | none → constrained (clears previous) |

### 3.3 Operations by mode

| Operation | Constrained | Unconstrained |
|---|---|---|
| Delete (auto-join) | Yes | No |
| Hold-to-drag splice | Yes | No |
| Delete (disconnect) | No | Yes |
| Bulk property edit | Yes | Yes |

## 4. SelectionValidator (graph-core)

Pure function, no DOM dependency. This is the most heavily tested component.

```typescript
interface SelectionResult {
  readonly valid: ReadonlySet<string>;
  readonly invalid: ReadonlySet<string>;
  readonly boundaryInput: GraphEdge | null;
  readonly boundaryOutput: GraphEdge | null;
}

function validateSelection(
  candidateIds: ReadonlySet<string>,
  model: GraphModel
): SelectionResult;
```

### 4.1 Algorithm

1. Compute boundary edges — edges with exactly one endpoint in the candidate set
2. Partition into inbound (source outside, target inside) and outbound
   (source inside, target outside)
3. If exactly 1 inbound and 1 outbound:
   a. **Internal connectivity check** — starting from the entry node
      (`inbound.target`), traverse only edges whose both endpoints are in the
      candidate set. If the exit node (`outbound.source`) is reachable, all
      candidates are valid. If not, all candidates are invalid (the selection
      spans disconnected subgraphs that happen to satisfy 1-in/1-out by
      coincidence).
   b. This is O(|internal edges|) — negligible for expected graph sizes.
4. If boundary count ≠ (1 in, 1 out) → all candidates are invalid. The
   algorithm is binary: either the full candidate set satisfies the
   constraint (all valid) or it doesn't (all invalid). No partial subset
   extraction is attempted during rubber-band drag — this keeps live
   feedback deterministic and fast.

### 4.2 Shift-click validation

```typescript
function canAddToSelection(
  nodeId: string,
  currentSelection: ReadonlySet<string>,
  model: GraphModel
): SelectionResult;

function canRemoveFromSelection(
  nodeId: string,
  currentSelection: ReadonlySet<string>,
  model: GraphModel
): SelectionResult;
```

Both return the projected `SelectionResult` after the add/remove. The caller
checks if the result has `invalid.size === 0` to determine if the operation
is allowed.

### 4.3 Test matrix

| Category | Scenario | Expected |
|---|---|---|
| **Linear chain** | A→B→C→D, select {B,C} | valid: {B,C}, in: A→B, out: C→D |
| **Gap** | A→B→C→D, select {B,D} | invalid: all (2 in, 2 out) |
| **Branch** | A→B→C, A→D, select {B,D} | invalid: D causes 2nd boundary crossing |
| **Single node** | A→B→C, select {B} | valid: {B}, in: A→B, out: B→C |
| **Node with 2 out** | A→B, A→C, select {A} | invalid (2 outputs) |
| **Diamond** | A→B→D, A→C→D, select {B,C} | invalid (2 in, 2 out) |
| **Nested parent** | Parent P with child C, select {P} | valid if P has 1-in/1-out |
| **Empty set** | select {} | valid: empty, no boundary |
| **Entire graph** | Linear A→B→C, select all | valid if A has 0 in, C has 0 out → invalid (0≠1 boundary) |
| **Disconnected node** | No edges, select {X} | invalid (0 in, 0 out — fails 1-in/1-out) |
| **Disconnected groups** | X→A→B, C→D→Y, select {B,C} | invalid (1-in/1-out passes but B and C not internally connected) |
| **Shift-click add valid** | {B,C} selected, add D in A→B→C→D | valid: {B,C,D} |
| **Shift-click add invalid** | {B,C} selected, add A (no inbound) | rejected |
| **Shift-click remove valid** | {B,C,D} selected, remove D in chain | valid: {B,C} |
| **Shift-click remove dangles** | {B,C,D} selected, remove C (mid-chain) | rejected |
| **Type constraints** | Grammar allowedFrom/allowedTo restrict connection | validator respects grammar |
| **Max cardinality** | Node at outbound max, boundary crossing counts | validator uses current model cardinality |

Edge case: a disconnected node with 0 inbound and 0 outbound has 0 boundary
crossings, not 1. This fails the 1-in/1-out constraint, so a disconnected
node alone cannot form a constrained selection via drag-select. Single-node
operations (single-node delete, single-node drag-splice) already handle
disconnected nodes — no special-casing needed here.

## 5. Rubber-Band Interaction (graph-renderer)

Custom implementation — not ReactFlow's `selectionOnDrag` (D3).

### 5.1 Component: RubberBandSelect

Lives in `graph-renderer/src/editing/rubber-band-select.ts`, alongside
`node-move-coordinator.ts`.

**Constructor dependencies:**
- `containerEl: HTMLElement` — graph canvas container
- `screenToFlow: (x: number, y: number) => { x: number; y: number }` —
  coordinate transform (from ViewportBridge §4.8 of infra spec)
- `getNodes: () => ReactFlowNode[]` — current node positions and dimensions
- `getModel: () => GraphModel` — current graph model for validation
- `onComplete: (result: RubberBandResult) => void` — callback on release

```typescript
type RubberBandResult =
  | { type: 'selected'; nodeIds: ReadonlySet<string>; boundaryInput: GraphEdge; boundaryOutput: GraphEdge }
  | { type: 'empty' };
```

### 5.2 Interaction flow

1. **Pointer-down on canvas background** (not on a node, not on an edge) —
   record start position, begin tracking
2. **Pointer-move** — draw a selection rectangle as a semi-transparent overlay.
   The rectangle is rendered in **screen/DOM coordinates** (`position: fixed`)
   using pointer `clientX`/`clientY`. For hit-testing, the rectangle corners
   are converted to **flow coordinates** via `screenToFlowPosition()` and
   compared against node positions from the ReactFlow node array.
   On each move:
   a. Convert rectangle bounds to flow coordinates, then compute which
      ReactFlow nodes fall inside (compare node position + dimensions
      against converted rectangle bounds)
   b. Pass candidate node IDs through `validateSelection()`
   c. Apply CSS classes to node elements:
      - `multi-select-valid` → selection colour highlight
      - `multi-select-invalid` → warning colour highlight
   d. Remove classes from nodes no longer in the rectangle
3. **Pointer-up** — if `result.valid` is non-empty and satisfies 1-in/1-out,
   call `onComplete` with the valid set. Otherwise call with `{ type: 'empty' }`.
   Clean up rectangle overlay and all CSS classes.
4. **Escape** — cancel the drag immediately. Remove rectangle overlay, remove
   all CSS classes, call `onComplete({ type: 'empty' })`. Per infra spec §7,
   Escape dismisses the innermost active UI element — an in-progress
   rubber-band drag counts.

### 5.3 Visual design

```css
.multi-select-rect {
  position: fixed;
  border: 1.5px solid var(--pages-accent-9);
  background: var(--pages-accent-a3);
  pointer-events: none;
  z-index: 5;
}

.multi-select-valid .stencil-decoration-wrapper {
  outline: 2px solid var(--pages-accent-9);
  outline-offset: 2px;
  transition: outline 80ms ease-out;
}

.multi-select-invalid .stencil-decoration-wrapper {
  outline: 2px solid var(--pages-danger-9);
  outline-offset: 2px;
  opacity: 0.7;
  transition: outline 80ms ease-out, opacity 80ms ease-out;
}
```

### 5.4 Performance

`validateSelection()` runs on every pointer-move during drag. For typical
graph sizes (<100 nodes), boundary-edge computation is O(edges). Node
hit-testing is O(nodes). Both are sub-millisecond for expected graph sizes.
No debouncing needed.

## 6. Shift-Click Handler (graph-renderer)

**ReactFlow conflict prevention:** Add `multiSelectionKeyCode={null}` to the
`<ReactFlow>` element in `ReactFlowApp.tsx` to disable ReactFlow's built-in
Shift+click multi-selection. All multi-selection semantics are owned by the
custom handler. Without this, ReactFlow fires `onSelectionChange` with its
own selection state before the custom handler runs, producing spurious events.

Extension of existing `onNodeClick` in `GraphCanvas.ts`.

### 6.1 Logic

```
onNodeClick(nodeId, event):
  if not event.shiftKey:
    // Normal click — clear multi-select, single-select this node
    clearMultiSelect()
    setSingleSelect(nodeId)
    return

  if multiSelect.mode === 'none':
    // Starting unconstrained selection
    startUnconstrained(nodeId)
    return

  if multiSelect.mode === 'unconstrained':
    // Toggle node in unconstrained selection
    toggleUnconstrained(nodeId)
    return

  if multiSelect.mode === 'constrained':
    if nodeId in multiSelect.selectedNodeIds:
      result = canRemoveFromSelection(nodeId, selectedNodeIds, model)
    else:
      result = canAddToSelection(nodeId, selectedNodeIds, model)

    if result.invalid.size === 0:
      applyConstrainedUpdate(result)
    else:
      flashWarning(nodeId)  // brief red flash, ~300ms
```

### 6.2 Visual feedback

- Selected nodes (either mode): `multi-select-active` CSS class with
  selection colour outline
- Warning flash on rejected shift-click: `multi-select-rejected` CSS class
  for 300ms, then auto-removed

## 7. Segment Delete

### 7.1 Constrained (auto-join)

When Delete/Backspace is pressed with a constrained selection:

1. The selection has exactly 1 `boundaryInput` and 1 `boundaryOutput`
2. Build a compound `GraphEdit`:
   a. Remove all selected nodes (each with `strategy: 'disconnect'` — edges
      are cleaned up by the compound operation)
   b. Add a bridge edge: `boundaryInput.source → boundaryOutput.target`
      with type inherited from `boundaryInput.type`
3. Validate the bridge edge via `editPolicy.canConnect()` — this should
   always pass (the original chain was valid), but validate as a safety check.
   If validation fails (e.g., custom EditPolicy, edge type mismatch), fall
   back to disconnect strategy — remove all selected nodes and their edges
   without creating a bridge.
4. Dispatch as `onMutation({ type: 'compound', edits: [...] })`

### 7.2 Unconstrained (disconnect)

When Delete/Backspace is pressed with an unconstrained selection:

1. For each selected node, compute `getDeleteStrategy(node, model, deletionSet)`
   where `deletionSet` is the full set of selected node IDs
2. Apply all strategies — leaf-first ordering per infra spec §9
3. Dispatch as a single compound edit
4. Single undo unit

## 8. Segment Hold-to-Drag Splice

Extends `NodeMoveCoordinator` directly — not a separate class. The coordinator
accepts a `DragSubject` discriminated union:

```typescript
type DragSubject =
  | { type: 'single'; nodeId: string }
  | { type: 'segment'; nodeIds: ReadonlySet<string>;
      entryNodeId: string; exitNodeId: string;
      boundaryInput: GraphEdge; boundaryOutput: GraphEdge };
```

The coordinator branches on subject type at: eligibility check, ghost creation,
edge-skip filter, splice validation, and result construction. Everything else
(hold timer, drag threshold, pointer lifecycle, edge hit-testing mechanics) is
shared. This avoids duplicating 300 lines of interaction logic.

### 8.1 Eligibility

A constrained multi-selection is eligible for drag if:
- All selected nodes are root-level (no `parentId`) — same constraint as
  single-node drag
- The selection has valid `boundaryInput` and `boundaryOutput`

### 8.2 Interaction lifecycle

1. **Pointer-down on a node in the selection** — start 300ms hold timer.
   If the pointer-down is on a node NOT in the selection, clear the
   selection and fall through to single-node drag (existing behaviour).
2. **Hold confirmed** — all selected nodes get `node-move-ghost` class.
   Create a bounding-box ghost element:
   - Compute bounding box of all selected nodes' positions
   - Render a simple rectangle with node count label
   - `position: fixed`, `opacity: 0.85`, `pointer-events: none`
3. **Pointer-move (active phase)** — ghost follows cursor. Edge hit-testing:
   a. Find edge under pointer via `document.elementsFromPoint()`
   b. Skip edges connected to any selected node (source or target in selection)
   c. Validate splice: `canConnect(edge.source, segment.entryNode)` AND
      `canConnect(segment.exitNode, edge.target)` using projected model
      (remove target edge + all edges connected to selected nodes)
   d. Highlight valid edges with `edge-splice-valid` class
4. **Pointer-up over valid edge** — dispatch compound edit:
   a. Source cleanup: remove all selected nodes' external edges, add bridge
      edge (predecessor → successor) for auto-join
   b. Target splice: remove target edge, add `edge.source → entryNode` and
      `exitNode → edge.target`
5. **Pointer-up over nothing** — cancel, restore all nodes

### 8.3 Entry and exit node identification

- **Entry node**: the node in the selection that is the target of `boundaryInput`
  (`boundaryInput.target`)
- **Exit node**: the node in the selection that is the source of `boundaryOutput`
  (`boundaryOutput.source`)

These are cached when the constrained selection is created and updated on
shift-click modifications.

### 8.4 Splice validation (projected model)

Extends `splice-validation.ts`. The projected model for segment splice:

1. Remove the target edge (the edge being spliced onto)
2. Remove all edges where source OR target is in the selected set
3. Check `editPolicy.canConnect(edge.source, entryNode, projected)` — source
   of splice point connects to segment's entry
4. Check `editPolicy.canConnect(exitNode, edge.target, projected)` — segment's
   exit connects to target of splice point

This follows the same projected-model pattern as single-node splice validation
in the edge-splice spec, extended to treat the segment's entry/exit as the
connection points.

## 9. Bulk Property Edit (deferred — nice-to-have)

When multi-selection is active, the property palette shows common properties:

1. Collect stencil type of each selected node
2. Same type → full property schema; values from first node; changed values
   apply to all
3. Mixed types → intersection of property schemas (matching name AND type)
4. No common properties → "No common properties" message
5. Differing values → "mixed" placeholder indicator

Changes apply to all selected nodes as a single compound edit (single undo unit).

This can ship as a follow-up if it adds too much scope to the initial
implementation.

## 10. Component Placement

| Component | Package | File |
|---|---|---|
| `SelectionValidator` | `graph-core` | `src/selection-validator.ts` |
| `validateSelection`, `canAddToSelection`, `canRemoveFromSelection` | `graph-core` | (exported from selection-validator) |
| `RubberBandSelect` | `graph-renderer` | `src/editing/rubber-band-select.ts` |
| `DragSubject` + segment support | `graph-renderer` | `src/editing/node-move-coordinator.ts` (extended) |
| Shift-click handler | `graph-renderer` | `src/bridge/GraphCanvas.ts` (extended) |
| Segment splice validation | `graph-renderer` | `src/editing/splice-validation.ts` (extended) |
| Compound graph edits | `graph-core` | `src/edit.ts` (extended: `removeNodes` — iterated `removeNode` with leaf-first containment ordering) |
| `applyGraphEdit` segment ops | `graph-renderer` | `src/editing/apply-graph-edit.ts` (extended) |
| Selection CSS | `graph-renderer` | `src/css/multi-select.css` |

## 11. Test Strategy

~70% of test effort on `SelectionValidator` — it's pure logic with a large
combinatorial surface.

### 11.1 SelectionValidator (graph-core) — exhaustive

- **Topology matrix**: linear chains, branches, diamonds, cycles, disconnected
  nodes, nested containers, mixed
- **Boundary counting**: 0-in/0-out, 1-in/0-out, 0-in/1-out, 1-in/1-out,
  2-in/1-out, 1-in/2-out, N-in/M-out
- **Shift-click add**: valid additions, rejected additions (breaks constraint),
  additions that change boundary edges
- **Shift-click remove**: valid removals, rejected removals (dangles internal
  edges), removals from ends vs middle
- **Grammar constraints**: `allowedFrom`/`allowedTo` restrictions, cardinality
  limits at boundary
- **Internal connectivity**: disconnected groups that coincidentally satisfy
  1-in/1-out boundary counts (must fail reachability check)
- **Edge cases**: empty set, single node, entire graph, selection containing
  all but one node

### 11.2 RubberBandSelect (graph-renderer) — integration

- Rectangle → node hit detection with coordinate transforms
- Validator integration → CSS class application during drag
- Pointer lifecycle: down → move → up with valid/invalid/empty outcomes
- Edge case: drag starts, no nodes in rectangle, release

### 11.3 NodeMoveCoordinator segment mode (graph-renderer) — integration

- Hold timer (300ms) with threshold (5px)
- Ghost creation for multi-node selection (bounding box)
- Edge hit-testing skips edges connected to selected nodes
- Splice validation uses entry/exit nodes correctly
- Source cleanup auto-join creates correct bridge edge
- Click on unselected node clears selection, falls through to single-node

### 11.4 Compound edits (graph-core) — unit

- `removeNodes` (plural) produces valid graph with no orphan edges
- Bridge edge creation with correct type inheritance
- Batch-aware delete strategy (`deletionSet` parameter)

### 11.5 Shift-click handler (graph-renderer) — integration

- Mode transitions: none→unconstrained, none→constrained (via drag first),
  constrained→constrained (valid add/remove), constrained+rejected
- Clear on canvas click, clear on non-shift node click

## 12. Protocols Consulted

- **pages-event-contract** (PP-20260705-bac842) — event emission for
  selection change events
- **graph-core pure data** (PP-20260826-507928) — SelectionValidator
  is pure logic in graph-core, no DOM dependency

## 13. References

- `specs/diagram-editing-infrastructure/2026-08-26-diagram-editing-infrastructure-design.md` — interaction #11 (multi-select batch delete), EditPolicy SPI, GraphEdit union
- `specs/panel-activation-hash-binding/2026-08-28-edge-splice-design.md` — NodeMoveCoordinator pattern, projected-model splice validation, SourceCleanupStrategy
- `packages/graph-core/src/model.ts` — GraphNode, GraphEdge, GraphModel
- `packages/graph-core/src/grammar.ts` — StencilGrammar, ConnectionRules
- `packages/graph-core/src/edit.ts` — removeNode, splitEdge
- `packages/graph-core/src/query.ts` — inboundEdges, outboundEdges, edgesOf
- `packages/graph-renderer/src/bridge/GraphCanvas.ts` — current single-select, onNodeClick
- `packages/graph-renderer/src/bridge/ReactFlowApp.tsx` — onSelectionChange, ViewportBridge
- `packages/graph-renderer/src/editing/node-move-coordinator.ts` — hold-to-drag pattern
- `packages/graph-renderer/src/editing/splice-validation.ts` — projected-model canSplice
- `packages/graph-renderer/src/editing/edit-policy.ts` — canConnect, getDeleteStrategy
- `packages/graph-renderer/src/editing/apply-graph-edit.ts` — GraphEdit application
- `packages/graph-renderer/src/editing/types.ts` — GraphEdit, DeleteStrategy, DragEndResult
- `packages/pages-property-palette/src/palette/pages-property-palette.ts` — JSON Schema property editor
