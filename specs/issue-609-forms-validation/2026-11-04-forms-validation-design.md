# Forms Validation Design

**Date:** 2026-11-04
**Status:** Draft
**Scope:** Server-side JSON Schema constraint enforcement, structured validation error response, field-level error mapping, validate-on-blur
**Issues:** #609 (epic), #610, #611, #612

## Problem

The JSON Apps forms infrastructure has client-side validation (`PagesSchemaForm.validate()` → `validateField()`) but the server only enforces types via `CollectionSchema.validate()`. Three gaps:

1. **No server constraint enforcement.** `CollectionSchema` checks types only — `required`, `minLength`, `maxLength`, `pattern`, `minimum`, `maximum` are client-only. Direct API calls bypass all constraint validation.
2. **Opaque server errors.** `SchemaViolationException` carries a single string. `MutationExceptionMapper` maps `SCHEMA_VIOLATION` to HTTP 500 with `{"error": message, "code": code}`. The REST adapter returns `"HTTP 400: Bad Request"` without reading the response body. No path exists from server errors to field-level display.
3. **Fragmented client validation.** Three separate `validateField` implementations exist with different constraint coverage. `PagesPropertyPalette` imports from `pages-ui-components` (most complete); `PagesSchemaForm` and `FormScopeState` import from `pages-component` (missing enum, multipleOf, array constraints); `activation.ts` has an inline copy (missing exclusiveMin/Max).
4. **Incomplete blur validation wiring.** `PagesPropertyPalette._handleChange()` validates unconditionally on every field change — blur validation partially works for schema forms. But it's not gated by `validateOnBlur`, the `FormScopeState` blur path uses the incomplete `pages-component` implementation, and `PagesFormInput` has `errorMessage` but no blur handler.

## Design Principles

1. **Server is the trust boundary** — full JSON Schema draft-07 enforcement on the server. Client validates a subset for fast UX feedback.
2. **Structured errors end-to-end** — machine-readable constraint codes from server to form field. No opaque strings.
3. **Both form systems** — validation works for `PagesSchemaForm` (JSON Apps) and `PagesFormInput` (native page forms).
4. **Existing patterns** — extend `SaveResult`, use `pages-save-error`, route via `pagePath`. No new event types.

## Architecture Overview

```
Client (blur)                    Client (submit)                Server (save)
─────────────                    ───────────────                ─────────────
PagesSchemaForm                  PagesSchemaForm.validate()     CollectionSchema.validate()
  blur → validateField()           ↓ all fields                  ↓ type + constraint checks
  ↓ single field error             ↓ Map<field, error>           ↓ networknt json-schema-validator
  palette.setFieldErrors()         palette.setFieldErrors()      ↓ SchemaValidationResult
                                                                 ↓ SchemaViolationException
PagesFormInput                   PagesFormInput                   ↓ HTTP 422 + violations JSON
  blur → validateField()           (via form submit)             REST adapter parses body
  ↓ this.errorMessage              ↓                             ↓ SaveResult.fieldErrors
                                                                 runtime routes to form
                                                                 ↓ setFieldErrors() / errorMessage
```

## 1. Server-Side Constraint Enforcement (#610)

### 1.1 Extend CollectionSchema with Constraint Metadata

`CollectionSchema` currently stores `Map<String, String> fieldTypes` — type name only, no constraints. The app.yaml parsing pipeline must preserve constraint metadata.

New record to carry per-field constraints:

```java
public record FieldConstraints(
    String type,
    boolean required,
    Integer minLength,
    Integer maxLength,
    String pattern,
    Number minimum,
    Number maximum,
    Number exclusiveMinimum,
    Number exclusiveMaximum,
    Number multipleOf,
    List<String> enumValues,
    String format
) {
    public static FieldConstraints typeOnly(String type) {
        return new FieldConstraints(type, false, null, null, null,
            null, null, null, null, null, null, null);
    }
}
```

`CollectionSchema` gains a new field:

```java
public record CollectionSchema(
    String collection,
    Map<String, String> fieldTypes,
    Map<String, Object> defaults,
    Map<String, String> refs,
    Map<String, FieldConstraints> fieldConstraints  // NEW
) {}
```

Backward compatibility: existing callers that construct `CollectionSchema` with 4 args continue to work via a static factory or compact constructor that defaults `fieldConstraints` to an empty map. The YAML/JSON parsing pipeline that builds `CollectionSchema` from app definitions is updated to extract constraint keywords into `FieldConstraints`.

### 1.2 JSON Schema Document Construction

A new `JsonSchemaBuilder` class converts `CollectionSchema.fieldConstraints()` into a JSON Schema document suitable for the networknt library:

```java
public class JsonSchemaBuilder {
    public static ObjectNode buildSchema(Map<String, FieldConstraints> constraints) {
        // Builds {"type": "object", "properties": {...}, "required": [...]}
        // Each field's constraints become JSON Schema keywords
    }
}
```

### 1.3 Validation via networknt/json-schema-validator

New Maven dependency:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>json-schema-validator</artifactId>
    <version>1.5.6</version>
</dependency>
```

The library provides full JSON Schema draft-07 support. Two methods — `validate()` preserves the existing throwing API for all current callers; `validateToResult()` returns structured results for callers that want them:

```java
// Existing API — throws on failure (backward compatible)
public void validate(JsonNode document) {
    SchemaValidationResult result = validateToResult(document);
    if (!result.valid()) {
        String summary = result.violations().stream()
            .map(v -> v.field() + ": " + v.message())
            .collect(Collectors.joining("; "));
        throw new SchemaViolationException(summary, result.violations());
    }
}

// NEW — returns structured result without throwing
public SchemaValidationResult validateToResult(JsonNode document) {
    List<FieldViolation> violations = new ArrayList<>();

    // 1. Existing type checks (preserved)
    validateTypes(document, violations);

    // 2. NEW: constraint validation via library
    if (!fieldConstraints.isEmpty()) {
        ObjectNode schema = JsonSchemaBuilder.buildSchema(fieldConstraints);
        JsonSchema jsonSchema = JsonSchemaFactory
            .getInstance(SpecVersion.VersionFlag.V7)
            .getSchema(schema);
        Set<ValidationMessage> messages = jsonSchema.validate(document);
        for (ValidationMessage msg : messages) {
            violations.add(toFieldViolation(msg));
        }
    }

    return new SchemaValidationResult(violations.isEmpty(), violations);
}
```

All existing callers (`PageResource`, `AppCatalogueResource`, `JsonDataProvider`, tests) continue calling `validate()` unchanged. The `SchemaViolationException` now carries structured violations, which `MutationExceptionMapper` uses to produce 422 responses (§2.1).

### 1.4 FieldViolation and SchemaValidationResult

```java
public record FieldViolation(
    String field,
    String constraint,
    Map<String, Object> params,
    String message
) {}

public record SchemaValidationResult(
    boolean valid,
    List<FieldViolation> violations
) {}
```

The `toFieldViolation()` adapter extracts:
- `field` — from `ValidationMessage.getPath()`, stripping the `$.` prefix
- `constraint` — from `ValidationMessage.getType()` (maps to JSON Schema keyword)
- `params` — extracted from the schema node for that field (e.g. `{"min": 5}` for minLength)
- `message` — from `ValidationMessage.getMessage()` (human-readable fallback)

### 1.5 Exception Update

`SchemaViolationException` gains a violations list:

```java
public class SchemaViolationException extends MutationException {
    private final List<FieldViolation> violations;

    public SchemaViolationException(String message, List<FieldViolation> violations) {
        super("SCHEMA_VIOLATION", message);
        this.violations = List.copyOf(violations);
    }

    // Backward-compatible constructor
    public SchemaViolationException(String message) {
        this(message, List.of());
    }

