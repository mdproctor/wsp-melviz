# Design: Data Pipeline Examples for Graph Editing Showcase

**Date:** 2026-08-27
**Issue:** #378 (diagram editing infrastructure)
**Also covers:** #373 (property palette)
**Status:** Draft

## 1. Purpose

Showcase examples in the pages gallery demonstrating graph editing infrastructure and property palette integration. Uses a made-up "Data Pipeline" domain (not SWF/Case — those are blocks-ui domain adapters). The examples demonstrate capabilities and extension points for customisation.

## 2. Domain: Data Pipeline

Five node types with distinct connection constraints:

| Type | Color Token | Icon | Inbound max | Outbound max | allowedTo |
|------|------------|------|-------------|--------------|-----------|
| Source | success-9 | ⬇ | 0 | 2 | transform, filter, join |
| Transform | accent-9 | ⚙ | 3 | 2 | transform, filter, join, sink |
| Filter | warning-9 | ⧖ | 1 | 2 | transform, filter, join, sink |
| Join | info-9 | ⨝ | 4 | 1 | transform, filter, sink |
| Sink | danger-9 | ⬆ | 2 | 0 | (none) |

Property schemas per type:

| Type | Properties |
|------|-----------|
| Source | name: string, url: string, format: enum[json,csv,xml], pollIntervalSec: number |
| Transform | name: string, expression: string, language: enum[jsonata,jq,javascript] |
| Filter | name: string, condition: string, passLabel: string, failLabel: string |
| Join | name: string, strategy: enum[merge,zip,concat], windowMs: number |
| Sink | name: string, url: string, format: enum[json,csv], batchSize: number |

## 3. Stencil Rendering

Each stencil renders via a lit-html template function (`StencilRenderFn`). All use the same layout: a rounded rectangle with the type's token color as background, a large icon character, and the node's `name` property (or type label as fallback) below.

```
┌──────────────┐
│      ⚙       │  ← icon character, color from token
│  Clean Data  │  ← node.properties.name or type label
└──────────────┘
```

Stencils use inline styles with CSS custom properties from pages-ui-tokens for automatic theme support in both light and dark modes.

## 4. Examples

### 4.1 Basic Pipeline (static rendering)

**Demonstrates:** Stencil registration, GraphModel creation, ELK auto-layout, custom node rendering.

Pre-built graph: `API Source → Parse JSON → Filter Valid → DB Sink` with a branch from Parse JSON to a second sink (Log Sink). Shows that layout, stencil rendering, and the minimap work out of the box.

No editing enabled — purely read-only visualisation.

### 4.2 Interactive Editing

**Demonstrates:** Connection drawing, edge reconnection, node deletion strategies, EditPolicy constraint enforcement, onMutation callback.

Same starting graph as Basic Pipeline, but with `editPolicy` and `onMutation` wired on the canvas. A status log panel below the canvas shows each mutation event.

Key interactions to try:
- Draw a connection from Source to Sink → **blocked** (Source's allowedTo doesn't include sink... wait, actually it does include it indirectly through Transform/Filter. Let me fix: Source's allowedTo does NOT include 'sink' directly, so connecting Source→Sink is blocked)
- Draw a connection from API Source handle → Filter Valid handle → **allowed**
- Delete the Parse JSON node → **auto-join** reconnects API Source → Filter Valid
- Delete API Source → **disconnect** (no predecessor to join)
- Reconnect an edge endpoint by dragging it to a different node

The companion TS wires a `defaultEditPolicy()` and an `onMutation` that logs to a `<pre>` element below the canvas.

### 4.3 Pipeline with Properties

**Demonstrates:** PropertyPaletteSource, schema-driven form generation, selection → palette binding, onChange handler.

Split layout: canvas on the left (70%), property palette on the right (30%). Clicking a node selects it and populates the property palette with that node type's schema and current property values.

The companion TS:
1. Creates a `<pages-property-palette>` element
2. Listens for `graph:node:click` events on the canvas
3. Looks up the clicked node's type → resolves the property schema
4. Sets the palette's `source` property with schema, data, and onChange callback
5. onChange updates the node's properties in the model (no re-layout needed — properties don't affect graph structure)

## 5. Technical Integration

### 5.1 Package dependencies

Add to `examples/package.json`:
```json
"@casehubio/graph-core": "workspace:*",
"@casehubio/graph-renderer": "workspace:*",
"@casehubio/pages-property-palette": "workspace:*"
```

### 5.2 Entry point changes

`examples/src/casehub-entry.ts` gains:
```typescript
import '@casehubio/graph-renderer';
import '@casehubio/pages-property-palette';
import './pipeline-stencils.js';

export {
  registerStencil, clearRegistry, defaultEditPolicy, applyGraphEdit,
} from '@casehubio/graph-renderer';
export { registerGrammar, clearGrammarRegistry } from '@casehubio/graph-core';
export type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
```

### 5.3 Stencil setup module

`examples/src/pipeline-stencils.ts` — imported as side-effect by the entry point. Registers all 5 pipeline stencils with their grammar and render functions. Exports:
- `createBasicPipelineModel(): GraphModel` — the starter graph
- `PIPELINE_SCHEMAS: Record<string, FieldSchema>` — property schemas per type

### 5.4 Gallery YAML files

Each example is a `.dash.yaml` with `type: html` rendering the canvas element, plus a companion `.ts` that configures it. The YAML provides sizing and any surrounding layout (e.g., the split panel for the properties example).

### 5.5 Category registration

Add `'Graph Editing'` to `CATEGORY_ORDER` in `examples/scripts/generate-samples.js`.

## References

- `packages/graph-renderer/src/editing/` — EditPolicy SPI, GraphEdit types, defaultEditPolicy
- `packages/graph-renderer/src/bridge/GraphCanvas.ts` — pages-graph-canvas element
- `packages/graph-renderer/src/registry/stencil-registry.ts` — StencilDescriptor, registerStencil
- `packages/pages-property-palette/` — PropertyPaletteSource, PagesPropertyPalette
- `examples/src/casehub-entry.ts` — UMD bundle entry point
- `examples/src/app.js` — gallery companion script loading (stripTs + new Function)
- GE-20260803-50ddbd — lit-html → React Flow bridge pattern
- GE-20260809-2cbc61 — ReactFlow for directed diagrams
