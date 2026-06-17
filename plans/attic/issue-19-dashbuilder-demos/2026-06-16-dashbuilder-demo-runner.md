# DashBuilder Demo Runner Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Render all 32 DashBuilder sample YAML dashboards in the existing examples gallery using the pure TypeScript casehub runtime instead of the GWT iframe.

**Architecture:** Extend the examples gallery with a webpack bundle that packages `@casehub/runtime`, `@casehub/viz`, and `js-yaml`. Replace the iframe rendering in `app.js` with direct `loadSite()` calls. Add axis/grid desugaring (#21) and markdown parsing (#14) to support all dashboard features. Validate all 32 samples (#22).

**Tech Stack:** TypeScript, Webpack 5, Vitest, js-yaml, marked, ECharts (via @casehub/viz)

**Note:** `loadSite()` accepts `string | Component` but `parsePage()` throws on strings — the demo runner must parse YAML to a JS object first, then pass the object to `loadSite()`.

---

## Task 1: Fix loadSite string handling (#20 prerequisite)

`loadSite()` declares `source: string | Component` but passes strings directly to `parsePage()` which throws `"Invalid input: expected an object"`. Fix this so the type signature is honest.

**Files:**
- Modify: `packages/casehub-runtime/src/site.ts:41`
- Modify: `packages/casehub-runtime/package.json`
- Test: `packages/casehub-runtime/src/site.test.ts`

- [ ] **Step 1: Write failing test — loadSite with YAML string**

Add to `packages/casehub-runtime/src/site.test.ts`:

```typescript
it("accepts a YAML string as source", async () => {
  const yaml = `
pages:
  - components:
      - html: "<p>hello</p>"
`;
  const target = document.createElement("div");
  const site = await loadSite(target, yaml);
  expect(site.root.type).toBe("page");
  expect(target.innerHTML).toContain("hello");
  site.dispose();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehub/runtime run test -- site.test.ts`
Expected: FAIL — `parsePage` throws "Invalid input: expected an object"

- [ ] **Step 3: Add js-yaml dependency to runtime**

Add `js-yaml` as a production dependency in `packages/casehub-runtime/package.json`:

```json
"dependencies": {
  "@casehub/component": "workspace:*",
  "@casehub/data": "workspace:*",
  "@casehub/ui": "workspace:*",
  "@casehub/viz": "workspace:*",
  "js-yaml": "^4.1.0"
}
```

Also add the type stubs as devDependency:

```json
"devDependencies": {
  ...existing...,
  "@types/js-yaml": "^4.0.9"
}
```

Run: `yarn install`

- [ ] **Step 4: Fix loadSite to parse YAML strings**

In `packages/casehub-runtime/src/site.ts`, add the import and fix line 41:

```typescript
import { load as yamlLoad } from "js-yaml";
```

Replace line 41:
```typescript
const root = typeof source === "string" ? parsePage(source) : source;
```
With:
```typescript
const root = typeof source === "string" ? parsePage(yamlLoad(source) as Record<string, unknown>) : source;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehub/runtime run test -- site.test.ts`
Expected: PASS

- [ ] **Step 6: Run full runtime test suite**

Run: `yarn workspace @casehub/runtime run test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/casehub-runtime/src/site.ts packages/casehub-runtime/package.json packages/casehub-runtime/src/site.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "fix(@casehub/runtime): parse YAML strings in loadSite

loadSite() declared source: string | Component but parsePage() threw on
strings. Add js-yaml dependency and parse strings before passing to
parsePage().

Refs #20"
```

---

## Task 2: Webpack bundle for examples gallery (#20)

Create the webpack config and TypeScript entry point that bundles the casehub runtime for browser use.

**Files:**
- Create: `examples/src/casehub-entry.ts`
- Create: `examples/webpack.config.js`
- Create: `examples/tsconfig.json`
- Modify: `examples/package.json`

- [ ] **Step 1: Create TypeScript entry point**

Create `examples/src/casehub-entry.ts`:

```typescript
import "@casehub/viz";
import { loadSite } from "@casehub/runtime";
import type { LiveSite, SiteOptions } from "@casehub/runtime";

export { loadSite };
export type { LiveSite, SiteOptions };
```

- [ ] **Step 2: Create tsconfig.json for examples**

Create `examples/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 3: Create webpack.config.js**

Create `examples/webpack.config.js`:

```javascript
const path = require("path");
const commonConfig = require("@melviz/webpack-base");

module.exports = (env = {}) => {
  const common = commonConfig({ dev: !!env.dev });

  return {
    ...common,
    entry: {
      "casehub-bundle": path.resolve(__dirname, "src/casehub-entry.ts"),
    },
    output: {
      path: path.resolve(__dirname, "dist"),
      filename: "[name].js",
      library: {
        name: "casehub",
        type: "umd",
      },
      globalObject: "this",
    },
    resolve: {
      ...common.resolve,
      alias: {
        "@casehub/runtime": path.resolve(__dirname, "../packages/casehub-runtime/dist"),
        "@casehub/viz": path.resolve(__dirname, "../packages/casehub-viz/dist"),
        "@casehub/ui": path.resolve(__dirname, "../packages/casehub-ui/dist"),
        "@casehub/component": path.resolve(__dirname, "../packages/casehub-component/dist"),
        "@casehub/data": path.resolve(__dirname, "../packages/core/dist"),
      },
    },
  };
};
```

- [ ] **Step 4: Update examples/package.json**

Update `examples/package.json` — add webpack deps, casehub deps, and build:bundle script:

```json
{
  "name": "@melviz/examples",
  "version": "0.0.0",
  "description": "Melviz Dashboard Examples Gallery",
  "private": true,
  "scripts": {
    "clean": "rimraf dist",
    "generate-samples": "node scripts/generate-samples.js",
    "copy-dashboards": "node scripts/copy-dashboards.js",
    "build:bundle": "webpack --env prod",
    "build:bundle:dev": "webpack --env dev",
    "build": "yarn clean && yarn generate-samples && yarn build:bundle && yarn copy-dashboards",
    "serve": "npx http-server dist -c-1 -p 8080 -o",
    "dev": "node scripts/dev-server.js"
  },
  "dependencies": {
    "@casehub/runtime": "workspace:*",
    "@casehub/viz": "workspace:*",
    "js-yaml": "^4.1.0"
  },
  "devDependencies": {
    "browser-sync": "^2.29.3",
    "chokidar": "^3.5.3",
    "http-server": "^14.1.1",
    "rimraf": "^6.1.0",
    "ts-loader": "^9.5.0",
    "typescript": "^5.6.0",
    "webpack": "^5.90.0",
    "webpack-cli": "^5.1.0",
    "source-map-loader": "^5.0.0",
    "@melviz/webpack-base": "0.0.0"
  },
  "keywords": ["melviz", "dashboard", "examples"],
  "license": "Apache-2.0"
}
```

Note: removed `copy-melviz` from build (no longer copying GWT output) and removed `@melviz/webapp` dependency.

- [ ] **Step 5: Install dependencies and test webpack build**

Run: `yarn install && yarn workspace @melviz/examples run build:bundle:dev`
Expected: `dist/casehub-bundle.js` created without errors

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add examples/src/casehub-entry.ts examples/webpack.config.js examples/tsconfig.json examples/package.json
git -C /Users/mdproctor/claude/melviz commit -m "feat(examples): webpack bundle for casehub runtime

Creates casehub-bundle.js that packages @casehub/runtime, @casehub/viz,
echarts, and js-yaml into a single UMD bundle for browser use.

Refs #20"
```

---

## Task 3: Replace iframe with loadSite in gallery (#20)

Modify the gallery HTML and JS to render dashboards via `loadSite()` instead of the GWT iframe.

**Files:**
- Modify: `examples/src/index.html:64-66`
- Modify: `examples/src/app.js`
- Modify: `examples/src/styles.css`

- [ ] **Step 1: Replace iframe with div in index.html**

In `examples/src/index.html`, replace line 66:
```html
<iframe id="dashboard-iframe" src="melviz-webapp" frameborder="0"></iframe>
```
With:
```html
<div id="dashboard-target"></div>
```

Add script tag before `app.js` (before line 78):
```html
<script src="casehub-bundle.js"></script>
```

- [ ] **Step 2: Rewrite dashboard loading in app.js**

Replace the `dashboardIframe` DOM reference (line 10):
```javascript
const dashboardTarget = document.getElementById('dashboard-target');
```

Replace `loadDashboardInIframe` function (lines 152-155) with:
```javascript
let currentSite = null;

async function loadDashboardInTarget(dashboardPath) {
    try {
        const response = await fetch(`dashboards/${dashboardPath}`);
        const yamlText = await response.text();

        if (currentSite) {
            currentSite.dispose();
            currentSite = null;
        }

        dashboardTarget.innerHTML = "";
        dashboardTarget.className = "";

        currentSite = await window.casehub.loadSite(dashboardTarget, yamlText);
    } catch (error) {
        console.error('Error loading dashboard:', error);
        dashboardTarget.innerHTML = `
            <div style="padding: 24px; color: #d32f2f; background: #fce4ec; border-radius: 8px; margin: 16px;">
                <strong>Error loading dashboard</strong>
                <p style="margin-top: 8px; font-family: monospace; font-size: 13px;">${error.message || error}</p>
            </div>
        `;
    }
}
```

Update `loadDashboard` function (line 131) — change `loadDashboardInIframe` to `loadDashboardInTarget`:
```javascript
loadDashboardInTarget(dashboard.path);
```

- [ ] **Step 3: Add CSS for dashboard target**

Append to `examples/src/styles.css`:

```css
#dashboard-target {
    flex: 1;
    overflow: auto;
    padding: 16px;
    min-width: 0;
    min-height: 0;
}
```

Replace the `#dashboard-iframe` rule (lines 298-303) — it's no longer needed.

- [ ] **Step 4: Update copy-dashboards.js to also copy HTML and CSS**

Read `examples/scripts/copy-dashboards.js` to understand what it copies, then ensure it also copies `src/index.html`, `src/app.js`, and `src/styles.css` to `dist/`. If not already handled, add:

```javascript
fs.copyFileSync(
  path.join(__dirname, '../src/index.html'),
  path.join(distDir, 'index.html')
);
fs.copyFileSync(
  path.join(__dirname, '../src/app.js'),
  path.join(distDir, 'app.js')
);
fs.copyFileSync(
  path.join(__dirname, '../src/styles.css'),
  path.join(distDir, 'styles.css')
);
```

- [ ] **Step 5: Build and test the gallery**

Run: `yarn build:packages && yarn workspace @melviz/examples run build`
Then: `yarn workspace @melviz/examples run serve`

Open `http://localhost:8080` in browser. Click "Simple Chart" in sidebar. Verify chart renders in the div (not iframe).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add examples/src/index.html examples/src/app.js examples/src/styles.css examples/scripts/copy-dashboards.js
git -C /Users/mdproctor/claude/melviz commit -m "feat(examples): replace GWT iframe with casehub loadSite

Gallery now renders dashboards via loadSite() into a div instead of
loading the GWT webapp in an iframe. Error handling shows inline banners.

Closes #20"
```

---

## Task 4: Axis settings desugaring (#21)

Extract axis label angle, axis label visibility, and axis title from DashBuilder YAML into ChartSettings props.

**Files:**
- Modify: `packages/casehub-ui/src/model/displayer-types.ts:34`
- Modify: `packages/casehub-ui/src/parser/displayer-desugar.ts:117`
- Test: `packages/casehub-ui/src/parser/displayer-desugar.test.ts`

- [ ] **Step 1: Write failing tests for axis extraction**

Add to `packages/casehub-ui/src/parser/displayer-desugar.test.ts`:

```typescript
it("extracts top-level axis.x settings", () => {
  const result = desugarDisplayer({
    type: "BARCHART",
    axis: { x: { labels_angle: 30, title: "Month" } },
    lookup: { uuid: "data" },
  });
  expect((result.props as any).xAxis).toEqual({
    labelAngle: 30,
    title: "Month",
  });
});

it("extracts top-level axis.y settings", () => {
  const result = desugarDisplayer({
    type: "BARCHART",
    axis: { y: { title: "Revenue", labels_show: false } },
    lookup: { uuid: "data" },
  });
  expect((result.props as any).yAxis).toEqual({
    title: "Revenue",
    showLabels: false,
  });
});

it("extracts axis from chart.axis (nested)", () => {
  const result = desugarDisplayer({
    type: "LINECHART",
    chart: { resizable: true, axis: { x: { labels_angle: 10 } } },
    lookup: { uuid: "data" },
  });
  expect(result.props?.["resizable"]).toBe(true);
  expect((result.props as any).xAxis).toEqual({ labelAngle: 10 });
});

it("prefers top-level axis over chart.axis", () => {
  const result = desugarDisplayer({
    type: "BARCHART",
    axis: { x: { labels_angle: 30 } },
    chart: { axis: { x: { labels_angle: 10 } } },
    lookup: { uuid: "data" },
  });
  expect((result.props as any).xAxis).toEqual({ labelAngle: 30 });
});

it("extracts both x and y axis simultaneously", () => {
  const result = desugarDisplayer({
    type: "BARCHART",
    axis: {
      x: { labels_angle: 30, title: "X" },
      y: { title: "Y", labels_show: false },
    },
    lookup: { uuid: "data" },
  });
  expect((result.props as any).xAxis).toEqual({ labelAngle: 30, title: "X" });
  expect((result.props as any).yAxis).toEqual({ title: "Y", showLabels: false });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/ui run test -- displayer-desugar.test.ts`
Expected: 5 new tests FAIL

- [ ] **Step 3: Add axis extraction to displayer-desugar.ts**

In `packages/casehub-ui/src/parser/displayer-desugar.ts`, after the chart settings block (after line 117), add:

```typescript
  // Extract axis settings (top-level axis takes precedence over chart.axis)
  const chart = raw.chart as Record<string, unknown> | undefined;
  const axisSource = (raw.axis ?? chart?.axis) as Record<string, unknown> | undefined;
  if (axisSource && typeof axisSource === "object") {
    const xRaw = axisSource.x as Record<string, unknown> | undefined;
    if (xRaw && typeof xRaw === "object") {
      const xAxis: Record<string, unknown> = {};
      if (xRaw.title != null) xAxis.title = xRaw.title;
      if (xRaw.labels_show != null) xAxis.showLabels = xRaw.labels_show;
      if (xRaw.labels_angle != null) xAxis.labelAngle = xRaw.labels_angle;
      if (Object.keys(xAxis).length > 0) props.xAxis = xAxis;
    }

    const yRaw = axisSource.y as Record<string, unknown> | undefined;
    if (yRaw && typeof yRaw === "object") {
      const yAxis: Record<string, unknown> = {};
      if (yRaw.title != null) yAxis.title = yRaw.title;
      if (yRaw.labels_show != null) yAxis.showLabels = yRaw.labels_show;
      if (yRaw.labels_angle != null) yAxis.labelAngle = yRaw.labels_angle;
      if (Object.keys(yAxis).length > 0) props.yAxis = yAxis;
    }
  }
```

- [ ] **Step 4: Extend ChartSettings type**

In `packages/casehub-ui/src/model/displayer-types.ts`, replace line 34:
```typescript
readonly xAxis?: { readonly title?: string; readonly showLabels?: boolean };
readonly yAxis?: { readonly title?: string; readonly showLabels?: boolean };
```
With:
```typescript
readonly xAxis?: { readonly title?: string; readonly showLabels?: boolean; readonly labelAngle?: number };
readonly yAxis?: { readonly title?: string; readonly showLabels?: boolean; readonly labelAngle?: number };
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehub/ui run test -- displayer-desugar.test.ts`
Expected: All tests pass including 5 new ones

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/casehub-ui/src/parser/displayer-desugar.ts packages/casehub-ui/src/parser/displayer-desugar.test.ts packages/casehub-ui/src/model/displayer-types.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat(@casehub/ui): extract axis settings from DashBuilder YAML

Desugars displayer.axis.x/y (labels_angle, labels_show, title) into
xAxis/yAxis props. Supports both top-level axis and chart.axis nested
form.

Refs #21"
```

---

## Task 5: Grid visibility desugaring (#21)

Extract grid line visibility settings from `chart.grid.x/y`.

**Files:**
- Modify: `packages/casehub-ui/src/model/displayer-types.ts:19`
- Modify: `packages/casehub-ui/src/parser/displayer-desugar.ts`
- Test: `packages/casehub-ui/src/parser/displayer-desugar.test.ts`

- [ ] **Step 1: Write failing tests for grid extraction**

Add to `packages/casehub-ui/src/parser/displayer-desugar.test.ts`:

```typescript
it("extracts chart.grid visibility settings", () => {
  const result = desugarDisplayer({
    type: "BARCHART",
    chart: { grid: { x: false, y: false } },
    lookup: { uuid: "data" },
  });
  expect((result.props as any).grid).toEqual({ x: false, y: false });
});

it("extracts chart.grid with only x", () => {
  const result = desugarDisplayer({
    type: "LINECHART",
    chart: { grid: { x: false } },
    lookup: { uuid: "data" },
  });
  expect((result.props as any).grid).toEqual({ x: false });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/ui run test -- displayer-desugar.test.ts`
Expected: 2 new tests FAIL

- [ ] **Step 3: Add grid extraction to displayer-desugar.ts**

After the axis extraction code added in Task 4, add:

```typescript
  // Extract grid visibility (chart.grid.x/y controls splitLine show)
  const gridSource = (raw.chart as Record<string, unknown> | undefined)?.grid as Record<string, unknown> | undefined;
  if (gridSource && typeof gridSource === "object") {
    const grid: Record<string, unknown> = {};
    if (gridSource.x != null) grid.x = gridSource.x;
    if (gridSource.y != null) grid.y = gridSource.y;
    if (Object.keys(grid).length > 0) props.grid = grid;
  }
```

- [ ] **Step 4: Add grid to ChartSettings type**

In `packages/casehub-ui/src/model/displayer-types.ts`, add after the `yAxis` line (line 35):

```typescript
readonly grid?: { readonly x?: boolean; readonly y?: boolean };
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehub/ui run test -- displayer-desugar.test.ts`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/casehub-ui/src/parser/displayer-desugar.ts packages/casehub-ui/src/parser/displayer-desugar.test.ts packages/casehub-ui/src/model/displayer-types.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat(@casehub/ui): extract grid visibility from DashBuilder YAML

Desugars chart.grid.x/y into grid props for controlling splitLine
visibility in echarts.

Refs #21"
```

---

## Task 6: Apply axis labelAngle and grid visibility in option pipeline (#21)

Wire the new ChartSettings fields through to echarts options.

**Files:**
- Modify: `packages/casehub-viz/src/charts/option-pipeline.ts:84-111`
- Test: `packages/casehub-viz/src/charts/option-pipeline.test.ts`

- [ ] **Step 1: Write failing tests for labelAngle and grid in option pipeline**

Add to `packages/casehub-viz/src/charts/option-pipeline.test.ts`:

```typescript
it("applies xAxis labelAngle as axisLabel.rotate", () => {
  const option = {};
  const props = { xAxis: { labelAngle: 30 } };

  const result = applyChartSettings(option, props);

  expect(result.xAxis).toEqual({ axisLabel: { rotate: 30 } });
});

it("applies yAxis labelAngle as axisLabel.rotate", () => {
  const option = {};
  const props = { yAxis: { labelAngle: -10 } };

  const result = applyChartSettings(option, props);

  expect(result.yAxis).toEqual({ axisLabel: { rotate: -10 } });
});

it("merges labelAngle with existing axisLabel settings", () => {
  const option = {};
  const props = { xAxis: { showLabels: true, labelAngle: 30 } };

  const result = applyChartSettings(option, props);

  expect(result.xAxis).toEqual({ axisLabel: { show: true, rotate: 30 } });
});

it("applies grid.x false as xAxis.splitLine.show false", () => {
  const option = {};
  const props = { grid: { x: false } };

  const result = applyChartSettings(option, props);

  expect(result.xAxis).toEqual({ splitLine: { show: false } });
});

it("applies grid.y false as yAxis.splitLine.show false", () => {
  const option = {};
  const props = { grid: { y: false } };

  const result = applyChartSettings(option, props);

  expect(result.yAxis).toEqual({ splitLine: { show: false } });
});

it("applies grid visibility with existing axis settings", () => {
  const option = {};
  const props = {
    xAxis: { title: "Month", labelAngle: 30 },
    grid: { x: false, y: false },
  };

  const result = applyChartSettings(option, props);

  expect(result.xAxis).toEqual({
    name: "Month",
    axisLabel: { rotate: 30 },
    splitLine: { show: false },
  });
  expect(result.yAxis).toEqual({ splitLine: { show: false } });
});

it("skips grid settings when cartesianAxes is false", () => {
  const option = {};
  const props = { grid: { x: false, y: false } };

  const result = applyChartSettings(option, props, { cartesianAxes: false });

  expect(result.xAxis).toBeUndefined();
  expect(result.yAxis).toBeUndefined();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehub/viz run test -- option-pipeline.test.ts`
Expected: 7 new tests FAIL

- [ ] **Step 3: Add labelAngle and grid handling to applyChartSettings**

In `packages/casehub-viz/src/charts/option-pipeline.ts`, inside `applyChartSettings`, after the existing xAxis block (after line 96), add labelAngle handling:

```typescript
  // X-Axis label angle
  if (withAxes && props.xAxis?.labelAngle != null) {
    const xAxis: Record<string, unknown> = { ...((option.xAxis as Record<string, unknown>) || {}) };
    const existing = (xAxis.axisLabel as Record<string, unknown>) || {};
    xAxis.axisLabel = { ...existing, rotate: props.xAxis.labelAngle };
    option.xAxis = xAxis;
  }
```

After the existing yAxis block (after line 111), add:

```typescript
  // Y-Axis label angle
  if (withAxes && props.yAxis?.labelAngle != null) {
    const yAxis: Record<string, unknown> = { ...((option.yAxis as Record<string, unknown>) || {}) };
    const existing = (yAxis.axisLabel as Record<string, unknown>) || {};
    yAxis.axisLabel = { ...existing, rotate: props.yAxis.labelAngle };
    option.yAxis = yAxis;
  }
```

Before the `return option;` at the end, add grid visibility:

```typescript
  // Grid line visibility (splitLine controls gridlines)
  if (withAxes && props.grid !== undefined) {
    if (props.grid.x === false) {
      const xAxis: Record<string, unknown> = { ...((option.xAxis as Record<string, unknown>) || {}) };
      const existing = (xAxis.splitLine as Record<string, unknown>) || {};
      xAxis.splitLine = { ...existing, show: false };
      option.xAxis = xAxis;
    }
    if (props.grid.y === false) {
      const yAxis: Record<string, unknown> = { ...((option.yAxis as Record<string, unknown>) || {}) };
      const existing = (yAxis.splitLine as Record<string, unknown>) || {};
      yAxis.splitLine = { ...existing, show: false };
      option.yAxis = yAxis;
    }
  }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehub/viz run test -- option-pipeline.test.ts`
Expected: All tests pass including 7 new ones

- [ ] **Step 5: Run full viz and ui test suites**

Run: `yarn workspace @casehub/viz run test && yarn workspace @casehub/ui run test`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/casehub-viz/src/charts/option-pipeline.ts packages/casehub-viz/src/charts/option-pipeline.test.ts
git -C /Users/mdproctor/claude/melviz commit -m "feat(@casehub/viz): apply axis labelAngle and grid visibility

Maps xAxis/yAxis.labelAngle to echarts axisLabel.rotate and
grid.x/y to splitLine.show for controlling gridline visibility.

Closes #21"
```

---

## Task 7: Markdown parser (#14)

Replace the `<pre>` markdown rendering with proper markdown-to-HTML parsing.

**Files:**
- Modify: `packages/casehub-runtime/package.json`
- Modify: `packages/casehub-runtime/src/content.ts:17-21`
- Test: `packages/casehub-runtime/src/content.test.ts` (create)

- [ ] **Step 1: Create content.test.ts with failing markdown tests**

Create `packages/casehub-runtime/src/content.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { renderMarkdown, renderHtml, renderTitle } from "./content.js";

describe("renderMarkdown", () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement("div");
  });

  it("renders bold text as <strong>", () => {
    renderMarkdown(el, { content: "**bold**" });
    expect(el.querySelector(".casehub-markdown")?.innerHTML).toContain("<strong>bold</strong>");
  });

  it("renders heading", () => {
    renderMarkdown(el, { content: "### Heading" });
    expect(el.querySelector(".casehub-markdown")?.innerHTML).toContain("<h3>Heading</h3>");
  });

  it("renders mixed markdown and HTML", () => {
    renderMarkdown(el, { content: "**Filters**\n<br />\nSome text" });
    const html = el.querySelector(".casehub-markdown")?.innerHTML ?? "";
    expect(html).toContain("<strong>Filters</strong>");
    expect(html).toContain("<br");
  });

  it("renders empty string without error", () => {
    renderMarkdown(el, { content: "" });
    expect(el.querySelector(".casehub-markdown")).toBeDefined();
  });

  it("handles non-string content gracefully", () => {
    renderMarkdown(el, { content: 42 });
    expect(el.querySelector(".casehub-markdown")?.textContent).toBe("");
  });

  it("adds casehub-markdown class to wrapper", () => {
    renderMarkdown(el, { content: "text" });
    expect(el.querySelector(".casehub-markdown")).not.toBeNull();
  });
});

describe("renderTitle", () => {
  it("creates h1 by default", () => {
    const el = document.createElement("div");
    renderTitle(el, { text: "Hello" });
    expect(el.querySelector("h1")?.textContent).toBe("Hello");
  });

  it("creates h3 when size specified", () => {
    const el = document.createElement("div");
    renderTitle(el, { text: "Hello", size: "h3" });
    expect(el.querySelector("h3")?.textContent).toBe("Hello");
  });
});

describe("renderHtml", () => {
  it("sets innerHTML from content", () => {
    const el = document.createElement("div");
    renderHtml(el, { content: "<p>hello</p>" });
    expect(el.innerHTML).toBe("<p>hello</p>");
  });
});
```

- [ ] **Step 2: Run tests to verify markdown tests fail**

Run: `yarn workspace @casehub/runtime run test -- content.test.ts`
Expected: renderMarkdown tests FAIL (current impl uses `<pre>`, not parsed HTML). renderTitle and renderHtml tests should PASS.

- [ ] **Step 3: Add marked dependency**

In `packages/casehub-runtime/package.json`, add to dependencies:

```json
"marked": "^15.0.0"
```

Run: `yarn install`

- [ ] **Step 4: Replace renderMarkdown implementation**

In `packages/casehub-runtime/src/content.ts`, replace the entire file:

```typescript
import { marked } from "marked";

export function renderTitle(el: HTMLElement, props: Record<string, unknown>): void {
  const text = typeof props.text === "string" ? props.text : "";
  const size = typeof props.size === "string" ? props.size : "h1";
  const validSizes = ["h1", "h2", "h3", "h4", "h5", "h6"];
  const tag = validSizes.includes(size) ? size : "h1";
  const heading = document.createElement(tag);
  heading.textContent = text;
  el.appendChild(heading);
}

export function renderHtml(el: HTMLElement, props: Record<string, unknown>): void {
  if (typeof props.content === "string") {
    el.innerHTML = props.content;
  }
}

export function renderMarkdown(el: HTMLElement, props: Record<string, unknown>): void {
  const content = typeof props.content === "string" ? props.content : "";
  const wrapper = document.createElement("div");
  wrapper.classList.add("casehub-markdown");
  wrapper.innerHTML = marked.parse(content) as string;
  el.appendChild(wrapper);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehub/runtime run test -- content.test.ts`
Expected: All tests pass

- [ ] **Step 6: Run full runtime test suite**

Run: `yarn workspace @casehub/runtime run test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/melviz add packages/casehub-runtime/src/content.ts packages/casehub-runtime/src/content.test.ts packages/casehub-runtime/package.json
git -C /Users/mdproctor/claude/melviz commit -m "feat(@casehub/runtime): markdown parser for content components

Replaces <pre> rendering with marked.parse() for proper markdown-to-HTML
conversion. Supports bold, headings, links, and mixed markdown+HTML.

Closes #14"
```

---

## Task 8: Build and validate all dashboards (#22)

Build the complete gallery and test every dashboard.

**Files:**
- No new source files — this is a validation task
- Create (workspace): validation matrix document

- [ ] **Step 1: Build all packages**

Run: `yarn build:packages`
Expected: All TypeScript packages compile without errors

- [ ] **Step 2: Build examples gallery**

Run: `yarn workspace @melviz/examples run build`
Expected: `examples/dist/` contains `casehub-bundle.js`, `index.html`, `app.js`, `styles.css`, `samples.json`, `dashboards/` directory

- [ ] **Step 3: Serve and test inline-data dashboards**

Run: `yarn workspace @melviz/examples run serve`

Open `http://localhost:8080` and test each inline-data dashboard:
- Simple Chart
- InlineDataset
- Filter
- Filter With Table
- Histogram
- DarkMode
- Column with rows
- Most Spoken Languages
- Kitchensink
- Accumulate Flag
- Decal Pattern
- Global Column settings
- Global Lookup Operation
- Date test
- Backstage Metrics
- Developers Registration
- Greeting Serverless Workflow Heatmap

For each: verify layout renders, charts display data, no console errors.

- [ ] **Step 4: Test external-URL dashboards**

Test dashboards with external data sources (expect some CORS failures):
- Table (GitHub Gists API)
- Google Spreadsheet (Google Sheets API)
- FIFA 2022 Goals (FIFA API)
- Github Repositories (GitHub API)
- Podman Stats (local socket)
- Prometheus Basic / HTTP Requests (Prometheus API)

Document which ones succeed vs fail with CORS or dead endpoints.

- [ ] **Step 5: Test local-file dashboards**

Test dashboards referencing local data files:
- JVM Monitoring
- Quarkus Monitoring
- Real Time JVM Monitoring
- Open Telemetry Basic
- Jupyter Hub Metrics / Summary
- Kepler Metrics
- ModelMesh Metrics
- Triton Metrics
- Ansible Metrics
- TimeSeries

Document which ones need local data files that aren't present.

- [ ] **Step 6: Document validation results**

Create validation matrix at workspace (not project repo):

```bash
# Write results to workspace
```

Format: table with columns: Dashboard | Category | Data Source | Status | Notes

- [ ] **Step 7: File follow-up issues for any new gaps**

For any dashboards that fail due to code issues (not CORS/missing data), create GitHub issues.

- [ ] **Step 8: Commit validation results to workspace**

```bash
git -C /Users/mdproctor/claude/public/melviz add validation/
git -C /Users/mdproctor/claude/public/melviz commit -m "docs: dashboard validation matrix

Closes #22"
```

- [ ] **Step 9: Run full test suite across all packages**

Run: `yarn test`
Expected: All tests pass across all packages

- [ ] **Step 10: Final commit — close epic**

If any final adjustments were needed, commit them. Reference #19 as closed:

```bash
git -C /Users/mdproctor/claude/melviz commit --allow-empty -m "feat: DashBuilder demo runner complete

All 32 DashBuilder sample dashboards render via casehub runtime in the
examples gallery. No GWT, no Java.

Closes #19"
```
