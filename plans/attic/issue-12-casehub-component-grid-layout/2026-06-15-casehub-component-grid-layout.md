# @casehub/component Extraction and Grid Layout Renderer — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract zero-dep component primitives from `@casehub/ui` into `@casehub/component`, fix DSL slot naming, and implement a CSS Grid layout renderer.

**Architecture:** New `packages/casehub-component/` package with model types (moved from `@casehub/ui`), type guards, and a layout renderer. `@casehub/ui` re-exports from `@casehub/component` — no consumer migration. The renderer takes a `Component` tree and produces DOM using CSS Grid, with activation containers for unknown types.

**Tech Stack:** TypeScript 5.6, Vitest, jsdom, CSS Grid, DOM APIs

**Spec:** `docs/superpowers/specs/2026-06-15-casehub-component-grid-layout-design.md`

---

## File Structure

### New files (packages/casehub-component/)

| File | Responsibility |
|------|---------------|
| `package.json` | Package metadata, zero runtime deps |
| `tsconfig.json` | TS config with DOM lib |
| `vitest.config.ts` | jsdom environment for renderer tests |
| `src/index.ts` | Top-level barrel export |
| `src/model/types.ts` | `Component`, `GridItem`, `GridPlacement`, `AccessControl`, `PermissionContext`, `ALLOW_ALL` |
| `src/model/component-props.ts` | All layout/content/behavioural props interfaces |
| `src/model/type-guards.ts` | `ComponentTypeRegistry` (layout entries), `getProps()`, `is*()` guards |
| `src/model/index.ts` | Model barrel export |
| `src/model/types.test.ts` | Tests moved from `@casehub/ui` |
| `src/model/component-props.test.ts` | Tests moved from `@casehub/ui` |
| `src/model/type-guards.test.ts` | Layout type guard tests (split from `@casehub/ui`) |
| `src/renderer/render.ts` | `renderComponent()` — main entry point |
| `src/renderer/grid.ts` | Grid placement → CSS Grid properties |
| `src/renderer/slots.ts` | Recursive slot composition |
| `src/renderer/id.ts` | Deterministic ID generation |
| `src/renderer/access.ts` | Access control check |
| `src/renderer/layout.ts` | Layout type CSS application (grid, columns, rows, etc.) |
| `src/renderer/interactive.ts` | Tabs, pills, accordion, carousel interactivity |
| `src/renderer/index.ts` | Renderer barrel export |
| `src/renderer/render.test.ts` | Core renderer tests |
| `src/renderer/grid.test.ts` | Grid placement tests |
| `src/renderer/layout.test.ts` | Layout type CSS tests |
| `src/renderer/interactive.test.ts` | Interactive layout tests |
| `src/renderer/access.test.ts` | Access control tests |
| `src/renderer/id.test.ts` | Deterministic ID tests |

### Modified files

| File | Change |
|------|--------|
| `packages/casehub-ui/src/model/types.ts` | Replace content with re-exports from `@casehub/component` |
| `packages/casehub-ui/src/model/component-props.ts` | Replace content with re-exports from `@casehub/component` |
| `packages/casehub-ui/src/model/type-guards.ts` | Import base, extend registry, add chart/data guards, re-export with widened `getProps` |
| `packages/casehub-ui/src/model/type-guards.test.ts` | Remove layout guard tests (moved), keep chart/data guard tests |
| `packages/casehub-ui/src/model/page-types.ts` | Remove dead imports (`FilterSettings`, `RefreshSettings`, `ColumnId`, `ColumnType`) |
| `packages/casehub-ui/src/model/index.ts` | Add re-export of renderer from `@casehub/component` |
| `packages/casehub-ui/package.json` | Add `@casehub/component` dependency |
| `packages/casehub-ui/src/dsl/builders.ts` | Fix `rows()`, `stack()`, `panel()` slot names + stack type |
| `packages/casehub-ui/src/dsl/builders.test.ts` | Update assertions for new slot names and stack type |
| `package.json` (root) | Update `build:packages` script order |

---

### Task 1: Package scaffold

**Files:**
- Create: `packages/casehub-component/package.json`
- Create: `packages/casehub-component/tsconfig.json`
- Create: `packages/casehub-component/vitest.config.ts`
- Create: `packages/casehub-component/src/index.ts`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehub/component",
  "version": "0.0.1",
  "description": "CaseHub Component — layout primitives, type guards, and renderer",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {},
  "devDependencies": {
    "typescript": "^5.6.0",
    "vitest": "^3.0.0",
    "rimraf": "^6.1.0"
  },
  "license": "Apache-2.0"
}
```

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

- [ ] **Step 4: Create placeholder src/index.ts**

```typescript
export * from "./model/index.js";
```

- [ ] **Step 5: Install dependencies**

Run: `yarn install`
Expected: Success — yarn discovers the new workspace package.

- [ ] **Step 6: Commit**

```
git add packages/casehub-component/
git commit -m "chore: scaffold @casehub/component package

Refs #12"
```

---

### Task 2: Move model types to @casehub/component

**Files:**
- Create: `packages/casehub-component/src/model/types.ts`
- Create: `packages/casehub-component/src/model/component-props.ts`
- Create: `packages/casehub-component/src/model/index.ts`
- Create: `packages/casehub-component/src/model/types.test.ts`
- Create: `packages/casehub-component/src/model/component-props.test.ts`
- Modify: `packages/casehub-ui/src/model/types.ts`
- Modify: `packages/casehub-ui/src/model/component-props.ts`
- Modify: `packages/casehub-ui/package.json`

- [ ] **Step 1: Copy types.ts to @casehub/component**

Copy `packages/casehub-ui/src/model/types.ts` verbatim to `packages/casehub-component/src/model/types.ts`. Contents are identical — `Component`, `GridItem`, `GridPlacement`, `AccessControl`, `PermissionContext`, `ALLOW_ALL`.

- [ ] **Step 2: Copy component-props.ts to @casehub/component**

Copy `packages/casehub-ui/src/model/component-props.ts` verbatim to `packages/casehub-component/src/model/component-props.ts`. Contents are identical — all layout/content/behavioural props interfaces.

- [ ] **Step 3: Create model/index.ts**

```typescript
export type {
  Component,
  AccessControl,
  GridPlacement,
  GridItem,
  PermissionContext,
} from "./types.js";
export { ALLOW_ALL } from "./types.js";

