# Conditional WebSocket Endpoint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #747 — Conditional WebSocket endpoint mechanism
**Issue group:** #747

**Goal:** Provide a default `@WebSocket` push endpoint in `push-runtime`
so apps get WebSocket push for free, with three customization levels:
zero-config, add handlers, full override.

**Architecture:** A new `DefaultPushWebSocket` class in `push-runtime`
handles the standard push protocol (listen/unlisten) and delegates
unknown request types to the existing `PushRequestHandler` chain. It is
conditional via `@UnlessBuildProperty(name="casehub.push.custom-endpoint",
stringValue="true")` so apps can supply their own endpoint. A new
`PushConnectionListener` SPI in `push` provides hooks for app-specific
connect/disconnect cleanup.

**Tech Stack:** Java 21, Quarkus (WebSocket Next, Arc CDI),
Jackson, JUnit 5, AssertJ

## Global Constraints

- `@UnlessBuildProperty` for conditionality, not `@DefaultBean` (D1
  decision — `@DefaultBean` has no effect on `@WebSocket` classes)
- Build property name: `casehub.push.custom-endpoint`
- WebSocket path: `/ws/push`
- `PushConnectionListener` interface goes in `backend/push` (SPI
  module), not `push-runtime`
- All new classes need Jandex indexing (already configured via
  `jandex-maven-plugin` in both modules)

---

## Batch 1: SPI + Default Endpoint

### Task 1: PushConnectionListener SPI interface

**Files:**
- Create: `backend/push/src/main/java/io/casehub/pages/push/PushConnectionListener.java`
- Test: `backend/push/src/test/java/io/casehub/pages/push/PushConnectionListenerTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `PushConnectionListener` interface with `default void onConnect(String connectionId)` and `default void onDisconnect(String connectionId)` — consumed by Task 2's `DefaultPushWebSocket`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.pages.push;

import org.junit.jupiter.api.Test;
import java.util.ArrayList;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class PushConnectionListenerTest {

    @Test
    void default_methods_are_no_ops() {
        PushConnectionListener listener = new PushConnectionListener() {};
        listener.onConnect("conn-1");
        listener.onDisconnect("conn-1");
        // No exception = default methods work as no-ops
    }

    @Test
    void implementation_receives_callbacks() {
        List<String> events = new ArrayList<>();
        PushConnectionListener listener = new PushConnectionListener() {
            @Override
            public void onConnect(String connectionId) {
                events.add("connect:" + connectionId);
            }
            @Override
            public void onDisconnect(String connectionId) {
                events.add("disconnect:" + connectionId);
            }
        };

        listener.onConnect("conn-1");
        listener.onDisconnect("conn-1");

        assertThat(events).containsExactly("connect:conn-1", "disconnect:conn-1");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f backend/push/pom.xml test -pl . -Dtest=PushConnectionListenerTest -Dsurefire.useFile=false`
Expected: FAIL — class `PushConnectionListener` does not exist

- [ ] **Step 3: Write the interface**

```java
package io.casehub.pages.push;

public interface PushConnectionListener {
    default void onConnect(String connectionId) {}
    default void onDisconnect(String connectionId) {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f backend/push/pom.xml test -pl . -Dtest=PushConnectionListenerTest -Dsurefire.useFile=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/push/src/main/java/io/casehub/pages/push/PushConnectionListener.java backend/push/src/test/java/io/casehub/pages/push/PushConnectionListenerTest.java
git commit -m "feat(push): add PushConnectionListener SPI interface Refs #747"
```

### Task 2: DefaultPushWebSocket with handler chain

