# Durable EventStore Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #113 — feat: durable EventStore implementations (JDBC, Redis)
**Issue group:** #113

**Goal:** Provide durable EventStore backends (PostgreSQL JDBC, Redis Streams) as optional Maven modules activated by classpath presence, following the CDI priority ladder protocol.

**Architecture:** The EventStore SPI gets two breaking changes (timestamp on StoredEvent, limit on replay). Two new backend modules (`push-store-jdbc` at Tier 2, `push-store-redis` at Tier 3) implement the SPI. Each module is self-contained: classpath presence activates it, no consumer configuration needed.

**Tech Stack:** Java 21, Quarkus ARC (CDI), raw JDBC + Agroal (PostgreSQL), quarkus-redis-client (Redis Streams), Quarkus @Scheduled (retention), JUnit 5 + AssertJ

## Global Constraints

- Pre-release platform — breaking changes cost nothing
- CDI priority ladder: `@DefaultBean` (InMemory) < `@ApplicationScoped` (JDBC) < `@Alternative @Priority(1)` (Redis)
- Each backend in its own Maven module — `cdi-classpath-presence-requires-module-separation.md`
- Jandex index required for ARC bean discovery — all modules use `jandex-maven-plugin`
- Redis 6.2+ required for exclusive XRANGE prefix
- `push_topic_seq` rows are never deleted — topics survive event expiry

---

### Task 1: SPI Changes — StoredEvent, EventStore, InMemoryEventStore

**Files:**
- Modify: `backend/push/src/main/java/io/casehub/pages/push/StoredEvent.java`
- Modify: `backend/push/src/main/java/io/casehub/pages/push/EventStore.java`
- Modify: `backend/push/src/main/java/io/casehub/pages/push/InMemoryEventStore.java`
- Modify: `backend/push/src/test/java/io/casehub/pages/push/InMemoryEventStoreTest.java`
- Modify: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/EventStoreOverrideTest.java`
- Modify: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/BroadcastIntegrationTest.java`
- Modify: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushProducersTest.java`
- Modify: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/CustomCapacityTest.java`

**Interfaces:**
- Produces: `StoredEvent(String topic, String payloadJson, long seq, Instant createdAt)` — record with 4 fields
- Produces: `EventStore.replay(String topic, long sinceSeq, int limit)` — returns `List<StoredEvent>` with at most `limit` entries
- Produces: `EventStore.append(String topic, String payloadJson)` — returns `long` seq, stores `Instant.now()` as createdAt

- [ ] **Step 1: Update StoredEvent record**

Use `ide_replace_member` on `StoredEvent` (member = `StoredEvent`) in `backend/push/src/main/java/io/casehub/pages/push/StoredEvent.java`:

```java
public record StoredEvent(String topic, String payloadJson, long seq, Instant createdAt) {
    public StoredEvent {
        Objects.requireNonNull(topic, "topic");
        Objects.requireNonNull(payloadJson, "payloadJson");
        Objects.requireNonNull(createdAt, "createdAt");
    }
}
```

Add `import java.time.Instant;` to the file imports.

- [ ] **Step 2: Update EventStore.replay() signature**

Use `ide_replace_member` on `replay` in `backend/push/src/main/java/io/casehub/pages/push/EventStore.java`:

```java
/**
 * Replay events with sequence numbers greater than {@code sinceSeq}.
 *
 * @param topic topic name (not null)
 * @param sinceSeq last known sequence; returns events with seq &gt; sinceSeq
 * @param limit maximum number of events to return (must be &gt; 0)
 * @return stored events ordered by seq ascending (empty if no events match)
 * @throws IllegalArgumentException if limit is not positive
 */
List<StoredEvent> replay(String topic, long sinceSeq, int limit);
```

- [ ] **Step 3: Update InMemoryEventStore to match new SPI**

Use `ide_replace_member` on `append` in `InMemoryEventStore.java` — the `TopicBuffer.append` method:

```java
long append(String payloadJson) {
    synchronized (lock) {
        long seq = ++seqCounter;
        StoredEvent event = new StoredEvent(topic, payloadJson, seq, java.time.Instant.now());
        events.addLast(event);
        if (events.size() > maxEventsPerTopic) {
            events.removeFirst();
        }
        return seq;
    }
}
```

Use `ide_replace_member` on `replay` in `InMemoryEventStore.java` — the outer class method:

```java
@Override
public List<StoredEvent> replay(String topic, long sinceSeq, int limit) {
    Objects.requireNonNull(topic, "topic");
    if (limit <= 0) {
        throw new IllegalArgumentException("limit must be positive");
    }

    TopicBuffer buffer = buffers.get(topic);
    if (buffer == null) {
        return List.of();
    }
    return buffer.replay(sinceSeq, limit);
}
```

