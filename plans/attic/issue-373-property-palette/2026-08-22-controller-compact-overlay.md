# Controller Compact Overlay Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehub-pages#347 — Scenario controller display modes
**Issue group:** #347

**Goal:** Add a compact floating overlay mode to the scenario controller so it can be embedded in the helpdesk UI as a draggable pill that expands to show full controls.

**Architecture:** Add a `mode` property (`full` | `compact`) to `PagesScenarioController`. In compact mode, render a collapsed pill (play/pause icon, scenario name, progress) that expands to the full outline + transport on click. Position with `position: fixed`, make draggable via pointer events. Embed in the helpdesk `index.html` with `mode="compact"`.

**Tech Stack:** Lit 3, TypeScript, CSS (Shadow DOM), pointer events API

## Global Constraints

- All changes are in the `pages-aria` package (TS) and the helpdesk example (HTML)
- Shadow DOM encapsulates all styles — no global CSS leakage
- `full` mode (default) must be completely unchanged — zero visual regression
- The component must work without a running scenario (idle pill shows "No scenario")

---

## Batch 1: Compact overlay mode + helpdesk integration

### Task 1: Compact mode — mode property, pill render, expand/collapse, drag, CSS

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Test: Manual — rebuild bundle, reload browser, verify in helpdesk

**Interfaces:**
- Consumes: existing `_renderOutline()`, `_renderTransport()`, `_renderStatus()`, `ScenarioConnectionController.state`
- Produces: `mode` property (`'full' | 'compact'`), compact pill render, expand/collapse toggle, drag handler

- [ ] **Step 1: Add the `mode` property and `_expanded` state**

Add to the class properties section (after the existing `baseUrl` property at line 73):

```typescript
@property() mode: 'full' | 'compact' = 'full';
@state() private _expanded = false;
```

Use `ide_replace_text_in_file` to insert after the `baseUrl` property line.

- [ ] **Step 2: Add compact CSS styles**

Append to the existing `static override styles` CSS block (before the closing backtick+semicolon). Add styles for:

```css
/* Compact mode — host */
:host([mode="compact"]) {
  position: fixed;
  bottom: 16px;
  right: 16px;
  z-index: 9999;
  width: auto;
  font-size: var(--pages-font-size-sm, 12px);
}

/* Compact pill (collapsed) */
.compact-pill {
  display: flex;
  align-items: center;
  gap: var(--pages-space-2, 8px);
  padding: var(--pages-space-2, 8px) var(--pages-space-3, 12px);
  background: rgba(15, 23, 42, 0.9);
  backdrop-filter: blur(8px);
  color: #e2e8f0;
  border-radius: var(--pages-radius-lg, 8px);
  cursor: pointer;
  user-select: none;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
  white-space: nowrap;
}
.compact-pill:hover { background: rgba(15, 23, 42, 0.95); }
.compact-pill button {
  background: none; border: none; color: #38bdf8;
  cursor: pointer; font-size: 14px; padding: 0; line-height: 1;
}
.compact-pill .scenario-name {
  color: #94a3b8; font-size: var(--pages-font-size-sm, 12px);
  max-width: 160px; overflow: hidden; text-overflow: ellipsis;
}
.compact-pill .progress-pct {
  color: #38bdf8; font-weight: 600;
  font-size: var(--pages-font-size-sm, 12px);
}

/* Compact expanded card */
.compact-card {
  background: rgba(15, 23, 42, 0.95);
  backdrop-filter: blur(12px);
  border-radius: var(--pages-radius-lg, 8px);
  box-shadow: 0 8px 24px rgba(0,0,0,0.4);
  color: #e2e8f0;
  width: 280px;
  max-height: 60vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}
.compact-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--pages-space-2, 8px) var(--pages-space-3, 12px);
  cursor: grab;
  border-bottom: 1px solid rgba(255,255,255,0.1);
}
.compact-header:active { cursor: grabbing; }
.compact-header .scenario-name { color: #94a3b8; font-size: var(--pages-font-size-sm, 12px); }
.compact-header button {
  background: none; border: none; color: #64748b;
  cursor: pointer; font-size: 14px; padding: 0; line-height: 1;
}
.compact-header button:hover { color: #e2e8f0; }
.compact-body { overflow-y: auto; flex: 1; }

/* Override inner styles in compact mode */
:host([mode="compact"]) .outline-empty { color: #64748b; }
:host([mode="compact"]) .outline-heading { color: #e2e8f0; }
:host([mode="compact"]) .outline-heading:hover { background: rgba(255,255,255,0.05); }
:host([mode="compact"]) .outline-step { color: #94a3b8; }
:host([mode="compact"]) .outline-step:hover { background: rgba(255,255,255,0.05); }
:host([mode="compact"]) .outline-step.current { background: rgba(56,189,248,0.15); color: #38bdf8; }
:host([mode="compact"]) .outline-step.completed { color: #475569; }
:host([mode="compact"]) .transport {
  border-color: rgba(255,255,255,0.1);
}
:host([mode="compact"]) .transport button {
  color: #94a3b8; border-color: rgba(255,255,255,0.15);
}
:host([mode="compact"]) .transport button:hover:not(:disabled) {
  background: rgba(255,255,255,0.05); color: #e2e8f0;
}
:host([mode="compact"]) .status-bar { color: #475569; }
:host([mode="compact"]) .connection-status.connected { color: #4ade80; }
:host([mode="compact"]) .connection-status.disconnected { color: #f87171; }
:host([mode="compact"]) .speed-label { color: #94a3b8; }
:host([mode="compact"]) .progress { color: #38bdf8; }
```

