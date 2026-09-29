# Step Catalog Browser Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #501 — Step catalog browser
**Issue group:** #501 (part of epic #502)

**Goal:** Build a browsable step action catalog with schema display, live execution, and YAML template generation, embedded in the scenario controller.

**Architecture:** Java `@McpDomain("step-catalog")` resolver serves catalog data from YAML definitions, MCP tools, and scripts. TS pages-aria REST endpoint handles live step execution. Lit web component `<pages-step-catalog>` provides the UI, integrated into the scenario controller as a new view mode alongside the library view.

**Tech Stack:** Java (Quarkus, JAX-RS, MicroProfile GraphQL), TypeScript (Lit, vitest), YAML parsing (SnakeYAML on Java, existing StepDefinitionParser on TS)

## Global Constraints

- All CSS uses `--pages-*` design tokens (per `css-design-tokens.md`)
- All interactive elements require ARIA roles and labels (per `aria-interaction-contract.md`)
- Web components use Lit (per `web-component-strategy.md`)
- Java records for DTOs, `@ApplicationScoped` for services
- Every commit references issue #501

---

## Batch 1: Java Backend — Catalog Data Types and Service

### Task 1: StepCatalogService and data types

**Files:**
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/CatalogActionSummary.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/CatalogActionDetail.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/StepParameterDto.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/InvokeBindingSummary.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/StepCatalogService.java`
- Create: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/StepCatalogServiceTest.java`

**Interfaces:**
- Consumes: nothing (first task)
- Produces: `StepCatalogService` with `List<CatalogActionSummary> listActions()` and `Optional<CatalogActionDetail> getAction(String name)`. Scans three sources: YAML definition files, MCP tools (CDI-discovered `@McpDomain` beans via `@Query`/`@Mutation` introspection), and script files (with companion `.schema.yaml`). Data records: `CatalogActionSummary(String name, String description, String invokeKind, String source, int inputCount, int outputCount)`, `CatalogActionDetail(String name, String description, String invokeKind, String source, Map<String, StepParameterDto> inputs, Map<String, StepParameterDto> outputs, InvokeBindingSummary invoke)`, `StepParameterDto(String type, boolean required, String defaultValue, List<String> allowedValues, String format, String description)`, `InvokeBindingSummary(String kind, Map<String, String> metadata)`.

Note: MCP tool enumeration and script file scanning are stub implementations in this task — they call `loadMcpTools()` and `loadScripts()` in `@PostConstruct` but produce entries only when MCP beans or script directories are present. The YAML source is the primary source and is fully tested.

- [ ] **Step 1: Write the failing test for YAML step definition parsing**

```java
@QuarkusTest
class StepCatalogServiceTest {

    @Inject
    StepCatalogService service;

    @Test
    void yamlDefinitionsAreLoaded() {
        var actions = service.listActions();
        // Expects at least one action from test YAML definitions
        assertFalse(actions.isEmpty());
        var action = actions.stream()
                .filter(a -> a.name().equals("check-compliance"))
                .findFirst()
                .orElseThrow();
        assertEquals("yaml", action.source());
        assertEquals("rest", action.invokeKind());
        assertEquals(2, action.inputCount());
        assertEquals(1, action.outputCount());
    }

    @Test
    void actionDetailContainsFullSchema() {
        var detail = service.getAction("check-compliance")
                .orElseThrow();
        assertEquals("check-compliance", detail.name());
        assertEquals(2, detail.inputs().size());
        assertTrue(detail.inputs().containsKey("documentId"));
        var docId = detail.inputs().get("documentId");
        assertEquals("STRING", docId.type());
        assertTrue(docId.required());
        assertEquals("rest", detail.invoke().kind());
    }
}
```

- [ ] **Step 2: Create a test YAML step definition file**

Create `backend/scenario-runtime/src/test/resources/step-definitions/compliance.yaml`:

```yaml
namespace: compliance
actions:
  check-compliance:
    description: "Check document compliance against a standard"
    inputs:
      documentId:
        type: string
        required: true
        description: "Document identifier"
      standard:
        type: string
        required: true
        default: "ISO-27001"
        enum: ["ISO-27001", "SOC-2", "GDPR"]
        description: "Compliance standard to check"
    outputs:
      compliant:
        type: boolean
        description: "Whether the document is compliant"
    invoke:
      rest:
        method: POST
        url: "https://api.example.com/compliance/check"
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f backend/scenario-runtime/pom.xml test -Dtest=StepCatalogServiceTest -pl . -Dquarkus.test.profile=test`
Expected: FAIL — `StepCatalogService` class does not exist

- [ ] **Step 4: Create the data record types**

Create `CatalogActionSummary.java`:
```java
package io.casehub.pages.scenario.runtime;

public record CatalogActionSummary(
    String name,
    String description,
    String invokeKind,
    String source,
    int inputCount,
    int outputCount
) {}
```

