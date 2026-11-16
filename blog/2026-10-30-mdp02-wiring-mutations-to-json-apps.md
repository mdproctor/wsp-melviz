---
layout: post
title: "Wiring Mutations to JSON Apps"
date: 2026-10-30
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [json-apps, data-pipeline, optimistic-concurrency, mutable-rest-source]
series: issue-597-wire-mutable-rest-source
---

# Wiring Mutations to JSON Apps

Pages has had `mutableRestSource` for a while — a data source that reads from one endpoint and dispatches create/update/delete actions to write endpoints. It worked well for wiring to arbitrary REST APIs. But the JSON Apps initiative needed something more specific: a way for `app.yaml` to declare a collection, and have the runtime automatically wire up read and write paths without the page author touching a single line of TypeScript.

The gap was three things: optimistic concurrency headers, structured error handling, and automatic binding generation. I'll take each in turn because the design choices are worth understanding, not just the implementation.

## Why If-Match Matters

Most web applications handle concurrent edits badly. Two users open the same ticket. Both edit the priority. The second save silently overwrites the first. The data is wrong and nobody knows it happened.

`If-Match` is the HTTP mechanism for optimistic concurrency. The server returns a version with each document. When the client sends an update, it includes `If-Match: <version>`. If the document has changed since the client read it, the server rejects the update with `409 Conflict` instead of silently overwriting.

![Mutation flow from browser through mutableRestSource to SQLite and back](images/collection-write-flow.svg)

The existing `WriteConfig` had a `headers` field — but it was static. Same headers for every request. Optimistic concurrency needs per-request headers because each row has its own version.

I considered three approaches. A dynamic headers callback (`headers: (action) => Record<string,string>`) was the most general, but it pushes header construction responsibility onto every caller. A `versionColumn` config that tracks row state internally was automatic, but it means the source has to maintain a mirror of the server's version state — and stale mirrors are exactly the problem we're trying to solve.

The approach I chose was simpler: add an optional `version` field to `DataAction` itself. The caller passes the version it sees in the data. `mutableRestSource` adds `If-Match` when it's present. No internal state tracking. No callback indirection. The concurrency contract is explicit in every mutation call.

```typescript
export type DataAction =
  | { type: "update"; key: string; changes: Record<string, unknown>; version?: string }
  | { type: "create"; data: Record<string, unknown> }
  | { type: "delete"; key: string; version?: string };
```

Existing callers that don't pass `version` are unaffected. The `If-Match` header simply doesn't appear, and the server treats it as an unconditional write. Backward-compatible by default, safe when opted in.

## Typed Errors Instead of Status Codes

Before this change, `mutableRestSource` threw a generic `Error` with the status code and response text concatenated into the message. Callers had to parse strings to figure out what went wrong. That's fine for "something failed" but terrible for "show the user which fields are invalid."

The `CollectionResource` backend returns structured responses: `422` with a `{ errors: { field: ["message", ...] } }` body for validation failures, and `409` with `{ currentVersion: "7" }` for version conflicts. The TS side now has typed error classes to match:

```typescript
// Caller can render field-level validation messages
try {
  await source.dispatch({ type: "create", data: formValues });
} catch (e) {
  if (e instanceof ValidationError) {
    // e.fieldErrors: { title: ["required"], priority: ["invalid value"] }
    showFieldErrors(e.fieldErrors);
  }
  if (e instanceof ConflictError) {
    // e.currentVersion tells you what the server has now
    offerMerge(e.currentVersion);
  }
}
```

The point isn't the error classes themselves — those are straightforward. The point is that the error contract between server and client is now typed end-to-end. A form component can render per-field validation messages. A conflict dialog can show the current server version and offer a retry. Neither of those is possible when all you have is `"422: {"errors":{"title":["required"]}}"` as a string.

## From 41 Files to 4

Here's where it comes together. A JSON app declares its collections in `app.yaml`:

```yaml
collections:
  tickets:
    fields:
      subject: string
      priority: { type: enum, values: [LOW, MEDIUM, HIGH, URGENT] }
      status: { type: enum, values: [OPEN, TRIAGED, RESOLVED, CLOSED] }
      assigneeId: string
  agents:
    fields:
      name: string
      email: string
      active: boolean
```

The `collectionBindings()` utility takes this metadata and returns a ready-to-use `DataSourceBinding[]` — one per collection, each with `mutableRestSource` pre-wired to the correct endpoints:

```typescript
const bindings = collectionBindings("helpdesk", collections);
// Returns 2 bindings:
//   json:helpdesk/tickets  → CRUD at /api/collections/helpdesk/tickets
//   json:helpdesk/agents   → CRUD at /api/collections/helpdesk/agents
```

Each binding has `keyColumn: "id"`, columns derived from the field metadata, and create/update/delete endpoints pointing at the `CollectionResource`. The runtime injects these into the `DataSetScope` alongside regular YAML datasets. From the data pipeline's perspective, a collection binding looks exactly like any other data source — the `isBinding()` type guard routes it through the same connect/deliver/refresh path that powers every other dataset.

![The JSON App stack — from YAML files at the top to SQLite at the bottom](images/json-app-stack.svg)

The helpdesk proof of concept went from 41 Java files in a compiled Quarkus project to 4 YAML/JSON files. No Maven. No compilation. No deployment pipeline. Edit the YAML, reload the page. The data mutations flow through the same `mutableRestSource` that has been battle-tested against arbitrary REST APIs — but now the wiring is automatic.

## What This Opens Up

The immediate practical effect is that a class of CaseHub applications — CRUD over structured data with forms, tables, and charts — no longer need a Java project. That changes the time-to-first-app from hours to minutes. But there's a less obvious implication: because the binding generation is driven by metadata, not code, the same mechanism can power runtime app creation. A user defines collections in a UI, the backend creates the storage, and the frontend gets working CRUD bindings without anyone deploying anything.

The typed error classes are the other piece worth watching. Right now they enable field-level validation display and conflict resolution dialogs. But they're also the foundation for form-level validation flows — where the server validates against a JSON schema and the client maps errors back to form fields automatically. That's a separate issue, and the wiring for it is now in place.
