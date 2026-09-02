# Multi-Select Graph Nodes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #402 — Multi-select: rubber-band drag-select and shift-click for graph nodes
**Issue group:** #402

**Goal:** Add multi-select to the graph editor with constrained drag-select (1-in/1-out), unconstrained shift-click, segment delete with auto-join, and segment hold-to-drag-and-splice.

**Architecture:** A `SelectionValidator` in graph-core provides pure selection validation logic (boundary edge computation, internal connectivity). A `RubberBandSelect` in graph-renderer handles the custom drag-select interaction with live feedback. The existing `NodeMoveCoordinator` is extended with a `DragSubject` union to support both single-node and segment drag. GraphCanvas manages the `MultiSelectState` and wires shift-click, rubber-band, and coordinator dispatch.

**Tech Stack:** TypeScript, Vitest, Lit (GraphCanvas), React (ReactFlowApp), @xyflow/react

## Global Constraints

- graph-core stays pure data — no DOM, no callbacks (PP-20260826-507928)
- All events follow pages-event contract (PP-20260705-bac842)
- `nodesDraggable` remains `false` — auto-layout owns positions
- Test command: `npx vitest run` from the package directory
- Imports use `.js` extension (ESM convention in this project)

---

## Batch 1: SelectionValidator — Pure Logic + Exhaustive Tests

This batch delivers the core validation algorithm in graph-core with no UI dependency. After this batch, the validator is fully tested and ready for consumption by the UI layer.

### Task 1: SelectionValidator — boundary computation and 1-in/1-out check

**Files:**
- Create: `packages/graph-core/src/selection-validator.ts`
- Create: `packages/graph-core/src/selection-validator.test.ts`
- Modify: `packages/graph-core/src/index.ts` (add exports)

**Interfaces:**
- Consumes: `GraphModel`, `GraphEdge`, `GraphNode` from `model.ts`; `edgesOf`, `inboundEdges`, `outboundEdges` from `query.ts`
- Produces:
  - `SelectionResult { valid: ReadonlySet<string>; invalid: ReadonlySet<string>; boundaryInput: GraphEdge | null; boundaryOutput: GraphEdge | null }`
  - `validateSelection(candidateIds: ReadonlySet<string>, model: GraphModel): SelectionResult`
  - `canAddToSelection(nodeId: string, currentSelection: ReadonlySet<string>, model: GraphModel): SelectionResult`
  - `canRemoveFromSelection(nodeId: string, currentSelection: ReadonlySet<string>, model: GraphModel): SelectionResult`

- [ ] **Step 1: Write failing tests for boundary computation — valid linear chain**

```typescript
import { describe, it, expect } from 'vitest';
import { validateSelection } from './selection-validator.js';
import { createGraph } from './graph.js';
import type { GraphNode, GraphEdge } from './model.js';

function node(id: string, type = 'step'): GraphNode {
  return { id, type, properties: {} };
}

function edge(id: string, source: string, target: string): GraphEdge {
  return { id, type: 'default', source, target };
}

describe('validateSelection', () => {
  describe('linear chains', () => {
    // A→B→C→D
    const model = createGraph(
      [node('A'), node('B'), node('C'), node('D')],
      [edge('e1', 'A', 'B'), edge('e2', 'B', 'C'), edge('e3', 'C', 'D')],
    );

    it('selects a contiguous middle segment {B,C}', () => {
      const result = validateSelection(new Set(['B', 'C']), model);
      expect(result.valid).toEqual(new Set(['B', 'C']));
      expect(result.invalid).toEqual(new Set());
      expect(result.boundaryInput?.id).toBe('e1');
      expect(result.boundaryOutput?.id).toBe('e3');
    });

    it('selects a single middle node {B}', () => {
      const result = validateSelection(new Set(['B']), model);
      expect(result.valid).toEqual(new Set(['B']));
      expect(result.invalid).toEqual(new Set());
      expect(result.boundaryInput?.id).toBe('e1');
      expect(result.boundaryOutput?.id).toBe('e2');
    });

    it('rejects non-contiguous {B,D} — gap at C creates 2 in, 2 out', () => {
      const result = validateSelection(new Set(['B', 'D']), model);
      expect(result.valid).toEqual(new Set());
      expect(result.invalid).toEqual(new Set(['B', 'D']));
    });
  });

  describe('boundary counting', () => {
    it('rejects selection with 2 outbound — branch node A→B, A→C, select {A}', () => {
      const model = createGraph(
        [node('A'), node('B'), node('C')],
        [edge('e1', 'A', 'B'), edge('e2', 'A', 'C')],
      );
      const result = validateSelection(new Set(['A']), model);
      expect(result.valid).toEqual(new Set());
      expect(result.invalid).toEqual(new Set(['A']));
    });

    it('rejects diamond — A→B→D, A→C→D, select {B,C} has 2 in, 2 out', () => {
      const model = createGraph(
        [node('A'), node('B'), node('C'), node('D')],
        [edge('e1', 'A', 'B'), edge('e2', 'A', 'C'), edge('e3', 'B', 'D'), edge('e4', 'C', 'D')],
      );
      const result = validateSelection(new Set(['B', 'C']), model);
      expect(result.valid).toEqual(new Set());
      expect(result.invalid).toEqual(new Set(['B', 'C']));
    });

    it('rejects disconnected node — 0 in, 0 out', () => {
      const model = createGraph([node('X')], []);
      const result = validateSelection(new Set(['X']), model);
      expect(result.valid).toEqual(new Set());
      expect(result.invalid).toEqual(new Set(['X']));
    });

    it('returns empty result for empty candidate set', () => {
      const model = createGraph([node('A')], []);
      const result = validateSelection(new Set(), model);
      expect(result.valid).toEqual(new Set());
      expect(result.invalid).toEqual(new Set());
      expect(result.boundaryInput).toBeNull();
      expect(result.boundaryOutput).toBeNull();
    });
  });

  describe('internal connectivity', () => {
    it('rejects disconnected groups that pass 1-in/1-out — X→A→B, C→D→Y, select {B,C}', () => {
      const model = createGraph(
        [node('X'), node('A'), node('B'), node('C'), node('D'), node('Y')],
        [edge('e1', 'X', 'A'), edge('e2', 'A', 'B'), edge('e3', 'C', 'D'), edge('e4', 'D', 'Y')],
      );
      // B has 1 inbound from outside (A→B), C has 1 outbound to outside (D→Y)? No.
      // Wait: boundary edges for {B,C}: A→B (in), and... C→D (out). That's 1-in/1-out!
      // But B and C have no internal edge. Must fail connectivity check.
      const result = validateSelection(new Set(['B', 'C']), model);
      expect(result.valid).toEqual(new Set());
      expect(result.invalid).toEqual(new Set(['B', 'C']));
    });
  });

  describe('containment', () => {
    it('validates parent with 1-in/1-out', () => {
      // A→P→C where P has child Ch
      const model = createGraph(
        [node('A'), { id: 'P', type: 'container', properties: {} }, node('C'),
         { id: 'Ch', type: 'step', parentId: 'P', properties: {} }],
        [edge('e1', 'A', 'P'), edge('e2', 'P', 'C')],
      );
      const result = validateSelection(new Set(['P']), model);
      expect(result.valid).toEqual(new Set(['P']));
      expect(result.boundaryInput?.id).toBe('e1');
      expect(result.boundaryOutput?.id).toBe('e2');
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/selection-validator.test.ts` from `packages/graph-core`
Expected: FAIL — module not found