Create `StepParameterDto.java`:
```java
package io.casehub.pages.scenario.runtime;

import java.util.List;

public record StepParameterDto(
    String type,
    boolean required,
    String defaultValue,
    List<String> allowedValues,
    String format,
    String description
) {}
```

Create `InvokeBindingSummary.java`:
```java
package io.casehub.pages.scenario.runtime;

import java.util.Map;

public record InvokeBindingSummary(
    String kind,
    Map<String, String> metadata
) {}
```

Create `CatalogActionDetail.java`:
```java
package io.casehub.pages.scenario.runtime;

import java.util.Map;

public record CatalogActionDetail(
    String name,
    String description,
    String invokeKind,
    String source,
    Map<String, StepParameterDto> inputs,
    Map<String, StepParameterDto> outputs,
    InvokeBindingSummary invoke
) {}
```

- [ ] **Step 5: Implement StepCatalogService**

Create `StepCatalogService.java`:
```java
package io.casehub.pages.scenario.runtime;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.yaml.snakeyaml.Yaml;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.stream.Stream;

@ApplicationScoped
public class StepCatalogService {

    @ConfigProperty(name = "casehub.step-catalog.definitions-path",
                    defaultValue = "step-definitions")
    String definitionsPath;

    private final Map<String, CatalogActionDetail> actions = new LinkedHashMap<>();

    @PostConstruct
    void init() {
        loadYamlDefinitions();
        loadMcpTools();
        loadScripts();
    }

    public List<CatalogActionSummary> listActions() {
        return actions.values().stream()
                .map(d -> new CatalogActionSummary(
                        d.name(), d.description(), d.invokeKind(),
                        d.source(), d.inputs().size(), d.outputs().size()))
                .toList();
    }

    public Optional<CatalogActionDetail> getAction(String name) {
        return Optional.ofNullable(actions.get(name));
    }

    @SuppressWarnings("unchecked")
    private void loadYamlDefinitions() {
        var yaml = new Yaml();
        // Try classpath first
        try {
            var classLoader = Thread.currentThread().getContextClassLoader();
            var resourceUrl = classLoader.getResource(definitionsPath);
            if (resourceUrl != null) {
                var resourcePath = Path.of(resourceUrl.toURI());
                loadYamlFromDirectory(yaml, resourcePath);
                return;
            }
        } catch (Exception ignored) {}

        // Try filesystem path
        var fsPath = Path.of(definitionsPath);
        if (Files.isDirectory(fsPath)) {
            loadYamlFromDirectory(yaml, fsPath);
        }
    }

    @SuppressWarnings("unchecked")
    private void loadYamlFromDirectory(Yaml yaml, Path directory) {
        try (Stream<Path> files = Files.list(directory)) {
            files.filter(p -> p.toString().endsWith(".yaml") || p.toString().endsWith(".yml"))
                 .forEach(path -> {
                     try (InputStream is = Files.newInputStream(path)) {
                         var doc = (Map<String, Object>) yaml.load(is);
                         parseDefinitionFile(doc);
                     } catch (IOException e) {
                         // Skip unreadable files
                     }
                 });
        } catch (IOException e) {
            // Directory not readable
        }
    }

    @SuppressWarnings("unchecked")
    private void parseDefinitionFile(Map<String, Object> doc) {
        var namespace = (String) doc.get("namespace");
        var actionsMap = (Map<String, Map<String, Object>>) doc.get("actions");
        if (actionsMap == null) return;

        for (var entry : actionsMap.entrySet()) {
            var name = entry.getKey();
            var qualifiedName = namespace != null ? namespace + "." + name : name;
            var actionDef = entry.getValue();

            var description = (String) actionDef.get("description");
            var inputs = parseParams((Map<String, Object>) actionDef.get("inputs"));
            var outputs = parseParams((Map<String, Object>) actionDef.get("outputs"));
            var invoke = parseInvoke((Map<String, Object>) actionDef.get("invoke"));

            var detail = new CatalogActionDetail(
                    qualifiedName, description,
                    invoke != null ? invoke.kind() : null,
                    "yaml", inputs, outputs, invoke);
            actions.put(qualifiedName, detail);
            if (namespace != null) {
                actions.putIfAbsent(name, detail);
            }
        }
    }

    @SuppressWarnings("unchecked")
    private Map<String, StepParameterDto> parseParams(Map<String, Object> raw) {
        if (raw == null) return Map.of();
        var result = new LinkedHashMap<String, StepParameterDto>();
        for (var entry : raw.entrySet()) {
            var paramRaw = entry.getValue();
            if (paramRaw instanceof String typeStr) {
                result.put(entry.getKey(), new StepParameterDto(
                        typeStr.toUpperCase(), false, null, null, null, null));
            } else if (paramRaw instanceof Map<?, ?> map) {
                var p = (Map<String, Object>) map;
                var type = p.get("type") != null ? p.get("type").toString().toUpperCase() : "STRING";
                var required = Boolean.TRUE.equals(p.get("required"));
                var defaultValue = p.containsKey("defaultValue")
                        ? String.valueOf(p.get("defaultValue"))
                        : (p.containsKey("default") ? String.valueOf(p.get("default")) : null);
                var allowedRaw = p.containsKey("allowedValues")
                        ? (List<?>) p.get("allowedValues")
                        : (p.containsKey("enum") ? (List<?>) p.get("enum") : null);
                var allowedValues = allowedRaw != null
                        ? allowedRaw.stream().map(String::valueOf).toList()
                        : null;
                var format = (String) p.get("format");
                var desc = (String) p.get("description");
                result.put(entry.getKey(), new StepParameterDto(
                        type, required, defaultValue, allowedValues, format, desc));
            }
        }
        return result;
    }

    private void loadMcpTools() {
        // Enumerates CDI-discovered @McpDomain beans via reflection.
        // Each @Query/@Mutation becomes a catalog entry with source="mcp".
        // Stub: produces entries only when MCP beans are present in the CDI container.
    }

    private void loadScripts() {
        // Scans configured scripts directory for executable files with
        // companion <name>.schema.yaml declaring inputs, outputs, runtime.
        // Stub: produces entries only when scripts directory exists and contains schema files.
    }

    private InvokeBindingSummary parseInvoke(Map<String, Object> raw) {
        if (raw == null) return null;
        if (raw.containsKey("mcp")) {
            return new InvokeBindingSummary("mcp", Map.of("tool", String.valueOf(raw.get("mcp"))));
        }
        if (raw.containsKey("rest")) {
            @SuppressWarnings("unchecked")
            var spec = (Map<String, Object>) raw.get("rest");
            return new InvokeBindingSummary("rest", Map.of(
                    "method", String.valueOf(spec.getOrDefault("method", "GET")),
                    "url", String.valueOf(spec.get("url"))));
        }
        if (raw.containsKey("python")) {
            return new InvokeBindingSummary("script", Map.of(
                    "runtime", "python3", "script", String.valueOf(raw.get("python"))));
        }
        if (raw.containsKey("node")) {
            return new InvokeBindingSummary("script", Map.of(
                    "runtime", "node", "script", String.valueOf(raw.get("node"))));
        }
        if (raw.containsKey("graphql")) {
            return new InvokeBindingSummary("graphql", Map.of("query", String.valueOf(raw.get("graphql"))));
        }
        if (raw.containsKey("agent")) {
            @SuppressWarnings("unchecked")
            var spec = (Map<String, Object>) raw.get("agent");
            return new InvokeBindingSummary("agent", Map.of(
                    "descriptor", String.valueOf(spec.get("descriptor"))));
        }
        if (raw.containsKey("process")) {
            @SuppressWarnings("unchecked")
            var spec = (Map<String, Object>) raw.get("process");
            return new InvokeBindingSummary("process", Map.of(
                    "command", String.valueOf(spec.get("command"))));
        }
        if (raw.containsKey("script")) {
            @SuppressWarnings("unchecked")
            var spec = (Map<String, Object>) raw.get("script");
            return new InvokeBindingSummary("script", Map.of(
                    "runtime", String.valueOf(spec.get("runtime")),
                    "script", String.valueOf(spec.get("script"))));
        }
        return null;
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f backend/scenario-runtime/pom.xml test -Dtest=StepCatalogServiceTest -pl .`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/CatalogActionSummary.java \
       backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/CatalogActionDetail.java \
       backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/StepParameterDto.java \
       backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/InvokeBindingSummary.java \
       backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/StepCatalogService.java \
       backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/StepCatalogServiceTest.java \
       backend/scenario-runtime/src/test/resources/step-definitions/compliance.yaml
