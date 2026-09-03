## D1: Rendering model — recursive components

**Choice:** Recursive component rendering. Each JSON Schema node maps to exactly one component type. PagesSchemaForm walks the schema tree, creates the right component at each node, and nesting is just recursion terminating at leaf inputs.
**Alternatives:**
- Flat rendering with dotted-path field names — simpler code but loses structural boundaries; can't treat nested groups as components with their own value/validation
- Schema flattening with visual-only grouping via CSS fieldsets — conflates visual and data concerns, arrays are deeply awkward with path indices
**Rationale:** JSON Schema is recursive; the renderer should be too. Preserves component boundaries, clean value composition, each level manages its own state.
**Trade-offs:** Deeper DOM trees for deeply nested schemas (mitigated by practical depth limits; most forms are 2-3 levels).
**Sources:** PagesSchemaForm.ts (current flat rendering), FieldSchema type (already supports nesting)
**Exploration:** deep-analysis
**Status:** captured

## D2: Data output shape — structured records

**Choice:** Forms always produce structured records matching the JSON Schema shape. `currentValue` returns natural JSON: `{ address: { street: "...", city: "..." }, tags: ["a", "b"] }`.
**Alternatives:**
- Flat records with dotted paths (`{ "address.street": "..." }`) — requires path encoding/decoding, arrays need index-based paths, every consumer must understand the convention
**Rationale:** Natural JSON is the universal interchange format. Consistent with JSON Schema semantics. Flat conversion is a utility if needed, not a form concern.
**Trade-offs:** Pipeline consumers expecting flat records need an adapter layer.
**Sources:** JSON Schema specification, issue #222 data model question
**Exploration:** deep-analysis
**Status:** captured

## D3: Three new component types

**Choice:** Three new Lit components: `pages-object-group` (fieldset for object sub-schemas), `pages-array-group` (list with add/remove for array sub-schemas), `pages-variant-group` (discriminated union for oneOf).
**Alternatives:**
- Single monolithic component handling all nesting — loses separation of concerns, becomes unwieldy
- No new components — PagesSchemaForm handles everything internally — violates single responsibility, can't compose nested groups with layout primitives
**Rationale:** Each schema shape has distinct rendering logic and UI controls. Separate components keep responsibilities clear and enable independent testing.
**Trade-offs:** Three new components to maintain. Mitigated by shared value protocol.
**Sources:** mapFieldToComponentType (current leaf-only mapping), web-component-strategy protocol
**Exploration:** deep-analysis
**Status:** captured

## D4: Unified value protocol — FormValueProvider

**Choice:** All form components (leaf and composite) implement the same interface: `currentValue` getter (returns current value), `value` setter (sets initial value), `error` getter/setter (validation feedback).
**Alternatives:**
- Different interfaces for leaf vs composite — breaks uniformity, every consumer needs type switching
- Path-based central store (like Redux-form) — over-engineered for a component-based system, creates coupling
**Rationale:** Uniform protocol means PagesSchemaForm and formScope treat all children identically. Value collection is recursive: call `currentValue` on each child, composite children recursively call their children.
**Trade-offs:** Custom renderer elements must implement the protocol — documented as a contract.
**Sources:** PagesFormInput.currentValue (existing leaf protocol), readFieldValue/setFieldError (existing utilities)
**Exploration:** quick
**Status:** captured

## D5: Pipeline integration — adapter layer, not form concern

**Choice:** Forms work with structured records. Pipeline integration is an adapter concern with three strategies: standalone (no pipeline, form emits structured record), JSON column (single column holding JSON), flat projection (leaf fields map to flat columns, nested sub-trees serialize to JSON).
**Alternatives:**
- Extend TypedDataSet with OBJECT/ARRAY column types — much larger change rippling through entire pipeline for a form-specific need
- Dotted-path columns — pollutes column naming, arrays awkward, every pipeline consumer must understand the convention
**Rationale:** Separation of concerns. The form renders and collects structured data. How that maps to storage is orthogonal. Adapter layer is thin and can evolve independently.
**Trade-offs:** Nested field changes within a JSON column produce a full column-level update, not field-level granularity in the pipeline. Acceptable because auto-save already batches.
**Sources:** TypedDataSet (flat model), dataset-contract protocol, PagesSchemaForm.renderContent
**Exploration:** deep-analysis
**Status:** captured

## D6: formScope integration — composite fields as opaque

