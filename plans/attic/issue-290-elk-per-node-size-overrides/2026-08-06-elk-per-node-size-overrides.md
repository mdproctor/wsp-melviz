# ELK Per-Node Size Overrides Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #290 — ELK per-node size overrides for worker inline expand
**Issue group:** #290

**Goal:** Add an optional `nodeSizes` map to `ElkLayoutOptions` so callers can override default node dimensions per node ID.

**Architecture:** Single field addition to `ElkLayoutOptions`, threaded through `computeElkLayout()` to `buildElkNode()` where a map lookup replaces hardcoded defaults. No new files, no new exports, no dependency changes.

**Tech Stack:** TypeScript, Vitest, elkjs

## Global Constraints

- `ReadonlyMap` for the `nodeSizes` parameter — layout engine never mutates caller data
- Apply `nodeSizes` unconditionally (no leaf-vs-parent branching) — ELK overrides parent dimensions anyway

---

### Task 1: Add nodeSizes to ElkLayoutOptions and buildElkNode

**Files:**
- Modify: `packages/graph-renderer/src/layout/elk-layout.ts`
- Test: `packages/graph-renderer/src/layout/elk-layout.test.ts`

**Interfaces:**
- Consumes: existing `ElkLayoutOptions`, `buildElkNode()`, `computeElkLayout()`
- Produces: extended `ElkLayoutOptions` with `nodeSizes?: ReadonlyMap<string, { width: number; height: number }>`

- [ ] **Step 1: Write the first failing test — nodeSizes overrides leaf node dimensions**

```typescript
it('applies nodeSizes override to leaf node dimensions', async () => {
  const model = createGraph(
    [
      { id: '1', type: 'a', properties: {} },
      { id: '2', type: 'a', properties: {} },
    ],
    [{ id: 'e1', type: '', source: '1', target: '2' }],
  );
  const nodeSizes = new Map([['1', { width: 300, height: 200 }]]);
  const result = await computeElkLayout(model, { nodeSizes });
  const layout1 = result.nodeLayouts.get('1')!;
  expect(layout1.width).toBe(300);
  expect(layout1.height).toBe(200);
  const layout2 = result.nodeLayouts.get('2')!;
  expect(layout2.width).toBe(172);
  expect(layout2.height).toBe(36);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run elk-layout`
Expected: FAIL — `nodeSizes` is not a valid property of `ElkLayoutOptions` (type error) or dimensions don't match (172×36 instead of 300×200)

- [ ] **Step 3: Write the second failing test — missing node IDs are ignored**

```typescript
it('ignores nodeSizes entries for node IDs not in the model', async () => {
  const model = createGraph(
    [
      { id: '1', type: 'a', properties: {} },
      { id: '2', type: 'a', properties: {} },
    ],
    [{ id: 'e1', type: '', source: '1', target: '2' }],
  );
  const nodeSizes = new Map([['nonexistent', { width: 500, height: 400 }]]);
  const result = await computeElkLayout(model, { nodeSizes });
  expect(result.nodeLayouts.size).toBe(2);
  for (const layout of result.nodeLayouts.values()) {
    expect(layout.width).toBe(172);
    expect(layout.height).toBe(36);
  }
});
```

- [ ] **Step 4: Implement the changes in elk-layout.ts**

Three changes in `elk-layout.ts`:

**4a. Add `nodeSizes` to `ElkLayoutOptions` interface (line 8):**

```typescript
export interface ElkLayoutOptions {
  direction?: 'DOWN' | 'RIGHT' | 'LEFT' | 'UP';
  spacing?: number;
  containerPadding?: number;
  nodeSizes?: ReadonlyMap<string, { width: number; height: number }>;
}
```

**4b. Add `nodeSizes` parameter to `buildElkNode` and use it (lines 27-52):**

```typescript
function buildElkNode(
  model: GraphModel,
  node: GraphNode,
  visited: Set<string>,
  padding: number,
  nodeSizes?: ReadonlyMap<string, { width: number; height: number }>,
): ElkNode {
  if (visited.has(node.id)) {
    throw new Error(`Containment cycle at node '${node.id}'`);
  }
  visited.add(node.id);

  const children = childrenOf(model, node.id);
  const size = nodeSizes?.get(node.id);
  const elkNode: ElkNode = {
    id: node.id,
    width: size?.width ?? DEFAULT_NODE_WIDTH,
    height: size?.height ?? DEFAULT_NODE_HEIGHT,
  };
  if (children.length > 0) {
    elkNode.children = children.map(c => buildElkNode(model, c, visited, padding, nodeSizes));
    elkNode.layoutOptions = {
      'elk.hierarchyHandling': 'INCLUDE_CHILDREN',
      'elk.padding': `[top=${padding},left=${padding},bottom=${padding},right=${padding}]`,
    };
  }
  return elkNode;
}
```

**4c. Thread `nodeSizes` through `computeElkLayout` (line 83):**

```typescript
const nodeSizes = options.nodeSizes;
const rootChildren = roots.map(n => buildElkNode(model, n, new Set(), padding, nodeSizes));
```

- [ ] **Step 5: Run tests to verify both pass**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run elk-layout`
Expected: ALL PASS (existing + 2 new)

- [ ] **Step 6: Run typecheck to verify no type errors**

Run: `yarn typecheck`
Expected: PASS — no errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/layout/elk-layout.ts packages/graph-renderer/src/layout/elk-layout.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#290): add nodeSizes override to ElkLayoutOptions

Extend ElkLayoutOptions with optional nodeSizes map so callers can
specify per-node dimensions instead of using DEFAULT_NODE_WIDTH/HEIGHT.
Enables worker inline expand/collapse in casehub-diagram.

Closes #290

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```
