# Script Library & Automation Platform Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #408 — Cross-Platform Scenario Engine
**Issue group:** TBD (issues to be created per batch)

**Goal:** Extend the scenario engine from a demo tool into an automation
platform with a script library, parameterizable execution, data-driven
loops, and script composability.

**Architecture:** Backend changes land in `backend/scenario` (model) and
`backend/scenario-runtime` (CDI beans, REST). The compilation pipeline
consumes `casehub-platform-yaml-core` for shared forEach/when/CSV/variable
resolution. Frontend changes land in `packages/pages-aria` (library
browser in controller component).

**Tech Stack:** Java 21 (records, sealed interfaces), Quarkus (CDI, REST),
Jackson (YAML/JSON), `casehub-platform-yaml-core`, Lit (TypeScript web
components), Vitest (TS tests).

## Global Constraints

- `casehub-platform-yaml-core` is the shared YAML primitives module —
  do NOT reimplement VariableResolver, ForEachExpander, CsvParser, or
  Truthiness
- All new Java code must be J2CL-compatible where it lives in the
  `scenario` model module (no reflection, no CDI, no concurrency)
- CDI annotations are allowed in `scenario-runtime` only
- ARIA interaction contract (PP-20260817-a11y01): element targeting
  uses `{role, name}` with optional `within` scoping
- All commits reference an issue: `Refs #N` or `Closes #N`
- Pre-release platform — breaking changes are free

---

## Batch 1: Script Registry Foundation

After this batch: scripts can be loaded from classpath and filesystem,
listed via REST, uploaded, and deleted. The data model is established.

### Task 1: ScriptDescriptor model + ScriptRegistry service

**Files:**
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScriptDescriptor.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/ParamDescriptor.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScriptProvenance.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScriptMeta.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScriptRegistry.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/BundledScriptSource.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/UploadedScriptSource.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/ScriptDescriptorTest.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScriptRegistryTest.java`

**Interfaces:**
- Produces: `ScriptDescriptor(name, description, labels, tags, params, calls, provenance, firstStepTargets)`, `ParamDescriptor(name, type, required, defaultValue, enumValues)`, `ScriptProvenance.BUNDLED|UPLOADED|EXTERNAL`, `ScriptMeta(description, labels, tags)`
- Produces: `ScriptRegistry.list(labels, tags) → List<ScriptDescriptor>`, `ScriptRegistry.get(name) → Optional<ScriptDescriptor>`, `ScriptRegistry.getYaml(name) → Optional<String>`, `ScriptRegistry.upload(yaml) → ScriptDescriptor`, `ScriptRegistry.updateMeta(name, meta) → ScriptDescriptor`, `ScriptRegistry.delete(name) → boolean`

- [ ] **Step 1: Write model record tests**

```java
// ScriptDescriptorTest.java
@Test void descriptor_exposesMeta() {
    var desc = new ScriptDescriptor("onboard", "Onboard team", List.of("domain:hr"),
            List.of("setup"), List.of(), List.of(), ScriptProvenance.BUNDLED, List.of());
    assertThat(desc.name()).isEqualTo("onboard");
    assertThat(desc.labels()).containsExactly("domain:hr");
    assertThat(desc.provenance()).isEqualTo(ScriptProvenance.BUNDLED);
}

@Test void paramDescriptor_holdsSchema() {
    var param = new ParamDescriptor("name", "string", true, null, List.of());
    assertThat(param.required()).isTrue();
    assertThat(param.type()).isEqualTo("string");
}
```

- [ ] **Step 2: Run tests — verify they fail (classes don't exist)**

Run: `mvn -f backend/scenario/pom.xml test -pl . -Dtest=ScriptDescriptorTest`
Expected: compilation failure

- [ ] **Step 3: Implement model records**

```java
// ScriptDescriptor.java
package io.casehub.pages.scenario;
public record ScriptDescriptor(
    String name, String description,
    List<String> labels, List<String> tags,
    List<ParamDescriptor> params, List<String> calls,
    ScriptProvenance provenance,
    List<AriaTarget> firstStepTargets) {}

// ParamDescriptor.java
public record ParamDescriptor(
    String name, String type, boolean required,
    Object defaultValue, List<Object> enumValues) {}

// ScriptProvenance.java
public enum ScriptProvenance { BUNDLED, UPLOADED, EXTERNAL }

