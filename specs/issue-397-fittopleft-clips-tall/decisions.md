## D1: Bounds computation strategy for nested nodes

**Choice:** Use `internals.positionAbsolute` from nodeLookup — access the absolute positions ReactFlow already computes
**Alternatives:**
- Walk parent chain — compute absolute positions from public API; duplicates ReactFlow's work, O(N × depth)
- Use ReactFlow's getNodesBounds() — delegates to framework utility but may not be available in store selector context
**Rationale:** `nodeLookup` already stores `InternalNode` objects with `internals.positionAbsolute`. Using these avoids duplicating ReactFlow's parent-chain walk and is a minimal code change — swap the position source in `computeBounds` and `doFit`.
**Trade-offs:** Accesses ReactFlow's internal type (`internals.positionAbsolute`), not the public `Node` type. Could break on major version upgrades of `@xyflow/react`. Acceptable for a pre-release project.
**Sources:** ReactFlowApp.tsx:62-73 (computeBounds), ReactFlowApp.tsx:91-98 (doFit), stencil-wrapper.test.tsx:47-48 (positionAbsolute fields), issue #397
**Exploration:** quick
**Status:** captured
