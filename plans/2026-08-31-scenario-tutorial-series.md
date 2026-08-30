# Scenario Tutorial Series Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #395 — Scenario automation tutorial series — architecture intro + hands-on examples
**Issue group:** #395

**Goal:** Build tutorial infrastructure (catalog, registry, parser extension, sectioned runner) and two tutorial entries (Tutorial 0: slides-only, Tutorial 1: hands-on) that teach the scenario automation platform.

**Architecture:** Extend the pages-aria package with a discriminated-union Scenario type, a sectioned runner for browser-only tutorial execution, an HTML sanitizer for inline SVG in narratives, and a `<pages-tutorial-catalog>` component with tiled/list modes. A build-time script generates a tutorial registry JSON from tutorial YAML descriptors.

**Tech Stack:** TypeScript, Lit, Vitest, YAML, CSS custom properties, SVG

## Global Constraints

- Browser-only: tutorials must work without a server using the client-side runner
- Pre-release: breaking type changes are acceptable
- All ARIA targeting follows `aria-interaction-contract.md` protocol
- Design tokens: all styles use `--pages-*` CSS custom properties
- Test framework: Vitest
- Build: Yarn workspaces, existing `yarn build:packages` pipeline

---

## Batch 1: Foundation — Discriminated Union Types + Parser Extension

### Task 1: Refactor Scenario types to discriminated union

**Files:**
- Modify: `packages/pages-aria/src/scenario/types.ts`
- Modify: `packages/pages-aria/src/scenario/index.ts`
- Test: `packages/pages-aria/src/scenario/scenario.test.ts`

**Interfaces:**
- Produces: `ScenarioBase`, `FlatScenario`, `SectionedScenario`, `Scenario` (union), `TutorialMeta`, `TutorialSection`, `SectionContent`, `isSectioned()` type guard — all consumed by parser, runner, and controller

- [ ] **Step 1: Write failing tests for new type structure**

Add to `packages/pages-aria/src/scenario/scenario.test.ts`:

```typescript
import { isSectioned } from './types.js';
import type { FlatScenario, SectionedScenario, Scenario } from './types.js';

describe('isSectioned type guard', () => {
  it('returns false for flat scenarios', () => {
    const flat: FlatScenario = {
      scenario: 'test',
      steps: [{ delivery: 'aria', action: 'click', target: { role: 'button', name: 'Submit' } }],
    };
    expect(isSectioned(flat)).toBe(false);
  });

  it('returns true for sectioned scenarios', () => {
    const sectioned: SectionedScenario = {
      scenario: 'test',
      sections: [{ title: 'Intro', steps: [] }],
    };
    expect(isSectioned(sectioned)).toBe(true);
  });

  it('preserves meta on both variants', () => {
    const meta = { title: 'T', description: 'D', area: 'a' };
    const flat: FlatScenario = { scenario: 'test', steps: [], meta };
    const sectioned: SectionedScenario = { scenario: 'test', sections: [], meta };
    expect(flat.meta).toEqual(meta);
    expect(sectioned.meta).toEqual(meta);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-aria run test -- --run scenario.test`
Expected: FAIL — `isSectioned` not exported, `FlatScenario`/`SectionedScenario` don't exist

- [ ] **Step 3: Implement discriminated union types**

Replace the content of `packages/pages-aria/src/scenario/types.ts` with:

```typescript
import type { AriaTarget } from '@casehubio/pages-primitives';

export interface AwaitCondition {
  match: Record<string, unknown>;
  timeout?: number;
  interval?: number;
}

export type ScenarioStep =
  | { delivery: 'aria'; name?: string; action: string;
      target?: AriaTarget; value?: string;
      state?: Record<string, unknown>; timeout?: number }
  | { delivery: 'graphql'; name: string; domain: string;
      operation: string; params?: Record<string, unknown>;
      await?: AwaitCondition }
  | { delivery: 'simulated'; name?: string; dataset: string;
      data: Record<string, unknown> };

export interface TutorialMeta {
  title: string;
  description: string;
  area: string;
  labels?: string[];
  tags?: string[];
  estimated?: string;
  prerequisites?: string[];
  hero?: { title: string; subtitle?: string; icon?: string };
}

export interface SectionContent {
  type: 'inline' | 'template';
  markdown?: string;
  path?: string;
  section?: string;
}

export interface TutorialSection {
  title: string;
  content?: SectionContent;
  steps: ScenarioStep[];
}

export interface ScenarioBase {
  scenario: string;
  meta?: TutorialMeta;
}

export interface FlatScenario extends ScenarioBase {
  steps: ScenarioStep[];
}

export interface SectionedScenario extends ScenarioBase {
  sections: TutorialSection[];
}

export type Scenario = FlatScenario | SectionedScenario;

export function isSectioned(s: Scenario): s is SectionedScenario {
  return 'sections' in s;
}
```

- [ ] **Step 4: Update index.ts exports**

Update `packages/pages-aria/src/scenario/index.ts`:

```typescript
export { parseScenario } from './parser.js';
export { runScenario } from './runner.js';
export { isSectioned } from './types.js';
export type {
  Scenario, FlatScenario, SectionedScenario, ScenarioBase,
  ScenarioStep, TutorialMeta, TutorialSection, SectionContent,
} from './types.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-aria run test -- --run scenario.test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-aria/src/scenario/types.ts packages/pages-aria/src/scenario/index.ts packages/pages-aria/src/scenario/scenario.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): refactor Scenario types to discriminated union Refs #395"
```

### Task 2: Extend parser for sectioned format + fix runner