- [ ] **Step 3: Implement SelectionValidator**

```typescript
// packages/graph-core/src/selection-validator.ts
import type { GraphModel, GraphEdge } from './model.js';

export interface SelectionResult {
  readonly valid: ReadonlySet<string>;
  readonly invalid: ReadonlySet<string>;
  readonly boundaryInput: GraphEdge | null;
  readonly boundaryOutput: GraphEdge | null;
}

export function validateSelection(
  candidateIds: ReadonlySet<string>,
  model: GraphModel,
): SelectionResult {
  const empty: SelectionResult = {
    valid: new Set(), invalid: new Set(),
    boundaryInput: null, boundaryOutput: null,
  };
  if (candidateIds.size === 0) return empty;

  const inbound: GraphEdge[] = [];
  const outbound: GraphEdge[] = [];

  for (const e of model.edges) {
    const srcIn = candidateIds.has(e.source);
    const tgtIn = candidateIds.has(e.target);
    if (!srcIn && tgtIn) inbound.push(e);
    else if (srcIn && !tgtIn) outbound.push(e);
  }

  if (inbound.length !== 1 || outbound.length !== 1) {
    return { ...empty, invalid: new Set(candidateIds) };
  }

  // Internal connectivity: BFS from entry to exit using internal edges only
  const entryNodeId = inbound[0]!.target;
  const exitNodeId = outbound[0]!.source;

  if (entryNodeId === exitNodeId) {
    // Single node — trivially connected
    return {
      valid: new Set(candidateIds),
      invalid: new Set(),
      boundaryInput: inbound[0]!,
      boundaryOutput: outbound[0]!,
    };
  }

  const visited = new Set<string>();
  const queue = [entryNodeId];
  visited.add(entryNodeId);

  while (queue.length > 0) {
    const current = queue.shift()!;
    for (const e of model.edges) {
      if (e.source === current && candidateIds.has(e.target) && !visited.has(e.target)) {
        visited.add(e.target);
        queue.push(e.target);
      }
    }
  }

  if (!visited.has(exitNodeId)) {
    return { ...empty, invalid: new Set(candidateIds) };
  }

  return {
    valid: new Set(candidateIds),
    invalid: new Set(),
    boundaryInput: inbound[0]!,
    boundaryOutput: outbound[0]!,
  };
}

export function canAddToSelection(
  nodeId: string,
  currentSelection: ReadonlySet<string>,
  model: GraphModel,
): SelectionResult {
  const expanded = new Set(currentSelection);
  expanded.add(nodeId);
  return validateSelection(expanded, model);
}

export function canRemoveFromSelection(
  nodeId: string,
  currentSelection: ReadonlySet<string>,
  model: GraphModel,
): SelectionResult {
  const shrunk = new Set(currentSelection);
  shrunk.delete(nodeId);
  return validateSelection(shrunk, model);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/selection-validator.test.ts` from `packages/graph-core`
Expected: PASS

- [ ] **Step 5: Add shift-click validation tests**

Append to `selection-validator.test.ts`:

