# DeliveryHandler SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #516 — feat: DeliveryHandler SPI — plugin-extensible scenario delivery modes

**Goal:** Replace hardcoded delivery-mode dispatch with a CDI-discovered
DeliveryHandler SPI so plugin modules can add new delivery modes without
modifying Pages.

**Architecture:** Add `DeliveryHandler` interface, `StepOutcome`, and
`DeliveryContext` to `backend/scenario/`. Add `GenericStep` to the
sealed `ScenarioStep`. Simplify the parser to create `GenericStep` for
all `delivery:` values. Refactor `ScenarioExecutor` to discover handlers
via CDI and dispatch through them. Migrate existing dispatchers to
handler implementations.

**Tech Stack:** Java 21, Quarkus CDI, JUnit 5, AssertJ

## Global Constraints

- `DeliveryHandler`, `StepOutcome`, `DeliveryContext` live in `backend/scenario/` (model module)
- Handler implementations live in `backend/scenario-runtime/`
- `ScenarioStep` remains sealed — only `AriaStep` and `GenericStep` as permits
- Pages must stay domain-agnostic — no IoT/desired-state dependencies
- All existing parser and executor tests must still pass (adapted to new types)

---

## Batch 1: SPI Types and Step Model

### Task 1: DeliveryHandler SPI interface, StepOutcome, DeliveryContext, and GenericStep

