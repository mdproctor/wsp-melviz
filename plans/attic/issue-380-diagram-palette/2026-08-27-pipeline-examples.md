# Pipeline Examples Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #378 — Diagram editing infrastructure
**Issue group:** #378, #373 (property palette)

**Goal:** Create 3 showcase examples in the pages gallery demonstrating graph editing and property palette with a "Data Pipeline" domain (Source, Transform, Filter, Join, Sink nodes).

**Architecture:** Side-effect import in `casehub-entry.ts` registers pipeline stencils at bundle load. Each example is a YAML dashboard (using `type: html` for the canvas element) with a companion TS that configures the model and wiring. Graph packages and property palette are added to the examples build.

**Tech Stack:** TypeScript 5, lit-html (stencil templates), @xyflow/react (via graph-renderer), Webpack 5 (examples bundle), Vitest

## Global Constraints

- Pre-release project — no backward compatibility concerns
- `@typescript-eslint/strict-type-checked` — no `any` types in package source
- Companion TS scripts are runtime-stripped (imports removed, type annotations removed) — use `var` for variables, avoid lowercase type annotations, access bundle exports via `casehubPages.*` global
- Stencil render functions must use inline styles with CSS custom properties (no Shadow DOM, rendered in React Flow canvas)
- Test commands: `yarn workspace @casehubio/graph-renderer run test` and `yarn workspace @casehubio/graph-core run test`
- Build command: `GH_PACKAGES_TOKEN='' yarn build:packages && GH_PACKAGES_TOKEN='' yarn build:examples`

---

## Batch 1: Build infrastructure and first example

### Task 1: Package dependencies, webpack aliases, and pipeline stencils module

**Files:**
- Modify: `examples/package.json`
- Modify: `examples/webpack.config.js`
- Create: `examples/src/pipeline-stencils.ts`
- Modify: `examples/src/casehub-entry.ts`
- Modify: `examples/scripts/generate-samples.js`

**Interfaces:**
- Consumes: `registerStencil`, `StencilDescriptor`, `defaultEditPolicy` from `@casehubio/graph-renderer`; `GraphModel`, `GraphNode`, `registerGrammar` from `@casehubio/graph-core`; `html` from `lit-html`
- Produces: `createBasicPipelineModel(): GraphModel`, `PIPELINE_SCHEMAS: Record<string, object>` (exported via casehubPages UMD global)

- [ ] **Step 1: Add dependencies to examples/package.json**

Add to `dependencies`:
```json
"@casehubio/graph-core": "workspace:*",
"@casehubio/graph-renderer": "workspace:*",
"@casehubio/pages-property-palette": "workspace:*"
```

- [ ] **Step 2: Add webpack aliases for graph packages and missing ui-component subpaths**

In `examples/webpack.config.js`, add to the `alias` object:

```javascript
"@casehubio/graph-core": path.resolve(__dirname, "../packages/graph-core"),
"@casehubio/graph-renderer": path.resolve(__dirname, "../packages/graph-renderer"),
"@casehubio/pages-property-palette": path.resolve(__dirname, "../packages/pages-property-palette"),
"@casehubio/pages-ui-components/number-input": path.resolve(__dirname, "../packages/pages-ui-components/dist/number-input"),
"@casehubio/pages-ui-components/date-input": path.resolve(__dirname, "../packages/pages-ui-components/dist/date-input"),
"@casehubio/pages-ui-components/datetime-input": path.resolve(__dirname, "../packages/pages-ui-components/dist/datetime-input"),
"@casehubio/pages-ui-components/color-swatch": path.resolve(__dirname, "../packages/pages-ui-components/dist/color-swatch"),
"@casehubio/pages-ui-components/slider": path.resolve(__dirname, "../packages/pages-ui-components/dist/slider"),
"@casehubio/pages-ui-components/tag-editor": path.resolve(__dirname, "../packages/pages-ui-components/dist/tag-editor"),
"@casehubio/pages-ui-components/duration-input": path.resolve(__dirname, "../packages/pages-ui-components/dist/duration-input"),
"@casehubio/pages-ui-components/validation": path.resolve(__dirname, "../packages/pages-ui-components/dist/validation"),
```