```typescript
describe('canAddToSelection', () => {
  // A→B→C→D
  const model = createGraph(
    [node('A'), node('B'), node('C'), node('D')],
    [edge('e1', 'A', 'B'), edge('e2', 'B', 'C'), edge('e3', 'C', 'D')],
  );

  it('allows adding D to {B,C} — extends segment', () => {
    const result = canAddToSelection('D', new Set(['B', 'C']), model);
    expect(result.valid).toEqual(new Set(['B', 'C', 'D']));
    expect(result.invalid).toEqual(new Set());
    expect(result.boundaryOutput?.id).toBe('e3'); // now C→D is internal, D has no outbound
  });

  it('rejects adding A to {B,C} — A has 0 inbound, breaks 1-in', () => {
    const result = canAddToSelection('A', new Set(['B', 'C']), model);
    expect(result.valid).toEqual(new Set());
    expect(result.invalid.size).toBeGreaterThan(0);
  });
});

describe('canRemoveFromSelection', () => {
  // A→B→C→D
  const model = createGraph(
    [node('A'), node('B'), node('C'), node('D')],
    [edge('e1', 'A', 'B'), edge('e2', 'B', 'C'), edge('e3', 'C', 'D')],
  );

  it('allows removing D from {B,C,D} — shrinks segment', () => {
    const result = canRemoveFromSelection('D', new Set(['B', 'C', 'D']), model);
    expect(result.valid).toEqual(new Set(['B', 'C']));
    expect(result.boundaryOutput?.id).toBe('e3');
  });

  it('rejects removing C from {B,C,D} — dangles internal edges', () => {
    const result = canRemoveFromSelection('C', new Set(['B', 'C', 'D']), model);
    expect(result.valid).toEqual(new Set());
    expect(result.invalid.size).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 6: Run all tests**

Run: `npx vitest run src/selection-validator.test.ts` from `packages/graph-core`
Expected: PASS

- [ ] **Step 7: Add export to graph-core index**

Add to `packages/graph-core/src/index.ts`:
```typescript
export { validateSelection, canAddToSelection, canRemoveFromSelection } from './selection-validator.js';
export type { SelectionResult } from './selection-validator.js';
```

- [ ] **Step 8: Run full graph-core test suite**

Run: `npx vitest run` from `packages/graph-core`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add packages/graph-core/src/selection-validator.ts packages/graph-core/src/selection-validator.test.ts packages/graph-core/src/index.ts
git commit -m "feat(#402): add SelectionValidator with boundary computation and connectivity check Refs #402"
```

### Task 2: removeNodes compound edit

**Files:**
- Modify: `packages/graph-core/src/edit.ts` (add `removeNodes`)
- Modify: `packages/graph-core/src/edit.test.ts` (add tests)
- Modify: `packages/graph-core/src/index.ts` (add export)

**Interfaces:**
- Consumes: `removeNode` from `edit.ts`, `childrenOf` from `traversal.ts`
- Produces: `removeNodes(model: GraphModel, nodeIds: ReadonlySet<string>): EditResult`

- [ ] **Step 1: Write failing tests for removeNodes**

Append to `edit.test.ts`:

```typescript
describe('removeNodes', () => {
  beforeEach(() => {
    clearGrammarRegistry();
  });

  it('removes multiple nodes and their edges', () => {
    // A→B→C→D — remove {B,C}
    const model = createGraph(
      [node('A', 'step'), node('B', 'step'), node('C', 'step'), node('D', 'step')],
      [edge('e1', 'A', 'B'), edge('e2', 'B', 'C'), edge('e3', 'C', 'D')],
    );
    const result = removeNodes(model, new Set(['B', 'C']));
    expect(result.model.nodes).toHaveLength(2);
    expect(result.model.nodes.map(n => n.id).sort()).toEqual(['A', 'D']);
    // All edges touching B or C are removed
    expect(result.model.edges).toHaveLength(0);
  });

  it('handles empty set — returns same model', () => {
    const model = createGraph([node('A', 'step')], []);
    const result = removeNodes(model, new Set());
    expect(result.model.nodes).toHaveLength(1);
  });

  it('removes leaf nodes before parents in containment tree', () => {
    // P contains Ch; both selected
    const P = node('P', 'container');
    const Ch = { id: 'Ch', type: 'step', parentId: 'P', properties: {} } as GraphNode;
    const model = createGraph([P, Ch], []);
    const result = removeNodes(model, new Set(['P', 'Ch']));
    expect(result.model.nodes).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/edit.test.ts` from `packages/graph-core`
Expected: FAIL — removeNodes not exported

- [ ] **Step 3: Implement removeNodes**

Add to `packages/graph-core/src/edit.ts`:

```typescript
export function removeNodes(model: GraphModel, nodeIds: ReadonlySet<string>): EditResult {
  if (nodeIds.size === 0) return { model, violations: [] };

  // Order: leaf nodes first (nodes with no children in the deletion set)
  const ordered: string[] = [];
  const remaining = new Set(nodeIds);

  while (remaining.size > 0) {
    const leaves: string[] = [];
    for (const id of remaining) {
      const hasChildInSet = model.nodes.some(
        n => n.parentId === id && remaining.has(n.id),
      );
      if (!hasChildInSet) leaves.push(id);
    }
    if (leaves.length === 0) {
      // Cycle or error — flush remaining
      for (const id of remaining) ordered.push(id);
      break;
    }
    for (const id of leaves) {
      ordered.push(id);
      remaining.delete(id);
    }
  }

  let current = model;
  const allViolations: ConstraintViolation[] = [];
  for (const id of ordered) {
    const result = removeNode(current, id);
    current = result.model;
    allViolations.push(...result.violations);
  }
  return { model: current, violations: allViolations };
}
```

