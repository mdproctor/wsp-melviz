---
layout: post
title: "Scenario Outcome Tracking — Making Fire-and-Forget Tell Its Story"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [scenario, outcome, scheduler, cbr, yaml-core]
---

The scenario scheduler has always been fire-and-forget. It runs steps, emits progress events during execution, and when it's done, calls `onComplete` and walks away. No structured result. No record of what happened, which steps failed, how long anything took. The execution just evaporates.

Issue #500 adds outcome recording — a structured `ExecutionOutcome` record emitted as a `scenario:outcome` event when every scheduler run ends. The record captures per-step timing, success/failure disposition, aggregate counts, and optionally the full simulation journal.

The interesting design constraint was the CBR boundary. The issue originally scoped CBR case linkage inside pages, but that's wrong — CBR is a unified cross-runtime concern. Engine's CBR bridge (#1190) needs to consume outcomes from both TypeScript pages and Java platform playbooks through a single channel. Pages shouldn't know about CBR internals at all. Its job ends at producing a well-structured outcome record and publishing it through the existing push protocol.

That decision shaped everything downstream. The outcome data model lives in yaml-core (the shared step type system), not pages-aria. Java platform will mirror the same shape. The `OutcomeCollector` is a passive observer — it receives `stepStarted`/`stepCompleted` notifications and produces an `ExecutionOutcome` on `finalize()`. Three touch points in the scheduler: instantiation, step dispatch instrumentation, and emission at the three exit paths (completed, failed, deadlocked).

One subtlety we caught in self-review: the scheduler calls `simulationRuntime.popAll()` before its final state emission on the normal completion path. If outcome emission happened after `popAll()`, the overlay stack would already be cleared and journal entries would be lost. The fix was straightforward — emit the outcome before clearing overlays — but it's the kind of ordering bug that would have been invisible until someone wondered why simulation outcomes never carried journal data.

The whole change is 399 lines across 7 files, most of it tests. The production code is roughly 130 lines: three interfaces, one class, one new method on `SimulationRuntime`, and 18 lines of scheduler wiring. What engine does with these outcome events is #1190's problem.
