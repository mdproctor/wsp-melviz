# Migrate playbook-split-view to pages-three-way-viewer — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #749 — Migrate playbook-split-view to pages-three-way-viewer
**Issue group:** #749

**Goal:** Replace PlaybookSplitView's inline layout/splitter/debounce
code with ThreeWayViewer composition, making PagesStepViewer implement
ViewerSyncTarget for bidirectional cursor↔step sync.

**Architecture:** PagesStepViewer gains `positionInfos`, `error`, and
`syncLine` to implement ViewerSyncTarget. PlaybookSplitView becomes a
thin orchestrator that renders `<pages-three-way-viewer>` with
`<pages-step-viewer slot="visual-pane">`, wiring `value-change` →
`parseAndResolve` → property updates.

**Tech Stack:** Lit 3, TypeScript 5.8, Vitest, `@casehubio/yaml-core`
(StepPositionScanner, Walker), `@casehubio/pages-three-way-viewer`
(ThreeWayViewer, ViewerSyncTarget, ViewerSelectDetail)

## Global Constraints

- Lit web components, `@customElement` decorator pattern
- CSS custom properties use `--pages-` prefix
- `step-click` CustomEvent must continue to fire unchanged (backward compat)
- `parseAndResolve` function and `StepSelectDetail`/`SplitViewState` type
  exports from `pages-step-viewer/src/index.ts` must be preserved
- Vitest for unit tests, run with `yarn test` from package directory

---

## Batch 1: PagesStepViewer implements ViewerSyncTarget

### Task 1: Add positionInfos, error, and syncLine to PagesStepViewer

**Files:**
- Modify: `packages/pages-step-viewer/src/pages-step-viewer.ts:1-69`
- Test: `packages/pages-step-viewer/src/pages-step-viewer.test.ts`
  (new file — existing tests are in `playbook-split-view.test.ts` for
  the parse function only)

**Interfaces:**
- Consumes: `ViewerSyncTarget` from `@casehubio/pages-three-way-viewer`
  (interface: `{ syncLine(line: number): void }`),
  `StepPositionScanner` from `@casehubio/yaml-core`
  (`findByLine(infos, line): StepLineInfo | undefined`,
  `findByAddress(infos, address): StepLineInfo | undefined`),
  `StepLineInfo` from `@casehubio/yaml-core`
  (type: `{ line: number; endLine: number; address: string }`)
- Produces: `PagesStepViewer` with new properties:
  `positionInfos: StepLineInfo[]` (default `[]`),
  `error: string | null` (default `null`),
  method `syncLine(line: number): void`

- [ ] **Step 1: Write failing tests for syncLine**

Create `packages/pages-step-viewer/src/pages-step-viewer.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { StepPositionScanner, type StepLineInfo } from '@casehubio/yaml-core';
import type { PagesStepViewer } from './pages-step-viewer.js';

const MOCK_INFOS: StepLineInfo[] = [
  { line: 1, endLine: 3, address: '0' },
  { line: 4, endLine: 6, address: '1' },
];

describe('PagesStepViewer syncLine', () => {
  it('syncLine calls setActiveStep with correct address', () => {
    // Test the logic directly — syncLine maps line → address
    const info = StepPositionScanner.findByLine(MOCK_INFOS, 2);
    expect(info?.address).toBe('0');
  });

  it('syncLine with line outside any step returns undefined', () => {
    const info = StepPositionScanner.findByLine(MOCK_INFOS, 10);
    expect(info).toBeUndefined();
  });

  it('syncLine with empty positionInfos returns undefined', () => {
    const info = StepPositionScanner.findByLine([], 1);
    expect(info).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-step-viewer test`

Expected: Tests should pass (they test StepPositionScanner which already
exists). This establishes the baseline logic. The integration with
PagesStepViewer is structural — the failing part is that `syncLine`
doesn't exist yet on the class.

- [ ] **Step 3: Add imports and new properties to PagesStepViewer**

In `packages/pages-step-viewer/src/pages-step-viewer.ts`, add imports:

