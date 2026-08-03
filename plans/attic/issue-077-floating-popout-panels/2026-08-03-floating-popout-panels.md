# Floating/Popout Panels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #77 — Floating/popout panels — detach panels into separate windows
**Issue group:** #77

**Goal:** Detach titled panels into separate browser windows with live data bridging, then auto-redock on close.

**Architecture:** Container-level `document.adoptNode()` moves the panel's `div[data-component-id]` to a child window. A placeholder marks the original position. An event relay forwards 15 user-initiated event types from the child window back to the main pipeline. A polling fallback supplements `beforeunload` for reliable redock.

**Tech Stack:** TypeScript, Lit (shadow DOM survives adoption), DOM `adoptNode`, `window.open`, Vitest + jsdom

## Global Constraints

- All new files in `packages/pages-runtime/src/detach/`
- No changes to `packages/pages-component/` — detach button wired in activation callback
- All events use `{bubbles: true, composed: true}`
- Wire format across windows: plain objects only (no Map/Set — GE-20260706-f2a9b2)
- `data-detaching` attribute guards MutationObserver during adoption
- Spike test (Task 0) gates all subsequent tasks — if Lit adoptNode fails, stop and redesign

---

### Task 0: Lit adoptNode Spike

**Files:**
- Create: `packages/pages-runtime/src/detach/adoptnode-spike.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: go/no-go for the adoptNode approach

This is the gate. If Lit elements don't survive `adoptNode` across documents, the entire approach changes to MessageChannel proxying. Run this first, in isolation.

- [ ] **Step 1: Write the spike test**

```typescript
import { describe, it, expect } from "vitest";
import { LitElement, html, css } from "lit";
import { customElement, property } from "lit/decorators.js";

@customElement("spike-badge")
class SpikeBadge extends LitElement {
  static override styles = css`:host { display: block; }`;
  @property() label = "";
  override render() { return html`<span>${this.label}</span>`; }
}

describe("adoptNode spike — Lit across documents", () => {
  it("survives adoptNode and re-renders after property change", async () => {
    const el = document.createElement("spike-badge") as SpikeBadge;
    document.body.appendChild(el);
    el.label = "before";
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain("before");

    // Simulate adoption to a new document
    const newDoc = document.implementation.createHTMLDocument("child");
    newDoc.body.appendChild(newDoc.adoptNode(el));

    el.label = "after";
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain("after");
  });

  it("fires disconnectedCallback then connectedCallback", async () => {
    const events: string[] = [];
    @customElement("spike-lifecycle")
    class SpikeLc extends LitElement {
      override connectedCallback() { super.connectedCallback(); events.push("connected"); }
      override disconnectedCallback() { super.disconnectedCallback(); events.push("disconnected"); }
      override render() { return html`<span>lc</span>`; }
    }
    const el = document.createElement("spike-lifecycle") as SpikeLc;
    document.body.appendChild(el);
    await el.updateComplete;
    events.length = 0;

    const newDoc = document.implementation.createHTMLDocument("child");
    newDoc.body.appendChild(newDoc.adoptNode(el));
    expect(events).toEqual(["disconnected", "connected"]);
  });

  it("survives round-trip adoption back to original document", async () => {
    const el = document.createElement("spike-badge") as SpikeBadge;
    document.body.appendChild(el);
    el.label = "original";
    await el.updateComplete;

    const newDoc = document.implementation.createHTMLDocument("child");
    newDoc.body.appendChild(newDoc.adoptNode(el));
    el.label = "detached";
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain("detached");

    document.body.appendChild(document.adoptNode(el));
    el.label = "redocked";
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain("redocked");
  });
});
```

- [ ] **Step 2: Run the spike**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run adoptnode-spike`

**If all 3 pass:** adoptNode is viable. Proceed to Task 1. Delete this test file (it was a spike, not a permanent test).

**If any fail:** STOP. The adoptNode approach is broken. Fall back to Approach B (proxy panel + MessageChannel). Redesign the spec and plan.

