---
layout: post
title: "Pages Becomes an Application Runtime"
date: 2026-10-30
entry_type: article
subtype: diary
projects: [casehubio/casehub-pages]
tags: [architecture, pages, json-apps, design, sqlite, data-provider]
---

# Pages Becomes an Application Runtime

Every CaseHub application today requires a Quarkus project. Maven build, Java classes, deployment pipeline — even for an app that does nothing more than CRUD over a handful of JSON entities with some charts and a form. The helpdesk example in `casehub-examples` has 41 Java files. The domain model is four of them. The other 37 are infrastructure: REST endpoints, WebSocket plumbing, push observers, MCP tooling, demo bootstrapping.

I've been thinking about this for a while. The CaseHub APIs are largely the same across generic applications. What differs is the domain information — the shape of the data, the fields on the form, the columns in the table. For a class of simpler, less performance-sensitive apps, the entire compiled Java layer is overhead that buys nothing.

Pages already handles UI composition from YAML. Data pipelines bind to sources and transform with jq. Forms, charts, tables, selectors — all declarative. The only thing Pages couldn't do was persist data. Fix that, and Pages stops being a rendering engine and becomes an application runtime.

## What Already Existed

The surprise was how much of the foundation was already in place. Claude and I went through the codebase expecting to find a large gap between "renders pages" and "runs applications." The gap was narrower than either of us expected.

`MutableDataSource` — the TypeScript interface for CRUD — already existed in `pages-data`. Not as a stub. A full 188-line implementation called `mutableRestSource` with create, update, and delete dispatch. It was built for external API binding. Nobody had pointed it at a local datastore because no local datastore existed.

`DataProvider` — the backend SPI for pluggable data sources — already had implementations for SQL, Prometheus, and IoTDB. `DataResource` at `/api/dataset` already routed queries through `canHandle()`. Adding a JSON provider is the same pattern as `SqlDataProvider`.

`layout-sqlite` — the layout persistence module — already had SQLite in the dependency tree with HikariCP connection pooling and Flyway migrations. The database infrastructure was already built and tested. It just served one consumer.

The platform's `LabelPatternMatcher` already matched hierarchical paths with glob patterns. `LabelRule` already evaluated conditions against context maps and applied add/remove actions.

Taken individually, these are just components in a large codebase. Taken together, they're most of an application runtime waiting to be assembled.

## The Architecture

![Pages application stack — three tiers from full Quarkus to YAML-only](images/json-apps-stack.svg)

A JSON app is a directory of files. No build step. No deployment pipeline. No Java.

```yaml
# app.yaml
collections:
  tickets:
    fields:
      subject: string
      description: string
      category: { type: enum, values: [HARDWARE, SOFTWARE, ACCESS, OTHER] }
      priority: { type: enum, values: [LOW, MEDIUM, HIGH, URGENT] }
      status: { type: enum, values: [OPEN, TRIAGED, ASSIGNED, RESOLVED, CLOSED] }
      customerRef: string
      assigneeId: string
      resolution: string
```

That schema replaces four Java files — `Ticket.java`, `TicketStatus.java`, `TicketCategory.java`, `TicketPriority.java`. The `.page.yaml` files define dashboard metrics, table views, detail forms, creation forms, agent rosters — using exactly the same Pages components that already exist. The seed data is plain JSON.

![Before and after: 41 Java files become 4 YAML/JSON files](images/json-apps-before-after.svg)

## Inside the Data Layer

The backend is a new `data-json` module implementing the existing `DataProvider` SPI, backed by SQLite. The design has three layers, each independently testable.

![JsonDataProvider architecture — provider, store, translator layered on pages-sqlite](images/json-data-provider-architecture.svg)

`JsonDataProvider` is the CDI entry point — an `@ApplicationScoped` bean that matches any dataset ID starting with `json:`. It delegates CRUD operations to `JsonCollectionStore` and query translation to `JsonQueryTranslator`. The provider itself is thin: 120 lines of wiring and one interesting responsibility — flattening tree-shaped JSON documents into the tabular `DataSetResult` format that tables and charts already consume.

