# Floating/Popout Panels — Detach Panels into Separate Windows

**Issue:** casehubio/casehub-pages#77
**Date:** 2026-08-03
**Status:** Approved

## Goal

Detach any titled panel from the dashboard into a separate browser window. The panel keeps receiving live data updates from the main window's data pipeline. Closing the child window automatically returns the panel to its original layout position. Multiple panels can be detached simultaneously.

## Non-Goals

- Multi-monitor persistent layout (panels don't remember detach state across page loads)
- Cross-machine remote viewing (same browser only)
- Detaching non-panel components (bare charts, layout containers)

---

## 1. Approach — adoptNode

Use `document.adoptNode()` to transfer the panel's viz element from the main document to the child window's document. The JS object reference is preserved — the data pipeline's `ComponentRegistry` still holds the same object and can set `vizElement.dataSet` directly, with zero serialization.

Lit shadow DOM and `adoptedStyleSheets` travel with the element. CSS custom properties (design tokens) require copying the theme stylesheets to the child window.

### Why not BroadcastChannel / MessageChannel?

Those require serializing `TypedDataSet` on every update (structured clone). Garden entry GE-20260706-f2a9b2 documents that `Map` objects silently lose data in `postMessage` across window boundaries. adoptNode avoids the entire serialization problem — the element is the same JS object in a different document.

MessageChannel is the fallback if Lit's adoptNode behaviour proves broken.

---

## 2. DetachController

A class in `pages-runtime` managing the lifecycle of a single detached panel.

```typescript
class DetachController {
  readonly panelId: string;
  private sourceContainer: HTMLElement;
  private sourceIndex: number;
  private childWindow: Window | null;
  private vizElement: HTMLElement;
  private eventRelay: EventRelay;

  detach(): void;
  reattach(): void;
  dispose(): void;
}
```

**`detach()`:**
1. Record `sourceContainer` (the panel's parent `div[data-component-id]`) and `sourceIndex` (child position).
2. Open `window.open('', '_blank', 'width=800,height=600')`.
3. If `null` (popup blocked): emit toast notification, abort.
4. Copy theme stylesheets to child window via `copyStyles()`.
5. Set child window `<title>` to the panel title.
6. `childWindow.document.adoptNode(vizElement)` then `childWindow.document.body.appendChild(vizElement)`.
7. Hide `sourceContainer` in main layout, insert placeholder.
8. Start event relay.
9. Register `beforeunload` on child window → `reattach()`.

**`reattach()`:**
1. `document.adoptNode(vizElement)` back to main document.
2. Insert into `sourceContainer` at `sourceIndex`.
3. Un-hide container, remove placeholder.
4. Stop event relay.
5. Close child window if still open.

**`dispose()`:** Called on main window unload. Closes child window without redocking.

### DetachRegistry

```typescript
const detachRegistry = new Map<string, DetachController>();
```

Lives alongside `ComponentRegistry` in `pages-runtime`. API:

- `detachRegistry.has(id)` — double-detach guard
- `detachRegistry.get(id)` — access controller
- `detachRegistry.reattachAll()` — called on SPA page switch before DOM teardown

---

## 3. Event Relay

The detached element can't bubble events up the main window's DOM tree. `DetachController` bridges this gap.

**Relayed events:** `pages-filter`, `pages-sort`, `pages-data-request`, `pages-field-change`.

**Mechanism:** `DetachController` listens on `childWindow.document` for these event types. On capture, it re-dispatches a synthetic event with the same `detail` on `sourceContainer` in the main window. The data pipeline processes it as if it came from the panel in its original position.

**Direction:** One-way relay (child → main) for user-initiated events. Data flow (main → child) is handled by the existing pipeline — `vizElement.dataSet = dataset` works because the JS reference is preserved.

---

## 4. Style Transfer

### copyStyles(sourceDoc, targetDoc)

A utility that clones CSS from the main window to the child window. Runs once on detach.

**What it copies:**
1. All `<style>` elements from `<head>` (includes injected design tokens, `:root` custom properties).
2. All `<link rel="stylesheet">` elements from `<head>` (external stylesheets, fonts).

**What travels automatically (no action needed):**
- Lit `adoptedStyleSheets` on the shadow root — these are properties of the `ShadowRoot`, not the document, so they move with the element.

**Theme changes while detached:** Not synced. The panel uses the theme from detach-time. Acceptable for focus-mode use — redock picks up the current theme.

---

## 5. Panel Header Affordance

### Detach button

Added to the panel title bar (rendered in `renderNode()` for `type: "panel"` components). A `<button data-detach>` with ↗ icon, positioned right of the title text. Visible on hover over the panel header.

Click dispatches `pages-panel-detach` event (`{bubbles: true, composed: true}`, detail: `{componentId}`). The runtime listens for this and creates a `DetachController`.

### Detached state UI

**In child window:** The title bar shows a redock button (↙ icon) instead of detach. Click calls `controller.reattach()`.

**In main window:** The panel's container shows a placeholder: "Panel detached" with a "Focus" link that calls `childWindow.focus()`.

### Scope

Only `type: "panel"` components with a title. Non-panel components (bare charts, layout containers) do not get the affordance.

---

## 6. Lifecycle Edge Cases

| Scenario | Behaviour |
|----------|-----------|
| Child window closed | `beforeunload` → `reattach()` → panel returns to layout |
| Main window closed/navigated | `beforeunload` → `DetachRegistry.dispose()` all → child windows closed |
| SPA page switch (`loadSite()`) | `DetachRegistry.reattachAll()` before DOM teardown |
| Double-detach click | `detachRegistry.has(id)` → focus existing child window |
| Popup blocked | `window.open()` returns `null` → toast notification, no detach |
| Child window resize | Panel fills viewport (`width: 100%; height: 100vh`), resizes naturally |

---

## 7. File Plan

| File | Package | What |
|------|---------|------|
| `src/detach/detach-controller.ts` | pages-runtime | DetachController class |
| `src/detach/detach-registry.ts` | pages-runtime | DetachRegistry map + reattachAll |
| `src/detach/event-relay.ts` | pages-runtime | Cross-window event relay |
| `src/detach/copy-styles.ts` | pages-runtime | Style cloning utility |
| `src/detach/index.ts` | pages-runtime | Barrel export |
| `src/activation.ts` | pages-runtime | Wire `pages-panel-detach` listener |
| `src/site.ts` | pages-runtime | Call `reattachAll()` on teardown |
| `render.ts` | pages-component | Add detach button to panel title bar |

---

## 8. Testing Strategy

**Unit tests (Vitest + jsdom):**
- `DetachController` — detach/reattach lifecycle, sourceIndex preservation, dispose cleanup
- `copyStyles` — clones styles, handles empty head
- Event relay — captures and re-dispatches correct events with detail

**Limitation:** `window.open()` is not available in jsdom. Integration tests for the full detach flow require a browser environment (Playwright, future #94). Unit tests mock `window.open()` to return a fake window object with a minimal document.

**Manual verification:** Detach a panel, confirm data updates flow, close window, confirm panel returns. Test with multiple simultaneous detaches.

---

## 9. Risk: Lit adoptNode Behaviour

**Risk:** Lit may not cleanly handle `disconnectedCallback` → `connectedCallback` across document adoption. Reactive properties, scheduled updates, or `requestUpdate()` could break.

**Mitigation:** Spike test before implementation — create a minimal Lit element, adopt it into a new document, set a reactive property, verify it re-renders. If broken: fall back to Approach B (proxy panel + MessageChannel).

**Spike scope:** ~30 minutes, single test file. Gate for proceeding with implementation.
