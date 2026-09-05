# Decisions — PagesSchemaForm palette migration (#375)

## D1: Composite type handling

**Choice:** Custom resolver — PagesSchemaForm provides a custom `EditorResolver` to the embedded palette that returns render-function descriptors for composite types (object-group, array-group, variant-group), delegating to the existing pages-viz components.
**Alternatives:**
- Move composites to palette — since composites are actually pipeline-agnostic (they extend `FormValueMixin(LitElement)`, not `PagesElement`), they could live in the palette itself. Cleaner long-term but expands scope beyond #375.
- Split: palette for leaves only — palette handles leaf fields; PagesSchemaForm keeps its own rendering loop for composites. Defeats the purpose of embedding — two parallel rendering paths remain.
**Rationale:** The custom resolver keeps #375's scope bounded to the migration. The composites ARE pipeline-agnostic (R1-01 correction: they don't import TypedDataSet/DataSetLookup), so moving them to the palette is viable as a follow-up. But for this issue, using the resolver extension point (D5 in #373 decisions) lets us migrate without relocating components across packages.
**Trade-offs:** Composite rendering stays imperative in the resolver callbacks. Moving composites to the palette is a natural follow-up that would eliminate `mapFieldToComponentType` entirely (R1-07) and give all palette consumers native array/variant support.
**Sources:** `pages-property-palette/src/types.ts` (EditorResolver, EditorDescriptor), `pages-viz/src/form-inputs/PagesObjectGroup.ts`, `PagesArrayGroup.ts`, `PagesVariantGroup.ts`, #373 decisions D5; Decision review R1-01 (corrected rationale)
**Exploration:** quick
**Status:** revised (rationale corrected per R1-01 — composites are not pipeline-coupled)

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
**Trade-offs:** PagesSchemaForm maintains a data mirror alongside the palette's internal state. The mirror must be pre-seeded from initial dataset values (not just onChange), so `currentValue` works before any user interaction (R1-04).
**Sources:** `PagesSchemaForm.ts` validate() (lines 324-346), submit() (lines 348-374), currentValue (lines 88-97), `PropertyPaletteSource.onChange`; Decision review R1-04 (initialisation gap)
**Exploration:** quick
**Status:** revised (added initialisation strategy per R1-04)

## D4: Palette element caching (prerequisite fix)

**Choice:** Add element caching to the palette's `_renderTagEditor` — cache created elements by field key, reuse on re-render if tag name hasn't changed. Mirrors PagesSchemaForm's existing pattern.
**Alternatives:**
- Leave palette as-is, cache in PagesSchemaForm's resolver — the custom resolver callbacks could maintain their own element cache. Pushes caching responsibility to every consumer instead of fixing it once in the palette.
- Use Lit `repeat()` directive with keyed templates — cleaner Lit-idiomatic approach but requires reworking the palette's imperative rendering to declarative templates. Larger change.
**Rationale:** The palette currently re-creates DOM elements on every render cycle (R1-02). This causes text inputs to lose focus, dropdowns to close, and partially entered values to be lost. This is a bug in the palette, not a PagesSchemaForm concern — it must be fixed in the palette before any migration can happen. The element cache pattern is proven (PagesSchemaForm uses it at lines 127-131).
**Trade-offs:** Adds state (element cache Map) to the palette. But the alternative — broken inputs on every re-render — is not acceptable.
**Depends on:** None (prerequisite for D1, D3)
**Sources:** `pages-property-palette/src/palette/pages-property-palette.ts` `_renderTagEditor` (lines 203-280), `PagesSchemaForm.ts` element reuse (lines 127-131); Decision review R1-02
**Exploration:** quick
**Status:** captured

## D5: Select option enrichment from dataset

**Choice:** PagesSchemaForm enriches the schema before passing to the palette — when a field has no `enum` but the dataset has distinct values, PagesSchemaForm populates `schema.properties[field].enum` with extracted distinct values before constructing the PropertyPaletteSource.
**Alternatives:**
- Extend PropertyPaletteSource with an options callback — palette calls back to get options per field. Over-engineered for a single consumer's need.
**Rationale:** The palette resolves selects from `schema.enum`. PagesSchemaForm already extracts distinct values from multi-row datasets (R1-03). Schema enrichment before passing to the palette is the simplest bridge — no palette API changes needed.
**Trade-offs:** Mutates a copy of the schema. Must deep-clone relevant parts to avoid side effects on the original schema prop.
**Depends on:** D2 (data bridge)
**Sources:** `PagesSchemaForm.ts` `extractDistinctValues` (lines 291-303), `buildChildProps` (lines 269-275); Decision review R1-03
**Exploration:** quick
**Status:** captured

## D6: Data mirror pre-seeding

**Choice:** PagesSchemaForm pre-seeds the data mirror from the initial dataset extraction when building the PropertyPaletteSource, so `currentValue` returns correct values before any user interaction.
**Alternatives:**
- Read values from palette shadow DOM — would require the palette to expose its internal element state, breaking encapsulation.
**Rationale:** `source.onChange` only fires on user interaction (R1-04). The data mirror must be initialised from the same dataset extraction that populates `source.data`. This is a single assignment: `this._dataMirror = { ...sourceData }` at source construction time.
**Trade-offs:** Two copies of the data exist (source.data and mirror). Kept in sync by onChange callbacks. Simple and predictable.
**Depends on:** D2 (data bridge), D3 (thin wrapper)
**Sources:** `PropertyPaletteSource.onChange`, `PagesSchemaForm.ts` currentValue (lines 88-97); Decision review R1-04
**Exploration:** quick
**Status:** captured
