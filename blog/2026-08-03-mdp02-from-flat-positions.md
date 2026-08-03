---
layout: post
title: "From Flat Positions to a Containment Tree"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [graph-renderer, elk, layout, interaction]
series: issue-265-graph-renderer
---

*Part of a series on [#265 — Phase 1B graph-renderer](https://github.com/casehubio/casehub-pages/issues/265). Previous: [Three Issues, One Pipeline](2026-08-03-mdp01-three-issues-one-pipeline.md).*

Phase 0's ELK adapter took React Flow nodes as input, reconstructed the containment tree by scanning `parentId` on every node, built the ELK hierarchy, then returned... React Flow nodes with updated positions. The graph model already knew the containment structure. The adapter was throwing it away and reconstructing it from the rendering framework's flat representation.

The fix (#274) is the kind of change that looks obvious once you see it: `computeElkLayout` accepts `GraphModel` directly and walks the tree with `rootNodes()` and `childrenOf()` from graph-core. No reconstruction. The return type changes from positioned React Flow nodes to an `ElkLayoutResult` — a framework-agnostic map of node positions and dimensions. The mapping layer applies the layout when converting to React Flow types. Layout computation and framework adaptation are now independently testable, and `GraphCanvas` orchestrates a simpler pipeline: model in, ELK layout, mapping with positions, React Flow render.

The container sizing was the other half of this. Phase 0 used hardcoded 280x180 dimensions for parent nodes — containers didn't resize when children were added. ELK computes container dimensions from children plus padding, so applying those dimensions from the layout result means containers actually fit their content. A `containerPadding` option controls the space between container boundary and children.

The interaction layer (#275) uncovered a platform consistency bug. The event topics used dash-separated segments — `graph:node-click` — but the platform's `matchesTopic()` function splits on colons for hierarchical wildcard matching. A listener subscribing to `graph:*:click` wouldn't match `graph:node-click` because `node-click` is one opaque segment. Fixing this across all graph events (and enriching the pages-event-contract protocol to explicitly prohibit dashes within segments) was worth doing alongside the new edge click and viewport change events. The kind of bug that never fails a test but defeats a consumer pattern.

The toolbar (#276) was the lightest touch of the three — React Flow's `<Controls>` already renders zoom in/out/fit-to-view, so the only addition was a `ControlButton` for re-layout and CSS styling with `--pages-*` design tokens. A re-layout button that calls the same `_runLayout()` pipeline, no new code path.

Six issues across two sessions closes out Phase 1B. The graph-renderer now has a complete pipeline from domain model to rendered, interactive diagram: typed model input, grammar-driven stencil registration, Lit-to-React custom node rendering, hierarchical ELK layout from the containment tree, colon-separated interaction events, and basic controls. What it doesn't yet have — and what Phase 2 opens — is domain-specific stencils, graph editing operations wired to the UI, and the diagram toolbar that a casehub application would actually ship.
