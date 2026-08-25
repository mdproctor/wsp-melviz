# Scenario Controller UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-pages#341 — Scenario controller UI
**Issue group:** #341 (under epic casehubio/parent#408)

**Goal:** Build `<pages-scenario-controller>` and `<pages-scenario-narrative>` Lit
web components with a standalone `/scenario/remote` page, backed by
push wire state broadcasting and a REST outline endpoint.

**Architecture:** The orchestrator broadcasts `ScenarioState` on the
`scenario:state` push wire topic after every state mutation. A shared
`ScenarioConnectionController` (Lit ReactiveController) manages connection
lifecycle, topic listening, and state extraction for both components.
Commands are sent via REST. A recursive `OutlineNode` model projects
the scenario hierarchy for navigation.

**Tech Stack:** Java 21 (Quarkus), TypeScript 5, Lit 3, Vitest, Jackson

## Global Constraints

- Pre-release platform — breaking changes acceptable
- IntelliJ MCP required for Java edits — open workspace before starting:
  `ide_open_workspace({ modules: [".../backend/scenario", ".../backend/scenario-runtime", ".../backend/push", ".../backend/push-runtime"] })`
- TypeScript edits use standard tools (pages is a TS monorepo without IntelliJ project files)
- All commits reference `Refs #341`
- `lit` must be added as explicit dependency to pages-aria `package.json`
- Jackson `@JsonTypeInfo`/`@JsonSubTypes` on `NarrativeContent` for polymorphic wire format
- Custom element names use `pages-` prefix: `pages-scenario-controller`, `pages-scenario-narrative`
- pages-aria uses subpath exports in package.json — add `"./controller"` entry, not root index.ts
- `createEventConnection` must be imported statically — no dynamic `await import()`
- ScenarioOrchestrator uses constructor injection only — no `@Inject` field annotations
- Markdown rendering must sanitize HTML (DOMPurify or entity escaping) — no raw `unsafeHTML`

## Scope Notes

- **Start-from-remote deferred:** The current `POST /scenario/start` takes raw YAML content,
  not a scenario name. A `GET /scenario/list` endpoint and start UI in the controller are
  deferred to a follow-up issue. The remote page controls already-running scenarios only.
- **`runTo()` full semantics:** Implemented as part of Task 2 (max-speed fast-forward with
  pause at target label).

---

## Batch 1: Backend prerequisites and state broadcast

### Task 1: Jackson annotations on NarrativeContent + OutlineNode model

**Files:**
- Modify: `backend/scenario/src/main/java/io/casehub/pages/scenario/NarrativeContent.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/OutlineNode.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/NarrativeContentSerializationTest.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/OutlineNodeTest.java`

**Interfaces:**
- Produces: `NarrativeContent` with Jackson polymorphic annotations (wire format: `{"type":"inline","markdown":"..."}`)
- Produces: `OutlineNode(String label, String target, List<OutlineNode> children)` — recursive tree node

- [ ] **Step 1: Write failing test for NarrativeContent serialization**

```java
@Test
void inlineSerializesToTypeDiscriminator() throws Exception {
    var mapper = new ObjectMapper();
    NarrativeContent content = new NarrativeContent.Inline("Hello world");
    String json = mapper.writeValueAsString(content);
    var tree = mapper.readTree(json);
    assertEquals("inline", tree.get("type").asText());
    assertEquals("Hello world", tree.get("markdown").asText());
}

@Test
void templateSerializesWithType() throws Exception {
    var mapper = new ObjectMapper();
    NarrativeContent content = new NarrativeContent.Template("docs/intro.md", "overview", Map.of());
    String json = mapper.writeValueAsString(content);
    var tree = mapper.readTree(json);
    assertEquals("template", tree.get("type").asText());
    assertEquals("docs/intro.md", tree.get("path").asText());
}

@Test
void slideSerializesWithType() throws Exception {
    var mapper = new ObjectMapper();
    NarrativeContent content = new NarrativeContent.Slide("slide-3");
    String json = mapper.writeValueAsString(content);
    var tree = mapper.readTree(json);
    assertEquals("slide", tree.get("type").asText());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl backend/scenario -Dtest=NarrativeContentSerializationTest -f backend/pom.xml`
Expected: FAIL — no `type` property in serialized output

- [ ] **Step 3: Add Jackson annotations to NarrativeContent**

Add to `NarrativeContent.java`:

```java
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = NarrativeContent.Inline.class, name = "inline"),
    @JsonSubTypes.Type(value = NarrativeContent.Template.class, name = "template"),
    @JsonSubTypes.Type(value = NarrativeContent.Slide.class, name = "slide"),
})
public sealed interface NarrativeContent {
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl backend/scenario -Dtest=NarrativeContentSerializationTest -f backend/pom.xml`
Expected: PASS

