# Design: Diagram Editing Infrastructure — Interactive Graph Mutation UX

**Date:** 2026-08-26
**Parent spec:** `docs/specs/2026-08-01-visual-diagram-editor-design.md`
**Evolves:** Phase 4 (Structural Editing)
**Status:** Draft

---

## 1. Problem & Scope

The visual diagram editor spec (Phase 4) defines basic structural editing — add, remove, replace nodes via palette clicks and toolbar actions. This is insufficient for a productive graph editing UX. Users need to draw connections, insert nodes inline, drag from palettes, and delete with intelligent reconnection — all with real-time visual feedback showing what's valid and what isn't.

This spec evolves Phase 4 into a full interactive graph editing UX with 11 interaction types. All interactions end with ELK auto-layout recalculation — there is no free-form node positioning. The auto-layout constraint makes these interactions tractable: every mutation produces a complete, consistently-laid-out graph without manual position management.

**In scope:**
- Palette infrastructure (grouped, show/hide, any diagram type via SPI)
- Drag-to-connect (node to node)
- Edge insertion (click edge, choose node to insert)
- Node-on-edge drop (drag existing node onto edge)
- Node deletion with intelligent reconnection
- Drag-to-empty-space creation
- Edge deletion
- Edge reconnection (drag endpoint to new target)
- Right-click context menu
- Multi-select batch delete
- Inline node chooser popover
- EditPolicy SPI for domain-specific constraint logic

