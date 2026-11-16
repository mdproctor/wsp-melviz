---
layout: post
title: "From fighting dockview to composable frames"
date: 2026-08-15
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [floating-workspace, architecture, group-organiser, dockview, accordion]
series: issue-312-workspace-compositor
---

Continues from [workspace compositor — splits, tabs, and accordion view](2026-08-13-mp01-workspace-compositor.md).

The frames and tabs work. They've worked since the auto-wire epic landed. Drag, resize, close, pin, detach, snap — all solid. The problem was the accordion toggle. Switching a frame between tab view and accordion view had been unstable for a full day, and every fix created a new edge case.

The root cause wasn't a bug. It was architecture.

Dockview conflates "floating positioned container" with "tab group." You can't have the frame without the tab strip, and you can't put anything else inside it. So the accordion toggle required hiding dockview's rendering, building a parallel accordion in a sibling div, and reconciling state between two layout systems that each believed they owned the DOM. Every fix was a ceasefire, not a resolution — dockview's `setBounds()` clamps to the container's `getBoundingClientRect()`, so during async nested workspace creation (when the container is zero-sized), all positions get clamped to 0,0. Its `toJSON()` returns only the active anchor's properties, so `bounds.left` can be undefined. The `floatingGroups` internal array uses `fg.group` in some versions and `fg._group` in others. Each workaround uncovered the next assumption.

I stepped back and asked a different question: what if the frame and the interior were separate concerns?

A frame is a positioned, draggable, resizable rectangle. What's inside it — tabs, accordion sections, freely placed panels — is a configuration choice, not a structural constraint. The frame doesn't care which arrangement its content uses. The arrangement doesn't care where the frame is positioned.

This led to a model we're calling Group/Organiser. A Group is a universal container of entries. An Organiser is a pluggable physics strategy — Tab, Accordion, or FreeLayout — that determines how entries are arranged inside their container. Toggle between modes is: unmount the old organiser (detaching content elements from their slots), create the new organiser, mount it (reattaching the same content elements into new slots). Content never knows the toggle happened. Scroll positions, nested workspace state, event listeners — all preserved.

The key insight was about ownership. The previous approach destroyed and recreated content on every toggle because the layout containers (tab strip vs accordion headers vs positioned frames) are structurally different DOM trees. You can't CSS your way between them. But the content elements inside those containers don't need to change. They just need to be moved. The organiser owns its layout DOM — tab strip, accordion headers, resize handles. The entry owns its content element. These are independent lifecycles. The organiser is ephemeral; the content is persistent.

Rather than modifying the working code in place, we built a standalone sandbox — six new files in `packages/pages-runtime/src/frame-sandbox/`, zero imports from the existing frame system. The sandbox proves the architecture with real component rendering, not test stubs. Three organisers (Tab, Accordion, FreeLayout), a Group that toggles between them, recursive nesting three levels deep. Thirty-seven unit tests verify the data model and content preservation. Six Playwright tests verify drag, resize, tab switching, accordion expand/collapse, and nesting in a real browser.

The full toggle matrix — tab to accordion, accordion to free-layout, free-layout to tab, and back — passes with the same DOM elements throughout. The test literally checks identity: `expect(afterToggle).toBe(original)`. That's the proof the architecture holds.

Next is the migration: replace `dockview-backend.ts` with a new `FloatingFrameBackend` implementation that uses the sandbox primitives internally. The existing `FloatingFrameEngine`, all the feature modules (z-order, spatial nav, keyboard, detach, snap, animations), and the wire layer all program to the `FloatingFrameBackend` interface — only the implementation behind it changes. Six tasks in the migration plan, the hardest being the new backend itself and the activation swap.

The frames and tabs that work today will continue to work. The accordion toggle that doesn't will be replaced by something that can't have the same class of bugs, because there's no longer a seam between two competing layout systems.
