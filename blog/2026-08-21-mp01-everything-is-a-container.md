---
layout: post
title: "Everything is a Container"
date: 2026-08-21
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [floating-workspace, compositor, i3, sway, container-model, architecture]
series: issue-312-workspace-compositor
---

Continues from [Workspace compositor — splits, tabs, and accordion view](2026-08-13-mp01-workspace-compositor.md).

---

I had a naming problem and didn't know it.

The workspace compositor has a sandbox Group — a container that holds entries and displays them using pluggable organisers (tab strip, accordion sections, free-layout positioning). It works. Thirty-seven tests prove it. The organiser swap preserves content via detach/reattach — no state loss, no lifecycle bugs.

But when I started designing the migration from sandbox to production, the terminology fell apart. The production backend has frames, sub-frames, and split panes. The sandbox has Groups and Organisers. And when I tried to describe how they relate — "the Group handles entries inside a frame, except in splits where each pane has its own Group, but a sub-frame is also a Group when you're in free-layout mode" — I could hear the abstraction leaking.

Claude was walking through the design when I said something like "what you're referring to as a Group, I've been calling a SubFrame." And then immediately had to correct myself: no, a SubFrame is the result of splitting a frame. A frame nested inside content is just a frame. They're all containers.

*They are all containers.* That was the sentence that unstuck it.

Instead of inventing more terminology — Container vs Frame vs SubFrame vs Pane — I suggested we search for established models. The question is how tiling window managers represent this, and tiling window managers have been solving it for decades.

We found i3.

## The i3 model

i3 (and its Wayland successor Sway — same model, different display server) represents everything as a Container. A Container holds children and has a layout property. That's it. The layout determines how children are arranged:

| Layout | What it does |
|--------|-------------|
| `splith` | Children side by side with vertical dividers |
| `splitv` | Children stacked with horizontal dividers |
| `tabbed` | Children as tabs, one visible |
| `stacking` | Children stacked with visible headers |

There is no separate concept for "split pane" or "tab group" or "frame." A tabbed container's children are containers. A split container's children are containers. The tree is uniform all the way down.

This maps directly to what we're building. Our layout types are `free`, `tabbed`, `accordion`, `splith`, `splitv`, and `content` (leaf). Where i3 has `stacking`, we have `accordion`. Where i3 doesn't have floating positioning, we have `free`. But the principle is identical: one node type, pluggable layout, recursive composition.

## Why split was the tell

The naming confusion wasn't cosmetic. It was pointing at a real irregularity in the design.

Decision D11 from two sessions ago said "split is a frame-level layout mode, not an Organiser type." The reasoning at the time: a split has collapse semantics and binary structure that don't fit the Organiser interface (N entries, add/remove). So splits lived as a parallel data structure — `SplitLayout` on `FrameState`, alongside but separate from the Group tree.

This created a fork in every code path. Does the frame have a Group or a SplitLayout? Both. Neither manages the other. Every operation — add tab, toggle mode, remove pane — had to check which world it was in.

With the i3 model, split is just another layout. A Container with `splith` layout has N children arranged side by side with dividers. `addChild` adds a pane. `removeChild` removes one. When only one child remains, the split collapses automatically. No special cases. No parallel data structure. The Organiser interface objection from D11 dissolves because D12 already made splits N-ary — at which point `addChild` and `removeChild` are exactly the right operations.

The `SplitLayout` type, the `SplitPane` type, the `split-tree.ts` helper module with its `panePath: number[]` recursive addressing — all of it goes away. Containers are addressed by key, not by path.

## The rename

We renamed everything. `Group` became `Container`. `OrganiserType` became `Layout`. `setOrganiser` became `setLayout`. The string values changed too — `"free-layout"` shortened to `"free"`, `"tab"` became `"tabbed"` (matching i3's terminology exactly). Ten files renamed, every reference updated across the monorepo, all 823 tests passing.

I used IntelliJ's semantic rename for the symbols — `ide_refactor_rename` handles cross-file reference updates that text replacement misses. One interesting gotcha: when five renames ran in parallel, three timed out on the MCP response but all five completed successfully. The rename engine runs asynchronously; the timeout is on the response channel, not on the operation.

## What this opens up

The Container tree gives us state preservation for free. Each Container maintains a `Map<Layout, LayoutState>` — when you switch from tabbed to free and back, tab order and active key are restored. Per-child metadata (free-layout positions, accordion collapsed state) lives on the child Containers and survives layout switches because children persist. i3 doesn't do this — switching layout resets to defaults. We're doing better.

The production migration is now a clear path: `renderFrame` creates a Container, the Container handles everything inside (tabs, accordion, free-layout, splits), and the backend keeps only frame-level concerns (workspace positioning, z-order, decorations, cross-frame drag). The wire layer's 167-line `applyViewMode` function — which manually builds accordion DOM, saves sub-frame state, restores tab visibility — gets deleted entirely. `Container.setLayout` handles all of it.

Three tasks done this session, three remaining. The foundation is solid. The next session picks up the split strategies and then the production migration — which, thanks to the i3 model, is a simplification rather than a complication.
