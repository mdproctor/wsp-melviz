# IT Help Desk Example Application — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** New issue under casehubio/parent#408
**Issue group:** #408 (Cross-platform Scenario Engine)

**Goal:** Build a minimal IT help desk Quarkus application in
casehub-examples that demonstrates the demo-profile + scenario-driven
pattern: chat message → ticket creation → classification → resolution →
notification. Ships with a scenario file for both scripted demo and
verification.

**Architecture:** Standalone Quarkus app with in-memory state. Depends
on casehub-connectors (chat-spi, core) for inbound message handling
and casehub-platform-api for identity. No casehub-engine or casehub-work
in this first iteration — the ticket lifecycle is a simple in-memory
service. Engine/work integration is a planned enhancement for a later
iteration.

**Tech Stack:** Java 21+, Quarkus (arc, rest, rest-jackson), casehub-connectors
(chat-spi, core), casehub-platform-api, Jackson YAML, AssertJ

## Global Constraints

- All demo-profile classes use `@IfBuildProfile("demo")` — compile-time gate
- Demo alternatives use `@Alternative @Priority(300)` per demo-spi-convention
- SPI interfaces are app-local (not in platform modules) unless an existing
  platform SPI covers the need
- No persistence — all state in `ConcurrentHashMap` / `CopyOnWriteArrayList`
- Package root: `io.casehub.examples.helpdesk`

---

### Task 1: Project scaffold + domain model + ticket service

**Files:**
- Create: `casehub-examples/helpdesk/pom.xml`
- Create: `casehub-examples/helpdesk/src/main/resources/application.properties`
- Create: `casehub-examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/model/Ticket.java`
- Create: `casehub-examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/model/TicketCategory.java`
- Create: `casehub-examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/model/TicketPriority.java`
- Create: `casehub-examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/model/TicketStatus.java`
- Create: `casehub-examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/TicketService.java`
- Create: `casehub-examples/helpdesk/src/main/java/io/casehub/examples/helpdesk/rest/TicketResource.java`
- Test: `casehub-examples/helpdesk/src/test/java/io/casehub/examples/helpdesk/TicketServiceTest.java`
- Modify: `casehub-examples/pom.xml` (add `<module>helpdesk</module>`)

**Interfaces:**
- Produces: `TicketService.create(subject, description, customerRef) → Ticket`
- Produces: `TicketService.classify(ticketId, category, priority)`
- Produces: `TicketService.assign(ticketId, assigneeId)`
- Produces: `TicketService.resolve(ticketId, resolution) → Ticket`
- Produces: `TicketService.findById(ticketId) → Optional<Ticket>`
- Produces: `TicketService.findAll() → List<Ticket>`
- Produces: `TicketCategory` enum: `HARDWARE, SOFTWARE, ACCESS, OTHER`
- Produces: `TicketPriority` enum: `LOW, MEDIUM, HIGH, URGENT`
- Produces: `TicketStatus` enum: `OPEN, TRIAGED, ASSIGNED, RESOLVED, CLOSED`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.examples.helpdesk;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;
import io.casehub.examples.helpdesk.model.TicketStatus;

class TicketServiceTest {

    TicketService service;

    @BeforeEach
    void setUp() {
        service = new TicketService();
    }

    @Test
    void createTicketSetsOpenStatus() {
        var ticket = service.create("Laptop won't boot", "After update last night", "alice");
        assertThat(ticket.id()).isNotNull();
        assertThat(ticket.status()).isEqualTo(TicketStatus.OPEN);
        assertThat(ticket.customerRef()).isEqualTo("alice");
    }

    @Test
    void classifyTicketSetsTriagedStatus() {
        var ticket = service.create("Laptop won't boot", "After update", "alice");
        service.classify(ticket.id(), TicketCategory.HARDWARE, TicketPriority.HIGH);
        var updated = service.findById(ticket.id()).orElseThrow();
        assertThat(updated.status()).isEqualTo(TicketStatus.TRIAGED);
        assertThat(updated.category()).isEqualTo(TicketCategory.HARDWARE);
        assertThat(updated.priority()).isEqualTo(TicketPriority.HIGH);
    }