Use `ide_replace_member` on `replay` in `InMemoryEventStore.java` — the `TopicBuffer.replay` method:

```java
List<StoredEvent> replay(long sinceSeq, int limit) {
    synchronized (lock) {
        return events.stream()
            .filter(e -> e.seq() > sinceSeq)
            .limit(limit)
            .toList();
    }
}
```

- [ ] **Step 4: Update InMemoryEventStoreTest**

Update every `replay()` call to include a limit parameter. Use `Integer.MAX_VALUE` for tests that don't test limiting specifically. Add `import java.time.Instant;`.

Key changes (apply to all replay calls in the file):
- `store.replay("topic-a", 2)` → `store.replay("topic-a", 2, Integer.MAX_VALUE)`
- `store.replay("nonexistent", 0)` → `store.replay("nonexistent", 0, Integer.MAX_VALUE)`
- `store.replay("topic-a", 0)` → `store.replay("topic-a", 0, Integer.MAX_VALUE)`
- `store.replay("shared-topic", 0)` → `store.replay("shared-topic", 0, Integer.MAX_VALUE)`
- `store.replay("shared-topic", j % 10)` → `store.replay("shared-topic", j % 10, Integer.MAX_VALUE)`

Add new tests at the end of the class:

```java
@Test
void replay_respects_limit() {
    var store = new InMemoryEventStore(100);
    for (int i = 0; i < 10; i++) {
        store.append("topic-a", "{\"i\":" + i + "}");
    }

    List<StoredEvent> events = store.replay("topic-a", 0, 3);

    assertEquals(3, events.size(), "Should return at most 3 events");
    assertEquals(1, events.get(0).seq());
    assertEquals(3, events.get(2).seq());
}

@Test
void replay_limit_greater_than_available() {
    var store = new InMemoryEventStore(10);
    store.append("topic-a", "{\"i\":1}");
    store.append("topic-a", "{\"i\":2}");

    List<StoredEvent> events = store.replay("topic-a", 0, 100);

    assertEquals(2, events.size(), "Should return all available when limit exceeds count");
}

@Test
void replay_rejects_non_positive_limit() {
    var store = new InMemoryEventStore(10);
    assertThrows(IllegalArgumentException.class, () -> store.replay("t", 0, 0));
    assertThrows(IllegalArgumentException.class, () -> store.replay("t", 0, -1));
}

@Test
void stored_event_has_created_at() {
    var store = new InMemoryEventStore(10);
    Instant before = Instant.now();
    store.append("topic-a", "{\"data\":1}");
    Instant after = Instant.now();

    List<StoredEvent> events = store.replay("topic-a", 0, Integer.MAX_VALUE);
    Instant created = events.get(0).createdAt();

    assertNotNull(created);
    assertFalse(created.isBefore(before), "createdAt should be >= test start");
    assertFalse(created.isAfter(after), "createdAt should be <= test end");
}
```

- [ ] **Step 5: Run push module tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml test`
Expected: ALL PASS (all existing tests updated, new tests pass)

- [ ] **Step 6: Update push-runtime tests — EventStoreOverrideTest**

Update the `StubEventStore.replay()` signature in `EventStoreOverrideTest.java`:

```java
@Override
public List<StoredEvent> replay(String topic, long sinceSeq, int limit) {
    return List.of();
}
```

- [ ] **Step 7: Update push-runtime tests — BroadcastIntegrationTest**

Update replay calls:
- Line 51: `eventStore.replay("nobody:listens", 0)` → `eventStore.replay("nobody:listens", 0, Integer.MAX_VALUE)`
- Line 70: `eventStore.replay("t", 0)` → `eventStore.replay("t", 0, Integer.MAX_VALUE)`

- [ ] **Step 8: Update push-runtime tests — PushProducersTest**

Line 35: `eventStore.replay("capacity-test", 0)` → `eventStore.replay("capacity-test", 0, Integer.MAX_VALUE)`

- [ ] **Step 9: Update push-runtime tests — CustomCapacityTest**

Line 31: `eventStore.replay("cap-test", 0)` → `eventStore.replay("cap-test", 0, Integer.MAX_VALUE)`

- [ ] **Step 10: Run push-runtime tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push-runtime/pom.xml test`
Expected: ALL PASS

- [ ] **Step 11: Run full backend build**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml test`
Expected: ALL PASS across all modules

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/push/src backend/push-runtime/src
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#113): add createdAt to StoredEvent and limit to EventStore.replay()

Breaking SPI changes:
- StoredEvent: add Instant createdAt field
- EventStore.replay(): add int limit parameter (must be > 0)
- InMemoryEventStore: updated to match
- All push-runtime test callers updated

Refs #113"
```

