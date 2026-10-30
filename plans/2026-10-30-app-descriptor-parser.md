# App Descriptor Parser Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #596 — App descriptor parser — app.yaml schema, collection registration, seed data
**Issue group:** #596 (depends on #595 `data-json`, already merged)

**Goal:** Parse `app.yaml` descriptors from the classpath at startup, register JSON collections with `JsonDataProvider`, validate writes against declared schemas, load seed data, and enforce referential integrity across collections.

**Architecture:** `AppDescriptor` is a pure data record parsed from YAML. `CollectionSchema` lives at the SPI level (`backend/data`) so any provider can validate. `AppRegistry` is a CDI bean in `data-json` that scans the classpath for `org/apps/*/app.yaml`, parses them into `AppDescriptor` records, registers collections, and wires schema validation into the write path. Seed data is loaded lazily on first access, serialized by `BEGIN IMMEDIATE`. Reference fields declare cross-collection relationships with restrict-on-delete semantics, resolved at the pipeline layer via `hasReferences`.

**Tech Stack:** Java 21, Quarkus/CDI, Jackson + jackson-dataformat-yaml, SQLite JSONB, AssertJ

## Global Constraints

- Pre-release stage — no backward compatibility concerns
- All tests use `@QuarkusTest` with `@Inject` (following existing `data-json` test patterns)
- YAML parsing via `jackson-dataformat-yaml` (already used by `playbook` module)
- `CollectionSchema` goes in `backend/data` (SPI level), not `data-json`
- Classpath scan path: `org/apps/{appId}/app.yaml`
- Seed data path: relative to `app.yaml` location (e.g. `org/apps/{appId}/data/{file}`)
- IntelliJ MCP required for all code navigation and structural editing

---

## Batch 1: Core types — AppDescriptor and CollectionSchema

### Task 1: AppDescriptor record and YAML parsing

**Files:**
- Create: `data-json/src/main/java/io/casehub/pages/data/json/FieldType.java`
- Create: `data-json/src/main/java/io/casehub/pages/data/json/FieldDef.java`
- Create: `data-json/src/main/java/io/casehub/pages/data/json/CollectionDef.java`
- Create: `data-json/src/main/java/io/casehub/pages/data/json/AppDescriptor.java`
- Create: `data-json/src/main/java/io/casehub/pages/data/json/AppDescriptorParser.java`
- Modify: `data-json/pom.xml` (add jackson-dataformat-yaml dependency)
- Test: `data-json/src/test/java/io/casehub/pages/data/json/AppDescriptorParserTest.java`
- Create: `data-json/src/test/resources/org/apps/helpdesk/app.yaml`

**Interfaces:**
- Produces:
  - `FieldType` — enum: `STRING`, `NUMBER`, `BOOLEAN`, `DATE`, `ENUM`, `ARRAY`, `OBJECT`, `REF`
  - `FieldDef` — record: `String name`, `FieldType type`, `Object defaultValue`, `List<String> enumValues`, `String ref`
  - `CollectionDef` — record: `String name`, `List<FieldDef> fields`, `List<String> dataFiles`
  - `AppDescriptor` — record: `String appId`, `List<CollectionDef> collections`
  - `AppDescriptorParser.parse(String appId, InputStream yaml)` → `AppDescriptor`

- [ ] **Step 1: Add jackson-dataformat-yaml to data-json/pom.xml**

Add dependency in the `<dependencies>` section:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```

(Version managed by casehub-parent BOM via quarkus-bom)

- [ ] **Step 2: Create test resource app.yaml**

Create `data-json/src/test/resources/org/apps/helpdesk/app.yaml`:

```yaml
collections:
  tickets:
    fields:
      - name: title
        type: string
      - name: priority
        type: enum
        values: [low, medium, high, critical]
      - name: assignee
        type: ref
        ref: agents
      - name: tags
        type: array
      - name: created
        type: date
    data:
      - data/tickets.json
  agents:
    fields:
      - name: name
        type: string
      - name: email
        type: string
      - name: active
        type: boolean
        default: true
```

- [ ] **Step 3: Write the failing test**

Create `AppDescriptorParserTest.java`:

```java
package io.casehub.pages.data.json;

import org.junit.jupiter.api.Test;

