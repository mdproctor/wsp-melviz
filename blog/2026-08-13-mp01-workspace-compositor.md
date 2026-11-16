---
layout: post
title: "Workspace compositor — splits, tabs, and accordion view"
date: 2026-08-13
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [floating-workspace, compositor, layout, architecture]
---

Six issues closed this session, zero lines of code written for any of them. Commit `4cdc204` — "floating workspace UX polish" — had landed the full epic #309 in a prior session: detach, organiser toolbar, snap-to-edge, keyboard shortcuts, and frame animations. All five children implemented, tested, wired. The GitHub issues just never got closed.

Same story for #298 (selection context bridge) — `ContextManager.updateSelection()`, the `selection-change` listener in `site.ts`, clearing on refresh and page nav — all on main, all tested. Already closed by a prior session, it turned out.

I spent the first hour of this session discovering that three issues I was about to implement were already done. The lesson isn't "check before you start" — I did check, by running the tests and reading the code. The lesson is that the gap between "code on main" and "issue closed on GitHub" is real, and it accumulates silently. Six open issues, zero actual work remaining. The issue tracker was lying about the state of the project.

With the backlog cleaned up, I moved to something genuinely new: a workspace compositor. The floating workspace today is a single implicit tab — one `FloatingFrameEngine`, one set of frames, one persistence slot. The idea is to add three primitives above it: splits, tabs, and an accordion view.

The core constraint I settled on early: no nested splits. One split, two regions, that's it. The temptation to build a recursive container model is strong — it generalises cleanly — but past two levels of nesting, users lose spatial orientation. The binary split keeps the maximum depth to three: split region → tab/accordion → frames. Predictable.

Each tab in each region gets its own `FloatingFrameEngine`. This was the first real design question: extend the engine to be multi-workspace aware, or layer a compositor above it? I went with the layer. The engine is well-scoped — it manages frames, z-order, snap zones, and organiser presets for a single workspace. Multi-workspace coordination (which tab is active, how do frames move between engines, what happens when a split collapses) is a different concern. Mixing them would have meant every engine method needing workspace-awareness, and every test needing multi-workspace setup.

The accordion view is the most interesting of the three primitives. All tabs visible simultaneously as collapsible sections in a vertical stack, resizable dividers between them, per-region scroll. The implementation is CSS flexbox — each section's content area uses a stored height as `flex-basis`, collapsed sections go `flex: 0 0 auto`, and the region container gets `overflow-y: auto`. Each section's `FloatingFrameEngine` naturally scopes its snap zones to the section's dimensions via `getBoundingClientRect()` on its own overlay container. That was a satisfying moment — the per-section scoping fell out of the existing architecture without any special handling.

Cross-tab frame drag needs the compositor to coordinate a transfer between two independent engines. The transfer path is: snapshot restorable DOM state (scroll position, focus, input values) → remove from source engine → create in target engine at drop position → content factory re-renders → restore snapshot. Content factory means content is always reconstructable from config, so the transfer doesn't need to move DOM nodes. The snapshot catches the ephemeral state the factory doesn't restore — scroll position is the one users actually notice losing.

I got the compositor core and persistence implemented — 28 tests across two modules, all green. The state management handles tab lifecycle (create, close, rename, reorder, activate), split/collapse, view mode toggle, accordion heights, and cross-region tab move. Persistence captures and restores the full compositor tree, with a migration path from the old flat `FrameLayout[]`. Five tasks remain: renderer, drag handling, frame transfer, activation integration, and the examples gallery.

The compositor is the kind of feature where the data model is 80% of the design and 20% of the code. Getting the region tree right — the binary split constraint, the collapse semantics, the accordion height persistence — determines everything downstream. The renderer and drag handlers are pure wiring against a settled data model.
