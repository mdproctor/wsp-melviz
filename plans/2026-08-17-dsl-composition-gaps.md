# DSL Composition Gaps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #317 — epic: DSL composition gaps — heatmap, conditional styling, metric row, timeline, master-detail
**Issue group:** #317

**Goal:** Close 4 remaining DSL gaps: 7 missing builders + type guards, KPI metric enhancements, event timeline component, and master-detail convenience builder.

**Architecture:** Extend the existing builder/component/type-guard layering. Work item 1 is mechanical (freeze-and-return builders). Work item 2 enhances existing `metricGrid()` and `PagesMetric`. Work item 3 adds a new `PagesEventTimeline` component in pages-viz with strategy-pattern data transformation. Work item 4 adds a convenience builder that composes `split()` + selection wiring.

**Tech Stack:** TypeScript 5, Lit (web components), ECharts (existing timeline), Vitest

## Global Constraints

- All builders follow the `freeze({ type, props: { ...props } })` pattern
- All new component types must have: props interface, type guard, registry entry, activation entry, builder function
- Web components use `@customElement('pages-<name>')` with `--pages-*` CSS tokens
- New components extend `PagesElement` (data-bound) or `PagesContentElement` (props-only)
- Examples gallery auto-discovers `.dash.yaml` files from `examples/samples/` — no manual registration needed
- `mcp__intellij-index__*` tools for all code navigation and structural editing

---

## Batch 1: Missing builders and type guards

### Task 1: Add 4 missing type guard entries and 7 builder functions

**Files:**
- Modify: `packages/pages-component/src/model/type-guards.ts`
- Modify: `packages/pages-ui/src/dsl/builders.ts`
- Modify: `packages/pages-ui/src/dsl/builders.test.ts`

**Interfaces:**
- Consumes: `BadgeProps`, `CountdownProps`, `TimelineProps`, `GraphProps`, `HeatmapChartProps`, `TreemapChartProps`, `DensityHeatmapProps` from `displayer-types.ts`
- Produces: `badge()`, `countdown()`, `timeline()`, `graph()`, `heatmapChart()`, `treemapChart()`, `densityHeatmap()` builder functions; `isBadge()`, `isCountdown()`, `isTimeline()`, `isGraph()` type guards

- [ ] **Step 1: Write failing tests for all 7 builders**

Add to `packages/pages-ui/src/dsl/builders.test.ts`. Follow the existing `barChart` test pattern:

```typescript
describe("heatmapChart()", () => {
  it("creates heatmap-chart component with spread props", () => {
    const props = { lookup: { dataSetId: "test", operations: [] }, minColor: "#313695" };
    const result = heatmapChart(props as any);
    expect(result.type).toBe("heatmap-chart");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = heatmapChart({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});

describe("treemapChart()", () => {
  it("creates treemap-chart component with spread props", () => {
    const props = { lookup: { dataSetId: "test", operations: [] }, parentColumn: "parent" };
    const result = treemapChart(props as any);
    expect(result.type).toBe("treemap-chart");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = treemapChart({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});

describe("densityHeatmap()", () => {
  it("creates density-heatmap component with spread props", () => {
    const props = { lookup: { dataSetId: "test", operations: [] }, radius: 25 };
    const result = densityHeatmap(props as any);
    expect(result.type).toBe("density-heatmap");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = densityHeatmap({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});

describe("badge()", () => {
  it("creates badge component with spread props", () => {
    const props = { lookup: { dataSetId: "test", operations: [] }, column: "status" };
    const result = badge(props as any);
    expect(result.type).toBe("badge");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = badge({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});

describe("countdown()", () => {
  it("creates countdown component with spread props", () => {
    const props = { lookup: { dataSetId: "test", operations: [] }, format: "compact" };
    const result = countdown(props as any);
    expect(result.type).toBe("countdown");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = countdown({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});

describe("timeline()", () => {
  it("creates timeline component with spread props", () => {
    const props = { lookup: { dataSetId: "test", operations: [] }, startColumn: "begin" };
    const result = timeline(props as any);
    expect(result.type).toBe("timeline");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = timeline({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});

describe("graph()", () => {
  it("creates graph component with spread props", () => {
    const props = { lookup: { dataSetId: "test", operations: [] }, layout: "force" };
    const result = graph(props as any);
    expect(result.type).toBe("graph");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = graph({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders.test`
