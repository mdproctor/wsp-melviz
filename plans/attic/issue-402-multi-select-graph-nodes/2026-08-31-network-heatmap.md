# Network Heatmap Component — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #302 — pages-viz: composite network heatmap component (graph + density overlay)
**Issue group:** #302

**Goal:** Build a `PagesNetworkHeatmap` web component that composites a graph topology with a `@drdreo/heatmap` density overlay, showing traffic intensity along network edges.

**Architecture:** New `PagesElement<NetworkHeatmapProps>` subclass in pages-viz. Extracts edge data (source/target/traffic + coordinates) from a TypedDataSet, deduplicates nodes, interpolates heatmap points along edges weighted by traffic, renders via `@drdreo/heatmap` (base layer) with SVG overlay for node circles and labels. Follows `PagesDensityHeatmap` pattern exactly.

**Tech Stack:** TypeScript, Lit, @drdreo/heatmap, Vitest

## Global Constraints

- Extends `PagesElement<NetworkHeatmapProps>`, not raw LitElement
- Uses `@customElement("pages-network-heatmap")` decorator
- Lazy-loads `@drdreo/heatmap` (same module cache as PagesDensityHeatmap)
- Node positions from dataset columns — no force layout

---

## Batch 1: Type foundation and component

### Task 1: Add NetworkHeatmapProps and type guard

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts:200`
- Modify: `packages/pages-component/src/model/type-guards.ts:101,284`
- Modify: `packages/pages-component/src/model/index.ts:78`

**Interfaces:**
- Consumes: `DataComponentCommon`, `ColumnId` from displayer-types
- Produces: `NetworkHeatmapProps` interface, `isNetworkHeatmap()` type guard, `"network-heatmap"` type map entry

- [ ] **Step 1: Add NetworkHeatmapProps interface**

In `packages/pages-component/src/model/displayer-types.ts`, after `DensityHeatmapProps` (after line 200), add:

```typescript
export interface NetworkHeatmapProps extends DataComponentCommon {
  readonly sourceColumn?: ColumnId;
  readonly targetColumn?: ColumnId;
  readonly trafficColumn?: ColumnId;
  readonly sourceXColumn?: ColumnId;
  readonly sourceYColumn?: ColumnId;
  readonly targetXColumn?: ColumnId;
  readonly targetYColumn?: ColumnId;
  readonly nodeLabelColumn?: ColumnId;
  readonly showEdgeLines?: boolean;
  readonly showNodeLabels?: boolean;
  readonly pointsPerEdge?: number;
  readonly gradient?: readonly { readonly offset: number; readonly color: string }[];
  readonly radius?: number;
}
```

- [ ] **Step 2: Add type map entry and type guard**

In `packages/pages-component/src/model/type-guards.ts`:

Add to the `ComponentTypeMap` interface (after the `"density-heatmap"` entry at line 101):
```typescript
  "network-heatmap": NetworkHeatmapProps;
```

Add `NetworkHeatmapProps` to the import from `displayer-types.js`.

Add type guard function (after `isDensityHeatmap` at line 284):
```typescript
export function isNetworkHeatmap(c: Component): c is TypedComponent<"network-heatmap"> {
  return c.type === "network-heatmap";
}
```

- [ ] **Step 3: Export from index**

In `packages/pages-component/src/model/index.ts`, add `NetworkHeatmapProps` to the export list from `displayer-types.js` (after `DensityHeatmapProps` at line 78) and add `isNetworkHeatmap` to the export list from `type-guards.js`.

- [ ] **Step 4: Verify typecheck**

Run: `yarn typecheck`
Expected: No new type errors

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/displayer-types.ts packages/pages-component/src/model/type-guards.ts packages/pages-component/src/model/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#302): add NetworkHeatmapProps and type guard

New component type 'network-heatmap' for composite graph + density
overlay visualization.

Refs #302"
```

### Task 2: Implement PagesNetworkHeatmap component with tests

**Files:**
- Create: `packages/pages-viz/src/charts/PagesNetworkHeatmap.ts`
- Create: `packages/pages-viz/src/charts/PagesNetworkHeatmap.test.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`
- Modify: `packages/pages-viz/src/index.ts`

