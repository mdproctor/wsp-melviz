# Decisions — issue-413-broaden-spi-interfaces

## D1: SPI boundary for chart components

**Choice:** Universal + likely-portable concepts only
**Alternatives:**
- Universal concepts only — too restrictive, misses dataZoom range and animation duration which most charting libraries support
- Current backend surface (expose all stable ECharts options typed) — defeats the purpose of the SPI as an abstraction layer
**Rationale:** The SPI must remain implementation-agnostic so alternative charting backends could be swapped without breaking YAML. Options that most mature charting libraries support (even if API shapes differ slightly) are worth promoting. ECharts-specific concepts (toolbox features, formatter template syntax) stay in the typed escape hatch (D4).
**Trade-offs:** Minor adaptation cost if backend changes — promoted properties may need remapping to a new library's API.
**Sources:** Issue #413 constraints ("SPI interfaces must remain implementation-agnostic where multiple backends exist"), ECharts 5.6 type audit
**Exploration:** quick
**Status:** captured

## D2: Extract VizCommon from ChartSettings

**Choice:** Create a new `VizCommon` interface containing `resizable`, `zoom`, `maxWidth`, `maxHeight`, `extra`. ChartSettings extends VizCommon and adds chart-specific properties (legend, margin, xAxis, yAxis, grid). GraphProps and DensityHeatmapProps extend VizCommon directly.
**Alternatives:**
- Keep ChartSettings, document ignored props — misleading interface where GraphProps inherits xAxis/yAxis/grid that are meaningless for graphs
- Stop extending, duplicate shared props — simple but duplicated across interfaces; changes to shared properties require updating multiple interfaces
**Rationale:** GraphProps extending ChartSettings causes xAxis/yAxis/grid to appear in graph schemas, confusing YAML authors and accepting invalid properties. DensityHeatmapProps not extending ChartSettings means it lacks the `extra` escape hatch. VizCommon cleanly separates universal visualization concerns from chart-specific ones.
**Trade-offs:** Adds a new interface to the hierarchy. Downstream code that references ChartSettings directly may need updating if it only needs VizCommon properties.
**Sources:** displayer-types.ts current hierarchy, React Flow + ELK audit (xAxis/yAxis meaningless for graphs), @drdreo/heatmap audit (no extra escape hatch)
**Exploration:** quick
**Status:** captured

## D3: Graph layout property promotion

**Choice:** Promote abstract layout concepts (direction, spacing, algorithm) to GraphProps. Keep ELK-specific options behind a `layoutOptions?: Record<string, string>` passthrough.
**Alternatives:**
- Promote all internal ElkLayoutOptions — accepts coupling to ELK's model; a different layout engine would require SPI changes
- Keep all layout behind passthrough — only the existing layout enum is typed; direction and spacing (which any layout engine supports) would require YAML authors to know ELK's string-key API
**Rationale:** Direction (up/down/left/right), spacing (number), and algorithm selection are universal graph layout concepts. ELK-specific options like partitioning, inter-layer spacing, and wrapping are implementation details.
**Trade-offs:** The `algorithm` enum values need mapping from abstract names to ELK's internal algorithm names. If a new layout engine is added, abstract values would need a mapping layer.
**Sources:** graph-renderer ElkLayoutOptions audit, React Flow + ELK type audit
**Exploration:** quick
**Status:** captured

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

**Choice:** Fix existing property wiring bugs (e.g., `radius` declared in DensityHeatmapProps but not passed to @drdreo/heatmap config) within this branch.
**Alternatives:**
- Separate bug issues — keeps #413 scope clean but adds overhead for one-line fixes in files we're already editing
**Rationale:** Wiring bugs are in the same files we're modifying. Fixing them separately doubles the review and schema regeneration overhead for trivial changes.
**Trade-offs:** Slightly broader branch scope than pure SPI broadening.
**Sources:** @drdreo/heatmap audit (radius not wired at line 100-107)
**Exploration:** quick
**Status:** captured

## D6: Testing strategy

**Choice:** Schema staleness tests (existing) plus renderer unit tests verifying each new property reaches the upstream library's option object.
**Alternatives:**
- Schema staleness + sample YAML only — confirms properties parse but not that they reach the library; wiring bugs like the radius issue would pass
- Schema + visual snapshot tests — highest confidence but heavy infrastructure for a property-promotion change
**Rationale:** The radius wiring bug proves that "property is in the interface" doesn't mean "property reaches the library." Renderer unit tests catch this class of bug. Schema staleness tests catch interface/schema drift.
**Trade-offs:** More tests to write, but each test is small (verify one property appears in the option object passed to the library).
**Sources:** @drdreo/heatmap audit (radius wiring bug as motivating example)
**Exploration:** quick
**Depends on:** D5 (wiring bugs motivate renderer-level testing)
**Status:** captured
