# @casehub/viz Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `@casehub/viz` — 13 Web Component visualization wrappers per the design spec at `docs/superpowers/specs/2026-06-14-casehub-viz-design.md`.

**Architecture:** Three-level class hierarchy (`CasehubElement<P>` → `CasehubChartElement<P>` → concrete component). Connected components dispatch `casehub-data-request` events upward; data flows down via properties. ECharts 6.x for chart rendering, custom HTML for table/metric/selector/iframe. Four-stage option pipeline: dataset → subtype → typed settings → deep merge `extra`.

**Tech Stack:** TypeScript 5.6+, Apache ECharts 6.x (tree-shaken), Vitest, Web Components (Custom Elements v1, Shadow DOM)

**Spec:** `docs/superpowers/specs/2026-06-14-casehub-viz-design.md`

**Test runner:** `/opt/homebrew/bin/node /Users/mdproctor/claude/melviz/.yarn/releases/yarn-4.10.3.cjs --cwd /Users/mdproctor/claude/melviz workspace @casehub/viz run test -- --run`

---

## Pre-requisite: Fix MapProps in @casehub/ui

Before any viz work, fix `displayer-types.ts` in `@casehub/ui` per spec preamble.

### Task 0: Fix MapProps

**Files:**
- Modify: `packages/casehub-ui/src/model/displayer-types.ts`
- Modify: `packages/casehub-ui/src/model/displayer-types.test.ts` (if MapProps tests exist)

- [ ] **Step 1: Update MapProps to extend ChartSettings and add mapName**

In `packages/casehub-ui/src/model/displayer-types.ts`, change:

```typescript
export interface MapProps extends DataComponentCommon {
  readonly subtype?: "regions" | "markers";
  readonly colorScheme?: string;
}
```

To:

```typescript
export interface MapProps extends DataComponentCommon, ChartSettings {
  readonly subtype?: "regions" | "markers";
  readonly colorScheme?: string;
  readonly mapName?: string;
}
```

- [ ] **Step 2: Update type guard if needed**

Check `packages/casehub-ui/src/model/type-guards.ts` — `isMap()` should still work since it checks `type === "map"`, not props shape. Verify.

- [ ] **Step 3: Run @casehub/ui tests**

Run: `workspace @casehub/ui run test -- --run`
Expected: All 385 tests pass.

- [ ] **Step 4: Commit**

```
git add packages/casehub-ui/src/model/displayer-types.ts
git commit -m "fix: MapProps extends ChartSettings, add mapName property

Refs #11"
```

---

## Task 1: Package scaffold

**Files:**
- Create: `packages/casehub-viz/package.json`
- Create: `packages/casehub-viz/tsconfig.json`
- Create: `packages/casehub-viz/vitest.config.ts`
- Create: `packages/casehub-viz/src/index.ts` (placeholder)

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehub/viz",
  "version": "0.0.1",
  "description": "CaseHub Viz — Web Component visualization wrappers",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehub/data": "workspace:*",
    "@casehub/ui": "workspace:*",
    "echarts": "^5.6.0"
  },
  "devDependencies": {
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

Note: Using ECharts 5.6.x (already in the monorepo) rather than 6.x — upgrade to 6.x is a separate task once it's published on npm. The API is compatible.

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM"],
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "esModuleInterop": false,
    "isolatedModules": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

Note: includes `"DOM"` in lib (unlike `@casehub/ui`) — viz components use `HTMLElement`, `ShadowRoot`, `CustomEvent`, `ResizeObserver`.

- [ ] **Step 3: Create vitest.config.ts**

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "jsdom",
    include: ["src/**/*.test.ts"],
    coverage: {
      provider: "v8",
      include: ["src/**/*.ts"],
      exclude: ["src/**/*.test.ts", "src/index.ts"],
    },
  },
});
```

Note: uses `environment: "jsdom"` (not `"node"`) — Web Components need DOM APIs.

- [ ] **Step 4: Create placeholder index.ts**

```typescript
// @casehub/viz — Web Component visualization wrappers
// Components are registered via customElements.define() at import time.
```

- [ ] **Step 5: Install dependencies**

Run: `yarn install` from the monorepo root.
Expected: Workspace resolves, node_modules linked.

- [ ] **Step 6: Verify test runner works**

Run: `workspace @casehub/viz run test -- --run`
Expected: 0 tests, passes cleanly.

- [ ] **Step 7: Commit**

```
git add packages/casehub-viz/
git commit -m "feat(viz): scaffold @casehub/viz package

