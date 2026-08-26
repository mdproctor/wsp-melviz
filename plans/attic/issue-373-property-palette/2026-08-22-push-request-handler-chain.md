# Push Request Handler Chain — Zero-Config Orchestrator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehub-pages#343 — Move scenario push routing into push-runtime — zero-config orchestrator
**Issue group:** #343

**Goal:** Extract scenario-specific push wire routing from the helpdesk push endpoint into a handler chain SPI, so any Quarkus app with scenario-runtime + push-runtime on the classpath gets orchestrator wiring automatically.

**Architecture:** Create a `PushRequestHandler` interface in the `push` module (pure Java SPI). Implement `ScenarioPushHandler` in `scenario-runtime` as a CDI bean that handles `executor-register` and `step-result` ops by delegating to `ScenarioOrchestrator`. Simplify `HelpdeskPushEndpoint` to inject `Instance<PushRequestHandler>` and iterate handlers for any ops not handled as core protocol (`listen`/`unlisten`).

**Tech Stack:** Java 21, Quarkus CDI (Arc), JUnit 5, AssertJ

## Global Constraints

- `PushRequestHandler` interface lives in `push` module (no CDI dependency — pure Java SPI)
- `ScenarioPushHandler` lives in `scenario-runtime` (already depends on `push-runtime` → `push` transitively)
- Handler chain iterates all CDI-discovered `PushRequestHandler` beans — first match wins
- `Listen`/`Unlisten` remain hardcoded in the endpoint (core push protocol, not extensible)
- J2CL: `PushRequestHandler` is in `push` (core logic) — no CDI annotations, no reflection

---

## Batch 1: Handler chain SPI and scenario handler

### Task 1: PushRequestHandler interface + ScenarioPushHandler

**Files:**
- Create: `backend/push/src/main/java/io/casehub/pages/push/PushRequestHandler.java`
- Create: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioPushHandler.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioPushHandlerTest.java`

**Interfaces:**
- Consumes: `PushRequest.ExecutorRegister`, `PushRequest.StepResult`, `ScenarioOrchestrator.onExecutorRegister(String, PushRequest.ExecutorRegister)`, `ScenarioOrchestrator.onStepResult(PushRequest.StepResult)`
- Produces: `PushRequestHandler` interface (`boolean handles(PushRequest)`, `void handle(String connectionId, PushRequest request)`), `ScenarioPushHandler` CDI bean

- [ ] **Step 1: Write the failing test for ScenarioPushHandler**

Create the test class first. It exercises both dispatch paths (executor-register and step-result) and verifies non-scenario ops are rejected.

```java
// backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioPushHandlerTest.java
package io.casehub.pages.scenario.runtime;

