# Forms Validation Design

**Date:** 2026-11-04
**Status:** Draft
**Scope:** Server-side JSON Schema constraint enforcement, structured validation error response, field-level error mapping, validate-on-blur
**Issues:** #609 (epic), #610, #611, #612

## Problem

The JSON Apps forms infrastructure has client-side validation (`PagesSchemaForm.validate()` → `validateField()`) but the server only enforces types via `CollectionSchema.validate()`. Three gaps:

1. **No server constraint enforcement.** `CollectionSchema` checks types only — `required`, `minLength`, `maxLength`, `pattern`, `minimum`, `maximum` are client-only. Direct API calls bypass all constraint validation.
2. **Opaque server errors.** `SchemaViolationException` carries a single string. The REST adapter returns `"HTTP 400: Bad Request"` without reading the response body. No path exists from server errors to field-level display.
3. **No live validation feedback.** `PagesSchemaForm.validateOnBlur` exists as a prop but isn't wired. `PagesFormInput` has `errorMessage` but nothing writes to it on blur.

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

The library provides full JSON Schema draft-07 support. `CollectionSchema.validate()` is updated:

```java
public SchemaValidationResult validate(JsonNode document) {
    List<FieldViolation> violations = new ArrayList<>();

    // 1. Existing type checks (preserved for backward compat)
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

`CollectionSchema.validate()` throws when `!result.valid()`:

```java
SchemaValidationResult result = validate(document);
if (!result.valid()) {
    String summary = result.violations().stream()
        .map(v -> v.field() + ": " + v.message())
        .collect(Collectors.joining("; "));
    throw new SchemaViolationException(summary, result.violations());
}
```

## 2. Structured Error Response (#611)

### 2.1 HTTP 422 Response

`PageResource` and `AppCatalogueResource` catch blocks updated:

```java
catch (SchemaViolationException e) {
    Map<String, Object> body = new LinkedHashMap<>();
    body.put("error", "VALIDATION_FAILED");
    body.put("violations", e.violations().stream()
        .map(v -> Map.of(
            "field", v.field(),
            "constraint", v.constraint(),
            "params", v.params(),
            "message", v.message()
        ))
        .toList());
    return Response.status(422).entity(body).build();
}
```

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

When `violations()` is empty (legacy exception path), fall back to current behaviour:

```java
if (e.violations().isEmpty()) {
    return Response.status(400).entity(Map.of("error", e.getMessage())).build();
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

`routeFieldErrors` finds form components on the originating `pagePath`:

```typescript
function routeFieldErrors(
  pagePath: string,
  errors: readonly FieldViolation[],
  registry: ComponentRegistry,
): void {
  // Group errors by field
  const byField = new Map<string, string>();
  for (const e of errors) {
    // Use first error message per field for display
    if (!byField.has(e.field)) {
      byField.set(e.field, e.message);
    }
  }

  for (const [id, entry] of registry) {
    if (entry.pagePath !== pagePath) continue;
    const el = entry.vizElement;

    // PagesSchemaForm — has setFieldErrors via palette
    if (isPagesSchemaForm(el)) {
      el.setServerErrors(byField);
      return;
    }

    // PagesFormInput — set errorMessage directly
    if (isPagesFormInput(el) && el.props?.field && byField.has(el.props.field)) {
      el.errorMessage = byField.get(el.props.field);
    }
  }
}
```

`PagesSchemaForm` gains a `setServerErrors(errors: Map<string, string>)` method that delegates to `this._palette.setFieldErrors()` — same path as client validation errors.

The `pages-save-error` event detail is extended to include field errors:

```typescript
interface SaveErrorDetail {
  readonly pagePath: string;
  readonly error: string;
  readonly fieldErrors?: readonly FieldViolation[];  // NEW
}
```

This allows external listeners (e.g. hosted components) to observe structured errors.

## 3. Validate-on-Blur (#612)

### 3.1 PagesSchemaForm

The `validateOnBlur` prop already exists in `SchemaFormProps` but isn't wired. The wiring:

1. The palette's field elements emit `blur` events
2. PagesSchemaForm listens for blur on the palette container
3. On blur, identifies the field from the event target
4. Calls `validateField(schema, value, required)` for that single field
5. Pushes the single-field error to the palette via `setFieldErrors()` with a single-entry map

```typescript
private _handleFieldBlur(e: FocusEvent) {
  if (!this.validateOnBlur) return;
  const field = this._identifyField(e.target);
  if (!field) return;
  const schema = this._resolvedSchema?.properties?.[field];
  if (!schema) return;
  const value = this._palette.getFieldValue(field);
  const required = this._resolvedSchema?.required?.includes(field) ?? false;
  const error = validateField(schema, value, required);
  this._palette.setFieldErrors(new Map([[field, error ?? undefined]]));
}
```

### 3.2 PagesFormInput

`PagesFormInput` gains:

1. A `validateOnBlur` property (default `true`)
2. A `fieldSchema` property — constraint metadata from the page's collection schema, set by the runtime during activation
3. A blur handler that validates and sets `errorMessage`

```typescript
@property({ type: Boolean }) validateOnBlur = true;
@property({ attribute: false }) fieldSchema?: FieldSchema;

protected _handleBlur() {
  if (!this.validateOnBlur || !this.fieldSchema) return;
  const error = validateField(this.fieldSchema, this.currentValue, this.required);
  this.errorMessage = error ?? undefined;
}
```

The runtime sets `fieldSchema` during activation when the page has a `dataScope` with a collection schema:

```typescript
if (isFormInput(vizEl) && vizEl.props?.field && collectionSchema) {
  vizEl.fieldSchema = collectionSchema.fields[vizEl.props.field];
}
```

### 3.3 Client validateField Extension

`validateField()` currently returns the first matching error. For consistency with the server (D5 — all violations), update it to optionally return all violations:

```typescript
export function validateField(
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): string | null;

export function validateFieldAll(
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): readonly FieldViolation[];
```

`validateField()` remains unchanged (first error, backward compatible). `validateFieldAll()` returns the full list using the same `FieldViolation` type as the server response. Forms choose which to call based on display needs.

### 3.4 Error Clearing

Field errors are cleared on the next valid input:
- **On blur:** if validation passes, clear any existing error
- **On focus:** if the field has a server error (from save), clear it — the user is about to edit
- **On successful save:** runtime clears all field errors on the originating form

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
| 422 response handling | `backend/data-json` | `PageResource.java`, `AppCatalogueResource.java` |
| `FieldViolation` interface (TS) | `pages-component` | `model/field-validation.ts` |
| `SaveResult` extension | `pages-runtime` | `save-adapter.ts` |
| REST adapter 422 parsing | `pages-runtime` | `adapters/rest-adapter.ts` |
| `routeFieldErrors` | `pages-runtime` | `site.ts` |
| `SaveErrorDetail` extension | `pages-runtime` | `site.ts` |
| `setServerErrors()` | `pages-viz` | `PagesSchemaForm.ts` |
| Blur validation wiring (schema form) | `pages-viz` | `PagesSchemaForm.ts` |
| Blur validation wiring (native form) | `pages-viz` | `PagesFormInput.ts` |
| `validateFieldAll()` | `pages-component` | `model/field-validation.ts` |
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

- `packages/pages-component/src/model/field-validation.ts` — client-side validateField() reference implementation
- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — form component with validateOnBlur prop
- `packages/pages-viz/src/form-inputs/PagesFormInput.ts` — abstract input base with errorMessage
- `backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java` — server validation (type-only)
- `backend/data/src/main/java/io/casehub/pages/data/SchemaViolationException.java` — current exception
- `backend/data-json/src/main/java/io/casehub/pages/data/json/PageResource.java` — HTTP error response
- `packages/pages-runtime/src/adapters/rest-adapter.ts` — save pipeline, error handling
- `packages/pages-runtime/src/site.ts` — save handler, pages-save-error dispatch
- `docs/specs/2026-06-18-native-forms-design.md` — native forms architecture (deferred validation)
- `docs/specs/issue-222-schema-form-nested/` — schema form nested forms spec
- Protocol PP-20260705-bac842 (`pages-event-contract`) — reserved events: pages-field-change, pages-save-error
- Protocol PP-20260907-951cbe (`generated-schemas-not-hand-written`) — Zod schema generation
- networknt/json-schema-validator — JSON Schema draft-07 library
- decisions.md — D1–D10 design decisions