---

### Task 2: JDBC EventStore Module

**Files:**
- Modify: `backend/pom.xml` (add module + dependency management)
- Create: `backend/push-store-jdbc/pom.xml`
- Create: `backend/push-store-jdbc/src/main/java/io/casehub/pages/push/store/jdbc/JdbcEventStore.java`
- Create: `backend/push-store-jdbc/src/main/java/io/casehub/pages/push/store/jdbc/JdbcEventStoreRetention.java`
- Create: `backend/push-store-jdbc/src/test/java/io/casehub/pages/push/store/jdbc/JdbcEventStoreTest.java`
- Create: `backend/push-store-jdbc/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `EventStore` interface (from Task 1), `StoredEvent` record (from Task 1)
- Produces: `JdbcEventStore` — `@ApplicationScoped` bean implementing `EventStore`, displaces `@DefaultBean` InMemoryEventStore

- [ ] **Step 1: Add module to parent POM**

In `backend/pom.xml`, add `<module>push-store-jdbc</module>` to the modules list (after `push-runtime`). Add dependency management entries:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-push-store-jdbc</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Create push-store-jdbc/pom.xml**

Create `backend/push-store-jdbc/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-pages-backend</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-pages-push-store-jdbc</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Pages Push Store — JDBC</name>
    <description>PostgreSQL-backed EventStore for casehub-pages push protocol</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-push</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-scheduler</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-agroal</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-postgresql</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-agroal</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Create test application.properties**

Create `backend/push-store-jdbc/src/test/resources/application.properties`:

```properties
quarkus.datasource.devservices.enabled=true
casehub.pages.push.store.jdbc.max-events-per-topic=100
casehub.pages.push.store.jdbc.ttl=P0D
```

- [ ] **Step 4: Write the failing test — JdbcEventStoreTest**

Create `backend/push-store-jdbc/src/test/java/io/casehub/pages/push/store/jdbc/JdbcEventStoreTest.java`:

```java
package io.casehub.pages.push.store.jdbc;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.StoredEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class JdbcEventStoreTest {

    @Inject EventStore eventStore;

    @Test
    void is_jdbc_implementation() {
        assertThat(eventStore).isInstanceOf(JdbcEventStore.class);
    }

    @Test
    void append_assigns_monotonic_seq() {
        long seq1 = eventStore.append("jdbc-mono", "{\"v\":1}");
        long seq2 = eventStore.append("jdbc-mono", "{\"v\":2}");
        long seq3 = eventStore.append("jdbc-mono", "{\"v\":3}");

        assertThat(seq1).isEqualTo(1);
        assertThat(seq2).isEqualTo(2);
        assertThat(seq3).isEqualTo(3);
    }

    @Test
    void append_per_topic_isolation() {
        long seqA = eventStore.append("jdbc-iso-a", "{\"a\":1}");
        long seqB = eventStore.append("jdbc-iso-b", "{\"b\":1}");

        assertThat(seqA).isEqualTo(1);
        assertThat(seqB).isEqualTo(1);
    }

    @Test
    void replay_returns_events_after_sinceSeq() {
        eventStore.append("jdbc-replay", "{\"v\":1}");
        eventStore.append("jdbc-replay", "{\"v\":2}");
        eventStore.append("jdbc-replay", "{\"v\":3}");
        eventStore.append("jdbc-replay", "{\"v\":4}");

        List<StoredEvent> events = eventStore.replay("jdbc-replay", 2, Integer.MAX_VALUE);

        assertThat(events).hasSize(2);
        assertThat(events.get(0).seq()).isEqualTo(3);
        assertThat(events.get(1).seq()).isEqualTo(4);
    }

    @Test
    void replay_respects_limit() {
        for (int i = 0; i < 10; i++) {
            eventStore.append("jdbc-limit", "{\"i\":" + i + "}");
        }

        List<StoredEvent> events = eventStore.replay("jdbc-limit", 0, 3);

        assertThat(events).hasSize(3);
        assertThat(events.get(0).seq()).isEqualTo(1);
        assertThat(events.get(2).seq()).isEqualTo(3);
    }

    @Test
    void replay_empty_topic() {
        List<StoredEvent> events = eventStore.replay("jdbc-nonexistent", 0, Integer.MAX_VALUE);
        assertThat(events).isEmpty();
    }

    @Test
    void replay_rejects_non_positive_limit() {
        assertThatThrownBy(() -> eventStore.replay("t", 0, 0))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void topics_returns_all_with_events() {
        eventStore.append("jdbc-topics-a", "{}");
        eventStore.append("jdbc-topics-b", "{}");

        Set<String> topics = eventStore.topics();

        assertThat(topics).contains("jdbc-topics-a", "jdbc-topics-b");
    }

    @Test
    void stored_event_has_timestamp() {
        Instant before = Instant.now().minusSeconds(1);
        eventStore.append("jdbc-ts", "{\"v\":1}");
        Instant after = Instant.now().plusSeconds(1);

        List<StoredEvent> events = eventStore.replay("jdbc-ts", 0, Integer.MAX_VALUE);

        assertThat(events.get(0).createdAt()).isAfter(before).isBefore(after);
    }

    @Test
    void bounded_capacity_eviction() {
        for (int i = 0; i < 110; i++) {
            eventStore.append("jdbc-cap", "{\"i\":" + i + "}");
        }

        List<StoredEvent> events = eventStore.replay("jdbc-cap", 0, Integer.MAX_VALUE);

        assertThat(events).hasSize(100);
        assertThat(events.get(0).seq()).isEqualTo(11);
    }

    @Test
    void concurrent_append_thread_safety() throws InterruptedException {
        int threadCount = 10;
        int appendsPerThread = 100;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                try {
                    for (int j = 0; j < appendsPerThread; j++) {
                        eventStore.append("jdbc-concurrent", "{\"j\":" + j + "}");
                    }
                } finally {
                    latch.countDown();
                }
            });
        }

        assertThat(latch.await(30, TimeUnit.SECONDS)).isTrue();
        executor.shutdown();

        List<StoredEvent> all = eventStore.replay("jdbc-concurrent", 0, Integer.MAX_VALUE);
        assertThat(all).hasSize(threadCount * appendsPerThread);

        for (int i = 1; i < all.size(); i++) {
            assertThat(all.get(i).seq()).isEqualTo(all.get(i - 1).seq() + 1);
        }
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push-store-jdbc/pom.xml test`
Expected: FAIL — `JdbcEventStore` class does not exist