**Files:**
- Modify: `packages/pages-aria/src/scenario/parser.ts`
- Modify: `packages/pages-aria/src/scenario/runner.ts`
- Test: `packages/pages-aria/src/scenario/scenario.test.ts`

**Interfaces:**
- Consumes: `FlatScenario`, `SectionedScenario`, `Scenario`, `TutorialSection`, `SectionContent`, `isSectioned()` from Task 1
- Produces: `parseScenario()` now returns `Scenario` (union); `runScenario()` narrows to `FlatScenario`

- [ ] **Step 1: Write failing tests for sectioned parsing**

Add to `packages/pages-aria/src/scenario/scenario.test.ts`:

```typescript
describe('parseScenario — sectioned format', () => {
  it('parses sections with inline content', () => {
    const yaml = `
scenario: test-tutorial
meta:
  title: Test
  description: A test tutorial
  area: testing
sections:
  - title: Introduction
    content:
      type: inline
      markdown: "Hello world"
    steps: []
  - title: Demo
    steps:
      - click:
          role: button
          name: Submit
`;
    const result = parseScenario(yaml);
    expect(isSectioned(result)).toBe(true);
    if (!isSectioned(result)) throw new Error('Expected sectioned');
    expect(result.sections).toHaveLength(2);
    expect(result.sections[0].title).toBe('Introduction');
    expect(result.sections[0].content?.type).toBe('inline');
    expect(result.sections[0].content?.markdown).toBe('Hello world');
    expect(result.sections[0].steps).toHaveLength(0);
    expect(result.sections[1].title).toBe('Demo');
    expect(result.sections[1].steps).toHaveLength(1);
    expect(result.meta?.title).toBe('Test');
  });

  it('normalizes missing steps to empty array', () => {
    const yaml = `
scenario: slides-only
sections:
  - title: Slide 1
    content:
      type: inline
      markdown: Just a slide
`;
    const result = parseScenario(yaml);
    if (!isSectioned(result)) throw new Error('Expected sectioned');
    expect(result.sections[0].steps).toEqual([]);
  });

  it('rejects when both steps and sections present', () => {
    const yaml = `
scenario: invalid
steps:
  - click:
      role: button
      name: Test
sections:
  - title: Section 1
    steps: []
`;
    expect(() => parseScenario(yaml)).toThrow('mutually exclusive');
  });

  it('rejects when neither steps nor sections present', () => {
    const yaml = `
scenario: empty
`;
    expect(() => parseScenario(yaml)).toThrow('must have');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-aria run test -- --run scenario.test`
Expected: FAIL — parser doesn't handle sections

- [ ] **Step 3: Extend parser to handle sections**

Replace `parseScenario()` in `packages/pages-aria/src/scenario/parser.ts`:

```typescript
import { parse } from 'yaml';
import type { AriaTarget } from '@casehubio/pages-primitives';
import type {
  Scenario, FlatScenario, SectionedScenario,
  ScenarioStep, TutorialMeta, TutorialSection, SectionContent,
} from './types.js';

// ... keep expandAriaShorthand unchanged ...

function parseSteps(rawSteps: unknown[]): ScenarioStep[] {
  return rawSteps.map((raw: unknown) => {
    const step = raw as Record<string, unknown>;
    if (step.delivery) return step as ScenarioStep;
    return expandAriaShorthand(step);
  });
}

function parseSections(rawSections: unknown[]): TutorialSection[] {
  return rawSections.map((raw: unknown) => {
    const sec = raw as Record<string, unknown>;
    const title = sec.title as string;
    const content = sec.content as SectionContent | undefined;
    const rawSteps = Array.isArray(sec.steps) ? sec.steps : [];
    return { title, content, steps: parseSteps(rawSteps) };
  });
}

export function parseScenario(yamlString: string): Scenario {
  const parsed = parse(yamlString) as Record<string, unknown>;
  if (!parsed.scenario) {
    throw new Error('Invalid scenario: must have "scenario" name');
  }

  const hasSteps = Array.isArray(parsed.steps);
  const hasSections = Array.isArray(parsed.sections);

  if (hasSteps && hasSections) {
    throw new Error('Invalid scenario: "steps" and "sections" are mutually exclusive — use one or the other');
  }
  if (!hasSteps && !hasSections) {
    throw new Error('Invalid scenario: must have "steps" or "sections"');
  }

  const meta = parsed.meta as TutorialMeta | undefined;

  if (hasSections) {
    const result: SectionedScenario = {
      scenario: parsed.scenario as string,
      sections: parseSections(parsed.sections as unknown[]),
    };
    if (meta) result.meta = meta;
    return result;
  }

  const result: FlatScenario = {
    scenario: parsed.scenario as string,
    steps: parseSteps(parsed.steps as unknown[]),
  };
  if (meta) result.meta = meta;
  return result;
}
```

- [ ] **Step 4: Fix runner to narrow type**

Update `packages/pages-aria/src/scenario/runner.ts`:

```typescript
import { click, fill, select, expand, collapse, assertState, waitFor } from '../executor/index.js';
import { isSectioned } from './types.js';
import type { FlatScenario, Scenario, ScenarioStep } from './types.js';
import type { AriaState } from '@casehubio/pages-primitives';

// ... keep toAriaState and executeStep unchanged ...

export async function runScenario(scenario: Scenario): Promise<void> {
  if (isSectioned(scenario)) {
    throw new Error('runScenario does not support sectioned scenarios — use runSectionedScenario');
  }
  for (const step of scenario.steps) {
    await executeStep(step);
  }
}
```

- [ ] **Step 5: Run tests to verify all pass**

