# Decisions — Forms Validation (#609)

## D1: Validation scope — both form systems

**Choice:** Wire validation into both PagesSchemaForm (JSON Apps) and PagesFormInput (native page forms)
**Alternatives:**
- PagesSchemaForm only — simpler scope but leaves native forms without validation
- PagesSchemaForm + shared primitives — target one system but make infrastructure reusable
**Rationale:** Unified validation story across both form systems. Server constraints and structured errors are shared infrastructure; blur validation wires into both.
**Trade-offs:** Larger scope — PagesFormInput needs constraint awareness it doesn't currently have
**Sources:** native-forms-design.md §Validation (deferred constraints as future work), field-validation.ts, PagesFormInput.ts
**Exploration:** quick
**Status:** captured

## D2: Server constraint scope — full JSON Schema draft-07

**Choice:** Full JSON Schema draft-07 compliance on the server
**Alternatives:**
- Mirror client constraints only — parity but limited (8 keywords)
- Superset of client — more than client but hand-picked
**Rationale:** Server is the trust boundary. Full spec compliance means no gaps for direct API callers to exploit. Client validates a subset for fast UX feedback.
**Trade-offs:** Significant implementation scope for draft-07 edge cases (composition keywords, $ref, format) — mitigated by using a library (D3)
**Sources:** CollectionSchema.java (type-only validation), field-validation.ts (client subset)
**Exploration:** quick
**Status:** captured

## D3: Server validation implementation — library + adapter

**Choice:** Use networknt/json-schema-validator behind a CollectionSchema adapter
**Alternatives:**
- Library direct — couples callers to library error model
- Direct implementation — no dependency but massive scope for full draft-07
**Rationale:** Full draft-07 compliance without reimplementing the spec. Library stays encapsulated behind existing API. Adapter translates library errors to field-level contract.
**Trade-offs:** New dependency (~400KB). Adapter layer to maintain. Library updates may change error output.
**Sources:** CollectionSchema.java (existing API surface)
**Exploration:** deep-analysis
**Status:** captured

## D4: HTTP status code — 422 for validation failures

**Choice:** HTTP 422 Unprocessable Entity for schema validation failures
**Alternatives:**
- Keep HTTP 400 — no migration, client distinguishes by body structure
**Rationale:** Semantically correct — request is well-formed JSON but violates constraints. Client can branch on status code to distinguish validation from parse errors.
**Trade-offs:** Breaking change for any existing client expecting 400 for validation errors — but current errors are opaque strings, so no client is parsing them structurally
**Sources:** SchemaViolationException.java, PageResource.java (current 400 response)
**Exploration:** quick
**Status:** captured

## D5: Error detail level — all violations per field

**Choice:** Return every constraint violation for each field, not just the first
**Alternatives:**
- First violation per field — simpler but user fixes one error only to see the next
**Rationale:** More information, no loss. Client can display all or pick the most relevant.
**Trade-offs:** Slightly larger response payloads
**Sources:** issue #611 requirements
**Exploration:** quick
**Status:** captured

## D6: Error format — constraint codes + params

**Choice:** Machine-readable constraint code + params object + fallback message
**Alternatives:**
- Human-readable strings only — simpler but client can't distinguish types or localise
- Both as separate fields — redundant, codes+params already include a message
**Rationale:** Client controls message rendering. Codes enable localisation, conditional display, and programmatic handling.
**Trade-offs:** Client must maintain a constraint-code-to-message mapping (or use fallback message)
**Sources:** issue #611 requirements
**Exploration:** quick
**Status:** captured

## D7: Server validation result — structured record replaces opaque exception

**Choice:** New `SchemaValidationResult` record with `List<FieldViolation>`. `SchemaViolationException` updated to carry violations list.
**Alternatives:**
- Keep opaque exception, add field info as separate exception type — two exception paths to maintain
- Return violations directly, no exception — breaks existing try/catch flow in resources
**Rationale:** Single exception type carries structured data. Resource endpoints serialize to JSON. Backward compatible — message field still exists for non-structured consumers.
**Trade-offs:** SchemaViolationException constructor changes — callers need updating
**Sources:** SchemaViolationException.java, PageResource.java, AppCatalogueResource.java
**Exploration:** quick
**Status:** captured

## D8: Error response JSON shape — violations array at top level

**Choice:** `{"error": "VALIDATION_FAILED", "violations": [{"field", "constraint", "params", "message"}]}`
**Alternatives:**
- Field-keyed map `{"fields": {"name": [...]}}` — groups by field but less uniform for iteration
- Flat errors array without constraint codes — simpler but loses machine-readability (contradicts D6)
**Rationale:** Flat array is simplest to produce (direct serialization of FieldViolation list) and simplest to consume (filter by field on client). Groups naturally via `field` property.
**Trade-offs:** Client must group-by-field if it wants per-field display — trivial
**Sources:** D6 (constraint codes + params)
**Exploration:** quick
**Status:** captured

## D9: Error routing — runtime routes via pagePath to originating form

**Choice:** Runtime save handler checks `SaveResult.fieldErrors`, finds form components on the originating pagePath, routes errors via `setFieldErrors()` (PagesSchemaForm) or `errorMessage` (PagesFormInput)
**Alternatives:**
- Forms listen for `pages-save-error` and self-handle — decoupled but each form system reimplements routing
- Validation middleware before save — client validates twice (redundant with blur validation)
**Rationale:** Runtime already tracks pagePath in save pipeline. Single routing point serves both form systems. Consistent with how data is pushed to components.
**Trade-offs:** Runtime needs form-type awareness to pick the right error API
**Sources:** site.ts (save handler, pages-save-error dispatch), PagesPropertyPalette.setFieldErrors()
**Exploration:** quick
**Status:** captured

## D10: Validate-on-blur — wire both form systems

**Choice:** PagesSchemaForm: wire existing `validateOnBlur` prop to single-field `validateField()` on blur, push to palette. PagesFormInput: add `validateOnBlur` property (default true), call `validateField()` on blur, set `errorMessage`.
**Alternatives:**
- Submit-only validation — current state, no change needed but poor UX
- Debounced live validation (on input, not blur) — more responsive but noisy for partially typed values
**Rationale:** Blur is the standard UX pattern — validates when user moves away from field. Not noisy like on-input. Both systems have the error display infrastructure already.
**Trade-offs:** PagesFormInput needs access to constraint metadata from schema — requires schema-to-input propagation
**Depends on:** D1 (both form systems)
**Sources:** PagesSchemaForm.ts (validateOnBlur prop exists), PagesFormInput.ts (errorMessage property exists), field-validation.ts
**Exploration:** quick
**Status:** captured