- [ ] **Step 6: Implement JdbcEventStore**

Create `backend/push-store-jdbc/src/main/java/io/casehub/pages/push/store/jdbc/JdbcEventStore.java`:

```java
package io.casehub.pages.push.store.jdbc;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.StoredEvent;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.time.Instant;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Objects;
import java.util.Set;

@ApplicationScoped
public class JdbcEventStore implements EventStore {

    @Inject
    DataSource dataSource;

    @ConfigProperty(name = "casehub.pages.push.store.jdbc.max-events-per-topic",
                    defaultValue = "10000")
    int maxEventsPerTopic;

    @PostConstruct
    void initSchema() {
        try (Connection conn = dataSource.getConnection()) {
            try (var stmt = conn.createStatement()) {
                stmt.execute("""
                    CREATE TABLE IF NOT EXISTS push_topic_seq (
                        topic       VARCHAR(255) PRIMARY KEY,
                        next_seq    BIGINT NOT NULL DEFAULT 0
                    )""");
                stmt.execute("""
                    CREATE TABLE IF NOT EXISTS push_events (
                        topic        VARCHAR(255) NOT NULL,
                        seq          BIGINT NOT NULL,
                        payload_json TEXT NOT NULL,
                        created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
                        PRIMARY KEY (topic, seq)
                    )""");
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to initialize push event store schema", e);
        }
    }

    @Override
    public long append(String topic, String payloadJson) {
        Objects.requireNonNull(topic, "topic");
        Objects.requireNonNull(payloadJson, "payloadJson");

        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            try {
                long seq;
                try (PreparedStatement ps = conn.prepareStatement(
                        "INSERT INTO push_topic_seq (topic, next_seq) VALUES (?, 1) " +
                        "ON CONFLICT (topic) DO UPDATE SET next_seq = push_topic_seq.next_seq + 1 " +
                        "RETURNING next_seq")) {
                    ps.setString(1, topic);
                    try (ResultSet rs = ps.executeQuery()) {
                        rs.next();
                        seq = rs.getLong(1);
                    }
                }

                try (PreparedStatement ps = conn.prepareStatement(
                        "INSERT INTO push_events (topic, seq, payload_json, created_at) " +
                        "VALUES (?, ?, ?, now())")) {
                    ps.setString(1, topic);
                    ps.setLong(2, seq);
                    ps.setString(3, payloadJson);
                    ps.executeUpdate();
                }

                long threshold = seq - maxEventsPerTopic;
                if (threshold > 0) {
                    try (PreparedStatement ps = conn.prepareStatement(
                            "DELETE FROM push_events WHERE topic = ? AND seq <= ?")) {
                        ps.setString(1, topic);
                        ps.setLong(2, threshold);
                        ps.executeUpdate();
                    }
                }

                conn.commit();
                return seq;
            } catch (Exception e) {
                conn.rollback();
                throw e;
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to append event to topic: " + topic, e);
        }
    }

    @Override
    public List<StoredEvent> replay(String topic, long sinceSeq, int limit) {
        Objects.requireNonNull(topic, "topic");
        if (limit <= 0) {
            throw new IllegalArgumentException("limit must be positive");
        }

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT topic, seq, payload_json, created_at " +
                     "FROM push_events WHERE topic = ? AND seq > ? " +
                     "ORDER BY seq ASC LIMIT ?")) {
            ps.setString(1, topic);
            ps.setLong(2, sinceSeq);
            ps.setInt(3, limit);

            List<StoredEvent> results = new ArrayList<>();
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    results.add(new StoredEvent(
                            rs.getString("topic"),
                            rs.getString("payload_json"),
                            rs.getLong("seq"),
                            rs.getTimestamp("created_at").toInstant()));
                }
            }
            return results;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to replay events for topic: " + topic, e);
        }
    }

    @Override
    public Set<String> topics() {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement("SELECT topic FROM push_topic_seq");
             ResultSet rs = ps.executeQuery()) {
            Set<String> topics = new HashSet<>();
            while (rs.next()) {
                topics.add(rs.getString("topic"));
            }
            return topics;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to list topics", e);
        }
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push-store-jdbc/pom.xml test`
Expected: ALL PASS