Refs #11"
```

---

## Task 2: Utilities — deep-merge and cell-extract

**Files:**
- Create: `packages/casehub-viz/src/base/deep-merge.ts`
- Create: `packages/casehub-viz/src/base/deep-merge.test.ts`
- Create: `packages/casehub-viz/src/base/cell-extract.ts`
- Create: `packages/casehub-viz/src/base/cell-extract.test.ts`

### deep-merge

- [ ] **Step 1: Write deep-merge tests**

```typescript
import { describe, it, expect } from "vitest";
import { deepMerge } from "./deep-merge.js";

describe("deepMerge", () => {
  it("merges flat objects", () => {
    expect(deepMerge({ a: 1 }, { b: 2 })).toEqual({ a: 1, b: 2 });
  });

  it("deep-merges nested objects", () => {
    expect(deepMerge(
      { xAxis: { type: "category", name: "X" } },
      { xAxis: { name: "Revenue" } },
    )).toEqual({ xAxis: { type: "category", name: "Revenue" } });
  });

  it("replaces arrays entirely", () => {
    expect(deepMerge(
      { series: [{ type: "bar" }, { type: "line" }] },
      { series: [{ type: "scatter" }] },
    )).toEqual({ series: [{ type: "scatter" }] });
  });

  it("replaces primitives", () => {
    expect(deepMerge({ a: 1 }, { a: 2 })).toEqual({ a: 2 });
  });

  it("override wins over base for type conflicts", () => {
    expect(deepMerge({ a: { b: 1 } }, { a: 42 })).toEqual({ a: 42 });
  });

  it("handles undefined override values", () => {
    expect(deepMerge({ a: 1, b: 2 }, { b: undefined })).toEqual({ a: 1, b: undefined });
  });

  it("returns base when override is empty", () => {
    expect(deepMerge({ a: 1 }, {})).toEqual({ a: 1 });
  });

  it("handles null values", () => {
    expect(deepMerge({ a: 1 }, { a: null })).toEqual({ a: null });
  });
});
```

- [ ] **Step 2: Run test, verify RED**

Run: `workspace @casehub/viz run test -- --run src/base/deep-merge.test.ts`
Expected: FAIL — `deepMerge` not found.

- [ ] **Step 3: Implement deep-merge**

```typescript
export function deepMerge<T extends Record<string, unknown>>(
  base: T,
  override: Record<string, unknown>,
): T {
  const result = { ...base } as Record<string, unknown>;

  for (const key of Object.keys(override)) {
    const baseVal = result[key];
    const overrideVal = override[key];

    if (
      overrideVal !== null &&
      overrideVal !== undefined &&
      typeof overrideVal === "object" &&
      !Array.isArray(overrideVal) &&
      baseVal !== null &&
      baseVal !== undefined &&
      typeof baseVal === "object" &&
      !Array.isArray(baseVal)
    ) {
      result[key] = deepMerge(
        baseVal as Record<string, unknown>,
        overrideVal as Record<string, unknown>,
      );
    } else {
      result[key] = overrideVal;
    }
  }

  return result as T;
}
```

- [ ] **Step 4: Run test, verify GREEN**

Run: `workspace @casehub/viz run test -- --run src/base/deep-merge.test.ts`
Expected: All tests pass.

### cell-extract

- [ ] **Step 5: Write cell-extract tests**

```typescript
import { describe, it, expect } from "vitest";
import { cellToRaw, resolveColumnName } from "./cell-extract.js";
import { ColumnType } from "@casehub/data/dist/dataset/types.js";
import type { Column, ColumnId, ColumnSettings } from "@casehub/data/dist/dataset/types.js";