Run: `yarn workspace @casehubio/pages-aria run test -- --run scenario.test`
Expected: PASS (all existing + new tests)

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS — all imports resolve, no type errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-aria/src/scenario/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): extend parser for sectioned scenarios + validation Refs #395"
```

## Batch 2: Browser-Only Runtime — Narrative + Controller Fixes

### Task 3: HTML sanitizer + narrative htmlMode

**Files:**
- Create: `packages/pages-aria/src/controller/html-sanitizer.ts`
- Create: `packages/pages-aria/src/controller/html-sanitizer.test.ts`
- Modify: `packages/pages-aria/src/controller/scenario-narrative.ts`
- Test: `packages/pages-aria/src/controller/scenario-narrative.test.ts`

**Interfaces:**
- Produces: `sanitizeHtml(html: string): string` — used by narrative renderer when `htmlMode === 'sanitized'`

- [ ] **Step 1: Write failing tests for sanitizer**

Create `packages/pages-aria/src/controller/html-sanitizer.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { sanitizeHtml } from './html-sanitizer.js';

describe('sanitizeHtml', () => {
  it('passes through SVG elements', () => {
    const svg = '<svg viewBox="0 0 100 100"><rect fill="red" width="50" height="50"/></svg>';
    const result = sanitizeHtml(svg);
    expect(result).toContain('<svg');
    expect(result).toContain('<rect');
  });

  it('strips script tags', () => {
    const html = '<p>Hello</p><script>alert("xss")</script>';
    const result = sanitizeHtml(html);
    expect(result).not.toContain('<script');
    expect(result).toContain('Hello');
  });

  it('strips event handler attributes', () => {
    const html = '<svg onclick="alert(1)"><rect fill="red"/></svg>';
    const result = sanitizeHtml(html);
    expect(result).not.toContain('onclick');
  });

  it('allows CSS custom properties in style', () => {
    const svg = '<rect style="fill: var(--pages-neutral-3)"/>';
    const result = sanitizeHtml(`<svg>${svg}</svg>`);
    expect(result).toContain('var(--pages-neutral-3)');
  });

  it('strips style values with url()', () => {
    const svg = '<rect style="background: url(evil.png)"/>';
    const result = sanitizeHtml(`<svg>${svg}</svg>`);
    expect(result).not.toContain('url(');
  });

  it('strips iframe elements', () => {
    const html = '<iframe src="evil.com"></iframe>';
    const result = sanitizeHtml(html);
    expect(result).not.toContain('<iframe');
  });

  it('allows SVG text elements', () => {
    const svg = '<svg><text fill="var(--pages-neutral-12)" x="10" y="20">Label</text></svg>';
    const result = sanitizeHtml(svg);
    expect(result).toContain('<text');
    expect(result).toContain('Label');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-aria run test -- --run html-sanitizer`
Expected: FAIL — module doesn't exist

- [ ] **Step 3: Implement HTML sanitizer**

Create `packages/pages-aria/src/controller/html-sanitizer.ts`:

```typescript
const ALLOWED_ELEMENTS = new Set([
  'svg', 'g', 'defs', 'use', 'symbol', 'clippath', 'mask', 'pattern',
  'rect', 'circle', 'ellipse', 'line', 'polyline', 'polygon', 'path',
  'text', 'tspan',
  'lineargradient', 'radialgradient', 'stop', 'marker',
]);

const ALLOWED_ATTRS = new Set([
  'viewbox', 'xmlns', 'width', 'height', 'x', 'y', 'cx', 'cy',
  'r', 'rx', 'ry', 'x1', 'y1', 'x2', 'y2', 'd', 'points', 'transform',
  'fill', 'stroke', 'stroke-width', 'opacity', 'font-size', 'font-family',
  'text-anchor', 'dominant-baseline', 'stroke-dasharray', 'stroke-linecap',
  'style', 'id', 'class', 'aria-label',
  'offset', 'stop-color', 'stop-opacity', 'gradientunits', 'gradienttransform',
  'markerwidth', 'markerheight', 'refx', 'refy', 'orient',
]);

const DANGEROUS_STYLE = /url\s*\(|expression\s*\(|-moz-binding|javascript:/i;

function sanitizeStyle(value: string): string {
  return DANGEROUS_STYLE.test(value) ? '' : value;
}

function walkAndSanitize(node: Node): void {
  const toRemove: Node[] = [];

  for (const child of Array.from(node.childNodes)) {
    if (child.nodeType === 1) {
      const el = child as Element;
      const tag = el.tagName.toLowerCase();

      if (!ALLOWED_ELEMENTS.has(tag)) {
        toRemove.push(child);
        continue;
      }

      for (const attr of Array.from(el.attributes)) {
        const name = attr.name.toLowerCase();
        if (name.startsWith('on') || !ALLOWED_ATTRS.has(name)) {
          el.removeAttribute(attr.name);
        } else if (name === 'style') {
          const sanitized = sanitizeStyle(attr.value);
          if (sanitized) el.setAttribute('style', sanitized);
          else el.removeAttribute('style');
        }
      }

      walkAndSanitize(el);
    }
  }

  for (const n of toRemove) node.removeChild(n);
}

export function sanitizeHtml(html: string): string {
  const parser = new DOMParser();
  const doc = parser.parseFromString(`<body>${html}</body>`, 'text/html');
  walkAndSanitize(doc.body);
  return doc.body.innerHTML;
}
```

- [ ] **Step 4: Run sanitizer tests**

Run: `yarn workspace @casehubio/pages-aria run test -- --run html-sanitizer`
Expected: PASS

- [ ] **Step 5: Add htmlMode to narrative renderer**

Modify `packages/pages-aria/src/controller/scenario-narrative.ts`:

Add import at top:
```typescript
import { sanitizeHtml } from './html-sanitizer.js';
```

Add property:
```typescript
@property({ type: String }) htmlMode: 'escape' | 'sanitized' = 'escape';
```

Replace `_renderMarkdown` method:
```typescript
private _renderMarkdown(md: string): TemplateResult {
  let processed: string;

  if (this.htmlMode === 'sanitized') {
    processed = md
      .replace(/^### (.+)$/gm, '<h3>$1</h3>')
      .replace(/^## (.+)$/gm, '<h2>$1</h2>')
      .replace(/^# (.+)$/gm, '<h1>$1</h1>')
      .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
      .replace(/\*(.+?)\*/g, '<em>$1</em>')
      .replace(/`(.+?)`/g, '<code>$1</code>')
      .replace(/^- (.+)$/gm, '<li>$1</li>')
      .replace(/\n\n/g, '</p><p>')
      .replace(/^(?!<[hulo]|<svg)(.+)$/gm, '<p>$1</p>');
    processed = sanitizeHtml(processed);
  } else {
    const escaped = md
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
    processed = escaped
      .replace(/^### (.+)$/gm, '<h3>$1</h3>')
      .replace(/^## (.+)$/gm, '<h2>$1</h2>')
      .replace(/^# (.+)$/gm, '<h1>$1</h1>')
      .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
      .replace(/\*(.+?)\*/g, '<em>$1</em>')
      .replace(/`(.+?)`/g, '<code>$1</code>')
      .replace(/^- (.+)$/gm, '<li>$1</li>')
      .replace(/\n\n/g, '</p><p>')
      .replace(/^(?!<[hulo])(.+)$/gm, '<p>$1</p>');
  }

  const container = document.createElement('div');
  container.className = 'narrative-content';
  container.innerHTML = processed;
  return html`${container}`;
}
```

- [ ] **Step 6: Add contentBase property to narrative**

Add to `PagesScenarioNarrative`:

```typescript
@property() contentBase?: string;
```

Update `_fetchTemplate` to use `contentBase` when available:

```typescript
private async _fetchTemplate(path: string, section?: string): Promise<void> {
  try {
    const url = this.contentBase
      ? `${this.contentBase}/${path}`
      : `${this._conn.restBase}/scenario/content?path=${encodeURIComponent(path)}`;
    const resp = await fetch(url);
    if (resp.ok) {
      const text = await resp.text();
      this._templateCache.set(path, text);
      this._templateContent = this._extractSection(text, section);
    }
  } catch {
    // Ignore — show loading state
  }
}
```

- [ ] **Step 7: Run all controller tests**

Run: `yarn workspace @casehubio/pages-aria run test -- --run`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-aria/src/controller/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): HTML sanitizer + narrative htmlMode + contentBase Refs #395"
```

### Task 4: ScenarioConnectionController + Controller browser-only fixes

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-connection-controller.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Test: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: `ScenarioState`, `OutlineNode` from connection controller
- Produces: Browser-only mode support — eventTarget-only configuration, dual-mode `sendCommand`, outline via state payload, slides-only highlighting

- [ ] **Step 1: Write failing tests for browser-only mode**

Add to `packages/pages-aria/src/controller/scenario-controller.test.ts`:

```typescript
describe('browser-only mode', () => {
  it('renders without error when only eventTarget is provided', async () => {
    const el = document.createElement('pages-scenario-controller') as PagesScenarioController;
    el.eventTarget = new EventTarget();
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot?.querySelector('.error')).toBeNull();
    document.body.removeChild(el);
  });

  it('populates outline from state event', async () => {
    const target = new EventTarget();
    const el = document.createElement('pages-scenario-controller') as PagesScenarioController;
    el.eventTarget = target;
    document.body.appendChild(el);
    await el.updateComplete;

    target.dispatchEvent(new CustomEvent('pages-event', {
      detail: {
        topic: 'scenario:state',
        payload: {
          scenario: 'test',
          chapter: 'Test Tutorial',
          section: 'Intro',
          step: null,
          paused: true,
          speed: 1.0,
          progress: 0,
          content: { type: 'inline', markdown: 'Hello' },
          slides: null,
          outline: [
            { label: 'Intro', target: null, children: [] },
            { label: 'Demo', target: null, children: [
              { label: 'click-button-Submit', target: null, action: 'click', children: [] },
            ]},
          ],
        },
      },
    }));
    await el.updateComplete;

    const outlineSteps = el.shadowRoot?.querySelectorAll('.outline-step, .outline-heading');
    expect(outlineSteps?.length).toBeGreaterThan(0);
    document.body.removeChild(el);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-aria run test -- --run scenario-controller`
Expected: FAIL — eventTarget-only renders error div

- [ ] **Step 3: Fix ScenarioConnectionController.hostConnected()**

In `packages/pages-aria/src/controller/scenario-connection-controller.ts`, replace `hostConnected()`:

```typescript
hostConnected(): void {
  this._ensureConnection();
  const conn = this._getConnection();
  const target = this._getEventTarget();
  if (target) {
    target.addEventListener('pages-event', this._eventHandler);
  }
  if (conn && target) {
    void conn.listen(['scenario:state']);
    this.connectionStatus = conn.status ?? 'disconnected';
    void this._fetchInitialState();
  } else if (target && !conn) {
    this.connectionStatus = 'connected';
  }
}
```

- [ ] **Step 4: Fix sendCommand() dual-mode**

Replace `sendCommand()`:

```typescript
async sendCommand(path: string, body?: object): Promise<void> {
  const target = this._getEventTarget();
  if (!this._opts.baseUrl && !this._opts.connection && target) {
    const command = path.replace(/^\//, '');
    target.dispatchEvent(new CustomEvent('scenario-command', {
      detail: { command, ...body },
    }));
    return;
  }
  await fetch(`${this.restBase}/scenario${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    ...(body ? { body: JSON.stringify(body) } : {}),
  });
}
```

- [ ] **Step 5: Add outline and error fields to ScenarioState**

Update the `ScenarioState` interface:

```typescript
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
  outline?: OutlineNode[];
  error?: { step: string; message: string } | null;
}
```

- [ ] **Step 6: Fix PagesScenarioController.render() guard**

In `packages/pages-aria/src/controller/scenario-controller.ts`, change line 276:

```typescript
// Before:
if (!this.connection && !this.baseUrl) {
// After:
if (!this.connection && !this.baseUrl && !this.eventTarget) {
```

- [ ] **Step 7: Fix _onStateChange for outline from state**

Replace `_onStateChange()`:

```typescript
private _onStateChange(s: ScenarioState): void {
  if (s.outline) {
    this._outline = s.outline;
  } else if (s.scenario && this._outline.length === 0) {
    void this._fetchOutline();
  }
  if (!s.scenario) this._outline = [];
  this.updateComplete.then(() => this._scrollToCurrent());
}
```

- [ ] **Step 8: Fix highlighting for slides-only sections**

Update `_renderNode()` — when checking `isCurrent`, also match section-level nodes:

```typescript
private _renderNode(node: OutlineNode, depth: number): TemplateResult {
  const isLeaf = node.children.length === 0;
  const state = this._conn?.state;
  const isCurrent = isLeaf
    ? node.label === state?.step
    : (!state?.step && node.label === state?.section);
  const isCompleted = isLeaf
    ? this._isBeforeCurrent(node.label)
    : this._isSectionBeforeCurrent(node.label);
  // ... rest unchanged ...
}
```

Add section-level before-current helper:

```typescript
private _isSectionBeforeCurrent(sectionLabel: string): boolean {
  const state = this._conn?.state;
  if (!state?.section) return false;
  const sectionLabels = this._outline.map(n => n.label);
  const currentIdx = sectionLabels.indexOf(state.section);
  const labelIdx = sectionLabels.indexOf(sectionLabel);
  return labelIdx >= 0 && currentIdx >= 0 && labelIdx < currentIdx;
}
```

Update `_isBeforeCurrent()` to handle null step:

```typescript
private _isBeforeCurrent(label: string): boolean {
  const state = this._conn?.state;
  if (!state?.step && state?.section) {
    return this._isSectionBeforeCurrent(label);
  }
  const labels = this._flattenLabels(this._outline);
  const currentIdx = labels.indexOf(state?.step ?? '');
  const labelIdx = labels.indexOf(label);
  return labelIdx >= 0 && currentIdx >= 0 && labelIdx < currentIdx;
}
```

- [ ] **Step 9: Run all tests**

Run: `yarn workspace @casehubio/pages-aria run test -- --run`
Expected: PASS

- [ ] **Step 10: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-aria/src/controller/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): browser-only controller — eventTarget-only, dual-mode sendCommand, outline via state Refs #395"
```

## Batch 3: Sectioned Scenario Runner

### Task 5: Implement runSectionedScenario

**Files:**
- Create: `packages/pages-aria/src/scenario/sectioned-runner.ts`
- Create: `packages/pages-aria/src/scenario/sectioned-runner.test.ts`
- Modify: `packages/pages-aria/src/scenario/index.ts`

**Interfaces:**
- Consumes: `SectionedScenario`, `TutorialSection`, `ScenarioStep`, `executeStep()` from runner, `OutlineNode` from controller types
- Produces: `runSectionedScenario(scenario, options): TutorialRunner` — the browser-only tutorial execution engine

- [ ] **Step 1: Write failing tests for sectioned runner**

Create `packages/pages-aria/src/scenario/sectioned-runner.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import type { SectionedScenario } from './types.js';
import { runSectionedScenario, type TutorialRunnerOptions } from './sectioned-runner.js';

function makeScenario(sections: Array<{ title: string; hasSteps: boolean }>): SectionedScenario {
  return {
    scenario: 'test-tutorial',
    meta: { title: 'Test', description: 'Test', area: 'test' },
    sections: sections.map(s => ({
      title: s.title,
      content: { type: 'inline' as const, markdown: `Content for ${s.title}` },
      steps: s.hasSteps
        ? [{ delivery: 'aria' as const, action: 'click', name: 'test-step',
             target: { role: 'button', name: 'Test' } }]
        : [],
    })),
  };
}

describe('runSectionedScenario', () => {
  let eventTarget: EventTarget;
  let stateEvents: unknown[];

  beforeEach(() => {
    eventTarget = new EventTarget();
    stateEvents = [];
    eventTarget.addEventListener('pages-event', (e: Event) => {
      const detail = (e as CustomEvent).detail;
      if (detail?.topic === 'scenario:state') stateEvents.push(detail.payload);
    });
  });

  it('fires initial state event with outline', () => {
    const scenario = makeScenario([{ title: 'Intro', hasSteps: false }]);
    runSectionedScenario(scenario, { eventTarget, startPaused: true });
    expect(stateEvents.length).toBeGreaterThanOrEqual(1);
    const first = stateEvents[0] as Record<string, unknown>;
    expect(first.scenario).toBe('test-tutorial');
    expect(first.outline).toBeDefined();
    expect(first.paused).toBe(true);
  });

  it('starts paused by default', () => {
    const scenario = makeScenario([{ title: 'Intro', hasSteps: false }]);
    const runner = runSectionedScenario(scenario, { eventTarget });
    const first = stateEvents[0] as Record<string, unknown>;
    expect(first.paused).toBe(true);
    runner.dispose();
  });

  it('calls onComplete when all sections finish', async () => {
    const onComplete = vi.fn();
    const scenario = makeScenario([{ title: 'Intro', hasSteps: false }]);
    const runner = runSectionedScenario(scenario, { eventTarget, startPaused: false, onComplete });
    runner.step();
    await new Promise(r => setTimeout(r, 100));
    expect(onComplete).toHaveBeenCalledWith('test-tutorial');
    runner.dispose();
  });

  it('dispose fires null scenario state', () => {
    const scenario = makeScenario([{ title: 'Intro', hasSteps: false }]);
    const runner = runSectionedScenario(scenario, { eventTarget });
    runner.dispose();
    const last = stateEvents[stateEvents.length - 1] as Record<string, unknown>;
    expect(last.scenario).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-aria run test -- --run sectioned-runner`
Expected: FAIL — module doesn't exist

- [ ] **Step 3: Implement sectioned runner**

Create `packages/pages-aria/src/scenario/sectioned-runner.ts` implementing `runSectionedScenario()` as specified in spec §6.2. The runner:
- Builds outline from sections
- Pre-resolves template content via fetch
- Iterates sections, firing `pages-event` with `scenario:state` topic
- Handles pause/resume via Promise suspension
- Listens for `scenario-command` events for transport control
- Implements `play()`, `pause()`, `step()`, `runTo()`, `setSpeed()`, `dispose()`
- Calls `onComplete` when final section finishes

Full implementation follows the spec's runner behavior section (§6.2) and the `ScenarioState` mapping table. The pause/resume mechanism mirrors `scenario-handler.ts` (Promise-based suspension with `queueMicrotask` for step mode).

- [ ] **Step 4: Update index.ts exports**

Add to `packages/pages-aria/src/scenario/index.ts`:

```typescript
export { runSectionedScenario } from './sectioned-runner.js';
export type { TutorialRunner, TutorialRunnerOptions } from './sectioned-runner.js';
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-aria run test -- --run sectioned-runner`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria run test -- --run`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-aria/src/scenario/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): sectioned scenario runner — browser-only tutorial execution Refs #395"
```

## Batch 4: Tutorial Catalog Component + Registry

### Task 6: TutorialDescriptor types + catalog component

**Files:**
- Create: `packages/pages-aria/src/tutorial/types.ts`
- Create: `packages/pages-aria/src/tutorial/tutorial-catalog.ts`
- Create: `packages/pages-aria/src/tutorial/tutorial-catalog.test.ts`
- Create: `packages/pages-aria/src/tutorial/index.ts`

**Interfaces:**
- Consumes: `TutorialMeta` from scenario types (shared label format)
- Produces: `TutorialDescriptor`, `LearningPath` types, `<pages-tutorial-catalog>` component with `tutorial-select`, `area-select`, `path-select` events

- [ ] **Step 1: Write failing tests for catalog component**

Create `packages/pages-aria/src/tutorial/tutorial-catalog.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import type { TutorialDescriptor, LearningPath } from './types.js';
import './tutorial-catalog.js';

const DESCRIPTORS: TutorialDescriptor[] = [
  {
    scenario: 'arch-concepts', title: 'Architecture', description: 'Overview',
    area: 'scenario-automation', labels: ['difficulty:beginner'], tags: ['overview'],
    estimated: '15 min', prerequisites: [], path: 'tutorials/arch/tutorial.yaml',
    contentType: 'slides-only',
    hero: { title: 'Architecture', subtitle: 'Learn the basics', icon: '◎' },
  },
  {
    scenario: 'form-auto', title: 'Form Automation', description: 'Hands-on forms',
    area: 'scenario-automation', labels: ['difficulty:beginner', 'concept:aria'], tags: ['forms'],
    estimated: '10 min', prerequisites: ['arch-concepts'], path: 'tutorials/form/tutorial.yaml',
    contentType: 'hands-on',
    hero: { title: 'Form Automation', subtitle: 'Fill forms', icon: '✎' },
  },
];

describe('pages-tutorial-catalog', () => {
  let el: HTMLElement;

  beforeEach(async () => {
    el = document.createElement('pages-tutorial-catalog');
    (el as any).registry = DESCRIPTORS;
    document.body.appendChild(el);
    await (el as any).updateComplete;
  });

  afterEach(() => {
    document.body.removeChild(el);
  });

  it('renders area cards in tiles mode', () => {
    const cards = el.shadowRoot?.querySelectorAll('.area-card');
    expect(cards?.length).toBe(1); // one area: scenario-automation
  });

  it('fires area-select on area card click', async () => {
    const promise = new Promise<string>(resolve => {
      el.addEventListener('area-select', (e: Event) => {
        resolve((e as CustomEvent).detail.area);
      });
    });
    const card = el.shadowRoot?.querySelector('.area-card') as HTMLElement;
    card?.click();
    const area = await promise;
    expect(area).toBe('scenario-automation');
  });

  it('switches to list mode', async () => {
    (el as any).mode = 'list';
    await (el as any).updateComplete;
    const rows = el.shadowRoot?.querySelectorAll('.tutorial-row');
    expect(rows?.length).toBe(2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-aria run test -- --run tutorial-catalog`
Expected: FAIL — module doesn't exist

- [ ] **Step 3: Create tutorial types**

Create `packages/pages-aria/src/tutorial/types.ts`:

```typescript
export interface TutorialDescriptor {
  scenario: string;
  title: string;
  description: string;
  area: string;
  labels: string[];
  tags: string[];
  estimated?: string;
  prerequisites: string[];
  path: string;
  contentType: 'slides-only' | 'hands-on';
  hero?: { title: string; subtitle?: string; icon?: string };
}

export interface LearningPath {
  path: string;
  title: string;
  description: string;
  labels: string[];
  tutorials: string[];
}
```

- [ ] **Step 4: Implement catalog component**

Create `packages/pages-aria/src/tutorial/tutorial-catalog.ts` — a Lit web component implementing the spec's §4 catalog with:
- `registry`, `paths`, `mode`, `area`, `labels`, `activeTutorial` properties
- Tiled landing view (area cards at root, tutorial cards when area drilled)
- Compact list view (flat filterable rows)
- Mode toggle with localStorage persistence
- `tutorial-select`, `area-select`, `path-select` events
- Design token styling throughout

- [ ] **Step 5: Create module index**

Create `packages/pages-aria/src/tutorial/index.ts`:

```typescript
export { PagesTutorialCatalog } from './tutorial-catalog.js';
export type { TutorialDescriptor, LearningPath } from './types.js';
```

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/pages-aria run test -- --run tutorial-catalog`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-aria/src/tutorial/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): tutorial catalog component — tiled landing + compact list Refs #395"
```

### Task 7: Registry build script

**Files:**
- Create: `scripts/build-tutorial-registry.ts`
- Modify: `package.json` (add `build:tutorials` script)

**Interfaces:**
- Consumes: Tutorial YAML files from `tutorials/` directory
- Produces: `dist/tutorial-registry.json`

- [ ] **Step 1: Create build script**

Create `scripts/build-tutorial-registry.ts`:

```typescript
import { readFileSync, writeFileSync, readdirSync, existsSync, statSync } from 'fs';
import { join, dirname } from 'path';
import { parse } from 'yaml';

interface TutorialMeta {
  title: string;
  description: string;
  area: string;
  labels?: string[];
  tags?: string[];
  estimated?: string;
  prerequisites?: string[];
  hero?: { title: string; subtitle?: string; icon?: string };
}

interface TutorialDescriptor extends TutorialMeta {
  scenario: string;
  path: string;
  contentType: 'slides-only' | 'hands-on';
  labels: string[];
  tags: string[];
  prerequisites: string[];
}

const tutorialsDir = join(process.cwd(), 'tutorials');
const outputPath = join(process.cwd(), 'dist', 'tutorial-registry.json');

function scanTutorials(): TutorialDescriptor[] {
  if (!existsSync(tutorialsDir)) return [];
  const entries = readdirSync(tutorialsDir).filter(e => {
    const stat = statSync(join(tutorialsDir, e));
    return stat.isDirectory() && e !== 'paths';
  });

  const descriptors: TutorialDescriptor[] = [];
  const errors: string[] = [];

  for (const dir of entries) {
    const yamlPath = join(tutorialsDir, dir, 'tutorial.yaml');
    if (!existsSync(yamlPath)) continue;

    const content = readFileSync(yamlPath, 'utf-8');
    const parsed = parse(content) as Record<string, unknown>;

    if (!parsed.scenario) {
      errors.push(`${dir}: missing scenario name`);
      continue;
    }
    if (!parsed.meta) {
      errors.push(`${dir}: missing meta block`);
      continue;
    }

    const meta = parsed.meta as TutorialMeta;
    if (!meta.title || !meta.description || !meta.area) {
      errors.push(`${dir}: meta must have title, description, area`);
      continue;
    }

    // Validate labels format
    for (const label of meta.labels ?? []) {
      if (!label.includes(':')) {
        errors.push(`${dir}: label "${label}" must be namespace:value`);
      }
    }

    // Validate template file existence
    const sections = parsed.sections as Array<Record<string, unknown>> | undefined;
    if (sections) {
      for (const sec of sections) {
        const content = sec.content as { type?: string; path?: string } | undefined;
        if (content?.type === 'template' && content.path) {
          const templatePath = join(tutorialsDir, dir, content.path);
          if (!existsSync(templatePath)) {
            errors.push(`${dir}: section "${sec.title}" references missing file: ${content.path}`);
          }
        }
      }
    }

    // Derive contentType
    let contentType: 'slides-only' | 'hands-on' = 'slides-only';
    if (sections) {
      for (const sec of sections) {
        const steps = sec.steps as unknown[] | undefined;
        if (steps && steps.length > 0) { contentType = 'hands-on'; break; }
      }
    }

    descriptors.push({
      scenario: parsed.scenario as string,
      title: meta.title,
      description: meta.description,
      area: meta.area,
      labels: meta.labels ?? [],
      tags: meta.tags ?? [],
      estimated: meta.estimated,
      prerequisites: meta.prerequisites ?? [],
      path: `tutorials/${dir}/tutorial.yaml`,
      contentType,
      hero: meta.hero,
    });
  }

  if (errors.length > 0) {
    console.error('Tutorial registry build errors:');
    for (const err of errors) console.error(`  - ${err}`);
    process.exit(1);
  }

  // Validate scenario name uniqueness
  const seen = new Map<string, string>();
  for (const d of descriptors) {
    if (seen.has(d.scenario)) {
      console.error(`Duplicate scenario name "${d.scenario}" in ${d.path} and ${seen.get(d.scenario)}`);
      process.exit(1);
    }
    seen.set(d.scenario, d.path);
  }

  // Validate prerequisites (warnings for standalone build)
  for (const d of descriptors) {
    for (const prereq of d.prerequisites) {
      if (!seen.has(prereq)) {
        console.warn(`Warning: ${d.scenario} has prerequisite "${prereq}" not found in this registry`);
      }
    }
  }

  return descriptors;
}

const descriptors = scanTutorials();
writeFileSync(outputPath, JSON.stringify(descriptors, null, 2));
console.log(`Tutorial registry: ${descriptors.length} tutorial(s) → ${outputPath}`);
```

- [ ] **Step 2: Add build script to package.json**

Add to root `package.json` scripts:

```json
"build:tutorials": "ts-node scripts/build-tutorial-registry.ts"
```

- [ ] **Step 3: Test the build script**

Run: `yarn build:tutorials`
Expected: outputs `dist/tutorial-registry.json` (empty array if no tutorials yet, or error if tutorials dir doesn't exist — both acceptable at this stage)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add scripts/build-tutorial-registry.ts package.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): tutorial registry build script Refs #395"
```

## Batch 5: Tutorial Content

### Task 8: Tutorial 0 — Architecture & Concepts

**Files:**
- Create: `tutorials/architecture-concepts/tutorial.yaml`
- Create: `tutorials/architecture-concepts/content/intro.md`
- Create: `tutorials/architecture-concepts/content/executor-architecture.md`
- Create: `tutorials/architecture-concepts/content/push-wire.md`
- Create: `tutorials/architecture-concepts/content/aria-targeting.md`
- Create: `tutorials/architecture-concepts/content/compilation-pipeline.md`
- Create: `tutorials/architecture-concepts/content/script-library.md`
- Create: `tutorials/architecture-concepts/content/readiness-probes.md`

**Interfaces:**
- Consumes: Spec §9 (section list, meta block)
- Produces: A slides-only tutorial YAML with 7 sections, 4 inline SVG diagrams

- [ ] **Step 1: Create tutorial.yaml**

Create `tutorials/architecture-concepts/tutorial.yaml` with the meta block from spec §9.2 and 7 sections referencing content files. Each section has `steps: []` (slides-only).

- [ ] **Step 2: Write narrative content files**

Create each `.md` file in `tutorials/architecture-concepts/content/` with:
- Clear explanatory prose about each concept
- Inline SVG diagrams (4 total) using `var(--pages-*)` CSS custom properties for theming:
  - `executor-architecture.md` — orchestrator ↔ executor star topology
  - `aria-targeting.md` — DOM tree with role/name/index matching
  - `compilation-pipeline.md` — YAML → params → forEach → when → call → execution plan
  - `script-library.md` — 3 source types aggregating into registry

- [ ] **Step 3: Validate with build script**

Run: `yarn build:tutorials`
Expected: PASS — `dist/tutorial-registry.json` contains 1 entry with `contentType: "slides-only"`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add tutorials/architecture-concepts/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): Tutorial 0 — Architecture & Concepts (slides-only) Refs #395"
```

### Task 9: Tutorial 1 — Form Automation Basics + learning path

**Files:**
- Create: `tutorials/form-automation/tutorial.yaml`
- Create: `tutorials/form-automation/content/intro.md`
- Create: `tutorials/form-automation/content/yaml-structure.md`
- Create: `tutorials/form-automation/content/fill-command.md`
- Create: `tutorials/form-automation/content/select-command.md`
- Create: `tutorials/form-automation/content/click-command.md`
- Create: `tutorials/form-automation/content/controller-ui.md`
- Create: `tutorials/form-automation/content/recap.md`
- Create: `tutorials/paths/fundamentals.yaml`

**Interfaces:**
- Consumes: Spec §10 (sections, meta, target app), existing Form Automation example pattern
- Produces: A hands-on tutorial YAML with 7 sections (narrative + executable steps), a learning path manifest

- [ ] **Step 1: Create tutorial.yaml**

Create `tutorials/form-automation/tutorial.yaml` with the meta block from spec §10.3 and 7 sections. Sections 3-5 have executable ARIA steps (`fill`, `select`, `click`) targeting a form (same ARIA contract as the existing Form Automation example).

- [ ] **Step 2: Write narrative content files**

Create each `.md` file explaining the concept being demonstrated, keyed to the executable step in that section.

- [ ] **Step 3: Create learning path manifest**

Create `tutorials/paths/fundamentals.yaml`:

```yaml
path: scenario-fundamentals
title: "Scenario Engine Fundamentals"
description: "Complete introduction to the automation platform"
labels:
  - difficulty:beginner
tutorials:
  - architecture-concepts
  - form-automation-tutorial
```

- [ ] **Step 4: Validate with build script**

Run: `yarn build:tutorials`
Expected: PASS — `dist/tutorial-registry.json` contains 2 entries (1 slides-only, 1 hands-on)

- [ ] **Step 5: Run full build**

Run: `yarn build`
Expected: PASS — no type errors, no lint errors

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add tutorials/ dist/tutorial-registry.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#395): Tutorial 1 — Form Automation Basics + fundamentals learning path Refs #395"
```

## References

- [2026-08-30-scenario-tutorial-series-design.md] — design spec this plan implements
- [packages/pages-aria/src/scenario/types.ts] — current Scenario type
- [packages/pages-aria/src/scenario/parser.ts] — current parser
- [packages/pages-aria/src/scenario/runner.ts] — current runner
- [packages/pages-aria/src/controller/scenario-connection-controller.ts] — connection controller (lines 62-71: hostConnected, line 86-92: sendCommand)
- [packages/pages-aria/src/controller/scenario-controller.ts] — controller (line 231: _onStateChange, line 276: render guard, line 404: _renderNode highlighting)
- [packages/pages-aria/src/controller/scenario-narrative.ts] — narrative renderer (line 179: _renderMarkdown)
- [packages/pages-aria/src/controller/library-view.ts] — library view (card styling pattern)
- [docs/protocols/casehub/aria-interaction-contract.md] — ARIA targeting protocol
- [GitHub #395] — focal issue
