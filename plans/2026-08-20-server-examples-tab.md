# Server-Dependent Examples Tab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #294 — feat: server-dependent examples tab in showcase gallery
**Issue group:** #294

**Goal:** Add a "Server" tab to the examples gallery with a Docker Compose backend demonstrating push subscriptions, event replay, and durable persistence.

**Architecture:** Gallery UI gains top-level Client/Server tabs filtering categories by `requiresServer`. A minimal Quarkus app in `examples/server/` wires push-runtime + push-store-jdbc with a WebSocket endpoint bridging the push SDK. Docker Compose starts Postgres + Quarkus. Four YAML dashboards demonstrate live events, reconnect replay, persistence, and multi-topic.

**Tech Stack:** JavaScript (gallery UI), Java 17 / Quarkus (backend), Docker Compose, YAML (dashboards), Webpack (proxy)

## Global Constraints

- Topic delimiter is `:` (not `.`) — `demo:events`, `demo:*`
- Push protocol operations: `listen`/`unlisten` (not subscribe/unsubscribe) for event topics
- `TopicRegistry.listen(connectionId, List<String> topics)`, `.unlisten(connectionId, topics)`, `.removeConnection(connectionId)`
- `EventStore.replay(topic, sinceSeq, limit)` returns `List<StoredEvent>`
- `PushRequest.parse(message)` returns sealed interface: `Listen`, `Unlisten`, `Subscribe`, `Unsubscribe`, `CommandResult`
- `PushMessage.event(topic, payloadJson, seq)`, `.ack(id, topics, gaps)` for server→client messages
- JDBC-only backend — no Redis (avoids CDI priority conflict)

---

## Batch 1: Gallery UI — Client/Server tabs

### Task 1: Add tab switching to gallery UI

**Files:**
- Modify: `examples/src/index.html` — add tab markup above sidebar
- Modify: `examples/src/styles.css` — add tab styles and status indicator
- Modify: `examples/src/app.js` — tab filtering, status check, hash routing
- Modify: `examples/samples.json` — no changes yet (Server category added in Batch 4)

**Interfaces:**
- Consumes: `samplesData.categories[].requiresServer` (optional boolean in samples.json)
- Produces: Tab-filtered category rendering, URL hash routing with `server/` prefix, `checkServerHealth()` function

- [ ] **Step 1: Add tab HTML to index.html**

In `examples/src/index.html`, add tab bar inside sidebar before the search box:

```html
<div class="gallery-tabs" id="gallery-tabs">
    <button class="gallery-tab active" data-tab="client">Client</button>
    <button class="gallery-tab" data-tab="server">Server
        <span class="server-status" id="server-status"></span>
    </button>
</div>
```

Insert after `<div class="header-toggles">...</div>` and before `<div class="search-box">`.

- [ ] **Step 2: Add tab and status styles to styles.css**

Append to `examples/src/styles.css`:

```css
.gallery-tabs {
  display: flex;
  gap: 2px;
  padding: 0 16px 8px;
  border-bottom: 1px solid var(--pages-neutral-4, #444);
  margin-bottom: 8px;
}
.gallery-tab {
  flex: 1;
  padding: 6px 12px;
  border: 1px solid var(--pages-neutral-4, #555);
  border-bottom: none;
  border-radius: 6px 6px 0 0;
  background: transparent;
  color: var(--pages-text-secondary, #999);
  cursor: pointer;
  font-size: 13px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}
.gallery-tab.active {
  background: var(--pages-neutral-2, #333);
  color: var(--pages-text-primary, #eee);
  border-color: var(--pages-neutral-3, #666);
}
.server-status {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #666;
  display: inline-block;
}
.server-status.connected { background: #16a34a; }
.server-status.disconnected { background: #dc2626; }
.server-banner {
  padding: 12px 16px;
  background: var(--pages-neutral-2, #2a2a2a);
  border: 1px solid var(--pages-neutral-4, #555);
  border-radius: 6px;
  margin: 8px 16px;
  font-size: 13px;
  color: var(--pages-text-secondary, #aaa);
}
.server-banner code {
  background: var(--pages-neutral-3, #3a3a3a);
  padding: 2px 6px;
  border-radius: 3px;
  font-size: 12px;
}
```

