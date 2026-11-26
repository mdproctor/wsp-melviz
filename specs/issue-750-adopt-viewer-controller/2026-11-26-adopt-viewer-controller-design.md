# Adopt ViewerController in playbook controller

**Issue:** casehubio/casehub-pages#750
**Date:** 2026-11-26

## Summary

Create a `PanelController` in `pages-three-way-viewer` that manages
floating panel state (`closed`/`docked`/`floating`) following the same
architecture as `ViewerController`. The playbook controller adopts it
to replace ~60 lines of ad-hoc boolean state management.

## PanelController

A new class in `pages-three-way-viewer`, exported from `index.ts`.

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

  get mode(): PanelMode { return this._mode; }

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

**Design rationale:** Same pattern as ViewerController — constructor
with options, immutable after construction (onChange can't change),
no-op on same-state transitions, onChange notification. `toggle()`
always reopens as `docked` (matching current `_showYamlViewer`
behavior which sets `_docked = true`).

**Not included:** `cycle()` — the playbook controller's state
transitions are toggle + dock/undock, not a linear cycle. `splitRatio`
— not applicable to floating panels.

## Playbook controller changes

### State fields

**Remove:**
- `@state() private _yamlOpen = false;` (line 190)
- `@state() private _docked = true;` (line 201)

**Add:**
- `private _panel: PanelController;` — created in constructor or
  field initializer
- `@state() private _panelMode: PanelMode = 'closed';` — reactive
  mirror of controller state for Lit re-renders

The PanelController's `onChange` callback sets `_panelMode` and
triggers DOM side effects:

```typescript
private _panel = new PanelController({
  onChange: (mode) => {
    this._panelMode = mode;
    if (mode === 'closed') {
      this._hideYamlViewer();
    } else if (mode === 'docked') {
      this._ensureViewerCreated();
      this._yamlViewer!.style.display = 'block';
      requestAnimationFrame(() => {
        requestAnimationFrame(() => { this._snapViewerToController(); });
      });
    } else {
      this._ensureViewerCreated();
      this._yamlViewer!.style.display = 'block';
    }
  },
});
```

### Method changes

| Current method | After | Notes |
|---|---|---|
| `_toggleYaml()` | `this._panel.toggle()` | One-liner, inline at call site |
| `_showYamlViewer()` | Split: viewer creation → `_ensureViewerCreated()`, visibility/positioning → onChange | Creation is lazy and idempotent |
| `_hideYamlViewer()` | Unchanged — DOM operation called by onChange | |
| `_snapViewerToController()` | Unchanged — pure DOM positioning | |
| `_undockViewer()` | `this._panel.setMode('floating')` | One-liner, inline at call site |
| `_dockViewer()` | `this._panel.setMode('docked')` | One-liner, inline at call site |
| `_onViewerDrag()` | Calls `this._panel.setMode('floating')` instead of `this._docked = false` | |
| `_trySnap()` | Calls `this._panel.setMode('docked')` instead of `this._snapViewerToController()` | Snap positioning handled by onChange |
| `_onViewerDragEnd()` | Unchanged | |

**New method:** `_ensureViewerCreated()` — extracted from `_showYamlViewer()`.
Creates the viewer DOM element if it doesn't exist, sets properties and
callbacks. Idempotent.

### Render template changes

```typescript
// Before
${this._yamlOpen ? (this._docked
  ? html`<button aria-label="Undock viewer" ...>⊟</button>`
  : html`<button aria-label="Dock viewer" ...>⊞</button>`
) : nothing}

// After
${this._panelMode !== 'closed' ? (this._panelMode === 'docked'
  ? html`<button aria-label="Undock viewer" @click=${() => this._panel.setMode('floating')}>⊟</button>`
  : html`<button aria-label="Dock viewer" @click=${() => this._panel.setMode('docked')}>⊞</button>`
) : nothing}
```

### `_detachYaml()` — unchanged

Detach (pop-out window) is a separate concern: it closes the floating
panel via `this._panel.setMode('closed')` and opens a browser window.
The pop-out lifecycle (`_popoutWindow`, `_popoutPoll`) stays as-is.

### `disconnectedCallback()` — unchanged

Cleanup of viewer element and pop-out poll timer stays. No PanelController
cleanup needed — it has no subscriptions or timers.

## Package changes

`pages-three-way-viewer/src/index.ts` — export `PanelController`,
`PanelMode`, `PanelControllerOptions`.

`pages-aria` already depends on `pages-three-way-viewer` (check
package.json; if not, add `"@casehubio/pages-three-way-viewer": "workspace:^"`).

## Testing

### PanelController unit tests (new file)

- `setMode` changes mode and fires onChange
- `setMode` with same mode is a no-op (onChange not called)
- `toggle` from closed → docked
- `toggle` from docked → closed
- `toggle` from floating → closed
- Default initial mode is `closed`
- Custom initial mode is respected

### Playbook controller

Existing tests continue to pass. No new integration tests needed —
the behavior is identical, only the internal state management changes.

## References

- `packages/pages-three-way-viewer/src/viewer-controller.ts` — architectural pattern
- `packages/pages-aria/src/controller/playbook-controller.ts:190-201` — current state fields
- `packages/pages-aria/src/controller/playbook-controller.ts:598-612` — render template using _yamlOpen/_docked
- `packages/pages-aria/src/controller/playbook-controller.ts:661-746` — 8 methods being simplified
- `packages/pages-aria/src/controller/playbook-yaml-viewer.ts:169` — viewer's own mode property
- `decisions.md D1` — approach decision
- GitHub #750 — focal issue
- GitHub examples#217 — originating three-way-viewer work
