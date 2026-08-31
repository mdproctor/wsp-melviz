# pages-viz: composite network heatmap component

**Issue:** casehubio/casehub-pages#302
**Date:** 2026-08-31

## Summary

New `PagesNetworkHeatmap` web component (`pages-network-heatmap`, type
`network-heatmap`) that renders a graph topology with `@drdreo/heatmap`
density overlay — showing traffic intensity along network edges.

## Architecture

Two rendering layers composited in a single container:

1. **Base layer:** `@drdreo/heatmap` density canvas — interpolated points
   along each edge, weighted by traffic value
2. **Overlay:** SVG layer for node circles, labels, and optional edge lines

The component extends `PagesElement<NetworkHeatmapProps>`, following the
same pattern as `PagesDensityHeatmap`. It lazy-loads the heatmap module,
extracts data from a `TypedDataSet`, and renders both layers into a
container div.

## Data model

The dataset has one row per edge with these columns:

| Column prop | Purpose | Required |
|------------|---------|----------|
| `sourceColumn` | Source node ID | yes |
| `targetColumn` | Target node ID | yes |
| `trafficColumn` | Edge traffic value (weight) | yes |
| `sourceXColumn` | Source node X coordinate | yes |
| `sourceYColumn` | Source node Y coordinate | yes |
| `targetXColumn` | Target node X coordinate | yes |
| `targetYColumn` | Target node Y coordinate | yes |
| `nodeLabelColumn` | Node display label (defaults to node ID) | no |

Node positions come from the dataset columns — no force layout. Nodes are
deduplicated from source/target pairs.

## Props interface

```typescript
export interface NetworkHeatmapProps extends DataComponentCommon {
  readonly sourceColumn?: ColumnId;
  readonly targetColumn?: ColumnId;
  readonly trafficColumn?: ColumnId;
  readonly sourceXColumn?: ColumnId;
  readonly sourceYColumn?: ColumnId;
  readonly targetXColumn?: ColumnId;
  readonly targetYColumn?: ColumnId;
  readonly nodeLabelColumn?: ColumnId;
  readonly showEdgeLines?: boolean;
  readonly showNodeLabels?: boolean;
  readonly pointsPerEdge?: number;
  readonly gradient?: readonly { readonly offset: number; readonly color: string }[];
  readonly radius?: number;
}
```

## Rendering pipeline

1. **Extract edges** from dataset: for each row, read source/target IDs,
   coordinates, and traffic value
2. **Deduplicate nodes** from edge source/target pairs (by ID), keeping
   the first-seen coordinates for each node
3. **Normalize coordinates** to pixel space within the container
   (same approach as `PagesDensityHeatmap.extractAndNormalize`)
4. **Generate heatmap points** along each edge: interpolate N points
   (default 20, configurable via `pointsPerEdge`) between source and
   target coordinates, each with `value = traffic / pointsPerEdge`
5. **Create/update heatmap** via `createHeatmap()` / `setData()`
6. **Render SVG overlay** with node circles and optional labels/edge lines

## SVG overlay

The overlay SVG is positioned absolutely over the heatmap canvas. It
renders:
- **Node circles** at each deduplicated node position (filled, small radius)
- **Node labels** (optional, `showNodeLabels` defaults to true) —
  text positioned offset from each circle
- **Edge lines** (optional, `showEdgeLines` defaults to false) —
  semi-transparent lines between connected nodes

## Files to create/modify

| File | Change |
|------|--------|
| `packages/pages-component/src/model/displayer-types.ts` | Add `NetworkHeatmapProps` interface |
| `packages/pages-component/src/model/type-guards.ts` | Add type guard and type map entry |
| `packages/pages-component/src/model/index.ts` | Export `NetworkHeatmapProps` |
| `packages/pages-viz/src/charts/PagesNetworkHeatmap.ts` | New component class |
| `packages/pages-viz/src/charts/PagesNetworkHeatmap.test.ts` | Unit tests |
| `packages/pages-viz/src/custom-elements.ts` | Register tag |
| `packages/pages-viz/src/index.ts` | Export component |
| `examples/samples/Charts/Network Heatmap.dash.yaml` | Gallery sample |
| `examples/samples.json` | Add sample entry |

## Test plan

1. Edge extraction from dataset — correct source/target/traffic parsing
2. Node deduplication — unique nodes from edge pairs
3. Coordinate normalization — maps raw coords to pixel space
4. Heatmap point interpolation — N points per edge, weighted by traffic
5. SVG overlay renders nodes and labels
6. Gallery sample loads without errors

## References

- `packages/pages-viz/src/charts/PagesDensityHeatmap.ts` — heatmap integration pattern
- `packages/pages-component/src/model/displayer-types.ts:191-200` — DensityHeatmapProps
- `packages/pages-component/src/model/type-guards.ts:101` — type map entry pattern
- `docs/protocols/casehub/web-component-strategy.md` — Lit + PagesElement convention
- casehubio/casehub-pages#302 — issue
- casehubio/casehub-pages#301 — pages-density-heatmap (dependency, already landed)
