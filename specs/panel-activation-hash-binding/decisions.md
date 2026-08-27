# Panel Activation & Hash Binding — Decisions

## D1: API surface for activatePanel

**Choice:** Standalone function exported from `@casehubio/pages-runtime`
**Alternatives:**
- Method on ZoneLayoutEngine — couples topology engine to DOM, breaks its pure-data nature
- New WorkbenchController wrapper — extra abstraction layer for thin orchestration
**Rationale:** Matches existing patterns (navigate() is also a standalone function). Engine stays pure data. The function uses the engine's zoneMap for topology awareness and dispatches `pages-dock-toggle` for DOM work — reusing the existing event-driven activation path in site.ts.
**Trade-offs:** Module-level state dependency (function needs access to the site's root element, stored at loadSite time). Cannot be used before loadSite completes.
**Sources:** packages/pages-runtime/src/zone-layout-engine.ts, packages/pages-runtime/src/site.ts, PP-20260810-72779a (workbench integration pattern)
**Exploration:** quick
**Status:** captured

## D2: Hash composition model

**Choice:** Orthogonal `panel` param in DeepLink — `activePanel` field serialized as `?panel=backlog`
**Alternatives:**
- Panel-as-path (`#backlog` maps directly) — panel keys and page paths share namespace, collision risk
- Declarative config bindings — adds config surface for what is a generic URL concern
**Rationale:** Composes cleanly with existing page navigation. `#/page/dashboard?panel=backlog` navigates AND activates. Panel-only deep links: `#?panel=backlog`. No collision with page paths.
**Trade-offs:** Slightly more verbose than bare `#backlog`. Requires consumers to construct query-param-style URLs.
**Sources:** packages/pages-runtime/src/url.ts (DeepLink, serializeToUrl, parseFromUrl), GE-20260625-fa01da (pushState/popstate separation)
**Exploration:** quick
**Status:** captured

## D3: Multi-panel activation

**Choice:** Comma-separated list in the `panel` param — `?panel=backlog,properties`
**Alternatives:**
- Single panel only — simpler but deep links can't set up full workbench state across zones
**Rationale:** A deep link often needs to open panels in multiple zones simultaneously (e.g., nav panel on left + properties on right). `activatePanel(key)` stays single-panel; the URL param is the composition point. Minimal extra code — just split on comma and call activatePanel for each.
**Trade-offs:** None significant. Parsing a comma-separated list is trivial.
**Sources:** packages/pages-component/src/model/types.ts (DockZone — 6 zones, panels can be in different zones)
**Exploration:** quick
**Status:** captured

## D4: panel= and dock= URL interaction

**Choice:** Parse-time directive — `parseFromUrl` converts `panel=backlog` into dock activation intents
**Alternatives:**
- panel replaces dock entirely — breaks existing deep links, loses fine-grained state
- Separate state tracks — panel= is one-time intent, dock= is persisted state — complex dual management
**Rationale:** Minimal change to existing dock state machinery. On load/popstate, panel entries produce activatePanel calls. URL stays as authored. panel entries override dock entries for the same key. After activation, subsequent dock toggles update dock= normally.
**Trade-offs:** panel= and dock= can be redundant for the same key. panel= always wins on conflict.
**Sources:** packages/pages-runtime/src/url.ts (parseFromUrl, serializeToUrl), packages/pages-runtime/src/site.ts (popstate handler, dock state management)
**Exploration:** quick
**Depends on:** D2 (hash composition model), D3 (multi-panel)
**Status:** captured
