# Floating/Popout Panels — Detach Panels into Separate Windows

**Issue:** casehubio/casehub-pages#77
**Date:** 2026-08-03
**Status:** Approved (revised after light design review)

## Goal

Detach any panel from the dashboard into a separate browser window. The panel keeps receiving live data updates from the main window's data pipeline. Closing the child window automatically returns the panel to its original layout position. Multiple panels can be detached simultaneously.

## Non-Goals

- Multi-monitor persistent layout (panels don't remember detach state across page loads)
- Cross-machine remote viewing (same browser only)
- Detaching bare charts or layout containers without a panel wrapper

---

## 1. Approach — adoptNode on the container

Use `document.adoptNode()` to transfer the panel's **container element** (`div[data-component-id]`) — not the vizElement — from the main document to the child window's document. The container includes the title bar, the viz element, and any nested children.

**Why the container, not the vizElement:**
- `ComponentEntry.vizElement` is typed as `VizTarget` — an interface, not necessarily a DOM element. For `host-panel` with a lookup, it's a proxy object (`createHostPanelProxy()`). Adopting a proxy would fail.
- Adopting the container preserves the title bar in the child window (no need to recreate it).
- The viz element stays connected inside its container throughout the adoption — it is never removed from its parent. This avoids triggering MutationObserver-based cleanup (see §1.1).
- Enables host-panel detach (the container holds the custom element regardless of whether a vizElement proxy exists).

The JS object references in `ComponentRegistry` are preserved — the data pipeline can still set `vizElement.dataSet` directly with zero serialization.

Lit shadow DOM and `adoptedStyleSheets` travel with the element. CSS custom properties (design tokens) require copying the theme stylesheets to the child window.

### 1.1 MutationObserver guard

The data pipeline uses a MutationObserver to detect component unmount and unsubscribe push sources (WebSocket/SSE) when a component is removed from the DOM. `adoptNode` removes the container from the main document before inserting it into the child document — the observer may fire in between, interpreting the adoption as unmount.

**Guard strategy:** Before calling `adoptNode`, set a `data-detaching` attribute on the container. The MutationObserver callback checks for this attribute and skips cleanup when present. After successful adoption into the child window, remove the attribute. On reattach, set the attribute again before adopting back.

**Assumption A2:** `adoptNode()` + `appendChild()` completes synchronously before any queued microtask from MutationObserver callbacks runs. If this assumption fails (browser-specific), the guard attribute is the fallback regardless.

### Why not BroadcastChannel / MessageChannel?

Those require serializing `TypedDataSet` on every update (structured clone). Garden entry GE-20260706-f2a9b2 documents that `Map` objects silently lose data in `postMessage` across window boundaries. adoptNode avoids the entire serialization problem — the element is the same JS object in a different document.

MessageChannel is the fallback if Lit's adoptNode behaviour proves broken.

---

## 2. DetachController

A class in `pages-runtime` managing the lifecycle of a single detached panel.

```typescript
class DetachController {
  readonly componentId: string;
  private container: HTMLElement;
  private placeholder: HTMLElement;
  private childWindow: Window | null;
  private eventRelay: EventRelay;

  detach(): void;
  reattach(): void;
  dispose(): void;
}
```

**`detach()`:**
1. Locate the container: `div[data-component-id="${componentId}"]`.
2. Insert a placeholder element (`<div data-detach-placeholder="${componentId}">`) immediately before the container in the DOM. The placeholder marks the exact position for redock — no index tracking needed.
3. Open `window.open('', '_blank', 'width=800,height=600')`.
4. If `null` (popup blocked): remove placeholder, emit toast notification, abort.
5. Copy theme stylesheets to child window via `copyStyles()`.
6. Set child window `<title>` to the panel title.
7. Set `data-detaching` attribute on container (MutationObserver guard).
8. `childWindow.document.adoptNode(container)` then `childWindow.document.body.appendChild(container)`.
9. Remove `data-detaching` attribute.
10. Style placeholder in main layout: "Panel detached" with a "Focus" link.
11. Start event relay.
12. Register `beforeunload` on child window → `reattach()`.
13. Register polling fallback (see §6).

**`reattach()`:**
1. Stop event relay.
2. Set `data-detaching` attribute on container.
3. `document.adoptNode(container)` back to main document.
4. Insert container before `placeholder` in the DOM.
5. Remove placeholder.
6. Remove `data-detaching` attribute.
7. Close child window if still open.
8. Move focus to the reattached panel container.

**`dispose()`:** Called on main window unload. Reattaches if possible (element still in child window), otherwise closes child window. See §6 for ordering.

### DetachRegistry

```typescript
class DetachRegistry {
  private controllers = new Map<string, DetachController>();

  has(id: string): boolean;
  get(id: string): DetachController | undefined;
  register(id: string, controller: DetachController): void;
  remove(id: string): void;
  reattachAll(): void;
  disposeAll(): void;
}
```

Lives alongside `ComponentRegistry` in `pages-runtime`. Coordinates with `ComponentRegistry` disposal — see §6.

---

## 3. Event Relay

The detached container can't bubble events up the main window's DOM tree. `DetachController` bridges this gap.

**Relayed events (complete list — all events delegated via `target` in `loadSite()`):**

| Event | Purpose |
|-------|---------|
| `pages-filter` | Cross-component filtering |
| `pages-sort` | Column sort |
| `pages-data-request` | Data pipeline subscription |
| `pages-field-change` | Form field value change |
| `pages-page` | Pagination |
| `pages-text-filter` | Text search filtering |
| `pages-record-navigate` | Prev/next record navigation |
| `pages-record-create` | Record creation |
| `pages-record-delete` | Record deletion |
| `pages-action-request` | Action button click |
| `pages-refresh-request` | Manual data refresh |
| `pages-slot-change` | Tab/interactive container switch |
| `pages-dock-toggle` | Dock panel toggle |
| `pages-split-resize` | Split pane resize |
| `pages-event` | Inter-panel communication |

**Mechanism:** `DetachController` listens on `childWindow.document` for all listed event types. On capture, it creates a new `CustomEvent` with the same `type` and `detail`, and dispatches it on the `placeholder` element in the main window. The placeholder is in the original DOM position, so the event bubbles to `target` and the data pipeline processes it normally.

**Direction:** One-way relay (child → main) for user-initiated events. Data flow (main → child) is handled by the existing pipeline — `vizElement.dataSet = dataset` works because the JS reference is preserved.

**Excluded:** `pages-panel-detach` — consumed by the detach system itself, not relayed.

---

## 4. Style Transfer

### copyStyles(sourceDoc, targetDoc)

A utility that clones CSS from the main window to the child window. Runs once on detach.

**What it copies:**
1. All `<style>` elements from `<head>` (includes injected design tokens, `:root` custom properties).
2. All `<link rel="stylesheet">` elements from `<head>` (external stylesheets, fonts).

**What travels automatically (no action needed):**
- Lit `adoptedStyleSheets` on the shadow root — these are properties of the `ShadowRoot`, not the document, so they move with the element.

### Theme changes while detached

Theme sync is partial by default:
- `setTheme()` in `site.ts` sets `vizEl.theme` directly on each vizElement via `ComponentRegistry` — this **does** reach detached panels because the JS reference is preserved. ECharts re-renders with the new theme.
- `applyTheme()` sets `:root` CSS custom properties on `target` — this **does not** cascade to the child window's document tree. CSS token-based styling (colours, spacing) goes stale.

**Decision: full sync.** When `setTheme()` runs, if `DetachRegistry` has active controllers, call `copyStyles()` on each child window to refresh the CSS tokens. This is cheap (runs only on theme change, not on every data update) and prevents the confusing state where chart colours update but surrounding UI doesn't.

---

## 5. Panel Header Affordance

### Detach button

Added to the panel title bar for both `type: "panel"` and `type: "host-panel"` components. Rendered by the activation callback (not `renderNode()` in pages-component — avoids coupling the renderer to window management).

The activation callback in `site.ts`, when processing a panel or host-panel component, appends a `<button data-detach>` with ↗ icon to the title element. Positioned absolute right, visible on hover.

Click dispatches `pages-panel-detach` event (`{bubbles: true, composed: true}`, detail: `{componentId}`). The `loadSite()` event delegation in `site.ts` listens for this and creates a `DetachController`.

### Detached state UI

**In child window:** The title bar shows a redock button (↙ icon) instead of detach. Click calls `controller.reattach()`.

**In main window:** The placeholder shows "Panel detached" with a "Focus" link that calls `childWindow.focus()`.

### Scope

`type: "panel"` with a title, and `type: "host-panel"`. The container-adoption approach works for both — host-panel's proxy vizElement is never touched directly by adoptNode.

Non-panel components (bare charts, layout containers) do not get the affordance.

---

## 6. Lifecycle Edge Cases

| Scenario | Behaviour |
|----------|-----------|
| Child window closed | `beforeunload` → `reattach()` → panel returns to layout, focus moves to panel |
| `beforeunload` unreliable | Polling fallback: `setInterval` every 500ms checks `childWindow.closed`. If true and controller still active, call `reattach()`. Cleared on normal reattach/dispose. |
| Main window closed/navigated | `beforeunload` → `DetachRegistry.reattachAll()` first (saves panel state), then `disposeAll()` closes any remaining child windows |
| SPA page switch | `dispose()` in `site.ts` calls `DetachRegistry.reattachAll()` before `target.innerHTML = ""`. Panels return to their containers before the DOM is torn down. |
| Double-detach click | `detachRegistry.has(id)` → focus existing child window |
| Popup blocked | `window.open()` returns `null` → toast notification, placeholder removed, no detach |
| Child window resize | Panel fills viewport (`width: 100%; height: 100vh`), resizes naturally |

### Disposal ordering

On main window `beforeunload`:
1. `DetachRegistry.reattachAll()` — adopt all panels back to main document (preserves state for any session persistence).
2. `DetachRegistry.disposeAll()` — close any child windows that remain (edge case: reattach failed).
3. Normal `ComponentRegistry` disposal proceeds — all elements are back in the main document.

This order ensures `ComponentRegistry.dispose()` never encounters elements in foreign documents.

### Accessibility

**On detach:** Focus moves to the child window (via `childWindow.focus()`). The child window's document has `role="dialog"` and `aria-label` set to the panel title.

**On redock:** Focus moves to the reattached panel container (via `container.focus()` with `tabindex="-1"`). An `aria-live="polite"` region announces "Panel restored to dashboard."

---

## 7. File Plan

| File | Package | What |
|------|---------|------|
| `src/detach/detach-controller.ts` | pages-runtime | DetachController class |
| `src/detach/detach-registry.ts` | pages-runtime | DetachRegistry class |
| `src/detach/event-relay.ts` | pages-runtime | Cross-window event relay |
| `src/detach/copy-styles.ts` | pages-runtime | Style cloning utility |
| `src/detach/index.ts` | pages-runtime | Barrel export |
| `src/site.ts` | pages-runtime | Wire `pages-panel-detach` listener in `loadSite()`, call `reattachAll()` in `dispose()`, theme sync hook in `setTheme()` |
| `src/activation.ts` | pages-runtime | Append detach button to panel/host-panel title bars |

No changes to `pages-component/render.ts` — the detach button is wired in the activation callback, keeping window management out of the renderer.

---

## 8. Testing Strategy

**Unit tests (Vitest + jsdom):**
- `DetachController` — detach/reattach lifecycle, placeholder insertion/removal, dispose cleanup, MutationObserver guard attribute
- `DetachRegistry` — register/remove, reattachAll ordering, disposeAll
- `copyStyles` — clones styles and links, handles empty head
- Event relay — captures all 15 event types and re-dispatches with correct detail on placeholder

**Limitation:** `window.open()` is not available in jsdom. Unit tests mock `window.open()` to return a fake window object with a minimal document. Full integration tests require Playwright (future #94).

**Manual verification:** Detach a panel, confirm data updates flow, apply a filter, confirm it reaches the main pipeline, close window, confirm panel returns to original position. Test with multiple simultaneous detaches. Test theme switch while detached.

---

## 9. Risks

### 9.1 Lit adoptNode behaviour

**Risk:** Lit may not cleanly handle `disconnectedCallback` → `connectedCallback` across document adoption. Reactive properties, scheduled updates, or `requestUpdate()` could break.

**Mitigation:** Spike test before implementation — create a minimal Lit element, adopt its parent container into a new document, set a reactive property, verify it re-renders. If broken: fall back to Approach B (proxy panel + MessageChannel).

**Spike scope:** ~30 minutes, single test file. Gate for proceeding with implementation.

### 9.2 CSP restrictions

**Risk:** `window.open('')` creates an `about:blank` page. Some strict CSP policies may block `window.open()` or prevent injecting styles via `copyStyles()`.

**Mitigation:** If CSP blocks the blank page approach, serve a minimal `/detach.html` from the app's static assets — an empty page with just a `<body>` container. `copyStyles()` uses `cloneNode()` on existing stylesheet elements rather than inline script injection, which is CSP-safe.

---

## 10. Review Findings Log

Light design review (2026-08-03) surfaced 34 issues across coherence, structure, robustness, and cross-cutting dimensions. Key revisions:

- **Adopt container, not vizElement** — resolves type mismatch (VizTarget is not always DOM), enables host-panel support, avoids MutationObserver false-unmount
- **Placeholder replaces sourceIndex** — immune to sibling insertion/removal between detach and redock
- **Complete event relay list** — 15 events, matching `site.ts` delegation
- **MutationObserver guard** — `data-detaching` attribute prevents false unsubscription
- **Disposal ordering** — reattachAll before disposeAll, before ComponentRegistry disposal
- **Full theme sync** — copyStyles on theme change, not just at detach time
- **Scope expanded to host-panel** — container adoption makes this free
- **Detach button in activation.ts, not render.ts** — keeps pages-component free of window management
- **Event listener in site.ts loadSite()** — consistent with all other event delegation
- **beforeunload polling fallback** — 500ms interval catches cases where beforeunload doesn't fire
- **Accessibility** — focus management and aria-live announcement on detach/redock
- **CSP note** — fallback to served blank page if about:blank is blocked
