---
layout: post
title: "Teaching the Step System to Talk to the Outside World"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [gallery, invoke-bindings, concurrency, orchestration, step-system]
---

# Teaching the Step System to Talk to the Outside World

The step system has had structural workflow primitives — blocks, parallel, try/catch, select, barriers — for a while now. And the sequential coordination primitives landed recently: counters, flags, gauges, accumulators, latches, state machines. What was missing were the two pieces that make the system feel like it could actually run a real workflow: invoke bindings (how steps talk to external systems) and concurrent coordination (how parallel steps share state safely).

Both are gallery examples, not runtime implementations. That's a deliberate choice. `StructuralStepEvaluator.evaluateInvoke` still returns failure — there are no real invoke handlers yet. But the gallery examples establish what the YAML syntax looks like for each binding type, what realistic execution traces look like, and what the mock executor pattern is. When someone builds the real handlers, the gallery shows exactly what they're aiming for.

## Invoke bindings — six ways out

The six binding types map to the `InvokeBinding` union in `types.ts`: REST, MCP, GraphQL, Script, Agent, and Process. Each gets a sub-example with a picker, YAML in the code editor, and a simulated execution trace with realistic responses. The REST mock returns `200 [{id:1, name:"Alice"}, ...]` for a GET and `201 {orderId:"ORD-4782"}` for a POST. The Agent mock simulates dispatching to `claude-sonnet-5` and getting back structured output. The point isn't fidelity — it's showing what the integration surface looks like when the real handlers arrive.

The mock executor approach is simple: register plugins with `createStepRunner` that simulate what each binding type would do. The YAML in the editor shows the invoke syntax; the JavaScript steps array uses plugin equivalents. This is the same pattern the existing Step Workflows and Coordination Primitives examples use.

## Concurrent coordination — primitives that need parallelism

The sequential primitives (counter, flag, gauge, etc.) demonstrate fine in isolation. The concurrent ones — semaphore, channel, orc-map, spawned task, correlation scope, deadline context — are only meaningful under parallel execution. A semaphore with no contention is just a counter.

The semaphore example runs four workers through a two-permit gate using `parallel` structural steps. The trace shows workers A and B acquiring immediately while C and D block until permits are released. The channel example shows a producer-consumer pattern with a bounded buffer of capacity 2 — the producer blocks when the buffer is full. The deadline example creates a 600ms scope, runs phase-1 (300ms) within budget, then phase-2 (400ms) which exceeds the deadline — the `onDeadline` callback fires mid-execution and the final check shows `expired=true, remaining=0ms`.

`DefaultCorrelationScope` wasn't exported from the orchestration barrel or `casehub-entry.ts`. I added both — it follows the same pattern as every other primitive export, it was just missed when the class was first written.

## What's next

Three things fell out as deferred issues. The mock executors could be extracted as a reusable test utility (#503) — right now they're gallery-only, but they'd be useful for any test that needs to simulate invoke steps. The real invoke handlers (#504) are the big one — actually implementing REST calls, process spawning, agent delegation against the `InvokeHandler` interface. And a combined concurrency scenario (#505) that composes multiple primitives in a single pipeline — rate-limited producer-consumer with deadlines, say — to show how the primitives compose rather than demonstrating each in isolation.