    public List<FieldViolation> violations() {
        return violations;
    }
}
```

`validate()` delegates to `validateToResult()` and throws when invalid — see §1.3.

## 2. Structured Error Response (#611)

### 2.1 HTTP 422 Response via MutationExceptionMapper

The global `MutationExceptionMapper` (`@Provider`, `ExceptionMapper<MutationException>`) is the right architectural location for exception-to-response mapping. Currently it maps `SCHEMA_VIOLATION` to `default → 500`. Update it to produce 422 with structured violations:

```java
case "SCHEMA_VIOLATION" -> {
    if (exception instanceof SchemaViolationException sve && !sve.violations().isEmpty()) {
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("error", "VALIDATION_FAILED");
        body.put("violations", sve.violations().stream()
            .map(v -> Map.of(
                "field", v.field(),
                "constraint", v.constraint(),
                "params", v.params(),
                "message", v.message()
            ))
            .toList());
        yield Response.status(422).entity(body).build();
    }
    // Legacy path — no structured violations
    yield Response.status(400).entity(Map.of("error", exception.getMessage())).build();
}
```

This gives ALL code paths the correct response — `PageResource`, `AppCatalogueResource`, `JsonDataProvider.validateAndApplyDefaults()` (which has no local catch), and any future callers. The local `SchemaViolationException` catch blocks in `PageResource` and `AppCatalogueResource` can be removed — the mapper handles it.

Response body example:

```json
{
  "error": "VALIDATION_FAILED",
  "violations": [
    {
      "field": "name",
      "constraint": "minLength",
      "params": {"min": 5},
      "message": "must be at least 5 characters"
    },
    {
      "field": "name",
      "constraint": "pattern",
      "params": {"pattern": "^[A-Z]"},
      "message": "must match pattern ^[A-Z]"
    },
    {
      "field": "priority",
      "constraint": "required",
      "params": {},
      "message": "is required"
    }
  ]
}
```

### 2.2 SaveResult Extension (TypeScript)

```typescript
export interface FieldViolation {
  readonly field: string;
  readonly constraint: string;
  readonly params: Readonly<Record<string, unknown>>;
  readonly message: string;
}

