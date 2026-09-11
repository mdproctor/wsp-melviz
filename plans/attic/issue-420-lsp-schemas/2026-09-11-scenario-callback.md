# Scenario Callback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #416 — Scenario engine: add completion callback support for CaseHub worker integration

**Goal:** Add optional `callbackUrl`, `dispatchId`, and `callbackToken` parameters to the scenario engine so it can POST completion/failure notifications to external orchestrators.

**Architecture:** Three files modified (`ScenarioOrchestrator`, `ScenarioControlResource`, `ScenarioResolver`), one local record added (`WorkerCompletionPayload`). Completion detection in `onStepResult()` — fires HTTP POST on a virtual thread when all steps done or `on-error:stop` triggers. No new modules or dependencies.

**Tech Stack:** Java 21+, Quarkus, `java.net.http.HttpClient`, JUnit 5 + AssertJ

## Global Constraints

- Callback is optional — null `callbackUrl` preserves existing behaviour
- Payload must match `WorkerCompletionPayload(output, faulted, errorMessage)` from `casehubio/workers`
- Token sent as `X-Casehub-Callback-Token` header
- Callback URL includes `/{dispatchId}` path suffix
- Fire-and-forget — no retry, log failures

---

## Batch 1: Orchestrator callback plumbing

### Task 1: Step result accumulation and completion callback

