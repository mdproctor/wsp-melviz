---
layout: post
title: "From Grid Editor to App Builder: One Model, Two Modes"
date: 2026-11-08
entry_type: article
subtype: technical
projects: [casehubio/casehub-pages]
tags: [grid, drag-and-drop, ux, css-grid, compaction, layout-model, app-builder, poc]
series: issue-615-dnd-insert-yaml-roundtrip
---

# From Grid Editor to App Builder: One Model, Two Modes

*Follows [When Custom Stops Being an Advantage](/docs/blog/2026-11-06-mdp01-when-custom-stops-being-an-advantage.md). What started as a grid drag-and-drop fix became the foundation for an app builder.*

## The Problem With Two Layout Systems

CaseHub Pages had two ways to lay out content. The `rows`/`columns` system was declarative — you listed components with column spans and they flowed top-to-bottom, left-to-right. Responsive, simple, but no spanning. A chart couldn't stretch across two column containers.

The `items` system used explicit grid coordinates — `{ x: 0, y: 0, w: 6, h: 2 }` on a 12-column CSS Grid. Full control, arbitrary spanning, but no reflow.

Two vocabularies for the same runtime. The parser converted `rows`/`columns` into positioned `items` at parse time — at runtime, there was only CSS Grid. Two input formats, one output, mutual exclusion at page level, and LLMs generating YAML had to learn both.

![Unified layout model](images/unified-layout-model.svg)

The POC work on grid editing surfaced a cleaner answer: just `items`. Auto-flow items (`{ w: 6 }` — no coordinates) reflow like `rows`/`columns`. Positioned items (`{ x: 0, y: 0, w: 6, h: 2 }`) pin to coordinates. Both live in the same array. The presence of `x`/`y` determines the mode — no discriminator field needed. One vocabulary, strictly more expressive than either predecessor.

## Building the Interaction

The POC is a standalone HTML file — no framework, no YAML, no build step. A 12-column CSS Grid with coloured panels, pointer events, and the compaction algorithm in JavaScript. Every interaction is visible in the browser within seconds of writing it.

The compaction algorithm is one rule: fix the moved item at its new position, then place every other item at the lowest row where it fits without overlapping anything already placed. One rule handles move, resize, expand, shrink, and delete. 15 lines of code, 40 unit tests, round-trip stability (expand then shrink → original layout restored exactly).

But drag-and-drop alone isn't enough. Two iterations shaped the interaction model:

**Dual drag handles, not a popup.** The first design showed a popup on collision — "displace, split, replace, or swap?" Five options interrupted the drag flow. Internet research showed that Grafana, Gridstack, and React-Grid-Layout all use displacement-only, Figma uses modifier keys, and Visual Studio uses directional drop zones. The winning UX: two drag icons on each panel's title bar. ⠿ for displace (push down), ⇄ for swap (exchange positions). Hold Option during swap to replace (destructive). Intent declared before the drag, no interruption after.

**Row breaks as gravity boundaries.** The grid needed logical sections. Research showed Grafana uses full-width collapsible row headers. The POC implementation: a pencil-thin divider (4px) that expands on hover, showing a title and collapse toggle. Structurally, a break is an immovable full-width grid item that the compactor treats as a boundary.

![Gravity boundaries](images/gravity-boundaries.svg)

The critical invariant: **gravity never crosses a break.** Panels compact within their section only. If a displaced panel can't fit in its section, the break shifts down to make room — the section grows rather than the panel crossing the boundary. This required rewriting the compactor from first principles, with pre-assigned section membership and a while-loop that doesn't skip positions after pushing a break.

Dragging the break itself uses magnetic snap — a cursor line tracks the mouse freely, and when it gets close to a row boundary (40px threshold), the divider snaps into place showing a live displacement preview. Move away and the preview reverts. Release while snapped to commit; release while unsnapped to cancel. No thrashing, because `snapToCell` uses frozen row positions captured at drag start — the DOM changes from compaction don't feed back into the snap calculation.

## What the POC Proves

Eight capabilities validated in Playwright, not assumed:

1. **Displacement compaction** with vertical gravity and float modes
2. **Dual drag handles** — displace (⠿), swap (⇄), replace (⌥+⇄)
3. **Row breaks** as gravity boundaries with collapse, magnetic snap drag
4. **Expand-to-fill** — panel fills available space horizontally and vertically
5. **Sticky panels** — immovable, others reflow around them
6. **Undo/redo** — state stack with Ctrl+Z/Ctrl+Shift+Z, 50 levels deep
7. **DOM-based snap-to-cell** — reads actual row positions, handles variable-height rows
8. **Sizing modes** — proportional, fixed width, fixed height, content height — verified at 800px, default, and 1600px viewports

## Two Modes: Structure Then Content

The POC work led to a design decision that changes the scope of the project.

![Two-mode editor](images/two-mode-editor.svg)

**Layout mode** designs the grid structure. Split cells, merge cells, add row breaks, resize proportions. No drag-and-drop — the interactions are structural (click a boundary to split, select two cells to merge). The output is a skeleton of empty containers.

**Content mode** fills the containers. Drag components from a palette into empty cells. Swap, replace, rearrange. The containers are set; you're placing content into them.

An empty container is a first-class item in the unified model — `{ w: 6, h: 2 }` with no `component` key. The YAML shows the layout skeleton before any components are added. This is already implicit in the POC's two-stage delete, where removing a panel leaves an amber placeholder holding space. The design makes it intentional.

## App Builder, Not Just Grid Editor

The unified model with two-mode editing is the foundation for more than dashboards. A form is a page where the components are inputs instead of charts:

```yaml
# Dashboard
items:
  - { w: 6, h: 2, component: { type: chart, metric: revenue } }
  - { w: 6, h: 2, component: { type: table, query: orders } }

# Form — same model, different components
items:
  - { w: 6, component: { type: text-input, label: "First name" } }
  - { w: 6, component: { type: text-input, label: "Last name" } }
  - { w: 12, component: { type: textarea, label: "Notes" } }
```

Same containers, same split/merge, same drag-to-fill. The grid editor is the page builder is the form builder. The distinction is the component palette, not the layout engine.

Nested sub-grids extend this further. A grid item whose component has its own `items` array creates a grid-within-a-grid — what `rows`/`columns` nesting did, but using the same vocabulary at every level.

## What's Next

The POC is a 1500-line HTML file that proves the interaction model. Thirteen issues across four phases (epic #625) take it to production:

- **Phase 1** — layout mode with split, merge, and cell creation
- **Phase 2** — empty containers as drop zones with a component palette
- **Phase 3** — unified YAML output and nested sub-grids
- **Phase 4** — framework integration, wiring to the PageDocument model with YAML CST mutation and compound undo/redo

The POC stays as a living reference. Each phase is proved in the POC before it's built in the framework. Because this is now the foundation — and foundations are where guessing is most expensive.a