Also add a `sideEffects` rule for graph-renderer so tree-shaking preserves stencil registration:
```javascript
{
  test: /graph-renderer[\/]dist[\/]/,
  sideEffects: true,
},
```

- [ ] **Step 3: Create pipeline-stencils.ts**

Create `examples/src/pipeline-stencils.ts`:

```typescript
import { html } from 'lit-html';
import { registerStencil } from '@casehubio/graph-renderer';
import type { GraphModel, GraphNode } from '@casehubio/graph-core';
import type { StencilRenderFn } from '@casehubio/graph-renderer';

function nodeLabel(node: GraphNode, fallback: string): string {
  const name = node.properties['name'];
  return typeof name === 'string' && name.length > 0 ? name : fallback;
}

function stencilStyle(bg: string, fg: string): string {
  return `display:flex;flex-direction:column;align-items:center;justify-content:center;padding:8px 16px;min-width:100px;border-radius:8px;background:${bg};color:${fg};font-family:var(--pages-font-family,system-ui)`;
}

const renderSource: StencilRenderFn = (node) => html`
  <div style="${stencilStyle('var(--pages-success-9,#16a34a)', 'var(--pages-success-12,#fff)')}">
    <span style="font-size:20px;line-height:1">⬇</span>
    <span style="font-size:11px;margin-top:4px;white-space:nowrap">${nodeLabel(node, 'Source')}</span>
  </div>`;

const renderTransform: StencilRenderFn = (node) => html`
  <div style="${stencilStyle('var(--pages-accent-9,#5470c6)', 'var(--pages-accent-12,#fff)')}">
    <span style="font-size:20px;line-height:1">⚙</span>
    <span style="font-size:11px;margin-top:4px;white-space:nowrap">${nodeLabel(node, 'Transform')}</span>
  </div>`;

const renderFilter: StencilRenderFn = (node) => html`
  <div style="${stencilStyle('var(--pages-warning-9,#ca8a04)', 'var(--pages-warning-12,#fff)')}">
    <span style="font-size:20px;line-height:1">⧖</span>
    <span style="font-size:11px;margin-top:4px;white-space:nowrap">${nodeLabel(node, 'Filter')}</span>
  </div>`;

const renderJoin: StencilRenderFn = (node) => html`
  <div style="${stencilStyle('var(--pages-info-9,#0891b2)', 'var(--pages-info-12,#fff)')}">
    <span style="font-size:20px;line-height:1">⨝</span>
    <span style="font-size:11px;margin-top:4px;white-space:nowrap">${nodeLabel(node, 'Join')}</span>
  </div>`;

const renderSink: StencilRenderFn = (node) => html`
  <div style="${stencilStyle('var(--pages-danger-9,#dc2626)', 'var(--pages-danger-12,#fff)')}">
    <span style="font-size:20px;line-height:1">⬆</span>
    <span style="font-size:11px;margin-top:4px;white-space:nowrap">${nodeLabel(node, 'Sink')}</span>
  </div>`;

registerStencil({
  type: 'source', label: 'Source', icon: '⬇', render: renderSource,
  grammar: {
    type: 'source',
    connections: {
      inbound: { min: 0, max: 0, allowedFrom: [] },
      outbound: { min: 0, max: 2, allowedTo: ['transform', 'filter', 'join'] },
    },
  },
});

registerStencil({
  type: 'transform', label: 'Transform', icon: '⚙', render: renderTransform,
  grammar: {
    type: 'transform',
    connections: {
      inbound: { min: 0, max: 3, allowedFrom: [] },
      outbound: { min: 0, max: 2, allowedTo: ['transform', 'filter', 'join', 'sink'] },
    },
  },
});

registerStencil({
  type: 'filter', label: 'Filter', icon: '⧖', render: renderFilter,
  grammar: {
    type: 'filter',
    connections: {
      inbound: { min: 0, max: 1, allowedFrom: [] },
      outbound: { min: 0, max: 2, allowedTo: ['transform', 'filter', 'join', 'sink'] },
    },
  },
});

registerStencil({
  type: 'join', label: 'Join', icon: '⨝', render: renderJoin,
  grammar: {
    type: 'join',
    connections: {
      inbound: { min: 0, max: 4, allowedFrom: [] },
      outbound: { min: 0, max: 1, allowedTo: ['transform', 'filter', 'sink'] },
    },
  },
});

registerStencil({
  type: 'sink', label: 'Sink', icon: '⬆', render: renderSink,
  grammar: {
    type: 'sink',
    connections: {
      inbound: { min: 0, max: 2, allowedFrom: [] },
      outbound: { min: 0, max: 0, allowedTo: [] },
    },
  },
});

export function createBasicPipelineModel(): GraphModel {
  return {
    nodes: [
      { id: 'src1', type: 'source', properties: { name: 'API Source' } },
      { id: 'tx1', type: 'transform', properties: { name: 'Parse JSON' } },
      { id: 'fl1', type: 'filter', properties: { name: 'Valid Records' } },
      { id: 'sk1', type: 'sink', properties: { name: 'Database' } },
      { id: 'sk2', type: 'sink', properties: { name: 'Error Log' } },
    ],
    edges: [
      { id: 'e1', type: 'default', source: 'src1', target: 'tx1' },
      { id: 'e2', type: 'default', source: 'tx1', target: 'fl1' },
      { id: 'e3', type: 'default', source: 'fl1', target: 'sk1' },
      { id: 'e4', type: 'default', source: 'fl1', target: 'sk2' },
    ],
  };
}

export const PIPELINE_SCHEMAS: Record<string, object> = {
  source: {
    type: 'object',
    properties: {
      name: { type: 'string', title: 'Name' },
      url: { type: 'string', title: 'URL' },
      format: { type: 'string', title: 'Format', enum: ['json', 'csv', 'xml'] },
      pollIntervalSec: { type: 'number', title: 'Poll Interval (sec)', minimum: 1 },
    },
    required: ['name'],
  },
  transform: {
    type: 'object',
    properties: {
      name: { type: 'string', title: 'Name' },
      expression: { type: 'string', title: 'Expression' },
      language: { type: 'string', title: 'Language', enum: ['jsonata', 'jq', 'javascript'] },
    },
    required: ['name'],
  },
  filter: {
    type: 'object',
    properties: {
      name: { type: 'string', title: 'Name' },
      condition: { type: 'string', title: 'Condition' },
      passLabel: { type: 'string', title: 'Pass Label' },
      failLabel: { type: 'string', title: 'Fail Label' },
    },
    required: ['name'],
  },
  join: {
    type: 'object',
    properties: {
      name: { type: 'string', title: 'Name' },
      strategy: { type: 'string', title: 'Strategy', enum: ['merge', 'zip', 'concat'] },
      windowMs: { type: 'number', title: 'Window (ms)', minimum: 0 },
    },
    required: ['name'],
  },
  sink: {
    type: 'object',
    properties: {
      name: { type: 'string', title: 'Name' },
      url: { type: 'string', title: 'URL' },
      format: { type: 'string', title: 'Format', enum: ['json', 'csv'] },
      batchSize: { type: 'number', title: 'Batch Size', minimum: 1 },
    },
    required: ['name'],
  },
};
```

