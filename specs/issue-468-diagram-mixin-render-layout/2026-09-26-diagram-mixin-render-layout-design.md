# Design: DiagramBaseMixin — Layout Hooks and Composable Template Methods

**Date:** 2026-09-26
**Issues:** #468 (render template + event wiring), #467 (layout hook)
**Package:** `pages-diagram-core`
**Status:** Draft

---

## 1. Problem

DiagramBaseMixin provides ~700 lines of diagram infrastructure (YAML adapter, ELK layout, undo/redo, persistence, property editing, node picker, stencil palette, conflict handling) but forces subclasses into two expensive patterns:

**Layout duplication (#467):** `_fullRender` hardcodes `computeElkLayout` at line 415. Subclasses needing a different layout strategy (SWF stack-column, org radial/auto-detect) must override the entire `_fullRender` method (~40 lines), copy-pasting the render pipeline and using `(this as any)` casts to access protected fields. SWF uses 10 such casts; org uses 8.

**Render duplication (#468):** The mixin provides no `render()`. Each diagram (case, SWF, org) assembles its own template with manual event wiring. The 5 common graph events (`graph:node:click`, `graph:edge:click`, `graph:selection:change`, `graph:pane:click`, `graph:connect:end-on-empty`) are copy-pasted across all three, and new events must be wired in every diagram.

## 2. Decisions

See `decisions.md` for the full decision log (D1–D4).

| # | Decision | Choice |
|---|---|---|
| D1 | Layout extensibility | Two hooks: `_computeLayout` + `_postLayout` |
| D2 | `_computeLayout` return type | `LayoutResult { layout, direction }` |
| D3 | Template approach | Composable methods, not monolithic shell |
| D4 | Event dispatcher | switch/case with super delegation |

## 3. Layout Pipeline Hooks (#467)

### 3.1 Current pipeline

```
_adaptYaml(yaml) → computeElkLayout(model, options) → toReactFlowGraph → store _nodes/_edges
```

`computeElkLayout` is called directly at `diagram-base-mixin.ts:415`. No extension point.

### 3.2 New pipeline

```
_adaptYaml(yaml) → _computeLayout(model, options) → toReactFlowGraph → _postLayout(nodes, edges) → store _nodes/_edges
```

### 3.3 `_computeLayout`

```typescript
export interface LayoutResult {
  readonly layout: ElkLayoutResult;
  readonly direction?: 'DOWN' | 'RIGHT' | 'LEFT' | 'UP';
}

// In DiagramBaseMixin:
protected async _computeLayout(
  model: GraphModel,
  options: ElkLayoutOptions,
): Promise<LayoutResult> {
  const layout = await computeElkLayout(model, options);
  return { layout, direction: options.direction };
}
```

**Default:** delegates to `computeElkLayout`, passes through the direction from options.

**SWF override:**
```typescript
protected override async _computeLayout(
  model: GraphModel,
  options: ElkLayoutOptions,
): Promise<LayoutResult> {
  const layout = computeSwfStackLayout(model);
  return { layout, direction: options.direction };
}
```

**Org override:**
```typescript
protected override async _computeLayout(
  model: GraphModel,
  _options: ElkLayoutOptions,
): Promise<LayoutResult> {
  const primary = this._archetypeHint?.layout ?? 'force';

  if (primary === 'hub-spoke' || primary === 'circular') {
    try {
      return { layout: computeRadialLayout(model), direction: undefined };
    } catch { /* fall through to ELK */ }
  }

  const opts = this._buildElkOpts(primary);
  try {
    return { layout: await computeElkLayout(model, opts), direction: opts.direction };
  } catch {
    const fallback = this._buildElkOpts('force');
    return { layout: await computeElkLayout(model, fallback), direction: fallback.direction };
  }
}
```

### 3.4 `_postLayout`

```typescript
protected _postLayout(
  nodes: Node[],
  edges: Edge[],
): { nodes: Node[]; edges: Edge[] } {
  return { nodes, edges };
}
```

**Default:** identity — returns nodes and edges unchanged.

**Org override:**
```typescript
protected override _postLayout(
  nodes: Node[],
  edges: Edge[],
): { nodes: Node[]; edges: Edge[] } {
  if (this._lastFacts) {
    this._engine.postLayout(nodes, edges, this._lastFacts);
  }
  this._baseEdges = [...edges];
  const styled = applyOrgEdgeLabels(edges);
  const highlighted = applySelectionHighlight(styled, this._selectedNodeId || undefined);
  return { nodes, edges: highlighted };
}
```

### 3.5 Updated `_fullRender`

```typescript
protected async _fullRender(yamlStr: string): Promise<void> {
  if (this._renderInProgress) {
    this._pendingRenderYaml = yamlStr;
    return;
  }
  this._renderInProgress = true;
  try {
    this._error = '';
    const result = this._adaptYaml(yamlStr);
    this._adapterResult = result;
    const { layout, direction } = await this._computeLayout(
      result.model, this._layoutOptions(),
    );
    if (this._adapterResult !== result) {
      this._renderInProgress = false;
      await this._fullRender(this._currentYaml);
      return;
    }
    this._lastLayout = layout;
    const rfGraph = toReactFlowGraph(
      result.model, layout, this._decorations(), direction,
    );
    const processed = this._postLayout(rfGraph.nodes, rfGraph.edges);
    this._nodes = processed.nodes;
    this._edges = processed.edges;
  } catch (e) {
    this._error = String(e);
  } finally {
    this._renderInProgress = false;
    if (this._pendingRenderYaml && this._pendingRenderYaml !== yamlStr) {
      const pending = this._pendingRenderYaml;
      this._pendingRenderYaml = '';
      await this._fullRender(pending);
    } else {
      this._pendingRenderYaml = '';
    }
  }
}
```

### 3.6 `_updateWithoutLayout` alignment

`_updateWithoutLayout` (line 439) also calls `toReactFlowGraph`. It should also call `_postLayout` so that post-processing (e.g. org edge styling) applies consistently on property-only edits:

```typescript
protected _updateWithoutLayout(yamlStr: string): void {
  if (!this._lastLayout) return;
  try {
    this._error = '';
    this._adapterResult = this._adaptYaml(yamlStr);
    const rfGraph = toReactFlowGraph(
      this._adapterResult.model, this._lastLayout,
      this._decorations(), this._layoutOptions().direction,
    );
    const processed = this._postLayout(rfGraph.nodes, rfGraph.edges);
    this._nodes = processed.nodes;
    this._edges = processed.edges;
    this._updateSelectedNode();
  } catch (e) {
    this._error = `Edit failed: ${e}`;
    this._currentYaml = this._undoStack.pop() ?? this._currentYaml;
  }
}
```

## 4. Composable Template Methods (#468)

### 4.1 `_handleCanvasEvent`

Central event dispatcher. Subclasses call `super._handleCanvasEvent(e)` then handle domain-specific events.

```typescript
protected _handleCanvasEvent = (e: CustomEvent): void => {
  const topic = e.detail?.topic;
  switch (topic) {
    case 'graph:node:click':
      this._handleNodeClick(e);
      break;
    case 'graph:edge:click':
      this._handleEdgeClick(e);
      break;
    case 'graph:selection:change':
      this._handleSelectionChange(e);
      break;
    case 'graph:pane:click':
      this._showPickerAtPaneClick();
      break;
    case 'graph:connect:end-on-empty':
      this._showPickerAtConnectEnd(e.detail?.payload);
      break;
  }
};
```

Arrow function (like existing handlers) so `this` binding is stable when passed as `@pages-event=${this._handleCanvasEvent}`.

### 4.2 `_handleEdgeClick`

New default handler — no-op. SWF overrides for the split-edge picker flow.

```typescript
protected _handleEdgeClick(_e: CustomEvent): void {
  // Subclasses override for edge-click behavior (e.g. split-edge picker)
}
```

### 4.3 `_renderCanvas`

Renders `pages-graph-canvas` with standard props and event wiring.

```typescript
protected _renderCanvas(): TemplateResult {
  return html`
    <pages-graph-canvas
      .nodes=${this._nodes}
      .edges=${this._edges}
      .model=${this._adapterResult?.model}
      .editPolicy=${this._editPolicy()}
      .onMutation=${this._handleMutation}
      style="width:100%;height:100%;"
      @pages-event=${this._handleCanvasEvent}
    ></pages-graph-canvas>
  `;
}
```

Subclasses override for:
- Filtered nodes/edges (SWF: `_computeFilteredNodes`/`_computeFilteredEdges`)
- Extra props (`miniMapNodeColor`, `connectionsEnabled`)
- Accessibility attributes (`role`, `aria-label`)
- Different canvas component (org: `graph-canvas-core` directly)

### 4.4 `_renderDialogs`

Convenience method for the two dialogs that appear in every diagram.

```typescript
protected _renderDialogs(): TemplateResult {
  return html`
    ${this._showConflict ? this._renderConflictDialog() : nothing}
    ${this._confirmMessage ? this._renderDeleteConfirm() : nothing}
  `;
}
```

### 4.5 Interface declaration updates

Add to `DiagramBaseInterface`:

```typescript
// Layout hooks
protected _computeLayout(model: GraphModel, options: ElkLayoutOptions): Promise<LayoutResult>;
protected _postLayout(nodes: Node[], edges: Edge[]): { nodes: Node[]; edges: Edge[] };

// Template methods
_handleCanvasEvent: (e: CustomEvent) => void;
_handleEdgeClick(e: CustomEvent): void;
_renderCanvas(): TemplateResult;
_renderDialogs(): TemplateResult;
```

### 4.6 Exports

`LayoutResult` is exported from `pages-diagram-core` for subclass type safety:

```typescript
// index.ts
export type { LayoutResult } from './diagram-base-mixin.js';
```

## 5. Impact on Consumers (blocks-ui)

Changes to the three diagram consumers are downstream — they live in `blocks-ui`, not this repo. The mixin changes are backward compatible: all new methods have defaults that preserve current behavior. Consumer migration is optional but recommended.

**After migration:**

| Diagram | Lines removed | `(this as any)` removed | Key change |
|---------|--------------|------------------------|------------|
| casehub-diagram | ~15 | 0 | Inline event handler → `super._handleCanvasEvent` + domain override |
| swf-diagram | ~35 | 10 | Full `_fullRender` override → `_computeLayout` override (5 lines) |
| blocks-org-diagram | ~70 | 8 | Full `_fullRender` override → `_computeLayout` + `_postLayout` overrides (~25 lines) |

## 6. Testing

- Existing `diagram-base-mixin.test.ts` — extend with:
  - `_computeLayout` is called instead of `computeElkLayout` directly
  - `_postLayout` receives `toReactFlowGraph` output and its return is stored
  - `_handleCanvasEvent` dispatches to correct handlers for each topic
  - `_handleEdgeClick` is called on `graph:edge:click` topic
  - `_renderCanvas` returns template with `pages-graph-canvas` and event binding
  - `_renderDialogs` renders conflict/delete dialogs conditionally
  - `_updateWithoutLayout` calls `_postLayout`

## References

- `packages/pages-diagram-core/src/diagram-base-mixin.ts` — mixin source
- `packages/graph-renderer/src/layout/elk-layout.ts` — `computeElkLayout`, `ElkLayoutResult`, `ElkLayoutOptions`
- `packages/graph-renderer/src/bridge/PagesGraphCanvas.ts` — canvas component API
- `packages/graph-renderer/src/bridge/GraphCanvas.ts` — event emissions (18 `pages-event` topics)
- `docs/protocols/casehub/pages-event-contract.md` — event naming conventions
- `docs/protocols/casehub/graph-core-pure-data.md` — graph-core boundary
- blocks-ui `components/casehub-diagram/src/casehub-diagram.ts` — case diagram consumer
- blocks-ui `components/swf-diagram/src/swf-diagram.ts` — SWF diagram consumer
- blocks-ui `components/org-diagram/src/blocks-org-diagram.ts` — org diagram consumer