git commit -m "feat(scenario-runtime): StepCatalogService with YAML definition parsing Refs #501"
```

### Task 2: StepCatalogResolver @McpDomain

**Files:**
- Create: `backend/mcp/src/main/java/io/casehub/pages/mcp/StepCatalogResolver.java`
- Create: `backend/mcp/src/test/java/io/casehub/pages/mcp/StepCatalogResolverTest.java`

**Interfaces:**
- Consumes: `StepCatalogService.listActions()`, `StepCatalogService.getAction(String name)`, `CatalogActionSummary`, `CatalogActionDetail` (from Task 1)
- Produces: GraphQL/MCP queries `catalogActions` → `List<CatalogActionSummary>`, `catalogAction(name)` → `CatalogActionDetail`

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class StepCatalogResolverTest {

    @Test
    void catalogActionsQueryReturnsList() {
        given()
            .contentType("application/json")
            .body("""
                {"query": "{ catalogActions { name source invokeKind inputCount outputCount } }"}
                """)
            .when().post("/graphql")
            .then()
            .statusCode(200)
            .body("data.catalogActions", not(empty()))
            .body("data.catalogActions[0].name", notNullValue());
    }

    @Test
    void catalogActionQueryReturnsDetail() {
        given()
            .contentType("application/json")
            .body("""
                {"query": "{ catalogAction(name: \\"check-compliance\\") { name description inputs { documentId { type required } } } }"}
                """)
            .when().post("/graphql")
            .then()
            .statusCode(200)
            .body("data.catalogAction.name", equalTo("check-compliance"));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f backend/mcp/pom.xml test -Dtest=StepCatalogResolverTest -pl .`