- [ ] **Step 8: Implement JdbcEventStoreRetention**

Create `backend/push-store-jdbc/src/main/java/io/casehub/pages/push/store/jdbc/JdbcEventStoreRetention.java`:

```java
package io.casehub.pages.push.store.jdbc;

import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.time.Duration;

@ApplicationScoped
public class JdbcEventStoreRetention {

    @Inject
    DataSource dataSource;

    @ConfigProperty(name = "casehub.pages.push.store.jdbc.ttl", defaultValue = "P7D")
    Duration ttl;

    @Scheduled(every = "${casehub.pages.push.store.jdbc.cleanup-interval:PT1H}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void cleanup() {
        if (ttl.isZero() || ttl.isNegative()) {
            return;
        }
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "DELETE FROM push_events WHERE created_at < now() - CAST(? AS INTERVAL)")) {
            ps.setString(1, ttl.toString().replace("PT", "").replace("P", "")
                    .replace("D", " days ").replace("H", " hours ").replace("M", " minutes ")
                    .replace("S", " seconds").trim());
            ps.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to run TTL cleanup", e);
        }
    }
}
```

Wait — PostgreSQL interval from Duration is cleaner via seconds:

```java
package io.casehub.pages.push.store.jdbc;

import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.time.Duration;

@ApplicationScoped
public class JdbcEventStoreRetention {

    @Inject
    DataSource dataSource;

    @ConfigProperty(name = "casehub.pages.push.store.jdbc.ttl", defaultValue = "P7D")
    Duration ttl;

    @Scheduled(every = "${casehub.pages.push.store.jdbc.cleanup-interval:PT1H}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void cleanup() {
        if (ttl.isZero() || ttl.isNegative()) {
            return;
        }
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "DELETE FROM push_events WHERE created_at < now() - make_interval(secs => ?)")) {
            ps.setLong(1, ttl.toSeconds());
            ps.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to run TTL cleanup", e);
        }
    }
}
```

- [ ] **Step 9: Add TTL retention test to JdbcEventStoreTest**

Add to `JdbcEventStoreTest.java`:

```java
@Test
void ttl_retention_removes_old_events() throws Exception {
    eventStore.append("jdbc-ttl", "{\"old\":true}");

    // Manually age the event by updating created_at directly
    try (Connection conn = ((JdbcEventStore) eventStore).dataSource.getConnection();
         PreparedStatement ps = conn.prepareStatement(
                 "UPDATE push_events SET created_at = now() - interval '30 days' WHERE topic = ?")) {
        ps.setString(1, "jdbc-ttl");
        ps.executeUpdate();
    }

    // Trigger retention manually
    var retention = io.quarkus.arc.Arc.container().instance(JdbcEventStoreRetention.class).get();
    retention.cleanup();

    List<StoredEvent> events = eventStore.replay("jdbc-ttl", 0, Integer.MAX_VALUE);
    assertThat(events).isEmpty();

    // Topic should still be listed (push_topic_seq not deleted)
    assertThat(eventStore.topics()).contains("jdbc-ttl");
}
```

Update test `application.properties` to set a short TTL for testing:
```properties
casehub.pages.push.store.jdbc.ttl=P1D
```

