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
| 6 | **Drag-to-empty-space** | Drag | Custom | Drag from node center to empty canvas | Node chooser popover at drop point showing connectable types; selection creates + connects node, re-layout |
| 7 | **Node-on-edge drop** | Drag | Custom | Drag existing node onto an edge | Remove node from current position, insert at edge point, rewire edges, re-layout |
| 8 | **Node deletion** | Action | Application | Delete key or context menu | Auto-join (1-in, 1-out leaf), popover for ambiguous, cascade for containers. Clean up dangling edges. Re-layout |
| 9 | **Edge deletion** | Action | Application | Context menu on edge | Remove edge, re-layout |
| 10 | **Context menu** | Click | Application | Right-click node or edge | Show available actions (connect, insert, delete, properties) |
| 11 | **Multi-select delete** | Action | Application | Delete key with multiple nodes selected | Per-node delete strategy, re-layout once |

## 4. Architecture

### 4.1 Package placement

All new infrastructure goes into existing packages — no new packages created.

| Package | What it gains | Why here |
|---|---|---|
| `graph-renderer` | `EditPolicy` SPI interface, `setEditPolicy()` registration, `isValidConnection` wiring, palette drag utilities (ghost, hit-test, viewport transform) | Framework tier — owns interactions and rendering |
| `graph-core` | No changes | Pure data contract preserved (D10) |
| `casehub-diagram` (blocks-ui) | `<casehub-diagram-palette>` component, `EditPolicy` implementation for case domain, node chooser popover, context menu, keyboard shortcuts | Domain-specific UI shell |
| `graph-stencil-case` (blocks-ui) | `CaseEditPolicy` implementing `EditPolicy` | Domain adapter with full case definition knowledge |

### 4.2 EditPolicy SPI (defined in graph-renderer)

```typescript
interface EditPolicy {
  canConnect(source: GraphNode, target: GraphNode, model: GraphModel): boolean;
  getInsertableTypes(edge: GraphEdge, model: GraphModel): StencilTypeInfo[];
  getCreatableTypes(nearNode: GraphNode | null, model: GraphModel): StencilTypeInfo[];
  canDelete(node: GraphNode, model: GraphModel): boolean;
  getDeleteStrategy(node: GraphNode, model: GraphModel): DeleteStrategy;
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

interface StencilTypeInfo {
  type: string;
  label: string;
  icon: string;
  group?: string;
}

function setEditPolicy(policy: EditPolicy): void;
function getEditPolicy(): EditPolicy | undefined;
```

`EditPolicy` is the single domain integration point. graph-renderer calls it during interactions. The domain adapter (graph-stencil-case) implements it with full access to `getGrammar()` for static rules plus domain-specific logic.

`StencilTypeInfo` is the shared data contract for the palette and the node chooser popover — both display the same type information, filtered differently by context. The palette shows all registered types; the node chooser filters via `getInsertableTypes()` or `getCreatableTypes()` based on the interaction.

`DeleteStrategy` encodes the auto-decision logic from D6. `auto-join` bridges predecessor to successor. `disconnect` removes dangling edges. `prompt` shows a popover with explicit options (for ambiguous multi-edge cases). `cascade` removes the node and its subtree (for container nodes with children).

### 4.3 Default EditPolicy

graph-renderer provides a `defaultEditPolicy()` that derives all answers from `StencilGrammar` static rules:

- `canConnect`: checks `allowedTo`/`allowedFrom` and cardinality limits
- `getInsertableTypes`: returns types whose grammar allows inbound from source type AND outbound to target type of the split edge
- `getCreatableTypes`: returns all registered stencil types (unfiltered)
- `canDelete`: returns `true` for all nodes
- `getDeleteStrategy`: auto-join for leaf nodes with 1 inbound + 1 outbound; cascade for nodes with children; disconnect otherwise

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

A lightweight Lit component rendered in DOM space (not React Flow coordinate space). Positioned using `reactFlowInstance.flowToScreenPosition()` at the interaction point. Contains a filtered list of `StencilTypeInfo[]` grouped by `group` field. Includes a search input when the list exceeds 8 items. Dismisses on selection, Escape, or click-outside.