describe("cellToRaw", () => {
  it("extracts number value", () => {
    expect(cellToRaw({ type: ColumnType.NUMBER, value: 42 })).toBe(42);
  });

  it("extracts string value from LABEL", () => {
    expect(cellToRaw({ type: ColumnType.LABEL, value: "hello" })).toBe("hello");
  });

  it("extracts string value from TEXT", () => {
    expect(cellToRaw({ type: ColumnType.TEXT, value: "text" })).toBe("text");
  });

  it("extracts Date value", () => {
    const d = new Date("2024-01-01");
    expect(cellToRaw({ type: ColumnType.DATE, value: d })).toBe(d);
  });

  it("returns null for NULL cell", () => {
    expect(cellToRaw({ type: "NULL" })).toBeNull();
  });
});

describe("resolveColumnName", () => {
  const col: Column = {
    id: "revenue" as ColumnId,
    name: "revenue",
    type: ColumnType.NUMBER,
  };

  it("returns column.name when no overrides", () => {
    expect(resolveColumnName(col)).toBe("revenue");
  });

  it("returns override name from propsColumns", () => {
    const overrides: ColumnSettings[] = [
      { id: "revenue" as ColumnId, name: "Total Revenue" },
    ];
    expect(resolveColumnName(col, overrides)).toBe("Total Revenue");
  });

  it("returns column.settings.name when no propsColumns match", () => {
    const colWithSettings: Column = {
      ...col,
      settings: { id: "revenue" as ColumnId, name: "Rev" },
    };
    expect(resolveColumnName(colWithSettings)).toBe("Rev");
  });

  it("propsColumns takes priority over settings.name", () => {
    const colWithSettings: Column = {
      ...col,
      settings: { id: "revenue" as ColumnId, name: "Rev" },
    };
    const overrides: ColumnSettings[] = [
      { id: "revenue" as ColumnId, name: "Override" },
    ];
    expect(resolveColumnName(colWithSettings, overrides)).toBe("Override");
  });

  it("ignores propsColumns with non-matching id", () => {
    const overrides: ColumnSettings[] = [
      { id: "other" as ColumnId, name: "Other" },
    ];
    expect(resolveColumnName(col, overrides)).toBe("revenue");
  });
});
```

- [ ] **Step 6: Run test, verify RED**

Run: `workspace @casehub/viz run test -- --run src/base/cell-extract.test.ts`
Expected: FAIL — imports not found.

- [ ] **Step 7: Implement cell-extract**

```typescript
import type { CellValue, Column, ColumnId, ColumnSettings } from "@casehub/data/dist/dataset/types.js";

export function cellToRaw(cell: CellValue): string | number | Date | null {
  if (cell.type === "NULL") return null;
  return cell.value;
}

export function resolveColumnName(
  column: Column,
  propsColumns?: readonly ColumnSettings[],
): string {
  const override = propsColumns?.find((c) => c.id === column.id);
  return override?.name ?? column.settings?.name ?? column.name;
}
```

- [ ] **Step 8: Run test, verify GREEN**

Run: `workspace @casehub/viz run test -- --run src/base/cell-extract.test.ts`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```
git add packages/casehub-viz/src/base/
git commit -m "feat(viz): deep-merge and cell-extract utilities

Refs #11"
```

---

## Task 3: VizComponentProps and CasehubElement base class

**Files:**
- Create: `packages/casehub-viz/src/base/types.ts`
- Create: `packages/casehub-viz/src/base/CasehubElement.ts`
- Create: `packages/casehub-viz/src/base/CasehubElement.test.ts`

- [ ] **Step 1: Create types.ts with VizComponentProps**

```typescript
import type { DataSetLookup } from "@casehub/data/dist/dataset/lookup.js";
import type { ColumnSettings } from "@casehub/data/dist/dataset/types.js";
import type { FilterSettings, RefreshSettings } from "@casehub/ui/dist/model/component-props.js";

