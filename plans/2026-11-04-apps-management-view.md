# Apps Management View Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #598 — Apps management view
**Issue group:** #598

**Goal:** Add a management view showing installed JSON apps with status,
settings, and lifecycle controls (start/stop for dynamic apps).

**Architecture:** Extend the `_apps` collection schema with settings
fields (enabled, displayName, autoSeed, maxDocuments). New
`AppManagementResource` at `/api/apps` for listing and lifecycle.
Lifecycle gating in `PageResource` and `AppCatalogueResource` — disabled
apps' pages/child-apps return 404. Single `apps-management.page.yaml`
with tabs for list, settings, and lifecycle actions.

**Tech Stack:** Java 21 / Quarkus (JAX-RS, CDI), SQLite via
JsonCollectionStore, RestAssured for tests, YAML page definitions

## Global Constraints

- All web components use Lit (protocol: web-component-strategy)
- DatasetContract required for named datasets (protocol: dataset-contract)
- Static apps are always running — lifecycle controls apply to dynamic apps only
- Settings are stored as fields on _apps documents, not in PreferenceStore
- Stop = set enabled=false + evict caches; data preserved in SQLite
- Tests use QuarkusTest + RestAssured + `/dev/auth/login` pattern

---

## Batch 1: Schema Extension + App Management Endpoint

### Task 1: Extend _apps schema and add isAppEnabled

**Files:**
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java:54-67`
- Test: `backend/data-json/src/test/java/io/casehub/pages/data/json/AppRegistryTest.java`

**Interfaces:**
- Produces: `AppRegistry.isAppEnabled(String appId) → boolean` — checks the _apps document's `enabled` field. Returns `true` if no document found (static apps without _apps entries default to enabled).

- [ ] **Step 1: Write failing test for isAppEnabled**

In `AppRegistryTest.java`, add a test that checks `isAppEnabled` returns
true for a registered app, and false after its _apps document has
`enabled=false`.

```java
@Test
void isAppEnabled_returns_true_for_enabled_app() {
    assertTrue(appRegistry.isAppEnabled("contacts"));
}

