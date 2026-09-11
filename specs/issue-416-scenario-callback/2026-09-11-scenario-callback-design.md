# Scenario Engine Completion Callback

> **Issue:** casehubio/casehub-pages#416
> **Date:** 2026-09-11
> **Status:** Draft

## 1. Overview

The scenario engine needs to notify external orchestrators when a scenario
completes or fails. The primary consumer is `workers-scenario` in
`casehubio/workers` — it dispatches scenario executions as CaseHub case
steps and needs a callback to report completion back to the case engine.

The callback follows the existing CaseHub worker callback contract:
`POST /workers/complete/{dispatchId}` with a `WorkerCompletionPayload`
body and `X-Casehub-Callback-Token` authentication header.

## 2. API Changes

### 2.1 GraphQL — `scenarioSubmit`

```graphql
type Mutation {
    scenarioSubmit(
        yaml: String!
        callbackUrl: String
        dispatchId: String
        callbackToken: String
    ): ScenarioState
}
```

All three callback parameters are optional. When `callbackUrl` is null,
no callback fires — existing behaviour is preserved.

### 2.2 REST — `POST /scenario/start`

```java
public record StartRequest(
    String yaml,
    boolean paused,
    String callbackUrl,
    String dispatchId,
    String callbackToken
) {}
```

Same semantics: callback fires only when `callbackUrl` is non-null.

## 3. ScenarioOrchestrator Changes

### 3.1 New session state fields

```java
private volatile String callbackUrl;
private volatile String dispatchId;
private volatile String callbackToken;
private final ConcurrentHashMap<String, Map<String, Object>> stepResults
    = new ConcurrentHashMap<>();
```

Set in `start()`. Cleared in `stop()` alongside existing session state.

### 3.2 Start method signature

```java
public void start(String yaml, boolean startPaused,
                  String callbackUrl, String dispatchId,
                  String callbackToken)
```

The existing `start(String yaml)` and `start(String yaml, boolean paused)`
overloads are preserved for backward compatibility — they delegate with
null callback parameters.

### 3.3 Step result accumulation

In `onStepResult()`, before the existing logic:

```java
if (result.result() != null) {
    stepResults.put(result.stepName(), result.result());
}
```

### 3.4 Completion detection

At the end of `onStepResult()`, after dispatching triggered steps:

```java
if (completedSteps.size() == allSteps.size()) {
    boolean anyFailed = completedSteps.values().stream().anyMatch(ok -> !ok);
    fireCallback(anyFailed, anyFailed ? "One or more steps failed" : null);
}
```

### 3.5 Failure detection — on-error:stop

When `result.ok()` is false, check the scenario's `on-error` policy.
The `HierarchicalScenario` already parses `on-error` from the YAML.

```java
if (!result.ok() && scenario.onError() == OnErrorPolicy.STOP) {
    broadcastControl("stop", null);
    fireCallback(true, "Step '" + result.stepName() + "' failed: "
        + result.error());
}
```

`on-error: continue` and `on-error: pause` do not fire the callback
immediately — `continue` lets remaining steps execute (callback fires
on natural completion with `faulted` derived from any `ok=false` results),
and `pause` waits for operator intervention.

### 3.6 Callback firing

```java
private void fireCallback(boolean faulted, String errorMessage) {
    if (callbackUrl == null) return;

    Map<String, Object> output = new LinkedHashMap<>(stepResults);
    var payload = new WorkerCompletionPayload(output, faulted, errorMessage);

    Thread.ofVirtual().start(() -> {
        try {
            var client = HttpClient.newHttpClient();
            var body = JSON.writeValueAsString(payload);
            var request = HttpRequest.newBuilder()
                .uri(URI.create(callbackUrl + "/" + dispatchId))
                .header("Content-Type", "application/json")
                .header("X-Casehub-Callback-Token", callbackToken != null ? callbackToken : "")
                .POST(HttpRequest.BodyPublishers.ofString(body))
                .build();
            var response = client.send(request, HttpResponse.BodyHandlers.discarding());
            if (response.statusCode() >= 400) {
                log.warn("Callback failed: {} returned {}", callbackUrl, response.statusCode());
            }
        } catch (Exception e) {
            log.warn("Callback to {} failed: {}", callbackUrl, e.getMessage());
        }
    });
}
```

Fire-and-forget on a virtual thread. No retry — the workers platform
handles timeouts and re-registration. Failures are logged but do not
affect scenario state.

### 3.7 WorkerCompletionPayload (local record)

Define a local record in `scenario-runtime` matching the workers contract:

```java
record WorkerCompletionPayload(
    Map<String, Object> output,
    boolean faulted,
    String errorMessage
) {}
```

No dependency on `workers-common` — the record is a trivial DTO matching
the JSON shape. The contract is defined by the HTTP endpoint, not by a
shared type.

## 4. on-error Policy Enforcement

The `HierarchicalScenario` parses `on-error` from the YAML format (values:
`continue`, `stop`, `pause`; default: `continue`). Currently the
orchestrator only skips dependents on step failure — it doesn't enforce
`stop` or `pause`.

This issue adds enforcement for `stop` only:

| Policy | Current behaviour | After #416 |
|--------|------------------|------------|
| `continue` | Skip dependents, continue | Unchanged — callback fires on natural completion |
| `stop` | Skip dependents, continue | Abort all executors, fire failure callback |
| `pause` | Skip dependents, continue | Unchanged — operator intervention needed |

`pause` enforcement is deferred — it requires the controller UI to surface
the failure and offer resume/abort, which is a separate concern.

## 5. What Does Not Change

- Push wire protocol — no new op types
- Browser executor — unaffected
- Service executors — unaffected
- Controller UI — unaffected (state broadcast still works)
- MCP tools — `scenario_start` could accept callback params but this is
  deferred (MCP callers don't need callbacks — they poll `scenario_state`)

## 6. Testing

- **Unit test:** `ScenarioOrchestratorTest` — verify callback fires on
  completion (all steps done), on failure (on-error:stop + step fails),
  and does NOT fire when `callbackUrl` is null
- **Unit test:** Verify `stepResults` accumulation across multiple steps
- **Integration test:** `ScenarioControlResourceTest` — verify
  `StartRequest` accepts and passes through callback parameters
- **Integration test:** `ScenarioResolverTest` — verify `scenarioSubmit`
  accepts callback parameters via GraphQL

The callback HTTP POST itself is tested with a mock HTTP server
(WireMock or Quarkus test endpoint) that verifies the payload shape,
`dispatchId` in the path, and `X-Casehub-Callback-Token` header.

## References

- `ScenarioOrchestrator.java` — completion detection point (`onStepResult`)
- `ScenarioControlResource.java` — REST `StartRequest` record
- `ScenarioResolver.java` — GraphQL `scenarioSubmit` mutation
- `WorkerCallbackResource.java` (casehubio/workers) — callback contract
- `WorkerCompletionPayload.java` (casehubio/workers) — payload shape
- `PendingCompletion.java` (casehubio/workers) — callbackToken field
- Distributed executor protocol spec §4.8 — session lifecycle
- Distributed executor protocol spec §4.10 — on-error policies
