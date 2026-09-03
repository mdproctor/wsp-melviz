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
**Trade-offs:** Three new components to maintain. Mitigated by shared value protocol via FormValueMixin (D11).
**Sources:** mapFieldToComponentType (current leaf-only mapping), web-component-strategy protocol
**Exploration:** deep-analysis
**Status:** captured

## D4: Unified value protocol — FormValueProvider

**Choice:** All form components (leaf and composite) implement the FormValueProvider interface: `currentValue` getter (returns current value), `value` setter (sets initial value), `error` getter/setter (own validation error), and `validate(): boolean` (triggers validation, returns pass/fail). The interface is a concrete TypeScript interface in pages-component, providing a compile-time contract that custom renderers and composites must satisfy.
**Alternatives:**
- Different interfaces for leaf vs composite — breaks uniformity, every consumer needs type switching
- Path-based central store (like Redux-form) — over-engineered for a component-based system, creates coupling
- `errors` aggregation getter instead of per-component `error` — adds complexity; each component shows its own error in the DOM, parent only needs boolean pass/fail from `validate()`
**Rationale:** Uniform protocol means PagesSchemaForm and formScope treat all children identically. Value collection is recursive: call `currentValue` on each child, composite children recursively call their children. `validate()` provides the triggering mechanism for recursive validation. `error` is singular (the component's own validation feedback); child errors are displayed at the child level in the DOM tree, not aggregated upward.
**Trade-offs:** Custom renderer elements must implement the full protocol (including `validate()`). This is a stricter contract than the current duck-typed approach, but eliminates silent failures from missing methods.
**Sources:** PagesFormInput.currentValue (existing leaf protocol), readFieldValue/setFieldError (existing utilities), RovingTabindexMixin pattern (established mixin convention)
**Exploration:** quick
**Status:** revised — R1: formalized as TypeScript interface, added `validate(): boolean`, reconciled error model (singular `error` per component, boolean `validate()` for tree-wide pass/fail)

## D5: Pipeline integration — adapter layer, not form concern

**Choice:** Forms work with structured records. Pipeline integration is an adapter concern with three strategies: standalone (no pipeline, form emits structured record), JSON column (single column holding JSON), flat projection (leaf fields map to flat columns, nested sub-trees serialize to JSON). Adapter selection is automatic at form mount time: inspect the TypedDataSet's column structure — if individual columns exist for each top-level property, use flat projection; if a column type matches a JSON/TEXT shape for the schema root, use JSON column; if no pipeline, use standalone. Nested field changes are batched by the existing auto-save debouncing — composite re-emission (D14) produces committed events that enter the same batching pipeline as leaf field changes.
**Alternatives:**
- Extend TypedDataSet with OBJECT/ARRAY column types — much larger change rippling through entire pipeline for a form-specific need
- Dotted-path columns — pollutes column naming, arrays awkward, every pipeline consumer must understand the convention
- Manual adapter selection via `x-` extension or prop — adds configuration burden; the column structure already determines the correct strategy unambiguously
**Rationale:** Separation of concerns. The form renders and collects structured data. How that maps to storage is orthogonal. Adapter layer is thin and can evolve independently. Automatic selection from column structure avoids configuration and keeps the form/pipeline boundary clean.
**Trade-offs:** Nested field changes within a JSON column produce a full column-level update, not field-level granularity in the pipeline. Acceptable because auto-save already batches committed changes via `pages-form-commit`.
**Sources:** TypedDataSet (flat model), dataset-contract protocol, PagesSchemaForm.renderContent, activation.ts pages-field-change → pages-form-commit pipeline
**Exploration:** deep-analysis
**Status:** revised — R1: specified automatic adapter selection from column structure, clarified nested change batching via existing auto-save pipeline

## D6: formScope integration — composite fields as opaque

**Choice:** formScope treats object-group, array-group, and variant-group as single opaque fields. They register once with formScope via `pages-field-register`. formScope calls `currentValue` on them and gets the nested value. During `validateAll()`, formScope detects FormValueProvider conformance on registered elements and calls `validate()` instead of the leaf-only `validateField()` utility. No path flattening, no structural changes to formScope's flat field map.
**Alternatives:**
- Path-based field registration (address.street, address.city as separate formScope entries) — breaks formScope's flat map assumption, requires path parsing for value assembly
- formScope recursively validates nested schemas itself — breaks opacity, creates coupling between formScope and composite internals
**Rationale:** Preserves formScope's simplicity. Nesting concerns stay inside the composite components. formScope recognizes composites via FormValueProvider and delegates validation to them. The flat map model is preserved — composites are single entries, not exploded paths.
**Trade-offs:** formScope's `validateAll()` gains a FormValueProvider conformance check (does the element have `validate()`?) — a minimal change to support composite validation while maintaining opacity. formScope still cannot inspect individual nested fields — this is correct: the composite owns its sub-schema.
**Sources:** FormScopeState (current flat map), form-scope.ts, formScope design spec (#334 D7), FormValueProvider (D4)
**Exploration:** quick
**Status:** revised — R1: clarified validation path for composites (formScope calls `validate()` on FormValueProvider-conformant elements instead of `validateField()`)

## D7: Recursive validation

**Choice:** Each component validates its own sub-schema via the `validate(): boolean` method from FormValueProvider (D4). Object-group's `validate()` calls `validate()` on each child, then validates required sub-properties. Array-group's `validate()` calls `validate()` on each item, then validates minItems/maxItems/uniqueItems. Variant-group's `validate()` validates only the active variant. Top-level submit calls `validate()` on each top-level field via formScope's `validateAll()` or PagesSchemaForm's `submit()`.
**Alternatives:**
- Centralized validation (top-level validates entire schema tree) — requires deep knowledge of rendering tree, tight coupling, breaks when custom renderers are involved
- External `validateField()` for all types — current `validateField()` only handles leaf types (string/number constraints); extending it for objects/arrays/oneOf would create a monolithic validation function duplicating component-level knowledge
**Rationale:** `validate()` is the triggering mechanism. Each component owns its schema segment and its validation. Errors are set on each component's `error` property during `validate()`, surfacing at the deepest relevant level in the DOM. The existing `validateField()` utility is used internally by leaf components' `validate()` implementations — it stays leaf-only and does not need extension for composite types.
**Trade-offs:** Validation errors appear at each component level in the DOM. For form-level accessibility announcements, PagesSchemaForm's `submit()` counts how many top-level fields return `false` from `validate()` and announces the count.
**Sources:** validateField (existing leaf validation), PagesSchemaForm.submit (existing submit-time validation), FormValueProvider.validate() (D4)
**Exploration:** quick
**Status:** revised — R1: specified `validate()` as the triggering mechanism, clarified `validateField()` stays leaf-only (used internally by leaf `validate()` implementations)

## D8: $ref resolution — schema pre-processing

**Choice:** Schema references ($ref) are resolved at processing time, before rendering. A pre-processing utility walks the schema, resolves `$ref` pointers to inline definitions, and passes the resolved schema to PagesSchemaForm. Only local references (`#/$defs/...`) are supported — external references (URLs, relative file paths) are out of scope for a synchronous form renderer. Circular references are handled via cycle detection: the resolver tracks the set of $ref paths in the current resolution stack and stops when a cycle is detected, inserting a terminal empty schema at the cycle point.
**Alternatives:**
- Lazy resolution during rendering — adds complexity to every component, race conditions with async $ref resolution
- Use json-schema-ref-parser library — general-purpose with async/external support; the resolver is a single small pure function over local `$defs` — library weight is not justified
- Blanket depth limit (e.g., 5) — produces exponential expansion for schemas with multiple self-referencing properties (N^D copies where N is branching factor). Cycle detection is strictly superior: allows legitimate deep nesting while preventing infinite expansion
**Rationale:** Separation of concerns. Schema processing is a pure function. Rendering receives a fully resolved schema and doesn't need to handle references. Local-only scope is appropriate for form schemas embedded in page YAML. Cycle detection avoids the exponential expansion risk of depth-limited unrolling.
**Trade-offs:** Full schema must fit in memory after resolution. External $ref is not supported — schemas using external references must be pre-resolved before passing to PagesSchemaForm.
**Sources:** JSON Schema $ref specification
**Exploration:** quick
**Status:** revised — R1: scoped to local `#/$defs/` only, replaced blanket depth limit with cycle detection, explicitly rejected external $ref and library dependency

## D9: Custom renderers via x-renderer

**Choice:** Any schema node can specify `x-renderer: "custom-element-name"`. mapFieldToComponentType checks x-renderer first. The custom element must implement the FormValueProvider interface (D4). A runtime conformance check at field registration time verifies that the element exposes `currentValue` and `validate()` — if missing, a console.warn is logged identifying the non-conformant element. This prevents silent failures where broken custom renderers produce `undefined` values and skip validation.
**Alternatives:**
- Renderer registry (register callbacks per schema path) — more indirection, harder to discover, doesn't compose with YAML DSL
- No custom renderers (every type handled by built-in components) — too limiting for domain-specific needs (color pickers, maps, rich text)
- Hard error on non-conformance — too strict, would prevent progressive enhancement; a warning allows the element to still render while flagging the issue
**Rationale:** Uses the platform's existing custom element model. x- prefixed JSON Schema properties are standard extension points. Protocol-based contract is minimal and testable. Runtime conformance check catches integration errors early without breaking the page.
**Trade-offs:** Custom elements must be loaded before the form renders and must implement FormValueProvider. Standard web component lifecycle — not a new constraint.
**Sources:** FieldSchema `[key: x-${string}]` (already in the type), web-component-strategy protocol, FormValueProvider (D4)
**Exploration:** quick
**Status:** revised — R1: added runtime conformance check at field registration time with console.warn for non-conformant elements

## D10: Full scope — objects, arrays, oneOf, $ref, custom renderers, reordering

**Choice:** Single unified issue covers all nesting capabilities: nested objects, arrays (primitives and objects), oneOf discriminated unions, $ref resolution, x-renderer custom elements, array minItems/maxItems enforcement, array reordering (up/down buttons).
**Alternatives:**
- Split into separate issues (objects first, arrays later, oneOf later) — loses the unified architecture benefit; incremental approaches risk inconsistent designs
**Rationale:** The recursive model handles all cases uniformly. Designing them together ensures consistency. Implementation can still be phased (objects → arrays → oneOf → extras) within a single branch.
**Trade-offs:** Larger scope means longer branch. Mitigated by phased implementation within the branch.
**Sources:** Issue #222 scope discussion
**Exploration:** quick
**Status:** captured

## D11: Component base — FormValueMixin(LitElement)

**Choice:** The three new components (pages-object-group, pages-array-group, pages-variant-group) extend `FormValueMixin(LitElement)`, following the platform's established mixin pattern. FormValueMixin provides the shared FormValueProvider implementation: `currentValue` (delegates to abstract `collectValue()`), `value` setter (delegates to abstract `propagateValue()`), `error` getter/setter, and `validate()` (template method: validates children then calls abstract `validateSelf()`). Each composite implements the abstract methods for its specific schema type. Components use individual typed Lit properties (schema, label, editable, required) rather than PagesContentElement's generic `props` bag.
**Alternatives:**
- PagesContentElement base with interface overlay — PagesContentElement is presentational ("genuinely one-shot — props trigger render, no data machinery" per web-component-strategy protocol); its subtypes are PagesActionButton, PagesAlert, PagesLegend. Adding interactive form behavior via a separate interface causes triplication of value/error/validation logic across three components.
- Plain LitElement with FormValueProvider as a TypeScript interface only — no shared implementation; each component independently implements currentValue, value, error, validate() — guaranteed duplication
- PagesFormInput base — has `currentValue` and `dataSet` binding, but `dataSet` doesn't apply to nested sub-components that receive values from their parent, not from the pipeline
**Rationale:** FormValueMixin follows the established mixin convention: "Composition via mixins, not inheritance chains" (web-component-strategy protocol). Existing precedent: RovingTabindexMixin, FocusTrapMixin, LiveRegionMixin, KeyboardShortcutMixin (pages-primitives), DataSourceMixin (pages-component). The mixin provides shared implementation of the FormValueProvider contract while allowing each composite to specialize via abstract methods. Individual typed properties are more appropriate than a generic `props` bag for components with behavioral semantics (schema interpretation, value propagation, validation).
**Trade-offs:** Introduces a new mixin. Mitigated by following the exact pattern already used by 6+ existing mixins in the platform.
**Depends on:** D3 (three new component types), D4 (FormValueProvider protocol)
**Sources:** RovingTabindexMixin (pages-primitives/a11y), DataSourceMixin (pages-component/controller), web-component-strategy protocol (mixin convention, Lit base class hierarchy), PagesContentElement (pages-viz base class)
**Exploration:** quick
**Status:** revised — R1: replaced PagesContentElement inheritance with FormValueMixin(LitElement), following platform mixin convention; individual typed properties replace generic props bag

## D12: oneOf discriminator mechanism

**Choice:** pages-variant-group supports only discriminated unions — each variant sub-schema must contain a `const` value on a shared discriminator property. The discriminator property is auto-detected from `const` values in the variant sub-schemas' properties. It is rendered as a dropdown selector. Undiscriminated oneOf schemas (variants distinguished only by structural differences) are rejected at schema processing time with a clear error message. When the user switches variants, all entered data is cleared — no preservation or merging across variants.
**Alternatives:**
- Support undiscriminated oneOf (attempt validation against each variant) — this is a validation concept, not a form concept; the user has no clear way to select which variant they intend
- Discriminator via `x-discriminator` extension property — requires schema authors to add non-standard annotations; auto-detection from `const` values works with standard JSON Schema
- Preserve data on variant switch (merge where property names overlap) — risks type mismatches between variants; clear-on-switch is safer and more predictable
**Rationale:** Discriminated unions are the natural form concept — the user explicitly chooses which variant applies. Auto-detection from `const` keeps schemas standard. Clear-on-switch prevents stale data from one variant leaking into another.
**Trade-offs:** Undiscriminated oneOf is not supported. Schemas that use oneOf without const-based discrimination cannot use pages-variant-group and should use x-renderer for custom handling.
**Sources:** JSON Schema oneOf specification, D3 (pages-variant-group component)
**Exploration:** surfaced — implicit decision made explicit in R1 review
**Status:** captured

## D13: Array item identity for DOM reconciliation

**Choice:** pages-array-group uses synthetic monotonic keys for array items, internal to the component and not exposed in the data model. Each item receives a unique key on creation (monotonically incrementing counter). Reordering swaps items' positions without destroying/recreating DOM — Lit's `repeat()` directive uses the synthetic key for efficient reconciliation. `currentValue` returns items in visual (display) order, not insertion order.
**Alternatives:**
- Schema-driven key via `x-key` extension — leaks DOM rendering concerns into the schema; requires schema authors to understand LitElement internals
- Index-based keys — reordering destroys and recreates DOM for every item below the moved item, losing internal form state (cursor position, validation errors, focus)
- No keying — same DOM recycling penalty as index-based, plus Lit's default keying may produce incorrect reconciliation for heterogeneous item types
**Rationale:** Synthetic keys are invisible to the data model (keys never appear in `currentValue` output), avoid DOM destruction on reorder, and require no schema extensions. The monotonic counter guarantees uniqueness without collision risks.
**Trade-offs:** Keys are transient — they don't survive page reload. This is acceptable because array item identity is a rendering concern, not a persistence concern.
**Sources:** Lit repeat() directive documentation, D3 (pages-array-group component), D10 (reordering scope)
**Exploration:** surfaced — implicit decision made explicit in R1 review
**Status:** captured

## D14: Event propagation model for nested field changes

**Choice:** Composites use re-emission: they intercept `pages-field-change` events from their children, stop propagation, and re-emit a new `pages-field-change` event with the composite's own field name and full nested value from `currentValue`. Re-emission occurs only for `committed: true` events — uncommitted (keystroke-level) events are not re-emitted, preventing full-object serialization on every keystroke. This is consistent with D6's opacity model: formScope and the auto-save pipeline see the composite as a single opaque field changing its value.
**Alternatives:**
- Leaf events bubble through with dotted paths (`address.street`) — preserves granularity but contradicts D6's opacity model; formScope can't match dotted paths to registered fields
- No re-emission (let events bubble unmodified) — formScope receives events with child field names it doesn't recognize; auto-save receives changes for unregistered fields
- Re-emit all events (including uncommitted) — triggers full-object serialization and field-change processing on every keystroke in any nested field; unnecessary overhead
**Rationale:** Opacity requires that the composite mediates all communication between its children and the outside world. Re-emission on committed changes only avoids the performance concern of full-object serialization on every keystroke. The existing auto-save pipeline already batches committed changes via `pages-form-commit`.
**Trade-offs:** Nested field changes within a composite produce a full-object-level event, not a field-level diff. This means formScope's blur validation fires with the composite's full value, not the individual nested field. Acceptable because the composite's own `validate()` handles granular nested validation internally.
**Sources:** activation.ts (pages-field-change → pages-form-commit pipeline), formScope validateOnBlur pattern (#334 D7), D6 (opacity model)
**Exploration:** surfaced — implicit decision made explicit in R1 review
**Status:** captured