@Test
void isAppEnabled_returns_true_for_unknown_app() {
    assertTrue(appRegistry.isAppEnabled("nonexistent-app-xyz"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl backend/data-json -Dtest=AppRegistryTest#isAppEnabled_returns_true_for_enabled_app -f backend/pom.xml`
Expected: FAIL — `isAppEnabled` method not found

- [ ] **Step 3: Extend APPS_SCHEMA and implement isAppEnabled**

In `AppRegistry.java`, add four fields to `APPS_SCHEMA` (line 54-67):

```java
static final CollectionSchema APPS_SCHEMA = new CollectionSchema("_apps",
    Map.of(
        "appId", "string",
        "name", "string",
        "description", "string",
        "icon", "string",
        "order", "number",
        "route", "string",
        "labels", "array",
        "source", "string",
        "enabled", "boolean",
        "displayName", "string",
        "autoSeed", "boolean",
        "maxDocuments", "number"
    ),
    Map.of("order", (Object) 0,
           "enabled", (Object) true,
           "autoSeed", (Object) true,
           "maxDocuments", (Object) -1),
    Map.of()
);
```

Add `isAppEnabled` method:

```java
public boolean isAppEnabled(String appId) {
    DataContext ctx = new DataContext(TenancyConstants.PLATFORM_TENANT_ID);
    return store.get(ctx, TenancyConstants.PLATFORM_TENANT_ID, "_apps", appId)
        .map(doc -> !doc.has("enabled") || doc.get("enabled").asBoolean(true))
        .orElse(true);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl backend/data-json -Dtest=AppRegistryTest -f backend/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java
git add backend/data-json/src/test/java/io/casehub/pages/data/json/AppRegistryTest.java
git commit -m "feat(#598): extend _apps schema + isAppEnabled  Refs #598"
```

### Task 2: AppManagementResource — list and detail endpoints

**Files:**
- Create: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppManagementResource.java`
- Test: `backend/data-json/src/test/java/io/casehub/pages/data/json/AppManagementResourceTest.java`

**Interfaces:**
- Consumes: `AppRegistry.getRegisteredApps() → Set<String>`, `AppRegistry.getDescriptor(String) → Optional<AppDescriptor>`, `JsonCollectionStore.countDocuments(String, String, String) → int`, `AppRegistry.getStore() → JsonCollectionStore`, `JsonCollectionStore.get(DataContext, String, String, String) → Optional<JsonNode>`
- Produces: `GET /api/apps` returns JSON array of app objects with appId, name, displayName, description, icon, source, enabled, collectionCount, totalDocuments. `GET /api/apps/{appId}` returns single app object.

- [ ] **Step 1: Write failing tests**

Create `AppManagementResourceTest.java` following the `AppCatalogueResourceTest` pattern:

```java
package io.casehub.pages.data.json;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class AppManagementResourceTest {

    private String token;

    @BeforeEach
    void login() {
        token = given()
            .contentType(ContentType.JSON)
            .body(Map.of("name", "admin", "roles", List.of("admin", "developer")))
            .post("/dev/auth/login")
            .then().statusCode(200)
            .extract().path("token");
    }

    @Test
    void list_returns_all_registered_apps() {
        given()
            .header("Authorization", "Bearer " + token)
            .get("/api/apps")
            .then().statusCode(200)
            .body("size()", greaterThanOrEqualTo(1))
            .body("[0].appId", notNullValue())
            .body("[0].enabled", notNullValue());
    }

    @Test
    void list_includes_collection_counts() {
        given()
            .header("Authorization", "Bearer " + token)
            .get("/api/apps")
            .then().statusCode(200)
            .body("find { it.appId == 'contacts' }.collectionCount",
                  greaterThanOrEqualTo(1));
    }

    @Test
    void get_single_app() {
        given()
            .header("Authorization", "Bearer " + token)
            .get("/api/apps/contacts")
            .then().statusCode(200)
            .body("appId", equalTo("contacts"))
            .body("enabled", equalTo(true))
            .body("source", equalTo("static"));
    }

    @Test
    void get_unknown_app_returns_404() {
        given()
            .header("Authorization", "Bearer " + token)
            .get("/api/apps/nonexistent-app-xyz")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl backend/data-json -Dtest=AppManagementResourceTest -f backend/pom.xml`
Expected: FAIL — 404 on all endpoints (no resource registered)

- [ ] **Step 3: Implement AppManagementResource**

Create `AppManagementResource.java`:

```java
package io.casehub.pages.data.json;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.pages.data.DataContext;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.security.Authenticated;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.Map;

@Path("/api/apps")
@Authenticated
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class AppManagementResource {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final String STORE_ID = TenancyConstants.PLATFORM_TENANT_ID;

    @Inject
    AppRegistry appRegistry;

    @GET
    public Response list() {
        ArrayNode result = MAPPER.createArrayNode();
        DataContext ctx = new DataContext(STORE_ID);
        JsonCollectionStore store = appRegistry.getStore();

        for (String appId : appRegistry.getRegisteredApps()) {
            result.add(buildAppInfo(ctx, store, appId));
        }
        return Response.ok(result).build();
    }

    @GET
    @Path("/{appId}")
    public Response get(@PathParam("appId") String appId) {
        if (!appRegistry.getRegisteredApps().contains(appId)) {
            return Response.status(404)
                .entity(Map.of("error", "App not found: " + appId)).build();
        }
        DataContext ctx = new DataContext(STORE_ID);
        JsonCollectionStore store = appRegistry.getStore();
        return Response.ok(buildAppInfo(ctx, store, appId)).build();
    }

    private ObjectNode buildAppInfo(DataContext ctx,
                                     JsonCollectionStore store,
                                     String appId) {
        ObjectNode info = MAPPER.createObjectNode();
        info.put("appId", appId);

        var descriptor = appRegistry.getDescriptor(appId).orElse(null);
        var appsDoc = store.get(ctx, STORE_ID, "_apps", appId).orElse(null);

        if (appsDoc != null) {
            copyIfPresent(appsDoc, info, "name");
            copyIfPresent(appsDoc, info, "displayName");
            copyIfPresent(appsDoc, info, "description");
            copyIfPresent(appsDoc, info, "icon");
            copyIfPresent(appsDoc, info, "source");
            info.put("enabled",
                !appsDoc.has("enabled") || appsDoc.get("enabled").asBoolean(true));
        } else {
            info.put("enabled", true);
        }

        int collectionCount = 0;
        int totalDocuments = 0;
        if (descriptor != null) {
            collectionCount = descriptor.collections().size();
            for (var col : descriptor.collections()) {
                totalDocuments += store.countDocuments(
                    STORE_ID, STORE_ID, col.name());
            }
        }
        info.put("collectionCount", collectionCount);
        info.put("totalDocuments", totalDocuments);

        return info;
    }

    private void copyIfPresent(JsonNode source, ObjectNode target,
                                String field) {
        if (source.has(field) && !source.get(field).isNull()) {
            target.set(field, source.get(field));
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl backend/data-json -Dtest=AppManagementResourceTest -f backend/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/data-json/src/main/java/io/casehub/pages/data/json/AppManagementResource.java
git add backend/data-json/src/test/java/io/casehub/pages/data/json/AppManagementResourceTest.java
git commit -m "feat(#598): AppManagementResource — list and detail endpoints  Refs #598"
```

### Task 3: Lifecycle endpoint — start/stop with cache eviction

**Files:**
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppManagementResource.java`
- Modify: `backend/data-json/src/test/java/io/casehub/pages/data/json/AppManagementResourceTest.java`

**Interfaces:**
- Consumes: `AppRegistry.isAppEnabled(String) → boolean`, `AppRegistry.ensureSeeded(DataContext, String, String)`, `DataCacheService.invalidate(String, String)`, `JsonCollectionStore.update(DataContext, String, String, String, ObjectNode, OptionalInt, MergeStrategy)`
- Produces: `POST /api/apps/{appId}/lifecycle` accepting `{"action":"start"|"stop"}`. Returns updated app info. 409 if static app. 400 if invalid action.

- [ ] **Step 1: Write failing tests for lifecycle**

Add to `AppManagementResourceTest.java`:

```java
@Test
void stop_dynamic_app() {
    String appId = "lifecycle-stop-" + System.nanoTime();
    createDynamicApp(appId);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "stop"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(200)
        .body("enabled", equalTo(false));
}

@Test
void start_stopped_app() {
    String appId = "lifecycle-start-" + System.nanoTime();
    createDynamicApp(appId);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "stop"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(200);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "start"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(200)
        .body("enabled", equalTo(true));
}

@Test
void lifecycle_rejects_static_app() {
    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "stop"))
        .post("/api/apps/contacts/lifecycle")
        .then().statusCode(409)
        .body("error", containsString("static"));
}

@Test
void lifecycle_rejects_invalid_action() {
    String appId = "lifecycle-invalid-" + System.nanoTime();
    createDynamicApp(appId);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "restart"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(400);
}

@Test
void lifecycle_unknown_app_returns_404() {
    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "stop"))
        .post("/api/apps/nonexistent-xyz/lifecycle")
        .then().statusCode(404);
}

private void createDynamicApp(String appId) {
    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("appId", appId, "name", "Test " + appId,
                      "route", "/" + appId))
        .post("/api/apps/platform/apps")
        .then().statusCode(201);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl backend/data-json -Dtest=AppManagementResourceTest#stop_dynamic_app -f backend/pom.xml`
Expected: FAIL — 405 Method Not Allowed (no POST handler)

- [ ] **Step 3: Implement lifecycle endpoint**

Add to `AppManagementResource.java`:

```java
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import io.casehub.pages.data.DataCacheService;
import java.util.OptionalInt;
import java.util.Set;

// Add field:
@Inject
DataCacheService cacheService;

@POST
@Path("/{appId}/lifecycle")
@Consumes(MediaType.APPLICATION_JSON)
public Response lifecycle(@PathParam("appId") String appId, JsonNode body) {
    if (body == null || !body.has("action")) {
        return Response.status(400)
            .entity(Map.of("error", "Missing 'action' field")).build();
    }

    String action = body.get("action").asText();
    if (!Set.of("start", "stop").contains(action)) {
        return Response.status(400)
            .entity(Map.of("error", "Invalid action: " + action
                           + ". Must be 'start' or 'stop'")).build();
    }

    DataContext ctx = new DataContext(STORE_ID);
    JsonCollectionStore store = appRegistry.getStore();

    var appsDoc = store.get(ctx, STORE_ID, "_apps", appId).orElse(null);
    if (appsDoc == null) {
        return Response.status(404)
            .entity(Map.of("error", "App not found: " + appId)).build();
    }

    if (appsDoc.has("source")
        && "static".equals(appsDoc.get("source").asText())) {
        return Response.status(409)
            .entity(Map.of("error",
                "Cannot change lifecycle of static app: " + appId)).build();
    }

    boolean enable = "start".equals(action);
    ObjectNode patch = MAPPER.createObjectNode();
    patch.put("enabled", enable);
    store.update(ctx, STORE_ID, "_apps", appId, patch,
                 OptionalInt.empty(), MergeStrategy.RFC_7396);

    if (!enable) {
        cacheService.invalidate(STORE_ID, "json:" + appId);
    }

    if (enable) {
        var descriptor = appRegistry.getDescriptor(appId).orElse(null);
        if (descriptor != null) {
            boolean autoSeed = !appsDoc.has("autoSeed")
                || appsDoc.get("autoSeed").asBoolean(true);
            if (autoSeed) {
                for (var col : descriptor.collections()) {
                    appRegistry.ensureSeeded(ctx, appId, col.name());
                }
            }
        }
    }

    return Response.ok(buildAppInfo(ctx, store, appId)).build();
}
```

- [ ] **Step 4: Run all tests to verify they pass**

Run: `mvn test -pl backend/data-json -Dtest=AppManagementResourceTest -f backend/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/data-json/src/main/java/io/casehub/pages/data/json/AppManagementResource.java
git add backend/data-json/src/test/java/io/casehub/pages/data/json/AppManagementResourceTest.java
git commit -m "feat(#598): lifecycle endpoint — start/stop with cache eviction  Refs #598"
```

## Batch 2: Lifecycle Gating + Management Page

### Task 4: Lifecycle gate in PageResource and AppCatalogueResource

**Files:**
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/PageResource.java:55-60`
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppCatalogueResource.java:56-78`
- Test: `backend/data-json/src/test/java/io/casehub/pages/data/json/AppManagementResourceTest.java` (add gating tests)

**Interfaces:**
- Consumes: `AppRegistry.isAppEnabled(String) → boolean`
- Produces: 404 response with `{"error": "App is disabled: <appId>"}` when a disabled app's pages or child-apps are accessed

Note: `CollectionResource` is in `backend/data` (different module), does
not inject `AppRegistry`, and is not gated. Collection queries bypass the
lifecycle gate. This is acceptable — app data is accessed through pages,
not by hitting `/api/collections/` directly.

- [ ] **Step 1: Write failing tests for lifecycle gating**

Add to `AppManagementResourceTest.java`:

```java
@Test
void disabled_app_pages_return_404() {
    String appId = "gated-pages-" + System.nanoTime();
    createDynamicApp(appId);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "stop"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(200);

    given()
        .header("Authorization", "Bearer " + token)
        .get("/api/apps/" + appId + "/pages")
        .then().statusCode(404)
        .body("error", containsString("disabled"));
}

@Test
void disabled_app_child_apps_return_404() {
    String appId = "gated-catalogue-" + System.nanoTime();
    createDynamicApp(appId);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "stop"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(200);

    given()
        .header("Authorization", "Bearer " + token)
        .get("/api/apps/" + appId + "/apps")
        .then().statusCode(404)
        .body("error", containsString("disabled"));
}

@Test
void reenabled_app_pages_return_200() {
    String appId = "gated-reenable-" + System.nanoTime();
    createDynamicApp(appId);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "stop"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(200);

    given()
        .contentType(ContentType.JSON)
        .header("Authorization", "Bearer " + token)
        .body(Map.of("action", "start"))
        .post("/api/apps/" + appId + "/lifecycle")
        .then().statusCode(200);

    given()
        .header("Authorization", "Bearer " + token)
        .get("/api/apps/" + appId + "/pages")
        .then().statusCode(200);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl backend/data-json -Dtest=AppManagementResourceTest#disabled_app_pages_return_404 -f backend/pom.xml`
Expected: FAIL — returns 200 instead of 404 (no gating yet)

- [ ] **Step 3: Add lifecycle gate to PageResource**

In `PageResource.java`, add gate check at the start of each endpoint
method. Add this private method and call it at the top of `list()`,
`get()`, `create()`, `update()`, `delete()`:

```java
private Response checkAppEnabled(String appId) {
    if (!appRegistry.isAppEnabled(appId)) {
        return Response.status(404)
            .entity(Map.of("error", "App is disabled: " + appId)).build();
    }
    return null;
}
```

In each method, add at the top (after `@PathParam` extraction):
```java
Response gate = checkAppEnabled(appId);
if (gate != null) return gate;
```

- [ ] **Step 4: Add lifecycle gate to AppCatalogueResource**

Same pattern in `AppCatalogueResource.java` — add `checkAppEnabled`
method and gate check at the top of `list()`, `get()`, `create()`,
`update()`, `delete()`.

- [ ] **Step 5: Run all tests**

Run: `mvn test -pl backend/data-json -Dtest=AppManagementResourceTest -f backend/pom.xml`
Expected: PASS

Also run existing tests to verify no regressions:
Run: `mvn test -pl backend/data-json -Dtest=PageResourceTest,AppCatalogueResourceTest -f backend/pom.xml`
Expected: PASS (static apps are always enabled, so existing tests unaffected)

- [ ] **Step 6: Commit**

```bash
git add backend/data-json/src/main/java/io/casehub/pages/data/json/PageResource.java
git add backend/data-json/src/main/java/io/casehub/pages/data/json/AppCatalogueResource.java
git add backend/data-json/src/test/java/io/casehub/pages/data/json/AppManagementResourceTest.java
git commit -m "feat(#598): lifecycle gating — disabled apps return 404  Refs #598"
```

### Task 5: apps-management.page.yaml

**Files:**
- Create: `examples/samples/Domain Showcases/apps-management/apps-management.page.yaml`
- Create: `examples/server/src/main/resources/org/apps/platform/apps-management.page.yaml`
- Create: `backend/data-json/src/test/resources/org/apps/platform/apps-management.page.yaml`

**Interfaces:**
- Consumes: `GET /api/apps` (dataset), `POST /api/apps/{appId}/lifecycle` (action buttons), `PATCH /api/apps/{appId}/apps/{appId}` (settings save via REST adapter)

- [ ] **Step 1: Create apps-management.page.yaml for gallery**

Create `examples/samples/Domain Showcases/apps-management/apps-management.page.yaml`:

```yaml
datasets:
- uuid: apps
  url: /api/apps
  columns:
  - id: appId
    type: TEXT
  - id: name
    type: TEXT
  - id: displayName
    type: TEXT
  - id: source
    type: LABEL
  - id: enabled
    type: LABEL
  - id: collectionCount
    type: NUMBER
  - id: totalDocuments
    type: NUMBER

pages:
- name: Apps
  components:
  - title: App Management
  - type: metric
    properties:
      lookup:
        uuid: apps
      columns:
      - field: appId
        accumulate: COUNT
        label: Total Apps
  - type: data-table
    properties:
      pageSize: 20
      sortable: true
      filter:
        enabled: true
      lookup:
        uuid: apps
      columns:
      - field: appId
        label: ID
      - field: name
        label: Name
      - field: source
        label: Source
      - field: enabled
        label: Status
      - field: collectionCount
        label: Collections
      - field: totalDocuments
        label: Documents
  - page: App Settings
  - page: Lifecycle

- name: App Settings
  dataScope:
    dataset: apps
    idColumn: appId
  save:
    trigger: auto
    delay: 1500
    adapter: rest
  components:
  - title: "#{appId} Settings"
  - schema-form:
      schema:
        properties:
          displayName: { type: string }
          enabled: { type: boolean }
          autoSeed: { type: boolean }
          maxDocuments: { type: integer, minimum: -1 }
        required: []
      excludeFields: [appId, name, description, icon, order, route, labels, source, collectionCount, totalDocuments]
      labels:
        displayName: Display Name
        enabled: Enabled
        autoSeed: Auto-Seed Data
        maxDocuments: Max Documents (-1 = unlimited)

- name: Lifecycle
  dataScope:
    dataset: apps
    idColumn: appId
  components:
  - title: "#{appId} — Lifecycle"
  - type: markdown
    properties:
      content: |
        Current status: **#{enabled}**
  - type: action-button
    properties:
      label: Stop App
      style: danger
      url: /api/apps/#{appId}/lifecycle
      method: POST
      body: { action: stop }
      confirm: "Stop app #{appId}? Collections will return 404."
      disabledWhen: "#{enabled} == false"
      onSuccess:
        refresh: [apps]
        message: "App #{appId} stopped"
  - type: action-button
    properties:
      label: Start App
      style: primary
      url: /api/apps/#{appId}/lifecycle
      method: POST
      body: { action: start }
      disabledWhen: "#{enabled} == true"
      onSuccess:
        refresh: [apps]
        message: "App #{appId} started"
```

- [ ] **Step 2: Copy to server and test resources**

Copy the same file to:
- `examples/server/src/main/resources/org/apps/platform/apps-management.page.yaml`
- `backend/data-json/src/test/resources/org/apps/platform/apps-management.page.yaml`

These are the paths where the server and tests pick up page YAML for the
platform app. `PageBootstrap.seed()` scans `*.page.yaml` in each app dir.

- [ ] **Step 3: Check example server application.properties**

The example server excludes several auth-gated resources at line 14 of
`examples/server/src/main/resources/application.properties`:
```
quarkus.arc.exclude-types=...,io.casehub.pages.data.json.PageResource,io.casehub.pages.data.json.AppCatalogueResource
```

Add `AppManagementResource` to this exclude list if the example server
doesn't provide its own auth layer for these endpoints. If it does use
dev auth, leave it un-excluded so the management page works in the
server-connected gallery tab.

- [ ] **Step 4: Verify gallery sample generation picks it up**

Run: `node examples/scripts/generate-samples.js`

Check that `examples/samples.json` includes an entry under "Domain
Showcases" for `apps-management`.

- [ ] **Step 5: Run backend tests**

Run: `mvn test -pl backend/data-json -f backend/pom.xml`
Expected: PASS — the new page YAML is valid and picked up by PageBootstrap

- [ ] **Step 6: Commit**

```bash
git add "examples/samples/Domain Showcases/apps-management/"
git add examples/server/src/main/resources/org/apps/platform/apps-management.page.yaml
git add backend/data-json/src/test/resources/org/apps/platform/apps-management.page.yaml
git commit -m "feat(#598): apps-management.page.yaml — list, settings, lifecycle tabs  Refs #598"
```

## References

- [2026-11-04-apps-management-view-design.md] — design spec this plan implements
- `AppRegistry.java:36-98` — app registration, schema management, getRegisteredApps/getDescriptor API
- `AppCatalogueResource.java:35-221` — child-app CRUD pattern (model for new resource)
- `AppCatalogueResourceTest.java:17-170` — QuarkusTest + RestAssured test pattern
- `PageResource.java:35-60` — page CRUD resource, lifecycle gate target
- `CollectionResource.java:29-60` — in `backend/data` (different module, NOT gated)
- `JsonCollectionStore.java:191-204` — countDocuments method
- `ActionButtonProps` in `pages-component/src/model/action-types.ts:7-19` — action-button supports POST, body, confirm, onSuccess.refresh
- `helpdesk.page.yaml` — reference multi-tab management page pattern
- casehubio/casehub-pages#596 — app descriptor parser (closed)
- casehubio/casehub-pages#601 — dynamic page management (closed)
