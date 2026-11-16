---
layout: post
title: "One Model, Two Runtimes"
date: 2026-10-01
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [scenario, walker, step-catalog, refactoring, unification]
series: 507-unify-scenario-declaration-model
---

# One Model, Two Runtimes

The pages step system had drifted into two parallel worlds. The Walker resolves YAML into a tree of structural steps — block, parallel, if/else, match, try/catch, barrier, quorum, select — and the StructuralEvaluator walks that tree batch-style. Clean, recursive, well-tested. Meanwhile, the scenario system had its own parser, its own type hierarchy (`ScenarioStep` by delivery mode: aria, graphql, simulated), and its own execution model — a discrete event simulation with virtual time, pause/resume, step-by-step playback, and queue-based interleaving.

Same repo. Same YAML format. Different types, different parsers, different capabilities.

The scenario path couldn't use `if:` or `match:` because its parser didn't know about them. The Walker couldn't run aria actions because they weren't in the catalog. A step defined for one path didn't exist in the other. Every new capability — control flow, state machines, decorators — had to be implemented twice or not at all.

I wanted to fix this without destroying what makes each path good. The Walker's clean tree resolution shouldn't be complicated by virtual clocks. The scheduler's DES semantics — the ability to pause between any two steps, to interleave concurrent branches tick by tick, to advance virtual time during delays — shouldn't be sacrificed for type purity.

The answer turned out to be a clean separation: share the model, keep the execution engines distinct.

The scenario parser becomes a thin envelope handler. It reads the `scenario:` metadata, extracts the few constructs that are genuinely scheduler-specific (signal firing, barrier countdown, triggers), and hands everything else to `Walker.resolve()`. The Walker does what it already knows how to do — resolve YAML maps against a catalog, build a structural tree. The scenario system just needed to give it a catalog that includes aria actions.

That catalog question led to the most interesting design discussion. I'd originally thought YAML step definition files for aria actions — `click.step.yaml` declaring its inputs, an `AriaInvokeHandler` creating the execution wrapper. The review pushed back: no YAML definition files exist in the codebase, the `Action` interface doesn't fit aria's execution model, and built-in primitives shouldn't pretend to be user-extensible.

The review was technically correct on the facts but wrong on the principle. The whole point of the catalog is to be a uniform abstraction. Steps are steps — whether they ship with the framework, get loaded from YAML, arrive via MCP, or are registered programmatically. The consumer never sees the registration mechanism. Making aria actions programmatic instead of YAML-defined would leak the distinction "these are special built-in primitives" into what should be a seamless registry. I reverted the decision back to YAML definitions.

The practical upshot: `InvokeBinding` grows three new kinds — `aria`, `graphql-domain`, `simulated`. An `AriaInvokeHandler` wraps the existing command-executor functions, converting their fire-and-forget `Promise<void>` into the catalog's `Result` type. Each aria action (click, fill, navigate, seventeen in total) gets a definition declaring its inputs. After resolution, they're indistinguishable from any other catalog entry.

We got the yaml-core foundation done — new binding types, `DelayStep` as a structural kind, the Walker wrapping non-object action values instead of silently dropping them (a genuine gotcha — `navigate: "/dashboard"` was resolving with empty params), post-step delay semantics in the `DecoratorChain`, and `awaitAll`/`awaitCount` on `StepResultStore` for the DES scheduler's barrier and quorum dispatch.

The invoke handlers are in too — `AriaInvokeHandler` plus a `loadScenarioDefinitions()` function that provides all nineteen definitions (seventeen aria plus graphql and simulated) as a `DefinitionFile[]`. The YAML file stays as human-readable documentation; the programmatic definitions are the runtime source.

The rewiring turned out to be the satisfying part. The old type system — `ScenarioStep` with its delivery-mode discrimination, `OrchestratedStep`, `StepDecorators`, `OrchestrationConstruct` — all retired in favour of two types: `SchedulerStep` (the Walker's `ResolvedStep` union) and `PreExtractedStep` for the three constructs that are genuinely scheduler-specific (signal firing, barrier countdown, signal/barrier awaiting). Everything else — delays, parallel branches, conditional steps, action dispatch — the Walker already knew how to handle.

The parser became something I hadn't expected: a pre-extraction filter. Walk the raw YAML step array once. Standalone `signal:` becomes a `signal-fire` PreExtractedStep. `await: { signal: ... }` becomes `await-signal`. `concurrent:` gets transformed into Walker's `parallel:` with named block children. Everything else passes straight through to `Walker.resolve()`. A slot-type array tracks which positions are Walker-resolved and which are pre-extracted, then a merge function interleaves them back in original order. The whole parser is under 100 lines — down from 220 that handled aria shorthand expansion, orchestration construct parsing, delivery-type detection, and decorator extraction individually.

The scheduler dispatch went from a two-function split (`dispatchStep` checking `delivery === 'orchestration'`, then `dispatchOrchestration` switching on `construct`) to a single switch on `step.kind`. Plugin steps execute directly via `entry.action.execute()` with a `MapServiceRegistry` carrying the EventTarget and speed — no more `StepExecutor` interface, no more `AriaExecutor` wrapping command-executor functions, no more `executors` array in scheduler options. The error model flipped from try/catch to Result inspection, which meant the scheduler could handle failures without unwinding the call stack — retry logic reads cleaner when failure is a value, not an exception.

The one subtlety that wasn't in the design: `when:` decorators became `if:` decorators. The old scenario system used `when:` for conditional step execution; the Walker uses `if:` (matching its structural `if:`/`then:`/`else:` pattern). A small migration, but the kind of thing that would have been a mysterious runtime silent-skip if missed — the scheduler would check `decorators['when']`, find nothing, and execute unconditionally.

The cross-repo question is worth noting. The Java backend has the same divergence — `ScenarioStep` with four variants, its own parser, its own executor, all separate from the platform's `Declaration`/`StepWalker`/`StructuralStepEvaluator`. Once the TypeScript unification proves the pattern, the Java side should follow. And the new `InvokeBinding` kinds need Java-side recognition for portability validation — not for execution (Java can't drive a browser), but so the platform knows "this step exists and is TypeScript-only" rather than "this step is unknown."
