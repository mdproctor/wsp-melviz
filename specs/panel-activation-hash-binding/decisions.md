# Panel Activation & Hash Binding — Decisions

## D1: API surface for activateDockPanel

**Choice:** Method on `LiveSite` interface returned by `loadSite()`
**Alternatives:**
- Standalone function exported from pages-runtime — requires module-level mutable state to bridge loadSite closure; navigate() is NOT standalone (it's on LiveSite), so no pattern to match
- Method on ZoneLayoutEngine — couples topology engine to DOM, breaks its pure-data nature
- New WorkbenchController wrapper — extra abstraction layer for thin orchestration
**Rationale:** Follows existing pattern — `navigate()`, `setTheme()`, `dispose()` are all `LiveSite` methods because they need `loadSite()` closure state. `activateDockPanel` needs the same: `target` (root element), `dockState` (Map), `zoneEngine`, `syncUrl()`, `scheduleLayoutSave()`. A closure method has direct access without module-level state.
**Trade-offs:** Consumers must hold a reference to the LiveSite object. This is already the case for navigate().
**Sources:** packages/pages-runtime/src/site.ts (LiveSite interface, loadSite closure), PP-20260810-72779a (workbench integration pattern)
**Exploration:** quick
**Status:** revised (R1-01: navigate() is on LiveSite, not standalone)

## D2: Hash composition model

**Choice:** Orthogonal `panel` param in DeepLink — `activePanel` field serialized as `?panel=backlog`
**Alternatives:**
- Panel-as-path (`#backlog` maps directly) — panel keys and page paths share namespace, collision risk
- Declarative config bindings — adds config surface for what is a generic URL concern
**Rationale:** Composes cleanly with existing page navigation. `#/page/dashboard?panel=backlog` navigates AND activates. No collision with page paths.
**Trade-offs:** Slightly more verbose than bare `#backlog`. Panel-only deep links (`#?panel=backlog`) require extending `parseFromUrl` to handle hashes without the `#/page/` prefix — currently the parser gates on that prefix and ignores everything else. Small change: fallback to empty page path when no `#/page/` prefix but query params are present.
**Sources:** packages/pages-runtime/src/url.ts (DeepLink, serializeToUrl, parseFromUrl — line 79-80 gates on #/page/), GE-20260625-fa01da (pushState/popstate separation)
**Exploration:** quick
**Status:** revised (R1-02: parseFromUrl requires #/page/ prefix — panel-only links need parser extension)

## D3: Multi-panel activation

**Choice:** Comma-separated list in the `panel` param — `?panel=backlog,properties`
**Alternatives:**
- Single panel only — simpler but deep links can't set up full workbench state across zones
- Zone-qualified format (`?panel=left-top:backlog,right-top:properties`) — eliminates zone ambiguity but more verbose
**Rationale:** A deep link often needs to open panels in multiple zones simultaneously (e.g., nav panel on left + properties on right). `activateDockPanel(key)` stays single-panel; the URL param is the composition point. Zone assignment comes from the engine's zoneMap, not the URL.
**Trade-offs:** If two keys are in the same exclusive zone, last wins (sequential processing, left-to-right). This is inherent to exclusive dock bar semantics — not a bug, but callers should be aware. Processing order: left-to-right, guaranteed.
**Sources:** packages/pages-component/src/model/types.ts (DockZone — 6 zones, panels can be in different zones)
**Exploration:** quick
**Status:** revised (R1-03: zone-exclusive conflicts acknowledged)

## D4: panel= and dock= URL interaction

**Choice:** Parse-time directive — `parseFromUrl` converts `panel=backlog` into dock activation intents
**Alternatives:**
- panel replaces dock entirely — breaks existing deep links, loses fine-grained state
- Separate state tracks — panel= is one-time intent, dock= is persisted state — complex dual management
**Rationale:** Minimal change to existing dock state machinery. On load/popstate, panel entries produce `activateDockPanel` calls. URL stays as authored. panel entries override dock entries for the same key. After activation, subsequent dock toggles update dock= normally.
**Trade-offs:** panel= and dock= can be redundant for the same key. panel= always wins on conflict. Semantic distinction is intentional: panel= is additive activation intent (works even if new panels are added to the workbench), dock= is total state declaration (fragile to panel changes).
**Pre-requisite:** Existing bug — `restoreFromUrl` in site.ts updates the `dockState` map on popstate but never dispatches `pages-dock-toggle` events, so dock panels don't actually toggle on back/forward navigation. This must be fixed for panel= (and existing dock=) to work correctly on popstate. Fix: dispatch `pages-dock-toggle` for each changed dock entry during `restoreFromUrl`.
**Sources:** packages/pages-runtime/src/url.ts (parseFromUrl, serializeToUrl), packages/pages-runtime/src/site.ts (popstate handler line 1343, restoreFromUrl line 349-352 — sets dockState map but no DOM dispatch)
**Exploration:** quick
**Depends on:** D2 (hash composition model), D3 (multi-panel)
**Status:** revised (R1-04: popstate dock bug identified as pre-requisite; R1-05: panel= vs dock= distinction justified)

## D5: Naming — activateDockPanel, not activatePanel

**Choice:** `activateDockPanel(key)` — dock-panel-specific name
**Alternatives:**
- `activatePanel(key)` — suggests generality across tabs, accordions, carousels. The platform already has `activateSlot` for interactive containers and `pages-slot-change` for tab/pill navigation. Generic naming creates scope confusion.
- `toggleDock(key)` — accurate for the toggling behavior but doesn't convey "show this panel" intent
**Rationale:** This API is specifically for dock panel activation (exclusive zone mode, deferred rendering, dock-bar state sync). Non-dock interactive containers already have their own activation paths. Clear naming prevents scope creep.
**Trade-offs:** More verbose. If a future unified activation API is wanted, rename is needed.
**Sources:** packages/pages-runtime/src/site.ts (activateSlot), PP-20260705-bac842 (pages-event contract — pages-dock-toggle is dock-specific)
**Exploration:** quick
**Status:** captured (from R1-06)
