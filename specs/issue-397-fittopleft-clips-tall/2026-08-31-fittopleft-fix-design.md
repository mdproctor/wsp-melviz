# Fix: FitTopLeft clips tall diagrams with nested nodes

**Issue:** casehubio/casehub-pages#397
**Date:** 2026-08-31

## Problem

`FitTopLeft` in `ReactFlowApp.tsx` computes content bounds using
`node.position.x` and `node.position.y`. For child nodes inside container
nodes (try/catch blocks, nested groups), `node.position` is **relative to
the parent**, not the absolute viewport position. This causes the bounds
computation to produce wrong `minX`/`minY` values, leading to an incorrect
viewport offset that positions the diagram so content falls outside the
visible area.

The zoom formula itself (`Math.min(widthFit, heightFit, 1)` on line 106)
is correct — the bug is in the position data feeding into the bounds
computation.

### Concrete failure path

1. Container node at absolute (100, 0), measured 300×500
2. Child node at relative position (20, 350), measured 260×40
3. `computeBounds` uses `node.position.x = 20` for the child (relative)
4. `minX = min(100, 20) = 20` — wrong, should be `min(100, 120) = 100`
5. Viewport offset `x = -minX * zoom + pad = -20 * zoom + pad` — wrong origin
6. Content positioned incorrectly, tall container extends beyond viewport

### Why this only affects nested diagrams

Top-level nodes have absolute positions (their `position` IS absolute).
Only nodes with `parentId` set have relative positions. Flat diagrams
work correctly; the bug surfaces only with container nodes.

## Design

### Use `internals.positionAbsolute` from nodeLookup

ReactFlow v12's `nodeLookup` store contains `InternalNode` objects with
`internals.positionAbsolute` — the correctly computed absolute position
for every node, including children. The fix uses these absolute positions
instead of the relative `node.position`.

**Two locations need the fix:**

1. **`computeBounds` function (line 62)** — used by `boundsSelector` to
   detect when bounds change. Receives nodes from `nodeLookup.values()`,
   which are already `InternalNode` objects at runtime.

2. **`doFit` callback (line 87)** — computes bounds inline for the zoom
   calculation. Currently uses `getNodes()` which returns `Node[]` without
   absolute positions. Change to read from `nodeLookup` via the store
   instead.

### Implementation

Extract absolute position from each node with a fallback:

```typescript
const pos = (node as { internals?: { positionAbsolute?: { x: number; y: number } } })
  .internals?.positionAbsolute ?? node.position;
```

Use `pos.x` and `pos.y` instead of `node.position.x` and `node.position.y`
in both `computeBounds` and the inline bounds loop in `doFit`.

For `doFit`, replace `getNodes()` with a store selector that reads
`nodeLookup` — the same source `boundsSelector` already uses. This
ensures both the change-detection path and the fit-computation path
use the same absolute positions.

### Store access in doFit

`doFit` currently depends on `getNodes` from `useReactFlow()`. Replace
this with a `nodeLookup` reference from the store:

```typescript
const nodeLookupRef = useRef<Map<string, Node>>(new Map());
const nodeLookup = useStore((s) => s.nodeLookup);
nodeLookupRef.current = nodeLookup;
```

Then in `doFit`, use `Array.from(nodeLookupRef.current.values())` instead
of `getNodes()`. This gives `InternalNode` objects with `internals.positionAbsolute`.

## Test plan

1. **Existing tests pass** — no behavioral change for flat diagrams
2. **New test: bounds computation with nested nodes** — verify `computeBounds`
   produces correct bounds when nodes have `internals.positionAbsolute`
3. **Manual verification** — drill into risk-aggregator diagram in blocks-ui
   workbench, confirm try/catch container fully visible after fit

## Files changed

| File | Change |
|------|--------|
| `packages/graph-renderer/src/bridge/ReactFlowApp.tsx` | Use absolute positions in `computeBounds` and `doFit` |

## References

- `packages/graph-renderer/src/bridge/ReactFlowApp.tsx:62-98` — computeBounds + doFit
- `packages/graph-renderer/src/stencil-wrapper.test.tsx:47-48` — positionAbsolute fields
- `packages/graph-renderer/src/mapping.ts:81-83` — parent chain walking (shows the repo knows about relative positions)
- casehubio/casehub-pages#397 — issue with failure description
