---
layout: post
title: "The Model That Owns Nothing"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [graph-core, architecture, typescript]
---

The visual diagram editor needs a graph. Not the rendering — React Flow handles that — but the logical representation between YAML parsing and pixel drawing. `@casehubio/graph-core` is that middle layer: typed nodes, typed edges, a containment tree, and nothing else.

The interesting constraint was scope. The parent spec places six concerns in graph-core: the model, a stencil grammar registry, a constraint validator, edit operations, a persistence SPI, and decoration types. That's a lot of surface for one package. But the zero-dependency position is what makes it work — graph-core sits below everything in the graph package family, so any shared contract naturally lives here without pulling in transitive weight.

## Three interfaces, no classes

```typescript
interface GraphNode {
  readonly id: string;
  readonly type: string;
  readonly parentId?: string;
  readonly properties: Readonly<Record<string, unknown>>;
}
```

`GraphNode`, `GraphEdge`, `GraphModel` — plain TypeScript interfaces with `readonly` fields. No classes. The decision was structural typing over nominal: domain adapters return `{ nodes, edges }` directly from a `toGraph()` function without constructing anything. Any object matching the shape works. The properties field is `Record<string, unknown>` — graph-core carries domain data without knowing what it means.

## Where containment lives

Workers contain capabilities. Cases contain sub-cases. The graph needs hierarchy. I went with the simplest representation: `parentId` on nodes. Root nodes have no parent. The containment tree is implicit in the data and explicit in four traversal functions — `childrenOf`, `ancestorsOf`, `subtreeOf`, `rootNodes`.

`subtreeOf` is breadth-first. That matters for ELK layout, which needs level-order traversal to build its hierarchical `INCLUDE_CHILDREN` structure. The graph-renderer's `elk-layout.ts` currently builds its own containment map — when it depends on graph-core, it consolidates onto these functions instead.

## Validation that tells you everything at once

`createGraph` validates structural invariants — duplicate IDs, dangling edges, invalid parent references, containment cycles, empty IDs. The design review pushed for two improvements I hadn't initially specified: collect all violations before throwing (not fail-fast), and throw a structured `GraphValidationError` with a typed `violations` array. Both were right. A domain adapter debugging a malformed YAML parse needs to see every problem, not just the first one.

The constraint validator is a separate layer — it validates the graph against registered `StencilGrammar` rules (edge counts, type allowlists, containment constraints). Structural validation catches corrupt data; constraint validation catches grammar violations. The distinction matters because grammar violations are informational (an unresolvable external capability reference is normal, not an error), while structural violations are fatal.

## Edit operations close the loop

`addNode`, `removeNode`, `replaceNode` — each takes a valid model and produces a new one. They return `{ model, violations }` where violations are informational. `removeNode` cascades to children and associated edges. The key invariant: every model produced through `createGraph` or the edit operations satisfies the structural contracts. Together they form a closed system.

The full package: 99 tests, 7 source files, zero dependencies. Next up is the stencil grammar for case definitions and the renderer integration — the point where this model starts producing visible graphs.