- [ ] **Step 10: Run all JDBC tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push-store-jdbc/pom.xml test`
Expected: ALL PASS

- [ ] **Step 11: Run full backend build**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml test`
Expected: ALL PASS

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/push-store-jdbc backend/pom.xml
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#113): add JDBC EventStore implementation (PostgreSQL)

New module: casehub-pages-push-store-jdbc
- @ApplicationScoped (Tier 2) — displaces @DefaultBean InMemoryEventStore
- Raw JDBC with counter table for per-topic monotonic sequences
- Capacity-based retention on every append
- @Scheduled time-based TTL cleanup
- Schema auto-created on startup via @PostConstruct
- DevServices PostgreSQL tests with full behavioral coverage

Refs #113"
```

---

### Task 3: Redis EventStore Module

**Files:**
- Modify: `backend/pom.xml` (add module + dependency management)
- Create: `backend/push-store-redis/pom.xml`
- Create: `backend/push-store-redis/src/main/java/io/casehub/pages/push/store/redis/RedisEventStore.java`
- Create: `backend/push-store-redis/src/main/java/io/casehub/pages/push/store/redis/RedisEventStoreRetention.java`
- Create: `backend/push-store-redis/src/test/java/io/casehub/pages/push/store/redis/RedisEventStoreTest.java`
- Create: `backend/push-store-redis/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `EventStore` interface (from Task 1), `StoredEvent` record (from Task 1)
- Produces: `RedisEventStore` — `@Alternative @Priority(1) @ApplicationScoped` bean implementing `EventStore`, displaces JdbcEventStore and InMemoryEventStore

- [ ] **Step 1: Add module to parent POM**

In `backend/pom.xml`, add `<module>push-store-redis</module>` to the modules list. Add dependency management:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-push-store-redis</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Create push-store-redis/pom.xml**