**Interfaces:**
- Consumes: `NetworkHeatmapProps` from `@casehubio/pages-component`, `PagesElement` from `../base/PagesElement.js`, `cellToRaw` from `../base/cell-extract.js`, `@drdreo/heatmap` module
- Produces: `PagesNetworkHeatmap` class, `pages-network-heatmap` custom element

- [ ] **Step 1: Write test file with edge extraction and point interpolation tests**

Create `packages/pages-viz/src/charts/PagesNetworkHeatmap.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import type { DataSet, TypedDataSet, ColumnType, ColumnId } from "@casehubio/pages-data";
import type { NetworkHeatmapProps } from "@casehubio/pages-component";
import { toTypedDataSet } from "@casehubio/pages-data";

const mockHeatmap = {
  setData: vi.fn(),
  destroy: vi.fn(),
};

vi.mock("@drdreo/heatmap", () => ({
  createHeatmap: vi.fn(() => mockHeatmap),
  withTooltip: vi.fn(() => ({ type: "tooltip" })),
  withLegend: vi.fn(() => ({ type: "legend" })),
}));

import { createHeatmap } from "@drdreo/heatmap";
import { PagesNetworkHeatmap } from "./PagesNetworkHeatmap.js";

function makeDataSet(columns: [string, string][], rows: (string | number | null)[][]): TypedDataSet {
  const ds: DataSet = {
    columns: columns.map(([id, type]) => ({
      id: id as ColumnId,
      name: id,
      type: type as ColumnType,
    })),
    data: rows.map(row => row.map(cell => cell === null ? null : String(cell))),
  };
  return toTypedDataSet(ds);
}

function mockContainerSize(el: PagesNetworkHeatmap): void {
  const shadow = el.shadowRoot;
  if (!shadow) return;
  const container = shadow.querySelector("div");
  if (!container) return;
  Object.defineProperty(container, "offsetWidth", { value: 500, configurable: true });
  Object.defineProperty(container, "offsetHeight", { value: 400, configurable: true });
}

async function renderChart(el: PagesNetworkHeatmap, props: NetworkHeatmapProps, ds: TypedDataSet): Promise<void> {
  el.props = props;
  document.body.appendChild(el);
  await el.updateComplete;
  el.dataSet = ds;
  await el.updateComplete;
  mockContainerSize(el);
  el.requestUpdate();
  await el.updateComplete;
  await new Promise(r => { setTimeout(r, 0); });
  await el.updateComplete;
}

describe("PagesNetworkHeatmap", () => {
  let el: PagesNetworkHeatmap;

  beforeEach(() => {
    vi.clearAllMocks();
    el = document.createElement("pages-network-heatmap") as PagesNetworkHeatmap;
  });

  afterEach(() => {
    if (el.isConnected) el.remove();
  });

  it("creates heatmap with interpolated edge points", async () => {
    const ds = makeDataSet(
      [["source", "LABEL"], ["target", "LABEL"], ["traffic", "NUMBER"],
       ["sx", "NUMBER"], ["sy", "NUMBER"], ["tx", "NUMBER"], ["ty", "NUMBER"]],
      [["A", "B", "100", "0", "0", "100", "100"]],
    );

    await renderChart(el, {
      lookup: { dataSetId: "test", operations: [] } as never,
      sourceColumn: "source" as ColumnId,
      targetColumn: "target" as ColumnId,
      trafficColumn: "traffic" as ColumnId,
      sourceXColumn: "sx" as ColumnId,
      sourceYColumn: "sy" as ColumnId,
      targetXColumn: "tx" as ColumnId,
      targetYColumn: "ty" as ColumnId,
      pointsPerEdge: 5,
    }, ds);

    expect(createHeatmap).toHaveBeenCalled();
    const callArgs = vi.mocked(createHeatmap).mock.calls[0]![0] as { data: { x: number; y: number; value: number }[] };
    expect(callArgs.data.length).toBe(5);
  });

  it("deduplicates nodes from edge pairs", async () => {
    const ds = makeDataSet(
      [["source", "LABEL"], ["target", "LABEL"], ["traffic", "NUMBER"],
       ["sx", "NUMBER"], ["sy", "NUMBER"], ["tx", "NUMBER"], ["ty", "NUMBER"]],
      [
        ["A", "B", "50", "0", "0", "100", "0"],
        ["A", "C", "30", "0", "0", "50", "100"],
      ],
    );

    await renderChart(el, {
      lookup: { dataSetId: "test", operations: [] } as never,
      sourceColumn: "source" as ColumnId,
      targetColumn: "target" as ColumnId,
      trafficColumn: "traffic" as ColumnId,
      sourceXColumn: "sx" as ColumnId,
      sourceYColumn: "sy" as ColumnId,
      targetXColumn: "tx" as ColumnId,
      targetYColumn: "ty" as ColumnId,
      showNodeLabels: true,
    }, ds);

    const svg = el.shadowRoot?.querySelector("svg");
    expect(svg).toBeTruthy();
    const circles = svg!.querySelectorAll("circle");
    expect(circles.length).toBe(3);
  });

  it("renders edge lines when showEdgeLines is true", async () => {
    const ds = makeDataSet(
      [["source", "LABEL"], ["target", "LABEL"], ["traffic", "NUMBER"],
       ["sx", "NUMBER"], ["sy", "NUMBER"], ["tx", "NUMBER"], ["ty", "NUMBER"]],
      [["A", "B", "100", "0", "0", "100", "100"]],
    );

    await renderChart(el, {
      lookup: { dataSetId: "test", operations: [] } as never,
      sourceColumn: "source" as ColumnId,
      targetColumn: "target" as ColumnId,
      trafficColumn: "traffic" as ColumnId,
      sourceXColumn: "sx" as ColumnId,
      sourceYColumn: "sy" as ColumnId,
      targetXColumn: "tx" as ColumnId,
      targetYColumn: "ty" as ColumnId,
      showEdgeLines: true,
    }, ds);

    const svg = el.shadowRoot?.querySelector("svg");
    const lines = svg!.querySelectorAll("line");
    expect(lines.length).toBe(1);
  });

  it("cleans up heatmap on disconnect", async () => {
    const ds = makeDataSet(
      [["source", "LABEL"], ["target", "LABEL"], ["traffic", "NUMBER"],
       ["sx", "NUMBER"], ["sy", "NUMBER"], ["tx", "NUMBER"], ["ty", "NUMBER"]],
      [["A", "B", "100", "0", "0", "100", "100"]],
    );

    await renderChart(el, {
      lookup: { dataSetId: "test", operations: [] } as never,
      sourceColumn: "source" as ColumnId,
      targetColumn: "target" as ColumnId,
      trafficColumn: "traffic" as ColumnId,
      sourceXColumn: "sx" as ColumnId,
      sourceYColumn: "sy" as ColumnId,
      targetXColumn: "tx" as ColumnId,
      targetYColumn: "ty" as ColumnId,
    }, ds);

    el.remove();
    expect(mockHeatmap.destroy).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-viz run test -- --run PagesNetworkHeatmap.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PagesNetworkHeatmap**

Create `packages/pages-viz/src/charts/PagesNetworkHeatmap.ts`:

```typescript
import { html, css, svg, type TemplateResult } from "lit";
import { createRef, ref } from "lit/directives/ref.js";
import { customElement } from "lit/decorators.js";
import type { NetworkHeatmapProps } from "@casehubio/pages-component";
import type { TypedDataSet, TypedRow } from "@casehubio/pages-data";
import { PagesElement } from "../base/PagesElement.js";
import { cellToRaw } from "../base/cell-extract.js";

