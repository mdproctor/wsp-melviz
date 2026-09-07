# Decisions — issue-413-broaden-spi-interfaces

## D1: SPI boundary for chart components

**Choice:** Universal + likely-portable concepts only, where "likely-portable" is defined by a two-library mapping test: each promoted property must show a shape-preserving mapping (single value → single value) to at least two charting libraries' APIs. Properties where the mapping requires structural transformation belong in D4's typed escape hatch until a clean common-denominator abstraction is designed.
**Alternatives:**
- Universal concepts only — too restrictive, misses dataZoom range and animation duration which most charting libraries support
- Current backend surface (expose all stable ECharts options typed) — defeats the purpose of the SPI as an abstraction layer
**Rationale:** The SPI must remain implementation-agnostic so alternative charting backends could be swapped without breaking YAML. Options that most mature charting libraries support (even if API shapes differ slightly) are worth promoting. ECharts-specific concepts (toolbox features, formatter template syntax) stay in the typed escape hatch (D4).
**Trade-offs:** Minor adaptation cost if backend changes — promoted properties may need remapping to a new library's API. The two-library mapping test adds rigour but also adds upfront audit cost per property.
**Sources:** Issue #413 constraints ("SPI interfaces must remain implementation-agnostic where multiple backends exist"), ECharts 5.6 type audit
**Exploration:** quick
**Status:** revised — R1: added two-library mapping criterion for portability classification (from R1-04)

## D2: Split ChartSettings along the Cartesian boundary

**Choice:** Split `ChartSettings` into `ChartSettingsBase` (resizable, zoom, legend, margin, extra) and `ChartSettings extends ChartSettingsBase` adding Cartesian-specific properties (xAxis, yAxis, grid). Non-Cartesian ECharts charts (Graph, Pie, Map, Meter, Treemap) extend `ChartSettingsBase` instead of `ChartSettings`. `DensityHeatmapProps` gets `extra` added directly — it does not extend any chart settings interface. Move `maxWidth`/`maxHeight` from `ChartSettings` to `DataComponentCommon` alongside `width`/`height`.
**Alternatives:**
- VizCommon (original proposal) — cross-backend shared base containing resizable, zoom, maxWidth, maxHeight, extra. Rejected: these properties have divergent semantics across ECharts and @drdreo/heatmap — `zoom` means ECharts dataZoom but has no counterpart in @drdreo/heatmap. The abstraction leaks.
- Keep ChartSettings, document ignored props — misleading interface where GraphProps inherits xAxis/yAxis/grid that are meaningless for graphs. Fixing only the `{ cartesianAxes: false }` flag stops silent application but leaves xAxis/yAxis/grid in the Zod schema, suggesting to YAML authors that they're valid properties for graphs.
- Stop extending, duplicate shared props — simple but duplicated across interfaces; changes to shared properties require updating multiple interfaces
**Rationale:** The Cartesian split addresses the root cause: non-Cartesian charts inheriting axis properties. `ChartSettingsBase` is coherent because it targets a single backend family (ECharts) and contains only properties that every ECharts chart type supports (legend, margin, zoom, extra, resizable). `DensityHeatmapProps` stays independent because @drdreo/heatmap has different capabilities — forcing it through an ECharts-shaped base interface would create dead properties. `maxWidth`/`maxHeight` are CSS dimension constraints that apply to any renderable component, not just visualizations — they belong alongside `width`/`height` in `DataComponentCommon`.
**Trade-offs:** Adds `ChartSettingsBase` to the hierarchy. Non-Cartesian charts change their extends from `ChartSettings` to `ChartSettingsBase`. Downstream code referencing `ChartSettings` that only uses base properties needs updating.
**Sources:** displayer-types.ts current hierarchy, PagesGraph.ts wiring bug (missing `{ cartesianAxes: false }`), PagesDensityHeatmap.ts showing no ECharts dependency, R1-02 challenge, R1-08 challenge
**Exploration:** quick
**Status:** revised — R1: replaced VizCommon with Cartesian/non-Cartesian split, moved maxWidth/maxHeight to DataComponentCommon, scoped DensityHeatmapProps extra independently (from R1-02, R1-03, R1-08)

## D3: Graph layout property promotion (scoped to ECharts graph)

