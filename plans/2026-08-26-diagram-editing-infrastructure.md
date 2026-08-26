# Diagram Editing Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create before implementation begins
**Issue group:** Evolves Phase 4 from the visual diagram editor spec

**Goal:** Add edge operations to graph-core, define EditPolicy SPI and GraphEdit types in graph-renderer, expand ReactFlowApp's callback surface, and wire the viewport transform bridge — the full pages-side infrastructure for diagram editing interactions.

**Architecture:** graph-core gains four edge mutation functions following the existing addNode/removeNode/replaceNode pattern. graph-renderer gains EditPolicy SPI (interface + default + setter), GraphEdit discriminated union, ViewportBridge React component, and 7 new ReactFlowApp callback props. All work is in the pages repo; blocks-ui domain work (palette component, context menu, CaseEditPolicy) is a follow-on plan.

**Tech Stack:** TypeScript 5, Vitest, React 18, @xyflow/react v12, Lit 3.x

## Global Constraints

- Pre-release project — no backward compatibility concerns
- graph-core must remain a pure data package — no callbacks, no framework deps
- `@typescript-eslint/strict-type-checked` — no `any` types
- All new exports must be added to the package's `index.ts`
- Test commands: `yarn workspace @casehubio/graph-core run test` and `yarn workspace @casehubio/graph-renderer run test`

---

## Batch 1: graph-core edge operations

### Task 1: addEdge and removeEdge

**Files:**
- Modify: `packages/graph-core/src/edit.ts`
- Modify: `packages/graph-core/src/edit.test.ts`
- Modify: `packages/graph-core/src/index.ts`

**Interfaces:**
- Consumes: `GraphModel`, `GraphEdge`, `EditResult`, `validateConstraints()`, `nodeById()` from existing graph-core
- Produces: `addEdge(model: GraphModel, newEdge: GraphEdge): EditResult`, `removeEdge(model: GraphModel, edgeId: string): EditResult`

- [ ] **Step 1: Write failing tests for addEdge**

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { addEdge, removeEdge } from './edit.js';
import { registerGrammar, clearGrammarRegistry } from './grammar.js';
import type { GraphModel, GraphEdge } from './model.js';