**Files:**
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioOrchestrator.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioOrchestratorTest.java`

**Interfaces:**
- Consumes: `PushRequest.StepResult(id, sessionId, stepName, ok, error, result)`, `HierarchicalScenario.onError()` (String, nullable)
- Produces: `start(String yaml, boolean startPaused, String callbackUrl, String dispatchId, String callbackToken)` — new overload used by Task 2 and Task 3

- [ ] **Step 1: Write failing test — callback fires on completion**

```java
@Test
void callbackFiresOnCompletion() throws Exception {
    var sent = createCapture();
    var callbackRequests = new ArrayList<java.net.http.HttpRequest>();
    var orchestrator = new ScenarioOrchestrator(
        (connId, msg) -> sent.add(new SentMessage(connId, msg)), noopBroadcaster());

    orchestrator.onExecutorRegister("conn-1",
        new PushRequest.ExecutorRegister("1", "helpdesk",
            List.of("create-ticket", "verify-ticket")));

    var yaml = """
        scenario: callback-test
        steps:
          - label: "Create"
            target: helpdesk
            commands:
              - action: create-ticket
          - label: "Verify"
            target: helpdesk
            commands:
              - action: verify-ticket
        """;

    // Start a simple HTTP server to receive the callback
    var server = com.sun.net.httpserver.HttpServer.create(
        new java.net.InetSocketAddress(0), 0);
    var receivedPayloads = new ArrayList<String>();
    var receivedHeaders = new ArrayList<String>();
    server.createContext("/workers/complete/", exchange -> {
        receivedPayloads.add(new String(exchange.getRequestBody().readAllBytes()));
        receivedHeaders.add(exchange.getRequestHeaders().getFirst("X-Casehub-Callback-Token"));
        exchange.sendResponseHeaders(200, 0);
        exchange.close();
    });
    server.start();
    int port = server.getAddress().getPort();

    try {
        orchestrator.start(yaml, false,
            "http://localhost:" + port + "/workers/complete", "dispatch-123", "tok-abc");

        String sessionId = orchestrator.sessionId();
        orchestrator.onStepResult(new PushRequest.StepResult(
            "r1", sessionId, "Create", true, null, Map.of("ticketId", "T-001")));
        orchestrator.onStepResult(new PushRequest.StepResult(
            "r2", sessionId, "Verify", true, null, Map.of("status", "TRIAGED")));

        // Allow virtual thread to complete
        Thread.sleep(500);

        assertThat(receivedPayloads).hasSize(1);
        assertThat(receivedPayloads.getFirst()).contains("\"faulted\":false");
        assertThat(receivedPayloads.getFirst()).contains("\"Create\"");
        assertThat(receivedPayloads.getFirst()).contains("\"Verify\"");
        assertThat(receivedHeaders.getFirst()).isEqualTo("tok-abc");
    } finally {
        server.stop(0);
    }
}
```

- [ ] **Step 2: Write failing test — no callback when callbackUrl is null**

```java
@Test
void noCallbackWhenUrlIsNull() {
    var orchestrator = new ScenarioOrchestrator((c, m) -> {}, noopBroadcaster());

    orchestrator.onExecutorRegister("conn-1",
        new PushRequest.ExecutorRegister("1", "helpdesk",
            List.of("create-ticket")));

    var yaml = """
        scenario: no-callback-test
        steps:
          - label: "Create"
            target: helpdesk
            commands:
              - action: create-ticket
        """;
    orchestrator.start(yaml);

    String sessionId = orchestrator.sessionId();
    orchestrator.onStepResult(new PushRequest.StepResult(
        "r1", sessionId, "Create", true, null, Map.of()));

    assertThat(orchestrator.state().progress()).isEqualTo(1.0);
    // No exception, no side effects — existing behaviour preserved
}
```

- [ ] **Step 3: Write failing test — callback fires with faulted=true on on-error:stop**

```java
@Test
void callbackFiresWithFaultedOnErrorStop() throws Exception {
    var sent = createCapture();
    var orchestrator = new ScenarioOrchestrator(
        (connId, msg) -> sent.add(new SentMessage(connId, msg)), noopBroadcaster());

    orchestrator.onExecutorRegister("conn-1",
        new PushRequest.ExecutorRegister("1", "helpdesk",
            List.of("create-ticket")));

    var yaml = """
        scenario: fault-test
        on-error: stop
        steps:
          - label: "Create"
            target: helpdesk
            commands:
              - action: create-ticket
          - label: "Verify"
            target: helpdesk
            commands:
              - action: verify-ticket
        """;

    var server = com.sun.net.httpserver.HttpServer.create(
        new java.net.InetSocketAddress(0), 0);
    var receivedPayloads = new ArrayList<String>();
    server.createContext("/workers/complete/", exchange -> {
        receivedPayloads.add(new String(exchange.getRequestBody().readAllBytes()));
        exchange.sendResponseHeaders(200, 0);
        exchange.close();
    });
    server.start();
    int port = server.getAddress().getPort();

    try {
        orchestrator.start(yaml, false,
            "http://localhost:" + port + "/workers/complete", "dispatch-456", "tok-xyz");

        String sessionId = orchestrator.sessionId();
        orchestrator.onStepResult(new PushRequest.StepResult(
            "r1", sessionId, "Create", false, "ticket creation failed", Map.of()));

        Thread.sleep(500);

        assertThat(receivedPayloads).hasSize(1);
        assertThat(receivedPayloads.getFirst()).contains("\"faulted\":true");
        assertThat(receivedPayloads.getFirst()).contains("ticket creation failed");

        // Verify stop was broadcast to executors
        assertThat(sent.stream().anyMatch(s -> s.message().contains("\"command\":\"stop\""))).isTrue();
    } finally {
        server.stop(0);
    }
}
```

- [ ] **Step 4: Write failing test — faulted=true on natural completion with failed steps (on-error:continue)**

```java
@Test
void naturalCompletionWithFailedStepSetsFaulted() throws Exception {
    var orchestrator = new ScenarioOrchestrator((c, m) -> {}, noopBroadcaster());

    orchestrator.onExecutorRegister("conn-1",
        new PushRequest.ExecutorRegister("1", "helpdesk",
            List.of("create-ticket", "verify-ticket")));

    var yaml = """
        scenario: mixed-result-test
        steps:
          - label: "Create"
            target: helpdesk
            commands:
              - action: create-ticket
          - label: "Verify"
            target: helpdesk
            commands:
              - action: verify-ticket
        """;

    var server = com.sun.net.httpserver.HttpServer.create(
        new java.net.InetSocketAddress(0), 0);
    var receivedPayloads = new ArrayList<String>();
    server.createContext("/workers/complete/", exchange -> {
        receivedPayloads.add(new String(exchange.getRequestBody().readAllBytes()));
        exchange.sendResponseHeaders(200, 0);
        exchange.close();
    });
    server.start();
    int port = server.getAddress().getPort();

    try {
        orchestrator.start(yaml, false,
            "http://localhost:" + port + "/workers/complete", "d-789", "tok");

        String sessionId = orchestrator.sessionId();
        orchestrator.onStepResult(new PushRequest.StepResult(
            "r1", sessionId, "Create", false, "failed", Map.of()));
        orchestrator.onStepResult(new PushRequest.StepResult(
            "r2", sessionId, "Verify", true, null, Map.of()));

        Thread.sleep(500);

        assertThat(receivedPayloads).hasSize(1);
        assertThat(receivedPayloads.getFirst()).contains("\"faulted\":true");
    } finally {
        server.stop(0);
    }
}
```

- [ ] **Step 5: Run all four tests to verify they fail**

Run: `cd backend && mvn test -pl scenario-runtime -Dtest="ScenarioOrchestratorTest#callbackFiresOnCompletion+noCallbackWhenUrlIsNull+callbackFiresWithFaultedOnErrorStop+naturalCompletionWithFailedStepSetsFaulted" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `start` method doesn't accept callback parameters yet

- [ ] **Step 6: Implement — add session state fields and start overload**