- [ ] **Step 4: Add import for ConstraintViolation and export removeNodes**

Ensure `ConstraintViolation` is imported in `edit.ts` (from `validator.js`).
Add to `packages/graph-core/src/index.ts`:
```typescript
export { removeNodes } from './edit.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npx vitest run src/edit.test.ts` from `packages/graph-core`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/graph-core/src/edit.ts packages/graph-core/src/edit.test.ts packages/graph-core/src/index.ts
git commit -m "feat(#402): add removeNodes compound edit with leaf-first ordering Refs #402"
```

---

## Batch 2: Rubber-Band Select + Shift-Click — UI Interaction

After this batch, users can drag-select nodes with live valid/invalid feedback and use shift-click to extend selections. No structural operations yet.

### Task 3: MultiSelectState and GraphCanvas wiring

**Files:**
- Modify: `packages/graph-renderer/src/editing/types.ts` (add MultiSelectState, DragSubject)
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts` (add multi-select state, shift-click handler)
- Modify: `packages/graph-renderer/src/bridge/ReactFlowApp.tsx` (add `multiSelectionKeyCode={null}`)

**Interfaces:**
- Consumes: `SelectionResult`, `validateSelection`, `canAddToSelection`, `canRemoveFromSelection` from graph-core
- Produces:
  - `MultiSelectState { selectedNodeIds: ReadonlySet<string>; mode: 'none' | 'constrained' | 'unconstrained'; boundaryInput: GraphEdge | null; boundaryOutput: GraphEdge | null }`
  - `DragSubject = { type: 'single'; nodeId: string } | { type: 'segment'; nodeIds: ReadonlySet<string>; entryNodeId: string; exitNodeId: string; boundaryInput: GraphEdge; boundaryOutput: GraphEdge }`
  - `graph:multiselect:change` event with `MultiSelectState` payload

- [ ] **Step 1: Add types to types.ts**

Add `MultiSelectState` interface and `DragSubject` type to `packages/graph-renderer/src/editing/types.ts`:

```typescript
export interface MultiSelectState {
  readonly selectedNodeIds: ReadonlySet<string>;
  readonly mode: 'none' | 'constrained' | 'unconstrained';
  readonly boundaryInput: GraphEdge | null;
  readonly boundaryOutput: GraphEdge | null;
}

export type DragSubject =
  | { readonly type: 'single'; readonly nodeId: string }
  | { readonly type: 'segment'; readonly nodeIds: ReadonlySet<string>;
      readonly entryNodeId: string; readonly exitNodeId: string;
      readonly boundaryInput: GraphEdge; readonly boundaryOutput: GraphEdge };
```

- [ ] **Step 2: Disable ReactFlow built-in multi-select**

In `ReactFlowApp.tsx`, add `multiSelectionKeyCode={null}` to the `<ReactFlow>` element props. This prevents ReactFlow from handling Shift+click, giving full control to the custom handler.

- [ ] **Step 3: Add multi-select state and shift-click handler to GraphCanvas**

Add a `_multiSelect` field of type `MultiSelectState` (initialized to `{ selectedNodeIds: new Set(), mode: 'none', boundaryInput: null, boundaryOutput: null }`).

Modify the existing `onNodeClick` callback in `_renderReact()` to handle shift-click:
- If `event.shiftKey` and `mode === 'none'`: start unconstrained, add node
- If `event.shiftKey` and `mode === 'unconstrained'`: toggle node
- If `event.shiftKey` and `mode === 'constrained'`: validate via `canAddToSelection`/`canRemoveFromSelection`, apply or reject
- If not shift: clear multi-select, single-select as before

After any multi-select state change, apply CSS classes to node elements (`multi-select-active`) and emit `graph:multiselect:change` event.

- [ ] **Step 4: Run existing graph-renderer tests**

Run: `npx vitest run` from `packages/graph-renderer`
Expected: All existing tests PASS (no regressions)

- [ ] **Step 5: Commit**

```bash
git add packages/graph-renderer/src/editing/types.ts packages/graph-renderer/src/bridge/GraphCanvas.ts packages/graph-renderer/src/bridge/ReactFlowApp.tsx
git commit -m "feat(#402): add MultiSelectState, DragSubject types, shift-click handler in GraphCanvas Refs #402"
```

### Task 4: RubberBandSelect interaction