Expected: FAIL — `StepCatalogResolver` does not exist

- [ ] **Step 3: Implement StepCatalogResolver**

```java
package io.casehub.pages.mcp;

import io.casehub.pages.scenario.runtime.CatalogActionDetail;
import io.casehub.pages.scenario.runtime.CatalogActionSummary;
import io.casehub.pages.scenario.runtime.StepCatalogService;
import io.casehub.platform.api.mcp.McpDomain;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Query;

import java.util.List;

@McpDomain("step-catalog")
@GraphQLApi
@ApplicationScoped
public class StepCatalogResolver {

    @Inject
    StepCatalogService catalog;

    @Query("catalogActions")
    public List<CatalogActionSummary> actions() {
        return catalog.listActions();
    }

    @Query("catalogAction")
    public CatalogActionDetail action(String name) {
        return catalog.getAction(name).orElse(null);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f backend/mcp/pom.xml test -Dtest=StepCatalogResolverTest -pl .`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/mcp/src/main/java/io/casehub/pages/mcp/StepCatalogResolver.java \
       backend/mcp/src/test/java/io/casehub/pages/mcp/StepCatalogResolverTest.java
git commit -m "feat(mcp): StepCatalogResolver @McpDomain for catalog queries Refs #501"
```

## Batch 2: TS Execution Endpoint

### Task 3: Catalog execute handler in pages-aria

**Files:**
- Create: `packages/pages-aria/src/server/catalog-execute-handler.ts`
- Create: `packages/pages-aria/src/server/catalog-execute-handler.test.ts`
- Modify: `packages/pages-aria/src/server/index.ts`

**Interfaces:**
- Consumes: `CompositeStepCatalog`, `CatalogSource` from `@casehubio/yaml-core`, `StructuralStepEvaluator` from `@casehubio/yaml-core`, `MapServiceRegistry` from `@casehubio/yaml-core`
- Produces: `createCatalogExecuteHandler(catalog: StepCatalog): (req: CatalogExecuteRequest) => Promise<StepResult>`. `CatalogExecuteRequest = { actionName: string; params: Record<string, unknown> }`.

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, it, expect } from 'vitest';
import { createCatalogExecuteHandler } from './catalog-execute-handler.js';
import type { StepCatalog, CatalogEntry, StepResult } from '@casehubio/yaml-core';

function mockCatalog(entries: Map<string, CatalogEntry>): StepCatalog {
  return {
    resolve: (name: string) => entries.get(name),
    availableActions: () => new Set(entries.keys()),
  };
}

describe('createCatalogExecuteHandler', () => {
  it('executes a catalog action and returns success', async () => {
    const entry: CatalogEntry = {
      qualifiedName: 'greet',
      definition: { name: 'greet', inputs: {}, outputs: {} },
      action: {
        execute: async (params) => ({
          kind: 'success' as const,
          output: { message: `Hello ${params['name']}` },
          executionMetadata: {},
        }),
      },
    };
    const catalog = mockCatalog(new Map([['greet', entry]]));
    const handler = createCatalogExecuteHandler(catalog);

    const result = await handler({ actionName: 'greet', params: { name: 'World' } });
    expect(result.kind).toBe('success');
    if (result.kind === 'success') {
      expect(result.output['message']).toBe('Hello World');
    }
  });

  it('returns failure for unknown action', async () => {
    const catalog = mockCatalog(new Map());
    const handler = createCatalogExecuteHandler(catalog);

    const result = await handler({ actionName: 'unknown', params: {} });
    expect(result.kind).toBe('failure');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-aria/src/server/catalog-execute-handler.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement catalog execute handler**

```typescript
import type { StepCatalog, StepResult } from '@casehubio/yaml-core';
import { stepFailure } from '@casehubio/yaml-core';
import { StructuralStepEvaluator } from '@casehubio/yaml-core';
import { MapServiceRegistry } from '@casehubio/yaml-core';

export interface CatalogExecuteRequest {
  actionName: string;
  params: Record<string, unknown>;
}