`JsonCollectionStore` owns the `json_documents` table. Storage key is `(tenant_id, app_id, collection, doc_id)` — multi-tenant from the start, because retrofitting tenancy later is always worse than carrying it forward. Documents are stored as JSON text; SQLite's `json_extract()` queries them without deserialisation at the storage layer.

Slug generation for document IDs uses the same `SlugUtils` that already exists in the data module. If a document has an `_id` field, that's the ID. Otherwise, slugify a configurable field (default: `title`). If two documents in the same collection produce the same slug — `same-title` collides with an existing `same-title` — the store appends `-2`, `-3`, up to `-99`. A UUID fallback catches documents with no sluggable field at all.

### Optimistic Concurrency

Every document carries a `_version` integer, starting at 1, incremented atomically on every write. Updates use `WHERE _version = ?` — if the row was modified between the read and the write, zero rows update and the store throws `VERSION_CONFLICT`. The REST layer exposes this through standard HTTP: `ETag` headers on responses, `If-Match` on PATCH requests.

```java
// The atomic update — _version is both the guard and the counter
String sql = """
    UPDATE json_documents
    SET data = ?, _version = _version + 1, updated_at = ?
    WHERE tenant_id = ? AND app_id = ? AND collection = ?
      AND doc_id = ? AND _version = ?""";
```

No locking, no transactions beyond the single statement. SQLite's WAL mode and busy-timeout handle contention. For the scale target of JSON apps — under 100K documents per collection, single-digit concurrent writers — this is the right trade-off.

### Merge Strategies

PATCH semantics are strategy-based. The same store supports three merge behaviours, selectable per request and eventually per collection via the app descriptor.

![MergeStrategy — three built-in strategies for PATCH updates](images/json-merge-strategies.svg)

The default is RFC 7396 — the industry standard for JSON Merge Patch. Shallow merge with one addition: explicit `null` deletes a field. A `SHALLOW` strategy replaces top-level fields without the null-means-delete convention. A `DEEP` strategy recursively merges nested objects while replacing arrays wholesale.

I went back and forth on whether to hard-code RFC 7396 or make it pluggable. The deciding factor: different collections genuinely want different behaviour. A flat ticket with status fields wants RFC 7396. A deeply nested configuration document with partial sub-object updates wants `DEEP`. A log-style append collection might want something entirely custom. The `MergeStrategy` interface is one method — `JsonNode merge(JsonNode existing, JsonNode patch)` — so the extensibility cost is essentially zero.

### Query Translation

The query path is where `DataProvider` earns its keep. Pages components already send `DataSetLookup` requests with filter, sort, and group operations. `JsonQueryTranslator` translates those into SQLite SQL using `json_extract()`:

```sql
-- A filter like "priority >= 3 AND status = 'open'" becomes:
SELECT data FROM json_documents
WHERE tenant_id = ? AND app_id = ? AND collection = ?
  AND CAST(json_extract(data, '$.priority') AS REAL) >= ?
  AND json_extract(data, '$.status') = ?
ORDER BY json_extract(data, '$.priority') DESC
```

The full `FilterExpression` tree is translated — `And`, `Or`, `Not`, numeric comparisons, string matching, date ranges, `IN` lists, `IS NULL`. Anything the translator doesn't recognise (an `Unresolved` expression with an unknown function) gets returned in `remainingOps` for the pipeline to apply in memory. At the scale of JSON apps this fallback is fine; at 100K documents, server-side filtering via `json_extract()` is the thing that keeps queries fast.

The result mapping deserialises each row's JSON, discovers all unique top-level keys across the result set, and builds `ColumnDef` entries dynamically. A document `{"title": "Bug Report", "priority": 3, "tags": ["ui", "regression"]}` becomes three columns: `title` (text), `priority` (text — everything is text in `DataSetResult`), `tags` (serialised JSON string). Tables and charts already know how to render this format.

## Label Trees Do Triple Duty

The platform already had label trees — hierarchical path-based structures with pattern matching. I'd been thinking about them for app organisation, but Claude surfaced two more uses during our exploration.

