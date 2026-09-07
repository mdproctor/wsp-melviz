# Broaden SPI Interfaces Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #413 — Broaden SPI interfaces for chart, map, and graph components
**Issue group:** #413

**Goal:** Widen the typed SPI surface while preserving implementation-agnosticism — split ChartSettings along the Cartesian boundary, promote universal properties, add typed escape hatches, and register GraphCanvas as a new YAML component type.

**Architecture:** Extract `ChartSettingsBase` from `ChartSettings`, re-parent non-Cartesian charts, add universal properties (tooltip, animation, color, axis extensions), per-chart series options, typed escape hatches per backend (`echarts?`, `reactFlow?`, `elk?`, `heatmapJs?`), and a new `GraphCanvasProps` with YAML integration. The schema generator from #411 handles Zod schema production automatically — only interface changes needed.

**Tech Stack:** TypeScript, Lit, ECharts 5.6, @xyflow/react 12.4, elkjs 0.9, @drdreo/heatmap 1.4, Zod, ts-morph, Vitest

## Global Constraints

- SPI properties are universal/likely-portable (two-library mapping test per D1)
- `extra: Record<string, unknown>` stays as untyped last resort
- Typed escape hatches use curated subset types, not full upstream types
- Pre-release stage — no backward compatibility constraints
- After every interface change, regenerate schemas: `yarn workspace @casehubio/pages-schema run generate`
- Every file mentioned is relative to project root: `/Users/mdproctor/claude/casehub/slots/177/pages`

---

## Batch 1: Foundation — Interface hierarchy and base class refactoring

### Task 1: Extract ChartSettingsBase and restructure interface hierarchy

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts:6-226`
- Modify: `packages/pages-component/src/model/index.ts:56` (add `ChartSettingsBase` export)

**Interfaces:**
- Produces: `ChartSettingsBase` interface (resizable, legend, margin, extra), `ChartSettings extends ChartSettingsBase` (xAxis, yAxis, grid, zoom), `DataComponentCommon` + maxWidth/maxHeight

- [ ] **Step 1: Write failing test — ChartSettingsBase exists as a type**

File: `packages/pages-component/src/model/displayer-types.test.ts` (create)

```typescript
import { describe, it, expectTypeOf } from "vitest";
import type { ChartSettingsBase, ChartSettings, DataComponentCommon, PieChartProps, MapProps, GraphProps, MeterProps, TreemapChartProps } from "./displayer-types.js";

