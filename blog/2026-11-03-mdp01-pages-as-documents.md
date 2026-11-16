---
layout: post
title: "Pages as Documents — When Your App Framework Eats Itself"
date: 2026-11-03
entry_type: article
subtype: diary
projects: [casehubio/casehub-pages]
tags: [json-app, architecture, composition, dynamic-pages, acl, casehub]
series: issue-601-dynamic-page-management
---

# Pages as Documents — When Your App Framework Eats Itself

Two days ago the helpdesk app was four YAML files replacing forty-one Java files. That was satisfying, but it was still static. The pages shipped with the binary — change a dashboard, redeploy the server. The app descriptor and its collections were dynamic, the pages were not. That asymmetry bothered me.

The fix turned out to be the kind of idea that sounds obvious in hindsight and reveals unexpected depth when you build it: store pages as documents in a collection. The same collection infrastructure that holds tickets and agents can hold page YAML. A page is just a document with a `yaml` field.

But the interesting part isn't the storage. It's what happens to the concept of an "app" once pages are documents — and why that matters for anyone building application platforms.

## The redeployment tax

Every platform I've worked with treats UI layouts as code artefacts. Dashboards, forms, reports — they ship with the application binary. Want to add a chart to the operations dashboard? Write the layout, run the build, deploy to staging, test, promote to production. The change itself takes five minutes. The deployment ceremony takes hours.

For engineering teams this is tolerable. For business users who want to arrange their own data views — filter by their region, add a metric their team cares about, hide columns they never use — it's a wall. They file a ticket and wait.

The conventional answer is a drag-and-drop builder. But builders create a second system: the coded layouts and the built layouts, with different capabilities, different storage, different deployment models. Now you maintain both.

I wanted one system. Pages are documents. Documents live in collections. Collections have CRUD APIs, access control, and schema validation. No builder needed, no second system — just the same YAML that developers write, stored where users can create and modify it at runtime.

## Apps all the way down

An app in CaseHub is a self-contained unit: a root page, child pages, collections, a navigation tree, and declared input parameters. The helpdesk is an app. A contacts manager is an app. A task tracker is an app.

The parent web application — the thing you actually open in a browser — is also an app. It has its own root page (a catalogue), its own collections (the app registry), its own navigation. It happens to contain other apps as sub-navigation points, but structurally it's identical to any of them.

![App composition model](images/apps-composition-model.svg)

Any app can be the root. Deploy the helpdesk standalone and it IS the web application — no parent needed. Nest it inside a larger platform and it becomes one tab in a catalogue. The app doesn't know or care. Its pages resolve parameters from URL query params when root, from parent-provided bindings when nested. Same `#{params.ticketId}` template syntax either way.

This recursive model has a practical consequence: you don't design "the platform" and "the apps" as separate things. There is no platform layer with special powers. The catalogue that manages apps is itself a page that queries a collection. Adding a new app to the catalogue is creating a document. Removing one is deleting a document. The same API, the same access control, the same validation.

## The `_pages` collection

Every app gets an implicit `_pages` collection backed by the same SQLite JSONB store that handles user data. No declaration needed in `app.yaml` — the `AppRegistry` registers a hardcoded schema at startup.

Static `.page.yaml` files from the classpath are seeded into this collection at boot. Each page is individually hashed with SHA-256. On redeployment, the bootstrap compares hashes per page — changed source content gets updated, unchanged pages are skipped, dynamic pages are never touched.

![Page lifecycle — static bootstrap and dynamic creation](images/page-lifecycle-flow.svg)

This hash comparison solves a problem that catches every content-management system eventually: the silent upgrade failure. A developer modifies a static page in code, deploys the new version, and the old page content persists in the database because the seed mechanism skipped it ("already exists, nothing to do"). With per-page hashing, the bootstrap detects the source change and updates the stored content. No manual database intervention. No "have you cleared the cache?" support tickets.

System collections use an underscore prefix — `_pages`, `_apps` — and the data provider's `canHandle()` method rejects these namespaces outright for the generic collection endpoint. All access goes through dedicated resources. This isn't defence in depth — it's a single, hard enforcement point. Claude caught during the audit that the original design had a bypass: the generic `CollectionResource` could reach system collections through the same data provider. The fix was to reject system collection namespaces at the provider level, making the bypass structurally impossible rather than policy-forbidden.