- [ ] **Step 5: Write OutlineNode and test**

Create `OutlineNode.java`:

```java
package io.casehub.pages.scenario;

import java.util.List;

public record OutlineNode(String label, String target,
                          List<OutlineNode> children) {
    public OutlineNode(String label, List<OutlineNode> children) {
        this(label, null, children);
    }
    public OutlineNode(String label, String target) {
        this(label, target, List.of());
    }
}
```

Test that `OutlineNode` serializes correctly with Jackson (leaf node has empty children, non-leaf has null target).

- [ ] **Step 6: Run all tests, commit**

Run: `mvn test -pl backend/scenario -f backend/pom.xml`

```bash
git add backend/scenario/
git commit -m "feat(#341): Jackson annotations on NarrativeContent + OutlineNode model

Refs #341"
```

### Task 2: State broadcast and outline endpoint in ScenarioOrchestrator

**Files:**
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioOrchestrator.java`
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioControlResource.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioOrchestratorBroadcastTest.java`

**Interfaces:**
- Consumes: `EventBroadcaster` (from push-runtime, already CDI-produced)
- Consumes: `OutlineNode` (from Task 1)
- Produces: `broadcaster.broadcast("scenario:state", state())` after each mutation
- Produces: `outline()` method returning `List<OutlineNode>`
- Produces: `GET /scenario/outline` REST endpoint
- Produces: `stop()` method clearing session and broadcasting idle state

- [ ] **Step 1: Write failing test for state broadcast**

```java
@Test
void startBroadcastsState() {
    // Given: orchestrator with mock EventBroadcaster
    var captured = new ArrayList<Object>();
    var broadcaster = new EventBroadcaster(
            new InMemoryEventStore(100), new TopicRegistry(),
            (id, msg) -> {}, obj -> "{}") {
        @Override
        public <T> long broadcast(String topic, T event) {
            if ("scenario:state".equals(topic)) captured.add(event);
            return 0;
        }
    };
    var orchestrator = new ScenarioOrchestrator(sender, broadcaster);

    // When: start a scenario
    orchestrator.start(SIMPLE_YAML);

    // Then: state was broadcast
    assertFalse(captured.isEmpty());
    var state = (ScenarioState) captured.getLast();
    assertNotNull(state.scenario());
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `ScenarioOrchestrator` constructor doesn't accept `EventBroadcaster`

- [ ] **Step 3: Add EventBroadcaster to ScenarioOrchestrator**

Modify `ScenarioOrchestrator`:
1. Add `@Inject EventBroadcaster broadcaster` field
2. Update constructor to accept both `SessionSender` and `EventBroadcaster`
3. Add `private void broadcastState()` method calling `broadcaster.broadcast("scenario:state", state())`
4. Call `broadcastState()` at the end of: `start()`, `pause()`, `resume()`, `step()`, `speed()`, `onStepResult()`

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write failing test for stop()**

```java
@Test
void stopClearsSessionAndBroadcastsIdle() {
    orchestrator.start(SIMPLE_YAML);
    captured.clear();

    orchestrator.stop();

    var state = (ScenarioState) captured.getLast();
    assertNull(state.scenario());
    assertEquals(0.0, state.progress());
}
```

- [ ] **Step 6: Implement stop()**

```java
public void stop() {
    if (this.sessionId == null) return;
    broadcastControl("stop", null);   // notify executors BEFORE clearing sessionId
    this.sessionId = null;
    this.scenario = null;
    this.allSteps = List.of();
    this.completedSteps.clear();
    this.paused = false;
    this.speed = 1.0;
    this.runToTarget = null;
    broadcastState();                 // broadcast idle state to controllers
}
```

Wire `ScenarioControlResource.stop()` to call `orchestrator.stop()`.

- [ ] **Step 6b: Implement runTo() full semantics**

Add `runToTarget` field and implement max-speed fast-forward with pause-at-target:

```java
private volatile String runToTarget;

