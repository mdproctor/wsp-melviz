## D1: Config UI approach

**Choice:** Extend samples.json with optional `config` array — data-driven property controls
**Alternatives:**
- Auto-detect from YAML — zero metadata but brittle, can't know valid enum values
- Companion config files — full flexibility but scattered files, more maintenance
**Rationale:** Extends the existing `renderConfigBar` / `propertyOverrides` pattern. Data-driven — add new sample configs by editing `samples.json`, no code changes. Control types (enum, boolean) map directly to dropdown/checkbox.
**Trade-offs:** Requires manual metadata per sample — not auto-discovered.
**Sources:** examples/src/app.js:264-323 (existing config bar), examples/samples.json (sample schema)
**Exploration:** quick
**Status:** captured