## Validation as a security boundary

Dynamic pages are user-authored content. That means they're an attack surface. The YAML validation runs on every write and works in three layers.

**Root-level allow-list.** Only known page structure keys are permitted at the top level. Unknown keys are rejected outright — not logged, not warned, rejected. This catches yaml-core expansion keys (`variables`, `modules`, `imports`) that would enable Turing-complete template expansion and external module imports. Both are SSRF vectors in user-authored content.

**Recursive XSS scan.** The tree walk rejects `script` and `innerHTML` at any nesting level, plus any key matching the `on*` event handler pattern. This is the interesting bit — the original implementation used a deny-list of twenty-three specific event handlers. Claude's audit pointed out that HTML has sixty-plus event handler attributes and the list grows with every browser specification. Pattern matching (`on` + lowercase letter) catches `onpointerrawupdate` and every future handler without maintenance.

**Dataset semantic validation.** If the page declares datasets, any entry with a `url`, `content`, or `serverQuery` key is rejected. Dynamic pages can only reference the app's declared collection bindings, not arbitrary external endpoints. A user-authored page can display and filter the helpdesk's ticket data — it cannot make the server fetch from `https://evil.com/exfiltrate`.

## Data-driven access control

I wanted access control that lives in data, not in Java annotations. The `@RolesAllowed("admin")` pattern compiles permissions into the binary. Change who can edit pages and you redeploy the server. That's the redeployment tax again — now applied to security policy.

```yaml
access:
  read:
    roles: [authenticated]
  write:
    roles: [admin, developer]
```

![Access control — data-driven, not annotated](images/access-control-data-driven.svg)

The `PageResource` extracts roles from the JWT, loads the app's `AppAccessPolicy` from the descriptor, and evaluates `isSatisfiedBy()` on every request. The special role `authenticated` always passes for any authenticated user. Empty roles means allow all. The same `AccessControl` shape already exists at the component level in the frontend; this lifts it to the app and page level with matching Java types.

The practical value: a platform operator can grant page authoring to a specific team by editing `app.yaml`. No code change. No redeployment. No involvement from engineering. The security boundary moves from compiled code to configuration — auditable, changeable, reversible.

## What the bugs taught

The audit found three critical issues. Each one taught something about integration testing.

**Tenant scoping.** Pages are platform-level resources — they belong to the app, not to a tenant. The initial implementation used the JWT's tenant ID for store operations. In dev mode that's `"dev"`, but the bootstrap seeds data under `"platform"`. Result: seeded pages were invisible to the API. Every test passed because the tests created their own data — they never checked that bootstrap-seeded pages were actually reachable. The fix was straightforward (use the platform tenant ID for page store operations), but the gap in the test suite was the real lesson. Integration tests that only test what they create will miss every data-lifecycle mismatch.

**Slug collision.** The system slugifies page names for document IDs — "Dashboard" becomes "dashboard", "Ticket Queue" becomes "ticket-queue". The name collision check was comparing the raw name against document IDs. A POST with `name: "Dashboard"` wouldn't match the seeded document ID "dashboard", bypassing the static page protection entirely. The fix was a single `SlugUtils.slugify()` call. The lesson: whenever two representations of the same identity exist (display name vs. document ID), the collision check must normalise both.

**Deny-list vs. allow-list.** The spec said allow-list. The first implementation used a deny-list. Both pass the same tests — every listed XSS vector is caught. The difference only shows up for vectors not yet listed. The event handler test uses `onpointerrawupdate` specifically to prove the pattern-based approach catches what an enumeration would miss. Security tests should test the boundary of the defence, not just the known attacks.

## The memory question

Caching without eviction means memory grows with every page the user visits. In a catalogue with dozens of apps, each with multiple pages, that's a slow leak toward an out-of-memory error that only appears after extended use — the kind of bug that passes every test suite and fails in production.

![Memory lifecycle — navigate, cache, evict](images/memory-lifecycle-lru.svg)