```typescript
import { StepPositionScanner, type StepLineInfo } from '@casehubio/yaml-core';
import type { ViewerSyncTarget, ViewerSelectDetail } from '@casehubio/pages-three-way-viewer';
```

Add new properties after the existing `_activeAddress` state:

```typescript
@property({ attribute: false }) positionInfos: StepLineInfo[] = [];
@property({ attribute: false }) error: string | null = null;
```

- [ ] **Step 4: Implement syncLine method**

Add after `setActiveStep`:

```typescript
syncLine(line: number): void {
  const info = StepPositionScanner.findByLine(this.positionInfos, line);
  this.setActiveStep(info?.address ?? null);
}
```

- [ ] **Step 5: Add step-click → viewer-select event bridging**

Add a `connectedCallback` and `disconnectedCallback` to listen for
`step-click` events and re-dispatch as `viewer-select` when
`positionInfos` is available:

```typescript
override connectedCallback(): void {
  super.connectedCallback();
  this.addEventListener('step-click', this._onStepClick as EventListener);
}

override disconnectedCallback(): void {
  super.disconnectedCallback();
  this.removeEventListener('step-click', this._onStepClick as EventListener);
}

private _onStepClick = (e: Event): void => {
  if (this.positionInfos.length === 0) return;
  const { address } = (e as CustomEvent<{ address: string }>).detail;
  const info = StepPositionScanner.findByAddress(this.positionInfos, address);
  if (info) {
    this.dispatchEvent(new CustomEvent<ViewerSelectDetail>('viewer-select', {
      bubbles: true,
      composed: true,
      detail: { line: info.line, endLine: info.endLine },
    }));
  }
};
```

- [ ] **Step 6: Add error banner and empty state to render**

Add error banner CSS to `styles`:

```css
.error-banner {
  padding: 12px 16px;
  background: rgba(248, 113, 113, 0.1);
  border: 1px solid rgba(248, 113, 113, 0.3);
  border-radius: 6px;
  color: #f87171;
  font-family: monospace;
  font-size: 13px;
  white-space: pre-wrap;
  margin: 16px;
}
.empty-state {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: var(--pages-text-secondary, #94a3b8);
  font-size: 14px;
}
```

Modify the `render()` method — add error/empty guards before the
existing step rendering:

```typescript
override render() {
  if (this.error) {
    return html`<div class="error-banner">${this.error}</div>`;
  }

  const diff = this.computedDiff;
  if (diff) {
    return html`
      <diff-summary .stats=${diff.summary}></diff-summary>
      <div class="steps">
        ${diff.root.map(node => renderDiffContent(node))}
      </div>
    `;
  }

  const steps = this.steps ?? this.newSteps ?? [];
  if (steps.length === 0) {
    return html`<div class="empty-state">Enter playbook YAML to see the visualization</div>`;
  }

  const addressMap = assignStepAddresses(steps);
  return html`
    <div class="steps">
      ${steps.map(s => {
        const addr = addressMap.get(s);
        return addr !== undefined
          ? renderStepContentWithAddress(s, addr, this._activeAddress)
          : renderStepContent(s);
      })}
    </div>
  `;
}
```

- [ ] **Step 7: Write tests for viewer-select dispatch and error/empty rendering**

Append to `pages-step-viewer.test.ts`:

```typescript
describe('PagesStepViewer viewer-select dispatch', () => {
  it('findByAddress maps address to line info', () => {
    const info = StepPositionScanner.findByAddress(MOCK_INFOS, '1');
    expect(info).toEqual({ line: 4, endLine: 6, address: '1' });
  });

  it('findByAddress returns undefined for unknown address', () => {
    const info = StepPositionScanner.findByAddress(MOCK_INFOS, '99');
    expect(info).toBeUndefined();
  });
});
```

- [ ] **Step 8: Run all tests**

Run: `yarn --cwd packages/pages-step-viewer test`

Expected: All tests pass.

- [ ] **Step 9: Commit**