export interface SaveResult {
  readonly success: boolean;
  readonly error?: string;
  readonly updatedRecord?: Readonly<Record<string, unknown>>;
  readonly fieldErrors?: readonly FieldViolation[];  // NEW
}
```

### 2.3 REST Adapter Update

The REST adapter reads the response body on HTTP 422:

```typescript
if (!response.ok) {
  if (response.status === 422) {
    const body = await response.json();
    if (body.error === "VALIDATION_FAILED" && Array.isArray(body.violations)) {
      return {
        success: false,
        error: "VALIDATION_FAILED",
        fieldErrors: body.violations.map((v: any) => ({
          field: v.field,
          constraint: v.constraint,
          params: v.params ?? {},
          message: v.message ?? "",
        })),
      };
    }
  }
  return { success: false, error: `HTTP ${response.status}: ${response.statusText}` };
}
```

### 2.4 Error Routing in Runtime

The save handler in `site.ts` routes field errors to the originating form after `SaveResult` is received:

```typescript
if (!result.success && result.fieldErrors?.length) {
  routeFieldErrors(pagePath, result.fieldErrors, registry);
}
```

`routeFieldErrors` finds form components on the originating `pagePath` and passes structured violations through — not collapsed to flat strings:

```typescript
function routeFieldErrors(
  pagePath: string,
  errors: readonly FieldViolation[],
  registry: ComponentRegistry,
): void {
  // Group errors by field, preserving all violations
  const byField = new Map<string, readonly FieldViolation[]>();
  for (const e of errors) {
    const existing = byField.get(e.field) ?? [];
    byField.set(e.field, [...existing, e]);
  }

  for (const [id, entry] of registry) {
    if (entry.pagePath !== pagePath) continue;
    const el = entry.vizElement;

    // PagesSchemaForm — has setServerErrors via palette
    if (isPagesSchemaForm(el)) {
      el.setServerErrors(byField);
      return;
    }

    // PagesFormInput — set errorMessage from first violation
    if (isPagesFormInput(el) && el.props?.field && byField.has(el.props.field)) {
      const violations = byField.get(el.props.field)!;
      el.errorMessage = violations[0].message;
      el.serverViolations = violations;  // full structured data available
    }
  }
}
```

`PagesSchemaForm` gains a `setServerErrors(errors: Map<string, readonly FieldViolation[]>)` method. The palette's `setFieldErrors()` currently accepts `Map<string, string | undefined>` (flat messages). Two options:

1. **Extend palette** to accept `Map<string, FieldViolation[] | string | undefined>` — renders first message for display, stores full violations for programmatic access
2. **Collapse at the palette boundary** — `setServerErrors` extracts messages for display, stores raw violations separately for the `pages-save-error` event

Option 1 is cleaner — the palette tracks error source (server vs client) via the `FieldViolation` type. A `FieldViolation[]` value means server error; a `string` means client validation error. This resolves R1-08 (distinguishing error sources for clearing).

The `pages-save-error` event detail is extended to include field errors:

```typescript
interface SaveErrorDetail {
  readonly pagePath: string;
  readonly error: string;
  readonly fieldErrors?: readonly FieldViolation[];  // NEW
}
```

This allows external listeners (e.g. hosted components) to observe structured errors.

## 3. Validate-on-Blur and validateField Consolidation (#612)

### 3.1 Consolidate validateField Implementations

Three separate `validateField` implementations exist with different constraint coverage:

| Implementation | Location | Constraints |
|----------------|----------|-------------|
| `validateField` | `pages-ui-components/validation/validate-field.ts` | **Most complete:** required, pattern, minLength, maxLength, minimum, maximum, exclusiveMinimum, exclusiveMaximum, enum, multipleOf, minItems, maxItems |
| `validateField` | `pages-component/model/field-validation.ts` | Missing: enum, multipleOf, minItems, maxItems |
| `validateFormField` | `pages-runtime/activation.ts` (inline) | Missing: exclusiveMin/Max, enum, multipleOf, array constraints |

**Consolidation:** The `pages-ui-components` version is canonical. All other call sites are updated to import from it:

- `pages-component/model/field-validation.ts` — re-exports from `pages-ui-components` for backward compat (existing importers don't break), marked deprecated
- `pages-runtime/activation.ts` — inline `validateFormField()` replaced with import from `pages-ui-components`
- `pages-runtime/form-scope.ts` — `FormScopeState.validateField()` delegates to `pages-ui-components` version

This ensures all validation paths check the same constraints. The `pages-ui-components` version also gains `validateFieldAll()`:

```typescript
export function validateFieldAll(
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): readonly FieldViolation[];
```

Returns all violations using the same `FieldViolation` type as the server response.

### 3.2 PagesSchemaForm — Gate Existing Validation

The palette's `_handleChange()` already validates on every field change unconditionally. The `validateOnBlur` prop exists but doesn't gate this behaviour. Three changes:

1. **Gate palette validation:** `_handleChange()` should check `validateOnBlur` on the parent form before validating. When `validateOnBlur` is false, the palette only validates on explicit `validate()` calls (submit).

2. **Existing tests pass:** `schema-form.test.ts` has blur validation tests (lines 498, 521, 546) that confirm the palette validates on committed changes. These tests remain valid — the change is to make the `validateOnBlur` gate explicit rather than unconditional.

3. **No new blur handler needed in PagesSchemaForm.** The palette's `_handleChange()` already fires on blur for text inputs (the `change` event fires on blur). The work is gating it properly, not adding new wiring.

### 3.3 PagesFormInput — Wire Through FormScopeState

`FormScopeState` (`pages-runtime/form-scope.ts`) already provides:
- `validateOnBlur` parameter in constructor
- `validateField(field, value)` method
- Blur listener in `activation.ts` (line 243) when `__formScopeValidateOnBlur` is true

The gap: `FormScopeState.validateField()` delegates to the incomplete `pages-component` version. After consolidation (§3.1), it uses the canonical `pages-ui-components` version — which closes the constraint coverage gap.

Additional wiring for `PagesFormInput` outside form-scope contexts (standalone native form inputs):

```typescript
@property({ type: Boolean }) validateOnBlur = true;

