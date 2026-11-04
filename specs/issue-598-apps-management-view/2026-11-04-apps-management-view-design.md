# Apps Management View — Design Spec

**Issue:** casehubio/casehub-pages#598
**Date:** 2026-11-04
**Branch:** issue-598-apps-management-view

## Overview

A management view showing installed JSON apps with status, settings, and
lifecycle controls (start/stop). Builds on the AppRegistry and _apps
collection infrastructure from #596/#601.

## Scope

**In scope:**
- `AppManagementResource` — new JAX-RS resource for app listing and lifecycle
- Extended `_apps` collection schema with settings fields
- Lifecycle gating — disabled apps' collections return 404
- Cache eviction on lifecycle transitions
- `apps-management.page.yaml` — management page with tabs

**Out of scope:**
- Static app lifecycle (static apps are always running)
- PreferenceStore integration (settings stored in _apps documents)
- App installation from external sources (future)

## Architecture

### Layer 1: Extended _apps Schema

Add four fields to `APPS_SCHEMA` in `AppRegistry`:

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `enabled` | boolean | true | Lifecycle state — false means stopped |
| `displayName` | string | (none) | User-facing override of appId |
| `autoSeed` | boolean | true | Load seed data on startup |
| `maxDocuments` | number | -1 (unlimited) | Per-collection document limit (-1 = no limit) |

These fields are stored alongside existing _apps document fields (appId,
name, description, icon, order, route, labels, source). Schema-form can
render settings directly from the document schema.

### Layer 2: AppManagementResource

New `AppManagementResource.java` in `backend/data-json`:

```
@Path("/api/apps")
@Authenticated
@ApplicationScoped
```

#### GET /api/apps — List all apps with runtime status

Iterates `AppRegistry.getRegisteredApps()`. For each app, joins the
descriptor with the _apps collection document to produce:

```json
{
  "appId": "contacts",
  "name": "Contacts",
  "displayName": "Contact Manager",
  "description": "...",
  "icon": "contacts",
  "source": "static",
  "enabled": true,
  "collections": [
    { "name": "contacts", "documentCount": 42 }
  ],
  "collectionCount": 1,
  "totalDocuments": 42
}
```

Collection counts come from `JsonCollectionStore.countDocuments()`.
The `source` field distinguishes static (classpath) from dynamic
(user-created) apps.

#### GET /api/apps/{appId} — Single app detail

Same shape as above for one app. Returns 404 if appId not found.

#### POST /api/apps/{appId}/lifecycle — Start/stop

Accepts:
```json
{ "action": "start" | "stop" }
```

**Stop:**
1. Validate app is dynamic (`source != "static"`) — 409 if static
2. Set `enabled=false` on the _apps document
3. Invalidate `DataCacheService` entries for the app
4. Return the updated app status

**Start:**
1. Set `enabled=true` on the _apps document
2. If `autoSeed=true`, trigger `AppRegistry.ensureSeeded()` for each
   collection
3. Return the updated app status

### Layer 3: Lifecycle Gating

When a collection query arrives for a disabled app, the request must
return 404 before reaching the data store.

**Implementation:** Add an `isAppEnabled(String appId)` method to
`AppRegistry` that checks the _apps document's `enabled` field. Gate
in `CollectionResource` and `PageResource` — before processing any
request, check `appRegistry.isAppEnabled(appId)`. If false, return
404 with `{"error": "App is disabled: <appId>"}`.

This keeps the gating at the resource layer, not in the store — the
store remains unaware of lifecycle state, which keeps it simple and
testable.

The `AppCatalogueResource` (child-app CRUD) also checks this gate —
you can't manage child apps of a disabled parent.

### Layer 4: Cache Eviction on Stop

On `POST /api/apps/{appId}/lifecycle` with `action=stop`:

**Backend:**
- `DataCacheService.invalidate(storeId, "json:" + appId + "/*")` — evict
  all cached data for the app's collections

**Frontend (via response):**
- The lifecycle response includes a `cacheAction: "invalidate"` field
- The frontend management page can call `DynamicPageLoader.invalidateAll()`
  or targeted `invalidate(pageName)` for the affected app's pages

Note: Full frontend cache eviction requires either a push notification
(SSE/WebSocket) or polling. For the initial implementation, the
management page handles it locally — navigating away from a stopped
app's pages naturally evicts on next load attempt (404 → error boundary).

### Layer 5: apps-management.page.yaml

Single `.page.yaml` file with three pages (tabs), following the
helpdesk/contacts pattern:

#### Page 1: App List

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
      - field: enabled
        accumulate: COUNT
        condition: { field: enabled, eq: "true" }
        label: Running
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
```

#### Page 2: App Settings

```yaml
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
          maxDocuments: { type: integer, minimum: 0 }
        required: []
      excludeFields: [appId, name, description, icon, order, route, labels, source]
      labels:
        displayName: Display Name
        enabled: Enabled
        autoSeed: Auto-Seed Data
        maxDocuments: Max Documents (-1 = unlimited)
```

#### Page 3: Lifecycle Actions

A dedicated tab for lifecycle actions (start/stop) with visual feedback:

```yaml
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

### File Changes Summary

| File | Change |
|------|--------|
| `AppManagementResource.java` | **New** — GET /api/apps, GET /api/apps/{appId}, POST /api/apps/{appId}/lifecycle |
| `AppManagementResourceTest.java` | **New** — unit tests for all endpoints |
| `AppRegistry.java` | **Modify** — extend APPS_SCHEMA with 4 new fields, add `isAppEnabled()` method |
| `CollectionResource.java` | **Modify** — add lifecycle gate check |
| `PageResource.java` | **Modify** — add lifecycle gate check |
| `AppCatalogueResource.java` | **Modify** — add lifecycle gate check |
| `apps-management.page.yaml` | **New** — management page with 3 tabs |
| `examples/server/.../application.properties` | **Modify** — register AppManagementResource if needed |

### Testing Strategy

**Unit tests (AppManagementResourceTest):**
- GET /api/apps returns all registered apps with correct metadata
- GET /api/apps/{appId} returns single app detail
- POST lifecycle stop sets enabled=false, returns updated status
- POST lifecycle start sets enabled=true
- POST lifecycle on static app returns 409
- POST lifecycle with invalid action returns 400

**Lifecycle gating tests:**
- Collection query on disabled app returns 404
- Page query on disabled app returns 404
- Child-app CRUD on disabled parent returns 404
- Re-enabling app restores collection access

**Integration/E2E tests:**
- apps-management.page.yaml renders in gallery
- Table displays apps with correct columns
- Settings form saves changes
- Lifecycle action buttons work (if supported)

## References

- `AppRegistry.java:36-98` — existing app registration and schema management
- `AppCatalogueResource.java:35-221` — child-app CRUD pattern (model for new resource)
- `AppBootstrap.java` — static app seeding (source="static" convention)
- `DynamicPageLoader.ts:41-48` — cache invalidation methods
- `helpdesk.page.yaml` — reference for multi-tab management page pattern
- `platform.page.yaml` — existing App Catalogue page
- `CollectionResource.java` — collection query path for lifecycle gating
- casehubio/casehub-pages#596 — app descriptor parser (dependency, closed)
- casehubio/casehub-pages#601 — dynamic page management (dependency, closed)