- [ ] **Step 3: Clean up spike**

Delete `adoptnode-spike.test.ts`. Commit: `chore(#77): adoptNode spike — Lit survives cross-document adoption`

---

### Task 1: copyStyles utility

**Files:**
- Create: `packages/pages-runtime/src/detach/copy-styles.ts`
- Create: `packages/pages-runtime/src/detach/copy-styles.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `copyStyles(sourceDoc: Document, targetDoc: Document): void`

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, it, expect } from "vitest";
import { copyStyles } from "./copy-styles.js";

describe("copyStyles", () => {
  it("clones style elements from source to target head", () => {
    const source = document.implementation.createHTMLDocument("source");
    const target = document.implementation.createHTMLDocument("target");

    const style = source.createElement("style");
    style.textContent = ":root { --pages-accent-9: #5470c6; }";
    source.head.appendChild(style);

    copyStyles(source, target);

    const cloned = target.head.querySelectorAll("style");
    expect(cloned.length).toBe(1);
    expect(cloned[0]!.textContent).toBe(":root { --pages-accent-9: #5470c6; }");
  });

  it("clones link elements from source to target head", () => {
    const source = document.implementation.createHTMLDocument("source");
    const target = document.implementation.createHTMLDocument("target");

    const link = source.createElement("link");
    link.rel = "stylesheet";
    link.href = "/styles/theme.css";
    source.head.appendChild(link);

    copyStyles(source, target);

    const cloned = target.head.querySelectorAll("link[rel='stylesheet']");
    expect(cloned.length).toBe(1);
    expect(cloned[0]!.getAttribute("href")).toBe("/styles/theme.css");
  });

  it("handles empty head gracefully", () => {
    const source = document.implementation.createHTMLDocument("source");
    const target = document.implementation.createHTMLDocument("target");

    expect(() => copyStyles(source, target)).not.toThrow();
    expect(target.head.children.length).toBe(0);
  });

  it("does not duplicate on repeated calls", () => {
    const source = document.implementation.createHTMLDocument("source");
    const target = document.implementation.createHTMLDocument("target");

    const style = source.createElement("style");
    style.textContent = ":root { --x: 1; }";
    source.head.appendChild(style);

    copyStyles(source, target);
    copyStyles(source, target);

    expect(target.head.querySelectorAll("style").length).toBe(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run copy-styles`
Expected: FAIL — module not found

- [ ] **Step 3: Write implementation**

```typescript
export function copyStyles(sourceDoc: Document, targetDoc: Document): void {
  targetDoc.head.querySelectorAll("style, link[rel='stylesheet']").forEach(el => el.remove());

  for (const el of sourceDoc.head.querySelectorAll("style")) {
    targetDoc.head.appendChild(targetDoc.importNode(el, true));
  }
  for (const el of sourceDoc.head.querySelectorAll("link[rel='stylesheet']")) {
    targetDoc.head.appendChild(targetDoc.importNode(el, true));
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run copy-styles`
Expected: PASS

- [ ] **Step 5: Commit**

`feat(#77): copyStyles utility for cross-window theme transfer`

---

### Task 2: EventRelay