type HeatmapModule = typeof import("@drdreo/heatmap");
let heatmapModule: HeatmapModule | undefined;

async function loadHeatmapModule(): Promise<HeatmapModule> {
  if (!heatmapModule) {
    heatmapModule = await import("@drdreo/heatmap");
  }
  return heatmapModule;
}

interface HeatmapPoint { x: number; y: number; value: number; }
interface HeatmapInstance { setData(data: HeatmapPoint[]): void; destroy(): void; }
interface NetworkNode { id: string; x: number; y: number; label: string; }
interface NetworkEdge { source: NetworkNode; target: NetworkNode; traffic: number; }

@customElement("pages-network-heatmap")
export class PagesNetworkHeatmap extends PagesElement<NetworkHeatmapProps> {
  static override styles = css`
    ${PagesElement.styles}
    :host { display: block; position: relative; }
    .network-container { width: 100%; height: 100%; position: relative; }
    .network-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; }
  `;

  private _containerRef = createRef<HTMLDivElement>();
  private _heatmap: HeatmapInstance | undefined;

  override connectedCallback(): void {
    super.connectedCallback();
    this.setAttribute("aria-label", this.props?.title ?? "Network heatmap");
  }

  override render(): TemplateResult {
    if (this.props) this._applySizing(this.props);
    return super.render();
  }

