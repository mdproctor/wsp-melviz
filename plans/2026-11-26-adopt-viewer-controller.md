# Adopt ViewerController in playbook controller — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #750 — Adopt ViewerController in playbook controller
**Issue group:** #750

**Goal:** Create a PanelController in pages-three-way-viewer for floating
panel state management, then adopt it in the playbook controller to replace
ad-hoc boolean state.

**Architecture:** PanelController follows the same pattern as ViewerController
(constructor+options, state getters, transition methods, onChange callback)
but manages `closed`/`docked`/`floating` panel modes. The playbook controller
replaces `_yamlOpen` + `_docked` booleans with a PanelController instance,
simplifying its 8 viewer management methods.

**Tech Stack:** TypeScript 5.8, Lit 3, Vitest

## Global Constraints

- Lit web components, `@customElement` decorator pattern
- CSS custom properties use `--pages-` prefix
- `pages-three-way-viewer` exports: add PanelController, PanelMode, PanelControllerOptions
- Existing playbook controller tests must continue passing
- Existing `PagesPlaybookYamlViewer` component is unchanged

---

## Batch 1: PanelController + playbook controller adoption

### Task 1: Create PanelController with tests

**Files:**
- Create: `packages/pages-three-way-viewer/src/panel-controller.ts`
- Modify: `packages/pages-three-way-viewer/src/index.ts:1-6`
- Test: `packages/pages-three-way-viewer/src/panel-controller.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `PanelController` class with `mode` getter, `setMode(mode)`,
  `toggle()`. `PanelMode` type (`'closed' | 'docked' | 'floating'`).
  `PanelControllerOptions` interface.

- [ ] **Step 1: Write failing tests**

Create `packages/pages-three-way-viewer/src/panel-controller.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { PanelController } from './panel-controller.js';

