# YAML Fly-Out Viewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #349 — Controller YAML fly-out with position tracking
**Issue group:** #349 (part of epic casehubio/parent#408)

**Goal:** Add a floating YAML viewer to the scenario controller that
shows syntax-highlighted source with live position tracking and
detach-to-window support.

**Architecture:** A standalone `<pages-scenario-yaml-viewer>` Lit
component connects to the push wire independently via
`ScenarioConnectionController`. It fetches scenario YAML, tokenizes it
for syntax highlighting (regex), and maps step labels to line ranges
(yaml `parseDocument` AST). The controller toggles it as a second
floating element. A pop-out button opens a minimal HTML page with the
same component connected via its own push wire.

**Tech Stack:** Lit 3, yaml ^2.9.0 (existing dependency), esbuild
(existing controller bundle)

## Global Constraints

- Zero new dependencies — yaml package already in pages-aria
- Component lives in `packages/pages-aria/src/controller/`
- Dark-glass visual style matching existing compact controller card
- Bundle via existing esbuild `build:controller` script in pages-aria
- IntelliJ MCP for all .ts file operations (except slot clone — use
  Edit with `replace_all: true` if IntelliJ cannot see the file)

---

## Batch 1: YAML highlighter module

### Task 1: YAML syntax tokenizer and position mapper

**Files:**
- Create: `packages/pages-aria/src/controller/yaml-highlighter.ts`
- Test: `packages/pages-aria/src/controller/yaml-highlighter.test.ts`

**Interfaces:**
- Produces:
  - `tokenizeYamlLine(line: string): YamlToken[]`
    where `YamlToken = { text: string; type: 'key' | 'string' | 'comment' | 'literal' | 'punct' | 'plain' }`
  - `buildStepLineMap(yamlSource: string): Map<string, LineRange>`
    where `LineRange = { startLine: number; endLine: number }`

- [ ] **Step 1: Write failing tests for tokenizeYamlLine**

```typescript
import { describe, it, expect } from 'vitest';
import { tokenizeYamlLine, buildStepLineMap, type YamlToken } from './yaml-highlighter.js';

describe('tokenizeYamlLine', () => {
  it('tokenizes a key-value pair', () => {
    const tokens = tokenizeYamlLine('scenario: help-desk-demo');
    expect(tokens).toEqual([
      { text: 'scenario', type: 'key' },
      { text: ': ', type: 'punct' },
      { text: 'help-desk-demo', type: 'plain' },
    ]);
  });

  it('tokenizes a quoted string value', () => {
    const tokens = tokenizeYamlLine('  label: "Customer submits ticket"');
    expect(tokens).toEqual([
      { text: '  ', type: 'plain' },
      { text: 'label', type: 'key' },
      { text: ': ', type: 'punct' },
      { text: '"Customer submits ticket"', type: 'string' },
    ]);
  });

  it('tokenizes a comment', () => {
    const tokens = tokenizeYamlLine('# This is a comment');
    expect(tokens).toEqual([
      { text: '# This is a comment', type: 'comment' },
    ]);
  });

  it('tokenizes a list item', () => {
    const tokens = tokenizeYamlLine('  - action: click');
    expect(tokens).toEqual([
      { text: '  ', type: 'plain' },
      { text: '- ', type: 'punct' },
      { text: 'action', type: 'key' },
      { text: ': ', type: 'punct' },
      { text: 'click', type: 'plain' },
    ]);
  });

  it('tokenizes boolean and number literals', () => {
    const tokens = tokenizeYamlLine('  paused: true');
    expect(tokens).toEqual([
      { text: '  ', type: 'plain' },
      { text: 'paused', type: 'key' },
      { text: ': ', type: 'punct' },
      { text: 'true', type: 'literal' },
    ]);
  });

  it('tokenizes inline comment after value', () => {
    const tokens = tokenizeYamlLine('  speed: 0.5 # half speed');
    expect(tokens).toEqual([
      { text: '  ', type: 'plain' },
      { text: 'speed', type: 'key' },
      { text: ': ', type: 'punct' },
      { text: '0.5 ', type: 'plain' },
      { text: '# half speed', type: 'comment' },
    ]);
  });

  it('handles empty line', () => {
    expect(tokenizeYamlLine('')).toEqual([]);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-aria run test -- --run src/controller/yaml-highlighter.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement tokenizeYamlLine**

Create `packages/pages-aria/src/controller/yaml-highlighter.ts`:

```typescript
export interface YamlToken {
  text: string;
  type: 'key' | 'string' | 'comment' | 'literal' | 'punct' | 'plain';
}

