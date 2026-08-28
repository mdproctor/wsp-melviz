# Edge-Splice Interaction Design

**Date:** 2026-08-28
**Branch:** (pending — new branch needed)
**Refines:** Infra spec interaction #7 (node-on-edge drop)

## Overview

Drag a node and drop it onto an edge to splice it in. The node is inserted
between the edge's source and target, replacing the original edge with two
new edges through the dropped node. Grammar rules determine whether the
splice is valid, with visual feedback distinguishing valid drop zones from
non-drop zones.

This is a refinement of infra spec interaction #7. The infra spec designed #7
using React Flow's native drag as a gesture detector. This design replaces the
drag mechanism with a custom pointer-event ghost+clone system because
auto-layout contexts require the original node to stay in place during drag
(React Flow's drag visually moves the node, causing edges to rubber-band).

Note: infra spec interaction #6 (drag-to-empty-space) has the same
`nodesDraggable: false` problem — it also relies on `onNodeDragStop` which
doesn't fire when nodes are not draggable. The custom drag system designed
here can naturally extend to handle #6 (drag-end over empty space → show
node chooser). That extension is tracked separately as a follow-up issue.

## Constraints

- `nodesDraggable` remains `false` — auto-layout owns node positions
- graph-core stays pure data (PP-20260826-507928) — all interaction logic in graph-renderer
- Single undo unit for the entire operation (infra spec D11)
- Events follow `pages-event` contract (PP-20260705-bac842)

## Eligibility

A node is eligible for drag-to-splice if:
- It has no children (`childrenOf(model, nodeId).length === 0`)
- It has no `parentId` (is a root-level node, not inside a container)

Nodes inside containers are excluded to avoid containment hierarchy changes
(`allowedParentTypes` violations). This restriction can be relaxed later.

Disconnected nodes (no edges) are eligible — they skip source-side cleanup
entirely.

## Interaction Lifecycle

### 1. Drag Start — pointerdown on node body (pending phase)

The user presses on a node body (not a connection port). The coordinator
enters the **pending** phase — no visual changes yet:

1. Checks eligibility (leaf, no parentId)
2. Records the pointer start position (`clientX`, `clientY`)
3. Calls `setPointerCapture` on the event target

The ghost class and clone are deferred until the drag threshold is exceeded
(see §2 below). This avoids ghost flicker on simple click-to-select gestures,
where `pointerdown` is immediately followed by `pointerup` with no movement.
The infra spec §8.0 establishes this threshold pattern for interactions #6
and #7.

### 2. Drag Move — pointermove

**Before threshold exceeded (pending phase):**

If the pointer has moved less than `DRAG_THRESHOLD` (5 CSS pixels) from the
start position, ignore the event. The threshold is measured as Euclidean
distance: `Math.hypot(dx, dy) < DRAG_THRESHOLD`.

**Activation (pending → active transition, on first move exceeding threshold):**

1. Add CSS class `node-move-ghost` to the `.react-flow__node` element
2. Create a floating clone element from the node's stencil rendering

**Active phase (every subsequent move after activation):**

The clone follows the cursor. On each move:

1. Hit-test edges via `document.elementsFromPoint(x, y)` — find the nearest
   `.react-flow__edge` element under the cursor
2. Extract the edge ID from the element's `data-id` attribute (React Flow
   convention)
3. Look up the edge in the model — skip if the edge is connected to the
   dragged node (source or target equals nodeId, since splicing onto your
   own edge would create a self-loop)
4. Resolve source and target nodes
5. Call `editPolicy.canSpliceOntoEdge(edge, draggedNode, model)` — validates
   that **both** replacement edges are permitted using projected-model
   cardinality checks (see §EditPolicy Extension)
6. Update edge highlight based on result:
   - Valid (both connections pass) → `edge-splice-valid` (green)
   - Invalid → no class (not a drop zone)
7. Remove highlight from previously highlighted edge if cursor moved away

### 3. Drag End — pointerup

**If drag was never activated (pointerup before threshold exceeded):**
- Release pointer capture
- No visual changes, no model change
- The `click` event fires normally (pointer capture does not suppress click
  events per W3C Pointer Events spec), so React Flow's `onNodeClick` handler
  fires for node selection

**Over a highlighted edge (valid — drag was activated):**
- Remove ghost class and clone element
- Compute source-side cleanup strategy via `editPolicy.getDeleteStrategy(node, model)`
  mapped to `SourceCleanupStrategy` (`auto-join` or `disconnect`)
- Dispatch `onMutation({ type: 'moveNodeToEdge', nodeId, edgeId, sourceCleanup })`
- ELK re-layouts the entire graph