describe('addEdge', () => {
  const baseModel: GraphModel = {
    nodes: [
      { id: 'n1', type: 'source', properties: {} },
      { id: 'n2', type: 'target', properties: {} },
    ],
    edges: [],
  };

  it('adds an edge and validates constraints', () => {
    const edge: GraphEdge = { id: 'e1', type: 'default', source: 'n1', target: 'n2' };
    const result = addEdge(baseModel, edge);
    expect(result.model.edges).toHaveLength(1);
    expect(result.model.edges[0]).toEqual(edge);
  });

  it('throws on duplicate edge ID', () => {
    const edge: GraphEdge = { id: 'e1', type: 'default', source: 'n1', target: 'n2' };
    const modelWithEdge: GraphModel = { ...baseModel, edges: [edge] };
    expect(() => addEdge(modelWithEdge, edge)).toThrow("Duplicate edge ID 'e1'");
  });

  it('throws when source node does not exist', () => {
    const edge: GraphEdge = { id: 'e1', type: 'default', source: 'missing', target: 'n2' };
    expect(() => addEdge(baseModel, edge)).toThrow("Source node 'missing' not found");
  });

  it('throws when target node does not exist', () => {
    const edge: GraphEdge = { id: 'e1', type: 'default', source: 'n1', target: 'missing' };
    expect(() => addEdge(baseModel, edge)).toThrow("Target node 'missing' not found");
  });

  it('returns constraint violations when grammar is registered', () => {
    registerGrammar({
      type: 'source',
      connections: {
        inbound: { min: 0, max: 0, allowedFrom: [] },
        outbound: { min: 0, max: 0, allowedTo: [] },
      },
    });
    const edge: GraphEdge = { id: 'e1', type: 'default', source: 'n1', target: 'n2' };
    const result = addEdge(baseModel, edge);
    expect(result.violations.length).toBeGreaterThan(0);
    clearGrammarRegistry();
  });
});
```

- [ ] **Step 2: Write failing tests for removeEdge**

```typescript
describe('removeEdge', () => {
  const modelWithEdge: GraphModel = {
    nodes: [
      { id: 'n1', type: 'source', properties: {} },
      { id: 'n2', type: 'target', properties: {} },
    ],
    edges: [{ id: 'e1', type: 'default', source: 'n1', target: 'n2' }],
  };

  it('removes the edge', () => {
    const result = removeEdge(modelWithEdge, 'e1');
    expect(result.model.edges).toHaveLength(0);
    expect(result.model.nodes).toHaveLength(2);
  });

  it('throws when edge not found', () => {
    expect(() => removeEdge(modelWithEdge, 'missing')).toThrow("Edge 'missing' not found");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-core run test -- edit.test.ts`
Expected: FAIL — `addEdge` and `removeEdge` are not exported

- [ ] **Step 4: Implement addEdge and removeEdge**

Add to `packages/graph-core/src/edit.ts` after `replaceNode`:

```typescript
export function addEdge(model: GraphModel, newEdge: GraphEdge): EditResult {
  if (model.edges.some(e => e.id === newEdge.id)) {
    throw new Error(`Duplicate edge ID '${newEdge.id}'`);
  }
  if (!model.nodes.some(n => n.id === newEdge.source)) {
    throw new Error(`Source node '${newEdge.source}' not found`);
  }
  if (!model.nodes.some(n => n.id === newEdge.target)) {
    throw new Error(`Target node '${newEdge.target}' not found`);
  }
  const newModel: GraphModel = {
    ...model,
    edges: [...model.edges, newEdge],
  };
  return { model: newModel, violations: validateConstraints(newModel) };
}

export function removeEdge(model: GraphModel, edgeId: string): EditResult {
  if (!model.edges.some(e => e.id === edgeId)) {
    throw new Error(`Edge '${edgeId}' not found`);
  }
  const newModel: GraphModel = {
    ...model,
    edges: model.edges.filter(e => e.id !== edgeId),
  };
  return { model: newModel, violations: validateConstraints(newModel) };
}
```

Add the `GraphEdge` import at the top of `edit.ts`:
```typescript
import type { GraphModel, GraphNode, GraphEdge } from './model.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-core run test -- edit.test.ts`
Expected: PASS

- [ ] **Step 6: Export from index.ts**

Add to `packages/graph-core/src/index.ts` — change the edit.js export line:
```typescript
export { addNode, removeNode, replaceNode, addEdge, removeEdge } from './edit.js';
```

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-core/src/edit.ts packages/graph-core/src/edit.test.ts packages/graph-core/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add addEdge and removeEdge to graph-core edit operations Refs #<N>"
```

### Task 2: reconnectEdge and splitEdge

**Files:**
- Modify: `packages/graph-core/src/edit.ts`
- Modify: `packages/graph-core/src/edit.test.ts`
- Modify: `packages/graph-core/src/index.ts`

**Interfaces:**
- Consumes: `GraphModel`, `GraphEdge`, `GraphNode`, `EditResult`, `validateConstraints()`, `edgeById()`, `nodeById()` from graph-core
- Produces: `reconnectEdge(model: GraphModel, edgeId: string, newTargetId: string): EditResult`, `splitEdge(model: GraphModel, edgeId: string, insertNode: GraphNode): EditResult`

- [ ] **Step 1: Write failing tests for reconnectEdge**

```typescript
describe('reconnectEdge', () => {
  const model: GraphModel = {
    nodes: [
      { id: 'n1', type: 'a', properties: {} },
      { id: 'n2', type: 'b', properties: {} },
      { id: 'n3', type: 'c', properties: {} },
    ],
    edges: [{ id: 'e1', type: 'default', source: 'n1', target: 'n2' }],
  };

  it('changes the target of an existing edge', () => {
    const result = reconnectEdge(model, 'e1', 'n3');
    expect(result.model.edges).toHaveLength(1);
    expect(result.model.edges[0]!.target).toBe('n3');
    expect(result.model.edges[0]!.source).toBe('n1');
    expect(result.model.edges[0]!.id).toBe('e1');
  });

  it('throws when edge not found', () => {
    expect(() => reconnectEdge(model, 'missing', 'n3')).toThrow("Edge 'missing' not found");
  });

  it('throws when new target not found', () => {
    expect(() => reconnectEdge(model, 'e1', 'missing')).toThrow("Target node 'missing' not found");
  });
});
```

- [ ] **Step 2: Write failing tests for splitEdge**

```typescript
describe('splitEdge', () => {
  const model: GraphModel = {
    nodes: [
      { id: 'n1', type: 'a', properties: {} },
      { id: 'n2', type: 'b', properties: {} },
    ],
    edges: [{ id: 'e1', type: 'default', source: 'n1', target: 'n2' }],
  };

  it('removes original edge, adds node, adds two new edges', () => {
    const insertNode: GraphNode = { id: 'n3', type: 'c', properties: {} };
    const result = splitEdge(model, 'e1', insertNode);
    expect(result.model.nodes).toHaveLength(3);
    expect(result.model.nodes.find(n => n.id === 'n3')).toBeDefined();
    expect(result.model.edges).toHaveLength(2);
    expect(result.model.edges.find(e => e.source === 'n1' && e.target === 'n3')).toBeDefined();
    expect(result.model.edges.find(e => e.source === 'n3' && e.target === 'n2')).toBeDefined();
    expect(result.model.edges.find(e => e.id === 'e1')).toBeUndefined();
  });

  it('throws when edge not found', () => {
    const insertNode: GraphNode = { id: 'n3', type: 'c', properties: {} };
    expect(() => splitEdge(model, 'missing', insertNode)).toThrow("Edge 'missing' not found");
  });

  it('throws when insert node has duplicate ID', () => {
    const insertNode: GraphNode = { id: 'n1', type: 'c', properties: {} };
    expect(() => splitEdge(model, 'e1', insertNode)).toThrow("Duplicate node ID 'n1'");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-core run test -- edit.test.ts`
Expected: FAIL — `reconnectEdge` and `splitEdge` not exported

- [ ] **Step 4: Implement reconnectEdge and splitEdge**

Add to `packages/graph-core/src/edit.ts`:

```typescript
export function reconnectEdge(model: GraphModel, edgeId: string, newTargetId: string): EditResult {
  const edge = model.edges.find(e => e.id === edgeId);
  if (!edge) {
    throw new Error(`Edge '${edgeId}' not found`);
  }
  if (!model.nodes.some(n => n.id === newTargetId)) {
    throw new Error(`Target node '${newTargetId}' not found`);
  }
  const newModel: GraphModel = {
    ...model,
    edges: model.edges.map(e => e.id === edgeId ? { ...e, target: newTargetId } : e),
  };
  return { model: newModel, violations: validateConstraints(newModel) };
}

export function splitEdge(model: GraphModel, edgeId: string, insertNode: GraphNode): EditResult {
  const edge = model.edges.find(e => e.id === edgeId);
  if (!edge) {
    throw new Error(`Edge '${edgeId}' not found`);
  }
  if (model.nodes.some(n => n.id === insertNode.id)) {
    throw new Error(`Duplicate node ID '${insertNode.id}'`);
  }
  const newEdge1: GraphEdge = {
    id: `${edgeId}-pre`,
    type: edge.type,
    source: edge.source,
    target: insertNode.id,
  };
  const newEdge2: GraphEdge = {
    id: `${edgeId}-post`,
    type: edge.type,
    source: insertNode.id,
    target: edge.target,
  };
  const newModel: GraphModel = {
    nodes: [...model.nodes, insertNode],
    edges: [...model.edges.filter(e => e.id !== edgeId), newEdge1, newEdge2],
    metadata: model.metadata,
  };
  return { model: newModel, violations: validateConstraints(newModel) };
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-core run test -- edit.test.ts`
Expected: PASS

- [ ] **Step 6: Export from index.ts**

Update the edit.js export line in `packages/graph-core/src/index.ts`:
```typescript
export { addNode, removeNode, replaceNode, addEdge, removeEdge, reconnectEdge, splitEdge } from './edit.js';
```

- [ ] **Step 7: Run full graph-core test suite**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-core/src/edit.ts packages/graph-core/src/edit.test.ts packages/graph-core/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add reconnectEdge and splitEdge to graph-core edit operations Refs #<N>"
```

## Batch 2: EditPolicy SPI and GraphEdit types in graph-renderer

### Task 3: GraphEdit type and EditPolicy interface

**Files:**
- Create: `packages/graph-renderer/src/editing/types.ts`
- Create: `packages/graph-renderer/src/editing/edit-policy.ts`
- Create: `packages/graph-renderer/src/editing/edit-policy.test.ts`
- Modify: `packages/graph-renderer/src/index.ts`

**Interfaces:**
- Consumes: `GraphModel`, `GraphNode`, `GraphEdge`, `StencilDescriptor` from graph-core and stencil-registry
- Produces: `GraphEdit` type, `EditPolicy` interface, `StencilTypeInfo` type, `DeleteStrategy` type, `DeleteOption` type, `setEditPolicy(policy)`, `getEditPolicy()`, `defaultEditPolicy(model)`

- [ ] **Step 1: Create types.ts with GraphEdit, EditPolicy, and supporting types**

Create `packages/graph-renderer/src/editing/types.ts`:

```typescript
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import type { StencilDescriptor } from '../registry/stencil-registry.js';

export type StencilTypeInfo = Pick<StencilDescriptor, 'type' | 'label' | 'icon'> & {
  readonly group?: string;
};

export type DeleteStrategy =
  | { readonly type: 'auto-join' }
  | { readonly type: 'disconnect' }
  | { readonly type: 'prompt'; readonly options: readonly DeleteOption[] }
  | { readonly type: 'cascade' };

export interface DeleteOption {
  readonly label: string;
  readonly strategy: 'join' | 'disconnect';
  readonly targetNodeId?: string;
}

export interface EditPolicy {
  canConnect(source: GraphNode, target: GraphNode, model: GraphModel, edgeType?: string): boolean;
  getInsertableTypes(edge: GraphEdge, model: GraphModel): StencilTypeInfo[];
  getCreatableTypes(nearNode: GraphNode | null, model: GraphModel): StencilTypeInfo[];
  canDelete(node: GraphNode, model: GraphModel): boolean;
  getDeleteStrategy(node: GraphNode, model: GraphModel, deletionSet?: ReadonlySet<string>): DeleteStrategy;
}

export type GraphEdit =
  | { readonly type: 'addNode'; readonly nodeType: string; readonly properties?: Readonly<Record<string, unknown>> }
  | { readonly type: 'removeNode'; readonly nodeId: string; readonly strategy: DeleteStrategy }
  | { readonly type: 'addEdge'; readonly sourceId: string; readonly targetId: string; readonly edgeType?: string }
  | { readonly type: 'removeEdge'; readonly edgeId: string }
  | { readonly type: 'reconnectEdge'; readonly edgeId: string; readonly newTargetId: string }
  | { readonly type: 'splitEdge'; readonly edgeId: string; readonly insertNodeType: string }
  | { readonly type: 'moveNodeToEdge'; readonly nodeId: string; readonly edgeId: string }
  | { readonly type: 'compound'; readonly edits: readonly GraphEdit[] };
```

- [ ] **Step 2: Write failing tests for defaultEditPolicy**

Create `packages/graph-renderer/src/editing/edit-policy.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { defaultEditPolicy, setEditPolicy, getEditPolicy } from './edit-policy.js';
import { registerGrammar, clearGrammarRegistry } from '@casehubio/graph-core';
import { registerStencil, clearRegistry } from '../registry/stencil-registry.js';
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import { html } from 'lit';

const dummyRender = () => html`<div>test</div>`;

function makeGrammar(type: string, outMax: number, allowedTo: string[]) {
  return {
    type,
    connections: {
      inbound: { min: 0, max: 10, allowedFrom: [] as string[] },
      outbound: { min: 0, max: outMax, allowedTo },
    },
  };
}

describe('defaultEditPolicy', () => {
  beforeEach(() => {
    clearGrammarRegistry();
    clearRegistry();
  });

  afterEach(() => {
    clearGrammarRegistry();
    clearRegistry();
  });

  const model: GraphModel = {
    nodes: [
      { id: 'n1', type: 'binding', properties: {} },
      { id: 'n2', type: 'worker', properties: {} },
    ],
    edges: [{ id: 'e1', type: 'default', source: 'n1', target: 'n2' }],
  };

  describe('canConnect', () => {
    it('returns true when no grammar registered', () => {
      const policy = defaultEditPolicy(model);
      expect(policy.canConnect(model.nodes[0]!, model.nodes[1]!, model)).toBe(true);
    });

    it('returns false when outbound allowedTo excludes target type', () => {
      registerGrammar(makeGrammar('binding', 1, ['milestone']));
      const policy = defaultEditPolicy(model);
      expect(policy.canConnect(model.nodes[0]!, model.nodes[1]!, model)).toBe(false);
    });

    it('returns false when outbound cardinality exceeded', () => {
      registerGrammar(makeGrammar('binding', 1, ['worker']));
      const policy = defaultEditPolicy(model);
      expect(policy.canConnect(model.nodes[0]!, model.nodes[1]!, model)).toBe(false);
    });
  });

  describe('canDelete', () => {
    it('returns true for any node', () => {
      const policy = defaultEditPolicy(model);
      expect(policy.canDelete(model.nodes[0]!, model)).toBe(true);
    });
  });

  describe('getDeleteStrategy', () => {
    it('returns auto-join for leaf with 1 inbound + 1 outbound', () => {
      const chain: GraphModel = {
        nodes: [
          { id: 'a', type: 'x', properties: {} },
          { id: 'b', type: 'x', properties: {} },
          { id: 'c', type: 'x', properties: {} },
        ],
        edges: [
          { id: 'e1', type: 'd', source: 'a', target: 'b' },
          { id: 'e2', type: 'd', source: 'b', target: 'c' },
        ],
      };
      const policy = defaultEditPolicy(chain);
      expect(policy.getDeleteStrategy(chain.nodes[1]!, chain)).toEqual({ type: 'auto-join' });
    });

    it('returns disconnect for node with multiple inbound', () => {
      const multi: GraphModel = {
        nodes: [
          { id: 'a', type: 'x', properties: {} },
          { id: 'b', type: 'x', properties: {} },
          { id: 'c', type: 'x', properties: {} },
        ],
        edges: [
          { id: 'e1', type: 'd', source: 'a', target: 'c' },
          { id: 'e2', type: 'd', source: 'b', target: 'c' },
        ],
      };
      const policy = defaultEditPolicy(multi);
      expect(policy.getDeleteStrategy(multi.nodes[2]!, multi)).toEqual({ type: 'disconnect' });
    });

    it('returns cascade for node with children', () => {
      const container: GraphModel = {
        nodes: [
          { id: 'parent', type: 'x', properties: {} },
          { id: 'child', type: 'y', parentId: 'parent', properties: {} },
        ],
        edges: [],
      };
      const policy = defaultEditPolicy(container);
      expect(policy.getDeleteStrategy(container.nodes[0]!, container)).toEqual({ type: 'cascade' });
    });

    it('returns disconnect when auto-join target is in deletionSet', () => {
      const chain: GraphModel = {
        nodes: [
          { id: 'a', type: 'x', properties: {} },
          { id: 'b', type: 'x', properties: {} },
          { id: 'c', type: 'x', properties: {} },
        ],
        edges: [
          { id: 'e1', type: 'd', source: 'a', target: 'b' },
          { id: 'e2', type: 'd', source: 'b', target: 'c' },
        ],
      };
      const policy = defaultEditPolicy(chain);
      const deletionSet = new Set(['b', 'c']);
      expect(policy.getDeleteStrategy(chain.nodes[1]!, chain, deletionSet)).toEqual({ type: 'disconnect' });
    });
  });
});

describe('setEditPolicy / getEditPolicy', () => {
  it('stores and retrieves custom policy', () => {
    const custom: EditPolicy = {
      canConnect: () => false,
      getInsertableTypes: () => [],
      getCreatableTypes: () => [],
      canDelete: () => false,
      getDeleteStrategy: () => ({ type: 'disconnect' as const }),
    };
    setEditPolicy(custom);
    expect(getEditPolicy()).toBe(custom);
    setEditPolicy(undefined);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-renderer run test -- edit-policy.test.ts`
Expected: FAIL — module not found

- [ ] **Step 4: Implement edit-policy.ts**

Create `packages/graph-renderer/src/editing/edit-policy.ts`:

```typescript
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import { getGrammar, getAllGrammars, inboundEdges, outboundEdges, childrenOf } from '@casehubio/graph-core';
import { getAllStencils } from '../registry/stencil-registry.js';
import type { EditPolicy, StencilTypeInfo, DeleteStrategy } from './types.js';

let currentPolicy: EditPolicy | undefined;

export function setEditPolicy(policy: EditPolicy | undefined): void {
  currentPolicy = policy;
}

export function getEditPolicy(): EditPolicy | undefined {
  return currentPolicy;
}

export function defaultEditPolicy(model: GraphModel): EditPolicy {
  return {
    canConnect(source: GraphNode, target: GraphNode, _model: GraphModel, _edgeType?: string): boolean {
      const grammar = getGrammar(source.type);
      if (!grammar) return true;

      const { outbound } = grammar.connections;
      if (outbound.allowedTo.length > 0 && !outbound.allowedTo.includes(target.type)) {
        return false;
      }

      const currentOutbound = outboundEdges(_model, source.id);
      if (currentOutbound.length >= outbound.max) {
        return false;
      }

      const targetGrammar = getGrammar(target.type);
      if (targetGrammar) {
        const { inbound } = targetGrammar.connections;
        if (inbound.allowedFrom.length > 0 && !inbound.allowedFrom.includes(source.type)) {
          return false;
        }
        const currentInbound = inboundEdges(_model, target.id);
        if (currentInbound.length >= inbound.max) {
          return false;
        }
      }

      return true;
    },

    getInsertableTypes(edge: GraphEdge, _model: GraphModel): StencilTypeInfo[] {
      const sourceNode = _model.nodes.find(n => n.id === edge.source);
      const targetNode = _model.nodes.find(n => n.id === edge.target);
      if (!sourceNode || !targetNode) return [];

      return getAllStencils()
        .filter(s => {
          const g = s.grammar;
          const inboundOk = g.connections.inbound.allowedFrom.length === 0 ||
            g.connections.inbound.allowedFrom.includes(sourceNode.type);
          const outboundOk = g.connections.outbound.allowedTo.length === 0 ||
            g.connections.outbound.allowedTo.includes(targetNode.type);
          return inboundOk && outboundOk;
        })
        .map(s => ({ type: s.type, label: s.label, icon: s.icon }));
    },

    getCreatableTypes(_nearNode: GraphNode | null, _model: GraphModel): StencilTypeInfo[] {
      return getAllStencils()
        .filter(s => {
          if (!s.grammar.containment?.allowedParentTypes) return true;
          return false;
        })
        .map(s => ({ type: s.type, label: s.label, icon: s.icon }));
    },

    canDelete(_node: GraphNode, _model: GraphModel): boolean {
      return true;
    },

    getDeleteStrategy(node: GraphNode, _model: GraphModel, deletionSet?: ReadonlySet<string>): DeleteStrategy {
      const children = childrenOf(_model, node.id);
      if (children.length > 0) {
        return { type: 'cascade' };
      }

      const inbound = inboundEdges(_model, node.id);
      const outbound = outboundEdges(_model, node.id);

      if (inbound.length === 1 && outbound.length === 1) {
        const joinTargetId = outbound[0]!.target;
        if (deletionSet && deletionSet.has(joinTargetId)) {
          return { type: 'disconnect' };
        }
        return { type: 'auto-join' };
      }

      return { type: 'disconnect' };
    },
  };
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer run test -- edit-policy.test.ts`
Expected: PASS

- [ ] **Step 6: Export from index.ts**

Add to `packages/graph-renderer/src/index.ts`:
```typescript
export { setEditPolicy, getEditPolicy, defaultEditPolicy } from './editing/edit-policy.js';
export type { EditPolicy, GraphEdit, StencilTypeInfo, DeleteStrategy, DeleteOption } from './editing/types.js';
```

- [ ] **Step 7: Run full graph-renderer test suite**

Run: `yarn workspace @casehubio/graph-renderer run test`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/editing/ packages/graph-renderer/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add EditPolicy SPI, GraphEdit types, and defaultEditPolicy to graph-renderer Refs #<N>"
```

## Batch 3: Viewport bridge and ReactFlowApp expansion

### Task 4: ViewportBridge component and GraphCanvas methods

**Files:**
- Modify: `packages/graph-renderer/src/bridge/ReactFlowApp.tsx`
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts`
- Modify: `packages/graph-renderer/src/bridge/bridge.test.ts`

**Interfaces:**
- Consumes: `useReactFlow()` from `@xyflow/react`, `EditPolicy`, `getEditPolicy()` from editing/edit-policy
- Produces: `ViewportBridge` React component (internal), `GraphCanvas.screenToFlow(x, y)`, `GraphCanvas.flowToScreen(x, y)`, expanded ReactFlowApp props (`onConnect`, `isValidConnection`, `onReconnect`, `onPaneClick`, `onNodeDragStop`, `onPaneContextMenu`, `onReactFlowReady`)

- [ ] **Step 1: Write failing test for GraphCanvas viewport methods**

Add to `packages/graph-renderer/src/bridge/bridge.test.ts`:

```typescript
describe('GraphCanvas viewport bridge', () => {
  it('exposes screenToFlow method', () => {
    const canvas = document.createElement('pages-graph-canvas') as GraphCanvas;
    expect(typeof canvas.screenToFlow).toBe('function');
  });

  it('returns undefined when React Flow not ready', () => {
    const canvas = document.createElement('pages-graph-canvas') as GraphCanvas;
    expect(canvas.screenToFlow(100, 200)).toBeUndefined();
  });

  it('exposes flowToScreen method', () => {
    const canvas = document.createElement('pages-graph-canvas') as GraphCanvas;
    expect(typeof canvas.flowToScreen).toBe('function');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/graph-renderer run test -- bridge.test.ts`
Expected: FAIL — `screenToFlow` not a function

- [ ] **Step 3: Add ViewportBridge to ReactFlowApp.tsx**

Add to `packages/graph-renderer/src/bridge/ReactFlowApp.tsx`:

Import `useReactFlow` and `ReactFlowInstance`:
```typescript
import { ReactFlow, MiniMap, Controls, Background, useReactFlow, useStore, type ReactFlowInstance, type Node, type Edge, type NodeTypes, type OnMoveEnd, type EdgeMouseHandler, type Connection, type SelectionMode } from '@xyflow/react';
```

Add the ViewportBridge component before ReactFlowApp:
```typescript
function ViewportBridge({ onReactFlowReady }: { onReactFlowReady?: (instance: ReactFlowInstance) => void }) {
  const instance = useReactFlow();
  useEffect(() => {
    onReactFlowReady?.(instance);
  }, [instance, onReactFlowReady]);
  return null;
}
```

Add new props to ReactFlowAppProps:
```typescript
interface ReactFlowAppProps {
  nodes: Node[];
  edges: Edge[];
  nodeTypes: NodeTypes;
  onNodeClick?: (nodeId: string, node: Node) => void;
  onEdgeClick?: (edgeId: string, edge: Edge) => void;
  onSelectionChange?: (nodes: Node[], edges: Edge[]) => void;
  onViewportChange?: (viewport: { x: number; y: number; zoom: number }) => void;
  onRelayout?: () => void;
  onConnect?: (connection: Connection) => void;
  isValidConnection?: (connection: Connection) => boolean;
  onReconnect?: (oldEdge: Edge, newConnection: Connection) => void;
  onPaneClick?: (event: React.MouseEvent) => void;
  onNodeDragStop?: (event: React.MouseEvent, node: Node, nodes: Node[]) => void;
  onPaneContextMenu?: (event: React.MouseEvent) => void;
  onReactFlowReady?: (instance: ReactFlowInstance) => void;
}
```

Add `<ViewportBridge>` inside ReactFlow and wire new props to the `<ReactFlow>` element:
```tsx
<ReactFlow
  nodes={nodes}
  edges={edges}
  nodeTypes={nodeTypes}
  onNodeClick={handleNodeClick}
  onEdgeClick={handleEdgeClick}
  onSelectionChange={handleSelectionChange}
  onMoveEnd={handleMoveEnd}
  onConnect={onConnect}
  isValidConnection={isValidConnection}
  onReconnect={onReconnect}
  onPaneClick={onPaneClick}
  onNodeDragStop={onNodeDragStop}
  onContextMenu={onPaneContextMenu}
  reconnectEdges
  selectionOnDrag
  selectionMode={SelectionMode.Partial}
  fitView
>
  <ViewportBridge onReactFlowReady={onReactFlowReady} />
  <MiniMap />
  <Controls />
  <Background />
</ReactFlow>
```

- [ ] **Step 4: Add viewport methods and onMutation to GraphCanvas**

Add to `packages/graph-renderer/src/bridge/GraphCanvas.ts`:

Add a private field for the React Flow instance and the onMutation property:
```typescript
private _reactFlowInstance?: ReactFlowInstance;

@property({ attribute: false })
onMutation?: (edit: GraphEdit) => void;
```

Add public methods:
```typescript
screenToFlow(screenX: number, screenY: number): { x: number; y: number } | undefined {
  return this._reactFlowInstance?.screenToFlowPosition({ x: screenX, y: screenY });
}

flowToScreen(flowX: number, flowY: number): { x: number; y: number } | undefined {
  return this._reactFlowInstance?.flowToScreenPosition({ x: flowX, y: flowY });
}
```

Wire the `onReactFlowReady` callback in `_renderReact()`:
```typescript
onReactFlowReady: (instance: ReactFlowInstance) => {
  this._reactFlowInstance = instance;
},
```

Wire the connection callbacks in `_renderReact()`:
```typescript
onConnect: (connection: Connection) => {
  const policy = getEditPolicy();
  if (policy) {
    const source = nodeById(this._currentModel, connection.source);
    const target = nodeById(this._currentModel, connection.target);
    if (source && target && policy.canConnect(source, target, this._currentModel)) {
      this.onMutation?.({ type: 'addEdge', sourceId: connection.source, targetId: connection.target });
    }
  }
  emitPagesEvent(this, 'graph:edge:create', { sourceId: connection.source, targetId: connection.target });
},
isValidConnection: (connection: Connection) => {
  const policy = getEditPolicy();
  if (!policy) return true;
  const source = nodeById(this._currentModel, connection.source);
  const target = nodeById(this._currentModel, connection.target);
  if (!source || !target) return false;
  return policy.canConnect(source, target, this._currentModel);
},
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-renderer run test -- bridge.test.ts`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `yarn workspace @casehubio/graph-renderer run test`
Expected: All tests PASS

- [ ] **Step 7: Run cross-package type check**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/bridge/ReactFlowApp.tsx packages/graph-renderer/src/bridge/GraphCanvas.ts packages/graph-renderer/src/bridge/bridge.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add viewport bridge, onMutation callback, and expanded ReactFlowApp props Refs #<N>"
```

## Batch 4: Integration verification

### Task 5: Full build and cross-package integration test

**Files:**
- No new files — verification only

**Interfaces:**
- Consumes: all exports from Batches 1–3

- [ ] **Step 1: Run graph-core tests**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: All tests PASS

- [ ] **Step 2: Run graph-renderer tests**

Run: `yarn workspace @casehubio/graph-renderer run test`
Expected: All tests PASS

- [ ] **Step 3: Run full build**

Run: `yarn build:packages`
Expected: Build succeeds

- [ ] **Step 4: Run type check**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 5: Run lint**

Run: `yarn lint`
Expected: No lint errors (or only pre-existing ones)

- [ ] **Step 6: Commit (if any lint fixes needed)**

```bash
git -C /Users/mdproctor/claude/casehub/pages add -A
git -C /Users/mdproctor/claude/casehub/pages commit -m "chore: lint fixes for diagram editing infrastructure Refs #<N>"
```

---

## References

- [2026-08-26-diagram-editing-infrastructure-design.md] — design spec this plan implements
- [packages/graph-core/src/edit.ts] — existing node edit operations (addNode, removeNode, replaceNode)
- [packages/graph-core/src/query.ts] — edgesOf, inboundEdges, outboundEdges, nodeById, edgeById
- [packages/graph-core/src/validator.ts] — validateConstraints, ConstraintViolation
- [packages/graph-core/src/grammar.ts] — StencilGrammar, registerGrammar, getGrammar
- [packages/graph-core/src/traversal.ts] — childrenOf, subtreeOf
- [packages/graph-renderer/src/bridge/ReactFlowApp.tsx] — current 5-prop React Flow wrapper
- [packages/graph-renderer/src/bridge/GraphCanvas.ts] — Lit-to-React bridge
- [packages/graph-renderer/src/registry/stencil-registry.ts] — StencilDescriptor, getAllStencils
- [GE-20260825-309197] — coordinator pattern for multi-phase interaction state machines
- [PP-20260705-bac842] — pages-event-contract