**Files:**
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/DeliveryHandler.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/StepOutcome.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/DeliveryContext.java`
- Modify: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScenarioStep.java` — remove `GraphQLStep`, `SimulatedStep`, `RestStep`; add `GenericStep`
- Modify: `backend/scenario/src/main/java/io/casehub/pages/scenario/ScenarioParser.java` — replace delivery switch with GenericStep construction
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/StepOutcomeTest.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/ScenarioParserTest.java` (adapt existing)

**Interfaces:**
- Produces: `DeliveryHandler` interface with `String name()` and `StepOutcome execute(String stepName, Map<String,Object> data, DeliveryContext ctx)`
- Produces: `StepOutcome` record with `ok(String,Map)` and `fail(String,String)` factories
- Produces: `DeliveryContext` interface with `String config(String key)`, `String resolve(String template)`, `Map<String,Object> resolveMap(Map<String,Object> data)`
- Produces: `ScenarioStep.GenericStep(String name, String delivery, Map<String,Object> data)`

- [ ] **Step 1: Write tests for StepOutcome**

Create `backend/scenario/src/test/java/io/casehub/pages/scenario/StepOutcomeTest.java`:

```java
package io.casehub.pages.scenario;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class StepOutcomeTest {

    @Test
    void okFactoryCreatesSuccessfulOutcome() {
        StepOutcome outcome = StepOutcome.ok("step1", Map.of("key", "value"));
        assertThat(outcome.success()).isTrue();
        assertThat(outcome.stepName()).isEqualTo("step1");
        assertThat(outcome.result()).containsEntry("key", "value");
        assertThat(outcome.error()).isNull();
    }

    @Test
    void failFactoryCreatesFailedOutcome() {
        StepOutcome outcome = StepOutcome.fail("step2", "something broke");
        assertThat(outcome.success()).isFalse();
        assertThat(outcome.stepName()).isEqualTo("step2");
        assertThat(outcome.result()).isEmpty();
        assertThat(outcome.error()).isEqualTo("something broke");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn test -pl backend/scenario -Dtest=StepOutcomeTest -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: compilation error — `StepOutcome` does not exist

- [ ] **Step 3: Create the SPI types**

Create `DeliveryHandler.java` using `ide_create_file`:

```java
package io.casehub.pages.scenario;

import java.util.Map;

public interface DeliveryHandler {
    String name();
    StepOutcome execute(String stepName, Map<String, Object> data, DeliveryContext ctx);
}
```

Create `StepOutcome.java` using `ide_create_file`:

```java
package io.casehub.pages.scenario;

import java.util.Map;

public record StepOutcome(boolean success, String stepName,
                          Map<String, Object> result, String error) {
    public static StepOutcome ok(String stepName, Map<String, Object> result) {
        return new StepOutcome(true, stepName, result, null);
    }

    public static StepOutcome fail(String stepName, String error) {
        return new StepOutcome(false, stepName, Map.of(), error);
    }
}
```

Create `DeliveryContext.java` using `ide_create_file`:

```java
package io.casehub.pages.scenario;

import java.util.Map;

public interface DeliveryContext {
    String config(String key);
    String resolve(String template);
    Map<String, Object> resolveMap(Map<String, Object> data);
}
```

- [ ] **Step 4: Run StepOutcome test to verify it passes**

Run: `/opt/homebrew/bin/mvn test -pl backend/scenario -Dtest=StepOutcomeTest -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: PASS

- [ ] **Step 5: Add GenericStep to ScenarioStep, remove typed records**

Use `ide_edit_member` on `ScenarioStep.java` to replace the full sealed interface body. The new content:

```java
public sealed interface ScenarioStep {

    String name();

    record AriaStep(String name, String action, AriaTarget target,
                    String value, Map<String, Object> state,
                    Integer timeout) implements ScenarioStep {
        public AriaStep {
            Objects.requireNonNull(action, "action");
            state = state != null ? Map.copyOf(state) : null;
        }
    }

    record GenericStep(String name, String delivery,
                       Map<String, Object> data) implements ScenarioStep {
        public GenericStep {
            Objects.requireNonNull(delivery, "delivery");
            data = data != null ? Map.copyOf(data) : Map.of();
        }
    }
}
```

This removes `GraphQLStep`, `SimulatedStep`, and `RestStep`.

- [ ] **Step 6: Simplify ScenarioParser delivery switch**

Use `ide_replace_member` on `parseStep` method in `ScenarioParser.java`. Replace the delivery switch block (lines 94-99) with:

```java
return new ScenarioStep.GenericStep(
    (String) fields.get("name"),
    delivery,
    fields
);
```

Remove the `buildGraphQLStep`, `buildSimulatedStep`, and `buildRestStep` methods entirely using `ide_refactor_safe_delete` or `ide_edit_member`.

- [ ] **Step 7: Adapt existing parser tests**

Update `ScenarioParserTest.java`:
- Tests that cast to `GraphQLStep`, `RestStep`, `SimulatedStep` now cast to `GenericStep`
- Field assertions change from `step.domain()` to `step.data().get("domain")`
- `instanceof` checks change to `ScenarioStep.GenericStep.class`
- The `parsesGraphQLStep` test becomes:

```java
@Test
void parsesGraphQLStep() throws IOException {
    Scenario scenario = ScenarioParser.parse(fixture("graphql-inject-chat.yaml"));
    assertThat(scenario.steps()).hasSize(1);
    assertThat(scenario.steps().getFirst()).isInstanceOf(ScenarioStep.GenericStep.class);

    var step = (ScenarioStep.GenericStep) scenario.steps().getFirst();
    assertThat(step.name()).isEqualTo("inject-chat");
    assertThat(step.delivery()).isEqualTo("graphql");
    assertThat(step.data()).containsEntry("domain", "connectors");
    assertThat(step.data()).containsEntry("operation", "injectChat");
}
```

- The `parsesRestStep` test becomes:

```java
@Test
void parsesRestStep() {
    // same YAML as before
    Scenario scenario = ScenarioParser.parse(yaml);
    assertThat(scenario.steps()).hasSize(1);
    assertThat(scenario.steps().getFirst()).isInstanceOf(ScenarioStep.GenericStep.class);

    var step = (ScenarioStep.GenericStep) scenario.steps().getFirst();
    assertThat(step.name()).isEqualTo("create-trial");
    assertThat(step.delivery()).isEqualTo("rest");
    assertThat(step.data()).containsEntry("method", "POST");
    assertThat(step.data()).containsEntry("url", "/api/trials");
}
```

- Add a test for unknown delivery types now succeeding:

```java
@Test
void parsesUnknownDeliveryTypeAsGenericStep() {
    String yaml = """
            scenario: custom-test
            steps:
              - name: set-thermostat
                delivery: desired-state
                deviceId: "dev-001"
                properties:
                  targetTemperature: 22
            """;
    Scenario scenario = ScenarioParser.parse(yaml);
    var step = (ScenarioStep.GenericStep) scenario.steps().getFirst();
    assertThat(step.delivery()).isEqualTo("desired-state");
    assertThat(step.data()).containsEntry("deviceId", "dev-001");
}
```

- [ ] **Step 8: Fix compilation errors in ScenarioCompiler and ScenarioStepAdapter**

`ScenarioCompiler` and `ScenarioStepAdapter` don't reference the removed step types directly (they work with `HierarchicalStep` and `ScenarioCommand`). Verify with `ide_diagnostics` on each file. If any compilation errors appear, fix them.

- [ ] **Step 9: Run all scenario module tests**

Run: `/opt/homebrew/bin/mvn test -pl backend/scenario -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add backend/scenario/src/
git commit -m "feat(scenario): add DeliveryHandler SPI and GenericStep, simplify parser

Add DeliveryHandler interface, StepOutcome, DeliveryContext to the
scenario model module. Replace GraphQLStep/RestStep/SimulatedStep with
GenericStep. Parser now creates GenericStep for all delivery: values.

Refs #516"
```

## Batch 2: Executor Refactoring and Handler Implementations

### Task 2: DeliveryHandler implementations and ScenarioExecutor refactoring

**Files:**
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/GraphQLDeliveryHandler.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/RestDeliveryHandler.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/SimulatedDeliveryHandler.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/AriaDeliveryHandler.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/RuntimeDeliveryContext.java`
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioExecutor.java` — CDI-based dispatch
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/AwaitEngine.java` — generify to `Supplier<T>`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/GraphQLDeliveryHandlerTest.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/RestDeliveryHandlerTest.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/SimulatedDeliveryHandlerTest.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/AriaDeliveryHandlerTest.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioExecutorTest.java` (adapt existing)

**Interfaces:**
- Consumes: `DeliveryHandler` from Task 1
- Consumes: `StepOutcome` from Task 1
- Consumes: `DeliveryContext` from Task 1
- Consumes: `ScenarioStep.GenericStep` from Task 1
- Produces: Four CDI beans implementing `DeliveryHandler`
- Produces: `RuntimeDeliveryContext` implementing `DeliveryContext`

- [ ] **Step 1: Write test for GraphQLDeliveryHandler**

Create `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/GraphQLDeliveryHandlerTest.java`:

```java
package io.casehub.pages.scenario.runtime;

import io.casehub.pages.scenario.DeliveryContext;
import io.casehub.pages.scenario.StepOutcome;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class GraphQLDeliveryHandlerTest {

    @Test
    void nameReturnsGraphql() {
        var handler = new GraphQLDeliveryHandler(new GraphQLDispatcher());
        assertThat(handler.name()).isEqualTo("graphql");
    }

    @Test
    void executeDelegatesToDispatcher() {
        var dispatcher = new GraphQLDispatcher(null, null) {
            @Override
            public Map<String, Object> dispatch(String domain, String operation,
                                                 Map<String, Object> params,
                                                 String endpoint, VariableContext ctx) {
                return Map.of("caseId", "C-001");
            }
        };
        var handler = new GraphQLDeliveryHandler(dispatcher);
        var ctx = stubContext("http://localhost:8080/graphql");

        StepOutcome outcome = handler.execute("inject", Map.of(
                "domain", "connectors",
                "operation", "injectChat",
                "params", Map.of("sender", "Alice")
        ), ctx);

        assertThat(outcome.success()).isTrue();
        assertThat(outcome.result()).containsEntry("caseId", "C-001");
    }

    private static DeliveryContext stubContext(String graphqlEndpoint) {
        return new DeliveryContext() {
            @Override public String config(String key) {
                if (key.startsWith("graphql.endpoint.")) return graphqlEndpoint;
                return null;
            }
            @Override public String resolve(String template) { return template; }
            @Override public Map<String, Object> resolveMap(Map<String, Object> data) { return data; }
        };
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn test -pl backend/scenario-runtime -Dtest=GraphQLDeliveryHandlerTest -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: compilation error — `GraphQLDeliveryHandler` does not exist

- [ ] **Step 3: Refactor GraphQLDispatcher to accept raw data**

The existing `GraphQLDispatcher.dispatch()` takes a typed `ScenarioStep.GraphQLStep`. Add an overloaded method that accepts raw parameters:

```java
public Map<String, Object> dispatch(String domain, String operation,
                                     Map<String, Object> params,
                                     String endpoint, VariableContext ctx) {
    Map<String, Object> resolvedParams = ctx.resolveMap(params);
    String query = buildQuery(operation, resolvedParams);
    // ... same HTTP logic as existing dispatch()
}
```

Alternatively, the handler can construct the HTTP call itself using the dispatcher's existing `buildQuery` and `parseResponse` methods. Choose whichever avoids duplication.

- [ ] **Step 4: Create GraphQLDeliveryHandler**

Create `GraphQLDeliveryHandler.java` using `ide_create_file`:

```java
package io.casehub.pages.scenario.runtime;

import io.casehub.pages.scenario.DeliveryContext;
import io.casehub.pages.scenario.DeliveryHandler;
import io.casehub.pages.scenario.StepOutcome;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class GraphQLDeliveryHandler implements DeliveryHandler {

    private final GraphQLDispatcher dispatcher;

    @Inject
    public GraphQLDeliveryHandler(GraphQLDispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }

    @Override
    public String name() { return "graphql"; }

    @Override
    @SuppressWarnings("unchecked")
    public StepOutcome execute(String stepName, Map<String, Object> data,
                               DeliveryContext ctx) {
        try {
            String domain = (String) data.get("domain");
            String operation = (String) data.get("operation");
            Map<String, Object> params = data.containsKey("params")
                    ? (Map<String, Object>) data.get("params") : Map.of();
            String endpoint = ctx.config("graphql.endpoint." + domain);
            Map<String, Object> resolvedParams = ctx.resolveMap(params);
            Map<String, Object> result = dispatcher.dispatch(domain, operation,
                    resolvedParams, endpoint, null);
            return StepOutcome.ok(stepName, result);
        } catch (Exception e) {
            return StepOutcome.fail(stepName, e.getMessage());
        }
    }
}
```

- [ ] **Step 5: Run GraphQLDeliveryHandler test**

Run: `/opt/homebrew/bin/mvn test -pl backend/scenario-runtime -Dtest=GraphQLDeliveryHandlerTest -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: PASS

- [ ] **Step 6: Create RestDeliveryHandler, SimulatedDeliveryHandler, AriaDeliveryHandler**

Create each using `ide_create_file`. Follow the same pattern as GraphQLDeliveryHandler.

**RestDeliveryHandler:**
```java
@ApplicationScoped
public class RestDeliveryHandler implements DeliveryHandler {
    private final RestDispatcher dispatcher;

    @Inject
    public RestDeliveryHandler(RestDispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }

    @Override public String name() { return "rest"; }

    @Override
    @SuppressWarnings("unchecked")
    public StepOutcome execute(String stepName, Map<String, Object> data,
                               DeliveryContext ctx) {
        try {
            String method = (String) data.getOrDefault("method", "POST");
            String url = (String) data.get("url");
            Map<String, Object> body = data.containsKey("body")
                    ? (Map<String, Object>) data.get("body") : Map.of();
            Map<String, String> headers = extractHeaders(data);
            Integer expectedStatus = extractExpectedStatus(data);
            String baseUrl = ctx.config("rest.baseUrl");

            Map<String, Object> resolvedBody = ctx.resolveMap(body);
            String resolvedUrl = ctx.resolve(baseUrl + url);
            Map<String, Object> result = dispatcher.dispatch(
                    method, resolvedUrl, resolvedBody, headers, expectedStatus);
            return StepOutcome.ok(stepName, result);
        } catch (Exception e) {
            return StepOutcome.fail(stepName, e.getMessage());
        }
    }

    // extractHeaders and extractExpectedStatus helper methods
}
```

**SimulatedDeliveryHandler:**
```java
@ApplicationScoped
public class SimulatedDeliveryHandler implements DeliveryHandler {
    @Override public String name() { return "simulated"; }

    @Override
    public StepOutcome execute(String stepName, Map<String, Object> data,
                               DeliveryContext ctx) {
        return StepOutcome.ok(stepName, Map.of());
    }
}
```

**AriaDeliveryHandler:**
```java
@ApplicationScoped
public class AriaDeliveryHandler implements DeliveryHandler {
    private final AriaDispatcher dispatcher;

    @Inject
    public AriaDeliveryHandler(AriaDispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }

    @Override public String name() { return "aria"; }

    @Override
    public StepOutcome execute(String stepName, Map<String, Object> data,
                               DeliveryContext ctx) {
        if (dispatcher == null) {
            return StepOutcome.ok(stepName, Map.of());
        }
        try {
            var ariaStep = toAriaStep(stepName, data);
            var result = dispatcher.send(ariaStep);
            var resultMap = result.result() != null ? result.result() : Map.<String, Object>of();
            return StepOutcome.ok(stepName, resultMap);
        } catch (AriaCommandException e) {
            return StepOutcome.fail(stepName, e.getMessage());
        }
    }

    // toAriaStep converts data map back to AriaStep for the dispatcher
}
```

- [ ] **Step 7: Write tests for each handler**

Create test files for Rest, Simulated, and Aria handlers following the GraphQLDeliveryHandlerTest pattern. At minimum:
- `name()` returns correct string
- `execute()` delegates to underlying dispatcher (or returns ok for simulated)
- Failure scenarios return `StepOutcome.fail`

- [ ] **Step 8: Run handler tests**

Run: `/opt/homebrew/bin/mvn test -pl backend/scenario-runtime -Dtest="*DeliveryHandlerTest" -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: all PASS

- [ ] **Step 9: Create RuntimeDeliveryContext**

Create `RuntimeDeliveryContext.java` using `ide_create_file`:

```java
package io.casehub.pages.scenario.runtime;

import io.casehub.pages.scenario.DeliveryContext;
import java.util.Map;

class RuntimeDeliveryContext implements DeliveryContext {
    private final ScenarioConfig config;
    private final VariableContext variables;

    RuntimeDeliveryContext(ScenarioConfig config, VariableContext variables) {
        this.config = config;
        this.variables = variables;
    }

    @Override
    public String config(String key) {
        return switch (key) {
            case "rest.baseUrl" -> config.restBaseUrl();
            default -> {
                if (key.startsWith("graphql.endpoint.")) {
                    yield config.graphQLEndpoint(key.substring("graphql.endpoint.".length()));
                }
                if (key.startsWith("push.endpoint.")) {
                    yield config.pushEndpoint(key.substring("push.endpoint.".length()));
                }
                yield null;
            }
        };
    }

    @Override
    public String resolve(String template) {
        return variables.resolve(template);
    }

    @Override
    public Map<String, Object> resolveMap(Map<String, Object> data) {
        return variables.resolveMap(data);
    }
}
```

- [ ] **Step 10: Generify AwaitEngine**

Use `ide_edit_member` to change `AwaitEngine` from `Supplier<Map<String,Object>>` to generic `Supplier<T>`:

```java
public class AwaitEngine<T> {
    private final Supplier<T> queryExecutor;

    public AwaitEngine(Supplier<T> queryExecutor) {
        this.queryExecutor = queryExecutor;
    }

    public T poll(AwaitCondition condition) {
        // same polling logic, but return T instead of Map
        // matches() needs to work with StepOutcome — extract result map
    }
}
```

Note: the `matches()` method checks result fields against the `condition.match()` map. For `StepOutcome`, it needs to check `outcome.result()`. Add a `Function<T, Map<String,Object>> resultExtractor` parameter, or keep `AwaitEngine` non-generic and have the executor wrap the handler call to return `Map` for polling, then convert the final result to `StepOutcome`. The simpler approach: keep `AwaitEngine` as-is (returning `Map<String,Object>`), and have the executor wrap:

```java
if (await != null) {
    var engine = new AwaitEngine(() -> handler.execute(stepName, data, ctx).result());
    Map<String, Object> result = engine.poll(await);
    outcome = StepOutcome.ok(stepName, result);
} else {
    outcome = handler.execute(stepName, data, ctx);
}
```

This avoids generifying `AwaitEngine` entirely. The handler executes per poll iteration, and `AwaitEngine` sees only the result map for matching.

- [ ] **Step 11: Refactor ScenarioExecutor to CDI-based dispatch**

Use `ide_edit_member` on `ScenarioExecutor.java`. Replace the constructor and dispatch logic:

```java
@ApplicationScoped
public class ScenarioExecutor {

    private static final Set<String> NON_BATCHABLE_ACTIONS =
            Set.of("navigate", "wait", "assert");

    private final Map<String, DeliveryHandler> handlers;
    private final AriaDeliveryHandler ariaHandler;

    @Inject
    public ScenarioExecutor(@Any Instance<DeliveryHandler> handlerInstances) {
        this.handlers = new HashMap<>();
        AriaDeliveryHandler foundAria = null;
        for (DeliveryHandler h : handlerInstances) {
            handlers.put(h.name(), h);
            if (h instanceof AriaDeliveryHandler a) foundAria = a;
        }
        this.ariaHandler = foundAria;
    }

    // Test-friendly constructor
    public ScenarioExecutor(List<DeliveryHandler> handlerList) {
        this.handlers = new HashMap<>();
        AriaDeliveryHandler foundAria = null;
        for (DeliveryHandler h : handlerList) {
            handlers.put(h.name(), h);
            if (h instanceof AriaDeliveryHandler a) foundAria = a;
        }
        this.ariaHandler = foundAria;
    }

    public List<ExecutionResult> execute(Scenario scenario, ScenarioConfig config) {
        var context = new VariableContext();
        var results = new ArrayList<ExecutionResult>();
        var steps = scenario.steps();

        int i = 0;
        while (i < steps.size()) {
            ScenarioStep step = steps.get(i);

            if (step instanceof ScenarioStep.AriaStep as && isBatchable(as)) {
                var batch = collectBatch(steps, i);
                ExecutionResult result = executeBatch(batch, config, context);
                results.add(result);
                if (!result.success()) {
                    throw new RuntimeException("Batch failed: " + result.error());
                }
                i += batch.size();
            } else {
                ExecutionResult result = executeStep(step, config, context);
                results.add(result);
                if (!result.success()) {
                    throw new RuntimeException("Step '" + step.name()
                                               + "' failed: " + result.error());
                }
                if (result.result() != null && !result.result().isEmpty()
                        && step.name() != null) {
                    context.put(step.name(), result.result());
                }
                i++;
            }
        }
        return results;
    }

    private ExecutionResult executeStep(ScenarioStep step, ScenarioConfig config,
                                        VariableContext context) {
        String delivery;
        String stepName;
        Map<String, Object> data;

        switch (step) {
            case ScenarioStep.AriaStep as -> {
                delivery = "aria";
                stepName = as.name();
                data = ariaStepToMap(as);
            }
            case ScenarioStep.GenericStep gs -> {
                delivery = gs.delivery();
                stepName = gs.name();
                data = gs.data();
            }
        }

        DeliveryHandler handler = handlers.get(delivery);
        if (handler == null) {
            return ExecutionResult.fail(stepName,
                "No DeliveryHandler registered for delivery type: " + delivery);
        }

        DeliveryContext ctx = new RuntimeDeliveryContext(config, context);
        AwaitCondition await = extractAwait(data);

        try {
            StepOutcome outcome;
            if (await != null) {
                var engine = new AwaitEngine(() -> handler.execute(stepName, data, ctx).result());
                Map<String, Object> result = engine.poll(await);
                outcome = StepOutcome.ok(stepName, result);
            } else {
                outcome = handler.execute(stepName, data, ctx);
            }
            return new ExecutionResult(outcome.stepName(), outcome.success(),
                    outcome.result(), outcome.error());
        } catch (Exception e) {
            return ExecutionResult.fail(stepName, e.getMessage());
        }
    }

    @SuppressWarnings("unchecked")
    private AwaitCondition extractAwait(Map<String, Object> data) {
        Object awaitObj = data.get("await");
        if (awaitObj instanceof Map<?,?> awaitMap) {
            Map<String, Object> match = (Map<String, Object>) awaitMap.get("match");
            if (match == null) return null;
            Integer timeout = awaitMap.containsKey("timeout")
                    ? ((Number) awaitMap.get("timeout")).intValue() : null;
            Integer interval = awaitMap.containsKey("interval")
                    ? ((Number) awaitMap.get("interval")).intValue() : null;
            return new AwaitCondition(match, timeout, interval);
        }
        return null;
    }

    private Map<String, Object> ariaStepToMap(ScenarioStep.AriaStep as) {
        var map = new HashMap<String, Object>();
        map.put("action", as.action());
        if (as.target() != null) {
            map.put("target", Map.of("role", as.target().role(), "name", as.target().name()));
        }
        if (as.value() != null) map.put("value", as.value());
        if (as.state() != null) map.put("state", as.state());
        if (as.timeout() != null) map.put("timeout", as.timeout());
        return map;
    }

    // batching methods remain, delegating to ariaHandler
}
```

- [ ] **Step 12: Adapt existing ScenarioExecutorTest**

Update `ScenarioExecutorTest.java` to use the new `List<DeliveryHandler>` constructor instead of the old `GraphQLDispatcher`/`AriaDispatcher`/`RestDispatcher` constructor. Update step constructions from typed records to `GenericStep`. Aria tests still use `AriaStep`.

Key changes:
- `new ScenarioStep.GraphQLStep(...)` → `new ScenarioStep.GenericStep("name", "graphql", Map.of("domain", "...", "operation", "...", "params", Map.of(...)))`
- `new ScenarioExecutor(dispatcher)` → `new ScenarioExecutor(List.of(new GraphQLDeliveryHandler(dispatcher), new SimulatedDeliveryHandler()))`
- Stub dispatchers adapt to the new handler-based test pattern

- [ ] **Step 13: Add test for unknown delivery type**

Add to `ScenarioExecutorTest`:

```java
@Test
void unknownDeliveryTypeReturnsFailure() {
    var executor = new ScenarioExecutor(List.of(new SimulatedDeliveryHandler()));
    var scenario = new Scenario("test", List.of(
            new ScenarioStep.GenericStep("step1", "desired-state",
                    Map.of("deviceId", "dev-001"))));

    assertThatThrownBy(() -> executor.execute(scenario, ScenarioConfig.localhost()))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("No DeliveryHandler registered");
}
```

- [ ] **Step 14: Run all scenario-runtime tests**

Run: `/opt/homebrew/bin/mvn test -pl backend/scenario-runtime -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: all PASS

- [ ] **Step 15: Fix any remaining compilation errors across the project**

Run: `/opt/homebrew/bin/mvn compile -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Check `ide_diagnostics` on any files that reference the removed step types (`GraphQLStep`, `RestStep`, `SimulatedStep`). Fix them — they should now use `GenericStep` or the handler implementations.

Also check `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/GraphQLDispatcherTest.java` and `AriaDispatcherTest.java` — these test the dispatchers directly and may reference removed types.

- [ ] **Step 16: Run full project build**

Run: `/opt/homebrew/bin/mvn test -f /Users/mdproctor/claude/casehub/pages/pom.xml`
Expected: all PASS

- [ ] **Step 17: Commit**

```bash
git add backend/scenario-runtime/src/
git commit -m "feat(scenario-runtime): CDI-based DeliveryHandler dispatch

Implement GraphQL, REST, Simulated, and Aria delivery handlers.
Refactor ScenarioExecutor to discover handlers via CDI Instance.
Executor retains batching and await wrapping logic.

Refs #516"
```

## References

- [2026-10-04-deliveryhandler-spi-design.md] — design spec this plan implements
- [ScenarioStep.java] — sealed interface being extended with GenericStep
- [ScenarioParser.java:94-99] — hardcoded delivery switch being removed
- [ScenarioExecutor.java:73-79] — hardcoded step dispatch being refactored
- [DataProvider.java + DataResource.java:42] — CDI SPI pattern reference
- [GraphQLDispatcher.java] — existing dispatcher being wrapped by handler
- [RestDispatcher.java] — existing dispatcher being wrapped by handler
- [AwaitEngine.java] — await polling engine, kept non-generic
- [ScenarioExecutorTest.java] — existing tests to adapt
- [ScenarioParserTest.java] — existing tests to adapt
- [GitHub #516] — focal issue
