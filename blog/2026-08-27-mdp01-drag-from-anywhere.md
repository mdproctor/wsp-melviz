---
layout: post
title: "Drag from Anywhere"
date: 2026-08-27
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [graph-editing, react-flow, ux, editpolicy, connection-handling]
---

# Drag from Anywhere

The graph renderer could display diagrams. It couldn't edit them. That's
the gap this work closes — an editing infrastructure built on two ideas:
grammar-driven policies and anywhere-drag connection UX.

## The EditPolicy SPI

The core question for any diagram editor: what's allowed? Can this node
connect to that one? What types can be inserted on an existing edge?
What happens when you delete a node that sits between two others?

I wanted these answers to come from the grammar, not from hardcoded logic.
Every stencil already declares its connection rules — inbound max, outbound
max, allowed types. `defaultEditPolicy()` reads those declarations and
derives the full editing policy automatically. A consumer registers five
stencils with grammar rules and gets a complete editor without writing
validation code.

The policy is an SPI. Override `canConnect` if your domain has constraints
the grammar can't express. Override `getDeleteStrategy` if deleting a
filter node should auto-join its predecessor to its successor instead of
disconnecting. The default handles the common case; the SPI handles
everything else.

Operations are data, not methods. `GraphEdit` is a discriminated union —
`addNode`, `removeNode`, `addEdge`, `splitEdge`, `compound`. You can log
them, replay them, compose them. `applyGraphEdit()` is a pure function
that takes a graph and an edit and returns a new graph. Undo is just
applying the inverse. Compound edits let "insert a node on this edge"
decompose into removeEdge + addNode + addEdge + addEdge in a single
atomic operation.

## The Handle Problem

React Flow's connection model assumes small circular handles at specific
positions on each node. You drag from a handle, drop on a handle. For an
auto-layout editor where ELK positions everything, handle locations are
meaningless. What I wanted: drag from anywhere on a node to start a
connection, drop anywhere on a target to complete it.

The solution is invisible full-node handles. Each node gets two: a source
handle covering 100% of the node at z-index 2, and a target handle at
z-index 1 underneath it. When a drag starts, a CSS class toggles
`pointer-events: none` on all source handles — the target handles
underneath become the drop targets. The node IS the handle, in both
directions.

That wasn't enough. React Flow's `onConnectEnd` callback gives you an
event whose `target` is always the pane overlay — never the actual node
under the cursor. The pane sits above everything and intercepts all
pointer events during a connection drag. We had to use
`document.elementsFromPoint()` to see through the overlay stack and find
the real target. And even that failed initially — the connection line SVG
and edge interaction layers also blocked the hit test. Those needed
`pointer-events: none` too, but only during the drag, applied via the
same CSS class toggle.

Setting `connectionRadius` to zero was the final piece. React Flow's
default radius-based matching creates false positives — it connects to
nodes that are "close enough." With full-node handles and element-based
hit detection, the rule is simple: if the pointer is over the node, it
connects. If it isn't, it doesn't.

## The Type Picker

Dropping a connection line on empty space fires a `graph:connect:end-on-empty`
event. The companion script catches it, queries `getCreatableTypes()` from
the edit policy, and shows a picker at the drop location. Pick "Transform"
and the node appears, already connected to the source. Click an existing
edge and `getInsertableTypes()` drives the picker — the selected type
splits the edge with a compound edit.

The picker is SPI-driven end to end. The stencil implementor's grammar
defines what appears. The five-node pipeline domain — source, transform,
filter, join, sink — demonstrates this: a source can connect to transforms,
filters, and joins, but never to another source or directly to a sink.
The picker shows exactly what's valid for each context.

## What This Opens Up

Any consumer building a visual editor — workflow designers, data pipeline
builders, case modelling tools — gets grammar-driven editing, full-node
drag UX, type pickers, and auto-join deletion out of the box. The example
gallery has three working showcases: static rendering, interactive editing
with mutation logging, and a property palette wired to node selection with
rich form controls (colour pickers, sliders, date pickers, tag inputs).

The palette component itself is generic — `<pages-diagram-palette>` is
the next piece, giving consumers a draggable stencil tray alongside the
canvas. The editing infrastructure is the foundation; the palette is the
last UX primitive before domain-specific editors can be assembled purely
from configuration.
