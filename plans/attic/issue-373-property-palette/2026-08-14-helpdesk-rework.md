# Helpdesk Rework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/parent#416 — Slice 1 rework — helpdesk on real engine/work/workers instead of hand-coded services
**Issue group:** #413, #416

**Goal:** Replace the helpdesk example's hand-coded lifecycle with platform-driven orchestration — CaseDefinition YAML, engine bindings, CallableDispatcher worker functions, and casehub-work human tasks.

**Architecture:** The current helpdesk example hand-codes everything the platform automates: `TicketService` manages lifecycle transitions, `TicketCreationHandler` does manual orchestration, `DemoTicketClassifier` is a bespoke SPI. The rework replaces this with a declarative CaseDefinition YAML that defines capabilities, workers with `call:` steps dispatched via `CallableDispatcher`, and contextChange/humanTask bindings. A `ReceivedMessage` CDI event triggers `CaseHub.startCase()`, the engine's choreography evaluates bindings to dispatch classification and notification workers, and a humanTask binding creates a casehub-work WorkItem for specialist resolution.

**Tech Stack:** casehub-engine (runtime, schema, rest, flow, persistence-memory), casehub-work (runtime, rest, persistence-memory, engine-adapter), Quarkus 3.32.2, Java 21

## Global Constraints

- All dependencies use `${casehub.version}` (0.2-SNAPSHOT)
- Workers use `call:` steps routed via `CasehubCallableTaskBuilder` → `CallableDispatchRegistry` → `CallableDispatcher`
- CaseContext is flat (no nested `ticket` object) — keeps JQ expressions simple
- In-memory persistence only (persistence-memory modules for both engine and work)
- No Quarkus profile gating on the dispatchers — they replace `DemoTicketClassifier` entirely
- `@IfBuildProfile("demo")` remains only on `DemoChatPlatform` and `DemoInboundTranslator`
- All existing push infrastructure (SSE/WebSocket) is preserved — observers adapt to engine events

---