public RunToResult runTo(String label) {
    requireSession();
    int targetIndex = findStepIndex(label);
    if (targetIndex < 0) return RunToResult.NOT_FOUND;
    int currentIndex = completedSteps.size();
    if (targetIndex < currentIndex) return RunToResult.ALREADY_PAST;

    this.runToTarget = label;
    this.paused = false;
    broadcastControl("speed", 1000.0);  // max speed for fast-forward
    broadcastControl("resume", null);
    broadcastState();
    return RunToResult.OK;
}
```

In `onStepResult()`, add target check after recording the completed step:

```java
public void onStepResult(PushRequest.StepResult result) {
    if (sessionId == null || !sessionId.equals(result.sessionId())) return;
    completedSteps.put(result.stepName(), result.ok());

    // Check runTo target
    if (runToTarget != null && runToTarget.equals(result.stepName())) {
        runToTarget = null;
        this.speed = 1.0;  // restore normal speed
        pause();            // pause() broadcasts control + state
        return;
    }
    broadcastState();
}
```

- [ ] **Step 7: Write failing test for outline()**

```java
@Test
void outlineReturnsHierarchicalTree() {
    orchestrator.start(CHAPTERS_YAML);
    var outline = orchestrator.outline();

    assertEquals(2, outline.size()); // 2 chapters
    assertEquals("Customer Reports Issue", outline.get(0).label());
    assertNull(outline.get(0).target()); // chapter — no target
    assertFalse(outline.get(0).children().isEmpty());
    // leaf step has target
    var step = outline.get(0).children().get(0).children().get(0);
    assertNotNull(step.target());
}
```

- [ ] **Step 8: Implement outline() and wire REST endpoint**

```java
// In ScenarioOrchestrator:
public List<OutlineNode> outline() {
    if (scenario == null) return List.of();
    return buildOutline(scenario);
}

private List<OutlineNode> buildOutline(HierarchicalScenario s) {
    if (s.chapters() != null) {
        return s.chapters().stream()
            .map(c -> new OutlineNode(c.label(),
                c.sections().stream()
                    .map(sec -> new OutlineNode(sec.label(),
                        sec.steps().stream()
                            .map(st -> new OutlineNode(st.label(), st.target()))
                            .toList()))
                    .toList()))
            .toList();
    }
    if (s.sections() != null) {
        return s.sections().stream()
            .map(sec -> new OutlineNode(sec.label(),
                sec.steps().stream()
                    .map(st -> new OutlineNode(st.label(), st.target()))
                    .toList()))
            .toList();
    }
    if (s.steps() != null) {
        return s.steps().stream()
            .map(st -> new OutlineNode(st.label(), st.target()))
            .toList();
    }
    return List.of();
}
```

Add to `ScenarioControlResource`:

```java
@GET
@Path("/outline")
public List<OutlineNode> outline() {
    var result = orchestrator.outline();
    if (result.isEmpty() && orchestrator.sessionId() == null) {
        throw new NotFoundException("No active scenario");
    }
    return result;
}
```

- [ ] **Step 9: Run all tests, commit**

Run: `mvn test -pl backend/scenario-runtime -f backend/pom.xml`

```bash
git add backend/scenario-runtime/ backend/scenario/
git commit -m "feat(#341): scenario:state broadcast, stop(), outline endpoint

Refs #341"
```

---

## Batch 2: Frontend — scenario-controller component

### Task 3: Add lit dependency, scaffold reactive controller and component

**Files:**
- Modify: `packages/pages-aria/package.json` — add `lit` dependency + `"./controller"` subpath export
- Create: `packages/pages-aria/src/controller/index.ts`
- Create: `packages/pages-aria/src/controller/scenario-connection-controller.ts`
- Create: `packages/pages-aria/src/controller/scenario-controller.ts`
- Create: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: `EventConnection`, `createEventConnection` from `@casehubio/pages-data`
- Produces: `ScenarioConnectionController` — shared Lit ReactiveController for connection lifecycle
- Produces: `<pages-scenario-controller>` custom element

- [ ] **Step 1: Add lit dependency and subpath export**

```bash
yarn workspace @casehubio/pages-aria add lit
```

Add to `packages/pages-aria/package.json` exports:
```json
"./controller": "./src/controller/index.ts"
```

- [ ] **Step 2: Write failing test for component registration**

```typescript
// scenario-controller.test.ts
import { describe, it, expect } from 'vitest';
import './scenario-controller.js';

