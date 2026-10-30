# MutableDataProvider SPI and CollectionResource Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #594 — Implement MutableDataProvider SPI and CollectionResource
**Issue group:** #594

**Goal:** Add write-path support to the DataProvider architecture via a
new MutableDataProvider SPI, MutationException error handling, slug
utilities, and a CollectionResource REST endpoint.

**Architecture:** Six new files in `backend/data/` (the core SPI module).
MutableDataProvider extends DataProvider with CRUD + ref-check methods.
CollectionResource exposes POST/GET/PATCH/DELETE with optimistic
concurrency via If-Match/ETag and CMS-style slug IDs. Error handling via
MutationException + mapper, following the existing DataQueryException
pattern.

**Tech Stack:** Java 21, Quarkus (CDI, JAX-RS, SmallRye JWT), Jackson,
JUnit 5, AssertJ, REST Assured

## Global Constraints

- All new files in `backend/data/src/main/java/io/casehub/pages/data/`
- All new tests in `backend/data/src/test/java/io/casehub/pages/data/`
- No modifications to existing files
- No new Maven dependencies — Jackson, CDI, JAX-RS already available
- Follow existing code style: no Javadoc, no comments unless non-obvious
- Test style: plain unit tests for utilities, `@QuarkusTest` with `@Alternative` inner-class providers for resource tests
- Build command: `/opt/homebrew/bin/mvn -pl backend/data -am test`

---

## Batch 1: SPI Foundation — DataContext, MutableDataProvider, MutationException

After this batch: the write-path SPI contract is defined and the error
handling infrastructure is tested. No REST endpoints yet.

### Task 1: DataContext record + MutationException + MutationExceptionMapper

**Files:**
- Create: `backend/data/src/main/java/io/casehub/pages/data/DataContext.java`
- Create: `backend/data/src/main/java/io/casehub/pages/data/MutationException.java`
- Create: `backend/data/src/main/java/io/casehub/pages/data/MutationExceptionMapper.java`
- Test: `backend/data/src/test/java/io/casehub/pages/data/MutationExceptionMapperTest.java`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `DataContext(String tenantId)` — record used by all MutableDataProvider methods
  - `MutationException(String code, String message)` — runtime exception with `code()` accessor
  - `MutationException(String code, String message, Throwable cause)` — chained constructor
  - `MutationExceptionMapper implements ExceptionMapper<MutationException>` — maps codes to HTTP statuses

- [ ] **Step 1: Write MutationExceptionMapper tests**

Create `backend/data/src/test/java/io/casehub/pages/data/MutationExceptionMapperTest.java`:

```java
package io.casehub.pages.data;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class MutationExceptionMapperTest {

    private final MutationExceptionMapper mapper = new MutationExceptionMapper();

    @Test
    void validationErrorReturns422() {
        var ex = new MutationException("VALIDATION_ERROR", "missing required field: name");
        var response = mapper.toResponse(ex);
        assertThat(response.getStatus()).isEqualTo(422);
        @SuppressWarnings("unchecked")
        var entity = (Map<String, String>) response.getEntity();
        assertThat(entity.get("code")).isEqualTo("VALIDATION_ERROR");
        assertThat(entity.get("error")).isEqualTo("missing required field: name");
    }

    @Test
    void versionConflictReturns409() {
        var ex = new MutationException("VERSION_CONFLICT", "expected 3, found 5");
        var response = mapper.toResponse(ex);
        assertThat(response.getStatus()).isEqualTo(409);
        @SuppressWarnings("unchecked")
        var entity = (Map<String, String>) response.getEntity();
        assertThat(entity.get("code")).isEqualTo("VERSION_CONFLICT");
    }

    @Test
    void slugConflictReturns409() {
        var ex = new MutationException("SLUG_CONFLICT", "slug already exists");
        var response = mapper.toResponse(ex);
        assertThat(response.getStatus()).isEqualTo(409);
    }

    @Test
    void refDeleteBlockedReturns409() {
        var ex = new MutationException("REF_DELETE_BLOCKED", "document is referenced");
        var response = mapper.toResponse(ex);
        assertThat(response.getStatus()).isEqualTo(409);
    }

    @Test
    void notFoundReturns404() {
        var ex = new MutationException("NOT_FOUND", "document not found");
        var response = mapper.toResponse(ex);
        assertThat(response.getStatus()).isEqualTo(404);
    }

    @Test
    void unknownCodeReturns500() {
        var ex = new MutationException("UNKNOWN", "something broke");
        var response = mapper.toResponse(ex);
        assertThat(response.getStatus()).isEqualTo(500);
    }

    @Test
    void causeChainPreserved() {
        var cause = new RuntimeException("root cause");
        var ex = new MutationException("NOT_FOUND", "gone", cause);
        assertThat(ex.getCause()).isSameAs(cause);
        assertThat(ex.code()).isEqualTo("NOT_FOUND");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -Dtest=MutationExceptionMapperTest`