Expected: FAIL — functions not exported

- [ ] **Step 3: Add 4 registry entries and type guards to type-guards.ts**

In `packages/pages-component/src/model/type-guards.ts`, add to `ComponentTypeRegistry`:

```typescript
"badge": BadgeProps;
"countdown": CountdownProps;
"timeline": TimelineProps;
"graph": GraphProps;
```

And add type guard functions (after existing guards):

```typescript
export function isBadge(c: Component): c is TypedComponent<"badge"> {
  return c.type === "badge";
}

export function isCountdown(c: Component): c is TypedComponent<"countdown"> {
  return c.type === "countdown";
}

export function isTimeline(c: Component): c is TypedComponent<"timeline"> {
  return c.type === "timeline";
}

export function isGraph(c: Component): c is TypedComponent<"graph"> {
  return c.type === "graph";
}
```

Add imports for `BadgeProps`, `CountdownProps`, `TimelineProps`, `GraphProps` from `../model/displayer-types.js`.

- [ ] **Step 4: Add 7 builder functions to builders.ts**

In `packages/pages-ui/src/dsl/builders.ts`, add imports:

```typescript
import {
  // ... existing imports ...
  HeatmapChartProps,
  TreemapChartProps,
  DensityHeatmapProps,
  BadgeProps,
  CountdownProps,
  TimelineProps,
  GraphProps,
} from "@casehubio/pages-component";
```

Add builder functions after the existing `mapChart()` function (before `iframePlugin()`):

```typescript
export function heatmapChart(props: HeatmapChartProps): TypedComponent<"heatmap-chart"> {
  return freeze({ type: "heatmap-chart" as const, props: { ...props } });
}

export function treemapChart(props: TreemapChartProps): TypedComponent<"treemap-chart"> {
  return freeze({ type: "treemap-chart" as const, props: { ...props } });
}

export function densityHeatmap(props: DensityHeatmapProps): TypedComponent<"density-heatmap"> {
  return freeze({ type: "density-heatmap" as const, props: { ...props } });
}

export function badge(props: BadgeProps): TypedComponent<"badge"> {
  return freeze({ type: "badge" as const, props: { ...props } });
}

export function countdown(props: CountdownProps): TypedComponent<"countdown"> {
  return freeze({ type: "countdown" as const, props: { ...props } });
}

export function timeline(props: TimelineProps): TypedComponent<"timeline"> {
  return freeze({ type: "timeline" as const, props: { ...props } });
}

export function graph(props: GraphProps): TypedComponent<"graph"> {
  return freeze({ type: "graph" as const, props: { ...props } });
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders.test`
Expected: PASS

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts packages/pages-component/src/model/type-guards.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#317): add 7 missing DSL builders and 4 type guards

Add builder functions for heatmapChart, treemapChart, densityHeatmap,
badge, countdown, timeline, graph. Add ComponentTypeRegistry entries
and isX() type guards for badge, countdown, timeline, graph.

Refs #317"
```

## Batch 2: KPI metric enhancements

### Task 2: metricGrid direction option and MetricGridProps

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts`
- Modify: `packages/pages-component/src/model/type-guards.ts`
- Modify: `packages/pages-component/src/renderer/layout.ts`
- Modify: `packages/pages-component/src/renderer/layout.test.ts`
- Modify: `packages/pages-ui/src/dsl/builders.ts`
- Modify: `packages/pages-ui/src/dsl/builders.test.ts`

**Interfaces:**
- Consumes: existing `metricGrid()` builder, `applyLayoutStyles()` in `layout.ts`
- Produces: `MetricGridProps` interface, updated `metricGrid()` that accepts options, `direction: "row"` layout mode

- [ ] **Step 1: Write failing tests for metricGrid direction option**

In `packages/pages-ui/src/dsl/builders.test.ts`, update `metricGrid()` describe:

```typescript
it("accepts MetricGridOptions as first argument", () => {
  const child = html("a");
  const result = metricGrid({ direction: "row" }, child);

  expect(result.type).toBe("metric-grid");
  expect(result.props).toEqual({ direction: "row" });
  expect(result.slots).toEqual({ default: [child] });
});

it("treats first argument without direction as a child component", () => {
  const child = html("a");
  const result = metricGrid(child);

  expect(result.type).toBe("metric-grid");
  expect(result.props).toEqual({ direction: undefined });
  expect(result.slots).toEqual({ default: [child] });
});
```

