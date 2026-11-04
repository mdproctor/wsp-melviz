# Context Notes — Forms Validation (#609)

## Current Validation Architecture

### Client Side (TypeScript)

**`field-validation.ts`** (`pages-component`) — pure function `validateField(schema, value, required)` returns a string error message or null. Checks: `required`, `pattern`, `minLength`, `maxLength`, `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`. Constraint keywords come from the `FieldSchema` interface which mirrors JSON Schema (`form-input-types.ts`).

**`PagesSchemaForm.ts`** (`pages-viz`) — Lit web component. `validate()` iterates all fields against the resolved schema, calls `validateField()` per field, collects errors into a `Map<string, string | undefined>`, and pushes them to the palette via `setFieldErrors()`. `submit()` calls `validate()` first; if invalid, announces "Validation errors" via ARIA live region and returns null.

**`PagesFormInput.ts`** (`pages-viz`) — abstract base class for form inputs. Has an `errorMessage` property and `required` flag. No blur-based validation logic currently.

**`SchemaFormProps`** — already includes a `validateOnBlur?: boolean` property (not yet wired).

### Server Side (Java)

**`CollectionSchema.java`** (`backend/data`) — `validate(JsonNode)` checks **type only**: string, number, boolean, array, object, enum, date, ref. No constraint validation (no minLength, pattern, minimum, etc.). Skips null/missing values entirely.

**`SchemaViolationException.java`** — extends `MutationException(code, message)`. Code is always `"SCHEMA_VIOLATION"`. Message is a single opaque human-readable string.

**`MutationException.java`** — base `RuntimeException` with a `code` field.

### Error Flow (Server → Client)

**`PageResource.java`** and **`AppCatalogueResource.java`** catch `SchemaViolationException` and return HTTP 400 with `{"error": "<message>"}`. The error is a single string — no field name, no structured breakdown.

**`rest-adapter.ts`** (`pages-runtime`) — `SaveResult` has `success: boolean` and `error?: string`. On non-OK response, returns `{success: false, error: "HTTP 400: Bad Request"}`. Does **not** parse the response body for error details.

## Gaps

1. **Server validates types but not constraints** — `required`, `minLength`, `maxLength`, `pattern`, `minimum`, `maximum` are client-only. Direct API calls bypass all constraint enforcement.
2. **Server errors are opaque** — a single string with no field identification. No way to map back to specific form fields.
3. **REST adapter discards error body** — uses `response.statusText` instead of parsing the JSON error response.
4. **No validate-on-blur** — `SchemaFormProps.validateOnBlur` exists but isn't wired. Validation only runs on submit.
