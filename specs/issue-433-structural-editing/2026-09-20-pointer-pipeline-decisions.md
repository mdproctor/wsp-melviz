# Design Decisions — Graph Pointer Event Pipeline

## D1: Connection UX — full-node drag-to-connect

**Choice:** Keep full-node drag-to-connect (invisible handle covering entire node surface)
**Alternatives:**
- Edge ports only — visible port dots on node edges. Eliminates pointer fight but implies sticky ports, which the model doesn't have.
- Modifier key — Alt+drag for connect. Less discoverable.
**Rationale:** Visible port dots would mislead users into thinking connections bind to specific ports. The graph model has node-to-node connections, not port-to-port.
**Trade-offs:** The coordinator must disambiguate move vs connect on the same surface, which is harder than having separate interaction zones.
**Sources:** graph-renderer/src/editing/node-move-coordinator.ts, stencil-wrapper.tsx:237-240 (fullNodeHandle)
**Exploration:** quick
**Status:** captured

## D2: Gesture classification

**Choice:** Immediate drag = connect, hold 300ms then drag = move (ghost), click (no drag) = select
**Alternatives:**
- Select-first-then-drag — click selects, drag selected = move, drag unselected = connect. Clear but changes current UX.
- Drag zone — center = move, edge = connect. Spatial disambiguation but hard to learn.
**Rationale:** This is the existing gesture model and it's correct. The implementation is broken (pointer capture fight), not the design.
**Trade-offs:** 300ms hold adds latency to move operations. Acceptable because move is less frequent than connect.
**Sources:** node-move-coordinator.ts HOLD_DURATION=300 (line 6)
**Exploration:** quick
**Status:** captured

## D3: Drill-down interaction model

**Choice:** Drill-down stays as a visible ⤢ button on nodes (top-right). The coordinator ignores button clicks (button handles itself via stopPropagation).
**Alternatives:**
- Double-click = drill-down — less discoverable, conflicts with other double-click uses.
- Both button and double-click — two entry points for one action, harder to reason about.
**Rationale:** Clean separation: gestures operate on the node surface, buttons are discrete actions. The button is already implemented and understood.
**Trade-offs:** Button takes visual space on the node. Acceptable — it's small and only appears on drillable nodes.
**Sources:** blocks-ui graph-stencil-case/src/stencils/worker.ts:72, graph-stencil-swf/src/stencils/call.ts:38
**Exploration:** quick
**Status:** captured

## D4: Drill-down protocol location

**Choice:** Move the 3-phase drill-down protocol (emit → resolve → navigate) and navigation stack from blocks-ui into pages graph-renderer. Resolution stays domain-specific via a callback.
**Alternatives:**
- Keep in blocks-ui — less scope but every graph consumer would reimplement the stack.
- New package (graph-drill-down) — avoids bloating graph-renderer but adds a dependency.
**Rationale:** Drill-down is a generic graph interaction, not a CaseHub domain concept. Any graph with containment (sub-workflows, sub-processes, nested diagrams) needs it. graph-renderer is the right home.
**Trade-offs:** graph-renderer grows. Mitigated by the callback-based resolution — the renderer only owns the stack and navigation, not domain-specific YAML resolution.
**Sources:** blocks-ui diagram-workbench.ts:62-66 (stack), casehub-diagram.ts:760-805 (resolve)
**Exploration:** quick
**Depends on:** D3 (drill-down stays as button)
**Status:** captured

## D5: Pointer event interception strategy

**Choice:** Capture-phase pointerdown listener on the node container. During the 300ms hold, it calls stopPropagation to block React Flow's Handle from claiming the pointer. If no hold (immediate drag), it replays the event to let React Flow draw the connection.
**Alternatives:**
- Custom Handle component — replace React Flow's Handle with one that delegates to the coordinator. Couples to React Flow internals.
- CSS pointer-events toggle — set Handle to pointer-events:none for 300ms. Simpler but relies on CSS timing and programmatic re-trigger.
**Rationale:** One interception point, React Flow untouched. The capture phase fires before React Flow's bubble-phase listeners, giving the coordinator first claim on the event. Replaying the event for connect means React Flow's connection drawing works unmodified.
**Trade-offs:** Event replay is tricky — must reconstruct a synthetic PointerEvent with the original coordinates. Tested pattern in browser APIs.
**Sources:** node-move-coordinator.ts:80-84 (current broken pointer release), ReactFlowApp.tsx:247 (nodesDraggable=false)
**Exploration:** quick
**Status:** captured
