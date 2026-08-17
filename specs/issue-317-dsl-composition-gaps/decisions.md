# Decisions — #317 DSL Composition Gaps

## D1: Metric row — layout builder, not new component

**Choice:** `metricRow()` is a layout builder that composes N `metric()` instances into a `columns()` layout with compact styling. Sparkline and trend props are added to the existing `MetricProps`.
**Alternatives:**
- New `PagesMetricRow` web component — dedicated element with its own rendering, sparkline integration, and sizing logic
**Rationale:** The rendering is just layout (flex row, consistent child sizing). The individual `metric()` component already handles value display. Adding sparkline/trend to `MetricProps` is useful regardless of whether a row builder exists. A new component would duplicate layout logic that `columns()` already solves.
**Trade-offs:** Less control over inter-metric visual cohesion (spacing between metrics is managed by the layout engine, not a dedicated component). If future requirements need tight coordination between metrics (e.g., synchronized sparkline scales), a component refactor may be needed.
**Sources:** `packages/pages-viz/src/components/PagesMetric.ts`, `packages/pages-ui/src/dsl/builders.ts` (columns builder), `packages/pages-ui-components/src/sparkline/render-sparkline.ts`
**Exploration:** quick
**Status:** captured

## D2: Master-detail — convenience builder, not new component

**Choice:** `masterDetail()` is a convenience builder that composes `split()` + `dataTable()` + selection wiring. It sets `selection: "single"` on the master table and wires `selectionSource` on the detail pane automatically.
**Alternatives:**
- New `PagesDetailLayout` web component — manages selection state internally with dedicated rendering
**Rationale:** The selection forwarding infrastructure already exists (#299 `detailDataset()`, #300 `selection-forwarding.ts`, `pages-selection-changed` event). The gap is verbose wiring, not missing capability. A builder encapsulates the wiring pattern without adding components.
**Trade-offs:** The detail pane must be a `hostPanel()` (only host panels receive `pages-selection-changed` today). If non-host-panel detail panes are needed, the selection forwarding machinery would need extending.
**Sources:** `packages/pages-runtime/src/selection-forwarding.ts`, `packages/pages-table/src/pages-data-table.ts` (selection-change event), `packages/pages-ui/src/dsl/builders.ts` (split builder)
**Exploration:** quick
**Status:** captured

## D3: Heatmap — add missing builder functions only

**Choice:** Add `heatmapChart()`, `treemapChart()`, `densityHeatmap()` builder functions to `builders.ts`. Components, prop types, type guards, and activation are all already implemented.
**Alternatives:**
- None — this is the only reasonable path. The components exist; the builders don't.
**Rationale:** Mechanical gap — the same freeze-and-return pattern as every other chart builder.
**Trade-offs:** None.
**Sources:** `packages/pages-component/src/model/displayer-types.ts` (HeatmapChartProps, TreemapChartProps, DensityHeatmapProps), `packages/pages-component/src/model/type-guards.ts`, `packages/pages-runtime/src/activation.ts`
**Exploration:** quick
**Status:** captured

## D4: Timeline — new generic component in pages-viz

**Choice:** New `PagesTimeline` component in `pages-viz` extending `PagesElement`, with `TimelineStrategy` plugin interface, three layout renderers (vertical, horizontal, compact), and `timeline()` DSL builder. Design informed by blocks-ui's `blocks-timeline` but without domain coupling.
**Alternatives:**
- Wrap or re-export blocks-timeline from pages — rejected because blocks-timeline depends on blocks-ui-core domain types
- Build only the DSL builder and defer the component — rejected because there's no existing component to wrap
**Rationale:** blocks-timeline proved the design (strategy pattern, three renderers, node expansion, filtering). The generic rendering is domain-free. Building in pages enables blocks-ui to extend it (blocks-ui#123 filed).
**Trade-offs:** Timeline strategies written for blocks-timeline won't be plug-compatible until blocks-ui#123 refactors to extend PagesTimeline. Parallel implementations exist temporarily.
**Sources:** `blocks-ui/components/blocks-timeline/src/types.ts` (TimelineNode, TimelineStrategy), `blocks-ui/components/blocks-timeline/src/blocks-timeline.ts`, blocks-ui#123
**Exploration:** quick
**Status:** captured
