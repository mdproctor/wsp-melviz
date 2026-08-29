---
layout: post
title: "The scenario engine wants to be an automation platform"
date: 2026-08-29
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [scenario-engine, automation, script-library, yaml-core, design]
---

# The scenario engine wants to be an automation platform

The scenario engine was built for demos — scripted walkthroughs where a presenter steps through a help desk workflow while the audience watches. It works well for that. But teams are now asking for something different: they want to run automations. Onboarding workflows, data imports, environment setup. Same engine, different purpose.

I spent this session designing the extension. The core insight is that demos and automations share the same execution model — ordered steps dispatched to browser and service executors via the push wire protocol. What they don't share is the lifecycle around the script itself. Demos are one-off files you load by name. Automations need a library, parameters, data-driven loops, composability.

## Ten decisions, one shared primitive

The design landed on a script library with three sources: bundled scripts that ship with the distribution, uploaded scripts that users paste in, and external registries that serve JSON manifests. Each script declares typed parameters, CaseHub labels, and free-form tags. The library browser lives inside the existing scenario controller — browse, filter, run, paste — all one flow.

The most interesting decision was about `forEach` and `when` — the control flow constructs for data-driven iteration and conditional steps. Desired-state already has these exact constructs in its YAML surface. Same `${each.*}` interpolation, same truthiness evaluation, same named iteration groups. The question was whether to reimplement or share.

I went with a shared primitives library in the platform. Not a shared framework — the host structures are too different (desired-state operates on a DAG of typed nodes; scenario automation operates on ordered step sequences). But the leaf-level primitives — `VariableResolver`, `ForEachExpander`, `CsvParser`, `Truthiness` — are genuinely domain-agnostic. I briefed Eidos on the requirements, and `casehub-platform-yaml-core` was delivered within the session: zero dependencies, J2CL-transpilable, generic `ForEachAdapter<E>` so each domain provides its own element adapter.

The new dimension is CSV data sources. First row declares `columnName:type` pairs. Type enforcement at parse time. A step can `forEach` over CSV rows, accessing typed columns via `${each.member.name}`. Both desired-state and scenario automation get this for free once the shared core exists. Desired-state will migrate to the shared core next.

## Composability with teeth

Scripts can call other scripts. A `call` command references a script by name from the registry, passes parameters, and the callee's steps are inlined at the call site with name prefixing. The callee inherits the caller's full resolved context — no re-resolution.

Circular calls are rejected at load time, not execution time. When a script is loaded (from any source — bundled, uploaded, or pasted), the system builds the transitive call graph by resolving every `call` reference through the registry. If it finds a cycle, the script is rejected before it can run. This is enforcement at author time, not a runtime guard.

## What got built

Batch 1 of the implementation plan landed: `ScriptDescriptor` model, `ScriptDescriptorExtractor` (parses YAML to extract metadata without full compilation), `ScriptRegistry` (aggregates bundled + uploaded sources with label/tag filtering and name collision protection), and the REST API at `/scenario/library`. Four remaining batches cover the compilation pipeline (yaml-core integration), composability (call graph validation), library browser UI, and external registries.

The readiness probe was the most satisfying small design. The natural instinct is to put `ready: boolean` on the server's descriptor. But the server can't probe the browser DOM. The fix: extract ARIA targets from the first step at parse time, include them in the descriptor, let the client run `findByRole()` locally. Green, amber, red — without a single server-to-client round trip.
