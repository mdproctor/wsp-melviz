# TypeScript Strict Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce maximum TypeScript strict mode across all packages, eliminate all type casts, and add CI enforcement.

**Architecture:** Move component type definitions from pages-ui to pages-component (completing the component model layer). Make Component generic `Component<T, P>` so types propagate across package boundaries. Unify the split ComponentTypeRegistry. Upgrade the base tsconfig to maximum strict. Add tsconfig split for test file coverage and root-level typecheck script with CI enforcement.

**Tech Stack:** TypeScript 5.6, Vitest, Jest, Yarn 4 workspaces, Webpack 5

## Global Constraints

- All type changes in pages-component, pages-data, pages-ui, pages-viz, pages-runtime use existing `strict: true` tsconfigs — they are already strict.
- Legacy packages (iframe-api, iframe-dev, echarts-base, component-echarts, component-svg-heatmap) currently use TypeScript 4.6.2 — must upgrade to 5.6.0 when the base tsconfig changes.
- `verbatimModuleSyntax` requires `import type` for type-only imports — mechanical migration during tsconfig upgrade.
- `exactOptionalPropertyTypes` rejects `undefined` for optional properties (GE-20260612-d561ae) — use conditional object construction.
- After changing pages-component's types, downstream packages must rebuild to pick up new `.d.ts` (GE-20260616-e268d7) — `yarn build:packages` handles this.
- All commits reference `Refs #1` (casehubio/casehub-pages#1).
- Run `yarn build:packages && yarn test` after each task to verify no regressions.

---

### Task 1: Branded Type Constructors

Add `dataSetId()` and `columnId()` factory functions to pages-data so tests and production code can construct branded types without `as` casts.

**Files:**
- Create: `packages/pages-data/src/dataset/constructors.ts`
- Create: `packages/pages-data/src/dataset/constructors.test.ts`
- Modify: `packages/pages-data/src/dataset/types.ts` — add re-export

**Interfaces:**
- Produces: `dataSetId(id: string): DataSetId`, `columnId(id: string): ColumnId` — used by Tasks 6 and 7 to eliminate `as DataSetId` / `as ColumnId` casts

- [ ] **Step 1: Write the failing test**

```typescript
// packages/pages-data/src/dataset/constructors.test.ts
import { describe, it, expect } from "vitest";
import { dataSetId, columnId } from "./constructors.js";
import type { DataSetId, ColumnId } from "./types.js";

describe("branded type constructors", () => {
  it("dataSetId creates a DataSetId from a string", () => {
    const id: DataSetId = dataSetId("my-dataset");
    expect(id).toBe("my-dataset");
  });

  it("columnId creates a ColumnId from a string", () => {
    const id: ColumnId = columnId("col-1");
    expect(id).toBe("col-1");
  });

  it("DataSetId is assignable where DataSetId is expected", () => {
    const id = dataSetId("ds");
    const fn = (dsId: DataSetId) => dsId;
    expect(fn(id)).toBe("ds");
  });

  it("ColumnId is assignable where ColumnId is expected", () => {
    const id = columnId("c");
    const fn = (colId: ColumnId) => colId;
    expect(fn(id)).toBe("c");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/constructors.test.ts`
Expected: FAIL — cannot resolve `./constructors.js`

- [ ] **Step 3: Write minimal implementation**

```typescript
// packages/pages-data/src/dataset/constructors.ts
import type { DataSetId, ColumnId } from "./types.js";

export function dataSetId(id: string): DataSetId {
  return id as DataSetId;
}

export function columnId(id: string): ColumnId {
  return id as ColumnId;
}
```

- [ ] **Step 4: Export from types.ts**

Add to the end of `packages/pages-data/src/dataset/types.ts`:
```typescript
export { dataSetId, columnId } from "./constructors.js";
```

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehub/pages-data run test -- --reporter=verbose src/dataset/constructors.test.ts`
Expected: PASS — all 4 tests green

- [ ] **Step 6: Run full package tests**

Run: `yarn workspace @casehub/pages-data run test`
Expected: All existing tests pass (no regressions)

- [ ] **Step 7: Commit**

```bash
git add packages/pages-data/src/dataset/constructors.ts packages/pages-data/src/dataset/constructors.test.ts packages/pages-data/src/dataset/types.ts
git commit -m "feat: add branded type constructors for DataSetId and ColumnId  Refs #1"
```

---

### Task 2: Move Type Definitions from pages-ui to pages-component

Move displayer-types.ts, form-input-types.ts, and the component-level portion of page-types.ts from pages-ui to pages-component. Add pages-data as a dependency of pages-component (required because props reference DataSetLookup, DataSetId).

**Files:**
- Modify: `packages/pages-component/package.json` — add `@casehub/pages-data` dependency
- Move: `packages/pages-ui/src/model/displayer-types.ts` → `packages/pages-component/src/model/displayer-types.ts`
- Move: `packages/pages-ui/src/model/form-input-types.ts` → `packages/pages-component/src/model/form-input-types.ts`
- Create: `packages/pages-component/src/model/page-props.ts` — PageProps and supporting types extracted from page-types.ts
- Modify: `packages/pages-ui/src/model/page-types.ts` — remove moved types, import from pages-component
- Modify: `packages/pages-component/src/model/index.ts` — export new modules
- Modify: `packages/pages-ui/src/model/index.ts` — re-export from pages-component
- Modify: `packages/pages-ui/src/model/displayer-types.test.ts` — update imports
- Modify: `packages/pages-ui/src/model/form-input-types.test.ts` — update imports
- Modify: Root `package.json` — update build order (data before component)

**Interfaces:**
- Consumes: pages-data types (DataSetLookup, DataSetId, ColumnId, ExternalDataSetDef, etc.)
- Produces: All displayer props, form input props, PageProps available from `@casehub/pages-component`

- [ ] **Step 1: Add pages-data dependency to pages-component**

In `packages/pages-component/package.json`, add to `dependencies`:
```json
"dependencies": {
  "@casehub/pages-data": "workspace:*"
}
```

- [ ] **Step 2: Update build order in root package.json**

In root `package.json`, change `build:packages` script — move `pages-data` before `pages-component`:
```json
"build:packages": "yarn workspace @casehub/pages-iframe-api run build && yarn workspace @casehub/pages-echarts-base run build && yarn workspace @casehub/pages-iframe-dev run build && yarn workspace @casehub/pages-data run build && yarn workspace @casehub/pages-component run build && yarn workspace @casehub/pages-ui run build && yarn workspace @casehub/pages-viz run build && yarn workspace @casehub/pages-runtime run build"
```

- [ ] **Step 3: Move displayer-types.ts**

Copy `packages/pages-ui/src/model/displayer-types.ts` to `packages/pages-component/src/model/displayer-types.ts`.

Update the import paths in the moved file. The file currently imports:
```typescript
import type { DataSetLookup } from "@casehub/pages-data/dist/dataset/lookup.js";
import type { ColumnSettings } from "@casehub/pages-data/dist/dataset/types.js";
import type { FilterSettings, RefreshSettings } from "./component-props.js";
```
These imports all resolve correctly in pages-component (pages-data is now a dependency, and component-props.ts is local).

Delete the original at `packages/pages-ui/src/model/displayer-types.ts`.

- [ ] **Step 4: Move form-input-types.ts**

Copy `packages/pages-ui/src/model/form-input-types.ts` to `packages/pages-component/src/model/form-input-types.ts`.

Update the import:
```typescript
import type { DataSetId } from "@casehub/pages-data/dist/dataset/types.js";
```
This resolves correctly (pages-data is now a dependency).

Delete the original at `packages/pages-ui/src/model/form-input-types.ts`.

- [ ] **Step 5: Extract page-props.ts from page-types.ts**

Create `packages/pages-component/src/model/page-props.ts` with the types that move:

```typescript
import type { DataSetId } from "@casehub/pages-data/dist/dataset/types.js";
import type { DataSetLookup } from "@casehub/pages-data/dist/dataset/lookup.js";
import type { DataSetOp } from "@casehub/pages-data/dist/dataset/ops.js";
import type {
  ExternalDataSetDef,
  ExternalColumnDef,
  HttpMethod,
} from "@casehub/pages-data/dist/dataset/external/types.js";
import type { ChartSettings } from "./displayer-types.js";

export interface DataScopeRef {
  readonly $ref: string;
}

export interface DataScope {
  readonly dataset: DataSetId;
  readonly idColumn: string;
  readonly filter?: Readonly<Record<string, string | DataScopeRef>>;
}

export interface SaveConfig {
  readonly trigger?: "auto" | "field" | "button" | "manual";
  readonly delay?: number;
  readonly adapter: string;
  readonly adapterConfig?: Readonly<Record<string, unknown>>;
}

export interface PageProps {
  readonly name?: string;
  readonly datasets?: readonly ExternalDataSetDef[];
  readonly settings?: PageSettings;
  readonly properties?: Readonly<Record<string, string>>;
  readonly dataScope?: DataScope;
  readonly save?: SaveConfig;
}

export interface PageSettings {
  readonly mode?: "light" | "dark";
  readonly allowUrlProperties?: boolean;
  readonly dataComponentDefaults?: DataComponentDefaults;
  readonly datasetDefaults?: DataSetDefaults;
}

export interface DataComponentDefaults {
  readonly lookup?: LookupDefaults;
  readonly chart?: Partial<ChartSettings>;
}

export interface LookupDefaults {
  readonly dataSetId?: DataSetId;
  readonly operations?: readonly DataSetOp[];
  readonly rowCount?: number;
  readonly rowOffset?: number;
}

export interface DataSetDefaults {
  readonly url?: string;
  readonly content?: string;
  readonly method?: HttpMethod;
  readonly headers?: Readonly<Record<string, string>>;
  readonly columns?: readonly ExternalColumnDef[];
  readonly cacheEnabled?: boolean;
  readonly refreshTime?: string;
}
```

- [ ] **Step 6: Update pages-ui/src/model/page-types.ts**

Remove the moved types. Keep Site, ViewState, DrillDownStep, LayoutOverride, DeepLink. Update imports:

```typescript
import type { Component, GridPlacement } from "@casehub/pages-component";
import type { DataSetId } from "@casehub/pages-data/dist/dataset/types.js";
import type { ExternalDataSetDef } from "@casehub/pages-data/dist/dataset/external/types.js";
export type {
  PageProps, PageSettings, DataComponentDefaults, LookupDefaults,
  DataSetDefaults, DataScope, DataScopeRef, SaveConfig,
} from "@casehub/pages-component";

// Runtime types stay here
export interface ViewState { ... }
export interface DrillDownStep { ... }
export interface LayoutOverride { ... }
export interface DeepLink { ... }
export interface Site { ... }
```

- [ ] **Step 7: Update pages-component/src/model/index.ts**

Add exports for the new modules:
```typescript
export * from "./displayer-types.js";
export * from "./form-input-types.js";
export * from "./page-props.js";
```

- [ ] **Step 8: Update pages-ui/src/model/index.ts**

Replace direct exports of moved types with re-exports from pages-component. The key change: instead of importing from local `./displayer-types.js` and `./form-input-types.js`, re-export from `@casehub/pages-component`. Pages-ui's public API stays the same — consumers see no difference.

- [ ] **Step 9: Update test imports**

In `packages/pages-ui/src/model/displayer-types.test.ts` and `form-input-types.test.ts`: update import paths from `./displayer-types.js` to `@casehub/pages-component` (or the re-export barrel — whichever matches the test's intent).

- [ ] **Step 10: Build and test**

Run: `yarn build:packages && yarn test`
Expected: All tests pass. Build succeeds with new dependency order.

- [ ] **Step 11: Commit**

```bash
git add packages/pages-component/ packages/pages-ui/src/model/ package.json
git commit -m "refactor: move props types from pages-ui to pages-component  Refs #1"
```

---

### Task 3: Generic Component + Unified ComponentTypeRegistry

Make Component generic `Component<T, P>`. Expand ComponentTypeRegistry to include all component types. Update type guards to return `TypedComponent<T>`. Eliminate the cast-widened `getProps` re-export in pages-ui.

**Files:**
- Modify: `packages/pages-component/src/model/types.ts` — make Component generic
- Modify: `packages/pages-component/src/model/type-guards.ts` — expand registry, update guards
- Modify: `packages/pages-ui/src/model/type-guards.ts` — remove split registry, re-export from pages-component
- Modify: `packages/pages-component/src/model/types.test.ts` — add tests for generic Component
- Modify: `packages/pages-component/src/model/type-guards.test.ts` — add tests for typed narrowing

**Interfaces:**
- Consumes: All props types (now in pages-component from Task 2)
- Produces: `Component<T, P>`, `ComponentType`, `TypedComponent<T>`, `isComponentType<T>()`, updated `getProps<T>()` (no cast)

- [ ] **Step 1: Write failing test for generic Component**

Add to `packages/pages-component/src/model/types.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import type { Component, TypedComponent } from "./types.js";
import type { BarChartProps } from "./displayer-types.js";
import type { GridProps } from "./component-props.js";

describe("Component<T, P>", () => {
  it("accepts typed props without cast", () => {
    const c: Component<"bar-chart", BarChartProps> = {
      type: "bar-chart",
      props: { lookup: { dataSetId: dataSetId("ds"), operations: [] } },
    };
    expect(c.type).toBe("bar-chart");
    expect(c.props?.lookup?.dataSetId).toBe("ds");
  });

  it("default generic accepts any type string", () => {
    const c: Component = { type: "anything" };
    expect(c.type).toBe("anything");
  });

  it("TypedComponent narrows both type and props", () => {
    const c: TypedComponent<"grid"> = {
      type: "grid",
      props: { columns: 12 },
    };
    expect(c.props?.columns).toBe(12);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/pages-component run test -- --reporter=verbose src/model/types.test.ts`
Expected: FAIL — `TypedComponent` not exported, `Component` doesn't accept type parameters

- [ ] **Step 3: Make Component generic**

Update `packages/pages-component/src/model/types.ts`:

```typescript
export interface Component<
  T extends string = string,
  P extends Record<string, unknown> = Record<string, unknown>,
> {
  readonly type: T;
  readonly id?: string;
  readonly props?: Readonly<P>;
  readonly style?: Readonly<Record<string, string>>;
  readonly access?: AccessControl;
  readonly slots?: Readonly<Record<string, readonly Component[]>>;
  readonly items?: readonly GridItem[];
}
```

- [ ] **Step 4: Write failing test for unified registry and type guard narrowing**

Add to `packages/pages-component/src/model/type-guards.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { isBarChart, isGrid, isComponentType, getProps } from "./type-guards.js";
import type { Component, TypedComponent } from "./types.js";
import { dataSetId } from "@casehub/pages-data/dist/dataset/types.js";

describe("type guards narrow to TypedComponent", () => {
  it("isBarChart narrows type and props", () => {
    const c: Component = {
      type: "bar-chart",
      props: { lookup: { dataSetId: dataSetId("ds"), operations: [] } },
    };
    if (isBarChart(c)) {
      // TypeScript knows c.type === "bar-chart" and c.props is BarChartProps
      expect(c.type).toBe("bar-chart");
    }
  });

  it("isComponentType narrows generically", () => {
    const c: Component = { type: "grid", props: { columns: 12 } };
    if (isComponentType(c, "grid")) {
      expect(c.type).toBe("grid");
    }
  });

  it("getProps returns typed props for all registry types", () => {
    const c: Component = {
      type: "bar-chart",
      props: { lookup: { dataSetId: dataSetId("ds"), operations: [] } },
    };
    const props = getProps(c, "bar-chart");
    expect(props.lookup.dataSetId).toBe("ds");
  });
});
```

- [ ] **Step 5: Expand ComponentTypeRegistry and update type guards**

Update `packages/pages-component/src/model/type-guards.ts`:

1. Import all props types (displayer, form, page) — they're now local
2. Expand `ComponentTypeRegistry` to include ALL component types
3. Add `ComponentType` and `TypedComponent<T>` type exports
4. Add generic `isComponentType<T>()` guard
5. Update every existing guard return type from `c is Component & { props: XProps }` to `c is TypedComponent<"x">`
6. Add new guards for data components and form inputs (moved from pages-ui)
7. `getProps` now works with the full registry — no cast needed

- [ ] **Step 6: Update pages-ui/src/model/type-guards.ts**

Remove the split registry definition. Remove the cast-widened `getProps` re-export. Replace with re-exports from pages-component:

```typescript
export {
  ComponentTypeRegistry,
  ComponentType,
  TypedComponent,
  getProps,
  isComponentType,
  isGrid, isColumns, isRows, isStack, isTabs, isPills, isSidebar,
  isTree, isMenu, isAccordion, isCarousel, isAppGrid, isPanel,
  isHtml, isMarkdown, isTitle, isLazyPage,
  isBarChart, isLineChart, isAreaChart, isPieChart, isScatterChart,
  isBubbleChart, isTimeseries, isTable, isMetric, isMeter,
  isSelector, isMap, isIframePlugin,
  isTextInput, isNumberInput, isDropdown, isCheckbox, isDatePicker, isTextarea,
} from "@casehub/pages-component";

// isPage stays here — PageProps is in pages-component but isPage is a pages-ui concern
// (page is a top-level parser concept, not a visual component)
export function isPage(c: Component): c is TypedComponent<"page"> {
  return c.type === "page";
}

// isFormInput stays here — utility combining multiple guards
const FORM_INPUT_TYPES = new Set(["text-input", "number-input", "dropdown", "checkbox", "date-picker", "textarea"]);
export function isFormInput(c: Component): boolean {
  return FORM_INPUT_TYPES.has(c.type);
}
```

- [ ] **Step 7: Run all tests**

Run: `yarn build:packages && yarn test`
Expected: All tests pass

- [ ] **Step 8: Commit**

```bash
git add packages/pages-component/src/model/ packages/pages-ui/src/model/type-guards.ts
git commit -m "feat: generic Component<T,P> with unified ComponentTypeRegistry  Refs #1"
```

---

### Task 4: Typed Builders + Layout Casts

Update DSL builders to return typed components, eliminating 20 production `as unknown as` casts in builders.ts and 2 in layout.ts. Update getProps (1 cast in type-guards.ts already eliminated in Task 3).

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts` — return TypedComponent<T> from each builder
- Modify: `packages/pages-component/src/renderer/layout.ts` — use getProps instead of casts
- Modify: `packages/pages-ui/src/dsl/builders.test.ts` — verify typed access without casts
- Modify: `packages/pages-ui/src/dsl/builders-displayers.test.ts` — verify typed access

**Interfaces:**
- Consumes: `Component<T, P>`, `TypedComponent<T>` from Task 3
- Produces: Each builder function (`grid()`, `barChart()`, `table()`, etc.) returns `TypedComponent<T>` instead of `Component`

- [ ] **Step 1: Write failing test for typed builder output**

Add to `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
it("grid() returns typed component with accessible props", () => {
  const g = grid(12);
  // This should compile without 'as any' — g.props is GridProps
  expect(g.props?.columns).toBe(12);
  expect(g.type).toBe("grid");
});

it("barChart() returns typed component with accessible props", () => {
  const chart = barChart(lookup(dataSetId("ds")));
  // This should compile without 'as any' — chart.props is BarChartProps
  expect(chart.props?.lookup.dataSetId).toBe("ds");
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/pages-ui run test -- --reporter=verbose src/dsl/builders.test.ts`
Expected: FAIL — type errors accessing props without cast (builders still return `Component`)

- [ ] **Step 3: Update builder return types**

In `packages/pages-ui/src/dsl/builders.ts`, update each builder function. Pattern for every builder:

```typescript
// Before
export function grid(columns: number, ...items: readonly GridItem[]): Component {
  const props: GridProps = { columns };
  return freeze({
    type: "grid",
    props: props as unknown as Record<string, unknown>,  // CAST
    items,
  });
}

// After
export function grid(columns: number, ...items: readonly GridItem[]): TypedComponent<"grid"> {
  const props: GridProps = { columns };
  return freeze({
    type: "grid" as const,
    props,
    items,
  });
}
```

The key changes in each builder:
1. Return type: `Component` → `TypedComponent<"grid">` (or appropriate type)
2. Remove `props as unknown as Record<string, unknown>` cast — assign props directly
3. Add `as const` to the `type` literal so TypeScript infers the literal type

Apply this pattern to all 20 builder functions: `page`, `grid`, `columns`, `panel`, `html`, `markdown`, `title`, `barChart`, `lineChart`, `areaChart`, `pieChart`, `scatterChart`, `bubbleChart`, `timeseries`, `table`, `metric`, `meter`, `selector`, `mapChart`, `iframePlugin`.

- [ ] **Step 4: Update layout.ts — use getProps instead of casts**

In `packages/pages-component/src/renderer/layout.ts`, replace:

```typescript
// Before
const gridProps = props as unknown as GridProps | undefined;
// ...
const colProps = props as unknown as ColumnsProps | undefined;

// After
import { isGrid, isColumns, getProps } from "../model/type-guards.js";
// Use the component's type to get properly typed props:
if (isGrid(component)) {
  const gridProps = component.props;
  // gridProps is now GridProps — no cast
}
```

- [ ] **Step 5: Run tests**

Run: `yarn build:packages && yarn test`
Expected: All tests pass. Zero `as unknown as` casts remain in builders.ts and layout.ts.

- [ ] **Step 6: Verify cast elimination**

Run: `grep -c "as unknown as" packages/pages-ui/src/dsl/builders.ts packages/pages-component/src/renderer/layout.ts packages/pages-component/src/model/type-guards.ts`
Expected: 0 in each file

- [ ] **Step 7: Commit**

```bash
git add packages/pages-ui/src/dsl/ packages/pages-component/src/renderer/layout.ts
git commit -m "refactor: typed builders eliminate 22 production casts  Refs #1"
```

---

### Task 5: Update pages-viz Imports + Remove pages-ui Dependency

Change pages-viz to import all type definitions from pages-component instead of pages-ui. Remove pages-ui from pages-viz's dependencies.

**Files:**
- Modify: `packages/pages-viz/package.json` — remove `@casehub/pages-ui` dependency, add `@casehub/pages-component`
- Modify: All `packages/pages-viz/src/**/*.ts` files that import from `@casehub/pages-ui` — change to `@casehub/pages-component`
- Modify: `packages/pages-viz/src/form-inputs/CasehubDropdown.ts` — import `isFixedOptions` from `@casehub/pages-component`

**Interfaces:**
- Consumes: All props types + `isFixedOptions` from `@casehub/pages-component` (moved in Task 2)
- Produces: No API changes — viz exports are unchanged

- [ ] **Step 1: Update pages-viz/package.json**

Replace `@casehub/pages-ui` with `@casehub/pages-component` in dependencies:

```json
"dependencies": {
  "@casehub/pages-component": "workspace:*",
  "@casehub/pages-data": "workspace:*",
  "echarts": "^5.6.0"
}
```

- [ ] **Step 2: Update all import statements**

In every pages-viz source file, change:
```typescript
// Before
import type { BarChartProps } from "@casehub/pages-ui/dist/model/displayer-types.js";
import type { FilterSettings } from "@casehub/pages-ui/dist/model/component-props.js";
import type { TextInputProps } from "@casehub/pages-ui";
import { isFixedOptions } from "@casehub/pages-ui";

// After
import type { BarChartProps } from "@casehub/pages-component";
import type { FilterSettings } from "@casehub/pages-component";
import type { TextInputProps } from "@casehub/pages-component";
import { isFixedOptions } from "@casehub/pages-component";
```

Files to update (check each for `@casehub/pages-ui` imports):
- `src/charts/CasehubBarChart.ts`
- `src/charts/CasehubLineChart.ts`
- `src/charts/CasehubAreaChart.ts`
- `src/charts/CasehubPieChart.ts`
- `src/charts/CasehubScatterChart.ts`
- `src/charts/CasehubBubbleChart.ts`
- `src/charts/CasehubTimeseries.ts`
- `src/charts/CasehubMeter.ts`
- `src/charts/CasehubMap.ts`
- `src/components/CasehubTable.ts`
- `src/components/CasehubMetric.ts`
- `src/components/CasehubSelector.ts`
- `src/form-inputs/CasehubFormInput.ts`
- `src/form-inputs/CasehubTextInput.ts`
- `src/form-inputs/CasehubNumberInput.ts`
- `src/form-inputs/CasehubDropdown.ts` (runtime import of `isFixedOptions`)
- `src/form-inputs/CasehubCheckbox.ts`
- `src/form-inputs/CasehubDatePicker.ts`
- `src/form-inputs/CasehubTextarea.ts`

- [ ] **Step 3: Build and test**

Run: `yarn build:packages && yarn test`
Expected: All tests pass. pages-viz no longer depends on pages-ui.

- [ ] **Step 4: Verify dependency elimination**

Run: `grep "pages-ui" packages/pages-viz/package.json`
Expected: No output

Run: `grep -r "@casehub/pages-ui" packages/pages-viz/src/`
Expected: No output

- [ ] **Step 5: Commit**

```bash
git add packages/pages-viz/
git commit -m "refactor: pages-viz imports from pages-component, drops pages-ui dependency  Refs #1"
```

---

### Task 6: Eliminate All Test `as any` Casts

Work through all 119 test `as any` instances using the type infrastructure from Tasks 1–5.

**Files:**
- Modify: 21 test files across pages-data, pages-ui, pages-runtime, pages-viz (see per-file list in spec)
- Create: `packages/pages-viz/src/custom-elements.d.ts` — HTMLElementTagNameMap declarations
- Create: `packages/pages-data/src/dataset/test-helpers.ts` — test assertion helpers for DataSetOp narrowing

**Interfaces:**
- Consumes: `dataSetId()`, `columnId()` (Task 1), `TypedComponent<T>` (Task 3), typed builders (Task 4)
- Produces: Zero `as any` across all test files

- [ ] **Step 1: Create HTMLElementTagNameMap declarations**

```typescript
// packages/pages-viz/src/custom-elements.d.ts
import type { CasehubTable } from "./components/CasehubTable.js";
import type { CasehubMetric } from "./components/CasehubMetric.js";
import type { CasehubSelector } from "./components/CasehubSelector.js";
import type { CasehubBarChart } from "./charts/CasehubBarChart.js";
import type { CasehubLineChart } from "./charts/CasehubLineChart.js";
import type { CasehubAreaChart } from "./charts/CasehubAreaChart.js";
import type { CasehubPieChart } from "./charts/CasehubPieChart.js";
import type { CasehubScatterChart } from "./charts/CasehubScatterChart.js";
import type { CasehubBubbleChart } from "./charts/CasehubBubbleChart.js";
import type { CasehubTimeseries } from "./charts/CasehubTimeseries.js";
import type { CasehubMeter } from "./charts/CasehubMeter.js";
import type { CasehubMap } from "./charts/CasehubMap.js";

declare global {
  interface HTMLElementTagNameMap {
    "casehub-table": CasehubTable;
    "casehub-metric": CasehubMetric;
    "casehub-selector": CasehubSelector;
    "casehub-bar-chart": CasehubBarChart;
    "casehub-line-chart": CasehubLineChart;
    "casehub-area-chart": CasehubAreaChart;
    "casehub-pie-chart": CasehubPieChart;
    "casehub-scatter-chart": CasehubScatterChart;
    "casehub-bubble-chart": CasehubBubbleChart;
    "casehub-timeseries": CasehubTimeseries;
    "casehub-meter": CasehubMeter;
    "casehub-map": CasehubMap;
  }
}
```

- [ ] **Step 2: Create DataSetOp test helpers**

```typescript
// packages/pages-data/src/dataset/test-helpers.ts
import type { DataSetOp, GroupOp, FilterOp, SortOp } from "./ops.js";
import { expect } from "vitest";

export function expectGroupOp(op: DataSetOp): GroupOp {
  expect(op.type).toBe("group");
  return op as GroupOp;
}

export function expectFilterOp(op: DataSetOp): FilterOp {
  expect(op.type).toBe("filter");
  return op as FilterOp;
}

export function expectSortOp(op: DataSetOp): SortOp {
  expect(op.type).toBe("sort");
  return op as SortOp;
}
```

- [ ] **Step 3: Fix branded type construction (22 instances)**

In every test file that uses `"ds1" as DataSetId` or `"col" as ColumnId`, replace with factory functions:

```typescript
// Before
const lookup = { dataSetId: "ds1" as DataSetId, operations: [] };

// After
import { dataSetId, columnId } from "@casehub/pages-data/dist/dataset/types.js";
const lookup = { dataSetId: dataSetId("ds1"), operations: [] };
```

Files: `displayer-types.test.ts` (16), `page-types.test.ts` (3), `form-input-types.test.ts` (1), `option-pipeline.test.ts` (2).

- [ ] **Step 4: Fix union narrowing (22 instances)**

In test files that access DataSetOp-specific fields:

```typescript
// Before
const groupOp = lookup.operations[0] as any;
expect(groupOp.columns[0].kind).toBe("key");

// After
import { expectGroupOp } from "@casehub/pages-data/dist/dataset/test-helpers.js";
const groupOp = expectGroupOp(lookup.operations[0]);
expect(groupOp.columns[0].kind).toBe("key");
```

Files: `lookup-helpers.test.ts` (16), `lookup-parser.test.ts` (6).

- [ ] **Step 5: Fix component type erasure (59 instances)**

In test files that access `(result.props as any).margin`, the fix depends on how the result is produced:

**For builder outputs** (builders.test.ts, builders-displayers.test.ts — 4 instances):
Builders now return `TypedComponent<T>` — access props directly:
```typescript
// Before
expect((barChart(lookup(dataSetId("ds"))).props as any).lookup).toBeDefined();
// After
expect(barChart(lookup(dataSetId("ds"))).props?.lookup).toBeDefined();
```

**For parser outputs** (displayer-desugar.test.ts, form-desugar.test.ts, page-parser.test.ts, parse-boundaries.test.ts, property-substitution.test.ts — 55 instances):
Use type guards or `getProps` to narrow the parsed result:
```typescript
// Before
const result = parseDisplayer(yaml);
expect((result.props as any).margin).toEqual({ left: 80 });

// After
import { getProps } from "@casehub/pages-component";
const result = parseDisplayer(yaml);
const props = getProps(result, "bar-chart");
expect(props.margin).toEqual({ left: 80 });
```

Or if the parser function's return type is updated to return `TypedComponent<T>`:
```typescript
const result = parseDisplayer(yaml); // returns TypedComponent<"bar-chart">
expect(result.props?.margin).toEqual({ left: 80 });
```

- [ ] **Step 6: Fix DOM access (6 instances)**

With HTMLElementTagNameMap declared (Step 1):
```typescript
// Before
const tableEl = target.querySelector("casehub-table") as any;
// After
const tableEl = target.querySelector("casehub-table");
// tableEl is CasehubTable | null
```

Files: `form-edit.test.ts` (2), `form-equivalence.test.ts` (2), `form-integration.test.ts` (2).

- [ ] **Step 7: Fix mock functions (5 instances)**

```typescript
// Before
const mockFetch = vi.fn().mockResolvedValue({ json: () => data });
restAdapter.fetch(url, mockFetch as any);

// After — type the mock to match the expected signature
const mockFetch = vi.fn<[string, RequestInit?], Promise<Response>>()
  .mockResolvedValue({ json: () => Promise.resolve(data) } as Response);
restAdapter.fetch(url, mockFetch);
```

File: `rest-adapter.test.ts` (5 instances).

- [ ] **Step 8: Fix remaining (5 instances)**

- `json-resilience.test.ts` (1), `dataset-scope.test.ts` (1), `data-pipeline.test.ts` (1) — use test fixture factories or complete the partial objects
- `registry.test.ts` (2) — replace `(obj as any).register` with `"register" in obj`

- [ ] **Step 9: Verify zero as-any**

Run: `grep -rn "as any" packages/*/src/ components/*/src/ examples/src/ --include="*.ts" --include="*.tsx" | grep -v node_modules | grep -v _legacy`
Expected: Zero results

- [ ] **Step 10: Run full test suite**

Run: `yarn build:packages && yarn test`
Expected: All tests pass

- [ ] **Step 11: Commit**

```bash
git add packages/ components/
git commit -m "refactor: eliminate all 119 test as-any casts  Refs #1"
```

---

### Task 7: tsconfig Upgrade, Split, and CI Enforcement

Upgrade base tsconfig to maximum strict. Upgrade legacy TypeScript versions from 4.6.2 to 5.6.0. Add tsconfig split (tsconfig.json for checking, tsconfig.build.json for emit). Add root `typecheck` script. Update CI workflow.

**Files:**
- Modify: `packages/pages-tsconfig/tsconfig.json` — maximum strict
- Modify: All 12 package `tsconfig.json` files — extend base, add tsconfig.build.json where needed
- Modify: 5 legacy `package.json` files — upgrade TypeScript to ^5.6.0
- Modify: Root `package.json` — add `typecheck` script
- Modify: `.github/workflows/ci-javascript.yml` — add typecheck step
- Create: `packages/pages-data/tsconfig.build.json` (and similar for ui, viz, component, runtime, iframe-api, iframe-dev, echarts-base)

**Interfaces:**
- Consumes: All type fixes from Tasks 1–6
- Produces: `yarn typecheck` command that passes, CI enforcement

- [ ] **Step 1: Upgrade base tsconfig**

Replace `packages/pages-tsconfig/tsconfig.json`:

```json
{
  "exclude": ["node_modules"],
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM"],
    "jsx": "react-jsx"
  }
}
```

- [ ] **Step 2: Upgrade legacy TypeScript versions**

In each of these package.json files, change `"typescript": "^4.6.2"` to `"typescript": "^5.6.0"`:
- `packages/pages-iframe-api/package.json`
- `packages/pages-iframe-dev/package.json`
- `packages/pages-echarts-base/package.json`
- `components/pages-component-echarts/package.json`
- `components/pages-component-svg-heatmap/package.json`

Run: `yarn install` to update lock file.

- [ ] **Step 3: Update core package tsconfigs — extend base + tsconfig split**

For each of the 5 core packages (pages-data, pages-ui, pages-viz, pages-component, pages-runtime), replace the standalone tsconfig with base extension:

**`tsconfig.json`** (type-checking authority — includes tests):
```json
{
  "extends": "@casehub/pages-tsconfig/tsconfig.json",
  "include": ["src"]
}
```

**`tsconfig.build.json`** (build specialization — excludes tests):
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist"
  },
  "exclude": ["**/*.test.ts"]
}
```

Update each package.json `build` script from `"build": "tsc"` to `"build": "tsc -p tsconfig.build.json"`.

Add `"typecheck": "tsc --noEmit"` to each package.json scripts.

- [ ] **Step 4: Update legacy package tsconfigs — simplify**

For each of the 5 base-extending packages, simplify to minimal overrides:

```json
{
  "extends": "@casehub/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "outDir": "dist"
  },
  "include": ["src"]
}
```

For iframe components that need ES6 target (component-echarts, component-svg-heatmap):
```json
{
  "extends": "@casehub/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "target": "es6",
    "declaration": false,
    "declarationMap": false
  },
  "include": ["src"]
}
```

Update build scripts: add `"typecheck": "tsc --noEmit"` to each.

For tsc-compiled packages (iframe-api, iframe-dev, echarts-base), add tsconfig.build.json and update build script to use it.

- [ ] **Step 5: Update standalone packages (llm-prompter, examples)**

Replace standalone tsconfigs with base extension:

**pages-component-llm-prompter:**
```json
{
  "extends": "@casehub/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "target": "es6",
    "declaration": false,
    "declarationMap": false,
    "noEmit": true
  },
  "include": ["src"]
}
```

**examples:**
```json
{
  "extends": "@casehub/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "noEmit": true
  },
  "include": ["src"]
}
```

- [ ] **Step 6: Fix verbatimModuleSyntax migration**

Audit all source files for type-only imports that need `import type`:

```typescript
// Before
import { SomeType } from "./module.js";