Use `ide_replace_text_in_file` to find the last style rule before the closing `` `;`` and insert after it.

- [ ] **Step 3: Add the compact render methods**

Add two new private methods after `_renderStatus()`:

```typescript
private _renderCompactPill(): TemplateResult {
  const s = this._conn?.state;
  const name = s?.scenario ?? 'No scenario';
  const pct = Math.round((s?.progress ?? 0) * 100);
  return html`
    <div class="compact-pill"
         @click=${() => { this._expanded = true; }}
         @pointerdown=${this._onDragStart}>
      <button aria-label=${s?.paused !== false ? 'Resume' : 'Pause'}
              @click=${(e: Event) => {
                e.stopPropagation();
                if (s?.scenario) void this._conn.sendCommand(s.paused ? '/resume' : '/pause');
              }}>
        ${s?.paused !== false ? '▶' : '⏸'}
      </button>
      <span class="scenario-name">${name}</span>
      <span class="progress-pct">${pct}%</span>
    </div>
  `;
}

private _renderCompactCard(): TemplateResult {
  const s = this._conn?.state;
  const name = s?.scenario ?? 'No scenario';
  return html`
    <div class="compact-card">
      <div class="compact-header" @pointerdown=${this._onDragStart}>
        <span class="scenario-name">${name}</span>
        <button aria-label="Collapse" @click=${() => { this._expanded = false; }}>✕</button>
      </div>
      <div class="compact-body">
        ${this._renderOutline()}
      </div>
      ${this._renderTransport()}
      ${this._renderStatus()}
    </div>
  `;
}
```

- [ ] **Step 4: Add drag handler methods**

Add drag state and handler methods:

```typescript
private _dragOffset = { x: 0, y: 0 };

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
```

- [ ] **Step 5: Update the render() method for mode switching**

Replace the existing `render()` method body:

```typescript
override render(): TemplateResult {
  if (!this.connection && !this.baseUrl) {
    return html`<div class="error">No connection configured</div>`;
  }
  if (this.mode === 'compact') {
    return this._expanded ? this._renderCompactCard() : this._renderCompactPill();
  }
  return html`
    ${this._renderOutline()}
    ${this._renderTransport()}
    ${this._renderStatus()}
  `;
}
```

Use `ide_replace_text_in_file` to replace the existing render method.

- [ ] **Step 6: Build the controller bundle**

Run: `yarn workspace @casehubio/pages-aria run build:controller`
Expected: `dist/controller.js` built successfully.

Copy to helpdesk resources:
```python
import shutil
shutil.copy2('.../pages-aria/dist/controller.js', '.../helpdesk/.../scenario/controller.js')
```

- [ ] **Step 7: Verify full mode is unchanged**

Navigate to `http://localhost:8090/scenario/remote.html` (uses full mode).
Take screenshot. Verify layout matches the previous full-mode screenshot — outline tree, transport bar, status bar, same positions.

- [ ] **Step 8: Commit**

```bash
git -C $PROJECT add packages/pages-aria/src/controller/scenario-controller.ts
git commit -m "feat(#347): compact overlay mode for scenario controller

Add mode property (full | compact) to PagesScenarioController.
Compact mode renders a floating pill (play/pause, name, progress)
that expands to a card with full outline + transport on click.
Draggable via pointer events.

Refs casehubio/casehub-pages#347"
```

### Task 2: Embed compact controller in helpdesk UI

**Files:**
- Modify: `examples/helpdesk/src/main/resources/META-INF/resources/index.html`

**Interfaces:**
- Consumes: `<pages-scenario-controller mode="compact" baseurl="...">` from Task 1

- [ ] **Step 1: Add the controller component to helpdesk index.html**

Add before the closing `</body>` tag:

```html
<pages-scenario-controller mode="compact" baseurl=""></pages-scenario-controller>
<script>
  document.querySelector('pages-scenario-controller').setAttribute('baseurl', window.location.origin);
</script>
<script type="module">import '/scenario/controller.js';</script>
```

Use `ide_replace_text_in_file` on the helpdesk project to insert before `</body>`.

- [ ] **Step 2: Verify the overlay in the browser**

1. Navigate to `http://localhost:8090/` (helpdesk UI)
2. Take screenshot — verify floating pill appears in bottom-right corner
3. Start a scenario via curl:
   ```
   POST /scenario/start {"yaml": "...help-desk-basic.yaml..."}
   ```
4. Take screenshot — verify pill updates with scenario name and progress
5. Click the pill (via Playwright) — verify it expands to show outline + transport
6. Take screenshot of expanded state

- [ ] **Step 3: Commit**

```bash
git -C $HELPDESK add src/main/resources/META-INF/resources/index.html
git commit -m "feat(#347): embed compact scenario controller in helpdesk UI

Floating overlay in bottom-right corner shows scenario state
and provides transport controls without leaving the helpdesk app.

Refs casehubio/casehub-pages#347"
```

## References

- [2026-08-22-controller-compact-overlay-design.md] — design spec
- `packages/pages-aria/src/controller/scenario-controller.ts` — controller component
- `packages/pages-aria/src/controller/scenario-connection-controller.ts` — connection logic
- `examples/helpdesk/src/main/resources/META-INF/resources/index.html` — helpdesk UI
- D21, D22 — compact mode layout and scope decisions
- [GitHub casehub-pages#347] — focal issue