- [ ] **Step 4: Update casehub-entry.ts**

Add these imports and exports to `examples/src/casehub-entry.ts`:

```typescript
import '@casehubio/graph-renderer';
import '@casehubio/pages-property-palette';
import { createBasicPipelineModel, PIPELINE_SCHEMAS } from './pipeline-stencils.js';

export { createBasicPipelineModel, PIPELINE_SCHEMAS };
export { defaultEditPolicy } from '@casehubio/graph-renderer';
```

- [ ] **Step 5: Add 'Graph Editing' to category order in generate-samples.js**

In `examples/scripts/generate-samples.js`, add `'Graph Editing'` to the `CATEGORY_ORDER` array after `'Interactivity'`:

```javascript
const CATEGORY_ORDER = [
  'Charts',
  'Tables',
  'Metrics',
  'Maps',
  'Forms',
  'Layout',
  'Interactivity',
  'Graph Editing',
  'Live Data',
  'Content',
  'Custom Components',
  'Theming',
  'Monitoring',
  'Domain Showcases',
  'Server',
];
```

- [ ] **Step 6: Verify build**

Run: `GH_PACKAGES_TOKEN='' yarn install && GH_PACKAGES_TOKEN='' yarn build:packages && GH_PACKAGES_TOKEN='' yarn build:examples`
Expected: Build succeeds without errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/package.json examples/webpack.config.js examples/src/pipeline-stencils.ts examples/src/casehub-entry.ts examples/scripts/generate-samples.js
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add pipeline stencil domain for graph editing gallery examples Refs #378"
```

### Task 2: Basic Pipeline example

**Files:**
- Create: `examples/samples/Graph Editing/Basic Pipeline.dash.yaml`
- Create: `examples/samples/Graph Editing/Basic Pipeline.ts`

**Interfaces:**
- Consumes: `createBasicPipelineModel()` from `casehubPages` UMD global; `<pages-graph-canvas>` custom element (registered by bundle)

- [ ] **Step 1: Create the YAML dashboard**

Create `examples/samples/Graph Editing/Basic Pipeline.dash.yaml`:

```yaml
pages:
  - name: Basic Pipeline
    components:
      - type: markdown
        properties:
          content: |
            Static rendering of a data pipeline graph with custom stencils and ELK auto-layout.
            Five node types (Source, Transform, Filter, Join, Sink) with distinct colors and icons.

      - type: html
        properties:
          content: |
            <pages-graph-canvas id="pipeline-canvas" style="width: 100%; height: 500px; display: block; border: 1px solid var(--pages-neutral-4, #d1d5db); border-radius: 8px; overflow: hidden;"></pages-graph-canvas>