describe('PanelController', () => {
  it('defaults to closed', () => {
    const c = new PanelController();
    expect(c.mode).toBe('closed');
  });

  it('respects custom initial mode', () => {
    const c = new PanelController({ initialMode: 'docked' });
    expect(c.mode).toBe('docked');
  });

  it('setMode changes mode and fires onChange', () => {
    const onChange = vi.fn();
    const c = new PanelController({ onChange });
    c.setMode('docked');
    expect(c.mode).toBe('docked');
    expect(onChange).toHaveBeenCalledWith('docked');
  });

  it('setMode with same mode is a no-op', () => {
    const onChange = vi.fn();
    const c = new PanelController({ initialMode: 'docked', onChange });
    c.setMode('docked');
    expect(onChange).not.toHaveBeenCalled();
  });

  it('toggle from closed goes to docked', () => {
    const c = new PanelController();
    c.toggle();
    expect(c.mode).toBe('docked');
  });

  it('toggle from docked goes to closed', () => {
    const c = new PanelController({ initialMode: 'docked' });
    c.toggle();
    expect(c.mode).toBe('closed');
  });

  it('toggle from floating goes to closed', () => {
    const c = new PanelController({ initialMode: 'floating' });
    c.toggle();
    expect(c.mode).toBe('closed');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/pages-three-way-viewer test`

Expected: FAIL — `panel-controller.js` does not exist.

- [ ] **Step 3: Implement PanelController**

Create `packages/pages-three-way-viewer/src/panel-controller.ts`:

```typescript
export type PanelMode = 'closed' | 'docked' | 'floating';

export interface PanelControllerOptions {
  initialMode?: PanelMode;
  onChange?: (mode: PanelMode) => void;
}

export class PanelController {
  private _mode: PanelMode;
  private _onChange: ((mode: PanelMode) => void) | undefined;

  constructor(options: PanelControllerOptions = {}) {
    this._mode = options.initialMode ?? 'closed';
    this._onChange = options.onChange;
  }

  get mode(): PanelMode {
    return this._mode;
  }

  setMode(mode: PanelMode): void {
    if (mode === this._mode) return;
    this._mode = mode;
    this._onChange?.(this._mode);
  }

  toggle(): void {
    this.setMode(this._mode === 'closed' ? 'docked' : 'closed');
  }
}
```

- [ ] **Step 4: Export from index.ts**

In `packages/pages-three-way-viewer/src/index.ts`, add:

```typescript
export { PanelController } from './panel-controller.js';
export type { PanelMode, PanelControllerOptions } from './panel-controller.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn --cwd packages/pages-three-way-viewer test`

Expected: All pass (existing 13 + 7 new = 20).

- [ ] **Step 6: Commit**

```bash
git add packages/pages-three-way-viewer/src/panel-controller.ts \
       packages/pages-three-way-viewer/src/panel-controller.test.ts \
       packages/pages-three-way-viewer/src/index.ts
git commit -m "feat(three-way-viewer): add PanelController for floating panel state Refs #750"
```

### Task 2: Adopt PanelController in playbook controller

**Files:**
- Modify: `packages/pages-aria/package.json` (add dependency)
- Modify: `packages/pages-aria/src/controller/playbook-controller.ts:1-10,186-204,598-612,661-746`

**Interfaces:**
- Consumes: `PanelController`, `PanelMode` from `@casehubio/pages-three-way-viewer`
  (from Task 1)
- Produces: same public API — `PagesPlaybookController` with identical
  external behavior

- [ ] **Step 1: Add pages-three-way-viewer dependency**

In `packages/pages-aria/package.json`, add to `dependencies`:

```json
"@casehubio/pages-three-way-viewer": "workspace:^"
```

- [ ] **Step 2: Run `yarn install`**

Run: `yarn install`

- [ ] **Step 3: Add import**

In `packages/pages-aria/src/controller/playbook-controller.ts`, add
after the existing imports (after line 10):

```typescript
import { PanelController, type PanelMode } from '@casehubio/pages-three-way-viewer';
```

- [ ] **Step 4: Replace state fields**

Remove `_yamlOpen` and `_docked` fields (lines 190, 201). Add
PanelController and reactive mirror in their place.

Replace:
```typescript
  @state() private _yamlOpen = false;
```
with:
```typescript
  @state() private _panelMode: PanelMode = 'closed';
```

Remove:
```typescript
  @state() private _docked = true;
```

Add after `_resizeHandler` (line 204):

```typescript
  private _panel = new PanelController({
    onChange: (mode) => {
      this._panelMode = mode;
      if (mode === 'closed') {
        this._hideYamlViewer();
      } else {
        this._ensureViewerCreated();
        this._yamlViewer!.style.display = 'block';
        if (mode === 'docked') {
          requestAnimationFrame(() => {
            requestAnimationFrame(() => { this._snapViewerToController(); });
          });
        }
      }
    },
  });
```

- [ ] **Step 5: Extract `_ensureViewerCreated` from `_showYamlViewer`**

Replace `_showYamlViewer()` (lines 670-690) with `_ensureViewerCreated()` —
same body but only the creation logic (the `if (!this._yamlViewer)` block).
Remove the `style.display` and `requestAnimationFrame` lines — those are
now in the onChange callback.

```typescript
  private _ensureViewerCreated(): void {
    if (!this._yamlViewer) {
      const viewer = document.createElement('pages-playbook-yaml-viewer') as PagesPlaybookYamlViewer;
      viewer.connection = this.connection;
      viewer.eventTarget = this.eventTarget;
      if (this.baseUrl) viewer.baseUrl = this.baseUrl;
      if (this.scenario) viewer.scenario = this.scenario;
      viewer.onClose = () => { this._panel.toggle(); };
      viewer.onDetach = () => { this._detachYaml(); };
      viewer.onDragMove = (left: number, top: number) => { this._onViewerDrag(left, top); };
      viewer.onDragEnd = () => { this._onViewerDragEnd(); };
      viewer.onResize = () => { if (this._panelMode === 'docked') this._snapViewerToController(); };
      document.body.appendChild(viewer);
      this._yamlViewer = viewer;
    }
  }
```

- [ ] **Step 6: Simplify `_toggleYaml`**

Replace:
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
with:
```typescript
  private _toggleYaml(): void {
    this._panel.toggle();
  }
```

- [ ] **Step 7: Simplify dock/undock methods**

Replace `_undockViewer` and `_dockViewer`:

```typescript
  private _undockViewer(): void {
    this._panel.setMode('floating');
  }

  private _dockViewer(): void {
    this._panel.setMode('docked');
  }
```

- [ ] **Step 8: Update `_onViewerDrag` to use PanelController**

Replace `this._docked = false` (around line 710) with
`this._panel.setMode('floating')`. Replace `if (!this._docked) return`
with `if (this._panelMode !== 'docked') return`:

```typescript
  private _onViewerDrag(left: number, top: number): void {
    if (this._panelMode !== 'docked') return;
    const hostRect = this.getBoundingClientRect();
    const viewerRect = this._yamlViewer!.getBoundingClientRect();
    const gap = hostRect.left - (left + viewerRect.width);
    if (Math.abs(gap - 8) > 30) {
      this._panel.setMode('floating');
      return;
    }
    const viewerBottom = top + viewerRect.height;
    this.style.left = `${left + viewerRect.width + 8}px`;
    this.style.top = `${viewerBottom - hostRect.height}px`;
    this.style.right = 'auto';
    this.style.bottom = 'auto';
  }
```

- [ ] **Step 9: Update `_trySnap` to use PanelController**

Replace direct `_snapViewerToController()` call with
`this._panel.setMode('docked')` (snap happens via onChange):

```typescript
  private _trySnap(): void {
    if (!this._yamlViewer || this._panelMode === 'docked') return;
    const hostRect = this.getBoundingClientRect();
    const viewerRect = this._yamlViewer.getBoundingClientRect();
    const gap = hostRect.left - (viewerRect.left + viewerRect.width);
    if (Math.abs(gap) < 30 && Math.abs(hostRect.bottom - viewerRect.bottom) < 30) {
      this._panel.setMode('docked');
    }
  }
```

- [ ] **Step 10: Update `_detachYaml` to use PanelController**

Replace `this._yamlOpen = false` with `this._panel.setMode('closed')`:

Find the line `this._yamlOpen = false;` in `_detachYaml` and replace
with `this._panel.setMode('closed');`. Also replace
`this._hideYamlViewer();` above it since `setMode('closed')` triggers
the onChange which calls `_hideYamlViewer()`. Keep only
`this._panel.setMode('closed');`.

- [ ] **Step 11: Update render template**

In `_renderCompactCard` (around line 607-611), replace `_yamlOpen` and
`_docked` references:

Replace:
```typescript
          <button aria-label="Toggle source" @click=${() => { this._toggleYaml(); }}>&lt;/&gt;</button>
          ${this._yamlOpen ? (this._docked
            ? html`<button aria-label="Undock viewer" @click=${() => { this._undockViewer(); }} title="Undock panels">⊟</button>`
            : html`<button aria-label="Dock viewer" @click=${() => { this._dockViewer(); }} title="Dock panels">⊞</button>`
          ) : nothing}
```
with:
```typescript
          <button aria-label="Toggle source" @click=${() => { this._toggleYaml(); }}>&lt;/&gt;</button>
          ${this._panelMode !== 'closed' ? (this._panelMode === 'docked'
            ? html`<button aria-label="Undock viewer" @click=${() => { this._undockViewer(); }} title="Undock panels">⊟</button>`
            : html`<button aria-label="Dock viewer" @click=${() => { this._dockViewer(); }} title="Dock panels">⊞</button>`
          ) : nothing}
```

- [ ] **Step 12: Run all tests**

Run: `yarn --cwd packages/pages-three-way-viewer test`
Run: `yarn --cwd packages/pages-aria test`

Expected: All pass.

- [ ] **Step 13: Commit**

```bash
git add packages/pages-aria/package.json \
       packages/pages-aria/src/controller/playbook-controller.ts \
       yarn.lock
git commit -m "feat(playbook-controller): adopt PanelController for floating viewer state Refs #750"
```

## References

- `specs/issue-750-adopt-viewer-controller/2026-11-26-adopt-viewer-controller-design.md` — design spec
- `packages/pages-three-way-viewer/src/viewer-controller.ts` — architectural pattern
- `packages/pages-aria/src/controller/playbook-controller.ts:190-201` — current state fields
- `packages/pages-aria/src/controller/playbook-controller.ts:661-746` — methods being simplified
- `packages/pages-aria/src/controller/playbook-controller.ts:598-612` — render template
- `packages/pages-aria/src/controller/playbook-yaml-viewer.ts` — viewer component (unchanged)
- GitHub #750 — focal issue
- `decisions.md D1` — PanelController approach decision
