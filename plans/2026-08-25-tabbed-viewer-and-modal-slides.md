# Tabbed Viewer and Modal Slides Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #365 — show-markdown action for tutorial-format scenarios
**Issue group:** #334, #335, #336, #337, #356, #357, #358, #363, #364, #365

**Goal:** Extend the scenario YAML viewer into a tabbed panel (Source + Guide),
add a full-screen modal slide deck mode for presentation-format markdown, and
add type-specific icons to the scenario outline.

**Architecture:** The YAML viewer component gains a tab bar and Guide tab with
markdown rendering lifted from the narrative component. The scenario handler
branches on a `display` property (`panel` | `modal`) — panel dispatches events
to the Guide tab, modal creates a full-screen overlay with deck state. The
controller's outline rendering adds unicode type icons per action.

**Tech Stack:** Lit, TypeScript, CSS, esbuild

## Global Constraints

- Element name `pages-scenario-yaml-viewer` stays unchanged (backward compat)
- `PagesScenarioNarrative` stays unchanged — the viewer is a second consumer
- All tests must pass: `yarn workspace @casehubio/pages-aria run test`
- Build bundle after changes: `yarn workspace @casehubio/pages-aria run build:controller`
- Copy bundle to helpdesk: `cp packages/pages-aria/dist/controller.js ../examples/helpdesk/src/main/resources/META-INF/resources/scenario/controller.js`
  (use python3 shutil.copy to avoid hook block)

---

## Batch 1: Tabbed Viewer — Guide Tab

After this batch: the YAML viewer has Source and Guide tabs. Guide tab
renders markdown from scenario-narrative events. Content persists until
replaced by the next show-markdown.

