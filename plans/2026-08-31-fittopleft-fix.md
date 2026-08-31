# Fix: FitTopLeft clips tall diagrams — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #397 — fix: FitTopLeft clips tall diagrams — zoom should fit all content within viewport
**Issue group:** #397

**Goal:** Fix FitTopLeft to use absolute node positions for correct bounds computation in nested diagrams.

**Architecture:** Extract a helper function `nodeAbsolutePos(node)` that reads `internals.positionAbsolute` with fallback to `node.position`. Use it in both `computeBounds` and `doFit`. Change `doFit` to read from `nodeLookup` (which provides `InternalNode` with absolute positions) instead of `getNodes()`.

**Tech Stack:** TypeScript, React, @xyflow/react v12, Vitest

## Global Constraints

- No changes to the public `ReactFlowAppProps` interface
- No changes to the `@xyflow/react` dependency version
- Access to `internals.positionAbsolute` is typed via inline cast, not by importing internal types

---

## Batch 1: Fix and verify

### Task 1: Fix computeBounds and doFit to use absolute node positions

**Files:**
- Modify: `packages/graph-renderer/src/bridge/ReactFlowApp.tsx`
- Modify: `packages/graph-renderer/src/bridge/ReactFlowApp.test.tsx`

**Interfaces:**
- Consumes: `@xyflow/react` store's `nodeLookup: Map<string, Node>` (runtime `InternalNode` objects with `internals.positionAbsolute`)
- Produces: no new exports — behavioral fix only

- [ ] **Step 1: Write failing test for computeBounds with nested nodes**

`computeBounds` is currently not exported. First, export it in `ReactFlowApp.tsx` by adding `export` to line 62:

```typescript
export function computeBounds(nodes: Node[]): string {
```

Then add this test to `ReactFlowApp.test.tsx`:

```typescript
import { computeBounds } from './ReactFlowApp.js';

describe('computeBounds', () => {
  it('uses absolute position for child nodes', () => {
    const parent = {
      id: 'parent',
      position: { x: 100, y: 50 },
      measured: { width: 300, height: 400 },
      data: {},
    } as Node;

    const child = {
      id: 'child',
      position: { x: 20, y: 30 },
      parentId: 'parent',
      measured: { width: 260, height: 40 },
      data: {},
      internals: { positionAbsolute: { x: 120, y: 80 } },
    } as Node;

    const bounds = computeBounds([parent, child]);
    const [minX, minY, maxX, maxY] = bounds.split(',').map(Number);

    // Parent: absolute (100,50) to (400,450)
    // Child: absolute (120,80) to (380,120)
    // Combined: (100,50) to (400,450)
    expect(minX).toBe(100);
    expect(minY).toBe(50);
    expect(maxX).toBe(400);
    expect(maxY).toBe(450);
  });

  it('falls back to node.position when internals not present', () => {
    const node = {
      id: 'n1',
      position: { x: 10, y: 20 },
      measured: { width: 100, height: 50 },
      data: {},
    } as Node;

    const bounds = computeBounds([node]);
    const [minX, minY, maxX, maxY] = bounds.split(',').map(Number);

    expect(minX).toBe(10);
    expect(minY).toBe(20);
    expect(maxX).toBe(110);
    expect(maxY).toBe(70);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run ReactFlowApp.test.tsx`
Expected: "uses absolute position for child nodes" FAILS — `minX` is 20 (relative) not 100 (from parent's absolute position)

- [ ] **Step 3: Implement the fix**

In `ReactFlowApp.tsx`, add a helper type and function above `computeBounds`:

```typescript
interface InternalNodeLike {
  internals?: { positionAbsolute?: { x: number; y: number } };
}

function nodeAbsolutePos(node: Node): { x: number; y: number } {
  return (node as unknown as InternalNodeLike).internals?.positionAbsolute ?? node.position;
}
```

Then update `computeBounds` (line 62) to use the helper:

```typescript
export function computeBounds(nodes: Node[]): string {
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  for (const node of nodes) {
    const w = node.measured?.width ?? node.width ?? 150;
    const h = node.measured?.height ?? node.height ?? 40;
    const pos = nodeAbsolutePos(node);
    minX = Math.min(minX, pos.x);
    minY = Math.min(minY, pos.y);
    maxX = Math.max(maxX, pos.x + w);
    maxY = Math.max(maxY, pos.y + h);
  }
  return `${Math.round(minX)},${Math.round(minY)},${Math.round(maxX)},${Math.round(maxY)}`;
}
```

Then update the inline bounds loop in `doFit` (lines 91-98) to also use `nodeAbsolutePos` and read from `nodeLookup` instead of `getNodes()`:

Replace `doFit`'s node source and bounds loop. In the `FitTopLeft` component:

1. Add a store selector for nodeLookup:
```typescript
const nodeLookup = useStore((s) => s.nodeLookup);
```

2. Replace the `doFit` callback to use `nodeLookup` instead of `getNodes()`:
```typescript
const doFit = useCallback(() => {
  const measured = Array.from(nodeLookup.values());
  if (measured.length === 0 || vw === 0 || vh === 0) return;

  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  for (const node of measured) {
    const w = node.measured?.width ?? node.width ?? 150;
    const h = node.measured?.height ?? node.height ?? 40;
    const pos = nodeAbsolutePos(node);
    minX = Math.min(minX, pos.x);
    minY = Math.min(minY, pos.y);
    maxX = Math.max(maxX, pos.x + w);
    maxY = Math.max(maxY, pos.y + h);
  }

  const pad = 20;
  const contentW = maxX - minX;
  const contentH = maxY - minY;
  if (contentW <= 0 || contentH <= 0) return;

  const zoom = Math.min((vw - pad * 2) / contentW, (vh - pad * 2) / contentH, 1);
  setViewport({ x: -minX * zoom + pad, y: -minY * zoom + pad, zoom });
  lastFittedBounds.current = bounds;
  userInteracted.current = false;
}, [nodeLookup, setViewport, vw, vh, bounds]);
```

3. Remove `getNodes` from the `useReactFlow()` destructuring (no longer needed).

- [ ] **Step 4: Run all tests**

Run: `yarn workspace @casehubio/graph-renderer run test -- --run ReactFlowApp.test.tsx`
Expected: ALL tests pass

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: No new type errors (pre-existing examples/ errors are OK)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/bridge/ReactFlowApp.tsx packages/graph-renderer/src/bridge/ReactFlowApp.test.tsx
git -C /Users/mdproctor/claude/casehub/pages commit -m "fix(#397): FitTopLeft uses absolute positions for correct nested-node bounds

computeBounds and doFit now read internals.positionAbsolute (with
fallback to node.position) so child nodes inside container groups
contribute correct absolute coordinates. doFit reads from nodeLookup
instead of getNodes() to access InternalNode objects.

Closes #397"
```

## References

- [2026-08-31-fittopleft-fix-design.md] — design spec this plan implements
- `packages/graph-renderer/src/bridge/ReactFlowApp.tsx:62-110` — computeBounds + FitTopLeft + doFit
- `packages/graph-renderer/src/bridge/ReactFlowApp.test.tsx` — existing test structure and mocking pattern
- `packages/graph-renderer/src/stencil-wrapper.test.tsx:47-48` — positionAbsolute field usage
- `packages/graph-renderer/src/mapping.ts:81-83` — parent chain walking (confirms repo awareness of relative positions)
- [GitHub #397] — issue with observed clipping in risk-aggregator drill-down