**Out of scope:**
- Free-form drag positioning (deferred — auto-layout only)
- Collaborative editing
- Undo/redo implementation (already spec'd in parent spec §2.6; this spec defines integration points only)
- Property palette (separate issue #373)
- Runtime overlay interactions

## 2. Decisions

See `decisions.md` in this directory for the full decision log (D1–D11) including review revisions.

Key decisions summarised:

| # | Decision | Choice |
|---|---|---|
| D1 | Scope | Evolution of Phase 4 |
| D2 | Palette home | `casehub-diagram` (blocks-ui), Lit + Shadow DOM |
| D3 | Constraint SPI | Static grammar only in graph-core; dynamic validation in EditPolicy |
| D4 | Drag model | Hybrid: React Flow connections + custom pointer events for palette drag |
| D5 | Node chooser | Inline popover at interaction point |
| D6 | Delete UX | Context-dependent auto-decision with undo |
| D7 | Coordination | Per-handler architecture; React Flow is the coordinator for native interactions |
| D8 | Edit policy | Full EditPolicy SPI in graph-renderer |
| D9 | Interaction mapping | 11 interactions mapped to owner (React Flow / custom / application) |
| D10 | graph-core boundary | Pure data, no callbacks |
| D11 | Undo integration | YAML snapshot before mutation; compound ops are single undo unit |

## 3. Interaction Catalog

| # | Interaction | Category | Owner | Trigger | Result |
|---|---|---|---|---|---|
| 1 | **Palette drag** | Drag | Custom | Drag stencil from palette onto canvas | Create node at drop position, auto-connect if dropped near valid target, re-layout |
| 2 | **Connection drawing** | Drag | React Flow | Drag from Handle on source node to Handle on target node | Create edge (validated via `isValidConnection` → EditPolicy.canConnect), re-layout |
| 3 | **Edge reconnection** | Drag | React Flow | Drag existing edge endpoint to new target | Reconnect edge (validated), re-layout |
| 4 | **Edge insertion** | Click→popover | React Flow event | Click on edge → `onEdgeClick` | Node chooser popover at click point showing insertable types; selection splits edge, creates node, re-layout |
| 5 | **Empty-space creation** | Click→popover | React Flow event | Click empty canvas → `onPaneClick` | Node chooser popover at click point showing creatable types; selection creates node, re-layout |
| 6 | **Drag-to-empty-space** | Drag→detect | React Flow + app | Drag node (React Flow native), detect empty-space drop via `onNodeDragStop` | Node chooser popover at drop point showing connectable types; selection creates + connects node, re-layout |
| 7 | **Node-on-edge drop** | Drag→detect | React Flow + app | Drag node (React Flow native), detect edge proximity via `onNodeDragStop` | Remove node from old edges, insert at edge point, rewire edges, re-layout |
| 8 | **Node deletion** | Action | Application | Delete key or context menu | Auto-join (1-in, 1-out leaf), popover for ambiguous, cascade for containers. Clean up dangling edges. Re-layout |
| 9 | **Edge deletion** | Action | Application | Context menu on edge | Remove edge, re-layout |
| 10 | **Context menu** | Click | Application | Right-click node or edge | Show available actions (connect, insert, delete, properties) |
| 11 | **Multi-select delete** | Action | Application | Delete key with multiple nodes selected | Per-node delete strategy, re-layout once |

## 4. Architecture

### 4.1 Package placement

All new infrastructure goes into existing packages — no new packages created.

| Package | What it gains | Why here |
|---|---|---|
| `graph-renderer` | `EditPolicy` SPI interface, `setEditPolicy()` registration, `isValidConnection` wiring, palette drag utilities (ghost, hit-test, viewport transform), `GraphEdit` discriminated union, ReactFlowApp API expansion | Framework tier — owns interactions and rendering |
| `graph-core` | Edge operations: `addEdge`, `removeEdge`, `splitEdge`, `reconnectEdge` | Edit operations that were missing for edge mutations (R1-05) |
| `casehub-diagram` (blocks-ui) | `<casehub-diagram-palette>` component, `EditPolicy` implementation for case domain, node chooser popover, context menu, keyboard shortcuts | Domain-specific UI shell |
| `graph-stencil-case` (blocks-ui) | `CaseEditPolicy` implementing `EditPolicy` | Domain adapter with full case definition knowledge |

### 4.2 EditPolicy SPI (defined in graph-renderer)

```typescript
interface EditPolicy {
  canConnect(source: GraphNode, target: GraphNode, model: GraphModel, edgeType?: string): boolean;
  getInsertableTypes(edge: GraphEdge, model: GraphModel): StencilTypeInfo[];
  getCreatableTypes(nearNode: GraphNode | null, model: GraphModel): StencilTypeInfo[];
  canDelete(node: GraphNode, model: GraphModel): boolean;
  getDeleteStrategy(node: GraphNode, model: GraphModel, deletionSet?: ReadonlySet<string>): DeleteStrategy;
}

type DeleteStrategy =
  | { type: 'auto-join' }
  | { type: 'disconnect' }
  | { type: 'prompt'; options: DeleteOption[] }
  | { type: 'cascade' };

interface DeleteOption {
  label: string;
  strategy: 'join' | 'disconnect';
  targetNodeId?: string;
}

type StencilTypeInfo = Pick<StencilDescriptor, 'type' | 'label' | 'icon'> & { group?: string };

function setEditPolicy(policy: EditPolicy): void;
function getEditPolicy(): EditPolicy | undefined;
```

`EditPolicy` is the single domain integration point. graph-renderer calls it during interactions. The domain adapter (graph-stencil-case) implements it with full access to `getGrammar()` for static rules plus domain-specific logic.

`canConnect` accepts an optional `edgeType` for domains with multiple edge types (e.g., capability dispatch vs subCase spawn). When omitted, the domain infers the edge type from the source/target node types. The Case domain currently has a single implicit edge type with semantics inferred from node types, so the parameter is optional.

`getDeleteStrategy` accepts an optional `deletionSet` — the set of node IDs being deleted in a batch operation. This allows the strategy to account for batch context: auto-joining to a node that is itself being deleted is wrong, so the strategy downgrades to `disconnect` when the join target is in the deletion set.

`StencilTypeInfo` is derived from `StencilDescriptor` (via `Pick`) to avoid type drift. It is the shared data contract for the palette and the node chooser popover — both display the same type information, filtered differently by context. The palette normalises both `StencilDescriptor` (from stencil registry) and `WorkStencil` (from work registry) into `StencilTypeInfo[]` via a mapping step: `WorkStencil.name` → `type`, `WorkStencil.displayName` → `label`, `WorkStencil.category` → `group`.

`DeleteStrategy` encodes the auto-decision logic from D6. `auto-join` bridges predecessor to successor. `disconnect` removes dangling edges. `prompt` shows a popover with explicit options (for ambiguous multi-edge cases). `cascade` removes the node and its subtree (for container nodes with children).

### 4.3 Default EditPolicy

graph-renderer provides a `defaultEditPolicy()` that derives all answers from `StencilGrammar` static rules:

- `canConnect`: checks `allowedTo`/`allowedFrom` and cardinality limits (ignores `edgeType`)
- `getInsertableTypes`: returns types whose grammar allows inbound from source type AND outbound to target type of the split edge
- `getCreatableTypes`: returns stencil types filtered by containment rules — types with `containment.allowedParentTypes` that require a specific parent are excluded from root-level creation to prevent structurally invalid nodes
- `canDelete`: returns `true` for all nodes
- `getDeleteStrategy`: auto-join for leaf nodes with 1 inbound + 1 outbound (unless join target is in `deletionSet`); cascade for nodes with children; disconnect otherwise

Domain adapters override this default with domain-specific logic. If no EditPolicy is registered, `defaultEditPolicy()` is used as fallback.

### 4.4 Palette component (in casehub-diagram, blocks-ui)

`<casehub-diagram-palette>` is a Lit component with Shadow DOM. It consumes `getAllStencils()` from graph-renderer's stencil registry and `CategoryIndex` from graph-work-registry. It renders grouped stencil items with show/hide group toggles.

```typescript
interface PaletteGroup {
  name: string;
  label: string;
  icon?: string;
  collapsed: boolean;
  items: StencilTypeInfo[];
}
```

Groups are collapsible `<details>` elements. Collapse state persists in localStorage keyed by diagram type (`pages-palette-${diagramType}-${groupName}`). The palette emits pointer events for drag initiation — the drag handler in the diagram shell manages the ghost element and hit-testing against the canvas.

ARIA: `role="toolbar"` with `aria-orientation="vertical"`, `aria-label="Node palette"`. Each group is a `role="group"` with `aria-label` set to the group name. Each item is `role="button"` with `aria-label` set to the stencil label.

### 4.5 Node chooser popover

A lightweight Lit component with Shadow DOM, rendered in DOM space (not React Flow coordinate space). Positioned using the viewport bridge (§4.8) to convert flow coordinates to screen coordinates. Contains a filtered list of `StencilTypeInfo[]` grouped by `group` field. Includes a search input when the list exceeds 8 items. Dismisses on selection, Escape, click-outside, or viewport change (pan/zoom).

Dismissing on viewport change is the simplest correct behaviour — the popover is anchored to a screen position, not a graph position, so it would become disconnected from its anchor point during pan/zoom.

ARIA: `role="listbox"` with `role="option"` children. Search input has `role="searchbox"` with `aria-controls` pointing to the listbox.

### 4.6 Interaction wiring and mutation orchestration

The mutation path crosses a package boundary: EditPolicy (graph-renderer) makes editing decisions, but the domain adapter (graph-stencil-case, blocks-ui) executes the YAML mutation. GraphCanvas cannot import from graph-stencil-case — the dependency goes the other way.

The bridge is a **mutation callback** registered on GraphCanvas:

```typescript
// GraphCanvas exposes a mutation callback property
@property({ attribute: false })
onMutation?: (edit: GraphEdit) => void;

// casehub-diagram wires it:
canvas.onMutation = (edit) => {
  this._pushUndo();
  const newYaml = this._adapter.applyEdit(this._currentYaml, edit);
  this._fullRender(newYaml);
  this._emitEditEvent(edit);
};
```

The flow: React Flow callback (onConnect, onEdgeClick, etc.) → GraphCanvas builds a `GraphEdit` → calls `onMutation(edit)` → casehub-diagram receives it, pushes undo, calls the domain adapter's `applyEdit`, and triggers re-render.

```
casehub-diagram (shell)
├── palette → pointer events → PaletteDragHandler (ghost, hit-test, drop)
├── canvas (GraphCanvas/ReactFlowApp)
│   ├── onConnect → EditPolicy.canConnect → onMutation(createEdge)
│   ├── onReconnect → EditPolicy.canConnect → onMutation(reconnectEdge)
│   ├── onEdgeClick → node chooser → onMutation(splitEdge + addNode)
│   ├── onPaneClick → node chooser → onMutation(addNode)
│   ├── onNodeDragStop → detect edge drop → onMutation(moveNodeToEdge)
│   └── isValidConnection → EditPolicy.canConnect (drop zone feedback)
├── keyboard (Delete) → EditPolicy.getDeleteStrategy → onMutation(removeNode/removeEdge)
├── context menu (right-click) → available actions from EditPolicy
└── undo stack → pushUndo() before every onMutation
```

### 4.7 GraphEdit discriminated union

All edit operations are described by a `GraphEdit` type. This is the contract between the interaction layer (graph-renderer) and the domain adapter (blocks-ui):

```typescript
type GraphEdit =
  | { type: 'addNode'; nodeType: string; properties?: Record<string, unknown> }
  | { type: 'removeNode'; nodeId: string; strategy: DeleteStrategy }
  | { type: 'addEdge'; sourceId: string; targetId: string; edgeType?: string }
  | { type: 'removeEdge'; edgeId: string }
  | { type: 'reconnectEdge'; edgeId: string; newTargetId: string }
  | { type: 'splitEdge'; edgeId: string; insertNodeType: string }
  | { type: 'moveNodeToEdge'; nodeId: string; edgeId: string }
  | { type: 'compound'; edits: GraphEdit[] };
```

The `compound` variant groups multiple edits into a single undo unit (e.g., palette drag → addNode + addEdge). The domain adapter's `applyEdit(yaml: string, edit: GraphEdit): string` applies the edit to the YAML string and returns the modified YAML.

### 4.8 Viewport transform bridge (Lit ↔ React)

React Flow's viewport transform methods (`screenToFlowPosition`, `flowToScreenPosition`) are only available via the `useReactFlow()` hook inside a React component. GraphCanvas (Lit) needs these for the palette drag handler and node chooser positioning.

The bridge uses a React component inside ReactFlowApp that captures the instance and exposes it via callback:

```typescript
// Inside ReactFlowApp.tsx
function ViewportBridge({ onReactFlowReady }: { onReactFlowReady: (instance: ReactFlowInstance) => void }) {
  const instance = useReactFlow();
  useEffect(() => { onReactFlowReady(instance); }, [instance, onReactFlowReady]);
  return null;
}

// ReactFlowApp renders it inside the provider:
<ReactFlowProvider>
  <ReactFlow ...>
    <ViewportBridge onReactFlowReady={onReactFlowReady} />
  </ReactFlow>
</ReactFlowProvider>
```

GraphCanvas receives the instance via the `onReactFlowReady` callback prop and stores it:

```typescript
// GraphCanvas.ts
private _reactFlowInstance?: ReactFlowInstance;

screenToFlow(screenX: number, screenY: number): { x: number; y: number } | undefined {
  return this._reactFlowInstance?.screenToFlowPosition({ x: screenX, y: screenY });
}

flowToScreen(flowX: number, flowY: number): { x: number; y: number } | undefined {
  return this._reactFlowInstance?.flowToScreenPosition({ x: flowX, y: flowY });
}
```

These methods are callable from the Lit shell (casehub-diagram) for palette drag hit-testing and node chooser positioning.

### 4.9 ReactFlowApp API expansion

ReactFlowApp requires 7 new callback props to support the editing interactions:

| Prop | Type | Interaction |
|---|---|---|
| `onConnect` | `(connection: Connection) => void` | #2 Connection drawing |
| `isValidConnection` | `(connection: Connection) => boolean` | #2, #3 Drop zone validation |
| `onReconnect` | `(oldEdge: Edge, newConnection: Connection) => void` | #3 Edge reconnection |
| `onPaneClick` | `(event: MouseEvent) => void` | #5 Empty-space creation |
| `onNodeDragStop` | `(event: MouseEvent, node: Node, nodes: Node[]) => void` | #6, #7 Post-drag detection |
| `onReactFlowReady` | `(instance: ReactFlowInstance) => void` | Viewport bridge (§4.8) |
| `onPaneContextMenu` | `(event: MouseEvent) => void` | #10 Canvas context menu |

Additionally, the `ReactFlow` element gains:
- `reconnectEdges` prop (enables edge endpoint dragging)
- `onContextMenu` handler on node/edge elements (for right-click menus)

Each callback is threaded through GraphCanvas's `_renderReact()` and exposed as either a Lit `@property` or emitted as a `pages-event`.

### 4.10 Mutation flow

All 11 interactions converge on the same mutation path:

```
Interaction → GraphCanvas.onMutation(edit) → casehub-diagram._pushUndo() → adapter.applyEdit(yaml, edit) → re-parse → ELK layout → React Flow render → emit pages-event
```

Compound operations (palette drag → create + connect) use `{ type: 'compound', edits: [...] }` — the domain adapter applies all sub-edits in sequence, and the shell pushes a single undo snapshot before the compound.

### 4.11 Editing events

New `pages-event` topics for mutation notifications (per pages-event-contract, colon-separated):

| Topic | Payload | When |
|---|---|---|
| `graph:node:create` | `{ nodeId: string, nodeType: string }` | Node added |
| `graph:node:delete` | `{ nodeId: string, nodeType: string }` | Node removed |
| `graph:edge:create` | `{ edgeId: string, sourceId: string, targetId: string }` | Edge added |
| `graph:edge:delete` | `{ edgeId: string }` | Edge removed |
| `graph:edge:reconnect` | `{ edgeId: string, oldTargetId: string, newTargetId: string }` | Edge endpoint moved |
| `graph:edit:undo` | `{}` | Undo executed |
| `graph:edit:redo` | `{}` | Redo executed |

These are emitted by casehub-diagram after the mutation completes and re-layout finishes. They bubble with `composed: true` per protocol.

### 4.12 Edge operations in graph-core

graph-core's `edit.ts` currently has `addNode`, `removeNode`, `replaceNode`. The following edge operations are added:

```typescript
function addEdge(model: GraphModel, edge: GraphEdge): EditResult;
function removeEdge(model: GraphModel, edgeId: string): EditResult;
function reconnectEdge(model: GraphModel, edgeId: string, newTargetId: string): EditResult;
function splitEdge(model: GraphModel, edgeId: string, insertNode: GraphNode): EditResult;
```

Each returns `EditResult { model, violations }` consistent with existing operations. `splitEdge` is a compound operation: remove the original edge, add the new node, add two new edges (source→newNode, newNode→originalTarget). All operations validate against registered grammars.

## 5. Visual Feedback & Drop Zones

### 5.1 Connection drawing (React Flow native)

React Flow's connection system provides visual feedback via `isValidConnection`:
- **Valid target handles**: highlighted (green border pulse via `.react-flow__handle-valid` CSS class)
- **Invalid target handles**: dimmed (React Flow default — handles remain but don't highlight)
- **Connection line**: rendered by React Flow during drag, snaps to nearest valid handle

The `isValidConnection` callback delegates to `EditPolicy.canConnect()`:

```typescript
const isValidConnection = useCallback((connection: Connection) => {
  const policy = getEditPolicy();
  if (!policy) return true;
  const source = nodeById(model, connection.source);
  const target = nodeById(model, connection.target);
  if (!source || !target) return false;
  return policy.canConnect(source, target, model);
}, [model]);
```

### 5.2 Palette drag (custom)

| Phase | Visual |
|---|---|
| Drag starts | Ghost element (stencil icon + label) follows pointer. Palette item gets `dragging` state. |
| Over canvas, no valid target | Ghost shows neutral state. Canvas nodes unchanged. |
| Over canvas, near valid target | Target node highlights (border glow). Ghost snaps toward target. |
| Over canvas edge | Edge highlights (thicker stroke, accent colour). Ghost centers on edge midpoint. |
| Over empty space | Ghost shows "create" indicator (+ badge). |
| Drop on valid target | Create node + connect. Re-layout. |
| Drop on edge | Insert node, split edge. Re-layout. |
| Drop on empty space | Create node at logical position. Re-layout. |
| Escape or drop outside canvas | Cancel. Ghost removed. No mutation. |

Ghost element is rendered in DOM space (not inside React Flow's viewport), positioned at pointer coordinates. Hit-testing converts pointer screen coordinates to React Flow flow coordinates via `reactFlowInstance.screenToFlowPosition()`.

### 5.3 Node-on-edge drop (custom)

When dragging an existing node onto an edge:
- **Valid edge** (EditPolicy allows insertion of this node type at this edge): edge highlights with thicker stroke + accent colour, insertion indicator shows between the two connected nodes
- **Invalid edge**: no highlight, cursor shows not-allowed
- **Drop**: remove node from current position (clean up old edges per D6 strategy), insert at edge point (split edge into two, wire through dropped node), single undo unit, re-layout

### 5.4 Edge reconnection (React Flow native)

React Flow's `reconnectEdges` prop enables dragging edge endpoints. Visual feedback is the same as connection drawing — valid handles highlight, invalid dim. The `onReconnect` callback validates via `EditPolicy.canConnect()` before committing.

### 5.5 Delete feedback

- **Single node, auto-join case**: brief flash showing the join — predecessor and successor edges animate to merge before re-layout
- **Single node, disconnect case**: edges fade out before removal
- **Container node**: subtree dims briefly before cascade removal
- **Cancelled delete**: no visual change

## 6. Context Menu

Right-click on a node or edge opens a context menu. Menu items are determined by the interaction context and EditPolicy.

### 6.1 Node context menu

| Item | Condition | Action |
|---|---|---|
| **Connect to...** | `getCreatableTypes().length > 0` | Opens node chooser with connectable types |
| **Insert after...** | Node has outbound edges | Opens node chooser with insertable types for first outbound edge |
| **Delete** | `canDelete(node)` returns true | Executes delete per D6 strategy |
| **Properties** | Property palette registered | Selects node, opens property palette (#373) |

### 6.2 Edge context menu

| Item | Condition | Action |
|---|---|---|
| **Insert node...** | `getInsertableTypes(edge).length > 0` | Opens node chooser at edge midpoint |
| **Delete connection** | Always | Removes edge, re-layout |

### 6.3 Implementation

The context menu is a Lit component in casehub-diagram (Shadow DOM enabled). Positioned in DOM space at right-click coordinates. Dismissed on selection, Escape, or click-outside.

ARIA: `role="menu"` with `role="menuitem"` children. Focus trap within the menu while open. First item auto-focused on open. Arrow keys navigate items.

## 7. Keyboard Shortcuts

| Key | Context | Action |
|---|---|---|
| `Delete` / `Backspace` | Node(s) selected | Delete per D6 strategy (auto-join/disconnect/popover/cascade) |
| `Delete` / `Backspace` | Edge selected | Delete edge |
| `Escape` | Palette drag active | Cancel drag |
| `Escape` | Node chooser open | Dismiss popover |
| `Escape` | Context menu open | Dismiss menu |
| `Escape` | Node(s) selected | Deselect all |
| `Ctrl+Z` | Any | Undo (parent spec §2.6) |
| `Ctrl+Shift+Z` | Any | Redo (parent spec §2.6) |

Escape priority: innermost active UI element dismisses first (context menu > node chooser > palette drag > selection).

### 7.1 Keyboard accessibility for drag interactions

Drag interactions (#1 palette drag, #2 connection, #3 reconnect, #6 drag-to-empty, #7 node-on-edge) have no direct keyboard equivalents. The context menu provides keyboard-accessible alternatives for all outcomes:

| Drag interaction | Keyboard alternative |
|---|---|
| #1 Palette drag (add node) | Palette item click (keyboard-focusable button) or context menu "Connect to..." |
| #2 Connection drawing | Context menu "Connect to..." on source node |
| #3 Edge reconnection | Context menu "Delete connection" + context menu "Connect to..." on source |
| #6 Drag-to-empty-space | Context menu "Connect to..." on source node |
| #7 Node-on-edge drop | Context menu "Insert after..." on source edge |

Interactions #6 and #7 are mouse-only power-user shortcuts — the context menu and node chooser provide equivalent results via keyboard. This satisfies the aria-interaction-contract (PP-20260817-a11y01) requirement that all interactive elements have accessible alternatives.

## 8. Drag Interaction Details

### 8.0 Interaction #6 and #7 — React Flow node drag with post-drop detection

Interactions #6 (drag-to-empty-space) and #7 (node-on-edge drop) do NOT conflict with React Flow's node drag because they USE it. React Flow handles the node drag natively. On `onNodeDragStop`, application logic inspects the drop position:

1. **Edge proximity check**: convert drop position to flow coordinates, check proximity to each edge (point-to-line-segment distance < threshold). If near a valid edge → interaction #7 (node-on-edge drop).
2. **Empty-space check**: if the node was dragged to a position away from other nodes and edges, and the node moved beyond a minimum threshold (to distinguish from a click) → interaction #6 (drag-to-empty-space, shows node chooser).
3. **Normal move**: if neither condition is met, React Flow's default node move applies — but since we use auto-layout, the node snaps back to its ELK-computed position on re-layout. This is the expected behaviour in auto-layout mode.

Since ELK re-layout repositions all nodes anyway, React Flow's node drag acts as a gesture detector, not a position setter. The visual drag provides the user feedback about their intent; the `onNodeDragStop` handler interprets the drop target.

### 8.1 Palette drag handler

The only true custom drag interaction (pointer events managed entirely by application code). All other drag interactions use React Flow's native drag system.

### 8.2 Lifecycle

```
pointerdown on palette item
  → create ghost element (stencil icon + label, position: fixed)
  → set paletteDragActive = true
  → capture pointer on palette item

pointermove
  → update ghost position (screenX, screenY)
  → convert to flow coordinates via screenToFlowPosition()
  → hit-test against nodes and edges in the model
  → update visual feedback (node highlight, edge highlight, or create indicator)

pointerup
  → if over valid target node: create node + connect
  → if over valid edge: insert node at edge, split edge
  → if over empty canvas: create node at position
  → if outside canvas or no valid target: cancel
  → remove ghost element
  → set paletteDragActive = false
  → re-layout

Escape during drag
  → cancel immediately
  → remove ghost element
  → set paletteDragActive = false
```

### 8.3 Hit-testing

Hit-testing during palette drag uses the graph model's node positions (from the last ELK layout result) and edge paths:

1. Convert pointer screen coordinates to flow coordinates
2. Check each node's bounding box (position + dimensions from layout result) for containment
3. Check each edge for proximity (point-to-line-segment distance < threshold)
4. Nearest hit wins (node takes priority over edge)
5. Validate hit against EditPolicy: node hit → `canConnect(newNode, targetNode)`, edge hit → `getInsertableTypes(edge)` must include the dragged type

### 8.4 Viewport transform

Uses the viewport bridge (§4.8) — `GraphCanvas.screenToFlow()` and `GraphCanvas.flowToScreen()` — to coordinate between the palette's DOM space and the React Flow canvas coordinate system.

## 9. Multi-Select Batch Delete

When multiple nodes are selected and Delete is pressed:

1. Compute `getDeleteStrategy()` for each selected node
2. If all strategies are unambiguous (auto-join or disconnect or cascade): execute all, re-layout once
3. If any strategy is `prompt`: show a single summary popover listing the ambiguous nodes and their options, then execute all on confirmation
4. Undo restores the entire batch as a single undo unit

Batch-aware strategies: `getDeleteStrategy` receives the full `deletionSet` (set of all node IDs being deleted). This allows the strategy to detect when an auto-join target is itself being deleted and downgrade to `disconnect`.

Processing order: leaf nodes first (nodes with no children in the containment tree), then their parents. This is a containment-tree ordering, not a topological sort of the edge graph — it doesn't assume the edge graph is a DAG. Case definitions may contain cycles (e.g., mutually-triggering bindings), which is valid; the containment tree is always a tree.

## 10. Shadow DOM Topology (new components)

Extends the parent spec's shadow DOM topology table:

| Component | Shadow DOM | Rationale |
|---|---|---|
| `<casehub-diagram-palette>` | Enabled | Self-contained Lit component; Shadow DOM provides CSS encapsulation. Per parent spec §2.2. |
| Node chooser popover | Enabled | Self-contained Lit overlay; Shadow DOM prevents style bleeding from host. |
| Context menu | Enabled | Self-contained Lit overlay; Shadow DOM prevents style bleeding. Per §6.3. |
| Ghost element (palette drag) | None | Plain `HTMLElement` created via `document.createElement`, positioned `fixed`. No shadow root — it's a transient visual artifact, not a component. |

## 11. Implementation Phasing

This spec can be implemented in two sub-phases within Phase 4:

### Phase 4a: Foundation + basic editing

- EditPolicy SPI in graph-renderer with `defaultEditPolicy()`
- `isValidConnection` wiring in ReactFlowApp
- React Flow connection drawing (interaction #2)
- React Flow edge reconnection (interaction #3)
- Edge deletion via context menu (interaction #9)
- Node deletion with auto-decision logic (interaction #8)
- Context menu component (interaction #10)
- Keyboard shortcuts
- CaseEditPolicy in graph-stencil-case

### Phase 4b: Drag interactions + advanced editing

- Palette component with grouping and drag initiation
- Palette drag handler with ghost, hit-testing, viewport transform (interaction #1)
- Node chooser popover component
- Edge insertion via click→popover (interaction #4)
- Empty-space creation via click→popover (interaction #5)
- Drag-to-empty-space (interaction #6)
- Node-on-edge drop (interaction #7)
- Multi-select batch delete (interaction #11)

**4a and 4b are sequential** — 4b builds on EditPolicy, context menu, and deletion logic from 4a.

## 12. Protocols Consulted

- **aria-interaction-contract** (PP-20260817-a11y01) — ARIA roles for palette, context menu, node chooser
- **pages-event-contract** (PP-20260705-bac842) — event emission patterns
- **web-component-strategy** (PP-20260705-c7687d) — Lit component conventions, Shadow DOM decisions

## 13. References

- `docs/specs/2026-08-01-visual-diagram-editor-design.md` — parent visual diagram editor spec (Phases 1–7)
- `docs/specs/issue-265-graph-renderer/2026-08-03-stencil-registry-design.md` — StencilDescriptor registry
- `docs/specs/issue-265-graph-renderer/2026-08-03-interaction-layer-design.md` — interaction events (node click, edge click, selection, viewport)
- `docs/specs/issue-373-property-palette/2026-08-26-property-palette-design.md` — property palette SPI
- `packages/graph-core/src/grammar.ts` — StencilGrammar interface (static connection rules)
- `packages/graph-core/src/edit.ts` — addNode, removeNode, replaceNode operations
- `packages/graph-core/src/validator.ts` — constraint validation
- `packages/graph-renderer/src/stencil-wrapper.tsx` — Handle component rendering
- `packages/graph-renderer/src/bridge/ReactFlowApp.tsx` — React Flow wrapper
- `packages/graph-renderer/src/registry/stencil-registry.ts` — stencil registration
- `packages/pages-runtime/src/dock-drag.ts` — existing drag pattern (uses mouse events; palette drag upgrades to pointer events for consolidated mouse/touch/stylus and pointer capture)
- GE-20260825-309197 — coordinator pattern for multi-phase interaction state machines
- GE-20260826-ee71b5 — DOM event bubbling with stopPropagation for nested DnD scoping
- GE-20260809-2cbc61 — ReactFlow vs D3 for graph rendering (informed D4 hybrid approach)