```

- [ ] **Step 2: Create the companion TS**

Create `examples/samples/Graph Editing/Basic Pipeline.ts`:

```typescript
var canvas = document.getElementById('pipeline-canvas');
if (canvas) {
  (canvas as any).model = (window as any).casehubPages.createBasicPipelineModel();
}
```

- [ ] **Step 3: Verify example appears in gallery**

Run: `node examples/scripts/generate-samples.js`
Check: `cat examples/samples.json | grep "Basic Pipeline"` shows the entry with `tsPath`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add "examples/samples/Graph Editing/"
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add Basic Pipeline graph editing gallery example Refs #378"
```

## Batch 2: Interactive examples

### Task 3: Interactive Editing example

**Files:**
- Create: `examples/samples/Graph Editing/Interactive Editing.dash.yaml`
- Create: `examples/samples/Graph Editing/Interactive Editing.ts`

**Interfaces:**
- Consumes: `createBasicPipelineModel()`, `defaultEditPolicy()` from `casehubPages` UMD global

- [ ] **Step 1: Create the YAML dashboard**

Create `examples/samples/Graph Editing/Interactive Editing.dash.yaml`:

```yaml
pages:
  - name: Interactive Editing
    components:
      - type: markdown
        properties:
          content: |
            Interactive data pipeline editor with constraint validation.

            **Try:** Drag from a node handle to draw connections • Delete a middle node (select + Delete key) to see auto-join • Right-click for context menus • Drag an edge endpoint to reconnect it

            Connection rules enforce domain constraints — Source nodes cannot connect to other Sources, Sink nodes have no outbound connections.

      - type: html
        properties:
          content: |
            <pages-graph-canvas id="edit-canvas" style="width: 100%; height: 450px; display: block; border: 1px solid var(--pages-neutral-4, #d1d5db); border-radius: 8px; overflow: hidden;"></pages-graph-canvas>
            <div style="margin-top: 8px; font-family: var(--pages-font-family, system-ui);">
              <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 4px;">
                <strong style="font-size: 13px; color: var(--pages-neutral-11, #374151);">Mutation Log</strong>
                <button id="clear-log" style="font-size: 11px; padding: 2px 8px; border: 1px solid var(--pages-neutral-5); border-radius: 4px; background: var(--pages-neutral-3); color: var(--pages-neutral-11); cursor: pointer;">Clear</button>
              </div>
              <pre id="mutation-log" style="font-size: 11px; max-height: 120px; overflow-y: auto; padding: 8px; background: var(--pages-neutral-2, #1e1e1e); border-radius: 4px; color: var(--pages-neutral-11, #d4d4d4); margin: 0;"></pre>
            </div>
```