// After (if SomeType is only used as a type)
import type { SomeType } from "./module.js";
```

This is mechanical — TypeScript will report errors for each import that needs changing.

- [ ] **Step 7: Fix any remaining strict errors in legacy packages**

Run `yarn typecheck` and fix errors in the 5 legacy packages. Common fixes:
- `strictPropertyInitialization` — add definite assignment assertions or initialize in constructor
- `strictFunctionTypes` — fix contravariant callback signatures
- `alwaysStrict` — should have no impact (already in strict-mode-implied modules)

- [ ] **Step 8: Add root typecheck script**

In root `package.json`, add:
```json
"typecheck": "yarn workspaces foreach -Apt run typecheck"
```

- [ ] **Step 9: Run root typecheck**

Run: `yarn typecheck`
Expected: All packages pass — zero type errors

- [ ] **Step 10: Update CI workflow**

In `.github/workflows/ci-javascript.yml`, add typecheck step after install and before build:

```yaml
      - name: Type check all packages
        run: yarn typecheck

      - name: Build all workspace packages
        run: yarn build
```

- [ ] **Step 11: Full build and test**

Run: `yarn build && yarn test`
Expected: Everything passes

- [ ] **Step 12: Commit**

```bash
git add packages/ components/ examples/ .github/ package.json yarn.lock
git commit -m "feat: maximum strict tsconfig, tsconfig split, CI type enforcement  Refs #1"
```

---

## Self-Review Checklist

1. **Spec coverage:** All 10 spec items mapped to tasks. Type migration (Tasks 2, 5), generic Component (Task 3), unified registry (Task 3), branded constructors (Task 1), builder casts (Task 4), test as-any (Task 6), tsconfig (Task 7), CI (Task 7), HTMLElementTagNameMap (Task 6), build order (Task 2).

2. **Placeholder scan:** No TBD/TODO. Step 5 of Task 3 says "update every existing guard" — the specific guards and their new signatures are enumerated in the spec. Step 5 of Task 6 shows `getProps` pattern for parser outputs — the parser return type update is implicit in the typed builder pattern.

3. **Type consistency:** `TypedComponent<T>` used consistently across Tasks 3–6. `ComponentTypeRegistry` naming matches existing codebase (not `ComponentTypeMap` from earlier drafts). `dataSetId()`/`columnId()` from Task 1 used in Tasks 6.
