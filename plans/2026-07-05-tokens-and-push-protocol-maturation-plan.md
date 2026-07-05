# Design Tokens & Push Protocol Maturation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adopt blocks-ui's 12-step OKLCH design token system as the pages foundation, and mature the push protocol with request-response correlation, wildcard topic matching, and event replay.

**Architecture:** Two independent domains implemented in dependency order. §4 (tokens) creates a new `pages-ui-tokens` package and migrates all consumers. §1-§3 (push protocol) evolves the `backend/push` Java module and TypeScript push infrastructure with a general correlation layer, prefix-based wildcards, and per-topic event replay with an EventStore SPI.

**Tech Stack:** TypeScript 5 / Vitest / Yarn 4 workspaces (tokens + TS push). Java 21 / JUnit 5 / jackson-core (Java push). OKLCH colour space / CSS custom properties (tokens).

## Global Constraints

- **No backward compatibility.** Breaking changes are the point — they force callers to be explicit.
- **TDD.** Write failing tests before implementation code. Run tests to confirm failure, then implement.
- **No compatibility aliases.** Old `--pages-*` flat tokens are replaced directly with 12-step equivalents.
- **Spec:** `docs/superpowers/specs/2026-07-05-tokens-and-push-protocol-maturation-design.md` — the reviewed, approved design. All section references (§N.M) refer to this spec.
- **Java module:** `backend/push/` — plain JUnit 5, no Quarkus test infrastructure.
- **TypeScript packages:** Yarn workspace, `vitest run`, ES modules, strict TypeScript.
- **Build order:** `pages-ui-tokens` must be added to `build:packages` before `pages-viz` (which will depend on it).

---

### Task 1: `pages-ui-tokens` package — colour generation, tokens, and theme CSS

Create the complete `@casehubio/pages-ui-tokens` package with OKLCH 12-step colour scale generation, non-colour token definitions, theme CSS generation, and theme injection. This is the entire token system — §4.1 through §4.5 of the spec.

**Files:**
- Create: `packages/pages-ui-tokens/package.json`
- Create: `packages/pages-ui-tokens/tsconfig.json`
- Create: `packages/pages-ui-tokens/tsconfig.build.json`
- Create: `packages/pages-ui-tokens/vitest.config.ts`
- Create: `packages/pages-ui-tokens/src/index.ts`
- Create: `packages/pages-ui-tokens/src/colours.ts`
- Create: `packages/pages-ui-tokens/src/colours.test.ts`
- Create: `packages/pages-ui-tokens/src/tokens.ts`
- Create: `packages/pages-ui-tokens/src/themes.ts`
- Create: `packages/pages-ui-tokens/src/themes.test.ts`
- Modify: `package.json` (root — add to workspaces and build:packages)

**Interfaces:**
- Consumes: nothing (standalone package, no workspace deps)
- Produces:
  - `generateScale(hue: number, chroma: number, contrast: number, isDark: boolean): Record<string, string>`
  - `SPACING_SCALE`, `TYPOGRAPHY`, `ELEVATION_LIGHT`, `ELEVATION_DARK`, `MOTION`, `RADIUS`, `DENSITY_COMPACT_OVERRIDES`
  - `ThemeConfig` interface, `DEFAULT_THEME` constant
  - `generateThemeCSS(config: ThemeConfig): string`
  - `injectTheme(config: ThemeConfig, target?: HTMLElement): void`
  - `applyThemeMode(element: HTMLElement, mode: "light" | "dark"): void`

- [ ] **Step 1: Create package scaffold**

Create `packages/pages-ui-tokens/package.json`:
```json
{
  "name": "@casehubio/pages-ui-tokens",
  "version": "0.3.0",
  "description": "CaseHub Pages design tokens — OKLCH 12-step colour scales, spacing, typography, elevation, motion",
  "repository": { "type": "git", "url": "https://github.com/casehubio/casehub-pages.git" },
  "publishConfig": { "registry": "https://npm.pkg.github.com" },
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
  "devDependencies": {
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1",
    "@casehubio/pages-tsconfig": "workspace:*"
  }
}
```

Create `tsconfig.json`, `tsconfig.build.json`, `vitest.config.ts` following the same patterns as `pages-data`.

- [ ] **Step 2: Write failing colour generation tests**

