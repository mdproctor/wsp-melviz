---
layout: post
title: "Two Things Code Review Found in push-runtime"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [push, testing, quarkus, validation, code-review]
---

# Two Things Code Review Found in push-runtime

Code review of the new `PushPublishResource` (#721) turned up two issues worth fixing before they bit someone. Neither was a bug in the feature — both were pre-existing gaps that the new code either inherited or exposed.

## Trusting the Wire

`PushPublishResource.publish()` accepts a raw `String` body and passes it straight to `EventBroadcaster.broadcast()`. The `@Consumes(MediaType.APPLICATION_JSON)` annotation tells JAX-RS to expect JSON, but it doesn't validate it — it just sets the content type expectation. If a caller sends `not valid json {{{` with a JSON content type header, the endpoint accepts it, stores it in the event store, and broadcasts it to every subscriber. Every downstream consumer then has to deal with malformed data.

The fix is a guard clause: parse the body with `ObjectMapper.readTree()` before the broadcast call. If it fails, return 400 with a clear message. Five lines of code. The interesting part isn't the fix — it's the question of where validation belongs.

```java
try {
    new ObjectMapper().readTree(body);
} catch (JsonProcessingException e) {
    throw new BadRequestException("request body must be valid JSON");
}
```

The endpoint's current callers are YAML workers via `call: http`, and they construct valid JSON. So why bother? Because the system boundary is the place to enforce invariants, not the caller. A future caller — a webhook integration, a debug tool, a misconfigured worker — will send garbage eventually. Catching it at the door means every component downstream can trust that what it receives is well-formed. Defense in depth at system boundaries is cheap insurance.

## The Shared Counter Problem

The second finding was subtler. `push-runtime` has two `@QuarkusTest` classes that exercise the event broadcaster: `BroadcastIntegrationTest` (direct CDI injection) and `PushPublishResourceTest` (HTTP endpoint). Both share a single Quarkus application instance within a test run, which means they share a single `@ApplicationScoped` `InMemoryEventStore`.

The event store assigns monotonically increasing sequence numbers per topic. When both test classes broadcast to the same topic (`nobody:listens`), the second class to run gets a sequence number that depends on what the first class already did. Any test asserting `seq == 1` is order-dependent.

The earlier session's blog entry noted this and fixed the specific collision — switching the overlapping assertion from `equalTo(1)` to `greaterThan(0)`. But it also said that `equalTo(1)` was "safe" for topics unique to a single test class. That's true today and wrong tomorrow. The moment someone adds a new test class that broadcasts to `orders:new`, the previously-safe assertion breaks — and the failure will be intermittent, because test class ordering isn't deterministic across JVM restarts.

I went through every seq assertion in both test classes and replaced them all:

| Before | After | Why |
|--------|-------|-----|
| `equalTo(1)` | `greaterThan(0)` | Seq is positive, not necessarily 1 |
| `isEqualTo(1)` | `isGreaterThan(0)` | Same pattern, AssertJ variant |
| `contains("\"seq\":1")` | `contains("\"seq\":")` | Verify seq is present, not its value |
| `isEqualTo(1)` / `isEqualTo(2)` for replay ordering | `get(1).seq() > get(0).seq()` | Monotonic ordering, not absolute values |
| `hasSize(1)` for cross-test topic | `isNotEmpty()` | Event stored, count unreliable |

The pattern generalises: any `@ApplicationScoped` bean that accumulates state — counters, caches, registries — will carry that state across `@QuarkusTest` classes in the same test run. The safe path is asserting properties (positive, non-empty, monotonic) rather than specific values. The unsafe path is writing `equalTo(1)` and hoping no future test class shares the topic.