    @Test
    void assignTicketSetsAssignedStatus() {
        var ticket = service.create("Laptop won't boot", "After update", "alice");
        service.classify(ticket.id(), TicketCategory.HARDWARE, TicketPriority.HIGH);
        service.assign(ticket.id(), "hw-specialist");
        var updated = service.findById(ticket.id()).orElseThrow();
        assertThat(updated.status()).isEqualTo(TicketStatus.ASSIGNED);
        assertThat(updated.assigneeId()).isEqualTo("hw-specialist");
    }

    @Test
    void resolveTicketSetsResolvedStatus() {
        var ticket = service.create("Laptop won't boot", "After update", "alice");
        service.classify(ticket.id(), TicketCategory.HARDWARE, TicketPriority.HIGH);
        service.assign(ticket.id(), "hw-specialist");
        var resolved = service.resolve(ticket.id(), "BIOS reset fixed it");
        assertThat(resolved.status()).isEqualTo(TicketStatus.RESOLVED);
        assertThat(resolved.resolution()).isEqualTo("BIOS reset fixed it");
        assertThat(resolved.resolvedAt()).isNotNull();
    }

    @Test
    void findAllReturnsAllTickets() {
        service.create("Issue 1", "Desc 1", "alice");
        service.create("Issue 2", "Desc 2", "bob");
        assertThat(service.findAll()).hasSize(2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl helpdesk -Dtest=TicketServiceTest -f /path/to/casehub-examples/pom.xml`
Expected: Compilation error — classes don't exist yet.

- [ ] **Step 3: Create pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub.examples</groupId>
  <artifactId>casehub-examples-helpdesk</artifactId>
  <version>0.1-SNAPSHOT</version>
  <name>CaseHub Examples — IT Help Desk</name>
  <description>
    Minimal help desk app demonstrating demo-profile switching and
    scenario-driven demos. Chat inbound triggers ticket creation,
    automated classification, specialist resolution, customer notification.
  </description>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <quarkus.version>3.32.2</quarkus.version>
    <casehub.version>0.2-SNAPSHOT</casehub.version>
    <assertj.version>3.27.3</assertj.version>
    <surefire.version>3.5.2</surefire.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-bom</artifactId>
        <version>${quarkus.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-spi</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
      <version>${casehub.version}</version>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.dataformat</groupId>
      <artifactId>jackson-dataformat-yaml</artifactId>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <version>${assertj.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.version}</version>
        <extensions>true</extensions>
        <executions>
          <execution>
            <goals><goal>build</goal></goals>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${surefire.version}</version>
      </plugin>
    </plugins>
  </build>

  <repositories>
    <repository>
      <id>github</id>
      <url>https://maven.pkg.github.com/casehubio/*</url>
    </repository>
  </repositories>
</project>
```

- [ ] **Step 4: Create domain model**

```java
// TicketCategory.java
package io.casehub.examples.helpdesk.model;

public enum TicketCategory { HARDWARE, SOFTWARE, ACCESS, OTHER }
```

```java
// TicketPriority.java
package io.casehub.examples.helpdesk.model;

public enum TicketPriority { LOW, MEDIUM, HIGH, URGENT }
```

```java
// TicketStatus.java
package io.casehub.examples.helpdesk.model;

public enum TicketStatus { OPEN, TRIAGED, ASSIGNED, RESOLVED, CLOSED }
```

```java
// Ticket.java
package io.casehub.examples.helpdesk.model;

import java.time.Instant;
import java.util.UUID;

public record Ticket(
        UUID id,
        String subject,
        String description,
        TicketCategory category,
        TicketPriority priority,
        TicketStatus status,
        String customerRef,
        String assigneeId,
        String resolution,
        Instant createdAt,
        Instant resolvedAt) {

    public Ticket withStatus(TicketStatus newStatus) {
        return new Ticket(id, subject, description, category, priority,
                newStatus, customerRef, assigneeId, resolution, createdAt, resolvedAt);
    }

    public Ticket withClassification(TicketCategory cat, TicketPriority pri) {
        return new Ticket(id, subject, description, cat, pri,
                TicketStatus.TRIAGED, customerRef, assigneeId, resolution, createdAt, resolvedAt);
    }

    public Ticket withAssignee(String assignee) {
        return new Ticket(id, subject, description, category, priority,
                TicketStatus.ASSIGNED, customerRef, assignee, resolution, createdAt, resolvedAt);
    }

    public Ticket withResolution(String res) {
        return new Ticket(id, subject, description, category, priority,
                TicketStatus.RESOLVED, customerRef, assigneeId, res, createdAt, Instant.now());
    }
}
```

- [ ] **Step 5: Create TicketService**

```java
package io.casehub.examples.helpdesk;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.examples.helpdesk.model.Ticket;
import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;
import io.casehub.examples.helpdesk.model.TicketStatus;

@ApplicationScoped
public class TicketService {

    private final Map<UUID, Ticket> tickets = new ConcurrentHashMap<>();

    public Ticket create(String subject, String description, String customerRef) {
        var ticket = new Ticket(
                UUID.randomUUID(), subject, description,
                null, null, TicketStatus.OPEN,
                customerRef, null, null,
                Instant.now(), null);
        tickets.put(ticket.id(), ticket);
        return ticket;
    }

    public void classify(UUID ticketId, TicketCategory category, TicketPriority priority) {
        tickets.computeIfPresent(ticketId, (id, t) -> t.withClassification(category, priority));
    }

    public void assign(UUID ticketId, String assigneeId) {
        tickets.computeIfPresent(ticketId, (id, t) -> t.withAssignee(assigneeId));
    }

    public Ticket resolve(UUID ticketId, String resolution) {
        var resolved = tickets.computeIfPresent(ticketId, (id, t) -> t.withResolution(resolution));
        return resolved;
    }

    public Optional<Ticket> findById(UUID ticketId) {
        return Optional.ofNullable(tickets.get(ticketId));
    }

    public List<Ticket> findAll() {
        return List.copyOf(tickets.values());
    }
}
```

- [ ] **Step 6: Create application.properties**

```properties
quarkus.http.port=8090
quarkus.log.category."io.casehub.examples".level=DEBUG
```

- [ ] **Step 7: Create TicketResource (basic REST)**

```java
package io.casehub.examples.helpdesk.rest;

import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.PUT;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.casehub.examples.helpdesk.TicketService;
import io.casehub.examples.helpdesk.model.Ticket;

@Path("/tickets")
@Produces(MediaType.APPLICATION_JSON)
public class TicketResource {

    @Inject
    TicketService ticketService;

    @GET
    public List<Ticket> list() {
        return ticketService.findAll();
    }

    @GET
    @Path("/{id}")
    public Response get(@PathParam("id") UUID id) {
        return ticketService.findById(id)
                .map(t -> Response.ok(t).build())
                .orElse(Response.status(404).build());
    }

    @PUT
    @Path("/{id}/resolve")
    public Response resolve(@PathParam("id") UUID id, ResolveRequest request) {
        var ticket = ticketService.resolve(id, request.resolution());
        return ticket != null ? Response.ok(ticket).build() : Response.status(404).build();
    }

    public record ResolveRequest(String resolution) {}
}
```

- [ ] **Step 8: Add helpdesk module to parent pom**

Add `<module>helpdesk</module>` to `casehub-examples/pom.xml` modules section.

- [ ] **Step 9: Run tests to verify they pass**

Run: `mvn test -pl helpdesk -f /path/to/casehub-examples/pom.xml`
Expected: All 5 tests PASS.

- [ ] **Step 10: Commit**

```bash
git add helpdesk/ pom.xml
git commit -m "feat(helpdesk): project scaffold, domain model, ticket service

Refs casehubio/parent#408"
```

---

### Task 2: TicketClassifier SPI + demo implementation

**Files:**
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/spi/TicketClassifier.java`
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/spi/Classification.java`
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/DemoTicketClassifier.java`
- Test: `helpdesk/src/test/java/io/casehub/examples/helpdesk/demo/DemoTicketClassifierTest.java`

**Interfaces:**
- Consumes: `TicketCategory`, `TicketPriority` (from Task 1)
- Produces: `TicketClassifier.classify(subject, description) → Classification`
- Produces: `Classification(TicketCategory category, TicketPriority priority)`
- Produces: `DemoTicketClassifier.loadClassifications(List<ClassificationEntry>)` — bootstrap from scenario data

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.examples.helpdesk.demo;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;

class DemoTicketClassifierTest {

    @Test
    void matchesLoadedClassification() {
        var classifier = new DemoTicketClassifier();
        classifier.loadClassifications(List.of(
                new DemoTicketClassifier.ClassificationEntry(
                        "laptop won't boot", TicketCategory.HARDWARE, TicketPriority.HIGH)));

        var result = classifier.classify("My laptop won't boot after the update", "Details here");
        assertThat(result.category()).isEqualTo(TicketCategory.HARDWARE);
        assertThat(result.priority()).isEqualTo(TicketPriority.HIGH);
    }

    @Test
    void returnsDefaultWhenNoMatch() {
        var classifier = new DemoTicketClassifier();
        classifier.loadClassifications(List.of());

        var result = classifier.classify("Something random", "No match");
        assertThat(result.category()).isEqualTo(TicketCategory.OTHER);
        assertThat(result.priority()).isEqualTo(TicketPriority.MEDIUM);
    }

    @Test
    void matchIsCaseInsensitiveSubstring() {
        var classifier = new DemoTicketClassifier();
        classifier.loadClassifications(List.of(
                new DemoTicketClassifier.ClassificationEntry(
                        "password reset", TicketCategory.ACCESS, TicketPriority.LOW)));

        var result = classifier.classify("I need a PASSWORD RESET please", "Urgent");
        assertThat(result.category()).isEqualTo(TicketCategory.ACCESS);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl helpdesk -Dtest=DemoTicketClassifierTest`
Expected: Compilation error — classes don't exist.

- [ ] **Step 3: Create SPI interface + Classification record**

```java
// TicketClassifier.java
package io.casehub.examples.helpdesk.spi;

public interface TicketClassifier {
    Classification classify(String subject, String description);
}
```

```java
// Classification.java
package io.casehub.examples.helpdesk.spi;

import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;

public record Classification(TicketCategory category, TicketPriority priority) {}
```

- [ ] **Step 4: Create DemoTicketClassifier**

```java
package io.casehub.examples.helpdesk.demo;

import java.util.List;
import java.util.Locale;
import java.util.concurrent.CopyOnWriteArrayList;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkus.arc.profile.IfBuildProfile;
import jakarta.annotation.Priority;
import jakarta.enterprise.inject.Alternative;

import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;
import io.casehub.examples.helpdesk.spi.Classification;
import io.casehub.examples.helpdesk.spi.TicketClassifier;

@ApplicationScoped
@Alternative
@Priority(300)
@IfBuildProfile("demo")
public class DemoTicketClassifier implements TicketClassifier {

    private final List<ClassificationEntry> entries = new CopyOnWriteArrayList<>();

    @Override
    public Classification classify(String subject, String description) {
        var combined = (subject + " " + description).toLowerCase(Locale.ROOT);
        return entries.stream()
                .filter(e -> combined.contains(e.match().toLowerCase(Locale.ROOT)))
                .findFirst()
                .map(e -> new Classification(e.category(), e.priority()))
                .orElse(new Classification(TicketCategory.OTHER, TicketPriority.MEDIUM));
    }

    public void loadClassifications(List<ClassificationEntry> data) {
        entries.clear();
        entries.addAll(data);
    }

    public record ClassificationEntry(String match, TicketCategory category, TicketPriority priority) {}
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl helpdesk -Dtest=DemoTicketClassifierTest`
Expected: All 3 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add helpdesk/src/
git commit -m "feat(helpdesk): TicketClassifier SPI + demo lookup implementation

Refs casehubio/parent#408"
```

---

### Task 3: Chat inbound pipeline — injection, translation, ticket creation

**Files:**
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/DemoInboundTranslator.java`
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/ChatInjectionResource.java`
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/DemoChatPlatform.java`
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/TicketCreationHandler.java`
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/NotificationService.java`
- Test: `helpdesk/src/test/java/io/casehub/examples/helpdesk/TicketCreationHandlerTest.java`
- Test: `helpdesk/src/test/java/io/casehub/examples/helpdesk/demo/ChatInjectionResourceTest.java`

**Interfaces:**
- Consumes: `TicketService.create()`, `TicketService.classify()`, `TicketService.assign()` (Task 1)
- Consumes: `TicketClassifier.classify()` (Task 2)
- Consumes: `InboundMessage`, `ReceivedMessage`, `InboundTranslator` (from casehub-connectors)
- Produces: `ChatInjectionResource` — POST /scenario/inject/chat (fires InboundMessage CDI event)
- Produces: `DemoInboundTranslator` — translates InboundMessage to ReceivedMessage for type "demo"
- Produces: `DemoChatPlatform` — Messaging records outbound, all others degraded
- Produces: `TicketCreationHandler` — observes ReceivedMessage, creates + classifies ticket
- Produces: `NotificationService.notify(customerRef, message)` — sends via ChatPlatform, records for verification
- Produces: `NotificationService.getSentNotifications() → List<SentNotification>`

- [ ] **Step 1: Write the failing test for TicketCreationHandler**

```java
package io.casehub.examples.helpdesk;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.examples.helpdesk.demo.DemoTicketClassifier;
import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;
import io.casehub.examples.helpdesk.model.TicketStatus;

class TicketCreationHandlerTest {

    TicketService ticketService;
    DemoTicketClassifier classifier;
    TicketCreationHandler handler;

    @BeforeEach
    void setUp() {
        ticketService = new TicketService();
        classifier = new DemoTicketClassifier();
        classifier.loadClassifications(List.of(
                new DemoTicketClassifier.ClassificationEntry(
                        "laptop", TicketCategory.HARDWARE, TicketPriority.HIGH)));
        handler = new TicketCreationHandler(ticketService, classifier);
    }

    @Test
    void createsAndClassifiesTicketFromChatMessage() {
        var channel = new ChatChannelRef("support");
        var msg = new ReceivedMessage(
                "demo", channel,
                new ChatMessageRef(channel, "msg-1"), null,
                new MemberRef("alice"),
                new ChatContent("My laptop won't boot"),
                Instant.now());

        handler.onMessage(msg);

        var tickets = ticketService.findAll();
        assertThat(tickets).hasSize(1);
        var ticket = tickets.getFirst();
        assertThat(ticket.subject()).isEqualTo("My laptop won't boot");
        assertThat(ticket.customerRef()).isEqualTo("alice");
        assertThat(ticket.status()).isEqualTo(TicketStatus.TRIAGED);
        assertThat(ticket.category()).isEqualTo(TicketCategory.HARDWARE);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl helpdesk -Dtest=TicketCreationHandlerTest`
Expected: Compilation error — TicketCreationHandler doesn't exist.

- [ ] **Step 3: Create TicketCreationHandler**

```java
package io.casehub.examples.helpdesk;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.examples.helpdesk.spi.TicketClassifier;

@ApplicationScoped
public class TicketCreationHandler {

    private static final Logger LOG = Logger.getLogger(TicketCreationHandler.class);

    private final TicketService ticketService;
    private final TicketClassifier classifier;

    @Inject
    public TicketCreationHandler(TicketService ticketService, TicketClassifier classifier) {
        this.ticketService = ticketService;
        this.classifier = classifier;
    }

    public void onMessage(@ObservesAsync ReceivedMessage message) {
        var subject = message.content().text();
        var ticket = ticketService.create(subject, subject, message.sender().id());
        var classification = classifier.classify(subject, "");
        ticketService.classify(ticket.id(), classification.category(), classification.priority());
        ticketService.assign(ticket.id(), routeByCategory(classification.category()));
        LOG.infof("Ticket %s created from chat message: %s → %s",
                ticket.id(), subject, classification.category());
    }

    private String routeByCategory(io.casehub.examples.helpdesk.model.TicketCategory category) {
        return switch (category) {
            case HARDWARE -> "hw-specialist";
            case SOFTWARE -> "sw-specialist";
            case ACCESS -> "access-specialist";
            case OTHER -> "general-specialist";
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl helpdesk -Dtest=TicketCreationHandlerTest`
Expected: PASS.

- [ ] **Step 5: Create DemoInboundTranslator**

```java
package io.casehub.examples.helpdesk.demo;

import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.MemberRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.InboundTranslator;

@ApplicationScoped
public class DemoInboundTranslator implements InboundTranslator {

    @Override
    public String connectorType() {
        return "demo";
    }

    @Override
    public ReceivedMessage translate(InboundMessage msg) {
        var channel = new ChatChannelRef(
                msg.externalChannelRef() != null ? msg.externalChannelRef() : "support");
        return new ReceivedMessage(
                "demo", channel,
                new ChatMessageRef(channel, UUID.randomUUID().toString()),
                null,
                new MemberRef(msg.externalSenderId()),
                new ChatContent(msg.content(), null, msg.attachments(), List.of()),
                msg.receivedAt());
    }
}
```

- [ ] **Step 6: Create ChatInjectionResource**

```java
package io.casehub.examples.helpdesk.demo;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkus.arc.profile.IfBuildProfile;

import io.casehub.connectors.InboundMessage;

@Path("/scenario/inject/chat")
@IfBuildProfile("demo")
@ApplicationScoped
public class ChatInjectionResource {

    @Inject
    Event<InboundMessage> inboundEvent;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    public Response inject(ChatInjectionRequest request) {
        var msg = new InboundMessage(
                "demo", "demo",
                request.from(), request.channelId(),
                request.text(), List.of(),
                Instant.now(), Map.of(), null);
        inboundEvent.fireAsync(msg);
        return Response.accepted().build();
    }

    public record ChatInjectionRequest(String from, String channelId, String text) {}
}
```

- [ ] **Step 7: Create DemoChatPlatform**

```java
package io.casehub.examples.helpdesk.demo;

import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkus.arc.profile.IfBuildProfile;

import io.casehub.connectors.chat.model.ChatMessageRef;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.MemberManagement;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.MessageHistory;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Threading;
import io.casehub.connectors.chat.spi.ChannelManagement;

@ApplicationScoped
@Alternative
@Priority(300)
@IfBuildProfile("demo")
public class DemoChatPlatform implements ChatPlatform {

    public record SentMessage(String channelId, String text, Instant sentAt) {}

    private final List<SentMessage> sentMessages = new CopyOnWriteArrayList<>();

    @Override
    public String id() { return "demo"; }

    @Override
    public Messaging messaging() {
        return (channel, content) -> {
            sentMessages.add(new SentMessage(channel.id(), content.text(), Instant.now()));
            var ref = new ChatMessageRef(channel, UUID.randomUUID().toString());
            return SendResult.success(ref, Instant.now());
        };
    }

    public List<SentMessage> getSentMessages() { return List.copyOf(sentMessages); }

    private final ChatPlatform degraded = ChatPlatform.builder("demo")
            .messaging((ch, c) -> SendResult.success(
                    new ChatMessageRef(ch, UUID.randomUUID().toString()), Instant.now()))
            .build();

    @Override public Threading threading() { return degraded.threading(); }
    @Override public Discovery discovery() { return degraded.discovery(); }
    @Override public Reactions reactions() { return degraded.reactions(); }
    @Override public Presence presence() { return degraded.presence(); }
    @Override public Members members() { return degraded.members(); }
    @Override public ChannelManagement channelManagement() { return degraded.channelManagement(); }
    @Override public MemberManagement memberManagement() { return degraded.memberManagement(); }
    @Override public MessageHistory messageHistory() { return degraded.messageHistory(); }
    @Override public boolean supports(Class<?> capability) { return Messaging.class.equals(capability); }
}
```

- [ ] **Step 8: Create NotificationService**

```java
package io.casehub.examples.helpdesk;

import java.time.Instant;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.spi.ChatPlatform;

@ApplicationScoped
public class NotificationService {

    private final ChatPlatform chatPlatform;
    private final List<SentNotification> sent = new CopyOnWriteArrayList<>();

    @Inject
    public NotificationService(ChatPlatform chatPlatform) {
        this.chatPlatform = chatPlatform;
    }

    public void notify(String customerRef, String message) {
        chatPlatform.messaging().send(new ChatChannelRef(customerRef), new ChatContent(message));
        sent.add(new SentNotification(customerRef, message, Instant.now()));
    }

    public List<SentNotification> getSentNotifications() {
        return List.copyOf(sent);
    }

    public record SentNotification(String to, String message, Instant sentAt) {}
}
```

- [ ] **Step 9: Run all tests**

Run: `mvn test -pl helpdesk`
Expected: All tests PASS.

- [ ] **Step 10: Commit**

```bash
git add helpdesk/src/
git commit -m "feat(helpdesk): chat inbound pipeline + demo alternatives

DemoInboundTranslator, ChatInjectionResource, DemoChatPlatform,
TicketCreationHandler, NotificationService.

Refs casehubio/parent#408"
```

---

### Task 4: Scenario bootstrap, verification endpoints, scenario file + integration test

**Files:**
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/ScenarioBootstrapResource.java`
- Create: `helpdesk/src/main/java/io/casehub/examples/helpdesk/demo/VerificationResource.java`
- Create: `helpdesk/src/main/resources/scenarios/help-desk-basic.yaml`
- Modify: `helpdesk/src/main/java/io/casehub/examples/helpdesk/rest/TicketResource.java` (add resolve-with-notification)
- Test: `helpdesk/src/test/java/io/casehub/examples/helpdesk/HelpDeskScenarioTest.java`

**Interfaces:**
- Consumes: `DemoTicketClassifier.loadClassifications()` (Task 2)
- Consumes: `TicketService`, `NotificationService` (Tasks 1, 3)
- Produces: `ScenarioBootstrapResource` — POST /scenario/bootstrap/helpdesk
- Produces: `VerificationResource` — GET /scenario/verify/notifications, GET /scenario/verify/tickets

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.examples.helpdesk;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(DemoTestProfile.class)
class HelpDeskScenarioTest {

    @Test
    void chatMessageCreatesClassifiedTicket() {
        // Bootstrap: load classification lookup
        given()
            .contentType("application/json")
            .body(Map.of("ticket-classifications", List.of(
                    Map.of("match", "laptop won't boot",
                            "category", "HARDWARE",
                            "priority", "HIGH"))))
            .when().post("/scenario/bootstrap/helpdesk")
            .then().statusCode(200);

        // Inject: simulate a customer chat message
        given()
            .contentType("application/json")
            .body(Map.of("from", "Alice Customer",
                    "channelId", "support",
                    "text", "My laptop won't boot after the update last night"))
            .when().post("/scenario/inject/chat")
            .then().statusCode(202);

        // Verify: ticket was created and classified
        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            var tickets = given()
                    .when().get("/scenario/verify/tickets")
                    .then().statusCode(200)
                    .extract().jsonPath().getList(".", Map.class);
            assertThat(tickets).hasSize(1);
            assertThat(tickets.getFirst().get("status")).isEqualTo("TRIAGED");
            assertThat(tickets.getFirst().get("category")).isEqualTo("HARDWARE");
        });
    }
}
```

- [ ] **Step 2: Create DemoTestProfile**

```java
package io.casehub.examples.helpdesk;

import java.util.Map;

import io.quarkus.test.junit.QuarkusTestProfile;

public class DemoTestProfile implements QuarkusTestProfile {
    @Override
    public String getConfigProfile() {
        return "demo";
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl helpdesk -Dtest=HelpDeskScenarioTest`
Expected: Compilation error — missing classes. Or 404 from missing endpoints.

- [ ] **Step 4: Create ScenarioBootstrapResource**

```java
package io.casehub.examples.helpdesk.demo;

import java.util.List;
import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkus.arc.profile.IfBuildProfile;

import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;

@Path("/scenario/bootstrap/helpdesk")
@IfBuildProfile("demo")
@ApplicationScoped
public class ScenarioBootstrapResource {

    @Inject
    DemoTicketClassifier classifier;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    public Response bootstrap(BootstrapRequest request) {
        if (request.ticketClassifications() != null) {
            var entries = request.ticketClassifications().stream()
                    .map(e -> new DemoTicketClassifier.ClassificationEntry(
                            e.get("match").toString(),
                            TicketCategory.valueOf(e.get("category").toString()),
                            TicketPriority.valueOf(e.get("priority").toString())))
                    .toList();
            classifier.loadClassifications(entries);
        }
        return Response.ok().build();
    }

    public record BootstrapRequest(List<Map<String, Object>> ticketClassifications) {}
}
```

- [ ] **Step 5: Create VerificationResource**

```java
package io.casehub.examples.helpdesk.demo;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

import io.quarkus.arc.profile.IfBuildProfile;

import io.casehub.examples.helpdesk.NotificationService;
import io.casehub.examples.helpdesk.TicketService;
import io.casehub.examples.helpdesk.model.Ticket;

@Path("/scenario/verify")
@IfBuildProfile("demo")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class VerificationResource {

    @Inject
    TicketService ticketService;

    @Inject
    NotificationService notificationService;

    @GET
    @Path("/tickets")
    public List<Ticket> tickets() {
        return ticketService.findAll();
    }

    @GET
    @Path("/notifications")
    public List<NotificationService.SentNotification> notifications() {
        return notificationService.getSentNotifications();
    }
}
```

- [ ] **Step 6: Update TicketResource — add resolve-with-notification**

Add to `TicketResource.resolve()`: after resolving, call `notificationService.notify()`.

```java
// Add @Inject NotificationService notificationService; field

// Update resolve method:
@PUT
@Path("/{id}/resolve")
public Response resolve(@PathParam("id") UUID id, ResolveRequest request) {
    var ticket = ticketService.resolve(id, request.resolution());
    if (ticket != null) {
        notificationService.notify(ticket.customerRef(),
                "Your ticket has been resolved: " + request.resolution());
        return Response.ok(ticket).build();
    }
    return Response.status(404).build();
}
```

- [ ] **Step 7: Add test dependencies (rest-assured, awaitility)**

Add to pom.xml test dependencies:
```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 8: Create scenario file**

```yaml
# help-desk-basic.yaml
scenario: help-desk-basic
description: "Customer reports a hardware issue, specialist resolves it"
speed: 1
on-error: stop

data:
  ticket-classifications:
    - match: "laptop won't boot"
      category: HARDWARE
      priority: HIGH
    - match: "password reset"
      category: ACCESS
      priority: LOW
    - match: "install software"
      category: SOFTWARE
      priority: MEDIUM

steps:
  - name: bootstrap
    action: load-demo-data
    delivery: rest
    endpoint: POST /scenario/bootstrap/helpdesk
    data:
      ticket-classifications: { source: "data.ticket-classifications" }

  - name: customer-message
    action: customer-sends-message
    delivery: simulated
    target: chat
    trigger: { after: bootstrap }
    data:
      from: "Alice Customer"
      channelId: "support"
      text: "My laptop won't boot after the update last night"

  - name: verify-case-created
    action: verify-ticket-exists
    delivery: rest
    endpoint: GET /scenario/verify/tickets
    trigger: { after: customer-message, delay: 2000 }
    await:
      endpoint: GET /scenario/verify/tickets
      match: { status: "TRIAGED", category: "HARDWARE" }

  - name: specialist-resolves
    action: specialist-resolves-ticket
    delivery: rest
    endpoint: PUT /tickets/{id}/resolve
    trigger: { after: verify-case-created }
    actor: hw-specialist
    data:
      resolution: "BIOS reset resolved the boot issue. No hardware damage."

  - name: verify-notification
    action: verify-customer-notified
    delivery: rest
    endpoint: GET /scenario/verify/notifications
    trigger: { after: specialist-resolves, delay: 1000 }
    await:
      endpoint: GET /scenario/verify/notifications
      match: { to: "Alice Customer" }
```

- [ ] **Step 9: Run integration test**

Run: `mvn test -pl helpdesk -Dtest=HelpDeskScenarioTest`
Expected: PASS — bootstrap loads classifications, chat injection creates a classified ticket.

- [ ] **Step 10: Run full test suite**

Run: `mvn test -pl helpdesk`
Expected: All tests PASS (unit + integration).

- [ ] **Step 11: Commit**

```bash
git add helpdesk/
git commit -m "feat(helpdesk): scenario bootstrap, verification, scenario file + integration test

ScenarioBootstrapResource, VerificationResource, help-desk-basic.yaml.
Full pipeline: inject chat → create ticket → classify → verify.

Refs casehubio/parent#408"
```