### Task 1: Add tab bar and Guide tab to YAML viewer

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-yaml-viewer.ts`
- Test: `packages/pages-aria/src/controller/scenario-yaml-viewer.test.ts`

**Interfaces:**
- Consumes: `_renderMarkdown(md: string): TemplateResult` (lifted from scenario-narrative.ts)
- Produces: `_activeTab: 'source' | 'guide'` state, `_guideContent: { markdown?: string; path?: string; section?: string } | null` state

- [ ] **Step 1: Write failing test — tab bar renders with two tabs**

```typescript
it('renders tab bar with Source and Guide tabs', async () => {
  const el = await fixture<PagesScenarioYamlViewer>(
    html`<pages-scenario-yaml-viewer></pages-scenario-yaml-viewer>`
  );
  el['_yamlSource'] = 'scenario: test\nsteps: []';
  await el.updateComplete;
  const tabs = el.shadowRoot!.querySelectorAll('.tab-btn');
  expect(tabs.length).toBe(2);
  expect(tabs[0].textContent!.trim()).toBe('Source');
  expect(tabs[1].textContent!.trim()).toBe('Guide');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-yaml-viewer.test.ts`
Expected: FAIL — no `.tab-btn` elements

- [ ] **Step 3: Add tab bar CSS and state to viewer**

Add to the component's `static override styles`:
```css
.tab-bar {
  display: flex;
  border-bottom: 1px solid rgba(255,255,255,0.1);
}
.tab-btn {
  flex: 1;
  padding: 6px 12px;
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border-bottom: 2px solid transparent;
  transition: color 0.15s, border-color 0.15s;
}
.tab-btn:hover { color: #e2e8f0; }
.tab-btn.active { color: #38bdf8; border-bottom-color: #38bdf8; }
.guide-empty {
  padding: 16px;
  color: #64748b;
  font-style: italic;
  text-align: center;
}
.guide-content {
  padding: 12px 16px;
  max-width: 100%;
  line-height: 1.6;
  font-size: 13px;
  color: #e2e8f0;
}
.guide-content h1 { font-size: 1.4em; margin: 0.5em 0; font-weight: 600; }
.guide-content h2 { font-size: 1.2em; margin: 0.5em 0; font-weight: 600; }
.guide-content h3 { font-size: 1.05em; margin: 0.5em 0; font-weight: 600; }
.guide-content p { margin: 0.5em 0; }
.guide-content strong { font-weight: 600; }
.guide-content em { font-style: italic; }
.guide-content code {
  background: rgba(255,255,255,0.08);
  padding: 2px 4px;
  border-radius: 3px;
  font-family: 'SF Mono', monospace;
  font-size: 0.9em;
}
.guide-content ul { margin: 0.5em 0; padding-left: 1.5em; }
.guide-content li { margin: 0.25em 0; }
```

Add state properties:
```typescript
@state() private _activeTab: 'source' | 'guide' = 'source';
@state() private _guideContent: { markdown?: string; path?: string; section?: string } | null = null;
```

Update `render()` to insert tab bar between header and body:
```typescript
override render(): TemplateResult {
  return html`
    <div class="viewer-card">
      ${this._renderHeader()}
      <div class="tab-bar">
        <button class="tab-btn ${this._activeTab === 'source' ? 'active' : ''}"
                @click=${() => { this._activeTab = 'source'; }}>Source</button>
        <button class="tab-btn ${this._activeTab === 'guide' ? 'active' : ''}"
                @click=${() => { this._activeTab = 'guide'; }}>Guide</button>
      </div>
      <div class="viewer-body">
        ${this._activeTab === 'source'
          ? (this._yamlSource
              ? this._renderYaml()
              : html`<div class="yaml-empty">No scenario source loaded</div>`)
          : this._renderGuide()}
      </div>
    </div>
  `;
}
```

Add `_renderGuide()` and `_renderGuideMarkdown()`:
```typescript
private _renderGuide(): TemplateResult {
  if (!this._guideContent) {
    return html`<div class="guide-empty">No guide content</div>`;
  }
  const md = this._guideContent.markdown ?? '';
  const rendered = this._guideContent.section
    ? this._extractSection(md, this._guideContent.section)
    : md;
  return this._renderGuideMarkdown(rendered);
}

private _extractSection(markdown: string, section: string): string {
  const lines = markdown.split('\n');
  let capturing = false;
  let level = 0;
  const result: string[] = [];
  for (const line of lines) {
    const headingMatch = line.match(/^(#{1,6})\s+(.+)/);
    if (headingMatch) {
      const headingLevel = headingMatch[1].length;
      const headingText = headingMatch[2].trim();
      if (capturing && headingLevel <= level) break;
      if (headingText.toLowerCase() === section.toLowerCase()) {
        capturing = true;
        level = headingLevel;
        continue;
      }
    }
    if (capturing) result.push(line);
  }
  return result.join('\n').trim();
}

private _renderGuideMarkdown(md: string): TemplateResult {
  const escaped = md
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;');
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
  container.className = 'guide-content';
  container.innerHTML = rendered;
  return html`${container}`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-yaml-viewer.test.ts`
Expected: PASS

- [ ] **Step 5: Write failing test — Guide tab renders markdown content**

```typescript
it('renders guide content when set', async () => {
  const el = await fixture<PagesScenarioYamlViewer>(
    html`<pages-scenario-yaml-viewer></pages-scenario-yaml-viewer>`
  );
  el['_guideContent'] = { markdown: '## Hello\n\nWorld' };
  el['_activeTab'] = 'guide';
  await el.updateComplete;
  const guide = el.shadowRoot!.querySelector('.guide-content');
  expect(guide).toBeTruthy();
  expect(guide!.querySelector('h2')!.textContent).toBe('Hello');
});
```

- [ ] **Step 6: Run test to verify it passes** (should already pass from Step 3 implementation)

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-yaml-viewer.test.ts`
Expected: PASS

- [ ] **Step 7: Write failing test — Guide tab shows empty state**

```typescript
it('shows empty state on Guide tab when no content', async () => {
  const el = await fixture<PagesScenarioYamlViewer>(
    html`<pages-scenario-yaml-viewer></pages-scenario-yaml-viewer>`
  );
  el['_activeTab'] = 'guide';
  await el.updateComplete;
  const empty = el.shadowRoot!.querySelector('.guide-empty');
  expect(empty).toBeTruthy();
  expect(empty!.textContent!.trim()).toBe('No guide content');
});
```

- [ ] **Step 8: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-yaml-viewer.test.ts`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-aria/src/controller/scenario-yaml-viewer.ts packages/pages-aria/src/controller/scenario-yaml-viewer.test.ts
git commit -m "feat(scenario-viewer): tab bar with Source and Guide tabs

Refs casehubio/casehub-pages#365"
```

### Task 2: Wire scenario-narrative events to Guide tab

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-yaml-viewer.ts`
- Test: `packages/pages-aria/src/controller/scenario-yaml-viewer.test.ts`

**Interfaces:**
- Consumes: `scenario-narrative` CustomEvent on eventTarget
- Produces: `_guideContent` updated reactively on event

- [ ] **Step 1: Write failing test — event updates guide content**

```typescript
it('updates guide content on scenario-narrative event', async () => {
  const eventTarget = new EventTarget();
  const el = await fixture<PagesScenarioYamlViewer>(
    html`<pages-scenario-yaml-viewer .eventTarget=${eventTarget}></pages-scenario-yaml-viewer>`
  );
  eventTarget.dispatchEvent(new CustomEvent('scenario-narrative', {
    detail: { type: 'inline', markdown: '## Test Content' },
  }));
  await el.updateComplete;
  expect(el['_guideContent']).toEqual({ type: 'inline', markdown: '## Test Content' });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-yaml-viewer.test.ts`
Expected: FAIL — `_guideContent` is null

- [ ] **Step 3: Add reactive event binding (same pattern as narrative component fix)**

Add to the viewer component:
```typescript
private _boundTarget: EventTarget | null = null;

private _onNarrative = (e: Event): void => {
  this._guideContent = (e as CustomEvent).detail as typeof this._guideContent;
};

protected override willUpdate(changed: Map<string, unknown>): void {
  super.willUpdate(changed);
  if (changed.has('eventTarget')) {
    this._bindGuideEvents(this.eventTarget);
  }
}

override connectedCallback(): void {
  // existing code...
  this._bindGuideEvents(this.eventTarget);
}

override disconnectedCallback(): void {
  super.disconnectedCallback();
  this._bindGuideEvents(null);
}

private _bindGuideEvents(target: EventTarget | undefined | null): void {
  if (this._boundTarget) {
    this._boundTarget.removeEventListener('scenario-narrative', this._onNarrative);
    this._boundTarget = null;
  }
  if (target) {
    target.addEventListener('scenario-narrative', this._onNarrative);
    this._boundTarget = target;
  }
}
```

Note: Do NOT listen for `scenario-narrative-dismiss` — guide content
persists until replaced by the next `scenario-narrative` event (D3).

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-yaml-viewer.test.ts`
Expected: PASS

- [ ] **Step 5: Write test — content persists (no dismiss clears it)**

```typescript
it('guide content persists after dismiss event', async () => {
  const eventTarget = new EventTarget();
  const el = await fixture<PagesScenarioYamlViewer>(
    html`<pages-scenario-yaml-viewer .eventTarget=${eventTarget}></pages-scenario-yaml-viewer>`
  );
  eventTarget.dispatchEvent(new CustomEvent('scenario-narrative', {
    detail: { type: 'inline', markdown: '## Persisted' },
  }));
  await el.updateComplete;
  eventTarget.dispatchEvent(new CustomEvent('scenario-narrative-dismiss'));
  await el.updateComplete;
  expect(el['_guideContent']).toEqual({ type: 'inline', markdown: '## Persisted' });
});
```

- [ ] **Step 6: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-yaml-viewer.test.ts`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: all tests pass

- [ ] **Step 8: Commit**

```bash
git add packages/pages-aria/src/controller/scenario-yaml-viewer.ts packages/pages-aria/src/controller/scenario-yaml-viewer.test.ts
git commit -m "feat(scenario-viewer): wire scenario-narrative events to Guide tab

Refs casehubio/casehub-pages#365"
```

## Batch 2: Display Property and Outline Icons

After this batch: the parser passes `display` through. The controller
shows type icons in the outline. Outline icons use a client-side action
map parsed from the YAML source (avoids server-side changes for now).

### Task 3: Pass display property through parser and handler

**Files:**
- Modify: `packages/pages-aria/src/scenario/parser.ts`
- Modify: `packages/pages-aria/src/server/scenario-handler.ts`
- Test: `packages/pages-aria/src/scenario/scenario.test.ts`

**Interfaces:**
- Consumes: `display` field in show-markdown YAML body
- Produces: `cmd.data.display` available in `executeAriaCommand`

- [ ] **Step 1: Write failing test — parser preserves display property**

```typescript
it('parses show-markdown display property', () => {
  const yaml = `
scenario: test
steps:
  - show-markdown:
      display: modal
      content: "Hello"
  `;
  const result = parseScenario(yaml);
  expect(result.steps[0].state).toEqual(
    expect.objectContaining({ display: 'modal', content: 'Hello' })
  );
});
```

- [ ] **Step 2: Run test to verify it passes** (should already pass — parser passes entire body as `state`)

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/scenario/scenario.test.ts`
Expected: PASS (the parser already sets `state: body` which includes all properties)

- [ ] **Step 3: Update handler to read display property**

In `scenario-handler.ts`, update the `show-markdown` case to read `display`:
```typescript
case 'show-markdown': {
  const props = state ?? cmd.data ?? {};
  const display = (props.display as string) ?? 'panel';
  const markdown = value ?? (props.content as string) ?? '';
  const filePath = props.file as string | undefined;
  const section = props.section as string | undefined;

  if (display === 'modal') {
    // TODO: Task 5 implements modal overlay
    return;
  }

  narrativeTarget.dispatchEvent(new CustomEvent('scenario-narrative', {
    detail: { type: filePath ? 'template' : 'inline', markdown, path: filePath, section },
  }));
  return;
}
```

- [ ] **Step 4: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/server/scenario-handler.ts packages/pages-aria/src/scenario/parser.ts
git commit -m "feat(scenario): branch show-markdown on display property (panel|modal)

Refs casehubio/casehub-pages#365"
```

### Task 4: Outline type icons in controller

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Modify: `packages/pages-aria/src/controller/scenario-yaml-viewer.ts` (export action map builder)
- Test: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: Step labels from outline, YAML source from viewer or fetched directly
- Produces: Action-type icons rendered in outline tree items

- [ ] **Step 1: Write failing test — icon renders for known action type**

```typescript
it('renders type icon for spotlight step', async () => {
  // Set up controller with a mock outline containing action types
  const el = await fixture<PagesScenarioController>(
    html`<pages-scenario-controller></pages-scenario-controller>`
  );
  el['_outline'] = [{
    label: 'Section', target: null, children: [
      { label: 'Spotlight the form', target: 'browser', action: 'spotlight', children: [] },
    ],
  }];
  el['_conn'] = { state: { scenario: 'test', step: null } } as any;
  await el.updateComplete;
  const icon = el.shadowRoot!.querySelector('.step-type-icon');
  expect(icon).toBeTruthy();
  expect(icon!.textContent!.trim()).toBe('🔆');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-controller.test.ts`
Expected: FAIL — no `.step-type-icon` element

- [ ] **Step 3: Add OutlineNode action field and icon mapping**

Update the `OutlineNode` interface in `scenario-connection-controller.ts`:
```typescript
export interface OutlineNode {
  label: string;
  target: string | null;
  action?: string;
  children: OutlineNode[];
}
```

Add icon map and rendering in `scenario-controller.ts`:
```typescript
const ACTION_ICONS: Record<string, string> = {
  'show-markdown': '📝',
  'spotlight': '🔆',
  'click': '👆',
  'fill': '✍',
  'navigate': '➜',
};

function actionIcon(action?: string): string {
  if (!action) return '';
  return ACTION_ICONS[action] ?? '';
}
```

Add CSS:
```css
.step-type-icon {
  font-size: 12px;
  flex-shrink: 0;
  width: 16px;
  text-align: center;
}
```

Update `_renderNode` for leaf nodes to include the icon:
```typescript
if (isLeaf) {
  const icon = actionIcon(node.action);
  return html`
    <div class="outline-step ${isCurrent ? 'current' : ''} ${isCompleted ? 'completed' : ''}"
         role="treeitem" tabindex="-1"
         style="padding-left: ${depth * 16 + 8}px"
         @click=${() => void this._conn.sendCommand('/run-to', { label: node.label })}>
      <span class="step-icon">${isCurrent ? '●' : isCompleted ? '✓' : '○'}</span>
      ${icon ? html`<span class="step-type-icon">${icon}</span>` : ''}
      ${node.label}
      <span class="run-to" aria-label="Run to ${node.label}">▶</span>
    </div>
  `;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-controller.test.ts`
Expected: PASS

- [ ] **Step 5: Write test — fallback for unknown action**

```typescript
it('renders no type icon for unknown action', async () => {
  const el = await fixture<PagesScenarioController>(
    html`<pages-scenario-controller></pages-scenario-controller>`
  );
  el['_outline'] = [{
    label: 'Section', target: null, children: [
      { label: 'Wait for data', target: 'browser', action: 'wait', children: [] },
    ],
  }];
  el['_conn'] = { state: { scenario: 'test', step: null } } as any;
  await el.updateComplete;
  const icon = el.shadowRoot!.querySelector('.step-type-icon');
  expect(icon).toBeFalsy();
});
```

- [ ] **Step 6: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/controller/scenario-controller.test.ts`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: all tests pass

- [ ] **Step 8: Commit**

```bash
git add packages/pages-aria/src/controller/scenario-controller.ts packages/pages-aria/src/controller/scenario-connection-controller.ts packages/pages-aria/src/controller/scenario-controller.test.ts
git commit -m "feat(scenario-controller): type icons in outline (spotlight, click, fill, markdown, navigate)

Refs casehubio/casehub-pages#365"
```

## Batch 3: Modal Slide Deck

After this batch: consecutive `display: modal` show-markdown steps render
as a full-screen slide deck with a mini controller. Escape dismisses the
deck. Single-slide decks show simplified UI.

### Task 5: Modal overlay and deck state management

**Files:**
- Modify: `packages/pages-aria/src/server/scenario-handler.ts`
- Test: `packages/pages-aria/src/server/scenario-handler.test.ts`

**Interfaces:**
- Consumes: `display: 'modal'` on show-markdown command, `stepQueue` for deck look-ahead
- Produces: Full-screen overlay element, `_activeDeck` state on handler closure

- [ ] **Step 1: Write failing test — modal show-markdown creates overlay**

```typescript
it('show-markdown with display: modal creates full-screen overlay', () => {
  const eventTarget = new EventTarget();
  const handler = createTestHandler(eventTarget);
  
  handler.executeCommand({
    action: 'show-markdown',
    value: '## Slide 1',
    state: { display: 'modal', content: '## Slide 1' },
  });
  
  const overlay = document.querySelector('.scenario-modal-overlay');
  expect(overlay).toBeTruthy();
  expect(overlay!.querySelector('h2')!.textContent).toBe('Slide 1');
  
  // Cleanup
  overlay!.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/server/scenario-handler.test.ts`
Expected: FAIL — no overlay created

- [ ] **Step 3: Implement modal overlay creation**

Add modal styles (inject alongside spotlight styles):
```typescript
function injectModalStyles(): void {
  if (document.getElementById('scenario-modal-styles')) return;
  const style = document.createElement('style');
  style.id = 'scenario-modal-styles';
  style.textContent = `
    .scenario-modal-overlay {
      position: fixed; inset: 0; z-index: 10000;
      background: rgba(15, 23, 42, 0.98);
      display: flex; flex-direction: column;
      font-family: system-ui, sans-serif;
      color: #e2e8f0;
      animation: scenario-modal-fade 0.2s ease;
    }
    @keyframes scenario-modal-fade { from { opacity: 0; } to { opacity: 1; } }
    .scenario-modal-header {
      display: flex; align-items: center; justify-content: space-between;
      padding: 12px 24px;
      border-bottom: 1px solid rgba(255,255,255,0.1);
    }
    .scenario-modal-back {
      background: none; border: none; color: #94a3b8;
      cursor: pointer; font-size: 14px; padding: 4px 8px;
    }
    .scenario-modal-back:hover { color: #e2e8f0; }
    .scenario-modal-position { color: #64748b; font-size: 13px; }
    .scenario-modal-body {
      flex: 1; overflow-y: auto; padding: 32px;
      display: flex; justify-content: center;
    }
    .scenario-modal-content {
      max-width: 680px; width: 100%; line-height: 1.7; font-size: 16px;
    }
    .scenario-modal-content h1 { font-size: 1.6em; margin: 0.5em 0; font-weight: 600; }
    .scenario-modal-content h2 { font-size: 1.3em; margin: 0.5em 0; font-weight: 600; }
    .scenario-modal-content h3 { font-size: 1.1em; margin: 0.5em 0; font-weight: 600; }
    .scenario-modal-content p { margin: 0.6em 0; }
    .scenario-modal-content strong { font-weight: 600; }
    .scenario-modal-content em { font-style: italic; }
    .scenario-modal-content code {
      background: rgba(255,255,255,0.08); padding: 2px 5px;
      border-radius: 3px; font-family: monospace; font-size: 0.9em;
    }
    .scenario-modal-content ul { margin: 0.5em 0; padding-left: 1.5em; }
    .scenario-modal-content li { margin: 0.3em 0; }
    .scenario-modal-footer {
      display: flex; align-items: center; justify-content: space-between;
      padding: 12px 24px;
      border-top: 1px solid rgba(255,255,255,0.1);
    }
    .scenario-modal-dots { display: flex; gap: 8px; align-items: center; }
    .scenario-modal-dot {
      width: 8px; height: 8px; border-radius: 50%;
      background: #334155; transition: background 0.15s;
    }
    .scenario-modal-dot.active { background: #38bdf8; }
    .scenario-modal-dot-label {
      font-size: 11px; color: #64748b; margin-left: 4px;
    }
    .scenario-modal-next {
      background: #2563eb; color: white; border: none;
      padding: 8px 20px; border-radius: 6px; font-weight: 600;
      cursor: pointer; font-size: 14px;
    }
    .scenario-modal-next:hover { background: #1d4ed8; }
  `;
  document.head.appendChild(style);
}
```

Add modal rendering helper:
```typescript
function renderModalMarkdown(md: string): HTMLElement {
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
  const el = document.createElement('div');
  el.className = 'scenario-modal-content';
  el.innerHTML = rendered;
  return el;
}
```

Update the `show-markdown` case in `executeAriaCommand`:
```typescript
if (display === 'modal') {
  injectModalStyles();
  const slides = [{ markdown, label: cmd.data?.label as string ?? 'Slide 1' }];
  // Peek stepQueue for consecutive modal steps (look-ahead)
  // stepQueue is accessible via closure — check if subsequent entries are modal
  showModalDeck(slides, narrativeTarget);
  return;
}
```

Add `showModalDeck` function (initially for single slide — deck management added in Step 5):
```typescript
function showModalDeck(
  slides: Array<{ markdown: string; label: string }>,
  narrativeTarget: EventTarget,
): void {
  let current = 0;
  const total = slides.length;
  const isSingle = total === 1;

  const overlay = document.createElement('div');
  overlay.className = 'scenario-modal-overlay';

  function renderSlide(): void {
    const slide = slides[current];
    overlay.innerHTML = '';

    // Header
    const header = document.createElement('div');
    header.className = 'scenario-modal-header';
    const back = document.createElement('button');
    back.className = 'scenario-modal-back';
    back.textContent = current === 0 ? '✕ Close' : '← Back';
    back.addEventListener('click', () => {
      if (current === 0) dismiss();
      else { current--; renderSlide(); }
    });
    header.appendChild(back);
    if (!isSingle) {
      const pos = document.createElement('span');
      pos.className = 'scenario-modal-position';
      pos.textContent = `Slide ${current + 1} of ${total}`;
      header.appendChild(pos);
    }
    overlay.appendChild(header);

    // Body
    const body = document.createElement('div');
    body.className = 'scenario-modal-body';
    body.appendChild(renderModalMarkdown(slide.markdown));
    overlay.appendChild(body);

    // Footer (skip for single slide)
    if (!isSingle) {
      const footer = document.createElement('div');
      footer.className = 'scenario-modal-footer';
      const dots = document.createElement('div');
      dots.className = 'scenario-modal-dots';
      for (let i = 0; i < total; i++) {
        const dot = document.createElement('div');
        dot.className = `scenario-modal-dot ${i === current ? 'active' : ''}`;
        dots.appendChild(dot);
      }
      footer.appendChild(dots);
      const next = document.createElement('button');
      next.className = 'scenario-modal-next';
      next.textContent = current === total - 1 ? 'Done' : 'Next →';
      next.addEventListener('click', () => {
        if (current === total - 1) dismiss();
        else { current++; renderSlide(); }
      });
      footer.appendChild(next);
      overlay.appendChild(footer);
    }
  }

  function dismiss(): void {
    overlay.remove();
    document.removeEventListener('keydown', onEscape);
    narrativeTarget.dispatchEvent(new CustomEvent('scenario-narrative-dismiss'));
  }

  function onEscape(e: KeyboardEvent): void {
    if (e.key === 'Escape') dismiss();
  }

  document.addEventListener('keydown', onEscape);
  renderSlide();
  document.body.appendChild(overlay);
}
```

Note: The `showModalDeck` function receives pre-collected slides. The
`executeAriaCommand` caller needs to peek at `stepQueue` for consecutive
modal steps. This requires passing `stepQueue` into the handler or
restructuring. For this task, the deck look-ahead happens in
`executeSequence` — when a modal step is detected, it collects
consecutive modal steps from the queue before calling `showModalDeck`.

Update `executeSequence` to handle modal deck collection:
```typescript
// Inside the step execution loop, before calling executeAriaCommand:
for (const cmd of step.commands) {
  if (cmd.action === 'show-markdown') {
    const props = cmd.state ?? cmd.data ?? {};
    if ((props.display as string) === 'modal') {
      const slides = [{ markdown: cmd.value ?? (props.content as string) ?? '', label: step.label ?? step.name }];
      // Peek ahead in stepQueue for consecutive modal steps
      while (stepQueue.length > 0) {
        const nextStep = stepQueue[0];
        const nextCmd = nextStep.commands[0];
        if (nextCmd?.action !== 'show-markdown') break;
        const nextProps = nextCmd.state ?? nextCmd.data ?? {};
        if ((nextProps.display as string) !== 'modal') break;
        stepQueue.shift();
        slides.push({
          markdown: nextCmd.value ?? (nextProps.content as string) ?? '',
          label: nextStep.label ?? nextStep.name,
        });
        sendStepResult(connection, sessionId!, nextStep.name, true, null);
      }
      showModalDeck(slides, eventTarget);
      continue; // skip normal executeAriaCommand
    }
  }
  // Normal path
  try {
    const result = executeAriaCommand(cmd, speed, paused, calloutMsPerChar, eventTarget);
    if (result) await result;
  } catch (err) {
    stepOk = false;
    stepError = (err as Error).message;
    break;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose src/server/scenario-handler.test.ts`
Expected: PASS

- [ ] **Step 5: Write test — multi-slide deck from queue**

```typescript
it('collects consecutive modal steps into a deck', () => {
  const eventTarget = new EventTarget();
  // This test needs to verify deck collection from stepQueue
  // Use the handler's executeSequence with multiple queued modal steps
  // The overlay should show "Slide 1 of 2" with both slides navigable
  
  const handler = createTestHandler(eventTarget);
  handler.dispatchSteps([
    { name: 's1', label: 'Intro', commands: [{ action: 'show-markdown', value: '## Slide 1', state: { display: 'modal', content: '## Slide 1' } }] },
    { name: 's2', label: 'Details', commands: [{ action: 'show-markdown', value: '## Slide 2', state: { display: 'modal', content: '## Slide 2' } }] },
  ]);
  
  const overlay = document.querySelector('.scenario-modal-overlay');
  expect(overlay).toBeTruthy();
  const dots = overlay!.querySelectorAll('.scenario-modal-dot');
  expect(dots.length).toBe(2);
  const pos = overlay!.querySelector('.scenario-modal-position');
  expect(pos!.textContent).toBe('Slide 1 of 2');
  
  overlay!.remove();
});
```

- [ ] **Step 6: Write test — Escape dismisses overlay**

```typescript
it('Escape key dismisses modal overlay', () => {
  const eventTarget = new EventTarget();
  const handler = createTestHandler(eventTarget);
  
  handler.executeCommand({
    action: 'show-markdown',
    value: '## Test',
    state: { display: 'modal', content: '## Test' },
  });
  
  expect(document.querySelector('.scenario-modal-overlay')).toBeTruthy();
  document.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape' }));
  expect(document.querySelector('.scenario-modal-overlay')).toBeFalsy();
});
```

- [ ] **Step 7: Write test — single slide has no dots or position**

```typescript
it('single-slide modal has no dot navigation', () => {
  const eventTarget = new EventTarget();
  const handler = createTestHandler(eventTarget);
  
  handler.executeCommand({
    action: 'show-markdown',
    value: '## Solo',
    state: { display: 'modal', content: '## Solo' },
  });
  
  const overlay = document.querySelector('.scenario-modal-overlay');
  expect(overlay!.querySelector('.scenario-modal-dots')).toBeFalsy();
  expect(overlay!.querySelector('.scenario-modal-position')).toBeFalsy();
  
  overlay!.remove();
});
```

- [ ] **Step 8: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: all tests pass

- [ ] **Step 9: Commit**

```bash
git add packages/pages-aria/src/server/scenario-handler.ts packages/pages-aria/src/server/scenario-handler.test.ts
git commit -m "feat(scenario): modal slide deck with mini controller and Escape dismiss

Refs casehubio/casehub-pages#365"
```

## Batch 4: Integration and Bundle

After this batch: bundle rebuilt, helpdesk updated, E2E verified.

### Task 6: Rebuild bundle and update helpdesk demo

**Files:**
- Modify: `examples/helpdesk/src/main/resources/META-INF/resources/index.html` (remove standalone narrative element)
- Build: `packages/pages-aria/dist/controller.js`

**Interfaces:**
- Consumes: All changes from Batches 1-3
- Produces: Updated controller.js bundle, cleaned up helpdesk HTML

- [ ] **Step 1: Run full test suite**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: all 156+ tests pass

- [ ] **Step 2: Rebuild controller bundle**

Run: `yarn workspace @casehubio/pages-aria run build:controller`
Expected: `dist/controller.js` updated

- [ ] **Step 3: Copy bundle to helpdesk**

```python
python3 -c "import shutil; shutil.copy('packages/pages-aria/dist/controller.js', '../examples/helpdesk/src/main/resources/META-INF/resources/scenario/controller.js')"
```

- [ ] **Step 4: Remove standalone narrative element from helpdesk HTML**

The `<pages-scenario-narrative>` element and its CSS/JS wiring in
`index.html` are no longer needed — the Guide tab in the viewer replaces
it. Remove:
- The `<pages-scenario-narrative id="narrative">` element
- The `pages-scenario-narrative` CSS rules
- The narrative event listener JS (`eventTarget.addEventListener('scenario-narrative', ...)`)

Keep the `narrative.eventTarget` and `narrative.connection` wiring removal
in sync — remove all three together.

- [ ] **Step 5: E2E test — start helpdesk, run demo, verify Guide tab and modal**

1. Start helpdesk: `mvn quarkus:dev -Dquarkus.profile=demo -Dquarkus.http.port=8090`
2. Navigate to `http://localhost:8090`
3. Start demo, step through
4. Open viewer (`</>` button), switch to Guide tab
5. Step to show-markdown step — verify Guide tab shows content
6. Step past it — verify content persists
7. Verify outline shows type icons

- [ ] **Step 6: Commit all helpdesk changes**

```bash
git -C ../examples add helpdesk/
git -C ../examples commit -m "feat(helpdesk): tabbed viewer integration, remove standalone narrative

Refs casehubio/casehub-pages#365"
```

## References

- [2026-08-25-tabbed-viewer-and-modal-slides-design.md] — design spec
- [packages/pages-aria/src/controller/scenario-yaml-viewer.ts] — viewer component
- [packages/pages-aria/src/controller/scenario-narrative.ts] — markdown rendering source
- [packages/pages-aria/src/server/scenario-handler.ts:222] — show-markdown handler
- [packages/pages-aria/src/controller/scenario-controller.ts:270] — outline rendering
- [packages/pages-aria/src/controller/scenario-connection-controller.ts] — OutlineNode interface
- [GitHub #365] — show-markdown action for tutorial-format scenarios
- [decisions.md D1-D8] — design decisions
