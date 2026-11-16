---
layout: post
title: "One Model, Two Runtimes"
date: 2026-10-01
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [scenario, walker, step-model, unification, yaml-core]
series: 507-unify-scenario-declaration-model
---

# One Model, Two Runtimes

The pages repo has two step execution paths that do the same job differently. The step runner path uses `Walker.resolve()` to parse YAML into a 10-kind discriminated union (`ResolvedStep`), then `StructuralEvaluator` walks the tree batch-style — if/else, match, parallel, try/catch, the full control flow vocabulary. The scenario path uses its own parser to produce `ScenarioStep` (three delivery modes: aria, graphql, simulated) and feeds them into a DES scheduler with virtual time, queue-based interleaving, and pause/resume.

They share the same state primitives (`ScenarioScope`, `StepResultStore`) but everything else diverged. A step defined in one path doesn't work in the other. The scenario system can't express `if/match/block` because its parser doesn't know about them. The step runner can't play back scenarios because it has no concept of time or user-controlled playback.

The design question was which direction to push. Making the `StructuralEvaluator` DES-aware would have bloated a clean recursive evaluator into a hybrid. Making the DES scheduler use the Walker's type system but keep its own execution mechanics preserves both sides' strengths — Walker handles "what does this YAML mean structurally," the scheduler handles "how do I play it back with timing and user controls."

The interesting constraint came from the user: the step catalog is a uniform abstraction. Steps are steps, regardless of whether they're built-in aria primitives or YAML-defined actions or MCP tools. The catalog hides where steps come from. This killed the programmatic registration approach — if the catalog is supposed to erase the distinction between built-in and user-provided, then defining aria actions differently from other actions leaks an implementation detail. YAML step definitions for everything, same as any other catalog entry.

We got through the design (spec, two rounds of review, 25 issues addressed) and the first two implementation batches. yaml-core now has three new `InvokeBinding` kinds (aria, graphql-domain, simulated), a `DelayStep` structural kind for standalone delays, and the Walker wraps non-object action values instead of silently dropping them — that last one was a genuine gotcha. An `AriaInvokeHandler` wraps the existing command-executor functions and converts exceptions to `Result` values.

The remaining work is the heavy refactoring: rewriting the scenario parser to delegate to Walker, introducing `PreExtractedStep` for signal/await/barrier constructs that don't map to Walker's structural model, and rewiring the DES scheduler to dispatch `ResolvedStep` kinds instead of delivery modes. That's where the theory meets the codebase.
