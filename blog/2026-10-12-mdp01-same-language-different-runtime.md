---
layout: post
title: "Same Language, Different Runtime — Porting Playbook Primitives to TypeScript"
date: 2026-10-12
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [orchestration, typescript, playbook, concurrency, decorator-chain]
series: issue-536-ts-playbook-primitives
---

# Same Language, Different Runtime — Porting Playbook Primitives to TypeScript

The CaseHub playbook language has a clean separation between its grammar (YAML keywords, decorator chain ordering) and the runtime that executes it. Platform #562 landed three language extensions on the Java side — metric-threshold triggers (`at:`), inline continuations (`on-complete:`), and priority resource contention (`resource:/priority:`) — and the TypeScript runtime needed parity.

The port itself was mechanical. What I found interesting was how the concurrency model translates. Java's `DefaultPriorityOrcSemaphore` uses `ReentrantLock` with a `PriorityQueue` of `Condition` objects — each waiter gets its own condition variable, and `release()` polls the highest-priority waiter to signal. The TypeScript equivalent does the same thing with a sorted array and Promise resolvers. No locks, no condition variables — JavaScript's event loop gives you cooperative scheduling for free. The `insertWaiter` method is a six-line sorted insertion that replaces Java's entire `PriorityQueue<Waiter>` plus `Comparable` implementation.

The threshold trigger (`at:` decorator) shows the same pattern. Java registers a `DoubleConsumer` listener on `OrcNumericPrimitive` and blocks the calling virtual thread via `CompletableFuture`. TypeScript registers a callback and resolves a Promise. The observable interface is identical — `onThresholdChange(listener)` / `removeThresholdListener(listener)` — but the blocking mechanism disappears entirely. The decorator just awaits a Promise that resolves when all threshold conditions are met, cleaning up listeners afterward.

`OnCompleteExpander` was the most satisfying piece to port. It's a parse-phase transformation: walk the step list, find any step with `on-complete:`, desugar it into a `signal:/wait:` pair, and recurse into the continuations. No runtime involvement at all — by the time the decorator chain sees the steps, `on-complete:` has been replaced by primitives the chain already understands. The Java and TypeScript implementations are nearly line-for-line identical because there's no concurrency model to translate — it's pure data transformation.

The decorator chain ordering is the real contract between the two runtimes. `at:` sits between `timeout:` and `retry:` — inside the timeout boundary so threshold waits can be cancelled, outside retry so a threshold-gated step can retry on failure. `resource:` sits at the same level as `semaphore:` — it's a semaphore with priority ordering, not a new concept. Getting this ordering right matters more than the implementation of any individual decorator, because the ordering determines how decorators compose in YAML.

What this opens up: issues #537 and #538 cover the remaining parity work — `loop: continuous`, `cancel:`, `background:`, and eventually `from:` for stream processing. The foundation is in place. Each new decorator slots into the chain at its designated position, and the primitive interfaces (`OrcNumericPrimitive`, `PriorityOrcSemaphore`) give it the building blocks.