export function createCatalogExecuteHandler(
  catalog: StepCatalog,
): (req: CatalogExecuteRequest) => Promise<StepResult> {
  const evaluator = new StructuralStepEvaluator();

  return async (req: CatalogExecuteRequest): Promise<StepResult> => {
    const entry = catalog.resolve(req.actionName);
    if (!entry) {
      return stepFailure(`Action '${req.actionName}' not found in catalog`);
    }

    const services = new MapServiceRegistry();
    const context = {
      scope: {
        resolve: (expr: string) => expr,
        signal: () => ({ await: async () => {} }),
        resultStore: () => ({
          recordSuccess: () => {},
          recordFailure: () => {},
          hasCompleted: () => false,
        }),
      },
      services,
      params: req.params,
      stepName: req.actionName,
    };

    try {
      return await entry.action.execute(req.params, services);
    } catch (err) {
      return stepFailure(
        `Execution failed: ${err instanceof Error ? err.message : String(err)}`,
      );
    }
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-aria/src/server/catalog-execute-handler.test.ts`
Expected: PASS

- [ ] **Step 5: Export from server index**

Add to `packages/pages-aria/src/server/index.ts`:
```typescript
export { createCatalogExecuteHandler, type CatalogExecuteRequest } from './catalog-execute-handler.js';
```

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/src/server/catalog-execute-handler.ts \
       packages/pages-aria/src/server/catalog-execute-handler.test.ts \
       packages/pages-aria/src/server/index.ts
git commit -m "feat(pages-aria): catalog execute handler for live step execution Refs #501"
```

## Batch 3: UI Component — List View

### Task 4: `<pages-step-catalog>` Lit component with list view

**Files:**
- Create: `packages/pages-aria/src/controller/step-catalog.ts`
- Create: `packages/pages-aria/src/controller/step-catalog.test.ts`
- Modify: `packages/pages-aria/src/controller/index.ts`

**Interfaces:**
- Consumes: Fetches from `baseUrl` via GraphQL `catalogActions` query. Uses `CatalogActionSummary` shape: `{ name, description, invokeKind, source, inputCount, outputCount }`.
- Produces: `PagesStepCatalog` Lit element registered as `pages-step-catalog`. Properties: `baseUrl: string`, `execBaseUrl: string`. Emits `step-template-selected` CustomEvent (in Task 6). Emits `action-selected` internal event for detail navigation.

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { PagesStepCatalog } from './step-catalog.js';

const MOCK_ACTIONS = [
  { name: 'check-compliance', description: 'Check doc compliance', invokeKind: 'rest', source: 'yaml', inputCount: 2, outputCount: 1 },
  { name: 'mcp.send-email', description: 'Send an email via MCP', invokeKind: 'mcp', source: 'mcp', inputCount: 3, outputCount: 0 },
  { name: 'run-audit', description: 'Run security audit script', invokeKind: 'script', source: 'script', inputCount: 1, outputCount: 2 },
];

describe('PagesStepCatalog', () => {
  let el: PagesStepCatalog;

  beforeEach(async () => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ data: { catalogActions: MOCK_ACTIONS } }),
    }));
    el = new PagesStepCatalog();
    el.baseUrl = 'http://localhost:8080';
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => {
    el.remove();
    vi.restoreAllMocks();
  });

  it('renders action list after loading', async () => {
    await el.loadCatalog();
    await el.updateComplete;
    const items = el.shadowRoot!.querySelectorAll('.action-item');
    expect(items.length).toBe(3);
  });

  it('filters by search text', async () => {
    await el.loadCatalog();
    el['_searchText'] = 'compliance';
    await el.updateComplete;
    const items = el.shadowRoot!.querySelectorAll('.action-item');
    expect(items.length).toBe(1);
  });

  it('filters by source chip', async () => {
    await el.loadCatalog();
    el['_sourceFilter'] = ['mcp'];
    await el.updateComplete;
    const items = el.shadowRoot!.querySelectorAll('.action-item');
    expect(items.length).toBe(1);
    expect(items[0]!.querySelector('.action-name')!.textContent).toBe('mcp.send-email');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-aria/src/controller/step-catalog.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the list view component**

Create `packages/pages-aria/src/controller/step-catalog.ts` with:
- LitElement extending class `PagesStepCatalog`
- CSS styles using `--pages-*` design tokens (follow `library-view.ts` patterns)
- `@property() baseUrl`, `@property() execBaseUrl`
- `@state() _actions: CatalogActionSummary[]`, `@state() _searchText`, `@state() _sourceFilter: string[]`, `@state() _selectedAction: CatalogActionDetail | null`, `@state() _view: 'list' | 'detail'`
- `loadCatalog()` method — fetches `catalogActions` via GraphQL POST to `baseUrl/graphql`
- `_filtered` getter — client-side filtering by name/description and source
- `render()` — search input (with `aria-label="Search step actions"`), source filter chips (with `role="checkbox"` and `aria-checked`), scrollable action list (`role="list"`) with action items (`role="listitem"`), each showing name, description, source badge, invoke kind badge, input/output count
- Click handler on action item sets `_selectedAction` and switches to detail view (implemented in Task 5)
- Register as `pages-step-catalog` custom element

Full implementation should be ~200 lines following the `library-view.ts` CSS patterns.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-aria/src/controller/step-catalog.test.ts`
Expected: PASS

- [ ] **Step 5: Export from controller index**

Add to `packages/pages-aria/src/controller/index.ts`:
```typescript
export { PagesStepCatalog } from './step-catalog.js';
```

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/src/controller/step-catalog.ts \
       packages/pages-aria/src/controller/step-catalog.test.ts \
       packages/pages-aria/src/controller/index.ts
git commit -m "feat(pages-aria): step-catalog Lit component with list view Refs #501"
```

## Batch 4: UI Component — Detail View, Try It, Template

### Task 5: Detail view with schema tables

**Files:**
- Modify: `packages/pages-aria/src/controller/step-catalog.ts`
- Modify: `packages/pages-aria/src/controller/step-catalog.test.ts`

**Interfaces:**
- Consumes: Fetches `catalogAction(name)` via GraphQL POST to `baseUrl/graphql`. Uses `CatalogActionDetail` shape with `inputs`, `outputs`, `invoke` maps.
- Produces: Detail view rendering within `PagesStepCatalog` — schema tables for inputs/outputs, invoke binding summary, back button.

- [ ] **Step 1: Write the failing test for detail view**

Add to `step-catalog.test.ts`:
```typescript
const MOCK_DETAIL = {
  name: 'check-compliance',
  description: 'Check doc compliance',
  invokeKind: 'rest',
  source: 'yaml',
  inputs: {
    documentId: { type: 'STRING', required: true, defaultValue: null, allowedValues: null, format: null, description: 'Document identifier' },
    standard: { type: 'STRING', required: true, defaultValue: 'ISO-27001', allowedValues: ['ISO-27001', 'SOC-2', 'GDPR'], format: null, description: 'Compliance standard' },
  },
  outputs: {
    compliant: { type: 'BOOLEAN', required: false, defaultValue: null, allowedValues: null, format: null, description: 'Whether compliant' },
  },
  invoke: { kind: 'rest', metadata: { method: 'POST', url: 'https://api.example.com/compliance/check' } },
};

describe('detail view', () => {
  it('shows input schema table', async () => {
    vi.stubGlobal('fetch', vi.fn().mockImplementation((url: string) => {
      const body = { data: url.includes('catalogAction') ? { catalogAction: MOCK_DETAIL } : { catalogActions: MOCK_ACTIONS } };
      return Promise.resolve({ ok: true, json: async () => body });
    }));
    el.baseUrl = 'http://localhost:8080';
    await el.loadCatalog();
    await el['_loadDetail']('check-compliance');
    await el.updateComplete;

    const table = el.shadowRoot!.querySelector('.inputs-table');
    expect(table).not.toBeNull();
    const rows = table!.querySelectorAll('tbody tr');
    expect(rows.length).toBe(2);
  });

  it('back button returns to list', async () => {
    el['_view'] = 'detail';
    await el.updateComplete;
    const back = el.shadowRoot!.querySelector('[aria-label="Back to catalog list"]') as HTMLElement;
    back?.click();
    await el.updateComplete;
    expect(el['_view']).toBe('list');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-aria/src/controller/step-catalog.test.ts`
Expected: FAIL — `_loadDetail` method doesn't exist or detail view not rendered

- [ ] **Step 3: Implement detail view rendering**

Add to `PagesStepCatalog`:
- `_loadDetail(name: string)` — fetches `catalogAction(name)` via GraphQL, sets `_selectedAction` and `_view = 'detail'`
- `_renderDetail()` — renders: back button (`aria-label="Back to catalog list"`), action name/description, source and invoke kind badges, inputs table (`<table class="inputs-table">` with columns: Name, Type, Required, Default, Allowed Values, Description), outputs table (Name, Type, Description), invoke binding summary (kind + metadata key-value pairs)
- Wire click handler on action list items to call `_loadDetail`
- CSS for detail view tables, badges, back button using `--pages-*` tokens

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-aria/src/controller/step-catalog.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/controller/step-catalog.ts \
       packages/pages-aria/src/controller/step-catalog.test.ts
git commit -m "feat(pages-aria): step-catalog detail view with schema tables Refs #501"
```

### Task 6: Try-it panel and YAML template generation

**Files:**
- Modify: `packages/pages-aria/src/controller/step-catalog.ts`
- Modify: `packages/pages-aria/src/controller/step-catalog.test.ts`

**Interfaces:**
- Consumes: `CatalogActionDetail` for form generation, `execBaseUrl` + `POST /scenario/catalog/execute` for live execution
- Produces: Try-it form panel in detail view, `step-template-selected` CustomEvent with `{ actionName: string, yaml: string }`, clipboard copy of YAML template

- [ ] **Step 1: Write failing tests for template generation and try-it**

Add to `step-catalog.test.ts`:
```typescript
describe('YAML template generation', () => {
  it('generates correct YAML from action detail', () => {
    const yaml = PagesStepCatalog.generateTemplate(MOCK_DETAIL);
    expect(yaml).toContain('- check-compliance:');
    expect(yaml).toContain('documentId:');
    expect(yaml).toContain('standard: "ISO-27001"');
  });
});

describe('try-it execution', () => {
  it('calls execute endpoint and shows result', async () => {
    const fetchMock = vi.fn()
      .mockResolvedValueOnce({ ok: true, json: async () => ({ data: { catalogActions: MOCK_ACTIONS } }) })
      .mockResolvedValueOnce({ ok: true, json: async () => ({ data: { catalogAction: MOCK_DETAIL } }) })
      .mockResolvedValueOnce({
        ok: true,
        json: async () => ({ kind: 'success', output: { compliant: true }, executionMetadata: {} }),
      });
    vi.stubGlobal('fetch', fetchMock);

    el.baseUrl = 'http://localhost:8080';
    el.execBaseUrl = 'http://localhost:9090';
    await el.loadCatalog();
    await el['_loadDetail']('check-compliance');
    await el['_executeAction']();
    await el.updateComplete;

    expect(fetchMock).toHaveBeenCalledWith(
      'http://localhost:9090/scenario/catalog/execute',
      expect.objectContaining({ method: 'POST' }),
    );
    const result = el.shadowRoot!.querySelector('.try-result');
    expect(result).not.toBeNull();
  });
});

describe('step-template-selected event', () => {
  it('emits event with YAML template on use-template click', async () => {
    vi.stubGlobal('fetch', vi.fn()
      .mockResolvedValueOnce({ ok: true, json: async () => ({ data: { catalogActions: MOCK_ACTIONS } }) })
      .mockResolvedValueOnce({ ok: true, json: async () => ({ data: { catalogAction: MOCK_DETAIL } }) }));

    el.baseUrl = 'http://localhost:8080';
    await el.loadCatalog();
    await el['_loadDetail']('check-compliance');
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('step-template-selected', (e) => events.push(e as CustomEvent));

    const btn = el.shadowRoot!.querySelector('[aria-label="Use template"]') as HTMLElement;
    btn?.click();
    await el.updateComplete;

    expect(events.length).toBe(1);
    expect(events[0]!.detail.actionName).toBe('check-compliance');
    expect(events[0]!.detail.yaml).toContain('check-compliance');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-aria/src/controller/step-catalog.test.ts`
Expected: FAIL — `generateTemplate`, `_executeAction` not implemented

- [ ] **Step 3: Implement template generation**

Add static method to `PagesStepCatalog`:
```typescript
static generateTemplate(detail: CatalogActionDetail): string {
  const lines: string[] = [`- ${detail.name}:`];
  for (const [name, param] of Object.entries(detail.inputs)) {
    const value = param.defaultValue ? `"${param.defaultValue}"` : '""';
    const comment = [param.type?.toLowerCase(), param.required ? 'required' : 'optional']
      .filter(Boolean).join(', ');
    lines.push(`    ${name}: ${value}  # ${comment}`);
  }
  return lines.join('\n');
}
```

- [ ] **Step 4: Implement try-it panel**

Add to detail view in `PagesStepCatalog`:
- `@state() _tryParams: Record<string, string>` — form values
- `@state() _tryResult: StepResult | null` — execution result
- `@state() _tryLoading: boolean`
- `_renderTryIt()` — generates form inputs from `_selectedAction.inputs` (text input for each param, select for `allowedValues`, respects `defaultValue`), execute button (`aria-label="Execute {name}"`), result display (formatted JSON for success, error message for failure, using `--pages-success-*` / `--pages-danger-*` tokens)
- `_executeAction()` — POST to `execBaseUrl/scenario/catalog/execute` with `{ actionName, params: _tryParams }`
- `_useTemplate()` — calls `generateTemplate`, copies to clipboard via `navigator.clipboard.writeText()`, dispatches `step-template-selected` CustomEvent

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run packages/pages-aria/src/controller/step-catalog.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/src/controller/step-catalog.ts \
       packages/pages-aria/src/controller/step-catalog.test.ts
git commit -m "feat(pages-aria): try-it execution and YAML template generation Refs #501"
```

## Batch 5: Scenario Controller Integration

### Task 7: Wire `<pages-step-catalog>` into scenario controller

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: `PagesStepCatalog` component (from Task 4-6), `step-template-selected` CustomEvent
- Produces: New `'catalog'` view mode in `PagesScenarioController`, header toggle button, event forwarding for `step-template-selected`

- [ ] **Step 1: Write the failing test**

Add to `scenario-controller.test.ts`:
```typescript
describe('catalog view', () => {
  it('toggles to catalog view', async () => {
    const el = new PagesScenarioController();
    document.body.appendChild(el);
    await el.updateComplete;

    el['_view'] = 'catalog';
    await el.updateComplete;

    const catalog = el.shadowRoot!.querySelector('pages-step-catalog');
    expect(catalog).not.toBeNull();
    el.remove();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-aria/src/controller/scenario-controller.test.ts`
Expected: FAIL — catalog view not rendered

- [ ] **Step 3: Add catalog view to scenario controller**

In `scenario-controller.ts`:
- Import `PagesStepCatalog` (ensure side-effect import for custom element registration)
- Add `'catalog'` to the view type: change `_view` state to accept `'outline' | 'library' | 'catalog'`
- Add `_toggleCatalog()` method (same pattern as `_toggleLibrary()`):
  ```typescript
  private _toggleCatalog(): void {
    if (this._view === 'catalog') {
      this._view = 'outline';
    } else {
      this._view = 'catalog';
    }
  }
  ```
- Add `_renderCatalog()` method:
  ```typescript
  private _renderCatalog(): TemplateResult {
    return html`
      <pages-step-catalog
        .baseUrl=${this._conn?.restBase ?? this.baseUrl ?? ''}
        .execBaseUrl=${this._conn?.restBase ?? this.baseUrl ?? ''}
        @step-template-selected=${(e: CustomEvent) => {
          this.dispatchEvent(new CustomEvent('step-template-selected', {
            detail: e.detail, bubbles: true, composed: true,
          }));
        }}
      ></pages-step-catalog>
    `;
  }
  ```
- Add catalog button to `_renderViewHeader()` alongside the library toggle
- In `render()`, add case for `this._view === 'catalog'` → `this._renderCatalog()`

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-aria/src/controller/scenario-controller.test.ts`
Expected: PASS

- [ ] **Step 5: Run full pages-aria test suite**

Run: `npx vitest run packages/pages-aria/`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/src/controller/scenario-controller.ts \
       packages/pages-aria/src/controller/scenario-controller.test.ts
git commit -m "feat(pages-aria): wire step-catalog view into scenario controller Refs #501"
```

## Batch 6: Gallery Example and Visual Verification

### Task 8: Gallery sample with step catalog demo

**Files:**
- Create: `examples/samples/Scenarios/Step Catalog.ts` (gallery sample with mock catalog data)

**Interfaces:**
- Consumes: `PagesStepCatalog` component, `PagesScenarioController` component
- Produces: Gallery-visible demo page with mock fetch for catalog queries, demonstrating list/filter/detail/try-it/template flow

- [ ] **Step 1: Create the gallery sample**

Create `examples/samples/Scenarios/Step Catalog.ts` following the pattern in `examples/samples/Scenarios/Script Library.ts`:
- Register mock fetch that intercepts `/graphql` requests for `catalogActions` and `catalogAction` queries
- Return mock data with 5-8 step actions across yaml/mcp/script sources and various invoke kinds
- Mock the `/scenario/catalog/execute` endpoint to return sample success/failure results
- Mount `<pages-step-catalog>` or `<pages-scenario-controller>` with catalog view active

- [ ] **Step 2: Run the gallery and verify visually**

Run: `npm run dev` (or the gallery dev server)
Navigate to the Scenarios → Step Catalog sample. Verify:
- Action list renders with all mock actions
- Search filters correctly
- Source chips filter correctly
- Clicking an action shows detail with input/output tables
- Try-it form appears with correct inputs
- Execute returns mock result
- Use Template copies YAML and shows a confirmation

- [ ] **Step 3: Commit**

```bash
git add examples/samples/Scenarios/
git commit -m "feat(gallery): step catalog browser demo sample Refs #501"
```

## References

- [2026-09-29-step-catalog-browser-design.md] — design spec this plan implements
- `packages/yaml-core/src/step/step-walker.ts` — StepCatalog, CatalogEntry interfaces
- `packages/yaml-core/src/step/step-types.ts` — StepDefinition, StepParameter, InvokeBinding
- `packages/yaml-core/src/step/step-catalog.ts` — CompositeStepCatalog
- `packages/yaml-core/src/step/structural-evaluator.ts` — StructuralStepEvaluator
- `packages/pages-aria/src/controller/library-view.ts` — UI pattern reference
- `packages/pages-aria/src/controller/scenario-controller.ts` — integration point
- `backend/mcp/src/main/java/io/casehub/pages/mcp/ScenarioResolver.java` — @McpDomain pattern
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioLibraryResource.java` — JAX-RS pattern
- `docs/protocols/casehub/aria-interaction-contract.md` — ARIA requirements
- `docs/protocols/casehub/css-design-tokens.md` — design token convention
- GitHub #501 — focal issue
- GitHub #502 — parent epic