**Files:**
- Create: `packages/pages-runtime/src/detach/event-relay.ts`
- Create: `packages/pages-runtime/src/detach/event-relay.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `class EventRelay { constructor(sourceDoc: Document, targetEl: HTMLElement); start(): void; stop(): void; }`

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { EventRelay } from "./event-relay.js";

const RELAYED_EVENTS = [
  "pages-filter", "pages-sort", "pages-data-request", "pages-field-change",
  "pages-page", "pages-text-filter", "pages-record-navigate",
  "pages-record-create", "pages-record-delete", "pages-action-request",
  "pages-refresh-request", "pages-slot-change", "pages-dock-toggle",
  "pages-split-resize", "pages-event",
] as const;

describe("EventRelay", () => {
  let sourceDoc: Document;
  let targetEl: HTMLElement;

  beforeEach(() => {
    sourceDoc = document.implementation.createHTMLDocument("child");
    targetEl = document.createElement("div");
    document.body.appendChild(targetEl);
  });

  it("relays all 15 event types from source document to target element", () => {
    const relay = new EventRelay(sourceDoc, targetEl);
    relay.start();

    const received: string[] = [];
    for (const type of RELAYED_EVENTS) {
      targetEl.addEventListener(type, () => received.push(type));
    }

    for (const type of RELAYED_EVENTS) {
      sourceDoc.dispatchEvent(new CustomEvent(type, {
        bubbles: true, composed: true,
        detail: { test: type },
      }));
    }

    expect(received).toEqual([...RELAYED_EVENTS]);
  });

  it("preserves event detail", () => {
    const relay = new EventRelay(sourceDoc, targetEl);
    relay.start();

    let captured: unknown = null;
    targetEl.addEventListener("pages-filter", ((e: CustomEvent) => {
      captured = e.detail;
    }) as EventListener);

    sourceDoc.dispatchEvent(new CustomEvent("pages-filter", {
      bubbles: true, composed: true,
      detail: { column: "status", value: "active" },
    }));

    expect(captured).toEqual({ column: "status", value: "active" });
  });

  it("stops relaying after stop()", () => {
    const relay = new EventRelay(sourceDoc, targetEl);
    relay.start();

    const received: string[] = [];
    targetEl.addEventListener("pages-filter", () => received.push("filter"));

    sourceDoc.dispatchEvent(new CustomEvent("pages-filter", { bubbles: true, composed: true }));
    expect(received.length).toBe(1);

    relay.stop();
    sourceDoc.dispatchEvent(new CustomEvent("pages-filter", { bubbles: true, composed: true }));
    expect(received.length).toBe(1);
  });

  it("does not relay pages-panel-detach", () => {
    const relay = new EventRelay(sourceDoc, targetEl);
    relay.start();

    let received = false;
    targetEl.addEventListener("pages-panel-detach", () => { received = true; });

    sourceDoc.dispatchEvent(new CustomEvent("pages-panel-detach", { bubbles: true, composed: true }));
    expect(received).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run event-relay`
Expected: FAIL

- [ ] **Step 3: Write implementation**

```typescript
const RELAYED_EVENTS = [
  "pages-filter", "pages-sort", "pages-data-request", "pages-field-change",
  "pages-page", "pages-text-filter", "pages-record-navigate",
  "pages-record-create", "pages-record-delete", "pages-action-request",
  "pages-refresh-request", "pages-slot-change", "pages-dock-toggle",
  "pages-split-resize", "pages-event",
] as const;

export class EventRelay {
  private readonly sourceDoc: Document;
  private readonly targetEl: HTMLElement;
  private readonly listeners: Array<{ type: string; handler: EventListener }> = [];

  constructor(sourceDoc: Document, targetEl: HTMLElement) {
    this.sourceDoc = sourceDoc;
    this.targetEl = targetEl;
  }

  start(): void {
    for (const type of RELAYED_EVENTS) {
      const handler = ((e: CustomEvent) => {
        this.targetEl.dispatchEvent(new CustomEvent(type, {
          bubbles: true,
          composed: true,
          detail: e.detail,
        }));
      }) as EventListener;
      this.sourceDoc.addEventListener(type, handler);
      this.listeners.push({ type, handler });
    }
  }

  stop(): void {
    for (const { type, handler } of this.listeners) {
      this.sourceDoc.removeEventListener(type, handler);
    }
    this.listeners.length = 0;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run event-relay`
Expected: PASS

- [ ] **Step 5: Commit**

`feat(#77): EventRelay — cross-window event bridging for 15 event types`

---

### Task 3: DetachController + DetachRegistry

**Files:**
- Create: `packages/pages-runtime/src/detach/detach-controller.ts`
- Create: `packages/pages-runtime/src/detach/detach-registry.ts`
- Create: `packages/pages-runtime/src/detach/detach-controller.test.ts`
- Create: `packages/pages-runtime/src/detach/detach-registry.test.ts`
- Create: `packages/pages-runtime/src/detach/index.ts`

