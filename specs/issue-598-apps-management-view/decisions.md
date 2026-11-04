## D1: Lifecycle scope

**Choice:** Dynamic apps only — static classpath-bundled apps are always running
**Alternatives:**
- All apps stoppable — more powerful but adds complexity for infrastructure apps
- No real lifecycle — defer to follow-on issue, but issue explicitly calls for it
**Rationale:** Static apps are infrastructure; users shouldn't be able to break the platform by stopping them. Dynamic apps are user-installed and user-controlled.
**Trade-offs:** Cannot stop a misbehaving static app without redeployment
**Sources:** AppRegistry.java:226-246 (classpath scanning), AppBootstrap.java (static seeding)
**Exploration:** quick
**Status:** captured

## D2: Settings storage

**Choice:** Settings as fields on _apps collection documents — enabled, displayName, autoSeed, maxDocuments
**Alternatives:**
- New PreferenceStore REST endpoint — proper separation but adds platform-api dependency, won't work standalone
- Hybrid (core in _apps, advanced in PreferenceStore) — two sources of truth
**Rationale:** Self-contained in the existing data-json module. One REST call for all app data. Schema-form can render settings directly from the _apps document schema.
**Trade-offs:** Settings are coupled to app metadata; if PreferenceStore gains a REST endpoint later, migration needed
**Sources:** AppRegistry.java:54-67 (APPS_SCHEMA), PreferenceStore docs in design specs
**Exploration:** quick
**Status:** captured

## D3: Stop action semantics

**Choice:** Flag + cache eviction — set enabled=false, gate collection endpoints to 404, evict DynamicPageLoader and DataCacheService caches
**Alternatives:**
- Unregister from AppRegistry — cleaner isolation but requires disabled-descriptors stash and re-parse on start
- Flag-only gating — simplest but leaves stale caches causing broken-page UX
**Rationale:** Flag gating is minimal and reversible. Cache eviction prevents the stale-page-with-404-data UX problem and uses existing machinery (DynamicPageLoader.invalidateAll(), DataCacheService.invalidate()).
**Trade-offs:** Cache eviction is coarse-grained (invalidateAll vs per-app); may evict more than necessary
**Sources:** DynamicPageLoader.ts:41-48 (invalidate methods), AppCatalogueResource.java:137 (cache invalidation pattern)
**Exploration:** quick
**Status:** captured

## D4: Page structure

**Choice:** Single apps-management.page.yaml with navTree tabs (list, settings, optionally install)
**Alternatives:**
- Separate pages — more modular but breaks from established Domain Showcase pattern
**Rationale:** Follows the existing multi-page pattern exactly (helpdesk.page.yaml, contacts.page.yaml). NavTree provides tab navigation. Detail/settings page uses dataScope binding from selected table row.
**Trade-offs:** Single file grows with complexity; but existing examples handle 400+ lines fine
**Sources:** platform.page.yaml (100 lines), helpdesk.page.yaml (447 lines)
**Exploration:** quick
**Status:** captured

## D5: API shape

**Choice:** New AppManagementResource at GET /api/apps and POST /api/apps/{appId}/lifecycle
**Alternatives:**
- Extend AppCatalogueResource — less code but muddies child-app vs top-level distinction
**Rationale:** AppCatalogueResource is scoped to /api/apps/{appId}/apps (child apps within a parent). Top-level app listing is a different concern — all registered apps with runtime status.
**Trade-offs:** One more JAX-RS resource class to maintain
**Sources:** AppCatalogueResource.java:35 (path scoping)
**Exploration:** quick
**Status:** captured