describe("ChartSettingsBase hierarchy", () => {
  it("ChartSettingsBase has resizable, legend, margin, extra", () => {
    expectTypeOf<ChartSettingsBase>().toHaveProperty("resizable");
    expectTypeOf<ChartSettingsBase>().toHaveProperty("legend");
    expectTypeOf<ChartSettingsBase>().toHaveProperty("margin");
    expectTypeOf<ChartSettingsBase>().toHaveProperty("extra");
  });

  it("ChartSettingsBase does NOT have xAxis, yAxis, grid, zoom", () => {
    expectTypeOf<ChartSettingsBase>().not.toHaveProperty("xAxis");
    expectTypeOf<ChartSettingsBase>().not.toHaveProperty("yAxis");
    expectTypeOf<ChartSettingsBase>().not.toHaveProperty("grid");
    expectTypeOf<ChartSettingsBase>().not.toHaveProperty("zoom");
  });

  it("ChartSettings extends ChartSettingsBase with xAxis, yAxis, grid, zoom", () => {
    expectTypeOf<ChartSettings>().toHaveProperty("xAxis");
    expectTypeOf<ChartSettings>().toHaveProperty("yAxis");
    expectTypeOf<ChartSettings>().toHaveProperty("grid");
    expectTypeOf<ChartSettings>().toHaveProperty("zoom");
    expectTypeOf<ChartSettings>().toHaveProperty("resizable");
  });

  it("DataComponentCommon has maxWidth and maxHeight", () => {
    expectTypeOf<DataComponentCommon>().toHaveProperty("maxWidth");
    expectTypeOf<DataComponentCommon>().toHaveProperty("maxHeight");
  });

  it("non-Cartesian charts extend ChartSettingsBase, not ChartSettings", () => {
    expectTypeOf<PieChartProps>().toHaveProperty("resizable");
    expectTypeOf<PieChartProps>().not.toHaveProperty("xAxis");
    expectTypeOf<MapProps>().not.toHaveProperty("xAxis");
    expectTypeOf<GraphProps>().not.toHaveProperty("xAxis");
    expectTypeOf<MeterProps>().not.toHaveProperty("xAxis");
    expectTypeOf<TreemapChartProps>().not.toHaveProperty("xAxis");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-component/src/model/displayer-types.test.ts`
Expected: FAIL — `ChartSettingsBase` not exported

- [ ] **Step 3: Implement the interface restructuring**

Replace the ChartSettings block and re-parent non-Cartesian charts in `packages/pages-component/src/model/displayer-types.ts`:

```typescript
// lines 6-18: Add maxWidth/maxHeight to DataComponentCommon
export interface DataComponentCommon {
  readonly title?: string;
  readonly visible?: boolean;
  readonly width?: string;
  readonly height?: string;
  readonly maxWidth?: number;
  readonly maxHeight?: number;
  readonly csvExport?: boolean;
  readonly lookup: DataSetLookup;
  readonly rowCount?: number;
  readonly rowOffset?: number;
  readonly columns?: readonly ColumnSettings[];
  readonly filter?: FilterSettings;
  readonly refresh?: RefreshSettings;
}

// lines 20-39: Split ChartSettings
export interface ChartSettingsBase {
  readonly resizable?: boolean;
  readonly legend?: {
    readonly show?: boolean;
    readonly position?: "top" | "bottom" | "left" | "right";
  };
  readonly margin?: {
    readonly top?: number;
    readonly right?: number;
    readonly bottom?: number;
    readonly left?: number;
  };
  readonly extra?: Readonly<Record<string, unknown>>;
}

export interface ChartSettings extends ChartSettingsBase {
  readonly zoom?: boolean;
  readonly xAxis?: { readonly title?: string; readonly showLabels?: boolean; readonly labelAngle?: number };
  readonly yAxis?: { readonly title?: string; readonly showLabels?: boolean; readonly labelAngle?: number };
  readonly grid?: { readonly x?: boolean; readonly y?: boolean };
}

// Re-parent non-Cartesian charts:
// PieChartProps line 53: ChartSettings → ChartSettingsBase
export interface PieChartProps extends DataComponentCommon, ChartSettingsBase {
  readonly subtype?: "pie" | "donut";
}

// MeterProps line 148: ChartSettings → ChartSettingsBase
export interface MeterProps extends DataComponentCommon, ChartSettingsBase {
  readonly end?: number;
  readonly warning?: number;
  readonly critical?: number;
}

// MapProps line 158: ChartSettings → ChartSettingsBase
export interface MapProps extends DataComponentCommon, ChartSettingsBase {
  readonly subtype?: "regions" | "markers";
  readonly colorScheme?: string;
  readonly mapName?: string;
}

// TreemapChartProps line 200: ChartSettings → ChartSettingsBase
export interface TreemapChartProps extends DataComponentCommon, ChartSettingsBase {
  readonly parentColumn?: ColumnId;
  readonly colorColumn?: ColumnId;
}

// GraphProps line 216: ChartSettings → ChartSettingsBase
export interface GraphProps extends DataComponentCommon, ChartSettingsBase {
  readonly layout?: "force" | "circular" | "none";
  readonly sourceColumn?: ColumnId;
  readonly targetColumn?: ColumnId;
  readonly valueColumn?: ColumnId;
  readonly directed?: boolean;
  readonly nodeLabelColumn?: ColumnId;
  readonly nodeColorColumn?: ColumnId;
  readonly nodeColorMap?: Record<string, string>;
  readonly nodeSizeColumn?: ColumnId;
}
```

Cartesian charts (BarChartProps, LineChartProps, AreaChartProps, ScatterChartProps, BubbleChartProps, TimeseriesProps, HeatmapChartProps, TimelineProps) keep `extends DataComponentCommon, ChartSettings` — no change.

Add `ChartSettingsBase` to `packages/pages-component/src/model/index.ts` exports at line 56:

```typescript
  ChartSettingsBase,
  ChartSettings,
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/pages-component/src/model/displayer-types.test.ts`
Expected: PASS

- [ ] **Step 5: Verify TypeScript compilation**

Run: `yarn tsc --noEmit -p packages/pages-component/tsconfig.json`
Expected: Compilation errors in pages-viz (PagesChartElement constraint too narrow). This is expected — Task 2 fixes it.

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/src/model/displayer-types.ts packages/pages-component/src/model/displayer-types.test.ts packages/pages-component/src/model/index.ts
git commit -m "feat(#413): extract ChartSettingsBase, re-parent non-Cartesian charts, move maxWidth/maxHeight to DataComponentCommon Refs #413"
```

---

### Task 2: Widen PagesChartElement and refactor applyChartSettings

**Files:**
- Modify: `packages/pages-viz/src/base/PagesChartElement.ts:9,15-16,54`
- Modify: `packages/pages-viz/src/charts/option-pipeline.ts:2,37-44,53-188`
- Modify: `packages/pages-viz/src/charts/PagesBarChart.ts:56-61`
- Modify: `packages/pages-viz/src/charts/PagesLineChart.ts:54-59`
- Modify: `packages/pages-viz/src/charts/PagesPieChart.ts:44-50`
- Modify: `packages/pages-viz/src/charts/PagesScatterChart.ts:50-56`
- Modify: `packages/pages-viz/src/charts/PagesBubbleChart.ts` (same pattern)
- Modify: `packages/pages-viz/src/charts/PagesTimeseries.ts:50-56`
- Modify: `packages/pages-viz/src/charts/PagesHeatmapChart.ts:83-88`
- Modify: `packages/pages-viz/src/charts/PagesGraph.ts:8,167-173`
- Modify: `packages/pages-viz/src/charts/PagesMap.ts:127-132`
- Modify: `packages/pages-viz/src/charts/PagesMeter.ts:136-140`
- Modify: `packages/pages-viz/src/charts/PagesTreemapChart.ts:71-76`
- Modify: `packages/pages-viz/src/charts/PagesTimeline.ts` (same pattern)

**Interfaces:**
- Consumes: `ChartSettingsBase` from Task 1
- Produces: `applyChartSettings(option, props)` with widened signature, centralized escape hatch merging

- [ ] **Step 1: Write failing test — applyChartSettings accepts ChartSettingsBase**

File: `packages/pages-viz/src/charts/option-pipeline.test.ts` (append)

```typescript
import { describe, it, expect } from "vitest";
import { applyChartSettings } from "./option-pipeline.js";
import type { ChartSettingsBase } from "@casehubio/pages-component";

describe("applyChartSettings with ChartSettingsBase", () => {
  it("accepts ChartSettingsBase props without xAxis/yAxis", () => {
    const props: { title?: string } & ChartSettingsBase = {
      title: "Test",
      legend: { show: true },
    };
    const result = applyChartSettings({}, props);
    expect(result.title).toEqual({ text: "Test" });
    expect(result.legend).toEqual({ show: true });
  });

  it("skips xAxis/yAxis when not present in props", () => {
    const props: { title?: string } & ChartSettingsBase = { legend: { show: true } };
    const result = applyChartSettings({ xAxis: { type: "category" } }, props);
    expect(result.xAxis).toEqual({ type: "category" });
  });

  it("centralises extra merge — no manual deepMerge needed", () => {
    const props: { title?: string } & ChartSettingsBase = {
      extra: { customKey: "value" },
    };
    const result = applyChartSettings({ series: [] }, props);
    expect(result.customKey).toBe("value");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/pages-viz/src/charts/option-pipeline.test.ts`
Expected: FAIL — `applyChartSettings` signature mismatch

- [ ] **Step 3: Refactor applyChartSettings**

In `packages/pages-viz/src/charts/option-pipeline.ts`:

Change import line 2:
```typescript
import type { ChartSettingsBase } from "@casehubio/pages-component";
```

Remove `ChartSettingsOptions` interface (lines 37-44).

Replace `applyChartSettings` function (lines 53-188):

```typescript
export function applyChartSettings(
  option: Record<string, unknown>,
  props: { title?: string } & ChartSettingsBase,
): Record<string, unknown> {
  // Title
  if (props.title !== undefined) {
    option.title = { text: props.title };
  }

  // Legend
  if (props.legend !== undefined) {
    const legend: Record<string, unknown> = { ...((option.legend as Record<string, unknown> | undefined) ?? {}) };
    if (props.legend.show !== undefined) legend.show = props.legend.show;
    if (props.legend.position !== undefined) {
      switch (props.legend.position) {
        case "top": legend.top = 0; break;
        case "bottom": legend.bottom = 0; break;
        case "left": legend.left = 0; legend.orient = "vertical"; break;
        case "right": legend.right = 0; legend.orient = "vertical"; break;
      }
    }
    option.legend = legend;
  }

  // Cartesian: xAxis (structural access — only fires for ChartSettings props)
  if ("xAxis" in props && props.xAxis !== undefined) {
    const raw = props as unknown as Record<string, unknown>;
    const xAxisProps = raw.xAxis as Record<string, unknown>;
    const xAxis: Record<string, unknown> = { ...((option.xAxis as Record<string, unknown> | undefined) ?? {}) };
    if (xAxisProps.title !== undefined) xAxis.name = xAxisProps.title;
    if (xAxisProps.showLabels !== undefined) xAxis.axisLabel = { show: xAxisProps.showLabels };
    if (xAxisProps.labelAngle != null) {
      const existing = (xAxis.axisLabel as Record<string, unknown> | undefined) ?? {};
      xAxis.axisLabel = { ...existing, rotate: xAxisProps.labelAngle };
    }
    option.xAxis = xAxis;
  }

  // Cartesian: yAxis
  if ("yAxis" in props && props.yAxis !== undefined) {
    const raw = props as unknown as Record<string, unknown>;
    const yAxisProps = raw.yAxis as Record<string, unknown>;
    const yAxis: Record<string, unknown> = { ...((option.yAxis as Record<string, unknown> | undefined) ?? {}) };
    if (yAxisProps.title !== undefined) yAxis.name = yAxisProps.title;
    if (yAxisProps.showLabels !== undefined) yAxis.axisLabel = { show: yAxisProps.showLabels };
    if (yAxisProps.labelAngle != null) {
      const existing = (yAxis.axisLabel as Record<string, unknown> | undefined) ?? {};
      yAxis.axisLabel = { ...existing, rotate: yAxisProps.labelAngle };
    }
    option.yAxis = yAxis;
  }

  // Margins (via grid)
  if (props.margin !== undefined) {
    const grid: Record<string, unknown> = { ...((option.grid as Record<string, unknown> | undefined) ?? {}) };
    if (props.margin.top !== undefined) grid.top = props.margin.top;
    if (props.margin.right !== undefined) grid.right = props.margin.right;
    if (props.margin.bottom !== undefined) grid.bottom = props.margin.bottom;
    if (props.margin.left !== undefined) grid.left = props.margin.left;
    option.grid = grid;
  }

  // Cartesian: grid line visibility
  if ("grid" in props) {
    const raw = props as unknown as Record<string, unknown>;
    const gridProps = raw.grid as { x?: boolean; y?: boolean } | undefined;
    if (gridProps !== undefined) {
      if (gridProps.x === false) {
        const xAxis: Record<string, unknown> = { ...((option.xAxis as Record<string, unknown> | undefined) ?? {}) };
        const existing = (xAxis.splitLine as Record<string, unknown> | undefined) ?? {};
        xAxis.splitLine = { ...existing, show: false };
        option.xAxis = xAxis;
      }
      if (gridProps.y === false) {
        const yAxis: Record<string, unknown> = { ...((option.yAxis as Record<string, unknown> | undefined) ?? {}) };
        const existing = (yAxis.splitLine as Record<string, unknown> | undefined) ?? {};
        yAxis.splitLine = { ...existing, show: false };
        option.yAxis = yAxis;
      }
    }
  }

  // Compact grid.top when no title and Cartesian
  if ("xAxis" in props && props.title === undefined && props.margin?.top === undefined) {
    const grid: Record<string, unknown> = { ...((option.grid as Record<string, unknown> | undefined) ?? {}) };
    if (grid.top === undefined) {
      grid.top = 10;
      option.grid = grid;
    }
  }

  // Cartesian: zoom
  if ("zoom" in props) {
    const raw = props as unknown as Record<string, unknown>;
    if (raw.zoom === true) {
      option.dataZoom = [{ type: "inside" }, { type: "slider" }];
    }
  }

  // Centralised escape hatch merging
  const raw = props as unknown as Record<string, unknown>;
  if (raw.echarts !== undefined && raw.echarts !== null && typeof raw.echarts === "object") {
    const { deepMerge } = await_deepMerge();
    option = deepMerge(option, raw.echarts as Record<string, unknown>);
  }
  if (props.extra !== undefined) {
    const { deepMerge } = await_deepMerge();
    option = deepMerge(option, props.extra);
  }

  return option;
}
```

Wait — `deepMerge` is imported from a separate module. Let me use a direct import at the top instead. Revise: add import at top of file:

```typescript
import { deepMerge } from "../base/deep-merge.js";
```

And replace the escape hatch section with:

```typescript
  // Centralised escape hatch merging
  const raw = props as unknown as Record<string, unknown>;
  if (raw.echarts !== undefined && raw.echarts !== null && typeof raw.echarts === "object") {
    option = deepMerge(option, raw.echarts as Record<string, unknown>);
  }
  if (props.extra !== undefined) {
    option = deepMerge(option, props.extra);
  }

  return option;
}
```

- [ ] **Step 4: Widen PagesChartElement generic constraint**

In `packages/pages-viz/src/base/PagesChartElement.ts`:

Line 9: change import:
```typescript
import type { ChartSettingsBase } from "@casehubio/pages-component";
```

Lines 15-16: widen generic:
```typescript
export abstract class PagesChartElement<
  P extends VizComponentProps & ChartSettingsBase,
> extends PagesElement<P> {
```

Line 54: make backgroundColor conditional:
```typescript
      option['backgroundColor'] ??= 'transparent';
```

- [ ] **Step 5: Update all chart renderers — remove manual deepMerge calls and cartesianAxes flag**

In every renderer, remove the Stage 4 block. The pattern to remove in each file:

```typescript
    // Stage 4: Deep merge extra
    if (props.extra) {
      option = deepMerge(option, props.extra);
    }
```

And remove the `deepMerge` import. Also remove `{ cartesianAxes: false }` parameter from non-Cartesian charts.

Files to update:
- `PagesBarChart.ts:14,58-61` — remove deepMerge import, remove lines 58-61
- `PagesLineChart.ts:14,57-59` — same
- `PagesPieChart.ts:12,44-50` — remove deepMerge import, change line 45 from `applyChartSettings(option, props, { cartesianAxes: false })` to `applyChartSettings(option, props)`, remove lines 48-50
- `PagesScatterChart.ts:13,53-56` — remove deepMerge import, remove lines 53-56
- `PagesBubbleChart.ts` — same pattern
- `PagesTimeseries.ts:15,53-56` — same
- `PagesHeatmapChart.ts:12,85-88` — same
- `PagesGraph.ts:9,169-173` — remove deepMerge import, remove lines 169-173. Note: `applyChartSettings(option, props)` on line 167 stays as-is (already no `cartesianAxes` flag — the structural access fix is automatic)
- `PagesMap.ts:14,127,129-132` — remove deepMerge import, change line 127 to `applyChartSettings(option, props)`, remove lines 129-132
- `PagesMeter.ts:14,136,138-140` — same
- `PagesTreemapChart.ts:11,71,73-76` — same
- `PagesTimeline.ts` — same pattern

- [ ] **Step 6: Run tests**

Run: `yarn vitest run packages/pages-viz/ packages/pages-component/`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-viz/ packages/pages-component/
git commit -m "feat(#413): widen PagesChartElement to ChartSettingsBase, refactor applyChartSettings, centralise escape hatch merging Refs #413"
```

---

## Batch 2: SPI broadening — shared and Cartesian properties

### Task 3: Add ChartSettingsBase shared properties and legend extensions

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts` (ChartSettingsBase)
- Modify: `packages/pages-viz/src/charts/option-pipeline.ts` (wire tooltip, animation, color, backgroundColor, legend orient/selectedMode)
- Test: `packages/pages-viz/src/charts/option-pipeline.test.ts`
- Test: `packages/pages-ui/src/parser/displayer-desugar.test.ts`

**Interfaces:**
- Consumes: `ChartSettingsBase` from Task 1
- Produces: Extended `ChartSettingsBase` with tooltip, animation, color, backgroundColor, legend orient/selectedMode

- [ ] **Step 1: Write failing tests**

Append to `packages/pages-viz/src/charts/option-pipeline.test.ts`:

```typescript
describe("ChartSettingsBase shared properties", () => {
  it("applies tooltip settings", () => {
    const props = { tooltip: { show: true, trigger: "axis" as const } };
    const result = applyChartSettings({}, props);
    expect(result.tooltip).toEqual({ show: true, trigger: "axis" });
  });

  it("applies animation=false", () => {
    const result = applyChartSettings({}, { animation: false });
    expect(result.animation).toBe(false);
  });

  it("applies color palette", () => {
    const result = applyChartSettings({}, { color: ["#f00", "#0f0"] });
    expect(result.color).toEqual(["#f00", "#0f0"]);
  });

  it("applies backgroundColor", () => {
    const result = applyChartSettings({}, { backgroundColor: "#fff" });
    expect(result.backgroundColor).toBe("#fff");
  });

  it("applies legend orient", () => {
    const result = applyChartSettings({}, { legend: { orient: "vertical" } });
    expect((result.legend as Record<string, unknown>).orient).toBe("vertical");
  });

  it("applies legend selectedMode", () => {
    const result = applyChartSettings({}, { legend: { selectedMode: "single" } });
    expect((result.legend as Record<string, unknown>).selectedMode).toBe("single");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-viz/src/charts/option-pipeline.test.ts`
Expected: FAIL — properties not in interface

- [ ] **Step 3: Add properties to ChartSettingsBase**

In `packages/pages-component/src/model/displayer-types.ts`, expand `ChartSettingsBase`:

```typescript
export interface ChartSettingsBase {
  readonly resizable?: boolean;
  readonly tooltip?: {
    readonly show?: boolean;
    readonly trigger?: "item" | "axis" | "none";
  };
  readonly animation?: boolean;
  readonly color?: readonly string[];
  readonly backgroundColor?: string;
  readonly legend?: {
    readonly show?: boolean;
    readonly position?: "top" | "bottom" | "left" | "right";
    readonly orient?: "horizontal" | "vertical";
    readonly selectedMode?: boolean | "single" | "multiple";
  };
  readonly margin?: {
    readonly top?: number;
    readonly right?: number;
    readonly bottom?: number;
    readonly left?: number;
  };
  readonly extra?: Readonly<Record<string, unknown>>;
}
```

- [ ] **Step 4: Wire through applyChartSettings**

Add to `applyChartSettings` in `option-pipeline.ts`, before the Cartesian section:

```typescript
  // Tooltip
  if (props.tooltip !== undefined) {
    const tooltip: Record<string, unknown> = { ...((option.tooltip as Record<string, unknown> | undefined) ?? {}) };
    if (props.tooltip.show !== undefined) tooltip.show = props.tooltip.show;
    if (props.tooltip.trigger !== undefined) tooltip.trigger = props.tooltip.trigger;
    option.tooltip = tooltip;
  }

  // Animation
  if (props.animation !== undefined) {
    option.animation = props.animation;
  }

  // Color palette
  if (props.color !== undefined) {
    option.color = [...props.color];
  }

  // Background color
  if (props.backgroundColor !== undefined) {
    option.backgroundColor = props.backgroundColor;
  }
```

In the legend block, add orient and selectedMode handling:

```typescript
    if (props.legend.orient !== undefined) {
      legend.orient = props.legend.orient;
    }
    if (props.legend.selectedMode !== undefined) {
      legend.selectedMode = props.legend.selectedMode;
    }
```

- [ ] **Step 5: Regenerate schemas and run tests**

Run: `yarn workspace @casehubio/pages-schema run generate`
Run: `yarn vitest run packages/pages-viz/src/charts/option-pipeline.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/ packages/pages-viz/ packages/pages-schema/
git commit -m "feat(#413): add tooltip, animation, color, backgroundColor, legend orient/selectedMode to ChartSettingsBase Refs #413"
```

---

### Task 4: Add ChartSettings Cartesian promotions

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts` (ChartSettings — axis extensions, zoom evolution)
- Modify: `packages/pages-viz/src/charts/option-pipeline.ts`
- Test: `packages/pages-viz/src/charts/option-pipeline.test.ts`

**Interfaces:**
- Consumes: `ChartSettings extends ChartSettingsBase` from Task 1
- Produces: Extended axis types (type/min/max/inverse), zoom union (boolean | object)

- [ ] **Step 1: Write failing tests**

```typescript
describe("ChartSettings Cartesian promotions", () => {
  it("applies axis type", () => {
    const props = { xAxis: { type: "log" as const } };
    const result = applyChartSettings({}, props);
    expect((result.xAxis as Record<string, unknown>).type).toBe("log");
  });

  it("applies axis min/max", () => {
    const props = { yAxis: { min: 0, max: "dataMax" as const } };
    const result = applyChartSettings({}, props);
    const yAxis = result.yAxis as Record<string, unknown>;
    expect(yAxis.min).toBe(0);
    expect(yAxis.max).toBe("dataMax");
  });

  it("applies axis inverse", () => {
    const props = { xAxis: { inverse: true } };
    const result = applyChartSettings({}, props);
    expect((result.xAxis as Record<string, unknown>).inverse).toBe(true);
  });

  it("zoom object with start/end produces dataZoom", () => {
    const props = { zoom: { enabled: true, start: 20, end: 80 } };
    const result = applyChartSettings({}, props);
    const dz = result.dataZoom as Record<string, unknown>[];
    expect(dz).toHaveLength(2);
    expect(dz[0]).toMatchObject({ type: "inside", start: 20, end: 80 });
    expect(dz[1]).toMatchObject({ type: "slider", start: 20, end: 80 });
  });

  it("zoom: true still works (backward compat)", () => {
    const props = { zoom: true };
    const result = applyChartSettings({}, props);
    expect(result.dataZoom).toHaveLength(2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Expand axis and zoom types in ChartSettings**

In `displayer-types.ts`, update `ChartSettings`:

```typescript
export interface ChartSettings extends ChartSettingsBase {
  readonly zoom?: boolean | {
    readonly enabled?: boolean;
    readonly start?: number;
    readonly end?: number;
  };
  readonly xAxis?: {
    readonly title?: string;
    readonly showLabels?: boolean;
    readonly labelAngle?: number;
    readonly type?: "value" | "category" | "time" | "log";
    readonly min?: number | "dataMin";
    readonly max?: number | "dataMax";
    readonly inverse?: boolean;
  };
  readonly yAxis?: {
    readonly title?: string;
    readonly showLabels?: boolean;
    readonly labelAngle?: number;
    readonly type?: "value" | "category" | "time" | "log";
    readonly min?: number | "dataMin";
    readonly max?: number | "dataMax";
    readonly inverse?: boolean;
  };
  readonly grid?: { readonly x?: boolean; readonly y?: boolean };
}
```

- [ ] **Step 4: Wire axis extensions and zoom evolution in applyChartSettings**

In the xAxis/yAxis structural access blocks, add:

```typescript
    if (xAxisProps.type !== undefined) xAxis.type = xAxisProps.type;
    if (xAxisProps.min !== undefined) xAxis.min = xAxisProps.min;
    if (xAxisProps.max !== undefined) xAxis.max = xAxisProps.max;
    if (xAxisProps.inverse !== undefined) xAxis.inverse = xAxisProps.inverse;
```

(Same for yAxis block.)

Replace the zoom handling:

```typescript
  // Cartesian: zoom
  if ("zoom" in props) {
    const raw = props as unknown as Record<string, unknown>;
    const zoomVal = raw.zoom;
    if (zoomVal === true) {
      option.dataZoom = [{ type: "inside" }, { type: "slider" }];
    } else if (typeof zoomVal === "object" && zoomVal !== null) {
      const z = zoomVal as { enabled?: boolean; start?: number; end?: number };
      if (z.enabled !== false) {
        const base: Record<string, unknown> = {};
        if (z.start !== undefined) base.start = z.start;
        if (z.end !== undefined) base.end = z.end;
        option.dataZoom = [{ type: "inside", ...base }, { type: "slider", ...base }];
      }
    }
  }
```

- [ ] **Step 5: Regenerate schemas and run tests**

Run: `yarn workspace @casehubio/pages-schema run generate`
Run: `yarn vitest run packages/pages-viz/src/charts/option-pipeline.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/ packages/pages-viz/ packages/pages-schema/
git commit -m "feat(#413): add axis type/min/max/inverse, zoom evolution to ChartSettings Refs #413"
```

---

### Task 5: Per-chart series promotions

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts` (per-chart props)
- Modify: `packages/pages-viz/src/charts/PagesBarChart.ts`
- Modify: `packages/pages-viz/src/charts/PagesLineChart.ts`
- Modify: `packages/pages-viz/src/charts/PagesPieChart.ts`
- Modify: `packages/pages-viz/src/charts/PagesScatterChart.ts`
- Modify: `packages/pages-viz/src/charts/PagesHeatmapChart.ts`
- Modify: `packages/pages-viz/src/charts/PagesTreemapChart.ts`
- Modify: `packages/pages-viz/src/charts/PagesMeter.ts`
- Modify: `packages/pages-viz/src/charts/PagesTimeseries.ts`

**Interfaces:**
- Consumes: Existing per-chart Props interfaces
- Produces: Extended per-chart interfaces with series-level properties

- [ ] **Step 1: Add all per-chart properties to interfaces**

In `displayer-types.ts`, expand each interface:

```typescript
export interface BarChartProps extends DataComponentCommon, ChartSettings {
  readonly subtype?: "column" | "column-stacked" | "bar" | "bar-stacked";
  readonly barWidth?: string | number;
  readonly barGap?: string;
}

export interface LineChartProps extends DataComponentCommon, ChartSettings {
  readonly subtype?: "line" | "smooth";
  readonly step?: false | "start" | "end" | "middle";
  readonly connectNulls?: boolean;
  readonly showSymbol?: boolean;
}

// PieChartProps already re-parented in Task 1, add:
export interface PieChartProps extends DataComponentCommon, ChartSettingsBase {
  readonly subtype?: "pie" | "donut";
  readonly roseType?: "radius" | "area";
  readonly startAngle?: number;
  readonly clockwise?: boolean;
}

export interface ScatterChartProps extends DataComponentCommon, ChartSettings {
  readonly symbolSize?: number;
}

export interface TimeseriesProps extends DataComponentCommon, ChartSettings {
  readonly connectNulls?: boolean;
}

export interface HeatmapChartProps extends DataComponentCommon, ChartSettings {
  readonly minColor?: string;
  readonly maxColor?: string;
  readonly blurSize?: number;
  readonly minOpacity?: number;
  readonly maxOpacity?: number;
}

export interface TreemapChartProps extends DataComponentCommon, ChartSettingsBase {
  readonly parentColumn?: ColumnId;
  readonly colorColumn?: ColumnId;
  readonly sort?: boolean | "asc" | "desc";
  readonly leafDepth?: number;
  readonly nodeClick?: "zoomToNode" | "link" | false;
}

export interface MeterProps extends DataComponentCommon, ChartSettingsBase {
  readonly end?: number;
  readonly warning?: number;
  readonly critical?: number;
  readonly startAngle?: number;
  readonly endAngle?: number;
  readonly clockwise?: boolean;
}
```

- [ ] **Step 2: Wire bar properties**

In `PagesBarChart.ts` `buildOption`, after building each series entry, add:

```typescript
      if (props.barWidth !== undefined) seriesEntry.barWidth = props.barWidth;
      if (props.barGap !== undefined) seriesEntry.barGap = props.barGap;
```

- [ ] **Step 3: Wire line properties**

In `PagesLineChart.ts` `buildOption`, after building each series entry:

```typescript
      if (props.step !== undefined) seriesEntry.step = props.step;
      if (props.connectNulls !== undefined) seriesEntry.connectNulls = props.connectNulls;
      if (props.showSymbol !== undefined) seriesEntry.showSymbol = props.showSymbol;
```

- [ ] **Step 4: Wire pie properties**

In `PagesPieChart.ts` `buildOption`, after building the series object:

```typescript
    if (props.roseType !== undefined) series.roseType = props.roseType;
    if (props.startAngle !== undefined) series.startAngle = props.startAngle;
    if (props.clockwise !== undefined) series.clockwise = props.clockwise;
```

- [ ] **Step 5: Wire scatter symbolSize**

In `PagesScatterChart.ts` `buildOption`, on the series object:

```typescript
    if (props.symbolSize !== undefined) {
      series.symbolSize = props.symbolSize;
    }
```

(Only when dataset has < 3 columns, since ≥3 columns uses the dynamic callback.)

- [ ] **Step 6: Wire heatmap-chart blurSize/opacity**

In `PagesHeatmapChart.ts` `buildOption`, on the series:

```typescript
    if (props.blurSize !== undefined) {
      (option.series as Record<string, unknown>[])[0].blurSize = props.blurSize;
    }
    if (props.minOpacity !== undefined) {
      (option.series as Record<string, unknown>[])[0].minOpacity = props.minOpacity;
    }
    if (props.maxOpacity !== undefined) {
      (option.series as Record<string, unknown>[])[0].maxOpacity = props.maxOpacity;
    }
```

- [ ] **Step 7: Wire treemap sort/leafDepth/nodeClick**

In `PagesTreemapChart.ts` `buildOption`:

```typescript
    if (props.sort !== undefined) series.sort = props.sort;
    if (props.leafDepth !== undefined) series.leafDepth = props.leafDepth;
    if (props.nodeClick !== undefined) series.nodeClick = props.nodeClick;
```

- [ ] **Step 8: Wire meter startAngle/endAngle/clockwise**

In `PagesMeter.ts` `buildOption`, replace hardcoded `startAngle: 180, endAngle: 0` in the gauge series:

```typescript
          startAngle: props.startAngle ?? 180,
          endAngle: props.endAngle ?? 0,
          clockwise: props.clockwise ?? false,
```

- [ ] **Step 9: Wire timeseries connectNulls**

In `PagesTimeseries.ts` `buildOption`, on each series entry:

```typescript
      if (props.connectNulls !== undefined) seriesEntry.connectNulls = props.connectNulls;
```

- [ ] **Step 10: Regenerate schemas and run tests**

Run: `yarn workspace @casehubio/pages-schema run generate`
Run: `yarn vitest run packages/pages-viz/ packages/pages-component/`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add packages/pages-component/ packages/pages-viz/ packages/pages-schema/
git commit -m "feat(#413): add per-chart series properties — bar, line, pie, scatter, heatmap, treemap, meter, timeseries Refs #413"
```

---

## Batch 3: Component-specific broadening

### Task 6: MapProps, GraphProps, and DensityHeatmapProps promotions + radius bug fix

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts` (MapProps, GraphProps, DensityHeatmapProps)
- Modify: `packages/pages-viz/src/charts/PagesMap.ts`
- Modify: `packages/pages-viz/src/charts/PagesGraph.ts`
- Modify: `packages/pages-viz/src/charts/PagesDensityHeatmap.ts:100-107`

**Interfaces:**
- Consumes: ChartSettingsBase from Task 1
- Produces: Extended MapProps (roam, center, zoom, scaleLimit, showLabel, selectedMode), GraphProps (repulsion, edgeLabel, roam, symbol), DensityHeatmapProps (blur, opacity, intensity, extra)

- [ ] **Step 1: Expand interfaces**

MapProps in `displayer-types.ts`:
```typescript
export interface MapProps extends DataComponentCommon, ChartSettingsBase {
  readonly subtype?: "regions" | "markers";
  readonly colorScheme?: string;
  readonly mapName?: string;
  readonly roam?: boolean | "pan" | "zoom";
  readonly center?: readonly [number, number];
  readonly zoom?: number;
  readonly scaleLimit?: { readonly min?: number; readonly max?: number };
  readonly showLabel?: boolean;
  readonly selectedMode?: "single" | "multiple" | boolean;
}
```

GraphProps:
```typescript
export interface GraphProps extends DataComponentCommon, ChartSettingsBase {
  readonly layout?: "force" | "circular" | "none";
  readonly sourceColumn?: ColumnId;
  readonly targetColumn?: ColumnId;
  readonly valueColumn?: ColumnId;
  readonly directed?: boolean;
  readonly nodeLabelColumn?: ColumnId;
  readonly nodeColorColumn?: ColumnId;
  readonly nodeColorMap?: Record<string, string>;
  readonly nodeSizeColumn?: ColumnId;
  readonly repulsion?: number;
  readonly edgeLabel?: boolean;
  readonly roam?: boolean | "pan" | "zoom";
  readonly symbol?: string;
}
```

DensityHeatmapProps:
```typescript
export interface DensityHeatmapProps extends DataComponentCommon {
  readonly xColumn?: ColumnId;
  readonly yColumn?: ColumnId;
  readonly valueColumn?: ColumnId;
  readonly gradient?: readonly { readonly offset: number; readonly color: string }[];
  readonly radius?: number;
  readonly aggregation?: "max" | "sum" | "mean" | "count";
  readonly showTooltip?: boolean;
  readonly showLegend?: boolean;
  readonly blur?: number;
  readonly maxOpacity?: number;
  readonly minOpacity?: number;
  readonly intensityExponent?: number;
  readonly valueMin?: number;
  readonly valueMax?: number;
  readonly extra?: Readonly<Record<string, unknown>>;
}
```

- [ ] **Step 2: Wire MapProps through PagesMap**

In `PagesMap.ts`, in the markers geo block and the regions series block, add the new properties:

For markers (geo object):
```typescript
        geo: {
          map: mapName,
          roam: props.roam ?? true,
          ...(props.center ? { center: props.center } : {}),
          ...(props.zoom !== undefined ? { zoom: props.zoom } : {}),
          ...(props.scaleLimit ? { scaleLimit: props.scaleLimit } : {}),
          ...(props.showLabel !== undefined ? { label: { show: props.showLabel } } : {}),
          ...(props.selectedMode !== undefined ? { selectedMode: props.selectedMode } : {}),
        },
```

For regions (series map), add to the series entry:
```typescript
          ...(props.showLabel !== undefined ? { label: { show: props.showLabel } } : {}),
          ...(props.selectedMode !== undefined ? { selectedMode: props.selectedMode } : {}),
```

And add geo properties to the regions option:
```typescript
        if (props.roam !== undefined) option.geo = { ...((option.geo ?? {}) as Record<string, unknown>), roam: props.roam };
```

- [ ] **Step 3: Wire GraphProps through PagesGraph**

In `PagesGraph.ts`, in `buildOption`:

Replace hardcoded `series.force = { repulsion: 100 }`:
```typescript
    if (layout === "force") {
      series.force = { repulsion: props.repulsion ?? 100 };
    }
```

Add after the series construction:
```typescript
    if (props.edgeLabel) {
      series.edgeLabel = { show: true };
    }
    if (props.roam !== undefined) {
      series.roam = props.roam;
    }
    if (props.symbol !== undefined) {
      series.symbol = props.symbol;
    }
```

- [ ] **Step 4: Wire DensityHeatmapProps + fix radius bug**

In `PagesDensityHeatmap.ts`, replace `createInstance` method (lines 94-118):

```typescript
  private createInstance(
    mod: HeatmapModule,
    container: HTMLDivElement,
    data: HeatmapPoint[],
    props: DensityHeatmapProps,
  ): HeatmapInstance {
    const config: Record<string, unknown> = { container, data };

    if (props.gradient) config.gradient = props.gradient;
    if (props.aggregation) config.aggregationMode = props.aggregation;
    if (props.radius !== undefined) config.radius = props.radius;
    if (props.blur !== undefined) config.blur = props.blur;
    if (props.maxOpacity !== undefined) config.maxOpacity = props.maxOpacity;
    if (props.minOpacity !== undefined) config.minOpacity = props.minOpacity;
    if (props.intensityExponent !== undefined) config.intensityExponent = props.intensityExponent;
    if (props.valueMin !== undefined) config.valueMin = props.valueMin;
    if (props.valueMax !== undefined) config.valueMax = props.valueMax;

    // Escape hatch merging
    if (props.extra !== undefined) {
      Object.assign(config, props.extra);
    }

    const features: unknown[] = [];
    if (props.showTooltip) features.push(mod.withTooltip());
    if (props.showLegend) features.push(mod.withLegend());

    return mod.createHeatmap(config as never, ...features as never[]);
  }
```

- [ ] **Step 5: Regenerate schemas and run tests**

Run: `yarn workspace @casehubio/pages-schema run generate`
Run: `yarn vitest run packages/pages-viz/ packages/pages-component/`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/ packages/pages-viz/ packages/pages-schema/
git commit -m "feat(#413): broaden MapProps, GraphProps, DensityHeatmapProps, fix radius wiring bug Refs #413"
```

---

### Task 7: Create typed escape hatch extension types

**Files:**
- Create: `packages/pages-component/src/model/echarts-extension.ts`
- Create: `packages/pages-component/src/model/heatmap-extension.ts`
- Create: `packages/pages-component/src/model/reactflow-extension.ts`
- Create: `packages/pages-component/src/model/elk-extension.ts`
- Modify: `packages/pages-component/src/model/displayer-types.ts` (add echarts? to ChartSettingsBase, heatmapJs? to DensityHeatmapProps)
- Modify: `packages/pages-component/src/model/index.ts` (export extension types)

**Interfaces:**
- Produces: CasehubEChartsExtension, CasehubHeatmapExtension, CasehubReactFlowExtension, CasehubElkExtension

- [ ] **Step 1: Create extension type files**

`packages/pages-component/src/model/echarts-extension.ts`:
```typescript
export interface CasehubEChartsExtension {
  readonly toolbox?: {
    readonly show?: boolean;
    readonly feature?: {
      readonly saveAsImage?: { readonly show?: boolean; readonly title?: string };
      readonly dataZoom?: { readonly show?: boolean };
      readonly restore?: { readonly show?: boolean };
      readonly dataView?: { readonly show?: boolean; readonly readOnly?: boolean };
    };
    readonly orient?: "horizontal" | "vertical";
  };
  readonly dataZoom?: readonly {
    readonly type?: "slider" | "inside";
    readonly filterMode?: "filter" | "weakFilter" | "empty" | "none";
    readonly xAxisIndex?: number | readonly number[];
    readonly yAxisIndex?: number | readonly number[];
  }[];
  readonly animationDuration?: number;
  readonly animationEasing?: string;
  readonly animationDurationUpdate?: number;
  readonly animationThreshold?: number;
  readonly darkMode?: boolean;
  readonly series?: Readonly<Record<string, unknown>>;
}
```

`packages/pages-component/src/model/heatmap-extension.ts`:
```typescript
export interface CasehubHeatmapExtension {
  readonly blendMode?: string;
}
```

`packages/pages-component/src/model/reactflow-extension.ts`:
```typescript
export interface CasehubReactFlowExtension {
  readonly panOnDrag?: boolean;
  readonly zoomOnScroll?: boolean;
  readonly zoomOnPinch?: boolean;
  readonly snapToGrid?: boolean;
  readonly snapGrid?: readonly [number, number];
  readonly colorMode?: "light" | "dark" | "system";
  readonly defaultMarkerColor?: string;
  readonly selectionMode?: "full" | "partial";
  readonly elementsSelectable?: boolean;
}
```

`packages/pages-component/src/model/elk-extension.ts`:
```typescript
export interface CasehubElkExtension {
  readonly wrapping?: boolean;
  readonly headerHeight?: number;
  readonly elkOptions?: Readonly<Record<string, string>>;
}
```

- [ ] **Step 2: Add escape hatch properties to interfaces**

In `displayer-types.ts`, add import and property:

```typescript
import type { CasehubEChartsExtension } from "./echarts-extension.js";
import type { CasehubHeatmapExtension } from "./heatmap-extension.js";
```

Add to `ChartSettingsBase`:
```typescript
  readonly echarts?: CasehubEChartsExtension;
```

Add to `DensityHeatmapProps`:
```typescript
  readonly heatmapJs?: CasehubHeatmapExtension;
```

- [ ] **Step 3: Update DensityHeatmap merge pipeline**

In `PagesDensityHeatmap.ts`, in `createInstance`, add before `extra` merging:

```typescript
    // Typed escape hatch
    if (props.heatmapJs !== undefined) {
      Object.assign(config, props.heatmapJs);
    }
```

- [ ] **Step 4: Export new types from index.ts**

Add to `packages/pages-component/src/model/index.ts`:

```typescript
export type { CasehubEChartsExtension } from "./echarts-extension.js";
export type { CasehubHeatmapExtension } from "./heatmap-extension.js";
export type { CasehubReactFlowExtension } from "./reactflow-extension.js";
export type { CasehubElkExtension } from "./elk-extension.js";
```

- [ ] **Step 5: Regenerate schemas and run tests**

Run: `yarn workspace @casehubio/pages-schema run generate`
Run: `yarn vitest run packages/pages-viz/ packages/pages-component/`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-component/ packages/pages-viz/ packages/pages-schema/
git commit -m "feat(#413): add typed escape hatch extension types — echarts, heatmapJs, reactFlow, elk Refs #413"
```

---

## Batch 4: Schema regeneration and end-to-end validation

### Task 8: Regenerate schemas and validate full pipeline

**Files:**
- Modify: `packages/pages-schema/src/component-schemas.generated.ts` (regenerated)
- Test: `packages/pages-schema/src/generator.test.ts`
- Test: `packages/pages-ui/src/parser/displayer-desugar.test.ts`

- [ ] **Step 1: Regenerate schemas**

Run: `yarn workspace @casehubio/pages-schema run generate`

- [ ] **Step 2: Run staleness test**

Run: `yarn vitest run packages/pages-schema/src/generator.test.ts`
Expected: PASS

- [ ] **Step 3: Write desugarer round-trip tests for key new properties**

Append to `packages/pages-ui/src/parser/displayer-desugar.test.ts`:

```typescript
describe("SPI broadening round-trip", () => {
  it("barWidth passes through desugarer", () => {
    const raw = { type: "BARCHART", properties: { barWidth: 20 }, lookup: { uuid: "ds", columnCount: 2 } };
    const result = desugarDisplayer(raw);
    expect(result?.props?.barWidth).toBe(20);
  });

  it("tooltip passes through desugarer", () => {
    const raw = { type: "LINECHART", properties: { tooltip: { show: false } }, lookup: { uuid: "ds", columnCount: 2 } };
    const result = desugarDisplayer(raw);
    expect(result?.props?.tooltip).toEqual({ show: false });
  });

  it("echarts escape hatch passes through", () => {
    const raw = { type: "BARCHART", properties: { echarts: { darkMode: true } }, lookup: { uuid: "ds", columnCount: 2 } };
    const result = desugarDisplayer(raw);
    expect(result?.props?.echarts).toEqual({ darkMode: true });
  });

  it("MapProps zoom passes through (not conflicting with ChartSettings zoom)", () => {
    const raw = { type: "MAP", properties: { zoom: 5 }, lookup: { uuid: "ds", columnCount: 2 } };
    const result = desugarDisplayer(raw);
    expect(result?.props?.zoom).toBe(5);
  });

  it("DensityHeatmap blur passes through", () => {
    const raw = { type: "DENSITY-HEATMAP", properties: { blur: 0.5 }, lookup: { uuid: "ds", columnCount: 3 } };
    const result = desugarDisplayer(raw);
    expect(result?.props?.blur).toBe(0.5);
  });
});
```

- [ ] **Step 4: Run full test suite**

Run: `yarn vitest run packages/pages-ui/ packages/pages-viz/ packages/pages-component/ packages/pages-schema/`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-schema/ packages/pages-ui/
git commit -m "feat(#413): regenerate schemas, add desugarer round-trip tests for SPI broadening Refs #413"
```

---

## Batch 5: GraphCanvas YAML integration

### Task 9: GraphCanvasProps interface and registration

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts` (add GraphCanvasProps)
- Modify: `packages/pages-component/src/model/type-guards.ts:113` (add to ComponentTypeRegistry)
- Modify: `packages/pages-component/src/model/index.ts` (export GraphCanvasProps, isGraphCanvas)
- Modify: `packages/pages-ui/src/parser/displayer-desugar.ts:10-43` (add TYPE_MAP entries)
- Modify: `packages/pages-schema/src/schema-registry.ts:68` (add registry entry)
- Modify: `packages/pages-schema/src/index.ts` (export graphCanvasPropsSchema)
- Modify: `packages/graph-renderer/src/bridge/GraphCanvas.ts:22` (rename tag)
- Modify: `packages/graph-renderer/src/bridge/bridge.test.ts:21,31` (update references)

**Interfaces:**
- Consumes: CasehubReactFlowExtension, CasehubElkExtension from Task 7
- Produces: GraphCanvasProps interface, `"graph-canvas"` component type registration

- [ ] **Step 1: Add GraphCanvasProps interface**

In `displayer-types.ts`, add imports and interface:

```typescript
import type { CasehubReactFlowExtension } from "./reactflow-extension.js";
import type { CasehubElkExtension } from "./elk-extension.js";

export interface GraphCanvasProps extends DataComponentCommon {
  readonly sourceColumn?: ColumnId;
  readonly targetColumn?: ColumnId;
  readonly nodeLabelColumn?: ColumnId;
  readonly nodeColorColumn?: ColumnId;
  readonly nodeColorMap?: Record<string, string>;
  readonly nodeSizeColumn?: ColumnId;
  readonly valueColumn?: ColumnId;
  readonly directed?: boolean;
  readonly direction?: "DOWN" | "RIGHT" | "LEFT" | "UP";
  readonly spacing?: number;
  readonly algorithm?: "layered" | "tree" | "radial" | "force" | "stress";
  readonly containerPadding?: number;
  readonly connectionsEnabled?: boolean;
  readonly fitView?: boolean;
  readonly nodesDraggable?: boolean;
  readonly minZoom?: number;
  readonly maxZoom?: number;
  readonly edgeType?: "default" | "straight" | "step" | "smoothstep";
  readonly edgeAnimated?: boolean;
  readonly extra?: Readonly<Record<string, unknown>>;
  readonly reactFlow?: CasehubReactFlowExtension;
  readonly elk?: CasehubElkExtension;
}
```

- [ ] **Step 2: Add to ComponentTypeRegistry**

In `type-guards.ts`, after line 113 (`graph`):

```typescript
  "graph-canvas": GraphCanvasProps;
```

Add `isGraphCanvas` function:

```typescript
export function isGraphCanvas(c: Component): c is TypedComponent<"graph-canvas"> {
  return isComponentType(c, "graph-canvas");
}
```

- [ ] **Step 3: Add TYPE_MAP entries**

In `displayer-desugar.ts`, add to TYPE_MAP:

```typescript
  GRAPH_CANVAS: "graph-canvas",
  "GRAPH-CANVAS": "graph-canvas",
```

- [ ] **Step 4: Rename existing GraphCanvas tag**

In `packages/graph-renderer/src/bridge/GraphCanvas.ts`, line 22:

```typescript
@customElement('graph-canvas-core')
```

In `packages/graph-renderer/src/bridge/bridge.test.ts`, update:
- Line 21: `document.createElement('graph-canvas-core')`
- Line 31: `customElements.get('graph-canvas-core')`

- [ ] **Step 5: Export and register schema**

Add to `packages/pages-component/src/model/index.ts`:
```typescript
  GraphCanvasProps,
  // ... in type-guards exports:
  isGraphCanvas,
```

Regenerate schemas: `yarn workspace @casehubio/pages-schema run generate`

Add to `packages/pages-schema/src/schema-registry.ts` imports and map:
```typescript
  graphCanvasPropsSchema,
```

```typescript
  ["graph-canvas", graphCanvasPropsSchema],
```

Add to `packages/pages-schema/src/index.ts` exports:
```typescript
  graphCanvasPropsSchema,
```

- [ ] **Step 6: Run tests**

Run: `yarn vitest run packages/pages-component/ packages/pages-ui/ packages/pages-schema/ packages/graph-renderer/`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-component/ packages/pages-ui/ packages/pages-schema/ packages/graph-renderer/
git commit -m "feat(#413): add GraphCanvasProps interface and register as YAML component type, rename GraphCanvas tag to graph-canvas-core Refs #413"
```

---

### Task 10: PagesGraphCanvas YAML bridge

**Files:**
- Create: `packages/graph-renderer/src/bridge/PagesGraphCanvas.ts`
- Create: `packages/graph-renderer/src/bridge/pages-graph-canvas.test.ts`
- Modify: `packages/graph-renderer/src/index.ts` (export PagesGraphCanvas)
- Modify: `packages/graph-renderer/package.json` (add @casehubio/pages-component dependency)

**Interfaces:**
- Consumes: GraphCanvasProps, DataSourceController from pages-component; GraphModel from graph-core; ElkLayoutOptions from elk-layout
- Produces: `<pages-graph-canvas>` custom element that bridges YAML data to GraphCanvas

- [ ] **Step 1: Add dependency**

In `packages/graph-renderer/package.json`, add to dependencies:
```json
"@casehubio/pages-component": "workspace:*"
```

Run: `yarn install`

- [ ] **Step 2: Write failing test**

File: `packages/graph-renderer/src/bridge/pages-graph-canvas.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

vi.mock('@casehubio/pages-ui-tokens', () => ({
  applyTheme: vi.fn(),
  getTheme: vi.fn(() => ''),
  listThemes: vi.fn(() => ['default-light']),
  registerTheme: vi.fn(),
}));

vi.mock('../layout/elk-layout.js', () => ({
  computeElkLayout: vi.fn(async (nodes: unknown[]) => nodes),
}));

import './PagesGraphCanvas.js';

describe('PagesGraphCanvas', () => {
  let element: HTMLElement;

  beforeEach(() => {
    element = document.createElement('pages-graph-canvas');
  });

  afterEach(() => {
    element.remove();
  });

  it('registers as pages-graph-canvas custom element', () => {
    expect(customElements.get('pages-graph-canvas')).toBeDefined();
  });

  it('accepts props property', () => {
    const props = {
      sourceColumn: 'src',
      targetColumn: 'tgt',
      direction: 'RIGHT' as const,
      lookup: { uuid: 'ds', columnCount: 2 },
    };
    (element as any).props = props;
    expect((element as any).props).toBe(props);
  });

  it('maps algorithm "tree" to ELK "mrtree"', () => {
    const props = { algorithm: 'tree' as const, lookup: { uuid: 'ds', columnCount: 2 } };
    (element as any).props = props;
    const opts = (element as any).buildLayoutOptions();
    expect(opts.algorithm).toBe('mrtree');
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `yarn vitest run packages/graph-renderer/src/bridge/pages-graph-canvas.test.ts`
Expected: FAIL — module not found

- [ ] **Step 4: Create PagesGraphCanvas bridge**

File: `packages/graph-renderer/src/bridge/PagesGraphCanvas.ts`:

```typescript
import { LitElement, html, type TemplateResult } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { GraphCanvasProps } from '@casehubio/pages-component';
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import type { ElkLayoutOptions } from '../layout/elk-layout.js';
import './GraphCanvas.js';

const ALGORITHM_MAP: Record<string, string> = {
  tree: 'mrtree',
  layered: 'layered',
  radial: 'radial',
  force: 'force',
  stress: 'stress',
};

@customElement('pages-graph-canvas')
export class PagesGraphCanvas extends LitElement {
  @property({ attribute: false }) props: GraphCanvasProps | undefined;
  @property({ attribute: false }) dataSet: { columns: { id: string }[]; rows: { cells: { type: string; value: unknown }[] }[] } | undefined;

  override createRenderRoot(): HTMLElement {
    return this;
  }

  buildLayoutOptions(): ElkLayoutOptions | undefined {
    if (!this.props) return undefined;
    const opts: ElkLayoutOptions = {};
    if (this.props.direction) opts.direction = this.props.direction;
    if (this.props.spacing !== undefined) opts.spacing = this.props.spacing;
    if (this.props.algorithm) opts.algorithm = (ALGORITHM_MAP[this.props.algorithm] ?? this.props.algorithm) as ElkLayoutOptions['algorithm'];
    if (this.props.containerPadding !== undefined) opts.containerPadding = this.props.containerPadding;
    if (this.props.elk) {
      if (this.props.elk.wrapping !== undefined) opts.wrapping = this.props.elk.wrapping;
      if (this.props.elk.headerHeight !== undefined) opts.headerHeight = this.props.elk.headerHeight;
      if (this.props.elk.elkOptions) opts.elkOptions = this.props.elk.elkOptions;
    }
    return Object.keys(opts).length > 0 ? opts : undefined;
  }

  private buildModel(): GraphModel | undefined {
    if (!this.props || !this.dataSet) return undefined;
    const ds = this.dataSet;
    const sourceIdx = this.props.sourceColumn ? ds.columns.findIndex(c => c.id === this.props!.sourceColumn) : 0;
    const targetIdx = this.props.targetColumn ? ds.columns.findIndex(c => c.id === this.props!.targetColumn) : 1;
    if (sourceIdx < 0 || targetIdx < 0) return undefined;

    const nodeMap = new Map<string, GraphNode>();
    const edges: GraphEdge[] = [];

    for (const row of ds.rows) {
      const srcCell = row.cells[sourceIdx];
      const tgtCell = row.cells[targetIdx];
      if (!srcCell || !tgtCell) continue;
      const src = String(srcCell.value ?? '');
      const tgt = String(tgtCell.value ?? '');

      if (!nodeMap.has(src)) nodeMap.set(src, { id: src, type: 'default', properties: { label: src } });
      if (!nodeMap.has(tgt)) nodeMap.set(tgt, { id: tgt, type: 'default', properties: { label: tgt } });

      edges.push({
        id: `${src}->${tgt}`,
        sourceId: src,
        targetId: tgt,
        type: this.props.directed ? 'directed' : 'default',
      });
    }

    return { nodes: [...nodeMap.values()], edges };
  }

  override render(): TemplateResult {
    const model = this.buildModel();
    const layoutOptions = this.buildLayoutOptions();

    return html`
      <graph-canvas-core
        .model=${model}
        .layoutOptions=${layoutOptions}
        .connectionsEnabled=${this.props?.connectionsEnabled ?? true}
      ></graph-canvas-core>
    `;
  }
}
```

- [ ] **Step 5: Export from index**

Add to `packages/graph-renderer/src/index.ts`:
```typescript
export { PagesGraphCanvas } from './bridge/PagesGraphCanvas.js';
```

- [ ] **Step 6: Run tests**

Run: `yarn vitest run packages/graph-renderer/`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/graph-renderer/
git commit -m "feat(#413): create PagesGraphCanvas YAML bridge — data-to-model, layout option mapping Refs #413"
```

---

## References

- [specs/issue-413-broaden-spi-interfaces/2026-09-07-broaden-spi-interfaces-design.md] — design spec
- [packages/pages-component/src/model/displayer-types.ts] — SPI interfaces
- [packages/pages-component/src/model/type-guards.ts:64-134] — ComponentTypeRegistry
- [packages/pages-viz/src/base/PagesChartElement.ts:15-16,54] — generic constraint, backgroundColor
- [packages/pages-viz/src/charts/option-pipeline.ts:53-188] — applyChartSettings
- [packages/pages-viz/src/charts/PagesGraph.ts:167] — missing cartesianAxes:false bug
- [packages/pages-viz/src/charts/PagesDensityHeatmap.ts:100-107] — radius wiring bug
- [packages/graph-renderer/src/bridge/GraphCanvas.ts:22] — pages-graph-canvas tag
- [packages/pages-schema/src/schema-registry.ts] — componentSchemaRegistry
- [packages/pages-ui/src/parser/displayer-desugar.ts:10-43] — TYPE_MAP
- [docs/protocols/casehub/yaml-properties-require-interface-declaration.md] — PP-20260907-47c747
- [docs/protocols/casehub/generated-schemas-not-hand-written.md] — PP-20260907-951cbe
- [casehubio/casehub-pages#411] — auto-generated Zod schemas (predecessor)
