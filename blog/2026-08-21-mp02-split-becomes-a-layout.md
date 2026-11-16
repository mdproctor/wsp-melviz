---
layout: post
title: "Split Becomes a Layout"
date: 2026-08-21
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [floating-workspace, compositor, split, layout-strategy, tdd]
series: issue-312-workspace-compositor
---

Continues from [Everything is a Container](2026-08-21-mp01-everything-is-a-container.md).

---

With the Container foundation landed (rename, per-layout metadata, tree mutations), today's session was about making split a regular layout type.

The i3 model says everything is a Container and split is just another layout. The implementation backs that up: `createSplitStrategy("horizontal" | "vertical")` returns a `LayoutStrategy` — same interface as tabbed, accordion, and free-layout. Flex container with panes and dividers, equal ratios by default, positional insertion for edge-drop placement.

The collapse callback is the interesting piece. When `removeEntry` leaves a split with one child, the strategy fires `onCollapse(remainingChild)` instead of rebuilding the DOM. The caller (wire layer, eventually) handles the tree surgery — replacing the degenerate split with its sole surviving child. This mirrors i3's automatic collapse and makes the undo path for splits natural: close children until one remains, the split dissolves.

One scoping issue: `buildOrganiser` lives at module scope in `container.ts`, so it can't see `config.onCollapse`. Had to thread `onCollapse` as a parameter. Clean enough — the alternative (moving `buildOrganiser` inside `createContainer`) would close over too much state.

Twenty-three new tests. Batch 2 complete. Batch 3 (production migration) is next — the 2100-line backend file.