![Label trees serve organization, UI data, and navigation/ACL simultaneously](images/label-tree-triple-duty.svg)

**Organization.** `org/apps/helpdesk/collections/tickets` — the hierarchy structures apps, collections, and shared data. Org-level shared collections live under `org/shared/` and multiple apps reference them.

**UI data.** A label tree like `regions/europe/uk` directly populates a cascading dropdown. No separate JSON lookup file needed. The same structure that organises the data *is* the data for form fields.

**Navigation and ACL.** The tree drives the menu — render it, and users see their apps. Restrict label paths by role, and the menu becomes an access control boundary. `dashboards/finance/**` visible only to the finance team isn't a feature to build — it's a query against existing label infrastructure.

One structure, defined once, serving three purposes without duplication.

## Why This Changes the Barrier to Entry

The target user for JSON apps is someone who can write YAML and jq expressions but is not a Java developer. They want to define a data model, lay out a UI, and see it working — in seconds, not after a build cycle.

The current path to a CaseHub application: learn Java, learn Maven, learn Quarkus, scaffold a project, write model classes, write REST endpoints, write WebSocket observers, write push plumbing, configure deployment, deploy. If you want to change a field on a form, recompile and redeploy.

The JSON apps path: write `app.yaml`, write `.page.yaml`, drop in seed data. Edit the YAML, see the change. The iteration loop drops from minutes to seconds. The skill requirement drops from "Java developer" to "can write YAML."

This doesn't replace full Quarkus apps. Complex integrations, high-throughput systems, custom business logic that goes beyond jq expressions — those still need compiled Java. The point is that most apps don't start there, and many never get there. The helpdesk is a good example: ticket CRUD with status tracking, category filtering, priority sorting, a dashboard with metrics. None of that needs Java. All of it can be expressed declaratively.

## The Dashboard Problem Falls Out for Free

There was a separate long-standing gap: Pages are static files. No runtime creation, no dynamic organisation, no access control on views. Users coming from tools like Grafana expect to create dashboards at runtime, organise them into folders, control who sees what.

With the JSON app infrastructure, a page becomes a document in a collection. The `pages` collection stores `.page.yaml` content. Creating a dashboard is saving a document. Organising dashboards into folders is labeling with tree paths. Access control is restricting label path visibility by role. The entire dashboard lifecycle falls out of the same infrastructure that makes JSON apps work. It's not a separate feature — it's a use case of the same mechanism.

## Mosaic Layers On Top

Pages is CaseHub-agnostic. The JSON apps runtime stays in Pages. But Mosaic — the CaseHub integration layer — can layer case lifecycle on top of the same collections.

A "case app" is a JSON app where Mosaic registers case-aware forms in the same page slots, intercepts mutations to inject case lifecycle (create a document → open a case, label change → state transition), and enriches the navigation with case counts and SLA indicators. The Pages foundation exposes SPIs; Mosaic registers its implementations.

The helpdesk is the proof of concept for both layers. Phase 1 runs as a pure JSON app — ticket CRUD, manual status changes, charts and forms. Phase 2 adds the case engine — automatic triage, human task routing, notification on resolution. Same data, same UI components, different lifecycle semantics layered on by Mosaic.

## Where It Stands

The shared SQLite infrastructure is extracted and landed. The `data-json` module — `JsonDataProvider`, `JsonCollectionStore`, `JsonQueryTranslator`, and `MergeStrategy` — is implemented and tested. The `MutableDataProvider` SPI and `CollectionResource` REST endpoints are wired. Two of the nine child issues in the epic are closed; the datastore is the third, and the core of the critical path.

What's next is the app descriptor parser — the `app.yaml` file that declares collections, schemas, references between collections, and seed data. That's where collection-level configuration lands: which merge strategy, what validation constraints, which fields are required. After that, wiring the frontend's existing `mutableRestSource` to the new backend endpoints. The frontend pieces already exist — they just need a target.

I'm pausing the ops apps I was building on Quarkus. They're exactly the class of application that should be JSON apps instead. Building them on Quarkus now means rewriting them in a week when this lands. Better to invest the week in the foundation and build everything on it.
