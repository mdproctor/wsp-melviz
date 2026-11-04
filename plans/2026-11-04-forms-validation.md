# Forms Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #609 — Epic: Forms Validation
**Issue group:** #609, #610, #611, #612

**Goal:** Close the validation trust gap — server enforces full JSON Schema draft-07 constraints, returns structured field-level errors, and both form systems display them inline with blur-time feedback.

**Architecture:** Server uses networknt/json-schema-validator behind a CollectionSchema adapter. Structured `FieldViolation` objects flow from server (422 response) through REST adapter into `SaveResult.fieldErrors`, then routed to form components via `routeFieldErrors()`. Client-side `validateField` is consolidated to one canonical implementation in `pages-ui-components`. Blur validation gates through `validateOnBlur` prop.

**Tech Stack:** Java 21 / Quarkus (backend), TypeScript / Lit (frontend), networknt json-schema-validator 1.5.x, Vitest (TS tests), JUnit + AssertJ (Java tests)

## Global Constraints

- networknt json-schema-validator must use draft-07 (`SpecVersion.VersionFlag.V7`)
- HTTP 422 for structured validation errors; HTTP 400 preserved for legacy opaque errors
- `FieldViolation` shape is identical in Java and TypeScript (field, constraint, params, message)
- All `validateField` call sites must use the canonical `pages-ui-components` version after consolidation
- IntelliJ MCP required for structural editing during execution

---

## Batch 1: Server Validation Infrastructure (#610)

### Task 1: FieldViolation, SchemaValidationResult, FieldConstraints records + networknt dependency

**Files:**
- Create: `backend/data/src/main/java/io/casehub/pages/data/FieldViolation.java`
- Create: `backend/data/src/main/java/io/casehub/pages/data/SchemaValidationResult.java`
- Modify: `backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java` (add FieldConstraints inner record, add fieldConstraints field, backward-compat constructor)
- Modify: `backend/data/pom.xml` (add networknt dependency)
- Test: `backend/data/src/test/java/io/casehub/pages/data/CollectionSchemaTest.java` (update constructor calls)

**Interfaces:**
- Produces: `FieldViolation(String field, String constraint, Map<String, Object> params, String message)` — used by Task 2, Task 3
- Produces: `SchemaValidationResult(boolean valid, List<FieldViolation> violations)` — used by Task 2
- Produces: `FieldConstraints` inner record on `CollectionSchema` — used by Task 2
- Produces: `CollectionSchema` backward-compat constructor (4 args → defaults `fieldConstraints` to empty map) — used by all existing callers

- [ ] **Step 1: Add networknt dependency to pom.xml**

Add to `backend/data/pom.xml` `<dependencies>` section:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>json-schema-validator</artifactId>
</dependency>
```

Add version to parent pom `<dependencyManagement>` if not already managed. Check latest 1.5.x version:

```bash
curl -s "https://search.maven.org/solrsearch/select?q=g:com.networknt+a:json-schema-validator&rows=1&wt=json" | python3 -c "import sys,json; print(json.load(sys.stdin)['response']['docs'][0]['latestVersion'])"
```

- [ ] **Step 2: Create FieldViolation record**

Create `backend/data/src/main/java/io/casehub/pages/data/FieldViolation.java`:

```java
package io.casehub.pages.data;

import java.util.Map;

public record FieldViolation(
    String field,
    String constraint,
    Map<String, Object> params,
    String message
) {
    public FieldViolation {
        params = params != null ? Map.copyOf(params) : Map.of();
    }
}
```

- [ ] **Step 3: Create SchemaValidationResult record**

Create `backend/data/src/main/java/io/casehub/pages/data/SchemaValidationResult.java`:

```java
package io.casehub.pages.data;

import java.util.List;