**Over nothing / non-highlighted edge (drag was activated):**
- Remove ghost class and clone element
- No model change — node returns to normal

## Architecture

### NodeMoveCoordinator

Standalone module in `graph-renderer/src/editing/node-move-coordinator.ts`,
following the coordinator pattern (GE-20260825-309197). Owns all mutable drag
state. Never calls structural operations directly — returns a result that the
caller dispatches.

```typescript
interface NodeMoveCoordinator {
  startDrag(nodeId: string, pointerEvent: PointerEvent, model: GraphModel): void;
  dispose(): void;
}

type SourceCleanupStrategy = 'auto-join' | 'disconnect';

type DragEndResult =
  | { type: 'splice'; nodeId: string; edgeId: string; sourceCleanup: SourceCleanupStrategy }
  | { type: 'cancelled' };
```

**Invariant:** The `model` reference passed to `startDrag` is valid for the
entire duration of the drag. GraphCanvas must not re-render the graph or
trigger ELK re-layout while a drag is active. The drag is a modal
interaction — no concurrent graph edits are permitted. The model is used
during every `pointermove` (edge lookup, validity checks) and at drag end
(source cleanup strategy computation). Because the user cannot perform other
graph edits while mid-drag, the model snapshot captured at `startDrag` is
guaranteed to reflect the true graph state throughout the gesture.

**Constructor dependencies (injected, read-only):**
- `editPolicy: EditPolicy` — validation and source cleanup strategy
- `containerEl: HTMLElement` — the graph canvas container element (for clone
  positioning, hit-testing, and event delegation)
- `onResult: (result: DragEndResult) => void` — callback on drag end

**Internal state (owned by coordinator, cleaned up on dispose):**
- `activeModel: GraphModel | null` — model snapshot for the active drag
- `dragStartPosition: { x: number; y: number } | null` — pointer position
  at `pointerdown`, used for threshold check
- `dragActive: boolean` — `false` during pending phase, `true` after
  threshold exceeded
- `ghostedNodeId: string | null`
- `cloneElement: HTMLElement | null`
- `highlightedEdgeId: string | null`
- Pointer event listeners (attached on start, removed on end)

### EditPolicy Extension

New optional method on `EditPolicy`:

```typescript
canSpliceOntoEdge?(
  edge: GraphEdge,
  node: GraphNode,
  model: GraphModel
): boolean;
```

Implemented in `defaultEditPolicy()` using a **projected model** to avoid
false negatives from stale cardinality counts:

1. Build a projected model reflecting post-operation state:
   a. Remove the target edge from the model (the splice replaces it)
   b. Remove all edges connected to the dragged node (source-side cleanup
      will remove or auto-join them — the auto-join creates an edge between
      the dragged node's predecessor and successor, but this does not affect
      the dragged node's own cardinality or the splice endpoints)
2. Check `canConnect(sourceNode, node, projectedModel)` — can the edge's
   source connect to the dragged node in the post-cleanup state?
3. Check `canConnect(node, targetNode, projectedModel)` — can the dragged
   node connect to the edge's target in the post-cleanup state?
4. Return `true` only if both checks pass

The projected model accounts for cardinality slots freed by removing the
target edge and the dragged node's existing edges. Without projection,
`canConnect` checks cardinality against the current model where nodes may
be at their limits — even though the splice operation itself frees the
necessary slots. Example: node A has `outbound.max: 1` and one outbound
edge A→B. Without projection, `canConnect(A, draggedNode)` returns false.
With projection, A→B is removed from the model, A has 0 outbound edges,
and the check correctly returns true.

The method does NOT check whether the edge is connected to the node — that
guard belongs in the coordinator (it's a drag constraint, not a grammar rule).

Domain adapters may override `canSpliceOntoEdge` for custom splice validation
(e.g., forbidding splicing entirely for certain diagram types). When the method
is not provided by the current policy, the coordinator falls back to a
standalone `defaultCanSpliceOntoEdge` function that executes the same
projected-model algorithm but calls **the current policy's** `canConnect`:

```typescript
function defaultCanSpliceOntoEdge(
  policy: EditPolicy, edge: GraphEdge, node: GraphNode, model: GraphModel
): boolean {
  const projected = buildProjectedModel(model, edge, node);
  const source = nodeById(projected, edge.source);
  const target = nodeById(projected, edge.target);
  if (!source || !target) return false;
  return policy.canConnect(source, node, projected)
      && policy.canConnect(node, target, projected);
}

// Coordinator usage:
const canSplice = editPolicy.canSpliceOntoEdge?.(edge, node, model)
  ?? defaultCanSpliceOntoEdge(editPolicy, edge, node, model);
```

This ensures domain-specific `canConnect` rules are never bypassed in the
fallback path. `defaultEditPolicy()` still provides `canSpliceOntoEdge` as
a built-in convenience — policies created via `defaultEditPolicy()` have it
on the object itself, so the fallback is never reached for them. The fallback
matters only for domain adapters that implement `EditPolicy` independently
with custom `canConnect` rules but choose not to implement
`canSpliceOntoEdge`.

**Source-side cleanup strategy:** The coordinator pre-computes the source-side
cleanup strategy by calling the existing `getDeleteStrategy(node, model)` and
mapping the result:
- `auto-join` → `'auto-join'` (1-in, 1-out leaf, predecessor and successor
  can connect)
- `disconnect` / any other → `'disconnect'` (remove all connected edges)

This reuses the existing `getDeleteStrategy` method — no new EditPolicy method
is needed. The mapping is trivial because eligibility guarantees the node is
a leaf (no `cascade`), and the drag is deterministic (no `prompt`).

### GraphEdit Extension

The existing `moveNodeToEdge` type in the `GraphEdit` union is extended:

```typescript
{ type: 'moveNodeToEdge'; nodeId: string; edgeId: string; sourceCleanup: SourceCleanupStrategy }
```

Where `SourceCleanupStrategy = 'auto-join' | 'disconnect'`.

The `sourceCleanup` field carries the pre-computed strategy so that
`applyGraphEdit` remains policy-free — the same architectural pattern as
`removeNode` carrying its `DeleteStrategy`. The coordinator calls
`getDeleteStrategy` before dispatching the edit; `applyGraphEdit` executes
the strategy without needing access to `EditPolicy`.

### applyGraphEdit — moveNodeToEdge Implementation

Replaces the current stub that throws. Performs atomically:

**Source-side cleanup** (based on pre-computed `sourceCleanup`):
1. If `sourceCleanup === 'auto-join'`:
   - Find the single inbound and single outbound edge of the node
   - Remove both edges
   - Add auto-join edge (predecessor → successor) inheriting the inbound
     edge's type
2. If `sourceCleanup === 'disconnect'`:
   - Remove all edges connected to the node (`edgesOf(model, nodeId)`)

**Target-side splice:**
1. Look up the target edge, resolve its source and target nodes
2. Remove the target edge
3. Add edge source→node with `type: originalEdge.type` (inherits the
   original edge's type, consistent with `splitEdge` in
   `graph-core/src/edit.ts:110-115`)
4. Add edge node→target with `type: originalEdge.type` (same type
   inheritance)

New splice edges always inherit the original edge's type. This is consistent
with `splitEdge` which also preserves the edge type through the split
operation.

Return single `EditResult` with new model and any constraint violations.

### GraphCanvas Wiring

GraphCanvas creates the coordinator and handles the result:

```typescript
// On coordinator result
private handleMoveResult(result: DragEndResult): void {
  if (result.type === 'splice') {
    this.onMutation?.({
      type: 'moveNodeToEdge',
      nodeId: result.nodeId,
      edgeId: result.edgeId,
      sourceCleanup: result.sourceCleanup,
    });
  }
}
```

**Event delegation across Lit/React boundary:** The coordinator attaches a
single `pointerdown` listener on `containerEl` (the `.diagram-root` div
created by GraphCanvas) using event delegation. This single listener survives
React re-renders — node additions and removals do not require re-attachment.

The delegation handler:
1. Checks `event.target.closest('.stencil-source-handle')` — if the click
   hit the source handle port, ignore and let React Flow handle connection
   initiation
2. Finds the node wrapper via `event.target.closest('.stencil-decoration-wrapper')`
   — if not found, ignore (click was on canvas, edge, or other non-node
   element)
3. Resolves the node ID from the ancestor `.react-flow__node[data-id]`
   element
4. Starts the drag via `coordinator.startDrag(nodeId, event, this.model)`

After handle shrinking (blocker — see below), the source handle occupies a
small port at the bottom edge of the node. The majority of the node body
surface is drag-eligible. The target handle sits behind the wrapper
(`zIndex: 1` vs wrapper's `position: relative`) and does not intercept body
clicks in normal mode. During connection mode (`.graph-connecting` CSS class
active), the `stencil-source-handle` has `pointer-events: none` — this is
already handled by the existing CSS in `css-isolation.ts`.

## Visual Design

### Ghosted Original

```css
.node-move-ghost .stencil-decoration-wrapper {
  opacity: 0.3;
  pointer-events: none;
  transition: opacity 120ms ease-out;
}
```

### Drag Clone

- Cloned from the node's `.stencil-decoration-wrapper` innerHTML
- `position: fixed`, offset from cursor by initial grab point delta
- `pointer-events: none`
- `opacity: 0.85`
- `filter: drop-shadow(0 4px 12px rgba(0,0,0,0.2))`
- `z-index: 1000` (above React Flow layers)

### Edge Highlights

```css
.edge-splice-valid .react-flow__edge-path {
  stroke: var(--pages-success-9);
  stroke-width: 3px;
  filter: drop-shadow(0 0 4px var(--pages-success-9));
  transition: stroke-width 100ms, filter 100ms;
}
```

Edge highlight classes are applied to the `.react-flow__edge[data-id="..."]`
element wrapping the SVG path. Only valid edges (both connections pass) are
highlighted — there is no partial/amber state.

## Handle Shrinking (Blocker)

This is a **blocking prerequisite** — not bundled into edge-splice.
It affects all node interactions and must be completed before edge-splice
can be implemented. Tracked as a separate GitHub issue.

Current state: source handles cover 100% of node surface
(`stencil-wrapper.tsx` renders the source handle with
`position: absolute, top: 0, left: 0, width: 100%, height: 100%, zIndex: 2`
and `css-isolation.ts` adds `cursor: crosshair; pointer-events: all` via
`.stencil-source-handle`). Every `pointerdown` on a node body starts a
connection — the coordinator's drag handler literally cannot fire without
this change.

Target state:
- **Source handle** shrinks to a small visible port at the bottom edge of the
  node (matching ELK downward flow direction)
  - Small circle, accent colour on hover, subtle at rest
  - `cursor: crosshair` — initiates connections
- **Target handle** stays full-node invisible — receives connections anywhere
  on the node body
- Node body (outside the source port) becomes the drag-to-move surface

The target handle staying full-node means connection drops are still forgiving
(drop anywhere on the target node). Only connection initiation requires the
port.

Interaction consequences (touch target sizing per WCAG 2.5.5, port
discoverability, interaction with `connectionRadius={0}` on ReactFlow) are
scoped to the handle-shrinking issue's own design, not this spec.

## Events

The coordinator does not emit custom events — it returns a `DragEndResult` to
GraphCanvas. GraphCanvas dispatches the mutation via `onMutation`. If consumers
need drag lifecycle events, they can be added later via `emitPagesEvent`:

- `graph:node:move:start` — drag started
- `graph:node:move:end` — drag completed (splice or cancel)

## Testing

- **Unit tests** for `canSpliceOntoEdge` — valid and invalid states with
  various grammar configurations, including projected-model correctness:
  nodes at cardinality limits where the splice frees the necessary slots
- **Unit test** for `defaultCanSpliceOntoEdge` fallback — verify that a
  custom `EditPolicy` with domain-specific `canConnect` restrictions (but
  no `canSpliceOntoEdge`) has its `canConnect` rules respected via the
  standalone fallback function
- **Unit tests** for `applyGraphEdit('moveNodeToEdge')` — source cleanup
  (auto-join, disconnect, no edges) × target splice, verifying:
  - Edge type inheritance from original edge
  - Auto-join edge creation with correct type
  - All connected edges removed on disconnect
- **Unit tests** for `NodeMoveCoordinator` — mock pointer events, verify
  ghost class toggle, clone creation/removal, edge highlight state,
  source handle discrimination (clicks on `.stencil-source-handle` ignored),
  drag threshold (pointerdown + pointerup with <5px movement produces no
  ghost/clone and fires `cancelled` result)
- **Integration test** — full lifecycle: pointerdown → pointermove over edge
  → pointerup → model mutation verified

## References

- [infra spec §5.3, §8.0](docs/specs/diagram-editing-infrastructure/2026-08-26-diagram-editing-infrastructure-design.md) — interaction #7 design, visual feedback
- [GE-20260825-309197] — standalone coordinator pattern for multi-phase interactions
- [GE-20260827-ed8606] — elementsFromPoint() for hit-testing through React Flow overlays
- [PP-20260826-507928] — graph-core pure data principle
- [PP-20260705-bac842] — pages-event contract
- `graph-renderer/src/editing/types.ts:35` — existing `moveNodeToEdge` GraphEdit type
- `graph-renderer/src/editing/apply-graph-edit.ts:61` — existing stub
- `graph-renderer/src/editing/edit-policy.ts:82` — existing `getDeleteStrategy`
- `graph-core/src/edit.ts:102` — `splitEdge` (reference for target-side mechanics)