  protected override renderContent(props: NetworkHeatmapProps, dataset: TypedDataSet): TemplateResult {
    const { nodes, edges } = this._extractGraph(dataset, props);
    const container = this._containerRef.value;
    const w = container?.offsetWidth ?? 500;
    const h = container?.offsetHeight ?? 400;
    const pad = 30;
    const { nodeMap, scaleX, scaleY } = this._normalizePositions(nodes, w, h, pad);

    return html`
      <div ${ref(this._containerRef)} class="network-container"></div>
      <svg class="network-overlay" viewBox="0 0 ${w} ${h}">
        ${props.showEdgeLines ? edges.map(e => {
          const s = nodeMap.get(e.source.id)!;
          const t = nodeMap.get(e.target.id)!;
          return svg`<line x1="${s.x}" y1="${s.y}" x2="${t.x}" y2="${t.y}" stroke="rgba(255,255,255,0.3)" stroke-width="1"/>`;
        }) : ""}
        ${Array.from(nodeMap.values()).map(n => svg`
          <circle cx="${n.x}" cy="${n.y}" r="5" fill="var(--pages-interactive, #2563eb)"/>
          ${props.showNodeLabels !== false ? svg`
            <text x="${n.x}" y="${n.y - 10}" text-anchor="middle" fill="var(--pages-text-primary, #e5e5e5)" font-size="11">${n.label}</text>
          ` : ""}
        `)}
      </svg>
    `;
  }

  override updated(): void {
    const container = this._containerRef.value;
    if (!container || !this.props || !this.dataSet) return;
    if (container.offsetWidth === 0 || container.offsetHeight === 0) return;
    void this._updateHeatmap(container);
  }

  private async _updateHeatmap(container: HTMLDivElement): Promise<void> {
    const mod = await loadHeatmapModule();
    if (!this.props || !this.dataSet) return;

    const { nodes, edges } = this._extractGraph(this.dataSet, this.props);
    const pad = 30;
    const { nodeMap } = this._normalizePositions(
      nodes, container.offsetWidth, container.offsetHeight, pad,
    );

    const pointsPerEdge = this.props.pointsPerEdge ?? 20;
    const heatPoints: HeatmapPoint[] = [];

    for (const edge of edges) {
      const s = nodeMap.get(edge.source.id)!;
      const t = nodeMap.get(edge.target.id)!;
      const valuePerPoint = edge.traffic / pointsPerEdge;
      for (let i = 0; i < pointsPerEdge; i++) {
        const frac = pointsPerEdge === 1 ? 0.5 : i / (pointsPerEdge - 1);
        heatPoints.push({
          x: s.x + (t.x - s.x) * frac,
          y: s.y + (t.y - s.y) * frac,
          value: valuePerPoint,
        });
      }
    }

    if (this._heatmap) {
      this._heatmap.setData(heatPoints);
    } else {
      const config: Record<string, unknown> = { container, data: heatPoints };
      if (this.props.gradient) config.gradient = this.props.gradient;
      if (this.props.radius) config.radius = this.props.radius;
      this._heatmap = mod.createHeatmap(config as never) as unknown as HeatmapInstance;
    }
  }