import java.io.InputStream;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AppDescriptorParserTest {

    @Test
    void parses_collections_with_fields() {
        InputStream yaml = getClass().getClassLoader()
            .getResourceAsStream("org/apps/helpdesk/app.yaml");

        AppDescriptor descriptor = AppDescriptorParser.parse("helpdesk", yaml);

        assertThat(descriptor.appId()).isEqualTo("helpdesk");
        assertThat(descriptor.collections()).hasSize(2);

        CollectionDef tickets = descriptor.collections().stream()
            .filter(c -> c.name().equals("tickets"))
            .findFirst().orElseThrow();

        assertThat(tickets.fields()).hasSize(5);

        FieldDef title = tickets.fields().stream()
            .filter(f -> f.name().equals("title")).findFirst().orElseThrow();
        assertThat(title.type()).isEqualTo(FieldType.STRING);

        FieldDef priority = tickets.fields().stream()
            .filter(f -> f.name().equals("priority")).findFirst().orElseThrow();
        assertThat(priority.type()).isEqualTo(FieldType.ENUM);
        assertThat(priority.enumValues()).containsExactly("low", "medium", "high", "critical");

        FieldDef assignee = tickets.fields().stream()
            .filter(f -> f.name().equals("assignee")).findFirst().orElseThrow();
        assertThat(assignee.type()).isEqualTo(FieldType.REF);
        assertThat(assignee.ref()).isEqualTo("agents");

        assertThat(tickets.dataFiles()).containsExactly("data/tickets.json");
    }

    @Test
    void parses_field_defaults() {
        InputStream yaml = getClass().getClassLoader()
            .getResourceAsStream("org/apps/helpdesk/app.yaml");

        AppDescriptor descriptor = AppDescriptorParser.parse("helpdesk", yaml);
        CollectionDef agents = descriptor.collections().stream()
            .filter(c -> c.name().equals("agents"))
            .findFirst().orElseThrow();

        FieldDef active = agents.fields().stream()
            .filter(f -> f.name().equals("active")).findFirst().orElseThrow();
        assertThat(active.type()).isEqualTo(FieldType.BOOLEAN);
        assertThat(active.defaultValue()).isEqualTo(true);
    }

    @Test
    void rejects_null_input() {
        assertThatThrownBy(() -> AppDescriptorParser.parse("test", null))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn test -pl data-json -Dtest=AppDescriptorParserTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: FAIL — classes do not exist yet

- [ ] **Step 5: Create FieldType enum**

Create `FieldType.java`:

```java
package io.casehub.pages.data.json;

import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonValue;

public enum FieldType {
    STRING, NUMBER, BOOLEAN, DATE, ENUM, ARRAY, OBJECT, REF;

    @JsonValue
    public String toJson() {
        return name().toLowerCase();
    }

    @JsonCreator
    public static FieldType fromJson(String value) {
        return valueOf(value.toUpperCase());
    }
}
```

- [ ] **Step 6: Create FieldDef record**

Create `FieldDef.java`:

```java
package io.casehub.pages.data.json;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

public record FieldDef(
    String name,
    FieldType type,
    @JsonProperty("default") Object defaultValue,
    @JsonProperty("values") List<String> enumValues,
    String ref
) {
    public FieldDef {
        if (enumValues == null) enumValues = List.of();
    }
}
```

- [ ] **Step 7: Create CollectionDef record**

Create `CollectionDef.java`:

```java
package io.casehub.pages.data.json;

import java.util.List;

public record CollectionDef(
    String name,
    List<FieldDef> fields,
    List<String> dataFiles
) {
    public CollectionDef {
        if (fields == null) fields = List.of();
        if (dataFiles == null) dataFiles = List.of();
    }
}
```

- [ ] **Step 8: Create AppDescriptor record**

Create `AppDescriptor.java`:

```java
package io.casehub.pages.data.json;

import java.util.List;

public record AppDescriptor(
    String appId,
    List<CollectionDef> collections
) {
    public AppDescriptor {
        if (collections == null) collections = List.of();
    }
}
```

- [ ] **Step 9: Create AppDescriptorParser**

Create `AppDescriptorParser.java`:

```java
package io.casehub.pages.data.json;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

public final class AppDescriptorParser {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());

    private AppDescriptorParser() {}

    public static AppDescriptor parse(String appId, InputStream yaml) {
        if (yaml == null) {
            throw new IllegalArgumentException("YAML input must not be null for app: " + appId);
        }
        try {
            JsonNode root = YAML.readTree(yaml);
            List<CollectionDef> collections = parseCollections(root.get("collections"));
            return new AppDescriptor(appId, collections);
        } catch (IOException e) {
            throw new IllegalArgumentException("Failed to parse app.yaml for app: " + appId, e);
        }
    }

    private static List<CollectionDef> parseCollections(JsonNode collectionsNode) {
        if (collectionsNode == null || !collectionsNode.isObject()) return List.of();

        List<CollectionDef> result = new ArrayList<>();
        Iterator<Map.Entry<String, JsonNode>> it = collectionsNode.fields();
        while (it.hasNext()) {
            Map.Entry<String, JsonNode> entry = it.next();
            String name = entry.getKey();
            JsonNode colNode = entry.getValue();
            List<FieldDef> fields = parseFields(colNode.get("fields"));
            List<String> dataFiles = parseDataFiles(colNode.get("data"));
            result.add(new CollectionDef(name, fields, dataFiles));
        }
        return List.copyOf(result);
    }

    private static List<FieldDef> parseFields(JsonNode fieldsNode) {
        if (fieldsNode == null || !fieldsNode.isArray()) return List.of();

        List<FieldDef> result = new ArrayList<>();
        for (JsonNode fieldNode : fieldsNode) {
            String name = fieldNode.get("name").asText();
            FieldType type = FieldType.fromJson(fieldNode.get("type").asText());
            Object defaultValue = parseDefault(fieldNode.get("default"), type);
            List<String> enumValues = parseStringArray(fieldNode.get("values"));
            String ref = fieldNode.has("ref") ? fieldNode.get("ref").asText() : null;
            result.add(new FieldDef(name, type, defaultValue, enumValues, ref));
        }
        return List.copyOf(result);
    }

    private static Object parseDefault(JsonNode node, FieldType type) {
        if (node == null || node.isNull()) return null;
        return switch (type) {
            case BOOLEAN -> node.asBoolean();
            case NUMBER -> node.numberValue();
            case STRING, DATE, ENUM -> node.asText();
            default -> node.toString();
        };
    }

    private static List<String> parseStringArray(JsonNode node) {
        if (node == null || !node.isArray()) return List.of();
        List<String> result = new ArrayList<>();
        for (JsonNode item : node) {
            result.add(item.asText());
        }
        return List.copyOf(result);
    }

    private static List<String> parseDataFiles(JsonNode node) {
        if (node == null || !node.isArray()) return List.of();
        List<String> result = new ArrayList<>();
        for (JsonNode item : node) {
            result.add(item.asText());
        }
        return List.copyOf(result);
    }
}
```

- [ ] **Step 10: Run tests**

Run: `mvn test -pl data-json -Dtest=AppDescriptorParserTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add data-json/pom.xml data-json/src/
git commit -m "feat(#596): AppDescriptor record and YAML parser

Refs #596"
```

---

### Task 2: CollectionSchema in the data SPI module

**Files:**
- Create: `data/src/main/java/io/casehub/pages/data/CollectionSchema.java`
- Create: `data/src/main/java/io/casehub/pages/data/SchemaViolationException.java`
- Test: `data/src/test/java/io/casehub/pages/data/CollectionSchemaTest.java`

**Interfaces:**
- Consumes: `FieldType` from Task 1 (but `CollectionSchema` uses its own field-type representation at SPI level to avoid coupling `data` → `data-json`). Uses a `Map<String, String>` of field name → type name.
- Produces:
  - `CollectionSchema` — record: `String collection`, `Map<String, String> fieldTypes`, `Map<String, Object> defaults`, `Map<String, String> refs`
  - `CollectionSchema.validate(JsonNode document)` — throws `SchemaViolationException`
  - `SchemaViolationException` — extends `MutationException` with code `SCHEMA_VIOLATION`

- [ ] **Step 1: Write the failing test**

Create `CollectionSchemaTest.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class CollectionSchemaTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void validate_accepts_conforming_document() {
        CollectionSchema schema = new CollectionSchema("tickets",
            Map.of("title", "string", "priority", "number", "active", "boolean"),
            Map.of("active", true),
            Map.of());

        ObjectNode doc = MAPPER.createObjectNode()
            .put("title", "Test")
            .put("priority", 1)
            .put("active", true);

        schema.validate(doc);
    }

    @Test
    void validate_rejects_wrong_type() {
        CollectionSchema schema = new CollectionSchema("tickets",
            Map.of("priority", "number"),
            Map.of(),
            Map.of());

        ObjectNode doc = MAPPER.createObjectNode().put("priority", "not-a-number");

        assertThatThrownBy(() -> schema.validate(doc))
            .isInstanceOf(SchemaViolationException.class)
            .hasMessageContaining("priority");
    }

    @Test
    void validate_allows_unknown_fields() {
        CollectionSchema schema = new CollectionSchema("tickets",
            Map.of("title", "string"),
            Map.of(),
            Map.of());

        ObjectNode doc = MAPPER.createObjectNode()
            .put("title", "Test")
            .put("extra", "value");

        schema.validate(doc);
    }

    @Test
    void validate_checks_enum_values() {
        CollectionSchema schema = new CollectionSchema("tickets",
            Map.of("status", "enum:open,closed,pending"),
            Map.of(),
            Map.of());

        ObjectNode valid = MAPPER.createObjectNode().put("status", "open");
        schema.validate(valid);

        ObjectNode invalid = MAPPER.createObjectNode().put("status", "invalid");
        assertThatThrownBy(() -> schema.validate(invalid))
            .isInstanceOf(SchemaViolationException.class)
            .hasMessageContaining("status");
    }

    @Test
    void applyDefaults_fills_missing_fields() {
        CollectionSchema schema = new CollectionSchema("tickets",
            Map.of("title", "string", "active", "boolean"),
            Map.of("active", true),
            Map.of());

        ObjectNode doc = MAPPER.createObjectNode().put("title", "Test");
        schema.applyDefaults(doc);

        assertThat(doc.get("active").asBoolean()).isTrue();
    }

    @Test
    void refs_tracks_reference_fields() {
        CollectionSchema schema = new CollectionSchema("tickets",
            Map.of("assignee", "ref"),
            Map.of(),
            Map.of("assignee", "agents"));

        assertThat(schema.refs()).containsEntry("assignee", "agents");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl data -Dtest=CollectionSchemaTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: FAIL — class does not exist

- [ ] **Step 3: Create SchemaViolationException**

Create `SchemaViolationException.java`:

```java
package io.casehub.pages.data;

public class SchemaViolationException extends MutationException {

    public SchemaViolationException(String message) {
        super("SCHEMA_VIOLATION", message);
    }
}
```

- [ ] **Step 4: Create CollectionSchema record**

Create `CollectionSchema.java`:

```java
package io.casehub.pages.data;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;

import java.util.Map;

public record CollectionSchema(
    String collection,
    Map<String, String> fieldTypes,
    Map<String, Object> defaults,
    Map<String, String> refs
) {
    public CollectionSchema {
        fieldTypes = Map.copyOf(fieldTypes);
        defaults = Map.copyOf(defaults);
        refs = Map.copyOf(refs);
    }

    public void validate(JsonNode document) {
        for (var entry : fieldTypes.entrySet()) {
            String field = entry.getKey();
            String type = entry.getValue();
            JsonNode value = document.get(field);
            if (value == null || value.isNull()) continue;
            validateType(field, type, value);
        }
    }

    public void applyDefaults(ObjectNode document) {
        for (var entry : defaults.entrySet()) {
            String field = entry.getKey();
            if (!document.has(field)) {
                Object val = entry.getValue();
                if (val instanceof Boolean b) document.put(field, b);
                else if (val instanceof Number n) document.put(field, n.doubleValue());
                else if (val instanceof String s) document.put(field, s);
            }
        }
    }

    private void validateType(String field, String type, JsonNode value) {
        if (type.startsWith("enum:")) {
            String[] allowed = type.substring(5).split(",");
            String actual = value.asText();
            for (String a : allowed) {
                if (a.equals(actual)) return;
            }
            throw new SchemaViolationException(
                "Field '" + field + "' value '" + actual + "' not in allowed values: " + type.substring(5));
        }

        boolean valid = switch (type) {
            case "string", "date", "ref" -> value.isTextual();
            case "number" -> value.isNumber();
            case "boolean" -> value.isBoolean();
            case "array" -> value.isArray();
            case "object" -> value.isObject();
            default -> true;
        };
        if (!valid) {
            throw new SchemaViolationException(
                "Field '" + field + "' expected type '" + type + "' but got " + value.getNodeType());
        }
    }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl data -Dtest=CollectionSchemaTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add data/src/
git commit -m "feat(#596): CollectionSchema with type validation and defaults

SPI-level schema validation for JSON collections.

Refs #596"
```

---

## Batch 2: Registry — classpath scan and collection registration

### Task 3: AppRegistry CDI bean

**Files:**
- Create: `data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java`
- Test: `data-json/src/test/java/io/casehub/pages/data/json/AppRegistryTest.java`
- Create: `data-json/src/test/resources/org/apps/helpdesk/data/tickets.json` (test seed data for later use)

**Interfaces:**
- Consumes:
  - `AppDescriptorParser.parse(String, InputStream)` from Task 1
  - `CollectionSchema` from Task 2
  - `FieldType` from Task 1
  - `FieldDef` from Task 1
- Produces:
  - `AppRegistry` — `@ApplicationScoped` CDI bean
  - `AppRegistry.getSchema(String appId, String collection)` → `Optional<CollectionSchema>`
  - `AppRegistry.getDescriptor(String appId)` → `Optional<AppDescriptor>`
  - `AppRegistry.getRegisteredApps()` → `Set<String>`

- [ ] **Step 1: Write the failing test**

Create `AppRegistryTest.java`:

```java
package io.casehub.pages.data.json;

import io.casehub.pages.data.CollectionSchema;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class AppRegistryTest {

    @Inject
    AppRegistry registry;

    @Test
    void discovers_app_from_classpath() {
        assertThat(registry.getRegisteredApps()).contains("helpdesk");
    }

    @Test
    void returns_descriptor_for_registered_app() {
        Optional<AppDescriptor> descriptor = registry.getDescriptor("helpdesk");
        assertThat(descriptor).isPresent();
        assertThat(descriptor.get().collections()).isNotEmpty();
    }

    @Test
    void returns_schema_for_collection() {
        Optional<CollectionSchema> schema = registry.getSchema("helpdesk", "tickets");
        assertThat(schema).isPresent();
        assertThat(schema.get().fieldTypes()).containsKey("title");
        assertThat(schema.get().fieldTypes().get("title")).isEqualTo("string");
    }

    @Test
    void returns_schema_with_enum_encoding() {
        Optional<CollectionSchema> schema = registry.getSchema("helpdesk", "tickets");
        assertThat(schema).isPresent();
        assertThat(schema.get().fieldTypes().get("priority"))
            .startsWith("enum:");
    }

    @Test
    void returns_schema_with_ref_mapping() {
        Optional<CollectionSchema> schema = registry.getSchema("helpdesk", "tickets");
        assertThat(schema).isPresent();
        assertThat(schema.get().refs()).containsEntry("assignee", "agents");
    }

    @Test
    void returns_empty_for_unknown_app() {
        assertThat(registry.getDescriptor("nonexistent")).isEmpty();
    }

    @Test
    void returns_empty_for_unknown_collection() {
        assertThat(registry.getSchema("helpdesk", "nonexistent")).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl data-json -Dtest=AppRegistryTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: FAIL — class does not exist

- [ ] **Step 3: Create AppRegistry**

Create `AppRegistry.java`:

```java
package io.casehub.pages.data.json;

import io.casehub.pages.data.CollectionSchema;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Enumeration;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

@ApplicationScoped
public class AppRegistry {

    private static final Logger LOG = Logger.getLogger(AppRegistry.class);
    private static final String APPS_PREFIX = "org/apps/";

    private final Map<String, AppDescriptor> descriptors = new LinkedHashMap<>();
    private final Map<String, Map<String, CollectionSchema>> schemas = new LinkedHashMap<>();

    @PostConstruct
    void init() {
        scanClasspath();
    }

    public Set<String> getRegisteredApps() {
        return Set.copyOf(descriptors.keySet());
    }

    public Optional<AppDescriptor> getDescriptor(String appId) {
        return Optional.ofNullable(descriptors.get(appId));
    }

    public Optional<CollectionSchema> getSchema(String appId, String collection) {
        Map<String, CollectionSchema> appSchemas = schemas.get(appId);
        if (appSchemas == null) return Optional.empty();
        return Optional.ofNullable(appSchemas.get(collection));
    }

    private void scanClasspath() {
        try {
            Enumeration<URL> resources = Thread.currentThread().getContextClassLoader()
                .getResources(APPS_PREFIX);

            while (resources.hasMoreElements()) {
                URL url = resources.nextElement();
                if ("file".equals(url.getProtocol())) {
                    Path appsDir = Path.of(url.toURI());
                    if (Files.isDirectory(appsDir)) {
                        try (var dirs = Files.list(appsDir)) {
                            dirs.filter(Files::isDirectory).forEach(this::loadApp);
                        }
                    }
                }
            }
        } catch (Exception e) {
            LOG.warn("Failed to scan classpath for app descriptors", e);
        }
    }

    private void loadApp(Path appDir) {
        Path yamlFile = appDir.resolve("app.yaml");
        if (!Files.exists(yamlFile)) return;

        String appId = appDir.getFileName().toString();
        try (InputStream is = Files.newInputStream(yamlFile)) {
            AppDescriptor descriptor = AppDescriptorParser.parse(appId, is);
            descriptors.put(appId, descriptor);
            registerSchemas(descriptor);
            LOG.infof("Registered app '%s' with %d collections", appId, descriptor.collections().size());
        } catch (IOException e) {
            LOG.warnf(e, "Failed to load app descriptor for '%s'", appId);
        }
    }

    private void registerSchemas(AppDescriptor descriptor) {
        Map<String, CollectionSchema> appSchemas = new LinkedHashMap<>();
        for (CollectionDef col : descriptor.collections()) {
            CollectionSchema schema = toSchema(col);
            appSchemas.put(col.name(), schema);
        }
        schemas.put(descriptor.appId(), appSchemas);
    }

    private CollectionSchema toSchema(CollectionDef col) {
        Map<String, String> fieldTypes = new LinkedHashMap<>();
        Map<String, Object> defaults = new LinkedHashMap<>();
        Map<String, String> refs = new LinkedHashMap<>();

        for (FieldDef field : col.fields()) {
            String typeStr = switch (field.type()) {
                case ENUM -> "enum:" + String.join(",", field.enumValues());
                default -> field.type().toJson();
            };
            fieldTypes.put(field.name(), typeStr);

            if (field.defaultValue() != null) {
                defaults.put(field.name(), field.defaultValue());
            }
            if (field.ref() != null) {
                refs.put(field.name(), field.ref());
            }
        }
        return new CollectionSchema(col.name(), fieldTypes, defaults, refs);
    }
}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl data-json -Dtest=AppRegistryTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add data-json/src/
git commit -m "feat(#596): AppRegistry CDI bean — classpath scan and schema registration

Scans org/apps/*/app.yaml at startup, parses descriptors, and builds
CollectionSchema instances for each declared collection.

Refs #596"
```

---

## Batch 3: Schema validation on writes and seed data loading

### Task 4: Schema validation on writes

**Files:**
- Modify: `data-json/src/main/java/io/casehub/pages/data/json/JsonDataProvider.java` — inject `AppRegistry`, validate + apply defaults on `create` and `update`
- Test: `data-json/src/test/java/io/casehub/pages/data/json/SchemaValidationTest.java`

**Interfaces:**
- Consumes:
  - `AppRegistry.getSchema(String appId, String collection)` from Task 3
  - `CollectionSchema.validate(JsonNode)` from Task 2
  - `CollectionSchema.applyDefaults(ObjectNode)` from Task 2

- [ ] **Step 1: Write the failing test**

Create `SchemaValidationTest.java`:

```java
package io.casehub.pages.data.json;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.pages.data.DataContext;
import io.casehub.pages.data.SchemaViolationException;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.OptionalInt;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class SchemaValidationTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Inject
    JsonDataProvider provider;

    @Test
    void create_validates_against_schema() {
        DataContext ctx = new DataContext("schema-test");
        ObjectNode doc = MAPPER.createObjectNode()
            .put("title", "Valid Ticket")
            .put("priority", "high");

        String id = provider.create(ctx, "tickets", doc, "title");
        assertThat(id).isNotBlank();
    }

    @Test
    void create_rejects_invalid_enum_value() {
        DataContext ctx = new DataContext("schema-test-enum");
        ObjectNode doc = MAPPER.createObjectNode()
            .put("title", "Bad Priority")
            .put("priority", "extreme");

        assertThatThrownBy(() -> provider.create(ctx, "tickets", doc, "title"))
            .isInstanceOf(SchemaViolationException.class)
            .hasMessageContaining("priority");
    }

    @Test
    void create_applies_defaults() {
        DataContext ctx = new DataContext("schema-test-defaults");
        ObjectNode doc = MAPPER.createObjectNode()
            .put("name", "Agent Smith")
            .put("email", "smith@test.com");

        String id = provider.create(ctx, "agents", doc, "name");
        var loaded = provider.get(ctx, "agents", id).orElseThrow();
        assertThat(loaded.get("active").asBoolean()).isTrue();
    }

    @Test
    void update_validates_against_schema() {
        DataContext ctx = new DataContext("schema-test-update");
        ObjectNode doc = MAPPER.createObjectNode()
            .put("title", "Update Test")
            .put("priority", "low");
        String id = provider.create(ctx, "tickets", doc, "title");

        ObjectNode badPatch = MAPPER.createObjectNode().put("priority", "extreme");
        assertThatThrownBy(() ->
            provider.update(ctx, "tickets", id, badPatch, OptionalInt.empty()))
            .isInstanceOf(SchemaViolationException.class);
    }

    @Test
    void create_allows_unregistered_collections() {
        DataContext ctx = new DataContext("schema-test-unreg");
        ObjectNode doc = MAPPER.createObjectNode().put("data", "anything");

        String id = provider.create(ctx, "unregistered-collection", doc, "data");
        assertThat(id).isNotBlank();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl data-json -Dtest=SchemaValidationTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: FAIL — enum validation not wired in yet

- [ ] **Step 3: Modify JsonDataProvider to inject AppRegistry and validate**

Add to `JsonDataProvider`:
- `@Inject AppRegistry appRegistry;` field
- Validation + defaults logic in `create()` and `update()` methods
- Use `parseDataSetId()` to extract appId, then `appRegistry.getSchema(appId, collection)`

Modified `create` method:

```java
@Override
public String create(DataContext ctx, String collection, JsonNode document, String slugField) {
    String appId = ctx.tenantId();
    validateAndApplyDefaults(appId, collection, document);
    return store.create(ctx, ctx.tenantId(), collection, document, slugField);
}
```

Modified `update` method:

```java
@Override
public void update(DataContext ctx, String collection, String docId,
                   JsonNode changes, OptionalInt expectedVersion) {
    String appId = ctx.tenantId();
    validateSchema(appId, collection, changes);
    store.update(ctx, ctx.tenantId(), collection, docId, changes, expectedVersion, MergeStrategy.RFC_7396);
}
```

New private methods:

```java
private void validateAndApplyDefaults(String appId, String collection, JsonNode document) {
    appRegistry.getSchema(appId, collection).ifPresent(schema -> {
        if (document instanceof ObjectNode obj) {
            schema.applyDefaults(obj);
        }
        schema.validate(document);
    });
}

private void validateSchema(String appId, String collection, JsonNode document) {
    appRegistry.getSchema(appId, collection).ifPresent(schema ->
        schema.validate(document));
}
```

Note: The `appId` for schema lookup needs to come from the registered app, not just the tenant. The `parseDataSetId` approach extracts the app ID from the dataset prefix (`json:appId/collection`). For the `MutableDataProvider` interface, the collection parameter is just the collection name — the app ID context needs to come from the registry. Since the test resource registers "helpdesk" as an app, and `CollectionResource` passes `appId` from the URL path, the schema lookup should use the URL-derived appId.

This requires a design adjustment: `JsonDataProvider.create/update` receive only `collection` from the `MutableDataProvider` interface, not `appId`. The `CollectionResource` knows the `appId` from `@PathParam`. Two options:
1. Search all registered apps for the collection name
2. Extend `DataContext` to carry appId

Option 2 is cleaner. Add `appId` to `DataContext` (or use a new overload). But since `DataContext` is SPI-level, this needs care.

Simpler approach: `AppRegistry` provides a `getSchemaForCollection(String collection)` method that searches across all apps. Collections should have unique names within a tenant anyway.

```java
public Optional<CollectionSchema> getSchemaForCollection(String collection) {
    for (Map<String, CollectionSchema> appSchemas : schemas.values()) {
        CollectionSchema schema = appSchemas.get(collection);
        if (schema != null) return Optional.of(schema);
    }
    return Optional.empty();
}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl data-json -Dtest=SchemaValidationTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

Also run existing tests to verify no regression:
Run: `mvn test -pl data-json -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add data-json/src/
git commit -m "feat(#596): wire schema validation into JsonDataProvider create/update

Validates documents against CollectionSchema on writes. Applies field
defaults on create. Unregistered collections pass through without
validation.

Refs #596"
```

---

### Task 5: Seed data loading

**Files:**
- Modify: `data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java` — add seed loading
- Modify: `data-json/src/main/java/io/casehub/pages/data/json/JsonCollectionStore.java` — add `count` and `createInTransaction` methods
- Create: `data-json/src/test/resources/org/apps/helpdesk/data/tickets.json`
- Test: `data-json/src/test/java/io/casehub/pages/data/json/SeedDataTest.java`

**Interfaces:**
- Consumes:
  - `AppDescriptor.collections()` → `CollectionDef.dataFiles()`
  - `JsonCollectionStore.create(...)` from #595
- Produces:
  - `AppRegistry.ensureSeeded(DataContext ctx, String appId, String collection)` — lazy, idempotent, `BEGIN IMMEDIATE`
  - `JsonCollectionStore.countDocuments(String tenantId, String appId, String collection)` → `int`

- [ ] **Step 1: Create test seed data file**

Create `data-json/src/test/resources/org/apps/helpdesk/data/tickets.json`:

```json
[
  {"title": "Welcome Ticket", "priority": "low", "tags": []},
  {"title": "Getting Started", "priority": "medium", "tags": ["onboarding"]}
]
```

- [ ] **Step 2: Write the failing test**

Create `SeedDataTest.java`:

```java
package io.casehub.pages.data.json;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.pages.data.DataContext;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class SeedDataTest {

    @Inject
    AppRegistry registry;

    @Inject
    JsonDataProvider provider;

    @Test
    void seeds_data_on_first_access() {
        DataContext ctx = new DataContext("seed-test");
        registry.ensureSeeded(ctx, "helpdesk", "tickets");

        Optional<JsonNode> ticket = provider.get(ctx, "tickets", "welcome-ticket");
        assertThat(ticket).isPresent();
        assertThat(ticket.get().get("title").asText()).isEqualTo("Welcome Ticket");
    }

    @Test
    void seed_is_idempotent() {
        DataContext ctx = new DataContext("seed-idempotent");
        registry.ensureSeeded(ctx, "helpdesk", "tickets");
        registry.ensureSeeded(ctx, "helpdesk", "tickets");

        // Should not throw or duplicate
        Optional<JsonNode> ticket = provider.get(ctx, "tickets", "welcome-ticket");
        assertThat(ticket).isPresent();
    }

    @Test
    void seed_skips_if_collection_has_data() {
        DataContext ctx = new DataContext("seed-existing");
        var doc = new com.fasterxml.jackson.databind.ObjectMapper()
            .createObjectNode().put("title", "Existing");
        provider.create(ctx, "tickets", doc, "title");

        registry.ensureSeeded(ctx, "helpdesk", "tickets");

        // Seed data should NOT be loaded since collection already has documents
        Optional<JsonNode> seeded = provider.get(ctx, "tickets", "welcome-ticket");
        assertThat(seeded).isEmpty();
    }

    @Test
    void seed_skips_unknown_app() {
        DataContext ctx = new DataContext("seed-unknown");
        registry.ensureSeeded(ctx, "nonexistent", "whatever");
        // No exception — just a no-op
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl data-json -Dtest=SeedDataTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: FAIL — `ensureSeeded` method does not exist

- [ ] **Step 4: Add countDocuments to JsonCollectionStore**

Add method:

```java
public int countDocuments(String tenantId, String appId, String collection) {
    String sql = "SELECT COUNT(*) FROM json_documents WHERE tenant_id = ? AND app_id = ? AND collection = ?";
    try (Connection conn = connectionFactory.getConnection();
         PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, tenantId);
        ps.setString(2, appId);
        ps.setString(3, collection);
        try (ResultSet rs = ps.executeQuery()) {
            return rs.next() ? rs.getInt(1) : 0;
        }
    } catch (SQLException e) {
        throw new MutationException("STORAGE_ERROR", "Count query failed: " + e.getMessage(), e);
    }
}
```

- [ ] **Step 5: Add ensureSeeded to AppRegistry**

Inject `JsonCollectionStore` (via `JsonDataProvider` or directly via `SqliteConnectionFactory`). Add seed loading:

```java
@Inject
SqliteConnectionFactory connectionFactory;

private JsonCollectionStore store;

@PostConstruct
void init() {
    store = new JsonCollectionStore(connectionFactory);
    scanClasspath();
}

public void ensureSeeded(DataContext ctx, String appId, String collection) {
    AppDescriptor descriptor = descriptors.get(appId);
    if (descriptor == null) return;

    CollectionDef colDef = descriptor.collections().stream()
        .filter(c -> c.name().equals(collection))
        .findFirst().orElse(null);
    if (colDef == null || colDef.dataFiles().isEmpty()) return;

    if (store.countDocuments(ctx.tenantId(), appId, collection) > 0) return;

    loadSeedData(ctx, appId, collection, colDef.dataFiles());
}

private void loadSeedData(DataContext ctx, String appId, String collection,
                          List<String> dataFiles) {
    try (Connection conn = connectionFactory.getConnection()) {
        conn.setAutoCommit(false);
        try (var stmt = conn.prepareStatement("BEGIN IMMEDIATE")) {
            stmt.execute();
        }

        // Re-check inside transaction (TOCTOU guard)
        if (store.countDocuments(ctx.tenantId(), appId, collection) > 0) {
            conn.rollback();
            return;
        }

        ObjectMapper mapper = new ObjectMapper();
        for (String dataFile : dataFiles) {
            String resourcePath = APPS_PREFIX + appId + "/" + dataFile;
            try (InputStream is = Thread.currentThread().getContextClassLoader()
                    .getResourceAsStream(resourcePath)) {
                if (is == null) {
                    LOG.warnf("Seed data file not found: %s", resourcePath);
                    continue;
                }
                JsonNode array = mapper.readTree(is);
                if (!array.isArray()) continue;
                for (JsonNode doc : array) {
                    store.create(ctx, appId, collection, doc, "title");
                }
            }
        }
        conn.commit();
    } catch (Exception e) {
        LOG.warnf(e, "Failed to load seed data for %s/%s", appId, collection);
    }
}
```

Note: SQLite's `BEGIN IMMEDIATE` acquires a write lock immediately, preventing TOCTOU races. The re-check inside the transaction ensures idempotency under concurrent access.

- [ ] **Step 6: Run tests**

Run: `mvn test -pl data-json -Dtest=SeedDataTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

Also run all data-json tests:
Run: `mvn test -pl data-json -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add data-json/src/
git commit -m "feat(#596): seed data loading — lazy, idempotent, BEGIN IMMEDIATE

Loads JSON seed files on first collection access. Skips if collection
already has documents. Uses BEGIN IMMEDIATE to prevent TOCTOU races.

Refs #596"
```

---

## Batch 4: Referential integrity

### Task 6: Reference field resolution and restrict-on-delete

**Files:**
- Modify: `data-json/src/main/java/io/casehub/pages/data/json/JsonDataProvider.java` — implement `hasReferences`
- Modify: `data-json/src/main/java/io/casehub/pages/data/json/JsonCollectionStore.java` — add reference lookup query
- Modify: `data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java` — add ref lookup method
- Test: `data-json/src/test/java/io/casehub/pages/data/json/ReferentialIntegrityTest.java`

**Interfaces:**
- Consumes:
  - `CollectionSchema.refs()` → `Map<String, String>` (field → target collection)
  - `AppRegistry.getSchema(...)` from Task 3
  - `JsonCollectionStore` query methods from #595
- Produces:
  - `AppRegistry.getReferencingCollections(String targetCollection)` → `Map<String, String>` (sourceCollection → field name)
  - `JsonDataProvider.hasReferences(DataContext, String, String)` → real implementation

- [ ] **Step 1: Write the failing test**

Create `ReferentialIntegrityTest.java`:

```java
package io.casehub.pages.data.json;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.pages.data.DataContext;
import io.casehub.pages.data.MutationException;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class ReferentialIntegrityTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Inject
    JsonDataProvider provider;

    @Inject
    AppRegistry registry;

    @Test
    void hasReferences_returns_true_when_referenced() {
        DataContext ctx = new DataContext("ref-integrity");

        // Create an agent
        ObjectNode agent = MAPPER.createObjectNode()
            .put("name", "Agent One")
            .put("email", "one@test.com");
        String agentId = provider.create(ctx, "agents", agent, "name");

        // Create a ticket referencing that agent
        ObjectNode ticket = MAPPER.createObjectNode()
            .put("title", "Ref Test Ticket")
            .put("priority", "low")
            .put("assignee", agentId);
        provider.create(ctx, "tickets", ticket, "title");

        assertThat(provider.hasReferences(ctx, "agents", agentId)).isTrue();
    }

    @Test
    void hasReferences_returns_false_when_not_referenced() {
        DataContext ctx = new DataContext("ref-integrity-none");

        ObjectNode agent = MAPPER.createObjectNode()
            .put("name", "Lonely Agent")
            .put("email", "lonely@test.com");
        String agentId = provider.create(ctx, "agents", agent, "name");

        assertThat(provider.hasReferences(ctx, "agents", agentId)).isFalse();
    }

    @Test
    void delete_blocked_when_referenced() {
        DataContext ctx = new DataContext("ref-delete-block");

        ObjectNode agent = MAPPER.createObjectNode()
            .put("name", "Protected Agent")
            .put("email", "protected@test.com");
        String agentId = provider.create(ctx, "agents", agent, "name");

        ObjectNode ticket = MAPPER.createObjectNode()
            .put("title", "Blocking Ticket")
            .put("priority", "high")
            .put("assignee", agentId);
        provider.create(ctx, "tickets", ticket, "title");

        // CollectionResource calls hasReferences before delete — we test the
        // provider method directly here. The REST layer's restrict-on-delete
        // already exists in CollectionResource.delete().
        assertThat(provider.hasReferences(ctx, "agents", agentId)).isTrue();
    }

    @Test
    void hasReferences_returns_false_for_unregistered_collections() {
        DataContext ctx = new DataContext("ref-unreg");
        assertThat(provider.hasReferences(ctx, "unknown-collection", "any-id")).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl data-json -Dtest=ReferentialIntegrityTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: FAIL — `hasReferences` returns false always

- [ ] **Step 3: Add getReferencingCollections to AppRegistry**

```java
public Map<String, String> getReferencingCollections(String targetCollection) {
    Map<String, String> result = new LinkedHashMap<>();
    for (Map.Entry<String, Map<String, CollectionSchema>> appEntry : schemas.entrySet()) {
        for (Map.Entry<String, CollectionSchema> colEntry : appEntry.getValue().entrySet()) {
            CollectionSchema schema = colEntry.getValue();
            for (Map.Entry<String, String> refEntry : schema.refs().entrySet()) {
                if (refEntry.getValue().equals(targetCollection)) {
                    result.put(colEntry.getKey(), refEntry.getKey());
                }
            }
        }
    }
    return result;
}
```

- [ ] **Step 4: Add reference check query to JsonCollectionStore**

```java
public boolean hasDocumentReferencing(String tenantId, String appId,
                                      String sourceCollection, String fieldName,
                                      String targetDocId) {
    String sql = "SELECT COUNT(*) FROM json_documents " +
        "WHERE tenant_id = ? AND app_id = ? AND collection = ? " +
        "AND json_extract(data, '$.' || ?) = ?";
    try (Connection conn = connectionFactory.getConnection();
         PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, tenantId);
        ps.setString(2, appId);
        ps.setString(3, sourceCollection);
        ps.setString(4, fieldName);
        ps.setString(5, targetDocId);
        try (ResultSet rs = ps.executeQuery()) {
            return rs.next() && rs.getInt(1) > 0;
        }
    } catch (SQLException e) {
        throw new MutationException("STORAGE_ERROR",
            "Reference check failed: " + e.getMessage(), e);
    }
}
```

- [ ] **Step 5: Implement hasReferences in JsonDataProvider**

Replace the current stub:

```java
@Inject
AppRegistry appRegistry;

@Override
public boolean hasReferences(DataContext ctx, String collection, String docId) {
    Map<String, String> referencing = appRegistry.getReferencingCollections(collection);
    if (referencing.isEmpty()) return false;

    for (Map.Entry<String, String> entry : referencing.entrySet()) {
        String sourceCollection = entry.getKey();
        String fieldName = entry.getValue();
        if (store.hasDocumentReferencing(ctx.tenantId(), ctx.tenantId(),
                sourceCollection, fieldName, docId)) {
            return true;
        }
    }
    return false;
}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl data-json -Dtest=ReferentialIntegrityTest -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

Also run all data-json tests:
Run: `mvn test -pl data-json -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

And run the data SPI tests:
Run: `mvn test -pl data -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add data-json/src/
git commit -m "feat(#596): referential integrity — hasReferences via schema refs

Queries json_extract to find documents referencing a target doc ID.
CollectionResource already calls hasReferences before delete, so
restrict-on-delete is now enforced for declared ref fields.

Refs #596"
```

---

## Schema Evolution (deferred validation)

The issue specifies schema evolution rules:
- New required fields must declare a `default`
- Type widening allowed, narrowing rejected

These rules apply when an `app.yaml` changes between deployments, not at runtime.
Since we're pre-release with no deployed apps yet, schema evolution validation
is tracked but not implemented in this issue. The `defaults` mechanism in
`CollectionSchema.applyDefaults()` already handles the "new field with default"
case for writes. Narrowing detection would require comparing old vs new schema
versions, which needs a schema version store — better as a follow-up issue.

---

## References

- `backend/data-json/src/main/java/io/casehub/pages/data/json/JsonDataProvider.java` — storage provider being extended
- `backend/data-json/src/main/java/io/casehub/pages/data/json/JsonCollectionStore.java` — SQLite JSONB store
- `backend/data/src/main/java/io/casehub/pages/data/MutableDataProvider.java` — SPI interface
- `backend/data/src/main/java/io/casehub/pages/data/CollectionResource.java` — REST layer (already calls `hasReferences`)
- `backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/ScriptRegistryProducer.java` — classpath scan pattern
- `docs/protocols/casehub/dataset-contract.md` — dataset contract protocol
- GitHub #596 — focal issue
- GitHub #595 — dependency (data-json module)
- GitHub #592 — parent issue