Add to `ScenarioOrchestrator`:

```java
// New session state fields
private volatile String callbackUrl;
private volatile String dispatchId;
private volatile String callbackToken;
private final ConcurrentHashMap<String, Map<String, Object>> stepResults = new ConcurrentHashMap<>();

// New start overload
public void start(String yaml, boolean startPaused,
                  String callbackUrl, String dispatchId, String callbackToken) {
    this.callbackUrl   = callbackUrl;
    this.dispatchId    = dispatchId;
    this.callbackToken = callbackToken;
    this.stepResults.clear();
    start(yaml, startPaused);  // delegates to existing logic
}
```

Move `start(yaml, startPaused)` initialization of `stepResults.clear()` into the existing method too, and clear callback fields in `stop()`:

```java
// In stop():
this.callbackUrl   = null;
this.dispatchId    = null;
this.callbackToken = null;
this.stepResults.clear();
```

- [ ] **Step 7: Implement — step result accumulation in onStepResult()**

In `onStepResult()`, after `completedSteps.put(result.stepName(), result.ok())`:

```java
if (result.result() != null) {
    stepResults.put(result.stepName(), result.result());
}
```

- [ ] **Step 8: Implement — completion detection and callback firing**

At the end of `onStepResult()`, after `dispatchTriggeredSteps`:

```java
if (completedSteps.size() == allSteps.size()) {
    boolean anyFailed = completedSteps.values().stream().anyMatch(ok -> !ok);
    fireCallback(anyFailed, anyFailed ? "One or more steps failed" : null);
}
```

And for `on-error:stop`, inside the `if (!result.ok())` path (before dispatching triggered steps):

```java
if (!result.ok() && scenario != null && "stop".equals(scenario.onError())) {
    broadcastControl("stop", null);
    fireCallback(true, "Step '" + result.stepName() + "' failed: " + result.error());
    return;  // don't dispatch triggered steps
}
```

- [ ] **Step 9: Implement — fireCallback method and WorkerCompletionPayload record**

```java
record WorkerCompletionPayload(Map<String, Object> output, boolean faulted, String errorMessage) {}

private void fireCallback(boolean faulted, String errorMessage) {
    if (callbackUrl == null) return;

    Map<String, Object> output = new java.util.LinkedHashMap<>(stepResults);
    var payload = new WorkerCompletionPayload(output, faulted, errorMessage);

    Thread.ofVirtual().start(() -> {
        try {
            var client = java.net.http.HttpClient.newHttpClient();
            var body = JSON.writeValueAsString(payload);
            var request = java.net.http.HttpRequest.newBuilder()
                .uri(java.net.URI.create(callbackUrl + "/" + dispatchId))
                .header("Content-Type", "application/json")
                .header("X-Casehub-Callback-Token", callbackToken != null ? callbackToken : "")
                .POST(java.net.http.HttpRequest.BodyPublishers.ofString(body))
                .build();
            var response = client.send(request, java.net.http.HttpResponse.BodyHandlers.discarding());
            if (response.statusCode() >= 400) {
                System.err.println("Callback to " + callbackUrl + " returned " + response.statusCode());
            }
        } catch (Exception e) {
            System.err.println("Callback to " + callbackUrl + " failed: " + e.getMessage());
        }
    });
}
```

- [ ] **Step 10: Run all four tests to verify they pass**

Run: `cd backend && mvn test -pl scenario-runtime -Dtest="ScenarioOrchestratorTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioOrchestrator.java backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioOrchestratorTest.java
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#416): add completion callback to ScenarioOrchestrator

Stores callbackUrl, dispatchId, callbackToken in session state.
Accumulates step results. Fires HTTP POST on completion or
on-error:stop failure. Fire-and-forget on virtual thread.

Closes #416"
```

---

## Batch 2: API surface — REST and GraphQL

### Task 2: REST StartRequest callback parameters