// ScriptMeta.java
public record ScriptMeta(String description, List<String> labels, List<String> tags) {}
```

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Write ScriptRegistry tests**

```java
// ScriptRegistryTest.java
@Test void list_returnsBundledScripts() {
    var registry = new ScriptRegistry(bundledSource, uploadedSource);
    var scripts = registry.list(List.of(), List.of());
    assertThat(scripts).extracting("name").contains("helpdesk-intake");
}

@Test void upload_savesAndReturns() {
    var registry = new ScriptRegistry(bundledSource, uploadedSource);
    var desc = registry.upload(SAMPLE_YAML);
    assertThat(desc.name()).isEqualTo("sample-script");
    assertThat(desc.provenance()).isEqualTo(ScriptProvenance.UPLOADED);
}

@Test void upload_rejectsBundledNameCollision() {
    var registry = new ScriptRegistry(bundledSource, uploadedSource);
    assertThatThrownBy(() -> registry.upload(BUNDLED_NAME_YAML))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("bundled");
}

@Test void delete_removesUploadedOnly() {
    var registry = new ScriptRegistry(bundledSource, uploadedSource);
    registry.upload(SAMPLE_YAML);
    assertThat(registry.delete("sample-script")).isTrue();
    assertThat(registry.delete("helpdesk-intake")).isFalse(); // bundled
}

