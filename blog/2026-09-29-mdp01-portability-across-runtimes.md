---
layout: post
title: "Portability Across Runtimes"
date: 2026-09-29
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [portability, catalog, multi-source, lit]
series: feat/506-unified-step-catalog
---

# Portability Across Runtimes

The unified step catalog needed a way to say "this action only works in Java" or "this one runs anywhere." Without that, a user browsing the catalog in a TS-only environment would try to execute a Java-only action and get an opaque failure at runtime. The portability model makes that constraint visible before anyone clicks Execute.

## The Type System

Four values: `universal`, `java`, `ts`, `both`. The inference rule is simple — if an action's invoke binding is REST, GraphQL, or process, it's `universal` because those protocols don't care which runtime sends the request. MCP, script, and agent bindings are `ts` because they depend on the Node runtime. Explicit annotation overrides inference, so a Java-native action declared in YAML can say `portability: java` and the parser respects it.

The interesting design choice was where to put the boundary between typed and untyped. The foundation layer (`yaml-core`) uses a union type — `'universal' | 'java' | 'ts' | 'both'` — with `isCompatible()` and `validatePortability()` as the enforcement functions. The UI layer uses `portability: string` on its interfaces. The cast happens at the call site, and the safe direction is deliberate: an unknown string fails `isCompatible`, which blocks execution rather than allowing it. A stricter approach would thread the union type all the way up, but that creates a hard dependency between the UI package and foundation types for a single field. The pragmatic boundary felt right.

## Multi-Source Merge

The catalog component now accepts an array of `CatalogDataSource` objects instead of hard-wiring a single GraphQL fetch. Three implementations exist — `RegistryCatalogSource` for local plugin registries, `RestCatalogSource` for the TS server endpoint, `GraphqlCatalogSource` for the Java backend. Priority-based deduplication means the lowest-priority source wins on name collisions, so a local plugin always shadows a remote definition of the same name.

The merge itself is straightforward: fetch all sources in parallel, catch failures per-source so one broken endpoint doesn't take down the whole catalog, and build a source map that tracks which `CatalogDataSource` provided each action. Detail fetches go back to the originating source rather than always hitting GraphQL. The existing `loadCatalog()` path still works for backward compatibility — sources are opt-in.

## The Pre-Flight Gate

The execute handler gained a `runtime` parameter. Before touching the action's `execute()` method, it checks `isCompatible(portability, runtime)`. Incompatible actions fail with a clear message naming both the action's portability and the current runtime. The same check runs in the UI — the Try It panel shows an inline warning and disables the Execute button when the selected action can't run in the component's runtime. No silent failures, no cryptic stack traces from a Java action trying to execute in a browser.

The portability badges — coloured chips next to each action in the catalog list — make the constraint visible at browse time. Green for universal, orange for Java, blue for TS, purple for both. Filter chips let you narrow by portability alongside the existing source filters, with AND logic between the two dimensions.

With platform#483 landed on the Java side, the contract between runtimes is stable. The catalog can now show a merged view of actions from both worlds, and the user knows before clicking which ones will actually work.
