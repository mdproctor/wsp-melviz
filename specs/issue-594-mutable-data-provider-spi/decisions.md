## D1: Module placement

**Choice:** SPI (MutableDataProvider, DataContext, MutationException) and CollectionResource in `backend/data/`. JSON-SQLite implementation in a separate `data-json/` module later.
**Alternatives:**
- All in `backend/data-json/` — tighter coupling, self-contained but mixes SPI with impl
- Split: SPI in `data/`, resource in `data-json/` — resource lives with its only consumer but separates resource from the SPI it's designed around
**Rationale:** Follows the existing pattern where DataProvider SPI + DataResource both live in `backend/data/`. Implementations (data-sql, data-prometheus) are separate modules.
**Trade-offs:** CollectionResource has no implementation to call until data-json exists. Tests will need a mock or test-only implementation.
**Sources:** `backend/data/` module structure, DataProvider/DataResource co-location pattern
**Exploration:** quick
**Status:** captured

## D2: Validation error shape

**Choice:** New `MutationException` class with a `code` field + `MutationExceptionMapper`. Flat error body: `{"error": "message", "code": "CODE"}`.
**Alternatives:**
- Field-level errors array — richer but no existing precedent in the codebase
- Reuse DataQueryException with new codes — conflates read errors and write errors
**Rationale:** Matches the existing `DataQueryException` + mapper pattern. Separates mutation errors from query errors conceptually.
**Trade-offs:** No field-level validation detail. If we need it later, we add an optional `fields` array.
**Sources:** DataQueryException.java, DataQueryExceptionMapper.java
**Exploration:** quick
**Status:** captured

## D3: MutationException error code taxonomy

**Choice:** Single `MutationException` class with codes: `VALIDATION_ERROR` (422), `VERSION_CONFLICT` (409), `NOT_FOUND` (404), `SLUG_CONFLICT` (409), `REF_DELETE_BLOCKED` (409).
**Alternatives:**
- Typed subclasses (VersionConflictException, etc.) — type-safe but more classes and mappers for 5 error kinds
**Rationale:** Consistent with DataQueryException pattern. One mapper, one class, codes distinguish error kinds.
**Trade-offs:** Callers must check string codes rather than catching typed exceptions. Acceptable since the mapper handles all dispatch.
**Sources:** DataQueryException pattern
**Exploration:** quick
**Status:** captured

## D4: Document IDs as CMS-style slugs

**Choice:** CMS-style slugs. Client can provide a slug explicitly; if absent, server auto-derives from a configurable slug source field. Slug is immutable after creation. Collision resolution via numeric suffix (-1, -2, -3).
**Alternatives:**
- Server-generated UUID — opaque, not human-readable, poor for URLs
- Client-provided only — no auto-derivation, worse UX for forms
- Mutable slugs with redirect — complex, requires alias table
**Rationale:** JSON apps represent real-world entities (tickets, contacts, articles). Human-readable, URL-friendly IDs improve DX and end-user UX. Immutability prevents reference breakage.
**Trade-offs:** Slug generation logic adds complexity to the SPI. Sequential collision probing could theoretically be slow with many collisions, but in practice CMS content rarely has that many identical titles.
**Sources:** CMS patterns from WordPress, Ghost, Strapi
**Exploration:** quick
**Status:** captured

## D5: MutableDataProvider.create() signature

**Choice:** Direct parameters: `create(DataContext ctx, String collection, JsonNode document, String slugField)`. Returns the generated slug as a `String`.
**Alternatives:**
- CreateRequest record — more extensible but premature abstraction for 3 fields
- Overloaded methods — awkward in Java interfaces
**Rationale:** YAGNI. Matches the simple style of the existing DataProvider interface. Can refactor to records later if parameter count grows.
**Trade-offs:** If create needs more options (e.g., custom slug format, skip validation), we'll need to refactor the signature.
**Sources:** DataProvider.query(DataSetLookup) style
**Exploration:** quick
**Status:** captured

## D6: Ref-delete checking location

**Choice:** Resource-level. CollectionResource checks for references before calling `provider.delete()`. The SPI stays simple — just CRUD.
**Alternatives:**
- SPI-level — MutableDataProvider.delete() throws RefDeleteException. Atomic but mixes cross-collection concern into the provider
- Defer to data-json — leaves the SPI incomplete
**Rationale:** Ref-checking is a cross-collection concern (querying other collections for references). It doesn't belong in a single-collection provider implementation. The resource layer has access to all providers and can query across collections.
**Trade-offs:** Non-atomic: a race between the ref-check and the delete could allow a concurrent create to add a reference. Acceptable for a CMS workload (low contention).
**Sources:** Issue #594 description
**Exploration:** quick
**Status:** captured

## D7: Single-document GET via dedicated method

**Choice:** MutableDataProvider adds `get(DataContext ctx, String collection, String docId)` returning `Optional<JsonNode>`. Separate from the tabular `query()` path.
**Alternatives:**
- Reuse DataProvider.query() with a filter — consistent but forces tabular-to-document conversion and round-trips through the dataset pipeline
**Rationale:** Clean separation. Document-level reads return the document shape (JsonNode), not tabular data (columns + rows). The query() path is for dataset listing; get() is for single-document fetch.
**Trade-offs:** Two read paths exist: query() for tabular and get() for document. Consumers must know which to use. But the use cases are clearly distinct (listing vs. detail).
**Sources:** DataProvider.query() return type (QueryResult with DataSetResult)
**Exploration:** quick
**Status:** captured

## D8: MutableDataProvider discovery

**Choice:** `Instance<MutableDataProvider>` with `canHandle()` routing via `json:{appId}/{collection}` namespace. Same CDI pattern as DataResource uses for DataProvider.
**Alternatives:**
- Inject specific implementation by type — simpler wiring but breaks the SPI abstraction
**Rationale:** Consistent with the established DataProvider discovery pattern. Allows multiple MutableDataProvider implementations in the future.
**Trade-offs:** First-match routing means provider ordering matters if namespaces overlap. Acceptable since the `json:` prefix is unambiguous.
**Sources:** DataResource.resolveProvider() pattern
**Exploration:** quick
**Status:** captured

## D9: DataContext shape

**Choice:** `DataContext(String tenantId)` — record carrying only tenantId.
**Alternatives:**
- DataContext(String tenantId, String appId) — bundles full scope but appId is already in the collection path parameter, making it redundant
**Rationale:** Matches the epic spec. appId is a collection-routing concern passed separately. DataContext encapsulates the security/isolation context, not the business routing.
**Trade-offs:** If providers frequently need both tenantId and appId together, the separate parameter passing may feel verbose. But it keeps DataContext reusable for non-app-scoped providers.
**Sources:** Epic #592 spec
**Exploration:** quick
**Status:** captured
