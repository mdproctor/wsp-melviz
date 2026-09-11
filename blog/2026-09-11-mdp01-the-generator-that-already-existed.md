---
layout: post
title: "The Generator That Already Existed"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [lsp, ts-morph, zod, code-generation, schema]
series: issue-420-lsp-schemas
---

# The Generator That Already Existed

The component schema generator in `pages-schema` had been running for weeks — reading TypeScript interfaces via ts-morph, emitting Zod schemas, catching upstream drift in CI. Issue #420 was about extending the same idea to the four domain YAML formats: CaseDefinition, SWF, HTN, Org.

The hand-written schemas in `lsp-schemas` were fine as a baseline. But CaseDefinition alone has 826 lines of auto-generated TypeScript types from the YAML schema spec, and those types have `[k: string]: unknown` index signatures on most sub-objects — a JSON Schema artifact that needs stripping before you can emit useful Zod for LSP completion.

The interesting part was what ts-morph does with these types. CaseDefinition's `Binding` type is an intersection: `{ declared properties } & { [k: string]: unknown }`. TypeScript resolves that to an intersection type, not an object type. The generator's `isObject()` check missed it entirely — bindings came out as `z.unknown()`. The fix: add an `isIntersection()` branch that calls `getProperties()` on the intersection (which returns the flattened set from all members), then filters out the index signatures.

A second gotcha: `getNonNullableType()` on `unknown` returns `{}` — the empty object type. TypeScript's type algebra is correct (`NonNullable<unknown>` IS `{}`), but a code generator that strips optionality before mapping types silently turns `input?: unknown` into `z.object({}).optional()`. The fix was to check for `unknown` before calling `getNonNullableType()`.

HTN was the recursion test. `HtnTaskYaml` has methods, each method has tasks, each task can have methods. The generator's cycle detection via a `visited` set emits `z.lazy(() => htnTaskYamlSchema)` at the cycle point — but `htnTaskYamlSchema` doesn't exist as a variable. The solution was a lazy reference accumulator: collect cycle-referenced types in a Map during the walk, then emit their `z.lazy()` declarations as a prelude before the main schema.

All four formats now generate from TypeScript source types, with a staleness test that compares committed schemas against fresh generation. When someone adds a field to `CaseHub` in the engine repo, CI in blocks-ui fails until the schemas are regenerated. That's the point — silent drift was the problem this whole issue exists to solve.