import io.casehub.pages.push.EventBroadcaster;
import io.casehub.pages.push.InMemoryEventStore;
import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.TopicRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class ScenarioPushHandlerTest {

    private ScenarioOrchestrator orchestrator;
    private ScenarioPushHandler handler;

    @BeforeEach
    void setUp() {
        var broadcaster = new EventBroadcaster(
            new InMemoryEventStore(10), new TopicRegistry(),
            (id, msg) -> {}, obj -> "{}");
        orchestrator = new ScenarioOrchestrator((id, msg) -> {}, broadcaster);
        handler = new ScenarioPushHandler(orchestrator);
    }

    @Test
    void handlesExecutorRegister() {
        var reg = new PushRequest.ExecutorRegister("1", "helpdesk",
            List.of("create-ticket"));
        assertThat(handler.handles(reg)).isTrue();
    }

    @Test
    void handlesStepResult() {
        var result = new PushRequest.StepResult("1", "s-1", "step-1",
            true, null, Map.of());
        assertThat(handler.handles(result)).isTrue();
    }

    @Test
    void doesNotHandleListen() {
        var listen = new PushRequest.Listen("1", List.of("topic"),
            Map.of());
        assertThat(handler.handles(listen)).isFalse();
    }

    @Test
    void handleDelegatesExecutorRegisterToOrchestrator() {
        var reg = new PushRequest.ExecutorRegister("1", "browser",
            List.of("click"));
        handler.handle("conn-1", reg);
        // Verify executor was registered by starting a scenario that
        // requires this executor — no exception means it registered
        var yaml = """
            scenario: test
            steps:
              - label: "Click"
                target: browser
                commands:
                  - action: click
            """;
        assertThatCode(() -> orchestrator.start(yaml))
            .doesNotThrowAnyException();
    }

    @Test
    void handleDelegatesStepResultToOrchestrator() {
        // Register executor, start scenario, then send step result
        orchestrator.onExecutorRegister("conn-1",
            new PushRequest.ExecutorRegister("1", "browser",
                List.of("click")));
        var yaml = """
            scenario: test
            steps:
              - label: "Click"
                target: browser
                commands:
                  - action: click
            """;
        orchestrator.start(yaml);

        var result = new PushRequest.StepResult("2",
            orchestrator.sessionId(), "Click",
            true, null, Map.of());
        handler.handle("conn-1", result);

        assertThat(orchestrator.state().progress()).isEqualTo(1.0);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `mvn test -pl backend/scenario-runtime -Dtest=ScenarioPushHandlerTest -f backend/pom.xml`
Expected: compilation failure — `PushRequestHandler` and `ScenarioPushHandler` do not exist yet.

- [ ] **Step 3: Create PushRequestHandler interface**

```java
// backend/push/src/main/java/io/casehub/pages/push/PushRequestHandler.java
package io.casehub.pages.push;

public interface PushRequestHandler {
    boolean handles(PushRequest request);
    void handle(String connectionId, PushRequest request);
}
```

- [ ] **Step 4: Create ScenarioPushHandler implementation**

```java
// backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioPushHandler.java
package io.casehub.pages.scenario.runtime;

import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.PushRequestHandler;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class ScenarioPushHandler implements PushRequestHandler {

    private final ScenarioOrchestrator orchestrator;

    @Inject
    public ScenarioPushHandler(ScenarioOrchestrator orchestrator) {
        this.orchestrator = orchestrator;
    }

    @Override
    public boolean handles(PushRequest request) {
        return request instanceof PushRequest.ExecutorRegister
            || request instanceof PushRequest.StepResult;
    }

    @Override
    public void handle(String connectionId, PushRequest request) {
        switch (request) {
            case PushRequest.ExecutorRegister reg ->
                orchestrator.onExecutorRegister(connectionId, reg);
            case PushRequest.StepResult result ->
                orchestrator.onStepResult(result);
            default ->
                throw new IllegalArgumentException(
                    "Unhandled op: " + request.op());
        }
    }
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `mvn test -pl backend/scenario-runtime -Dtest=ScenarioPushHandlerTest -f backend/pom.xml`
Expected: all 5 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add backend/push/src/main/java/io/casehub/pages/push/PushRequestHandler.java
git add backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioPushHandler.java
git add backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioPushHandlerTest.java
git commit -m "feat(#343): PushRequestHandler SPI + ScenarioPushHandler

Introduces PushRequestHandler interface in push module as an SPI for
extensible push wire request routing. ScenarioPushHandler in
scenario-runtime implements it, delegating executor-register and
step-result ops to ScenarioOrchestrator.

Refs casehubio/casehub-pages#343"
```

---

## Batch 2: Simplify helpdesk push endpoint

### Task 2: Wire handler chain into HelpdeskPushEndpoint

**Files:**
- Modify: `examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/push/HelpdeskPushEndpoint.java`
- Modify: `examples/helpdesk/src/test/java/io/casehub/examples/helpdesk/push/PushEndpointTest.java`

**Interfaces:**
- Consumes: `PushRequestHandler` (from Task 1), `Instance<PushRequestHandler>` (CDI injection)
- Produces: simplified endpoint — no `ScenarioOrchestrator` import, handler chain delegates extensible ops

- [ ] **Step 1: Write a failing integration test for handler chain delegation**

Add a test to `PushEndpointTest` that sends an `executor-register` message and verifies an ack is returned. This exercises the handler chain path (since the endpoint no longer handles `executor-register` directly — the `ScenarioPushHandler` bean does it via CDI auto-discovery).

```java
// Add to PushEndpointTest.java
@Test
void executor_register_dispatched_via_handler_chain() throws Exception {
    var messages = new CopyOnWriteArrayList<String>();
    var latch = new CountDownLatch(1);

    var wsUri = URI.create(pushUri.toString().replace("http://", "ws://"));
    var ws = HttpClient.newHttpClient().newWebSocketBuilder()
            .buildAsync(wsUri, new WebSocket.Listener() {
                @Override
                public CompletionStage<?> onText(WebSocket webSocket,
                        CharSequence data, boolean last) {
                    messages.add(data.toString());
                    latch.countDown();
                    webSocket.request(1);
                    return null;
                }
            }).join();

    ws.sendText("""
        {"op":"executor-register","id":"req-2",\
        "name":"test-exec","actions":["do-thing"]}""", true);

    assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
    assertThat(messages).hasSize(1);
    assertThat(messages.get(0)).contains("\"op\":\"ack\"");
    assertThat(messages.get(0)).contains("\"id\":\"req-2\"");

    ws.sendClose(WebSocket.NORMAL_CLOSURE, "done").join();
}
```

- [ ] **Step 2: Run the test to verify it passes (it should pass even before the refactor, since the current endpoint handles executor-register directly)**

Run: `mvn test -pl examples/helpdesk -Dtest=PushEndpointTest -f examples/helpdesk/pom.xml`
Expected: PASS — confirms the test is valid against the current implementation.

- [ ] **Step 3: Modify HelpdeskPushEndpoint — remove ScenarioOrchestrator, add handler chain**

Replace the `ScenarioOrchestrator` injection with `Instance<PushRequestHandler>`. Remove the `ExecutorRegister` and `StepResult` switch cases. Route unhandled ops through the handler chain.

Modified `HelpdeskPushEndpoint.java`:

```java
package io.casehub.examples.helpdesk.push;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.PushMessage;
import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.PushRequestHandler;
import io.casehub.pages.push.StoredEvent;
import io.casehub.pages.push.TopicRegistry;
import io.quarkus.websockets.next.OnClose;
import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.OnTextMessage;
import io.quarkus.websockets.next.WebSocket;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import io.quarkus.websockets.next.UserData;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@WebSocket(path = "/push")
public class HelpdeskPushEndpoint {

    static final UserData.TypedKey<String> CONN_ID_KEY =
        new UserData.TypedKey<>("connId");

    @Inject ConnectionRegistry connectionRegistry;
    @Inject TopicRegistry topicRegistry;
    @Inject EventStore eventStore;
    @Inject Instance<PushRequestHandler> handlers;

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        String connId = UUID.randomUUID().toString();
        connection.userData().put(CONN_ID_KEY, connId);
        connectionRegistry.register(connId, connection);
    }

    @OnTextMessage
    void onMessage(WebSocketConnection connection, String message) {
        String connId = connection.userData().get(CONN_ID_KEY);
        PushRequest request = PushRequest.parse(message);

        switch (request) {
            case PushRequest.Listen listen -> {
                topicRegistry.listen(connId, listen.topics());

                List<String> gaps = new ArrayList<>();
                for (var entry : listen.since().entrySet()) {
                    List<StoredEvent> events = eventStore.replay(
                        entry.getKey(), entry.getValue(), 1000);
                    if (events.isEmpty() && entry.getValue() > 0) {
                        gaps.add(entry.getKey());
                    }
                    for (var stored : events) {
                        connection.sendTextAndAwait(
                            PushMessage.event(stored.topic(),
                                stored.payloadJson(), stored.seq()));
                    }
                }

                connection.sendTextAndAwait(
                    PushMessage.ack(listen.id(), listen.topics(), gaps));
            }
            case PushRequest.Unlisten unlisten -> {
                topicRegistry.unlisten(connId, unlisten.topics());
                connection.sendTextAndAwait(
                    PushMessage.ack(unlisten.id(), unlisten.topics(),
                        List.of()));
            }
            default -> {
                boolean handled = false;
                for (PushRequestHandler handler : handlers) {
                    if (handler.handles(request)) {
                        handler.handle(connId, request);
                        connection.sendTextAndAwait(
                            PushMessage.ack(request.id()));
                        handled = true;
                        break;
                    }
                }
                if (!handled) {
                    connection.sendTextAndAwait(
                        PushMessage.error(request.id(),
                            "Unsupported op: " + request.op()));
                }
            }
        }
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        String connId = connection.userData().get(CONN_ID_KEY);
        if (connId != null) {
            topicRegistry.removeConnection(connId);
            connectionRegistry.unregister(connId);
        }
    }
}
```

- [ ] **Step 4: Run all helpdesk tests to verify nothing broke**

Run: `mvn test -pl examples/helpdesk -f examples/helpdesk/pom.xml`
Expected: all tests PASS (including both `PushEndpointTest` tests).

- [ ] **Step 5: Run scenario-runtime tests to verify handler + orchestrator integration**

Run: `mvn test -pl backend/scenario-runtime -f backend/pom.xml`
Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/push/HelpdeskPushEndpoint.java
git add examples/helpdesk/src/test/java/io/casehub/examples/helpdesk/push/PushEndpointTest.java
git commit -m "feat(#343): wire handler chain into HelpdeskPushEndpoint

Remove direct ScenarioOrchestrator injection from HelpdeskPushEndpoint.
Extensible ops (executor-register, step-result) now route through
Instance<PushRequestHandler> — ScenarioPushHandler is auto-discovered
via CDI when scenario-runtime is on the classpath.

Endpoint only handles listen/unlisten directly (core push protocol).

Refs casehubio/casehub-pages#343"
```

## References

- [casehub-pages#343](https://github.com/casehubio/casehub-pages/issues/343) — focal issue
- `backend/push/src/main/java/io/casehub/pages/push/PushRequest.java` — sealed push request interface (7 variants)
- `backend/push/src/main/java/io/casehub/pages/push/CommandResultHandler.java` — existing single-variant handler pattern
- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/CdiCommandResultHandler.java` — CDI @DefaultBean handler pattern
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioOrchestrator.java` — orchestrator with `onExecutorRegister()` and `onStepResult()`
- `examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/push/HelpdeskPushEndpoint.java` — current hardcoded routing
- `docs/specs/2026-07-08-push-runtime-cdi-design.md` — push-runtime CDI design spec
- `specs/issue-408-scenario-engine/2026-08-20-distributed-executor-protocol-design.md` — dispatch protocol spec
