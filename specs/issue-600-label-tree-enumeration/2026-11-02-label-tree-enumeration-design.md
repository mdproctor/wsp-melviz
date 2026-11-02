# Label Tree Enumeration for Navigation and Dropdowns

**Issue:** casehubio/casehub-pages#600
**Parent:** casehubio/casehub-pages#592 (JSON Apps Runtime)
**Depends on:** casehubio/platform#694 (LabelRegistry SPI — landed)

## Summary

Wire Pages to consume the platform's `LabelRegistry` SPI and `LabelApi` endpoint for hierarchical label trees. Pages registers app-defined labels at startup and integrates label enumeration into dropdowns and navigation components.

No new storage, no new REST endpoint in Pages. The platform provides the SPI, storage (inmem now, JPA later), and the `@McpDomain("labels")` endpoint.

## Platform Deliverables (already landed — platform #694)

- `LabelDescriptor` record + `LabelRegistry` SPI + CDI events in `platform-api`
- `LabelApi @McpDomain("labels")` with children/resolve/subtree queries
- `LabelServiceCore` in `platform-core` (tenancyId from `CurrentPrincipal`)
- `NoOpLabelRegistry @DefaultBean` in platform
- `InMemoryLabelRegistry` in `label-inmem-core` + CDI wiring in `label-inmem`

Pages depends on this SPI. Labels are tenant-scoped, not app-scoped. Apps register labels under `PLATFORM_TENANT_ID`; tenants can extend with their own. Resolution merges both with tenant-wins precedence.

## Pages Deliverable 1: app.yaml Label Parsing and Registration

### YAML Surface

The `app.yaml` descriptor gains a top-level `labels` key, sibling to `collections`:

```yaml
collections:
  tickets:
    fields:
      - name: category
        type: ref
        ref: labels/categories
      - name: priority
        type: ref
        ref: labels/priorities
    data:
      - data/tickets.json

labels:
  categories:
    - IT/Hardware/Laptops
    - IT/Hardware/Monitors
    - IT/Software/OS
    - IT/Software/Applications
    - HR/Onboarding
    - HR/Offboarding
  priorities:
    - P1/Critical
    - P1/Blocker
    - P2/High
    - P3/Medium
    - P4/Low
```

Each key under `labels` is a named label tree. Values are leaf paths — the platform `LabelRegistry` auto-creates ancestor nodes when a leaf is registered.

### Backend Changes

**New record — `LabelTreeDef`** (`data-json`):
```java
public record LabelTreeDef(String name, List<String> paths) {}
```

**Extended — `AppDescriptor`**:
```java
public record AppDescriptor(String appId, List<CollectionDef> collections,
                            List<LabelTreeDef> labelTrees) {}
```

**Extended — `AppDescriptorParser.parse()`**:
- Parse the `labels` top-level key from YAML
- For each entry: create a `LabelTreeDef(name, paths)`

**Extended — `AppRegistry`**:
- Inject `LabelRegistry`
- On `@PostConstruct`, after collection registration, iterate label trees
- For each path in each tree, call:
  ```java
  Path fullPath = Path.parse(appId + "/" + treeName + "/" + leafPath);
  LabelDescriptor desc = new LabelDescriptor(
      PLATFORM_TENANT_ID, fullPath, deriveDisplayName(fullPath), 0, Map.of());
  labelRegistry.register(PLATFORM_TENANT_ID, desc);
  ```
- The app ID and tree name become path prefixes: `helpdesk/categories/IT/Hardware`

### Path Structure

```
helpdesk/                          ← app root (auto-created)
  categories/                      ← tree root (auto-created)
    IT/                            ← auto-created ancestor
      Hardware/                    ← auto-created ancestor
        Laptops                    ← registered leaf
        Monitors                   ← registered leaf
      Software/
        OS
        Applications
    HR/
      Onboarding
      Offboarding
  priorities/
    P1/
      Critical
      Blocker
    P2/
      High
    P3/
      Medium
    P4/
      Low
```

## Pages Deliverable 2: Dropdown Integration via `ref: labels/{tree}`

### app.yaml Field Syntax

```yaml
fields:
  - name: category
    type: ref
    ref: labels/categories
```

The `labels/` prefix on the `ref` field distinguishes label references from collection references (e.g., `ref: agents`).

### Backend Changes

**`AppRegistry`**:
- Add `isLabelRef(String ref)` — returns true if ref starts with `labels/`
- Existing `getReferencingCollections()` and `hasReferences()` skip label refs (no referential integrity for labels)