@Test void list_filtersByLabels() {
    var registry = new ScriptRegistry(bundledSource, uploadedSource);
    var filtered = registry.list(List.of("domain:hr"), List.of());
    assertThat(filtered).allMatch(d -> d.labels().contains("domain:hr"));
}
```

- [ ] **Step 6: Implement ScriptRegistry, BundledScriptSource, UploadedScriptSource**

`ScriptRegistry` is a `@ApplicationScoped` CDI bean. `BundledScriptSource`
scans `META-INF/scenarios/*.yaml` on the classpath at startup.
`UploadedScriptSource` reads/writes to a configurable filesystem path
(`casehub.scenario.library.path`).

Both sources parse each YAML to extract `scenario` name, `meta` block
(if present), and `params` block (if present) into `ScriptDescriptor`
records. Full YAML content is returned on demand via `getYaml(name)`.

The `firstStepTargets` field is extracted by finding the first step's
first command with an ARIA `target` field.

- [ ] **Step 7: Run all tests — verify pass**

- [ ] **Step 8: Commit**

```bash
git add backend/scenario/src backend/scenario-runtime/src
git commit -m "feat(#408): ScriptDescriptor model + ScriptRegistry with bundled/uploaded sources"
```

### Task 2: ScenarioLibraryResource — REST endpoints

**Files:**
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioLibraryResource.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioLibraryResourceTest.java`

**Interfaces:**
- Consumes: `ScriptRegistry.list()`, `.get()`, `.getYaml()`, `.upload()`, `.updateMeta()`, `.delete()`
- Produces: REST endpoints at `/scenario/library/*`

- [ ] **Step 1: Write REST endpoint tests**

```java
@QuarkusTest
class ScenarioLibraryResourceTest {
    @Test void get_library_returnsAllScripts() {
        given().when().get("/scenario/library")
                .then().statusCode(200)
                .body("$.size()", greaterThan(0));
    }

    @Test void get_library_filtersLabels() {
        given().queryParam("labels", "domain:hr")
                .when().get("/scenario/library")
                .then().statusCode(200);
    }

    @Test void get_scriptYaml_returnsContent() {
        given().when().get("/scenario/library/helpdesk-intake/yaml")
                .then().statusCode(200)
                .contentType("text/yaml");
    }

    @Test void post_upload_createsScript() {
        given().contentType("text/yaml").body(SAMPLE_YAML)
                .when().post("/scenario/library")
                .then().statusCode(201)
                .body("name", equalTo("sample-script"));
    }

    @Test void delete_uploaded_succeeds() {
        given().contentType("text/yaml").body(SAMPLE_YAML)
                .when().post("/scenario/library");
        given().when().delete("/scenario/library/sample-script")
                .then().statusCode(204);
    }

    @Test void delete_bundled_rejected() {
        given().when().delete("/scenario/library/helpdesk-intake")
                .then().statusCode(403);
    }
}
```

- [ ] **Step 2: Run tests — verify fail**

- [ ] **Step 3: Implement ScenarioLibraryResource**

```java
@Path("/scenario/library")
@ApplicationScoped
public class ScenarioLibraryResource {
    @Inject ScriptRegistry registry;

    @GET public List<ScriptDescriptor> list(
            @QueryParam("labels") List<String> labels,
            @QueryParam("tags") List<String> tags) {
        return registry.list(labels, tags);
    }

    @GET @Path("/{name}") public Response get(@PathParam("name") String name) { ... }
    @GET @Path("/{name}/yaml") @Produces("text/yaml")
    public Response getYaml(@PathParam("name") String name) { ... }

    @POST @Consumes("text/yaml")
    public Response upload(String yaml) { ... }

    @PUT @Path("/{name}/meta")
    public Response updateMeta(@PathParam("name") String name, ScriptMeta meta) { ... }

    @DELETE @Path("/{name}")
    public Response delete(@PathParam("name") String name) { ... }
}
```

- [ ] **Step 4: Run tests — verify pass**
- [ ] **Step 5: Commit**

---

## Batch 2: Compilation Pipeline

After this batch: scenario YAML supports `params`, `data` (CSV), `iterations`,
`forEach`, and `when`. The ScenarioCompiler resolves variables, expands
loops, and evaluates conditionals using casehub-platform-yaml-core.

### Task 3: Add yaml-core dependency + ScenarioCompiler

**Files:**
- Modify: `backend/scenario/pom.xml` — add `casehub-platform-yaml-core` dependency
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScenarioCompiler.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScenarioStepAdapter.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/CompiledScenario.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/ScenarioCompilerTest.java`
- Test fixture: `backend/scenario/src/test/resources/scenarios/parameterized-onboard.yaml`
- Test fixture: `backend/scenario/src/test/resources/scenarios/foreach-csv-inline.yaml`

**Interfaces:**
- Consumes: `io.casehub.yaml.core.resolver.VariableResolver`, `io.casehub.yaml.core.resolver.VariableSource`, `io.casehub.yaml.core.foreach.ForEachExpander`, `io.casehub.yaml.core.foreach.ForEachAdapter`, `io.casehub.yaml.core.data.CsvParser`, `io.casehub.yaml.core.condition.Truthiness`
- Produces: `ScenarioCompiler.compile(yaml, callerParams) → CompiledScenario`, `CompiledScenario(steps, callRefs)`, `ScenarioStepAdapter implements ForEachAdapter<HierarchicalStep>`

- [ ] **Step 1: Add yaml-core dependency to pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-core</artifactId>
</dependency>
```

- [ ] **Step 2: Write test fixtures**

`parameterized-onboard.yaml`:
```yaml
scenario: parameterized-onboard
params:
  - name: projectName
    type: string
    required: true
  - name: enableCI
    type: boolean
    default: true
steps:
  - label: "Create project"
    target: browser
    commands:
      - action: fill
        target: {role: textbox, name: "Project Name"}
        value: "${params.projectName}"
  - label: "Enable CI"
    target: browser
    when: "${params.enableCI}"
    commands:
      - action: click
        target: {role: button, name: "Enable CI"}
```

`foreach-csv-inline.yaml`:
```yaml
scenario: foreach-csv-inline
data:
  members:
    inline: |
      name:string,role:string,admin:boolean
      Alice,Developer,true
      Bob,Viewer,false
steps:
  - label: "Create member"
    target: browser
    forEach:
      as: member
      in: members
    commands:
      - action: fill
        target: {role: textbox, name: "Full Name"}
        value: "${each.member.name}"
  - label: "Grant admin"
    target: browser
    forEach:
      as: member
      in: members
    when: "${each.member.admin}"
    commands:
      - action: click
        target: {role: button, name: "Grant Admin"}
```

- [ ] **Step 3: Write ScenarioCompiler tests**

```java
@Test void compile_resolvesParams() {
    var compiled = ScenarioCompiler.compile(
            fixture("parameterized-onboard.yaml"),
            Map.of("projectName", "Acme"));
    var firstStep = compiled.steps().get(0);
    assertThat(firstStep.commands().get(0).value()).isEqualTo("Acme");
}

@Test void compile_missingRequiredParam_throws() {
    assertThatThrownBy(() -> ScenarioCompiler.compile(
            fixture("parameterized-onboard.yaml"), Map.of()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("projectName");
}

@Test void compile_whenFalse_excludesStep() {
    var compiled = ScenarioCompiler.compile(
            fixture("parameterized-onboard.yaml"),
            Map.of("projectName", "Acme", "enableCI", "false"));
    assertThat(compiled.steps()).hasSize(1);
    assertThat(compiled.steps().get(0).label()).isEqualTo("Create project");
}

@Test void compile_forEachCsv_stampsPerRow() {
    var compiled = ScenarioCompiler.compile(
            fixture("foreach-csv-inline.yaml"), Map.of());
    // 2 rows × 2 forEach steps, minus 1 excluded by when (Bob is not admin)
    // = create-member.Alice, create-member.Bob, grant-admin.Alice
    assertThat(compiled.steps()).hasSize(3);
}

@Test void compile_forEachCsv_resolvesColumnValues() {
    var compiled = ScenarioCompiler.compile(
            fixture("foreach-csv-inline.yaml"), Map.of());
    var aliceStep = compiled.steps().get(0);
    assertThat(aliceStep.commands().get(0).value()).isEqualTo("Alice");
}
```

- [ ] **Step 4: Run tests — verify fail**

- [ ] **Step 5: Implement ScenarioStepAdapter and ScenarioCompiler**

`ScenarioStepAdapter` implements `ForEachAdapter<HierarchicalStep>`:
- `stamp()` — clone step with stamped ID and resolved command values
- `getForEach()` — return the `forEach` field from the parsed step
- `getId()` — return the step name/label slug
- `getWhen()` — return the `when` field

`ScenarioCompiler.compile(yaml, callerParams)`:
1. Parse YAML via `HierarchicalParser.parse(yaml)` (extended to read
   `params`, `data`, `iterations`, `meta` blocks)
2. Validate required params are supplied
3. Wire `VariableResolver` with `VariableSource.chain(callerParams, defaults, config)`
4. Load CSV data sources via `CsvParser.parse()`
5. Convert steps to `LinkedHashMap` for `ForEachExpander`
6. Call `ForEachExpander.expand()` with `ScenarioStepAdapter`
7. Convert `ExpansionResult` back to ordered step list
8. Extract `callRefs` (scripts referenced by `call` commands)
9. Return `CompiledScenario(steps, callRefs)`

- [ ] **Step 6: Extend HierarchicalParser to parse new YAML blocks**

Extend the Jackson ObjectMapper deserialization to read `params`,
`data`, `iterations`, `meta`, `forEach`, and `when` fields from the
scenario YAML. Add these as fields on `HierarchicalScenario`.

- [ ] **Step 7: Run tests — verify pass**
- [ ] **Step 8: Commit**

---

## Batch 3: Script Composability

After this batch: scripts can `call` other scripts. Acyclic enforcement
rejects circular call graphs at load time. Callee steps are inlined
with name prefixing and context inheritance.

### Task 4: CallGraphValidator + call command resolution

**Files:**
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/CallGraphValidator.java`
- Modify: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScenarioCompiler.java` — add call resolution (steps 7-9)
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/CallGraphValidatorTest.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/ScenarioCompilerCallTest.java`
- Test fixture: `backend/scenario/src/test/resources/scenarios/caller-script.yaml`
- Test fixture: `backend/scenario/src/test/resources/scenarios/callee-create-user.yaml`
- Test fixture: `backend/scenario/src/test/resources/scenarios/cyclic-a.yaml`
- Test fixture: `backend/scenario/src/test/resources/scenarios/cyclic-b.yaml`

**Interfaces:**
- Consumes: `ScriptRegistry.getYaml(name)` for transitive resolution
- Produces: `CallGraphValidator.validate(rootScript, scriptResolver) → void (throws on cycle)`, updated `ScenarioCompiler.compile()` with inlined callee steps

- [ ] **Step 1: Write CallGraphValidator tests**

```java
@Test void validate_noCalls_passes() {
    CallGraphValidator.validate("root",
            name -> Optional.of(new CallGraphValidator.ScriptRef(name, List.of())));
}

@Test void validate_linearChain_passes() {
    // root → A → B (no cycle)
    CallGraphValidator.validate("root", name -> switch (name) {
        case "root" -> Optional.of(new ScriptRef("root", List.of("A")));
        case "A" -> Optional.of(new ScriptRef("A", List.of("B")));
        case "B" -> Optional.of(new ScriptRef("B", List.of()));
        default -> Optional.empty();
    });
}

@Test void validate_cycle_throws() {
    // root → A → B → root (cycle)
    assertThatThrownBy(() -> CallGraphValidator.validate("root", name -> switch (name) {
        case "root" -> Optional.of(new ScriptRef("root", List.of("A")));
        case "A" -> Optional.of(new ScriptRef("A", List.of("B")));
        case "B" -> Optional.of(new ScriptRef("B", List.of("root")));
        default -> Optional.empty();
    })).isInstanceOf(IllegalArgumentException.class)
       .hasMessageContaining("root → A → B → root");
}
```

- [ ] **Step 2: Implement CallGraphValidator**

DFS traversal with path tracking. `ScriptRef(name, calls)` extracted
from compiled scenario's `callRefs`. Throws with the full cycle path
on detection.

- [ ] **Step 3: Write call command compilation tests**

```java
@Test void compile_callInlinesCalleeSteps() {
    var compiled = compileWithRegistry("caller-script.yaml");
    assertThat(compiled.steps()).extracting("label")
            .contains("callee-create-user.Fill name");
}

@Test void compile_callPrefixesStepNames() {
    var compiled = compileWithRegistry("caller-script.yaml");
    var calleeStep = compiled.steps().stream()
            .filter(s -> s.label().startsWith("callee-create-user."))
            .findFirst().orElseThrow();
    assertThat(calleeStep.name()).startsWith("callee-create-user.");
}

@Test void compile_callPassesParams() {
    var compiled = compileWithRegistry("caller-script.yaml");
    var calleeStep = compiled.steps().stream()
            .filter(s -> s.label().contains("Fill name"))
            .findFirst().orElseThrow();
    assertThat(calleeStep.commands().get(0).value()).isEqualTo("Alice");
}
```

- [ ] **Step 4: Extend ScenarioCompiler with call resolution**

After forEach/when expansion (step 6 in the pipeline), walk the
expanded steps looking for `action: call` commands. For each:
1. Resolve the script from the registry
2. Compile the callee with merged params (caller context + explicit)
3. Validate acyclicity via `CallGraphValidator`
4. Replace the `call` command's parent step with the callee's inlined
   steps, prefixed with `callee-name.`

- [ ] **Step 5: Run all tests — verify pass**
- [ ] **Step 6: Commit**

---

## Batch 4: Library Browser UI

After this batch: the scenario controller has a library view with search,
label/tag filtering, readiness probes, and paste/upload.

### Task 5: Library view in scenario controller

**Files:**
- Create: `packages/pages-aria/src/controller/library-view.ts`
- Create: `packages/pages-aria/src/controller/readiness-probe.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts` — add library mode toggle
- Test: `packages/pages-aria/src/controller/library-view.test.ts`
- Test: `packages/pages-aria/src/controller/readiness-probe.test.ts`

**Interfaces:**
- Consumes: `GET /scenario/library` REST endpoint, `findByRole()` from `../walker/tree-walker.ts`
- Produces: `LibraryView` Lit component (internal to controller), `probeReadiness(targets: AriaTarget[]) → 'ready' | 'unknown' | 'not-ready'`

- [ ] **Step 1: Write readiness probe tests**

```typescript
describe('probeReadiness', () => {
    it('returns ready when all targets found', () => {
        document.body.innerHTML = '<button>Submit</button>';
        const result = probeReadiness([{role: 'button', name: 'Submit'}]);
        expect(result).toBe('ready');
    });

    it('returns not-ready when targets missing', () => {
        document.body.innerHTML = '<div></div>';
        const result = probeReadiness([{role: 'button', name: 'Submit'}]);
        expect(result).toBe('not-ready');
    });

    it('returns unknown when no targets', () => {
        const result = probeReadiness([]);
        expect(result).toBe('unknown');
    });
});
```

- [ ] **Step 2: Implement readiness probe**

```typescript
export function probeReadiness(targets: AriaTarget[]): 'ready' | 'unknown' | 'not-ready' {
    if (targets.length === 0) return 'unknown';
    for (const target of targets) {
        const found = findByRole(document.body, target.role, target.name, target.within);
        if (!found) return 'not-ready';
    }
    return 'ready';
}
```

- [ ] **Step 3: Write library view tests**

```typescript
describe('LibraryView', () => {
    it('fetches and renders script list', async () => { ... });
    it('filters by label', async () => { ... });
    it('shows readiness indicator per script', async () => { ... });
    it('emits script-selected event on run click', async () => { ... });
});
```

- [ ] **Step 4: Implement LibraryView Lit component**

Internal component rendered by `scenario-controller` when in library
mode. Fetches `GET /scenario/library` via `ScenarioConnectionController`.
Renders filterable list with search input, label/tag chips, and
per-script readiness indicators. "Run" button emits `script-selected`
custom event. Paste/upload button opens a text area for YAML paste or
file picker, POSTs to `/scenario/library`.

- [ ] **Step 5: Wire library toggle into scenario-controller**

Add a library icon button to the controller header. Clicking toggles
between `this._view = 'outline'` and `this._view = 'library'`. When
library view is active, render `LibraryView`. On `script-selected`
event, load the script YAML and switch back to outline view.

- [ ] **Step 6: Run all tests — verify pass**
- [ ] **Step 7: Commit**

---

## Batch 5: External Registries

After this batch: the server can aggregate scripts from configured
external registry URLs (JSON manifests) alongside bundled and uploaded
sources.

### Task 6: External registry source + multi-source aggregation

**Files:**
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ExternalRegistrySource.java`
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScriptRegistry.java` — add external sources
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ExternalRegistrySourceTest.java`

**Interfaces:**
- Consumes: JSON manifest from configured URLs, `ScenarioConfig` for registry URL list
- Produces: `ExternalRegistrySource.fetch() → List<ScriptDescriptor>`, `ExternalRegistrySource.getYaml(name) → String` (fetches from `contentUrl`)

- [ ] **Step 1: Write external registry tests**

```java
@Test void fetch_parsesJsonManifest() {
    var source = new ExternalRegistrySource("test-registry",
            URI.create(mockServer.url("/manifest.json")), httpClient);
    mockServer.enqueue(new MockResponse().setBody(MANIFEST_JSON));
    var scripts = source.fetch();
    assertThat(scripts).hasSize(2);
    assertThat(scripts.get(0).provenance()).isEqualTo(ScriptProvenance.EXTERNAL);
}

@Test void fetch_cachesWithTtl() {
    // Second call within TTL returns cached
    source.fetch(); source.fetch();
    assertThat(mockServer.getRequestCount()).isEqualTo(1);
}

@Test void getYaml_fetchesFromContentUrl() {
    mockServer.enqueue(new MockResponse().setBody(SAMPLE_YAML));
    var yaml = source.getYaml("onboard-team");
    assertThat(yaml).contains("scenario: onboard-team");
}
```

- [ ] **Step 2: Implement ExternalRegistrySource**

HTTP client fetches the manifest JSON, deserializes to
`List<ScriptDescriptor>`. Caches with configurable TTL
(`casehub.scenario.registry.cache-ttl`, default 5 minutes).
`getYaml()` resolves `contentUrl` relative to the manifest URL
and fetches on demand.

Configuration:
```properties
casehub.scenario.registries[0].url=https://scripts.example.com/manifest.json
casehub.scenario.registries[0].name=company-scripts
```

- [ ] **Step 3: Wire external sources into ScriptRegistry**

`ScriptRegistry` accepts `List<ExternalRegistrySource>` injected from
config. `list()` aggregates all three source types. External scripts
are read-only — `upload()`, `updateMeta()`, and `delete()` reject
external names.

- [ ] **Step 4: Run all tests — verify pass**
- [ ] **Step 5: Commit**

---

## References

- [2026-08-29-script-library-automation-platform-design.md] — design spec this plan implements
- [ScenarioParser.java] — existing parser to extend
- [HierarchicalParser.java] — hierarchical YAML parser
- [ScenarioControlResource.java] — existing REST controller (sibling for new library resource)
- [ScenarioOrchestrator.java] — orchestrator wiring point for compiled scenarios
- [scenario-handler.ts] — browser executor (unchanged by this plan)
- [scenario-controller.ts] — controller UI to extend with library view
- [casehub-platform-yaml-core] — shared YAML primitives module
- [PP-20260817-a11y01] — ARIA interaction contract
- [GitHub #408] — scenario engine epic
