---
title: "When the Gallery Met the Server"
date: 2026-11-04
author: mdp
entry_type: note
subtype: diary
series: issue-605-json-apps-polish
projects: [casehubio/casehub-pages]
tags: [gallery, json-apps, xss, shadow-dom, web-components, demo-server]
---

# When the Gallery Met the Server

The examples gallery has always been client-side. 111 samples rendered from static YAML files, datasets resolved against mock Prometheus data or inline JSON. Good for showcasing layout, charts, and interactivity. Useless for demonstrating the thing that actually matters: an application backed by real data, with real CRUD, served from a real backend.

The JSON Apps runtime shipped last week — `AppRegistry`, `AppBootstrap`, `PageResource`, collection-scoped REST endpoints, lifecycle gating. Four apps deployed to the demo server: helpdesk (12 tickets), contacts (8 entries), tasks (8 items), and a platform app that catalogues them. All running on port 8090 with SQLite persistence. The gallery couldn't touch any of it.

## The split that made it work

The obvious approach was to use `loadSite`'s existing `appId` option. Pass `{ appId: 'helpdesk', baseUrl: 'http://localhost:8090' }` and let `DynamicPageLoader` fetch the page tree from `PageResource`. The plumbing already existed — `PageApiClient`, `DynamicPageLoader`, the whole chain from `SiteOptions.appId` through to dynamic page injection.

It returned 401 Unauthorized.

The demo server, it turned out, excludes `PageResource` entirely:

```properties
quarkus.arc.exclude-types=io.casehub.pages.data.DataResource,...,io.casehub.pages.data.json.PageResource
```

The auth-gated resources are absent from the demo runtime. `DemoCollectionResource` handles collection data at `/api/apps/{appId}/{collection}` without auth — that's how the gallery discovers apps from `_apps`. But `PageResource` at `/api/apps/{appId}/pages` doesn't exist, so `PageApiClient.list()` hits a wall.

The fix split the problem in two. Page YAML comes from the client-side samples (the gallery already has `samples/Domain Showcases/helpdesk/helpdesk.page.yaml`). Data comes from the backend via `baseUrl`. The gallery fetches the YAML locally, then calls `loadSite(target, yamlText, { baseUrl: SERVER_URL, fetch: corsWrapper })`. The datasets in the YAML declare URLs like `/api/apps/helpdesk/tickets` — with `baseUrl` pointing at the backend, those resolve to real REST endpoints. Structure from samples, data from the server.

![Gallery data flow — YAML from samples, data from backend](images/gallery-server-data-flow.svg)

## The shadow DOM boundary

The platform app's catalogue shows a table of all registered apps. I wanted double-clicking a row to navigate the gallery to that app — click "helpdesk" in the catalogue, see the helpdesk dashboard. The obvious approach was to listen for `selection-change` on the sample target div.

Nothing fired.

`PagesDataTable` dispatches `selection-change` inside its shadow root. The event doesn't have `composed: true`, so it never crosses the shadow boundary. A listener on any ancestor element receives nothing — the event is invisible outside the component.

The fix uses `dblclick`, which is a native DOM event with `composed: true` by default. But `dblclick`'s `e.target` gets retargeted at the shadow boundary — the event target becomes the shadow host (`pages-data-table`), not the internal `<tr>` element. So `e.target.closest('[role="row"]')` returns null.

`composedPath()` solves this. It returns the full event path including elements inside shadow roots:

```javascript
sampleTarget.addEventListener('dblclick', (e) => {
    const path = e.composedPath();
    const row = path.find(el => el.getAttribute?.('role') === 'row');
    if (!row) return;
    const cells = row.querySelectorAll('[role="gridcell"]');
    const appId = cells[0]?.textContent?.trim();
    // ... navigate to app
});
```

This pattern — `composedPath()` to traverse shadow boundaries in bubbling events — is worth knowing for any web component system. Lit, Stencil, native custom elements — whenever you need to inspect the actual click target from outside a shadow root, `composedPath()` is the tool.

## Per-variable XSS escape

The template parser resolves `#{params.name}` by looking up `params.name` in the runtime context. URL query parameters are user input. Components like `html` and `markdown` already escape via their escape mode (`"html"` and `"markdown"` respectively). But `title` and `alert` use `"none"` — title because it sets `textContent` (inherently safe), alert because Lit auto-escapes.

The problem is host panels. They also use `"none"`, and they receive resolved values via `configure()`. If a panel renders via `innerHTML`, a `#{params.x}` with `<script>alert('xss')</script>` passes through unescaped.

The fix is per-variable, not per-component. When escape mode is `"none"` and the template path starts with `params.`, upgrade to `"html"` for that specific variable:

```javascript
const effectiveEscape = (escape === "none" && trimmed.startsWith("params."))
    ? "html" : escape;
```

A template like `Hello #{params.name}, id: #{datasets.users.first.id}` escapes `params.name` (user input) but leaves `datasets.users.first.id` raw (trusted pipeline data). Per-component escape would force all values through HTML escaping — including dataset values that might contain intentional HTML. Per-variable targets the actual threat surface.

## What the gallery shows now

Nine samples on the Server tab — five push-protocol demos plus four discovered apps. The helpdesk renders with 9 open tickets, pie charts by category, bar charts by priority. Contacts shows 8 entries with company, category, and status labels. The platform catalogue lists all four apps and navigating into any of them replaces the render target with that app's full dashboard.

The gallery went from "here's what the YAML looks like" to "here's the application running." That shift — from describing capability to demonstrating it — is the difference between documentation and evidence.
