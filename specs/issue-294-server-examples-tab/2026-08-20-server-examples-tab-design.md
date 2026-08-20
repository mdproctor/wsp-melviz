# Server-Dependent Examples Tab in Showcase Gallery

**Issue:** #294
**Date:** 2026-08-20

## Problem

The examples gallery runs entirely client-side. With durable EventStore backends (#113) landed, there's no way to demonstrate push subscriptions, event replay, or durable persistence without a running server.

## Design

### 1. Gallery UI — Top-Level Tabs

Two tabs above the category sidebar: **Client** (default, existing samples) and **Server** (new).

**`samples.json` change:** Each category gains an optional `requiresServer: boolean` field. When absent or false, the category appears under the Client tab. When true, under the Server tab.

```json
{
  "category": "Live Data",
  "requiresServer": true,
  "samples": [...]
}
```

**Tab rendering** in `app.js`:
- Two tab buttons above `#categories-nav`
- Active tab filters `samplesData.categories` by `requiresServer`
- Tab state persisted in the URL hash prefix: `#server/Live%20Data/sample-path` vs `#client/Charts/sample-path` (backward compatible — bare `#Charts/sample-path` defaults to Client)

**Server connection indicator:** When the Server tab is active, a status dot in the tab bar. On tab switch to Server, fire a health check:
```javascript
fetch('/q/health/ready').then(r => r.ok ? setConnected() : setDisconnected())
                        .catch(() => setDisconnected());
```
Disconnected state shows: "Server not running — start with `docker compose up` in `examples/`"

**Sample count per tab:** The stats display updates to show the count for the active tab only.

### 2. Backend — `examples/server/`

Minimal Quarkus app. Dependencies:
- `casehub-pages-push-runtime` (CDI producers)
- `casehub-pages-push-store-jdbc` (Postgres durable store — overrides `@DefaultBean InMemoryEventStore`)
- `quarkus-websockets-next` (WebSocket endpoint for push protocol)
- `quarkus-jdbc-postgresql` + `quarkus-agroal` (Postgres connection)
- `io.quarkus:quarkus-redis-client` (Redis connection for `push-store-redis`)

**WebSocket endpoint** (`PushWebSocket.java`):
```java
@WebSocket(path = "/ws/push")
public class PushWebSocket {
    @Inject EventBroadcaster broadcaster;
    @Inject TopicRegistry registry;

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        registry.register(connection.id(), /* topics from handshake */);
    }

    @OnTextMessage
    void onMessage(WebSocketConnection connection, String message) {
        // Parse PushRequest, handle subscribe/unsubscribe/ack
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        registry.deregister(connection.id());
    }
}
```

The `SessionSender` CDI bean bridges WebSocket connections to the push protocol:
```java
@ApplicationScoped
public class WebSocketSessionSender implements SessionSender {
    @Inject WebSocketConnection connection; // or connection map
    public void send(String connectionId, String message) { ... }
}
```

**Demo data generator** (`DemoEventGenerator.java`):
- REST endpoint `POST /api/demo/generate` — generates a burst of events on configurable topics
- Scheduled timer `@Scheduled(every = "2s")` — continuous stream of demo events on `demo.events`, `demo.metrics`, `demo.alerts`
- Events are realistic: timestamps, random metrics, status changes, alert levels

**`application.properties`:**
```properties
quarkus.http.port=8090
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/pages_demo
quarkus.redis.hosts=redis://localhost:6379
casehub.pages.push.max-events-per-topic=500
```

### 3. Docker Compose — `examples/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pages_demo
      POSTGRES_USER: pages
      POSTGRES_PASSWORD: pages
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  server:
    build: ./server
    depends_on: [postgres, redis]
    ports: ["8090:8090"]
    environment:
      QUARKUS_DATASOURCE_JDBC_URL: jdbc:postgresql://postgres:5432/pages_demo
      QUARKUS_REDIS_HOSTS: redis://redis:6379
```

Consumer runs: `cd examples && docker compose up` — Quarkus + Postgres + Redis start together. Gallery serves from `yarn serve` on port 8080, proxies `/ws/push` and `/api/` to port 8090.

### 4. Server-Tab Example Dashboards

Four YAML samples in `examples/samples/Server/`:

**4a. Live Event Stream** (`live-events.yaml`)
- Subscribes to `demo.events` via WebSocket push source
- Shows an event timeline component (`pages-event-timeline`) with live-updating entries
- Metric tile showing event count and events/sec rate

**4b. Reconnect Replay** (`reconnect-replay.yaml`)
- Same `demo.events` subscription
- Dashboard includes a "Disconnect" button (action-button component) that closes the WebSocket
- On reconnect, EventStore replays missed events — timeline shows the catch-up
- Counter showing "replayed: N events"

**4c. Persistence Demo** (`persistence.yaml`)
- Subscribes to `demo.persistence` topic
- Shows EventStore state: total events stored, oldest event timestamp, storage backend (JDBC/Redis)
- Instructions: "Restart the server (`docker compose restart server`), then reload — events survive"

**4d. Multi-Topic Dashboard** (`multi-topic.yaml`)
- Wildcard subscription `demo.*` — receives events from all demo topics
- Cross-topic data table showing events from all topics with a topic column
- Selector component filtering by topic prefix
- Metric tiles per topic showing independent counts

### 5. Webpack Proxy Configuration

`examples/webpack.config.js` gains a `devServer.proxy` entry to forward API and WebSocket requests to the Quarkus backend:

```javascript
devServer: {
  proxy: [
    { context: ['/api/', '/ws/'], target: 'http://localhost:8090', ws: true }
  ]
}
```

This means `yarn serve` (port 8080) proxies `/ws/push` and `/api/demo/*` to the Quarkus server (port 8090). No CORS configuration needed.

## Package Changes

| Location | Change | New |
|----------|--------|-----|
| `examples/src/app.js` | Tab UI, server status indicator, tab-filtered categories | No |
| `examples/src/index.html` | Tab markup | No |
| `examples/src/styles.css` | Tab styles, status dot | No |
| `examples/samples.json` | `requiresServer` field on new categories | No |
| `examples/samples/Server/` | 4 YAML dashboards | Yes |
| `examples/server/` | Quarkus app (pom.xml, WebSocket, DemoEventGenerator) | Yes |
| `examples/docker-compose.yml` | Postgres + Redis + Quarkus | Yes |
| `examples/webpack.config.js` | Proxy config for API/WS | No |

## Testing

### Gallery UI
1. Tab switching shows correct categories (client vs server)
2. URL hash with `server/` prefix loads server tab
3. Backward-compatible: bare `#Charts/sample` still works
4. Server status indicator shows disconnected when backend not running
5. Sample count updates per tab

### Backend
6. WebSocket endpoint accepts push protocol messages
7. DemoEventGenerator produces events on scheduled interval
8. Events persist in Postgres via JdbcEventStore
9. EventStore replays events after reconnect

### Integration
10. `docker compose up` starts all services
11. Gallery with Server tab loads live data from backend
12. Disconnect/reconnect shows replay behavior

## Out of Scope

- Authentication on the demo server
- Production deployment configuration
- EventStore retention/cleanup policies (configurable via existing `max-events-per-topic`)

## References

- examples/src/app.js — current gallery application
- examples/samples.json — sample registry
- backend/push/src/main/java/io/casehub/pages/push/EventBroadcaster.java — broadcast engine
- backend/push/src/main/java/io/casehub/pages/push/EventStore.java — store SPI
- backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushProducers.java — CDI producers
- backend/push-store-jdbc/ — JDBC durable store
- backend/push-store-redis/ — Redis durable store
- backend/push/src/main/java/io/casehub/pages/push/SessionSender.java — WebSocket bridge SPI
- GitHub #113 — durable EventStore (landed)
- GitHub #294 — this issue
