## D1: Label storage architecture

**Choice:** Consume platform LabelRegistry SPI (platform #694) — no Pages-local storage
**Alternatives:**
- Pages-local SQLite LabelTreeStore — keeps JSON apps self-contained but duplicates platform concept
- Labels as JSON collection — reuses existing data-json but blurs structural metadata vs user data
**Rationale:** Labels are a platform concept, not a Pages concept. The platform has consumption infrastructure (Path, LabelPatternMatcher, LabelRule, SubjectViewSpec) but lacked definition infrastructure. Platform #694 adds LabelRegistry SPI following EndpointRegistry/DataSourceRegistry patterns. Labels are tenant-scoped (not app-scoped) — apps register under PLATFORM_TENANT_ID, tenant extensions merge with tenant-wins precedence. App-level scoping uses root paths (consumer concern).
**Trade-offs:** Pages #600 depends on platform #694 landing first. Pages cannot run label trees without the platform dependency.
**Sources:** platform-api Path.java, LabelPatternMatcher.java, SubjectViewSpec.java; platform #694 design; connectors-project-spi ProjectPlatform.Labels
**Exploration:** deep-analysis
**Status:** captured

## D2: YAML surface for label tree declarations

**Choice:** Top-level `labels` key in app.yaml, sibling to `collections`
**Alternatives:**
- Inside collections as a special type — labels aren't documents, forcing the abstraction
- Separate labels.yaml file — cleaner separation but more files to manage
**Rationale:** Labels are structurally distinct from collections. A top-level `labels` key matches the platform concept (labels are not collections) and keeps app.yaml as the single app descriptor file.
**Trade-offs:** Requires extending AppDescriptorParser to handle a new top-level key.
**Sources:** backend/data-json AppDescriptorParser.java, AppDescriptor.java; helpdesk app.yaml
**Exploration:** quick
**Status:** captured

## D3: Label enumeration endpoint

**Choice:** Consume platform's `LabelApi @McpDomain("labels")` — no Pages-side endpoint
**Alternatives:**
- Pages-local LabelResource at `/api/labels/{appId}/{prefix}` — duplicates what platform already provides
**Rationale:** Platform #694 delivered `LabelApi` with `@McpDomain("labels")` providing children/resolve/subtree queries via REST + GraphQL + MCP. `LabelServiceCore` handles tenancyId from `CurrentPrincipal`. Pages frontend calls the platform endpoint directly. App-scoping is handled by prefixing the app's root path to queries (consumer-side concern).
**Trade-offs:** Frontend must know the platform endpoint URL pattern rather than a Pages-local one. Coupling is to the platform API contract.
**Sources:** platform LabelApi @McpDomain("labels"), LabelServiceCore
**Exploration:** quick
**Status:** revised (was Pages-local endpoint, now platform endpoint)

## D4: Dropdown/selector integration mechanism

**Choice:** New `ref: labels/{tree}` syntax on field definitions
**Depends on:** D1 (LabelRegistry SPI provides the data)
**Alternatives:**
- Wire existing DataSetOptions runtime — labels aren't really a dataset, forcing the abstraction
- Both mechanisms — maximum flexibility but more surface area to maintain
**Rationale:** The app.yaml field schema already has `ref` for collection references (e.g., `ref: agents`). Extending ref to support `labels/{rootPath}` is a natural evolution. Backend resolves labels at query time via LabelRegistry.
**Trade-offs:** Overloads the `ref` field with two different resolution paths (collection vs labels). Need clear convention to distinguish.
**Sources:** backend/data-json FieldDef.java, FieldType.java; pages-viz schema-types.ts
**Exploration:** quick
**Status:** captured

## D5: Navigation tree binding model

**Choice:** Populate nav component options — no dynamic page generation
**Alternatives:**
- Dynamic page generation — each label path generates a virtual page; too complex
- Filter-only binding — loses the navigation metaphor
**Rationale:** A navTree group with `labelSource: '{rootPath}'` fetches children from LabelRegistry and renders them as selectable items in a tree/menu component. Selecting an item filters the associated data view. No dynamic page creation needed.
**Trade-offs:** Navigation items don't navigate to distinct pages — they filter within the current view. Full page-per-label would require a template system.
**Sources:** pages-ui nav-desugar.ts, pages-runtime navigation.ts
**Exploration:** quick
**Status:** captured
