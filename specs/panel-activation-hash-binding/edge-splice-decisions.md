# Edge-Splice Decisions

This feature is a refinement of **infra spec interaction #7** (node-on-edge drop).
The infra spec designed #7 using React Flow native drag; these decisions refine
the interaction model for auto-layout contexts where the intermediate drag visual
matters.

## D1: Edge cleanup on source removal

**Choice:** Auto-join — if dragged node had exactly 1 inbound + 1 outbound, reconnect them. Otherwise disconnect all.
**Alternatives:**
- Always disconnect — simpler, but may leave orphaned chain ends
- Preserve and reconnect all valid pairs — most forgiving but significantly more complex
**Rationale:** Mirrors existing `getDeleteStrategy` auto-join logic. Consistent behavior whether a node is deleted or moved.
**Trade-offs:** Nodes with multiple connections lose all edges rather than getting partial reconnection.
**Sources:** `graph-renderer/src/editing/edit-policy.ts:getDeleteStrategy`, `graph-core/src/edit.ts:removeNode`
**Exploration:** quick
**Status:** captured

## D2: Drag interaction model

**Choice:** Custom pointer-event ghost+clone — `nodesDraggable` stays `false`. Pointerdown on node body ghosts the original in place, a cloned preview follows the cursor, pointerup on a valid edge completes the splice.
**Alternatives:**
- React Flow native drag as gesture detector (infra spec #7 approach) — ELK re-layouts anyway so drag is just gesture detection. BUT: during drag, the node visually moves and edges rubber-band, which looks like the node is being torn out of the graph. Confusing in auto-layout context.
- HTML5 DnD API — poor dragover precision, bitmap ghost, no touch support
**Rationale:** Auto-layout contexts need the original node to stay in place during drag so the layout context is preserved. Ghost+clone achieves this. Infra spec's "gesture detector" insight is correct in principle but the intermediate visual state (stretched edges) is unacceptable.
**Trade-offs:** More custom code. Diverges from infra spec interaction #7's implementation approach (the interaction itself is the same, the mechanism differs). Infra spec to be updated with this refinement.
**Sources:** GE-20260825-309197 (standalone coordinator pattern), infra spec §8.0 interaction #7
**Exploration:** quick → revised after decision review (R1-02)
**Status:** revised

## D3: Gesture split — move vs connect

**Choice:** Body=move, handle=connect — shrink source handles to small connection ports at node edges. Drag from body starts ghost+clone move. Drag from port starts connection.
**Alternatives:**
- Default=move, modifier=connect — hides the connect gesture behind a modifier key
- Default=connect, modifier=move — preserves current UX but hides the move gesture
- Dwell time — adds latency to one gesture
**Rationale:** Clear, standard UX. Both gestures are discoverable without modifier keys.
**Trade-offs:** Changes existing full-node-drag-to-connect UX. Connection ports add visual elements to nodes. This is a cross-cutting change that affects all node interactions (per review R1-05), but is a prerequisite for edge-splice.
**Sources:** GE-20260827-f03231 (full-node handles technique — being replaced), `stencil-wrapper.tsx`, infra spec D9 interaction mapping
**Exploration:** quick
**Status:** captured

## D4: Container node scope

**Choice:** Leaf nodes only — only nodes without children can be dragged for edge-splice. Nodes with `parentId` (children of containers) are also excluded initially to avoid containment hierarchy changes.
**Alternatives:**
- All nodes — any node including containers with children can be dragged, moving the entire subtree
- Leaf nodes including container children — allows leaf nodes inside containers to be moved, requiring parentId changes and containment validation
**Rationale:** Simpler implementation, avoids subtree complexity and containment grammar (`allowedParentTypes`) violations. Can be relaxed later.
**Trade-offs:** Only root-level leaf nodes can be spliced. Nodes inside containers cannot participate.
**Sources:** `graph-core/src/traversal.ts:childrenOf`, `graph-core/src/grammar.ts:ContainmentRules` (review R1-06)
**Exploration:** quick → refined after decision review (R1-06)
**Status:** revised

## D5: Partial validity behavior — three visual states

**Choice:** Three-state visual feedback during drag, immediate action on drop:
- **Green** (both valid): edge highlights green. Drop splices node with both connections.
- **Amber** (one valid): edge highlights amber. Drop creates only the valid connection, leaves the other end disconnected.
- **None** (neither valid): no highlight, edge is not a drop zone.
**Alternatives:**
- Both-must-be-valid — simpler, matches infra spec's two-state model, but prevents useful partial splices
- Confirmation dialog on partial — adds friction
**Rationale:** Maximizes user flexibility. The amber indicator communicates the partial nature before the user commits.
**Trade-offs:** `splitEdge` in graph-core always creates both edges — partial splice needs a custom operation that removes the target edge, adds the node, and adds only the valid edge(s). New visual language (amber) not used elsewhere in the editing infrastructure.
**Sources:** `graph-renderer/src/editing/edit-policy.ts:canConnect`, `graph-core/src/edit.ts:splitEdge` (review R1-04)
**Exploration:** quick → revised after decision review (R1-04)
**Status:** revised

## D6: Composition model — moveNodeToEdge as atomic edit

**Choice:** Implement `moveNodeToEdge` in `apply-graph-edit.ts` as a single atomic edit that performs both source-side cleanup and target-side splice. Not a `compound` edit.
**Alternatives:**
- `compound` edit with `removeNode` + `splitEdge` — exposes intermediate invalid states, two undo points unless wrapped
- Two sequential edits — validation runs between them, risking partial failure
**Rationale:** Single atomic edit = single undo unit (per infra spec D11). The implementation handles source cleanup (auto-join or disconnect), target edge removal, and new edge creation in one pass. No intermediate states exposed to validation. The existing `moveNodeToEdge` stub in `apply-graph-edit.ts:61` was designed for exactly this.
**Trade-offs:** More complex implementation in one function vs. composing simpler primitives.
**Sources:** `graph-renderer/src/editing/apply-graph-edit.ts:61` (existing stub), `graph-renderer/src/editing/types.ts:35` (existing type), infra spec D11 (undo)
**Exploration:** quick (surfaced by decision review R1-03)
**Status:** captured
**Depends on:** D1, D5