protected _handleBlur() {
  if (!this.validateOnBlur || !this.fieldSchema) return;
  const error = validateField(this.fieldSchema, this.currentValue, this.required);
  this.errorMessage = error ?? undefined;
}
```

`fieldSchema` is set by the runtime during activation when the page has a `dataScope`:

```typescript
if (isFormInput(vizEl) && vizEl.props?.field && collectionSchema) {
  vizEl.fieldSchema = collectionSchema.fields[vizEl.props.field];
}
```

When a `PagesFormInput` is inside a form-scope, `FormScopeState` handles blur validation (existing mechanism). When standalone (no form-scope), the input's own `_handleBlur` handles it. The two paths don't conflict — `FormScopeState.registerField()` would not be called for standalone inputs.

### 3.4 Error Clearing and Source Tracking

The palette stores errors in `_errors: Map<string, string>` with no source tracking. After §2.4's changes, the palette can distinguish error sources:

- **Client validation error:** `string` value in the error map (from `_handleChange()` / `validateField()`)
- **Server validation error:** `FieldViolation[]` value (from `setServerErrors()`)

Clearing rules:
- **On blur (client validation passes):** clear client error for that field. Server errors persist until the next save attempt.
- **On focus:** clear server errors for that field — the user is about to edit, so the server error is stale.
- **On successful save:** runtime clears all field errors (client and server) on the originating form.
- **On new server error:** overwrites any existing error (client or server) for that field.

## 4. FieldSchema Availability

A cross-cutting concern: `validateField()` requires a `FieldSchema` object with constraint metadata. This already exists in the `PagesSchemaForm` context (JSON Apps forms generate FieldSchema from their schema definitions). For native `PagesFormInput`, the constraint metadata needs to reach the input element.

**Source:** Collection schemas defined in app.yaml carry constraint keywords. The YAML parsing pipeline already extracts them into `FieldSchema` on the client side for `PagesSchemaForm`. For native form inputs, the same pipeline provides the schema — the runtime passes it through during activation.

**Server side:** The same app.yaml definitions are parsed by the backend into `CollectionSchema`. Extending `CollectionSchema` with `FieldConstraints` (§1.1) ensures the server has the same constraint metadata.

## Testing Strategy

### Server (#610)
- Unit tests for `JsonSchemaBuilder` — verify generated JSON Schema matches FieldConstraints
- Unit tests for `CollectionSchema.validate()` — each constraint type with valid/invalid values
- Unit tests for `toFieldViolation()` adapter — verify field path extraction, constraint mapping
- Integration test: full save request with constraint violations → 422 response

### Error Contract (#611)
- Integration test: 422 response body matches contract schema
- REST adapter unit test: parse 422 body → SaveResult.fieldErrors populated
- Runtime unit test: routeFieldErrors delivers to PagesSchemaForm.setServerErrors()
- Runtime unit test: routeFieldErrors delivers to PagesFormInput.errorMessage
- E2E test: submit form with invalid data → field-level error displayed

### Validate-on-blur (#612)
- PagesSchemaForm unit test: blur triggers single-field validation, error displayed
- PagesFormInput unit test: blur triggers validation, errorMessage set
- Unit test: error cleared on focus (server errors)
- Unit test: error cleared on valid blur
- E2E test: tab through form fields → errors appear/disappear on blur

## Package Placement

| Concern | Package | File |
|---------|---------|------|
| `FieldConstraints` record | `backend/data` | `CollectionSchema.java` (inner record) |
| `FieldViolation` record | `backend/data` | `FieldViolation.java` (new) |
| `SchemaValidationResult` record | `backend/data` | `SchemaValidationResult.java` (new) |
| `JsonSchemaBuilder` | `backend/data` | `JsonSchemaBuilder.java` (new) |
| `SchemaViolationException` update | `backend/data` | `SchemaViolationException.java` |
| `MutationExceptionMapper` update | `backend/data` | `MutationExceptionMapper.java` |
| Remove local catch blocks | `backend/data-json` | `PageResource.java`, `AppCatalogueResource.java` |
| `FieldViolation` interface (TS) | `pages-ui-components` | `validation/validate-field.ts` |
| `validateFieldAll()` | `pages-ui-components` | `validation/validate-field.ts` |
| Deprecate duplicate validateField | `pages-component` | `model/field-validation.ts` (re-export) |
| Remove inline validateFormField | `pages-runtime` | `activation.ts` |
| Update FormScopeState import | `pages-runtime` | `form-scope.ts` |
| `SaveResult` extension | `pages-runtime` | `save-adapter.ts` |
| REST adapter 422 parsing | `pages-runtime` | `adapters/rest-adapter.ts` |
| `routeFieldErrors` | `pages-runtime` | `site.ts` |
| `SaveErrorDetail` extension | `pages-runtime` | `site.ts` |
| `setServerErrors()` | `pages-viz` | `PagesSchemaForm.ts` |
| Palette error source tracking | `pages-viz` | `pages-property-palette.ts` |
| Gate palette validation behind validateOnBlur | `pages-viz` | `pages-property-palette.ts` |
| Blur validation (native form) | `pages-viz` | `PagesFormInput.ts` |
| networknt dependency | `backend/data` | `pom.xml` |

## Out of Scope

| Item | Why deferred |
|------|-------------|
| Cross-field validation | Requires a rule engine beyond JSON Schema per-field constraints |
| Async validation (e.g. uniqueness) | Server round-trip on blur adds latency; needs debounce strategy |
| Custom error messages in schema | JSON Schema `errorMessage` keyword is draft-07 extension, not standard |
| Localisation | Constraint codes + params enable it; message catalog is a separate concern |
| Nested object validation | CollectionSchema is flat (field→type); nested schemas need recursive support |

## References

- `packages/pages-ui-components/src/validation/validate-field.ts` — canonical validateField() (most complete)
- `packages/pages-component/src/model/field-validation.ts` — duplicate validateField() (to deprecate)
- `packages/pages-runtime/src/activation.ts:35-51` — inline validateFormField() (to remove)
- `packages/pages-runtime/src/form-scope.ts` — FormScopeState blur validation infrastructure
- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — form component with validateOnBlur prop
- `packages/pages-viz/src/form-inputs/PagesFormInput.ts` — abstract input base with errorMessage
- `packages/pages-viz/src/pages-property-palette.ts` — palette with _handleChange() (unconditional validation)
- `backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java` — server validation (type-only)
- `backend/data/src/main/java/io/casehub/pages/data/SchemaViolationException.java` — current exception
- `backend/data/src/main/java/io/casehub/pages/data/MutationExceptionMapper.java` — global exception→response mapper
- `backend/data/src/main/java/io/casehub/pages/data/json/JsonDataProvider.java` — validateAndApplyDefaults (no local catch)
- `backend/data-json/src/main/java/io/casehub/pages/data/json/PageResource.java` — local catch blocks (to remove)
- `packages/pages-runtime/src/adapters/rest-adapter.ts` — save pipeline, error handling
- `packages/pages-runtime/src/site.ts` — save handler, pages-save-error dispatch
- `docs/specs/2026-06-18-native-forms-design.md` — native forms architecture (deferred validation)
- Protocol PP-20260705-bac842 (`pages-event-contract`) — reserved events: pages-field-change, pages-save-error
- Protocol PP-20260907-951cbe (`generated-schemas-not-hand-written`) — Zod schema generation
- networknt/json-schema-validator — JSON Schema draft-07 library
- decisions.md — D1–D10 design decisions
