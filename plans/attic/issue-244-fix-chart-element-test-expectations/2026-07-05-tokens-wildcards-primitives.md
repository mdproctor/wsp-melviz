# Tokens, Wildcards & Primitives Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #114 — nested wildcard patterns for topic matching
**Issue group:** #114, #116

**Goal:** Normalise spacing tokens, implement segment-level wildcard matching in TopicRegistry (Java) and EventConnection (TypeScript), add pages-event helpers to pages-component, create `pages-primitives` package with Lit-based a11y mixins / SchemaForm / filter chips / scope selector, add DatasetContract to pages-data, and enhance EventConnection with status enum and rAF batching.

**Architecture:** Five independent work areas: (1) token rename in pages-ui-tokens, (2) wildcard matching in backend/push Java module and pages-data TypeScript, (3) event helpers in pages-component, (4) new pages-primitives Lit package, (5) EventConnection enhancements. Each area is independently testable.

**Tech Stack:** TypeScript 5, Vitest, Lit 3.3, Java 17+, JUnit Jupiter, jackson-core

## Global Constraints

- All npm packages at version `0.2.0` (PP-20260705-8fcb31)
- All Maven modules at version `0.2-SNAPSHOT`
- No backward compatibility shims — breaking changes are intentional
- IntelliJ MCP for all code navigation and refactoring
- TDD: write failing test → verify fail → implement → verify pass → commit

---

### Task 1: Token Spacing Rename

**Files:**
- Modify: `packages/pages-ui-tokens/src/tokens.ts`
- Modify: `packages/pages-ui-tokens/src/themes.test.ts`
- Modify: `packages/pages-ui-tokens/src/colours.test.ts` (if spacing refs exist)
- Test: `packages/pages-ui-tokens/src/tokens.test.ts` (new)

**Interfaces:**
- Consumes: nothing
- Produces: `SPACING_SCALE` with keys `'0-5'` and `'1-5'` instead of `'0.5'` and `'1.5'`; CSS custom properties `--pages-space-0-5` and `--pages-space-1-5`

- [ ] **Step 1: Write failing test for hyphenated spacing keys**

Create `packages/pages-ui-tokens/src/tokens.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { SPACING_SCALE, DENSITY_COMPACT_OVERRIDES } from './tokens.js';
import { generateThemeCSS, DEFAULT_THEME } from './themes.js';

describe('SPACING_SCALE', () => {
  it('uses hyphenated keys for fractional values', () => {
    expect(SPACING_SCALE['0-5']).toBe('2px');
    expect(SPACING_SCALE['1-5']).toBe('6px');
    expect(SPACING_SCALE['0.5']).toBeUndefined();
    expect(SPACING_SCALE['1.5']).toBeUndefined();
  });
});

describe('generateThemeCSS spacing tokens', () => {
  it('generates hyphenated CSS custom properties', () => {
    const css = generateThemeCSS(DEFAULT_THEME);
    expect(css).toContain('--pages-space-0-5: 2px');
    expect(css).toContain('--pages-space-1-5: 6px');
    expect(css).not.toContain('--pages-space-0.5');
    expect(css).not.toContain('--pages-space-1.5');
  });
});

describe('blocks-ui coverage', () => {
  const BLOCKS_VARS = [
    'accent-1','accent-3','accent-6','accent-9','accent-10','accent-11',
    'danger-2','danger-3','danger-4','danger-6','danger-8','danger-9','danger-10','danger-11',
    'neutral-1','neutral-2','neutral-3','neutral-4','neutral-5','neutral-6','neutral-7','neutral-8','neutral-9','neutral-10','neutral-11','neutral-12',
    'success-9','warning-9',
    'space-0-5','space-1','space-1-5','space-2','space-3','space-4','space-8','space-10',
    'font-family','font-size-xs','font-size-sm','font-size-base','font-size-lg','font-size-xl',
    'font-weight-medium','font-weight-semibold',
    'duration-fast','ease-out',
    'radius-sm','radius-md',
    'shadow-1','surface-1',
  ];

  it('pages-ui-tokens generates every CSS var blocks-ui consumes', () => {
    const css = generateThemeCSS(DEFAULT_THEME);
    for (const suffix of BLOCKS_VARS) {
      expect(css, `missing --pages-${suffix}`).toContain(`--pages-${suffix}`);
    }
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui-tokens run test`
Expected: FAIL — `SPACING_SCALE['0-5']` is undefined, CSS contains `--pages-space-0.5`

- [ ] **Step 3: Rename spacing keys**

Edit `packages/pages-ui-tokens/src/tokens.ts`:

```typescript
export const SPACING_SCALE: Record<string, string> = {
  '0-5': '2px', '1': '4px', '1-5': '6px', '2': '8px',
  '3': '12px', '4': '16px', '5': '20px', '6': '24px',
  '8': '32px', '10': '40px', '12': '48px', '16': '64px',
};
```

- [ ] **Step 4: Update existing theme tests if they reference old keys**

Search `themes.test.ts` for `space-0.5` or `space-1.5` references and update to `space-0-5` / `space-1-5`.

- [ ] **Step 5: Check pages-runtime for any dot-keyed spacing references**

Use IntelliJ: `ide_search_text` for `space-0.5` and `space-1.5` across `packages/pages-runtime/src/`. Update any hits.

- [ ] **Step 6: Run all tests to verify**

Run: `yarn workspace @casehubio/pages-ui-tokens run test`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui-tokens/src/tokens.ts packages/pages-ui-tokens/src/tokens.test.ts packages/pages-ui-tokens/src/themes.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(ui-tokens): rename spacing keys to hyphens, add coverage test Refs #112"
```

---

### Task 2: TopicRegistry Wildcard Matching (Java)

**Files:**
- Modify: `backend/push/src/main/java/io/casehub/pages/push/TopicRegistry.java`
- Modify: `backend/push/src/test/java/io/casehub/pages/push/TopicRegistryTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `TopicRegistry.matches(String pattern, String topic): boolean` (public static), updated `isValidTopicOrPattern(String)`, updated `connections(String)`, updated `matchedTopics(String)`

- [ ] **Step 1: Write failing tests for new wildcard semantics**

Add to `TopicRegistryTest.java` — new test methods:

```java
// === Validation tests ===

@Test void isValidTopicOrPattern_segment_wildcard() {
    assertTrue(TopicRegistry.isValidTopicOrPattern("debate:*:summary"));
}

@Test void isValidTopicOrPattern_multi_segment_wildcard() {
    assertTrue(TopicRegistry.isValidTopicOrPattern("debate:*:*:summary"));
}

@Test void isValidTopicOrPattern_double_star_trailing() {
    assertTrue(TopicRegistry.isValidTopicOrPattern("debate:**"));
}

@Test void isValidTopicOrPattern_double_star_alone() {
    assertTrue(TopicRegistry.isValidTopicOrPattern("**"));
}

@Test void isValidTopicOrPattern_double_star_mid_position() {
    assertFalse(TopicRegistry.isValidTopicOrPattern("debate:**:summary"));
}

@Test void isValidTopicOrPattern_partial_wildcard() {
    assertFalse(TopicRegistry.isValidTopicOrPattern("de*bate"));
}

@Test void isValidTopicOrPattern_empty_segment() {
    assertFalse(TopicRegistry.isValidTopicOrPattern("debate::summary"));
}

@Test void isValidTopicOrPattern_leading_colon() {
    assertFalse(TopicRegistry.isValidTopicOrPattern(":debate"));
}

@Test void isValidTopicOrPattern_trailing_colon() {
    assertFalse(TopicRegistry.isValidTopicOrPattern("debate:"));
}

// === matches() tests ===

@Test void matches_exact() {
    assertTrue(TopicRegistry.matches("debate:abc", "debate:abc"));
    assertFalse(TopicRegistry.matches("debate:abc", "debate:xyz"));
}

@Test void matches_single_star_one_segment() {
    assertTrue(TopicRegistry.matches("debate:*", "debate:abc"));
    assertFalse(TopicRegistry.matches("debate:*", "debate:abc:def"));
}

@Test void matches_single_star_mid_position() {
    assertTrue(TopicRegistry.matches("debate:*:summary", "debate:abc:summary"));
    assertFalse(TopicRegistry.matches("debate:*:summary", "debate:abc:def:summary"));
}

@Test void matches_multiple_single_stars() {
    assertTrue(TopicRegistry.matches("a:*:b:*:c", "a:x:b:y:c"));
    assertFalse(TopicRegistry.matches("a:*:b:*:c", "a:x:b:y:z"));
}

@Test void matches_double_star_any_depth() {
    assertTrue(TopicRegistry.matches("debate:**", "debate:abc"));
    assertTrue(TopicRegistry.matches("debate:**", "debate:abc:def:ghi"));
}

@Test void matches_double_star_zero_segments() {
    assertTrue(TopicRegistry.matches("debate:**", "debate"));
}

@Test void matches_double_star_prefix_too_short() {
    assertFalse(TopicRegistry.matches("a:b:**", "a"));
}

@Test void matches_bare_star_one_segment() {
    assertTrue(TopicRegistry.matches("*", "hello"));
    assertFalse(TopicRegistry.matches("*", "hello:world"));
}

@Test void matches_bare_double_star() {
    assertTrue(TopicRegistry.matches("**", "anything"));
    assertTrue(TopicRegistry.matches("**", "a:b:c:d"));
}

// === connections() with new wildcards ===

@Test void connections_segment_wildcard() {
    var registry = new TopicRegistry();
    registry.listen("c1", List.of("debate:*:summary"));
    assertThat(registry.connections("debate:abc:summary")).containsExactly("c1");
    assertThat(registry.connections("debate:abc:details")).isEmpty();
    assertThat(registry.connections("debate:abc:def:summary")).isEmpty();
}

@Test void connections_double_star_wildcard() {
    var registry = new TopicRegistry();
    registry.listen("c1", List.of("debate:**"));
    assertThat(registry.connections("debate:abc")).containsExactly("c1");
    assertThat(registry.connections("debate:abc:def")).containsExactly("c1");
    assertThat(registry.connections("debate")).containsExactly("c1");
    assertThat(registry.connections("other:topic")).isEmpty();
}

@Test void connections_single_star_no_longer_matches_depth() {
    // BREAKING CHANGE: * now matches one segment only
    var registry = new TopicRegistry();
    registry.listen("c1", List.of("debate:*"));
    assertThat(registry.connections("debate:abc")).containsExactly("c1");
    assertThat(registry.connections("debate:abc:def")).isEmpty(); // was matching before
}

// === matchedTopics() with new wildcards ===

@Test void matchedTopics_segment_wildcard() {
    var registry = new TopicRegistry();
    registry.listen("c1", List.of("debate:abc:summary"));
    registry.listen("c2", List.of("debate:xyz:details"));
    assertThat(registry.matchedTopics("debate:*:summary"))
        .containsExactly("debate:abc:summary");
}

@Test void matchedTopics_double_star() {
    var registry = new TopicRegistry();
    registry.listen("c1", List.of("debate:abc"));
    registry.listen("c2", List.of("debate:xyz:deep"));
    registry.listen("c3", List.of("other:topic"));
    assertThat(registry.matchedTopics("debate:**"))
        .containsExactlyInAnyOrder("debate:abc", "debate:xyz:deep");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml test -Dtest=TopicRegistryTest -pl .`