- [ ] **Step 3: Implement tab logic in app.js**

Add these variables after the existing DOM element declarations in `app.js`:

```javascript
let activeTab = 'client';
const galleryTabs = document.getElementById('gallery-tabs');
const serverStatus = document.getElementById('server-status');
```

Add tab switching function:

```javascript
function setActiveTab(tab) {
    activeTab = tab;
    document.querySelectorAll('.gallery-tab').forEach(t => {
        t.classList.toggle('active', t.dataset.tab === tab);
    });
    renderCategories();
    renderStats();
    if (tab === 'server') {
        checkServerHealth();
    }
}

function checkServerHealth() {
    fetch('/api/demo/health')
        .then(r => r.ok ? r.json() : Promise.reject())
        .then(() => {
            serverStatus.className = 'server-status connected';
            removeBanner();
        })
        .catch(() => {
            serverStatus.className = 'server-status disconnected';
            showBanner();
        });
}

function showBanner() {
    removeBanner();
    const banner = document.createElement('div');
    banner.className = 'server-banner';
    banner.id = 'server-banner';
    banner.innerHTML = 'Server not running — start with <code>docker compose up</code> in <code>examples/</code>';
    categoriesNav.parentElement.insertBefore(banner, categoriesNav);
}

function removeBanner() {
    document.getElementById('server-banner')?.remove();
}
```

- [ ] **Step 4: Modify renderCategories to filter by active tab**

Replace the existing `renderCategories()` function body to filter categories:

```javascript
function renderCategories() {
    categoriesNav.innerHTML = '';
    const filtered = samplesData.categories.filter(c =>
        activeTab === 'server' ? c.requiresServer === true : !c.requiresServer
    );

    filtered.forEach(category => {
        // ... existing category rendering code (unchanged)
    });
}
```

- [ ] **Step 5: Modify renderStats to count per tab**

Update `renderStats()` to count samples for the active tab:

```javascript
function renderStats() {
    const filtered = samplesData.categories.filter(c =>
        activeTab === 'server' ? c.requiresServer === true : !c.requiresServer
    );
    const totalSamples = filtered.reduce((sum, c) => sum + c.samples.length, 0);
    const categoryCount = filtered.length;
    // ... existing stat rendering with these counts
}
```

- [ ] **Step 6: Update hash routing for tab prefix**

Modify `initializeApp()` to parse tab from hash:

```javascript
const hash = window.location.hash.slice(1);
if (hash) {
    if (hash.startsWith('server/')) {
        setActiveTab('server');
        const rest = hash.slice(7);
        const [category, samplePath] = rest.split('/');
        loadSampleFromHash(category, samplePath);
    } else if (hash.startsWith('client/')) {
        setActiveTab('client');
        const rest = hash.slice(7);
        const [category, samplePath] = rest.split('/');
        loadSampleFromHash(category, samplePath);
    } else {
        // backward compat — bare hash defaults to client
        const [category, samplePath] = hash.split('/');
        loadSampleFromHash(category, samplePath);
    }
}
```

Update `loadSample()` to include tab prefix in hash:

```javascript
window.location.hash = `${activeTab}/${sample.category}/${encodeURIComponent(sample.path)}`;
```

- [ ] **Step 7: Wire tab click handlers in setupEventListeners**

Add in `setupEventListeners()`:

```javascript
galleryTabs.addEventListener('click', (e) => {
    const tab = e.target.closest('.gallery-tab');
    if (tab) setActiveTab(tab.dataset.tab);
});
```

- [ ] **Step 8: Verify manually — yarn serve shows two tabs, client tab has existing samples, server tab is empty**

Run: `cd examples && GH_PACKAGES_TOKEN=dummy yarn serve`
Expected: Gallery loads with Client/Server tabs. Client tab shows all categories. Server tab shows empty (no `requiresServer` categories yet). Server status dot is red.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/src/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(examples): add Client/Server tab switching to gallery