export interface VizComponentProps {
  readonly lookup?: DataSetLookup;
  readonly filter?: FilterSettings;
  readonly refresh?: RefreshSettings;
  readonly columns?: readonly ColumnSettings[];
}
```

- [ ] **Step 2: Write CasehubElement lifecycle tests**

Full test file covering: connectedCallback dispatches data-request (props before DOM, DOM before props), lookup change triggers re-request, disconnect/reconnect resets _dataRequested, error property clears dataset, update guards (no props → loading, no dataset → loading, error → error display).

Tests use a concrete test subclass:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { CasehubElement } from "./CasehubElement.js";
import type { VizComponentProps } from "./types.js";
import type { TypedDataSet } from "@casehub/data/dist/dataset/types.js";
import type { DataSetLookup } from "@casehub/data/dist/dataset/lookup.js";

interface TestProps extends VizComponentProps {
  readonly label?: string;
}

class TestElement extends CasehubElement<TestProps> {
  renderCalls: Array<{ props: TestProps; dataset: TypedDataSet }> = [];

  protected override render(
    _container: HTMLDivElement,
    props: TestProps,
    dataset: TypedDataSet,
  ): void {
    this.renderCalls.push({ props, dataset });
  }
}

customElements.define("test-element", TestElement);

function mockLookup(id: string): DataSetLookup {
  return { dataSetId: id, operations: [] } as unknown as DataSetLookup;
}

function mockDataSet(): TypedDataSet {
  return { columns: [], rows: [] } as unknown as TypedDataSet;
}

describe("CasehubElement", () => {
  let el: TestElement;

  beforeEach(() => {
    el = document.createElement("test-element") as TestElement;
  });

  afterEach(() => {
    el.remove();
  });

  describe("data request lifecycle", () => {
    it("dispatches casehub-data-request when props set before DOM insertion", () => {
      const events: CustomEvent[] = [];
      document.body.addEventListener("casehub-data-request", (e) =>
        events.push(e as CustomEvent),
      );

      el.props = { lookup: mockLookup("ds1") };
      document.body.appendChild(el);

      expect(events).toHaveLength(1);
      expect(events[0]!.detail.lookup.dataSetId).toBe("ds1");

      document.body.removeEventListener("casehub-data-request", () => {});
    });

    it("dispatches casehub-data-request when DOM inserted before props", () => {
      const events: CustomEvent[] = [];
      document.body.addEventListener("casehub-data-request", (e) =>
        events.push(e as CustomEvent),
      );

      document.body.appendChild(el);
      el.props = { lookup: mockLookup("ds2") };

      expect(events).toHaveLength(1);
      expect(events[0]!.detail.lookup.dataSetId).toBe("ds2");

      document.body.removeEventListener("casehub-data-request", () => {});
    });

    it("does not dispatch duplicate requests for same lookup", () => {
      const events: CustomEvent[] = [];
      document.body.addEventListener("casehub-data-request", (e) =>
        events.push(e as CustomEvent),
      );

      const lookup = mockLookup("ds1");
      el.props = { lookup };
      document.body.appendChild(el);
      el.props = { lookup }; // same reference

      expect(events).toHaveLength(1);

      document.body.removeEventListener("casehub-data-request", () => {});
    });

    it("dispatches new request when lookup changes", () => {
      const events: CustomEvent[] = [];
      document.body.addEventListener("casehub-data-request", (e) =>
        events.push(e as CustomEvent),
      );

      el.props = { lookup: mockLookup("ds1") };
      document.body.appendChild(el);
      el.props = { lookup: mockLookup("ds2") };

      expect(events).toHaveLength(2);
      expect(events[1]!.detail.lookup.dataSetId).toBe("ds2");

      document.body.removeEventListener("casehub-data-request", () => {});
    });

    it("re-requests on disconnect/reconnect", () => {
      const events: CustomEvent[] = [];
      document.body.addEventListener("casehub-data-request", (e) =>
        events.push(e as CustomEvent),
      );

      el.props = { lookup: mockLookup("ds1") };
      document.body.appendChild(el);
      expect(events).toHaveLength(1);

      el.remove();
      document.body.appendChild(el);
      expect(events).toHaveLength(2);

      document.body.removeEventListener("casehub-data-request", () => {});
    });
  });

  describe("update guards", () => {
    it("does not render without props", () => {
      document.body.appendChild(el);
      el.dataset = mockDataSet();
      expect(el.renderCalls).toHaveLength(0);
    });

    it("does not render without dataset", () => {
      document.body.appendChild(el);
      el.props = { label: "test" };
      expect(el.renderCalls).toHaveLength(0);
    });

    it("renders when both props and dataset are set", () => {
      document.body.appendChild(el);
      el.props = { label: "test" };
      el.dataset = mockDataSet();
      expect(el.renderCalls).toHaveLength(1);
    });

    it("clears dataset on lookup change", () => {
      document.body.appendChild(el);
      el.props = { lookup: mockLookup("ds1") };
      el.dataset = mockDataSet();
      expect(el.renderCalls).toHaveLength(1);

      el.props = { lookup: mockLookup("ds2") };
      expect(el.dataset).toBeUndefined();
    });
  });

  describe("error handling", () => {
    it("setting error clears dataset", () => {
      document.body.appendChild(el);
      el.props = { label: "test" };
      el.dataset = mockDataSet();
      el.error = "Something failed";

      expect(el.dataset).toBeUndefined();
    });

    it("setting dataset clears error", () => {
      document.body.appendChild(el);
      el.props = { label: "test" };
      el.error = "Something failed";
      el.dataset = mockDataSet();

      expect(el.error).toBeUndefined();
    });
  });

  describe("standalone usage", () => {
    it("renders without lookup (no data request event)", () => {
      const events: CustomEvent[] = [];
      document.body.addEventListener("casehub-data-request", (e) =>
        events.push(e as CustomEvent),
      );

      document.body.appendChild(el);
      el.props = { label: "standalone" };
      el.dataset = mockDataSet();

      expect(events).toHaveLength(0);
      expect(el.renderCalls).toHaveLength(1);

      document.body.removeEventListener("casehub-data-request", () => {});
    });
  });
});
```