**Choice:** formScope treats object-group, array-group, and variant-group as single opaque fields. They register once with formScope via `pages-field-register`. formScope calls `currentValue` on them and gets the nested value — no path flattening, no formScope structural changes.
**Alternatives:**
- Path-based field registration (address.street, address.city as separate formScope entries) — breaks formScope's flat map assumption, requires path parsing for value assembly
**Rationale:** Preserves formScope's simplicity. Nesting concerns stay inside the composite components. formScope only needs to recognize composite types in readFieldValue.
**Trade-offs:** formScope can't validate individual nested fields — validation is delegated to the composite component. This is correct: the composite owns its sub-schema.
**Sources:** FormScopeState (current flat map), form-scope.ts, formScope design spec (#334)
**Exploration:** quick
**Status:** captured

## D7: Recursive validation

**Choice:** Each component validates its own sub-schema. Object-group validates required sub-properties and delegates leaf validation to children. Array-group validates minItems/maxItems/uniqueItems and delegates item validation. Variant-group validates only the active variant. Top-level submit walks the tree.
**Alternatives:**
- Centralized validation (top-level validates entire schema tree) — requires deep knowledge of rendering tree, tight coupling, breaks when custom renderers are involved
**Rationale:** Mirrors the rendering tree. Each component owns its schema segment and its validation. Errors surface at the deepest relevant level.
**Trade-offs:** Validation errors must bubble up for form-level announcements (accessibility). Solved by each component exposing an `errors` getter that includes child errors.
**Sources:** validateField (existing leaf validation), PagesSchemaForm.submit (existing submit-time validation)
**Exploration:** quick
**Status:** captured

## D8: $ref resolution — schema pre-processing

**Choice:** Schema references ($ref) are resolved at processing time, before rendering. A pre-processing utility walks the schema, resolves $ref pointers to inline definitions, and passes the resolved schema to PagesSchemaForm. Circular references get a depth limit (default 5).
**Alternatives:**
- Lazy resolution during rendering — adds complexity to every component, race conditions with async $ref resolution
**Rationale:** Separation of concerns. Schema processing is a pure function. Rendering receives a fully resolved schema and doesn't need to handle references.
**Trade-offs:** Full schema must fit in memory after resolution. Acceptable for form schemas (not multi-MB documents).
**Sources:** JSON Schema $ref specification
**Exploration:** quick
**Status:** captured

## D9: Custom renderers via x-renderer

**Choice:** Any schema node can specify `x-renderer: "custom-element-name"`. mapFieldToComponentType checks x-renderer first. The custom element must implement the FormValueProvider protocol. Provides an escape hatch for domain-specific inputs without polluting the core mapping.
**Alternatives:**
- Renderer registry (register callbacks per schema path) — more indirection, harder to discover, doesn't compose with YAML DSL
- No custom renderers (every type handled by built-in components) — too limiting for domain-specific needs (color pickers, maps, rich text)
**Rationale:** Uses the platform's existing custom element model. x- prefixed JSON Schema properties are standard extension points. Protocol-based contract is minimal and testable.
**Trade-offs:** Custom elements must be loaded before the form renders. Standard web component lifecycle — not a new constraint.
**Sources:** FieldSchema `[key: x-${string}]` (already in the type), web-component-strategy protocol
**Exploration:** quick
**Status:** captured

## D10: Full scope — objects, arrays, oneOf, $ref, custom renderers, reordering

**Choice:** Single unified issue covers all nesting capabilities: nested objects, arrays (primitives and objects), oneOf discriminated unions, $ref resolution, x-renderer custom elements, array minItems/maxItems enforcement, array reordering (up/down buttons).
**Alternatives:**
- Split into separate issues (objects first, arrays later, oneOf later) — loses the unified architecture benefit; incremental approaches risk inconsistent designs
**Rationale:** The recursive model handles all cases uniformly. Designing them together ensures consistency. Implementation can still be phased (objects → arrays → oneOf → extras) within a single branch.
**Trade-offs:** Larger scope means longer branch. Mitigated by phased implementation within the branch.
**Sources:** Issue #222 scope discussion
**Exploration:** quick
**Status:** captured

## D11: Component base class — PagesContentElement

**Choice:** The three new components (pages-object-group, pages-array-group, pages-variant-group) extend PagesContentElement. They are pure renderers: receive values from their parent, not from the pipeline. PagesSchemaForm remains the sole pipeline participant.
**Alternatives:**
- Plain LitElement — lighter but loses the `props` convention that all pages-viz components share
- PagesFormInput — has `currentValue` and `dataSet`, but `dataSet` binding doesn't apply to nested sub-components that receive values from their parent
**Rationale:** PagesContentElement is "props trigger render, no data machinery" — exactly the contract for nested sub-components. The FormValueProvider protocol (currentValue/value/error) is implemented directly on each component as an additional interface.
**Trade-offs:** PagesContentElement provides `renderContent(props)` without a `dataset` parameter. Nested components receive their value via a separate `value` property, not through the PagesElement data pipeline. This is the correct separation.
**Depends on:** D3 (three new component types), D4 (FormValueProvider protocol)
**Sources:** PagesContentElement (pages-viz base class), web-component-strategy protocol (Lit base class hierarchy)
**Exploration:** quick
**Status:** captured
