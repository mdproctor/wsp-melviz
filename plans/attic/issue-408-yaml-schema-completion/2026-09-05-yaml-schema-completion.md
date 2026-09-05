# Schema-Driven YAML Completion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #408 — feat: schema-driven YAML completion for pages-code-editor
**Issue group:** #408

**Goal:** Replace hardcoded YAML completion data with a Zod schema-driven system that serves completion, validation, and future LSP diagnostics.

**Architecture:** New `@casehubio/pages-schema` package contains Zod schemas for all 55 component types plus a composed dashboard document schema using `z.discriminatedUnion`. A generic schema walker in `pages-code-editor` produces CodeMirror completions from any Zod schema via a `createSchemaCompletion(schema)` factory. Existing Zod schemas in `pages-data` are exported publicly.

**Tech Stack:** TypeScript, Zod 3.23+, CodeMirror 6, Vitest

## Global Constraints

- Pre-release platform — breaking changes to exports are fine
- `zod` version `^3.23.0` (matches `pages-data`)
- Package naming: `@casehubio/pages-schema`
- All Zod schemas must produce types assignable to their corresponding TypeScript interfaces (compile-time verified)
- No IntelliJ search indexing for this TypeScript project — use `Read`/`Edit`/`Write` tools for code, `ide_read_file`/`ide_file_structure` for navigation
- Test runner: `vitest run` (no vitest.config.ts needed — uses workspace defaults)

---

## Batch 1: Foundation — pages-schema scaffold + pages-data schema exports

### Task 1: Scaffold pages-schema package with shared base schemas

**Files:**
- Create: `packages/pages-schema/package.json`
- Create: `packages/pages-schema/tsconfig.json`
- Create: `packages/pages-schema/tsconfig.build.json`
- Create: `packages/pages-schema/src/index.ts`
- Create: `packages/pages-schema/src/base-schemas.ts`
- Create: `packages/pages-schema/src/base-schemas.test.ts`

**Interfaces:**
- Consumes: `lookupSchema` from `@casehubio/pages-data` (exported in Task 2)
- Produces: `dataComponentCommonSchema`, `chartSettingsSchema`, `filterSettingsSchema`, `refreshSettingsSchema`, `columnSettingsSchema` — used by all component schemas in Tasks 3-4

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/pages-schema",
  "version": "0.1.0",
  "description": "CaseHub Pages YAML schema — Zod schemas for component types, document structure, and validation",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/casehub-pages.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/pages-data": "workspace:*",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@casehubio/pages-component": "workspace:*",
    "@casehubio/pages-tsconfig": "workspace:*",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "@casehubio/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true
  },
  "include": ["src"],
  "references": []
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "emitDeclarationOnly": false,
    "composite": false
  },
  "exclude": ["**/*.test.ts"]
}
```

- [ ] **Step 4: Write failing tests for base schemas**

Create `packages/pages-schema/src/base-schemas.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { z } from "zod";
import {
  filterSettingsSchema,
  refreshSettingsSchema,
  chartSettingsSchema,
  dataComponentCommonSchema,
} from "./base-schemas.js";
import type {
  FilterSettings,
  RefreshSettings,
} from "@casehubio/pages-component";
import type {
  ChartSettings,
  DataComponentCommon,
} from "@casehubio/pages-component";

describe("base schemas", () => {
  describe("filterSettingsSchema", () => {
    it("parses valid filter settings", () => {
      const input = {
        enabled: true,
        notification: false,
        listening: true,
        selfApply: false,
        drillDown: { target: "Detail", parameters: { region: "region" } },
      };
      const result = filterSettingsSchema.parse(input);
      expect(result.enabled).toBe(true);
      expect(result.drillDown?.target).toBe("Detail");
    });

    it("accepts empty object", () => {
      expect(() => filterSettingsSchema.parse({})).not.toThrow();
    });

    it("type-checks against FilterSettings", () => {
      const _check: FilterSettings = {} as z.output<typeof filterSettingsSchema>;
      expect(_check).toBeDefined();
    });
  });

  describe("refreshSettingsSchema", () => {
    it("parses interval and showStaleIndicator", () => {
      const result = refreshSettingsSchema.parse({ interval: 30, showStaleIndicator: true });
      expect(result.interval).toBe(30);
      expect(result.showStaleIndicator).toBe(true);
    });

    it("type-checks against RefreshSettings", () => {
      const _check: RefreshSettings = {} as z.output<typeof refreshSettingsSchema>;
      expect(_check).toBeDefined();
    });
  });

  describe("chartSettingsSchema", () => {
    it("parses nested chart settings", () => {
      const input = {
        resizable: true,
        zoom: false,
        legend: { show: true, position: "bottom" as const },
        margin: { top: 10, right: 20 },
        xAxis: { title: "Date", showLabels: true },
        grid: { x: true, y: false },
      };
      const result = chartSettingsSchema.parse(input);
      expect(result.legend?.position).toBe("bottom");
      expect(result.margin?.top).toBe(10);
    });

    it("rejects invalid legend position", () => {
      expect(() => chartSettingsSchema.parse({
        legend: { position: "center" },
      })).toThrow();
    });

    it("type-checks against ChartSettings", () => {
      const _check: ChartSettings = {} as z.output<typeof chartSettingsSchema>;
      expect(_check).toBeDefined();
    });
  });

  describe("dataComponentCommonSchema", () => {
    it("parses lookup with uuid", () => {
      const input = {
        title: "Sales",
        visible: true,
        lookup: { uuid: "ds-1" },
      };
      const result = dataComponentCommonSchema.parse(input);
      expect(result.title).toBe("Sales");
    });

    it("requires lookup", () => {
      expect(() => dataComponentCommonSchema.parse({ title: "X" })).toThrow();
    });
  });
});
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: FAIL — module `./base-schemas.js` not found

- [ ] **Step 6: Implement base schemas**

Create `packages/pages-schema/src/base-schemas.ts`:

```typescript
import { z } from "zod";
import { lookupSchema } from "@casehubio/pages-data";

const drillDownSchema = z.object({
  target: z.string(),
  parameters: z.record(z.string()).optional(),
});

export const filterSettingsSchema = z.object({
  enabled: z.boolean().optional(),
  notification: z.boolean().optional(),
  listening: z.boolean().optional(),
  selfApply: z.boolean().optional(),
  group: z.string().optional(),
  drillDown: drillDownSchema.optional(),
});

export const refreshSettingsSchema = z.object({
  interval: z.number().optional(),
  showStaleIndicator: z.boolean().optional(),
});

export const columnSettingsSchema = z.object({
  id: z.string(),
  label: z.string().optional(),
  sortable: z.boolean().optional(),
  visible: z.boolean().optional(),
  width: z.string().optional(),
  minWidth: z.string().optional(),
  align: z.enum(["start", "center", "end"]).optional(),
  filterable: z.boolean().optional(),
  mergeRows: z.boolean().optional(),
});

export const chartSettingsSchema = z.object({
  resizable: z.boolean().optional(),
  zoom: z.boolean().optional(),
  maxWidth: z.number().optional(),
  maxHeight: z.number().optional(),
  legend: z.object({
    show: z.boolean().optional(),
    position: z.enum(["top", "bottom", "left", "right"]).optional(),
  }).optional(),
  margin: z.object({
    top: z.number().optional(),
    right: z.number().optional(),
    bottom: z.number().optional(),
    left: z.number().optional(),
  }).optional(),
  xAxis: z.object({
    title: z.string().optional(),
    showLabels: z.boolean().optional(),
    labelAngle: z.number().optional(),
  }).optional(),
  yAxis: z.object({
    title: z.string().optional(),
    showLabels: z.boolean().optional(),
    labelAngle: z.number().optional(),
  }).optional(),
  grid: z.object({
    x: z.boolean().optional(),
    y: z.boolean().optional(),
  }).optional(),
  extra: z.record(z.unknown()).optional(),
});

export const dataComponentCommonSchema = z.object({
  title: z.string().optional(),
  visible: z.boolean().optional(),
  width: z.string().optional(),
  height: z.string().optional(),
  csvExport: z.boolean().optional(),
  lookup: lookupSchema,
  rowCount: z.number().optional(),
  rowOffset: z.number().optional(),
  columns: z.array(columnSettingsSchema).optional(),
  filter: filterSettingsSchema.optional(),
  refresh: refreshSettingsSchema.optional(),
});
```

- [ ] **Step 7: Create barrel export**

Create `packages/pages-schema/src/index.ts`:

```typescript
export {
  filterSettingsSchema,
  refreshSettingsSchema,
  columnSettingsSchema,
  chartSettingsSchema,
  dataComponentCommonSchema,
} from "./base-schemas.js";
```

- [ ] **Step 8: Install dependencies and run tests**