**Files:**
- Create: `packages/graph-renderer/src/editing/rubber-band-select.ts`
- Create: `packages/graph-renderer/src/editing/rubber-band-select.test.ts`
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts` (wire rubber-band)

**Interfaces:**
- Consumes: `validateSelection` from graph-core, `screenToFlowPosition` from ReactFlow instance, `MultiSelectState` from types.ts
- Produces:
  - `RubberBandResult = { type: 'selected'; nodeIds: ReadonlySet<string>; boundaryInput: GraphEdge; boundaryOutput: GraphEdge } | { type: 'empty' }`
  - `createRubberBandSelect(opts: RubberBandOptions): RubberBandSelect`
  - `RubberBandSelect { attach(): void; dispose(): void }`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { createRubberBandSelect } from './rubber-band-select.js';
import { createGraph } from '@nicecases/graph-core';
import type { GraphNode, GraphEdge, GraphModel } from '@nicecases/graph-core';

function node(id: string): GraphNode {
  return { id, type: 'step', properties: {} };
}
function edge(id: string, source: string, target: string): GraphEdge {
  return { id, type: 'default', source, target };
}

// Mock ReactFlow nodes with position and dimensions
function rfNode(id: string, x: number, y: number, w = 100, h = 50) {
  return {
    id, position: { x, y }, measured: { width: w, height: h },
    type: 'custom', data: {},
  };
}

describe('RubberBandSelect', () => {
  let container: HTMLDivElement;
  let onComplete: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    container = document.createElement('div');
    document.body.appendChild(container);
    onComplete = vi.fn();
  });

  afterEach(() => {
    document.body.removeChild(container);
  });

  it('selects nodes inside the rubber-band rectangle', () => {
    // A(0,0)→B(200,0)→C(400,0) — rubber band covers B only
    const model = createGraph(
      [node('A'), node('B'), node('C')],
      [edge('e1', 'A', 'B'), edge('e2', 'B', 'C')],
    );
    const nodes = [rfNode('A', 0, 0), rfNode('B', 200, 0), rfNode('C', 400, 0)];

    const rb = createRubberBandSelect({
      containerEl: container,
      screenToFlow: (x, y) => ({ x, y }), // identity for testing
      getNodes: () => nodes,
      getModel: () => model,
      onComplete,
    });
    rb.attach();

    // Simulate pointer events — rectangle from (150,-10) to (350,100) covers B
    container.dispatchEvent(new PointerEvent('pointerdown', { clientX: 150, clientY: -10, bubbles: true }));
    container.dispatchEvent(new PointerEvent('pointermove', { clientX: 350, clientY: 100, bubbles: true }));
    container.dispatchEvent(new PointerEvent('pointerup', { clientX: 350, clientY: 100, bubbles: true }));

    expect(onComplete).toHaveBeenCalledWith(expect.objectContaining({
      type: 'selected',
      nodeIds: new Set(['B']),
    }));

    rb.dispose();
  });

  it('calls onComplete with empty when no valid selection', () => {
    const model = createGraph([node('A')], []);
    const nodes = [rfNode('A', 0, 0)];

    const rb = createRubberBandSelect({
      containerEl: container,
      screenToFlow: (x, y) => ({ x, y }),
      getNodes: () => nodes,
      getModel: () => model,
      onComplete,
    });
    rb.attach();

    // Select disconnected node A — 0 in/0 out → invalid
    container.dispatchEvent(new PointerEvent('pointerdown', { clientX: -10, clientY: -10, bubbles: true }));
    container.dispatchEvent(new PointerEvent('pointermove', { clientX: 200, clientY: 100, bubbles: true }));
    container.dispatchEvent(new PointerEvent('pointerup', { clientX: 200, clientY: 100, bubbles: true }));

    expect(onComplete).toHaveBeenCalledWith({ type: 'empty' });
    rb.dispose();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/editing/rubber-band-select.test.ts` from `packages/graph-renderer`
Expected: FAIL — module not found

- [ ] **Step 3: Implement RubberBandSelect**

Create `packages/graph-renderer/src/editing/rubber-band-select.ts`:

