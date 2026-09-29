# Decisions — feat/506-unified-step-catalog

## D1: Catalog merge point — client-side in the component

**Choice:** Client-side merge in `<pages-action-catalog>`. The component accepts multiple catalog sources (direct PluginRegistry refs, REST endpoints, GraphQL endpoints), fetches from each, and merges in the browser.
**Alternatives:**
- Server-side CatalogSource — add a GraphqlCatalogSource to CompositeCatalog. Single fetch, but hardcodes full-stack assumption and can't work standalone.
- TS REST proxies Java — TS endpoint calls Java GraphQL internally. One fetch but tight server coupling.
**Rationale:** Must support standalone browser mode (no backend). Client-side merge naturally degrades: standalone gets only TS registry, TS-backend adds REST source, full-stack adds Java GraphQL source.
**Trade-offs:** Multiple network calls in full-stack mode instead of one. Merge logic duplicated between CompositeCatalog (server-side execution) and the component (browser display).
**Sources:** packages/yaml-core/src/step/catalog.ts (CompositeCatalog), packages/pages-aria/src/controller/step-catalog.ts (PagesActionCatalog), casehubio/platform#483 (Java API contract)
**Exploration:** quick
**Status:** captured