Create `backend/push-store-redis/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-pages-backend</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-pages-push-store-redis</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Pages Push Store — Redis</name>
    <description>Redis Streams-backed EventStore for casehub-pages push protocol</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-push</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-redis-client</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-scheduler</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Create test application.properties**

Create `backend/push-store-redis/src/test/resources/application.properties`:

```properties
quarkus.redis.devservices.enabled=true
casehub.pages.push.store.redis.max-events-per-topic=100
casehub.pages.push.store.redis.ttl=P0D
casehub.pages.push.store.redis.key-prefix=pushtest
```

- [ ] **Step 4: Write the failing test — RedisEventStoreTest**

Create `backend/push-store-redis/src/test/java/io/casehub/pages/push/store/redis/RedisEventStoreTest.java`:

```java
package io.casehub.pages.push.store.redis;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.StoredEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class RedisEventStoreTest {

    @Inject EventStore eventStore;

    @Test
    void is_redis_implementation() {
        assertThat(eventStore).isInstanceOf(RedisEventStore.class);
    }

    @Test
    void append_assigns_monotonic_seq() {
        long seq1 = eventStore.append("redis-mono", "{\"v\":1}");
        long seq2 = eventStore.append("redis-mono", "{\"v\":2}");
        long seq3 = eventStore.append("redis-mono", "{\"v\":3}");

        assertThat(seq1).isEqualTo(1);
        assertThat(seq2).isEqualTo(2);
        assertThat(seq3).isEqualTo(3);
    }

    @Test
    void append_per_topic_isolation() {
        long seqA = eventStore.append("redis-iso-a", "{\"a\":1}");
        long seqB = eventStore.append("redis-iso-b", "{\"b\":1}");

        assertThat(seqA).isEqualTo(1);
        assertThat(seqB).isEqualTo(1);
    }

    @Test
    void replay_returns_events_after_sinceSeq() {
        eventStore.append("redis-replay", "{\"v\":1}");
        eventStore.append("redis-replay", "{\"v\":2}");
        eventStore.append("redis-replay", "{\"v\":3}");
        eventStore.append("redis-replay", "{\"v\":4}");

        List<StoredEvent> events = eventStore.replay("redis-replay", 2, Integer.MAX_VALUE);

        assertThat(events).hasSize(2);
        assertThat(events.get(0).seq()).isEqualTo(3);
        assertThat(events.get(1).seq()).isEqualTo(4);
    }

    @Test
    void replay_respects_limit() {
        for (int i = 0; i < 10; i++) {
            eventStore.append("redis-limit", "{\"i\":" + i + "}");
        }

        List<StoredEvent> events = eventStore.replay("redis-limit", 0, 3);

        assertThat(events).hasSize(3);
        assertThat(events.get(0).seq()).isEqualTo(1);
        assertThat(events.get(2).seq()).isEqualTo(3);
    }

    @Test
    void replay_empty_topic() {
        List<StoredEvent> events = eventStore.replay("redis-nonexistent", 0, Integer.MAX_VALUE);
        assertThat(events).isEmpty();
    }

    @Test
    void replay_rejects_non_positive_limit() {
        assertThatThrownBy(() -> eventStore.replay("t", 0, 0))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void topics_returns_all_with_events() {
        eventStore.append("redis-topics-a", "{}");
        eventStore.append("redis-topics-b", "{}");

        Set<String> topics = eventStore.topics();

        assertThat(topics).contains("redis-topics-a", "redis-topics-b");
    }

    @Test
    void stored_event_has_timestamp() {
        Instant before = Instant.now().minusSeconds(1);
        eventStore.append("redis-ts", "{\"v\":1}");
        Instant after = Instant.now().plusSeconds(1);

        List<StoredEvent> events = eventStore.replay("redis-ts", 0, Integer.MAX_VALUE);

        assertThat(events.get(0).createdAt()).isAfter(before).isBefore(after);
    }

    @Test
    void stream_id_maps_to_seq() {
        long seq = eventStore.append("redis-id", "{\"v\":1}");

        List<StoredEvent> events = eventStore.replay("redis-id", 0, Integer.MAX_VALUE);

        assertThat(events.get(0).seq()).isEqualTo(seq);
    }

    @Test
    void xtrim_enforces_capacity() {
        for (int i = 0; i < 150; i++) {
            eventStore.append("redis-cap", "{\"i\":" + i + "}");
        }

        List<StoredEvent> events = eventStore.replay("redis-cap", 0, Integer.MAX_VALUE);

        // XTRIM MAXLEN ~ is approximate — allow some margin
        assertThat(events.size()).isLessThanOrEqualTo(110);
        assertThat(events.size()).isGreaterThanOrEqualTo(90);
    }

    @Test
    void concurrent_append_thread_safety() throws InterruptedException {
        int threadCount = 10;
        int appendsPerThread = 100;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                try {
                    for (int j = 0; j < appendsPerThread; j++) {
                        eventStore.append("redis-concurrent", "{\"j\":" + j + "}");
                    }
                } finally {
                    latch.countDown();
                }
            });
        }

        assertThat(latch.await(30, TimeUnit.SECONDS)).isTrue();
        executor.shutdown();

        List<StoredEvent> all = eventStore.replay("redis-concurrent", 0, Integer.MAX_VALUE);
        assertThat(all).hasSize(threadCount * appendsPerThread);

        // Verify monotonic (no duplicates) — gaps are acceptable for Redis
        for (int i = 1; i < all.size(); i++) {
            assertThat(all.get(i).seq()).isGreaterThan(all.get(i - 1).seq());
        }
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push-store-redis/pom.xml test`
Expected: FAIL — `RedisEventStore` class does not exist

- [ ] **Step 6: Implement RedisEventStore**

Create `backend/push-store-redis/src/main/java/io/casehub/pages/push/store/redis/RedisEventStore.java`:

```java
package io.casehub.pages.push.store.redis;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.StoredEvent;
import io.quarkus.redis.datasource.RedisDataSource;
import io.quarkus.redis.datasource.stream.StreamRange;
import io.quarkus.redis.datasource.stream.XAddArgs;
import io.quarkus.redis.datasource.stream.XTrimArgs;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

@Alternative
@Priority(1)
@ApplicationScoped
public class RedisEventStore implements EventStore {

    @Inject
    RedisDataSource redis;

    @ConfigProperty(name = "casehub.pages.push.store.redis.max-events-per-topic",
                    defaultValue = "10000")
    int maxEventsPerTopic;

    @ConfigProperty(name = "casehub.pages.push.store.redis.key-prefix",
                    defaultValue = "push")
    String keyPrefix;

    private String seqKey(String topic) {
        return keyPrefix + ":seq:" + topic;
    }

    private String eventsKey(String topic) {
        return keyPrefix + ":events:" + topic;
    }

    private String topicsKey() {
        return keyPrefix + ":topics";
    }

    @Override
    public long append(String topic, String payloadJson) {
        Objects.requireNonNull(topic, "topic");
        Objects.requireNonNull(payloadJson, "payloadJson");

        long seq = redis.value(Long.class).incr(seqKey(topic));

        redis.set(String.class).sadd(topicsKey(), topic);

        long createdAt = Instant.now().toEpochMilli();
        String streamId = seq + "-0";

        var commands = redis.stream(String.class);
        commands.xadd(eventsKey(topic),
                new XAddArgs().id(streamId),
                Map.of("payload", payloadJson, "createdAt", String.valueOf(createdAt)));

        commands.xtrim(eventsKey(topic), new XTrimArgs().maxlen(maxEventsPerTopic).nearlyExact());

        return seq;
    }