Run: `yarn install` (from repo root to link the new workspace)
Run: `yarn workspace @casehubio/pages-schema run test`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-schema/
git commit -m "feat(#408): scaffold pages-schema package with shared base schemas

Refs #408"
```

---

### Task 2: Export existing Zod schemas from pages-data

**Files:**
- Modify: `packages/pages-data/src/dataset/lookup-parser.ts` (add exports)
- Modify: `packages/pages-data/src/dataset/external/schema.ts` (add export)
- Modify: `packages/pages-data/src/dataset/external/index.ts` (add re-export)
- Modify: `packages/pages-data/src/index.ts` (add re-export)
- Create: `packages/pages-data/src/dataset/schema-export.test.ts`

**Interfaces:**
- Consumes: nothing new
- Produces: `lookupSchema`, `externalDataSetDefSchema` — public exports from `@casehubio/pages-data`, consumed by `pages-schema` base schemas and document schema

- [ ] **Step 1: Write failing test for public schema exports**

Create `packages/pages-data/src/dataset/schema-export.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { z } from "zod";
import { lookupSchema, externalDataSetDefSchema } from "@casehubio/pages-data";

describe("public schema exports", () => {
  it("lookupSchema is a Zod schema", () => {
    expect(lookupSchema).toBeDefined();
    expect(lookupSchema instanceof z.ZodType).toBe(true);
  });

  it("lookupSchema parses a valid lookup", () => {
    const result = lookupSchema.parse({ uuid: "ds-1" });
    expect(result.uuid).toBe("ds-1");
  });

  it("externalDataSetDefSchema is a Zod schema", () => {
    expect(externalDataSetDefSchema).toBeDefined();
    expect(externalDataSetDefSchema instanceof z.ZodType).toBe(true);
  });

  it("externalDataSetDefSchema parses a valid URL dataset", () => {
    const result = externalDataSetDefSchema.parse({
      uuid: "ds-1",
      url: "https://api.example.com/data",
    });
    expect(result.uuid).toBe("ds-1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-data run test -- src/dataset/schema-export.test.ts`
Expected: FAIL — `lookupSchema` is not exported

- [ ] **Step 3: Export lookupSchema from lookup-parser.ts**

In `packages/pages-data/src/dataset/lookup-parser.ts`, the `lookupSchema` is currently a `const` without `export`. Add `export` to the declaration. Find line:

```typescript
const lookupSchema = z.object({
```

Change to:

```typescript
export const lookupSchema = z.object({
```

- [ ] **Step 4: Export externalDataSetDefSchema from external/schema.ts**

In `packages/pages-data/src/dataset/external/schema.ts`, the `externalDataSetDefSchema` is currently a `const` without `export`. Add `export` to the declaration. Find line:

```typescript
const externalDataSetDefSchema = z.object({
```

Change to:

```typescript
export const externalDataSetDefSchema = z.object({
```

- [ ] **Step 5: Re-export from external/index.ts**

Add to `packages/pages-data/src/dataset/external/index.ts`:

```typescript
export { externalDataSetDefSchema } from "./schema.js";
```

- [ ] **Step 6: Re-export from pages-data barrel**

Add to `packages/pages-data/src/index.ts`:

```typescript
export { lookupSchema } from "./dataset/lookup-parser.js";
```

The `externalDataSetDefSchema` is already reachable via the external barrel. Add it explicitly:

```typescript
export { externalDataSetDefSchema } from "./dataset/external/schema.js";
```

- [ ] **Step 7: Run tests**

Run: `yarn workspace @casehubio/pages-data run test`
Expected: All tests PASS (existing + new)

- [ ] **Step 8: Commit**

```bash
git add packages/pages-data/
git commit -m "feat(#408): export lookupSchema and externalDataSetDefSchema publicly

Refs #408"
```

---

## Batch 2: Component Schemas + Document Schema

### Task 3: Per-component Zod schemas for all 55 component types

**Files:**
- Create: `packages/pages-schema/src/component-schemas.ts`
- Create: `packages/pages-schema/src/component-schemas.test.ts`
- Modify: `packages/pages-schema/src/index.ts` (add exports)

**Interfaces:**
- Consumes: `dataComponentCommonSchema`, `chartSettingsSchema`, `filterSettingsSchema`, `refreshSettingsSchema` from Task 1
- Produces: One named Zod schema per component type (e.g., `barChartPropsSchema`, `dataTablePropsSchema`, etc.) — consumed by document schema in Task 4

The 55 component types fall into these categories. Each category shares a base schema pattern:

**Category A — Chart data components (10 types):** Extend `dataComponentCommon.merge(chartSettings)`. Types: `bar-chart`, `line-chart`, `area-chart`, `pie-chart`, `scatter-chart`, `bubble-chart`, `timeseries`, `heatmap-chart`, `treemap-chart`, `density-heatmap`.

**Category B — Non-chart data components (12 types):** Extend `dataComponentCommon`. Types: `data-table`, `grid-table`, `metric`, `meter`, `selector`, `map`, `badge`, `countdown`, `timeline`, `graph`, `event-timeline`, `metric-grid`.

**Category C — Grouped data (1 type):** Extends `dataComponentCommon` with group-specific fields. Type: `grouped-view`.

**Category D — Layout components (11 types):** Standalone schemas, most are empty records. Types: `grid`, `columns`, `rows`, `stack`, `tabs`, `pills`, `sidebar`, `tree`, `menu`, `accordion`, `carousel`.

**Category E — Workbench components (4 types):** Standalone schemas. Types: `split`, `dock-bar`, `host-panel`, `floating-workspace`.

**Category F — Content/wrapper/page (6 types):** Simple schemas. Types: `panel`, `html`, `markdown`, `title`, `lazy-page`, `page`.

**Category G — Form components (6 types):** Share `formInputCommon` base. Types: `input`, `number-input`, `select`, `checkbox`, `date-picker`, `textarea`.

**Category H — Other components (5 types):** Standalone schemas. Types: `schema-form`, `action-button`, `form-scope`, `submit-button`, `iframe-plugin`.

- [ ] **Step 1: Write failing tests**

Create `packages/pages-schema/src/component-schemas.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { z } from "zod";
import type {
  BarChartProps, DataTableProps, MetricProps, MeterProps,
  LineChartProps, AreaChartProps, PieChartProps,
  ScatterChartProps, BubbleChartProps, TimeseriesProps,
  HeatmapChartProps, TreemapChartProps, DensityHeatmapProps,
  GridTableProps, SelectorProps, MapProps, BadgeProps,
  CountdownProps, TimelineProps, GraphProps, EventTimelineProps,
  MetricGridProps, GroupedViewProps, IframePluginProps,
  SchemaFormProps,
} from "@casehubio/pages-component";
import type {
  GridProps, ColumnsProps, RowsProps, StackProps,
  TabsProps, PillsProps, SidebarProps, TreeProps, MenuProps,
  AccordionProps, CarouselProps, SplitProps, DockBarProps,
  HostPanelProps, FloatingWorkspaceProps,
  PanelProps, HtmlProps, MarkdownProps, TitleProps, LazyPageProps,
  FilterSettings, RefreshSettings,
} from "@casehubio/pages-component";
import type { ActionButtonProps, SubmitButtonProps } from "@casehubio/pages-component";
import type { FormScopeProps } from "@casehubio/pages-component";
import {
  barChartPropsSchema, lineChartPropsSchema, areaChartPropsSchema,
  pieChartPropsSchema, scatterChartPropsSchema, bubbleChartPropsSchema,
  timeseriesPropsSchema, heatmapChartPropsSchema, treemapChartPropsSchema,
  densityHeatmapPropsSchema, metricGridPropsSchema,
  dataTablePropsSchema, gridTablePropsSchema,
  metricPropsSchema, meterPropsSchema, selectorPropsSchema,
  mapPropsSchema, badgePropsSchema, countdownPropsSchema,
  timelinePropsSchema, graphPropsSchema, eventTimelinePropsSchema,
  groupedViewPropsSchema,
  gridPropsSchema, columnsPropsSchema, rowsPropsSchema, stackPropsSchema,
  tabsPropsSchema, pillsPropsSchema, sidebarPropsSchema, treePropsSchema,
  menuPropsSchema, accordionPropsSchema, carouselPropsSchema,
  splitPropsSchema, dockBarPropsSchema, hostPanelPropsSchema,
  floatingWorkspacePropsSchema,
  panelPropsSchema, htmlPropsSchema, markdownPropsSchema, titlePropsSchema,
  lazyPagePropsSchema, pagePropsSchema,
  textInputPropsSchema, numberInputPropsSchema, dropdownPropsSchema,
  checkboxPropsSchema, datePickerPropsSchema, textareaPropsSchema,
  schemaFormPropsSchema, actionButtonPropsSchema,
  formScopePropsSchema, submitButtonPropsSchema,
  iframePluginPropsSchema,
} from "./component-schemas.js";

describe("component schemas", () => {
  describe("chart data components", () => {
    it("barChartPropsSchema parses valid props", () => {
      const result = barChartPropsSchema.parse({
        lookup: { uuid: "ds-1" },
        subtype: "column-stacked",
        zoom: true,
        legend: { show: true, position: "bottom" },
      });
      expect(result.subtype).toBe("column-stacked");
    });

    it("barChartPropsSchema rejects invalid subtype", () => {
      expect(() => barChartPropsSchema.parse({
        lookup: { uuid: "ds-1" },
        subtype: "invalid",
      })).toThrow();
    });

    it("lineChartPropsSchema accepts smooth subtype", () => {
      const result = lineChartPropsSchema.parse({
        lookup: { uuid: "ds-1" },
        subtype: "smooth",
      });
      expect(result.subtype).toBe("smooth");
    });
  });

  describe("non-chart data components", () => {
    it("dataTablePropsSchema parses valid props", () => {
      const result = dataTablePropsSchema.parse({
        lookup: { uuid: "ds-1" },
        pageSize: 25,
        sortable: true,
        selection: "multi",
        selectionKey: "id",
      });
      expect(result.pageSize).toBe(25);
      expect(result.selection).toBe("multi");
    });

    it("metricPropsSchema parses card subtype", () => {
      const result = metricPropsSchema.parse({
        lookup: { uuid: "ds-1" },
        subtype: "card2",
        trend: "up",
      });
      expect(result.subtype).toBe("card2");
    });
  });

  describe("layout components", () => {
    it("gridPropsSchema requires columns", () => {
      const result = gridPropsSchema.parse({ columns: 12 });
      expect(result.columns).toBe(12);
    });

    it("columnsPropsSchema requires distribution", () => {
      const result = columnsPropsSchema.parse({ distribution: [1, 2, 1] });
      expect(result.distribution).toEqual([1, 2, 1]);
    });

    it("rowsPropsSchema accepts empty object", () => {
      expect(() => rowsPropsSchema.parse({})).not.toThrow();
    });

    it("splitPropsSchema parses direction and ratio", () => {
      const result = splitPropsSchema.parse({
        direction: "horizontal",
        ratio: [1, 2],
      });
      expect(result.direction).toBe("horizontal");
    });
  });

  describe("content components", () => {
    it("htmlPropsSchema requires content", () => {
      const result = htmlPropsSchema.parse({ content: "<h1>Hi</h1>" });
      expect(result.content).toBe("<h1>Hi</h1>");
    });

    it("titlePropsSchema requires text", () => {
      const result = titlePropsSchema.parse({ text: "Hello" });
      expect(result.text).toBe("Hello");
    });
  });

  describe("form components", () => {
    it("textInputPropsSchema parses field and placeholder", () => {
      const result = textInputPropsSchema.parse({
        field: "name",
        placeholder: "Enter name",
      });
      expect(result.field).toBe("name");
    });

    it("actionButtonPropsSchema parses style enum", () => {
      const result = actionButtonPropsSchema.parse({
        label: "Delete",
        url: "/api/delete",
        style: "danger",
      });
      expect(result.style).toBe("danger");
    });
  });

  describe("type compatibility", () => {
    it("all chart schemas assignable to their interfaces", () => {
      const _bar: BarChartProps = {} as z.output<typeof barChartPropsSchema>;
      const _line: LineChartProps = {} as z.output<typeof lineChartPropsSchema>;
      const _area: AreaChartProps = {} as z.output<typeof areaChartPropsSchema>;
      const _pie: PieChartProps = {} as z.output<typeof pieChartPropsSchema>;
      const _scatter: ScatterChartProps = {} as z.output<typeof scatterChartPropsSchema>;
      const _bubble: BubbleChartProps = {} as z.output<typeof bubbleChartPropsSchema>;
      const _ts: TimeseriesProps = {} as z.output<typeof timeseriesPropsSchema>;
      expect(true).toBe(true);
    });

    it("data component schemas assignable", () => {
      const _dt: DataTableProps = {} as z.output<typeof dataTablePropsSchema>;
      const _metric: MetricProps = {} as z.output<typeof metricPropsSchema>;
      const _meter: MeterProps = {} as z.output<typeof meterPropsSchema>;
      expect(true).toBe(true);
    });

    it("layout schemas assignable", () => {
      const _grid: GridProps = {} as z.output<typeof gridPropsSchema>;
      const _cols: ColumnsProps = {} as z.output<typeof columnsPropsSchema>;
      const _split: SplitProps = {} as z.output<typeof splitPropsSchema>;
      expect(true).toBe(true);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: FAIL — `./component-schemas.js` not found

- [ ] **Step 3: Implement all component schemas**

Create `packages/pages-schema/src/component-schemas.ts`. This is a large file — all 55 component type schemas. The implementation follows these patterns:

**Pattern A — Chart data component:** `dataComponentCommonSchema.merge(chartSettingsSchema).extend({...type-specific fields})`

**Pattern B — Non-chart data component:** `dataComponentCommonSchema.extend({...type-specific fields})`

**Pattern C — Simple component:** `z.object({...fields})`

**Pattern D — Empty component:** `z.object({})`

**Pattern E — Form input:** `formInputCommonSchema.extend({...type-specific fields})`

Implementation:

```typescript
import { z } from "zod";
import { dataComponentCommonSchema, chartSettingsSchema, filterSettingsSchema, refreshSettingsSchema } from "./base-schemas.js";

// --- Form input common ---
const submitConfigSchema = z.object({
  url: z.string(),
  method: z.enum(["POST", "PUT"]).optional(),
  fieldName: z.string().optional(),
  clearOnSubmit: z.boolean().optional(),
  onSuccess: z.object({ refresh: z.array(z.string()).optional(), message: z.string().optional() }).optional(),
  onError: z.object({ message: z.string().optional() }).optional(),
});

const formInputCommonSchema = z.object({
  field: z.string(),
  label: z.string().optional(),
  required: z.boolean().optional(),
  readonly: z.boolean().optional(),
  submit: submitConfigSchema.optional(),
});

// ============================
// Category A: Chart data components
// ============================
const chartDataBase = dataComponentCommonSchema.merge(chartSettingsSchema);

export const barChartPropsSchema = chartDataBase.extend({
  subtype: z.enum(["column", "column-stacked", "bar", "bar-stacked"]).optional(),
});

export const lineChartPropsSchema = chartDataBase.extend({
  subtype: z.enum(["line", "smooth"]).optional(),
});

export const areaChartPropsSchema = chartDataBase.extend({
  subtype: z.enum(["area", "area-stacked"]).optional(),
});

export const pieChartPropsSchema = chartDataBase.extend({
  subtype: z.enum(["pie", "donut"]).optional(),
});

export const scatterChartPropsSchema = chartDataBase;

export const bubbleChartPropsSchema = chartDataBase.extend({
  minRadius: z.number().optional(),
  maxRadius: z.number().optional(),
});

export const timeseriesPropsSchema = chartDataBase;

export const heatmapChartPropsSchema = chartDataBase.extend({
  minColor: z.string().optional(),
  maxColor: z.string().optional(),
});

export const treemapChartPropsSchema = chartDataBase.extend({
  parentColumn: z.string().optional(),
  colorColumn: z.string().optional(),
});

export const densityHeatmapPropsSchema = dataComponentCommonSchema.extend({
  xColumn: z.string().optional(),
  yColumn: z.string().optional(),
  valueColumn: z.string().optional(),
  gradient: z.array(z.object({ offset: z.number(), color: z.string() })).optional(),
  radius: z.number().optional(),
  aggregation: z.enum(["max", "sum", "mean", "count"]).optional(),
  showTooltip: z.boolean().optional(),
  showLegend: z.boolean().optional(),
});

export const metricGridPropsSchema = z.object({
  direction: z.enum(["row", "grid"]).optional(),
});

// ============================
// Category B: Non-chart data components
// ============================
export const dataTablePropsSchema = dataComponentCommonSchema.extend({
  pageSize: z.number().optional(),
  sortable: z.boolean().optional(),
  resizable: z.boolean().optional(),
  selection: z.enum(["none", "single", "multi"]).optional(),
  selectionKey: z.string().optional(),
});

export const gridTablePropsSchema = dataComponentCommonSchema.extend({
  columnHeaders: z.boolean().optional(),
  rowHeaders: z.boolean().optional(),
  cellDisplay: z.record(z.enum(["text", "boolean", "color", "badge", "number"])).optional(),
  compact: z.boolean().optional(),
  stripe: z.enum(["rows", "columns", "both"]).optional(),
  verticalLines: z.boolean().optional(),
  transpose: z.boolean().optional(),
});

export const metricPropsSchema = dataComponentCommonSchema.extend({
  subtype: z.enum(["card", "card2", "plain-text", "quota"]).optional(),
  pattern: z.string().optional(),
  html: z.object({
    template: z.string().optional(),
    javascript: z.string().optional(),
  }).optional(),
  sparklineData: z.array(z.number()).optional(),
  trend: z.enum(["up", "down", "flat"]).optional(),
});

export const meterPropsSchema = dataComponentCommonSchema.merge(chartSettingsSchema).extend({
  end: z.number().optional(),
  warning: z.number().optional(),
  critical: z.number().optional(),
});

export const selectorPropsSchema = dataComponentCommonSchema.extend({
  subtype: z.enum(["dropdown", "slider", "labels"]).optional(),
});

export const mapPropsSchema = dataComponentCommonSchema.merge(chartSettingsSchema).extend({
  subtype: z.enum(["regions", "markers"]).optional(),
  colorScheme: z.string().optional(),
  mapName: z.string().optional(),
});

export const badgePropsSchema = dataComponentCommonSchema.extend({
  column: z.string().optional(),
  colorMap: z.record(z.string()).optional(),
});

export const countdownPropsSchema = dataComponentCommonSchema.extend({
  deadlineColumn: z.string().optional(),
  format: z.enum(["full", "compact", "days-only"]).optional(),
  warningThreshold: z.string().optional(),
  criticalThreshold: z.string().optional(),
});

export const timelinePropsSchema = dataComponentCommonSchema.merge(chartSettingsSchema).extend({
  startColumn: z.string().optional(),
  endColumn: z.string().optional(),
  labelColumn: z.string().optional(),
  categoryColumn: z.string().optional(),
});

export const graphPropsSchema = dataComponentCommonSchema.merge(chartSettingsSchema).extend({
  layout: z.enum(["force", "circular", "none"]).optional(),
  sourceColumn: z.string().optional(),
  targetColumn: z.string().optional(),
  valueColumn: z.string().optional(),
  directed: z.boolean().optional(),
  nodeLabelColumn: z.string().optional(),
  nodeColorColumn: z.string().optional(),
  nodeColorMap: z.record(z.string()).optional(),
  nodeSizeColumn: z.string().optional(),
});

export const eventTimelinePropsSchema = dataComponentCommonSchema.extend({
  layout: z.enum(["vertical", "horizontal", "compact"]).optional(),
  pageSize: z.number().optional(),
  strategyKey: z.string().optional(),
});

// ============================
// Category C: Grouped data
// ============================
const groupingKeySchema = z.object({
  sourceId: z.string(),
  columnId: z.string(),
  strategy: z.object({ mode: z.string() }).passthrough(),
  maxIntervals: z.number().optional(),
  emptyIntervals: z.boolean().optional(),
  ascendingOrder: z.boolean().optional(),
});

const aggregationBindingSchema = z.object({
  column: z.string(),
  fn: z.object({ fn: z.string() }).passthrough(),
});

const rowAccentConfigSchema = z.object({
  column: z.string(),
  colorMap: z.record(z.string()),
  default: z.string().optional(),
  columns: z.union([z.literal("all"), z.array(z.string())]).optional(),
});

export const groupedViewPropsSchema = dataComponentCommonSchema.extend({
  groupBy: z.union([groupingKeySchema, z.array(groupingKeySchema)]),
  preset: z.enum(["spreadsheet", "sectioned", "list"]).optional(),
  groupDisplay: z.enum(["table-row", "section-heading"]).optional(),
  contentDisplay: z.enum(["table", "list"]).optional(),
  defaultExpanded: z.boolean().optional(),
  showGroupSummary: z.boolean().optional(),
  aggregations: z.array(aggregationBindingSchema).optional(),
  order: z.enum(["asc", "desc"]).optional(),
  emptyGroups: z.boolean().optional(),
  rowAccent: rowAccentConfigSchema.optional(),
  selection: z.enum(["none", "single", "multi"]).optional(),
  sortable: z.boolean().optional(),
  clientSort: z.boolean().optional(),
});

// ============================
// Category D: Layout components
// ============================
export const gridPropsSchema = z.object({
  columns: z.number(),
});

export const columnsPropsSchema = z.object({
  distribution: z.array(z.number()),
});

export const rowsPropsSchema = z.object({});
export const stackPropsSchema = z.object({});
export const tabsPropsSchema = z.object({});
export const pillsPropsSchema = z.object({});
export const sidebarPropsSchema = z.object({});
export const treePropsSchema = z.object({});
export const menuPropsSchema = z.object({});
export const accordionPropsSchema = z.object({});
export const carouselPropsSchema = z.object({});

// ============================
// Category E: Workbench components
// ============================
export const splitPropsSchema = z.object({
  direction: z.enum(["horizontal", "vertical"]),
  ratio: z.array(z.number()).optional(),
  minSizes: z.array(z.number()).optional(),
});

const dockItemSchema = z.object({
  icon: z.string(),
  label: z.string(),
  panelId: z.string(),
  defaultOpen: z.boolean().optional(),
  zone: z.string().optional(),
  fixed: z.boolean().optional(),
});

export const dockBarPropsSchema = z.object({
  orientation: z.enum(["vertical", "horizontal"]),
  items: z.array(dockItemSchema),
  exclusive: z.boolean().optional(),
  side: z.enum(["left", "right", "bottom"]).optional(),
});

export const hostPanelPropsSchema = z.object({
  typeName: z.string(),
  panelProps: z.record(z.unknown()).optional(),
  lookup: z.lazy(() => z.any()).optional(),
  selectionSource: z.string().optional(),
});

export const floatingWorkspacePropsSchema = z.object({
  centre: z.any(),
  frames: z.array(z.any()).optional(),
  organisers: z.boolean().optional(),
});

// ============================
// Category F: Content / wrapper / page
// ============================
export const panelPropsSchema = z.object({
  title: z.string(),
});

export const htmlPropsSchema = z.object({
  content: z.string(),
});

export const markdownPropsSchema = z.object({
  content: z.string(),
});

export const titlePropsSchema = z.object({
  text: z.string(),
  size: z.string().optional(),
});

export const lazyPagePropsSchema = z.object({
  name: z.string(),
  href: z.string(),
});

export const pagePropsSchema = z.object({
  name: z.string().optional(),
  datasets: z.array(z.any()).optional(),
  settings: z.object({
    mode: z.enum(["light", "dark"]).optional(),
    allowUrlProperties: z.boolean().optional(),
  }).passthrough().optional(),
  properties: z.record(z.string()).optional(),
});

// ============================
// Category G: Form components
// ============================
export const textInputPropsSchema = formInputCommonSchema.extend({
  placeholder: z.string().optional(),
  maxLength: z.number().optional(),
});

export const numberInputPropsSchema = formInputCommonSchema.extend({
  min: z.number().optional(),
  max: z.number().optional(),
  step: z.number().optional(),
});

export const dropdownPropsSchema = formInputCommonSchema.extend({
  options: z.union([
    z.object({ values: z.array(z.string()) }),
    z.object({
      dataset: z.string(),
      labelColumn: z.string(),
      valueColumn: z.string(),
      filterField: z.string().optional(),
      filterColumn: z.string().optional(),
    }),
  ]),
});

export const checkboxPropsSchema = formInputCommonSchema;

export const datePickerPropsSchema = formInputCommonSchema.extend({
  min: z.string().optional(),
  max: z.string().optional(),
});

export const textareaPropsSchema = formInputCommonSchema.extend({
  rows: z.number().optional(),
  maxLength: z.number().optional(),
});

// ============================
// Category H: Other components
// ============================
const fieldSchemaZod: z.ZodType<unknown> = z.lazy(() =>
  z.object({
    type: z.union([z.string(), z.array(z.string())]).optional(),
    format: z.string().optional(),
    title: z.string().optional(),
    description: z.string().optional(),
    enum: z.array(z.string()).optional(),
    properties: z.record(fieldSchemaZod).optional(),
    required: z.array(z.string()).optional(),
    items: fieldSchemaZod.optional(),
    oneOf: z.array(fieldSchemaZod).optional(),
    $ref: z.string().optional(),
    $defs: z.record(fieldSchemaZod).optional(),
  }).passthrough()
);

export const schemaFormPropsSchema = z.object({
  schema: fieldSchemaZod.optional(),
  mode: z.enum(["display", "edit"]).optional(),
  forceCreate: z.boolean().optional(),
  validateOnBlur: z.boolean().optional(),
  excludeFields: z.array(z.string()).optional(),
  fieldOrder: z.array(z.string()).optional(),
  fields: z.array(z.string()).optional(),
  labels: z.record(z.string()).optional(),
  fieldsOnly: z.boolean().optional(),
});

export const actionButtonPropsSchema = z.object({
  label: z.string(),
  url: z.string(),
  method: z.enum(["POST", "PUT", "DELETE"]).optional(),
  body: z.record(z.unknown()).optional(),
  headers: z.record(z.string()).optional(),
  confirm: z.string().optional(),
  style: z.enum(["primary", "danger", "secondary", "ghost", "outline"]).optional(),
  disabled: z.boolean().optional(),
  disabledWhen: z.string().optional(),
  onSuccess: z.object({ refresh: z.array(z.string()).optional(), message: z.string().optional() }).optional(),
  onError: z.object({ message: z.string().optional() }).optional(),
});

export const formScopePropsSchema = z.object({
  schema: fieldSchemaZod.optional(),
  validateOnBlur: z.boolean().optional(),
  mode: z.enum(["display", "edit"]).optional(),
});

export const submitButtonPropsSchema = z.object({
  label: z.string(),
  style: z.enum(["primary", "danger", "secondary", "ghost", "outline"]).optional(),
  disabled: z.boolean().optional(),
});

export const iframePluginPropsSchema = z.object({
  componentId: z.string(),
  settings: z.record(z.unknown()).optional(),
  lookup: z.lazy(() => z.any()).optional(),
  title: z.string().optional(),
  visible: z.boolean().optional(),
  width: z.string().optional(),
  height: z.string().optional(),
  filter: filterSettingsSchema.optional(),
  refresh: refreshSettingsSchema.optional(),
});
```

- [ ] **Step 4: Update barrel exports**

Add to `packages/pages-schema/src/index.ts`:

```typescript
export {
  barChartPropsSchema, lineChartPropsSchema, areaChartPropsSchema,
  pieChartPropsSchema, scatterChartPropsSchema, bubbleChartPropsSchema,
  timeseriesPropsSchema, heatmapChartPropsSchema, treemapChartPropsSchema,
  densityHeatmapPropsSchema, metricGridPropsSchema,
  dataTablePropsSchema, gridTablePropsSchema,
  metricPropsSchema, meterPropsSchema, selectorPropsSchema,
  mapPropsSchema, badgePropsSchema, countdownPropsSchema,
  timelinePropsSchema, graphPropsSchema, eventTimelinePropsSchema,
  groupedViewPropsSchema,
  gridPropsSchema, columnsPropsSchema, rowsPropsSchema, stackPropsSchema,
  tabsPropsSchema, pillsPropsSchema, sidebarPropsSchema, treePropsSchema,
  menuPropsSchema, accordionPropsSchema, carouselPropsSchema,
  splitPropsSchema, dockBarPropsSchema, hostPanelPropsSchema,
  floatingWorkspacePropsSchema,
  panelPropsSchema, htmlPropsSchema, markdownPropsSchema, titlePropsSchema,
  lazyPagePropsSchema, pagePropsSchema,
  textInputPropsSchema, numberInputPropsSchema, dropdownPropsSchema,
  checkboxPropsSchema, datePickerPropsSchema, textareaPropsSchema,
  schemaFormPropsSchema, actionButtonPropsSchema,
  formScopePropsSchema, submitButtonPropsSchema,
  iframePluginPropsSchema,
} from "./component-schemas.js";
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-schema/
git commit -m "feat(#408): add Zod schemas for all 55 component types

Refs #408"
```

---

### Task 4: Composed document schema and registry

**Files:**
- Create: `packages/pages-schema/src/document-schema.ts`
- Create: `packages/pages-schema/src/document-schema.test.ts`
- Create: `packages/pages-schema/src/schema-registry.ts`
- Create: `packages/pages-schema/src/schema-registry.test.ts`
- Modify: `packages/pages-schema/src/index.ts` (add exports)

**Interfaces:**
- Consumes: All per-component schemas from Task 3, `externalDataSetDefSchema` from Task 2
- Produces: `dashboardSchema`, `componentSchema`, `componentSchemaRegistry` — consumed by completion engine in Task 5

- [ ] **Step 1: Write failing tests for document schema**

Create `packages/pages-schema/src/document-schema.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { componentSchema, dashboardSchema } from "./document-schema.js";

describe("componentSchema", () => {
  it("validates a bar-chart component", () => {
    const result = componentSchema.parse({
      type: "bar-chart",
      properties: {
        lookup: { uuid: "ds-1" },
        subtype: "column",
      },
    });
    expect(result.type).toBe("bar-chart");
  });

  it("validates a data-table with id and style", () => {
    const result = componentSchema.parse({
      type: "data-table",
      id: "my-table",
      style: { "margin-top": "1rem" },
      properties: {
        lookup: { uuid: "ds-1" },
        pageSize: 25,
      },
    });
    expect(result.id).toBe("my-table");
  });

  it("validates a simple layout component", () => {
    const result = componentSchema.parse({
      type: "rows",
    });
    expect(result.type).toBe("rows");
  });

  it("validates a title component", () => {
    const result = componentSchema.parse({
      type: "title",
      properties: { text: "Hello", size: "h2" },
    });
    expect(result.type).toBe("title");
  });

  it("rejects unknown component type", () => {
    expect(() => componentSchema.parse({
      type: "nonexistent-widget",
    })).toThrow();
  });
});

describe("dashboardSchema", () => {
  it("validates a complete dashboard", () => {
    const result = dashboardSchema.parse({
      pages: [{
        name: "Overview",
        components: [
          { type: "title", properties: { text: "Dashboard" } },
          { type: "bar-chart", properties: { lookup: { uuid: "ds-1" } } },
        ],
      }],
      datasets: [{ uuid: "ds-1", url: "https://api.example.com/data" }],
      properties: { theme: "dark" },
    });
    expect(result.pages).toHaveLength(1);
  });

  it("validates empty dashboard", () => {
    expect(() => dashboardSchema.parse({})).not.toThrow();
  });
});
```

- [ ] **Step 2: Write failing tests for schema registry**

Create `packages/pages-schema/src/schema-registry.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { componentSchemaRegistry } from "./schema-registry.js";

describe("componentSchemaRegistry", () => {
  it("has entries for all 55 component types", () => {
    expect(componentSchemaRegistry.size).toBe(55);
  });

  it("includes bar-chart", () => {
    expect(componentSchemaRegistry.has("bar-chart")).toBe(true);
  });

  it("includes data-table", () => {
    expect(componentSchemaRegistry.has("data-table")).toBe(true);
  });

  it("includes form components", () => {
    expect(componentSchemaRegistry.has("input")).toBe(true);
    expect(componentSchemaRegistry.has("select")).toBe(true);
    expect(componentSchemaRegistry.has("schema-form")).toBe(true);
  });

  it("includes layout components", () => {
    expect(componentSchemaRegistry.has("grid")).toBe(true);
    expect(componentSchemaRegistry.has("columns")).toBe(true);
    expect(componentSchemaRegistry.has("split")).toBe(true);
  });

  it("every entry is a valid Zod schema", () => {
    for (const [name, schema] of componentSchemaRegistry) {
      expect(schema, `${name} should be defined`).toBeDefined();
    }
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: FAIL — modules not found

- [ ] **Step 4: Implement document schema**

Create `packages/pages-schema/src/document-schema.ts`:

```typescript
import { z } from "zod";
import { externalDataSetDefSchema } from "@casehubio/pages-data";
import {
  barChartPropsSchema, lineChartPropsSchema, areaChartPropsSchema,
  pieChartPropsSchema, scatterChartPropsSchema, bubbleChartPropsSchema,
  timeseriesPropsSchema, heatmapChartPropsSchema, treemapChartPropsSchema,
  densityHeatmapPropsSchema, metricGridPropsSchema,
  dataTablePropsSchema, gridTablePropsSchema,
  metricPropsSchema, meterPropsSchema, selectorPropsSchema,
  mapPropsSchema, badgePropsSchema, countdownPropsSchema,
  timelinePropsSchema, graphPropsSchema, eventTimelinePropsSchema,
  groupedViewPropsSchema,
  gridPropsSchema, columnsPropsSchema, rowsPropsSchema, stackPropsSchema,
  tabsPropsSchema, pillsPropsSchema, sidebarPropsSchema, treePropsSchema,
  menuPropsSchema, accordionPropsSchema, carouselPropsSchema,
  splitPropsSchema, dockBarPropsSchema, hostPanelPropsSchema,
  floatingWorkspacePropsSchema,
  panelPropsSchema, htmlPropsSchema, markdownPropsSchema, titlePropsSchema,
  lazyPagePropsSchema, pagePropsSchema,
  textInputPropsSchema, numberInputPropsSchema, dropdownPropsSchema,
  checkboxPropsSchema, datePickerPropsSchema, textareaPropsSchema,
  schemaFormPropsSchema, actionButtonPropsSchema,
  formScopePropsSchema, submitButtonPropsSchema,
  iframePluginPropsSchema,
} from "./component-schemas.js";

const componentBase = z.object({
  id: z.string().optional(),
  style: z.record(z.string()).optional(),
  visibleWhen: z.string().optional(),
});

export const componentSchema = z.discriminatedUnion("type", [
  componentBase.extend({ type: z.literal("grid"), properties: gridPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("columns"), properties: columnsPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("rows"), properties: rowsPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("stack"), properties: stackPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("tabs"), properties: tabsPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("pills"), properties: pillsPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("sidebar"), properties: sidebarPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("tree"), properties: treePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("menu"), properties: menuPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("accordion"), properties: accordionPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("carousel"), properties: carouselPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("split"), properties: splitPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("dock-bar"), properties: dockBarPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("host-panel"), properties: hostPanelPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("floating-workspace"), properties: floatingWorkspacePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("panel"), properties: panelPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("html"), properties: htmlPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("markdown"), properties: markdownPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("title"), properties: titlePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("lazy-page"), properties: lazyPagePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("page"), properties: pagePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("bar-chart"), properties: barChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("line-chart"), properties: lineChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("area-chart"), properties: areaChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("pie-chart"), properties: pieChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("scatter-chart"), properties: scatterChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("bubble-chart"), properties: bubbleChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("timeseries"), properties: timeseriesPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("heatmap-chart"), properties: heatmapChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("treemap-chart"), properties: treemapChartPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("density-heatmap"), properties: densityHeatmapPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("metric-grid"), properties: metricGridPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("data-table"), properties: dataTablePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("grid-table"), properties: gridTablePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("metric"), properties: metricPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("meter"), properties: meterPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("selector"), properties: selectorPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("map"), properties: mapPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("badge"), properties: badgePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("countdown"), properties: countdownPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("timeline"), properties: timelinePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("graph"), properties: graphPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("event-timeline"), properties: eventTimelinePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("grouped-view"), properties: groupedViewPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("iframe-plugin"), properties: iframePluginPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("input"), properties: textInputPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("number-input"), properties: numberInputPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("select"), properties: dropdownPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("checkbox"), properties: checkboxPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("date-picker"), properties: datePickerPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("textarea"), properties: textareaPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("schema-form"), properties: schemaFormPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("action-button"), properties: actionButtonPropsSchema.optional() }),
  componentBase.extend({ type: z.literal("form-scope"), properties: formScopePropsSchema.optional() }),
  componentBase.extend({ type: z.literal("submit-button"), properties: submitButtonPropsSchema.optional() }),
]);

const navItemSchema: z.ZodType<unknown> = z.lazy(() =>
  z.object({
    type: z.enum(["GROUP", "ITEM"]).optional(),
    id: z.string().optional(),
    children: z.array(navItemSchema).optional(),
    page: z.string().optional(),
  })
);

const navTreeSchema = z.object({
  root_items: z.array(navItemSchema).optional(),
});

const rowSchema = z.object({
  columns: z.array(z.any()).optional(),
  properties: z.record(z.unknown()).optional(),
});

const columnSchema = z.object({
  span: z.number().optional(),
  components: z.array(componentSchema).optional(),
  properties: z.record(z.unknown()).optional(),
});

const pageEntrySchema = z.object({
  name: z.string().optional(),
  components: z.array(componentSchema).optional(),
  rows: z.array(rowSchema).optional(),
  columns: z.array(columnSchema).optional(),
  properties: z.record(z.string()).optional(),
});

export const dashboardSchema = z.object({
  pages: z.array(pageEntrySchema).optional(),
  datasets: z.array(externalDataSetDefSchema).optional(),
  navTree: navTreeSchema.optional(),
  properties: z.record(z.string()).optional(),
});
```

- [ ] **Step 5: Implement schema registry**

Create `packages/pages-schema/src/schema-registry.ts`:

```typescript
import { z } from "zod";
import {
  barChartPropsSchema, lineChartPropsSchema, areaChartPropsSchema,
  pieChartPropsSchema, scatterChartPropsSchema, bubbleChartPropsSchema,
  timeseriesPropsSchema, heatmapChartPropsSchema, treemapChartPropsSchema,
  densityHeatmapPropsSchema, metricGridPropsSchema,
  dataTablePropsSchema, gridTablePropsSchema,
  metricPropsSchema, meterPropsSchema, selectorPropsSchema,
  mapPropsSchema, badgePropsSchema, countdownPropsSchema,
  timelinePropsSchema, graphPropsSchema, eventTimelinePropsSchema,
  groupedViewPropsSchema,
  gridPropsSchema, columnsPropsSchema, rowsPropsSchema, stackPropsSchema,
  tabsPropsSchema, pillsPropsSchema, sidebarPropsSchema, treePropsSchema,
  menuPropsSchema, accordionPropsSchema, carouselPropsSchema,
  splitPropsSchema, dockBarPropsSchema, hostPanelPropsSchema,
  floatingWorkspacePropsSchema,
  panelPropsSchema, htmlPropsSchema, markdownPropsSchema, titlePropsSchema,
  lazyPagePropsSchema, pagePropsSchema,
  textInputPropsSchema, numberInputPropsSchema, dropdownPropsSchema,
  checkboxPropsSchema, datePickerPropsSchema, textareaPropsSchema,
  schemaFormPropsSchema, actionButtonPropsSchema,
  formScopePropsSchema, submitButtonPropsSchema,
  iframePluginPropsSchema,
} from "./component-schemas.js";

export const componentSchemaRegistry: ReadonlyMap<string, z.ZodType> = new Map([
  ["grid", gridPropsSchema],
  ["columns", columnsPropsSchema],
  ["rows", rowsPropsSchema],
  ["stack", stackPropsSchema],
  ["tabs", tabsPropsSchema],
  ["pills", pillsPropsSchema],
  ["sidebar", sidebarPropsSchema],
  ["tree", treePropsSchema],
  ["menu", menuPropsSchema],
  ["accordion", accordionPropsSchema],
  ["carousel", carouselPropsSchema],
  ["split", splitPropsSchema],
  ["dock-bar", dockBarPropsSchema],
  ["host-panel", hostPanelPropsSchema],
  ["floating-workspace", floatingWorkspacePropsSchema],
  ["panel", panelPropsSchema],
  ["html", htmlPropsSchema],
  ["markdown", markdownPropsSchema],
  ["title", titlePropsSchema],
  ["lazy-page", lazyPagePropsSchema],
  ["page", pagePropsSchema],
  ["bar-chart", barChartPropsSchema],
  ["line-chart", lineChartPropsSchema],
  ["area-chart", areaChartPropsSchema],
  ["pie-chart", pieChartPropsSchema],
  ["scatter-chart", scatterChartPropsSchema],
  ["bubble-chart", bubbleChartPropsSchema],
  ["timeseries", timeseriesPropsSchema],
  ["heatmap-chart", heatmapChartPropsSchema],
  ["treemap-chart", treemapChartPropsSchema],
  ["density-heatmap", densityHeatmapPropsSchema],
  ["metric-grid", metricGridPropsSchema],
  ["data-table", dataTablePropsSchema],
  ["grid-table", gridTablePropsSchema],
  ["metric", metricPropsSchema],
  ["meter", meterPropsSchema],
  ["selector", selectorPropsSchema],
  ["map", mapPropsSchema],
  ["badge", badgePropsSchema],
  ["countdown", countdownPropsSchema],
  ["timeline", timelinePropsSchema],
  ["graph", graphPropsSchema],
  ["event-timeline", eventTimelinePropsSchema],
  ["grouped-view", groupedViewPropsSchema],
  ["iframe-plugin", iframePluginPropsSchema],
  ["input", textInputPropsSchema],
  ["number-input", numberInputPropsSchema],
  ["select", dropdownPropsSchema],
  ["checkbox", checkboxPropsSchema],
  ["date-picker", datePickerPropsSchema],
  ["textarea", textareaPropsSchema],
  ["schema-form", schemaFormPropsSchema],
  ["action-button", actionButtonPropsSchema],
  ["form-scope", formScopePropsSchema],
  ["submit-button", submitButtonPropsSchema],
]);
```

- [ ] **Step 6: Update barrel exports**

Add to `packages/pages-schema/src/index.ts`:

```typescript
export { componentSchema, dashboardSchema } from "./document-schema.js";
export { componentSchemaRegistry } from "./schema-registry.js";
```

- [ ] **Step 7: Run tests**

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add packages/pages-schema/
git commit -m "feat(#408): add composed document schema and component registry

Refs #408"
```

---

## Batch 3: Completion Engine + Integration

### Task 5: Schema walker and createSchemaCompletion factory

**Files:**
- Create: `packages/pages-code-editor/src/schema-completion.ts`
- Create: `packages/pages-code-editor/src/schema-completion.test.ts`
- Modify: `packages/pages-code-editor/src/index.ts` (add export)
- Modify: `packages/pages-code-editor/package.json` (add zod dependency)

**Interfaces:**
- Consumes: Any `z.ZodType` (generic — works with any Zod schema)
- Produces: `createSchemaCompletion(schema: ZodType): Extension` — CodeMirror extension factory consumed by examples gallery in Task 6

- [ ] **Step 1: Add zod dependency to pages-code-editor**

Add to `packages/pages-code-editor/package.json` dependencies:

```json
"zod": "^3.23.0"
```

Run: `yarn install`

- [ ] **Step 2: Write failing tests for schema walker**

Create `packages/pages-code-editor/src/schema-completion.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { z } from "zod";
import {
  buildYamlContext,
  navigateSchema,
  schemaToCompletions,
} from "./schema-completion.js";

describe("buildYamlContext", () => {
  it("returns empty path at top level", () => {
    const ctx = buildYamlContext("", 0);
    expect(ctx.path).toEqual([]);
  });

  it("builds ancestor path from indented YAML", () => {
    const doc = "pages:\n  - name: Overview\n    components:\n      - type: ";
    const ctx = buildYamlContext(doc, doc.length);
    expect(ctx.path).toContain("pages");
    expect(ctx.path).toContain("components");
  });

  it("extracts sibling key-value pairs", () => {
    const doc = "- type: bar-chart\n  properties:\n    sub";
    const pos = doc.length;
    const ctx = buildYamlContext(doc, pos);
    expect(ctx.siblings.type).toBe("bar-chart");
  });
});

describe("navigateSchema", () => {
  const testSchema = z.object({
    pages: z.array(z.object({
      name: z.string().optional(),
      components: z.array(z.object({
        type: z.string(),
      })),
    })),
    properties: z.record(z.string()),
  });

  it("returns root schema for empty path", () => {
    const result = navigateSchema(testSchema, []);
    expect(result).toBe(testSchema);
  });

  it("descends into object keys", () => {
    const result = navigateSchema(testSchema, ["pages"]);
    expect(result).toBeDefined();
  });

  it("descends through arrays", () => {
    const result = navigateSchema(testSchema, ["pages", "components"]);
    expect(result).toBeDefined();
  });

  it("unwraps optionals", () => {
    const schema = z.object({ foo: z.string().optional() });
    const result = navigateSchema(schema, ["foo"]);
    expect(result).toBeDefined();
  });

  it("handles discriminated union with sibling type", () => {
    const union = z.discriminatedUnion("type", [
      z.object({ type: z.literal("a"), props: z.object({ x: z.number() }) }),
      z.object({ type: z.literal("b"), props: z.object({ y: z.string() }) }),
    ]);
    const result = navigateSchema(union, ["props"], { type: "a" });
    expect(result).toBeDefined();
  });
});

describe("schemaToCompletions", () => {
  it("produces key completions from ZodObject", () => {
    const schema = z.object({
      title: z.string(),
      visible: z.boolean(),
      count: z.number(),
    });
    const completions = schemaToCompletions(schema);
    const labels = completions.map(c => c.label);
    expect(labels).toContain("title");
    expect(labels).toContain("visible");
    expect(labels).toContain("count");
  });

  it("produces value completions from ZodEnum", () => {
    const schema = z.enum(["asc", "desc"]);
    const completions = schemaToCompletions(schema);
    const labels = completions.map(c => c.label);
    expect(labels).toContain("asc");
    expect(labels).toContain("desc");
  });

  it("produces true/false from ZodBoolean", () => {
    const schema = z.boolean();
    const completions = schemaToCompletions(schema);
    const labels = completions.map(c => c.label);
    expect(labels).toContain("true");
    expect(labels).toContain("false");
  });

  it("produces literal value from ZodLiteral", () => {
    const schema = z.literal("fixed-value");
    const completions = schemaToCompletions(schema);
    expect(completions[0]?.label).toBe("fixed-value");
  });

  it("uses description as detail text", () => {
    const schema = z.object({
      subtype: z.enum(["a", "b"]).describe("chart variant"),
    });
    const completions = schemaToCompletions(schema);
    const subtype = completions.find(c => c.label === "subtype");
    expect(subtype?.detail).toBe("chart variant");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-code-editor run test`
Expected: FAIL — `./schema-completion.js` not found

- [ ] **Step 4: Implement schema walker and completion factory**

Create `packages/pages-code-editor/src/schema-completion.ts`:

```typescript
import { autocompletion, type CompletionContext, type CompletionResult } from '@codemirror/autocomplete';
import { type Extension } from '@codemirror/state';
import { z } from 'zod';

interface CompletionEntry {
  label: string;
  detail?: string;
  type: 'property' | 'enum';
  apply?: string;
}

export interface YamlContext {
  path: string[];
  siblings: Record<string, string>;
}

function getIndentLevel(line: string): number {
  const match = line.match(/^(\s*)/);
  return match?.[1]?.length ?? 0;
}

function extractKeyValue(line: string): { key: string; value: string } | null {
  const trimmed = line.trim().replace(/^-\s*/, '');
  const match = trimmed.match(/^(\w[\w-]*):\s*(.*)/);
  if (!match) return null;
  return { key: match[1]!, value: (match[2] ?? '').trim() };
}

export function buildYamlContext(doc: string, pos: number): YamlContext {
  const lines = doc.substring(0, pos).split('\n');
  const currentLine = lines[lines.length - 1] ?? '';
  const currentIndent = getIndentLevel(currentLine);
  const path: string[] = [];
  const siblings: Record<string, string> = {};
  let targetIndent = currentIndent;

  for (let i = lines.length - 2; i >= 0; i--) {
    const rawLine = lines[i] ?? '';
    if (rawLine.trim() === '') continue;
    const lineIndent = getIndentLevel(rawLine);

    if (lineIndent === currentIndent) {
      const kv = extractKeyValue(rawLine);
      if (kv && !(kv.key in siblings)) {
        siblings[kv.key] = kv.value;
      }
    }

    if (lineIndent < targetIndent) {
      const kv = extractKeyValue(rawLine);
      if (kv) {
        path.unshift(kv.key);
        targetIndent = lineIndent;
      }
      if (lineIndent === 0) break;
    }
  }
  return { path, siblings };
}

function unwrap(schema: z.ZodType): z.ZodType {
  if (schema instanceof z.ZodOptional || schema instanceof z.ZodDefault) {
    return unwrap((schema as z.ZodOptional<z.ZodType>)._def.innerType);
  }
  if (schema instanceof z.ZodNullable) {
    return unwrap((schema as z.ZodNullable<z.ZodType>)._def.innerType);
  }
  if (schema instanceof z.ZodLazy) {
    return unwrap(schema._def.getter());
  }
  return schema;
}

export function navigateSchema(
  schema: z.ZodType,
  path: string[],
  siblings?: Record<string, string>,
): z.ZodType | null {
  let current = unwrap(schema);

  for (const key of path) {
    current = unwrap(current);

    if (current instanceof z.ZodObject) {
      const shape = current.shape;
      if (key in shape) {
        current = unwrap(shape[key] as z.ZodType);

        if (current instanceof z.ZodArray) {
          current = unwrap(current._def.type);
        }
      } else {
        return null;
      }
    } else if (current instanceof z.ZodArray) {
      current = unwrap(current._def.type);
      if (current instanceof z.ZodObject) {
        const shape = current.shape;
        if (key in shape) {
          current = unwrap(shape[key] as z.ZodType);
          if (current instanceof z.ZodArray) {
            current = unwrap(current._def.type);
          }
        } else {
          return null;
        }
      }
    } else if (current instanceof z.ZodDiscriminatedUnion) {
      const discriminator = current._def.discriminator as string;
      const typeValue = siblings?.[discriminator];
      if (typeValue) {
        const optionsMap = current._def.optionsMap as Map<string, z.ZodType>;
        const branch = optionsMap.get(typeValue);
        if (branch) {
          current = unwrap(branch);
          if (current instanceof z.ZodObject) {
            const shape = current.shape;
            if (key in shape) {
              current = unwrap(shape[key] as z.ZodType);
              if (current instanceof z.ZodArray) {
                current = unwrap(current._def.type);
              }
            } else {
              return null;
            }
          }
        } else {
          return null;
        }
      } else {
        return null;
      }
    } else if (current instanceof z.ZodUnion) {
      for (const option of current._def.options as z.ZodType[]) {
        const result = navigateSchema(option, [key], siblings);
        if (result) {
          current = result;
          break;
        }
      }
    } else {
      return null;
    }
  }
  return current;
}

export function schemaToCompletions(schema: z.ZodType): CompletionEntry[] {
  const unwrapped = unwrap(schema);

  if (unwrapped instanceof z.ZodObject) {
    const shape = unwrapped.shape as Record<string, z.ZodType>;
    return Object.entries(shape).map(([key, fieldSchema]) => ({
      label: key,
      detail: fieldSchema.description ?? describeType(unwrap(fieldSchema)),
      type: 'property' as const,
      apply: key + ': ',
    }));
  }

  if (unwrapped instanceof z.ZodEnum) {
    return (unwrapped._def.values as string[]).map(v => ({
      label: v,
      type: 'enum' as const,
    }));
  }

  if (unwrapped instanceof z.ZodNativeEnum) {
    const values = Object.values(unwrapped._def.values as Record<string, string | number>)
      .filter((v): v is string => typeof v === 'string');
    return values.map(v => ({
      label: v,
      type: 'enum' as const,
    }));
  }

  if (unwrapped instanceof z.ZodLiteral) {
    return [{
      label: String(unwrapped._def.value),
      type: 'enum' as const,
    }];
  }

  if (unwrapped instanceof z.ZodBoolean) {
    return [
      { label: 'true', type: 'enum' as const },
      { label: 'false', type: 'enum' as const },
    ];
  }

  if (unwrapped instanceof z.ZodDiscriminatedUnion) {
    const discriminator = unwrapped._def.discriminator as string;
    const optionsMap = unwrapped._def.optionsMap as Map<string, z.ZodType>;
    const typeValues = [...optionsMap.keys()];
    const firstBranch = optionsMap.values().next().value;
    const branchCompletions = firstBranch ? schemaToCompletions(firstBranch) : [];
    const commonKeys = branchCompletions.filter(c =>
      c.label !== discriminator
    );
    return [
      { label: discriminator, detail: typeValues.join(' | '), type: 'property' as const, apply: discriminator + ': ' },
      ...commonKeys,
    ];
  }

  if (unwrapped instanceof z.ZodUnion) {
    const allCompletions: CompletionEntry[] = [];
    for (const option of unwrapped._def.options as z.ZodType[]) {
      allCompletions.push(...schemaToCompletions(option));
    }
    const seen = new Set<string>();
    return allCompletions.filter(c => {
      if (seen.has(c.label)) return false;
      seen.add(c.label);
      return true;
    });
  }

  return [];
}

function describeType(schema: z.ZodType): string | undefined {
  if (schema instanceof z.ZodString) return 'string';
  if (schema instanceof z.ZodNumber) return 'number';
  if (schema instanceof z.ZodBoolean) return 'boolean';
  if (schema instanceof z.ZodEnum) return (schema._def.values as string[]).join(' | ');
  if (schema instanceof z.ZodArray) return 'array';
  if (schema instanceof z.ZodObject) return 'object';
  if (schema instanceof z.ZodRecord) return 'record';
  return undefined;
}

function schemaCompletionSource(
  schema: z.ZodType,
): (context: CompletionContext) => CompletionResult | null {
  return (context: CompletionContext) => {
    const line = context.state.doc.lineAt(context.pos);
    const textBefore = line.text.substring(0, context.pos - line.from);
    const doc = context.state.doc.toString();
    const yamlCtx = buildYamlContext(doc, context.pos);

    const afterValueColon = textBefore.match(/(?:^|\s)-?\s*(\w[\w-]*):\s*(\S*)$/);
    if (afterValueColon) {
      const key = afterValueColon[1] ?? '';
      const prefix = afterValueColon[2] ?? '';
      const resolved = navigateSchema(schema, [...yamlCtx.path, key], yamlCtx.siblings);
      if (resolved) {
        const completions = schemaToCompletions(resolved);
        if (completions.length > 0 && completions[0]?.type === 'enum') {
          return {
            from: context.pos - prefix.length,
            options: completions.map(c => ({ ...c, type: 'enum' as const })),
          };
        }
      }
      return null;
    }

    const keyMatch = textBefore.match(/(?:^|\s)-?\s*(\w[\w-]*)$/);
    const resolved = navigateSchema(schema, yamlCtx.path, yamlCtx.siblings);
    if (!resolved) return null;
    const completions = schemaToCompletions(resolved);
    if (completions.length === 0) return null;

    if (keyMatch) {
      const prefix = keyMatch[1] ?? '';
      if (!prefix && !context.explicit) return null;
      return {
        from: context.pos - prefix.length,
        options: completions.map(c => ({
          label: c.label,
          detail: c.detail,
          type: c.type,
          apply: c.apply,
        })),
      };
    }

    const emptyMatch = textBefore.match(/(?:^|\s)-?\s*$/);
    if (emptyMatch && context.explicit) {
      return {
        from: context.pos,
        options: completions.map(c => ({
          label: c.label,
          detail: c.detail,
          type: c.type,
          apply: c.apply,
        })),
      };
    }

    return null;
  };
}

export function createSchemaCompletion(schema: z.ZodType): Extension {
  return autocompletion({
    override: [schemaCompletionSource(schema)],
    activateOnTyping: true,
  });
}
```

- [ ] **Step 5: Add export to barrel**

Add to `packages/pages-code-editor/src/index.ts`:

```typescript
export { createSchemaCompletion } from './schema-completion.js';
```

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/pages-code-editor run test`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-code-editor/
git commit -m "feat(#408): add schema-driven completion walker and factory

Refs #408"
```

---

### Task 6: Remove hardcoded completion, update examples gallery, build chain

**Files:**
- Delete: `packages/pages-code-editor/src/yaml-completion.ts` (replaced by schema-completion.ts)
- Modify: `packages/pages-code-editor/src/index.ts` (remove old export)
- Modify: `examples/src/casehub-entry.ts` (switch to schema-driven completion)
- Modify: `package.json` (add pages-schema to build:packages chain)

**Interfaces:**
- Consumes: `createSchemaCompletion` from Task 5, `dashboardSchema` from Task 4
- Produces: updated `casehub-bundle` with schema-driven completion

- [ ] **Step 1: Remove hardcoded yaml-completion.ts**

Delete `packages/pages-code-editor/src/yaml-completion.ts`.

- [ ] **Step 2: Update pages-code-editor barrel**

Replace the contents of `packages/pages-code-editor/src/index.ts`:

```typescript
export { PagesCodeEditor } from './pages-code-editor.js';
export { createSchemaCompletion } from './schema-completion.js';
```

- [ ] **Step 3: Update examples gallery entry point**

In `examples/src/casehub-entry.ts`:

Replace the import:
```typescript
import { yamlCompletion } from "@casehubio/pages-code-editor";
```

With:
```typescript
import { createSchemaCompletion } from "@casehubio/pages-code-editor";
import { dashboardSchema } from "@casehubio/pages-schema";
```

Replace the export:
```typescript
export { yamlCompletion };
```

With:
```typescript
export const yamlCompletion = createSchemaCompletion(dashboardSchema);
```

- [ ] **Step 4: Add pages-schema to build chain**

In root `package.json`, add `pages-schema` to the `build:packages` script. It must build after `pages-data` and before `pages-code-editor`. Insert after the `pages-component` build step:

```
yarn workspace @casehubio/pages-schema run build &&
```

- [ ] **Step 5: Add pages-schema dependency to examples**

Check if examples/package.json needs `@casehubio/pages-schema` as a dependency. If examples uses workspace resolution, add:

```json
"@casehubio/pages-schema": "workspace:*"
```

- [ ] **Step 6: Run full build and tests**

Run: `yarn run build:packages`
Expected: All packages build successfully

Run: `yarn workspace @casehubio/pages-code-editor run test`
Expected: All tests PASS

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: All tests PASS

- [ ] **Step 7: Verify examples build**

Run: `yarn workspace examples run build` (or the appropriate build command)
Expected: Build succeeds — `casehub-bundle` includes schema-driven completion

- [ ] **Step 8: Commit**

```bash
git add packages/pages-code-editor/ packages/pages-schema/ examples/ package.json
git commit -m "feat(#408): replace hardcoded completion with schema-driven engine

Removes yaml-completion.ts, wires dashboardSchema through createSchemaCompletion
factory in examples gallery.

Refs #408"
```

---

## References

- [specs/issue-408-yaml-schema-completion/2026-09-05-yaml-schema-completion-design.md] — design spec this plan implements
- [packages/pages-code-editor/src/yaml-completion.ts] — existing hardcoded completion (replaced)
- [packages/pages-component/src/model/type-guards.ts:64-134] — ComponentTypeRegistry (55 component types)
- [packages/pages-component/src/model/displayer-types.ts] — data component TypeScript interfaces
- [packages/pages-component/src/model/component-props.ts] — layout/content component TypeScript interfaces
- [packages/pages-component/src/model/form-input-types.ts] — form input TypeScript interfaces
- [packages/pages-component/src/model/action-types.ts] — action button TypeScript interfaces
- [packages/pages-data/src/dataset/lookup-parser.ts] — existing Zod lookup schema
- [packages/pages-data/src/dataset/external/schema.ts] — existing Zod dataset-def schema
- [GE-20260813-674be0] — YAML desugarer drops unknown component props silently
- [GE-20260823-590f19] — three-point registration pain
- [GitHub #408] — focal issue
- [GitHub #407] — LSP server (downstream consumer)
- [GitHub #372] — pages-code-editor (parent work)