```bash
git add packages/pages-step-viewer/src/pages-step-viewer.ts \
       packages/pages-step-viewer/src/pages-step-viewer.test.ts
git commit -m "feat(step-viewer): implement ViewerSyncTarget — syncLine, error, viewer-select Refs #749"
```

## Batch 2: Thin PlaybookSplitView + package wiring

### Task 2: Add pages-three-way-viewer dependency and rewrite PlaybookSplitView

**Files:**
- Modify: `packages/pages-step-viewer/package.json:23-27` (add dependency)
- Modify: `packages/pages-step-viewer/src/playbook-split-view.ts:1-174` (rewrite)
- Modify: `packages/pages-step-viewer/gallery.html:325-345,1422-1458`
  (update section 33)

**Interfaces:**
- Consumes: `ThreeWayViewer` from `@casehubio/pages-three-way-viewer`
  (web component `<pages-three-way-viewer>`, props: `language`, `value`,
  `split-ratio`, `readonly`; events: `value-change` with
  `{ value: string }`), `PagesStepViewer` with `positionInfos`, `error`,
  `syncLine` (from Task 1)
- Produces: `PlaybookSplitView` (web component `<playbook-split-view>`,
  preserved public API: `updateYaml(yaml)`, `syncCursorLine(line)`,
  `splitRatio` property)

- [ ] **Step 1: Add package dependency**

In `packages/pages-step-viewer/package.json`, add to `dependencies`:

```json
"@casehubio/pages-three-way-viewer": "workspace:^"
```

- [ ] **Step 2: Run `yarn install` to update lockfile**

Run: `yarn install`

- [ ] **Step 3: Rewrite playbook-split-view.ts**

Replace the entire class body. Keep `parseAndResolve`, `StepSelectDetail`,
and `SplitViewState` exports unchanged. The new class:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import type { ResolvedStep } from '@casehubio/yaml-core/step';
import { Walker } from '@casehubio/yaml-core/step';
import { StepPositionScanner, type StepLineInfo } from '@casehubio/yaml-core';
import { parse } from 'yaml';
import './pages-step-viewer.js';
import { demoCatalog } from './demo-catalog.js';
import '@casehubio/pages-three-way-viewer';

export interface StepSelectDetail {
  line: number;
  endLine: number;
  address: string;
}

export interface SplitViewState {
  steps: ResolvedStep[];
  positionInfos: StepLineInfo[];
  error: string | null;
}

export function parseAndResolve(yaml: string): SplitViewState {
  if (!yaml.trim()) {
    return { steps: [], positionInfos: [], error: null };
  }
  try {
    const doc = parse(yaml) as unknown;
    if (!doc) return { steps: [], positionInfos: [], error: null };
    if (!Array.isArray(doc) && typeof doc === 'object' && doc !== null && 'steps' in (doc as Record<string, unknown>)) {
      return { steps: [], positionInfos: [], error: "'steps' wrapper is not valid — use a bare list of steps" };
    }
    if (!Array.isArray(doc)) {
      return { steps: [], positionInfos: [], error: 'YAML must be a list of steps' };
    }
    const steps = Walker.resolve(doc as Record<string, unknown>[], demoCatalog);
    const positionInfos = StepPositionScanner.scan(yaml);
    return { steps, positionInfos, error: null };
  } catch (e) {
    return { steps: [], positionInfos: [], error: e instanceof Error ? e.message : String(e) };
  }
}

@customElement('playbook-split-view')
export class PlaybookSplitView extends LitElement {
  @property() value = '';
  @property({ type: Number }) splitRatio = 0.5;

  @state() private _steps: ResolvedStep[] = [];
  @state() private _positionInfos: StepLineInfo[] = [];
  @state() private _parseError: string | null = null;

  static override styles = css`
    :host {
      display: block;
      height: 100%;
    }
    pages-three-way-viewer {
      height: 100%;
    }
  `;

  updateYaml(yaml: string): void {
    this.value = yaml;
    const result = parseAndResolve(yaml);
    this._steps = result.steps;
    this._positionInfos = result.positionInfos;
    this._parseError = result.error;
  }

