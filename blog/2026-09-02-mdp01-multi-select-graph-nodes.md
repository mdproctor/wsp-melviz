---
layout: post
title: "Multi-Select in a Pipeline Editor: 1-in/1-out and the Constraints That Make It Work"
date: 2026-09-02
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [graph-editor, reactflow, selection, ux, typescript]
series: issue-402-multi-select-graph-nodes
---

# Multi-Select in a Pipeline Editor: 1-in/1-out and the Constraints That Make It Work

Selecting multiple nodes in a pipeline graph isn't the same problem as selecting files in a file manager. In a file manager, any combination is valid. In a pipeline editor, the selection must be structurally meaningful — otherwise delete, move, and splice operations produce corrupt graphs.

I spent today building multi-select for the CaseHub graph editor. The constraint that makes it work: a valid selection must have exactly one edge entering it and one edge leaving. This "1-in/1-out" rule guarantees that deleting the selection can auto-join the gap, and splicing it elsewhere produces a valid graph.

## The Constraint

Consider a pipeline `Source → Parse → Enrich → Validate → Sink`. Selecting `{Parse, Enrich}` gives us one inbound edge (`Source→Parse`) and one outbound edge (`Enrich→Validate`). Delete the selection, and you can bridge `Source→Validate` — a clean auto-join.

But add a branch. Say `Enrich` also connects to a `Normalise` node. Now selecting `{Parse, Enrich}` without `Normalise` creates two outbound edges from the selection boundary: `Enrich→Validate` and `Enrich→Normalise`. The 1-in/1-out invariant breaks. Delete would leave `Normalise` dangling.

The fix isn't to allow the delete and clean up — it's to prevent the selection. The constraint is enforced at selection time, not at operation time.

## Two Selection Modes

I needed two distinct gestures. Shift+drag draws a rubber-band rectangle — this is the constrained mode. It validates 1-in/1-out in real time: nodes inside the rectangle get blue outlines when valid, red when invalid. On release, only a valid set is selected (or nothing if the constraint can't be satisfied).

Shift-click on individual nodes is unconstrained. Toggle any nodes you like — useful for bulk delete (disconnect strategy, no auto-join) or eventual bulk property editing.

The modes don't mix. Drag-select creates a constrained selection. Shift-click from scratch creates an unconstrained one. Shift-clicking to extend a constrained selection still enforces the constraint — each toggle is validated before it's applied.

## SelectionValidator: Pure Logic, Heavily Tested

The validator lives in `graph-core` — pure function, no DOM dependency. Given a set of candidate node IDs and a graph model, it computes boundary edges and checks the constraint.

The algorithm is straightforward: partition all edges into inbound (source outside, target inside), outbound (source inside, target outside), and internal (both inside). If inbound count is exactly 1 and outbound count is exactly 1, check internal connectivity via BFS from the entry node to the exit node.

That connectivity check was a design review finding. Without it, two nodes from disconnected chains that happen to be spatially close could pass the boundary count — 1 inbound to one node, 1 outbound from the other — despite having no internal path between them. The BFS catches this.

## Fighting ReactFlow

The graph editor uses ReactFlow for rendering. Making multi-select work meant understanding where ReactFlow's opinions end and ours begin.

**Pan conflict.** ReactFlow's default click-drag on the canvas background pans the viewport. Shift+drag for rubber-band selection needed to coexist. The rubber-band handler uses capture-phase pointer events and stops propagation on `pointerdown` when Shift is held. Crucially, it does NOT stop `pointermove` — the rectangle renders in flow coordinates inside the `.react-flow__viewport` element, so it tracks correctly even if panning somehow occurs.

**Selection overlay.** When ReactFlow nodes are marked `selected: true`, it renders a `.react-flow__nodesselection-rect` overlay that intercepts all pointer events. This blocked the hold-to-drag gesture on selected nodes. The fix: `pointer-events: none` on that overlay element.

**Viewport-locked resize.** Playwright's `setViewportSize` locks the viewport so manual window resize doesn't work — a trap I fell into repeatedly while testing. The CSS itself uses `100vh` and flex throughout, which works correctly in a real browser.

## The Rubber-Band Rectangle

The rectangle lives inside ReactFlow's viewport element (`position: absolute` in flow coordinates), not on `document.body` in screen coordinates. This was a late fix — the original screen-coordinate approach caused ghosting when the viewport panned during drag, because the rectangle stayed fixed while the graph content moved underneath.

Rendering in flow coordinates means the rectangle moves with pan and zoom automatically. The `screenToFlow` bridge converts pointer screen positions to flow coordinates for both rectangle positioning and node hit-testing.

## Segment Drag-and-Splice

After selecting a valid segment, hold-clicking any node in the selection for 300ms activates drag mode. All selected nodes ghost. A bounding-box label shows the node count. Dragging over edges validates whether the segment can be spliced there — using the segment's entry node type for the source connection and exit node type for the target connection.

The splice operation preserves internal edges. This was a corruption bug found during manual testing: the original implementation removed ALL edges touching selected nodes, including the internal ones (like `Clean→Lookup` within a `{Clean, Lookup}` segment). After splicing, the segment nodes were inserted but disconnected from each other. The fix: only remove external edges — edges where exactly one endpoint is in the selection.

The `NodeMoveCoordinator` was extended with a `DragSubject` discriminated union rather than creating a separate `SegmentMoveCoordinator`. The coordinator branches on subject type at eligibility, ghost creation, edge-skip, splice validation, and result construction. Everything else — hold timer, drag threshold, pointer lifecycle — is shared.

## What's Still Missing

Bulk property editing on common schema intersections is designed but deferred. The handle-shrinking work (source handles currently cover 100% of node surface) remains a blocker for clean drag interactions — for now, `connectionsEnabled` can be set to `false` on diagrams that don't need connection drawing.

The selection validator is the piece I'm most confident in — 21 unit tests covering topology variations, boundary counting, internal connectivity, and shift-click add/remove. The interaction layer has more surface area to explore, particularly around edge cases in the pointer event lifecycle when Shift and mouse button are released in various orders.