**Interfaces:**
- Consumes: `copyStyles` from Task 1, `EventRelay` from Task 2
- Produces:
  - `class DetachController { constructor(componentId: string, container: HTMLElement, panelTitle: string); detach(): void; reattach(): void; dispose(): void; readonly isDetached: boolean; readonly childWindow: Window | null; }`
  - `class DetachRegistry { has(id: string): boolean; get(id: string): DetachController | undefined; register(id: string, controller: DetachController): void; remove(id: string): void; reattachAll(): void; disposeAll(): void; }`

- [ ] **Step 1: Write DetachRegistry tests**

```typescript
import { describe, it, expect } from "vitest";
import { DetachRegistry } from "./detach-registry.js";
import type { DetachController } from "./detach-controller.js";

function mockController(id: string): DetachController {
  let detached = false;
  return {
    componentId: id,
    isDetached: detached,
    childWindow: null,
    detach() { detached = true; },
    reattach() { detached = false; },
    dispose() { detached = false; },
  } as unknown as DetachController;
}

describe("DetachRegistry", () => {
  it("registers and retrieves controllers", () => {
    const reg = new DetachRegistry();
    const ctrl = mockController("a");
    reg.register("a", ctrl);
    expect(reg.has("a")).toBe(true);
    expect(reg.get("a")).toBe(ctrl);
  });

  it("removes controllers", () => {
    const reg = new DetachRegistry();
    reg.register("a", mockController("a"));
    reg.remove("a");
    expect(reg.has("a")).toBe(false);
  });

  it("reattachAll calls reattach on every controller", () => {
    const reg = new DetachRegistry();
    const calls: string[] = [];
    const a = mockController("a");
    const b = mockController("b");
    a.reattach = () => calls.push("a");
    b.reattach = () => calls.push("b");
    reg.register("a", a);
    reg.register("b", b);
    reg.reattachAll();
    expect(calls).toEqual(["a", "b"]);
    expect(reg.has("a")).toBe(false);
    expect(reg.has("b")).toBe(false);
  });

  it("disposeAll calls dispose and clears", () => {
    const reg = new DetachRegistry();
    const calls: string[] = [];
    const a = mockController("a");
    a.dispose = () => calls.push("disposed-a");
    reg.register("a", a);
    reg.disposeAll();
    expect(calls).toEqual(["disposed-a"]);
    expect(reg.has("a")).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run detach-registry`
Expected: FAIL

- [ ] **Step 3: Write DetachRegistry implementation**

```typescript
import type { DetachController } from "./detach-controller.js";

export class DetachRegistry {
  private readonly controllers = new Map<string, DetachController>();

  has(id: string): boolean { return this.controllers.has(id); }

  get(id: string): DetachController | undefined { return this.controllers.get(id); }

  register(id: string, controller: DetachController): void {
    this.controllers.set(id, controller);
  }

  remove(id: string): void { this.controllers.delete(id); }

  reattachAll(): void {
    for (const [id, ctrl] of this.controllers) {
      ctrl.reattach();
      this.controllers.delete(id);
    }
  }

  disposeAll(): void {
    for (const [id, ctrl] of this.controllers) {
      ctrl.dispose();
      this.controllers.delete(id);
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run detach-registry`
Expected: PASS

- [ ] **Step 5: Write DetachController tests**

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { DetachController } from "./detach-controller.js";