**Files:**
- Create: `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/DefaultPushWebSocket.java`
- Test: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/DefaultPushWebSocketTest.java`

**Interfaces:**
- Consumes: `PushRequestHandler` (from `backend/push` — `handles(PushRequest)`, `handle(String, PushRequest)`), `ConnectionRegistry` (from `push-runtime` — `add(String, WebSocketConnection)`, `remove(String)`, `get(String)`), `TopicRegistry` (`listen()`, `unlisten()`, `removeConnection()`), `EventStore` (`replay()`, `append()`), `PushConnectionListener` (from Task 1 — `onConnect(String)`, `onDisconnect(String)`), `PushMessage` (`ack()`, `event()`)
- Produces: `DefaultPushWebSocket` class — `@WebSocket(path="/ws/push")` with `@UnlessBuildProperty`. Methods: `onOpen(WebSocketConnection)`, `onMessage(WebSocketConnection, String)`, `onClose(WebSocketConnection)`

- [ ] **Step 1: Write the unit test**

This is a unit test verifying dispatch logic. It mocks CDI `Instance<>` with plain lists.

```java
package io.casehub.pages.push.runtime;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.InMemoryEventStore;
import io.casehub.pages.push.PushConnectionListener;
import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.PushRequestHandler;
import io.casehub.pages.push.TopicRegistry;
import io.quarkus.websockets.next.WebSocketConnection;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;
import static org.mockito.Mockito.verify;

class DefaultPushWebSocketTest {

    private ConnectionRegistry registry;
    private TopicRegistry topicRegistry;
    private EventStore eventStore;
    private List<String> handledOps;
    private List<String> listenerEvents;
    private DefaultPushWebSocket endpoint;

    @BeforeEach
    void setUp() {
        registry = new ConnectionRegistry();
        topicRegistry = new TopicRegistry();
        eventStore = new InMemoryEventStore(100);
        handledOps = new ArrayList<>();
        listenerEvents = new ArrayList<>();

        PushRequestHandler testHandler = new PushRequestHandler() {
            @Override
            public boolean handles(PushRequest request) {
                return request instanceof PushRequest.Subscribe;
            }
            @Override
            public void handle(String connectionId, PushRequest request) {
                handledOps.add(request.op());
            }
        };

        PushConnectionListener testListener = new PushConnectionListener() {
            @Override
            public void onConnect(String connectionId) {
                listenerEvents.add("connect:" + connectionId);
            }
            @Override
            public void onDisconnect(String connectionId) {
                listenerEvents.add("disconnect:" + connectionId);
            }
        };

        endpoint = new DefaultPushWebSocket(
            List.of(testHandler), registry, topicRegistry,
            eventStore, List.of(testListener));
    }

    @Test
    void onOpen_registers_connection_and_notifies_listeners() {
        WebSocketConnection conn = mockConnection("c1");
        endpoint.onOpen(conn);
        assertThat(registry.get("c1")).isSameAs(conn);
        assertThat(listenerEvents).containsExactly("connect:c1");
    }

    @Test
    void onClose_removes_connection_and_notifies_listeners() {
        WebSocketConnection conn = mockConnection("c1");
        endpoint.onOpen(conn);
        listenerEvents.clear();

        endpoint.onClose(conn);

        assertThat(registry.get("c1")).isNull();
        assertThat(listenerEvents).containsExactly("disconnect:c1");
    }

    @Test
    void listen_request_is_handled_directly() {
        WebSocketConnection conn = mockConnection("c1");
        endpoint.onOpen(conn);

        String msg = "{\"op\":\"listen\",\"id\":\"r1\",\"topics\":[\"t1\"]}";
        endpoint.onMessage(conn, msg);

        assertThat(topicRegistry.connections("t1")).contains("c1");
    }

    @Test
    void unlisten_request_is_handled_directly() {
        WebSocketConnection conn = mockConnection("c1");
        endpoint.onOpen(conn);
        topicRegistry.listen("c1", List.of("t1"));

        String msg = "{\"op\":\"unlisten\",\"id\":\"r1\",\"topics\":[\"t1\"]}";
        endpoint.onMessage(conn, msg);

        assertThat(topicRegistry.connections("t1")).isEmpty();
    }

    @Test
    void unknown_request_delegates_to_handler_chain() {
        WebSocketConnection conn = mockConnection("c1");
        endpoint.onOpen(conn);

        String msg = "{\"op\":\"subscribe\",\"id\":\"r1\",\"dataset\":\"events\"}";
        endpoint.onMessage(conn, msg);

        assertThat(handledOps).containsExactly("subscribe");
    }