In `packages/pages-component/src/renderer/layout.test.ts`, add:

```typescript
it("applies flex layout when direction is row", () => {
  const component: Component = { type: "metric-grid", props: { direction: "row" } };
  const element = document.createElement("div");
  applyLayoutStyles(element, component);
  expect(element.style.display).toBe("flex");
  expect(element.style.flexWrap).toBe("nowrap");
});

it("applies grid layout when direction is grid", () => {
  const component: Component = { type: "metric-grid", props: { direction: "grid" } };
  const element = document.createElement("div");
  applyLayoutStyles(element, component);
  expect(element.style.display).toBe("grid");
});

it("defaults to grid layout when direction is undefined", () => {
  const component: Component = { type: "metric-grid" };
  const element = document.createElement("div");
  applyLayoutStyles(element, component);
  expect(element.style.display).toBe("grid");
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders.test && yarn workspace @casehubio/pages-component run test -- --run layout.test`
Expected: FAIL

- [ ] **Step 3: Add MetricGridProps and update code**

In `packages/pages-component/src/model/displayer-types.ts`, add after `GridTableProps`:

```typescript
export interface MetricGridProps {
  readonly direction?: "row" | "grid";
}
```

In `packages/pages-component/src/model/type-guards.ts`, add to registry and guard:

```typescript
"metric-grid": MetricGridProps;
```

```typescript
export function isMetricGrid(c: Component): c is TypedComponent<"metric-grid"> {
  return c.type === "metric-grid";
}
```

In `packages/pages-ui/src/dsl/builders.ts`, update `metricGrid()`:

```typescript
export function metricGrid(
  ...args: [...Component[]] | [MetricGridProps, ...Component[]]
): Component {
  const first = args[0];
  const hasOptions = first != null && typeof first === 'object' && 'direction' in first;
  const options = hasOptions ? first as MetricGridProps : undefined;
  const children = (hasOptions ? args.slice(1) : args) as Component[];
  return {
    type: "metric-grid",
    props: { direction: options?.direction },
    slots: { default: freeze(children) },
  };
}
```

In `packages/pages-component/src/renderer/layout.ts`, update metric-grid case:

```typescript
case "metric-grid": {
  const direction = (component.props as { direction?: string } | undefined)?.direction;
  if (direction === "row") {
    element.style.display = "flex";
    element.style.gap = "var(--pages-space-2, 8px)";
    element.style.flexWrap = "nowrap";
    for (const child of element.children) {
      (child as HTMLElement).style.flex = "1 1 0";
    }
  } else {
    element.style.display = "grid";
    element.style.gridTemplateColumns = "repeat(auto-fill, minmax(140px, 1fr))";
    element.style.gap = "var(--pages-space-2, 8px)";
  }
  break;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders.test && yarn workspace @casehubio/pages-component run test -- --run layout.test`
Expected: PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/displayer-types.ts packages/pages-component/src/model/type-guards.ts packages/pages-component/src/renderer/layout.ts packages/pages-component/src/renderer/layout.test.ts packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#317): add metricGrid direction option with MetricGridProps

metricGrid({ direction: 'row' }, ...) renders a horizontal flex strip.
Default remains responsive grid. Adds MetricGridProps interface and
type guard.

Refs #317"
```

### Task 3: PagesMetric sparkline and trend rendering

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts`
- Modify: `packages/pages-viz/src/components/PagesMetric.ts`
- Modify: `packages/pages-viz/src/components/PagesMetric.test.ts`

**Interfaces:**
- Consumes: `renderSparkline()` from `@casehubio/pages-ui-components`, existing `PagesMetric.renderContent()`
- Produces: `sparklineData` and `trend` props on `MetricProps`

- [ ] **Step 1: Write failing tests for sparkline and trend rendering**

In `packages/pages-viz/src/components/PagesMetric.test.ts`, add:

```typescript
it("renders trend arrow when trend prop is set", async () => {
  const props = { lookup: { dataSetId: "test", operations: [] }, title: "P&L", trend: "up" as const };
  const result = await renderMetric(el, props, singleValueDataset);
  const trendEl = el.shadowRoot!.querySelector('.trend');
  expect(trendEl).toBeTruthy();
  expect(trendEl!.textContent!.trim()).toBe("▲");
  expect(trendEl!.classList.contains('trend-up')).toBe(true);
});

it("renders down trend with danger colour class", async () => {
  const props = { lookup: { dataSetId: "test", operations: [] }, title: "Cost", trend: "down" as const };
  await renderMetric(el, props, singleValueDataset);
  const trendEl = el.shadowRoot!.querySelector('.trend');
  expect(trendEl!.textContent!.trim()).toBe("▼");
  expect(trendEl!.classList.contains('trend-down')).toBe(true);
});

it("renders flat trend indicator", async () => {
  const props = { lookup: { dataSetId: "test", operations: [] }, title: "Rate", trend: "flat" as const };
  await renderMetric(el, props, singleValueDataset);
  const trendEl = el.shadowRoot!.querySelector('.trend');
  expect(trendEl!.textContent!.trim()).toBe("—");
});

it("renders sparkline SVG when sparklineData is provided", async () => {
  const props = { lookup: { dataSetId: "test", operations: [] }, title: "P&L", sparklineData: [1.0, 1.5, 1.3, 1.8] };
  await renderMetric(el, props, singleValueDataset);
  const svg = el.shadowRoot!.querySelector('svg.sparkline');
  expect(svg).toBeTruthy();
});

it("does not render sparkline when sparklineData is absent", async () => {
  const props = { lookup: { dataSetId: "test", operations: [] }, title: "P&L" };
  await renderMetric(el, props, singleValueDataset);
  const svg = el.shadowRoot!.querySelector('svg.sparkline');
  expect(svg).toBeNull();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-viz run test -- --run PagesMetric.test`
Expected: FAIL

- [ ] **Step 3: Add sparklineData and trend to MetricProps**

In `packages/pages-component/src/model/displayer-types.ts`, update `MetricProps`:

```typescript
export interface MetricProps extends DataComponentCommon {
  readonly subtype?: "card" | "card2" | "plain-text" | "quota";
  readonly pattern?: string;
  readonly html?: {
    readonly template?: string;
    readonly javascript?: string;
  };
  readonly sparklineData?: readonly number[];
  readonly trend?: "up" | "down" | "flat";
}
```

- [ ] **Step 4: Implement sparkline and trend rendering in PagesMetric**

In `packages/pages-viz/src/components/PagesMetric.ts`:

Add import: `import { renderSparkline } from "@casehubio/pages-ui-components";`

Add CSS styles for trend and sparkline:

```css
.trend { display: inline-block; margin-left: 4px; font-size: 0.7em; }
.trend-up { color: var(--pages-success-9, #16a34a); }
.trend-down { color: var(--pages-danger-9, #dc2626); }
.trend-flat { color: var(--pages-neutral-9, #6b7280); }
.sparkline-container { margin-top: 4px; display: flex; justify-content: center; }
```

In `renderWithValue()` / `renderCard()`, add trend arrow after the value and sparkline below:

```typescript
const trendHtml = props.trend
  ? html`<span class="trend trend-${props.trend}">${props.trend === "up" ? "▲" : props.trend === "down" ? "▼" : "—"}</span>`
  : nothing;

const sparklineHtml = props.sparklineData?.length
  ? html`<div class="sparkline-container">${renderSparkline(props.sparklineData as number[])}</div>`
  : nothing;
```

Insert `trendHtml` after the value span, and `sparklineHtml` after the value section in each subtype's template.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-viz run test -- --run PagesMetric.test`
Expected: PASS

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Add example — update Metrics and Gauges sample**

Update `examples/samples/Metrics/Metric Patterns.dash.yaml` to demonstrate trend and sparkline. Since sparklineData and trend are props, they work in YAML:

```yaml
# Add a new section to existing file showing trend indicators
- type: metric-grid
  properties:
    direction: row
  components:
    - type: metric
      properties:
        title: Revenue
        trend: up
        lookup:
          uuid: revenue
    - type: metric
      properties:
        title: Costs
        trend: down
        lookup:
          uuid: costs
    - type: metric
      properties:
        title: Margin
        trend: flat
        lookup:
          uuid: margin
```

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/displayer-types.ts packages/pages-viz/src/components/PagesMetric.ts packages/pages-viz/src/components/PagesMetric.test.ts examples/samples/Metrics/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#317): add sparkline and trend rendering to PagesMetric

MetricProps gains sparklineData (number[]) and trend (up/down/flat).
PagesMetric renders sparkline SVG below value and trend arrow beside it.
Updates Metric Patterns example with row-direction metricGrid demo.