export type {
  GridProps,
  ColumnsProps,
  RowsProps,
  StackProps,
  TabsProps,
  PillsProps,
  SidebarProps,
  TreeProps,
  MenuProps,
  AccordionProps,
  CarouselProps,
  AppGridProps,
  PanelProps,
  HtmlProps,
  MarkdownProps,
  TitleProps,
  LazyPageProps,
  FilterSettings,
  DrillDown,
  RefreshSettings,
} from "./component-props.js";
```

- [ ] **Step 4: Copy types.test.ts and component-props.test.ts**

Copy both test files from `packages/casehub-ui/src/model/` to `packages/casehub-component/src/model/`. Update imports to use relative paths (they already use `./types.js` and `./component-props.js` — no change needed).

- [ ] **Step 5: Run tests to verify they pass in the new location**

Run: `yarn workspace @casehub/component run test`
Expected: All tests pass.

- [ ] **Step 6: Replace @casehub/ui types.ts with re-exports**

Replace `packages/casehub-ui/src/model/types.ts` content with:

```typescript
export type {
  Component,
  AccessControl,
  GridPlacement,
  GridItem,
  PermissionContext,
} from "@casehub/component";
export { ALLOW_ALL } from "@casehub/component";
```

- [ ] **Step 7: Replace @casehub/ui component-props.ts with re-exports**

Replace `packages/casehub-ui/src/model/component-props.ts` content with:

```typescript
export type {
  GridProps,
  ColumnsProps,
  RowsProps,
  StackProps,
  TabsProps,
  PillsProps,
  SidebarProps,
  TreeProps,
  MenuProps,
  AccordionProps,
  CarouselProps,
  AppGridProps,
  PanelProps,
  HtmlProps,
  MarkdownProps,
  TitleProps,
  LazyPageProps,
  FilterSettings,
  DrillDown,
  RefreshSettings,
} from "@casehub/component";
```

- [ ] **Step 8: Add @casehub/component dependency to @casehub/ui**

In `packages/casehub-ui/package.json`, add to `dependencies`:
```json
"@casehub/component": "workspace:*"
```

- [ ] **Step 9: Remove duplicate test files from @casehub/ui**

Delete `packages/casehub-ui/src/model/types.test.ts` and `packages/casehub-ui/src/model/component-props.test.ts` — they now live in `@casehub/component`.

- [ ] **Step 10: Clean up dead imports in page-types.ts**

In `packages/casehub-ui/src/model/page-types.ts`, remove the unused imports:
- Remove `import type { FilterSettings, RefreshSettings } from "./component-props.js";`
- Remove `ColumnId,` and `ColumnType,` from the `@casehub/data` import block.

- [ ] **Step 11: Build and run all @casehub/ui tests**

Run: `yarn workspace @casehub/ui run build && yarn workspace @casehub/ui run test`
Expected: All tests pass — re-exports resolve correctly.

- [ ] **Step 12: Commit**

```
git add packages/casehub-component/src/model/ packages/casehub-ui/src/model/ packages/casehub-ui/package.json
git commit -m "feat: extract model types to @casehub/component, re-export from @casehub/ui

Refs #12"
```

---

### Task 3: Split type guards

**Files:**
- Create: `packages/casehub-component/src/model/type-guards.ts`
- Create: `packages/casehub-component/src/model/type-guards.test.ts`
- Modify: `packages/casehub-ui/src/model/type-guards.ts`
- Modify: `packages/casehub-ui/src/model/type-guards.test.ts`
- Modify: `packages/casehub-component/src/model/index.ts`
- Modify: `packages/casehub-ui/src/model/index.ts`

- [ ] **Step 1: Write failing test for base type guards in @casehub/component**

Create `packages/casehub-component/src/model/type-guards.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component } from "./types.js";
import {
  isGrid, isColumns, isRows, isStack, isTabs, isPills,
  isSidebar, isTree, isMenu, isAccordion, isCarousel,
  isAppGrid, isPanel, isHtml, isMarkdown, isTitle, isLazyPage,
  getProps,
} from "./type-guards.js";

describe("type guards - layout components", () => {
  it("isGrid narrows correctly", () => {
    const c: Component = { type: "grid", props: { columns: 12 }, items: [] };
    expect(isGrid(c)).toBe(true);
    if (isGrid(c)) {
      expect(c.props.columns).toBe(12);
    }
  });

  it("isGrid rejects wrong type", () => {
    const c: Component = { type: "columns", props: { distribution: [1, 1] } };
    expect(isGrid(c)).toBe(false);
  });

  it("isColumns narrows correctly", () => {
    const c: Component = { type: "columns", props: { distribution: [1, 2, 1] } };
    expect(isColumns(c)).toBe(true);
    if (isColumns(c)) {
      expect(c.props.distribution).toEqual([1, 2, 1]);
    }
  });

  it("isRows narrows correctly", () => {
    const c: Component = { type: "rows", props: {} };
    expect(isRows(c)).toBe(true);
  });

  it("isStack narrows correctly", () => {
    const c: Component = { type: "stack", props: {} };
    expect(isStack(c)).toBe(true);
  });

  it("isTabs narrows correctly", () => {
    const c: Component = { type: "tabs", props: {} };
    expect(isTabs(c)).toBe(true);
  });

  it("isPills narrows correctly", () => {
    const c: Component = { type: "pills", props: {} };
    expect(isPills(c)).toBe(true);
  });

  it("isSidebar narrows correctly", () => {
    const c: Component = { type: "sidebar", props: {} };
    expect(isSidebar(c)).toBe(true);
  });

  it("isTree narrows correctly", () => {
    const c: Component = { type: "tree", props: {} };
    expect(isTree(c)).toBe(true);
  });

  it("isMenu narrows correctly", () => {
    const c: Component = { type: "menu", props: {} };
    expect(isMenu(c)).toBe(true);
  });

  it("isAccordion narrows correctly", () => {
    const c: Component = { type: "accordion", props: {} };
    expect(isAccordion(c)).toBe(true);
  });

  it("isCarousel narrows correctly", () => {
    const c: Component = { type: "carousel", props: {} };
    expect(isCarousel(c)).toBe(true);
  });

  it("isAppGrid narrows correctly", () => {
    const c: Component = { type: "app-grid", props: {} };
    expect(isAppGrid(c)).toBe(true);
  });
});

describe("type guards - container components", () => {
  it("isPanel narrows correctly", () => {
    const c: Component = { type: "panel", props: { title: "Section" } };
    expect(isPanel(c)).toBe(true);
    if (isPanel(c)) {
      expect(c.props.title).toBe("Section");
    }
  });
});

describe("type guards - content components", () => {
  it("isHtml narrows correctly", () => {
    const c: Component = { type: "html", props: { content: "<p>hi</p>" } };
    expect(isHtml(c)).toBe(true);
    if (isHtml(c)) {
      expect(c.props.content).toBe("<p>hi</p>");
    }
  });

  it("isMarkdown narrows correctly", () => {
    const c: Component = { type: "markdown", props: { content: "# Heading" } };
    expect(isMarkdown(c)).toBe(true);
    if (isMarkdown(c)) {
      expect(c.props.content).toBe("# Heading");
    }
  });

  it("isTitle narrows correctly", () => {
    const c: Component = { type: "title", props: { text: "Dashboard", size: "large" } };
    expect(isTitle(c)).toBe(true);
    if (isTitle(c)) {
      expect(c.props.text).toBe("Dashboard");
    }
  });

  it("isLazyPage narrows correctly", () => {
    const c: Component = { type: "lazy-page", props: { name: "Reports", href: "/reports.yaml" } };
    expect(isLazyPage(c)).toBe(true);
    if (isLazyPage(c)) {
      expect(c.props.name).toBe("Reports");
    }
  });
});

