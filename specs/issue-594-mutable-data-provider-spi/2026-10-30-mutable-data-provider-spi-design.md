---
title: "MutableDataProvider SPI and CollectionResource"
date: 2026-10-30
issue: 594
parent: 592
depends-on: 593
---

# MutableDataProvider SPI and CollectionResource

Add write-path support to the DataProvider architecture. A new
`MutableDataProvider` SPI extends the read-only `DataProvider` with
CRUD operations on JSON documents. A new `CollectionResource` REST
endpoint exposes these operations over HTTP with optimistic concurrency
via `If-Match`/`ETag` and CMS-style slug IDs.

All new types live in `backend/data/` (the core SPI module). The
JSON-SQLite implementation that backs this SPI ships in a separate
`data-json/` module (issue #595+), following the same pattern as
`data-sql/` and `data-prometheus/`.

## DataContext

```java
package io.casehub.pages.data;

public record DataContext(String tenantId) {}
```

Carries tenant isolation context. Extracted from the JWT in the
resource layer, passed to all mutation methods. Intentionally carries
only `tenantId` — `appId` is a collection-routing concern passed as
a separate parameter.

## MutableDataProvider

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.Optional;
import java.util.OptionalInt;

public interface MutableDataProvider extends DataProvider {

    Optional<JsonNode> get(DataContext ctx, String collection, String docId);

    String create(DataContext ctx, String collection,
                  JsonNode document, String slugField);

    void update(DataContext ctx, String collection, String docId,
                JsonNode changes, OptionalInt expectedVersion);

    void delete(DataContext ctx, String collection, String docId);

    boolean hasReferences(DataContext ctx, String collection, String docId);
}
```

### Method contracts

**`get`** — Returns the full document as a `JsonNode` including
system fields (`_id`, `_version`, `_created_at`, `_updated_at`).
Returns `Optional.empty()` if the document does not exist.

**`create`** — Inserts a new document. `slugField` names the field
whose value is slugified to produce the document ID. If the document
contains an `_id` field, that value is used as the slug directly
(client-provided slug). If `_id` is absent and `slugField` is
non-null, the value of that field is slugified. If neither is
available, the implementation should fall back to a generated ID.

Returns the slug (document ID) of the created document.

Slug format: lowercase, non-alphanumeric characters replaced with
hyphens, leading/trailing hyphens stripped, consecutive hyphens
collapsed. On collision, append `-1`, `-2`, etc. until unique within
the `(tenant_id, app_id, collection)` scope.

Slugs are immutable after creation — updating the source field does
not change the slug.

**`update`** — Applies a partial update (merge patch semantics).
`changes` contains only the fields to modify. `expectedVersion` is
the optimistic concurrency guard — if present and the stored version
does not match, the implementation throws `MutationException` with
code `VERSION_CONFLICT`. The stored `_version` is incremented on
every successful update.

**`delete`** — Removes the document. The caller (CollectionResource)
is responsible for calling `hasReferences` before delete — the SPI
method itself does not check.

**`hasReferences`** — Returns `true` if any document in any collection
within the same `(tenant_id, app_id)` scope references the given
document. Used by CollectionResource for restrict-on-ref-delete.

### Inheritance from DataProvider

`MutableDataProvider` inherits `type()`, `canHandle(String)`, and
`query(DataSetLookup)` from `DataProvider`. Implementations use the
`json:{appId}/{collection}` namespace for `canHandle()` routing.
The `query()` method returns the collection contents as tabular
`QueryResult` data for the existing dataset pipeline. The `get()`
method returns a single document as `JsonNode` — these are distinct
read paths for distinct use cases (listing vs. detail).

## MutationException

```java
package io.casehub.pages.data;

public class MutationException extends RuntimeException {
    private final String code;

    public MutationException(String code, String message) {
        super(message);
        this.code = code;
    }

    public MutationException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public String code() { return code; }
}
```

### Error codes

| Code | HTTP Status | When |
|------|-------------|------|
| `VALIDATION_ERROR` | 422 | Malformed document, missing required fields |
| `VERSION_CONFLICT` | 409 | `If-Match` version does not match stored `_version` |
| `NOT_FOUND` | 404 | Document does not exist (update/delete) |
| `SLUG_CONFLICT` | 409 | Slug collision not resolvable (exhausted suffixes) |
| `REF_DELETE_BLOCKED` | 409 | Document is referenced by other documents |

Response body follows the existing `DataQueryException` pattern:

```json
{"error": "Version mismatch: expected 3, found 5", "code": "VERSION_CONFLICT"}
```

## MutationExceptionMapper

```java
package io.casehub.pages.data;

import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;
import java.util.Map;

@Provider
public class MutationExceptionMapper implements ExceptionMapper<MutationException> {
    @Override
    public Response toResponse(MutationException e) {
        int status = switch (e.code()) {
            case "VALIDATION_ERROR" -> 422;
            case "VERSION_CONFLICT", "SLUG_CONFLICT", "REF_DELETE_BLOCKED" -> 409;
            case "NOT_FOUND" -> 404;
            default -> 500;
        };
        return Response.status(status)
            .entity(Map.of("error", e.getMessage(), "code", e.code()))
            .type(MediaType.APPLICATION_JSON)
            .build();
    }
}
```

## CollectionResource

```
POST   /api/collections/{appId}/{collection}       → create
GET    /api/collections/{appId}/{collection}/{id}   → get
PATCH  /api/collections/{appId}/{collection}/{id}   → update
DELETE /api/collections/{appId}/{collection}/{id}   → delete
```

### Wiring

```java
package io.casehub.pages.data;

@Path("/api/collections/{appId}/{collection}")
@Authenticated
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class CollectionResource {

    @Inject JsonWebToken jwt;

    @ConfigProperty(name = "casehub.pages.data.tenant-claim",
                    defaultValue = "tenant_id")
    String tenantClaim;

    @Inject @Any Instance<MutableDataProvider> providers;
    @Inject DataCacheService cacheService;

    // ... endpoints below
}
```

Discovery uses `Instance<MutableDataProvider>` with the same
`canHandle()` routing pattern as `DataResource`. The dataset ID
is constructed as `json:{appId}/{collection}` from the path
parameters.

### Tenant extraction

Reuses the same JWT claim extraction pattern as `DataResource`.
Returns 401 if the tenant claim is missing. The extracted `tenantId`
is wrapped in a `DataContext`.

### POST — create document

Request: JSON document body. Optional `_id` field for client-provided
slug.

Response: 201 Created. Body: the created document including system
fields (`_id`, `_version`, `_created_at`, `_updated_at`).
`Location` header: `/api/collections/{appId}/{collection}/{slug}`.
`ETag` header: `"1"` (initial version).

Flow:
1. Extract tenant → `DataContext`
2. Resolve `MutableDataProvider` via `canHandle("json:{appId}/{collection}")`
3. Call `provider.create(ctx, collection, document, slugField)`
   — `slugField` comes from request context (query param or later
   from app.yaml config); defaults to `"title"` if absent
4. Fetch the created document via `provider.get()` for the response
5. Invalidate cache: `cacheService.invalidate(tenantId, dataSetId)`
6. Return 201 with document, Location, ETag

### GET — single document

Response: 200 OK. Body: the document as JSON. `ETag` header with
the `_version` value.

If not found: 404 with `{"error": "Not found", "code": "NOT_FOUND"}`.

### PATCH — partial update

Request: JSON merge patch body. `If-Match` header required (contains
the expected version as a quoted string, e.g., `"3"`).

Response: 200 OK. Body: the full updated document. `ETag` header
with the new `_version`.

Flow:
1. Extract tenant → `DataContext`
2. Parse `If-Match` header → `OptionalInt expectedVersion`
   — if `If-Match` is absent, pass `OptionalInt.empty()` (no
   concurrency check, last-write-wins)
3. Resolve provider, call `provider.update(ctx, collection, id, changes, expectedVersion)`
4. Fetch updated document via `provider.get()` for the response
5. Invalidate cache
6. Return 200 with document, ETag

If version mismatch: provider throws `MutationException("VERSION_CONFLICT", ...)`
→ mapper returns 409.

### DELETE — remove document

Request: no body.

Response: 204 No Content.

Flow:
1. Extract tenant → `DataContext`
2. Resolve provider
3. Call `provider.hasReferences(ctx, collection, id)`
   — if `true`, throw `MutationException("REF_DELETE_BLOCKED",
   "Document is referenced by other documents")`
4. Call `provider.delete(ctx, collection, id)`
5. Invalidate cache
6. Return 204

### Slug configuration

The `slugField` parameter for create operations comes from a query
parameter on the POST request: `?slugField=title`. This is a
temporary mechanism — once the app descriptor (`app.yaml`) lands
in a later issue, the slug source field will come from the
collection's schema definition. The query parameter provides a
workable default until then.

If neither `_id` nor `slugField` resolves to a value, the
implementation falls back to a generated identifier.

## Cache invalidation

Every mutation (create, update, delete) calls
`cacheService.invalidate(tenantId, dataSetId)` where `dataSetId`
is the `json:{appId}/{collection}` string. This clears any cached
`QueryResult` entries for that collection, ensuring subsequent
`DataResource.query()` calls see fresh data.

## Testing strategy

### Unit tests (in `backend/data/`)

- **MutationExceptionMapperTest** — verify each code maps to the
  correct HTTP status and response body shape
- **CollectionResourceTest** — mock `MutableDataProvider`, verify:
  - POST returns 201 + Location + ETag
  - GET returns 200 + ETag, 404 when missing
  - PATCH with If-Match passes expectedVersion to provider
  - PATCH without If-Match passes OptionalInt.empty()
  - DELETE calls hasReferences before delete
  - DELETE returns 409 when hasReferences is true
  - Missing tenant claim returns 401
  - No matching provider returns 400
  - Cache invalidation called after every mutation

### Slug utility tests

- Slugify: lowercasing, special char replacement, hyphen collapsing
- Collision: numeric suffix generation
- Client-provided slug passthrough
- Edge cases: empty string, all-special-chars, very long input

## Files to create

| File | Content |
|------|---------|
| `backend/data/.../DataContext.java` | Record: `DataContext(String tenantId)` |
| `backend/data/.../MutableDataProvider.java` | Interface extending DataProvider |
| `backend/data/.../MutationException.java` | Exception with code field |
| `backend/data/.../MutationExceptionMapper.java` | JAX-RS ExceptionMapper |
| `backend/data/.../CollectionResource.java` | REST resource |
| `backend/data/.../SlugUtils.java` | Slug generation utility |

All files under `backend/data/src/main/java/io/casehub/pages/data/`.

## Files to modify

| File | Change |
|------|--------|
| None | This issue adds new files only. No modifications to existing code. |

## Out of scope

- JSON-SQLite implementation of MutableDataProvider (issue #595+)
- App descriptor (app.yaml) for collection schema configuration
- Frontend wiring of mutableRestSource to the new endpoints
- Full-text search within collections
- Pagination on collection listing

## References

- `backend/data/src/main/java/io/casehub/pages/data/DataProvider.java` — existing read-only SPI
- `backend/data/src/main/java/io/casehub/pages/data/DataResource.java` — existing REST resource pattern
- `backend/data/src/main/java/io/casehub/pages/data/DataCacheService.java` — cache invalidation API
- `backend/data/src/main/java/io/casehub/pages/data/DataQueryException.java` — error pattern template
- `backend/data/src/main/java/io/casehub/pages/data/DataQueryExceptionMapper.java` — mapper pattern template
- `backend/pages-sqlite/` — shared SQLite infrastructure (dependency of data-json, not this issue)
- `backend/layout-sqlite/` — consumer pattern for pages-sqlite
- `packages/pages-data/src/datasource/types.ts` — frontend MutableDataSource/DataAction types
- `packages/pages-data/src/datasource/sources/mutable-rest-source.ts` — frontend write source
- Epic #592 — JSON Apps Runtime parent epic (settled architecture)
- Issue #593 — pages-sqlite shared infrastructure (dependency)
