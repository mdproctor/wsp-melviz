# Trie-Based TopicRegistry Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #119 — perf: trie-based pattern index for TopicRegistry wildcard matching
**Issue group:** #119

**Goal:** Replace TopicRegistry's dual-map internals with a segment trie,
reducing `connections()` and `matchedTopics()` from O(n) linear scans to
O(d) trie walks.

**Architecture:** Private `TrieNode` inner class replaces `exactTopics` and
`wildcardPatterns` ConcurrentHashMaps. Single trie root, same concurrency
primitives (ConcurrentHashMap for children, CopyOnWriteArraySet for
connection sets). `connectionToTopics` reverse index unchanged. Public API
unchanged. All 30 existing tests must pass without modification.

**Tech Stack:** Java 17, JUnit 5, AssertJ, Maven (casehub-pages-push module)

## Global Constraints

- No new dependencies — jackson-core only (already present)
- No public API changes to TopicRegistry
- `isValidTopicOrPattern()` and `matches()` static methods untouched
- Thread safety via ConcurrentHashMap + CopyOnWriteArraySet (no synchronized blocks)
- Semantic equivalence: trie walk results must exactly match what `matches()` would produce

## Pre-implementation reading

Before writing any code, read these protocols:
- `docs/protocols/casehub/pages-event-contract.md` — topic naming rules (`:` delimiter, `*`/`**` semantics)
- `docs/protocols/casehub/dataset-contract.md` — DatasetContract pattern (for platform consistency)
- Design spec: `docs/specs/2026-07-06-trie-topic-registry-design.md` — the reviewed spec is authoritative

---

### Task 1: Rewrite TopicRegistry internals from dual-map to trie

**Files:**
- Modify: `backend/push/src/main/java/io/casehub/pages/push/TopicRegistry.java`
- Modify: `backend/push/src/test/java/io/casehub/pages/push/TopicRegistryTest.java` (add 1 new test)

**Interfaces:**
- Consumes: nothing (standalone module, no callers in this repo)
- Produces: identical public API — `listen()`, `unlisten()`, `removeConnection()`, `connections()`, `matchedTopics()`, `isValidTopicOrPattern()`, `matches()`

#### Phase 1: Add the new test first (TDD — red)

- [ ] **Step 1: Add the `matchedTopics_double_star_zero_match` test**

The design review identified a coverage gap — `matchedTopics()` with `**`
matching zero trailing segments (the prefix itself). Add this test to
`TopicRegistryTest.java` after the existing `matchedTopics_double_star` test:

```java
@Test
void matchedTopics_double_star_zero_match() {
    var registry = new TopicRegistry();
    registry.listen("c1", List.of("debate"));
    registry.listen("c2", List.of("debate:abc"));
    assertThat(registry.matchedTopics("debate:**"))
        .containsExactlyInAnyOrder("debate", "debate:abc");
}
```

- [ ] **Step 2: Run to verify it passes with the current implementation**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml test -Dtest="TopicRegistryTest#matchedTopics_double_star_zero_match" -pl .
```

Expected: PASS (the current O(n) scan implementation handles this correctly).
This test is a safety net — it must continue passing after the trie rewrite.

- [ ] **Step 3: Run the full existing test suite as baseline**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml test -Dtest="TopicRegistryTest" -pl .
```

Expected: all 30+ tests PASS. Record the exact count — this is the baseline.

- [ ] **Step 4: Commit the new test**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/push/src/test/java/io/casehub/pages/push/TopicRegistryTest.java
git -C /Users/mdproctor/claude/casehub/pages commit -m "test: add matchedTopics double-star zero-match coverage

Refs #119"
```

#### Phase 2: Rewrite TopicRegistry internals (TDD — green)

- [ ] **Step 5: Replace TopicRegistry internals with trie**

Replace the body of `TopicRegistry.java` with the trie-based implementation.
Keep `isValidTopicOrPattern()` and `matches()` exactly as they are. Replace
everything else.

**New inner class — add as the first member of TopicRegistry:**

```java
private static final class TrieNode {
    final ConcurrentHashMap<String, TrieNode> children = new ConcurrentHashMap<>();
    final CopyOnWriteArraySet<String> connections = new CopyOnWriteArraySet<>();
    final CopyOnWriteArraySet<String> doubleStarConnections = new CopyOnWriteArraySet<>();
}
```

**New fields — replace the three maps with:**

```java
private final TrieNode root = new TrieNode();
private final ConcurrentHashMap<String, Set<String>> connectionToTopics = new ConcurrentHashMap<>();
```

Remove:
```java
private final ConcurrentHashMap<String, CopyOnWriteArraySet<String>> exactTopics = new ConcurrentHashMap<>();
private final ConcurrentHashMap<String, CopyOnWriteArraySet<String>> wildcardPatterns = new ConcurrentHashMap<>();
```

**`listen()` — rewrite to insert into trie:**

```java
public void listen(String connectionId, List<String> topics) {
    Set<String> connTopics = connectionToTopics.computeIfAbsent(connectionId,
            k -> ConcurrentHashMap.newKeySet());
    for (String topic : topics) {
        insertIntoTrie(connectionId, topic);
        connTopics.add(topic);
    }
}