**Choice:** Promote ECharts graph-specific layout properties to `GraphProps`: force repulsion strength, edge label visibility, roam (pan/zoom), and symbol (node shape). No additional passthrough property — niche ECharts graph options are covered by `extra` (from ChartSettingsBase) and `echarts?` (from D4). ELK layout concepts (direction, spacing, algorithm) are NOT promoted to GraphProps — they belong to the GraphCanvas rendering path (see D7).
**Alternatives:**
- Promote ELK layout concepts (direction, spacing, algorithm) to GraphProps — WRONG: GraphProps controls PagesGraph (ECharts), not GraphCanvas (React Flow + ELK). These would be dead properties. ELK concepts belong in ElkLayoutOptions and are only accessible when/if GraphCanvas gets YAML integration (D7).
- Add `layoutOptions?: Record<string, string>` as a GraphProps-specific passthrough — WRONG: `Record<string, string>` is the ELK pattern (flat string pairs like `'elk.algorithm': 'layered'`). ECharts graph options are structured JSON (nested objects like `{ repulsion: 100 }`, arrays like `["none", "arrow"]`, booleans, numbers). The type cannot represent ECharts options without serialization. Also redundant with `extra` and `echarts?`.
- Promote all internal ECharts graph options — accepts coupling to ECharts' model
- Keep all layout behind passthrough — only the existing layout enum is typed
**Rationale:** The original D3 conflated two architecturally independent rendering paths. PagesGraph (pages-viz, ECharts) and GraphCanvas (graph-renderer, React Flow + ELK) share no code, no props interface, and no rendering pipeline. GraphProps controls PagesGraph. Promoting ELK concepts to GraphProps would create dead properties that PagesGraph never reads. Scoping to ECharts graph properties promotes the options that PagesGraph actually uses. The `extra` and `echarts?` escape hatches already provide two tiers of ECharts-specific access for any graph options beyond the promoted set — adding a third (`layoutOptions`) would be redundant and with the wrong type.
**Trade-offs:** ELK layout properties are not SPI-accessible until D7 resolves GraphCanvas YAML integration. YAML authors wanting ELK features must use GraphCanvas programmatically.
**Sources:** PagesGraph.ts (ECharts graph component extending PagesChartElement), GraphCanvas.ts (React Flow + ELK, registered as pages-graph-canvas, NOT in TYPE_MAP), ElkLayoutOptions interface (packages/graph-renderer/src/layout/elk-layout.ts), ARC42STORIES.MD §5 (pages-viz = ECharts wrappers, graph-renderer = React Flow + ELK bridge)
**Exploration:** quick
**Status:** revised — R1: scoped to ECharts graph only, removed ELK concepts (from R1-01). R2: dropped layoutOptions passthrough — wrong type for ECharts, redundant with extra/echarts? (from R2-01)

## D4: Typed escape hatch pattern (complements SPI, does not replace it)

**Choice:** In addition to maximising the universal/portable SPI surface (D1), add a typed, backend-specific escape hatch property per component family — `echarts?` for charts/map, `reactFlow?` for graph interaction, `elk?` for graph layout, `heatmap?` for density heatmap. Each uses a curated subset type (e.g., `CasehubEChartsExtension`) containing the stable top-level option categories from the upstream library. The SPI captures everything that is common-denominator or likely-portable; the escape hatch covers the long tail of library-specific features that go beyond what any reasonable common denominator can express.
**Alternatives:**
- Import and passthrough — type as `Partial<EChartsOption>` in TypeScript but generate Zod schema as `z.record()` (permissive); YAML authors lose validation
- Full upstream generation — extend schema generator to handle all imported upstream types; significant complexity for ECharts' recursive/conditional types
- Keep `extra: Record<string, unknown>` as the only escape hatch — untyped, no schema validation, no IDE autocomplete
**Rationale:** The universal SPI is the primary interface — it's what YAML authors should reach for first. But ECharts has hundreds of options that go far beyond any common denominator. The typed escape hatch gives power users full library access with schema validation, making the coupling explicit through the property name. A curated subset keeps the schema generator tractable while covering ~95% of use cases.
**Trade-offs:** The curated type needs periodic maintenance as upstream libraries add features. Some very niche upstream options won't be in the curated type (they can still use `extra` as a last resort).
**Sources:** User proposal for `.as(ECharts.class)` pattern, ECharts type audit showing massive type surface beyond portable concepts
**Exploration:** quick
**Depends on:** D1 (SPI boundary — the escape hatch complements the SPI for options beyond the common denominator)
**Status:** captured

## D5: Wiring bug fixes in scope