- [ ] **Step 3: Run tests, verify RED**

Run: `workspace @casehub/viz run test -- --run src/base/CasehubElement.test.ts`
Expected: FAIL — `CasehubElement` not found.

- [ ] **Step 4: Implement CasehubElement**

Full implementation per spec §2, including: Shadow DOM, typed properties with getters/setters, `requestDataIfNeeded()` with lookup change detection, `connectedCallback`/`disconnectedCallback`, refresh timer, resize observer, `update()` guards, loading/error renderers.

- [ ] **Step 5: Run tests, verify GREEN**

Run: `workspace @casehub/viz run test -- --run src/base/CasehubElement.test.ts`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```
git add packages/casehub-viz/src/base/
git commit -m "feat(viz): CasehubElement base class — lifecycle, events, properties

Refs #11"
```

---

## Task 4: CasehubChartElement — ECharts intermediate class

**Files:**
- Create: `packages/casehub-viz/src/base/CasehubChartElement.ts`
- Create: `packages/casehub-viz/src/base/CasehubChartElement.test.ts`

- [ ] **Step 1: Write CasehubChartElement tests**

Tests covering: ECharts init/dispose, theme change (dispose + re-init), resize forwarding, click-to-filter event with correct detail, click handler uses current dataset not stale closure, buildOption called with typed props.

Mock `echarts` module — no canvas rendering needed. Assert `init()`, `setOption()`, `dispose()`, `on()`, `resize()` calls.

- [ ] **Step 2: Run tests, verify RED**

- [ ] **Step 3: Implement CasehubChartElement**

Per spec §2: extends `CasehubElement<P extends DataComponentCommon & ChartSettings>`, manages ECharts instance, registers click handler once (reads `this.dataset` at call time), overrides `onResize()` for `chart.resize()`, disposes in `disconnectedCallback`.

- [ ] **Step 4: Run tests, verify GREEN**

- [ ] **Step 5: Commit**