describe('pages-scenario-controller', () => {
  it('registers as custom element', () => {
    expect(customElements.get('pages-scenario-controller')).toBeDefined();
  });

  it('renders empty state when no connection', async () => {
    const el = document.createElement('pages-scenario-controller');
    document.body.appendChild(el);
    await (el as any).updateComplete;
    expect(el.shadowRoot?.textContent).toContain('No connection configured');
    el.remove();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose`
Expected: FAIL — module not found

- [ ] **Step 4: Create ScenarioConnectionController (shared reactive controller)**

```typescript
// scenario-connection-controller.ts
import type { ReactiveController, ReactiveControllerHost } from 'lit';
import { createEventConnection, type EventConnection } from '@casehubio/pages-data';

export interface ScenarioState {
  scenario: string | null;
  chapter: string | null;
  section: string | null;
  step: string | null;
  paused: boolean;
  speed: number;
  progress: number;
  content: { type: string; markdown?: string; path?: string; section?: string; ref?: unknown } | null;
  slides: string | null;
}

export interface ScenarioConnectionOptions {
  connection?: EventConnection;
  eventTarget?: EventTarget;
  baseUrl?: string;
  onState?: (state: ScenarioState) => void;
}

export class ScenarioConnectionController implements ReactiveController {
  private _host: ReactiveControllerHost;
  private _opts: ScenarioConnectionOptions;
  private _ownConnection?: EventConnection;
  private _ownEventTarget?: EventTarget;
  state: ScenarioState = {
    scenario: null, chapter: null, section: null, step: null,
    paused: false, speed: 1.0, progress: 0, content: null, slides: null,
  };
  connectionStatus: string = 'disconnected';

  constructor(host: ReactiveControllerHost, opts: ScenarioConnectionOptions) {
    this._host = host;
    this._opts = opts;
    host.addController(this);
  }

  get restBase(): string {
    return this._opts.baseUrl || window.location.origin;
  }

  private _eventHandler = (e: Event) => {
    const detail = (e as CustomEvent).detail as { topic?: string; payload?: unknown };
    if (detail?.topic !== 'scenario:state') return;
    this.state = detail.payload as ScenarioState;
    this._opts.onState?.(this.state);
    this._host.requestUpdate();
  };

  hostConnected(): void {
    const conn = this._resolveConnection();
    const target = this._resolveEventTarget();
    if (conn && target) {
      void conn.listen(['scenario:state']);
      target.addEventListener('pages-event', this._eventHandler);
      this.connectionStatus = conn.status;
    }
  }

  hostDisconnected(): void {
    const target = this._resolveEventTarget();
    if (target) target.removeEventListener('pages-event', this._eventHandler);
    const conn = this._resolveConnection();
    if (conn) void conn.unlisten(['scenario:state']);
    if (this._ownConnection) {
      this._ownConnection.close();
      this._ownConnection = undefined;
    }
  }

  private _resolveConnection(): EventConnection | undefined {
    if (this._opts.connection) return this._opts.connection;
    if (this._opts.baseUrl && !this._ownConnection) {
      const wsUrl = this._opts.baseUrl.replace(/^http/, 'ws') + '/ws/push';
      this._ownEventTarget = new EventTarget();
      this._ownConnection = createEventConnection(wsUrl, {
        config: { eventTarget: this._ownEventTarget },
      });
    }
    return this._ownConnection;
  }

  private _resolveEventTarget(): EventTarget | undefined {
    return this._opts.eventTarget ?? this._ownEventTarget;
  }

  async sendCommand(path: string, body?: object): Promise<void> {
    await fetch(`${this.restBase}/scenario${path}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      ...(body ? { body: JSON.stringify(body) } : {}),
    });
  }
}
```

- [ ] **Step 5: Create the component skeleton**

```typescript
// scenario-controller.ts
import { LitElement, html, css } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { EventConnection } from '@casehubio/pages-data';
import { ScenarioConnectionController, type ScenarioState } from './scenario-connection-controller.js';

interface OutlineNode {
  label: string;
  target: string | null;
  children: OutlineNode[];
}

export class PagesScenarioController extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .error { padding: var(--pages-space-4, 16px); color: var(--pages-danger-9, #dc2626); }
  `;

  @property({ attribute: false }) connection?: EventConnection;
  @property({ attribute: false }) eventTarget?: EventTarget;
  @property() baseUrl?: string;
  @state() private _outline: OutlineNode[] = [];

  private _conn!: ScenarioConnectionController;

  override connectedCallback(): void {
    this._conn = new ScenarioConnectionController(this, {
      connection: this.connection,
      eventTarget: this.eventTarget,
      baseUrl: this.baseUrl,
      onState: (s) => this._onStateChange(s),
    });
    super.connectedCallback();
  }

  private _onStateChange(s: ScenarioState): void {
    if (s.scenario && this._outline.length === 0) void this._fetchOutline();
    if (!s.scenario) this._outline = [];
  }

  private async _fetchOutline(): Promise<void> {
    try {
      const resp = await fetch(`${this._conn.restBase}/scenario/outline`);
      if (resp.ok) this._outline = await resp.json();
    } catch { /* ignore */ }
  }

  override render() {
    if (!this.connection && !this.baseUrl) {
      return html`<div class="error">No connection configured</div>`;
    }
    return html`<div>Controller placeholder</div>`;
  }
}

if (!customElements.get('pages-scenario-controller')) {
  customElements.define('pages-scenario-controller', PagesScenarioController);
}
```

Create `index.ts`:
```typescript
export { PagesScenarioController } from './scenario-controller.js';
export { ScenarioConnectionController } from './scenario-connection-controller.js';
```

- [ ] **Step 6: Run test to verify it passes**

- [ ] **Step 7: Commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): scaffold pages-scenario-controller with shared connection controller

Refs #341"
```

### Task 4: State listening and REST command tests

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: `ScenarioConnectionController` from Task 3 (handles connection lifecycle)
- Produces: state-driven rendering, REST command dispatch via controller

The connection lifecycle (listen, unlisten, mode resolution) is handled by
`ScenarioConnectionController`. This task verifies state updates flow through
the reactive controller to trigger re-renders.

- [ ] **Step 1: Write failing test for state update from push event**

```typescript
it('updates state from scenario:state event', async () => {
  const el = document.createElement('pages-scenario-controller') as PagesScenarioController;
  const mockTarget = new EventTarget();
  const mockConnection = {
    listen: vi.fn().mockResolvedValue({ topics: ['scenario:state'] }),
    unlisten: vi.fn().mockResolvedValue(undefined),
    send: vi.fn(),
    close: vi.fn(),
    connected: true,
    status: 'connected' as const,
  };
  el.connection = mockConnection;
  el.eventTarget = mockTarget;
  document.body.appendChild(el);
  await el.updateComplete;

  // Simulate push wire event
  mockTarget.dispatchEvent(new CustomEvent('pages-event', {
    detail: {
      topic: 'scenario:state',
      payload: {
        scenario: 'test-demo', chapter: 'Ch1', section: 'S1',
        step: 'Step 1', paused: false, speed: 1.0, progress: 0.25,
        content: null, slides: null,
      },
    },
  }));
  await el.updateComplete;

  expect(el.shadowRoot?.textContent).toContain('Step 1');
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — component renders placeholder, not state

- [ ] **Step 3: Wire render() to use ScenarioConnectionController state**

The controller's `_conn.state` is reactive (triggers `requestUpdate()` via
the ReactiveController). Update `render()` to read from `this._conn.state`.

- [ ] **Step 4: Write test for REST command dispatch**

```typescript
it('sends pause command via REST', async () => {
  global.fetch = vi.fn().mockResolvedValue({ ok: true, json: () => ({}) });
  // ... set up component with connection ...
  const pauseBtn = el.shadowRoot?.querySelector('[aria-label="Pause"]');
  pauseBtn?.click();
  expect(fetch).toHaveBeenCalledWith(
    expect.stringContaining('/scenario/pause'),
    expect.objectContaining({ method: 'POST' }),
  );
});
```

- [ ] **Step 5: Run tests, commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): state listening and REST command tests

Refs #341"
```

### Task 5: Outline tree and transport controls rendering

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: `_state`, `_outline`, `_sendCommand()` from Task 4
- Produces: fully rendered outline tree, transport bar, status bar

- [ ] **Step 1: Write failing test for outline rendering**

```typescript
it('renders outline tree with current position highlighted', async () => {
  // Set up component with mock connection
  // Set state to chapter 1, section 1, step 1
  // Set outline with 2 chapters, 2 sections each

  await el.updateComplete;

  const treeItems = el.shadowRoot?.querySelectorAll('[role="treeitem"]');
  expect(treeItems?.length).toBeGreaterThan(0);

  const current = el.shadowRoot?.querySelector('.current');
  expect(current?.textContent).toContain('Step 1');
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement outline tree rendering**

Add `_isBeforeCurrent()` helper — flattens the outline tree and checks if
a label appears before the current step:

```typescript
private _flattenLabels(nodes: OutlineNode[]): string[] {
  const result: string[] = [];
  for (const node of nodes) {
    if (node.children.length === 0) result.push(node.label);
    else result.push(...this._flattenLabels(node.children));
  }
  return result;
}

private _isBeforeCurrent(label: string): boolean {
  const labels = this._flattenLabels(this._outline);
  const currentIdx = labels.indexOf(this._conn.state.step ?? '');
  const labelIdx = labels.indexOf(label);
  return labelIdx >= 0 && currentIdx >= 0 && labelIdx < currentIdx;
}
```

Add `_renderOutline()` method that walks `_outline` recursively:

```typescript
private _renderOutline(): TemplateResult {
  if (this._outline.length === 0) {
    return html`<div class="outline-empty">No scenario running</div>`;
  }
  return html`
    <div class="outline" role="tree" aria-label="Scenario outline">
      ${this._outline.map(node => this._renderNode(node, 0))}
    </div>
  `;
}

private _renderNode(node: OutlineNode, depth: number): TemplateResult {
  const isLeaf = node.children.length === 0;
  const isCurrent = isLeaf && node.label === this._conn.state.step;
  const isCompleted = isLeaf && this._isBeforeCurrent(node.label);

  if (isLeaf) {
    return html`
      <div class="outline-step ${isCurrent ? 'current' : ''} ${isCompleted ? 'completed' : ''}"
           role="treeitem" tabindex="-1"
           style="padding-left: ${depth * 16 + 8}px"
           @click=${() => this._sendCommand('/run-to', { label: node.label })}>
        <span class="step-icon">${isCurrent ? '●' : isCompleted ? '✓' : '○'}</span>
        ${node.label}
      </div>
    `;
  }
  return html`
    <div class="outline-group" role="group">
      <div class="outline-heading" role="treeitem" tabindex="-1"
           style="padding-left: ${depth * 16}px"
           @click=${() => this._sendCommand('/run-to', { label: node.label })}>
        ${node.label}
      </div>
      ${node.children.map(child => this._renderNode(child, depth + 1))}
    </div>
  `;
}
```

- [ ] **Step 4: Write failing test for transport controls**

```typescript
it('toggles play/pause on button click', async () => {
  // Set state.paused = true
  await el.updateComplete;
  const playBtn = el.shadowRoot?.querySelector('[aria-label="Resume"]');
  expect(playBtn).toBeDefined();

  // Set state.paused = false
  await el.updateComplete;
  const pauseBtn = el.shadowRoot?.querySelector('[aria-label="Pause"]');
  expect(pauseBtn).toBeDefined();
});
```

- [ ] **Step 5: Implement transport controls**

Add `_renderTransport()` method:

```typescript
private _renderTransport(): TemplateResult {
  const hasScenario = !!this._conn.state.scenario;
  return html`
    <div class="transport">
      <button aria-label=${this._conn.state.paused ? 'Resume' : 'Pause'}
              ?disabled=${!hasScenario}
              @click=${() => this._sendCommand(this._conn.state.paused ? '/resume' : '/pause')}>
        ${this._conn.state.paused ? '▶' : '⏸'}
      </button>
      <button aria-label="Step" ?disabled=${!hasScenario}
              @click=${() => this._sendCommand('/step')}>⏩</button>
      <input type="range" min="-2" max="1" step="0.01"
             .value=${String(Math.log10(this._conn.state.speed))}
             ?disabled=${!hasScenario}
             aria-label="Speed" aria-valuetext="${this._conn.state.speed.toFixed(1)}x"
             @input=${this._onSpeedChange}>
      <span class="speed-label">${this._conn.state.speed.toFixed(1)}x</span>
      <span class="progress">${Math.round(this._conn.state.progress * 100)}%</span>
    </div>
  `;
}

private _speedDebounce: ReturnType<typeof setTimeout> | null = null;

private _onSpeedChange(e: Event): void {
  const logVal = parseFloat((e.target as HTMLInputElement).value);
  const speed = Math.pow(10, logVal);
  if (this._speedDebounce) clearTimeout(this._speedDebounce);
  this._speedDebounce = setTimeout(() => {
    void this._sendCommand('/speed', { speed: Math.round(speed * 100) / 100 });
  }, 250);
}
```

- [ ] **Step 6: Implement status bar and wire up render()**

```typescript
private _renderStatus(): TemplateResult {
  const breadcrumb = [this._conn.state.chapter, this._conn.state.section, this._conn.state.step]
    .filter(Boolean).join(' → ');
  return html`
    <div class="status-bar">
      <span class="breadcrumb">${breadcrumb || 'Idle'}</span>
      <span class="connection-status ${this._connectionStatus}">
        ● ${this._connectionStatus}
      </span>
    </div>
  `;
}

override render() {
  if (!this.connection && !this.baseUrl) {
    return html`<div class="error">No connection configured</div>`;
  }
  return html`
    ${this._renderOutline()}
    ${this._renderTransport()}
    ${this._renderStatus()}
  `;
}
```

- [ ] **Step 7: Add CSS styles**

Add comprehensive styles using pages-ui-tokens CSS custom properties for
outline tree, transport bar, status bar, current/completed step highlighting,
connection status dots.

- [ ] **Step 8: Run all tests, commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): outline tree, transport controls, status bar rendering

Refs #341"
```

---

## Batch 3: Narrative component and standalone remote page

### Task 6: `<pages-scenario-narrative>` component

**Files:**
- Create: `packages/pages-aria/src/controller/scenario-narrative.ts`
- Create: `packages/pages-aria/src/controller/scenario-narrative.test.ts`
- Modify: `packages/pages-aria/src/controller/index.ts` — add export

**Interfaces:**
- Consumes: `ScenarioConnectionController` from Task 3 (shared lifecycle)
- Produces: `<pages-scenario-narrative>` custom element rendering NarrativeContent

- [ ] **Step 1: Write failing test for inline markdown rendering**

```typescript
it('renders inline markdown content', async () => {
  const el = document.createElement('pages-scenario-narrative') as PagesScenarioNarrative;
  const mockTarget = new EventTarget();
  const mockConnection = {
    listen: vi.fn().mockResolvedValue({ topics: ['scenario:state'] }),
    unlisten: vi.fn().mockResolvedValue(undefined),
    send: vi.fn(), close: vi.fn(), connected: true, status: 'connected' as const,
  };
  el.connection = mockConnection;
  el.eventTarget = mockTarget;
  document.body.appendChild(el);
  await el.updateComplete;

  mockTarget.dispatchEvent(new CustomEvent('pages-event', {
    detail: {
      topic: 'scenario:state',
      payload: {
        scenario: 'test', content: { type: 'inline', markdown: '# Hello\n\nWorld' },
        chapter: null, section: null, step: null, paused: false, speed: 1.0, progress: 0, slides: null,
      },
    },
  }));
  await el.updateComplete;

  const rendered = el.shadowRoot?.querySelector('.narrative-content');
  expect(rendered?.innerHTML).toContain('<h1>');
  expect(rendered?.textContent).toContain('Hello');
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement scenario-narrative using ScenarioConnectionController**

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { property } from 'lit/decorators.js';
import type { EventConnection } from '@casehubio/pages-data';
import { ScenarioConnectionController, type ScenarioState } from './scenario-connection-controller.js';

export class PagesScenarioNarrative extends LitElement {
  static override styles = css`
    :host { display: block; }
    .narrative-content {
      padding: var(--pages-space-4, 16px);
      max-width: 680px; line-height: 1.6;
      font-size: var(--pages-font-size-base, 14px);
      font-family: var(--pages-font-family, system-ui, sans-serif);
    }
    .narrative-content h1 { font-size: 1.5em; margin: 0.5em 0; }
    .narrative-content h2 { font-size: 1.25em; margin: 0.5em 0; }
    .narrative-content p { margin: 0.5em 0; }
    .narrative-content code {
      background: var(--pages-neutral-3, #f5f5f5);
      padding: 2px 4px; border-radius: 3px;
    }
  `;

  @property({ attribute: false }) connection?: EventConnection;
  @property({ attribute: false }) eventTarget?: EventTarget;
  @property() baseUrl?: string;

  private _conn!: ScenarioConnectionController;

  override connectedCallback(): void {
    this._conn = new ScenarioConnectionController(this, {
      connection: this.connection,
      eventTarget: this.eventTarget,
      baseUrl: this.baseUrl,
    });
    super.connectedCallback();
  }

  override render() {
    const content = this._conn?.state?.content;
    if (!content) return nothing;

    switch (content.type) {
      case 'inline':
        return html`<div class="narrative-content">${this._renderMarkdown(content.markdown ?? '')}</div>`;
      case 'template':
        return html`<div class="narrative-content">Loading template...</div>`;
      case 'slide':
        return html`<div class="narrative-content">Slide: ${String(content.ref)}</div>`;
      default:
        return nothing;
    }
  }

  private _renderMarkdown(md: string): unknown {
    // Sanitize: escape HTML entities FIRST, then apply markdown formatting
    const escaped = md
      .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
    const rendered = escaped
      .replace(/^### (.+)$/gm, '<h3>$1</h3>')
      .replace(/^## (.+)$/gm, '<h2>$1</h2>')
      .replace(/^# (.+)$/gm, '<h1>$1</h1>')
      .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
      .replace(/\*(.+?)\*/g, '<em>$1</em>')
      .replace(/`(.+?)`/g, '<code>$1</code>')
      .replace(/^- (.+)$/gm, '<li>$1</li>')
      .replace(/\n\n/g, '</p><p>')
      .replace(/^(?!<[hulo])(.+)$/gm, '<p>$1</p>');

    const container = document.createElement('div');
    container.innerHTML = rendered;
    return html`${Array.from(container.childNodes)}`;
  }
}

if (!customElements.get('pages-scenario-narrative')) {
  customElements.define('pages-scenario-narrative', PagesScenarioNarrative);
}
```

Note: HTML is sanitized by escaping `<`, `>`, `&` before markdown processing,
preventing XSS. For production use with richer markdown, consider `marked` +
`DOMPurify`.

- [ ] **Step 4: Write test for empty content (renders nothing)**

```typescript
it('renders nothing when content is null', async () => {
  // Simulate state with content: null
  await el.updateComplete;
  expect(el.shadowRoot?.querySelector('.narrative-content')).toBeNull();
});
```

- [ ] **Step 5: Run all tests, commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): pages-scenario-narrative component with sanitized markdown

Refs #341"
```

### Task 7: Standalone remote page and bundle

**Files:**
- Create: `backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/remote.html`
- Create: `packages/pages-aria/src/controller/standalone.ts` — bundle entry point
- Modify: `packages/pages-aria/package.json` — add build:controller script

**Interfaces:**
- Consumes: `ScenarioController` from Task 3-5
- Produces: `/scenario/remote` page served by Quarkus
- Produces: `/scenario/controller.js` standalone bundle

- [ ] **Step 1: Create standalone entry point**

```typescript
// standalone.ts — bundle entry for remote page
import './scenario-controller.js';
import './scenario-narrative.js';
```

- [ ] **Step 2: Add build script to package.json**

Add to `packages/pages-aria/package.json` scripts:

```json
"build:controller": "esbuild src/controller/standalone.ts --bundle --format=esm --outfile=dist/controller.js --external:nothing"
```

Or use the existing build pipeline if pages-aria has one.

- [ ] **Step 3: Create remote.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Scenario Remote</title>
  <style>
    body { margin: 0; font-family: system-ui, sans-serif;
           background: var(--pages-neutral-1, #fafafa); }
    pages-scenario-controller { display: block; width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <pages-scenario-controller id="ctrl"></scenario-controller>
  <script type="module">
    import '/scenario/controller.js';
    document.getElementById('ctrl').baseUrl = window.location.origin;
  </script>
</body>
</html>
```

- [ ] **Step 4: Build and verify bundle**

Run: `yarn workspace @casehubio/pages-aria run build:controller`
Verify: `dist/controller.js` exists and is loadable.

- [ ] **Step 5: Copy bundle to META-INF/resources**

Add a build step or Maven resource filter that copies
`packages/pages-aria/dist/controller.js` to
`backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/controller.js`.

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/ backend/scenario-runtime/src/main/resources/
git commit -m "feat(#341): standalone /scenario/remote page with bundled controller

Refs #341"
```

---

## Batch 4: Integration verification

### Task 8: E2E integration test

**Files:**
- Test: `packages/pages-aria/src/controller/scenario-controller.integration.test.ts`

**Interfaces:**
- Consumes: all previous tasks
- Produces: verified end-to-end flow

- [ ] **Step 1: Write integration test**

Test the full flow: create a mock WebSocket server that sends
`scenario:state` events, verify the controller updates its outline
and state display, verify REST commands are sent on button clicks.

This can use Vitest with a mock WebSocket (via `vitest-websocket-mock`
or a simple mock) rather than full Playwright, since the component
tests can verify DOM rendering without a real browser.

- [ ] **Step 2: Run the full test suite**

```bash
yarn workspace @casehubio/pages-aria run test
```

- [ ] **Step 3: Run the Java backend tests**

```bash
mvn test -pl backend/scenario,backend/scenario-runtime -f backend/pom.xml
```

- [ ] **Step 4: Build everything**

```bash
yarn build:packages
```

- [ ] **Step 5: Commit final state**

```bash
git add .
git commit -m "feat(#341): scenario controller UI — complete implementation

Closes casehubio/casehub-pages#341
Refs casehubio/parent#408"
```

---

## References

- [2026-08-21-scenario-controller-ui-design.md] — design spec this plan implements
- [2026-08-20-distributed-executor-protocol-design.md] — protocol spec (§5 Controller API, §7 Controller UI)
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioOrchestrator.java` — orchestrator (no broadcast currently)
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioControlResource.java` — REST endpoints
- `backend/scenario/src/main/java/io/casehub/pages/scenario/NarrativeContent.java` — sealed interface needing Jackson annotations
- `packages/pages-data/src/dataset/external/sources/event-connection.ts` — push wire client
- `packages/pages-aria/src/server/scenario-handler.ts` — browser executor pattern (connection + eventTarget)
- `packages/pages-ui-components/src/button/pages-button.ts` — existing Lit component pattern
- [GE-20260818-c61c29] — topicSource adapter technique
- [GE-20260816-e89cda] — composable Lit reactive controllers technique
- casehubio/casehub-pages#341 — focal issue
- casehubio/parent#408 — parent epic