export interface LineRange {
  startLine: number;
  endLine: number;
}

const LITERAL_RE = /^(true|false|null|\d+(\.\d+)?)$/;

export function tokenizeYamlLine(line: string): YamlToken[] {
  if (line.length === 0) return [];

  const tokens: YamlToken[] = [];
  let rest = line;

  // Leading whitespace
  const leadingMatch = rest.match(/^(\s+)/);
  if (leadingMatch) {
    tokens.push({ text: leadingMatch[1], type: 'plain' });
    rest = rest.slice(leadingMatch[1].length);
  }

  // Full-line comment
  if (rest.startsWith('#')) {
    tokens.push({ text: rest, type: 'comment' });
    return tokens;
  }

  // List indicator
  if (rest.startsWith('- ')) {
    tokens.push({ text: '- ', type: 'punct' });
    rest = rest.slice(2);
  }

  // Key: value
  const kvMatch = rest.match(/^([\w][\w.-]*)(:\s*)/);
  if (kvMatch) {
    tokens.push({ text: kvMatch[1], type: 'key' });
    tokens.push({ text: kvMatch[2], type: 'punct' });
    rest = rest.slice(kvMatch[0].length);
  }

  if (rest.length === 0) return tokens;

  // Remaining value — check for quoted strings, inline comments, literals
  // Quoted string
  const quotedMatch = rest.match(/^("[^"]*"|'[^']*')/);
  if (quotedMatch) {
    tokens.push({ text: quotedMatch[1], type: 'string' });
    rest = rest.slice(quotedMatch[1].length);
  } else {
    // Inline comment split
    const commentIdx = rest.indexOf(' #');
    if (commentIdx >= 0) {
      const beforeComment = rest.slice(0, commentIdx + 1);
      const comment = rest.slice(commentIdx + 1);
      if (beforeComment.trim().length > 0) {
        const trimmed = beforeComment.trim();
        tokens.push({
          text: beforeComment,
          type: LITERAL_RE.test(trimmed) ? 'literal' : 'plain',
        });
      }
      tokens.push({ text: comment, type: 'comment' });
      rest = '';
    } else if (rest.trim().length > 0) {
      const trimmed = rest.trim();
      tokens.push({
        text: rest,
        type: LITERAL_RE.test(trimmed) ? 'literal' : 'plain',
      });
      rest = '';
    }
  }

  // Trailing after quoted string (e.g., inline comment)
  if (rest.length > 0) {
    const trailingComment = rest.match(/^(\s*)(#.*)/);
    if (trailingComment) {
      if (trailingComment[1]) tokens.push({ text: trailingComment[1], type: 'plain' });
      tokens.push({ text: trailingComment[2], type: 'comment' });
    } else if (rest.trim().length > 0) {
      tokens.push({ text: rest, type: 'plain' });
    }
  }

  return tokens;
}
```

- [ ] **Step 4: Run tokenizer tests to verify they pass**

Run: `yarn workspace @casehubio/pages-aria run test -- --run src/controller/yaml-highlighter.test.ts`
Expected: All tokenizer tests PASS

- [ ] **Step 5: Write failing tests for buildStepLineMap**

Append to `yaml-highlighter.test.ts`:

```typescript
describe('buildStepLineMap', () => {
  const SAMPLE_YAML = `scenario: help-desk-demo
description: "Full helpdesk demo"
speed: 0.5

sections:
  - label: "Customer submits ticket"
    steps:
      - label: "Load demo classifications"
        name: load-demo-data
        target: browser
        commands:
          - action: click
            target: {role: button, name: Load demo classification data}

      - label: "Fill in customer name"
        name: fill-name
        target: browser
        commands:
          - action: fill
            target: {role: textbox, name: Your name}
            value: "Alice"

  - label: "Backend processes ticket"
    steps:
      - label: "System creates and classifies ticket"
        name: verify-ticket
        target: helpdesk
        commands:
          - action: verify-ticket-exists
`;

  it('maps step labels to line ranges', () => {
    const map = buildStepLineMap(SAMPLE_YAML);
    expect(map.has('Load demo classifications')).toBe(true);
    const range = map.get('Load demo classifications')!;
    expect(range.startLine).toBeGreaterThan(0);
    expect(range.endLine).toBeGreaterThan(range.startLine);
  });

  it('maps step names to line ranges', () => {
    const map = buildStepLineMap(SAMPLE_YAML);
    expect(map.has('load-demo-data')).toBe(true);
  });

  it('maps all steps', () => {
    const map = buildStepLineMap(SAMPLE_YAML);
    expect(map.has('Load demo classifications')).toBe(true);
    expect(map.has('Fill in customer name')).toBe(true);
    expect(map.has('System creates and classifies ticket')).toBe(true);
  });

  it('returns empty map for non-scenario YAML', () => {
    const map = buildStepLineMap('key: value');
    expect(map.size).toBe(0);
  });
});
```

- [ ] **Step 6: Implement buildStepLineMap**

Append to `yaml-highlighter.ts`:

```typescript
import { parseDocument } from 'yaml';

function offsetToLine(source: string, offset: number): number {
  let line = 1;
  for (let i = 0; i < offset && i < source.length; i++) {
    if (source[i] === '\n') line++;
  }
  return line;
}

export function buildStepLineMap(yamlSource: string): Map<string, LineRange> {
  const map = new Map<string, LineRange>();

  let doc;
  try {
    doc = parseDocument(yamlSource);
  } catch {
    return map;
  }

  const root = doc.get('sections');
  if (!root || !('items' in root)) return map;

  for (const section of (root as any).items) {
    const steps = section.get('steps');
    if (!steps || !('items' in steps)) continue;

    for (const step of (steps as any).items) {
      const range = step.range;
      if (!range) continue;

      const label = step.get('label') as string | undefined;
      const name = step.get('name') as string | undefined;
      const startLine = offsetToLine(yamlSource, range[0]);
      const endLine = offsetToLine(yamlSource, range[2] - 1);

      if (label) map.set(label, { startLine, endLine });
      if (name) map.set(name, { startLine, endLine });
    }
  }

  return map;
}
```

- [ ] **Step 7: Run all tests to verify they pass**

Run: `yarn workspace @casehubio/pages-aria run test -- --run src/controller/yaml-highlighter.test.ts`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/pages add packages/pages-aria/src/controller/yaml-highlighter.ts packages/pages-aria/src/controller/yaml-highlighter.test.ts
git commit -m "feat(#349): YAML tokenizer and step-to-line position mapper

Refs casehubio/casehub-pages#349"
```

---

## Batch 2: YAML viewer component

### Task 2: `<pages-scenario-yaml-viewer>` Lit component

**Files:**
- Create: `packages/pages-aria/src/controller/scenario-yaml-viewer.ts`
- Test: `packages/pages-aria/src/controller/scenario-yaml-viewer.test.ts`
- Modify: `packages/pages-aria/src/controller/index.ts` — add export
- Modify: `packages/pages-aria/src/controller/standalone.ts` — add import

**Interfaces:**
- Consumes: `tokenizeYamlLine`, `buildStepLineMap`, `LineRange` from yaml-highlighter
- Consumes: `ScenarioConnectionController`, `ScenarioState` from scenario-connection-controller
- Produces: `PagesScenarioYamlViewer` custom element (`pages-scenario-yaml-viewer`)
  - Properties: `connection?: EventConnection`, `eventTarget?: EventTarget`, `baseUrl?: string`, `scenario?: string`

- [ ] **Step 1: Write failing component test**

Create `packages/pages-aria/src/controller/scenario-yaml-viewer.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import './scenario-yaml-viewer.js';

describe('PagesScenarioYamlViewer', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('pages-scenario-yaml-viewer');
  });

  it('is defined as a custom element', () => {
    expect(customElements.get('pages-scenario-yaml-viewer')).toBeDefined();
  });

  it('renders empty state when no scenario', async () => {
    document.body.appendChild(el);
    await (el as any).updateComplete;
    const shadow = el.shadowRoot!;
    expect(shadow.querySelector('.yaml-empty')).not.toBeNull();
    document.body.removeChild(el);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --run src/controller/scenario-yaml-viewer.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement PagesScenarioYamlViewer**

Create `packages/pages-aria/src/controller/scenario-yaml-viewer.ts`:

```typescript
import { LitElement, html, css, nothing, type TemplateResult } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { EventConnection } from '@casehubio/pages-data';
import { ScenarioConnectionController, type ScenarioState } from './scenario-connection-controller.js';
import { tokenizeYamlLine, buildStepLineMap, type LineRange } from './yaml-highlighter.js';

export class PagesScenarioYamlViewer extends LitElement {
  static override styles = css`
    :host {
      display: block;
      font-family: var(--pages-font-family, system-ui, sans-serif);
      font-size: var(--pages-font-size-sm, 12px);
      position: fixed;
      bottom: 16px;
      right: 320px;
      z-index: 9998;
    }
    .viewer-card {
      background: rgba(15, 23, 42, 0.95);
      backdrop-filter: blur(12px);
      border-radius: var(--pages-radius-lg, 8px);
      box-shadow: 0 8px 24px rgba(0,0,0,0.4);
      color: #e2e8f0;
      width: 360px;
      max-height: 60vh;
      overflow: hidden;
      display: flex;
      flex-direction: column;
    }
    .viewer-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 8px 12px;
      cursor: grab;
      border-bottom: 1px solid rgba(255,255,255,0.1);
      gap: 8px;
    }
    .viewer-header:active { cursor: grabbing; }
    .viewer-title {
      color: #94a3b8;
      font-size: 12px;
      flex: 1;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    .viewer-header button {
      background: none;
      border: none;
      color: #64748b;
      cursor: pointer;
      font-size: 14px;
      padding: 0 2px;
      line-height: 1;
    }
    .viewer-header button:hover { color: #e2e8f0; }
    .viewer-body {
      overflow-y: auto;
      flex: 1;
      padding: 8px 0;
    }
    .yaml-empty {
      padding: 16px;
      color: #64748b;
      font-style: italic;
      text-align: center;
    }
    .yaml-line {
      padding: 0 12px;
      font-family: 'SF Mono', 'Fira Code', monospace;
      font-size: 11px;
      line-height: 1.6;
      white-space: pre;
      display: flex;
    }
    .yaml-line.active {
      background: rgba(56, 189, 248, 0.12);
      border-left: 2px solid #38bdf8;
      padding-left: 10px;
    }
    .line-num {
      color: #475569;
      min-width: 28px;
      text-align: right;
      padding-right: 8px;
      user-select: none;
      flex-shrink: 0;
    }
    .yaml-key { color: #7dd3fc; }
    .yaml-string { color: #86efac; }
    .yaml-comment { color: #64748b; font-style: italic; }
    .yaml-literal { color: #fbbf24; }
    .yaml-punct { color: #94a3b8; }
    .yaml-plain { color: #e2e8f0; }
  `;

  @property({ attribute: false }) connection?: EventConnection;
  @property({ attribute: false }) eventTarget?: EventTarget;
  @property({ attribute: 'baseurl' }) baseUrl?: string;
  @property() scenario?: string;

  @state() private _yamlSource = '';
  @state() private _activeStep: string | null = null;

  private _conn!: ScenarioConnectionController;
  private _stepMap = new Map<string, LineRange>();
  private _dragOffset = { x: 0, y: 0 };

  onClose?: () => void;
  onDetach?: () => void;

  protected override firstUpdated(): void {
    this._conn = new ScenarioConnectionController(this, {
      connection: this.connection,
      eventTarget: this.eventTarget,
      baseUrl: this.baseUrl,
      onState: (s: ScenarioState) => this._onStateChange(s),
    });
    if (this.scenario) void this._fetchYaml();
  }

  override updated(changed: Map<string, unknown>): void {
    if (changed.has('scenario') && this.scenario) void this._fetchYaml();
  }

  private _onStateChange(s: ScenarioState): void {
    this._activeStep = s.step;
    if (s.scenario && !this._yamlSource && this.scenario) void this._fetchYaml();
    this.requestUpdate();
    this._scrollToActive();
  }

  private async _fetchYaml(): Promise<void> {
    if (!this.scenario) return;
    try {
      const base = this._conn?.restBase ?? this.baseUrl ?? '';
      const resp = await fetch(`${base}/scenarios/${this.scenario}.yaml`);
      if (resp.ok) {
        this._yamlSource = await resp.text();
        this._stepMap = buildStepLineMap(this._yamlSource);
      }
    } catch { /* retry on next state change */ }
  }

  private _scrollToActive(): void {
    if (!this._activeStep) return;
    const range = this._stepMap.get(this._activeStep);
    if (!range) return;
    requestAnimationFrame(() => {
      const body = this.shadowRoot?.querySelector('.viewer-body');
      const activeLine = this.shadowRoot?.querySelector('.yaml-line.active');
      if (body && activeLine) {
        activeLine.scrollIntoView({ block: 'center', behavior: 'smooth' });
      }
    });
  }

  private _getActiveRange(): LineRange | null {
    if (!this._activeStep) return null;
    return this._stepMap.get(this._activeStep) ?? null;
  }

  override render(): TemplateResult {
    return html`
      <div class="viewer-card">
        ${this._renderHeader()}
        <div class="viewer-body">
          ${this._yamlSource
            ? this._renderYaml()
            : html`<div class="yaml-empty">No scenario source loaded</div>`}
        </div>
      </div>
    `;
  }

  private _renderHeader(): TemplateResult {
    return html`
      <div class="viewer-header" @pointerdown=${this._onDragStart}>
        <span class="viewer-title">${this.scenario ?? 'YAML Source'}</span>
        <button aria-label="Detach to window" @click=${() => this.onDetach?.()}>⧉</button>
        <button aria-label="Close" @click=${() => this.onClose?.()}>✕</button>
      </div>
    `;
  }

  private _renderYaml(): TemplateResult {
    const lines = this._yamlSource.split('\n');
    const activeRange = this._getActiveRange();

    return html`${lines.map((line, i) => {
      const lineNum = i + 1;
      const isActive = activeRange
        && lineNum >= activeRange.startLine
        && lineNum <= activeRange.endLine;
      const tokens = tokenizeYamlLine(line);

      return html`
        <div class="yaml-line ${isActive ? 'active' : ''}">
          <span class="line-num">${lineNum}</span>
          <span>${tokens.map(t =>
            html`<span class="yaml-${t.type}">${t.text}</span>`
          )}</span>
        </div>
      `;
    })}`;
  }

  private _onDragStart = (e: PointerEvent): void => {
    if ((e.target as HTMLElement).tagName === 'BUTTON') return;
    const host = this.getBoundingClientRect();
    this._dragOffset = { x: e.clientX - host.left, y: e.clientY - host.top };
    (e.currentTarget as HTMLElement).setPointerCapture(e.pointerId);
    (e.currentTarget as HTMLElement).addEventListener('pointermove', this._onDragMove);
    (e.currentTarget as HTMLElement).addEventListener('pointerup', this._onDragEnd);
  };

  private _onDragMove = (e: PointerEvent): void => {
    this.style.left = `${e.clientX - this._dragOffset.x}px`;
    this.style.top = `${e.clientY - this._dragOffset.y}px`;
    this.style.right = 'auto';
    this.style.bottom = 'auto';
  };

  private _onDragEnd = (e: PointerEvent): void => {
    (e.currentTarget as HTMLElement).releasePointerCapture(e.pointerId);
    (e.currentTarget as HTMLElement).removeEventListener('pointermove', this._onDragMove);
    (e.currentTarget as HTMLElement).removeEventListener('pointerup', this._onDragEnd);
  };
}

if (!customElements.get('pages-scenario-yaml-viewer')) {
  customElements.define('pages-scenario-yaml-viewer', PagesScenarioYamlViewer);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --run src/controller/scenario-yaml-viewer.test.ts`
Expected: PASS

- [ ] **Step 5: Update exports — index.ts and standalone.ts**

In `packages/pages-aria/src/controller/index.ts`, add:
```typescript
export { PagesScenarioYamlViewer } from './scenario-yaml-viewer.js';
```

In `packages/pages-aria/src/controller/standalone.ts`, add:
```typescript
import './scenario-yaml-viewer.js';
```

- [ ] **Step 6: Build controller bundle**

Run: `yarn workspace @casehubio/pages-aria run build:controller`
Expected: Build succeeds, `dist/controller.js` includes yaml viewer

- [ ] **Step 7: Run all pages-aria tests**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: All tests PASS (existing + new)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/pages add packages/pages-aria/src/controller/scenario-yaml-viewer.ts packages/pages-aria/src/controller/scenario-yaml-viewer.test.ts packages/pages-aria/src/controller/index.ts packages/pages-aria/src/controller/standalone.ts
git commit -m "feat(#349): YAML viewer component with syntax highlighting and position tracking

Refs casehubio/casehub-pages#349"
```

---

## Batch 3: Controller integration and detach

### Task 3: Toggle button in controller + viewer lifecycle

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts` — add toggle, viewer creation, detach
- Modify: `packages/pages-aria/src/controller/scenario-controller.test.ts` — add toggle tests

**Interfaces:**
- Consumes: `PagesScenarioYamlViewer` from scenario-yaml-viewer

- [ ] **Step 1: Write failing test for toggle button**

Append to `scenario-controller.test.ts`:

```typescript
describe('YAML viewer toggle', () => {
  it('renders source toggle button in compact card', async () => {
    // Create controller in compact mode with baseUrl
    const el = document.createElement('pages-scenario-controller') as any;
    el.mode = 'compact';
    el.baseUrl = 'http://localhost:8080';
    document.body.appendChild(el);
    await el.updateComplete;
    // Expand the card
    el._expanded = true;
    await el.updateComplete;
    const shadow = el.shadowRoot!;
    const btn = shadow.querySelector('[aria-label="Toggle source"]');
    expect(btn).not.toBeNull();
    document.body.removeChild(el);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --run src/controller/scenario-controller.test.ts`
Expected: FAIL — no element with aria-label="Toggle source"

- [ ] **Step 3: Add toggle button and viewer lifecycle to scenario-controller.ts**

Add state and methods to `PagesScenarioController`:

1. Add `@state() private _yamlOpen = false;`
2. Add `private _yamlViewer: PagesScenarioYamlViewer | null = null;`
3. Add `private _popoutWindow: Window | null = null;`
4. Add `private _popoutPoll: ReturnType<typeof setInterval> | null = null;`

Add `_renderSourceToggle()` method:
```typescript
private _renderSourceToggle(): TemplateResult {
  return html`
    <button aria-label="Toggle source"
            @click=${() => this._toggleYaml()}>
      &lt;/&gt;
    </button>
  `;
}
```

Add `_toggleYaml()` method:
```typescript
private _toggleYaml(): void {
  this._yamlOpen = !this._yamlOpen;
  if (this._yamlOpen) {
    this._showYamlViewer();
  } else {
    this._hideYamlViewer();
  }
}
```

Add `_showYamlViewer()` method:
```typescript
private _showYamlViewer(): void {
  if (!this._yamlViewer) {
    this._yamlViewer = document.createElement('pages-scenario-yaml-viewer') as PagesScenarioYamlViewer;
    this._yamlViewer.connection = this.connection;
    this._yamlViewer.eventTarget = this.eventTarget;
    if (this.baseUrl) this._yamlViewer.baseUrl = this.baseUrl;
    if (this.scenario) this._yamlViewer.scenario = this.scenario;
    if (this._conn?.state.scenario) {
      this._yamlViewer.scenario = this._yamlViewer.scenario ?? this._conn.state.scenario;
    }
    this._yamlViewer.onClose = () => this._toggleYaml();
    this._yamlViewer.onDetach = () => this._detachYaml();
    document.body.appendChild(this._yamlViewer);
  }
  this._yamlViewer.style.display = 'block';
}
```

Add `_hideYamlViewer()` method:
```typescript
private _hideYamlViewer(): void {
  if (this._yamlViewer) {
    this._yamlViewer.style.display = 'none';
  }
}
```

Add `_detachYaml()` method:
```typescript
private _detachYaml(): void {
  const base = this._conn?.restBase ?? this.baseUrl ?? window.location.origin;
  const scenario = this.scenario ?? this._conn?.state.scenario ?? '';
  const url = `${base}/scenario/yaml-viewer?baseUrl=${encodeURIComponent(base)}&scenario=${encodeURIComponent(scenario)}`;
  this._popoutWindow = window.open(url, 'yaml-viewer', 'width=400,height=600');
  this._hideYamlViewer();
  this._yamlOpen = false;
  if (this._popoutPoll) clearInterval(this._popoutPoll);
  this._popoutPoll = setInterval(() => {
    if (this._popoutWindow?.closed) {
      this._popoutWindow = null;
      if (this._popoutPoll) {
        clearInterval(this._popoutPoll);
        this._popoutPoll = null;
      }
    }
  }, 500);
}
```

Insert `${this._renderSourceToggle()}` into `_renderCompactCard()` header,
next to the collapse button.

Add import at top:
```typescript
import type { PagesScenarioYamlViewer } from './scenario-yaml-viewer.js';
```

Override `disconnectedCallback` to clean up:
```typescript
override disconnectedCallback(): void {
  super.disconnectedCallback();
  if (this._yamlViewer?.parentNode) {
    this._yamlViewer.parentNode.removeChild(this._yamlViewer);
    this._yamlViewer = null;
  }
  if (this._popoutPoll) {
    clearInterval(this._popoutPoll);
    this._popoutPoll = null;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --run src/controller/scenario-controller.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/pages add packages/pages-aria/src/controller/scenario-controller.ts packages/pages-aria/src/controller/scenario-controller.test.ts
git commit -m "feat(#349): controller toggle button for YAML viewer + detach support

Refs casehubio/casehub-pages#349"
```

### Task 4: Pop-out HTML page for detach mode

**Files:**
- Create: `backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/yaml-viewer.html`

**Interfaces:**
- Consumes: `controller.js` bundle (includes `PagesScenarioYamlViewer` after Task 2)

- [ ] **Step 1: Create yaml-viewer.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Scenario YAML Source</title>
  <style>
    body {
      margin: 0;
      background: #0f172a;
      overflow: hidden;
    }
    pages-scenario-yaml-viewer {
      position: static;
      display: block;
      height: 100vh;
    }
  </style>
</head>
<body>
  <pages-scenario-yaml-viewer id="viewer"></pages-scenario-yaml-viewer>
  <script>
    const params = new URLSearchParams(window.location.search);
    const viewer = document.getElementById('viewer');
    const baseUrl = params.get('baseUrl') || window.location.origin;
    const scenario = params.get('scenario') || '';
    viewer.setAttribute('baseurl', baseUrl);
    if (scenario) viewer.setAttribute('scenario', scenario);
  </script>
  <script type="module">
    import '/scenario/controller.js';
  </script>
</body>
</html>
```

- [ ] **Step 2: Build controller bundle to include yaml viewer**

Run: `yarn workspace @casehubio/pages-aria run build:controller`
Expected: Build succeeds

- [ ] **Step 3: Copy built controller.js to backend resources**

```bash
cp /Users/mdproctor/claude/casehub/slots/112/pages/packages/pages-aria/dist/controller.js /Users/mdproctor/claude/casehub/slots/112/pages/backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/controller.js
```

- [ ] **Step 4: Run all pages-aria tests**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/112/pages add backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/yaml-viewer.html backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/controller.js packages/pages-aria/dist/controller.js
git commit -m "feat(#349): YAML viewer pop-out page and updated controller bundle

Closes casehubio/casehub-pages#349"
```

---

## References

- [2026-08-23-yaml-flyout-viewer-design.md] — design spec
- `packages/pages-aria/src/controller/scenario-controller.ts` — existing controller
- `packages/pages-aria/src/controller/scenario-connection-controller.ts` — push wire pattern
- `packages/pages-aria/src/controller/standalone.ts` — bundle entry point
- `backend/scenario-runtime/.../scenario/remote.html` — existing pop-out page pattern
- D23, D24, D25 in decisions.md
- casehubio/casehub-pages#349