```
git commit -m "feat(viz): CasehubChartElement — ECharts lifecycle, click-to-filter

Refs #11"
```

---

## Task 5: Shared option pipeline — datasetToSource and applyChartSettings

**Files:**
- Create: `packages/casehub-viz/src/charts/option-pipeline.ts`
- Create: `packages/casehub-viz/src/charts/option-pipeline.test.ts`

These are shared Stage 1 and Stage 3 functions used by all chart `buildOption()` implementations.

- [ ] **Step 1: Write option-pipeline tests**

Tests covering:
- `datasetToSource()` — converts TypedDataSet to ECharts source array with display name resolution, null handling, type preservation (numbers stay numbers, dates stay dates)
- `applyChartSettings()` — maps typed props to ECharts option: title, legend (all 4 positions), axes, margins, zoom, skip for non-Cartesian

- [ ] **Step 2: Run tests, verify RED**

- [ ] **Step 3: Implement option-pipeline**

`datasetToSource(dataset, propsColumns?)` — Stage 1 from spec §4.
`applyChartSettings(option, props)` — Stage 3 from spec §4.

- [ ] **Step 4: Run tests, verify GREEN**

- [ ] **Step 5: Commit**

```
git commit -m "feat(viz): shared option pipeline — datasetToSource, applyChartSettings

Refs #11"
```

---

## Task 6: Bar, Line, Area charts

**Files:**
- Create: `packages/casehub-viz/src/charts/CasehubBarChart.ts`
- Create: `packages/casehub-viz/src/charts/CasehubBarChart.test.ts`
- Create: `packages/casehub-viz/src/charts/CasehubLineChart.ts`
- Create: `packages/casehub-viz/src/charts/CasehubLineChart.test.ts`
- Create: `packages/casehub-viz/src/charts/CasehubAreaChart.ts`
- Create: `packages/casehub-viz/src/charts/CasehubAreaChart.test.ts`

These three share the same Cartesian axis pattern and differ only in series type and subtype mapping. Implement together.

- [ ] **Step 1: Write BarChart buildOption tests**

Cover: default (column), bar (rotated axes), column-stacked, bar-stacked, with legend, with margins, with extra merge, empty dataset, single row, null values.

- [ ] **Step 2: Run, verify RED**

- [ ] **Step 3: Implement CasehubBarChart**

`buildOption()` calls `datasetToSource()`, builds series with `type: "bar"`, applies subtype mapping (stack, axis rotation), calls `applyChartSettings()`, calls `deepMerge()` with `props.extra`. Registers `BarChart`, `GridComponent`, `TooltipComponent`, `LegendComponent`, `DataZoomComponent` via `use()`.

- [ ] **Step 4: Run, verify GREEN**

- [ ] **Step 5: Write LineChart and AreaChart tests, implement, verify GREEN**

Same pattern. LineChart subtypes: `line` (default), `smooth`. AreaChart subtypes: `area`, `area-stacked`.

- [ ] **Step 6: Commit**

```
git commit -m "feat(viz): bar, line, area chart components

Refs #11"
```

---

## Task 7: Pie, Scatter, Bubble charts

**Files:**
- Create: `packages/casehub-viz/src/charts/CasehubPieChart.ts`
- Create: `packages/casehub-viz/src/charts/CasehubPieChart.test.ts`
- Create: `packages/casehub-viz/src/charts/CasehubScatterChart.ts`
- Create: `packages/casehub-viz/src/charts/CasehubScatterChart.test.ts`
- Create: `packages/casehub-viz/src/charts/CasehubBubbleChart.ts`
- Create: `packages/casehub-viz/src/charts/CasehubBubbleChart.test.ts`

- [ ] **Step 1: Write PieChart tests**

Cover: pie (default), donut (radius: ['40%', '70%']), legend, extra merge. Pie skips axis settings in Stage 3.

- [ ] **Step 2: Implement PieChart, verify GREEN**

- [ ] **Step 3: Write ScatterChart tests**

Cover: scatter type, 3-column dataset with symbolSize from column 3, 2-column dataset (no symbolSize).

