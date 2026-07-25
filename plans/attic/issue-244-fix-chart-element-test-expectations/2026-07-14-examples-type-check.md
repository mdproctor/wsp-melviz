# Examples Type-Check Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task uses
> ide-tooling for verification. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #179 — chore: make TypeScript examples type-checkable
**Issue group:** #179

**Goal:** Migrate 22 legacy `page()` examples to the current API signature and wire `samples/` into the examples tsconfig so all 44 `.ts` files are type-checked.

**Architecture:** Each legacy file uses a 4-arg `page(properties, defaults, datasetsArray, componentsArray)` convention. The current API is `page(name, ...components, options?)`. Migration is mechanical per-file: extract name from filename, move properties into `PageOptions.properties`, push displayer defaults into individual components, unwrap component arrays to varargs. Several files also have broken `restSource`/`inlineSource` calls (syntax errors from the original YAML→TS translation) that need rebuilding.

**Tech Stack:** TypeScript, `@casehubio/pages-ui` DSL builders

## Global Constraints

- Page names derived from filename (e.g., `"Prometheus Basic"` from `Prometheus Basic.ts`)
- No `displayer` defaults cascading — push `chart.resizable`, `chart.height`, etc. into each component
- Drop `refresh.interval` (monitoring URLs won't be live in examples gallery)
- Drop `lookup.uuid` defaults (already handled by per-component `lookup()` calls)
- Preserve `properties` via `PageOptions.properties`
- Fix all broken `restSource`/`inlineSource` syntax errors — reconstruct from surrounding context
- Remove `import type { DataSetId, ColumnId }` and `as DataSetId`/`as ColumnId` casts when the current API doesn't require them (the builder functions accept plain strings)
- Remove stale comments like `// TypeScript companion to "X.yml"` — these files ARE the examples now, not companions to YAML

---

### Task 1: Migrate misc/ examples (6 files)

**Files:**
- Modify: `examples/samples/misc/TimeSeries.ts`
- Modify: `examples/samples/misc/Most Spoken Languages.ts`
- Modify: `examples/samples/misc/Github Repositories.ts`
- Modify: `examples/samples/misc/Podman Stats.ts`
- Modify: `examples/samples/misc/FIFA 2022 Goals.ts`

**Interfaces:**
- Consumes: `page`, `bind`, `restSource`, `inlineSource`, `html`, `metric`, `lineChart`, `barChart`, `bubbleChart`, `table`, `columns`, `selector`, `tabs`, `div`, `withStyle`, `markdown`, `title`, `lookup` from `@casehubio/pages-ui`
- Produces: Valid `export default page("Name", ...components, { datasets, properties })` for each file

**Pattern (applies to all files in this task):**

The legacy convention is:
```typescript
export default page(
  { prop1: "val" },           // → PageOptions.properties
  { displayer: { ... } },     // → push into components; drop refresh/lookup.uuid
  [ /* broken restSource */ ], // → fix syntax, keep as dataset bindings above page()
  [ comp1, comp2 ],           // → unwrap to varargs
  { datasets: [...] }         // → keep in PageOptions
);
```

Becomes:
```typescript
export default page("Page Name",
  comp1,
  comp2,
  { properties: { prop1: "val" }, datasets: [...] }
);
```

- [ ] **Step 1: Migrate `TimeSeries.ts`**

Current file has empty properties `{}`, empty defaults `{}`, one `timeseries` component, and `chart.resizable` needs pushing into the component. The `restSource` call is intact.

Rewrite to:
```typescript
import { page, bind, restSource, timeseries, lookup } from "@casehubio/pages-ui";

const timeseriesDs = bind("timeseries", restSource("data/sample_timeseries.json", {}));

export default page("TimeSeries",
  timeseries({
    lookup: lookup("timeseries"),
    chart: { resizable: true },
  }),
  { datasets: [timeseriesDs] }
);
```

- [ ] **Step 2: Migrate `Most Spoken Languages.ts`**

Broken `inlineSource` call on line 8 (`bind("langs", inlineSource([;`). The inline data is split into the page's 3rd arg (lines 13-19). Reconstruct the full `inlineSource` call, then restructure `page()`.

Rewrite to:
```typescript
import { page, bind, inlineSource, html, barChart, table, lookup, groupBy, col } from "@casehubio/pages-ui";

const langsDs = bind("langs", inlineSource([
  ["English", "Hello World", 1132],
  ["Mandarin", "你好世界", 1117],
  ["Hindi", "नमस्ते दुनिया", 615],
  ["Spanish", "Hola Mundo", 534],
  ["French", "Bonjour le monde", 280],
], {
  columns: [
    { id: "Language", type: "LABEL" },
    { id: "Greeting", type: "LABEL" },
    { id: "Speakers", type: "NUMBER" },
  ],
}));

export default page("Most Spoken Languages",
  html(`<p style="font-size: xx-large; margin-bottom: 30px"> Most spoken languages</p><hr style=""/>`),
  barChart({
    lookup: lookup("langs", groupBy("Language", col("Language"), col("Speakers"))),
    chart: { resizable: true },
  }),
  table({
    lookup: lookup("langs"),
    chart: { resizable: true },
  }),
  { datasets: [langsDs] }
);
```

Note: Replace raw `"Column 0"` / `"Column 2"` references with named columns since we're defining the schema. Use the new `groupBy`/`col` helpers matching the already-migrated examples pattern.

- [ ] **Step 3: Migrate `Github Repositories.ts`**

Broken `restSource` on line 8. The restSource config (expression, columns) is split into the page's 3rd arg. Reconstruct. Push `chart.resizable` into components. Drop `lookup.uuid` default.

Rewrite to:
```typescript
import { page, bind, restSource, title, barChart, table, lookup } from "@casehubio/pages-ui";

const githubReposDs = bind("github_repos", restSource("https://api.github.com/search/repositories?q=stars:>1&s=stars", {
  cacheEnabled: "true",
  refreshTime: "10minute",
  expression: `$.items.[name, stargazers_count, forks, watchers_count, open_issues, owner.login, created_at, language ? language : '-', description ]`,
  columns: [
    { id: "name", type: "label" },
    { id: "stars", type: "number" },
    { id: "forks", type: "number" },
    { id: "watchers", type: "number" },
    { id: "open_issues", type: "number" },
    { id: "owner_login", type: "label" },
    { id: "created", type: "label" },
    { id: "language", type: "label" },
    { id: "description", type: "text" },
  ],
}));

export default page("Github Repositories",
  title("Top 10 GitHub Repositories by Stars"),
  barChart({
    lookup: lookup("github_repos", { type: "rowCount", count: 10 },
      {
        type: "group",
        groupingKey: { sourceId: "name" },
        functions: [
          { source: "name" },
          { source: "stars" },
        ],
      }),
    axis: { x: { labels_angle: -10 } },
    chart: { resizable: true },
  }),
  title("List of top repositories by stars"),
  table({
    lookup: lookup("github_repos"),
    chart: { resizable: true },
  }),
  { datasets: [githubReposDs] }
);
```

- [ ] **Step 4: Migrate `Podman Stats.ts`**

Broken `restSource` calls on lines 20-21. Two data sources with config split into page's 3rd arg. Complex multi-page layout with tabs/div navigation. Push `chart.resizable` and `table.sort` from defaults into components.

This file is large (200 lines) — read it fully, reconstruct both `restSource` calls with their expression/columns config, restructure `page()` to current API. Properties: `{ baseUrl: "http://localhost:8000" }`.

- [ ] **Step 5: Migrate `FIFA 2022 Goals.ts`**

Broken `restSource` on line 8. Complex FIFA API expression. Push `chart.resizable`, `chart.height: 300`, `chart.grid` from defaults into components. Properties: `{ GoalsFunction: "AVERAGE", SeriesColor: "cyan" }`. The `mode: "dark"` and `export.png` from defaults are dropped (not in current PageOptions).

Reconstruct restSource with the full expression and columns config. Restructure page() — all component code stays the same, just unwrap from array.

- [ ] **Step 6: Run typecheck on migrated files**

Run: `yarn workspace @casehubio/pages-examples run typecheck 2>&1 | head -50`

Note: This will fail because `samples/` isn't in tsconfig yet. That's expected — just verify the migrated files don't have syntax errors by checking them individually:

Run: `npx tsc --noEmit --project examples/tsconfig.json 2>&1 | head -30`

- [ ] **Step 7: Commit**

```bash
git add examples/samples/misc/
git commit -m "refactor: migrate misc/ examples to current page() API

Refs #179"
```

---

### Task 2: Migrate Prometheus/ and Micrometer/ examples (5 files)

**Files:**
- Modify: `examples/samples/Prometheus/Prometheus Basic.ts`
- Modify: `examples/samples/Prometheus/Prometheus HTTP Requests.ts`
- Modify: `examples/samples/Micrometer/JVM Monitoring.ts`
- Modify: `examples/samples/Micrometer/Quarkus Monitoring.ts`
- Modify: `examples/samples/Micrometer/Real Time JVM Monitoring.ts`

**Interfaces:**
- Same pattern as Task 1

- [ ] **Step 1: Migrate `Prometheus Basic.ts`**

Properties `{ prometheusUrl }` and defaults `{ displayer: { refresh, lookup, chart } }` are intact. Two `restSource` calls are intact (lines 8-9). Components are in an array (lines 23-142). Push `chart.resizable` into each chart component. Drop `refresh.interval` and `lookup.uuid`.

Rewrite: `page("Prometheus Basic", ...components, { properties: { prometheusUrl: "http://localhost:9090" }, datasets: [prometheusDs, prometheusInstantDs] })`.

Remove `as DataSetId` and `as ColumnId` casts — use plain strings.

- [ ] **Step 2: Migrate `Prometheus HTTP Requests.ts`**

Two intact `restSource` calls. Components are already inline (not in an array — the 3rd arg is the components, not datasets). Properties: `{ prometheusUrl, refreshInterval }`. Push `chart.resizable` from defaults. Drop `refresh.interval`.

- [ ] **Step 3: Migrate `JVM Monitoring.ts`**

Broken `restSource` on line 8 (`{;`). Broken page structure — line 23 has bare `columns:` outside any object. Need to reconstruct the restSource with its columns config, then restructure the components. Properties: `{ refreshInterval: 5, metricsUrl: "data/quarkus/metrics" }`.

- [ ] **Step 4: Migrate `Quarkus Monitoring.ts`**

Broken `restSource` on line 8. Properties: `{ refreshInterval: 10, metricsUrl: "data/quarkus/metrics" }`. The defaults contain `columns` and `lookup.uuid` which need dropping. Complex layout with metric cards, heap/nonheap memory bars, threads bar. Push `chart.resizable`, `chart.height: 350`, `chart.grid.x: false` into each component.

- [ ] **Step 5: Migrate `Real Time JVM Monitoring.ts`**

One broken `restSource` on line 12 (`{;`), one intact on line 11. Two datasets with accumulate/cache config in the broken one. Properties: `{ metricsUrl, historyUrl }`. Push `chart.resizable` from defaults.

- [ ] **Step 6: Commit**

```bash
git add examples/samples/Prometheus/ examples/samples/Micrometer/
git commit -m "refactor: migrate Prometheus/ and Micrometer/ examples to current page() API

Refs #179"
```

---

### Task 3: Migrate remaining domain examples (11 files)

**Files:**
- Modify: `examples/samples/Backstage/Backstage Metrics.ts`
- Modify: `examples/samples/ansible/Ansible Metrics.ts`
- Modify: `examples/samples/Clinical/Patient Tracker.ts`
- Modify: `examples/samples/IoT/Fleet Monitor.ts`
- Modify: `examples/samples/jupyterhub/metrics/Jupyter Hub Metrics Histograms.ts`
- Modify: `examples/samples/jupyterhub/metrics/Jupyter Metrics Summary.ts`
- Modify: `examples/samples/kepler/Kepler Metrics.ts`
- Modify: `examples/samples/modelmesh/ModelMeshMetrics.ts`
- Modify: `examples/samples/OpenTelemetry/Open Telemetry Basic.ts`
- Modify: `examples/samples/People/Workforce Analytics.ts`
- Modify: `examples/samples/Sales/Sales Dashboard.ts`
- Modify: `examples/samples/triton/Triton Inference Server Model Metrics.ts`

**Interfaces:**
- Same pattern as Tasks 1-2

- [ ] **Step 1: Migrate `Backstage Metrics.ts`**

Intact `restSource`. Defaults contain `extraConfiguration`, `columns` expression, and `lookup.uuid`. The `extraConfiguration` and `columns` expression from defaults were applied to all components — push them into each `barChart` that needs them. The `lookup.uuid` is already per-component via `lookup()`.

- [ ] **Step 2: Migrate `Ansible Metrics.ts`**

Broken `restSource` on line 8. Reconstruct with `cacheEnabled`, `refreshTime`, `columns`, `headers` config from the page's 3rd arg. Properties: `{ token, authorizationHeader, towerUrl, proxyUrl, subTitlesStyle }`. Drop `lookup.uuid` and `columns` pattern from defaults.

- [ ] **Step 3: Migrate `Clinical/Patient Tracker.ts`**

Read file, apply same pattern. This is a larger file (~150 lines) — read fully before migrating.

- [ ] **Step 4: Migrate `IoT/Fleet Monitor.ts`**

Read file, apply same pattern. Another large file (~108 lines).

- [ ] **Step 5: Migrate `jupyterhub/metrics/Jupyter Hub Metrics Histograms.ts`**

Read file, apply same pattern.

- [ ] **Step 6: Migrate `jupyterhub/metrics/Jupyter Metrics Summary.ts`**

Read file, apply same pattern.

- [ ] **Step 7: Migrate `kepler/Kepler Metrics.ts`**

Read file, apply same pattern.

- [ ] **Step 8: Migrate `modelmesh/ModelMeshMetrics.ts`**

Read file, apply same pattern.

- [ ] **Step 9: Migrate `OpenTelemetry/Open Telemetry Basic.ts`**

Read file, apply same pattern.

- [ ] **Step 10: Migrate `People/Workforce Analytics.ts`**

Read file, apply same pattern.

- [ ] **Step 11: Migrate `Sales/Sales Dashboard.ts`**

Hybrid pattern — top-level `page()` uses legacy 4-arg convention but nested `page("name", ...)` calls are already current. Fix top-level only: `page("Sales Dashboard", sidebar(...), page("Overview", ...), ..., { datasets: [...] })`. Properties: `{ GoalsFunction: "SUM" }`. Push `chart.resizable` from defaults into components that don't already set it.

- [ ] **Step 12: Migrate `triton/Triton Inference Server Model Metrics.ts`**

Read file, apply same pattern.

- [ ] **Step 13: Commit**

```bash
git add examples/samples/Backstage/ examples/samples/ansible/ examples/samples/Clinical/ examples/samples/IoT/ examples/samples/jupyterhub/ examples/samples/kepler/ examples/samples/modelmesh/ examples/samples/OpenTelemetry/ examples/samples/People/ examples/samples/Sales/ examples/samples/triton/
git commit -m "refactor: migrate remaining domain examples to current page() API

Refs #179"
```

---

### Task 4: Update tsconfig and fix type errors

**Files:**
- Modify: `examples/tsconfig.json`

**Interfaces:**
- Consumes: All 44 migrated `.ts` files in `examples/samples/`
- Produces: Clean typecheck across all examples

- [ ] **Step 1: Update tsconfig.json**

Add `"samples"` to `include` and add missing project references:

```json
{
  "extends": "@casehubio/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "rootDir": ".",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true
  },
  "include": [
    "src",
    "samples"
  ],
  "references": [
    { "path": "../packages/pages-runtime" },
    { "path": "../packages/pages-viz" },
    { "path": "../packages/pages-data" },
    { "path": "../packages/pages-ui" }
  ]
}
```

Note: `rootDir` changes from `"src"` to `"."` because `samples/` is a sibling of `src/`, not inside it. Both must be under rootDir.

- [ ] **Step 2: Run typecheck**

Run: `yarn typecheck 2>&1 | tail -30`

Expected: May surface type errors in migrated files. Fix any that appear.

- [ ] **Step 3: Run lint**

Run: `yarn lint 2>&1 | tail -30`

Expected: May surface lint errors. Fix any that appear.

- [ ] **Step 4: Run examples build**

Run: `yarn build:examples 2>&1 | tail -20`

Expected: Build succeeds — examples gallery generates correctly.

- [ ] **Step 5: Commit**

```bash
git add examples/tsconfig.json
git commit -m "build: add samples/ to examples tsconfig with project references

Closes #179"
```
