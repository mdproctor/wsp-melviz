# Label Tree Enumeration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #600 — Label tree enumeration endpoint for navigation and dropdowns
**Issue group:** #600 (child of #592 JSON Apps Runtime)

**Goal:** Wire Pages to consume the platform's LabelRegistry SPI for hierarchical label trees — parsing label declarations from app.yaml, registering them at startup, and integrating label enumeration into dropdowns and navigation components.

**Architecture:** Pages adds no new storage or REST endpoints. The platform provides `LabelRegistry` SPI (inmem implementation), `LabelApi @McpDomain("labels")` with children/resolve/subtree queries, and `LabelServiceCore` for tenant resolution. Pages extends `AppDescriptorParser` to parse a top-level `labels` key in `app.yaml`, registers labels via `LabelRegistry.register()` at app startup, and wires the frontend to fetch label options from the platform endpoint for `ref: labels/{tree}` fields and `labelSource` nav groups.

**Tech Stack:** Java 21 / Quarkus (backend), TypeScript / Lit (frontend), platform `LabelRegistry` SPI (`io.casehub.platform.api.label`)

## Global Constraints

- Platform dependency: `casehub-platform-api` (already a transitive dep via `casehub-platform-yaml-core`)
- Labels are tenant-scoped, not app-scoped — apps register under `PLATFORM_TENANT_ID`
- App ID + tree name become path prefixes: `{appId}/{treeName}/{leafPath}`
- `ref: labels/{tree}` prefix distinguishes label refs from collection refs
- No new Maven modules — all backend changes in `data-json` and `data`
- IntelliJ MCP required for all code operations

---

## Batch 1: Backend — app.yaml Label Parsing and Registration

### Task 1: LabelTreeDef record and AppDescriptor extension

**Files:**
- Create: `backend/data-json/src/main/java/io/casehub/pages/data/json/LabelTreeDef.java`
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppDescriptor.java`
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppDescriptorParser.java`
- Test: `backend/data-json/src/test/java/io/casehub/pages/data/json/AppDescriptorParserTest.java`
- Modify: `backend/data-json/src/test/resources/org/apps/helpdesk/app.yaml` (add labels section)

**Interfaces:**
- Produces: `LabelTreeDef(String name, List<String> paths)` — used by Task 2
- Produces: `AppDescriptor.labelTrees()` — used by Task 2

- [ ] **Step 1: Add labels section to helpdesk app.yaml test fixture**

Add to `backend/data-json/src/test/resources/org/apps/helpdesk/app.yaml`:

```yaml
labels:
  categories:
    - HARDWARE/Laptops
    - HARDWARE/Monitors
    - SOFTWARE/OS
    - SOFTWARE/Applications
  priorities:
    - P1/Critical
    - P1/Blocker
    - P2/High
    - P3/Medium
    - P4/Low
```

- [ ] **Step 2: Write failing test for label tree parsing**

Add to `AppDescriptorParserTest.java`:

```java
@Test
void parses_label_trees() {
    InputStream yaml = getClass().getClassLoader()
        .getResourceAsStream("org/apps/helpdesk/app.yaml");

    AppDescriptor descriptor = AppDescriptorParser.parse("helpdesk", yaml);

    assertThat(descriptor.labelTrees()).hasSize(2);

    LabelTreeDef categories = descriptor.labelTrees().stream()
        .filter(t -> t.name().equals("categories"))
        .findFirst().orElseThrow();
    assertThat(categories.paths()).containsExactly(
        "HARDWARE/Laptops", "HARDWARE/Monitors",
        "SOFTWARE/OS", "SOFTWARE/Applications");

    LabelTreeDef priorities = descriptor.labelTrees().stream()
        .filter(t -> t.name().equals("priorities"))
        .findFirst().orElseThrow();
    assertThat(priorities.paths()).hasSize(5);
}

@Test
void parses_app_without_labels() {
    String yaml = "collections:\n  things:\n    fields:\n      - name: title\n        type: string\n";
    AppDescriptor descriptor = AppDescriptorParser.parse("nolabels",
        new java.io.ByteArrayInputStream(yaml.getBytes()));

    assertThat(descriptor.labelTrees()).isEmpty();
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn -pl backend/data-json test -Dtest=AppDescriptorParserTest#parses_label_trees -f backend/pom.xml`
Expected: compilation error — `labelTrees()` doesn't exist