Refs #317"
```

## Batch 3: Event timeline component

### Task 4: EventTimelineProps, types, type guards, and activation

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts`
- Modify: `packages/pages-component/src/model/type-guards.ts`
- Modify: `packages/pages-runtime/src/activation.ts`
- Create: `packages/pages-viz/src/components/event-timeline-types.ts`

**Interfaces:**
- Consumes: `DataComponentCommon` from `displayer-types.ts`
- Produces: `EventTimelineProps`, `EventTimelineLayout`, `EventTimelineNode`, `EventTimelineStrategy`, `EventNodeStatus`, `isEventTimeline()` type guard

- [ ] **Step 1: Add EventTimelineProps to displayer-types.ts**

```typescript
export type EventTimelineLayout = "vertical" | "horizontal" | "compact";

export interface EventTimelineProps extends DataComponentCommon {
  readonly layout?: EventTimelineLayout;
  readonly pageSize?: number;
  readonly strategyKey?: string;
}
```

- [ ] **Step 2: Add type guard and registry entry**

In `type-guards.ts`, add to registry:

```typescript
"event-timeline": EventTimelineProps;
```

Add guard:

```typescript
export function isEventTimeline(c: Component): c is TypedComponent<"event-timeline"> {
  return c.type === "event-timeline";
}
```

- [ ] **Step 3: Add to DATA_COMPONENT_TYPES in activation.ts**

Add `"event-timeline"` to the `DATA_COMPONENT_TYPES` array.

- [ ] **Step 4: Create event-timeline-types.ts**

Create `packages/pages-viz/src/components/event-timeline-types.ts`:

```typescript
import type { EventTimelineLayout } from "@casehubio/pages-component";

export type EventNodeStatus = "completed" | "active" | "pending" | "failed" | "skipped";

export interface EventTimelineNode {
  readonly key: string;
  readonly label: string;
  readonly status: EventNodeStatus;
  readonly timestamp?: string;
  readonly actor?: string;
  readonly detail?: unknown;
  readonly category?: string;
}

export interface EventTimelineStrategy<T = unknown> {
  toNodes(data: T): EventTimelineNode[];
  transformData?: (raw: unknown) => T;
  defaultLayout: EventTimelineLayout;
  renderNode?: (node: EventTimelineNode) => unknown;
  renderDetail?: (node: EventTimelineNode) => unknown;
  filterCategories?: string[];
}
```

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/displayer-types.ts packages/pages-component/src/model/type-guards.ts packages/pages-runtime/src/activation.ts packages/pages-viz/src/components/event-timeline-types.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#317): add EventTimeline types, type guard, and activation

EventTimelineProps in displayer-types, EventTimelineNode and
EventTimelineStrategy in pages-viz (rendering-adjacent types).
Registry entry, type guard, and activation registration.

Refs #317"
```

### Task 5: PagesEventTimeline component implementation

**Files:**
- Create: `packages/pages-viz/src/components/PagesEventTimeline.ts`
- Create: `packages/pages-viz/src/components/PagesEventTimeline.test.ts`
- Modify: `packages/pages-viz/src/index.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`

**Interfaces:**
- Consumes: `PagesElement` base class, `EventTimelineProps`, `EventTimelineNode`, `EventTimelineStrategy`, `emitPagesEvent`
- Produces: `PagesEventTimeline` web component (`pages-event-timeline`), `PagesEventTimeline.registerStrategy()` static method

- [ ] **Step 1: Write failing tests**

Create `packages/pages-viz/src/components/PagesEventTimeline.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import type { EventTimelineNode, EventTimelineStrategy } from "./event-timeline-types.js";

// Test the component renders nodes in vertical layout
// Test strategy.toNodes is called with data
// Test node expansion toggles
// Test filter bar filters by category
// Test keyboard navigation (ArrowDown/ArrowUp)
// Test strategyKey resolves from registry
// Test status colours map correctly
// Test click emits pages-event with topic event-timeline:node-selected