private void insertIntoTrie(String connectionId, String pattern) {
    String[] segments = pattern.split(":", -1);
    TrieNode current = root;
    for (String segment : segments) {
        if ("**".equals(segment)) {
            current.doubleStarConnections.add(connectionId);
            return;
        }
        current = current.children.computeIfAbsent(segment, k -> new TrieNode());
    }
    current.connections.add(connectionId);
}
```

**`unlisten()` — rewrite to remove from trie:**

```java
public void unlisten(String connectionId, List<String> topics) {
    for (String topic : topics) {
        removeFromTrie(connectionId, topic);
    }
    Set<String> connTopics = connectionToTopics.get(connectionId);
    if (connTopics != null) {
        connTopics.removeAll(topics);
        if (connTopics.isEmpty()) {
            connectionToTopics.remove(connectionId);
        }
    }
}

private void removeFromTrie(String connectionId, String pattern) {
    String[] segments = pattern.split(":", -1);
    TrieNode current = root;
    for (String segment : segments) {
        if ("**".equals(segment)) {
            current.doubleStarConnections.remove(connectionId);
            return;
        }
        TrieNode child = current.children.get(segment);
        if (child == null) return;
        current = child;
    }
    current.connections.remove(connectionId);
}
```

**`removeConnection()` — rewrite to use trie removal:**

```java
public void removeConnection(String connectionId) {
    Set<String> topics = connectionToTopics.remove(connectionId);
    if (topics != null) {
        for (String topic : topics) {
            removeFromTrie(connectionId, topic);
        }
    }
}
```

**`connections()` — rewrite as trie walk:**

```java
public Set<String> connections(String topic) {
    Set<String> result = new HashSet<>();
    String[] segments = topic.split(":", -1);
    walkConnections(root, segments, 0, result);
    return Set.copyOf(result);
}

private void walkConnections(TrieNode node, String[] segments, int depth, Set<String> result) {
    result.addAll(node.doubleStarConnections);

    if (depth == segments.length) {
        result.addAll(node.connections);
        return;
    }

    String segment = segments[depth];

    TrieNode exactChild = node.children.get(segment);
    if (exactChild != null) {
        walkConnections(exactChild, segments, depth + 1, result);
    }

    TrieNode starChild = node.children.get("*");
    if (starChild != null) {
        walkConnections(starChild, segments, depth + 1, result);
    }
}
```

**`matchedTopics()` — rewrite as trie walk:**

```java
public Set<String> matchedTopics(String pattern) {
    if (!pattern.contains("*")) {
        return hasExactTopic(pattern) ? Set.of(pattern) : Set.of();
    }

    Set<String> result = new HashSet<>();
    String[] segments = pattern.split(":", -1);
    walkMatchedTopics(root, segments, 0, new ArrayList<>(), result);
    return Set.copyOf(result);
}

private boolean hasExactTopic(String topic) {
    String[] segments = topic.split(":", -1);
    TrieNode current = root;
    for (String segment : segments) {
        TrieNode child = current.children.get(segment);
        if (child == null) return false;
        current = child;
    }
    return !current.connections.isEmpty();
}

private void walkMatchedTopics(TrieNode node, String[] segments, int depth,
                                List<String> path, Set<String> result) {
    if (depth == segments.length) {
        if (!node.connections.isEmpty()) {
            result.add(String.join(":", path));
        }
        return;
    }

    String segment = segments[depth];

    if ("**".equals(segment)) {
        collectAllDescendantTopics(node, path, result);
        return;
    }

    if ("*".equals(segment)) {
        for (var entry : node.children.entrySet()) {
            if (!"*".equals(entry.getKey())) {
                List<String> childPath = new ArrayList<>(path);
                childPath.add(entry.getKey());
                walkMatchedTopics(entry.getValue(), segments, depth + 1, childPath, result);
            }
        }
        return;
    }

    TrieNode exactChild = node.children.get(segment);
    if (exactChild != null) {
        List<String> childPath = new ArrayList<>(path);
        childPath.add(segment);
        walkMatchedTopics(exactChild, segments, depth + 1, childPath, result);
    }
}

private void collectAllDescendantTopics(TrieNode node, List<String> path, Set<String> result) {
    if (!node.connections.isEmpty()) {
        result.add(String.join(":", path));
    }
    for (var entry : node.children.entrySet()) {
        if (!"*".equals(entry.getKey())) {
            List<String> childPath = new ArrayList<>(path);
            childPath.add(entry.getKey());
            collectAllDescendantTopics(entry.getValue(), childPath, result);
        }
    }
}
```

**New import needed:** `java.util.ArrayList`

- [ ] **Step 6: Run the full test suite**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml test -Dtest="TopicRegistryTest" -pl .
```

Expected: ALL tests PASS (same count as baseline from Step 3, plus the new test).
If any test fails: debug by comparing trie walk output against `matches()` for the
failing case. The semantic equivalence invariant from the spec is the contract.

- [ ] **Step 7: Commit the rewrite**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/push/src/main/java/io/casehub/pages/push/TopicRegistry.java
git -C /Users/mdproctor/claude/casehub/pages commit -m "perf: replace TopicRegistry dual-map with segment trie

O(n) linear scan → O(d) trie walk for connections() and matchedTopics().
TrieNode is a private inner class. Public API unchanged. All existing
tests pass without modification.

Closes #119"
```

#### Phase 3: Verify and commit spec

- [ ] **Step 8: Run the full push module build**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml verify -pl .
```

Expected: BUILD SUCCESS — all tests pass, no compilation errors.

---

## Verification

After completion:

1. **All 31+ tests pass** — `mvn test -Dtest=TopicRegistryTest`
2. **No public API changes** — `TopicRegistry`'s public method signatures are identical
3. **No new dependencies** — `pom.xml` unchanged
4. **Semantic equivalence** — for any topic/pattern pair, `connections()` returns the
   same set as a manual scan with `matches()` would produce
5. **Thread safety preserved** — concurrent test passes (8 threads × 1000 ops)