public record SchemaValidationResult(
    boolean valid,
    List<FieldViolation> violations
) {
    public SchemaValidationResult {
        violations = List.copyOf(violations);
    }

    public static SchemaValidationResult ok() {
        return new SchemaValidationResult(true, List.of());
    }
}
```

- [ ] **Step 4: Add FieldConstraints inner record and fieldConstraints field to CollectionSchema**

Read `CollectionSchema.java` to see the exact current record definition. Add `FieldConstraints` as a public inner record, add `fieldConstraints` as a new field. Add a backward-compatible static factory:

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

The `CollectionSchema` record becomes:

```java
public record CollectionSchema(
    String collection,
    Map<String, String> fieldTypes,
    Map<String, Object> defaults,
    Map<String, String> refs,
    Map<String, FieldConstraints> fieldConstraints
) {
    // Backward-compat: existing 4-arg callers
    public CollectionSchema(String collection, Map<String, String> fieldTypes,
                            Map<String, Object> defaults, Map<String, String> refs) {
        this(collection, fieldTypes, defaults, refs, Map.of());
    }
}
```

- [ ] **Step 5: Update existing CollectionSchemaTest constructor calls**

Read `CollectionSchemaTest.java`. All existing `new CollectionSchema(...)` calls use 4 args — they will continue to work with the backward-compat constructor. Verify by running:

```bash
/opt/homebrew/bin/mvn -pl backend/data test -Dtest=CollectionSchemaTest -q
```

Expected: all existing tests pass.

- [ ] **Step 6: Commit**

```bash
git add backend/data/src/main/java/io/casehub/pages/data/FieldViolation.java backend/data/src/main/java/io/casehub/pages/data/SchemaValidationResult.java backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java backend/data/pom.xml
git commit -m "feat(#610): add FieldViolation, SchemaValidationResult, FieldConstraints + networknt dep Refs #610"
```

### Task 2: JsonSchemaBuilder + CollectionSchema.validateToResult()

**Files:**
- Create: `backend/data/src/main/java/io/casehub/pages/data/JsonSchemaBuilder.java`
- Create: `backend/data/src/test/java/io/casehub/pages/data/JsonSchemaBuilderTest.java`
- Modify: `backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java` (add validateToResult, update validate)
- Modify: `backend/data/src/main/java/io/casehub/pages/data/SchemaViolationException.java` (add violations list)
- Modify: `backend/data/src/test/java/io/casehub/pages/data/CollectionSchemaTest.java` (add constraint tests)

**Interfaces:**
- Consumes: `FieldViolation`, `SchemaValidationResult`, `FieldConstraints` from Task 1
- Produces: `JsonSchemaBuilder.buildSchema(Map<String, FieldConstraints>)` → `ObjectNode`
- Produces: `CollectionSchema.validateToResult(JsonNode)` → `SchemaValidationResult`
- Produces: `SchemaViolationException.violations()` → `List<FieldViolation>`

- [ ] **Step 1: Write failing tests for JsonSchemaBuilder**

Create `backend/data/src/test/java/io/casehub/pages/data/JsonSchemaBuilderTest.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class JsonSchemaBuilderTest {

    @Test
    void buildsSchemaWithStringConstraints() {
        var constraints = Map.of("name", new CollectionSchema.FieldConstraints(
            "string", true, 2, 50, "^[A-Z]", null, null, null, null, null, null, null));
        ObjectNode schema = JsonSchemaBuilder.buildSchema(constraints);

        assertThat(schema.path("type").asText()).isEqualTo("object");
        assertThat(schema.path("required").toString()).contains("name");
        ObjectNode nameProps = (ObjectNode) schema.path("properties").path("name");
        assertThat(nameProps.path("type").asText()).isEqualTo("string");
        assertThat(nameProps.path("minLength").asInt()).isEqualTo(2);
        assertThat(nameProps.path("maxLength").asInt()).isEqualTo(50);
        assertThat(nameProps.path("pattern").asText()).isEqualTo("^[A-Z]");
    }

    @Test
    void buildsSchemaWithNumericConstraints() {
        var constraints = Map.of("age", new CollectionSchema.FieldConstraints(
            "number", false, null, null, null, 0, 150, null, null, null, null, null));
        ObjectNode schema = JsonSchemaBuilder.buildSchema(constraints);

        ObjectNode ageProps = (ObjectNode) schema.path("properties").path("age");
        assertThat(ageProps.path("type").asText()).isEqualTo("number");
        assertThat(ageProps.path("minimum").asInt()).isEqualTo(0);
        assertThat(ageProps.path("maximum").asInt()).isEqualTo(150);
    }

    @Test
    void buildsSchemaWithEnumConstraints() {
        var constraints = Map.of("status", new CollectionSchema.FieldConstraints(
            "string", false, null, null, null, null, null, null, null, null,
            List.of("OPEN", "CLOSED"), null));
        ObjectNode schema = JsonSchemaBuilder.buildSchema(constraints);

        ObjectNode statusProps = (ObjectNode) schema.path("properties").path("status");
        assertThat(statusProps.path("enum").toString()).contains("OPEN", "CLOSED");
    }

    @Test
    void omitsNullConstraints() {
        var constraints = Map.of("desc", CollectionSchema.FieldConstraints.typeOnly("string"));
        ObjectNode schema = JsonSchemaBuilder.buildSchema(constraints);

        ObjectNode descProps = (ObjectNode) schema.path("properties").path("desc");
        assertThat(descProps.has("minLength")).isFalse();
        assertThat(descProps.has("pattern")).isFalse();
        assertThat(schema.path("required").isEmpty()).isTrue();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
/opt/homebrew/bin/mvn -pl backend/data test -Dtest=JsonSchemaBuilderTest -q
```

Expected: compilation error — `JsonSchemaBuilder` does not exist.

- [ ] **Step 3: Implement JsonSchemaBuilder**

Create `backend/data/src/main/java/io/casehub/pages/data/JsonSchemaBuilder.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import java.util.Map;

public final class JsonSchemaBuilder {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private JsonSchemaBuilder() {}

    public static ObjectNode buildSchema(Map<String, CollectionSchema.FieldConstraints> constraints) {
        ObjectNode schema = MAPPER.createObjectNode();
        schema.put("type", "object");
        ObjectNode properties = schema.putObject("properties");
        ArrayNode required = schema.putArray("required");

        for (var entry : constraints.entrySet()) {
            String fieldName = entry.getKey();
            CollectionSchema.FieldConstraints fc = entry.getValue();
            ObjectNode prop = properties.putObject(fieldName);

            if (fc.type() != null) prop.put("type", mapType(fc.type()));
            if (fc.minLength() != null) prop.put("minLength", fc.minLength());
            if (fc.maxLength() != null) prop.put("maxLength", fc.maxLength());
            if (fc.pattern() != null) prop.put("pattern", fc.pattern());
            if (fc.minimum() != null) prop.put("minimum", fc.minimum().doubleValue());
            if (fc.maximum() != null) prop.put("maximum", fc.maximum().doubleValue());
            if (fc.exclusiveMinimum() != null) prop.put("exclusiveMinimum", fc.exclusiveMinimum().doubleValue());
            if (fc.exclusiveMaximum() != null) prop.put("exclusiveMaximum", fc.exclusiveMaximum().doubleValue());
            if (fc.multipleOf() != null) prop.put("multipleOf", fc.multipleOf().doubleValue());
            if (fc.format() != null) prop.put("format", fc.format());
            if (fc.enumValues() != null && !fc.enumValues().isEmpty()) {
                ArrayNode enumNode = prop.putArray("enum");
                fc.enumValues().forEach(enumNode::add);
            }
            if (fc.required()) required.add(fieldName);
        }

        return schema;
    }

    private static String mapType(String pagesType) {
        return switch (pagesType.toLowerCase()) {
            case "number", "date" -> "number";
            case "boolean" -> "boolean";
            case "array" -> "array";
            case "object" -> "object";
            default -> "string";
        };
    }
}
```

- [ ] **Step 4: Run JsonSchemaBuilder tests**

```bash
/opt/homebrew/bin/mvn -pl backend/data test -Dtest=JsonSchemaBuilderTest -q
```

Expected: all 4 tests pass.

- [ ] **Step 5: Update SchemaViolationException with violations list**

Read `SchemaViolationException.java`. Modify to add violations field:

```java
public class SchemaViolationException extends MutationException {
    private final List<FieldViolation> violations;

    public SchemaViolationException(String message, List<FieldViolation> violations) {
        super("SCHEMA_VIOLATION", message);
        this.violations = List.copyOf(violations);
    }

    public SchemaViolationException(String message) {
        this(message, List.of());
    }

    public List<FieldViolation> violations() {
        return violations;
    }
}
```

- [ ] **Step 6: Write failing tests for CollectionSchema constraint validation**

Add to `CollectionSchemaTest.java`:

```java
@Test
void validateToResultRejectsMinLengthViolation() {
    var constraints = Map.of("name", new CollectionSchema.FieldConstraints(
        "string", false, 5, null, null, null, null, null, null, null, null, null));
    CollectionSchema schema = new CollectionSchema("tickets",
        Map.of("name", "STRING"), Map.of(), Map.of(), constraints);

    ObjectNode doc = new ObjectMapper().createObjectNode();
    doc.put("name", "ab");
    SchemaValidationResult result = schema.validateToResult(doc);

    assertThat(result.valid()).isFalse();
    assertThat(result.violations()).hasSize(1);
    assertThat(result.violations().get(0).field()).isEqualTo("name");
    assertThat(result.violations().get(0).constraint()).isEqualTo("minLength");
}

@Test
void validateToResultRejectsRequiredMissing() {
    var constraints = Map.of("name", new CollectionSchema.FieldConstraints(
        "string", true, null, null, null, null, null, null, null, null, null, null));
    CollectionSchema schema = new CollectionSchema("tickets",
        Map.of("name", "STRING"), Map.of(), Map.of(), constraints);

    ObjectNode doc = new ObjectMapper().createObjectNode();
    SchemaValidationResult result = schema.validateToResult(doc);

    assertThat(result.valid()).isFalse();
    assertThat(result.violations().get(0).constraint()).isEqualTo("required");
}

@Test
void validateToResultAcceptsValidDocument() {
    var constraints = Map.of("name", new CollectionSchema.FieldConstraints(
        "string", true, 2, 50, null, null, null, null, null, null, null, null));
    CollectionSchema schema = new CollectionSchema("tickets",
        Map.of("name", "STRING"), Map.of(), Map.of(), constraints);

    ObjectNode doc = new ObjectMapper().createObjectNode();
    doc.put("name", "Valid Name");
    SchemaValidationResult result = schema.validateToResult(doc);

    assertThat(result.valid()).isTrue();
    assertThat(result.violations()).isEmpty();
}

@Test
void validateThrowsWithViolations() {
    var constraints = Map.of("name", new CollectionSchema.FieldConstraints(
        "string", true, 5, null, null, null, null, null, null, null, null, null));
    CollectionSchema schema = new CollectionSchema("tickets",
        Map.of("name", "STRING"), Map.of(), Map.of(), constraints);

    ObjectNode doc = new ObjectMapper().createObjectNode();
    doc.put("name", "ab");

    assertThatThrownBy(() -> schema.validate(doc))
        .isInstanceOf(SchemaViolationException.class)
        .satisfies(e -> {
            SchemaViolationException sve = (SchemaViolationException) e;
            assertThat(sve.violations()).isNotEmpty();
            assertThat(sve.violations().get(0).field()).isEqualTo("name");
        });
}
```

- [ ] **Step 7: Run tests to verify they fail**

```bash
/opt/homebrew/bin/mvn -pl backend/data test -Dtest=CollectionSchemaTest -q
```

Expected: compilation error — `validateToResult` does not exist.

- [ ] **Step 8: Implement validateToResult() and update validate()**

Read `CollectionSchema.java` to see the current `validate()` method. Refactor:

1. Extract existing type-checking logic into a private `validateTypes(JsonNode, List<FieldViolation>)` method that collects violations into the list instead of throwing
2. Add `validateToResult(JsonNode)` that calls `validateTypes` + networknt library
3. Update `validate(JsonNode)` to delegate to `validateToResult` and throw on failure

The `toFieldViolation` adapter extracts field name from the library's `ValidationMessage.getPath()` (strip `$.` prefix), constraint from `getType()`, and message from `getMessage()`.

- [ ] **Step 9: Run all CollectionSchema tests**

```bash
/opt/homebrew/bin/mvn -pl backend/data test -Dtest=CollectionSchemaTest -q
```

Expected: all tests pass (existing + new).

- [ ] **Step 10: Run full backend test suite**

```bash
/opt/homebrew/bin/mvn -pl backend/data test -q
```

Expected: all tests pass.

- [ ] **Step 11: Commit**

```bash
git add backend/data/
git commit -m "feat(#610): JsonSchemaBuilder + CollectionSchema.validateToResult() with networknt Refs #610"
```

---

## Batch 2: Structured Error Response (#611)

### Task 3: MutationExceptionMapper 422 + remove local catch blocks

**Files:**
- Modify: `backend/data/src/main/java/io/casehub/pages/data/MutationExceptionMapper.java` (add SCHEMA_VIOLATION → 422)
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/PageResource.java` (remove local SchemaViolationException catch blocks)
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppCatalogueResource.java` (remove local catch blocks)
- Test: `backend/data-json/src/test/java/io/casehub/pages/data/json/SchemaValidationTest.java` (update expected status codes)

**Interfaces:**
- Consumes: `SchemaViolationException.violations()` from Task 2
- Produces: HTTP 422 response with `{"error": "VALIDATION_FAILED", "violations": [...]}` body

- [ ] **Step 1: Write failing integration test for 422 response**

Read `SchemaValidationTest.java` to understand the existing test structure. Add a test that expects 422 with structured body when constraint violations occur. This requires a collection with `fieldConstraints` — wire one up through `AppRegistry`.

Read `AppRegistry.java:311` to understand how `CollectionSchema` is built from parsed collection definitions. The constraint extraction needs to happen here — add constraint keywords from the collection's field definitions into `FieldConstraints`.

- [ ] **Step 2: Run test to verify it fails**

```bash
/opt/homebrew/bin/mvn -pl backend/data-json test -Dtest=SchemaValidationTest -q
```

Expected: test fails — currently returns 400/500 instead of 422.

- [ ] **Step 3: Update MutationExceptionMapper**

Read `MutationExceptionMapper.java`. Add `SCHEMA_VIOLATION` case to the switch:

```java
case "SCHEMA_VIOLATION" -> {
    if (exception instanceof SchemaViolationException sve && !sve.violations().isEmpty()) {
        var body = new java.util.LinkedHashMap<String, Object>();
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
    yield Response.status(400).entity(Map.of("error", exception.getMessage())).build();
}
```

- [ ] **Step 4: Remove local SchemaViolationException catch blocks**

In `PageResource.java`, remove the `catch (SchemaViolationException e)` blocks (currently at lines ~134 and ~184). The mapper now handles it globally.

In `AppCatalogueResource.java`, remove the same catch blocks.

- [ ] **Step 5: Update AppRegistry to extract constraints**

Read `AppRegistry.java:311` where `CollectionSchema` is constructed. Update to extract constraint keywords from the collection's field definitions into `FieldConstraints`:

```java
Map<String, CollectionSchema.FieldConstraints> fieldConstraints = new HashMap<>();
for (var field : col.fields()) {
    var fc = new CollectionSchema.FieldConstraints(
        fieldTypes.get(field.id()),
        field.required(),
        field.minLength(),
        field.maxLength(),
        field.pattern(),
        field.minimum(),
        field.maximum(),
        field.exclusiveMinimum(),
        field.exclusiveMaximum(),
        field.multipleOf(),
        field.enumValues(),
        field.format()
    );
    fieldConstraints.put(field.id(), fc);
}
return new CollectionSchema(col.name(), fieldTypes, defaults, refs, fieldConstraints);
```

Note: the exact field accessor names depend on the collection definition class — read it first. If constraint fields don't exist on the definition class yet, add them.

- [ ] **Step 6: Run integration tests**

```bash
/opt/homebrew/bin/mvn -pl backend/data-json test -q
```

Expected: all tests pass, including the new 422 test.

- [ ] **Step 7: Commit**

```bash
git add backend/data/src/main/java/io/casehub/pages/data/MutationExceptionMapper.java backend/data-json/
git commit -m "feat(#611): MutationExceptionMapper returns 422 with structured violations Refs #611"
```

### Task 4: Client-side FieldViolation type + SaveResult extension + REST adapter 422 parsing

**Files:**
- Modify: `packages/pages-runtime/src/save-adapter.ts` (add FieldViolation interface, extend SaveResult)
- Modify: `packages/pages-runtime/src/adapters/rest-adapter.ts` (parse 422 response body)
- Create: `packages/pages-runtime/src/adapters/rest-adapter.test.ts` (or add to existing test file)

**Interfaces:**
- Consumes: HTTP 422 response from server (Task 3)
- Produces: `FieldViolation` interface — used by Task 5
- Produces: `SaveResult.fieldErrors?: readonly FieldViolation[]` — used by Task 5

- [ ] **Step 1: Write failing test for REST adapter 422 parsing**

Find the existing test file for rest-adapter (if any). Create or extend:

```typescript
import { describe, it, expect, vi } from "vitest";

describe("rest-adapter 422 handling", () => {
  it("parses VALIDATION_FAILED response into fieldErrors", async () => {
    const mockResponse = {
      ok: false,
      status: 422,
      statusText: "Unprocessable Entity",
      json: () => Promise.resolve({
        error: "VALIDATION_FAILED",
        violations: [
          { field: "name", constraint: "minLength", params: { min: 5 }, message: "must be at least 5 characters" },
          { field: "name", constraint: "pattern", params: { pattern: "^[A-Z]" }, message: "must match pattern" },
        ],
      }),
    };
    // Mock fetch to return mockResponse, call the adapter's save method
    // Assert result.success === false
    // Assert result.fieldErrors has 2 entries
    // Assert result.fieldErrors[0].field === "name"
    // Assert result.fieldErrors[0].constraint === "minLength"
  });

  it("falls back to statusText for non-422 errors", async () => {
    const mockResponse = {
      ok: false,
      status: 500,
      statusText: "Internal Server Error",
      json: () => Promise.reject("should not be called"),
    };
    // Assert result.error === "HTTP 500: Internal Server Error"
    // Assert result.fieldErrors is undefined
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/runtime test -- --run rest-adapter
```

Expected: FAIL — FieldViolation type doesn't exist, adapter doesn't parse 422.

- [ ] **Step 3: Add FieldViolation interface and extend SaveResult**

In `packages/pages-runtime/src/save-adapter.ts`, add:

```typescript
export interface FieldViolation {
  readonly field: string;
  readonly constraint: string;
  readonly params: Readonly<Record<string, unknown>>;
  readonly message: string;
}
```

Extend `SaveResult`:

```typescript
export interface SaveResult {
  readonly success: boolean;
  readonly error?: string;
  readonly updatedRecord?: Readonly<Record<string, unknown>>;
  readonly fieldErrors?: readonly FieldViolation[];
}
```

- [ ] **Step 4: Update REST adapter to parse 422**

In `packages/pages-runtime/src/adapters/rest-adapter.ts`, update the error handling in all three methods (save, create, delete). Before the generic `return { success: false, error: ... }`, add:

```typescript
if (response.status === 422) {
  try {
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
  } catch {
    // JSON parse failed — fall through to generic error
  }
}
```

- [ ] **Step 5: Run tests**

```bash
yarn workspace @casehubio/runtime test -- --run rest-adapter
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add packages/pages-runtime/src/save-adapter.ts packages/pages-runtime/src/adapters/rest-adapter.ts packages/pages-runtime/src/adapters/rest-adapter.test.ts
git commit -m "feat(#611): FieldViolation type + REST adapter parses 422 structured errors Refs #611"
```

### Task 5: Error routing in runtime + setServerErrors on forms

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` (add routeFieldErrors, extend SaveErrorDetail, wire into save handler)
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` (add setServerErrors method)
- Modify: `packages/pages-viz/src/form-inputs/PagesFormInput.ts` (add serverViolations property)
- Modify: `packages/pages-viz/src/pages-property-palette.ts` (extend setFieldErrors for source tracking)

**Interfaces:**
- Consumes: `SaveResult.fieldErrors` from Task 4
- Consumes: `FieldViolation` from Task 4
- Produces: `routeFieldErrors(pagePath, errors, registry)` — routes errors to form components
- Produces: `PagesSchemaForm.setServerErrors(Map<string, readonly FieldViolation[]>)` — displays server errors
- Produces: `PagesFormInput.serverViolations` property — structured violations on native inputs

- [ ] **Step 1: Write failing test for routeFieldErrors**

Create or extend a test file for the save pipeline routing:

```typescript
describe("routeFieldErrors", () => {
  it("routes violations to PagesSchemaForm via setServerErrors", () => {
    // Mock registry with a PagesSchemaForm entry
    // Call routeFieldErrors with violations
    // Assert setServerErrors was called with grouped violations
  });

  it("routes violations to PagesFormInput via errorMessage", () => {
    // Mock registry with PagesFormInput entries
    // Call routeFieldErrors with violations
    // Assert errorMessage set on matching input
    // Assert serverViolations set with full FieldViolation[]
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/runtime test -- --run site
```

- [ ] **Step 3: Implement routeFieldErrors in site.ts**

Add the function and wire it into the save handler. After `showErrorBanner()` at the existing `pages-save-error` dispatch points, add:

```typescript
if (!result.success && result.fieldErrors?.length) {
  routeFieldErrors(pagePath, result.fieldErrors, registry);
}
```

Implement `routeFieldErrors` as specified in the design spec §2.4 — group by field, iterate registry, dispatch to form type.

- [ ] **Step 4: Extend SaveErrorDetail**

Update the `pages-save-error` event detail type to include `fieldErrors`:

```typescript
interface SaveErrorDetail {
  readonly pagePath: string;
  readonly error: string;
  readonly fieldErrors?: readonly FieldViolation[];
}
```

Update all `pages-save-error` dispatch sites to include `fieldErrors` when available.

- [ ] **Step 5: Add setServerErrors to PagesSchemaForm**

Read `PagesSchemaForm.ts`. Add:

```typescript
setServerErrors(errors: Map<string, readonly FieldViolation[]>): void {
  const fieldErrors = new Map<string, string | undefined>();
  for (const [field, violations] of errors) {
    fieldErrors.set(field, violations[0]?.message);
  }
  this._palette.setFieldErrors(fieldErrors);
}
```

- [ ] **Step 6: Add serverViolations to PagesFormInput**

Read `PagesFormInput.ts`. Add property:

```typescript
@property({ attribute: false }) serverViolations?: readonly FieldViolation[];
```

- [ ] **Step 7: Run tests**

```bash
yarn workspace @casehubio/runtime test -- --run
yarn workspace @casehubio/viz test -- --run
```

Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
git add packages/pages-runtime/src/site.ts packages/pages-viz/src/form-inputs/PagesSchemaForm.ts packages/pages-viz/src/form-inputs/PagesFormInput.ts packages/pages-viz/src/pages-property-palette.ts
git commit -m "feat(#611): routeFieldErrors + setServerErrors for structured error display Refs #611"
```

---

## Batch 3: Validation Consolidation and Blur (#612)

### Task 6: Consolidate validateField to canonical implementation + add validateFieldAll

**Files:**
- Modify: `packages/pages-ui-components/src/validation/validate-field.ts` (add validateFieldAll, export FieldViolation)
- Modify: `packages/pages-ui-components/src/validation/validate-field.test.ts` (add validateFieldAll tests)
- Modify: `packages/pages-component/src/model/field-validation.ts` (re-export from pages-ui-components, deprecate)
- Modify: `packages/pages-runtime/src/activation.ts` (replace inline validateFormField with import)
- Modify: `packages/pages-runtime/src/form-scope.ts` (update import source)

**Interfaces:**
- Produces: `validateFieldAll(schema, value, required)` → `readonly FieldViolation[]`
- Produces: canonical `validateField` import path at `@casehubio/ui-components/validation`

- [ ] **Step 1: Write failing test for validateFieldAll**

In `packages/pages-ui-components/src/validation/validate-field.test.ts`, add:

```typescript
describe("validateFieldAll", () => {
  it("returns all violations for a field", () => {
    const schema = { minLength: 5, pattern: "^[A-Z]" };
    const result = validateFieldAll(schema, "ab", false);
    expect(result).toHaveLength(2);
    expect(result[0].constraint).toBe("minLength");
    expect(result[1].constraint).toBe("pattern");
  });

  it("returns empty array for valid value", () => {
    const schema = { minLength: 2 };
    const result = validateFieldAll(schema, "Hello", false);
    expect(result).toHaveLength(0);
  });

  it("includes params in violations", () => {
    const schema = { minLength: 5 };
    const result = validateFieldAll(schema, "ab", false);
    expect(result[0].params).toEqual({ min: 5 });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/ui-components test -- --run validate-field
```

Expected: FAIL — `validateFieldAll` does not exist.

- [ ] **Step 3: Implement validateFieldAll**

Read `packages/pages-ui-components/src/validation/validate-field.ts`. Add `validateFieldAll` that checks ALL constraints and returns `FieldViolation[]` instead of returning on the first error:

```typescript
export function validateFieldAll(
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): readonly FieldViolation[] {
  const violations: FieldViolation[] = [];

  if (required && (value === undefined || value === null || value === "")) {
    violations.push({ field: "", constraint: "required", params: {}, message: "is required" });
  }
  if (schema.minLength != null && typeof value === "string" && value.length < schema.minLength) {
    violations.push({ field: "", constraint: "minLength", params: { min: schema.minLength }, message: `must be at least ${schema.minLength} characters` });
  }
  if (schema.maxLength != null && typeof value === "string" && value.length > schema.maxLength) {
    violations.push({ field: "", constraint: "maxLength", params: { max: schema.maxLength }, message: `must be at most ${schema.maxLength} characters` });
  }
  if (schema.pattern != null && typeof value === "string" && !new RegExp(schema.pattern).test(value)) {
    violations.push({ field: "", constraint: "pattern", params: { pattern: schema.pattern }, message: `must match pattern ${schema.pattern}` });
  }
  if (schema.minimum != null && typeof value === "number" && value < schema.minimum) {
    violations.push({ field: "", constraint: "minimum", params: { min: schema.minimum }, message: `must be >= ${schema.minimum}` });
  }
  if (schema.maximum != null && typeof value === "number" && value > schema.maximum) {
    violations.push({ field: "", constraint: "maximum", params: { max: schema.maximum }, message: `must be <= ${schema.maximum}` });
  }
  if (schema.exclusiveMinimum != null && typeof value === "number" && value <= schema.exclusiveMinimum) {
    violations.push({ field: "", constraint: "exclusiveMinimum", params: { min: schema.exclusiveMinimum }, message: `must be > ${schema.exclusiveMinimum}` });
  }
  if (schema.exclusiveMaximum != null && typeof value === "number" && value >= schema.exclusiveMaximum) {
    violations.push({ field: "", constraint: "exclusiveMaximum", params: { max: schema.exclusiveMaximum }, message: `must be < ${schema.exclusiveMaximum}` });
  }
  if (schema.multipleOf != null && typeof value === "number" && value % schema.multipleOf !== 0) {
    violations.push({ field: "", constraint: "multipleOf", params: { multipleOf: schema.multipleOf }, message: `must be a multiple of ${schema.multipleOf}` });
  }
  if (schema.enum != null && !schema.enum.includes(value as string)) {
    violations.push({ field: "", constraint: "enum", params: { values: schema.enum }, message: `must be one of: ${schema.enum.join(", ")}` });
  }
  // minItems, maxItems — follow same pattern for array values

  return violations;
}
```

The `field` is empty string here because the caller sets it (the form knows the field name, the function doesn't).

- [ ] **Step 4: Run tests**

```bash
yarn workspace @casehubio/ui-components test -- --run validate-field
```

Expected: all tests pass.

- [ ] **Step 5: Update pages-component/field-validation.ts to re-export**

Read `packages/pages-component/src/model/field-validation.ts`. Replace the implementation body with a re-export:

```typescript
/** @deprecated Import from @casehubio/ui-components/validation instead */
export { validateField, validateFieldAll } from "@casehubio/ui-components/validation";
export type { FieldViolation } from "@casehubio/ui-components/validation";
```

If there are types defined here that `pages-ui-components` imports FROM this file (circular), keep those type definitions and only redirect the function.

- [ ] **Step 6: Replace inline validateFormField in activation.ts**

Read `packages/pages-runtime/src/activation.ts:35-51`. Remove the inline `validateFormField` function. Replace with an import:

```typescript
import { validateField } from "@casehubio/ui-components/validation";
```

Update all call sites that used `validateFormField` to use `validateField`.

- [ ] **Step 7: Update FormScopeState import**

Read `packages/pages-runtime/src/form-scope.ts`. Change the `validateField` import from `@casehubio/component` to `@casehubio/ui-components/validation`.

- [ ] **Step 8: Run all affected package tests**

```bash
yarn workspace @casehubio/ui-components test -- --run
yarn workspace @casehubio/component test -- --run
yarn workspace @casehubio/runtime test -- --run
```

Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git add packages/pages-ui-components/ packages/pages-component/ packages/pages-runtime/
git commit -m "feat(#612): consolidate validateField to canonical impl + add validateFieldAll Refs #612"
```

### Task 7: Gate palette validation behind validateOnBlur + PagesFormInput blur handler

**Files:**
- Modify: `packages/pages-viz/src/pages-property-palette.ts` (gate _handleChange validation behind validateOnBlur)
- Modify: `packages/pages-viz/src/form-inputs/PagesFormInput.ts` (add _handleBlur, fieldSchema, validateOnBlur)
- Modify: `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` (ensure validateOnBlur gates palette)
- Test: `packages/pages-viz/src/form-inputs/schema-form.test.ts` (update existing blur tests, add gating tests)

**Interfaces:**
- Consumes: `validateField` from canonical `@casehubio/ui-components/validation` (Task 6)
- Consumes: `FieldSchema` type

- [ ] **Step 1: Write failing test — validateOnBlur=false should suppress palette validation**

In `packages/pages-viz/src/form-inputs/schema-form.test.ts`, add:

```typescript
it("validateOnBlur=false suppresses palette validation on change", async () => {
  // Create PagesSchemaForm with validateOnBlur=false
  // Trigger a field change (blur)
  // Assert no error is set on the field
});

it("validateOnBlur=true (default) validates on change", async () => {
  // Create PagesSchemaForm with validateOnBlur=true (default)
  // Trigger a field change with invalid value
  // Assert error IS set on the field
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/viz test -- --run schema-form
```

Expected: first test fails (palette validates unconditionally).

- [ ] **Step 3: Gate palette validation behind validateOnBlur**

Read `pages-property-palette.ts:368-384` (the `_handleChange` method). Add a `validateOnBlur` property or accept it as a parameter. Only call `validateField` when the parent form's `validateOnBlur` is true.

The palette needs to know the parent form's `validateOnBlur` setting. Options:
- Pass it as a constructor/property on the palette
- Check a property on the parent element

Read how the palette is created in `PagesSchemaForm` to determine the cleanest approach.

- [ ] **Step 4: Write failing test — PagesFormInput blur validation**

Create or extend test file for PagesFormInput:

```typescript
describe("PagesFormInput blur validation", () => {
  it("validates on blur when fieldSchema is set", () => {
    // Create a concrete PagesFormInput subclass
    // Set fieldSchema with minLength=5
    // Set currentValue to "ab"
    // Call _handleBlur()
    // Assert errorMessage is set
  });

  it("clears error on blur when value is valid", () => {
    // Set errorMessage to a previous error
    // Set currentValue to "Valid"
    // Call _handleBlur()
    // Assert errorMessage is undefined
  });

  it("skips validation when validateOnBlur is false", () => {
    // Set validateOnBlur=false
    // Call _handleBlur()
    // Assert errorMessage is unchanged
  });
});
```

- [ ] **Step 5: Implement PagesFormInput blur handler**

In `PagesFormInput.ts`, add:

```typescript
@property({ type: Boolean }) validateOnBlur = true;
@property({ attribute: false }) fieldSchema?: FieldSchema;

protected _handleBlur(): void {
  if (!this.validateOnBlur || !this.fieldSchema) return;
  const error = validateField(this.fieldSchema, this.currentValue, this.required);
  this.errorMessage = error ?? undefined;
}
```

Wire `_handleBlur` to the blur event in the render method or connectedCallback.

Add focus handler to clear server errors:

```typescript
protected _handleFocus(): void {
  if (this.serverViolations?.length) {
    this.serverViolations = undefined;
    this.errorMessage = undefined;
  }
}
```

- [ ] **Step 6: Run all viz tests**

```bash
yarn workspace @casehubio/viz test -- --run
```

Expected: all tests pass (existing blur tests + new gating tests).

- [ ] **Step 7: Run full test suite**

```bash
yarn workspaces foreach -Apt run test -- --run
```

Expected: all packages pass.

- [ ] **Step 8: Commit**

```bash
git add packages/pages-viz/
git commit -m "feat(#612): gate palette validation behind validateOnBlur + PagesFormInput blur handler Refs #612"
```

---

## References

- `specs/issue-609-forms-validation/2026-11-04-forms-validation-design.md` — design spec this plan implements
- `specs/issue-609-forms-validation/decisions.md` — D1–D10 design decisions
- `backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java` — server validation target
- `backend/data/src/main/java/io/casehub/pages/data/MutationExceptionMapper.java` — exception→response mapper
- `packages/pages-ui-components/src/validation/validate-field.ts` — canonical validateField
- `packages/pages-runtime/src/site.ts` — save handler, error routing
- `packages/pages-runtime/src/adapters/rest-adapter.ts` — 422 parsing
- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — schema form
- `packages/pages-viz/src/form-inputs/PagesFormInput.ts` — native form input base
- Protocol PP-20260705-bac842 (`pages-event-contract`) — reserved events
- GitHub #609, #610, #611, #612