- [ ] **Step 2: Create the companion TS**

Create `examples/samples/Graph Editing/Interactive Editing.ts`:

```typescript
var editCanvas = document.getElementById('edit-canvas');
var mutationLog = document.getElementById('mutation-log');
var clearBtn = document.getElementById('clear-log');

if (editCanvas) {
  (editCanvas as any).model = (window as any).casehubPages.createBasicPipelineModel();
  (editCanvas as any).editPolicy = (window as any).casehubPages.defaultEditPolicy();

  (editCanvas as any).onMutation = function(edit: any) {
    if (mutationLog) {
      var entry = new Date().toLocaleTimeString() + '  ' + JSON.stringify(edit) + '\n';
      mutationLog.textContent = entry + (mutationLog.textContent || '');
    }
  };
}

if (clearBtn && mutationLog) {
  clearBtn.addEventListener('click', function() {
    mutationLog.textContent = '';
  });
}
```

- [ ] **Step 3: Verify example appears in gallery**

Run: `node examples/scripts/generate-samples.js`
Check: `cat examples/samples.json | grep "Interactive Editing"` shows the entry with `tsPath`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add "examples/samples/Graph Editing/Interactive Editing.dash.yaml" "examples/samples/Graph Editing/Interactive Editing.ts"
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add Interactive Editing graph editing gallery example Refs #378"
```

### Task 4: Pipeline Properties example

**Files:**
- Create: `examples/samples/Graph Editing/Pipeline Properties.dash.yaml`
- Create: `examples/samples/Graph Editing/Pipeline Properties.ts`

**Interfaces:**
- Consumes: `createBasicPipelineModel()`, `defaultEditPolicy()`, `PIPELINE_SCHEMAS` from `casehubPages` UMD global; `<pages-property-palette>` custom element (registered by bundle)

- [ ] **Step 1: Create the YAML dashboard**

Create `examples/samples/Graph Editing/Pipeline Properties.dash.yaml`:

```yaml
pages:
  - name: Pipeline Properties
    components:
      - type: markdown
        properties:
          content: |
            Click a node to view and edit its properties in the side panel.
            The property palette renders a schema-driven form based on the selected node's type.

      - type: html
        properties:
          content: |
            <div style="display: flex; gap: 12px; height: 520px;">
              <pages-graph-canvas id="props-canvas" style="flex: 1; display: block; border: 1px solid var(--pages-neutral-4, #d1d5db); border-radius: 8px; overflow: hidden;"></pages-graph-canvas>
              <div id="props-panel" style="width: 280px; border: 1px solid var(--pages-neutral-4, #d1d5db); border-radius: 8px; padding: 12px; overflow-y: auto; background: var(--pages-neutral-2, #fafafa);">
                <div id="props-empty" style="color: var(--pages-neutral-8, #9ca3af); font-size: 13px; font-family: var(--pages-font-family, system-ui); text-align: center; margin-top: 40px;">
                  Click a node to view its properties
                </div>
                <div id="props-header" style="display: none; margin-bottom: 8px;">
                  <span id="props-type" style="font-size: 11px; text-transform: uppercase; letter-spacing: 0.05em; color: var(--pages-neutral-8, #9ca3af); font-family: var(--pages-font-family, system-ui);"></span>
                  <div id="props-name" style="font-size: 15px; font-weight: 600; color: var(--pages-neutral-12, #111); font-family: var(--pages-font-family, system-ui);"></div>
                </div>
                <pages-property-palette id="props-palette" style="display: none;"></pages-property-palette>
              </div>
            </div>
```

- [ ] **Step 2: Create the companion TS**

Create `examples/samples/Graph Editing/Pipeline Properties.ts`:

```typescript
var propsCanvas = document.getElementById('props-canvas');
var propsEmpty = document.getElementById('props-empty');
var propsHeader = document.getElementById('props-header');
var propsType = document.getElementById('props-type');
var propsName = document.getElementById('props-name');
var propsPalette = document.getElementById('props-palette');

var currentModel = (window as any).casehubPages.createBasicPipelineModel();
var schemas = (window as any).casehubPages.PIPELINE_SCHEMAS;