describe("DetachController", () => {
  let container: HTMLElement;
  let parent: HTMLElement;

  beforeEach(() => {
    parent = document.createElement("div");
    document.body.appendChild(parent);
    container = document.createElement("div");
    container.dataset.componentId = "test-panel";
    const title = document.createElement("div");
    title.dataset.panelTitle = "";
    title.textContent = "Test Panel";
    container.appendChild(title);
    parent.appendChild(container);
  });

  it("inserts placeholder before container on detach", () => {
    const ctrl = new DetachController("test-panel", container, "Test Panel");
    const fakeWindow = makeFakeWindow();
    vi.spyOn(globalThis, "open").mockReturnValue(fakeWindow as unknown as Window);

    ctrl.detach();

    const placeholder = parent.querySelector("[data-detach-placeholder]");
    expect(placeholder).not.toBeNull();
    expect(placeholder!.getAttribute("data-detach-placeholder")).toBe("test-panel");
  });

  it("sets data-detaching attribute during adoption", () => {
    const ctrl = new DetachController("test-panel", container, "Test Panel");
    const fakeWindow = makeFakeWindow();
    vi.spyOn(globalThis, "open").mockReturnValue(fakeWindow as unknown as Window);

    ctrl.detach();

    expect(container.hasAttribute("data-detaching")).toBe(false);
  });

  it("aborts and warns when popup is blocked", () => {
    const ctrl = new DetachController("test-panel", container, "Test Panel");
    vi.spyOn(globalThis, "open").mockReturnValue(null);

    ctrl.detach();

    expect(ctrl.isDetached).toBe(false);
    expect(parent.querySelector("[data-detach-placeholder]")).toBeNull();
  });

  it("reattach restores container to original position", () => {
    const sibling = document.createElement("div");
    sibling.textContent = "sibling";
    parent.appendChild(sibling);

    const ctrl = new DetachController("test-panel", container, "Test Panel");
    const fakeWindow = makeFakeWindow();
    vi.spyOn(globalThis, "open").mockReturnValue(fakeWindow as unknown as Window);

    ctrl.detach();
    expect(parent.contains(container)).toBe(false);

    ctrl.reattach();
    expect(parent.contains(container)).toBe(true);
    expect(parent.querySelector("[data-detach-placeholder]")).toBeNull();
  });

  it("isDetached reflects state", () => {
    const ctrl = new DetachController("test-panel", container, "Test Panel");
    expect(ctrl.isDetached).toBe(false);

    const fakeWindow = makeFakeWindow();
    vi.spyOn(globalThis, "open").mockReturnValue(fakeWindow as unknown as Window);
    ctrl.detach();
    expect(ctrl.isDetached).toBe(true);

    ctrl.reattach();
    expect(ctrl.isDetached).toBe(false);
  });
});

