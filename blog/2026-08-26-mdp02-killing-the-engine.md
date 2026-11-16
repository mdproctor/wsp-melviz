---
layout: post
title: "Killing the Engine"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [refactoring, container-model, architecture, engine-elimination]
series: issue-367-container-toolbar-unification
---

# Killing the Engine

*Continues from [The DnD Protocol and the Engine Boundary](2026-08-26-mdp01-dnd-protocol-and-the-engine-boundary.md).*

The last entry ended with the engine boundary identified — everything below it built and tested, everything above it still talking to the engine. This session was about cutting across that seam.

The plan said Tasks 8-9: rewrite `wireFloatingWorkspace`, migrate `activation.ts` and `frame-detach-handler.ts`, delete the engine. What I found when I mapped the actual dependency graph was wider. The plan missed three consumers: `frame-keyboard.ts` uses `engine.focusDirection` and `engine.bringToFront` for spatial navigation, `frame-zone-picker.ts` uses snap methods, and `site.ts` holds the engine reference for layout capture during page transitions. You can't delete the engine without migrating all of them — it's genuinely atomic.

The zone picker discovery was the good surprise. The free-layout strategy already has its own zone picker built into each frame's chrome — the `showZonePicker` function creates the same 3x3 grid with zone buttons. The engine-level `createFrameZonePicker` was a parallel implementation that existed because the engine predated the strategy. Deleting it was subtraction, not migration.

The keyboard handler was a clean signature change: `engine: FloatingFrameEngine` became `rootContainer: Container`. Spatial navigation works by pulling positions from `organiser.getState()` and calling the existing `findSpatialTarget` pure function. I generalized `findSpatialTarget` to accept a `SpatialFrame` interface instead of the full `FrameLayout` type — both satisfy it, but the strategy's entry state is lighter.

The wire rewrite went from 362 lines to 140. The old function created an engine, wired a dozen backend callbacks (move, resize, close, pin, tab-drag-out, edge-split, tab-reorder, layout-change, view-mode-toggle, add-tab, tab-removed), created a detach handler, a zone picker, and a container toolbar. The new function creates a root Container with `layout: "free"`, mounts it, wires a detach handler and toolbar. The Container handles everything the engine+backend used to coordinate.

The `restoreChild` helper inside wire deserves mention. When restoring from saved `ContainerState`, each entry's child container tree needs recursive reconstruction. The function walks the `ContainerState.tabs` array, creates entries with `childContainer` for nested states, and restores layout + state on each child. It replaced the backend's 70-line `restoreContainerFromState` call site with its factory function plumbing.

After the wire landed, `activation.ts` simplified dramatically. The `ContentManager` (`workspace-content-lifecycle.ts`) dissolved — its job was bridging engine re-creation across page navigations, which the persistent Container doesn't need. The `createGroupOrganiserBackend()` call disappeared. The cross-frame drop handler disappeared — the free-layout strategy handles DnD internally. The accordion view-mode loop disappeared — Container layout is set during restoration.

Then came the question I'd been avoiding: is the backend dead? `createGroupOrganiserBackend` is a 1094-line function that renders frames, handles DnD, manages z-order, creates per-frame containers, and wires title-bar interactions. All of that is now done by the free-layout strategy. The backend was only called from `activation.ts`, and we just removed that call. It's exported from `index.ts`, but we're pre-release — no external consumers. I deleted it along with the `FloatingFrameBackend` interface and `FrameState` type.

Net result: 4036 lines deleted, 433 added across 28 files. The triple-state architecture (engine tracking positions + backend rendering DOM + strategy managing layout) is now a single root Container. Mode transitions that used to destroy one rendering system and rebuild another are now `rootContainer.setLayout("tabbed")`.

I found one pre-existing bug during the typecheck pass. The accordion strategy's `detachEntry` method used `hostElement?.querySelector(...)` where `hostElement` was never declared in scope — the correct variable was `containerEl`. Optional chaining made this a silent no-op instead of a ReferenceError. The DOM section element was never removed on detach, but no test caught it because `?.` on undefined returns undefined, and `undefined?.remove()` is also a no-op. A bug hiding behind two layers of null safety.

Remaining work is Task 10 — integration tests for DnD transfer, workspace mode switching, and migration edge cases. The architecture is done.