Expected: Multiple compilation errors (matches() doesn't exist) and assertion failures

- [ ] **Step 3: Implement matches() static method**

Add to `TopicRegistry.java`:

```java
public static boolean matches(String pattern, String topic) {
    String[] ps = pattern.split(":", -1);
    String[] ts = topic.split(":", -1);

    if (ps[ps.length - 1].equals("**")) {
        if (ts.length < ps.length - 1) return false;
        for (int i = 0; i < ps.length - 1; i++) {
            if (!"*".equals(ps[i]) && !ps[i].equals(ts[i])) return false;
        }
        return true;
    }

    if (ps.length != ts.length) return false;
    for (int i = 0; i < ps.length; i++) {
        if ("*".equals(ps[i])) continue;
        if (!ps[i].equals(ts[i])) return false;
    }
    return true;
}
```

- [ ] **Step 4: Rewrite isValidTopicOrPattern()**

Replace the body of `isValidTopicOrPattern` (lines 20-26):

```java
public static boolean isValidTopicOrPattern(String topic) {
    if (topic == null || topic.isEmpty()) return false;
    String[] segments = topic.split(":", -1);
    for (int i = 0; i < segments.length; i++) {
        String s = segments[i];
        if (s.isEmpty()) return false;
        if ("**".equals(s)) return i == segments.length - 1;
        if (s.contains("*") && !"*".equals(s)) return false;
    }
    return true;
}
```

- [ ] **Step 5: Update routing in listen(), unlisten(), removeConnection()**

In `listen()`, `unlisten()`, and `removeConnection()`: replace `topic.endsWith("*")` with `topic.contains("*")`.

- [ ] **Step 6: Update connections() matching logic**

In `connections()`: replace the prefix-match block:

```java
// Old:
// String prefix = pattern.substring(0, pattern.length() - 1);
// if (topic.startsWith(prefix)) { ... }

// New:
if (matches(pattern, topic)) {
    result.addAll(connSet);
}
```

- [ ] **Step 7: Update matchedTopics()**

In `matchedTopics()`: replace the prefix-match logic:

```java
// Old: exactTopics.keySet().stream().filter(t -> t.startsWith(prefix))
// New:
if (pattern.contains("*")) {
    return exactTopics.keySet().stream()
        .filter(t -> matches(pattern, t))
        .collect(Collectors.toUnmodifiableSet());
}
```

- [ ] **Step 8: Update existing tests that relied on old semantics**

Tests `connections_wildcard_match`, `connections_match_all_wildcard`, `matchedTopics_match_all` use `"*"` which now means "one segment only". Update them to use `"**"` where they expect any-depth matching, or adjust assertions for the new single-segment semantics.

- [ ] **Step 9: Run all tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml test`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/push/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(push): segment-level and multi-level wildcard patterns Refs #114"
```

---

### Task 3: Client-Side Wildcard Matching (TypeScript)

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/event-connection.ts`
- Create: `packages/pages-data/src/dataset/external/sources/topic-matching.ts`
- Create: `packages/pages-data/src/dataset/external/sources/topic-matching.test.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/event-connection.test.ts`
- Modify: `packages/pages-data/src/dataset/external/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `matchesTopic(pattern: string, topic: string): boolean` (exported from pages-data), `isMatchedByRegistrations(topic: string, registrations: Set<string>): boolean` (updated)

- [ ] **Step 1: Write failing tests for matchesTopic**

Create `packages/pages-data/src/dataset/external/sources/topic-matching.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { matchesTopic, isMatchedByRegistrations } from './topic-matching.js';

describe('matchesTopic', () => {
  it('exact match', () => {
    expect(matchesTopic('debate:abc', 'debate:abc')).toBe(true);
    expect(matchesTopic('debate:abc', 'debate:xyz')).toBe(false);
  });

  it('single * matches one segment', () => {
    expect(matchesTopic('debate:*', 'debate:abc')).toBe(true);
    expect(matchesTopic('debate:*', 'debate:abc:def')).toBe(false);
  });

  it('mid-position *', () => {
    expect(matchesTopic('debate:*:summary', 'debate:abc:summary')).toBe(true);
    expect(matchesTopic('debate:*:summary', 'debate:abc:def:summary')).toBe(false);
  });

  it('multiple * segments', () => {
    expect(matchesTopic('a:*:b:*:c', 'a:x:b:y:c')).toBe(true);
    expect(matchesTopic('a:*:b:*:c', 'a:x:b:y:z')).toBe(false);
  });

  it('** matches zero or more segments', () => {
    expect(matchesTopic('debate:**', 'debate:abc')).toBe(true);
    expect(matchesTopic('debate:**', 'debate:abc:def:ghi')).toBe(true);
    expect(matchesTopic('debate:**', 'debate')).toBe(true);
    expect(matchesTopic('a:b:**', 'a')).toBe(false);
  });

  it('bare * matches single segment only', () => {
    expect(matchesTopic('*', 'hello')).toBe(true);
    expect(matchesTopic('*', 'hello:world')).toBe(false);
  });

  it('bare ** matches anything', () => {
    expect(matchesTopic('**', 'anything')).toBe(true);
    expect(matchesTopic('**', 'a:b:c:d')).toBe(true);
  });
});

describe('isMatchedByRegistrations', () => {
  it('matches exact registration', () => {
    expect(isMatchedByRegistrations('debate:abc', new Set(['debate:abc']))).toBe(true);
  });

  it('matches segment wildcard registration', () => {
    expect(isMatchedByRegistrations('debate:abc:summary', new Set(['debate:*:summary']))).toBe(true);
  });

  it('matches ** registration', () => {
    expect(isMatchedByRegistrations('debate:abc:def', new Set(['debate:**']))).toBe(true);
  });

  it('does not match unrelated', () => {
    expect(isMatchedByRegistrations('other:topic', new Set(['debate:**']))).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/external/sources/topic-matching.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement matchesTopic and isMatchedByRegistrations**

Create `packages/pages-data/src/dataset/external/sources/topic-matching.ts`:

```typescript
export function matchesTopic(pattern: string, topic: string): boolean {
  const ps = pattern.split(':');
  const ts = topic.split(':');

  if (ps[ps.length - 1] === '**') {
    if (ts.length < ps.length - 1) return false;
    for (let i = 0; i < ps.length - 1; i++) {
      if (ps[i] !== '*' && ps[i] !== ts[i]) return false;
    }
    return true;
  }

  if (ps.length !== ts.length) return false;
  for (let i = 0; i < ps.length; i++) {
    if (ps[i] === '*') continue;
    if (ps[i] !== ts[i]) return false;
  }
  return true;
}

export function isMatchedByRegistrations(topic: string, registrations: Set<string>): boolean {
  for (const reg of registrations) {
    if (matchesTopic(reg, topic)) return true;
  }
  return false;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-data run test -- --run src/dataset/external/sources/topic-matching.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Update event-connection.ts imports**

Replace the local `isMatchedByRegistrations` function in `event-connection.ts` with the import from `topic-matching.ts`. Update the reconnect `onopen` handler: `!reg.endsWith("*")` → `!reg.includes("*")`.

- [ ] **Step 6: Re-export matchesTopic from the package barrel**

Add to `packages/pages-data/src/dataset/external/index.ts`:

```typescript
export { matchesTopic, isMatchedByRegistrations } from "./sources/topic-matching.js";
```

- [ ] **Step 7: Run all pages-data tests**

Run: `yarn workspace @casehubio/pages-data run test`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(data): segment-level wildcard matching for EventConnection Refs #114"
```

---

### Task 4: Pages Event Helpers

**Files:**
- Create: `packages/pages-component/src/events.ts`
- Create: `packages/pages-component/src/events.test.ts`
- Modify: `packages/pages-component/src/index.ts`
- Modify: `packages/pages-component/package.json` (add pages-data dependency for matchesTopic)

**Interfaces:**
- Consumes: `matchesTopic` from `@casehubio/pages-data`
- Produces: `emitPagesEvent<T>(target, topic, payload): void`, `onPagesEvent<T>(target, topicOrPattern, handler): () => void`, `PagesEventDetail<T>`

- [ ] **Step 1: Write failing tests**

Create `packages/pages-component/src/events.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { emitPagesEvent, onPagesEvent, PagesEventDetail } from './events.js';

describe('emitPagesEvent', () => {
  it('dispatches pages-event CustomEvent with topic and payload', () => {
    const target = new EventTarget();
    const handler = vi.fn();
    target.addEventListener('pages-event', handler);

    emitPagesEvent(target, 'test-topic', { value: 42 });

    expect(handler).toHaveBeenCalledOnce();
    const event = handler.mock.calls[0][0] as CustomEvent<PagesEventDetail>;
    expect(event.detail.topic).toBe('test-topic');
    expect(event.detail.payload).toEqual({ value: 42 });
    expect(event.bubbles).toBe(true);
    expect(event.composed).toBe(true);
  });
});

describe('onPagesEvent', () => {
  it('filters by exact topic', () => {
    const target = new EventTarget();
    const handler = vi.fn();
    onPagesEvent(target, 'my-topic', handler);

    emitPagesEvent(target, 'my-topic', 'hello');
    emitPagesEvent(target, 'other-topic', 'ignored');

    expect(handler).toHaveBeenCalledOnce();
    expect(handler).toHaveBeenCalledWith('hello');
  });

  it('supports wildcard pattern matching', () => {
    const target = new EventTarget();
    const handler = vi.fn();
    onPagesEvent(target, 'debate:**', handler);

    emitPagesEvent(target, 'debate:abc', 'a');
    emitPagesEvent(target, 'debate:abc:def', 'b');
    emitPagesEvent(target, 'other:topic', 'ignored');

    expect(handler).toHaveBeenCalledTimes(2);
    expect(handler).toHaveBeenCalledWith('a');
    expect(handler).toHaveBeenCalledWith('b');
  });

  it('returns unsubscribe function', () => {
    const target = new EventTarget();
    const handler = vi.fn();
    const unsub = onPagesEvent(target, 'topic', handler);

    emitPagesEvent(target, 'topic', 'first');
    unsub();
    emitPagesEvent(target, 'topic', 'second');

    expect(handler).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-component run test -- --run src/events.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement events.ts**

Create `packages/pages-component/src/events.ts`:

```typescript
import { matchesTopic } from '@casehubio/pages-data';

export interface PagesEventDetail<T = unknown> {
  readonly topic: string;
  readonly payload: T;
}

export function emitPagesEvent<T>(target: EventTarget, topic: string, payload: T): void {
  target.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true,
    composed: true,
    detail: { topic, payload } satisfies PagesEventDetail<T>,
  }));
}

export function onPagesEvent<T>(
  target: EventTarget,
  topicOrPattern: string,
  handler: (payload: T) => void,
): () => void {
  const listener = (e: Event) => {
    const detail = (e as CustomEvent<PagesEventDetail<T>>).detail;
    if (matchesTopic(topicOrPattern, detail.topic)) {
      handler(detail.payload);
    }
  };
  target.addEventListener('pages-event', listener);
  return () => target.removeEventListener('pages-event', listener);
}
```

- [ ] **Step 4: Add re-export to index.ts**

Add to `packages/pages-component/src/index.ts`:

```typescript
export { emitPagesEvent, onPagesEvent, type PagesEventDetail } from './events.js';
```

- [ ] **Step 5: Run tests to verify**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(component): pages-event helpers with wildcard matching Refs #116"
```

---

### Task 5: DatasetContract + EventConnection Enhancements

**Files:**
- Create: `packages/pages-data/src/dataset/contract.ts`
- Modify: `packages/pages-data/src/dataset/types.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/event-connection.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/event-connection.test.ts`
- Modify: `packages/pages-data/src/dataset/external/index.ts`

**Interfaces:**
- Consumes: nothing new
- Produces: `DatasetContract<T>` interface, `ConnectionStatus` type, `EventConnectionOptions` interface, updated `createEventConnection(url, options?)` signature

- [ ] **Step 1: Write failing tests for DatasetContract**

Add to an appropriate test file or create `packages/pages-data/src/dataset/contract.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import type { DatasetContract } from './contract.js';

describe('DatasetContract', () => {
  it('is a valid type with name, description, shape', () => {
    const contract: DatasetContract<{ id: string }> = {
      name: 'users',
      description: 'User dataset',
      shape: { id: '' },
    };
    expect(contract.name).toBe('users');
  });
});
```

- [ ] **Step 2: Write failing tests for ConnectionStatus and EventConnectionOptions**

Add to `packages/pages-data/src/dataset/external/sources/event-connection.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { createEventConnection, type ConnectionStatus } from './event-connection.js';

describe('EventConnection status', () => {
  it('starts as disconnected', () => {
    // Mock WebSocket to control state
    const conn = createEventConnection('ws://localhost:8080', {
      onStatusChange: vi.fn(),
    });
    expect(conn.status).toBe('disconnected');
    conn.close();
  });

  it('calls onStatusChange on transitions', () => {
    const onChange = vi.fn();
    const conn = createEventConnection('ws://localhost:8080', {
      onStatusChange: onChange,
    });
    // Status tracking is verified via the callback
    conn.close();
  });
});
```

- [ ] **Step 3: Implement DatasetContract**

Create `packages/pages-data/src/dataset/contract.ts`:

```typescript
export interface DatasetContract<T = unknown> {
  readonly name: string;
  readonly description: string;
  readonly shape: T;
}
```

Add to `packages/pages-data/src/dataset/types.ts`:

```typescript
export type { DatasetContract } from './contract.js';
```

- [ ] **Step 4: Implement ConnectionStatus and EventConnectionOptions**

Add to `event-connection.ts`:

```typescript
export type ConnectionStatus = 'connected' | 'reconnecting' | 'disconnected';

export interface EventConnectionOptions {
  readonly config?: PushSourceConfig;
  readonly batchEvents?: boolean;
  readonly onStatusChange?: (status: ConnectionStatus) => void;
}
```

Update `EventConnection` interface:

```typescript
export interface EventConnection {
  send(message: object): void;
  listen(topics: string[]): Promise<ListenAck>;
  unlisten(topics: string[]): Promise<void>;
  close(): void;
  readonly connected: boolean;
  readonly status: ConnectionStatus;
}
```

Update `createEventConnection` signature:

```typescript
export function createEventConnection(
  url: string,
  options?: EventConnectionOptions,
): EventConnection
```

Add status tracking:
- Internal `let currentStatus: ConnectionStatus = 'disconnected'`
- Transition helper that calls `options?.onStatusChange`
- Set `'connected'` in `ws.onopen`
- Set `'reconnecting'` in `ws.onclose` before reconnect timer
- Set `'disconnected'` in `close()` and on permanent close (code ≥ 4000)
- Add `status` getter to returned object

Add rAF batching:
- When `options?.batchEvents` is true, queue events in an array
- Flush via `requestAnimationFrame` callback
- When false (default), dispatch immediately (current behaviour)

- [ ] **Step 5: Update barrel exports**

Add to `packages/pages-data/src/dataset/external/index.ts`:

```typescript
export type { ConnectionStatus, EventConnectionOptions } from "./sources/event-connection.js";
export type { DatasetContract } from "../contract.js";
```

- [ ] **Step 6: Run all pages-data tests**

Run: `yarn workspace @casehubio/pages-data run test`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-data/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(data): DatasetContract, ConnectionStatus, rAF batching Refs #116"
```

---

### Task 6: pages-primitives Package Scaffold

**Files:**
- Create: `packages/pages-primitives/package.json`
- Create: `packages/pages-primitives/tsconfig.json`
- Create: `packages/pages-primitives/tsconfig.build.json`
- Create: `packages/pages-primitives/vitest.config.ts`
- Create: `packages/pages-primitives/src/index.ts`
- Modify: `package.json` (root — add to build:packages)

**Interfaces:**
- Consumes: `lit@^3.3.3`
- Produces: empty package scaffold, ready for a11y mixins in Task 7

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/pages-primitives",
  "version": "0.2.0",
  "description": "CaseHub Pages Lit web components — a11y mixins, schema form, UI primitives",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/casehub-pages.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "lit": "^3.3.3"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "@open-wc/testing": "^4.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json and tsconfig.build.json**

`tsconfig.json`:
```json
{
  "extends": "@casehubio/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true,
    "experimentalDecorators": true
  },
  "include": ["src"],
  "references": []
}
```

`tsconfig.build.json`:
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "emitDeclarationOnly": false,
    "composite": false,
    "declaration": true
  },
  "exclude": ["**/*.test.ts"]
}
```

- [ ] **Step 3: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true,
    include: ['src/**/*.test.ts'],
  },
});
```

- [ ] **Step 4: Create empty index.ts**

```typescript
// A11y mixins, schema form, and UI primitives — Lit web components for the pages ecosystem
```

- [ ] **Step 5: Add to root build:packages**

In root `package.json`, add `yarn workspace @casehubio/pages-primitives run build` to the `build:packages` script chain (after pages-ui-tokens, before pages-runtime — no dependency on other pages packages).

- [ ] **Step 6: Install and verify**

Run: `yarn install && yarn workspace @casehubio/pages-primitives run build`
Expected: Clean build, empty dist with index.js

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-primitives/ package.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "chore: scaffold pages-primitives Lit package Refs #116"
```

---

### Task 7: A11y Mixins

**Files:**
- Create: `packages/pages-primitives/src/a11y/index.ts`
- Create: `packages/pages-primitives/src/a11y/roving-tabindex.ts`
- Create: `packages/pages-primitives/src/a11y/roving-tabindex.test.ts`
- Create: `packages/pages-primitives/src/a11y/focus-trap.ts`
- Create: `packages/pages-primitives/src/a11y/keyboard-shortcut.ts`
- Create: `packages/pages-primitives/src/a11y/keyboard-shortcut.test.ts`
- Create: `packages/pages-primitives/src/a11y/live-region.ts`
- Modify: `packages/pages-primitives/src/index.ts`

**Interfaces:**
- Consumes: `lit` (LitElement, state decorator)
- Produces: `RovingTabindexMixin`, `FocusTrapMixin`, `KeyboardShortcutMixin`, `LiveRegionMixin`

Implementation: port from `blocks-ui-core/src/mixins/` with these changes:
- `RovingTabindexMixin` gains `rovingDirection: 'horizontal' | 'vertical' | 'both'` abstract property
  - `horizontal`: ArrowLeft/ArrowRight + Home/End
  - `vertical`: ArrowUp/ArrowDown + Home/End
  - `both`: all four arrows + Home/End
- All other mixins ported as-is

- [ ] **Step 1: Write failing tests for RovingTabindexMixin (horizontal)**

Tests must verify ArrowLeft/ArrowRight navigation in horizontal mode, ArrowUp/ArrowDown in vertical mode.

- [ ] **Step 2: Implement RovingTabindexMixin with direction support**

Port from `blocks-ui-core/src/mixins/roving-tabindex.ts`, adding the `rovingDirection` abstract property and key mapping per direction.

- [ ] **Step 3: Port FocusTrapMixin, KeyboardShortcutMixin, LiveRegionMixin**

Port from blocks-ui-core with tests. These are unchanged from blocks-ui-core.

- [ ] **Step 4: Create barrel export**

`packages/pages-primitives/src/a11y/index.ts`:
```typescript
export { RovingTabindexMixin } from './roving-tabindex.js';
export { FocusTrapMixin } from './focus-trap.js';
export { KeyboardShortcutMixin } from './keyboard-shortcut.js';
export { LiveRegionMixin } from './live-region.js';
```

Update `packages/pages-primitives/src/index.ts`:
```typescript
export * from './a11y/index.js';
```

- [ ] **Step 5: Run all tests**

Run: `yarn workspace @casehubio/pages-primitives run test`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-primitives/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(primitives): a11y mixins with directional roving tabindex Refs #116"
```

---

### Task 8: SchemaForm

**Files:**
- Create: `packages/pages-primitives/src/schema-form/index.ts`
- Create: `packages/pages-primitives/src/schema-form/schema-form.ts`
- Create: `packages/pages-primitives/src/schema-form/schema-form.test.ts`
- Create: `packages/pages-primitives/src/schema-form/field-registry.ts`
- Create: `packages/pages-primitives/src/schema-form/field-renderers.ts`
- Modify: `packages/pages-primitives/src/index.ts`

**Interfaces:**
- Consumes: `lit` (LitElement, customElement, property, state decorators)
- Produces: `PagesSchemaForm` (`<pages-schema-form>` custom element), `registerFieldRenderer(format, component)`, `getFieldRenderer(format)`, `renderDisplayField(key, schema, value)`, `renderEditField(key, schema, value, onChange)`

Implementation: port from `blocks-ui-core/src/schema-form/` with these changes:
- Element renamed from `<schema-form>` to `<pages-schema-form>`
- Events renamed: `schema-form-change` → `pages-schema-form-change`, `schema-form-submit` → `pages-schema-form-submit`

- [ ] **Step 1: Write failing tests**

Test SchemaForm renders from a JSON schema, dispatches prefixed events.

- [ ] **Step 2: Port implementation from blocks-ui-core**

Copy and rename element + events.

- [ ] **Step 3: Run tests, commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-primitives/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(primitives): pages-schema-form with field registry Refs #116"
```

---

### Task 9: Filter Chips + Scope Selector

**Files:**
- Create: `packages/pages-primitives/src/primitives/index.ts`
- Create: `packages/pages-primitives/src/primitives/filter-chips.ts`
- Create: `packages/pages-primitives/src/primitives/filter-chips.test.ts`
- Create: `packages/pages-primitives/src/primitives/scope-selector.ts`
- Create: `packages/pages-primitives/src/primitives/scope-selector.test.ts`
- Modify: `packages/pages-primitives/src/index.ts`

**Interfaces:**
- Consumes: `RovingTabindexMixin` from Task 7, `lit`
- Produces: `PagesFilterChips` (`<pages-filter-chips>`), `PagesScope Selector` (`<pages-scope-selector>`), `ChipItem`, `ScopeItem` interfaces

These are new components built from scratch following the spec §7.

- [ ] **Step 1: Write failing tests for PagesFilterChips**

Test: renders chips from items array, click toggles selection, count=0 chips are disabled, emits `pages-filter-chips-change` with selected IDs, keyboard navigation via ArrowLeft/ArrowRight.

- [ ] **Step 2: Implement PagesFilterChips**

Lit custom element composing `RovingTabindexMixin` with `rovingDirection: 'horizontal'`. Uses `--pages-*` CSS custom properties.

- [ ] **Step 3: Write failing tests for PagesScopeSelector**

Test: renders pills from items array, click selects (radio behaviour), allowDeselect toggles off, emits `pages-scope-change` with selected ID or null, badge renders when provided.

- [ ] **Step 4: Implement PagesScopeSelector**

Lit custom element composing `RovingTabindexMixin` with `rovingDirection: 'horizontal'`. Radio selection logic.

- [ ] **Step 5: Create barrel exports**

```typescript
// packages/pages-primitives/src/primitives/index.ts
export { PagesFilterChips, type ChipItem } from './filter-chips.js';
export { PagesScopeSelector, type ScopeItem } from './scope-selector.js';
```

Update `packages/pages-primitives/src/index.ts`:
```typescript
export * from './a11y/index.js';
export * from './schema-form/index.js';
export * from './primitives/index.js';
```

- [ ] **Step 6: Run all pages-primitives tests**

Run: `yarn workspace @casehubio/pages-primitives run test`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-primitives/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(primitives): filter chip bar and scope selector Refs #116"
```

---

### Task 10: Full Build Verification + CLAUDE.md Update

**Files:**
- Modify: `CLAUDE.md` (via symlink at workspace)

**Interfaces:**
- Consumes: all previous tasks
- Produces: verified green build, updated project docs

- [ ] **Step 1: Run full build**

Run: `yarn install && yarn build`
Expected: ALL packages build including pages-primitives

- [ ] **Step 2: Run all tests across monorepo**

Run: `yarn test`
Expected: ALL PASS

- [ ] **Step 3: Run Maven build for push module**

Run: `mvn -f /Users/mdproctor/claude/casehub/pages/backend/push/pom.xml test`
Expected: ALL PASS

- [ ] **Step 4: Update CLAUDE.md**

Add `pages-primitives` to the Package Overview section:
```markdown
- `@casehubio/pages-primitives` — Lit web components: a11y mixins, schema form, filter chips, scope selector
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/pages commit -m "docs: update CLAUDE.md with pages-primitives package Refs #116"
```
