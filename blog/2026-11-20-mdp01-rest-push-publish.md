---
layout: post
title: "Giving Workers a Voice: REST Publishing for Push Topics"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [push, rest, quarkus, workers, yaml]
---

# Giving Workers a Voice: REST Publishing for Push Topics

Until now, publishing events to push topics was a Java-only operation. `EventBroadcaster` sits in the `push` module — a pure Java class that appends to the event store, formats a wire message, and sends it to every connection subscribed to that topic. If you wanted to broadcast an event, you needed to be server-side Java code with a CDI-injected reference to the broadcaster.

That's fine for the platform itself. It's not fine for YAML workers.

Workers run in `do:` blocks and interact with the world through `call: http`. They can hit any REST endpoint the platform exposes. But there was no REST endpoint for push publishing. A worker that needed to notify subscribers — "this case just escalated", "new data arrived", "the debate round ended" — had no way to do it.

The fix is small: a single JAX-RS resource in `push-runtime` that accepts a POST with a JSON payload and delegates to `EventBroadcaster`.

## The Endpoint

```
POST /api/push/publish/{topic}
Content-Type: application/json
Authorization: Bearer <jwt>

{"item": "widget", "qty": 3}

→ 200 {"seq": 42}
```

The topic goes in the path. The body is arbitrary JSON — passed through to the broadcaster as-is. The response includes the sequence number assigned by the event store, which gives callers an ordering guarantee.

I chose `@Authenticated` to match every other `/api/` resource in the backend. Workers already have auth context from their session, and leaving an endpoint that can broadcast to any push topic unauthenticated seemed like asking for trouble.

## The Flaky Test

The interesting part wasn't the resource — it was a test ordering issue.

`push-runtime` already has `BroadcastIntegrationTest`, a `@QuarkusTest` that exercises the broadcaster directly through CDI injection. The new `PushPublishResourceTest` exercises the same broadcaster through the HTTP endpoint. Both are `@QuarkusTest` classes in the same module, which means they share a single Quarkus application instance — and with it, a single `@ApplicationScoped` `InMemoryEventStore`.

The event store assigns sequence numbers per topic. One of the new tests published to the topic `nobody:listens` and asserted `seq == 1`. The existing `BroadcastIntegrationTest` also publishes to `nobody:listens`. When the existing test ran first, the seq was already at 1 — so the new test got 2 and failed.

Claude caught this on the first full test run. The initial fix: assert `greaterThan(0)` instead of `equalTo(1)` for the specific topic that collided. A follow-up pass (#730) went further — replacing every exact seq assertion across both test classes with relative checks, because any topic could gain a cross-class collision in the future.

This is a general pattern worth knowing with Quarkus `@QuarkusTest`: `@ApplicationScoped` beans survive across test classes within the same test run. If those beans accumulate state — counters, caches, registries — test classes that assert specific values against that state are inherently order-dependent. The safe path is asserting properties (`greaterThan`, `isNotEmpty`, monotonic ordering) rather than specific values.

## What This Opens Up

YAML workers can now push events directly. A helpdesk playbook can broadcast "case assigned" to a dashboard. A data pipeline can notify subscribers when new records arrive. An IoT workflow can fan out sensor readings to monitoring views. All through `call: http` in a `do:` block — no Java, no custom extensions.

The endpoint is thin enough that there's nothing interesting in the implementation — 40 lines of Java, most of which is annotations. The interesting thing is what it unblocks.