Tab-filtered category sidebar with server health indicator.
URL hash routing with server/ prefix. Backward-compatible
bare hashes default to Client tab.

Refs #294"
```

## Batch 2: Backend server — Quarkus app with push WebSocket

### Task 2: Create minimal Quarkus server with push WebSocket endpoint

**Files:**
- Create: `examples/server/pom.xml`
- Create: `examples/server/src/main/java/io/casehub/pages/examples/ConnectionRegistry.java`
- Create: `examples/server/src/main/java/io/casehub/pages/examples/WebSocketSessionSender.java`
- Create: `examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java`
- Create: `examples/server/src/main/java/io/casehub/pages/examples/DemoEventGenerator.java`
- Create: `examples/server/src/main/java/io/casehub/pages/examples/DemoResource.java`
- Create: `examples/server/src/main/resources/application.properties`
- Create: `examples/server/src/main/resources/db/migration/V1__create_event_store.sql`
- Test: `examples/server/src/test/java/io/casehub/pages/examples/PushWebSocketTest.java`
- Test: `examples/server/src/test/java/io/casehub/pages/examples/DemoResourceTest.java`

**Interfaces:**
- Consumes: `TopicRegistry.listen(connId, topics)`, `.unlisten(connId, topics)`, `.removeConnection(connId)`, `EventStore.replay(topic, sinceSeq, limit)`, `EventBroadcaster.broadcast(topic, payloadJson)`, `PushRequest.parse(message)`, `PushMessage.event(topic, payload, seq)`, `PushMessage.ack(id, topics, gaps)`, `SessionSender.send(connId, message)`
- Produces: WebSocket endpoint at `/ws/push`, REST endpoints at `/api/demo/health`, `/api/demo/info`, `POST /api/demo/generate`

- [ ] **Step 1: Create pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
        <relativePath/>
    </parent>

    <artifactId>casehub-pages-examples-server</artifactId>
    <version>0.2-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>CaseHub Pages Examples Server</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-push-runtime</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-push-store-jdbc</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-websockets-next</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-jackson</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-scheduler</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-flyway</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <extensions>true</extensions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Create ConnectionRegistry — shared connection map**

`examples/server/src/main/java/io/casehub/pages/examples/ConnectionRegistry.java`:

```java
package io.casehub.pages.examples;

import io.quarkus.websocket.next.WebSocketConnection;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ConnectionRegistry {
    private final ConcurrentHashMap<String, WebSocketConnection> connections = new ConcurrentHashMap<>();

    public void add(String id, WebSocketConnection connection) {
        connections.put(id, connection);
    }

    public void remove(String id) {
        connections.remove(id);
    }

    public WebSocketConnection get(String id) {
        return connections.get(id);
    }
}
```

- [ ] **Step 3: Create WebSocketSessionSender**

`examples/server/src/main/java/io/casehub/pages/examples/WebSocketSessionSender.java`:

```java
package io.casehub.pages.examples;

import io.casehub.pages.push.SessionSender;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class WebSocketSessionSender implements SessionSender {
    @Inject ConnectionRegistry registry;

    @Override
    public void send(String connectionId, String message) {
        var conn = registry.get(connectionId);
        if (conn != null) {
            conn.sendTextAndAwait(message);
        }
    }
}
```

- [ ] **Step 4: Create PushWebSocket endpoint**

`examples/server/src/main/java/io/casehub/pages/examples/PushWebSocket.java`:

```java
package io.casehub.pages.examples;

import io.casehub.pages.push.*;
import io.quarkus.websocket.next.*;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.List;

@WebSocket(path = "/ws/push")
public class PushWebSocket {
    @Inject TopicRegistry topicRegistry;
    @Inject EventStore eventStore;
    @Inject ConnectionRegistry connectionRegistry;

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        connectionRegistry.add(connection.id(), connection);
    }

    @OnTextMessage
    void onMessage(WebSocketConnection connection, String message) {
        PushRequest request = PushRequest.parse(message);
        switch (request) {
            case PushRequest.Listen listen -> handleListen(connection, listen);
            case PushRequest.Unlisten unlisten -> handleUnlisten(connection, unlisten);
            default -> { }
        }
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        topicRegistry.removeConnection(connection.id());
        connectionRegistry.remove(connection.id());
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
}
```

- [ ] **Step 5: Create DemoEventGenerator**

`examples/server/src/main/java/io/casehub/pages/examples/DemoEventGenerator.java`:

```java
package io.casehub.pages.examples;

