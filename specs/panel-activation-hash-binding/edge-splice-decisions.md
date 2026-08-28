# Edge-Splice Decisions

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

**Choice:** Pointer-event-based ghost+clone — nodes are NOT draggable. Pointerdown ghosts the original in place, a cloned preview follows the cursor, pointerup on valid edge completes the splice.
**Alternatives:**
- React Flow drag with position freeze — fighting React Flow's position updates causes flicker
- HTML5 DnD API — poor dragover precision, bitmap ghost, no touch support
**Rationale:** Full control over visuals, no framework fighting. Node stays in layout position during drag. Only on valid drop does it get removed and auto-layout re-runs.
**Trade-offs:** More custom code — implementing drag from scratch rather than using React Flow's built-in system.
**Sources:** GE-20260825-309197 (standalone coordinator pattern), `graph-renderer/src/bridge/ReactFlowApp.tsx:nodesDraggable=false`
**Exploration:** quick
**Status:** captured

## D3: Gesture split — move vs connect

**Choice:** Body=move, handle=connect — shrink source handles to small connection ports at node edges. Drag from body starts ghost+clone move. Drag from port starts connection.
**Alternatives:**
- Default=move, modifier=connect — hides the connect gesture behind a modifier key
- Default=connect, modifier=move — preserves current UX but hides the move gesture
- Dwell time — adds latency to one gesture
**Rationale:** Clear, standard UX. Both gestures are discoverable without modifier keys.
**Trade-offs:** Changes existing full-node-drag-to-connect UX. Connection ports add visual elements to nodes.
**Sources:** GE-20260827-f03231 (full-node handles technique — being replaced), `stencil-wrapper.tsx`
**Exploration:** quick
**Status:** captured

## D4: Container node scope

**Choice:** Leaf nodes only — only nodes without children can be dragged for edge-splice.
**Alternatives:**
- All nodes — any node including containers with children can be dragged, moving the entire subtree
**Rationale:** Simpler implementation, avoids subtree complexity. Containers stay fixed.
**Trade-offs:** Users cannot rearrange container nodes via drag-to-splice.
**Sources:** `graph-core/src/traversal.ts:childrenOf`
**Exploration:** quick
**Status:** captured

## D5: Partial validity behavior

**Choice:** Drop with visual warning — amber indicator during hover, splice immediately on drop with valid connection only, leave invalid end disconnected.
**Alternatives:**
- Confirmation dialog — ask before partial splice
- Context menu on drop — offer multiple options
**Rationale:** Low-friction UX. The amber indicator communicates partial validity during drag. No dialog interruption.
**Trade-offs:** User might accidentally drop on a partial-validity edge without noticing the amber indicator.
**Sources:** `graph-renderer/src/editing/edit-policy.ts:canConnect`
**Exploration:** quick
**Status:** captured