**Choice:** Fix existing property wiring bugs within this branch:
1. `radius` declared in DensityHeatmapProps but not passed to @drdreo/heatmap config (PagesDensityHeatmap.ts createInstance method)
2. PagesGraph calls `applyChartSettings(option, props)` without `{ cartesianAxes: false }` — every other non-Cartesian ECharts chart (Pie, Map, Meter, Treemap) correctly passes `{ cartesianAxes: false }` to skip xAxis/yAxis application
**Alternatives:**
- Separate bug issues — keeps #413 scope clean but adds overhead for one-line fixes in files we're already editing
**Rationale:** Wiring bugs are in the same files we're modifying. The PagesGraph cartesianAxes bug is particularly relevant because D2 restructures the ChartSettings hierarchy — fixing the bug alongside the interface change ensures the runtime behavior matches the new type structure. Even after D2's Cartesian split removes xAxis/yAxis from GraphProps' schema, the `{ cartesianAxes: false }` flag is still needed as a runtime safety net (the function defaults to `true`).
**Trade-offs:** Slightly broader branch scope than pure SPI broadening.
**Sources:** @drdreo/heatmap audit (radius not wired at createInstance), PagesGraph.ts applyChartSettings call (missing { cartesianAxes: false }), comparison with PagesPieChart.ts, PagesMap.ts, PagesMeter.ts, PagesTreemapChart.ts (all correctly opt out)
**Exploration:** quick
**Status:** revised — R1: added PagesGraph cartesianAxes wiring bug (from R1-02)

## D6: Testing strategy

**Choice:** Schema staleness tests (existing) plus renderer unit tests verifying each new property reaches the upstream library's option object, plus YAML → desugarer → props round-trip tests verifying that each newly promoted property, when set in YAML, appears in the desugared component's props with the correct value.
**Alternatives:**
- Schema staleness + sample YAML only — confirms properties parse but not that they reach the library; wiring bugs like the radius issue would pass
- Schema + visual snapshot tests — highest confidence but heavy infrastructure for a property-promotion change
- Schema staleness + renderer tests only (original proposal) — misses the desugarer boundary: a property could be declared in the interface and schema, but not correctly extracted by the desugarer's schema-aware passthrough if the property name collides with a `handledKeys` entry
**Rationale:** The radius wiring bug proves that "property is in the interface" doesn't mean "property reaches the library." The desugarer's schema-aware passthrough (displayer-desugar.ts) passes properties that are in the Zod schema and NOT in `handledKeys` — but if a promoted property name collides with a `handledKeys` entry, it would be silently swallowed. Round-trip tests catch this entire class of bug: interface declaration → schema generation → YAML parsing → desugarer extraction → renderer wiring → library option object.
**Trade-offs:** More tests to write, but each test is small. The round-trip tests catch a failure mode that neither schema tests nor renderer tests alone would catch.
**Sources:** @drdreo/heatmap audit (radius wiring bug as motivating example), displayer-desugar.ts schema-aware passthrough and handledKeys set, R1-05 challenge
**Exploration:** quick
**Depends on:** D5 (wiring bugs motivate renderer-level testing)
**Status:** revised — R1: added desugarer round-trip tests (from R1-05)

## D7: GraphCanvas becomes YAML-accessible as a new component type

**Choice:** Add GraphCanvas to TYPE_MAP as a new YAML component type (e.g., `graph-canvas` or `diagram`). GraphCanvas gets its own props interface (`GraphCanvasProps`) separate from the existing `GraphProps` (which controls the ECharts PagesGraph). Both component types coexist: `graph` for ECharts-based dashboard visualization, `graph-canvas` for React Flow + ELK interactive diagrams.
**Alternatives:**
- Independent — PagesGraph stays YAML-accessible, GraphCanvas stays programmatic-only. Simplest but leaves GraphCanvas without YAML integration despite being the more capable renderer.
- Replacement — GraphCanvas replaces PagesGraph. Loses ECharts graph simplicity for cases where interactive editing is unnecessary.
**Rationale:** GraphCanvas (React Flow + ELK) is the more capable graph component with interactive editing, containment layout, stencil registry, and rich node rendering. Making it YAML-accessible gives YAML authors access to the full graph editing experience. Both renderers serve different use cases — ECharts graph is simpler for dashboard network visualizations; GraphCanvas supports interactive diagram editing.
**Trade-offs:** Adds a new component type to TYPE_MAP, a new props interface, and desugarer routing. Broadens #413 scope. GraphCanvasProps needs its own typed escape hatches (`reactFlow?`, `elk?`).
**Sources:** ARC42STORIES.MD §5, PagesGraph.ts, GraphCanvas.ts, user decision to make GraphCanvas YAML-accessible
**Exploration:** quick
**Depends on:** D3 (GraphProps remains ECharts-scoped), D4 (typed escape hatches apply per component type)
**Status:** captured