The `LruCache<K,V>` solves this with two mechanisms. First, size-based eviction: keep the N most recently accessed pages warm, evict the least recently used when a new one arrives. Flick between Helpdesk and Contacts and both are instant. Visit a fifth app and the oldest cache entry is dropped. Second, time-based expiry: a configurable TTL ensures stale pages are re-fetched even if the cache isn't full. A page edited by another user eventually refreshes without manual invalidation.

The implementation avoids the classic LRU's doubly-linked list. ES2015 guarantees that `Map` iterates in insertion order. Deleting a key and re-inserting it on access moves it to the end — the "most recently used" position. The first key from `keys().next()` is always the least recently used. Three lines of logic replace an entire data structure.

There's a subtlety beyond the page cache. An app's lifecycle includes dataset subscriptions, context managers, and DOM subtrees. Evicting a page from the cache is cheap; tearing down an entire app's resource graph requires explicit disposal. The plan is a two-tier model: the LRU cache handles page component trees, and a separate app lifecycle manager handles resource cleanup with a grace period. Navigate away and the resources stay live for a few seconds — just long enough for a back-navigation to be free. After the grace period, full teardown.

## Wiring it together

The building blocks — API client, page loader, LRU cache, URL param extraction, error boundary — all had unit tests. None of them were connected to the running application. `loadSite()`, the 1500-line function that orchestrates everything from theme application to event delegation to data pipeline wiring, didn't know dynamic pages existed.

The integration works through a single new option: `appId`. When present, `loadSite()` creates a `PageApiClient` with the configured fetch function and auth token, wraps it in a `DynamicPageLoader` with the site's global defaults, and fetches the page index. Each dynamic page becomes a placeholder in the navigation index — discoverable but not yet loaded.

![Runtime navigation — static vs dynamic page resolution](images/dynamic-page-navigation.svg)

The interesting design decision was how to render a dynamic page without destroying the static one. Three options: replace the entire DOM (destructive — you lose the static page and have to re-render it on back-navigation), inject into the existing navigation container (fragile — requires knowing the container structure), or toggle visibility between static and dynamic content.

The toggle approach won. Static content stays in the DOM but hides. A `[data-dynamic-page]` container receives the fetched and parsed page. Navigate back to a static page and the containers swap — static reappears instantly, dynamic clears. No re-rendering, no re-fetching, no flash of empty content. The user sees a seamless transition in both directions.

Fetch failures render an error boundary scoped to the dynamic container. The static pages keep working. A broken dynamic page doesn't take down the application.

## The showcase: apps within apps

The recursive model stayed theoretical until I built three apps alongside the helpdesk and composed them inside a fourth. The contacts app manages people — eight contacts across four categories with a company breakdown chart. The tasks app tracks work items and meeting notes — two collections, five pages, metrics by priority and assignee. Neither knows it will be nested inside something larger.

The platform app is the container. Its `app.yaml` declares no collections of its own. Its page YAML defines a catalogue page — a data-table bound to the `_apps` collection, a metric card counting available apps, and a form for registering new ones. When the server starts, `AppBootstrap` discovers all four apps on the classpath and seeds an entry for each into `_apps`:

```yaml
# Auto-seeded at startup
appId: helpdesk
name: Helpdesk
description: "2 collections: tickets, agents"
route: /helpdesk
source: static
```

The catalogue page reads from `_apps` the same way the helpdesk reads from `tickets`. Same collection API, same data-table component, same filter and sort operations. The data-table's mutable configuration points at the `AppCatalogueResource` — add an app via the form, it appears in the table. Delete it, it disappears. Static apps (discovered from the classpath) are protected — the API returns 409 on delete.

![App catalogue lifecycle — bootstrap, runtime CRUD, and protection](images/catalogue-flow.svg)

The `AppCatalogueResource` mirrors `PageResource` exactly. Same access control model (`AppAccessPolicy.isSatisfiedBy`), same error response format, same shared utilities (`ResourceUtils.matchesLabel`, `copyIfPresent`), same schema validation, same cache invalidation. The only difference is the collection name (`_apps` vs `_pages`) and the document shape (appId/route instead of yaml/sourceHash). Once the pattern existed for pages, applying it to apps was mechanical.

