# Selection-Based Visual Toolbar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #433 — Tree/visual structural editing
**Issue group:** #433

**Goal:** Replace hover-based per-component overlay toolbars with a selection-based toolbar on the scope outline.

**Architecture:** New `SelectionOverlay` class owns both the scope outline (blue border) and an expandable toolbar (trigger icon + 5 buttons). CSS-driven hover expansion — no JS mousemove tracking. Replaces both `ComponentOverlayManager` (deleted) and the inline scope overlay management in `_applyPreviewHighlight`.

**Tech Stack:** TypeScript, Vitest, DOM APIs

## Global Constraints

- All source files in `packages/pages-builder/src/`
- Tests use Vitest with `describe`/`it`/`expect`
- `TreeNodeType` imported from `../tree/builder-tree.js`
- CSS is inline (style strings) — no external stylesheets

---

## Batch 1: SelectionOverlay class with tests

### Task 1: SelectionOverlay — core rendering and events

**Files:**
- Create: `packages/pages-builder/src/overlay/selection-overlay.ts`
- Test: `packages/pages-builder/src/overlay/selection-overlay.test.ts`

**Interfaces:**
- Consumes: `TreeNodeType` from `../tree/builder-tree.js`
- Produces: `SelectionOverlay` class with `constructor(overlayRoot: HTMLElement)`, `update(bounds: DOMRect, nodeType: TreeNodeType, path: readonly (string | number)[], options?: { isContainer?: boolean }): void`, `hide(): void`, `setInsertMode(fragmentType: string): void`, `clearInsertMode(): void`, `dispose(): void`. Dispatches `selection-add`, `selection-insert`, `selection-cut`, `selection-copy`, `selection-delete` CustomEvents on `overlayRoot` with `detail: { path, nodeType }`.

- [ ] **Step 1: Write failing tests for SelectionOverlay**

Create `packages/pages-builder/src/overlay/selection-overlay.test.ts`:

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import { SelectionOverlay } from './selection-overlay.js';