    @Override
    public List<StoredEvent> replay(String topic, long sinceSeq, int limit) {
        Objects.requireNonNull(topic, "topic");
        if (limit <= 0) {
            throw new IllegalArgumentException("limit must be positive");
        }

        String startId = "(" + sinceSeq + "-0";

        var commands = redis.stream(String.class);
        var entries = commands.xrange(eventsKey(topic),
                StreamRange.of(startId, "+"),
                limit);

        List<StoredEvent> results = new ArrayList<>();
        for (var entry : entries) {
            String id = entry.id();
            long seq = Long.parseLong(id.substring(0, id.indexOf('-')));
            Map<String, String> fields = entry.payload();
            String payload = fields.get("payload");
            Instant created = Instant.ofEpochMilli(Long.parseLong(fields.get("createdAt")));
            results.add(new StoredEvent(topic, payload, seq, created));
        }
        return results;
    }

    @Override
    public Set<String> topics() {
        Set<String> members = redis.set(String.class).smembers(topicsKey());
        return members != null ? members : Set.of();
    }
}
```

**Note:** The exact Quarkus Redis API may differ slightly depending on the quarkus-redis-client version. The implementation will need to be adjusted to match the actual `RedisDataSource` stream API during implementation. The key operations are:
- `INCR` via `redis.value(Long.class).incr(key)`
- `SADD` via `redis.set(String.class).sadd(key, member)`
- `SMEMBERS` via `redis.set(String.class).smembers(key)`
- `XADD` via `redis.stream(String.class).xadd(key, args, fields)`
- `XRANGE` via `redis.stream(String.class).xrange(key, range, count)`
- `XTRIM` via `redis.stream(String.class).xtrim(key, args)`

If the type-safe stream API doesn't support explicit IDs or exclusive ranges, fall back to `redis.execute("XADD", ...)` and `redis.execute("XRANGE", ...)` raw commands.

- [ ] **Step 7: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push-store-redis/pom.xml test`
Expected: ALL PASS (may require API adjustments in Step 6)

- [ ] **Step 8: Implement RedisEventStoreRetention**

Create `backend/push-store-redis/src/main/java/io/casehub/pages/push/store/redis/RedisEventStoreRetention.java`:

```java
package io.casehub.pages.push.store.redis;

import io.quarkus.redis.datasource.RedisDataSource;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Duration;
import java.time.Instant;
import java.util.Set;

@ApplicationScoped
public class RedisEventStoreRetention {

    @Inject
    RedisDataSource redis;

    @ConfigProperty(name = "casehub.pages.push.store.redis.ttl", defaultValue = "P7D")
    Duration ttl;

    @ConfigProperty(name = "casehub.pages.push.store.redis.key-prefix", defaultValue = "push")
    String keyPrefix;

    @Scheduled(every = "${casehub.pages.push.store.redis.cleanup-interval:PT1H}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void cleanup() {
        if (ttl.isZero() || ttl.isNegative()) {
            return;
        }

        long thresholdMillis = Instant.now().minus(ttl).toEpochMilli();
        Set<String> topics = redis.set(String.class).smembers(keyPrefix + ":topics");
        if (topics == null || topics.isEmpty()) {
            return;
        }

        var commands = redis.stream(String.class);
        for (String topic : topics) {
            String streamKey = keyPrefix + ":events:" + topic;
            boolean done = false;
            while (!done) {
                var entries = commands.xrange(streamKey, "-", "+", 100);
                if (entries.isEmpty()) {
                    done = true;
                    continue;
                }
                for (var entry : entries) {
                    String createdAtStr = entry.payload().get("createdAt");
                    if (createdAtStr != null && Long.parseLong(createdAtStr) < thresholdMillis) {
                        commands.xdel(streamKey, entry.id());
                    } else {
                        done = true;
                        break;
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 9: Run all Redis tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push-store-redis/pom.xml test`
Expected: ALL PASS

- [ ] **Step 10: Run full backend build**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/pom.xml test`
Expected: ALL PASS across all modules

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/push-store-redis backend/pom.xml
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#113): add Redis Streams EventStore implementation

New module: casehub-pages-push-store-redis
- @Alternative @Priority(1) (Tier 3) — displaces JDBC and InMemory
- Redis Streams with explicit seq-based IDs (XADD <seq>-0)
- INCR counter for per-topic monotonic sequences
- XTRIM MAXLEN ~ for capacity-based retention
- XRANGE+XDEL for time-based TTL cleanup (XTRIM MINID incompatible with seq IDs)
- Redis 6.2+ required for exclusive XRANGE prefix
- DevServices Redis tests with full behavioral coverage

Closes #113"
```