- [ ] **Step 4: Implement ScatterChart, verify GREEN**

- [ ] **Step 5: Write BubbleChart tests**

Cover: scatter type with symbolSize linear interpolation from minRadius/maxRadius, default min/max values, column 3 value range mapping.

- [ ] **Step 6: Implement BubbleChart, verify GREEN**

- [ ] **Step 7: Commit**

```
git commit -m "feat(viz): pie, scatter, bubble chart components

Refs #11"
```

---

## Task 8: Timeseries chart

**Files:**
- Create: `packages/casehub-viz/src/charts/CasehubTimeseries.ts`
- Create: `packages/casehub-viz/src/charts/CasehubTimeseries.test.ts`

- [ ] **Step 1: Write Timeseries tests**

Cover: xAxis type: 'time', DATE column as x-axis, tooltip.trigger: 'axis', multiple series, chart settings (legend, margins, zoom).

- [ ] **Step 2: Implement Timeseries, verify GREEN**

- [ ] **Step 3: Commit**

```
git commit -m "feat(viz): timeseries chart component

Refs #11"
```

---

## Task 9: Meter (gauge) chart

**Files:**
- Create: `packages/casehub-viz/src/charts/CasehubMeter.ts`
- Create: `packages/casehub-viz/src/charts/CasehubMeter.test.ts`

- [ ] **Step 1: Write Meter tests**

Cover: gauge series type, value from first row/column, max from props.end, color bands from warning/critical proportions, single-color when warning/critical absent, extra merge.

- [ ] **Step 2: Implement Meter, verify GREEN**

- [ ] **Step 3: Commit**

```
git commit -m "feat(viz): meter (gauge) chart component

Refs #11"
```

---

## Task 10: Map chart

**Files:**
- Create: `packages/casehub-viz/src/charts/CasehubMap.ts`
- Create: `packages/casehub-viz/src/charts/CasehubMap.test.ts`

- [ ] **Step 1: Write Map tests**

Cover: regions subtype (map series, visualMap, mapName from props), markers subtype (scatter on geo, coordinate columns), default mapName "world", colorScheme applied, extra merge.

- [ ] **Step 2: Implement Map, verify GREEN**

- [ ] **Step 3: Commit**

```
git commit -m "feat(viz): map chart component (regions, markers)

Refs #11"
```

---

## Task 11: Table component

**Files:**
- Create: `packages/casehub-viz/src/components/CasehubTable.ts`
- Create: `packages/casehub-viz/src/components/CasehubTable.test.ts`

- [ ] **Step 1: Write Table tests**

Cover: renders `<table>` with headers and rows in shadow DOM, display name resolution in headers, client-side pagination (page state, slicing), server-side pagination (emits `casehub-page`), client-side sort (click header), server-side sort (emits `casehub-sort`), click-to-filter (emits `casehub-filter`), CSS custom property usage.

- [ ] **Step 2: Implement Table, verify GREEN**

Renders `<style>` + `<table>` in shadow root. Uses CSS custom properties per spec §7. Client-side vs server-side determined by `totalRows > dataset.rows.length`.

- [ ] **Step 3: Commit**

```
git commit -m "feat(viz): table component — pagination, sort, filter

Refs #11"
```

---

## Task 12: Metric component

**Files:**
- Create: `packages/casehub-viz/src/components/CasehubMetric.ts`
- Create: `packages/casehub-viz/src/components/CasehubMetric.test.ts`

- [ ] **Step 1: Write Metric tests**

Cover: card subtype, card2 subtype, plain-text subtype, quota subtype, html.template substitution, value from first row first column, CSS custom properties.

- [ ] **Step 2: Implement Metric, verify GREEN**

- [ ] **Step 3: Commit**

```
git commit -m "feat(viz): metric component — card, plain-text, quota subtypes

Refs #11"
```

---

## Task 13: Selector component

**Files:**
- Create: `packages/casehub-viz/src/components/CasehubSelector.ts`
- Create: `packages/casehub-viz/src/components/CasehubSelector.test.ts`