Expected: compilation failure — `MutationException`, `MutationExceptionMapper` do not exist

- [ ] **Step 3: Create DataContext record**

Create `backend/data/src/main/java/io/casehub/pages/data/DataContext.java`:

```java
package io.casehub.pages.data;

public record DataContext(String tenantId) {}
```

- [ ] **Step 4: Create MutationException**

Create `backend/data/src/main/java/io/casehub/pages/data/MutationException.java`:

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

- [ ] **Step 5: Create MutationExceptionMapper**

Create `backend/data/src/main/java/io/casehub/pages/data/MutationExceptionMapper.java`:

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

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -Dtest=MutationExceptionMapperTest`
Expected: all 7 tests PASS

- [ ] **Step 7: Commit**

```bash
git add backend/data/src/main/java/io/casehub/pages/data/DataContext.java
git add backend/data/src/main/java/io/casehub/pages/data/MutationException.java
git add backend/data/src/main/java/io/casehub/pages/data/MutationExceptionMapper.java
git add backend/data/src/test/java/io/casehub/pages/data/MutationExceptionMapperTest.java
git commit -m "feat(data): add DataContext, MutationException, and MutationExceptionMapper

Refs #594"
```

### Task 2: MutableDataProvider interface + SlugUtils

**Files:**
- Create: `backend/data/src/main/java/io/casehub/pages/data/MutableDataProvider.java`
- Create: `backend/data/src/main/java/io/casehub/pages/data/SlugUtils.java`
- Test: `backend/data/src/test/java/io/casehub/pages/data/SlugUtilsTest.java`

**Interfaces:**
- Consumes:
  - `DataContext(String tenantId)` from Task 1
  - `DataProvider` — existing interface with `type()`, `canHandle(String)`, `query(DataSetLookup)`
- Produces:
  - `MutableDataProvider extends DataProvider` — interface with `get`, `create`, `update`, `delete`, `hasReferences`
  - `SlugUtils.slugify(String input)` — returns `String`, the slugified form
  - `SlugUtils.resolveSlug(JsonNode document, String slugField)` — extracts and slugifies the slug source value from a document

- [ ] **Step 1: Write SlugUtils tests**

Create `backend/data/src/test/java/io/casehub/pages/data/SlugUtilsTest.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class SlugUtilsTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void slugifyLowercases() {
        assertThat(SlugUtils.slugify("Hello World")).isEqualTo("hello-world");
    }

    @Test
    void slugifyReplacesSpecialChars() {
        assertThat(SlugUtils.slugify("Hello, World! #1")).isEqualTo("hello-world-1");
    }

    @Test
    void slugifyCollapsesConsecutiveHyphens() {
        assertThat(SlugUtils.slugify("hello---world")).isEqualTo("hello-world");
    }

    @Test
    void slugifyStripsLeadingTrailingHyphens() {
        assertThat(SlugUtils.slugify("-hello-world-")).isEqualTo("hello-world");
    }

    @Test
    void slugifyHandlesEmptyString() {
        assertThat(SlugUtils.slugify("")).isEmpty();
    }

    @Test
    void slugifyHandlesAllSpecialChars() {
        assertThat(SlugUtils.slugify("!@#$%^&*()")).isEmpty();
    }

    @Test
    void slugifyPreservesNumbers() {
        assertThat(SlugUtils.slugify("Issue 42 - Critical Bug")).isEqualTo("issue-42-critical-bug");
    }

    @Test
    void slugifyTruncatesLongInput() {
        String longTitle = "a".repeat(200);
        String slug = SlugUtils.slugify(longTitle);
        assertThat(slug.length()).isLessThanOrEqualTo(128);
    }

    @Test
    void resolveSlugUsesExplicitId() {
        ObjectNode doc = mapper.createObjectNode();
        doc.put("_id", "my-custom-slug");
        doc.put("title", "Some Title");
        assertThat(SlugUtils.resolveSlug(doc, "title")).isEqualTo(Optional.of("my-custom-slug"));
    }

    @Test
    void resolveSlugDerivesFromSlugField() {
        ObjectNode doc = mapper.createObjectNode();
        doc.put("title", "My Great Article!");
        assertThat(SlugUtils.resolveSlug(doc, "title")).isEqualTo(Optional.of("my-great-article"));
    }

    @Test
    void resolveSlugReturnsEmptyWhenNoSource() {
        ObjectNode doc = mapper.createObjectNode();
        doc.put("body", "some content");
        assertThat(SlugUtils.resolveSlug(doc, null)).isEmpty();
    }

    @Test
    void resolveSlugReturnsEmptyWhenSlugFieldMissing() {
        ObjectNode doc = mapper.createObjectNode();
        doc.put("body", "some content");
        assertThat(SlugUtils.resolveSlug(doc, "title")).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -Dtest=SlugUtilsTest`
Expected: compilation failure — `SlugUtils` does not exist

- [ ] **Step 3: Create SlugUtils**

Create `backend/data/src/main/java/io/casehub/pages/data/SlugUtils.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.JsonNode;

import java.util.Optional;
import java.util.regex.Pattern;

public final class SlugUtils {

    private static final int MAX_SLUG_LENGTH = 128;
    private static final Pattern NON_ALNUM = Pattern.compile("[^a-z0-9]+");
    private static final Pattern LEADING_TRAILING_HYPHENS = Pattern.compile("^-+|-+$");

    private SlugUtils() {}

    public static String slugify(String input) {
        if (input == null || input.isBlank()) return "";
        String slug = NON_ALNUM.matcher(input.toLowerCase()).replaceAll("-");
        slug = LEADING_TRAILING_HYPHENS.matcher(slug).replaceAll("");
        if (slug.length() > MAX_SLUG_LENGTH) {
            slug = slug.substring(0, MAX_SLUG_LENGTH);
            slug = LEADING_TRAILING_HYPHENS.matcher(slug).replaceAll("");
        }
        return slug;
    }

    public static Optional<String> resolveSlug(JsonNode document, String slugField) {
        JsonNode idNode = document.get("_id");
        if (idNode != null && idNode.isTextual() && !idNode.asText().isBlank()) {
            return Optional.of(idNode.asText());
        }
        if (slugField != null) {
            JsonNode fieldNode = document.get(slugField);
            if (fieldNode != null && fieldNode.isTextual() && !fieldNode.asText().isBlank()) {
                String slugified = slugify(fieldNode.asText());
                return slugified.isEmpty() ? Optional.empty() : Optional.of(slugified);
            }
        }
        return Optional.empty();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -Dtest=SlugUtilsTest`
Expected: all 12 tests PASS

- [ ] **Step 5: Create MutableDataProvider interface**

Create `backend/data/src/main/java/io/casehub/pages/data/MutableDataProvider.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.JsonNode;

import java.util.Optional;
import java.util.OptionalInt;

public interface MutableDataProvider extends DataProvider {

    Optional<JsonNode> get(DataContext ctx, String collection, String docId);

    String create(DataContext ctx, String collection, JsonNode document, String slugField);

    void update(DataContext ctx, String collection, String docId,
                JsonNode changes, OptionalInt expectedVersion);

    void delete(DataContext ctx, String collection, String docId);

    boolean hasReferences(DataContext ctx, String collection, String docId);
}
```

- [ ] **Step 6: Run full module tests to verify nothing breaks**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test`
Expected: all existing + new tests PASS

- [ ] **Step 7: Commit**

```bash
git add backend/data/src/main/java/io/casehub/pages/data/MutableDataProvider.java
git add backend/data/src/main/java/io/casehub/pages/data/SlugUtils.java
git add backend/data/src/test/java/io/casehub/pages/data/SlugUtilsTest.java
git commit -m "feat(data): add MutableDataProvider interface and SlugUtils

Refs #594"
```

## Batch 2: CollectionResource — REST endpoint wiring

After this batch: the full REST endpoint is implemented and tested
with a mock MutableDataProvider. The issue is complete.

### Task 3: CollectionResource with integration tests

**Files:**
- Create: `backend/data/src/main/java/io/casehub/pages/data/CollectionResource.java`
- Test: `backend/data/src/test/java/io/casehub/pages/data/CollectionResourceTest.java`

**Interfaces:**
- Consumes:
  - `DataContext(String tenantId)` from Task 1
  - `MutationException(String code, String message)` from Task 1
  - `MutableDataProvider` from Task 2 — `get`, `create`, `update`, `delete`, `hasReferences`
  - `DataCacheService.invalidate(String tenantId, String dataSetId)` — existing
  - JWT tenant extraction pattern from `DataResource`
- Produces:
  - `CollectionResource` — REST resource at `/api/collections/{appId}/{collection}`
  - POST `/` → 201 + Location + ETag
  - GET `/{id}` → 200 + ETag or 404
  - PATCH `/{id}` → 200 + ETag (reads If-Match)
  - DELETE `/{id}` → 204 (ref-check first)

- [ ] **Step 1: Write CollectionResource integration tests**

Create `backend/data/src/test/java/io/casehub/pages/data/CollectionResourceTest.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.smallrye.jwt.build.Jwt;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Optional;
import java.util.OptionalInt;
import java.util.Set;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.endsWith;

@QuarkusTest
class CollectionResourceTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Alternative
    @Priority(1)
    @ApplicationScoped
    static class TestMutableDataProvider implements MutableDataProvider {

        @Override
        public String type() { return "test-mutable"; }

        @Override
        public boolean canHandle(String dataSetId) {
            return dataSetId.startsWith("json:");
        }

        @Override
        public QueryResult query(DataSetLookup lookup) {
            return QueryResult.complete(new DataSetResult(List.of(), List.of()));
        }

        @Override
        public Optional<JsonNode> get(DataContext ctx, String collection, String docId) {
            if ("not-found".equals(docId)) return Optional.empty();
            ObjectNode doc = MAPPER.createObjectNode();
            doc.put("_id", docId);
            doc.put("_version", 1);
            doc.put("_created_at", "2026-10-30T10:00:00Z");
            doc.put("_updated_at", "2026-10-30T10:00:00Z");
            doc.put("title", "Test Document");
            return Optional.of(doc);
        }

        @Override
        public String create(DataContext ctx, String collection,
                             JsonNode document, String slugField) {
            return "test-document";
        }

        @Override
        public void update(DataContext ctx, String collection, String docId,
                           JsonNode changes, OptionalInt expectedVersion) {
            if (expectedVersion.isPresent() && expectedVersion.getAsInt() == 99) {
                throw new MutationException("VERSION_CONFLICT",
                    "expected 99, found 1");
            }
        }

        @Override
        public void delete(DataContext ctx, String collection, String docId) {
            if ("not-found".equals(docId)) {
                throw new MutationException("NOT_FOUND", "document not found");
            }
        }

        @Override
        public boolean hasReferences(DataContext ctx, String collection, String docId) {
            return "referenced-doc".equals(docId);
        }
    }

    private String validToken() {
        return Jwt.claims()
            .subject("alice")
            .claim("tenant_id", "dev")
            .groups(Set.of("user"))
            .sign();
    }

    private String tokenWithoutTenant() {
        return Jwt.claims()
            .subject("alice")
            .groups(Set.of("user"))
            .sign();
    }

    @Test
    void post_creates_document_returns_201() {
        given()
            .auth().oauth2(validToken())
            .contentType(ContentType.JSON)
            .body("{\"title\":\"New Ticket\"}")
            .when()
            .post("/api/collections/helpdesk/tickets")
            .then()
            .statusCode(201)
            .header("Location", endsWith("/api/collections/helpdesk/tickets/test-document"))
            .header("ETag", equalTo("\"1\""))
            .body("_id", equalTo("test-document"));
    }

    @Test
    void post_with_slug_field_query_param() {
        given()
            .auth().oauth2(validToken())
            .contentType(ContentType.JSON)
            .queryParam("slugField", "name")
            .body("{\"name\":\"My Item\"}")
            .when()
            .post("/api/collections/helpdesk/tickets")
            .then()
            .statusCode(201);
    }

    @Test
    void get_returns_document_with_etag() {
        given()
            .auth().oauth2(validToken())
            .when()
            .get("/api/collections/helpdesk/tickets/my-ticket")
            .then()
            .statusCode(200)
            .header("ETag", equalTo("\"1\""))
            .body("_id", equalTo("my-ticket"))
            .body("title", equalTo("Test Document"));
    }

    @Test
    void get_not_found_returns_404() {
        given()
            .auth().oauth2(validToken())
            .when()
            .get("/api/collections/helpdesk/tickets/not-found")
            .then()
            .statusCode(404)
            .body("code", equalTo("NOT_FOUND"));
    }

    @Test
    void patch_with_if_match_returns_200() {
        given()
            .auth().oauth2(validToken())
            .contentType(ContentType.JSON)
            .header("If-Match", "\"1\"")
            .body("{\"priority\":\"high\"}")
            .when()
            .patch("/api/collections/helpdesk/tickets/my-ticket")
            .then()
            .statusCode(200)
            .header("ETag", equalTo("\"1\""))
            .body("_id", equalTo("my-ticket"));
    }

    @Test
    void patch_without_if_match_succeeds() {
        given()
            .auth().oauth2(validToken())
            .contentType(ContentType.JSON)
            .body("{\"priority\":\"high\"}")
            .when()
            .patch("/api/collections/helpdesk/tickets/my-ticket")
            .then()
            .statusCode(200);
    }

    @Test
    void patch_version_conflict_returns_409() {
        given()
            .auth().oauth2(validToken())
            .contentType(ContentType.JSON)
            .header("If-Match", "\"99\"")
            .body("{\"priority\":\"high\"}")
            .when()
            .patch("/api/collections/helpdesk/tickets/my-ticket")
            .then()
            .statusCode(409)
            .body("code", equalTo("VERSION_CONFLICT"));
    }

    @Test
    void delete_returns_204() {
        given()
            .auth().oauth2(validToken())
            .when()
            .delete("/api/collections/helpdesk/tickets/my-ticket")
            .then()
            .statusCode(204);
    }

    @Test
    void delete_referenced_doc_returns_409() {
        given()
            .auth().oauth2(validToken())
            .when()
            .delete("/api/collections/helpdesk/tickets/referenced-doc")
            .then()
            .statusCode(409)
            .body("code", equalTo("REF_DELETE_BLOCKED"));
    }

    @Test
    void missing_tenant_returns_401() {
        given()
            .auth().oauth2(tokenWithoutTenant())
            .contentType(ContentType.JSON)
            .body("{\"title\":\"test\"}")
            .when()
            .post("/api/collections/helpdesk/tickets")
            .then()
            .statusCode(401)
            .body("error", equalTo("Missing claim: tenant_id"));
    }

    @Test
    void no_auth_returns_401() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"title\":\"test\"}")
            .when()
            .post("/api/collections/helpdesk/tickets")
            .then()
            .statusCode(401);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -Dtest=CollectionResourceTest`
Expected: compilation failure — `CollectionResource` does not exist

- [ ] **Step 3: Create CollectionResource**

Create `backend/data/src/main/java/io/casehub/pages/data/CollectionResource.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.JsonNode;
import io.quarkus.security.Authenticated;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.DELETE;
import jakarta.ws.rs.DefaultValue;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.HeaderParam;
import jakarta.ws.rs.PATCH;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.jwt.JsonWebToken;

import java.net.URI;
import java.util.Map;
import java.util.OptionalInt;

@Path("/api/collections/{appId}/{collection}")
@Authenticated
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class CollectionResource {

    @Inject
    JsonWebToken jwt;

    @ConfigProperty(name = "casehub.pages.data.tenant-claim", defaultValue = "tenant_id")
    String tenantClaim;

    @Inject
    @Any
    Instance<MutableDataProvider> providers;

    @Inject
    DataCacheService cacheService;

    @POST
    public Response create(@PathParam("appId") String appId,
                           @PathParam("collection") String collection,
                           @QueryParam("slugField") @DefaultValue("title") String slugField,
                           JsonNode document) {
        String tenantId = extractTenant();
        if (tenantId == null) return missingTenantResponse();

        DataContext ctx = new DataContext(tenantId);
        String dataSetId = dataSetId(appId, collection);
        MutableDataProvider provider = resolveProvider(dataSetId);
        if (provider == null) return noProviderResponse(dataSetId);

        String slug = provider.create(ctx, collection, document, slugField);

        JsonNode created = provider.get(ctx, collection, slug).orElse(null);
        cacheService.invalidate(tenantId, dataSetId);

        int version = created != null && created.has("_version")
            ? created.get("_version").asInt() : 1;

        return Response.created(URI.create("/api/collections/" + appId + "/" + collection + "/" + slug))
            .entity(created)
            .header("ETag", "\"" + version + "\"")
            .build();
    }

    @GET
    @Path("/{id}")
    public Response get(@PathParam("appId") String appId,
                        @PathParam("collection") String collection,
                        @PathParam("id") String id) {
        String tenantId = extractTenant();
        if (tenantId == null) return missingTenantResponse();

        DataContext ctx = new DataContext(tenantId);
        String dataSetId = dataSetId(appId, collection);
        MutableDataProvider provider = resolveProvider(dataSetId);
        if (provider == null) return noProviderResponse(dataSetId);

        return provider.get(ctx, collection, id)
            .map(doc -> {
                int version = doc.has("_version") ? doc.get("_version").asInt() : 1;
                return Response.ok(doc)
                    .header("ETag", "\"" + version + "\"")
                    .build();
            })
            .orElseThrow(() -> new MutationException("NOT_FOUND", "Not found"));
    }

    @PATCH
    @Path("/{id}")
    public Response update(@PathParam("appId") String appId,
                           @PathParam("collection") String collection,
                           @PathParam("id") String id,
                           @HeaderParam("If-Match") String ifMatch,
                           JsonNode changes) {
        String tenantId = extractTenant();
        if (tenantId == null) return missingTenantResponse();

        DataContext ctx = new DataContext(tenantId);
        String dataSetId = dataSetId(appId, collection);
        MutableDataProvider provider = resolveProvider(dataSetId);
        if (provider == null) return noProviderResponse(dataSetId);

        OptionalInt expectedVersion = parseIfMatch(ifMatch);
        provider.update(ctx, collection, id, changes, expectedVersion);

        JsonNode updated = provider.get(ctx, collection, id).orElse(null);
        cacheService.invalidate(tenantId, dataSetId);

        int version = updated != null && updated.has("_version")
            ? updated.get("_version").asInt() : 1;

        return Response.ok(updated)
            .header("ETag", "\"" + version + "\"")
            .build();
    }

    @DELETE
    @Path("/{id}")
    public Response delete(@PathParam("appId") String appId,
                           @PathParam("collection") String collection,
                           @PathParam("id") String id) {
        String tenantId = extractTenant();
        if (tenantId == null) return missingTenantResponse();

        DataContext ctx = new DataContext(tenantId);
        String dataSetId = dataSetId(appId, collection);
        MutableDataProvider provider = resolveProvider(dataSetId);
        if (provider == null) return noProviderResponse(dataSetId);

        if (provider.hasReferences(ctx, collection, id)) {
            throw new MutationException("REF_DELETE_BLOCKED",
                "Document is referenced by other documents");
        }

        provider.delete(ctx, collection, id);
        cacheService.invalidate(tenantId, dataSetId);

        return Response.noContent().build();
    }

    private static String dataSetId(String appId, String collection) {
        return "json:" + appId + "/" + collection;
    }

    private MutableDataProvider resolveProvider(String dataSetId) {
        for (MutableDataProvider p : providers) {
            if (p.canHandle(dataSetId)) return p;
        }
        return null;
    }

    private static OptionalInt parseIfMatch(String ifMatch) {
        if (ifMatch == null || ifMatch.isBlank()) return OptionalInt.empty();
        String stripped = ifMatch.replace("\"", "").trim();
        try {
            return OptionalInt.of(Integer.parseInt(stripped));
        } catch (NumberFormatException e) {
            return OptionalInt.empty();
        }
    }

    private String extractTenant() {
        Object claim = jwt.getClaim(tenantClaim);
        return claim != null ? claim.toString() : null;
    }

    private Response missingTenantResponse() {
        return Response.status(Response.Status.UNAUTHORIZED)
            .entity(Map.of("error", "Missing claim: " + tenantClaim))
            .build();
    }

    private Response noProviderResponse(String dataSetId) {
        return Response.status(Response.Status.BAD_REQUEST)
            .entity(Map.of("error", "No provider found for dataset: " + dataSetId))
            .build();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test -Dtest=CollectionResourceTest`
Expected: all 11 tests PASS

- [ ] **Step 5: Run full module test suite**

Run: `/opt/homebrew/bin/mvn -pl backend/data -am test`
Expected: all tests PASS (existing + new)

- [ ] **Step 6: Commit**

```bash
git add backend/data/src/main/java/io/casehub/pages/data/CollectionResource.java
git add backend/data/src/test/java/io/casehub/pages/data/CollectionResourceTest.java
git commit -m "feat(data): add CollectionResource REST endpoint for collection CRUD

POST/GET/PATCH/DELETE at /api/collections/{appId}/{collection}
with If-Match/ETag optimistic concurrency and ref-delete checking.

Refs #594"
```

## References

- [2026-10-30-mutable-data-provider-spi-design.md] — design spec this plan implements
- `backend/data/src/main/java/io/casehub/pages/data/DataProvider.java` — existing read-only SPI
- `backend/data/src/main/java/io/casehub/pages/data/DataResource.java` — REST resource pattern
- `backend/data/src/main/java/io/casehub/pages/data/DataCacheService.java` — cache invalidation API
- `backend/data/src/main/java/io/casehub/pages/data/DataQueryException.java` — error pattern template
- `backend/data/src/main/java/io/casehub/pages/data/DataQueryExceptionMapper.java` — mapper pattern template
- `backend/data/src/test/java/io/casehub/pages/data/DataQueryExceptionMapperTest.java` — test pattern for mapper
- `backend/data/src/test/java/io/casehub/pages/data/DataResourceQueryTest.java` — test pattern for resource (QuarkusTest + Alternative)
- GitHub #594 — focal issue
- GitHub #592 — parent epic
- GitHub #593 — pages-sqlite dependency