```typescript
import { validateSelection } from '@nicecases/graph-core';
import type { GraphModel, GraphEdge } from '@nicecases/graph-core';

export type RubberBandResult =
  | { readonly type: 'selected'; readonly nodeIds: ReadonlySet<string>;
      readonly boundaryInput: GraphEdge; readonly boundaryOutput: GraphEdge }
  | { readonly type: 'empty' };

interface FlowNode {
  readonly id: string;
  readonly position: { readonly x: number; readonly y: number };
  readonly measured?: { readonly width?: number; readonly height?: number };
}

export interface RubberBandOptions {
  readonly containerEl: HTMLElement;
  readonly screenToFlow: (x: number, y: number) => { x: number; y: number };
  readonly getNodes: () => readonly FlowNode[];
  readonly getModel: () => GraphModel;
  readonly onComplete: (result: RubberBandResult) => void;
}

export interface RubberBandSelect {
  attach(): void;
  dispose(): void;
}

export function createRubberBandSelect(opts: RubberBandOptions): RubberBandSelect {
  const { containerEl, screenToFlow, getNodes, getModel, onComplete } = opts;

  let active = false;
  let startScreen = { x: 0, y: 0 };
  let rectEl: HTMLElement | null = null;

  const onPointerDown = (e: PointerEvent) => {
    // Only start on background (not on nodes or edges)
    const target = e.target as HTMLElement;
    if (target.closest('.react-flow__node') || target.closest('.react-flow__edge')) return;

    active = true;
    startScreen = { x: e.clientX, y: e.clientY };

    rectEl = document.createElement('div');
    rectEl.className = 'multi-select-rect';
    rectEl.style.position = 'fixed';
    rectEl.style.pointerEvents = 'none';
    rectEl.style.zIndex = '5';
    document.body.appendChild(rectEl);

    containerEl.setPointerCapture(e.pointerId);
  };

  const onPointerMove = (e: PointerEvent) => {
    if (!active || !rectEl) return;

    const x = Math.min(startScreen.x, e.clientX);
    const y = Math.min(startScreen.y, e.clientY);
    const w = Math.abs(e.clientX - startScreen.x);
    const h = Math.abs(e.clientY - startScreen.y);

    rectEl.style.left = `${x}px`;
    rectEl.style.top = `${y}px`;
    rectEl.style.width = `${w}px`;
    rectEl.style.height = `${h}px`;

    // Convert rectangle corners to flow coordinates for hit-testing
    const topLeft = screenToFlow(x, y);
    const bottomRight = screenToFlow(x + w, y + h);
    const flowRect = {
      x: Math.min(topLeft.x, bottomRight.x),
      y: Math.min(topLeft.y, bottomRight.y),
      w: Math.abs(bottomRight.x - topLeft.x),
      h: Math.abs(bottomRight.y - topLeft.y),
    };

    // Hit-test nodes
    const candidates = new Set<string>();
    for (const n of getNodes()) {
      const nw = n.measured?.width ?? 100;
      const nh = n.measured?.height ?? 50;
      if (
        n.position.x + nw > flowRect.x &&
        n.position.x < flowRect.x + flowRect.w &&
        n.position.y + nh > flowRect.y &&
        n.position.y < flowRect.y + flowRect.h
      ) {
        candidates.add(n.id);
      }
    }

    // Validate and apply CSS classes
    const result = validateSelection(candidates, getModel());
    const nodeEls = containerEl.querySelectorAll('.react-flow__node');
    for (const el of nodeEls) {
      const nodeId = (el as HTMLElement).dataset.id ?? '';
      el.classList.remove('multi-select-valid', 'multi-select-invalid');
      if (result.valid.has(nodeId)) el.classList.add('multi-select-valid');
      else if (result.invalid.has(nodeId)) el.classList.add('multi-select-invalid');
    }
  };

  const cleanup = () => {
    active = false;
    if (rectEl) {
      rectEl.remove();
      rectEl = null;
    }
    const nodeEls = containerEl.querySelectorAll('.react-flow__node');
    for (const el of nodeEls) {
      el.classList.remove('multi-select-valid', 'multi-select-invalid');
    }
  };

  const onPointerUp = (e: PointerEvent) => {
    if (!active) return;

    // Final hit-test
    const topLeft = screenToFlow(
      Math.min(startScreen.x, e.clientX),
      Math.min(startScreen.y, e.clientY),
    );
    const bottomRight = screenToFlow(
      Math.max(startScreen.x, e.clientX),
      Math.max(startScreen.y, e.clientY),
    );
    const flowRect = {
      x: Math.min(topLeft.x, bottomRight.x),
      y: Math.min(topLeft.y, bottomRight.y),
      w: Math.abs(bottomRight.x - topLeft.x),
      h: Math.abs(bottomRight.y - topLeft.y),
    };

    const candidates = new Set<string>();
    for (const n of getNodes()) {
      const nw = n.measured?.width ?? 100;
      const nh = n.measured?.height ?? 50;
      if (
        n.position.x + nw > flowRect.x &&
        n.position.x < flowRect.x + flowRect.w &&
        n.position.y + nh > flowRect.y &&
        n.position.y < flowRect.y + flowRect.h
      ) {
        candidates.add(n.id);
      }
    }

    const result = validateSelection(candidates, getModel());
    cleanup();

    if (result.valid.size > 0 && result.boundaryInput && result.boundaryOutput) {
      onComplete({
        type: 'selected',
        nodeIds: result.valid,
        boundaryInput: result.boundaryInput,
        boundaryOutput: result.boundaryOutput,
      });
    } else {
      onComplete({ type: 'empty' });
    }
  };

  const onKeyDown = (e: KeyboardEvent) => {
    if (active && e.key === 'Escape') {
      cleanup();
      onComplete({ type: 'empty' });
    }
  };

  return {
    attach() {
      containerEl.addEventListener('pointerdown', onPointerDown);
      containerEl.addEventListener('pointermove', onPointerMove);
      containerEl.addEventListener('pointerup', onPointerUp);
      document.addEventListener('keydown', onKeyDown);
    },
    dispose() {
      cleanup();
      containerEl.removeEventListener('pointerdown', onPointerDown);
      containerEl.removeEventListener('pointermove', onPointerMove);
      containerEl.removeEventListener('pointerup', onPointerUp);
      document.removeEventListener('keydown', onKeyDown);
    },
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/editing/rubber-band-select.test.ts` from `packages/graph-renderer`
Expected: PASS

- [ ] **Step 5: Wire RubberBandSelect into GraphCanvas**

In `GraphCanvas.connectedCallback()`, create and attach a `RubberBandSelect` instance. On complete, update `_multiSelect` state to constrained mode and apply `multi-select-active` CSS classes.

- [ ] **Step 6: Add multi-select CSS**

Create `packages/graph-renderer/src/css/multi-select.css` with the styles from the spec (§5.3) — `.multi-select-rect`, `.multi-select-valid`, `.multi-select-invalid`, `.multi-select-active`. Import it in the existing CSS aggregation point.

- [ ] **Step 7: Run full graph-renderer test suite**