    @Test
    void unhandled_request_is_silently_ignored() {
        WebSocketConnection conn = mockConnection("c1");
        endpoint.onOpen(conn);

        // command-result with no handler registered for it
        String msg = "{\"op\":\"command-result\",\"id\":\"r1\",\"ok\":true}";
        endpoint.onMessage(conn, msg);

        assertThat(handledOps).isEmpty();
    }

    private WebSocketConnection mockConnection(String id) {
        WebSocketConnection conn = mock(WebSocketConnection.class);
        when(conn.id()).thenReturn(id);
        return conn;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f backend/push-runtime/pom.xml test -pl . -Dtest=DefaultPushWebSocketTest -Dsurefire.useFile=false`
Expected: FAIL — class `DefaultPushWebSocket` does not exist

- [ ] **Step 3: Write the implementation**

The class has two constructors: one CDI-injected (uses `Instance<>`), one test-friendly (takes `List<>`).

```java
package io.casehub.pages.push.runtime;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.PushConnectionListener;
import io.casehub.pages.push.PushMessage;
import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.PushRequestHandler;
import io.casehub.pages.push.StoredEvent;
import io.casehub.pages.push.TopicRegistry;
import io.quarkus.arc.properties.UnlessBuildProperty;
import io.quarkus.websockets.next.OnClose;
import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.OnTextMessage;
import io.quarkus.websockets.next.WebSocket;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.List;

@WebSocket(path = "/ws/push")
@UnlessBuildProperty(name = "casehub.push.custom-endpoint", stringValue = "true")
public class DefaultPushWebSocket {

    private final Iterable<PushRequestHandler> handlers;
    private final ConnectionRegistry registry;
    private final TopicRegistry topicRegistry;
    private final EventStore eventStore;
    private final Iterable<PushConnectionListener> listeners;

    @Inject
    public DefaultPushWebSocket(Instance<PushRequestHandler> handlers,
                                ConnectionRegistry registry,
                                TopicRegistry topicRegistry,
                                EventStore eventStore,
                                Instance<PushConnectionListener> listeners) {
        this.handlers = handlers;
        this.registry = registry;
        this.topicRegistry = topicRegistry;
        this.eventStore = eventStore;
        this.listeners = listeners;
    }

    DefaultPushWebSocket(List<PushRequestHandler> handlers,
                         ConnectionRegistry registry,
                         TopicRegistry topicRegistry,
                         EventStore eventStore,
                         List<PushConnectionListener> listeners) {
        this.handlers = handlers;
        this.registry = registry;
        this.topicRegistry = topicRegistry;
        this.eventStore = eventStore;
        this.listeners = listeners;
    }

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        registry.add(connection.id(), connection);
        for (PushConnectionListener l : listeners) {
            l.onConnect(connection.id());
        }
    }

    @OnTextMessage
    void onMessage(WebSocketConnection connection, String message) {
        PushRequest request = PushRequest.parse(message);
        dispatch(connection, request);
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        topicRegistry.removeConnection(connection.id());
        for (PushConnectionListener l : listeners) {
            l.onDisconnect(connection.id());
        }
        registry.remove(connection.id());
    }

    private void dispatch(WebSocketConnection conn, PushRequest request) {
        switch (request) {
            case PushRequest.Listen listen -> handleListen(conn, listen);
            case PushRequest.Unlisten unlisten -> handleUnlisten(conn, unlisten);
            default -> delegateToHandlers(conn.id(), request);
        }
    }

    private void handleListen(WebSocketConnection conn, PushRequest.Listen listen) {
        topicRegistry.listen(conn.id(), listen.topics());

        List<String> gaps = new ArrayList<>();
        for (var entry : listen.since().entrySet()) {
            String topic = entry.getKey();
            long sinceSeq = entry.getValue();
            List<StoredEvent> replayed = eventStore.replay(topic, sinceSeq, 500);
            if (replayed.isEmpty() && sinceSeq > 0) {
                gaps.add(topic);
            }
            for (StoredEvent event : replayed) {
                conn.sendTextAndAwait(
                    PushMessage.event(event.topic(), event.payloadJson(), event.seq()));
            }
        }

        conn.sendTextAndAwait(PushMessage.ack(listen.id(), listen.topics(), gaps));
    }

    private void handleUnlisten(WebSocketConnection conn, PushRequest.Unlisten unlisten) {
        topicRegistry.unlisten(conn.id(), unlisten.topics());
    }

    private void delegateToHandlers(String connId, PushRequest request) {
        for (PushRequestHandler h : handlers) {
            if (h.handles(request)) {
                h.handle(connId, request);
                return;
            }
        }
    }
}
```

- [ ] **Step 4: Check if Mockito is available as a test dependency**

Run: `grep -r 'mockito' backend/push-runtime/pom.xml backend/pom.xml`

If Mockito is not available, add it to `backend/push-runtime/pom.xml`:
```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
</dependency>
```

Alternatively, write a minimal stub instead of using Mockito — create a
static inner class in the test:

```java
private static class StubConnection implements WebSocketConnection {
    private final String id;
    StubConnection(String id) { this.id = id; }
    @Override public String id() { return id; }
    // ... remaining methods throw UnsupportedOperationException
}
```

Use whichever approach matches the existing test conventions in the
project.

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f backend/push-runtime/pom.xml test -pl . -Dtest=DefaultPushWebSocketTest -Dsurefire.useFile=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/DefaultPushWebSocket.java backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/DefaultPushWebSocketTest.java
git commit -m "feat(push-runtime): add DefaultPushWebSocket with handler chain Refs #747"
```

### Task 3: Integration test — default endpoint starts

**Files:**
- Create: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/DefaultPushWebSocketTest.java` (add integration tests to existing file, or create `DefaultPushWebSocketIT.java` depending on project convention)

**Interfaces:**
- Consumes: `DefaultPushWebSocket` (from Task 2), all CDI beans from `push-runtime`
- Produces: Integration test verifying the default endpoint is available when `casehub.push.custom-endpoint` is not set

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.pages.push.runtime;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DefaultPushWebSocketIT {

    @Inject
    DefaultPushWebSocket endpoint;

    @Test
    void default_endpoint_bean_is_resolvable() {
        assertThat(endpoint).isNotNull();
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f backend/push-runtime/pom.xml test -pl . -Dtest=DefaultPushWebSocketIT -Dsurefire.useFile=false`
Expected: PASS — the default endpoint is active because
`casehub.push.custom-endpoint` is not set

- [ ] **Step 3: Commit**

```bash
git add backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/DefaultPushWebSocketIT.java
git commit -m "test(push-runtime): integration test for DefaultPushWebSocket Refs #747"
```

## Batch 2: Example App Migration

### Task 4: DemoDatasetHandler + example app cleanup

**Files:**
- Create: `examples/server/src/main/java/io/casehub/pages/examples/DemoDatasetHandler.java`
- Create: `examples/server/src/main/java/io/casehub/pages/examples/DatasetConnectionListener.java`
- Delete: `examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java`
- Delete: `examples/server/src/main/java/io/casehub/pages/examples/WebSocketSessionSender.java`
- Delete: `examples/server/src/main/java/io/casehub/pages/examples/ConnectionRegistry.java`
- Test: `examples/server/src/test/java/io/casehub/pages/examples/DemoDatasetHandlerTest.java`

**Interfaces:**
- Consumes: `PushRequestHandler` (`handles(PushRequest)`, `handle(String, PushRequest)`), `PushConnectionListener` (`onDisconnect(String)`), `DatasetRegistry` (`subscribe()`, `unsubscribe()`, `removeConnection()`), `DemoEventGenerator` (`initialSnapshot(String)`), `ConnectionRegistry` from push-runtime (`get(String)`)
- Produces: `DemoDatasetHandler implements PushRequestHandler` — handles `Subscribe`/`Unsubscribe`; `DatasetConnectionListener implements PushConnectionListener` — calls `datasetRegistry.removeConnection()` on disconnect

- [ ] **Step 1: Write the failing test for DemoDatasetHandler**

```java
package io.casehub.pages.examples;

import io.casehub.pages.push.PushRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DemoDatasetHandlerTest {

    private DatasetRegistry datasetRegistry;
    private DemoDatasetHandler handler;

    @BeforeEach
    void setUp() {
        datasetRegistry = new DatasetRegistry();
        handler = new DemoDatasetHandler(datasetRegistry, null, null);
    }

    @Test
    void handles_subscribe_and_unsubscribe() {
        assertThat(handler.handles(
            new PushRequest.Subscribe("r1", "events", null))).isTrue();
        assertThat(handler.handles(
            new PushRequest.Unsubscribe("r1", "events"))).isTrue();
    }

    @Test
    void does_not_handle_listen() {
        assertThat(handler.handles(
            new PushRequest.Listen("r1", java.util.List.of("t"), null))).isFalse();
    }

    @Test
    void handle_subscribe_registers_connection() {
        handler.handle("c1",
            new PushRequest.Subscribe("r1", "events", null));
        assertThat(datasetRegistry.connections("events")).contains("c1");
    }

    @Test
    void handle_unsubscribe_removes_connection() {
        datasetRegistry.subscribe("c1", "events");
        handler.handle("c1",
            new PushRequest.Unsubscribe("r1", "events"));
        assertThat(datasetRegistry.connections("events")).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml test -pl . -Dtest=DemoDatasetHandlerTest -Dsurefire.useFile=false`
Expected: FAIL — class `DemoDatasetHandler` does not exist

- [ ] **Step 3: Write DemoDatasetHandler**

```java
package io.casehub.pages.examples;

import io.casehub.pages.push.PushMessage;
import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.PushRequestHandler;
import io.casehub.pages.push.runtime.ConnectionRegistry;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class DemoDatasetHandler implements PushRequestHandler {

    private final DatasetRegistry datasetRegistry;
    private final DemoEventGenerator eventGenerator;
    private final ConnectionRegistry connectionRegistry;

    @Inject
    public DemoDatasetHandler(DatasetRegistry datasetRegistry,
                              DemoEventGenerator eventGenerator,
                              ConnectionRegistry connectionRegistry) {
        this.datasetRegistry = datasetRegistry;
        this.eventGenerator = eventGenerator;
        this.connectionRegistry = connectionRegistry;
    }

    @Override
    public boolean handles(PushRequest request) {
        return request instanceof PushRequest.Subscribe
            || request instanceof PushRequest.Unsubscribe;
    }

    @Override
    public void handle(String connectionId, PushRequest request) {
        switch (request) {
            case PushRequest.Subscribe subscribe -> {
                datasetRegistry.subscribe(connectionId, subscribe.dataset());
                WebSocketConnection conn = connectionRegistry.get(connectionId);
                if (conn != null) {
                    conn.sendTextAndAwait(PushMessage.ack(subscribe.id()));
                    String snapshot = eventGenerator.initialSnapshot(subscribe.dataset());
                    if (snapshot != null) {
                        conn.sendTextAndAwait(snapshot);
                    }
                }
            }
            case PushRequest.Unsubscribe unsubscribe ->
                datasetRegistry.unsubscribe(connectionId, unsubscribe.dataset());
            default ->
                throw new IllegalArgumentException("Unhandled op: " + request.op());
        }
    }
}
```

- [ ] **Step 4: Write DatasetConnectionListener**

```java
package io.casehub.pages.examples;

import io.casehub.pages.push.PushConnectionListener;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class DatasetConnectionListener implements PushConnectionListener {

    private final DatasetRegistry datasetRegistry;

    @Inject
    public DatasetConnectionListener(DatasetRegistry datasetRegistry) {
        this.datasetRegistry = datasetRegistry;
    }

    @Override
    public void onDisconnect(String connectionId) {
        datasetRegistry.removeConnection(connectionId);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml test -pl . -Dtest=DemoDatasetHandlerTest -Dsurefire.useFile=false`
Expected: PASS

- [ ] **Step 6: Delete obsolete example files**

Delete `examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java` — replaced by `DefaultPushWebSocket` in push-runtime.

Delete `examples/server/src/main/java/io/casehub/pages/examples/WebSocketSessionSender.java` — duplicate of push-runtime's `@DefaultBean` version.

Delete `examples/server/src/main/java/io/casehub/pages/examples/ConnectionRegistry.java` — duplicate of push-runtime's `ConnectionRegistry`.

Update `DemoEventGenerator.java` to inject `io.casehub.pages.push.runtime.ConnectionRegistry` instead of `io.casehub.pages.examples.ConnectionRegistry` (same API, different package).

- [ ] **Step 7: Run full example server test suite**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml test -pl . -Dsurefire.useFile=false`
Expected: PASS — all existing tests still work

- [ ] **Step 8: Commit**

```bash
git add examples/server/src/main/java/io/casehub/pages/examples/DemoDatasetHandler.java examples/server/src/main/java/io/casehub/pages/examples/DatasetConnectionListener.java examples/server/src/main/java/io/casehub/pages/examples/DemoEventGenerator.java
git rm examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java examples/server/src/main/java/io/casehub/pages/examples/WebSocketSessionSender.java examples/server/src/main/java/io/casehub/pages/examples/ConnectionRegistry.java
git commit -m "feat(examples): migrate PushWebSocket to handler chain pattern Refs #747"
```

## Batch 3: Override Test + Full Suite Verification

### Task 5: Override integration test with `@TestProfile`

**Files:**
- Create: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/CustomEndpointOverrideTest.java`

**Interfaces:**
- Consumes: `DefaultPushWebSocket` (from Task 2) — verifies it is NOT present when build property is set
- Produces: Integration test proving the override mechanism works

- [ ] **Step 1: Write the override test**

This test uses a Quarkus test profile that sets the build property. When
the property is set, `DefaultPushWebSocket` should not be a bean.

```java
package io.casehub.pages.push.runtime;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(CustomEndpointOverrideTest.OverrideProfile.class)
class CustomEndpointOverrideTest {

    @Inject
    Instance<DefaultPushWebSocket> endpoint;

    @Test
    void default_endpoint_is_absent_when_property_set() {
        assertThat(endpoint.isUnsatisfied()).isTrue();
    }

    public static class OverrideProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("casehub.push.custom-endpoint", "true");
        }
    }
}
```

**Note:** `@UnlessBuildProperty` is a build-time annotation. Quarkus test
profiles that override config may or may not affect build-time
properties depending on the Quarkus version. If this test fails because
the bean is still present, the alternative verification approach is:

1. Add `quarkus.arc.exclude-types=io.casehub.pages.push.runtime.DefaultPushWebSocket` to the profile config
2. Or verify via documentation + manual testing that the property works

The implementer should check which approach Quarkus supports in this
project's version and adjust accordingly.

- [ ] **Step 2: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f backend/push-runtime/pom.xml test -pl . -Dtest=CustomEndpointOverrideTest -Dsurefire.useFile=false`
Expected: PASS

- [ ] **Step 3: Run full push-runtime test suite**

Run: `/opt/homebrew/bin/mvn -f backend/push-runtime/pom.xml test -pl . -Dsurefire.useFile=false`
Expected: PASS — all existing tests still pass alongside new ones

- [ ] **Step 4: Commit**

```bash
git add backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/CustomEndpointOverrideTest.java
git commit -m "test(push-runtime): verify custom endpoint override via build property Refs #747"
```

### Task 6: Full backend build verification

**Files:**
- No new files — verification only

**Interfaces:**
- Consumes: all changes from Tasks 1-5
- Produces: confidence that the full backend compiles and tests pass

- [ ] **Step 1: Run full backend build**

Run: `/opt/homebrew/bin/mvn -f backend/pom.xml clean test -Dsurefire.useFile=false`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 2: Run example server build**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml clean test -Dsurefire.useFile=false`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit if any fixups were needed**

Only commit if compilation or tests revealed issues that required fixes.

## References

- [2026-11-26-conditional-websocket-endpoint-design.md] — design spec this plan implements
- [decisions.md D1] — `@UnlessBuildProperty` approach decision
- [backend/push/src/main/java/io/casehub/pages/push/PushRequestHandler.java] — existing SPI
- [backend/push/src/main/java/io/casehub/pages/push/PushRequest.java] — sealed request types
- [backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/ConnectionRegistry.java] — existing registry
- [backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushProducers.java] — existing CDI producers
- [backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/PlaybookPushHandler.java] — existing handler implementation
- [examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java] — boilerplate being replaced
- [GitHub #747] — focal issue
- [GitHub #745] — push infrastructure default beans (prior work)
