---
title: "The Lit Migration"
date: 2026-07-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [lit, web-components, refactoring, pages-viz]
---

# casehub-pages — The Lit Migration

**Date:** 2026-07-20
**Type:** phase-update

---

## What I was trying to achieve: one rendering model across all viz components

pages-table already uses Lit. pages-primitives uses Lit. But pages-viz — the package that holds every chart, metric, selector, badge, countdown, form input, and iframe wrapper — was still vanilla HTMLElement with manual shadow DOM and imperative `document.createElement` chains. Two rendering models in the same platform. Every time we tried to compose across the boundary (grouped-view delegating to pages-table, for instance), the mismatch forced workarounds.

Issue #188 (grouped-view composing pages-table) made the cost concrete. We got it working, but the spec explicitly deferred PagesElement migration as "separate concern, larger blast radius." This branch picks that up.

## What we believed going in: the base classes are the hard part, subclasses are mechanical

The hierarchy is four base classes (PagesElement, PagesChartElement, PagesFormInput, PagesContentElement) and 26 concrete subclasses. I expected the base class lifecycle translation — `connectedCallback` timing, `willUpdate` interactions, the `cache()` directive for chart DOM preservation — to be the real work. The subclasses, I thought, would be straightforward search-and-replace.

That was half right.

## The lifecycle traps

Two things bit us that weren't in any Lit docs I could find.

**The double-fire.** When you set a `@property` before `appendChild`, Lit queues an update. On connection, `connectedCallback` fires, then Lit processes the queued update and calls `willUpdate`. Both see the same props. Both want to fire a data request. The `_dataRequested` guard should prevent the duplicate — except `willUpdate` detects a "change" (undefined → value) and resets the guard. Fix: only reset when transitioning between two *defined* prop objects, not from undefined to defined.

**The async gap.** Our charts have async `buildOption()` — it returns a Promise. PagesChartElement calls it from `updated()` and uses `.then(apply)` to set the ECharts option. In tests, `await el.updateComplete` resolves *before* the Promise chain fires. `await Promise.resolve()` isn't enough either — the chain has multiple microtask levels. The fix: `await new Promise(r => { setTimeout(r, 0); })` to force a macrotask boundary, then `await el.updateComplete` again. Every chart test needs this pattern.

Both of these are now garden entries. They'll save someone hours.

## The migration itself

64 files changed. 4,135 insertions, 3,523 deletions — a net reduction despite adding Lit imports and decorator boilerplate. The imperative DOM construction was that verbose.

PagesContentElement (the simpler root) went first to validate the Lit setup end-to-end. Then PagesElement — the core with DataSourceController composition, refresh timer, resize observer, and the render dispatch state machine. The `cache()` directive turned out to be essential: without it, every loading→content transition destroys the chart div, leaving ECharts bound to detached DOM.

The 11 chart subclasses barely changed — they implement `buildOption()` returning option objects, not DOM. Add `@customElement`, remove `customElements.define`, done. PagesMap was the exception: it had a custom `render()` override for async map loading. That logic folded cleanly into `buildOption()` itself.

The form inputs and data components converted imperative DOM chains to `html` tagged templates. PagesGroupedView — 783 lines of entangled imperative DOM management — dropped to 629. The `_groupTables` Map, `_forwardToTables()` setter calls, and the entire imperative table lifecycle collapsed into Lit template bindings. Child `<pages-table>` elements rendered with `.dataSet=${groupData}` instead of manual property forwarding.

## What this enables

pages-viz and pages-table now share a rendering model. Composition between them is template-level, not ceremony-level. The a11y mixins from pages-primitives (RovingTabindexMixin, FocusTrapMixin) can now be applied to any viz component — that's the next step.

The activation.ts legend fix was a bonus: the runtime was constructing legend DOM imperatively instead of using `<pages-legend>`. Five lines replaced thirty.