### Task 1: Maven dependencies + CaseDefinition YAML + HelpDeskCaseHub

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml`
- Create: `/Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/src/main/resources/casehub/helpdesk-ticket.yaml`
- Create: `/Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/HelpDeskCaseHub.java`
- Test: `/Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/HelpDeskCaseHubTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.engine.YamlCaseHub`, `io.casehub.api.model.CaseDefinition`
- Produces: `HelpDeskCaseHub.getDefinition()` returning a fully-parsed `CaseDefinition`

- [ ] **Step 1: Add engine + work Maven dependencies**

Add to pom.xml `<dependencies>`:

```xml
<!-- Engine -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine</artifactId>
  <version>${casehub.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-schema</artifactId>
  <version>${casehub.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-rest</artifactId>
  <version>${casehub.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-flow</artifactId>
  <version>${casehub.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-persistence-memory</artifactId>
  <version>${casehub.version}</version>
</dependency>

<!-- Work -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-work</artifactId>
  <version>${casehub.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-work-rest</artifactId>
  <version>${casehub.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-work-persistence-memory</artifactId>
  <version>${casehub.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-work-engine-adapter</artifactId>
  <version>${casehub.version}</version>
</dependency>

<!-- Engine test support -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-testing</artifactId>
  <version>${casehub.version}</version>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Write CaseDefinition YAML**

Create `src/main/resources/casehub/helpdesk-ticket.yaml`:

```yaml
dsl: "1.0.0"
namespace: casehub-examples
name: helpdesk-ticket
version: "1.0.0"
title: IT Help Desk — Ticket Lifecycle
types:
  - support/helpdesk
labels:
  - example/slice-1

spec:
  capabilities:
    - name: classify-ticket
      description: "Classifies a ticket by category and priority using keyword matching"
      inputProjection: "{ text: .subject }"
      outputProjection: '{ category: .category, priority: .priority, status: "TRIAGED" }'

    - name: notify-customer
      description: "Sends a resolution notification to the customer"
      inputProjection: '{ customerRef: .customerRef, message: ("Your ticket has been resolved: " + .resolution) }'
      outputProjection: '{ notified: true }'

  workers:
    - name: keyword-classifier
      description: "Classifies tickets by keyword matching against bootstrapped data"
      capabilities: [ classify-ticket ]
      do:
        - classify:
            call: classify-ticket
            with:
              text: "${ .text }"

    - name: notification-sender
      description: "Sends customer notification via chat platform"
      capabilities: [ notify-customer ]
      do:
        - send:
            call: notify-customer
            with:
              customerRef: "${ .customerRef }"
              message: "${ .message }"

  bindings:
    - name: triage
      capability: classify-ticket
      on:
        contextChange:
          filter: '.status == "OPEN" and .category == null'

    - name: assign-specialist
      humanTask:
        titleExpression: '"Resolve ticket: " + .subject'
        candidateGroups: >-
          if .category == "HARDWARE" then ["hw-specialist"]
          elif .category == "SOFTWARE" then ["sw-specialist"]
          elif .category == "ACCESS" then ["access-specialist"]
          else ["general-specialist"]
          end
        scope: casehub-examples/helpdesk
        inputMapping: '{ subject: .subject, category: .category, priority: .priority, customerRef: .customerRef }'
        outputMapping: '{ status: "RESOLVED", resolution: .resolution, assigneeId: .assigneeId }'
      on:
        contextChange:
          filter: '.status == "TRIAGED" and .resolution == null'

    - name: notify-resolution
      capability: notify-customer
      on:
        contextChange:
          filter: '.status == "RESOLVED" and .notified == null'

  goals:
    - name: ticket-resolved
      description: "Ticket has been resolved and customer notified"
      condition: '.status == "RESOLVED" and .notified == true'
      kind: success

  completion:
    success:
      allOf:
        - ticket-resolved
```

- [ ] **Step 3: Write the failing test**

Create `HelpDeskCaseHubTest.java`:

```java
package io.casehub.examples.helpdesk.engine;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.CaseDefinition;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class HelpDeskCaseHubTest {

    @Inject
    HelpDeskCaseHub caseHub;

    @Test
    void definitionLoadsFromYaml() {
        CaseDefinition def = caseHub.getDefinition();
        assertNotNull(def);
        assertEquals("helpdesk-ticket", def.getName());
        assertEquals("casehub-examples", def.getNamespace());
    }

    @Test
    void definitionHasExpectedCapabilities() {
        CaseDefinition def = caseHub.getDefinition();
        assertEquals(2, def.getCapabilities().size());
        assertTrue(def.getCapabilities().stream()
                .anyMatch(c -> c.name().equals("classify-ticket")));
        assertTrue(def.getCapabilities().stream()
                .anyMatch(c -> c.name().equals("notify-customer")));
    }

    @Test
    void definitionHasExpectedBindings() {
        CaseDefinition def = caseHub.getDefinition();
        assertEquals(3, def.getBindings().size());
    }

    @Test
    void definitionHasExpectedWorkers() {
        CaseDefinition def = caseHub.getDefinition();
        assertEquals(2, def.getWorkers().size());
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=HelpDeskCaseHubTest -pl .`
Expected: Compilation failure — `HelpDeskCaseHub` does not exist

- [ ] **Step 5: Implement HelpDeskCaseHub**

Create `src/main/java/io/casehub/examples/helpdesk/engine/HelpDeskCaseHub.java`:

```java
package io.casehub.examples.helpdesk.engine;

import io.casehub.api.engine.YamlCaseHub;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class HelpDeskCaseHub extends YamlCaseHub {

    public HelpDeskCaseHub() {
        super("casehub/helpdesk-ticket.yaml");
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=HelpDeskCaseHubTest -pl .`
Expected: All 4 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples add helpdesk/pom.xml helpdesk/src/main/resources/casehub/helpdesk-ticket.yaml helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/HelpDeskCaseHub.java helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/HelpDeskCaseHubTest.java
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "feat(#416): CaseDefinition YAML + HelpDeskCaseHub — engine dependencies, declarative lifecycle
Refs #416"
```

---

### Task 2: KeywordClassifierDispatcher + NotificationDispatcher

**Files:**
- Create: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/KeywordClassifierDispatcher.java`
- Create: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/NotificationDispatcher.java`
- Create: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/HelpdeskDispatcherRegistrar.java`
- Test: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/KeywordClassifierDispatcherTest.java`
- Test: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/NotificationDispatcherTest.java`

**Interfaces:**
- Consumes: `io.casehub.engine.flow.CallableDispatcher`, `io.casehub.engine.flow.CallableDispatchRegistry`, `io.casehub.connectors.chat.spi.ChatPlatform`
- Produces: `KeywordClassifierDispatcher` (registered as `"classify-ticket"`), `NotificationDispatcher` (registered as `"notify-customer"`), `HelpdeskDispatcherRegistrar` (wires registrations at startup)

- [ ] **Step 1: Write failing test for KeywordClassifierDispatcher**

```java
package io.casehub.examples.helpdesk.engine;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ExecutionException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class KeywordClassifierDispatcherTest {

    private KeywordClassifierDispatcher dispatcher;

    @BeforeEach
    void setUp() {
        dispatcher = new KeywordClassifierDispatcher();
        dispatcher.loadClassifications(List.of(
                new KeywordClassifierDispatcher.ClassificationEntry("laptop", "HARDWARE", "HIGH"),
                new KeywordClassifierDispatcher.ClassificationEntry("password", "ACCESS", "LOW"),
                new KeywordClassifierDispatcher.ClassificationEntry("install", "SOFTWARE", "MEDIUM")));
    }

    @Test
    void classifiesMatchingKeyword() throws Exception {
        var result = dispatcher.dispatch("wf-1", Map.of("text", "My laptop won't boot")).get();
        assertEquals("HARDWARE", result.get("category"));
        assertEquals("HIGH", result.get("priority"));
    }

    @Test
    void fallsBackToOtherMedium() throws Exception {
        var result = dispatcher.dispatch("wf-1", Map.of("text", "Something unrelated")).get();
        assertEquals("OTHER", result.get("category"));
        assertEquals("MEDIUM", result.get("priority"));
    }

    @Test
    void caseInsensitiveMatching() throws Exception {
        var result = dispatcher.dispatch("wf-1", Map.of("text", "LAPTOP screen broken")).get();
        assertEquals("HARDWARE", result.get("category"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=KeywordClassifierDispatcherTest -pl .`
Expected: Compilation failure

- [ ] **Step 3: Implement KeywordClassifierDispatcher**

```java
package io.casehub.examples.helpdesk.engine;

import io.casehub.engine.flow.CallableDispatcher;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CopyOnWriteArrayList;

public class KeywordClassifierDispatcher implements CallableDispatcher {

    private final List<ClassificationEntry> entries = new CopyOnWriteArrayList<>();

    @Override
    public CompletableFuture<Map<String, Object>> dispatch(String workflowInstanceId,
                                                            Map<String, Object> args) {
        String text = String.valueOf(args.getOrDefault("text", "")).toLowerCase(Locale.ROOT);
        var match = entries.stream()
                .filter(e -> text.contains(e.match().toLowerCase(Locale.ROOT)))
                .findFirst();
        String category = match.map(ClassificationEntry::category).orElse("OTHER");
        String priority = match.map(ClassificationEntry::priority).orElse("MEDIUM");
        return CompletableFuture.completedFuture(Map.of("category", category, "priority", priority));
    }

    public void loadClassifications(List<ClassificationEntry> data) {
        entries.clear();
        entries.addAll(data);
    }

    public record ClassificationEntry(String match, String category, String priority) {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=KeywordClassifierDispatcherTest -pl .`
Expected: PASS

- [ ] **Step 5: Write failing test for NotificationDispatcher**

```java
package io.casehub.examples.helpdesk.engine;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.examples.helpdesk.NotificationService;
import java.util.Map;
import org.junit.jupiter.api.Test;

class NotificationDispatcherTest {

    @Test
    void dispatchSendsNotificationAndRecordsIt() throws Exception {
        var notificationService = new NotificationService(new StubChatPlatform());
        var dispatcher = new NotificationDispatcher(notificationService);

        var result = dispatcher.dispatch("wf-1",
                Map.of("customerRef", "alice", "message", "Resolved")).get();

        assertTrue((Boolean) result.get("notified"));
        assertEquals(1, notificationService.getSentNotifications().size());
        assertEquals("alice", notificationService.getSentNotifications().get(0).to());
    }
}
```

Note: `StubChatPlatform` — reuse or create a minimal stub that records sends. If `DemoChatPlatform` exists in the test classpath, use it. Otherwise create a test-only stub.

- [ ] **Step 6: Implement NotificationDispatcher**

```java
package io.casehub.examples.helpdesk.engine;

import io.casehub.engine.flow.CallableDispatcher;
import io.casehub.examples.helpdesk.NotificationService;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

public class NotificationDispatcher implements CallableDispatcher {

    private final NotificationService notificationService;

    public NotificationDispatcher(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @Override
    public CompletableFuture<Map<String, Object>> dispatch(String workflowInstanceId,
                                                            Map<String, Object> args) {
        String customerRef = (String) args.get("customerRef");
        String message = (String) args.get("message");
        notificationService.notify(customerRef, message);
        return CompletableFuture.completedFuture(Map.of("notified", true));
    }
}
```

- [ ] **Step 7: Run both tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest="KeywordClassifierDispatcherTest,NotificationDispatcherTest" -pl .`
Expected: PASS

- [ ] **Step 8: Implement HelpdeskDispatcherRegistrar**

Wires both dispatchers into `CallableDispatchRegistry` at startup:

```java
package io.casehub.examples.helpdesk.engine;

import io.casehub.engine.flow.CallableDispatchRegistry;
import io.casehub.examples.helpdesk.NotificationService;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class HelpdeskDispatcherRegistrar {

    @Inject CallableDispatchRegistry registry;
    @Inject NotificationService notificationService;

    private final KeywordClassifierDispatcher classifier = new KeywordClassifierDispatcher();

    @PostConstruct
    void register() {
        registry.register("classify-ticket", classifier);
        registry.register("notify-customer", new NotificationDispatcher(notificationService));
    }

    public KeywordClassifierDispatcher classifier() {
        return classifier;
    }
}
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples add helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/KeywordClassifierDispatcher.java helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/NotificationDispatcher.java helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/HelpdeskDispatcherRegistrar.java helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/KeywordClassifierDispatcherTest.java helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/NotificationDispatcherTest.java
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "feat(#416): CallableDispatcher worker functions — keyword classifier + notification sender
Refs #416"
```

---

### Task 3: ChatCaseCreationHandler — chat message to case creation

**Files:**
- Create: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/ChatCaseCreationHandler.java`
- Test: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/ChatCaseCreationHandlerTest.java`

**Interfaces:**
- Consumes: `io.casehub.connectors.chat.model.ReceivedMessage`, `HelpDeskCaseHub`
- Produces: `ChatCaseCreationHandler` — observes `ReceivedMessage`, calls `CaseHub.startCase(Map)`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.examples.helpdesk.engine;

import static org.awaitility.Awaitility.await;
import static org.junit.jupiter.api.Assertions.*;

import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

@QuarkusTest
class ChatCaseCreationHandlerTest {

    @Inject HelpDeskCaseHub caseHub;
    @Inject CaseInstanceCache caseInstanceCache;
    @Inject HelpdeskDispatcherRegistrar registrar;

    @Test
    void startCaseCreatesInstance() {
        registrar.classifier().loadClassifications(java.util.List.of(
                new KeywordClassifierDispatcher.ClassificationEntry("laptop", "HARDWARE", "HIGH")));

        var caseId = caseHub.startCase(Map.of(
                "subject", "My laptop won't boot",
                "customerRef", "alice",
                "status", "OPEN"));

        assertNotNull(caseId);

        await().atMost(10, TimeUnit.SECONDS).untilAsserted(() -> {
            var instance = caseInstanceCache.get(caseId);
            assertNotNull(instance);
        });
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=ChatCaseCreationHandlerTest -pl .`
Expected: Compilation failure or test failure — handler not wired

- [ ] **Step 3: Implement ChatCaseCreationHandler**

```java
package io.casehub.examples.helpdesk.engine;

import io.casehub.connectors.chat.model.ReceivedMessage;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ChatCaseCreationHandler {

    private static final Logger LOG = Logger.getLogger(ChatCaseCreationHandler.class);

    @Inject HelpDeskCaseHub caseHub;

    public void onMessage(@ObservesAsync ReceivedMessage message) {
        String subject = message.content().text();
        String customerRef = message.sender().id();

        UUID caseId = caseHub.startCase(Map.of(
                "subject", subject,
                "customerRef", customerRef,
                "status", "OPEN"));

        LOG.infof("Help desk case %s created from chat message: %s", caseId, subject);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=ChatCaseCreationHandlerTest -pl .`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples add helpdesk/src/main/java/io/casehub/examples/helpdesk/engine/ChatCaseCreationHandler.java helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/ChatCaseCreationHandlerTest.java
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "feat(#416): ChatCaseCreationHandler — ReceivedMessage → CaseHub.startCase
Refs #416"
```

---

### Task 4: Full lifecycle integration test

**Files:**
- Create: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/HelpdeskLifecycleTest.java`

**Interfaces:**
- Consumes: `HelpDeskCaseHub`, `CaseInstanceCache`, `HelpdeskDispatcherRegistrar`, `CallableDispatchRegistry`, casehub-work REST API (`/workitems`)
- Produces: End-to-end lifecycle verification

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.examples.helpdesk.engine;

import static io.restassured.RestAssured.given;
import static org.awaitility.Awaitility.await;
import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class HelpdeskLifecycleTest {

    @Inject HelpDeskCaseHub caseHub;
    @Inject CaseInstanceCache caseInstanceCache;
    @Inject HelpdeskDispatcherRegistrar registrar;

    @BeforeEach
    void setUp() {
        registrar.classifier().loadClassifications(List.of(
                new KeywordClassifierDispatcher.ClassificationEntry("laptop", "HARDWARE", "HIGH"),
                new KeywordClassifierDispatcher.ClassificationEntry("password", "ACCESS", "LOW")));
    }

    @Test
    void fullLifecycle_classifyCreateWorkItemResolveNotify() {
        UUID caseId = caseHub.startCase(Map.of(
                "subject", "My laptop won't boot",
                "customerRef", "alice",
                "status", "OPEN"));

        // Wait for triage binding to fire and classify
        await().atMost(10, TimeUnit.SECONDS).untilAsserted(() -> {
            Object category = caseHub.query(caseId, "category");
            assertNotNull(category, "Ticket should be classified");
            assertEquals("HARDWARE", category);
        });

        // Wait for status to be TRIAGED
        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            Object status = caseHub.query(caseId, "status");
            assertEquals("TRIAGED", status);
        });

        // Wait for WorkItem to be created by humanTask binding
        await().atMost(10, TimeUnit.SECONDS).untilAsserted(() -> {
            var workItems = given()
                    .queryParam("candidateGroups", "hw-specialist")
                    .when().get("/workitems")
                    .then().statusCode(200)
                    .extract().jsonPath().getList("$");
            assertFalse(workItems.isEmpty(), "WorkItem should be created for hw-specialist");
        });

        // Specialist claims and completes the WorkItem
        String workItemId = given()
                .queryParam("candidateGroups", "hw-specialist")
                .when().get("/workitems")
                .then().statusCode(200)
                .extract().jsonPath().getString("[0].id");

        given().when().put("/workitems/" + workItemId + "/start")
                .then().statusCode(200);

        given().contentType("application/json")
                .body(Map.of("resolution", "BIOS reset resolved the issue",
                             "assigneeId", "hw-specialist"))
                .when().put("/workitems/" + workItemId + "/complete")
                .then().statusCode(200);

        // Wait for case to complete (notification fired, goal met)
        await().atMost(15, TimeUnit.SECONDS).untilAsserted(() -> {
            var instance = caseInstanceCache.get(caseId);
            assertNotNull(instance);
            assertEquals(CaseStatus.COMPLETED, instance.getState(),
                    "Case should complete after resolution and notification");
        });
    }
}
```

- [ ] **Step 2: Run test to verify it passes (or iterate on YAML/dispatchers)**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=HelpdeskLifecycleTest -pl .`
Expected: PASS — full lifecycle executes end-to-end

This test may require iteration on the YAML definition (JQ expressions, binding filters) and dispatchers. Debug by checking:
- Case context after each binding fires (`caseHub.query(caseId, "key")`)
- Plan items (`GET /api/v1/cases/{caseId}/plan-items`)
- WorkItem status and resolution data
- Event log (`caseHub.eventLog(caseId)`)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples add helpdesk/src/test/java/io/casehub/examples/helpdesk/engine/HelpdeskLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "test(#416): full lifecycle integration test — classify → WorkItem → resolve → notify → complete
Refs #416"
```

---

### Task 5: Remove old services + update REST/demo endpoints

**Files:**
- Delete: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/TicketService.java`
- Delete: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/TicketCreationHandler.java`
- Delete: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/spi/TicketClassifier.java`
- Delete: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/spi/Classification.java`
- Delete: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/DemoTicketClassifier.java`
- Delete: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/event/TicketEvent.java`
- Delete: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/TicketServiceTest.java`
- Delete: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/TicketCreationHandlerTest.java`
- Delete: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/demo/DemoTicketClassifierTest.java`
- Delete: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/event/TicketEventFiringTest.java`
- Modify: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/rest/TicketResource.java`
- Modify: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/ScenarioBootstrapResource.java`
- Modify: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/VerificationResource.java`

**Interfaces:**
- Consumes: `HelpDeskCaseHub`, `CaseHubRuntime`, engine REST API
- Produces: Simplified `TicketResource` (query engine context), updated demo endpoints

- [ ] **Step 1: Delete old service files**

Use `ide_refactor_safe_delete` for each file. Delete in dependency order:
1. Tests first (no dependents)
2. Then `TicketCreationHandler` (depends on `TicketService`, `TicketClassifier`)
3. Then `DemoTicketClassifier` (implements `TicketClassifier`)
4. Then `TicketService`
5. Then `TicketClassifier`, `Classification`
6. Then `TicketEvent`

- [ ] **Step 2: Update TicketResource**

Replace TicketService dependency with engine queries. The engine REST module provides `GET /api/v1/cases` for listing. For helpdesk-specific ticket views, query case context:

The `/tickets` endpoint is no longer needed — the engine REST module provides `GET /api/v1/cases` with full case listing, and `GET /api/v1/cases/{caseId}/context` for ticket-shaped data. The dashboard YAML will be updated in Task 8 to poll these engine endpoints directly.

Delete `TicketResource.java` using `ide_refactor_safe_delete`. Also delete the `ResolveRequest` record it contains — WorkItem completion replaces the resolve endpoint.

For scenario executor backwards compatibility: the scenario file's `specialist-resolves` step will use `PUT /workitems/{id}/complete` instead of `PUT /tickets/{id}/resolve`. Update the scenario YAML in `src/main/resources/scenarios/help-desk-basic.yaml` if it exists.

- [ ] **Step 3: Update ScenarioBootstrapResource**

Change from loading `DemoTicketClassifier` to loading `KeywordClassifierDispatcher`:

```java
// In ScenarioBootstrapResource:
@Inject HelpdeskDispatcherRegistrar registrar;

// In the bootstrap method:
registrar.classifier().loadClassifications(
    request.ticketClassifications().stream()
        .map(e -> new KeywordClassifierDispatcher.ClassificationEntry(
            e.match(), e.category().name(), e.priority().name()))
        .toList());
```

- [ ] **Step 4: Update VerificationResource**

The `GET /scenario/verify/tickets` endpoint currently returns `TicketService.findAll()`. Replace with engine case queries. The `GET /scenario/verify/notifications` endpoint uses `NotificationService.getSentNotifications()` which still works unchanged.

```java
// In VerificationResource — replace TicketService with engine queries:
@Inject io.casehub.api.engine.CaseHubRuntime runtime;
@Inject io.casehub.engine.common.spi.CaseInstanceRepository caseInstanceRepository;

@GET @Path("/tickets")
public List<Map<String, Object>> tickets() {
    return caseInstanceRepository.findByNamespaceAndName("casehub-examples", "helpdesk-ticket")
            .stream()
            .map(instance -> {
                var ctx = runtime.query(instance.getId(), ".", Map.class);
                return ctx != null ? ctx : Map.<String, Object>of();
            })
            .toList();
}
```

- [ ] **Step 5: Run all tests to verify compilation and existing tests pass**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -pl .`
Expected: All tests pass (new engine tests + remaining tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples add -A helpdesk/
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "refactor(#416): remove hand-coded services, wire REST endpoints to engine
Refs #416"
```

---

### Task 6: Update push infrastructure

**Files:**
- Modify: `.../helpdesk/src/main/java/io/casehub/examples/helpdesk/push/TicketPushObserver.java`
- Modify: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/push/TicketPushObserverTest.java`

**Interfaces:**
- Consumes: Engine case events or case context change events (replacing `TicketEvent`)
- Produces: SSE push messages to connected dashboard clients

- [ ] **Step 1: Identify the engine event to observe**

The engine fires CDI events on context changes. Options:
- Observe `CaseContextChangedEvent` from the engine
- Poll the engine REST API from the push observer
- Use the pages-push `EventBroadcaster` to push case state changes

Check which engine CDI events are available and adapt `TicketPushObserver` to observe them instead of `TicketEvent`.

- [ ] **Step 2: Update TicketPushObserver**

Replace `@ObservesAsync TicketEvent` with engine event observation. The observer should extract ticket-relevant fields from the case context and push them via the existing SSE mechanism.

- [ ] **Step 3: Update TicketPushObserverTest**

Adapt to fire engine events instead of `TicketEvent`.

- [ ] **Step 4: Run push-related tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest="TicketPushObserverTest,PushEndpointTest" -pl .`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples add helpdesk/src/main/java/io/casehub/examples/helpdesk/push/ helpdesk/src/test/java/io/casehub/examples/helpdesk/push/
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "refactor(#416): push observer wired to engine events instead of TicketEvent
Refs #416"
```

---

### Task 7: Update HelpDeskScenarioTest

**Files:**
- Modify: `.../helpdesk/src/test/java/io/casehub/examples/helpdesk/HelpDeskScenarioTest.java`

**Interfaces:**
- Consumes: Engine REST API (`/api/v1/cases`), work REST API (`/workitems`), chat injection endpoint
- Produces: Updated end-to-end scenario test using engine/work APIs

- [ ] **Step 1: Update scenario test to use engine endpoints**

The existing `HelpDeskScenarioTest` tests the full flow via REST. Update it to:
1. Bootstrap classifications via `/scenario/bootstrap/helpdesk`
2. Inject a chat message via `/scenario/inject/chat`
3. Wait for case to appear via `GET /api/v1/cases`
4. Wait for ticket to be classified (check case context)
5. Wait for WorkItem to appear via `GET /workitems`
6. Complete WorkItem via `PUT /workitems/{id}/complete`
7. Verify notification via `/scenario/verify/notifications`
8. Verify case completed via `GET /api/v1/cases/{id}`

- [ ] **Step 2: Run scenario test**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml test -Dtest=HelpDeskScenarioTest -pl .`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples add helpdesk/src/test/java/io/casehub/examples/helpdesk/HelpDeskScenarioTest.java
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "test(#416): update scenario test for engine/work-driven lifecycle
Refs #416"
```

---

### Task 8: Update Helpdesk Ops dashboard (casehub-pages)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/slots/112/pages/examples/samples.json` (if source sample is added)
- Create: `/Users/mdproctor/claude/casehub/slots/112/pages/examples/samples/Scenario Demos/Helpdesk Ops.dash.yaml` (if not already in source)

**Interfaces:**
- Consumes: Engine REST API (`/api/v1/cases`), work REST API (`/workitems`)
- Produces: Updated dashboard with engine/work data sources and ops tabs

- [ ] **Step 1: Add engine cases data source**

Add a new dataset polling the engine cases API:

```yaml
- uuid: cases
  url: ${api}/api/v1/cases
  refreshTime: 2second
  expression: >-
    $.content.[id, namespace, name, state, createdAt]
  columns:
    - id: id
      type: LABEL
    - id: namespace
      type: LABEL
    - id: name
      type: LABEL
    - id: state
      type: LABEL
    - id: created
      type: DATE
```

- [ ] **Step 2: Add work items data source**

```yaml
- uuid: workitems
  url: ${api}/workitems?candidateGroups=hw-specialist,sw-specialist,access-specialist,general-specialist
  refreshTime: 2second
  expression: >-
    $.[id, title, status, assigneeId, candidateGroups, createdAt]
  columns:
    - id: id
      type: LABEL
    - id: title
      type: TEXT
    - id: status
      type: LABEL
    - id: assignee
      type: LABEL
    - id: groups
      type: LABEL
    - id: created
      type: DATE
```

- [ ] **Step 3: Add Cases and Work Items tabs**

Add two new pages to the navTree:
- **Cases** — data-table showing engine cases
- **Work Items** — data-table showing work items with claim/complete actions

- [ ] **Step 4: Update Ops Dashboard metrics**

Change the metrics on the Ops Dashboard to use the engine cases dataset instead of the tickets dataset (which now comes from engine context, not TicketService).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/pages add examples/
git -C /Users/mdproctor/claude/casehub/slots/112/pages commit -m "feat(#416): helpdesk dashboard wired to engine cases + work items
Refs #416"
```

---

### Task 9: Final verification + cleanup

**Files:**
- Various cleanup across both repos

- [ ] **Step 1: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml clean test -pl .`
Expected: All tests PASS

- [ ] **Step 2: Verify no unused imports or dead code**

Use `ide_diagnostics` on all modified/created files in the helpdesk module.

- [ ] **Step 3: Start the application and verify manually**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/slots/112/examples/helpdesk/pom.xml quarkus:dev -Dquarkus.profile=demo
```

1. Bootstrap classifications via `POST /scenario/bootstrap/helpdesk`
2. Submit a ticket via `POST /scenario/inject/chat`
3. Check `GET /api/v1/cases` — case should appear and progress through triage
4. Check `GET /workitems` — WorkItem should appear for the right specialist group
5. Complete the WorkItem via `PUT /workitems/{id}/start` then `PUT /workitems/{id}/complete`
6. Check case completes and notification is sent

- [ ] **Step 4: Final commit if any cleanup**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/examples commit -m "chore(#416): cleanup and final verification
Refs #416"
```