ARIA: `role="listbox"` with `role="option"` children. Search input has `role="searchbox"` with `aria-controls` pointing to the listbox.

### 4.6 Interaction wiring

```
casehub-diagram (shell)
├── palette → pointer events → PaletteDragHandler (ghost, hit-test, drop)
├── canvas (GraphCanvas/ReactFlowApp)
│   ├── onConnect → EditPolicy.canConnect → applyEdit (create edge)
│   ├── onReconnect → EditPolicy.canConnect → applyEdit (rewire edge)
│   ├── onEdgeClick → node chooser → applyEdit (split edge, insert node)
│   ├── onPaneClick → node chooser → applyEdit (create node)
│   └── isValidConnection → EditPolicy.canConnect (drop zone feedback)
├── keyboard (Delete) → EditPolicy.getDeleteStrategy → applyEdit or popover
├── context menu (right-click) → available actions from EditPolicy
└── undo stack → pushUndo() before every applyEdit
```

### 4.7 Mutation flow

All 11 interactions converge on the same mutation path:

```
Interaction → pushUndo(currentYaml) → domain adapter.applyEdit(yaml, edit) → re-parse → ELK layout → React Flow render
```

Every mutation follows this path: push undo, mutate YAML via CST-preserving domain adapter, re-parse to GraphModel, re-layout with ELK, re-render. Compound operations (palette drag → create + connect) are a single undo unit — one `pushUndo()` call before the compound mutation.

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
| **Properties** | Always | Selects node, opens property palette |

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

## 8. Palette Drag Handler

The only true custom drag interaction. All other drag interactions (connection drawing, edge reconnection, box selection, node movement) are handled by React Flow natively.

### 8.1 Lifecycle

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

### 8.2 Hit-testing

Hit-testing during palette drag uses the graph model's node positions (from the last ELK layout result) and edge paths:

1. Convert pointer screen coordinates to flow coordinates
2. Check each node's bounding box (position + dimensions from layout result) for containment
3. Check each edge for proximity (point-to-line-segment distance < threshold)
4. Nearest hit wins (node takes priority over edge)
5. Validate hit against EditPolicy: node hit → `canConnect(newNode, targetNode)`, edge hit → `getInsertableTypes(edge)` must include the dragged type

### 8.3 Viewport transform utilities

```typescript
interface ViewportUtils {
  screenToFlow(screenX: number, screenY: number): { x: number; y: number };
  flowToScreen(flowX: number, flowY: number): { x: number; y: number };
}
```

Wraps `reactFlowInstance.screenToFlowPosition()` and `flowToScreenPosition()`. Exposed from GraphCanvas as a method so the palette drag handler (in the Lit shell) can coordinate across the DOM boundary.

## 9. Multi-Select Batch Delete

When multiple nodes are selected and Delete is pressed:

1. Compute `getDeleteStrategy()` for each selected node
2. If all strategies are unambiguous (auto-join or disconnect or cascade): execute all, re-layout once
3. If any strategy is `prompt`: show a single summary popover listing the ambiguous nodes and their options, then execute all on confirmation
4. Undo restores the entire batch as a single undo unit

Edge case: if deleting node A would auto-join its neighbours, but neighbour B is also selected for deletion, process in topological order (leaves first, then their parents). This prevents joining to a node that's about to be deleted.

## 10. Implementation Phasing

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

## 11. Protocols Consulted

- **aria-interaction-contract** (PP-20260817-a11y01) — ARIA roles for palette, context menu, node chooser
- **pages-event-contract** (PP-20260705-bac842) — event emission patterns
- **web-component-strategy** (PP-20260705-c7687d) — Lit component conventions, Shadow DOM decisions

## 12. References

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
- `packages/pages-runtime/src/dock-drag.ts` — existing pointer-event drag pattern
- GE-20260825-309197 — coordinator pattern for multi-phase interaction state machines
- GE-20260826-ee71b5 — DOM event bubbling with stopPropagation for nested DnD scoping
- GE-20260809-2cbc61 — ReactFlow vs D3 for graph rendering (informed D4 hybrid approach)
