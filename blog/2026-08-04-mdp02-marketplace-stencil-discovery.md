---
layout: post
title: "When your diagram editor needs to know what workers actually do"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [graph-work-registry, marketplace, stencils, diagram-editor]
series: issue-280-graph-work-registry
---

The visual diagram editor so far can render a case definition — bindings, workers, milestones, goals — as a directed graph with auto-layout. You can select nodes, zoom, pan, see decorations for runtime state. But there's a gap between structural stencils and the real world of work types.

A Worker in a case definition says "I have capabilities." It doesn't say what technology it uses, what inputs it expects, what outputs it produces, or whether it runs synchronously or fires and waits. That metadata lives somewhere else entirely — in marketplace YAML files that describe work types as declarative descriptors. A "send-email" stencil declares its icon, its category (`connectors/messaging`), its property schema, its I/O contract, and whether it's async. No executable code — just metadata.

The `@casehubio/graph-work-registry` package is the bridge. It fetches marketplace YAML from configurable URLs, parses each descriptor into a typed `WorkStencil` object, validates the required fields, and makes everything queryable. The interesting design choice was the `CategoryIndex` — work stencils declare their category as a slash-separated path (`connectors/messaging`, `ai/agents`, `human/tasks`), and the index builds a tree from flat declarations. A palette component can render this as nested groups without the registry knowing anything about UI.

```typescript
interface WorkStencil {
  name: string;
  displayName: string;
  category: string;       // "connectors/messaging"
  icon: string;
  async: boolean;
  properties: PropertySchema;
  input: JSONSchema7;
  output: JSONSchema7;
}
```

The registry itself is URL-based with TTL caching — `load()` takes an array of marketplace URLs, fetches each independently, and collects stencils across all sources. A single failing URL doesn't break the rest. `refreshIfStale()` skips the network if the cache is fresh. The whole thing is a dependency-injected `FetchFn` so tests don't touch the network.

The second piece is in graph-renderer: a `defaultWorkStencilRenderer` that takes any `WorkStencil` and produces a lit-html template showing the icon, display name, category badge, async/sync indicator, and a summary of the I/O contract. `toWorkStencilDescriptor()` wraps this into a `StencilDescriptor` that plugs directly into the existing stencil registry — grammar auto-registered, React Flow custom node auto-generated, no special handling needed. When blocks-ui's palette loads marketplace data, each discovered work type just appears as a renderable node type.

The decision to keep work stencils purely declarative — no executable render functions from external URLs — was already made in the parent spec, but implementing it confirms the reasoning. Loading arbitrary code from marketplace URLs is a security hole you don't want. The default renderer handles every marketplace work type from metadata alone. If a specific work type ever needs custom visuals, that's a built-in stencil template in the codebase, not something fetched at runtime.

Phase 6 completes the foundation layer of the diagram editor epic. Phases 5 (SWF drill-down) and 7 (runtime overlay) can now run in parallel — they're independent of the work registry and each other. The bigger picture: between graph-core's model, graph-renderer's pipeline, and now graph-work-registry's discovery, the platform has everything blocks-ui needs to render a case definition with real work type metadata. The domain layer builds on top; the platform doesn't need to know about cases, workflows, or CaseHub-specific concepts.
