# Design: Broaden SPI Interfaces for Chart, Map, and Graph Components

**Issue:** casehubio/casehub-pages#413
**Date:** 2026-09-07
**Branch:** issue-413-broaden-spi-interfaces
**Predecessor:** #411 (auto-generated Zod schemas — landed)

## Overview

Four deliverables that widen the typed SPI surface while preserving implementation-agnosticism:

1. **Interface hierarchy restructuring** — split ChartSettings along the Cartesian boundary so non-Cartesian charts stop inheriting meaningless xAxis/yAxis/grid
2. **Property promotions** — add universal/likely-portable properties to SPI interfaces (two-library mapping test: each promoted property must map to at least two charting libraries' APIs)
3. **Typed escape hatch pattern** — per-backend typed properties (`echarts?`, `reactFlow?`, `elk?`, `heatmapJs?`) that give power users full library access with schema validation, complementing the universal SPI
4. **GraphCanvas YAML integration** — register `pages-graph-canvas` as a new YAML component type with its own props interface

Principle: the SPI is the common denominator. The typed escape hatch covers everything beyond it. `extra: Record<string, unknown>` remains as an untyped last resort.

## 1. Interface Hierarchy Restructuring

### Current hierarchy

```
DataComponentCommon
├── ChartSettings                    ← resizable, zoom, maxWidth, maxHeight, legend, margin, xAxis, yAxis, grid, extra
│   ├── BarChartProps, LineChartProps, AreaChartProps, ScatterChartProps, ...
│   ├── PieChartProps                ← inherits xAxis/yAxis/grid (meaningless)
│   ├── MapProps                     ← inherits xAxis/yAxis/grid (meaningless)
│   ├── MeterProps                   ← inherits xAxis/yAxis/grid (meaningless)
│   ├── TreemapChartProps            ← inherits xAxis/yAxis/grid (meaningless)
│   └── GraphProps                   ← inherits xAxis/yAxis/grid (meaningless)
└── DensityHeatmapProps              ← no extra escape hatch
```

### New hierarchy

```
DataComponentCommon                  ← + maxWidth?, maxHeight? (moved from ChartSettings)
├── ChartSettingsBase                ← NEW: resizable, legend, margin, tooltip, animation, color, backgroundColor, extra
│   ├── ChartSettings                ← xAxis, yAxis, grid, zoom (Cartesian only)
│   │   ├── BarChartProps
│   │   ├── LineChartProps
│   │   ├── AreaChartProps
│   │   ├── ScatterChartProps
│   │   ├── BubbleChartProps
│   │   ├── TimeseriesProps
│   │   ├── HeatmapChartProps
│   │   └── TimelineProps            ← Cartesian (xAxis: time, yAxis: category)
│   ├── PieChartProps                ← extends ChartSettingsBase (non-Cartesian)
│   ├── MapProps                     ← extends ChartSettingsBase (non-Cartesian)
│   ├── MeterProps                   ← extends ChartSettingsBase (non-Cartesian)
│   ├── TreemapChartProps            ← extends ChartSettingsBase (non-Cartesian)
│   └── GraphProps                   ← extends ChartSettingsBase (non-Cartesian, ECharts graph)
├── DensityHeatmapProps              ← extends DataComponentCommon only, + extra directly
├── EventTimelineProps               ← extends DataComponentCommon only (no ChartSettings)
└── GraphCanvasProps                 ← NEW: extends DataComponentCommon (React Flow + ELK)
```

### Why ChartSettingsBase, not VizCommon

The original proposal was a cross-backend `VizCommon` base. Decision review rejected this: `zoom` means ECharts dataZoom but has no counterpart in @drdreo/heatmap — the abstraction leaks. `ChartSettingsBase` is coherent because it targets a single backend family (ECharts) and contains only properties that every ECharts chart type supports.

`DensityHeatmapProps` stays independent because @drdreo/heatmap has different capabilities. `GraphCanvasProps` stays independent because React Flow has its own interaction model.

### maxWidth / maxHeight migration

`maxWidth` and `maxHeight` are CSS dimension constraints that apply to any renderable component, not just visualizations. They move from `ChartSettings` to `DataComponentCommon`, alongside the existing `width` and `height`.

### PagesChartElement generic constraint

The `PagesChartElement` base class in `pages-viz/src/base/PagesChartElement.ts` has the generic constraint:

```typescript
export abstract class PagesChartElement<
  P extends VizComponentProps & ChartSettings,
> extends PagesElement<P> {
```

After the split, non-Cartesian chart components (PagesPieChart, PagesMap, PagesMeter, PagesTreemapChart, PagesGraph) pass props extending `ChartSettingsBase`, not `ChartSettings`. The constraint widens to match:

```typescript
export abstract class PagesChartElement<
  P extends VizComponentProps & ChartSettingsBase,
> extends PagesElement<P> {
```

This is the same class of fix as `applyChartSettings` below — both are consequences of the hierarchy split.

### applyChartSettings refactoring

The current function signature in `option-pipeline.ts`:

```typescript
function applyChartSettings(
  option: Record<string, unknown>,
  props: { title?: string } & ChartSettings,
  settingsOptions?: ChartSettingsOptions,
): Record<string, unknown>
```

After the hierarchy split, the signature widens to accept `ChartSettingsBase`:

```typescript
function applyChartSettings(
  option: Record<string, unknown>,
  props: { title?: string } & ChartSettingsBase,
): Record<string, unknown>
```

Changes:

1. **Widen type** — `ChartSettings` → `ChartSettingsBase` so non-Cartesian components compile without casting
2. **Eliminate `cartesianAxes` flag** — remove the `ChartSettingsOptions` parameter entirely. Cartesian-specific properties (`xAxis`, `yAxis`, `grid`, `zoom`) are handled via structural access (`'xAxis' in props`). The type system does the work: non-Cartesian props don't carry these fields, so the structural check is false at runtime.
3. **Centralise escape hatch merging** — the function applies the full merge pipeline in order:
   1. SPI base properties (title, legend, margin, tooltip, animation, color, backgroundColor)
   2. Cartesian SPI properties via structural access (xAxis, yAxis, grid, zoom)
   3. `deepMerge(option, props.echarts)` — typed escape hatch
   4. `deepMerge(option, props.extra)` — untyped last resort

   Later stages override earlier ones. Each component's `buildOption()` simplifies to: build initial option → `applyChartSettings(option, props)` → return. The per-component `if (props.extra) { deepMerge(...) }` boilerplate is eliminated.

## 2. ChartSettingsBase — New Shared Properties

All ECharts chart types inherit these.

### Tooltip

```typescript
tooltip?: {
  show?: boolean;
  trigger?: "item" | "axis" | "none";
}
```

Two-library mapping: ECharts `tooltip.show`/`tooltip.trigger` ↔ Chart.js `plugins.tooltip.enabled`/`plugins.tooltip.mode` ↔ Plotly `hovermode`.

### Animation

```typescript
animation?: boolean;
```

Two-library mapping: ECharts `animation` ↔ Chart.js `animation` (false to disable) ↔ Plotly `layout.transition`.

### Color palette

```typescript
color?: string[];
backgroundColor?: string;
```

Two-library mapping: ECharts `color` ↔ Chart.js `defaults.color` ↔ Plotly `colorway`. Background: ECharts `backgroundColor` ↔ Chart.js `plugins.background`.

**Interaction with `PagesChartElement.updated()`:** The current code unconditionally sets `option['backgroundColor'] = 'transparent'` after `buildOption()` returns, overriding any value set by the merge pipeline. With `backgroundColor` now an SPI property, this default must become conditional: `option['backgroundColor'] ??= 'transparent'` — only apply the transparent default when no tier of the pipeline has set a value.

### Legend extensions

Expand the existing `legend` type:

```typescript
legend?: {
  show?: boolean;         // existing
  position?: "top" | "bottom" | "left" | "right";  // existing
  orient?: "horizontal" | "vertical";               // NEW
  selectedMode?: boolean | "single" | "multiple";   // NEW
}
```

Two-library mapping for `orient`: ECharts `legend.orient` ↔ Plotly `legend.orientation` ↔ Chart.js `legend.position` (vertical ↔ "left"/"right").

Two-library mapping for `selectedMode`: ECharts `legend.selectedMode` ↔ Plotly `legend.itemclick` (`"toggle"` = multiple, `"toggleothers"` = single, `false` = disabled) ↔ Chart.js `onClick` handler (null = disabled, default = multiple, custom = single).

### Typed escape hatch

```typescript
echarts?: CasehubEChartsExtension;
```

See §8 for the curated extension type.

## 3. ChartSettings — Cartesian Promotions

Only Cartesian charts (bar, line, area, scatter, bubble, timeseries, heatmap-chart) inherit these.

### Axis extensions

Expand the existing `xAxis`/`yAxis` types:

```typescript
xAxis?: {
  title?: string;           // existing
  showLabels?: boolean;     // existing
  labelAngle?: number;      // existing
  type?: "value" | "category" | "time" | "log";  // NEW
  min?: number | "dataMin";                        // NEW
  max?: number | "dataMax";                        // NEW
  inverse?: boolean;                               // NEW
}
// yAxis mirrors xAxis
```

Two-library mapping: ECharts `xAxis.type` ↔ Chart.js `scales.x.type` ↔ Plotly `xaxis.type` ↔ D3 scale types.

### DataZoom evolution

Expand `zoom` from boolean to union:

```typescript
zoom?: boolean | {
  enabled?: boolean;
  start?: number;   // 0-100 percent
  end?: number;     // 0-100 percent
}
```

`zoom: true` remains a shorthand for `{ enabled: true }`. Two-library mapping: ECharts `dataZoom[].start/end` ↔ Plotly `xaxis.rangeslider`.

## 4. Per-Chart Series Promotions

Per-chart properties promote domain-level chart concepts (stepped line, rose variant, zoom-on-click) to typed SPI fields. The two-library mapping test from §2-§3 is relaxed here: each promoted property must represent a concept with equivalents in at least two charting libraries, though the exact property names and semantics may differ. Properties that are raw library config knobs with no cross-library conceptual equivalent belong in the `echarts` typed escape hatch instead.

### BarChartProps

```typescript
interface BarChartProps extends DataComponentCommon, ChartSettings {
  subtype?: "column" | "column-stacked" | "bar" | "bar-stacked";  // existing
  barWidth?: string | number;  // NEW — bar width in px or percent
  barGap?: string;             // NEW — gap between bars in same category
}
```

### LineChartProps

```typescript
interface LineChartProps extends DataComponentCommon, ChartSettings {
  subtype?: "line" | "smooth";  // existing
  step?: false | "start" | "end" | "middle";  // NEW — stepped line
  connectNulls?: boolean;                      // NEW — bridge null values
  showSymbol?: boolean;                        // NEW — show data point markers
}
```

### AreaChartProps

No additional promotions — inherits ChartSettings extensions.

### PieChartProps

```typescript
interface PieChartProps extends DataComponentCommon, ChartSettingsBase {  // NOTE: ChartSettingsBase
  subtype?: "pie" | "donut";        // existing
  roseType?: "radius" | "area";     // NEW — nightingale/rose chart
  startAngle?: number;              // NEW — starting angle in degrees
  clockwise?: boolean;              // NEW — drawing direction
}
```

### ScatterChartProps

```typescript
interface ScatterChartProps extends DataComponentCommon, ChartSettings {
  symbolSize?: number;  // NEW — data point size in px
}
```

### BubbleChartProps

No additional promotions — already has `minRadius`/`maxRadius`.

### HeatmapChartProps (ECharts heatmap)

```typescript
interface HeatmapChartProps extends DataComponentCommon, ChartSettings {
  minColor?: string;                 // existing
  maxColor?: string;                 // existing
  blurSize?: number;                 // NEW — gradient blur radius
  minOpacity?: number;               // NEW — minimum cell opacity
  maxOpacity?: number;               // NEW — maximum cell opacity
}
```

### TreemapChartProps

```typescript
interface TreemapChartProps extends DataComponentCommon, ChartSettingsBase {  // NOTE: ChartSettingsBase
  parentColumn?: ColumnId;           // existing
  colorColumn?: ColumnId;            // existing
  sort?: boolean | "asc" | "desc";   // NEW — node sorting
  leafDepth?: number;                // NEW — visible depth levels
  nodeClick?: "zoomToNode" | "link" | false;  // NEW — click behavior
}
```

### MeterProps (gauge)

```typescript
interface MeterProps extends DataComponentCommon, ChartSettingsBase {  // NOTE: ChartSettingsBase
  end?: number;                  // existing
  warning?: number;              // existing
  critical?: number;             // existing
  startAngle?: number;           // NEW — gauge start angle
  endAngle?: number;             // NEW — gauge end angle
  clockwise?: boolean;           // NEW — drawing direction
}
```

### TimeseriesProps

```typescript
interface TimeseriesProps extends DataComponentCommon, ChartSettings {
  connectNulls?: boolean;  // NEW — bridge null values
}
```

### TimelineProps

No additional promotions — already has startColumn/endColumn/labelColumn/categoryColumn.

## 5. MapProps

```typescript
interface MapProps extends DataComponentCommon, ChartSettingsBase {  // NOTE: ChartSettingsBase
  subtype?: "regions" | "markers";   // existing
  colorScheme?: string;              // existing
  mapName?: string;                  // existing
  roam?: boolean | "pan" | "zoom";   // NEW — pan/zoom interaction
  center?: [number, number];         // NEW — initial center [lng, lat]
  zoom?: number;                     // NEW — initial zoom level
  scaleLimit?: { min?: number; max?: number };  // NEW — zoom bounds
  showLabel?: boolean;               // NEW — region name labels
  selectedMode?: "single" | "multiple" | boolean;  // NEW — region selection
}
```

`zoom` here is geographic zoom level (ECharts `geo.zoom`), distinct from `ChartSettings.zoom` which is dataZoom. No naming collision since `MapProps` extends `ChartSettingsBase` (which does not contain `zoom`), not `ChartSettings`.

## 6. GraphProps (ECharts Graph)

Scoped to the ECharts PagesGraph component (registered as `"graph"` in TYPE_MAP). Does NOT control GraphCanvas (React Flow + ELK).

```typescript
interface GraphProps extends DataComponentCommon, ChartSettingsBase {  // NOTE: ChartSettingsBase
  layout?: "force" | "circular" | "none";  // existing
  sourceColumn?: ColumnId;                  // existing
  targetColumn?: ColumnId;                  // existing
  valueColumn?: ColumnId;                   // existing
  directed?: boolean;                       // existing
  nodeLabelColumn?: ColumnId;               // existing
  nodeColorColumn?: ColumnId;               // existing
  nodeColorMap?: Record<string, string>;    // existing
  nodeSizeColumn?: ColumnId;                // existing
  repulsion?: number;                       // NEW — force layout repulsion strength
  edgeLabel?: boolean;                      // NEW — show edge labels
  roam?: boolean | "pan" | "zoom";          // NEW — pan/zoom interaction
  symbol?: string;                          // NEW — node shape
}
```

## 7. DensityHeatmapProps

```typescript
interface DensityHeatmapProps extends DataComponentCommon {
  xColumn?: ColumnId;               // existing
  yColumn?: ColumnId;               // existing
  valueColumn?: ColumnId;           // existing
  gradient?: readonly { offset: number; color: string }[];  // existing
  radius?: number;                  // existing (BUG: not wired through — fix)
  aggregation?: "max" | "sum" | "mean" | "count";          // existing
  showTooltip?: boolean;            // existing
  showLegend?: boolean;             // existing
  blur?: number;                    // NEW — gradient falloff (0-1)
  maxOpacity?: number;              // NEW — maximum opacity (0-1)
  minOpacity?: number;              // NEW — minimum opacity (0-1)
  intensityExponent?: number;       // NEW — non-linear intensity curve
  valueMin?: number;                // NEW — fixed scale minimum
  valueMax?: number;                // NEW — fixed scale maximum
  extra?: Readonly<Record<string, unknown>>;  // NEW — escape hatch
  heatmapJs?: CasehubHeatmapExtension;       // NEW — typed escape hatch
}
```

### Merge pipeline

`PagesDensityHeatmap` extends `PagesElement` (not `PagesChartElement`) and does not use `applyChartSettings`. Its merge pipeline follows the same precedence principle, applied to the `@drdreo/heatmap` config:

1. Component builds heatmap config from SPI properties (gradient, radius, aggregation, blur, maxOpacity, minOpacity, intensityExponent, valueMin, valueMax)
2. `deepMerge(config, props.heatmapJs)` — typed escape hatch for `@drdreo/heatmap`-specific options
3. `deepMerge(config, props.extra)` — untyped last resort

This is implemented in the `createInstance` method alongside the existing SPI property extraction.

### Wiring bug fix

`radius` is declared in `DensityHeatmapProps` but never passed to the `createHeatmap` config in `PagesDensityHeatmap.ts`. The `createInstance` method (around line 100-107) only passes `gradient` and `aggregationMode` — `radius` must be added to the config object.

## 8. Typed Escape Hatch Pattern

### Principle

The SPI captures universal/likely-portable properties. The typed escape hatch covers the long tail of library-specific features. The property name makes coupling explicit — YAML using `echarts:` knows it's tied to the ECharts backend.

Three tiers of access per component:
1. **SPI properties** — portable, always preferred
2. **Typed escape hatch** (`echarts?`, `reactFlow?`, etc.) — backend-specific but schema-validated
3. **`extra`** — untyped last resort for truly exotic options

### Merge pipeline

**Universal precedence principle:** Every component that has escape hatches applies them in the same order — SPI properties → typed escape hatch → `extra`. Later stages override earlier ones. This applies across all rendering backends (ECharts, @drdreo/heatmap, React Flow + ELK). Per-backend pipelines are specified in §7 (DensityHeatmap), §9 (GraphCanvas), and below (ECharts).

**ECharts merge pipeline** — centralised in `applyChartSettings` (see §1 refactoring) and applied in this order:

1. **Component `buildOption()`** — builds the initial ECharts option (series, axes, etc.)
2. **SPI base properties** — title, legend, margin, tooltip, animation, color, backgroundColor
3. **SPI Cartesian properties** (structural check) — xAxis, yAxis, grid, zoom
4. **`deepMerge(option, props.echarts)`** — typed escape hatch overrides SPI decisions
5. **`deepMerge(option, props.extra)`** — untyped last resort overrides everything

Later stages override earlier ones on conflict. If a user sets `animation: false` (SPI) and `echarts: { animationDuration: 1000 }`, the escape hatch wins — the explicit override is intentional.

**Array broadcast semantics:** `deepMerge` handles the case where an escape hatch property is an object and the existing option property is an array. The object is broadcast-merged into each array element. For example, `echarts: { series: { barCategoryGap: "1%" } }` merges `{ barCategoryGap: "1%" }` into every series item in the option, preserving the existing array structure. This is the intended usage for `CasehubEChartsExtension.series` — it is typed as `Record<string, unknown>` (not an array) to guide users toward the broadcast pattern rather than array replacement.

### CasehubEChartsExtension

Curated subset of stable ECharts top-level option categories. Lives in `pages-component/src/model/echarts-extension.ts`.

```typescript
export interface CasehubEChartsExtension {
  toolbox?: {
    show?: boolean;
    feature?: {
      saveAsImage?: { show?: boolean; title?: string };
      dataZoom?: { show?: boolean };
      restore?: { show?: boolean };
      dataView?: { show?: boolean; readOnly?: boolean };
    };
    orient?: "horizontal" | "vertical";
  };
  dataZoom?: Array<{
    type?: "slider" | "inside";
    filterMode?: "filter" | "weakFilter" | "empty" | "none";
    xAxisIndex?: number | number[];
    yAxisIndex?: number | number[];
  }>;
  animationDuration?: number;
  animationEasing?: string;
  animationDurationUpdate?: number;
  animationThreshold?: number;
  darkMode?: boolean;
  series?: Record<string, unknown>;
}
```

### CasehubReactFlowExtension

```typescript
export interface CasehubReactFlowExtension {
  panOnDrag?: boolean;
  zoomOnScroll?: boolean;
  zoomOnPinch?: boolean;
  snapToGrid?: boolean;
  snapGrid?: [number, number];
  colorMode?: "light" | "dark" | "system";
  defaultMarkerColor?: string;
  selectionMode?: "full" | "partial";
  elementsSelectable?: boolean;
}
```

### CasehubElkExtension

```typescript
export interface CasehubElkExtension {
  wrapping?: boolean;
  headerHeight?: number;
  elkOptions?: Readonly<Record<string, string>>;
}
```

### CasehubHeatmapExtension

```typescript
export interface CasehubHeatmapExtension {
  blendMode?: string;
}
```

### Schema generation

The schema generator already handles interfaces in `pages-component/src/model/`. The curated extension types live alongside the SPI interfaces and are processed identically — no generator changes needed. The generator produces Zod schemas from these types the same way it handles `ChartSettings` sub-objects.

## 9. GraphCanvasProps (NEW YAML Component Type)

### Registration

Five registration points:

1. **`ComponentTypeRegistry`** in `type-guards.ts`:

```typescript
"graph-canvas": GraphCanvasProps
```

2. **`DATA_COMPONENT_TYPES`** in `component-desugar.ts` — add `"graph-canvas"` to the set so YAML with `type: graph-canvas` routes through `desugarDisplayer`:

```typescript
"graph-canvas",
```

3. **`TYPE_MAP`** in `displayer-desugar.ts` — both legacy and modern entries (following the established pattern):

```typescript
GRAPH_CANVAS: "graph-canvas",       // legacy underscore format
"GRAPH-CANVAS": "graph-canvas",     // modern hyphenated format
```

4. **`componentSchemaRegistry`** in `schema-registry.ts` — hand-maintained Map entry:

```typescript
["graph-canvas", graphCanvasPropsSchema],
```

5. **`pages-schema/src/index.ts`** — add `graphCanvasPropsSchema` to the public exports from `component-schemas.generated.js`

### Interface

```typescript
export interface GraphCanvasProps extends DataComponentCommon {
  // Data mapping (same pattern as GraphProps)
  sourceColumn?: ColumnId;
  targetColumn?: ColumnId;
  nodeLabelColumn?: ColumnId;
  nodeColorColumn?: ColumnId;
  nodeColorMap?: Record<string, string>;
  nodeSizeColumn?: ColumnId;
  valueColumn?: ColumnId;
  directed?: boolean;

  // Layout (universal graph layout concepts — SPI names mapped to ELK internally)
  direction?: "DOWN" | "RIGHT" | "LEFT" | "UP";
  spacing?: number;
  algorithm?: "layered" | "tree" | "radial" | "force" | "stress";  // "tree" maps to ELK "mrtree"
  containerPadding?: number;

  // Interaction
  connectionsEnabled?: boolean;
  fitView?: boolean;
  nodesDraggable?: boolean;
  minZoom?: number;
  maxZoom?: number;

  // Edge appearance
  edgeType?: "default" | "straight" | "step" | "smoothstep";
  edgeAnimated?: boolean;

  // Escape hatches
  extra?: Readonly<Record<string, unknown>>;
  reactFlow?: CasehubReactFlowExtension;
  elk?: CasehubElkExtension;
}
```

### Data-to-model bridge

GraphCanvas currently accepts a `model: GraphModel` property (programmatic). For YAML integration, the desugarer needs a bridge that constructs a `GraphModel` from dataset columns — the same pattern PagesGraph uses to build ECharts graph data from `sourceColumn`/`targetColumn`/etc.

The bridge lives in `graph-renderer` as a new `PagesGraphCanvas` Lit element alongside the existing `GraphCanvas.ts` (which already wraps React Flow via `createRoot` and has `@xyflow/react` + `elkjs` dependencies). `pages-viz` wraps ECharts exclusively; `pages-ui` is the parser layer — neither is the right home. The `PagesGraphCanvas` element:
1. Accepts `GraphCanvasProps` from the desugarer
2. Subscribes to the dataset via `DataSourceController` (see below)
3. Builds a `GraphModel` from the dataset rows using the column mappings
4. Passes the model + layout options to the inner `<graph-canvas-core>` element

#### Custom element tag rename

The runtime activation convention creates elements via `pages-${type}`. For component type `"graph-canvas"`, this produces `pages-graph-canvas`. But that tag is already registered by the existing `GraphCanvas` element at `graph-renderer/src/bridge/GraphCanvas.ts` (`@customElement('pages-graph-canvas')`), which accepts `model: GraphModel` for the programmatic API.

The bridge must claim `pages-graph-canvas` (it's the YAML-facing element). The existing `GraphCanvas` is renamed from `pages-graph-canvas` to `graph-canvas-core` — it drops the `pages-` prefix since it's not a YAML-activated component. Programmatic consumers update their selectors (`pages-graph-canvas` → `graph-canvas-core`) — a mechanical migration.

#### Dependency addition

`graph-renderer/package.json` must add `@casehubio/pages-component` to its dependencies. This provides `DataSourceController` — the controller that manages dataset state (loading, data, error, sort, pagination).

#### Data subscription via DataSourceController

`PagesGraphCanvas` uses `DataSourceController` via **composition** (not by extending `PagesElement`). `PagesElement` is `pages-viz`'s base class — it couples data binding with ECharts-oriented rendering (loading skeletons, error states, chart sizing). `PagesGraphCanvas` doesn't need that rendering — it delegates to the existing `<pages-graph-canvas>` element.

The composition pattern:

```typescript
@customElement('pages-graph-canvas')
export class PagesGraphCanvas extends LitElement {
  @property({ attribute: false }) props: GraphCanvasProps | undefined;

  readonly controller = new DataSourceController({
    onChange: () => this.requestUpdate(),
    onRefresh: () => { this._dataRequested = false; this.requestDataIfNeeded(); },
  });

  // Same pages-data-request event dispatch protocol as PagesElement
  private requestDataIfNeeded(): void { /* ... */ }
}
```

This follows the same data subscription protocol (dispatching `pages-data-request` custom events) that `PagesElement` uses, without inheriting its rendering concerns. `DataSourceController` is the shared abstraction — it's already extracted into `pages-component` as a composition-friendly controller.

#### Merge pipeline

The GraphCanvas merge pipeline follows the same precedence principle as ECharts (SPI → typed escape hatch → extra), applied to two backend targets:

1. **React Flow options:** SPI interaction properties (fitView, nodesDraggable, connectionsEnabled, minZoom, maxZoom, edgeType, edgeAnimated) → `deepMerge(reactFlowProps, props.reactFlow)` → `deepMerge(reactFlowProps, props.extra)` (`extra` targets React Flow as the primary rendering backend)
2. **ELK options:** SPI layout properties (direction, spacing, algorithm, containerPadding) → `deepMerge(elkOptions, props.elk)` (ELK options are fully covered by `CasehubElkExtension.elkOptions` — no need for `extra`)

### Desugarer handling

The desugarer extracts `GraphCanvasProps` identically to `GraphProps` — the schema-aware passthrough validates against the generated Zod schema. No special extraction logic needed beyond adding the TYPE_MAP entry.

## 10. Wiring Bug Fixes

### Bug 1: DensityHeatmap `radius` not wired

**File:** `PagesDensityHeatmap.ts`, `createInstance` method (~line 100-107)
**Fix:** Add `radius` to the config object passed to `createHeatmap()`.

### Bug 2: PagesGraph missing `{ cartesianAxes: false }`

**File:** `PagesGraph.ts`, `applyChartSettings` call
**Current bug:** `PagesGraph` calls `applyChartSettings(option, props)` without `{ cartesianAxes: false }`, unlike PagesPieChart, PagesMap, PagesMeter, PagesTreemapChart.
**Fix:** Absorbed by the §1 `applyChartSettings` refactoring. The `cartesianAxes` flag is eliminated entirely — the refactored function uses structural access (`'xAxis' in props`). Since `GraphProps` extends `ChartSettingsBase` (not `ChartSettings`), the structural check is false at runtime and Cartesian properties are naturally skipped. No code change needed in `PagesGraph.ts` beyond adopting the new call signature (which loses the options parameter).

## 11. Testing Strategy

Three layers of testing per new property:

### Layer 1: Schema staleness (existing infrastructure)

The staleness test in `pages-schema` runs the generator in dry-run mode and compares output to the committed file. Adding properties to interfaces triggers a regeneration; the test catches drift.

### Layer 2: YAML → desugarer round-trip tests

For each newly promoted property, verify that when set in YAML, it appears in the desugared component's props with the correct value. This catches `handledKeys` collisions where the desugarer swallows a property that should pass through.

```typescript
test("barWidth passes through desugarer", () => {
  const yaml = { type: "BARCHART", properties: { barWidth: 20, lookup: "ds" } };
  const result = desugar(yaml);
  expect(result.props.barWidth).toBe(20);
});
```

### Layer 3: Renderer unit tests

For each newly promoted property, verify it reaches the upstream library's option object. This catches wiring bugs like the `radius` issue.

```typescript
test("barWidth reaches ECharts series option", () => {
  const props: BarChartProps = { barWidth: 20, lookup: mockLookup };
  const option = buildBarChartOption(props, mockData);
  expect(option.series[0].barWidth).toBe(20);
});
```

### GraphCanvasProps testing

- Registration test: verify `"graph-canvas"` is in `ComponentTypeRegistry`
- Desugarer test: verify YAML with `type: GRAPH-CANVAS` routes correctly
- Data bridge test: verify dataset columns produce a valid `GraphModel`
- Layout option passthrough: verify `direction`, `spacing`, `algorithm` reach `ElkLayoutOptions`

## 12. Execution Order

1. **Interface hierarchy + base class refactoring** — extract `ChartSettingsBase`, move `maxWidth`/`maxHeight` to `DataComponentCommon`, re-parent non-Cartesian charts, widen both `PagesChartElement<P>` constraint and `applyChartSettings` to `ChartSettingsBase`, eliminate `cartesianAxes` flag, centralise escape hatch merging, fix `PagesChartElement.updated()` `backgroundColor` guard
2. **ChartSettingsBase + ChartSettings promotions** — add tooltip, animation, color, axis extensions, zoom evolution
3. **Per-chart series promotions** — add bar, line, pie, scatter, heatmap-chart, treemap, meter, timeseries properties
4. **MapProps + GraphProps promotions** — add roam, center, zoom, labels, etc.
5. **DensityHeatmapProps** — add new properties, `extra`, fix `radius` wiring
6. **Typed escape hatch types** — create curated extension interfaces
7. **GraphCanvasProps + registration** — new interface, all five registration points (ComponentTypeRegistry, DATA_COMPONENT_TYPES, TYPE_MAP, componentSchemaRegistry, exports), rename existing `GraphCanvas` tag from `pages-graph-canvas` to `graph-canvas-core`, data-to-model bridge in `graph-renderer`
8. **Schema regeneration** — run `yarn workspace @casehubio/pages-schema run generate`
9. **Tests** — schema staleness, desugarer round-trips, renderer unit tests

Each batch is independently testable and committable. Batches 1-6 can proceed without batch 7 (GraphCanvas is additive).

## References

- [packages/pages-component/src/model/displayer-types.ts] — current SPI interfaces (audit target)
- [packages/pages-component/src/model/type-guards.ts:64-134] — ComponentTypeRegistry
- [packages/pages-schema/scripts/generate-schemas.ts] — schema generator
- [packages/pages-schema/src/component-schemas.generated.ts] — generated schemas
- [packages/pages-ui/src/parser/displayer-desugar.ts] — desugarer and TYPE_MAP
- [packages/pages-viz/package.json] — echarts ^5.6.0, @drdreo/heatmap ^1.4.1
- [packages/graph-renderer/package.json] — @xyflow/react ^12.4.0, elkjs ^0.9.3
- [packages/graph-renderer/src/GraphCanvas.ts] — React Flow + ELK component
- [packages/graph-renderer/src/layout/elk-layout.ts] — ElkLayoutOptions interface
- [docs/protocols/casehub/yaml-properties-require-interface-declaration.md] — PP-20260907-47c747
- [docs/protocols/casehub/generated-schemas-not-hand-written.md] — PP-20260907-951cbe
- [casehubio/casehub-pages#411] — auto-generated Zod schemas (predecessor, landed)