describe("PagesEventTimeline", () => {
  const testNodes: EventTimelineNode[] = [
    { key: "1", label: "Started", status: "completed", timestamp: "2026-01-01T00:00:00Z", category: "lifecycle" },
    { key: "2", label: "Processing", status: "active", timestamp: "2026-01-01T01:00:00Z", category: "task" },
    { key: "3", label: "Pending Review", status: "pending", category: "task" },
  ];

  const testStrategy: EventTimelineStrategy<EventTimelineNode[]> = {
    toNodes: (data) => data,
    defaultLayout: "vertical",
    filterCategories: ["lifecycle", "task"],
  };

  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement("pages-event-timeline");
    document.body.appendChild(el);
  });

  it("renders nodes from strategy", async () => {
    (el as any).strategy = testStrategy;
    (el as any).data = testNodes;
    await (el as any).updateComplete;
    const nodes = el.shadowRoot!.querySelectorAll(".timeline-node");
    expect(nodes.length).toBe(3);
  });

  it("applies status class to nodes", async () => {
    (el as any).strategy = testStrategy;
    (el as any).data = testNodes;
    await (el as any).updateComplete;
    const firstNode = el.shadowRoot!.querySelector(".timeline-node");
    expect(firstNode!.classList.contains("status-completed")).toBe(true);
  });

  it("toggles node expansion on click", async () => {
    (el as any).strategy = testStrategy;
    (el as any).data = testNodes;
    await (el as any).updateComplete;
    const firstNode = el.shadowRoot!.querySelector(".timeline-node") as HTMLElement;
    firstNode.click();
    await (el as any).updateComplete;
    expect(firstNode.classList.contains("expanded")).toBe(true);
  });

  it("filters nodes by category", async () => {
    (el as any).strategy = testStrategy;
    (el as any).data = testNodes;
    (el as any).activeFilters = new Set(["lifecycle"]);
    await (el as any).updateComplete;
    const nodes = el.shadowRoot!.querySelectorAll(".timeline-node");
    expect(nodes.length).toBe(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-viz run test -- --run PagesEventTimeline.test`
Expected: FAIL — component not defined

- [ ] **Step 3: Implement PagesEventTimeline**

Create `packages/pages-viz/src/components/PagesEventTimeline.ts`. The component:

- Extends `PagesElement<EventTimelineProps>`
- Static strategy registry: `private static _strategyRegistry = new Map<string, EventTimelineStrategy>()`
- `static registerStrategy(key: string, strategy: EventTimelineStrategy): void`
- `@property({ attribute: false }) strategy?: EventTimelineStrategy`
- `@property({ attribute: false }) data?: unknown`
- `@property({ attribute: false }) activeFilters?: Set<string> | string[]`
- `@state() private _nodes: EventTimelineNode[] = []`
- `@state() private _expandedKeys = new Set<string>()`
- Resolves strategy from `strategyKey` prop via registry, or from direct `strategy` property
- Built-in default strategy `"chronological"`: sorts nodes by timestamp, vertical layout
- `willUpdate()` calls `strategy.toNodes()` when data changes
- Vertical renderer: timeline line with status-coloured dots, labels, timestamps
- Node click toggles expansion, dispatches `emitPagesEvent(this, "event-timeline:node-selected", { node, index })`
- Filter bar with category chip buttons
- Keyboard: ArrowDown/ArrowUp moves focus between nodes
- CSS uses `--pages-*` tokens, status colours as specified in spec

Reference blocks-timeline renderers for the HTML/CSS structure — port the vertical renderer first, then horizontal and compact.

- [ ] **Step 4: Export and register**

In `packages/pages-viz/src/index.ts`, add:
```typescript
export { PagesEventTimeline } from "./components/PagesEventTimeline.js";
export type { EventTimelineNode, EventTimelineStrategy, EventNodeStatus } from "./components/event-timeline-types.js";
```

In `packages/pages-viz/src/custom-elements.ts`, add:
```typescript
import type { PagesEventTimeline } from "./components/PagesEventTimeline.js";
// In HTMLElementTagNameMap:
"pages-event-timeline": PagesEventTimeline;
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-viz run test -- --run PagesEventTimeline.test`
Expected: PASS

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Add eventTimeline() builder**

In `packages/pages-ui/src/dsl/builders.ts`, add import and builder:

```typescript
import { EventTimelineProps } from "@casehubio/pages-component";

export function eventTimeline(props: EventTimelineProps): TypedComponent<"event-timeline"> {
  return freeze({ type: "event-timeline" as const, props: { ...props } });
}
```

Add test in `builders.test.ts`:

```typescript
describe("eventTimeline()", () => {
  it("creates event-timeline component with spread props", () => {
    const props = { lookup: { dataSetId: "events", operations: [] }, layout: "vertical" };
    const result = eventTimeline(props as any);
    expect(result.type).toBe("event-timeline");
    expect(result.props).toEqual(props);
    expect(result.props).not.toBe(props);
  });

  it("freezes returned component", () => {
    const result = eventTimeline({ lookup: { dataSetId: "t", operations: [] } } as any);
    expect(Object.isFrozen(result)).toBe(true);
  });
});
```

- [ ] **Step 8: Run all tests**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders.test && yarn workspace @casehubio/pages-viz run test -- --run PagesEventTimeline.test`
Expected: PASS

- [ ] **Step 9: Add example**

Create `examples/samples/Charts/Event Timeline.dash.yaml` with inline event data demonstrating vertical layout, status colours, and categories:

```yaml
datasets:
  - uuid: events
    content: >-
      [
        ["1", "Case Started", "completed", "2026-01-15T09:00:00Z", "System", "lifecycle"],
        ["2", "Task Created", "completed", "2026-01-15T09:05:00Z", "John", "task"],
        ["3", "Processing", "active", "2026-01-15T09:30:00Z", "Jane", "task"],
        ["4", "Review Pending", "pending", "", "", "task"],
        ["5", "Approval Gate", "pending", "", "", "gate"]
      ]
    columns:
      - id: key
        type: LABEL
      - id: label
        type: LABEL
      - id: status
        type: LABEL
      - id: timestamp
        type: TEXT
      - id: actor
        type: LABEL
      - id: category
        type: LABEL

pages:
  - components:
      - type: title
        properties:
          text: Event Timeline
          size: h3
      - type: event-timeline
        properties:
          title: Case Activity
          layout: vertical
          lookup:
            uuid: events
```

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/components/event-timeline-types.ts packages/pages-viz/src/components/PagesEventTimeline.ts packages/pages-viz/src/components/PagesEventTimeline.test.ts packages/pages-viz/src/index.ts packages/pages-viz/src/custom-elements.ts packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts examples/samples/Charts/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#317): add PagesEventTimeline component and eventTimeline() builder

Strategy-pattern event timeline with vertical/horizontal/compact
renderers, node expansion, category filtering, keyboard navigation.
Static strategy registry for DSL-configurable strategies.
Includes Event Timeline example in gallery.

Refs #317"
```

## Batch 4: Master-detail builder and declarative selection

### Task 6: Add selection to DataTableProps and wire in activation

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts`
- Modify: `packages/pages-runtime/src/activation.ts`

**Interfaces:**
- Consumes: `SelectionMode` type from `displayer-types.ts`, `PagesDataTable.selection` property
- Produces: declarative `selection` prop on `DataTableProps`, activation wiring

- [ ] **Step 1: Add selection to DataTableProps**

In `packages/pages-component/src/model/displayer-types.ts`, update:

```typescript
export interface DataTableProps extends DataComponentCommon {
  readonly pageSize?: number;
  readonly sortable?: boolean;
  readonly resizable?: boolean;
  readonly rowStyle?: readonly RowStyleRule[];
  readonly expandable?: ExpandableConfig;
  readonly selection?: SelectionMode;
}
```

- [ ] **Step 2: Wire selection in activation callback**

In `packages/pages-runtime/src/activation.ts`, find where data-table props are applied to the element. Add selection wiring:

```typescript
if (props.selection) {
  (element as any).selection = props.selection;
}
```

This goes in the same activation path that already handles `pageSize`, `sortable`, `rowStyle`, etc.

- [ ] **Step 3: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/displayer-types.ts packages/pages-runtime/src/activation.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#317): add declarative selection prop to DataTableProps

DataTableProps gains selection?: SelectionMode. Activation callback
wires it to the PagesDataTable element. Prerequisite for masterDetail()
builder.

Refs #317"
```

### Task 7: masterDetail() convenience builder

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts`
- Modify: `packages/pages-ui/src/dsl/builders.test.ts`

**Interfaces:**
- Consumes: `split()` builder, `TypedComponent<"data-table">`, `TypedComponent<"host-panel">`, `DataSetLookup`
- Produces: `masterDetail()` builder function

- [ ] **Step 1: Write failing tests**

In `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
describe("masterDetail()", () => {
  it("creates a split with wired selection and selectionSource", () => {
    const master = dataTable({ lookup: { dataSetId: "strategies", operations: [] } });
    const detail = hostPanel("strategy-detail");
    const result = masterDetail({ master, detail });

    expect(result.type).toBe("split");
    // Master gets selection: "single"
    const masterSlot = result.slots!["0"]![0]!;
    expect(masterSlot.props.selection).toBe("single");
    // Detail gets selectionSource wired
    const detailSlot = result.slots!["1"]![0]!;
    expect(detailSlot.props.selectionSource).toBe("strategies");
  });

  it("respects custom direction and ratio", () => {
    const master = dataTable({ lookup: { dataSetId: "s", operations: [] } });
    const detail = hostPanel("d");
    const result = masterDetail({ master, detail, direction: "vertical", ratio: [30, 70] });

    expect(result.props.direction).toBe("vertical");
  });

  it("defaults to horizontal direction and 40:60 ratio", () => {
    const master = dataTable({ lookup: { dataSetId: "s", operations: [] } });
    const detail = hostPanel("d");
    const result = masterDetail({ master, detail });

    expect(result.props.direction).toBe("horizontal");
  });

  it("freezes returned component", () => {
    const master = dataTable({ lookup: { dataSetId: "s", operations: [] } });
    const detail = hostPanel("d");
    const result = masterDetail({ master, detail });
    expect(Object.isFrozen(result)).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders.test`
Expected: FAIL

- [ ] **Step 3: Implement masterDetail()**

In `packages/pages-ui/src/dsl/builders.ts`:

```typescript
export function masterDetail(config: {
  master: TypedComponent<"data-table">;
  detail: TypedComponent<"host-panel">;
  direction?: "horizontal" | "vertical";
  ratio?: [number, number];
}): TypedComponent<"split"> {
  const { master, detail, direction = "horizontal", ratio = [40, 60] } = config;

  const wiredMaster = freeze({
    ...master,
    props: { ...master.props, selection: "single" as const },
  });

  const masterLookup = master.props.lookup;
  const wiredDetail = freeze({
    ...detail,
    props: {
      ...detail.props,
      selectionSource: masterLookup.dataSetId,
    },
  });

  return split(direction, [wiredMaster, wiredDetail], { ratio });
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders.test`
Expected: PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Update Master Detail example**

The existing `examples/samples/Tables/Master Detail.ts` sets selection imperatively. Now that selection is declarative, add a YAML-only variant or update the existing `.dash.yaml` to demonstrate `masterDetail()` usage. Since `masterDetail()` is a TypeScript DSL builder (not YAML), create a new `.ts` example alongside the existing one:

Update `examples/samples/Tables/Master Detail.ts` to use declarative selection instead of imperative:

```typescript
// This file is now a no-op — selection is declared in YAML via the
// masterDetail pattern. Kept for backward compat with existing sample.
```

Or add a note in the existing YAML showing the selection property is now declarative.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts examples/samples/Tables/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#317): add masterDetail() convenience builder

Composes split() + dataTable(selection: single) + hostPanel(selectionSource)
with type-safe enforcement of host-panel detail pane. Defaults to
horizontal 40:60 split. Updates Master Detail example.

Closes #317"
```

## References

- [2026-08-17-dsl-composition-gaps-design.md] — design spec this plan implements
- `packages/pages-component/src/model/displayer-types.ts` — all prop type definitions
- `packages/pages-component/src/model/type-guards.ts` — component type registry
- `packages/pages-runtime/src/activation.ts` — DATA_COMPONENT_TYPES and prop wiring
- `packages/pages-ui/src/dsl/builders.ts` — existing builder functions
- `packages/pages-component/src/renderer/layout.ts:60` — metric-grid CSS layout
- `packages/pages-viz/src/components/PagesMetric.ts` — metric component
- `packages/pages-viz/src/charts/PagesTimeline.ts` — existing ECharts Gantt timeline
- `packages/pages-runtime/src/selection-forwarding.ts` — selection event forwarding
- `packages/pages-ui-components/src/sparkline/render-sparkline.ts` — sparkline renderer
- `blocks-ui/components/blocks-timeline/src/blocks-timeline.ts` — reference for event timeline renderers
- blocks-ui#122 — re-export cleanup
- blocks-ui#123 — blocks-timeline refactor
- casehubio/casehub-pages#317 — focal epic