describe("getProps (base registry)", () => {
  it("returns typed props for grid", () => {
    const c: Component = { type: "grid", props: { columns: 12 } };
    const props = getProps(c, "grid");
    expect(props.columns).toBe(12);
  });

  it("returns typed props for panel", () => {
    const c: Component = { type: "panel", props: { title: "Section" } };
    const props = getProps(c, "panel");
    expect(props.title).toBe("Section");
  });

  it("throws for mismatched type", () => {
    const c: Component = { type: "grid", props: { columns: 12 } };
    expect(() => getProps(c, "panel")).toThrow("Expected panel, got grid");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test`
Expected: FAIL — `type-guards.js` does not exist.

- [ ] **Step 3: Create type-guards.ts in @casehub/component**

Create `packages/casehub-component/src/model/type-guards.ts`:

```typescript
import type { Component } from "./types.js";
import type {
  GridProps, ColumnsProps, RowsProps, StackProps,
  TabsProps, PillsProps, SidebarProps, TreeProps,
  MenuProps, AccordionProps, CarouselProps, AppGridProps,
  PanelProps, HtmlProps, MarkdownProps, TitleProps, LazyPageProps,
} from "./component-props.js";

export interface ComponentTypeRegistry {
  grid: GridProps;
  columns: ColumnsProps;
  rows: RowsProps;
  stack: StackProps;
  tabs: TabsProps;
  pills: PillsProps;
  sidebar: SidebarProps;
  tree: TreeProps;
  menu: MenuProps;
  accordion: AccordionProps;
  carousel: CarouselProps;
  "app-grid": AppGridProps;
  panel: PanelProps;
  html: HtmlProps;
  markdown: MarkdownProps;
  title: TitleProps;
  "lazy-page": LazyPageProps;
}

export function getProps<T extends keyof ComponentTypeRegistry>(
  component: Component,
  type: T,
): ComponentTypeRegistry[T] {
  if (component.type !== type) {
    throw new Error(`Expected ${type}, got ${component.type}`);
  }
  return component.props as unknown as ComponentTypeRegistry[T];
}

export function isGrid(c: Component): c is Component & { props: GridProps } {
  return c.type === "grid";
}

export function isColumns(c: Component): c is Component & { props: ColumnsProps } {
  return c.type === "columns";
}

export function isRows(c: Component): c is Component & { props: RowsProps } {
  return c.type === "rows";
}

export function isStack(c: Component): c is Component & { props: StackProps } {
  return c.type === "stack";
}

export function isTabs(c: Component): c is Component & { props: TabsProps } {
  return c.type === "tabs";
}

export function isPills(c: Component): c is Component & { props: PillsProps } {
  return c.type === "pills";
}

export function isSidebar(c: Component): c is Component & { props: SidebarProps } {
  return c.type === "sidebar";
}

export function isTree(c: Component): c is Component & { props: TreeProps } {
  return c.type === "tree";
}

export function isMenu(c: Component): c is Component & { props: MenuProps } {
  return c.type === "menu";
}

export function isAccordion(c: Component): c is Component & { props: AccordionProps } {
  return c.type === "accordion";
}

export function isCarousel(c: Component): c is Component & { props: CarouselProps } {
  return c.type === "carousel";
}

export function isAppGrid(c: Component): c is Component & { props: AppGridProps } {
  return c.type === "app-grid";
}

export function isPanel(c: Component): c is Component & { props: PanelProps } {
  return c.type === "panel";
}

export function isHtml(c: Component): c is Component & { props: HtmlProps } {
  return c.type === "html";
}

export function isMarkdown(c: Component): c is Component & { props: MarkdownProps } {
  return c.type === "markdown";
}

export function isTitle(c: Component): c is Component & { props: TitleProps } {
  return c.type === "title";
}

export function isLazyPage(c: Component): c is Component & { props: LazyPageProps } {
  return c.type === "lazy-page";
}
```

- [ ] **Step 4: Update model/index.ts to export type guards**

Add to `packages/casehub-component/src/model/index.ts`:

```typescript
export type { ComponentTypeRegistry } from "./type-guards.js";
export {
  getProps,
  isGrid, isColumns, isRows, isStack,
  isTabs, isPills, isSidebar, isTree, isMenu,
  isAccordion, isCarousel, isAppGrid,
  isPanel, isHtml, isMarkdown, isTitle, isLazyPage,
} from "./type-guards.js";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehub/component run test`
Expected: All tests pass.

- [ ] **Step 6: Rewrite @casehub/ui type-guards.ts**

Replace `packages/casehub-ui/src/model/type-guards.ts` with:

```typescript
import type { Component } from "./types.js";
import type {
  ComponentTypeRegistry as BaseRegistry,
} from "@casehub/component";
import { getProps as baseGetProps } from "@casehub/component";
import type { PageProps } from "./page-types.js";
import type {
  BarChartProps, LineChartProps, AreaChartProps, PieChartProps,
  ScatterChartProps, BubbleChartProps, TimeseriesProps,
  TableProps, MetricProps, MeterProps, SelectorProps, MapProps,
  IframePluginProps,
} from "./displayer-types.js";

// Re-export all base guards
export {
  isGrid, isColumns, isRows, isStack,
  isTabs, isPills, isSidebar, isTree, isMenu,
  isAccordion, isCarousel, isAppGrid,
  isPanel, isHtml, isMarkdown, isTitle, isLazyPage,
} from "@casehub/component";

export interface ComponentTypeRegistry extends BaseRegistry {
  page: PageProps;
  "bar-chart": BarChartProps;
  "line-chart": LineChartProps;
  "area-chart": AreaChartProps;
  "pie-chart": PieChartProps;
  "scatter-chart": ScatterChartProps;
  "bubble-chart": BubbleChartProps;
  timeseries: TimeseriesProps;
  table: TableProps;
  metric: MetricProps;
  meter: MeterProps;
  selector: SelectorProps;
  map: MapProps;
  "iframe-plugin": IframePluginProps;
}

export const getProps = baseGetProps as <T extends keyof ComponentTypeRegistry>(
  component: Component,
  type: T,
) => ComponentTypeRegistry[T];

export function isPage(c: Component): c is Component & { props: PageProps } {
  return c.type === "page";
}

export function isBarChart(c: Component): c is Component & { props: BarChartProps } {
  return c.type === "bar-chart";
}

export function isLineChart(c: Component): c is Component & { props: LineChartProps } {
  return c.type === "line-chart";
}

export function isAreaChart(c: Component): c is Component & { props: AreaChartProps } {
  return c.type === "area-chart";
}

export function isPieChart(c: Component): c is Component & { props: PieChartProps } {
  return c.type === "pie-chart";
}

export function isScatterChart(c: Component): c is Component & { props: ScatterChartProps } {
  return c.type === "scatter-chart";
}

export function isBubbleChart(c: Component): c is Component & { props: BubbleChartProps } {
  return c.type === "bubble-chart";
}

export function isTimeseries(c: Component): c is Component & { props: TimeseriesProps } {
  return c.type === "timeseries";
}

export function isTable(c: Component): c is Component & { props: TableProps } {
  return c.type === "table";
}

export function isMetric(c: Component): c is Component & { props: MetricProps } {
  return c.type === "metric";
}

export function isMeter(c: Component): c is Component & { props: MeterProps } {
  return c.type === "meter";
}

export function isSelector(c: Component): c is Component & { props: SelectorProps } {
  return c.type === "selector";
}

export function isMap(c: Component): c is Component & { props: MapProps } {
  return c.type === "map";
}

export function isIframePlugin(c: Component): c is Component & { props: IframePluginProps } {
  return c.type === "iframe-plugin";
}
```

- [ ] **Step 7: Update @casehub/ui type-guards.test.ts**

Remove the layout guard tests (they now live in `@casehub/component`). Keep the chart/data guard tests and the `getProps` tests that use chart types. Update imports — `isGrid`, `isColumns`, `isRows`, `isStack`, `isTabs`, `isPills`, `isSidebar`, `isTree`, `isMenu`, `isAccordion`, `isCarousel`, `isAppGrid` are now re-exported from the same file, so imports don't change — just remove the `describe("type guards - layout components")` block.

- [ ] **Step 8: Build @casehub/component, then build and test @casehub/ui**

Run: `yarn workspace @casehub/component run build && yarn workspace @casehub/ui run build && yarn workspace @casehub/ui run test`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```
git add packages/casehub-component/src/model/type-guards.ts packages/casehub-component/src/model/type-guards.test.ts packages/casehub-component/src/model/index.ts packages/casehub-ui/src/model/type-guards.ts packages/casehub-ui/src/model/type-guards.test.ts
git commit -m "feat: split type guards — base in @casehub/component, extended in @casehub/ui

Refs #12"
```

---

### Task 4: Fix DSL slot names and stack type

**Files:**
- Modify: `packages/casehub-ui/src/dsl/builders.ts`
- Modify: `packages/casehub-ui/src/dsl/builders.test.ts`

- [ ] **Step 1: Update builders.test.ts assertions first (TDD — update expected values)**

In `packages/casehub-ui/src/dsl/builders.test.ts`, make these changes:

1. `rows()` test (line 203): change `{ content: [child1, child2] }` to `{ default: [child1, child2] }`
2. `stack()` test (lines 213-219): change to:
```typescript
  describe("stack()", () => {
    it("creates a stack component with children in slots.default", () => {
      const child = html("test");
      const result = stack(child);

      expect(result.type).toBe("stack");
      expect(result.slots).toEqual({ default: [child] });
    });
  });
```
3. `panel()` test (line 274): change `{ content: [child1, child2] }` to `{ default: [child1, child2] }`

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/ui run test -- src/dsl/builders.test.ts`
Expected: FAIL — `rows()` produces `slots.content`, `stack()` produces `type: "rows"`, `panel()` produces `slots.content`.

- [ ] **Step 3: Fix rows() in builders.ts**

In `packages/casehub-ui/src/dsl/builders.ts`, change line 175 from:
```typescript
    slots: { content: children },
```
to:
```typescript
    slots: { default: children },
```

- [ ] **Step 4: Fix stack() in builders.ts**

Replace lines 179-182:
```typescript
export function stack(...children: Component[]): Component {
  // Alias for rows
  return rows(...children);
}
```
with:
```typescript
export function stack(...children: Component[]): Component {
  return freeze({
    type: "stack",
    slots: { default: children },
  });
}
```

- [ ] **Step 5: Fix panel() in builders.ts**

In `packages/casehub-ui/src/dsl/builders.ts`, change line 238 from:
```typescript
    slots: { content: children },
```
to:
```typescript
    slots: { default: children },
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehub/ui run test -- src/dsl/builders.test.ts`
Expected: All tests pass.

- [ ] **Step 7: Run the full @casehub/ui test suite**

Run: `yarn workspace @casehub/ui run test`
Expected: All tests pass. The parser tests and other DSL tests should still pass — the parser produces grid items (not `type: "rows"` components), and the integration test on line 511 asserts `slots?.content` for `page()` which is unchanged.

Note: The integration test at line 511 (`expect(dashboard.slots?.content).toHaveLength(1)`) tests `page()`, which still uses `slots.content`. This is correct — `page()` is NOT changed in this task (see spec §7 — `page()` slot rename is a follow-up).

- [ ] **Step 8: Commit**

```
git add packages/casehub-ui/src/dsl/builders.ts packages/casehub-ui/src/dsl/builders.test.ts
git commit -m "fix: normalise DSL slot names — content→default, stack produces own type

rows() and panel() now use slots.default (Web Components convention).
stack() returns { type: 'stack' } instead of aliasing rows().

Refs #12"
```

---

### Task 5: Renderer core — ID generation and access control

**Files:**
- Create: `packages/casehub-component/src/renderer/id.ts`
- Create: `packages/casehub-component/src/renderer/id.test.ts`
- Create: `packages/casehub-component/src/renderer/access.ts`
- Create: `packages/casehub-component/src/renderer/access.test.ts`

- [ ] **Step 1: Write failing test for ID generation**

Create `packages/casehub-component/src/renderer/id.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { generateId } from "./id.js";

describe("generateId", () => {
  it("returns root for no parent", () => {
    expect(generateId(undefined, undefined, undefined)).toBe("root");
  });

  it("generates slot child ID", () => {
    expect(generateId("root", "main", 0)).toBe("root::main::0");
    expect(generateId("root", "nav", 1)).toBe("root::nav::1");
  });

  it("generates grid item ID", () => {
    expect(generateId("root", 3, 5)).toBe("root::3::5");
  });

  it("handles nested IDs", () => {
    expect(generateId("root::main::0", "default", 2)).toBe("root::main::0::default::2");
  });

  it("avoids collision with underscored slot names", () => {
    const id1 = generateId("root", "a_b", 0);
    const id2 = generateId("root::a", "b", 0);
    expect(id1).not.toBe(id2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test -- src/renderer/id.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement id.ts**

Create `packages/casehub-component/src/renderer/id.ts`:

```typescript
export function generateId(
  parentId: string | undefined,
  slotOrX: string | number | undefined,
  indexOrY: number | undefined,
): string {
  if (parentId === undefined) return "root";
  return `${parentId}::${slotOrX}::${indexOrY}`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehub/component run test -- src/renderer/id.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Write failing test for access control**

Create `packages/casehub-component/src/renderer/access.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { AccessControl, PermissionContext } from "../model/types.js";
import { ALLOW_ALL } from "../model/types.js";
import { checkAccess } from "./access.js";

describe("checkAccess", () => {
  const adminOnly: AccessControl = { roles: ["admin"] };
  const readPerm: AccessControl = { permissions: ["read"] };
  const mixed: AccessControl = { roles: ["editor"], permissions: ["write"] };

  const adminCtx: PermissionContext = {
    hasRole: (r) => r === "admin",
    hasPermission: () => false,
  };

  const readerCtx: PermissionContext = {
    hasRole: () => false,
    hasPermission: (p) => p === "read",
  };

  it("allows when no access control", () => {
    expect(checkAccess(undefined, adminCtx)).toBe(true);
  });

  it("allows when role matches", () => {
    expect(checkAccess(adminOnly, adminCtx)).toBe(true);
  });

  it("denies when role does not match", () => {
    expect(checkAccess(adminOnly, readerCtx)).toBe(false);
  });

  it("allows when permission matches", () => {
    expect(checkAccess(readPerm, readerCtx)).toBe(true);
  });

  it("denies when permission does not match", () => {
    expect(checkAccess(readPerm, adminCtx)).toBe(false);
  });

  it("allows with ALLOW_ALL", () => {
    expect(checkAccess(adminOnly, ALLOW_ALL)).toBe(true);
    expect(checkAccess(readPerm, ALLOW_ALL)).toBe(true);
  });

  it("allows when either role or permission matches", () => {
    expect(checkAccess(mixed, adminCtx)).toBe(false);
    expect(checkAccess(mixed, readerCtx)).toBe(false);
    const editorCtx: PermissionContext = {
      hasRole: (r) => r === "editor",
      hasPermission: () => false,
    };
    expect(checkAccess(mixed, editorCtx)).toBe(true);
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test -- src/renderer/access.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 7: Implement access.ts**

Create `packages/casehub-component/src/renderer/access.ts`:

```typescript
import type { AccessControl, PermissionContext } from "../model/types.js";

export function checkAccess(
  access: AccessControl | undefined,
  permissions: PermissionContext,
): boolean {
  if (!access) return true;

  if (access.roles && access.roles.length > 0) {
    if (access.roles.some((r) => permissions.hasRole(r))) return true;
  }

  if (access.permissions && access.permissions.length > 0) {
    if (access.permissions.some((p) => permissions.hasPermission(p))) return true;
  }

  if ((!access.roles || access.roles.length === 0) &&
      (!access.permissions || access.permissions.length === 0)) {
    return true;
  }

  return false;
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `yarn workspace @casehub/component run test -- src/renderer/access.test.ts`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```
git add packages/casehub-component/src/renderer/
git commit -m "feat: renderer utilities — deterministic ID generation and access control

Refs #12"
```

---

### Task 6: Renderer — grid placement and slot composition

**Files:**
- Create: `packages/casehub-component/src/renderer/grid.ts`
- Create: `packages/casehub-component/src/renderer/grid.test.ts`
- Create: `packages/casehub-component/src/renderer/slots.ts`

- [ ] **Step 1: Write failing test for grid placement**

Create `packages/casehub-component/src/renderer/grid.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { applyGridPlacement } from "./grid.js";

describe("applyGridPlacement", () => {
  it("maps 0-based placement to 1-based CSS Grid", () => {
    const el = document.createElement("div");
    applyGridPlacement(el, { x: 0, y: 0, w: 6, h: 2 });
    expect(el.style.gridColumn).toBe("1 / span 6");
    expect(el.style.gridRow).toBe("1 / span 2");
  });

  it("handles offset positions", () => {
    const el = document.createElement("div");
    applyGridPlacement(el, { x: 8, y: 3, w: 4, h: 1 });
    expect(el.style.gridColumn).toBe("9 / span 4");
    expect(el.style.gridRow).toBe("4 / span 1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test -- src/renderer/grid.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement grid.ts**

Create `packages/casehub-component/src/renderer/grid.ts`:

```typescript
import type { GridPlacement } from "../model/types.js";

export function applyGridPlacement(
  element: HTMLElement,
  placement: GridPlacement,
): void {
  element.style.gridColumn = `${placement.x + 1} / span ${placement.w}`;
  element.style.gridRow = `${placement.y + 1} / span ${placement.h}`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehub/component run test -- src/renderer/grid.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Create slots.ts**

Create `packages/casehub-component/src/renderer/slots.ts`:

```typescript
import type { Component } from "../model/types.js";

export function getSlotChildren(
  component: Component,
  slotName: string,
): readonly Component[] {
  return component.slots?.[slotName] ?? [];
}

export function getSlotNames(component: Component): readonly string[] {
  if (!component.slots) return [];
  return Object.keys(component.slots);
}
```

Slot helpers are trivial — no separate test file needed. They're exercised by the integration tests in the main render tests.

- [ ] **Step 6: Commit**

```
git add packages/casehub-component/src/renderer/grid.ts packages/casehub-component/src/renderer/grid.test.ts packages/casehub-component/src/renderer/slots.ts
git commit -m "feat: renderer — grid placement and slot composition utilities

Refs #12"
```

---

### Task 7: Renderer — layout type CSS

**Files:**
- Create: `packages/casehub-component/src/renderer/layout.ts`
- Create: `packages/casehub-component/src/renderer/layout.test.ts`

- [ ] **Step 1: Write failing test for layout CSS application**

Create `packages/casehub-component/src/renderer/layout.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { applyLayoutCSS, isLayoutType } from "./layout.js";

describe("isLayoutType", () => {
  it("recognises layout types", () => {
    expect(isLayoutType("grid")).toBe(true);
    expect(isLayoutType("columns")).toBe(true);
    expect(isLayoutType("rows")).toBe(true);
    expect(isLayoutType("stack")).toBe(true);
    expect(isLayoutType("tabs")).toBe(true);
    expect(isLayoutType("pills")).toBe(true);
    expect(isLayoutType("accordion")).toBe(true);
    expect(isLayoutType("carousel")).toBe(true);
    expect(isLayoutType("sidebar")).toBe(true);
    expect(isLayoutType("panel")).toBe(true);
    expect(isLayoutType("app-grid")).toBe(true);
  });

  it("rejects non-layout types", () => {
    expect(isLayoutType("bar-chart")).toBe(false);
    expect(isLayoutType("html")).toBe(false);
    expect(isLayoutType("page")).toBe(false);
    expect(isLayoutType("tree")).toBe(false);
    expect(isLayoutType("menu")).toBe(false);
  });
});

describe("applyLayoutCSS", () => {
  it("applies grid CSS", () => {
    const el = document.createElement("div");
    applyLayoutCSS(el, "grid", { columns: 12 });
    expect(el.style.display).toBe("grid");
    expect(el.style.gridTemplateColumns).toBe("repeat(12, 1fr)");
  });

  it("applies columns CSS with distribution", () => {
    const el = document.createElement("div");
    applyLayoutCSS(el, "columns", { distribution: [2, 1] });
    expect(el.style.display).toBe("grid");
    expect(el.style.gridTemplateColumns).toBe("2fr 1fr");
  });

  it("applies rows CSS", () => {
    const el = document.createElement("div");
    applyLayoutCSS(el, "rows", {});
    expect(el.style.display).toBe("flex");
    expect(el.style.flexDirection).toBe("column");
  });

  it("applies stack CSS — no grid, no display override", () => {
    const el = document.createElement("div");
    applyLayoutCSS(el, "stack", {});
    expect(el.style.display).not.toBe("grid");
  });

  it("applies sidebar CSS", () => {
    const el = document.createElement("div");
    applyLayoutCSS(el, "sidebar", {});
    expect(el.style.display).toBe("grid");
    expect(el.style.gridTemplateColumns).toBe("auto 1fr");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test -- src/renderer/layout.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement layout.ts**

Create `packages/casehub-component/src/renderer/layout.ts`:

```typescript
import type { GridProps, ColumnsProps } from "../model/component-props.js";

const LAYOUT_TYPES = new Set([
  "grid", "columns", "rows", "stack",
  "tabs", "pills", "accordion", "carousel",
  "sidebar", "panel", "app-grid",
]);

export function isLayoutType(type: string): boolean {
  return LAYOUT_TYPES.has(type);
}

export function applyLayoutCSS(
  element: HTMLElement,
  type: string,
  props: Readonly<Record<string, unknown>> | undefined,
): void {
  switch (type) {
    case "grid": {
      const gridProps = props as unknown as GridProps | undefined;
      element.style.display = "grid";
      element.style.gridTemplateColumns = `repeat(${gridProps?.columns ?? 12}, 1fr)`;
      break;
    }
    case "columns": {
      const colProps = props as unknown as ColumnsProps | undefined;
      element.style.display = "grid";
      if (colProps?.distribution) {
        element.style.gridTemplateColumns = colProps.distribution.map((n) => `${n}fr`).join(" ");
      }
      break;
    }
    case "rows":
      element.style.display = "flex";
      element.style.flexDirection = "column";
      break;
    case "stack":
    case "tabs":
    case "pills":
    case "carousel":
      break;
    case "accordion":
      element.style.display = "flex";
      element.style.flexDirection = "column";
      break;
    case "sidebar":
      element.style.display = "grid";
      element.style.gridTemplateColumns = "auto 1fr";
      break;
    case "panel":
      break;
    case "app-grid":
      element.style.display = "grid";
      element.style.gridTemplateAreas = '"header header" "nav main" "footer footer"';
      element.style.gridTemplateColumns = "auto 1fr";
      element.style.gridTemplateRows = "auto 1fr auto";
      break;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehub/component run test -- src/renderer/layout.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```
git add packages/casehub-component/src/renderer/layout.ts packages/casehub-component/src/renderer/layout.test.ts
git commit -m "feat: renderer — layout type CSS application

Refs #12"
```

---

### Task 8: Renderer — interactive layout types

**Files:**
- Create: `packages/casehub-component/src/renderer/interactive.ts`
- Create: `packages/casehub-component/src/renderer/interactive.test.ts`

- [ ] **Step 1: Write failing test for tab interactivity**

Create `packages/casehub-component/src/renderer/interactive.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { wireInteractivity } from "./interactive.js";

function makeSlotContainers(slotNames: string[]): {
  container: HTMLDivElement;
  panels: Map<string, HTMLDivElement>;
} {
  const container = document.createElement("div");
  const panels = new Map<string, HTMLDivElement>();
  for (const name of slotNames) {
    const panel = document.createElement("div");
    panel.dataset.slot = name;
    container.appendChild(panel);
    panels.set(name, panel);
  }
  return { container, panels };
}

describe("wireInteractivity — tabs", () => {
  it("creates tab bar with buttons for each slot", () => {
    const { container, panels } = makeSlotContainers(["Sales", "Costs"]);
    wireInteractivity(container, "tabs", ["Sales", "Costs"], panels);
    const bar = container.querySelector("[data-tab-bar]") as HTMLElement;
    expect(bar).toBeTruthy();
    const buttons = bar.querySelectorAll("button");
    expect(buttons).toHaveLength(2);
    expect(buttons[0]!.textContent).toBe("Sales");
    expect(buttons[1]!.textContent).toBe("Costs");
  });

  it("first tab visible by default, rest hidden", () => {
    const { container, panels } = makeSlotContainers(["A", "B", "C"]);
    wireInteractivity(container, "tabs", ["A", "B", "C"], panels);
    expect(panels.get("A")!.style.display).not.toBe("none");
    expect(panels.get("B")!.style.display).toBe("none");
    expect(panels.get("C")!.style.display).toBe("none");
  });

  it("clicking a tab shows target, hides others", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    wireInteractivity(container, "tabs", ["A", "B"], panels);
    const bar = container.querySelector("[data-tab-bar]") as HTMLElement;
    const buttons = bar.querySelectorAll("button");
    buttons[1]!.click();
    expect(panels.get("A")!.style.display).toBe("none");
    expect(panels.get("B")!.style.display).not.toBe("none");
  });
});

describe("wireInteractivity — pills", () => {
  it("behaves same as tabs with pill CSS class", () => {
    const { container, panels } = makeSlotContainers(["X", "Y"]);
    wireInteractivity(container, "pills", ["X", "Y"], panels);
    const bar = container.querySelector("[data-tab-bar]") as HTMLElement;
    expect(bar.classList.contains("casehub-pills")).toBe(true);
  });
});

describe("wireInteractivity — accordion", () => {
  it("creates disclosure headers for each slot", () => {
    const { container, panels } = makeSlotContainers(["Section A", "Section B"]);
    wireInteractivity(container, "accordion", ["Section A", "Section B"], panels);
    const headers = container.querySelectorAll("[data-accordion-header]");
    expect(headers).toHaveLength(2);
    expect(headers[0]!.textContent).toBe("Section A");
    expect(headers[1]!.textContent).toBe("Section B");
  });

  it("all sections expanded by default", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    wireInteractivity(container, "accordion", ["A", "B"], panels);
    expect(panels.get("A")!.style.display).not.toBe("none");
    expect(panels.get("B")!.style.display).not.toBe("none");
  });

  it("clicking header toggles section", () => {
    const { container, panels } = makeSlotContainers(["A"]);
    wireInteractivity(container, "accordion", ["A"], panels);
    const header = container.querySelector("[data-accordion-header]") as HTMLElement;
    header.click();
    expect(panels.get("A")!.style.display).toBe("none");
    header.click();
    expect(panels.get("A")!.style.display).not.toBe("none");
  });
});

describe("wireInteractivity — carousel", () => {
  it("first child visible, rest hidden", () => {
    const { container, panels } = makeSlotContainers(["S1", "S2", "S3"]);
    wireInteractivity(container, "carousel", ["S1", "S2", "S3"], panels);
    expect(panels.get("S1")!.style.display).not.toBe("none");
    expect(panels.get("S2")!.style.display).toBe("none");
    expect(panels.get("S3")!.style.display).toBe("none");
  });

  it("next button cycles forward", () => {
    const { container, panels } = makeSlotContainers(["S1", "S2"]);
    wireInteractivity(container, "carousel", ["S1", "S2"], panels);
    const next = container.querySelector("[data-carousel-next]") as HTMLElement;
    next.click();
    expect(panels.get("S1")!.style.display).toBe("none");
    expect(panels.get("S2")!.style.display).not.toBe("none");
  });

  it("prev button cycles backward (wraps)", () => {
    const { container, panels } = makeSlotContainers(["S1", "S2"]);
    wireInteractivity(container, "carousel", ["S1", "S2"], panels);
    const prev = container.querySelector("[data-carousel-prev]") as HTMLElement;
    prev.click();
    expect(panels.get("S1")!.style.display).toBe("none");
    expect(panels.get("S2")!.style.display).not.toBe("none");
  });
});

describe("wireInteractivity — stack", () => {
  it("first child visible, rest hidden", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    wireInteractivity(container, "stack", ["A", "B"], panels);
    expect(panels.get("A")!.style.display).not.toBe("none");
    expect(panels.get("B")!.style.display).toBe("none");
  });

  it("no controls created for stack", () => {
    const { container, panels } = makeSlotContainers(["A", "B"]);
    wireInteractivity(container, "stack", ["A", "B"], panels);
    expect(container.querySelector("[data-tab-bar]")).toBeNull();
    expect(container.querySelector("[data-carousel-prev]")).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test -- src/renderer/interactive.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement interactive.ts**

Create `packages/casehub-component/src/renderer/interactive.ts`:

```typescript
export function wireInteractivity(
  container: HTMLElement,
  type: string,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
): void {
  switch (type) {
    case "tabs":
    case "pills":
      wireTabs(container, type, slotNames, panels);
      break;
    case "accordion":
      wireAccordion(container, slotNames, panels);
      break;
    case "carousel":
      wireCarousel(container, slotNames, panels);
      break;
    case "stack":
      applyOneVisible(slotNames, panels, 0);
      break;
  }
}

function applyOneVisible(
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
  activeIndex: number,
): void {
  slotNames.forEach((name, i) => {
    const panel = panels.get(name);
    if (panel) {
      panel.style.display = i === activeIndex ? "" : "none";
    }
  });
}

function wireTabs(
  container: HTMLElement,
  type: string,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
): void {
  const bar = document.createElement("div");
  bar.setAttribute("data-tab-bar", "");
  if (type === "pills") bar.classList.add("casehub-pills");
  else bar.classList.add("casehub-tabs");

  for (const name of slotNames) {
    const btn = document.createElement("button");
    btn.textContent = name;
    btn.dataset.slot = name;
    bar.appendChild(btn);
  }

  container.insertBefore(bar, container.firstChild);

  applyOneVisible(slotNames, panels, 0);

  bar.addEventListener("click", (e) => {
    const target = e.target as HTMLElement;
    if (target.tagName !== "BUTTON" || !target.dataset.slot) return;
    const slotName = target.dataset.slot;
    panels.forEach((panel, name) => {
      panel.style.display = name === slotName ? "" : "none";
    });
  });
}

function wireAccordion(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
): void {
  for (const name of slotNames) {
    const panel = panels.get(name);
    if (!panel) continue;
    const header = document.createElement("button");
    header.setAttribute("data-accordion-header", "");
    header.textContent = name;
    container.insertBefore(header, panel);

    header.addEventListener("click", () => {
      panel.style.display = panel.style.display === "none" ? "" : "none";
    });
  }
}

function wireCarousel(
  container: HTMLElement,
  slotNames: readonly string[],
  panels: Map<string, HTMLElement>,
): void {
  let current = 0;
  applyOneVisible(slotNames, panels, current);

  const nav = document.createElement("div");
  const prev = document.createElement("button");
  prev.setAttribute("data-carousel-prev", "");
  prev.textContent = "←";
  const next = document.createElement("button");
  next.setAttribute("data-carousel-next", "");
  next.textContent = "→";
  nav.appendChild(prev);
  nav.appendChild(next);
  container.appendChild(nav);

  prev.addEventListener("click", () => {
    current = (current - 1 + slotNames.length) % slotNames.length;
    applyOneVisible(slotNames, panels, current);
  });

  next.addEventListener("click", () => {
    current = (current + 1) % slotNames.length;
    applyOneVisible(slotNames, panels, current);
  });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehub/component run test -- src/renderer/interactive.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```
git add packages/casehub-component/src/renderer/interactive.ts packages/casehub-component/src/renderer/interactive.test.ts
git commit -m "feat: renderer — interactive layout types (tabs, pills, accordion, carousel)

Refs #12"
```

---

### Task 9: Renderer — renderComponent main entry point

**Files:**
- Create: `packages/casehub-component/src/renderer/render.ts`
- Create: `packages/casehub-component/src/renderer/render.test.ts`
- Create: `packages/casehub-component/src/renderer/index.ts`
- Modify: `packages/casehub-component/src/index.ts`

- [ ] **Step 1: Write failing test for renderComponent**

Create `packages/casehub-component/src/renderer/render.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component } from "../model/types.js";
import { ALLOW_ALL } from "../model/types.js";
import { renderComponent } from "./render.js";

function render(component: Component, options?: Parameters<typeof renderComponent>[2]): HTMLElement {
  const target = document.createElement("div");
  renderComponent(target, component, options);
  return target;
}

describe("renderComponent — basic", () => {
  it("renders unknown type as activation container", () => {
    const target = render({ type: "bar-chart", props: { subtype: "column" } });
    const el = target.firstElementChild as HTMLElement;
    expect(el.dataset.componentType).toBe("bar-chart");
    expect(el.dataset.componentId).toBe("root");
    expect(JSON.parse(el.dataset.componentProps!)).toEqual({ subtype: "column" });
  });

  it("clears target before rendering", () => {
    const target = document.createElement("div");
    target.innerHTML = "<p>old</p>";
    renderComponent(target, { type: "html", props: { content: "new" } });
    expect(target.querySelector("p")).toBeNull();
    expect(target.firstElementChild!.dataset.componentType).toBe("html");
  });

  it("auto-generates deterministic IDs with :: separator", () => {
    const component: Component = {
      type: "rows",
      slots: { default: [{ type: "html", props: { content: "a" } }] },
    };
    const target = render(component);
    const child = target.querySelector("[data-component-type='html']") as HTMLElement;
    expect(child.dataset.componentId).toBe("root::default::0");
  });

  it("uses explicit ID when provided", () => {
    const target = render({ type: "html", id: "my-id", props: { content: "a" } });
    const el = target.firstElementChild as HTMLElement;
    expect(el.dataset.componentId).toBe("my-id");
  });
});

describe("renderComponent — Component.style", () => {
  it("applies style as inline CSS", () => {
    const target = render({
      type: "html",
      props: { content: "a" },
      style: { "background-color": "#f5f5f5", padding: "16px" },
    });
    const el = target.firstElementChild as HTMLElement;
    expect(el.style.backgroundColor).toBe("rgb(245, 245, 245)");
    expect(el.style.padding).toBe("16px");
  });

  it("author style overrides layout CSS", () => {
    const target = render({
      type: "rows",
      props: {},
      style: { display: "block" },
      slots: { default: [] },
    });
    const el = target.firstElementChild as HTMLElement;
    expect(el.style.display).toBe("block");
  });
});

describe("renderComponent — layout types", () => {
  it("renders grid with items", () => {
    const component: Component = {
      type: "grid",
      props: { columns: 12 },
      items: [
        { placement: { x: 0, y: 0, w: 6, h: 2 }, component: { type: "bar-chart", props: {} } },
        { placement: { x: 6, y: 0, w: 6, h: 1 }, component: { type: "metric", props: {} } },
      ],
    };
    const target = render(component);
    const grid = target.firstElementChild as HTMLElement;
    expect(grid.style.display).toBe("grid");
    expect(grid.style.gridTemplateColumns).toBe("repeat(12, 1fr)");

    const children = grid.children;
    expect(children).toHaveLength(2);
    expect((children[0] as HTMLElement).style.gridColumn).toBe("1 / span 6");
    expect((children[0] as HTMLElement).style.gridRow).toBe("1 / span 2");
    expect((children[1] as HTMLElement).style.gridColumn).toBe("7 / span 6");
    expect((children[1] as HTMLElement).style.gridRow).toBe("1 / span 1");
  });

  it("renders columns with col-N slots", () => {
    const component: Component = {
      type: "columns",
      props: { distribution: [2, 1] },
      slots: {
        "col-0": [{ type: "html", props: { content: "left" } }],
        "col-1": [{ type: "html", props: { content: "right" } }],
      },
    };
    const target = render(component);
    const cols = target.firstElementChild as HTMLElement;
    expect(cols.style.gridTemplateColumns).toBe("2fr 1fr");
  });

  it("renders rows with flex column", () => {
    const component: Component = {
      type: "rows",
      slots: { default: [{ type: "html", props: { content: "a" } }] },
    };
    const target = render(component);
    const rows = target.firstElementChild as HTMLElement;
    expect(rows.style.display).toBe("flex");
    expect(rows.style.flexDirection).toBe("column");
  });

  it("renders sidebar with nav and main slots", () => {
    const component: Component = {
      type: "sidebar",
      slots: {
        nav: [{ type: "html", props: { content: "nav" } }],
        main: [{ type: "html", props: { content: "main" } }],
      },
    };
    const target = render(component);
    const sidebar = target.firstElementChild as HTMLElement;
    expect(sidebar.style.gridTemplateColumns).toBe("auto 1fr");
  });

  it("renders panel with title header", () => {
    const component: Component = {
      type: "panel",
      props: { title: "My Panel" },
      slots: { default: [{ type: "html", props: { content: "body" } }] },
    };
    const target = render(component);
    const panel = target.firstElementChild as HTMLElement;
    const header = panel.querySelector("[data-panel-title]") as HTMLElement;
    expect(header.textContent).toBe("My Panel");
  });
});

describe("renderComponent — access control", () => {
  it("skips component when role denied", () => {
    const component: Component = {
      type: "html",
      props: { content: "secret" },
      access: { roles: ["admin"] },
    };
    const target = render(component, {
      permissions: {
        hasRole: () => false,
        hasPermission: () => false,
      },
    });
    expect(target.children).toHaveLength(0);
  });

  it("skips entire subtree when parent denied", () => {
    const component: Component = {
      type: "rows",
      access: { roles: ["admin"] },
      slots: { default: [{ type: "html", props: { content: "child" } }] },
    };
    const target = render(component, {
      permissions: {
        hasRole: () => false,
        hasPermission: () => false,
      },
    });
    expect(target.children).toHaveLength(0);
  });

  it("defaults to ALLOW_ALL when no permissions provided", () => {
    const component: Component = {
      type: "html",
      props: { content: "visible" },
      access: { roles: ["admin"] },
    };
    const target = render(component);
    expect(target.children).toHaveLength(1);
  });
});

describe("renderComponent — page handling", () => {
  it("renders page as activation container but recurses into slots", () => {
    const component: Component = {
      type: "page",
      props: { name: "Dashboard" },
      slots: { content: [{ type: "html", props: { content: "hello" } }] },
    };
    const target = render(component);
    const page = target.firstElementChild as HTMLElement;
    expect(page.dataset.componentType).toBe("page");
    const htmlEl = page.querySelector("[data-component-type='html']") as HTMLElement;
    expect(htmlEl).toBeTruthy();
  });
});

describe("renderComponent — DOM attributes on layout types", () => {
  it("layout types carry all three data attributes", () => {
    const component: Component = {
      type: "grid",
      props: { columns: 12 },
      items: [],
    };
    const target = render(component);
    const grid = target.firstElementChild as HTMLElement;
    expect(grid.dataset.componentType).toBe("grid");
    expect(grid.dataset.componentId).toBe("root");
    expect(JSON.parse(grid.dataset.componentProps!)).toEqual({ columns: 12 });
  });
});

describe("renderComponent — items vs slots precedence", () => {
  it("items take precedence when both present", () => {
    const component: Component = {
      type: "grid",
      props: { columns: 12 },
      items: [
        { placement: { x: 0, y: 0, w: 12, h: 1 }, component: { type: "html", props: { content: "item" } } },
      ],
      slots: { default: [{ type: "html", props: { content: "slot" } }] },
    };
    const target = render(component);
    const grid = target.firstElementChild as HTMLElement;
    const child = grid.querySelector("[data-component-type='html']") as HTMLElement;
    expect(JSON.parse(child.dataset.componentProps!).content).toBe("item");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/component run test -- src/renderer/render.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement render.ts**

Create `packages/casehub-component/src/renderer/render.ts`:

```typescript
import type { Component, PermissionContext } from "../model/types.js";
import type { PanelProps } from "../model/component-props.js";
import { ALLOW_ALL } from "../model/types.js";
import { generateId } from "./id.js";
import { checkAccess } from "./access.js";
import { applyGridPlacement } from "./grid.js";
import { getSlotNames } from "./slots.js";
import { isLayoutType, applyLayoutCSS } from "./layout.js";
import { wireInteractivity } from "./interactive.js";

export interface RenderOptions {
  readonly permissions?: PermissionContext;
  readonly document?: Document;
}

export function renderComponent(
  target: HTMLElement,
  component: Component,
  options?: RenderOptions,
): void {
  const doc = options?.document ?? globalThis.document;
  const permissions = options?.permissions ?? ALLOW_ALL;
  target.innerHTML = "";
  renderNode(target, component, undefined, undefined, undefined, permissions, doc);
}

function renderNode(
  parent: HTMLElement,
  component: Component,
  parentId: string | undefined,
  slotOrX: string | number | undefined,
  indexOrY: number | undefined,
  permissions: PermissionContext,
  doc: Document,
): void {
  if (!checkAccess(component.access, permissions)) return;

  const id = component.id ?? generateId(parentId, slotOrX, indexOrY);
  const el = doc.createElement("div");

  el.dataset.componentType = component.type;
  el.dataset.componentId = id;
  if (component.props) {
    el.dataset.componentProps = JSON.stringify(component.props);
  }

  if (isLayoutType(component.type)) {
    applyLayoutCSS(el, component.type, component.props);
  }

  if (component.style) {
    for (const [prop, value] of Object.entries(component.style)) {
      el.style.setProperty(prop, value);
    }
  }

  if (component.type === "panel") {
    const panelProps = component.props as unknown as PanelProps | undefined;
    if (panelProps?.title) {
      const header = doc.createElement("div");
      header.setAttribute("data-panel-title", "");
      header.textContent = panelProps.title;
      el.appendChild(header);
    }
  }

  const slotNames = getSlotNames(component);
  const panels = new Map<string, HTMLElement>();

  if (component.items && component.items.length > 0) {
    for (const item of component.items) {
      renderNode(el, item.component, id, item.placement.x, item.placement.y, permissions, doc);
      const childEl = el.lastElementChild as HTMLElement;
      if (childEl) {
        applyGridPlacement(childEl, item.placement);
      }
    }
  } else if (slotNames.length > 0) {
    for (const slotName of slotNames) {
      const children = component.slots![slotName]!;
      const slotContainer = doc.createElement("div");
      slotContainer.dataset.slot = slotName;
      panels.set(slotName, slotContainer);

      for (let i = 0; i < children.length; i++) {
        renderNode(slotContainer, children[i]!, id, slotName, i, permissions, doc);
      }

      el.appendChild(slotContainer);
    }
  }

  const interactiveTypes = new Set(["tabs", "pills", "accordion", "carousel", "stack"]);
  if (interactiveTypes.has(component.type) && slotNames.length > 0) {
    wireInteractivity(el, component.type, slotNames, panels);
  }

  parent.appendChild(el);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehub/component run test -- src/renderer/render.test.ts`
Expected: All tests pass. Some tests may need adjustment based on exact DOM structure — fix iteratively.

- [ ] **Step 5: Create renderer/index.ts**

Create `packages/casehub-component/src/renderer/index.ts`:

```typescript
export { renderComponent } from "./render.js";
export type { RenderOptions } from "./render.js";
```

- [ ] **Step 6: Update top-level index.ts**

Replace `packages/casehub-component/src/index.ts` with:

```typescript
export * from "./model/index.js";
export * from "./renderer/index.js";
```

- [ ] **Step 7: Run all @casehub/component tests**

Run: `yarn workspace @casehub/component run test`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```
git add packages/casehub-component/src/renderer/ packages/casehub-component/src/index.ts
git commit -m "feat: implement renderComponent — CSS Grid layout renderer

Recursive DOM builder with grid placement, slot composition,
access control, Component.style, interactive layout types,
deterministic IDs, and activation containers for unknown types.

Refs #12"
```

---

### Task 10: Build integration and final verification

**Files:**
- Modify: `package.json` (root)
- Modify: `packages/casehub-ui/src/model/index.ts`

- [ ] **Step 1: Update build:packages script**

In root `package.json`, update the `build:packages` script. Insert `yarn workspace @casehub/component run build &&` after `@melviz/component-dev` and before `@casehub/data`:

```json
"build:packages": "yarn workspace @melviz/component-api run build && yarn workspace @melviz/component-echarts-base run build && yarn workspace @melviz/component-dev run build && yarn workspace @casehub/component run build && yarn workspace @casehub/data run build && yarn workspace @casehub/ui run build && yarn workspace @casehub/viz run build"
```

- [ ] **Step 2: Re-export renderer from @casehub/ui**

In `packages/casehub-ui/src/model/index.ts`, add at the end:

```typescript
// Renderer re-export
export { renderComponent } from "@casehub/component";
export type { RenderOptions } from "@casehub/component";
```

Or alternatively, in `packages/casehub-ui/src/index.ts`, add:

```typescript
export { renderComponent } from "@casehub/component";
export type { RenderOptions } from "@casehub/component";
```

- [ ] **Step 3: Run full build**

Run: `yarn build:packages`
Expected: All packages build successfully in order.

- [ ] **Step 4: Run all tests across all packages**

Run: `yarn workspace @casehub/component run test && yarn workspace @casehub/ui run test && yarn workspace @casehub/viz run test`
Expected: All tests pass across all three packages.

- [ ] **Step 5: Commit**

```
git add package.json packages/casehub-ui/src/index.ts
git commit -m "chore: integrate @casehub/component into build pipeline

Updated build:packages order, re-export renderer from @casehub/ui.

Refs #12"
```

- [ ] **Step 6: Push branch**

Run: `git push -u fork issue-12-casehub-component-grid-layout`
Expected: Branch pushed to fork remote.