import io.casehub.pages.push.EventBroadcaster;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.concurrent.ThreadLocalRandom;

@ApplicationScoped
public class DemoEventGenerator {
    @Inject EventBroadcaster broadcaster;

    private static final String[] EVENT_TYPES = {"claim.created", "claim.updated", "claim.resolved", "alert.triggered", "metric.recorded"};
    private static final String[] SEVERITIES = {"info", "warning", "critical"};

    @Scheduled(every = "2s")
    void generateEvents() {
        var rng = ThreadLocalRandom.current();
        String type = EVENT_TYPES[rng.nextInt(EVENT_TYPES.length)];
        String severity = SEVERITIES[rng.nextInt(SEVERITIES.length)];
        double value = Math.round(rng.nextDouble() * 1000.0) / 10.0;

        String payload = String.format(
            "{\"type\":\"%s\",\"severity\":\"%s\",\"value\":%.1f,\"timestamp\":\"%s\"}",
            type, severity, value, Instant.now());

        broadcaster.broadcast("demo:events", payload);

        if (rng.nextInt(3) == 0) {
            broadcaster.broadcast("demo:metrics", String.format(
                "{\"cpu\":%.1f,\"memory\":%.1f,\"timestamp\":\"%s\"}",
                rng.nextDouble() * 100, rng.nextDouble() * 100, Instant.now()));
        }

        if (rng.nextInt(5) == 0) {
            broadcaster.broadcast("demo:alerts", String.format(
                "{\"level\":\"%s\",\"message\":\"Demo alert %d\",\"timestamp\":\"%s\"}",
                severity, rng.nextInt(1000), Instant.now()));
        }

        broadcaster.broadcast("demo:persistence", payload);
    }
}
```

- [ ] **Step 6: Create DemoResource — REST endpoints**

`examples/server/src/main/java/io/casehub/pages/examples/DemoResource.java`:

```java
package io.casehub.pages.examples;

import io.casehub.pages.push.EventBroadcaster;
import io.casehub.pages.push.EventStore;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import java.util.Map;

@Path("/api/demo")
@Produces(MediaType.APPLICATION_JSON)
public class DemoResource {
    @Inject EventBroadcaster broadcaster;
    @Inject EventStore eventStore;

    @GET
    @Path("/health")
    public Map<String, String> health() {
        return Map.of("status", "ok", "storeType", "jdbc");
    }

    @GET
    @Path("/info")
    public Map<String, Object> info() {
        var topics = eventStore.topics();
        long totalEvents = 0;
        for (String topic : topics) {
            var events = eventStore.replay(topic, 0, Integer.MAX_VALUE);
            totalEvents += events.size();
        }
        return Map.of(
            "storeType", "jdbc",
            "topicCount", topics.size(),
            "totalEvents", totalEvents,
            "topics", topics
        );
    }

    @POST
    @Path("/generate")
    public Map<String, Object> generate(@QueryParam("topic") @DefaultValue("demo:events") String topic,
                                         @QueryParam("count") @DefaultValue("10") int count) {
        for (int i = 0; i < count; i++) {
            broadcaster.broadcast(topic,
                String.format("{\"burst\":true,\"index\":%d,\"total\":%d}", i + 1, count));
        }
        return Map.of("generated", count, "topic", topic);
    }
}
```

- [ ] **Step 7: Create application.properties**

`examples/server/src/main/resources/application.properties`:

```properties
quarkus.http.port=8090
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:8080

quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/pages_demo
quarkus.datasource.username=pages
quarkus.datasource.password=pages

quarkus.flyway.migrate-at-start=true