Create `packages/pages-ui-tokens/src/colours.test.ts` with tests from §4.9:
- `generateScale` returns 12 steps (keys "1"–"12")
- Values match `oklch(N% C H)` format
- Chroma reduction at extremes (steps 1, 12 have lower chroma than step 6)
- Light step 1 ≈ 98.5% lightness, dark step 1 ≈ 8%
- Contrast modifier shifts lightness values

Run: `yarn workspace @casehubio/pages-ui-tokens run test`
Expected: FAIL — `colours.ts` does not exist

- [ ] **Step 3: Implement colour generation**

Create `packages/pages-ui-tokens/src/colours.ts` — port from `blocks-ui-core/src/tokens/colours.ts` (§4.3). Replace `--blocks-` references with `--pages-` prefix. The function itself is prefix-agnostic — it returns `Record<string, string>` keyed by step number.

Run tests — all colour tests pass.

- [ ] **Step 4: Write failing theme generation tests**

Create `packages/pages-ui-tokens/src/themes.test.ts` with tests from §4.9:
- `generateThemeCSS` contains `.pages-theme-light`, `.pages-theme-dark`, `.pages-density-compact` classes
- Contains all 72 colour tokens (6 hues × 12 steps as `--pages-{hue}-{step}`)
- Contains spacing tokens (`--pages-space-*`)
- Contains typography tokens (`--pages-font-family`, `--pages-font-size-*`, `--pages-line-height-*`, `--pages-font-weight-*`)
- Contains elevation tokens (`--pages-shadow-*`, light and dark variants differ)
- Contains motion tokens (`--pages-duration-*`, `--pages-ease-*`)
- Contains radius tokens (`--pages-radius-sm`, `-md`, `-lg`)
- Contains surface tokens (`--pages-surface-*`)
- `DEFAULT_THEME` produces no NaN or undefined
- Density compact overrides shrink space and font tokens
- `applyThemeMode` sets correct CSS class (needs jsdom)
- `injectTheme` creates and replaces `<style data-pages-theme>` element (needs jsdom)

Run: `yarn workspace @casehubio/pages-ui-tokens run test`
Expected: FAIL — `themes.ts`, `tokens.ts` do not exist

- [ ] **Step 5: Implement tokens and theme generation**

Create `packages/pages-ui-tokens/src/tokens.ts` — port from `blocks-ui-core/src/tokens/tokens.ts`. All constants identical to blocks-ui except `DENSITY_COMPACT_OVERRIDES` uses `--pages-` prefix instead of `--blocks-`.

Create `packages/pages-ui-tokens/src/themes.ts` — port from `blocks-ui-core/src/tokens/themes.ts`:
- `ThemeConfig` interface and `DEFAULT_THEME` constant (§4.2)
- `SEMANTIC_HUES` map with fixed hues for success/warning/danger/info (§4.3)
- `generateThemeCSS(config)` → CSS string with three class definitions (§4.5)
- `injectTheme(config, target?)` → creates `<style data-pages-theme>`, prepends to target (§4.5)
- `applyThemeMode(element, mode)` → sets/removes `pages-theme-light`/`pages-theme-dark` class (§4.5)

All CSS custom properties use `--pages-` prefix (not `--blocks-`).

Create `packages/pages-ui-tokens/src/index.ts` — export everything:
```typescript
export { generateScale } from './colours.js';
export { SPACING_SCALE, TYPOGRAPHY, MOTION, RADIUS } from './tokens.js';
export { generateThemeCSS, injectTheme, applyThemeMode, DEFAULT_THEME, type ThemeConfig } from './themes.js';
```

Run tests — all pass.

- [ ] **Step 6: Wire into workspace build**

Update root `package.json` `build:packages` script: add `yarn workspace @casehubio/pages-ui-tokens run build &&` BEFORE the `pages-viz` build step (pages-viz will depend on tokens after Task 2).

Run: `yarn install && yarn workspace @casehubio/pages-ui-tokens run build`
Expected: builds successfully, `dist/` created with JS + declarations.

- [ ] **Step 7: Commit**

```
feat(pages-ui-tokens): OKLCH 12-step design token system

Port blocks-ui's complete token vocabulary — 72 colour tokens (6 hues × 12 steps),
spacing, typography, elevation, motion, radius, surface, density.

Refs #101
```