  private _extractGraph(
    dataset: TypedDataSet,
    props: NetworkHeatmapProps,
  ): { nodes: NetworkNode[]; edges: NetworkEdge[] } {
    const srcIdx = props.sourceColumn ? dataset.columns.findIndex(c => c.id === props.sourceColumn) : 0;
    const tgtIdx = props.targetColumn ? dataset.columns.findIndex(c => c.id === props.targetColumn) : 1;
    const trafIdx = props.trafficColumn ? dataset.columns.findIndex(c => c.id === props.trafficColumn) : 2;
    const sxIdx = props.sourceXColumn ? dataset.columns.findIndex(c => c.id === props.sourceXColumn) : 3;
    const syIdx = props.sourceYColumn ? dataset.columns.findIndex(c => c.id === props.sourceYColumn) : 4;
    const txIdx = props.targetXColumn ? dataset.columns.findIndex(c => c.id === props.targetXColumn) : 5;
    const tyIdx = props.targetYColumn ? dataset.columns.findIndex(c => c.id === props.targetYColumn) : 6;
    const lblIdx = props.nodeLabelColumn ? dataset.columns.findIndex(c => c.id === props.nodeLabelColumn) : -1;

    const nodeMap = new Map<string, NetworkNode>();
    const edges: NetworkEdge[] = [];

    for (const row of dataset.rows) {
      const srcId = String(cellToRaw(row.cells[srcIdx]!) ?? "");
      const tgtId = String(cellToRaw(row.cells[tgtIdx]!) ?? "");
      const traffic = Number(cellToRaw(row.cells[trafIdx]!) ?? 0);
      const sx = Number(cellToRaw(row.cells[sxIdx]!) ?? 0);
      const sy = Number(cellToRaw(row.cells[syIdx]!) ?? 0);
      const tx = Number(cellToRaw(row.cells[txIdx]!) ?? 0);
      const ty = Number(cellToRaw(row.cells[tyIdx]!) ?? 0);

      if (!nodeMap.has(srcId)) {
        const label = lblIdx >= 0 ? String(cellToRaw(row.cells[lblIdx]!) ?? srcId) : srcId;
        nodeMap.set(srcId, { id: srcId, x: sx, y: sy, label });
      }
      if (!nodeMap.has(tgtId)) {
        nodeMap.set(tgtId, { id: tgtId, x: tx, y: ty, label: tgtId });
      }

      edges.push({
        source: nodeMap.get(srcId)!,
        target: nodeMap.get(tgtId)!,
        traffic,
      });
    }

    return { nodes: Array.from(nodeMap.values()), edges };
  }

  private _normalizePositions(
    nodes: NetworkNode[],
    containerW: number,
    containerH: number,
    pad: number,
  ): { nodeMap: Map<string, { id: string; x: number; y: number; label: string }>; scaleX: number; scaleY: number } {
    let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;
    for (const n of nodes) {
      if (n.x < minX) minX = n.x;
      if (n.x > maxX) maxX = n.x;
      if (n.y < minY) minY = n.y;
      if (n.y > maxY) maxY = n.y;
    }

    const w = containerW - pad * 2;
    const h = containerH - pad * 2;
    const rangeX = maxX - minX || 1;
    const rangeY = maxY - minY || 1;
    const scaleX = w / rangeX;
    const scaleY = h / rangeY;

    const nodeMap = new Map<string, { id: string; x: number; y: number; label: string }>();
    for (const n of nodes) {
      nodeMap.set(n.id, {
        id: n.id,
        x: pad + (n.x - minX) * scaleX,
        y: pad + (n.y - minY) * scaleY,
        label: n.label,
      });
    }

    return { nodeMap, scaleX, scaleY };
  }

  private _applySizing(props: NetworkHeatmapProps): void {
    const raw = props as unknown as Readonly<Record<string, unknown>>;
    const h = raw.height;
    if (typeof h === "number") {
      this.style.minHeight = `${String(h)}px`;
      this.style.height = `${String(h)}px`;
    } else if (typeof h === "string") {
      this.style.minHeight = h;
      this.style.height = h;
    } else {
      this.style.minHeight = "400px";
    }
    const w = raw.width;
    if (typeof w === "number") {
      this.style.width = `${String(w)}px`;
    } else if (typeof w === "string") {
      this.style.width = w;
    } else {
      this.style.width = "100%";
    }
  }

