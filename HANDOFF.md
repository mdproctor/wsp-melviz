# Melviz Session Handover — 2026-06-16 (session 2)

## Last Session

Epic #19: Load DashBuilder demo dashboards via casehub runtime. 25+ bug fixes, 1100+ unit tests, 20 Playwright integration tests. Gallery at `examples/` renders 23+ of 32 dashboards via pure TypeScript `loadSite()` — no GWT, no Java. Branch `issue-19-dashbuilder-demos` on both repos, pushed to fork.

## Branch State

Both repos on `issue-19-dashbuilder-demos`. Fork remote is current. ~30 commits on the branch.

## What Works (visually verified)

Simple Chart, InlineDataset, Filter, Filter With Table, Column with rows, DarkMode, Decal Pattern, Date test, Global Column settings, Global Lookup Operation, Accumulate Flag, Histogram, Google Spreadsheet, Table, Most Spoken Languages, FIFA 2022 Goals, Github Repositories, Backstage Metrics, JVM Monitoring, Triton (10/10), Greeting Heatmap, TimeSeries, Jupyter Hub Histograms.

## What Still Needs Fixing — Per Dashboard

### Quarkus Monitoring (5/7 components render)
- 2 bar charts fail: `UNKNOWN_COLUMN: Column "labels" not found`
- Root cause: global displayer has `columns: [{id: Total, pattern: "#"}, {id: Value, pattern: "#"}]` which renames output columns to capitalised IDs. The filter references lowercase `labels` but the dataset columns are now capitalised after the global column override is applied.
- Fix approach: trace the full column rename chain through global defaults → displayer desugar → extraction explicit columns → group eval. The case-insensitive helper (#31) was added but the rename happens at a different layer.
- Compare against: the canonical Quarkus sample may differ from our local copy.

### Ansible Metrics (0/16 — all fail)
- All components: `UNKNOWN_COLUMN: Column "value" not found`
- Root cause: Ansible has TWO datasets — one with `${towerUrl}?metrics` (external, falls back to mock) and one with `data/metrics` (local Prometheus file). The local data file loads correctly and gets Prometheus column naming (`metric`, `labels`, `value`). But the components reference `value` in filters/groups and it's not found after the group operation transforms the output.
- Fix approach: check if the Ansible YAML's group operations rename `value` to something else (like the Triton `column: Duration` pattern). The graceful sort fix helped Triton but filter/group still throw on unknown columns. Consider making filter graceful too (skip unknown filter columns instead of throwing).

### Prometheus Basic (0/2)
- `EXTRACTION_ERROR: Unrecognized data shape`
- Root cause: uses `type: prometheus` preset which expects Prometheus HTTP API JSON response format (`{status, data: {resultType, result: [{metric, value}]}}`) — NOT text exposition format. Our mock data is text format. The preset's JSONata expression (`$.data.(...)`) fails because the mock data doesn't have the expected JSON structure.
- Fix: create a mock Prometheus API JSON response file (`mock-data/prometheus-api-response.json` already exists but isn't wired to the gallery fallback for `type: prometheus` datasets). The gallery's `galleryFetch` needs to serve JSON mock for URLs containing `/api/v1/query`.

### Prometheus HTTP Requests (1/6)
- 5 components: `UNKNOWN_COLUMN: Column "handler" not found` and `Column "code" not found`
- Root cause: dashboard filters by `handler` and `code` columns which come from Prometheus label keys. Our mock Prometheus text data has these labels but the `type: prometheus` preset transforms the JSON API response differently — it creates columns from label keys. With mock text data, the Prometheus naming only produces `metric`, `labels`, `value` — no individual label columns.
- Fix: same as Prometheus Basic — need JSON API mock that includes the label keys as columns.

### Backstage Metrics (renders but content duplicated)
- First row of metrics appears twice
- Root cause: multi-page dashboard. `index` page has `screen: Cards` (page-ref that embeds the Cards page). But page-ref resolution isn't implemented (#27). Both the index page's embed AND the direct Cards page render, showing the same metrics twice.
- Fix: implement page-ref resolution (#27) — when `screen: Cards` is encountered, replace it with the Cards page content instead of rendering both.

### FIFA 2022 Goals (renders but has layout gaps)
- Large vertical gaps between metrics row and charts row
- Root cause: CSS grid with `grid-auto-rows: min-content` on pages that have items. The metrics (small height) and charts (300px height) are in different grid rows, and the grid doesn't collapse empty space between them. Also multi-page rendering stacks pages with gaps.
- Fix: investigate the grid placement coordinates — the metrics and charts may be on non-consecutive grid rows (e.g. row 1 and row 5) with empty rows 2-4 creating gaps.

### Podman Stats (6/8 with mock data)
- 2 components: `UNKNOWN_COLUMN: Column "state" not found`
- Root cause: Podman dashboard expects JSON data from the Podman API (`/containers/json`), not Prometheus format. Mock fallback serves Prometheus text which produces `metric`, `labels`, `value` columns — no `state` column. These 2 components filter by container `state`.
- Fix: create a Podman-specific JSON mock with container data including `state`, `image`, `names` columns. Or make filter graceful (skip unknown columns instead of throwing).

### Jupyter Metrics Summary (4/8)
- 4 components: `UNKNOWN_COLUMN: Column "metric" not found`
- Root cause: some components in this dashboard use a different dataset that may not have the Prometheus column naming applied. Or the JSONata expression transforms the columns.
- Fix: check the YAML for multiple datasets and whether each gets correct column naming.

### Kitchensink (partially renders — charts/tables/metrics work)
- Missing: `tree`, `menu`, `tiles`, `slot-target`, `page-ref` components (#27)
- Timeseries: `FETCH_FAILED` — external data file not present (expected)
- Map: `Cannot read properties of undefined (reading 'regions')` — CasehubMap component has a bug
- Fix: #27 covers the navigation components. Map bug is separate — the map component tries to access a property that doesn't exist.

### Model Mesh Metrics (mostly empty)
- Uses `${modelMeshMetricsUrl}` property variable — unresolved, falls back to mock Prometheus data
- Multiple datasets with different column expectations
- Fix: check canonical sample, may need specific mock data or the YAML may differ from our local copy.

## Issues Touched This Session (all under epic #19)

| # | Title | Status | Notes |
|---|-------|--------|-------|
| #19 | Epic: Load DashBuilder demos | OPEN | Parent epic |
| #20 | Demo runner — host page + dev server | CLOSED | Gallery works |
| #21 | Axis and grid settings desugaring | CLOSED | labelAngle + grid visibility |
| #14 | Markdown parser | CLOSED | marked.parse() |
| #22 | Migrate and validate samples | OPEN | Validation done, some dashboards still need fixes |
| #23 | Domain-specific example dashboards | OPEN | Future — new examples for Sales, DevOps, IoT etc. |
| #24 | Move to casehubio org | OPEN | Repo restructuring — after #19 |
| #25 | Global displayer defaults merged | CLOSED | Deep merge in component-desugar |
| #26 | Inline content / dataSet shorthand | CLOSED | resolveInlineDataSet in activation |
| #27 | Navigation components | OPEN | tree, menu, tiles, slot-target, page-ref |
| #28 | fetch binding | CLOSED | globalThis.fetch.bind() |
| #29 | URL configuration form | OPEN | Gallery UI for external URLs |
| #30 | Update to canonical samples | OPEN | Sync with kie-samples |
| #31 | Case-insensitive column matching | CLOSED | findColumn helper |

## Critical Working Practices

**Test as you investigate, not just when you fix.** Every area you probe, every assumption you check, every "I think this works" — write a unit test. Tests harden the system as you iterate. This session added 40+ tests for investigated-but-not-broken weak spots and they caught real issues. See CLAUDE.md `## Testing Discipline`.

**Visual verification — no exceptions.** DOM checks lie. A chart can have `hasData: true` and a canvas element but render as a flat line with no bars. A table can have rows but display `[object Object]`. This session wasted hours trusting DOM checks that said "OK" when the human saw broken charts. Use Playwright screenshots for EVERY dashboard you touch. Compare against the live DashBuilder gallery.

**Build cache kills you silently.** `yarn build:packages` skips unchanged dist. After editing source, always: `yarn workspace <pkg> run clean && yarn workspace <pkg> run build`. Multiple "fix not taking effect" spirals in this session traced back to stale compiled output.

## What NOT to Try

- Don't guess DashBuilder conventions — compare: https://www.dashbuilder.org/kitchensink/?samples#
- Don't trust DOM checks alone — take Playwright screenshots, verify charts visually
- Don't assume `build:packages` recompiles — always `clean && build` per package
- Don't add event listeners after renderComponent — connectedCallback fires during appendChild

## Key Architecture Decisions

1. Event listeners BEFORE renderComponent (connectedCallback timing)
2. Registry entry BEFORE appendChild (same timing)
3. Lookup parsing in displayer-desugar (uuid→dataSetId)
4. Deep merge arrays element-by-element (extraConfiguration series)
5. Mock data fallback for unresolved ${property} URLs
6. Prometheus column naming as post-tabulation rename

## External Sources (ranked by usefulness)

1. **Live gallery** — https://www.dashbuilder.org/kitchensink/?samples# — MOST USEFUL. Shows exactly what each dashboard should look like when rendered correctly. Use `?import=<yaml-url>` to test any YAML. Compare every fix against this.
2. **Canonical samples** — https://github.com/kiegroup/kie-samples/tree/main/samples — AUTHORITATIVE. The "official" dashboard YAMLs. Our local `examples/dashboards/` are modified forks (local URLs, barchart instead of victory-charts, some missing column defs). When a dashboard doesn't match the live gallery, check the canonical version first — our local copy may be the problem (#30).
3. **Community samples** — https://github.com/jesuino/dashbuilder-yaml-samples/tree/main — SUPPLEMENTARY. More examples by a DashBuilder maintainer. Includes data files (e.g. `triton/data/metrics`). Some canonical samples reference data URLs in this repo. Useful for understanding data file formats.
4. **DashBuilder source** — https://github.com/kiegroup/kie-tools/tree/main/packages — REFERENCE ONLY. The Java/GWT DashBuilder implementation. Huge monorepo — look in `packages/dashbuilder-*` for the rendering engine. Useful when you need to understand how DashBuilder handles a specific feature (e.g. how `type: prometheus` transforms data, how column naming works). Don't try to read the whole thing — search for specific class names.
5. **Our local examples** — `examples/dashboards/` — MODIFIED FORKS. Differ from canonical in: local relative URLs (good for offline), `type: barchart` instead of `component: victory-charts`, some data files bundled locally. When debugging, always check if the local YAML matches the canonical version.

**Staleness:** The live gallery runs current DashBuilder. Canonical samples are maintained. Our local copies haven't been synced — some may be outdated versions.

## Project References

- Spec: `docs/superpowers/specs/2026-06-16-dashbuilder-demo-runner-design.md`
- Playwright: `examples/tests/gallery.spec.ts` (20 tests)
- Mock data: `examples/mock-data/prometheus-metrics.txt`
- Filed: #27 (nav components), #29 (URL forms), #30 (canonical samples update)