---

### Task 2: Component migration — delete old theme, migrate all consumers

Delete `pages-viz/src/base/theme.ts`, update `site.ts` to use `pages-ui-tokens`, and migrate all 21 component files from flat tokens to 12-step equivalents. §4.6 through §4.8 of the spec.

**Files:**
- Delete: `packages/pages-viz/src/base/theme.ts`
- Delete: `packages/pages-viz/src/base/theme.test.ts`
- Modify: `packages/pages-viz/src/index.ts` — remove theme exports
- Modify: `packages/pages-viz/package.json` — (no new dep needed — components use CSS vars, not imports)
- Modify: `packages/pages-runtime/package.json` — add `@casehubio/pages-ui-tokens` dep
- Modify: `packages/pages-runtime/src/site.ts` — use new theme API (§4.6)
- Modify: `packages/pages-runtime/src/site.test.ts` — update theme assertions
- Modify: 14 `pages-viz` component files — token migration per §4.8 mapping table
- Modify: `packages/pages-viz/src/components/PagesAlert.test.ts` — update assertions
- Modify: `packages/pages-viz/src/components/table-row-style.test.ts` — update assertions
- Modify: `packages/pages-component/src/renderer/interactive.ts` — token migration
- Modify: `packages/pages-runtime/src/activation.ts` — token migration

**Interfaces:**
- Consumes: `injectTheme`, `applyThemeMode`, `DEFAULT_THEME`, `ThemeConfig` from `@casehubio/pages-ui-tokens`
- Produces: Updated `SiteOptions` with `themeConfig?: ThemeConfig`, updated `LiveSite.setTheme(mode: "light" | "dark")`

- [ ] **Step 1: Update site.ts and SiteOptions**

Modify `packages/pages-runtime/package.json`: add `"@casehubio/pages-ui-tokens": "workspace:*"` to dependencies.

Modify `packages/pages-runtime/src/site.ts` per §4.6:
- Replace theme imports: `import { injectTheme, applyThemeMode, DEFAULT_THEME } from "@casehubio/pages-ui-tokens";` and `import type { ThemeConfig } from "@casehubio/pages-ui-tokens";`
- Remove old imports: `applyTheme`, `LIGHT_THEME`, `DARK_THEME`, `PagesTheme` from `pages-viz`
- Add `themeConfig?: ThemeConfig` to `SiteOptions`
- Replace `applyTheme(target, isDark ? DARK_THEME : LIGHT_THEME)` with `injectTheme(options?.themeConfig ?? DEFAULT_THEME, target); applyThemeMode(target, isDark ? "dark" : "light");`
- Update `setTheme` on LiveSite per §4.6 — takes `"light" | "dark"` only, calls `applyThemeMode`, preserves ECharts theme sync

- [ ] **Step 2: Update site.test.ts**

Update theme assertions — old tests check `getPropertyValue("--pages-bg")` etc. New tests should verify:
- `pages-theme-light` or `pages-theme-dark` class is applied
- `data-pages-theme` attribute is no longer set (or updated to check the class)
- ECharts theme passthrough still works

- [ ] **Step 3: Delete old theme and update exports**

Delete `packages/pages-viz/src/base/theme.ts` and `theme.test.ts`.
Update `packages/pages-viz/src/index.ts`: remove exports of `PagesTheme`, `LIGHT_THEME`, `DARK_THEME`, `applyTheme`, `clearTheme`.

- [ ] **Step 4: Migrate pages-viz components (batch)**

Apply the §4.8 mapping table across all 14 component files. This is a mechanical find-and-replace per file. For each file:
- `var(--pages-text, #333)` → `var(--pages-neutral-12, #333)`
- `var(--pages-bg, #fff)` → `var(--pages-neutral-1, #fff)`
- `var(--pages-border, #e0e0e0)` → `var(--pages-neutral-6, #e0e0e0)`
- `var(--pages-accent, #5470c6)` → `var(--pages-accent-9, #5470c6)`
- `var(--pages-font, ...)` → `var(--pages-font-family, ...)`
- `var(--pages-font-size, 14px)` → `var(--pages-font-size-base, 14px)`
- `var(--pages-radius, 4px)` → `var(--pages-radius-sm, 4px)`
- All alert/row/button/error/success tokens per the mapping table
- Retain fallback values for defensive rendering