- [ ] **Step 1: Write Selector tests**

Cover: dropdown subtype renders `<select>` with distinct values, slider subtype renders `<input type="range">`, labels subtype renders clickable chips, selection emits `casehub-filter`, reset clears selection, CSS custom properties.

- [ ] **Step 2: Implement Selector, verify GREEN**

- [ ] **Step 3: Commit**

```
git commit -m "feat(viz): selector component — dropdown, slider, labels

Refs #11"
```

---

## Task 14: IframePlugin component

**Files:**
- Create: `packages/casehub-viz/src/components/CasehubIframePlugin.ts`
- Create: `packages/casehub-viz/src/components/CasehubIframePlugin.test.ts`

- [ ] **Step 1: Write IframePlugin tests**

Cover: creates `<iframe>` in shadow DOM, sends postMessage with serialized DataSet (via `toWireDataSet()` from `@casehub/data`), receives filter messages from iframe and re-emits as `casehub-filter`, componentId in message properties.

- [ ] **Step 2: Implement IframePlugin, verify GREEN**

- [ ] **Step 3: Commit**

```
git commit -m "feat(viz): iframe-plugin component — postMessage bridge

Refs #11"
```

---

## Task 15: Package index and ECharts tree-shaking registration

**Files:**
- Modify: `packages/casehub-viz/src/index.ts`

- [ ] **Step 1: Write index.ts**

Import all 13 components (triggers `customElements.define()` registration) and re-export all classes, types, and utilities:

```typescript
// Base
export { CasehubElement } from "./base/CasehubElement.js";
export { CasehubChartElement } from "./base/CasehubChartElement.js";
export type { VizComponentProps } from "./base/types.js";
export { cellToRaw, resolveColumnName } from "./base/cell-extract.js";
export { deepMerge } from "./base/deep-merge.js";

// Charts — import triggers customElements.define() and ECharts use() registration
export { CasehubBarChart } from "./charts/CasehubBarChart.js";
export { CasehubLineChart } from "./charts/CasehubLineChart.js";
export { CasehubAreaChart } from "./charts/CasehubAreaChart.js";
export { CasehubPieChart } from "./charts/CasehubPieChart.js";
export { CasehubScatterChart } from "./charts/CasehubScatterChart.js";
export { CasehubBubbleChart } from "./charts/CasehubBubbleChart.js";
export { CasehubTimeseries } from "./charts/CasehubTimeseries.js";
export { CasehubMeter } from "./charts/CasehubMeter.js";
export { CasehubMap } from "./charts/CasehubMap.js";

// HTML components
export { CasehubTable } from "./components/CasehubTable.js";
export { CasehubMetric } from "./components/CasehubMetric.js";
export { CasehubSelector } from "./components/CasehubSelector.js";
export { CasehubIframePlugin } from "./components/CasehubIframePlugin.js";

// Shared pipeline (for custom components)
export { datasetToSource, applyChartSettings } from "./charts/option-pipeline.js";
```

- [ ] **Step 2: Run full test suite**

Run: `workspace @casehub/viz run test -- --run`
Expected: All tests pass.

- [ ] **Step 3: Build**

Run: `workspace @casehub/viz run build`
Expected: Compiles to `dist/` with declarations.

- [ ] **Step 4: Commit**

```
git commit -m "feat(viz): package index with all exports and ECharts registration

Refs #11"
```

---

## Task 16: Cross-package integration test

- [ ] **Step 1: Run all workspace tests**

Run: `workspaces foreach -Apt run test`
Expected: @casehub/data, @casehub/ui, @casehub/viz all pass.

- [ ] **Step 2: Build all packages**

Run: `yarn build:packages` (may need updating in root package.json to include @casehub/viz)
Expected: All packages compile.

- [ ] **Step 3: Update root package.json build scripts if needed**

Add `@casehub/viz` build step to `build:packages` if not auto-discovered by workspace.

- [ ] **Step 4: Final commit**

```
git commit -m "feat(viz): cross-package integration verified

Closes #11"
```