This is the practical payoff of the recursive architecture. The `_apps` collection didn't require new infrastructure — it used the same `CollectionSchema` registration, the same `JsonCollectionStore`, the same system collection isolation (`canHandle()` rejects `json:*/_*`). The `AppBootstrap` followed the same seed-if-absent pattern as `PageBootstrap`. The e2e test followed the same REST-assured patterns as `PageResourceE2ETest`. Build it once for pages, get it free for apps, get it free for whatever `_` collection comes next.

## What the full audit revealed

The code review after building the showcase found eleven issues. The three most instructive:

**Case-insensitive event handlers.** The YAML validator's `on*` pattern matched `onclick` but not `onClick` or `ONCLICK`. Browsers treat event attributes case-insensitively — `setAttribute("onClick", "alert(1)")` fires. The fix was `key.toLowerCase()` before matching. The lesson: when your validation enforces a constraint the browser doesn't enforce, test the boundary the browser uses, not the one your language uses.

**Navigation race condition.** `navigateInternal()` set `_navigating = false` before the async `fetchPage()` resolved. A rapid second navigation during the fetch created two concurrent `.then()` handlers writing to the same container. The fix was a generation counter — each navigation increments the counter, and the `.then()` handler checks that its generation is still current before touching the DOM. The lesson: any async state transition that clears a guard before the async work completes has a race. The guard and the work must share a sequence number.

**Display value preservation.** `hideStaticContent()` set `display = "none"` without saving the original value. Elements with `display: flex` or `display: grid` lost their layout on round-trip — the restore function checked `dataset.pagesDisplay` but nobody had written it. The fix was one line: `el.dataset.pagesDisplay = el.style.display` before hiding. The lesson: hide/show toggling that touches `style.display` directly must save-and-restore, not set-and-clear. Every CSS framework learns this; every new DOM manipulation library re-learns it.

## Seeing it run

The theory is one thing. Watching someone click "Contacts" in a platform catalogue, see a contacts dashboard render with real data, click back to "Catalogue" and see the app listing reappear — that's different. The recursive model stops being an architectural talking point and becomes a thing you navigate.

The demo runs inside the examples gallery as a Server tab entry. The platform app's catalogue page — a data-table bound to `_apps` — renders four discovered apps. Click Helpdesk and a dashboard appears: nine open tickets, two urgent, pie charts by category, a sortable ticket queue with twelve rows of real data from the SQLite store. Click Tasks and a task board replaces it: seven open tasks, priority and assignee breakdowns. Click Catalogue and the app listing is back.

That last transition — back to the catalogue — exposed a bug. After navigating Contacts then Helpdesk, the ticket queue showed "Dataset 'tickets' not found in scope for page 'content/Contact Detail'". Page names from the contacts app had leaked into the helpdesk's scope. The demo page was nulling the previous site object without calling `dispose()`. One line — `currentSite.dispose()` before `currentSite = null` — and the leak vanished. It's the kind of bug you only find by actually using the thing: the unit tests passed, the integration tests passed, the API tests passed. Only the multi-app navigation revealed that site teardown was incomplete.

The catalogue itself is the meta observation. It's a page. The page is a YAML document stored in `_pages`. The document describes a data-table that queries `_apps`. The apps listed in `_apps` each contain their own `_pages`. The catalogue is simultaneously a component (embeddable in any app as a page), a composed app (the platform app wraps it with navigation), and a standalone host (the demo HTML shell loads it directly). No version of it is special. No version has different rules or different infrastructure.

## The compound effect

A page is a document. A collection of pages is part of an app. An app catalogue is a page that queries a collection of apps. The platform that hosts the catalogue is itself an app. No layer is special. No level has different rules.

The same YAML, the same store, the same validation, the same access control — all the way down. And because each level uses the same infrastructure, every capability built for one level automatically applies to every other. CRUD, ACL, schema validation, cache invalidation, hash-based bootstrap, static/dynamic protection — build once, available everywhere.

125 backend tests, 82 frontend tests, four apps, three system collections, two dedicated JAX-RS resources, one recursive model. The work compounds.