casehub.pages.push.max-events-per-topic=500
```

- [ ] **Step 8: Create Flyway migration for EventStore table**

Check what the JdbcEventStore expects:

```bash
grep -n "CREATE TABLE\|INSERT INTO\|SELECT.*FROM" /Users/mdproctor/claude/casehub/pages/backend/push-store-jdbc/src/main/resources/db/migration/*.sql 2>/dev/null
```

Copy the migration from `push-store-jdbc` to `examples/server/src/main/resources/db/migration/V1__create_event_store.sql`.

- [ ] **Step 9: Write DemoResource test**

`examples/server/src/test/java/io/casehub/pages/examples/DemoResourceTest.java`:

```java
package io.casehub.pages.examples;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class DemoResourceTest {
    @Test
    void healthEndpointReturnsOk() {
        given().when().get("/api/demo/health")
            .then().statusCode(200)
            .body("status", is("ok"))
            .body("storeType", is("jdbc"));
    }

    @Test
    void infoEndpointReturnStoreMetadata() {
        given().when().get("/api/demo/info")
            .then().statusCode(200)
            .body("storeType", is("jdbc"))
            .body("topicCount", greaterThanOrEqualTo(0));
    }

    @Test
    void generateEndpointProducesEvents() {
        given().queryParam("topic", "test:burst").queryParam("count", "5")
            .when().post("/api/demo/generate")
            .then().statusCode(200)
            .body("generated", is(5))
            .body("topic", is("test:burst"));
    }
}
```

- [ ] **Step 10: Build and test the server**

Run: `/opt/homebrew/bin/mvn -f examples/server/pom.xml compile`
Expected: BUILD SUCCESS

Note: Full `@QuarkusTest` tests require a running Postgres — they run via Docker Compose or with Testcontainers. The compile step verifies all wiring.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/server/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(examples): add Quarkus demo server with push WebSocket endpoint

Wires push-runtime + push-store-jdbc with a WebSocket endpoint that
handles Listen/Unlisten, replays from EventStore on reconnect, and
generates demo events on a 2s timer. REST endpoints for health,
info, and burst generation.

Refs #294"
```

## Batch 3: Docker Compose + Webpack proxy

### Task 3: Add Docker Compose and webpack devServer proxy

**Files:**
- Create: `examples/docker-compose.yml`
- Create: `examples/server/Dockerfile`
- Modify: `examples/webpack.config.js` — add devServer block

**Interfaces:**
- Consumes: Server from Task 2 on port 8090
- Produces: `docker compose up` starts Postgres + Quarkus; `yarn serve` proxies `/api/` and `/ws/` to 8090

- [ ] **Step 1: Create Dockerfile for Quarkus**

`examples/server/Dockerfile`:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src src
RUN mvn package -DskipTests -Dquarkus.package.jar.type=uber-jar

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*-runner.jar app.jar
EXPOSE 8090
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- [ ] **Step 2: Create docker-compose.yml**

`examples/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pages_demo
      POSTGRES_USER: pages
      POSTGRES_PASSWORD: pages
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pages -d pages_demo"]
      interval: 5s
      timeout: 3s
      retries: 5

  server:
    build: ./server
    depends_on:
      postgres:
        condition: service_healthy
    ports: ["8090:8090"]
    environment:
      QUARKUS_DATASOURCE_JDBC_URL: jdbc:postgresql://postgres:5432/pages_demo
      QUARKUS_DATASOURCE_USERNAME: pages
      QUARKUS_DATASOURCE_PASSWORD: pages
```

- [ ] **Step 3: Add devServer block to webpack.config.js**

Add `devServer` property to the returned config object in `examples/webpack.config.js`, before the closing `};`:

```javascript
devServer: {
    port: 8080,
    proxy: [
        {
            context: ['/api/', '/ws/'],
            target: 'http://localhost:8090',
            ws: true,
            changeOrigin: true,
        }
    ],
    historyApiFallback: true,
},
```

- [ ] **Step 4: Verify webpack serves with proxy config**

Run: `cd examples && GH_PACKAGES_TOKEN=dummy yarn serve`
Expected: Webpack dev server starts on port 8080 without errors. Proxy config accepted.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/docker-compose.yml examples/server/Dockerfile examples/webpack.config.js
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(examples): Docker Compose + webpack proxy for demo server

Postgres + Quarkus via docker compose up. Webpack devServer proxies
/api/ and /ws/ to port 8090.

Refs #294"
```

## Batch 4: Server-tab example dashboards

### Task 4: Add four server-dependent YAML dashboards

**Files:**
- Create: `examples/samples/Server/live-events.dash.yaml`
- Create: `examples/samples/Server/reconnect-replay.dash.yaml`
- Create: `examples/samples/Server/reconnect-replay.ts`
- Create: `examples/samples/Server/persistence.dash.yaml`
- Create: `examples/samples/Server/multi-topic.dash.yaml`
- Modify: `examples/samples.json` — add Server category with `requiresServer: true`

**Interfaces:**
- Consumes: Push WebSocket at `/ws/push`, REST at `/api/demo/info`, `/api/demo/generate`
- Produces: 4 working dashboard samples visible in the Server tab

- [ ] **Step 1: Create live-events.dash.yaml**

`examples/samples/Server/live-events.dash.yaml`:

```yaml
properties:
  pushUrl: ws://localhost:8090/ws/push

datasets:
  - id: events
    push:
      url: "${pushUrl}"
      topics: ["demo:events"]

pages:
  - name: Live Event Stream
    rows:
      - columns:
          - components:
              - type: metric
                lookup: { dataSetId: events }
                props:
                  label: "Live Events"
                  value: "#{datasets.events.rowCount}"
          - components:
              - type: event-timeline
                lookup: { dataSetId: events }
```

- [ ] **Step 2: Create reconnect-replay.dash.yaml and companion TS**

`examples/samples/Server/reconnect-replay.dash.yaml`:

```yaml
properties:
  pushUrl: ws://localhost:8090/ws/push

datasets:
  - id: events
    push:
      url: "${pushUrl}"
      topics: ["demo:events"]

pages:
  - name: Reconnect Replay Demo
    rows:
      - columns:
          - components:
              - type: title
                props: { text: "Reconnect Replay Demo", size: "h2" }
              - type: html
                props:
                  content: "<p>Click Disconnect to simulate a network drop. Events continue arriving on the server. Click Reconnect to see missed events replayed from the EventStore.</p>"
      - columns:
          - span: 4
            components:
              - type: metric
                lookup: { dataSetId: events }
                props:
                  label: "Total Events"
                  value: "#{datasets.events.rowCount}"
          - span: 8
            components:
              - type: event-timeline
                lookup: { dataSetId: events }
```

`examples/samples/Server/reconnect-replay.ts`:

```typescript
// Companion script — adds disconnect/reconnect controls
const target = document.getElementById('sample-target');
if (target) {
    const controls = document.createElement('div');
    controls.style.cssText = 'padding: 8px 16px; display: flex; gap: 8px;';
    controls.innerHTML = `
        <button id="demo-disconnect" style="padding: 6px 16px; background: #dc2626; color: white; border: none; border-radius: 4px; cursor: pointer;">Disconnect</button>
        <button id="demo-reconnect" style="padding: 6px 16px; background: #16a34a; color: white; border: none; border-radius: 4px; cursor: pointer;" disabled>Reconnect</button>
        <span id="demo-status" style="padding: 6px; color: #16a34a;">● Connected</span>
    `;
    target.prepend(controls);
}
```

- [ ] **Step 3: Create persistence.dash.yaml**

`examples/samples/Server/persistence.dash.yaml`:

```yaml
properties:
  pushUrl: ws://localhost:8090/ws/push
  apiBase: http://localhost:8090

datasets:
  - id: storeInfo
    url: "${apiBase}/api/demo/info"
  - id: events
    push:
      url: "${pushUrl}"
      topics: ["demo:persistence"]

pages:
  - name: Durable Persistence Demo
    rows:
      - columns:
          - components:
              - type: title
                props: { text: "Durable EventStore", size: "h2" }
              - type: html
                props:
                  content: "<p>Events survive server restarts. Try: <code>docker compose restart server</code>, then reload this page.</p>"
      - columns:
          - span: 4
            components:
              - type: metric
                lookup: { dataSetId: storeInfo }
                props:
                  label: "Store Type"
                  value: "#{datasets.storeInfo.first.storeType}"
          - span: 4
            components:
              - type: metric
                lookup: { dataSetId: storeInfo }
                props:
                  label: "Total Events"
                  value: "#{datasets.storeInfo.first.totalEvents}"
          - span: 4
            components:
              - type: metric
                lookup: { dataSetId: storeInfo }
                props:
                  label: "Topics"
                  value: "#{datasets.storeInfo.first.topicCount}"
      - columns:
          - components:
              - type: event-timeline
                lookup: { dataSetId: events }
```

- [ ] **Step 4: Create multi-topic.dash.yaml**

`examples/samples/Server/multi-topic.dash.yaml`:

```yaml
properties:
  pushUrl: ws://localhost:8090/ws/push

datasets:
  - id: allEvents
    push:
      url: "${pushUrl}"
      topics: ["demo:**"]

pages:
  - name: Multi-Topic Dashboard
    rows:
      - columns:
          - components:
              - type: title
                props: { text: "Multi-Topic Dashboard", size: "h2" }
              - type: html
                props:
                  content: "<p>Wildcard subscription <code>demo:**</code> — receives events from all demo topics.</p>"
      - columns:
          - components:
              - type: data-table
                lookup: { dataSetId: allEvents }
```

- [ ] **Step 5: Update samples.json with Server category**

Add to the `categories` array in `examples/samples.json`:

```json
{
    "category": "Server",
    "requiresServer": true,
    "samples": [
        {
            "name": "Live Event Stream",
            "path": "Server/live-events.dash.yaml",
            "category": "Server",
            "file": "live-events.dash.yaml"
        },
        {
            "name": "Reconnect Replay",
            "path": "Server/reconnect-replay.dash.yaml",
            "category": "Server",
            "file": "reconnect-replay.dash.yaml",
            "tsPath": "Server/reconnect-replay.ts"
        },
        {
            "name": "Persistence Demo",
            "path": "Server/persistence.dash.yaml",
            "category": "Server",
            "file": "persistence.dash.yaml"
        },
        {
            "name": "Multi-Topic Dashboard",
            "path": "Server/multi-topic.dash.yaml",
            "category": "Server",
            "file": "multi-topic.dash.yaml"
        }
    ]
}
```

Update `totalSamples` count (+4).

- [ ] **Step 6: Verify gallery shows Server tab with 4 samples**

Run: `cd examples && GH_PACKAGES_TOKEN=dummy yarn serve`
Expected: Server tab shows "Server" category with 4 samples. Clicking a sample shows dashboard (with "server not running" if backend isn't up — uses galleryFetch fallback).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/samples/ examples/samples.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(examples): add 4 server-dependent dashboard samples

Live Event Stream, Reconnect Replay, Persistence Demo, and
Multi-Topic Dashboard. All use push WebSocket source with
demo:* topics. Server category with requiresServer: true.

Closes #294"
```

## References

- [2026-08-20-server-examples-tab-design.md] — design spec this plan implements
- [examples/src/app.js] — gallery application
- [examples/webpack.config.js] — webpack config (no devServer currently)
- [examples/samples.json] — sample registry
- [backend/push/src/main/java/io/casehub/pages/push/PushRequest.java:14] — sealed request types
- [backend/push/src/main/java/io/casehub/pages/push/PushMessage.java:12] — server→client builders
- [backend/push/src/main/java/io/casehub/pages/push/TopicRegistry.java:77] — listen(), unlisten(), removeConnection()
- [backend/push/src/main/java/io/casehub/pages/push/EventStore.java:15] — replay(), append(), topics()
- [backend/push/src/main/java/io/casehub/pages/push/StoredEvent.java:14] — event record
- [backend/push/src/main/java/io/casehub/pages/push/SessionSender.java:4] — functional interface
- [backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushProducers.java] — CDI producers
- [GitHub #294] — server-dependent examples tab
- [GitHub #113] — durable EventStore (landed)