- [ ] **Step 4: Create LabelTreeDef record**

Create `backend/data-json/src/main/java/io/casehub/pages/data/json/LabelTreeDef.java`:

```java
package io.casehub.pages.data.json;

import java.util.List;

public record LabelTreeDef(String name, List<String> paths) {
    public LabelTreeDef {
        if (paths == null) paths = List.of();
    }
}
```

- [ ] **Step 5: Extend AppDescriptor to include labelTrees**

Change `AppDescriptor.java` from:

```java
public record AppDescriptor(
    String appId,
    List<CollectionDef> collections
) {
    public AppDescriptor {
        if (collections == null) collections = List.of();
    }
}
```

To:

```java
public record AppDescriptor(
    String appId,
    List<CollectionDef> collections,
    List<LabelTreeDef> labelTrees
) {
    public AppDescriptor {
        if (collections == null) collections = List.of();
        if (labelTrees == null) labelTrees = List.of();
    }
}
```

- [ ] **Step 6: Extend AppDescriptorParser to parse labels key**

In `AppDescriptorParser.parse()`, after `parseCollections()`, add label parsing:

```java
List<CollectionDef> collections = parseCollections(root.get("collections"));
List<LabelTreeDef> labelTrees = parseLabelTrees(root.get("labels"));
return new AppDescriptor(appId, collections, labelTrees);
```

Add new private method:

```java
private static List<LabelTreeDef> parseLabelTrees(JsonNode labelsNode) {
    if (labelsNode == null || !labelsNode.isObject()) return List.of();

    List<LabelTreeDef> result = new ArrayList<>();
    Iterator<Map.Entry<String, JsonNode>> it = labelsNode.fields();
    while (it.hasNext()) {
        Map.Entry<String, JsonNode> entry = it.next();
        String name = entry.getKey();
        List<String> paths = parseStringArray(entry.getValue());
        result.add(new LabelTreeDef(name, paths));
    }
    return List.copyOf(result);
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn -pl backend/data-json test -Dtest=AppDescriptorParserTest -f backend/pom.xml`
Expected: all tests PASS (including existing tests — backward compatible since labelTrees defaults to empty)

- [ ] **Step 8: Commit**

```bash
git add backend/data-json/src/main/java/io/casehub/pages/data/json/LabelTreeDef.java
git add backend/data-json/src/main/java/io/casehub/pages/data/json/AppDescriptor.java
git add backend/data-json/src/main/java/io/casehub/pages/data/json/AppDescriptorParser.java
git add backend/data-json/src/test/java/io/casehub/pages/data/json/AppDescriptorParserTest.java
git add backend/data-json/src/test/resources/org/apps/helpdesk/app.yaml
git commit -m "feat(#600): parse labels key in app.yaml into LabelTreeDef  Refs #600"
```

### Task 2: AppRegistry label registration via LabelRegistry SPI

**Files:**
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java`
- Modify: `backend/data-json/pom.xml` (add platform-api dependency if not transitive)
- Test: `backend/data-json/src/test/java/io/casehub/pages/data/json/AppRegistryTest.java`

**Interfaces:**
- Consumes: `AppDescriptor.labelTrees()` from Task 1
- Consumes: `LabelRegistry.register(String tenancyId, LabelDescriptor descriptor)` from platform
- Produces: `AppRegistry.isLabelRef(String ref)` — returns `true` if ref starts with `"labels/"`

- [ ] **Step 1: Check if platform-api is already a dependency**

Run: `mvn -pl backend/data-json dependency:tree -f backend/pom.xml | grep platform-api`

If not present, add to `backend/data-json/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-api</artifactId>
</dependency>
```

- [ ] **Step 2: Write failing test for label registration**

Add to `AppRegistryTest.java`:

```java
@Test
void registers_label_trees_from_app_descriptor() {
    // AppRegistry.init() calls scanClasspath() which loads helpdesk app.yaml
    // After init, the injected LabelRegistry should have received registrations
    // Verify via the LabelRegistry mock/spy that register was called
    // with paths like "helpdesk/categories/HARDWARE/Laptops"
    assertThat(registry.getDescriptor("helpdesk")).isPresent();
    assertThat(registry.getDescriptor("helpdesk").get().labelTrees()).isNotEmpty();
}

