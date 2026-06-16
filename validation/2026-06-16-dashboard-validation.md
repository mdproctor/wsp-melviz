# DashBuilder Dashboard Validation Matrix

**Date:** 2026-06-16
**Epic:** #19
**Issue:** #22

## Summary

35 DashBuilder YAML dashboards validated against the casehub runtime pipeline.

- **13 inline-data dashboards** — expected to render fully (self-contained data)
- **2 mixed dashboards** — have inline content + external URL references
- **20 external-URL dashboards** — depend on external APIs; expected CORS failures when served from localhost

All dashboards parse correctly through `parsePage()`. The gallery builds and serves without errors. Rendering validation for inline dashboards requires browser testing.

## Validation Matrix

| Dashboard | Category | Data Source | Features Used | Expected Status |
|-----------|----------|------------|---------------|-----------------|
| Simple Chart | Basic Usage | inline | - | Full |
| InlineDataset | Basic Usage | inline | - | Full |
| Filter | Basic Usage | inline | - | Full |
| Filter With Table | Basic Usage | inline | - | Full |
| Column with rows | Basic Usage | inline | - | Full |
| DarkMode | Basic Usage | inline | - | Full |
| Date test | Basic Usage | inline | - | Full |
| Decal Pattern | Basic Usage | inline | - | Full |
| Global Column settings | Basic Usage | inline | expression | Full |
| Global Lookup Operation | Basic Usage | inline | - | Full |
| Most Spoken Languages | misc | inline | - | Full |
| Accumulate Flag | Basic Usage | inline | expression | Full |
| Histogram | Basic Usage | inline | expression | Full |
| Kitchensink | Basic Usage | mixed | markdown | Full (inline data renders; external may fail) |
| Greeting Serverless Workflow Heatmap | Basic Usage | mixed | expression | Full (inline data renders) |
| Table | Basic Usage | external | expression | Data error — GitHub Gists API (CORS) |
| Google Spreadsheet | Basic Usage | external | - | Data error — Google Sheets API (CORS) |
| Developers Registration | Basic Usage | external | expression | Data error — external URL |
| FIFA 2022 Goals | misc | external | axis, grid, expression | Data error — FIFA API (CORS) |
| Github Repositories | misc | external | axis, expression | Data error — GitHub API (CORS) |
| Podman Stats | misc | external | markdown, expression | Data error — local socket |
| TimeSeries | misc | external | - | Data error — local file |
| JVM Monitoring | Micrometer | external | axis, expression | Data error — local metrics |
| Quarkus Monitoring | Micrometer | external | grid, expression | Data error — local metrics |
| Real Time JVM Monitoring | Micrometer | external | expression | Data error — local metrics |
| Prometheus Basic | Prometheus | external | markdown | Data error — Prometheus API |
| Prometheus HTTP Requests | Prometheus | external | - | Data error — Prometheus API |
| Jupyter Hub Metrics Histograms | jupyterhub | external | axis | Data error — local metrics |
| Jupyter Metrics Summary | jupyterhub | external | axis, expression | Data error — local metrics |
| Kepler Metrics | kepler | external | markdown, grid, expression | Data error — Prometheus API |
| Open Telemetry Basic | OpenTelemetry | external | axis, expression | Data error — local file |
| ModelMesh Metrics | modelmesh | external | axis, grid, expression | Data error — custom API |
| Triton Inference Server Metrics | triton | external | axis, expression | Data error — local metrics |
| Ansible Metrics | ansible | external | - | Data error — local metrics |
| Backstage Metrics | Backstage | external | axis, expression | Data error — external URL |

## Feature Coverage

| Feature | Dashboards Using It | Implementation Status |
|---------|--------------------|-----------------------|
| Inline data (`content`) | 15 | Supported |
| External URL (`url`) | 22 | Supported (CORS-dependent) |
| JSONata expressions | 20 | Supported |
| Axis settings (`axis.x/y`) | 10 | Supported (Task 4, #21) |
| Grid visibility (`chart.grid`) | 5 | Supported (Task 5, #21) |
| Markdown content | 5 | Supported (Task 7, #14) |
| Property substitution (`${name}`) | ~15 | Supported |
| Dark mode (`global.mode: dark`) | 1 | Supported |

## Build Verification

- `yarn build:packages` — all TypeScript packages compile
- `yarn workspace @melviz/examples run build` — gallery builds (848KB bundle)
- `yarn test` — all tests pass across all packages
- `examples/dist/` contains: `casehub-bundle.js`, `index.html`, `app.js`, `styles.css`, `samples.json`, `dashboards/`

## Known Limitations

1. **External URL dashboards** — 20 of 35 depend on external APIs that will fail with CORS when served from `localhost:8080`. These show error banners in the gallery, which is the expected behavior.
2. **Local file dashboards** — Micrometer, Prometheus, JupyterHub, etc. reference local data files (`data/`, `metrics`) that aren't bundled. These will fail to fetch.
3. **No visual regression baseline** — no screenshots or Playwright tests yet (#23 future work). Validation is structural (parse + build) not visual.
