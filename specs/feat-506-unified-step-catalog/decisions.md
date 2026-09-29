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

## D2: Portability assignment — explicit with smart defaults

**Choice:** `portability` is an explicit optional field on `Definition`. Defaults: `'ts'` for `PluginRegistry.register()` calls, `'universal'` for protocol-based invoke bindings (REST, GraphQL, process). YAML definitions can set it explicitly via a `portability:` key.
**Alternatives:**
- Fully automatic from invoke binding — zero author burden but inflexible for edge cases (e.g., an MCP tool that proxies a Java service is effectively universal)
- Explicit required, no defaults — maximum clarity but unnecessary boilerplate for the 90% case
**Rationale:** Smart defaults match the common case (programmatic TS plugins are TS-only, protocol bindings are runtime-agnostic) while allowing explicit override for edge cases.
**Trade-offs:** Authors of unusual configurations must know to set portability explicitly.
**Sources:** casehubio/platform#483 (portability model spec), packages/yaml-core/src/step/types.ts (Definition interface)
**Exploration:** quick
**Status:** captured

## D3: Portability rendering — colored chips

**Choice:** Colored chip badge next to each action name in the catalog browser. universal=green, java=orange, ts=blue, both=purple. Consistent with existing source filter chips.
**Alternatives:**
- Icon only — more compact but less discoverable for new users
**Rationale:** Text chips are self-documenting and match the existing UI pattern for source filter chips.
**Trade-offs:** Slightly more horizontal space per row.
**Sources:** packages/pages-aria/src/controller/step-catalog.ts (existing chip styling)
**Exploration:** quick
**Depends on:** D2 (portability values determine chip labels)
**Status:** captured