**`CollectionSchema.validate()`**:
- Skip ref validation for label refs — label values are paths, not document IDs

### Frontend Changes

**`collectionBindings()` (`pages-data`)**:
- When a field has `ref: labels/{tree}`, map column type to `LABEL` (not `TEXT`)

**`columnToFieldSchema()` (`pages-viz`)**:
- When the field's ref starts with `labels/`, fetch options from the platform's `LabelApi` children endpoint using the app's root path + tree name
- Set fetched `displayName` values as the `enum` array on the field schema
- The property palette then renders this as a `pages-select` dropdown (existing path)

**Cascading dropdowns**:
- When two fields share the same label tree ref, selecting a value in the first field narrows the second field's options by re-fetching with the selected path as prefix
- Uses the existing filter notification mechanism between components

### Data Flow

1. Schema-form renders a field with `ref: labels/categories`
2. Frontend resolves to platform endpoint: `GET /labels` children query with path `helpdesk/categories`
3. Returns top-level children: `[{path: "IT", displayName: "IT", childCount: 2}, {path: "HR", ...}]`
4. Rendered as `pages-select` dropdown options
5. For cascading: selecting `IT` triggers re-fetch at `helpdesk/categories/IT` for next-level children

## Pages Deliverable 3: Navigation via `labelSource`

### .page.yaml Syntax

```yaml
navTree:
  root_items:
  - type: GROUP
    id: CategoryNav
    labelSource: categories
    children: []
```

The `labelSource` property on a navTree group binds it to a label tree. Children are populated dynamically from the `LabelRegistry` instead of being declared statically.

### Frontend Changes

**`document-schema.ts` (`pages-schema`)**:
- Extend `navItemSchema` to accept optional `labelSource: z.string().optional()`

**`nav-desugar.ts` (`pages-ui`)**:
- In `resolveNavGroup()`: if a group has `labelSource`, fetch children from the platform `LabelApi` and generate slot entries dynamically (one per label child)
- Each generated slot entry uses the label's `displayName` as the slot name

**`navigation.ts` (`pages-runtime`)**:
- Selecting a label-generated nav item emits a `pages-filter` event with the label path as the filter value
- The bound data view listens for this filter and restricts its dataset accordingly

### Data Flow

1. Page parser encounters `labelSource: categories` on nav group `CategoryNav`
2. At render time, fetches children from platform: `helpdesk/categories` top-level
3. Renders `IT`, `HR` as selectable items in the tree/menu/tabs component
4. Selecting `IT` → emits filter event → data table shows only items labelled under `IT/**`
5. Tree component can expand `IT` → fetches `helpdesk/categories/IT` children → shows `Hardware`, `Software`

## Files Changed

### Backend (`backend/data-json`)
- `LabelTreeDef.java` — new record
- `AppDescriptor.java` — add `labelTrees` field
- `AppDescriptorParser.java` — parse `labels` top-level key
- `AppRegistry.java` — inject `LabelRegistry`, register labels at startup, add `isLabelRef()`
- `CollectionSchema.java` — skip ref validation for label refs

### Frontend
- `pages-schema/document-schema.ts` — extend navItemSchema with `labelSource`
- `pages-schema/component-schemas.ts` — accept `labels/` prefix in ref field validation
- `pages-data/datasource/sources/collection-bindings.ts` — handle `labels/` ref type mapping
- `pages-viz/form-inputs/schema-types.ts` — fetch label options for `labels/` refs
- `pages-ui/parser/nav-desugar.ts` — resolve `labelSource` groups dynamically
- `pages-runtime/navigation.ts` — emit filter events for label nav items

### Test Resources
- `backend/data-json/src/test/resources/org/apps/helpdesk/app.yaml` — add `labels` section

## References

- `platform-api/path/Path.java` — hierarchical path type used by LabelDescriptor
- `platform-api/view/LabelPatternMatcher.java` — glob matching for label paths
- `platform-api/label/LabelRule.java`, `LabelAction.java` — rules engine for label mutations
- `platform-api/view/SubjectViewSpec.java` — views filtered by label pattern + scope
- `platform #694` — LabelRegistry SPI, LabelApi @McpDomain, InMemoryLabelRegistry
- `backend/data-json/AppDescriptorParser.java` — existing app.yaml parser
- `backend/data-json/AppRegistry.java` — existing app registry with collection + seed loading
- `pages-ui/parser/nav-desugar.ts` — navTree resolution logic
- `pages-viz/form-inputs/schema-types.ts` — field-to-component mapping
- `pages-data/datasource/sources/collection-bindings.ts` — collection data source wiring
