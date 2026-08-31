## D1: Canonical location for FieldSchema

**Choice:** `pages-component` — merge superset fields into the existing FieldSchema in `form-input-types.ts`. Delete the duplicate in `displayer-types.ts`. Delete from `pages-ui-components`. No re-exports anywhere.
**Alternatives:**
- pages-ui-components — superset lives there but would reverse the dependency direction (pages-component → pages-ui-components)
- pages-data — lowest level but FieldSchema is a UI/form concept, doesn't belong in the data layer
**Rationale:** `pages-component` is the lowest UI-tier package. Everything that needs FieldSchema already depends on it. No new dependency directions. The constraint "no re-exports" means every consumer imports directly from `@casehubio/pages-component`.
**Trade-offs:** `pages-component` gets a richer type than it currently uses internally. Acceptable — it's a type definition, not runtime code.
**Sources:** pages-component/src/model/form-input-types.ts:55, pages-component/src/model/displayer-types.ts:233, pages-ui-components/src/types.ts:6, dependency graph (pages-data ← pages-component ← pages-ui-components ← pages-property-palette)
**Exploration:** quick
**Status:** captured
