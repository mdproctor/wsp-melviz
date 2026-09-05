# Decisions — PagesSchemaForm palette migration (#375)

## D1: Composite type handling

**Choice:** Custom resolver — PagesSchemaForm provides a custom `EditorResolver` to the embedded palette that returns render-function descriptors for composite types (object-group, array-group, variant-group), delegating to the existing pages-viz components.
**Alternatives:**
- Extend the palette — add composite type support directly to pages-property-palette, making it the single source of truth for nested forms. Would increase palette complexity and couple it to data-pipeline-aware form concerns.
- Split: palette for leaves only — palette handles leaf fields; PagesSchemaForm keeps its own rendering loop for composites. Defeats the purpose of embedding — two parallel rendering paths remain.
**Rationale:** The custom resolver is the palette's designed extension point (D5 in #373 decisions). Composite types are pages-viz-specific (they need TypedDataSet, DataSetLookup, cell extraction). The palette stays decoupled from the data pipeline. The resolver returns `{ kind: 'render', render: ... }` descriptors that render the existing composite components.
**Trade-offs:** Composite rendering stays imperative in the resolver callbacks rather than benefiting from the palette's declarative rendering. But these components already have their own well-tested rendering — the resolver just bridges them.
**Sources:** `pages-property-palette/src/types.ts` (EditorResolver, EditorDescriptor), `pages-viz/src/form-inputs/PagesObjectGroup.ts`, `PagesArrayGroup.ts`, `PagesVariantGroup.ts`, #373 decisions D5
**Exploration:** quick
**Status:** captured

## D2: Data pipeline bridge

**Choice:** Adapter in PagesSchemaForm — PagesSchemaForm builds a flat `data: Record<string, unknown>` from the TypedDataSet row and passes it to `PropertyPaletteSource.data`. Extraction logic stays in pages-viz where the data pipeline types live.
**Alternatives:**
- Separate utility function (`datasetToSourceData()`) — reusable by other pipeline-aware palette consumers. Premature abstraction — no other consumer exists yet.
**Rationale:** PagesSchemaForm already does this extraction (reading cells, converting types, extracting distinct values for selects). Moving it to a PropertyPaletteSource builder is a natural refactor within the same file. No new abstractions needed.
**Trade-offs:** If another pipeline-aware palette consumer appears later, the extraction logic would need extracting at that point. YAGNI for now.
**Sources:** `PagesSchemaForm.ts` lines 120-215 (cell reading, type conversion), `pages-property-palette/src/types.ts` (PropertyPaletteSource)
**Exploration:** quick
**Status:** captured

## D3: Public API preservation

**Choice:** Thin wrapper — PagesSchemaForm becomes a thin wrapper that builds a `PropertyPaletteSource` from the dataset and renders `<pages-property-palette>`, but keeps its own `validate()`, `submit()`, and `currentValue` by maintaining a data mirror updated via `source.onChange`.
**Alternatives:**
- Full delegation — extend the palette's API with validate/submit/currentValue, have PagesSchemaForm just proxy. Would bloat the palette with data-pipeline-specific concerns (record creation events, ARIA announcements) that don't belong in a generic property inspector.
**Rationale:** The palette is a rendering engine, not a form controller. Validation, submission, and value collection are PagesSchemaForm's domain responsibilities. The palette calls `source.onChange` when values change; PagesSchemaForm maintains the current record state from those callbacks and provides validate/submit on top. Public API is unchanged — consumers don't need to know the palette exists.
**Trade-offs:** PagesSchemaForm maintains a data mirror alongside the palette's internal state. But this is simple (onChange updates a plain object) and keeps responsibilities cleanly separated.
**Sources:** `PagesSchemaForm.ts` validate() (lines 324-346), submit() (lines 348-374), currentValue (lines 88-97), `PropertyPaletteSource.onChange`
**Exploration:** quick
**Status:** captured
