---
layout: post
title: "Giving the Workbench a Public API"
date: 2026-08-27
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [workbench, dock-panel, deep-linking, api]
---

# Giving the Workbench a Public API

The dock workbench has been rendering panels, toggling them on click, and persisting their state for a while now. What it didn't have was a way for consumers to say "show the backlog panel" from code, or a URL that means "open this page with these panels visible." Trellis needs both.

## The exclusivity problem I didn't know I had

I started with what seemed like the straightforward part: add an `activateDockPanel(key)` method to `LiveSite`, dispatch `pages-dock-toggle`, done. But the design review caught something I'd missed — zone-group exclusivity (only one panel visible per zone in an exclusive dock bar) was enforced in the *click handler*, not in the event handler. Any code dispatching `pages-dock-toggle` directly — and there were already several call sites doing this — bypassed exclusivity entirely. Both panels would show simultaneously, and the dock-bar buttons would diverge from actual panel state.

The fix was to centralise everything in the `pages-dock-toggle` handler in `site.ts`: exclusivity enforcement, button `data-active` sync, and cascade expand/collapse. The click handler became a one-liner — dispatch the event and nothing else. Same for `initDockZoneGroup` and the rearrange handler.

One subtlety: the `exclusive` prop is on the dock-bar component, not the zone. I needed a way for the centralised handler to know whether the bar is exclusive without importing the component's props. The answer was `data-exclusive` on the dock-bar DOM element — set at render time, checked by the handler via `btn.closest('[data-component-type="dock-bar"]')`. Clean DOM-contract communication, no coupling.

## activateDockPanel

With exclusivity centralised, `activateDockPanel` was simple:

```typescript
function activateDockPanel(key: string): boolean {
  const panelEl = target.querySelector(`[data-component-id="${CSS.escape(key)}"]`);
  if (!panelEl) return false;
  target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
    bubbles: true, composed: true,
    detail: { panelId: key, visible: true },
  }));
  return true;
}
```

DOM lookup instead of `zoneEngine.zoneMap` — so it works for both `dockWorkbench()` compositions and manual dock-bar setups where no zone engine exists. Returns `false` for unknown keys without throwing.

## Hash panel binding

The second feature extends `DeepLink` with a `panel` field: `#/page/dashboard?panel=backlog,properties`. The interesting design choice was the relationship between `panel=` and the existing `dock=` param.

`dock=` is total state declaration — every panel's open/closed status. It's what `syncUrl` produces internally. `panel=` is activation intent — "show these panels." It's additive: a deep link with `?panel=backlog` works even if new panels are added to the workbench later. `dock=nav:open,files:closed,props:closed` breaks the moment someone adds a fourth panel.

When both appear for the same key, `panel=` wins. The merge happens inside `restoreFromUrl` before dispatching events, so there's no redundant hide-then-show churn.

I also added support for panel-only deep links: `#?panel=backlog` (no page path). This required extending `parseFromUrl` to handle the `#?` prefix and updating `serializeToUrl` to produce `#?` when the page is empty.

## The popstate bug

While wiring the hash binding, I found that `restoreFromUrl` was updating the `dockState` map on popstate but never dispatching `pages-dock-toggle` events. Dock panels didn't actually show or hide on browser back/forward. This was a pre-existing bug — fixed it as a prerequisite before adding the panel merge.

## What this opens up

Trellis can now wire keyboard shortcuts to `site.activateDockPanel("terminal")` and share deep links like `#/page/workspace?panel=terminal,agent`. The API is small — one method and one URL param — but it's the minimum a workbench consumer needs for programmatic control without reaching into DOM internals.