Run: `npx vitest run` from `packages/graph-renderer`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add packages/graph-renderer/src/editing/rubber-band-select.ts packages/graph-renderer/src/editing/rubber-band-select.test.ts packages/graph-renderer/src/bridge/GraphCanvas.ts packages/graph-renderer/src/css/multi-select.css
git commit -m "feat(#402): add RubberBandSelect with live validation feedback, wire into GraphCanvas Refs #402"
```

---

## Batch 3: Segment Delete + Hold-to-Drag Splice

After this batch, all structural operations work: constrained delete with auto-join, unconstrained delete with disconnect, and segment hold-to-drag-and-splice onto edges.

### Task 5: Segment delete (constrained auto-join + unconstrained disconnect)

**Files:**
- Modify: `packages/graph-renderer/src/editing/apply-graph-edit.ts` (add `removeSegment` compound edit)
- Modify: `packages/graph-renderer/src/editing/types.ts` (add `removeSegment` to GraphEdit union)
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts` (add delete key handler)
- Modify: `packages/graph-renderer/src/editing/apply-graph-edit.test.ts` or create if absent

**Interfaces:**
- Consumes: `removeNodes` from graph-core, `EditPolicy.canConnect`, `EditPolicy.getDeleteStrategy`, `MultiSelectState`
- Produces: `GraphEdit` variant `{ type: 'removeSegment'; nodeIds: ReadonlySet<string>; bridgeEdge?: { sourceId: string; targetId: string; edgeType: string } }`

- [ ] **Step 1: Add `removeSegment` to GraphEdit union in types.ts**

```typescript
| { readonly type: 'removeSegment'; readonly nodeIds: ReadonlySet<string>;
    readonly bridgeEdge?: { readonly sourceId: string; readonly targetId: string; readonly edgeType: string } }
```

- [ ] **Step 2: Write failing test for applyGraphEdit('removeSegment')**

```typescript
it('removes segment and creates bridge edge', () => {
  // A→B→C→D — remove {B,C}, bridge A→D
  const model = createGraph(
    [node('A'), node('B'), node('C'), node('D')],
    [edge('e1', 'A', 'B'), edge('e2', 'B', 'C'), edge('e3', 'C', 'D')],
  );
  const result = applyGraphEdit(model, {
    type: 'removeSegment',
    nodeIds: new Set(['B', 'C']),
    bridgeEdge: { sourceId: 'A', targetId: 'D', edgeType: 'default' },
  });
  expect(result.model.nodes).toHaveLength(2);
  expect(result.model.edges).toHaveLength(1);
  expect(result.model.edges[0]!.source).toBe('A');
  expect(result.model.edges[0]!.target).toBe('D');
});

it('removes segment without bridge (disconnect mode)', () => {
  // A→B→C — remove {B} with no bridge
  const model = createGraph(
    [node('A'), node('B'), node('C')],
    [edge('e1', 'A', 'B'), edge('e2', 'B', 'C')],
  );
  const result = applyGraphEdit(model, {
    type: 'removeSegment',
    nodeIds: new Set(['B']),
  });
  expect(result.model.nodes).toHaveLength(2);
  expect(result.model.edges).toHaveLength(0);
});
```

- [ ] **Step 3: Implement removeSegment in applyGraphEdit**

Add a case for `removeSegment` that calls `removeNodes()` from graph-core and optionally adds a bridge edge via `addEdge()`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run` from `packages/graph-renderer`
Expected: PASS

- [ ] **Step 5: Add Delete/Backspace key handler to GraphCanvas**

In `GraphCanvas`, listen for `keydown` events. When Delete or Backspace is pressed and `_multiSelect.mode !== 'none'`:
- Constrained mode: build `removeSegment` edit with bridge from `boundaryInput.source` to `boundaryOutput.target`. Validate bridge via `editPolicy.canConnect()`. If validation fails, fall back to no bridge (disconnect).
- Unconstrained mode: build `removeSegment` edit with no bridge (disconnect).
- Dispatch via `onMutation`.
- Clear multi-select state.

- [ ] **Step 6: Run all tests**

Run: `npx vitest run` from `packages/graph-renderer`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/graph-renderer/src/editing/types.ts packages/graph-renderer/src/editing/apply-graph-edit.ts packages/graph-renderer/src/bridge/GraphCanvas.ts
git commit -m "feat(#402): segment delete with auto-join (constrained) and disconnect (unconstrained) Refs #402"
```

### Task 6: Extend NodeMoveCoordinator for segment drag

**Files:**
- Modify: `packages/graph-renderer/src/editing/node-move-coordinator.ts` (add DragSubject support)
- Modify: `packages/graph-renderer/src/editing/node-move-coordinator.test.ts` (add segment tests)
- Modify: `packages/graph-renderer/src/editing/splice-validation.ts` (add segment splice)
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts` (dispatch logic)

**Interfaces:**
- Consumes: `DragSubject` from types.ts, `MultiSelectState` from GraphCanvas, `EditPolicy.canConnect`
- Produces: Extended `DragEndResult` with `{ type: 'splice-segment'; nodeIds: ReadonlySet<string>; edgeId: string; sourceCleanup: SourceCleanupStrategy; bridgeEdge: { sourceId: string; targetId: string; edgeType: string } }`

- [ ] **Step 1: Extend DragEndResult type**

In `node-move-coordinator.ts`, extend `DragEndResult`:

```typescript
export type DragEndResult =
  | { type: 'splice'; nodeId: string; edgeId: string; sourceCleanup: SourceCleanupStrategy }
  | { type: 'splice-segment'; nodeIds: ReadonlySet<string>; edgeId: string;
      sourceCleanup: SourceCleanupStrategy;
      bridgeEdge: { sourceId: string; targetId: string; edgeType: string } }
  | { type: 'cancelled' };
