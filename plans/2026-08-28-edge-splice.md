# Edge-Splice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #388 — Drag node onto edge to splice it in
**Issue group:** #387, #388

**Goal:** Enable dragging a node onto an edge to splice it in — ghosting
the original, showing a clone cursor, highlighting valid edges, and
completing the splice on drop.

**Architecture:** A standalone `NodeMoveCoordinator` (coordinator pattern)
handles the drag lifecycle via pointer events. It uses `EditPolicy` for
splice validation and returns a `DragEndResult` discriminated union. The
existing `moveNodeToEdge` GraphEdit type is extended with a
`sourceCleanup` strategy and implemented in `applyGraphEdit`. Handle
shrinking (#387) is a blocking prerequisite — it shrinks the full-node
source handle to a small port so the node body is available for drag.

**Tech Stack:** TypeScript, React (stencil-wrapper), Lit (GraphCanvas),
React Flow v12, Vitest

## Global Constraints

- `nodesDraggable` stays `false` — auto-layout owns node positions
- graph-core stays pure data — all interaction logic in graph-renderer
- Single undo unit per operation (infra spec D11)
- Vitest for all tests, `vi.mock('@xyflow/react', ...)` in renderer tests
- Source handles use `cursor: crosshair`, target handles stay full-node

---

## Batch 1: Handle Shrinking (blocker #387)

### Task 1: Shrink source handle to visible connection port

**Files:**
- Modify: `packages/graph-renderer/src/stencil-wrapper.tsx:159-191`
- Modify: `packages/graph-renderer/src/bridge/css-isolation.ts:37-48`
- Test: `packages/graph-renderer/src/stencil-wrapper.test.tsx`

**Interfaces:**
- Consumes: React Flow `Handle` component, `Position` enum
- Produces: Source handle renders as a small port (not full-node), node body
  is clickable without triggering connection drag

- [ ] **Step 1: Write failing test — source handle is not full-node**

In `stencil-wrapper.test.tsx`, add:

```typescript
it('renders source handle as a small port, not full-node', () => {
  const { container } = render(<StencilNode {...defaultProps} />);
  const sourceHandle = container.querySelector('.stencil-source-handle');
  expect(sourceHandle).toBeTruthy();
  const style = sourceHandle!.getAttribute('style') ?? '';
  expect(style).not.toContain('width: 100%');
  expect(style).not.toContain('height: 100%');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run stencil-wrapper.test`
Expected: FAIL — source handle currently has `width: 100%; height: 100%`

- [ ] **Step 3: Modify source handle in stencil-wrapper.tsx**

Replace the source handle rendering (lines 187-192). The source handle
becomes a small positioned port at the bottom edge:

```tsx
{!hideHandles && hasSource && grammar?.connections.outbound.max !== 0 &&
  <Handle key="source-full" id={`source-${sourcePos === Position.Bottom ? 'bottom' : 'right'}`}
    type="source" position={sourcePos}
    className="stencil-source-handle"
    style={{
      width: 12, height: 12,
      borderRadius: '50%',
      background: 'var(--pages-neutral-5, #999)',
      border: '2px solid var(--pages-neutral-1, #fafafa)',
      opacity: 1,
      cursor: 'crosshair',
      zIndex: 3,
      transition: 'background 0.15s, transform 0.15s',
    }} />
}
```

The `fullNodeHandle` style is no longer used for the source handle. The
target handle (line 173-177) stays as-is with `fullNodeHandle` at
`zIndex: 1`.

- [ ] **Step 4: Update CSS in css-isolation.ts**

Replace the existing `.stencil-source-handle` and `.react-flow__handle`
rules (lines 37-48) with:

```css
.react-flow__handle {
  border: none;
  background: transparent;
  pointer-events: all;
}
.stencil-source-handle {
  cursor: crosshair;
  pointer-events: all;
}
.stencil-source-handle:hover {
  background: var(--pages-accent-9, #5470c6) !important;
  transform: scale(1.3);
}
.graph-connecting .stencil-source-handle {
  pointer-events: none !important;
  cursor: default;
}
```

Remove the `opacity: 0; width: 1px; height: 1px;` from
`.react-flow__handle` — the target handle's opacity is set inline via
`fullNodeHandle` style, and the source handle is now visible.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add packages/graph-renderer/src/stencil-wrapper.tsx packages/graph-renderer/src/bridge/css-isolation.ts packages/graph-renderer/src/stencil-wrapper.test.tsx
git commit -m "feat(graph): shrink source handle to visible connection port

Replaces full-node invisible source handle with a small visible port at
the bottom edge. Target handle stays full-node for forgiving connection
drops. Node body is now available for drag-to-move gestures.

Refs #387"
```

---

## Batch 2: Splice Validation and Mutation

### Task 2: Add canSpliceOntoEdge to EditPolicy

**Files:**
- Modify: `packages/graph-renderer/src/editing/types.ts:20-26`
- Modify: `packages/graph-renderer/src/editing/edit-policy.ts`
- Create: `packages/graph-renderer/src/editing/splice-validation.ts`
- Test: `packages/graph-renderer/src/editing/edit-policy.test.ts`

**Interfaces:**
- Consumes: `EditPolicy.canConnect`, `edgesOf`, `removeEdge`, `nodeById`
  from graph-core
- Produces:
  - `canSpliceOntoEdge?(edge: GraphEdge, node: GraphNode, model: GraphModel): boolean`
    on `EditPolicy` (optional method)
  - `defaultCanSpliceOntoEdge(policy: EditPolicy, edge: GraphEdge, node: GraphNode, model: GraphModel): boolean`
    standalone fallback in `splice-validation.ts`
  - `buildProjectedModel(model: GraphModel, edge: GraphEdge, node: GraphNode): GraphModel`
    exported from `splice-validation.ts` for testing

- [ ] **Step 1: Write failing test — canSpliceOntoEdge returns true for valid splice**

In `edit-policy.test.ts`, add a new `describe('canSpliceOntoEdge')` block:

```typescript
describe('canSpliceOntoEdge', () => {
  it('returns true when both directions are valid', () => {
    registerGrammar(makeGrammar('start', 1, ['worker']));
    registerGrammar(makeGrammar('worker', 1, ['end']));
    registerGrammar(makeGrammar('end', 0, []));
    registerStencil({ type: 'worker', label: 'Worker', icon: 'w',
      grammar: { type: 'worker', connections: { inbound: { min: 0, max: 10, allowedFrom: [] }, outbound: { min: 0, max: 1, allowedTo: ['end'] } } },
      render: dummyRender });

    const m: GraphModel = {
      nodes: [
        { id: 'a', type: 'start', properties: {} },
        { id: 'b', type: 'end', properties: {} },
        { id: 'x', type: 'worker', properties: {} },
      ],
      edges: [{ id: 'e1', type: 'default', source: 'a', target: 'b' }],
    };
    const edge = m.edges[0]!;
    const nodeX = m.nodes[2]!;
    const policy = defaultEditPolicy();
    expect(policy.canSpliceOntoEdge!(edge, nodeX, m)).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run edit-policy.test`
Expected: FAIL — `policy.canSpliceOntoEdge` is undefined

- [ ] **Step 3: Add optional canSpliceOntoEdge to EditPolicy interface**

In `types.ts` line 25, add after `getDeleteStrategy`:

```typescript
canSpliceOntoEdge?(edge: GraphEdge, node: GraphNode, model: GraphModel): boolean;
```

- [ ] **Step 4: Create splice-validation.ts with projected-model logic**

Create `packages/graph-renderer/src/editing/splice-validation.ts`:

```typescript
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import { nodeById, edgesOf, removeEdge } from '@casehubio/graph-core';
import type { EditPolicy } from './types.js';

export function buildProjectedModel(
  model: GraphModel,
  targetEdge: GraphEdge,
  draggedNode: GraphNode,
): GraphModel {
  const edgesToRemove = new Set<string>();
  edgesToRemove.add(targetEdge.id);
  for (const e of edgesOf(model, draggedNode.id)) {
    edgesToRemove.add(e.id);
  }
  return {
    ...model,
    edges: model.edges.filter(e => !edgesToRemove.has(e.id)),
  };
}

export function defaultCanSpliceOntoEdge(
  policy: EditPolicy,
  edge: GraphEdge,
  node: GraphNode,
  model: GraphModel,
): boolean {
  const projected = buildProjectedModel(model, edge, node);
  const source = nodeById(projected, edge.source);
  const target = nodeById(projected, edge.target);
  if (!source || !target) return false;
  return policy.canConnect(source, node, projected)
      && policy.canConnect(node, target, projected);
}
```

- [ ] **Step 5: Implement canSpliceOntoEdge in defaultEditPolicy**

In `edit-policy.ts`, import and use:

```typescript
import { defaultCanSpliceOntoEdge } from './splice-validation.js';
```

Add to the policy object (after `getDeleteStrategy`):

```typescript
canSpliceOntoEdge(edge: GraphEdge, node: GraphNode, model: GraphModel): boolean {
  return defaultCanSpliceOntoEdge(policy, edge, node, model);
},
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run edit-policy.test`
Expected: PASS

- [ ] **Step 7: Write additional tests — cardinality limits, invalid types**

Add tests for:
- Returns false when source grammar forbids connection to node type
- Returns false when node grammar forbids connection to target type
- Returns true even when source is at outbound max (projected model frees the slot)
- Returns false when edge is connected to the dragged node (caller responsibility — canSpliceOntoEdge itself returns true, coordinator skips)
- `defaultCanSpliceOntoEdge` fallback respects custom `canConnect` on a non-default policy

- [ ] **Step 8: Run all tests, commit**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run`

```bash
git add packages/graph-renderer/src/editing/types.ts packages/graph-renderer/src/editing/edit-policy.ts packages/graph-renderer/src/editing/splice-validation.ts packages/graph-renderer/src/editing/edit-policy.test.ts
git commit -m "feat(graph): add canSpliceOntoEdge to EditPolicy with projected-model validation

Uses a projected model that removes the target edge and the dragged
node's existing edges before checking canConnect. This avoids false
negatives when nodes are at cardinality limits.

Refs #388"
```

### Task 3: Implement moveNodeToEdge in applyGraphEdit

**Files:**
- Modify: `packages/graph-renderer/src/editing/types.ts:35`
- Modify: `packages/graph-renderer/src/editing/apply-graph-edit.ts:61-62`
- Test: `packages/graph-renderer/src/editing/edit-policy.test.ts`

**Interfaces:**
- Consumes: `edgesOf`, `inboundEdges`, `outboundEdges`, `removeEdge`,
  `addEdge`, `nodeById` from graph-core.
  `SourceCleanupStrategy` type from `types.ts`.
- Produces:
  - `SourceCleanupStrategy = 'auto-join' | 'disconnect'` exported from
    `types.ts`
  - `moveNodeToEdge` GraphEdit variant:
    `{ type: 'moveNodeToEdge'; nodeId: string; edgeId: string; sourceCleanup: SourceCleanupStrategy }`
  - `applyGraphEdit` handles `moveNodeToEdge` without throwing

- [ ] **Step 1: Write failing test — moveNodeToEdge with auto-join**

```typescript
describe('applyGraphEdit moveNodeToEdge', () => {
  it('splices node onto edge with auto-join at source', () => {
    registerGrammar(makeGrammar('start', 10, []));
    registerGrammar(makeGrammar('worker', 10, []));
    registerGrammar(makeGrammar('end', 0, []));

    const m: GraphModel = {
      nodes: [
        { id: 'a', type: 'start', properties: {} },
        { id: 'x', type: 'worker', properties: {} },
        { id: 'b', type: 'end', properties: {} },
        { id: 'p', type: 'start', properties: {} },
        { id: 'q', type: 'end', properties: {} },
      ],
      edges: [
        { id: 'e-ax', type: 'default', source: 'a', target: 'x' },
        { id: 'e-xb', type: 'default', source: 'x', target: 'b' },
        { id: 'e-pq', type: 'flow', source: 'p', target: 'q' },
      ],
    };

    const result = applyGraphEdit(m, {
      type: 'moveNodeToEdge',
      nodeId: 'x',
      edgeId: 'e-pq',
      sourceCleanup: 'auto-join',
    });

    // Source side: a→x and x→b removed, a→b auto-joined
    expect(result.model.edges.find(e => e.id === 'e-ax')).toBeUndefined();
    expect(result.model.edges.find(e => e.id === 'e-xb')).toBeUndefined();
    const joinEdge = result.model.edges.find(e => e.source === 'a' && e.target === 'b');
    expect(joinEdge).toBeTruthy();
    expect(joinEdge!.type).toBe('default'); // inherits inbound edge type

    // Target side: e-pq removed, p→x and x→q created
    expect(result.model.edges.find(e => e.id === 'e-pq')).toBeUndefined();
    const preEdge = result.model.edges.find(e => e.source === 'p' && e.target === 'x');
    const postEdge = result.model.edges.find(e => e.source === 'x' && e.target === 'q');
    expect(preEdge).toBeTruthy();
    expect(postEdge).toBeTruthy();
    expect(preEdge!.type).toBe('flow'); // inherits original edge type
    expect(postEdge!.type).toBe('flow');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run edit-policy.test`
Expected: FAIL — throws "moveNodeToEdge requires domain adapter"

- [ ] **Step 3: Add SourceCleanupStrategy to types.ts**

In `types.ts`, add before the `GraphEdit` type:

```typescript
export type SourceCleanupStrategy = 'auto-join' | 'disconnect';
```

Update the `moveNodeToEdge` variant (line 35):

```typescript
| { readonly type: 'moveNodeToEdge'; readonly nodeId: string; readonly edgeId: string; readonly sourceCleanup: SourceCleanupStrategy }
```

- [ ] **Step 4: Implement moveNodeToEdge in applyGraphEdit**

Replace the throw at line 61-62 with:

```typescript
case 'moveNodeToEdge': {
  let result: EditResult = { model, violations: [] };

  // Source-side cleanup
  if (edit.sourceCleanup === 'auto-join') {
    const inEdges = inboundEdges(model, edit.nodeId);
    const outEdges = outboundEdges(model, edit.nodeId);
    for (const e of [...inEdges, ...outEdges]) {
      result = removeEdge(result.model, e.id);
    }
    if (inEdges.length === 1 && outEdges.length === 1) {
      const joinEdge: GraphEdge = {
        id: nextId('edge'),
        type: inEdges[0]!.type,
        source: inEdges[0]!.source,
        target: outEdges[0]!.target,
      };
      result = addEdge(result.model, joinEdge);
    }
  } else {
    const connected = [...inboundEdges(model, edit.nodeId), ...outboundEdges(model, edit.nodeId)];
    for (const e of connected) {
      result = removeEdge(result.model, e.id);
    }
  }

  // Target-side splice
  const targetEdge = model.edges.find(e => e.id === edit.edgeId);
  if (!targetEdge) throw new Error(`Edge ${edit.edgeId} not found`);

  result = removeEdge(result.model, edit.edgeId);
  result = addEdge(result.model, {
    id: nextId('edge'),
    type: targetEdge.type,
    source: targetEdge.source,
    target: edit.nodeId,
  });
  result = addEdge(result.model, {
    id: nextId('edge'),
    type: targetEdge.type,
    source: edit.nodeId,
    target: targetEdge.target,
  });

  return result;
}
```

Add import for `edgesOf` if not already imported from `@casehubio/graph-core`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run edit-policy.test`
Expected: PASS

- [ ] **Step 6: Write additional tests**

Add tests for:
- `sourceCleanup: 'disconnect'` — all edges removed, no auto-join
- Disconnected node (no edges) — only target-side splice happens
- Edge type inheritance — new edges carry the original edge's type

- [ ] **Step 7: Run all tests, commit**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run`

```bash
git add packages/graph-renderer/src/editing/types.ts packages/graph-renderer/src/editing/apply-graph-edit.ts packages/graph-renderer/src/editing/edit-policy.test.ts
git commit -m "feat(graph): implement moveNodeToEdge in applyGraphEdit

Replaces the stub that threw. Performs source-side cleanup (auto-join or
disconnect) and target-side splice atomically. New edges inherit the
original edge's type.

Refs #388"
```

---

## Batch 3: Drag Coordinator and Wiring

### Task 4: Create NodeMoveCoordinator with visual feedback

**Files:**
- Create: `packages/graph-renderer/src/editing/node-move-coordinator.ts`
- Modify: `packages/graph-renderer/src/bridge/css-isolation.ts`
- Create: `packages/graph-renderer/src/editing/node-move-coordinator.test.ts`

**Interfaces:**
- Consumes:
  - `EditPolicy` — `canSpliceOntoEdge?()`, `getDeleteStrategy()`
  - `defaultCanSpliceOntoEdge()` from `splice-validation.ts`
  - `GraphModel`, `GraphNode`, `nodeById`, `childrenOf`, `edgesOf`
    from graph-core
- Produces:
  - `SourceCleanupStrategy` re-exported from `types.ts`
  - `DragEndResult = { type: 'splice'; nodeId: string; edgeId: string; sourceCleanup: SourceCleanupStrategy } | { type: 'cancelled' }`
  - `createNodeMoveCoordinator(opts: NodeMoveCoordinatorOptions): NodeMoveCoordinator`
  - `interface NodeMoveCoordinator { startDrag(nodeId: string, event: PointerEvent, model: GraphModel): void; dispose(): void }`

- [ ] **Step 1: Write failing test — coordinator ghosts node on drag activation**

Create `node-move-coordinator.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { clearGrammarRegistry, registerGrammar } from '@casehubio/graph-core';
import type { GraphModel } from '@casehubio/graph-core';
import { createNodeMoveCoordinator } from './node-move-coordinator.js';
import { defaultEditPolicy } from './edit-policy.js';

vi.mock('@xyflow/react', () => ({
  Handle: () => null,
  Position: { Top: 'top', Bottom: 'bottom', Left: 'left', Right: 'right' },
}));

function makeModel(): GraphModel {
  return {
    nodes: [
      { id: 'a', type: 'start', properties: {} },
      { id: 'x', type: 'worker', properties: {} },
      { id: 'b', type: 'end', properties: {} },
    ],
    edges: [
      { id: 'e1', type: 'default', source: 'a', target: 'x' },
      { id: 'e2', type: 'default', source: 'x', target: 'b' },
    ],
  };
}

describe('NodeMoveCoordinator', () => {
  let container: HTMLDivElement;
  let onResult: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    clearGrammarRegistry();
    container = document.createElement('div');
    document.body.appendChild(container);

    // Create a mock node element
    const nodeEl = document.createElement('div');
    nodeEl.className = 'react-flow__node';
    nodeEl.dataset['id'] = 'x';
    const wrapper = document.createElement('div');
    wrapper.className = 'stencil-decoration-wrapper';
    wrapper.textContent = 'Node X';
    nodeEl.appendChild(wrapper);
    container.appendChild(nodeEl);

    onResult = vi.fn();
  });

  afterEach(() => {
    clearGrammarRegistry();
    container.remove();
  });

  it('does not ghost before drag threshold is exceeded', () => {
    const coord = createNodeMoveCoordinator({
      editPolicy: defaultEditPolicy(),
      containerEl: container,
      onResult,
    });

    const event = new PointerEvent('pointerdown', {
      clientX: 100, clientY: 100, bubbles: true,
    });
    Object.defineProperty(event, 'target', { value: container.querySelector('.stencil-decoration-wrapper') });

    coord.startDrag('x', event, makeModel());

    const nodeEl = container.querySelector('.react-flow__node');
    expect(nodeEl!.classList.contains('node-move-ghost')).toBe(false);

    coord.dispose();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run node-move-coordinator.test`
Expected: FAIL — module not found

- [ ] **Step 3: Create node-move-coordinator.ts**

Create `packages/graph-renderer/src/editing/node-move-coordinator.ts`:

```typescript
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import { nodeById, childrenOf, edgesOf } from '@casehubio/graph-core';
import type { EditPolicy } from './types.js';
import type { SourceCleanupStrategy } from './types.js';
import { defaultCanSpliceOntoEdge } from './splice-validation.js';

const DRAG_THRESHOLD = 5;

export type DragEndResult =
  | { type: 'splice'; nodeId: string; edgeId: string; sourceCleanup: SourceCleanupStrategy }
  | { type: 'cancelled' };

export interface NodeMoveCoordinator {
  startDrag(nodeId: string, event: PointerEvent, model: GraphModel): void;
  dispose(): void;
}

export interface NodeMoveCoordinatorOptions {
  editPolicy: EditPolicy;
  containerEl: HTMLElement;
  onResult: (result: DragEndResult) => void;
}

export function createNodeMoveCoordinator(opts: NodeMoveCoordinatorOptions): NodeMoveCoordinator {
  const { editPolicy, containerEl, onResult } = opts;

  let activeModel: GraphModel | null = null;
  let draggedNodeId: string | null = null;
  let startPos: { x: number; y: number } | null = null;
  let dragActive = false;
  let ghostedNodeEl: HTMLElement | null = null;
  let cloneEl: HTMLElement | null = null;
  let highlightedEdgeEl: HTMLElement | null = null;
  let grabOffset = { x: 0, y: 0 };

  function isEligible(nodeId: string, model: GraphModel): boolean {
    const node = nodeById(model, nodeId);
    if (!node) return false;
    if (node.parentId) return false;
    if (childrenOf(model, nodeId).length > 0) return false;
    return true;
  }

  function canSplice(edge: GraphEdge, node: GraphNode, model: GraphModel): boolean {
    return editPolicy.canSpliceOntoEdge?.(edge, node, model)
      ?? defaultCanSpliceOntoEdge(editPolicy, edge, node, model);
  }

  function getSourceCleanup(node: GraphNode, model: GraphModel): SourceCleanupStrategy {
    const strategy = editPolicy.getDeleteStrategy(node, model);
    return strategy.type === 'auto-join' ? 'auto-join' : 'disconnect';
  }

  function activate(e: PointerEvent): void {
    dragActive = true;

    // Ghost the original
    const nodeEl = containerEl.querySelector(`.react-flow__node[data-id="${draggedNodeId}"]`) as HTMLElement | null;
    if (nodeEl) {
      nodeEl.classList.add('node-move-ghost');
      ghostedNodeEl = nodeEl;
    }

    // Create clone
    const wrapper = ghostedNodeEl?.querySelector('.stencil-decoration-wrapper');
    if (wrapper) {
      cloneEl = document.createElement('div');
      cloneEl.innerHTML = wrapper.innerHTML;
      cloneEl.style.position = 'fixed';
      cloneEl.style.pointerEvents = 'none';
      cloneEl.style.opacity = '0.85';
      cloneEl.style.zIndex = '1000';
      cloneEl.style.filter = 'drop-shadow(0 4px 12px rgba(0,0,0,0.2))';
      cloneEl.style.left = `${e.clientX - grabOffset.x}px`;
      cloneEl.style.top = `${e.clientY - grabOffset.y}px`;

      const rect = wrapper.getBoundingClientRect();
      cloneEl.style.width = `${rect.width}px`;
      cloneEl.style.height = `${rect.height}px`;
      document.body.appendChild(cloneEl);
    }
  }

  function onPointerMove(e: PointerEvent): void {
    if (!startPos || !activeModel || !draggedNodeId) return;

    if (!dragActive) {
      const dx = e.clientX - startPos.x;
      const dy = e.clientY - startPos.y;
      if (Math.hypot(dx, dy) < DRAG_THRESHOLD) return;
      activate(e);
    }

    // Move clone
    if (cloneEl) {
      cloneEl.style.left = `${e.clientX - grabOffset.x}px`;
      cloneEl.style.top = `${e.clientY - grabOffset.y}px`;
    }

    // Hit-test edges
    clearEdgeHighlight();
    const draggedNode = nodeById(activeModel, draggedNodeId);
    if (!draggedNode) return;

    for (const hitEl of document.elementsFromPoint(e.clientX, e.clientY)) {
      const edgeEl = hitEl.closest('.react-flow__edge') as HTMLElement | null;
      if (!edgeEl) continue;
      const edgeId = edgeEl.dataset['id'];
      if (!edgeId) continue;

      const edge = activeModel.edges.find(ed => ed.id === edgeId);
      if (!edge) continue;
      if (edge.source === draggedNodeId || edge.target === draggedNodeId) continue;

      if (canSplice(edge, draggedNode, activeModel)) {
        edgeEl.classList.add('edge-splice-valid');
        highlightedEdgeEl = edgeEl;
      }
      break;
    }
  }

  function onPointerUp(_e: PointerEvent): void {
    cleanup();

    if (!dragActive || !activeModel || !draggedNodeId) {
      onResult({ type: 'cancelled' });
      return;
    }

    if (highlightedEdgeEl) {
      const edgeId = highlightedEdgeEl.dataset['id'];
      const node = nodeById(activeModel, draggedNodeId);
      if (edgeId && node) {
        onResult({
          type: 'splice',
          nodeId: draggedNodeId,
          edgeId,
          sourceCleanup: getSourceCleanup(node, activeModel),
        });
        return;
      }
    }

    onResult({ type: 'cancelled' });
  }

  function clearEdgeHighlight(): void {
    if (highlightedEdgeEl) {
      highlightedEdgeEl.classList.remove('edge-splice-valid');
      highlightedEdgeEl = null;
    }
  }

  function cleanup(): void {
    document.removeEventListener('pointermove', onPointerMove);
    document.removeEventListener('pointerup', onPointerUp);

    if (ghostedNodeEl) {
      ghostedNodeEl.classList.remove('node-move-ghost');
      ghostedNodeEl = null;
    }
    if (cloneEl) {
      cloneEl.remove();
      cloneEl = null;
    }
    clearEdgeHighlight();

    activeModel = null;
    draggedNodeId = null;
    startPos = null;
    dragActive = false;
  }

  return {
    startDrag(nodeId: string, event: PointerEvent, model: GraphModel): void {
      if (!isEligible(nodeId, model)) return;

      draggedNodeId = nodeId;
      activeModel = model;
      startPos = { x: event.clientX, y: event.clientY };

      const wrapper = containerEl.querySelector(
        `.react-flow__node[data-id="${nodeId}"] .stencil-decoration-wrapper`
      );
      if (wrapper) {
        const rect = wrapper.getBoundingClientRect();
        grabOffset = { x: event.clientX - rect.left, y: event.clientY - rect.top };
      }

      (event.target as HTMLElement)?.setPointerCapture?.(event.pointerId);
      document.addEventListener('pointermove', onPointerMove);
      document.addEventListener('pointerup', onPointerUp);
    },

    dispose(): void {
      cleanup();
    },
  };
}
```

- [ ] **Step 4: Add CSS for ghost, clone, and edge highlight**

In `css-isolation.ts`, add after the existing rules (before the closing
`.trim()`):

```css
.node-move-ghost .stencil-decoration-wrapper {
  opacity: 0.3;
  pointer-events: none;
  transition: opacity 120ms ease-out;
}
.edge-splice-valid .react-flow__edge-path {
  stroke: var(--pages-success-9, #16a34a) !important;
  stroke-width: 3px !important;
  filter: drop-shadow(0 0 4px var(--pages-success-9, #16a34a));
  transition: stroke-width 100ms, filter 100ms;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run node-move-coordinator.test`
Expected: PASS

- [ ] **Step 6: Write additional coordinator tests**

Add tests for:
- `dispose()` removes ghost class and clone element
- Ineligible node (has children) — `startDrag` is a no-op
- Ineligible node (has parentId) — `startDrag` is a no-op
- Drag cancelled (pointerup without hitting valid edge) → result is `{ type: 'cancelled' }`
- Threshold test — pointerdown + pointerup with <5px movement, no ghost

- [ ] **Step 7: Run all tests, commit**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run`

```bash
git add packages/graph-renderer/src/editing/node-move-coordinator.ts packages/graph-renderer/src/editing/node-move-coordinator.test.ts packages/graph-renderer/src/bridge/css-isolation.ts
git commit -m "feat(graph): create NodeMoveCoordinator for drag-to-splice interaction

Standalone coordinator using pointer events. Ghosts original node,
creates floating clone, hit-tests edges via elementsFromPoint, validates
splice via EditPolicy, and returns a DragEndResult on drop.

Refs #388"
```

### Task 5: Wire coordinator into GraphCanvas

**Files:**
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts`
- Modify: `packages/graph-renderer/src/index.ts` (export new types)
- Test: `packages/graph-renderer/src/bridge/bridge.test.ts`

**Interfaces:**
- Consumes: `createNodeMoveCoordinator`, `DragEndResult`,
  `NodeMoveCoordinator` from `node-move-coordinator.ts`.
  `GraphCanvas.onMutation` callback, `GraphCanvas.editPolicy`,
  `GraphCanvas.model`.
- Produces: GraphCanvas wires pointerdown delegation on `.diagram-root`,
  dispatches `moveNodeToEdge` edits via `onMutation`

- [ ] **Step 1: Write failing test — GraphCanvas creates coordinator on pointerdown**

In `bridge.test.ts`, add a test that verifies the `moveNodeToEdge` edit
is dispatched when the coordinator completes a splice. (Integration-level
test using the actual GraphCanvas element and synthetic pointer events.)

- [ ] **Step 2: Wire coordinator in GraphCanvas**

In `GraphCanvas.ts`, add:

Import at the top:
```typescript
import { createNodeMoveCoordinator } from '../editing/node-move-coordinator.js';
import type { NodeMoveCoordinator, DragEndResult } from '../editing/node-move-coordinator.js';
```

Add field:
```typescript
private _moveCoordinator: NodeMoveCoordinator | null = null;
```

In `connectedCallback()`, after the container is created, set up event
delegation:

```typescript
this._container.addEventListener('pointerdown', (e: PointerEvent) => {
  const target = e.target as HTMLElement;
  if (target.closest('.stencil-source-handle')) return;
  const wrapper = target.closest('.stencil-decoration-wrapper');
  if (!wrapper) return;
  const nodeEl = target.closest('.react-flow__node') as HTMLElement | null;
  const nodeId = nodeEl?.dataset['id'];
  if (!nodeId || !this.model || !this.editPolicy) return;

  if (!this._moveCoordinator) {
    this._moveCoordinator = createNodeMoveCoordinator({
      editPolicy: this.editPolicy,
      containerEl: this._container,
      onResult: (result: DragEndResult) => this._handleMoveResult(result),
    });
  }
  this._moveCoordinator.startDrag(nodeId, e, this.model);
});
```

Add the result handler:

```typescript
private _handleMoveResult(result: DragEndResult): void {
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

In `disconnectedCallback()`, dispose the coordinator:

```typescript
this._moveCoordinator?.dispose();
this._moveCoordinator = null;
```

- [ ] **Step 3: Export new types from index.ts**

Ensure `DragEndResult`, `SourceCleanupStrategy`, `NodeMoveCoordinator`
are exported from the package's `index.ts`.

- [ ] **Step 4: Run all tests**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run`
Expected: ALL PASS

- [ ] **Step 5: Run cross-package type check**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add packages/graph-renderer/src/bridge/GraphCanvas.ts packages/graph-renderer/src/index.ts packages/graph-renderer/src/bridge/bridge.test.ts
git commit -m "feat(graph): wire NodeMoveCoordinator into GraphCanvas

Event delegation on .diagram-root dispatches pointerdown to the
coordinator. Source handle clicks are ignored (connection initiation).
Splice results dispatch moveNodeToEdge via onMutation.

Closes #388"
```

---

## References

- [2026-08-28-edge-splice-design.md] — design spec this plan implements
- [2026-08-26-diagram-editing-infrastructure-design.md] — infra spec interaction #7
- `packages/graph-renderer/src/editing/types.ts` — EditPolicy, GraphEdit
- `packages/graph-renderer/src/editing/edit-policy.ts` — defaultEditPolicy
- `packages/graph-renderer/src/editing/apply-graph-edit.ts` — applyGraphEdit
- `packages/graph-renderer/src/stencil-wrapper.tsx:159-191` — handle rendering
- `packages/graph-renderer/src/bridge/GraphCanvas.ts` — canvas element
- `packages/graph-renderer/src/bridge/css-isolation.ts` — CSS isolation
- [GE-20260825-309197] — standalone coordinator pattern
- [GE-20260827-ed8606] — elementsFromPoint for hit-testing
- [PP-20260826-507928] — graph-core pure data
- [GitHub #387] — handle shrinking blocker
- [GitHub #388] — edge-splice feature
