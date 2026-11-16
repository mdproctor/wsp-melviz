---
layout: post
title: "The DnD Protocol and the Engine Boundary"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [drag-and-drop, container-model, architecture, refactoring]
series: issue-367-container-toolbar-unification
---

# The DnD Protocol and the Engine Boundary

The recursive container model from #345 gave us containers that nest — tabbed inside free, split inside tabbed, arbitrarily deep. What it didn't give us was a way for entries to move between containers. Today was about building that protocol.

The problem is scoping. In a nested structure, a tab dragged in an inner container needs to reach exactly the parent free-layout — not the grandparent, not every ancestor. We solved this with two DOM events and `stopPropagation`: `pages-tab-drag-start` fires when a drag begins (caught and stopped by the nearest free-layout DnD handler), and `pages-tab-escaped` fires when the pointer exits the handler's bounds, handing the drag up one level. Each level is independent. No central coordinator. The DOM tree structure does the routing.

The interesting design question was where to dispatch `pages-tab-drag-start`. The tabbed strategy detects the gesture, but it has no reference to its parent Container — strategies are intentionally decoupled. The event needs `sourceContainer` in its detail so the DnD handler knows where to call `detachEntry`. The answer: intercept the strategy's callback in the Container's `wrappedCallbacks()` and dispatch the event there. One abstraction level up from where the gesture occurs. The forward reference to the Container instance (a `const` defined later in the same function) is safe because the callback only executes during user interaction, well after construction.

Edge splits came out cleaner than expected. The DnD handler needed edge-zone detection and split geometry — both already existed as pure functions in `frame-boundaries.ts`, written for snap-zone and resize features. They composed directly into the DnD state machine's pointermove handler. No adapters, no wrappers. `detectEdgeZone` returns which edge, `splitGeometry` returns both half-frame geometries. The DnD handler just wires them together.

I also wrote the persistence migration function — `migrateFrameLayout` converts the old flat `FrameLayout[]` array to a `ContainerState` tree with a root free-layout and tabbed children. Pure function, six tests, straightforward.

That's seven of ten tasks done on the workspace-as-container plan. The remaining three — rewriting `wireFloatingWorkspace`, migrating `activation.ts`, and deleting the engine — are the hard part. The engine is consumed by seven files and has 58 tests. `activation.ts` has seven direct `handle.engine` call sites plus delegates to `workspace-content-lifecycle.ts`. It's a coupled refactor across roughly ten files and seventy tests, and it needs to land atomically because removing the engine breaks everything that imports it.

The engine boundary is the architectural seam. Everything below it — Container, strategies, DnD, layout-math, persistence — is built and tested. Everything above it — activation, detach handler, keyboard handler, content lifecycle — still talks to the engine. The next session is about cutting across that seam cleanly.
