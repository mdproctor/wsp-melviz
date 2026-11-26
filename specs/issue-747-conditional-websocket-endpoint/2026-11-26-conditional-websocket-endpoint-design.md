# Conditional @WebSocket endpoint mechanism for runtime modules

**Issue:** casehubio/casehub-pages#747
**Date:** 2026-11-26

## Summary

Provide a default `@WebSocket` push endpoint in `push-runtime` that apps
get for free. The endpoint is conditional via `@UnlessBuildProperty` and
delegates request handling to a `PushRequestHandler` chain (SPI already
exists). Three levels of customization: zero-config, add handlers, full
override.

## Problem

Quarkus WebSocket Next discovers `@WebSocket` classes via Jandex/Arc bean
scanning. Two `@WebSocket` classes at the same path cause a build failure.
Unlike `@DefaultBean` producers (which resolve injection conflicts),
`@DefaultBean` on a `@WebSocket` class has no effect — both endpoints
are registered and the path conflicts.

`@UnlessBuildProperty` works because Arc excludes the bean before the
WebSocket endpoint build step iterates `beanStream().classBeans()`.

## Design

### Default endpoint: `DefaultPushWebSocket`

New class in `push-runtime`:

```java
@WebSocket(path = "/ws/push")
@UnlessBuildProperty(
    name = "casehub.push.custom-endpoint",
    stringValue = "true")
public class DefaultPushWebSocket {

    @Inject Instance<PushRequestHandler> handlers;
    @Inject ConnectionRegistry registry;
    @Inject TopicRegistry topicRegistry;
    @Inject EventStore eventStore;

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        registry.add(connection.id(), connection);
    }

    @OnTextMessage
    void onMessage(WebSocketConnection connection, String message) {
        PushRequest request = PushRequest.parse(message);
        dispatch(connection.id(), connection, request);
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        topicRegistry.removeConnection(connection.id());
        registry.remove(connection.id());
    }

    private void dispatch(String connId, WebSocketConnection conn,
                          PushRequest request) {
        switch (request) {
            case PushRequest.Listen listen ->
                handleListen(conn, listen);
            case PushRequest.Unlisten unlisten ->
                handleUnlisten(conn, unlisten);
            default -> delegateToHandlers(connId, request);
        }
    }

    private void delegateToHandlers(String connId, PushRequest request) {
        for (PushRequestHandler h : handlers) {
            if (h.handles(request)) {
                h.handle(connId, request);
                return;
            }
        }
    }

    // ... standard listen/unlisten implementation
    // (identical to current PushWebSocket in examples)
}
```

### Handler chain

The `PushRequestHandler` SPI already exists in `backend/push`:

```java
public interface PushRequestHandler {
    boolean handles(PushRequest request);
    void handle(String connectionId, PushRequest request);
}
```

And already has an implementation — `PlaybookPushHandler` in
`playbook-runtime` handles `ExecutorRegister` and `StepResult`.

The default endpoint's `dispatch` handles the standard protocol
(Listen, Unlisten) directly, then delegates unknown request types to
the handler chain. Apps extend by adding their own
`@ApplicationScoped PushRequestHandler` beans — no endpoint override
needed.

### Three customization levels

| Level | What the app does | Build property | Endpoint |
|-------|-------------------|----------------|----------|
| **Zero config** | Nothing | Not set | `DefaultPushWebSocket` + standard protocol |
| **Add handlers** | Implement `PushRequestHandler` | Not set | `DefaultPushWebSocket` delegates to app handlers |
| **Full override** | Write own `@WebSocket`, set property | `casehub.push.custom-endpoint=true` | App's endpoint only |

### Example app changes

The example app's `PushWebSocket` can be deleted entirely. Its
app-specific handlers (Subscribe/Unsubscribe for `DatasetRegistry`,
`DemoEventGenerator`) become a `PushRequestHandler` implementation:

```java
@ApplicationScoped
public class DemoDatasetHandler implements PushRequestHandler {
    @Inject DatasetRegistry datasetRegistry;
    @Inject DemoEventGenerator eventGenerator;

    @Override
    public boolean handles(PushRequest request) {
        return request instanceof PushRequest.Subscribe
            || request instanceof PushRequest.Unsubscribe;
    }

    @Override
    public void handle(String connectionId, PushRequest request) {
        // ... subscribe/unsubscribe logic
    }
}
```

The example app's `WebSocketSessionSender` can also be deleted — it's
identical to the `@DefaultBean` version in `push-runtime`.

### OnClose handler chain

`OnClose` currently calls `topicRegistry.removeConnection()` and
`registry.remove()`. App-specific cleanup (e.g., `datasetRegistry
.removeConnection()`) needs a hook. Add a `PushConnectionListener`
interface:

```java
public interface PushConnectionListener {
    default void onConnect(String connectionId) {}
    default void onDisconnect(String connectionId) {}
}
```

The default endpoint injects `Instance<PushConnectionListener>` and
calls `onConnect`/`onDisconnect` during `@OnOpen`/`@OnClose`.
`DatasetRegistry` implements it to clean up subscriptions on disconnect.

### Package placement

| Component | Package | Module |
|-----------|---------|--------|
| `PushRequestHandler` | `io.casehub.pages.push` | `backend/push` (already exists) |
| `PushConnectionListener` | `io.casehub.pages.push` | `backend/push` (new interface) |
| `DefaultPushWebSocket` | `io.casehub.pages.push.runtime` | `backend/push-runtime` |

## Testing

- **Unit test:** `DefaultPushWebSocket` dispatches to handler chain
  correctly — mock `Instance<PushRequestHandler>`, verify delegation
- **Integration test:** `@QuarkusTest` verifying default endpoint starts
  when build property is not set
- **Override test:** `@QuarkusTest` with `@TestProfile` setting
  `casehub.push.custom-endpoint=true` and providing an app endpoint —
  verify no path conflict
- **Handler chain test:** Verify `PlaybookPushHandler` and a test
  handler both receive their respective request types

## References

- `backend/push/src/main/java/io/casehub/pages/push/PushRequestHandler.java` — existing SPI
- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/` — existing default beans
- `backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/PlaybookPushHandler.java` — existing handler implementation
- `examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java` — current boilerplate
- Quarkus WebSocket Next `WebSocketProcessor.collectEndpoints` — `beanStream().classBeans()` respects Arc exclusions
- `decisions.md D1` — approach decision
- GitHub #745 — push infrastructure default beans
- GitHub examples#212 — zero-Java goal
