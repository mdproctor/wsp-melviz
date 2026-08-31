## D1: Node positioning strategy

**Choice:** Dataset columns — require x/y coordinate columns in the dataset. Consumer provides fixed positions.
**Alternatives:**
- Force-directed layout — compute positions via force simulation. Adds complexity, non-determinism, layout instability on data updates.
- Both with fallback — accept x/y if present, force layout if absent. Two code paths, double the testing surface.
**Rationale:** Simplest, most predictable. The use case (Hortora/grove network) has known node positions from the topology model. Force layout can be added later if needed.
**Trade-offs:** No auto-layout — consumers must provide coordinates. Acceptable for the target use case.
**Sources:** PagesDensityHeatmap.ts (existing x/y/value column pattern), GraphProps (source/target/value pattern), issue #302
**Exploration:** quick
**Status:** captured