  override onResize(): void {
    if (!this._heatmap || !this._containerRef.value || !this.props || !this.dataSet) return;
    this._heatmap.destroy();
    this._heatmap = undefined;
    void this._updateHeatmap(this._containerRef.value);
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    if (this._heatmap) {
      this._heatmap.destroy();
      this._heatmap = undefined;
    }
  }
}
```

- [ ] **Step 4: Register in custom-elements.ts and export from index.ts**

In `packages/pages-viz/src/custom-elements.ts`, add import and tag registration:

After the `PagesDensityHeatmap` import (line 18), add:
```typescript
import type { PagesNetworkHeatmap } from "./charts/PagesNetworkHeatmap.js";
```

In the `HTMLElementTagNameMap` declaration, after `"pages-density-heatmap"` (line 50), add:
```typescript
    "pages-network-heatmap": PagesNetworkHeatmap;
```

In `packages/pages-viz/src/index.ts`, after the `PagesDensityHeatmap` export (line 45), add:
```typescript
export { PagesNetworkHeatmap } from "./charts/PagesNetworkHeatmap.js";
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-viz run test -- --run PagesNetworkHeatmap.test.ts`
Expected: ALL tests pass

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: No new type errors

- [ ] **Step 7: Add gallery sample**

Create `examples/samples/Charts/Network Heatmap.dash.yaml`:

```yaml
datasets:
  - uuid: network
    content: >-
      [
        ["Gateway", "AppServer1", "850", "50", "50", "200", "30"],
        ["Gateway", "AppServer2", "620", "50", "50", "200", "80"],
        ["AppServer1", "Database", "400", "200", "30", "350", "55"],
        ["AppServer2", "Database", "310", "200", "80", "350", "55"],
        ["AppServer1", "Cache", "230", "200", "30", "350", "20"],
        ["Gateway", "CDN", "180", "50", "50", "50", "100"]
      ]
    columns:
      - id: source
        type: LABEL
      - id: target
        type: LABEL
      - id: traffic
        type: NUMBER
      - id: sx
        type: NUMBER
      - id: sy
        type: NUMBER
      - id: tx
        type: NUMBER
      - id: ty
        type: NUMBER

pages:
  - components:
      - type: network-heatmap
        properties:
          height: 500
          sourceColumn: source
          targetColumn: target
          trafficColumn: traffic
          sourceXColumn: sx
          sourceYColumn: sy
          targetXColumn: tx
          targetYColumn: ty
          showNodeLabels: true
          showEdgeLines: true
          pointsPerEdge: 30
          lookup:
            uuid: network
```

Add to `examples/samples.json` in the Charts category:
```json
{
  "name": "Network Heatmap",
  "path": "Charts/Network Heatmap.dash.yaml",
  "category": "Charts",
  "file": "Network Heatmap.dash.yaml"
}
```

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/charts/PagesNetworkHeatmap.ts packages/pages-viz/src/charts/PagesNetworkHeatmap.test.ts packages/pages-viz/src/custom-elements.ts packages/pages-viz/src/index.ts examples/samples/Charts/Network\ Heatmap.dash.yaml examples/samples.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#302): PagesNetworkHeatmap — graph topology with density overlay

New pages-network-heatmap web component. Extracts edge data from dataset
columns, interpolates heatmap points along edges weighted by traffic,
renders @drdreo/heatmap base layer with SVG overlay for node circles
and labels. Gallery sample included.

Closes #302"
```

## References

- [2026-08-31-network-heatmap-design.md] — design spec
- `packages/pages-viz/src/charts/PagesDensityHeatmap.ts` — heatmap integration pattern
- `packages/pages-viz/src/charts/PagesDensityHeatmap.test.ts` — test pattern
- `packages/pages-component/src/model/displayer-types.ts:191-200` — DensityHeatmapProps pattern
- `packages/pages-component/src/model/type-guards.ts:101,282` — type map + guard pattern
- `packages/pages-viz/src/custom-elements.ts` — tag registration
- `packages/pages-viz/src/index.ts` — barrel export
- `docs/protocols/casehub/web-component-strategy.md` — Lit + PagesElement convention
- [GitHub #302] — issue