describe('SelectionOverlay', () => {
  let overlayRoot: HTMLDivElement;
  let overlay: SelectionOverlay;

  afterEach(() => {
    overlay?.dispose();
    overlayRoot?.remove();
  });

  function setup(): void {
    overlayRoot = document.createElement('div');
    document.body.appendChild(overlayRoot);
    overlay = new SelectionOverlay(overlayRoot);
  }

  function makeBounds(x = 10, y = 20, w = 200, h = 100): DOMRect {
    return new DOMRect(x, y, w, h);
  }

  it('update renders scope outline at given bounds', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    const el = overlayRoot.querySelector('.builder-scope-overlay') as HTMLElement;
    expect(el).toBeTruthy();
    expect(el.style.display).not.toBe('none');
  });

  it('update renders toolbar trigger icon', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    const trigger = overlayRoot.querySelector('.toolbar-trigger');
    expect(trigger).toBeTruthy();
    expect(trigger!.textContent).toBe('⋮');
  });

  it('renders all 5 buttons for component node type', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0], { isContainer: true });
    expect(overlayRoot.querySelector('.sel-add-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-insert-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-cut-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-copy-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-delete-btn')).toBeTruthy();
  });

  it('omits add button for non-container component', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    expect(overlayRoot.querySelector('.sel-add-btn')).toBeNull();
    expect(overlayRoot.querySelector('.sel-insert-btn')).toBeTruthy();
  });

  it('omits insert/cut/copy for page node type', () => {
    setup();
    overlay.update(makeBounds(), 'page', ['pages', 0]);
    expect(overlayRoot.querySelector('.sel-add-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-insert-btn')).toBeNull();
    expect(overlayRoot.querySelector('.sel-cut-btn')).toBeNull();
    expect(overlayRoot.querySelector('.sel-copy-btn')).toBeNull();
    expect(overlayRoot.querySelector('.sel-delete-btn')).toBeTruthy();
  });

  it('shows all structural buttons for row node type', () => {
    setup();
    overlay.update(makeBounds(), 'row', ['pages', 0, 'rows', 0]);
    expect(overlayRoot.querySelector('.sel-add-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-insert-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-cut-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-copy-btn')).toBeTruthy();
    expect(overlayRoot.querySelector('.sel-delete-btn')).toBeTruthy();
  });

  it('dispatches selection-cut on cut button click', () => {
    setup();
    const path = ['pages', 0, 'components', 0] as const;
    overlay.update(makeBounds(), 'component', path);
    const events: CustomEvent[] = [];
    overlayRoot.addEventListener('selection-cut', ((e: CustomEvent) => events.push(e)) as EventListener);
    (overlayRoot.querySelector('.sel-cut-btn') as HTMLButtonElement).click();
    expect(events).toHaveLength(1);
    expect(events[0]!.detail.path).toEqual(path);
    expect(events[0]!.detail.nodeType).toBe('component');
  });

  it('dispatches selection-add on add button click', () => {
    setup();
    const path = ['pages', 0] as const;
    overlay.update(makeBounds(), 'page', path);
    const events: CustomEvent[] = [];
    overlayRoot.addEventListener('selection-add', ((e: CustomEvent) => events.push(e)) as EventListener);
    (overlayRoot.querySelector('.sel-add-btn') as HTMLButtonElement).click();
    expect(events).toHaveLength(1);
    expect(events[0]!.detail.nodeType).toBe('page');
  });

  it('hide makes overlay invisible', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    overlay.hide();
    const el = overlayRoot.querySelector('.builder-scope-overlay') as HTMLElement;
    expect(el.style.display).toBe('none');
  });

  it('setInsertMode adds paste-target class', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    overlay.setInsertMode('component');
    const el = overlayRoot.querySelector('.builder-scope-overlay') as HTMLElement;
    expect(el.classList.contains('paste-target')).toBe(true);
  });

  it('clearInsertMode removes paste-target class', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    overlay.setInsertMode('component');
    overlay.clearInsertMode();
    const el = overlayRoot.querySelector('.builder-scope-overlay') as HTMLElement;
    expect(el.classList.contains('paste-target')).toBe(false);
  });

  it('dispose removes overlay from DOM', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    expect(overlayRoot.querySelector('.builder-scope-overlay')).toBeTruthy();
    overlay.dispose();
    expect(overlayRoot.querySelector('.builder-scope-overlay')).toBeNull();
  });

  it('update reuses existing overlay element', () => {
    setup();
    overlay.update(makeBounds(10, 20, 200, 100), 'component', ['pages', 0, 'components', 0]);
    const el1 = overlayRoot.querySelector('.builder-scope-overlay');
    overlay.update(makeBounds(50, 60, 300, 150), 'row', ['pages', 0, 'rows', 0]);
    const el2 = overlayRoot.querySelector('.builder-scope-overlay');
    expect(el1).toBe(el2);
  });

  it('toolbar buttons have pointer-events auto', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    const toolbar = overlayRoot.querySelector('.selection-toolbar') as HTMLElement;
    expect(toolbar.style.pointerEvents).toBe('auto');
  });

  it('scope overlay is pointer-events none', () => {
    setup();
    overlay.update(makeBounds(), 'component', ['pages', 0, 'components', 0]);
    const el = overlayRoot.querySelector('.builder-scope-overlay') as HTMLElement;
    expect(el.style.pointerEvents).toBe('none');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-builder/src/overlay/selection-overlay.test.ts`
Expected: FAIL — module `./selection-overlay.js` not found

- [ ] **Step 3: Implement SelectionOverlay**

Create `packages/pages-builder/src/overlay/selection-overlay.ts`:

```typescript
import type { TreeNodeType } from '../tree/builder-tree.js';

type ButtonDef = readonly [cls: string, label: string, event: string];

const ALL_BUTTONS: ButtonDef[] = [
  ['sel-add-btn', '+', 'selection-add'],
  ['sel-insert-btn', '↓', 'selection-insert'],
  ['sel-cut-btn', '✂', 'selection-cut'],
  ['sel-copy-btn', '⎘', 'selection-copy'],
  ['sel-delete-btn', '✕', 'selection-delete'],
];

function getButtonsForNodeType(
  nodeType: TreeNodeType,
  isContainer: boolean,
): ButtonDef[] {
  if (nodeType === 'page') {
    return ALL_BUTTONS.filter(([, , evt]) =>
      evt === 'selection-add' || evt === 'selection-delete');
  }
  if (nodeType === 'component' && !isContainer) {
    return ALL_BUTTONS.filter(([, , evt]) => evt !== 'selection-add');
  }
  return ALL_BUTTONS;
}

const BTN_STYLE = `
  width: 20px; height: 20px; border: 1px solid #dadce0;
  border-radius: 4px; background: #fff; color: #5f6368;
  cursor: pointer; font-size: 12px; line-height: 1; padding: 0;
  display: flex; align-items: center; justify-content: center;
`;

export class SelectionOverlay {
  private _overlayRoot: HTMLElement;
  private _overlay: HTMLElement | null = null;

  constructor(overlayRoot: HTMLElement) {
    this._overlayRoot = overlayRoot;
  }

  update(
    bounds: DOMRect,
    nodeType: TreeNodeType,
    path: readonly (string | number)[],
    options?: { isContainer?: boolean },
  ): void {
    if (!this._overlay) {
      this._overlay = document.createElement('div');
      this._overlay.className = 'builder-scope-overlay';
      this._overlayRoot.appendChild(this._overlay);
    }

    this._overlay.style.cssText = `
      position: absolute; pointer-events: none;
      left: ${bounds.x}px; top: ${bounds.y}px;
      width: ${bounds.width}px; height: ${bounds.height}px;
      outline: 2px solid var(--pages-primary, #4285f4);
      outline-offset: 0;
      border-radius: 6px;
      background: rgba(66, 133, 244, 0.04);
      z-index: 10;
      transition: all 0.15s ease;
    `;

    this._overlay.innerHTML = '';
    const toolbar = document.createElement('div');
    toolbar.className = 'selection-toolbar';
    toolbar.style.cssText = `
      position: absolute; top: -2px; right: -2px;
      display: flex; align-items: center;
      pointer-events: auto; z-index: 11;
    `;

    const buttonsContainer = document.createElement('div');
    buttonsContainer.className = 'toolbar-buttons';
    buttonsContainer.style.cssText = `
      display: flex; gap: 2px;
      max-width: 0; overflow: hidden; opacity: 0;
      transition: max-width 150ms ease, opacity 150ms ease;
    `;

    const buttons = getButtonsForNodeType(nodeType, options?.isContainer ?? false);
    for (const [cls, label, eventName] of buttons) {
      const btn = document.createElement('button');
      btn.className = cls;
      btn.textContent = label;
      btn.style.cssText = BTN_STYLE;
      btn.addEventListener('click', (e) => {
        e.stopPropagation();
        this._overlayRoot.dispatchEvent(new CustomEvent(eventName, {
          bubbles: true,
          detail: { path, nodeType },
        }));
      });
      buttonsContainer.appendChild(btn);
    }

    const trigger = document.createElement('div');
    trigger.className = 'toolbar-trigger';
    trigger.textContent = '⋮';
    trigger.style.cssText = `
      width: 20px; height: 20px;
      display: flex; align-items: center; justify-content: center;
      cursor: pointer; font-size: 14px; font-weight: bold;
      color: #5f6368; background: #fff;
      border: 1px solid #dadce0; border-radius: 4px;
    `;

    toolbar.appendChild(buttonsContainer);
    toolbar.appendChild(trigger);

    toolbar.addEventListener('mouseenter', () => {
      buttonsContainer.style.maxWidth = '200px';
      buttonsContainer.style.opacity = '1';
    });
    toolbar.addEventListener('mouseleave', () => {
      buttonsContainer.style.maxWidth = '0';
      buttonsContainer.style.opacity = '0';
    });

    this._overlay.appendChild(toolbar);
  }

  hide(): void {
    if (this._overlay) this._overlay.style.display = 'none';
  }

  setInsertMode(_fragmentType: string): void {
    this._overlay?.classList.add('paste-target');
  }

  clearInsertMode(): void {
    this._overlay?.classList.remove('paste-target');
  }

  dispose(): void {
    this._overlay?.remove();
    this._overlay = null;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-builder/src/overlay/selection-overlay.test.ts`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-builder/src/overlay/selection-overlay.ts packages/pages-builder/src/overlay/selection-overlay.test.ts
git commit -m "feat(builder): add SelectionOverlay class with tests Refs #433"
```

## Batch 2: Wire SelectionOverlay into builder-shell

### Task 2: Replace ComponentOverlayManager with SelectionOverlay in builder-shell

**Files:**
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` — lines 22 (import), 142 (field), 277-314 (`_refreshPreview`), 354-405 (`_applyPreviewHighlight`, `_highlightPreviewNode`)
- Delete: `packages/pages-builder/src/overlay/component-overlay.ts`
- Delete: `packages/pages-builder/src/overlay/component-overlay.test.ts`

**Interfaces:**
- Consumes: `SelectionOverlay` from Task 1
- Produces: Updated builder-shell that uses SelectionOverlay. Overlay events (`selection-add`, `selection-insert`, `selection-cut`, `selection-copy`, `selection-delete`) wired to existing tree action handlers.

- [ ] **Step 1: Run existing builder-shell tests to confirm baseline**

Run: `yarn vitest run packages/pages-builder/src/shell/builder-shell.test.ts`
Expected: All PASS

- [ ] **Step 2: Change import — replace ComponentOverlayManager with SelectionOverlay**

In `packages/pages-builder/src/shell/builder-shell.ts`, replace line 22:

```typescript
// old:
import { ComponentOverlayManager } from '../overlay/component-overlay.js';
// new:
import { SelectionOverlay } from '../overlay/selection-overlay.js';
```

- [ ] **Step 3: Change field type**

In `packages/pages-builder/src/shell/builder-shell.ts`, replace line 142:

```typescript
// old:
private _overlayManager: ComponentOverlayManager | undefined;
// new:
private _selectionOverlay: SelectionOverlay | undefined;
```

- [ ] **Step 4: Rewrite _refreshPreview — remove ComponentOverlayManager creation**

Replace the `afterRender` closure in `_refreshPreview` (lines 283-298). Remove `ComponentOverlayManager` creation. Initialize `SelectionOverlay` if not yet created. After render, re-apply highlight if there's a selection:

```typescript
const afterRender = () => {
  this._previewClickCleanup?.();
  this._previewClickCleanup = this._wirePreviewClicks(container);
  const overlayRoot = this.shadowRoot?.querySelector('.overlay-root') as HTMLElement;
  if (overlayRoot && !this._selectionOverlay) {
    this._selectionOverlay = new SelectionOverlay(overlayRoot);
    this._wireSelectionOverlayEvents(overlayRoot);
  }
  if (this._selectedPath && this._selectedNodeType) {
    this._applyPreviewHighlight(this._selectedPath, this._selectedNodeType);
  }
};
```

- [ ] **Step 5: Rewrite _applyPreviewHighlight — delegate to SelectionOverlay**

Replace `_applyPreviewHighlight` (lines 359-405). Keep bounds computation, delegate rendering to `SelectionOverlay`:

```typescript
private _applyPreviewHighlight(path: readonly (string | number)[], nodeType: TreeNodeType): void {
  const container = this.shadowRoot?.querySelector('.preview-container') as HTMLElement;
  const overlayRoot = this.shadowRoot?.querySelector('.overlay-root') as HTMLElement;
  if (!container || !overlayRoot) return;

  if (!this._selectionOverlay) {
    this._selectionOverlay = new SelectionOverlay(overlayRoot);
    this._wireSelectionOverlayEvents(overlayRoot);
  }

  const scopeComps = collectComponentsInScope(this._document, path, nodeType);
  const renderedIndex = this._buildRenderedIndex(container);
  const elements: HTMLElement[] = [];
  for (const comp of scopeComps) {
    const el = renderedIndex.get(JSON.stringify(comp.path));
    if (el) elements.push(el);
  }

  if (elements.length === 0) {
    this._selectionOverlay.hide();
    return;
  }

  const containerRect = container.getBoundingClientRect();
  const rects = elements.map(el => el.getBoundingClientRect());
  const pad = 4;
  const minX = Math.min(...rects.map(r => r.left)) - containerRect.left - pad;
  const minY = Math.min(...rects.map(r => r.top)) - containerRect.top + container.scrollTop - pad;
  const maxX = Math.max(...rects.map(r => r.right)) - containerRect.left + pad;
  const maxY = Math.max(...rects.map(r => r.bottom)) - containerRect.top + container.scrollTop + pad;
  const bounds = new DOMRect(minX, minY, maxX - minX, maxY - minY);

  let isContainer = false;
  if (nodeType === 'component') {
    const comp = this._findComponentAtPath(path);
    isContainer = comp?.isContainer() ?? false;
  }

  this._selectionOverlay.update(bounds, nodeType, path, { isContainer });

  const clipboard = getClipboard();
  if (clipboard.insertMode && clipboard.fragmentType) {
    this._selectionOverlay.setInsertMode(clipboard.fragmentType);
  }

  elements[0]!.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
}
```

- [ ] **Step 6: Add _wireSelectionOverlayEvents method**

Add new method to builder-shell that listens for SelectionOverlay events and delegates to existing handlers:

```typescript
private _selectionOverlayCleanup: (() => void) | undefined;

private _wireSelectionOverlayEvents(overlayRoot: HTMLElement): void {
  this._selectionOverlayCleanup?.();

  const onAdd = (e: Event) => {
    const { path, nodeType } = (e as CustomEvent).detail;
    this._handleTreeAdd(new CustomEvent('tree-add', {
      detail: { path, nodeType },
    }));
  };
  const onInsert = (e: Event) => {
    const { path, nodeType } = (e as CustomEvent).detail;
    this._handleTreeInsert(new CustomEvent('tree-insert', {
      detail: { path, nodeType },
    }));
  };
  const onCut = (e: Event) => {
    const { path, nodeType } = (e as CustomEvent).detail;
    this._handleTreeCut(new CustomEvent('tree-cut', {
      detail: { path, nodeType },
    }));
  };
  const onCopy = (e: Event) => {
    const { path, nodeType } = (e as CustomEvent).detail;
    this._handleTreeCopy(new CustomEvent('tree-copy', {
      detail: { path, nodeType },
    }));
  };
  const onDelete = (e: Event) => {
    const { path, nodeType } = (e as CustomEvent).detail;
    this._applyEdit('toolbar', () => {
      this._deleteAtPath(path, nodeType);
    });
  };

  overlayRoot.addEventListener('selection-add', onAdd);
  overlayRoot.addEventListener('selection-insert', onInsert);
  overlayRoot.addEventListener('selection-cut', onCut);
  overlayRoot.addEventListener('selection-copy', onCopy);
  overlayRoot.addEventListener('selection-delete', onDelete);

  this._selectionOverlayCleanup = () => {
    overlayRoot.removeEventListener('selection-add', onAdd);
    overlayRoot.removeEventListener('selection-insert', onInsert);
    overlayRoot.removeEventListener('selection-cut', onCut);
    overlayRoot.removeEventListener('selection-copy', onCopy);
    overlayRoot.removeEventListener('selection-delete', onDelete);
  };
}
```

- [ ] **Step 7: Update disconnectedCallback — dispose SelectionOverlay**

In `disconnectedCallback`, add cleanup:

```typescript
this._selectionOverlay?.dispose();
this._selectionOverlayCleanup?.();
```

- [ ] **Step 8: Delete ComponentOverlayManager files**

Delete `packages/pages-builder/src/overlay/component-overlay.ts` and `packages/pages-builder/src/overlay/component-overlay.test.ts`.

- [ ] **Step 9: Run all builder tests**

Run: `yarn vitest run packages/pages-builder/`
Expected: All PASS. The overlay-root test (line 1061) should still pass — the overlay-root div is still rendered in the centre panel.

- [ ] **Step 10: Commit**

```bash
git add packages/pages-builder/src/shell/builder-shell.ts
git rm packages/pages-builder/src/overlay/component-overlay.ts packages/pages-builder/src/overlay/component-overlay.test.ts
git commit -m "feat(builder): wire SelectionOverlay, delete ComponentOverlayManager Refs #433"
```

## References

- [2026-09-18-selection-toolbar-design.md] — design spec this plan implements
- [overlay/component-overlay.ts] — current overlay class (deleted)
- [shell/builder-shell.ts:22,142,277-314,354-405] — import, field, _refreshPreview, _applyPreviewHighlight
- [shell/scope-collector.ts] — collectComponentsInScope (unchanged)
- [overlay/component-overlay.test.ts] — current overlay tests (deleted)
- [shell/builder-shell.test.ts:1061] — overlay-root existence test (unchanged)
- [GitHub #433] — Tree/visual structural editing