Files: `PagesTable.ts`, `PagesMetric.ts`, `PagesAlert.ts`, `PagesBadge.ts`, `PagesCountdown.ts`, `PagesActionButton.ts`, `PagesSelector.ts`, `PagesElement.ts`, `PagesTextInput.ts`, `PagesDatePicker.ts`, `PagesCheckbox.ts`, `PagesTextarea.ts`, `PagesNumberInput.ts`, `PagesDropdown.ts`

- [ ] **Step 5: Migrate pages-component and pages-runtime**

Apply the same mapping table to:
- `packages/pages-component/src/renderer/interactive.ts`
- `packages/pages-runtime/src/activation.ts`

- [ ] **Step 6: Update test assertions**

Update `PagesAlert.test.ts` and `table-row-style.test.ts` — assertions that check for old token names (e.g., `--pages-row-danger-bg`, `--pages-alert-info-bg`) must use the new 12-step names (`--pages-danger-3`, `--pages-info-3`).

- [ ] **Step 7: Run full test suite**

Run: `yarn workspace @casehubio/pages-viz run test && yarn workspace @casehubio/pages-runtime run test && yarn workspace @casehubio/pages-component run test`
Expected: all pass. Fix any remaining references to old tokens.

Run: `yarn typecheck`
Expected: no type errors.

- [ ] **Step 8: Commit**

```
feat(tokens): migrate all consumers to 12-step design token system

Delete pages-viz/src/base/theme.ts (flat 13-token system). Replace with
pages-ui-tokens (OKLCH 12-step). Migrate 21 files per §4.8 mapping table.

LiveSite.setTheme() now takes "light" | "dark" only. SiteOptions gains
themeConfig?: ThemeConfig for custom colour schemes.

Refs #101
```

---

### Task 3: Java — Wire protocol correlation layer + seq type change

Add required `id` field to all `PushRequest` records, add `ack`/`error` builders to `PushMessage`, and change `seq` from String to Long. §1.1 through §1.4 of the spec.

**Files:**
- Modify: `backend/push/src/main/java/io/casehub/pages/push/PushRequest.java`
- Modify: `backend/push/src/main/java/io/casehub/pages/push/PushMessage.java`
- Modify: `backend/push/src/test/java/io/casehub/pages/push/PushRequestTest.java`
- Modify: `backend/push/src/test/java/io/casehub/pages/push/PushMessageTest.java`

**Interfaces:**
- Consumes: existing `PushRequest`, `PushMessage`, `PushColumn`, `JsonFactory`/`JsonGenerator`
- Produces:
  - `PushRequest.id()` — on sealed interface (all records implement)
  - `PushRequest.Listen.since()` — `Map<String, Long>`, defaults to `Map.of()`
  - `PushMessage.ack(String id)`, `PushMessage.ack(String id, List<String> topics)`, `PushMessage.ack(String id, List<String> topics, List<String> gaps)`
  - `PushMessage.error(String id, String message)`
  - `PushMessage.event(String topic, String payloadJson, Long seq)` — seq is now `Long`

- [ ] **Step 1: Write failing PushRequest tests**

Add to `PushRequestTest.java`:
- `parse()` with `id` field on all four ops extracts `id` correctly
- `parse()` missing `id` throws `IllegalArgumentException`
- `parse()` listen with `since` as JSON object → `Map<String, Long>`
- `parse()` listen without `since` → empty map
- `parse()` subscribe with string `since` → works as before
- `parse()` with `since` before `op` (field-order independence)

