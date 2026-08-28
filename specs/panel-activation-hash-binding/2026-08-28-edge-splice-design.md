# Edge-Splice Interaction Design

**Date:** 2026-08-28
**Branch:** (pending — new branch needed)
**Refines:** Infra spec interaction #7 (node-on-edge drop)

## Overview

Drag a node and drop it onto an edge to splice it in. The node is inserted
between the edge's source and target, replacing the original edge with up to
two new edges through the dropped node. Grammar rules determine which
connections are valid — both, one, or neither — with distinct visual feedback
for each state.

This is a refinement of infra spec interaction #7. The infra spec designed #7
using React Flow's native drag as a gesture detector. This design replaces the
drag mechanism with a custom pointer-event ghost+clone system because
auto-layout contexts require the original node to stay in place during drag
(React Flow's drag visually moves the node, causing edges to rubber-band).

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

### 1. Drag Start — pointerdown on node body

The user presses on a node body (not a connection port). The coordinator:

1. Checks eligibility (leaf, no parentId)
2. Adds CSS class `node-move-ghost` to the `.react-flow__node` element
3. Creates a floating clone element from the node's stencil rendering
4. Calls `setPointerCapture` on the event target

### 2. Drag Move — pointermove

The clone follows the cursor. On each move:

1. Hit-test edges via `document.elementsFromPoint(x, y)` — find the nearest
   `.react-flow__edge` element under the cursor
2. Extract the edge ID from the element's `data-id` attribute (React Flow
   convention)
3. Look up the edge in the model — skip if the edge is connected to the
   dragged node (source or target equals nodeId, since splicing onto your
   own edge would create a self-loop)
4. Resolve source and target nodes
5. Call `editPolicy.canSpliceOntoEdge(edge, draggedNode, model)` which
   composes two `canConnect` checks:
   - `canConnect(sourceNode, draggedNode, model)` — can the edge's source
     connect to the dragged node?
   - `canConnect(draggedNode, targetNode, model)` — can the dragged node
     connect to the edge's target?
5. Update edge highlight based on result:
   - Both valid → `edge-splice-valid` (green)
   - One valid → `edge-splice-partial` (amber)
   - Neither valid → no class (not a drop zone)
6. Remove highlight from previously highlighted edge if cursor moved away

### 3. Drag End — pointerup

**Over a highlighted edge (valid or partial):**
- Remove ghost class and clone element
- Dispatch `onMutation({ type: 'moveNodeToEdge', nodeId, edgeId, validity })`
- ELK re-layouts the entire graph

**Over nothing / non-highlighted edge:**
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
  startDrag(nodeId: string, pointerEvent: PointerEvent): void;
  dispose(): void;
}

type SpliceValidity = 'full' | 'source-only' | 'target-only';

type DragEndResult =
  | { type: 'splice'; nodeId: string; edgeId: string; validity: SpliceValidity }
  | { type: 'cancelled' };
```

**Constructor dependencies (injected, read-only):**
- `model: GraphModel` — current graph state
- `editPolicy: EditPolicy` — validation
- `containerEl: HTMLElement` — the graph canvas element (for clone positioning
  and hit-testing)
- `onResult: (result: DragEndResult) => void` — callback on drag end

**Internal state (owned by coordinator, cleaned up on dispose):**
- `ghostedNodeId: string | null`
- `cloneElement: HTMLElement | null`
- `highlightedEdgeId: string | null`
- Pointer event listeners (attached on start, removed on end)

### EditPolicy Extension

New method on `EditPolicy`:

```typescript
canSpliceOntoEdge(
  edge: GraphEdge,
  node: GraphNode,
  model: GraphModel
): { valid: boolean; validity?: SpliceValidity }
```

Implemented in `defaultEditPolicy()`:
1. Look up source and target nodes of the edge
2. Check `canConnect(sourceNode, node, model)` → sourceValid
3. Check `canConnect(node, targetNode, model)` → targetValid
4. Return:
   - Both true → `{ valid: true, validity: 'full' }`
   - Only source → `{ valid: true, validity: 'source-only' }`
   - Only target → `{ valid: true, validity: 'target-only' }`
   - Neither → `{ valid: false }`

The method does NOT check whether the edge is connected to the node — that
guard belongs in the coordinator (it's a drag constraint, not a grammar rule).

### GraphEdit Extension

The existing `moveNodeToEdge` type in the `GraphEdit` union is extended:

```typescript
{ type: 'moveNodeToEdge'; nodeId: string; edgeId: string; validity?: SpliceValidity }
```

`validity` defaults to `'full'` if omitted (backward compatible).

### applyGraphEdit — moveNodeToEdge Implementation

Replaces the current stub that throws. Performs atomically:

**Source-side cleanup:**
1. Find all edges connected to the node (`edgesOf(model, nodeId)`)
2. If no edges: skip (disconnected node)
3. If exactly 1 inbound + 1 outbound:
   - Check `canConnect(predecessor, successor)` in the model-without-the-node
   - If valid: remove both edges, add auto-join edge (predecessor → successor)
   - If invalid: remove both edges (disconnect)
4. Otherwise: remove all connected edges

**Target-side splice:**
1. Look up the target edge, resolve its source and target nodes
2. Remove the target edge
3. Based on `validity`:
   - `'full'`: add edge source→node AND edge node→target
   - `'source-only'`: add only edge source→node
   - `'target-only'`: add only edge node→target

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
      validity: result.validity,
    });
  }
}
```

Pointer events on node bodies are wired in `ReactFlowApp` — the coordinator
attaches to pointerdown on `.stencil-decoration-wrapper` elements that are NOT
connection ports.

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

.edge-splice-partial .react-flow__edge-path {
  stroke: var(--pages-warning-9);
  stroke-width: 3px;
  filter: drop-shadow(0 0 4px var(--pages-warning-9));
  transition: stroke-width 100ms, filter 100ms;
}
```

Edge highlight classes are applied to the `.react-flow__edge[data-id="..."]`
element wrapping the SVG path.

## Handle Shrinking (Prerequisite)

This is a **separate, prerequisite change** — not bundled into edge-splice.
It affects all node interactions.

Current state: source handles cover 100% of node surface (full-node invisible
handles). This prevents body-drag because every pointerdown starts a connection.

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

## Events

The coordinator does not emit custom events — it returns a `DragEndResult` to
GraphCanvas. GraphCanvas dispatches the mutation via `onMutation`. If consumers
need drag lifecycle events, they can be added later via `emitPagesEvent`:

- `graph:node:move:start` — drag started
- `graph:node:move:end` — drag completed (splice or cancel)

## Testing

- **Unit tests** for `canSpliceOntoEdge` — all three validity states with
  various grammar configurations
- **Unit tests** for `applyGraphEdit('moveNodeToEdge')` — source cleanup
  (auto-join, disconnect, no edges) × target splice (full, source-only,
  target-only)
- **Unit tests** for `NodeMoveCoordinator` — mock pointer events, verify
  ghost class toggle, clone creation/removal, edge highlight state
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
- `graph-renderer/src/editing/edit-policy.ts:37` — existing `getInsertableTypes`
- `graph-core/src/edit.ts:102` — `splitEdge` (reference for target-side mechanics)