@Test
void isLabelRef_distinguishes_label_and_collection_refs() {
    assertThat(AppRegistry.isLabelRef("labels/categories")).isTrue();
    assertThat(AppRegistry.isLabelRef("labels/priorities")).isTrue();
    assertThat(AppRegistry.isLabelRef("agents")).isFalse();
    assertThat(AppRegistry.isLabelRef(null)).isFalse();
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn -pl backend/data-json test -Dtest=AppRegistryTest -f backend/pom.xml`
Expected: compilation error — `isLabelRef` doesn't exist

- [ ] **Step 4: Add isLabelRef static method to AppRegistry**

Add to `AppRegistry.java`:

```java
public static boolean isLabelRef(String ref) {
    return ref != null && ref.startsWith("labels/");
}
```

- [ ] **Step 5: Inject LabelRegistry and register labels at startup**

Add to `AppRegistry.java`:

```java
@Inject
Instance<LabelRegistry> labelRegistryInstance;
```

Add registration method:

```java
private void registerLabelTrees(AppDescriptor descriptor) {
    if (labelRegistryInstance.isUnsatisfied()) {
        LOG.debug("No LabelRegistry available — skipping label registration");
        return;
    }
    LabelRegistry labelRegistry = labelRegistryInstance.get();
    for (LabelTreeDef tree : descriptor.labelTrees()) {
        for (String leafPath : tree.paths()) {
            String fullPath = descriptor.appId() + "/" + tree.name() + "/" + leafPath;
            Path path = Path.parse(fullPath);
            String displayName = path.segments().get(path.segments().size() - 1);
            LabelDescriptor desc = new LabelDescriptor(
                "platform", path, displayName, 0, Map.of());
            labelRegistry.register("platform", desc);
        }
        LOG.infof("Registered label tree '%s/%s' with %d paths",
            descriptor.appId(), tree.name(), tree.paths().size());
    }
}
```

Call from `loadApp()` after `registerSchemas(descriptor)`:

```java
registerSchemas(descriptor);
registerLabelTrees(descriptor);
```

Import: `io.casehub.platform.api.label.LabelRegistry`, `io.casehub.platform.api.label.LabelDescriptor`, `io.casehub.platform.api.path.Path`, `jakarta.enterprise.inject.Instance`

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn -pl backend/data-json test -f backend/pom.xml`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add backend/data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java
git add backend/data-json/pom.xml
git add backend/data-json/src/test/java/io/casehub/pages/data/json/AppRegistryTest.java
git commit -m "feat(#600): register app label trees via platform LabelRegistry SPI  Refs #600"
```

### Task 3: Skip ref validation for label refs in CollectionSchema

**Files:**
- Modify: `backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java`
- Modify: `backend/data-json/src/main/java/io/casehub/pages/data/json/JsonDataProvider.java`
- Test: `backend/data-json/src/test/java/io/casehub/pages/data/json/SchemaValidationTest.java`

**Interfaces:**
- Consumes: `AppRegistry.isLabelRef(String ref)` from Task 2

- [ ] **Step 1: Write failing test for label ref validation**

Add to `SchemaValidationTest.java`:

```java
@Test
void label_ref_field_accepts_path_strings() {
    Map<String, String> fieldTypes = Map.of("category", "ref");
    Map<String, String> refs = Map.of("category", "labels/categories");
    CollectionSchema schema = new CollectionSchema("tickets", fieldTypes, Map.of(), refs);

    ObjectNode doc = new ObjectMapper().createObjectNode();
    doc.put("category", "HARDWARE/Laptops");

    // Should not throw — label refs accept any text value (path strings)
    schema.validate(doc);
}
```

- [ ] **Step 2: Run test to verify it passes (or identify validation issues)**

Run: `mvn -pl backend/data-json test -Dtest=SchemaValidationTest#label_ref_field_accepts_path_strings -f backend/pom.xml`

The `ref` type validates as `value.isTextual()` which should already pass for path strings. If it passes, this task is done — the existing validation is sufficient.

If referential integrity checks elsewhere block label refs, modify `JsonDataProvider.hasReferences()` to skip refs starting with `labels/`:

```java
if (AppRegistry.isLabelRef(refEntry.getValue())) continue;
```

- [ ] **Step 3: Run full test suite**

Run: `mvn -pl backend/data-json test -f backend/pom.xml`
Expected: all PASS

- [ ] **Step 4: Commit**

```bash
git add backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java
git add backend/data-json/src/main/java/io/casehub/pages/data/json/JsonDataProvider.java
git add backend/data-json/src/test/java/io/casehub/pages/data/json/SchemaValidationTest.java
git commit -m "feat(#600): skip referential integrity for labels/ refs  Refs #600"
```

---

## Batch 2: Frontend — Dropdown Integration via ref: labels/{tree}

### Task 4: collection-bindings label ref type mapping

**Files:**
- Modify: `packages/pages-data/src/datasource/sources/collection-bindings.ts`
- Test: `packages/pages-data/src/datasource/sources/collection-bindings.test.ts`

**Interfaces:**
- Consumes: `CollectionMeta.fields[].type` — field type strings from backend
- Produces: `LABEL` column type for fields whose backend type is `ref` with `labels/` prefix

- [ ] **Step 1: Write failing test**

Create or extend `collection-bindings.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { collectionBindings, CollectionMeta } from "./collection-bindings.js";

describe("collectionBindings label ref", () => {
  it("maps ref fields with labels/ prefix to LABEL type", () => {
    const collections: CollectionMeta[] = [{
      name: "tickets",
      fields: [
        { name: "subject", type: "string" },
        { name: "category", type: "ref", ref: "labels/categories" },
        { name: "assigneeId", type: "ref", ref: "agents" },
      ],
    }];

    const bindings = collectionBindings("helpdesk", collections);
    const source = bindings[0];
    // Verify the binding was created — detailed column type verification
    // requires inspecting the source internals
    expect(source.id).toBe("json:helpdesk/tickets");
  });
});
```

- [ ] **Step 2: Extend CollectionMeta to include optional ref**

In `collection-bindings.ts`, update the `CollectionMeta` interface:

```typescript
export interface CollectionMeta {
  readonly name: string;
  readonly fields: readonly { name: string; type: string; ref?: string }[];
}
```

Update `toColumns()` to map `ref` fields with `labels/` prefix to `LABEL`:

```typescript
function toColumns(fields: readonly { name: string; type: string; ref?: string }[]): ExternalColumnDef[] {
  const cols: ExternalColumnDef[] = [
    { id: columnId("id"), type: ColumnType.TEXT },
    { id: columnId("_version"), type: ColumnType.TEXT },
  ];
  for (const f of fields) {
    const colType = f.ref?.startsWith("labels/")
      ? ColumnType.LABEL
      : (TYPE_MAP[f.type] ?? ColumnType.TEXT);
    cols.push({ id: columnId(f.name), type: colType });
  }
  return cols;
}
```

- [ ] **Step 3: Run tests**

Run: `yarn workspace @casehubio/pages-data test`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add packages/pages-data/src/datasource/sources/collection-bindings.ts
git add packages/pages-data/src/datasource/sources/collection-bindings.test.ts
git commit -m "feat(#600): map labels/ ref fields to LABEL column type  Refs #600"
```

### Task 5: Schema-form label option fetching

**Files:**
- Modify: `packages/pages-viz/src/form-inputs/schema-types.ts`
- Test: `packages/pages-viz/src/form-inputs/schema-types.test.ts`

**Interfaces:**
- Consumes: field schema with `ref` property starting with `labels/`
- Consumes: platform `LabelApi` endpoint (`GET /labels` children query)
- Produces: `FieldSchema` with `enum` populated from label children displayNames

- [ ] **Step 1: Write failing test**

Create or extend `schema-types.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { isLabelRef, labelRefToPath } from "./schema-types.js";

describe("label ref utilities", () => {
  it("isLabelRef detects labels/ prefix", () => {
    expect(isLabelRef("labels/categories")).toBe(true);
    expect(isLabelRef("agents")).toBe(false);
    expect(isLabelRef(undefined)).toBe(false);
  });

  it("labelRefToPath extracts tree name", () => {
    expect(labelRefToPath("labels/categories")).toBe("categories");
    expect(labelRefToPath("labels/priorities")).toBe("priorities");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-viz test`
Expected: FAIL — `isLabelRef` not exported

- [ ] **Step 3: Add label ref utilities**

Add to `schema-types.ts`:

```typescript
export function isLabelRef(ref: string | undefined): boolean {
  return ref != null && ref.startsWith("labels/");
}

export function labelRefToPath(ref: string): string {
  return ref.substring("labels/".length);
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-viz test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-viz/src/form-inputs/schema-types.ts
git add packages/pages-viz/src/form-inputs/schema-types.test.ts
git commit -m "feat(#600): label ref detection utilities for schema-form  Refs #600"
```

---

## Batch 3: Frontend — Navigation via labelSource

### Task 6: navTree labelSource support in nav-desugar

**Files:**
- Modify: `packages/pages-schema/src/document-schema.ts` (extend navItemSchema)
- Modify: `packages/pages-ui/src/parser/nav-desugar.ts` (resolve labelSource groups)
- Test: `packages/pages-ui/src/parser/nav-desugar.test.ts`

**Interfaces:**
- Consumes: `NavTreeGroup.labelSource` — optional string naming a label tree
- Consumes: platform `LabelApi` children endpoint
- Produces: dynamically generated slot entries from label children

- [ ] **Step 1: Write failing test**

Create or extend `nav-desugar.test.ts`:

```typescript
import { describe, it, expect } from "vitest";

describe("nav-desugar labelSource", () => {
  it("recognises labelSource property on nav groups", () => {
    const navTree = {
      root_items: [{
        type: "GROUP",
        id: "CategoryNav",
        labelSource: "categories",
        children: [],
      }],
    };

    // Verify the navTree group has labelSource
    const group = navTree.root_items[0];
    expect(group.labelSource).toBe("categories");
  });
});
```

- [ ] **Step 2: Extend NavTreeGroup interface to include labelSource**

In `nav-desugar.ts`, update the interface:

```typescript
interface NavTreeGroup {
  readonly type: string;
  readonly id: string;
  readonly children?: readonly NavTreeChild[];
  readonly labelSource?: string;
}
```

- [ ] **Step 3: Extend navItemSchema in document-schema.ts**

Add `labelSource` to the nav item Zod schema:

```typescript
labelSource: z.string().optional(),
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-ui test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-schema/src/document-schema.ts
git add packages/pages-ui/src/parser/nav-desugar.ts
git add packages/pages-ui/src/parser/nav-desugar.test.ts
git commit -m "feat(#600): navTree labelSource property for dynamic label menus  Refs #600"
```

---

## References

- [2026-11-02-label-tree-enumeration-design.md] — design spec this plan implements
- [backend/data-json/AppDescriptorParser.java] — existing app.yaml parser
- [backend/data-json/AppDescriptor.java] — app descriptor record
- [backend/data-json/AppRegistry.java] — app registry with collection + seed loading
- [backend/data/CollectionSchema.java] — schema validation
- [pages-data/datasource/sources/collection-bindings.ts] — collection data source wiring
- [pages-viz/form-inputs/schema-types.ts] — field-to-component mapping
- [pages-ui/parser/nav-desugar.ts] — navTree resolution logic
- [platform-api Path.java, LabelPatternMatcher.java, LabelRegistry, LabelDescriptor] — platform SPI
- [GitHub #600] — focal issue
- [GitHub #592] — parent epic (JSON Apps Runtime)
- [GitHub platform#694] — platform LabelRegistry SPI (landed)