```

- [ ] **Step 2: Add segment splice validation**

In `splice-validation.ts`, add:

```typescript
export function defaultCanSpliceSegmentOntoEdge(
  policy: EditPolicy,
  edge: GraphEdge,
  entryNode: GraphNode,
  exitNode: GraphNode,
  selectedNodeIds: ReadonlySet<string>,
  model: GraphModel,
): boolean {
  // Build projected model: remove target edge + all edges touching selected nodes
  const projectedEdges = model.edges.filter(
    e => e.id !== edge.id &&
    !selectedNodeIds.has(e.source) &&
    !selectedNodeIds.has(e.target),
  );
  const projected: GraphModel = { ...model, edges: projectedEdges };

  const source = nodeById(projected, edge.source);
  const target = nodeById(projected, edge.target);
  if (!source || !target) return false;

  return policy.canConnect(source, entryNode, projected)
      && policy.canConnect(exitNode, target, projected);
}
```

- [ ] **Step 3: Write failing tests for segment drag**

Append to `node-move-coordinator.test.ts`:

```typescript
describe('segment drag', () => {
  it('accepts segment DragSubject and produces splice-segment result', () => {
    // ... test with mock pointer events for segment subject
  });

  it('skips edges connected to any node in the segment', () => {
    // ... test that edges touching selected nodes are not valid drop targets
  });

  it('creates bounding-box ghost for segment', () => {
    // ... test ghost element creation with node count label
  });
});
```

- [ ] **Step 4: Implement DragSubject branching in NodeMoveCoordinator**

Modify `createNodeMoveCoordinator` to accept `startDrag(subject: DragSubject, event, model)` (or keep `startDrag(nodeId, event, model)` and add `startSegmentDrag(subject, event, model)` if a unified signature is too disruptive to existing callers).

Branch on subject type at:
- **Eligibility**: segment checks all nodes are root-level
- **Ghost creation**: segment creates bounding-box element with count label
- **Edge skip filter**: segment skips edges where source OR target is in `nodeIds`
- **Splice validation**: segment calls `defaultCanSpliceSegmentOntoEdge` with entry/exit
- **Result construction**: segment produces `splice-segment` type

- [ ] **Step 5: Run tests to verify they pass**

Run: `npx vitest run src/editing/node-move-coordinator.test.ts` from `packages/graph-renderer`
Expected: PASS

- [ ] **Step 6: Wire dispatch logic in GraphCanvas**

Modify the `pointerdown` handler in GraphCanvas:
1. Check if clicked node is in `_multiSelect.selectedNodeIds`
2. If yes and mode is `constrained`: pass `DragSubject { type: 'segment', ... }` to coordinator
3. If no: clear multi-select, use existing `DragSubject { type: 'single', nodeId }` flow

Handle the `splice-segment` result in `_handleMoveResult`: build a compound `GraphEdit` (source cleanup + target splice) and dispatch via `onMutation`.

- [ ] **Step 7: Add `moveSegmentToEdge` to applyGraphEdit**

Add a case in `applyGraphEdit` for `moveSegmentToEdge`:
1. Source cleanup: remove all edges touching selected nodes, add bridge edge
2. Target splice: remove target edge, add entry edge and exit edge

- [ ] **Step 8: Run all tests**

Run: `npx vitest run` from `packages/graph-renderer`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add packages/graph-renderer/src/editing/node-move-coordinator.ts packages/graph-renderer/src/editing/node-move-coordinator.test.ts packages/graph-renderer/src/editing/splice-validation.ts packages/graph-renderer/src/editing/apply-graph-edit.ts packages/graph-renderer/src/bridge/GraphCanvas.ts
git commit -m "feat(#402): extend NodeMoveCoordinator for segment drag-and-splice Refs #402"
```

---

## References

- [2026-09-02-multi-select-graph-nodes-design.md] — design spec this plan implements
- [decisions.md] — decision log (D1-D7)
- [2026-08-28-edge-splice-design.md] — NodeMoveCoordinator pattern, splice validation
- [2026-08-26-diagram-editing-infrastructure-design.md] — EditPolicy SPI, GraphEdit union
- `packages/graph-core/src/model.ts:1-20` — GraphNode, GraphEdge, GraphModel
- `packages/graph-core/src/edit.ts:1-135` — edit operations
- `packages/graph-core/src/query.ts:1-21` — query helpers
- `packages/graph-core/src/traversal.ts:1-58` — tree traversal
- `packages/graph-renderer/src/editing/node-move-coordinator.ts:1-293` — existing coordinator
- `packages/graph-renderer/src/editing/splice-validation.ts:1-33` — projected-model validation
- `packages/graph-renderer/src/editing/types.ts:1-39` — GraphEdit, EditPolicy, DeleteStrategy
- `packages/graph-renderer/src/editing/apply-graph-edit.ts:1-115` — edit application
- `packages/graph-renderer/src/bridge/GraphCanvas.ts:1-293` — Lit web component wrapper
- GitHub #402 — focal issue