if (propsCanvas) {
  (propsCanvas as any).model = currentModel;
  (propsCanvas as any).editPolicy = (window as any).casehubPages.defaultEditPolicy();

  propsCanvas.addEventListener('pages-event', function(e: any) {
    var detail = e.detail;
    if (detail.topic !== 'graph:node:click') return;

    var nodeId = detail.payload.nodeId;
    var node = currentModel.nodes.find(function(n: any) { return n.id === nodeId; });
    if (!node || !schemas[node.type]) return;

    if (propsEmpty) propsEmpty.style.display = 'none';
    if (propsHeader) propsHeader.style.display = 'block';
    if (propsType) propsType.textContent = node.type;
    if (propsName) propsName.textContent = String(node.properties.name || node.type);
    if (propsPalette) {
      (propsPalette as any).style.display = 'block';
      (propsPalette as any).source = {
        schema: schemas[node.type],
        data: Object.assign({}, node.properties),
        onChange: function(field: any, value: any) {
          node.properties[field[0]] = value;
          if (field[0] === 'name' && propsName) {
            propsName.textContent = String(value);
          }
        },
      };
    }
  });
}
```

- [ ] **Step 3: Verify example appears in gallery**

Run: `node examples/scripts/generate-samples.js`
Check: `cat examples/samples.json | grep "Pipeline Properties"` shows the entry with `tsPath`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add "examples/samples/Graph Editing/Pipeline Properties.dash.yaml" "examples/samples/Graph Editing/Pipeline Properties.ts"
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: add Pipeline Properties graph editing gallery example Refs #378"
```

## Batch 3: Full build verification

### Task 5: End-to-end build and gallery verification

**Files:**
- No new files — verification only

- [ ] **Step 1: Run full build**

Run: `GH_PACKAGES_TOKEN='' yarn build:packages && GH_PACKAGES_TOKEN='' yarn build:examples`
Expected: Build succeeds

- [ ] **Step 2: Generate samples index**

Run: `node examples/scripts/generate-samples.js`
Expected: Output shows "Graph Editing" category with 3 samples

- [ ] **Step 3: Verify samples.json**

Run: `grep -A 2 "Graph Editing" examples/samples.json`
Expected: Category with 3 entries (Basic Pipeline, Interactive Editing, Pipeline Properties), each with `tsPath`

- [ ] **Step 4: Serve gallery and visually verify**

Run: `GH_PACKAGES_TOKEN='' yarn workspace @casehubio/pages-examples run serve`
Open browser, navigate to Graph Editing category:
- Basic Pipeline: renders a 5-node graph with colored stencils and auto-layout
- Interactive Editing: renders the same graph with connection handles visible; mutation log panel below
- Pipeline Properties: renders the graph with property panel on the right; clicking a node shows the property form

- [ ] **Step 5: Commit (if any fixes needed)**

```bash
git -C /Users/mdproctor/claude/casehub/pages add -A
git -C /Users/mdproctor/claude/casehub/pages commit -m "chore: build verification fixes for graph editing examples Refs #378"
```

---

## References

- [specs/issue-378-diagram-editing-infra/2026-08-27-pipeline-examples-design.md] — design spec
- [specs/diagram-editing-infrastructure/2026-08-26-diagram-editing-infrastructure-design.md] — parent editing infrastructure spec
- [packages/graph-renderer/src/editing/types.ts] — EditPolicy, GraphEdit types
- [packages/graph-renderer/src/editing/edit-policy.ts] — defaultEditPolicy
- [packages/graph-renderer/src/bridge/GraphCanvas.ts] — pages-graph-canvas element
- [packages/graph-renderer/src/registry/stencil-registry.ts] — registerStencil, StencilDescriptor
- [packages/pages-property-palette/src/types.ts] — PropertyPaletteSource
- [examples/src/casehub-entry.ts] — UMD bundle entry
- [examples/src/app.js] — gallery companion loading (stripTs)
- [examples/scripts/generate-samples.js] — sample index generation
- [examples/webpack.config.js] — webpack aliases
- GE-20260803-50ddbd — lit-html → React Flow bridge pattern
- GitHub #378 — diagram editing infrastructure
- GitHub #373 — property palette