**Files:**
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioControlResource.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioControlResourceTest.java` (create if not exists)

**Interfaces:**
- Consumes: `ScenarioOrchestrator.start(yaml, paused, callbackUrl, dispatchId, callbackToken)` from Task 1
- Produces: `POST /scenario/start` accepts `callbackUrl`, `dispatchId`, `callbackToken` in request body

- [ ] **Step 1: Write failing test — StartRequest passes callback params to orchestrator**

```java
@Test
void startPassesCallbackParamsToOrchestrator() {
    // Use a mock orchestrator or verify via state
    var resource = new ScenarioControlResource();
    // Inject a test orchestrator that captures the start call args
    var capturedArgs = new ArrayList<String>();
    var testOrchestrator = new ScenarioOrchestrator((c, m) -> {}, noopBroadcaster()) {
        @Override
        public void start(String yaml, boolean startPaused,
                          String callbackUrl, String dispatchId, String callbackToken) {
            capturedArgs.add(callbackUrl);
            capturedArgs.add(dispatchId);
            capturedArgs.add(callbackToken);
            // Don't actually start — no executors registered
        }
    };
    resource.orchestrator = testOrchestrator;

    var req = new ScenarioControlResource.StartRequest(
        "scenario: test\nsteps: []", false,
        "http://callback.example/workers/complete", "d-123", "tok-abc");

    // This will throw because no executors, but we verify params were passed
    try { resource.start(req); } catch (Exception ignored) {}

    assertThat(capturedArgs).containsExactly(
        "http://callback.example/workers/complete", "d-123", "tok-abc");
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `StartRequest` doesn't have callback fields yet

- [ ] **Step 3: Implement — update StartRequest and start method**

Update `ScenarioControlResource`:

```java
public record StartRequest(String yaml, boolean paused,
                           String callbackUrl, String dispatchId,
                           String callbackToken) {}

@POST
@Path("/start")
public ScenarioState start(StartRequest req) {
    if (req.callbackUrl() != null) {
        orchestrator.start(req.yaml(), req.paused(),
            req.callbackUrl(), req.dispatchId(), req.callbackToken());
    } else {
        orchestrator.start(req.yaml(), req.paused());
    }
    return orchestrator.state();
}
```

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/scenario-runtime/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#416): add callback params to POST /scenario/start

StartRequest accepts callbackUrl, dispatchId, callbackToken.
Passes through to orchestrator when callbackUrl is non-null.

Refs #416"
```

### Task 3: GraphQL scenarioSubmit callback parameters

**Files:**
- Modify: `backend/mcp/src/main/java/io/casehub/pages/mcp/ScenarioResolver.java`
- Test: `backend/mcp/src/test/java/io/casehub/pages/mcp/ScenarioResolverTest.java`

**Interfaces:**
- Consumes: `ScenarioOrchestrator.start(yaml, paused, callbackUrl, dispatchId, callbackToken)` from Task 1
- Produces: `scenarioSubmit(yaml, callbackUrl, dispatchId, callbackToken)` GraphQL mutation

- [ ] **Step 1: Write failing test — submit passes callback params**

Add to `ScenarioResolverTest`:

```java
@Test
void submitWithCallbackPassesParamsToOrchestrator() {
    // Similar pattern to Task 2 — verify orchestrator receives params
    var resolver = new ScenarioResolver();
    resolver.orchestrator = orchestrator;  // existing test field

    // Register executor to prevent missing-executor exception
    orchestrator.onExecutorRegister("conn-1",
        new PushRequest.ExecutorRegister("1", "browser", List.of("click")));

    var yaml = """
        scenario: gql-callback-test
        steps:
          - label: "Click"
            target: browser
            commands:
              - action: click
        """;

    resolver.submit(yaml, "http://callback/workers/complete", "d-gql", "tok-gql");

    // Verify session started (orchestrator has a sessionId)
    assertThat(orchestrator.sessionId()).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `submit` method doesn't accept callback parameters

- [ ] **Step 3: Implement — update submit method**

```java
@Mutation("scenarioSubmit")
public ScenarioState submit(String yaml, String callbackUrl,
                            String dispatchId, String callbackToken) {
    if (callbackUrl != null) {
        orchestrator.start(yaml, false, callbackUrl, dispatchId, callbackToken);
    } else {
        orchestrator.start(yaml);
    }
    return orchestrator.state();
}
```

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS

- [ ] **Step 5: Run full test suite for both modules**

Run: `cd backend && mvn test -pl scenario-runtime,mcp`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/mcp/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#416): add callback params to scenarioSubmit GraphQL mutation

scenarioSubmit accepts optional callbackUrl, dispatchId, callbackToken.
Passes through to orchestrator when callbackUrl is non-null.

Refs #416"
```

---

## References

- [2026-09-11-scenario-callback-design.md] — design spec this plan implements
- [ScenarioOrchestrator.java] — completion detection point (`onStepResult`)
- [ScenarioControlResource.java] — REST `StartRequest` record
- [ScenarioResolver.java] — GraphQL `scenarioSubmit` mutation
- [ScenarioOrchestratorTest.java] — existing test patterns (executor registration, step results)
- [WorkerCallbackResource.java] (casehubio/workers) — callback contract
- [WorkerCompletionPayload.java] (casehubio/workers) — payload shape
- [PushRequest.StepResult] — step result record (stepName, ok, error, result)
- [HierarchicalScenario.onError()] — on-error policy field (String, nullable)
- [GitHub #416] — focal issue