Run: `mvn -f backend/push/pom.xml test`
Expected: compilation errors (id doesn't exist yet)

- [ ] **Step 2: Implement PushRequest changes**

Update `PushRequest.java` per §1.2:
- Add `String id()` to sealed interface
- Add `String id` as first parameter to all four records with `Objects.requireNonNull(id, "id")`
- Add `Map<String, Long> since` to `Listen` record with defensive copy
- Update `parse()`:
  - Add `id` field extraction
  - Implement dual-type `since` parsing per §1.2 parse strategy: `VALUE_STRING` → `stringSince`, `START_OBJECT` → `mapSince`
  - Throw if `id` is null after parsing

Run tests — PushRequest tests pass.

- [ ] **Step 3: Write failing PushMessage tests**

Add to `PushMessageTest.java`:
- `ack(id)` produces `{"op":"ack","id":"r1"}`
- `ack(id, topics)` includes `topics` array
- `ack(id, topics, gaps)` includes `gaps` array
- `error(id, message)` produces `{"op":"error","id":"r1","message":"..."}`
- `event(topic, payload, Long seq)` encodes seq as JSON number (not string)
- Existing `event(topic, payload)` tests still pass (no seq = no seq field)

Run: `mvn -f backend/push/pom.xml test`
Expected: compilation errors (ack/error don't exist, seq type changed)

- [ ] **Step 4: Implement PushMessage changes**

Update `PushMessage.java` per §1.3 and §1.4:
- Change `event(String topic, String payloadJson, String seq)` to `event(String topic, String payloadJson, Long seq)` — use `g.writeNumberField("seq", seq)` instead of `writeStringField`
- Add `ack(String id)` — writes `{"op":"ack","id":"..."}` via `JsonGenerator`
- Add `ack(String id, List<String> topics)` — adds `topics` JSON array
- Add `ack(String id, List<String> topics, List<String> gaps)` — adds `gaps` array (omitted if null/empty)
- Add `error(String id, String message)` — writes `{"op":"error","id":"...","message":"..."}`

Run tests — all PushMessage tests pass.

- [ ] **Step 5: Fix existing tests broken by id requirement**

Update all existing `PushRequestTest` cases to include `"id":"..."` in the JSON input. Update all existing assertions on `PushRequest` records to pass `id` as first constructor argument.

Run: `mvn -f backend/push/pom.xml test`
Expected: ALL tests pass.

- [ ] **Step 6: Commit**

```
feat(push): wire protocol correlation layer — id on all ops, ack/error builders

PushRequest: required id on Subscribe/Unsubscribe/Listen/Unlisten. Listen gains
Map<String,Long> since with dual-type parsing. PushMessage: ack/error builders,
seq changed from String to Long.

Refs #107
```

---

### Task 4: Java — Wildcard topic matching

Split `TopicRegistry` into exact and wildcard maps, add prefix-based matching to `connections()`, add validation and introspection methods. §2.1 through §2.4 of the spec.

**Files:**
- Modify: `backend/push/src/main/java/io/casehub/pages/push/TopicRegistry.java`
- Modify: `backend/push/src/test/java/io/casehub/pages/push/TopicRegistryTest.java`

**Interfaces:**
- Consumes: existing `TopicRegistry` contract
- Produces:
  - `TopicRegistry.isValidTopicOrPattern(String topic)` — static validation
  - `TopicRegistry.matchedTopics(String pattern)` — introspection
  - `TopicRegistry.connections(String topic)` — now wildcard-aware

- [ ] **Step 1: Write failing wildcard tests**

Add to `TopicRegistryTest.java` — all tests from §2.6:
- `isValidTopicOrPattern`: exact → true, trailing wildcard → true, match-all → true, mid-wildcard → false, null/empty → false
- `connections` wildcard match: register `"debate:*"`, query `connections("debate:abc")` → found
- `connections` wildcard + exact union: both types return their connections
- `connections` match-all `"*"`: returns connections for any topic
- `unlisten` wildcard: pattern removed, no longer matches
- `removeConnection` cleans wildcards
- `matchedTopics`: `"debate:*"` finds concrete `["debate:abc", "debate:xyz"]`
- Thread safety with wildcards: concurrent listen/connections across both maps

Run: `mvn -f backend/push/pom.xml test`
Expected: FAIL — new methods don't exist, wildcard matching not implemented

- [ ] **Step 2: Implement wildcard TopicRegistry**

Rewrite `TopicRegistry.java` per §2.3:
- Split `topicToConnections` into `exactTopics` and `wildcardPatterns` (both `ConcurrentHashMap<String, CopyOnWriteArraySet<String>>`)
- `listen()`: route to exact or wildcard map based on `topic.endsWith("*")`
- `unlisten()`: route to correct map
- `removeConnection()`: clean up both maps via reverse map
- `connections(topic)`: union exact match + wildcard prefix scan
- Add static `isValidTopicOrPattern(String topic)` per §2.2
- Add `matchedTopics(String pattern)` per §2.4

Run tests — all pass.

- [ ] **Step 3: Verify existing tests still pass**

Run: `mvn -f backend/push/pom.xml test`
Expected: ALL tests (existing + new) pass.

- [ ] **Step 4: Commit**

```
feat(push): wildcard topic matching — prefix-based pattern support

TopicRegistry supports trailing-* patterns. connections() checks both exact
and wildcard maps. Static isValidTopicOrPattern() validates patterns.
matchedTopics() for app-level introspection.

Refs #105
```

---

### Task 5: Java — EventStore SPI + InMemoryEventStore

Create the `EventStore` interface, `StoredEvent` record, and `InMemoryEventStore` default implementation with bounded ring buffer. §3.1 through §3.2 of the spec.

**Files:**
- Create: `backend/push/src/main/java/io/casehub/pages/push/EventStore.java`
- Create: `backend/push/src/main/java/io/casehub/pages/push/StoredEvent.java`
- Create: `backend/push/src/main/java/io/casehub/pages/push/InMemoryEventStore.java`
- Create: `backend/push/src/test/java/io/casehub/pages/push/InMemoryEventStoreTest.java`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `EventStore.append(String topic, String payloadJson)` → `long` seq
  - `EventStore.replay(String topic, long sinceSeq)` → `List<StoredEvent>`
  - `EventStore.topics()` → `Set<String>`
  - `StoredEvent(String topic, String payloadJson, long seq)` record
  - `InMemoryEventStore(int maxEventsPerTopic)` constructor

- [ ] **Step 1: Write failing EventStore tests**

Create `InMemoryEventStoreTest.java` with all tests from §3.7:
- `append` assigns monotonic seq (1, 2, 3...)
- Per-topic isolation (topic A seq independent of topic B)
- `replay` returns events after `sinceSeq`
- `replay` on empty topic → empty list
- Bounded eviction (max 3 → oldest pruned on 4th)
- `topics()` empty on fresh store
- `topics()` returns topic after append
- `topics()` survives eviction (topic still listed after ring buffer rotation)
- Thread safety: concurrent append/replay

Run: `mvn -f backend/push/pom.xml test`
Expected: compilation errors

- [ ] **Step 2: Implement EventStore, StoredEvent, InMemoryEventStore**

Create `EventStore.java` (interface, §3.1), `StoredEvent.java` (record, §3.1), `InMemoryEventStore.java` (§3.2):
- `InMemoryEventStore`: `ConcurrentHashMap<String, TopicBuffer>` where `TopicBuffer` holds an `ArrayDeque<StoredEvent>` + `AtomicLong` seq counter, synchronized on per-topic lock
- `append`: assign `seq.incrementAndGet()`, add to deque, poll oldest if `size > max`
- `replay`: filter deque for `seq > sinceSeq`, return as `List.copyOf()`
- `topics`: return `Set.copyOf(map.keySet())`

Run tests — all pass.

- [ ] **Step 3: Commit**

```
feat(push): EventStore SPI + InMemoryEventStore bounded ring buffer

Per-topic sequence numbers, bounded eviction, thread-safe. Default for
apps that need event replay; apps bring their own durable store (JDBC, Redis).

Refs #106
```

---

### Task 6: TypeScript — push-wire.ts correlation support

Add `nextRequestId`, update `sendListen`/`sendUnlisten` with `id` parameter, add `sendSubscribe`/`sendUnsubscribe`, add `since` parameter to `sendListen`. §1.6 of the spec.

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/push-wire.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/push-wire.test.ts`

**Interfaces:**
- Consumes: nothing (standalone utilities)
- Produces:
  - `nextRequestId(): string`
  - `sendListen(ws, id, topics, since?)` — updated signature
  - `sendUnlisten(ws, id, topics)` — updated signature
  - `sendSubscribe(ws, id, dataset, since?)` — new
  - `sendUnsubscribe(ws, id, dataset)` — new

- [ ] **Step 1: Write failing tests**

Add to `push-wire.test.ts`:
- `nextRequestId` returns monotonic sequential strings: "1", "2", "3"
- `sendListen` includes `id` field in JSON
- `sendListen` with `since` includes the map in JSON
- `sendUnlisten` includes `id` field
- `sendSubscribe` serializes `{ op: "subscribe", id, dataset }`
- `sendSubscribe` with `since` includes string since
- `sendUnsubscribe` serializes `{ op: "unsubscribe", id, dataset }`

Run: `yarn workspace @casehubio/pages-data run test -- --testPathPattern push-wire`
Expected: FAIL

- [ ] **Step 2: Implement push-wire.ts changes**

Update `push-wire.ts`:
- Add `let _reqCounter = 0;` module-level counter
- Add `export function nextRequestId(): string { return String(++_reqCounter); }`
- Update `sendListen`: add `id` as second parameter, add optional `since?: Record<string, number>`
- Update `sendUnlisten`: add `id` as second parameter
- Add `sendSubscribe(ws, id, dataset, since?)` and `sendUnsubscribe(ws, id, dataset)`

Run tests — all pass.

- [ ] **Step 3: Commit**

```
feat(push): push-wire.ts correlation support — id on all wire ops

nextRequestId() monotonic counter, sendSubscribe/sendUnsubscribe new,
sendListen/sendUnlisten gain id parameter. sendListen gains optional since map.

Refs #107
```

---

### Task 7: TypeScript — EventConnection evolution (ack/error + seq tracking + reconnect)

The largest TypeScript change. `listen()`/`unlisten()` become Promise-based with ack/error handling, pending request map with cleanup semantics, seq tracking with client-side dedup, and two-phase reconnect since construction. §1.5, §3.6 of the spec.

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/event-connection.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/event-connection.test.ts`
- Modify: `packages/pages-data/src/dataset/external/index.ts` — export `ListenAck`

**Interfaces:**
- Consumes: `nextRequestId`, `sendListen`, `sendUnlisten`, `dispatchWireEvent` from `push-wire.ts`
- Produces:
  - `EventConnection.listen(topics): Promise<ListenAck>`
  - `EventConnection.unlisten(topics): Promise<void>`
  - `ListenAck { topics: string[]; gaps?: string[] }`

- [ ] **Step 1: Write failing tests for ack/error handling**

Add to `event-connection.test.ts`:
- `listen` resolves on incoming ack with matching `id` → `ListenAck`
- `listen` rejects on incoming error with matching `id` → error message
- `unlisten` resolves on ack
- Pending request timeout → rejects after 10s, entry removed from map
- Late ack after timeout → no-op
- `close()` rejects all pending with "connection closed", clears map
- Connection reset (`onclose`) rejects all pending

Run: `yarn workspace @casehubio/pages-data run test -- --testPathPattern event-connection`
Expected: FAIL

- [ ] **Step 2: Write failing tests for seq tracking**

Add to `event-connection.test.ts`:
- Incoming event with numeric `seq` updates internal `topicSeqs` map
- Event with `seq <= tracked seq` is silently skipped (dedup)
- Reconnect sends `since` map with accumulated seq positions
- Reconnect `since` filters stale entries (after unlisten, excluded)
- Reconnect `since` includes wildcard matches
- Reconnect `since` seeds exact topics at 0

Expected: FAIL

- [ ] **Step 3: Implement EventConnection evolution**

Rewrite `event-connection.ts`:
- Add `pending: Map<string, { resolve, reject, timer }>` for request correlation
- `listen(topics)`: generate `id` via `nextRequestId()`, send listen, create Promise that stores resolve/reject in pending map, set timeout that rejects AND removes entry
- `unlisten(topics)`: same pattern, resolve void
- `handleMessage`: add ack/error routing — match by `id` in pending map, resolve/reject accordingly, clear timeout, delete entry
- Add `topicSeqs: Map<string, number>` for seq tracking
- In event handling: if `msg.seq` is a number, check dedup (`seq <= topicSeqs.get(topic)` → skip), then update `topicSeqs.set(topic, seq)`
- `unlisten()`: after removing from `listenRegistrations`, also clean `topicSeqs` entries for unlisted topics (or topics only matching removed wildcards)
- `close()` and `ws.onclose`: reject all pending entries, clear map
- Reconnect `onopen`: two-phase since construction per §3.6 — Phase 1 seeds exact topics, Phase 2 adds concrete positions from topicSeqs

Update `ListenAck` interface and export from index.ts.

Run tests — all pass.

- [ ] **Step 4: Commit**

```
feat(push): EventConnection ack/error + seq tracking + reconnect replay

listen()/unlisten() are Promise-based with 10s timeout. Pending request map
with cleanup on timeout, close, and connection reset. Per-topic seq tracking
with client-side dedup. Two-phase reconnect since construction for symmetric
exact/wildcard handling.

Refs #106, #107
```

---

### Task 8: TypeScript — websocket-source.ts id support

Add request `id` to subscribe/unsubscribe in the dataset path. Handle ack/error in the message handler. §1.7 of the spec.

**Files:**
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.ts`
- Modify: `packages/pages-data/src/dataset/external/sources/websocket-source.test.ts`

**Interfaces:**
- Consumes: `nextRequestId`, `sendSubscribe`, `sendUnsubscribe` from `push-wire.ts`
- Produces: updated subscribe/unsubscribe wire format (internal — no public API change)

- [ ] **Step 1: Write failing tests**

Add to `websocket-source.test.ts`:
- Subscribe sends wire message with `id` field
- Unsubscribe sends wire message with `id` field
- Ack message for subscribe is handled (logged, no crash)
- Error message for subscribe is handled (logged as warning)

Run: `yarn workspace @casehubio/pages-data run test -- --testPathPattern websocket-source`
Expected: FAIL

- [ ] **Step 2: Implement websocket-source.ts changes**

Update subscribe/unsubscribe calls to use `sendSubscribe`/`sendUnsubscribe` from push-wire.ts with `nextRequestId()`. Add ack/error routing to the message handler — ack is informational (debug log), error is a warning log. No Promise-based API for the dataset path (existing consumers don't need it).

Run tests — all pass.

- [ ] **Step 3: Run full TypeScript test suite**

Run: `yarn workspace @casehubio/pages-data run test`
Expected: all pass.

- [ ] **Step 4: Commit**

```
feat(push): websocket-source.ts subscribe/unsubscribe with request id

Dataset subscription wire ops now include correlation id. Ack/error
messages handled in message router (informational logging).

Refs #107
```

---

### Task 9: Full build verification + typecheck

Final verification that everything builds and type-checks across the entire monorepo.

**Files:** none (verification only)

- [ ] **Step 1: Full TypeScript build**

Run: `yarn build`
Expected: all packages build successfully.

- [ ] **Step 2: Full type check**

Run: `yarn typecheck`
Expected: no errors.

- [ ] **Step 3: Full Java build**

Run: `mvn -f backend/push/pom.xml verify`
Expected: all tests pass, build succeeds.

- [ ] **Step 4: Full TypeScript tests**

Run: `yarn test`
Expected: all workspace tests pass.

---

## Self-Review Checklist

**Spec coverage:**
- §1.1–§1.4 (wire format, PushRequest, seq, PushMessage) → Task 3 ✓
- §1.5 (EventConnection Promise API + cleanup) → Task 7 ✓
- §1.6 (push-wire.ts) → Task 6 ✓
- §1.7 (websocket-source.ts) → Task 8 ✓
- §1.8 (tests) → Tasks 3, 6, 7, 8 ✓
- §2.1–§2.4 (wildcard matching) → Task 4 ✓
- §2.6 (tests) → Task 4 ✓
- §3.1–§3.2 (EventStore SPI + InMemoryEventStore) → Task 5 ✓
- §3.3–§3.5 (wire format, replay delivery, server integration) → Tasks 3, 5 ✓ (types), server integration is app-level (documented in spec, not implemented in push library)
- §3.6 (EventConnection seq tracking) → Task 7 ✓
- §3.7 (tests) → Tasks 5, 7 ✓
- §4.1–§4.5 (token package) → Task 1 ✓
- §4.6–§4.8 (site.ts, deleted exports, component migration) → Task 2 ✓
- §4.9 (tests) → Tasks 1, 2 ✓

**Placeholder scan:** No TBD, TODO, or "implement later" found.

**Type consistency:** `ListenAck` used consistently across Task 7 (implementation) and spec. `ThemeConfig` used consistently across Task 1 (definition) and Task 2 (consumption). `EventStore`/`StoredEvent` types consistent between Task 5 definition and spec §3.5 integration pattern.