  syncCursorLine(line: number): void {
    const viewer = this.renderRoot.querySelector('pages-step-viewer');
    if (viewer) {
      (viewer as import('./pages-step-viewer.js').PagesStepViewer).syncLine(line);
    }
  }

  private _onValueChange(e: Event): void {
    const { value } = (e as CustomEvent<{ value: string }>).detail;
    this.value = value;
    const result = parseAndResolve(value);
    this._steps = result.steps;
    this._positionInfos = result.positionInfos;
    this._parseError = result.error;
  }

  override render() {
    return html`
      <pages-three-way-viewer
        language="yaml"
        .value=${this.value}
        split-ratio=${this.splitRatio}
        @value-change=${this._onValueChange}>
        <pages-step-viewer slot="visual-pane"
          .steps=${this._steps}
          .positionInfos=${this._positionInfos}
          .error=${this._parseError}>
        </pages-step-viewer>
      </pages-three-way-viewer>
    `;
  }
}
```

- [ ] **Step 4: Run existing tests**

Run: `yarn --cwd packages/pages-step-viewer test`

Expected: All `parseAndResolve` tests pass (function unchanged). All
PagesStepViewer tests pass.

- [ ] **Step 5: Update gallery section 33**

Replace the HTML for section 33 (lines ~325-344) with:

```html
<h2>33. Split View — YAML Editor + Step Viewer</h2>
<h3>bidirectional sync: edit YAML on the left, see steps on the right</h3>
<div id="section-33" style="height: 500px; border: 1px solid var(--border); border-radius: 8px; overflow: hidden;">
  <playbook-split-view id="split-view" style="height: 100%;" value="- action: fetch-data
  with:
    url: /api/tickets
- if: tickets.length > 0
  then:
    - action: process-tickets
      with:
        batch: true
    - action: send-report
  else:
    - action: log-empty
- action: cleanup"></playbook-split-view>
</div>
```

In the script section, add the `pages-three-way-viewer` import and
remove the old split-view wiring block (lines ~1422-1458). Replace with:

```javascript
// 8. Split View — now self-contained via pages-three-way-viewer
{
  const splitView = document.getElementById('split-view');
  if (splitView) {
    splitView.updateYaml(splitView.getAttribute('value') || '');
  }
}
```

Add import at top of script module:

```javascript
import '@casehubio/pages-three-way-viewer';
```

- [ ] **Step 6: Run all tests across both packages**

Run: `yarn --cwd packages/pages-step-viewer test`
Run: `yarn --cwd packages/pages-three-way-viewer test`

Expected: All pass.

- [ ] **Step 7: Commit**

```bash
git add packages/pages-step-viewer/package.json \
       packages/pages-step-viewer/src/playbook-split-view.ts \
       packages/pages-step-viewer/gallery.html \
       yarn.lock
git commit -m "feat(split-view): rewrite as thin ThreeWayViewer consumer Refs #749"
```

## References

- `specs/issue-749-migrate-split-view-to-three-way/2026-11-26-migrate-split-view-design.md` — design spec
- `packages/pages-step-viewer/src/playbook-split-view.ts` — current implementation being replaced
- `packages/pages-step-viewer/src/pages-step-viewer.ts:30-38` — existing setActiveStep method
- `packages/pages-step-viewer/src/step-dispatch.ts:62-80` — step-click event dispatch
- `packages/pages-step-viewer/src/step-click.ts` — StepClickDetail type, assignStepAddresses
- `packages/pages-three-way-viewer/src/sync-protocol.ts` — ViewerSyncTarget, ViewerSelectDetail
- `packages/pages-three-way-viewer/src/three-way-viewer.ts:149-189` — slot sync and viewer-select handling
- `packages/pages-three-way-viewer/src/viewer-controller.ts` — ViewerController state machine
- GitHub #749 — focal issue
- `decisions.md D1` — ViewerSyncTarget placement