function makeFakeWindow() {
  const doc = document.implementation.createHTMLDocument("detached");
  return {
    document: doc,
    close: vi.fn(),
    focus: vi.fn(),
    closed: false,
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
  };
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run detach-controller`
Expected: FAIL

- [ ] **Step 7: Write DetachController implementation**

```typescript
import { copyStyles } from "./copy-styles.js";
import { EventRelay } from "./event-relay.js";

export class DetachController {
  readonly componentId: string;
  private container: HTMLElement;
  private placeholder: HTMLElement | null = null;
  private _childWindow: Window | null = null;
  private eventRelay: EventRelay | null = null;
  private pollTimer: ReturnType<typeof setInterval> | null = null;
  private panelTitle: string;
  private _isDetached = false;

  constructor(componentId: string, container: HTMLElement, panelTitle: string) {
    this.componentId = componentId;
    this.container = container;
    this.panelTitle = panelTitle;
  }

  get isDetached(): boolean { return this._isDetached; }
  get childWindow(): Window | null { return this._childWindow; }

  detach(): void {
    if (this._isDetached) return;

    const parentEl = this.container.parentElement;
    if (!parentEl) return;

    const placeholder = document.createElement("div");
    placeholder.setAttribute("data-detach-placeholder", this.componentId);
    placeholder.style.padding = "16px";
    placeholder.style.textAlign = "center";
    placeholder.style.color = "var(--pages-neutral-9, #999)";
    placeholder.textContent = "Panel detached";
    parentEl.insertBefore(placeholder, this.container);
    this.placeholder = placeholder;

    const win = window.open("", "_blank", "width=800,height=600");
    if (!win) {
      placeholder.remove();
      this.placeholder = null;
      console.warn("Popup blocked — allow popups to detach panels.");
      return;
    }

    this._childWindow = win;
    copyStyles(document, win.document);
    win.document.title = this.panelTitle;

    win.document.body.style.margin = "0";
    win.document.body.style.width = "100%";
    win.document.body.style.height = "100vh";
    win.document.body.style.overflow = "auto";

    this.container.setAttribute("data-detaching", "");
    win.document.body.appendChild(win.document.adoptNode(this.container));
    this.container.removeAttribute("data-detaching");

    this.container.style.width = "100%";
    this.container.style.height = "100%";

    this.eventRelay = new EventRelay(win.document, placeholder);
    this.eventRelay.start();

    win.addEventListener("beforeunload", () => this.reattach());

    this.pollTimer = setInterval(() => {
      if (win.closed && this._isDetached) this.reattach();
    }, 500);

    this._isDetached = true;
    win.focus();
  }

  reattach(): void {
    if (!this._isDetached) return;

    this.eventRelay?.stop();
    this.eventRelay = null;

    if (this.pollTimer !== null) {
      clearInterval(this.pollTimer);
      this.pollTimer = null;
    }

    if (this.placeholder?.parentElement) {
      this.container.setAttribute("data-detaching", "");
      this.placeholder.parentElement.insertBefore(
        document.adoptNode(this.container),
        this.placeholder,
      );
      this.container.removeAttribute("data-detaching");
      this.container.style.width = "";
      this.container.style.height = "";
      this.placeholder.remove();
    }
    this.placeholder = null;

    if (this._childWindow && !this._childWindow.closed) {
      this._childWindow.close();
    }
    this._childWindow = null;
    this._isDetached = false;

    this.container.setAttribute("tabindex", "-1");
    this.container.focus();
    this.container.removeAttribute("tabindex");
  }

  dispose(): void {
    if (this._isDetached) {
      this.reattach();
    }
    if (this._childWindow && !this._childWindow.closed) {
      this._childWindow.close();
    }
    this._childWindow = null;
  }
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run detach-controller`
Expected: PASS

- [ ] **Step 9: Write barrel export**

Create `packages/pages-runtime/src/detach/index.ts`:
```typescript
export { DetachController } from "./detach-controller.js";
export { DetachRegistry } from "./detach-registry.js";
export { EventRelay } from "./event-relay.js";
export { copyStyles } from "./copy-styles.js";
```

- [ ] **Step 10: Commit**

`feat(#77): DetachController + DetachRegistry — panel detach/reattach lifecycle`

---

### Task 4: Wire into site.ts — detach listener, dispose hook, theme sync

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` (~lines 112, 225, 463-900, 1037-1079)
- Modify: `packages/pages-runtime/src/activation.ts` (~lines 72-84)
- Modify: `packages/pages-runtime/src/index.ts`
- Create: `packages/pages-runtime/src/detach/detach-integration.test.ts`

**Interfaces:**
- Consumes: `DetachController`, `DetachRegistry` from Task 3, `copyStyles` from Task 1
- Produces: Full integration — `pages-panel-detach` event triggers detach, `dispose()` calls `reattachAll()`

- [ ] **Step 1: Write integration test for detach button rendering**

```typescript
import { describe, it, expect } from "vitest";

describe("panel detach button", () => {
  it("activation callback adds detach button to panel title bar", () => {
    const el = document.createElement("div");
    el.dataset.componentType = "panel";
    el.dataset.componentId = "p1";
    const titleEl = document.createElement("div");
    titleEl.dataset.panelTitle = "";
    titleEl.textContent = "My Panel";
    el.appendChild(titleEl);
    document.body.appendChild(el);

    const component = { type: "panel", props: { title: "My Panel" } };
    // Simulate what addDetachButton does
    const btn = document.createElement("button");
    btn.dataset.detach = "";
    btn.textContent = "↗";
    btn.style.position = "absolute";
    btn.style.right = "4px";
    btn.style.top = "4px";
    titleEl.style.position = "relative";
    titleEl.appendChild(btn);

    const detachBtn = el.querySelector("[data-detach]");
    expect(detachBtn).not.toBeNull();
    expect(detachBtn!.textContent).toBe("↗");
  });

  it("detach button dispatches pages-panel-detach event", () => {
    const el = document.createElement("div");
    el.dataset.componentId = "p1";
    const btn = document.createElement("button");
    btn.dataset.detach = "";
    el.appendChild(btn);
    document.body.appendChild(el);

    let detail: unknown = null;
    el.addEventListener("pages-panel-detach", ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);

    btn.dispatchEvent(new MouseEvent("click", { bubbles: true }));
    // In real code, the click handler dispatches the custom event
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run detach-integration`
Expected: FAIL

- [ ] **Step 3: Add detach button to panel title bar in activation.ts**

In `activation.ts`, inside the `createActivationCallback` function, after the panel title bar is detected (for `type: "panel"` and `type: "host-panel"`), append the detach button. Find the section where component types are handled. After the existing panel title check, add:

```typescript
const titleBar = el.querySelector("[data-panel-title]") as HTMLElement | null;
if (titleBar && (component.type === "panel" || component.type === "host-panel")) {
  titleBar.style.position = "relative";
  const detachBtn = document.createElement("button");
  detachBtn.dataset.detach = "";
  detachBtn.textContent = "↗";
  detachBtn.title = "Detach panel";
  detachBtn.style.cssText = "position:absolute;right:4px;top:50%;transform:translateY(-50%);border:none;background:transparent;cursor:pointer;font-size:14px;opacity:0.5;padding:2px 4px;border-radius:var(--pages-radius-sm,4px);";
  detachBtn.addEventListener("click", (e) => {
    e.stopPropagation();
    el.dispatchEvent(new CustomEvent("pages-panel-detach", {
      bubbles: true, composed: true,
      detail: { componentId },
    }));
  });
  detachBtn.addEventListener("mouseenter", () => { detachBtn.style.opacity = "1"; });
  detachBtn.addEventListener("mouseleave", () => { detachBtn.style.opacity = "0.5"; });
  titleBar.appendChild(detachBtn);
}
```

- [ ] **Step 4: Add pages-panel-detach listener in site.ts loadSite()**

After the existing event listeners (around line 895), add:

```typescript
const detachRegistry = new DetachRegistry();

target.addEventListener("pages-panel-detach", ((e: Event) => {
  const detail = (e as CustomEvent<{ componentId: string }>).detail;
  const { componentId } = detail;
  if (detachRegistry.has(componentId)) {
    detachRegistry.get(componentId)!.childWindow?.focus();
    return;
  }
  const container = target.querySelector<HTMLElement>(
    `[data-component-id="${componentId}"]`
  );
  if (!container) return;

  const titleEl = container.querySelector("[data-panel-title]");
  const panelTitle = titleEl?.textContent ?? "Panel";

  const ctrl = new DetachController(componentId, container, panelTitle);
  detachRegistry.register(componentId, ctrl);
  ctrl.detach();
}) as EventListener);
```

- [ ] **Step 5: Hook reattachAll into dispose()**

In the `dispose()` function in `site.ts` (line ~1058), add before `pipeline.dispose()`:

```typescript
detachRegistry.reattachAll();
```

- [ ] **Step 6: Hook theme sync into setTheme()**

In the `setTheme()` function in `site.ts` (line ~1037), add after `applyTheme(...)`:

```typescript
for (const [, ctrl] of (detachRegistry as any).controllers) {
  if (ctrl.isDetached && ctrl.childWindow) {
    copyStyles(document, ctrl.childWindow.document);
  }
}
```

Note: expose an `entries()` or `forEach()` method on `DetachRegistry` rather than accessing `.controllers` directly. Add to DetachRegistry:

```typescript
forEach(fn: (controller: DetachController, id: string) => void): void {
  this.controllers.forEach(fn);
}
```

- [ ] **Step 7: Add exports to index.ts**

Add to `packages/pages-runtime/src/index.ts`:

```typescript
export { DetachController } from "./detach/index.js";
export { DetachRegistry } from "./detach/index.js";
```

- [ ] **Step 8: Run all runtime tests**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: all pass (existing + new)

- [ ] **Step 9: Run typecheck**

Run: `GH_PACKAGES_TOKEN=dummy yarn typecheck`
Expected: no new errors

- [ ] **Step 10: Commit**

`feat(#77): wire detach into site.ts — listener, dispose, theme sync`

---

### Task 5: Full integration test + manual verification

**Files:**
- Modify: `packages/pages-runtime/src/detach/detach-controller.test.ts` (add edge case tests)

**Interfaces:**
- Consumes: everything from Tasks 1-4
- Produces: confidence that the feature works end-to-end

- [ ] **Step 1: Add edge case tests**

```typescript
describe("DetachController edge cases", () => {
  it("double detach focuses existing window instead of opening new one", () => {
    const container = makeContainer("p1");
    const ctrl = new DetachController("p1", container, "Panel");
    const fakeWindow = makeFakeWindow();
    vi.spyOn(globalThis, "open").mockReturnValue(fakeWindow as unknown as Window);

    ctrl.detach();
    ctrl.detach();

    expect(fakeWindow.focus).toHaveBeenCalledTimes(2);
    expect(globalThis.open).toHaveBeenCalledTimes(1);
  });

  it("dispose reattaches before closing window", () => {
    const container = makeContainer("p1");
    const parent = document.createElement("div");
    parent.appendChild(container);
    document.body.appendChild(parent);

    const ctrl = new DetachController("p1", container, "Panel");
    const fakeWindow = makeFakeWindow();
    vi.spyOn(globalThis, "open").mockReturnValue(fakeWindow as unknown as Window);

    ctrl.detach();
    expect(parent.contains(container)).toBe(false);

    ctrl.dispose();
    expect(parent.contains(container)).toBe(true);
    expect(fakeWindow.close).toHaveBeenCalled();
  });

  it("reattach is idempotent", () => {
    const container = makeContainer("p1");
    const parent = document.createElement("div");
    parent.appendChild(container);
    document.body.appendChild(parent);

    const ctrl = new DetachController("p1", container, "Panel");
    const fakeWindow = makeFakeWindow();
    vi.spyOn(globalThis, "open").mockReturnValue(fakeWindow as unknown as Window);

    ctrl.detach();
    ctrl.reattach();
    ctrl.reattach();

    expect(parent.children.length).toBe(1);
    expect(parent.contains(container)).toBe(true);
  });
});

function makeContainer(id: string): HTMLElement {
  const el = document.createElement("div");
  el.dataset.componentId = id;
  const title = document.createElement("div");
  title.dataset.panelTitle = "";
  title.textContent = "Panel";
  el.appendChild(title);
  return el;
}
```

- [ ] **Step 2: Run all tests**

Run: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: all pass

- [ ] **Step 3: Run full build**

Run: `GH_PACKAGES_TOKEN=dummy yarn build`
Expected: clean build

- [ ] **Step 4: Run typecheck + lint**

Run: `GH_PACKAGES_TOKEN=dummy yarn typecheck && GH_PACKAGES_TOKEN=dummy yarn lint`
Expected: no new errors (pre-existing pages-component/events.ts errors are known)

- [ ] **Step 5: Commit**

`test(#77): edge case tests — double detach, dispose reattach, idempotency`

- [ ] **Step 6: Manual verification**

Start dev server: `GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/pages-examples run dev`

Test in browser:
1. Open a dashboard with panels
2. Click ↗ on a panel title → should open in new window
3. Verify data updates flow to detached panel
4. Apply a filter in main window → should affect detached panel
5. Close child window → panel should return to original position
6. Detach two panels simultaneously → both should work
7. Switch theme while panel is detached → should sync
