# FloatingFrameEngine Auto-Wire Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #303 — FloatingFrameEngine: auto-wire backend events, pluggable chrome, tab-bar drag fix
**Issue group:** #303

**Goal:** Extract engine-backend wiring boilerplate into a reusable wire function, add pluggable frame chrome, fix pin visual state accessibility, add pin drag lock, and fix tab-bar empty-space drag.

**Architecture:** The engine stays as a pure state machine (no DOM dependency). A new `wireFloatingWorkspace()` function handles all callback→event→state integration. The backend interface gains `onFrameClose`/`onFramePin` callbacks, `updatePinState()` for visual feedback, and `extraButtons` config on `attach()`. activation.ts and site.ts simplify to use the wire function.

**Tech Stack:** TypeScript 5, Vitest, dockview-core ^7.0.0, Lit (for a11y attributes)

## Global Constraints

- All CustomEvents dispatched with `bubbles: true, composed: true`
- Event names must match the reserved names in `docs/protocols/casehub/pages-event-contract.md`
- Engine must remain pure — no DOM imports, no HTMLElement references
- Backend interface changes must be reflected in the error backend (`createErrorBackend()`)
- All new public types/functions exported from `packages/pages-runtime/src/index.ts`

---

### Task 1: Engine — Add updatePosition and updateSize methods

**Files:**
- Modify: `packages/pages-runtime/src/floating-frame-engine.ts`
- Modify: `packages/pages-runtime/src/floating-frame-engine.test.ts`

**Interfaces:**
- Consumes: existing `FloatingFrameEngine` interface, `FrameLayout` type from `@casehubio/pages-component`
- Produces: `updatePosition(key: string, pos: { x: number; y: number }): void` and `updateSize(key: string, size: { width: number; height: number }): void` on `FloatingFrameEngine`

- [ ] **Step 1: Write failing tests for updatePosition and updateSize**

Add to `floating-frame-engine.test.ts`:

```typescript
describe("position/size sync", () => {
  it("updatePosition updates frame position without backend call", () => {
    engine.createFrame(makeFrameConfig("f1"));
    (backend.updatePosition as ReturnType<typeof vi.fn>).mockClear();
    engine.updatePosition("f1", { x: 200, y: 300 });
    expect(engine.frames.get("f1")!.position).toEqual({ x: 200, y: 300 });
    expect(backend.updatePosition).not.toHaveBeenCalled();
  });

  it("updateSize updates frame size without backend call", () => {
    engine.createFrame(makeFrameConfig("f1"));
    (backend.updateSize as ReturnType<typeof vi.fn>).mockClear();
    engine.updateSize("f1", { width: 600, height: 500 });
    expect(engine.frames.get("f1")!.size).toEqual({ width: 600, height: 500 });
    expect(backend.updateSize).not.toHaveBeenCalled();
  });

  it("updatePosition on unknown key is a no-op", () => {
    engine.updatePosition("unknown", { x: 0, y: 0 });
    expect(engine.frames.size).toBe(0);
  });

  it("updateSize on unknown key is a no-op", () => {
    engine.updateSize("unknown", { width: 0, height: 0 });
    expect(engine.frames.size).toBe(0);
  });

  it("captureLayout reflects updated position after updatePosition", () => {
    engine.createFrame(makeFrameConfig("f1"));
    engine.updatePosition("f1", { x: 999, y: 888 });
    const saved = engine.captureLayout();
    expect(saved[0]!.position).toEqual({ x: 999, y: 888 });
  });

  it("captureLayout reflects updated size after updateSize", () => {
    engine.createFrame(makeFrameConfig("f1"));
    engine.updateSize("f1", { width: 777, height: 666 });
    const saved = engine.captureLayout();
    expect(saved[0]!.size).toEqual({ width: 777, height: 666 });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine.test.ts`
Expected: FAIL — `engine.updatePosition is not a function`

- [ ] **Step 3: Add updatePosition and updateSize to the engine interface and implementation**

In `floating-frame-engine.ts`, add to the `FloatingFrameEngine` interface (after `togglePin`):

```typescript
updatePosition(key: string, pos: { x: number; y: number }): void;
updateSize(key: string, size: { width: number; height: number }): void;
```

Add to the engine implementation object (after the `togglePin` method):

```typescript
updatePosition(key: string, pos: { x: number; y: number }) {
  assertAlive();
  const frame = frames.get(key);
  if (!frame) return;
  frames.set(key, { key: frame.key, order: frame.order, position: pos, size: frame.size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: frame.hidden, tabs: frame.tabs, activeTabKey: frame.activeTabKey });
},

updateSize(key: string, size: { width: number; height: number }) {
  assertAlive();
  const frame = frames.get(key);
  if (!frame) return;
  frames.set(key, { key: frame.key, order: frame.order, position: frame.position, size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: frame.hidden, tabs: frame.tabs, activeTabKey: frame.activeTabKey });
},
```

- [ ] **Step 4: Update mockBackend in test to include updatePinState (forward compat)**

The mock will need `updatePinState` for Task 2's backend changes. Add it now so the mock stays in sync:

```typescript
function mockBackend(): FloatingFrameBackend {
  return {
    attach: vi.fn(), detach: vi.fn(),
    renderFrame: vi.fn(), removeFrame: vi.fn(),
    updatePosition: vi.fn(), updateSize: vi.fn(), bringToFront: vi.fn(),
    addTab: vi.fn(), removeTab: vi.fn(), setActiveTab: vi.fn(),
    onFrameMove: vi.fn(), onFrameResize: vi.fn(), onTabDragOut: vi.fn(), onTabReorder: vi.fn(),
    onFrameClose: vi.fn(), onFramePin: vi.fn(),
    updatePinState: vi.fn(),
    dispose: vi.fn(), unwrap: vi.fn(() => null),
  };
}
```

Note: this will fail to compile until Task 2 adds the methods to the interface. For now, add a `// @ts-expect-error — added in Task 2` comment above the mock, or implement Task 2's interface changes first. **Preferred:** implement Task 1 and Task 2 interface changes together, then the tests pass.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/floating-frame-engine.ts packages/pages-runtime/src/floating-frame-engine.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#303): add updatePosition/updateSize to FloatingFrameEngine

State-only methods that update the engine's internal frames map
without calling the backend. Fixes the stale-position bug —
captureLayout() now reflects actual drag/resize positions.

Refs casehubio/casehub-pages#303"
```

---

### Task 2: Backend interface — Add onFrameClose, onFramePin, updatePinState, and attach options

**Files:**
- Modify: `packages/pages-runtime/src/floating-frame-backend.ts`

**Interfaces:**
- Consumes: existing `FloatingFrameBackend` interface, `FrameLayout`, `FrameTabConfig`, `ContentFactory` from `@casehubio/pages-component`
- Produces:
  - `onFrameClose(cb: (key: string) => void): void` on `FloatingFrameBackend`
  - `onFramePin(cb: (key: string) => void): void` on `FloatingFrameBackend`
  - `updatePinState(key: string, pinned: boolean): void` on `FloatingFrameBackend`
  - `FrameButtonConfig` type: `{ readonly icon: string; readonly title: string; readonly className?: string; readonly onClick: (frameKey: string) => void }`
  - `BackendAttachOptions` type: `{ readonly extraButtons?: readonly FrameButtonConfig[] }`
  - Updated `attach` signature: `attach(container: HTMLElement, contentFactory: ContentFactory, options?: BackendAttachOptions): void`

- [ ] **Step 1: Update the backend interface**

Replace the contents of `floating-frame-backend.ts`:

```typescript
import type { FrameLayout, FrameTabConfig, ContentFactory } from "@casehubio/pages-component";

export interface FrameButtonConfig {
  readonly icon: string;
  readonly title: string;
  readonly className?: string;
  readonly onClick: (frameKey: string) => void;
}

export interface BackendAttachOptions {
  readonly extraButtons?: readonly FrameButtonConfig[];
}

export interface FloatingFrameBackend {
  attach(container: HTMLElement, contentFactory: ContentFactory, options?: BackendAttachOptions): void;
  detach(): void;

  renderFrame(layout: FrameLayout): void;
  removeFrame(key: string): void;
  updatePosition(key: string, pos: { x: number; y: number }): void;
  updateSize(key: string, size: { width: number; height: number }): void;
  bringToFront(key: string): void;

  addTab(frameKey: string, tab: FrameTabConfig): void;
  removeTab(frameKey: string, tabKey: string): void;
  setActiveTab(frameKey: string, tabKey: string): void;

  onFrameMove(cb: (key: string, pos: { x: number; y: number }) => void): void;
  onFrameResize(cb: (key: string, size: { width: number; height: number }) => void): void;
  onTabDragOut(cb: (fromFrame: string, tabKey: string, position: { x: number; y: number }) => void): void;
  onTabReorder(cb: (frameKey: string, tabKeys: string[]) => void): void;
  onFrameClose(cb: (key: string) => void): void;
  onFramePin(cb: (key: string) => void): void;

  updatePinState(key: string, pinned: boolean): void;

  dispose(): void;
  unwrap(): unknown | null;
}
```

- [ ] **Step 2: Run type check to identify all compile errors**

Run: `yarn typecheck`
Expected: Errors in `dockview-backend.ts` (missing methods), `floating-frame-engine.test.ts` (mock missing methods). These will be fixed in subsequent tasks.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/floating-frame-backend.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#303): extend FloatingFrameBackend with close/pin callbacks, updatePinState, extraButtons

Adds onFrameClose, onFramePin callbacks for chrome button signals.
Adds updatePinState for engine-driven pin visual feedback.
Adds FrameButtonConfig and BackendAttachOptions for pluggable chrome.
Compile errors in dockview-backend.ts expected — fixed in next task.

Refs casehubio/casehub-pages#303"
```

---

### Task 3: DockviewBackend — Implement new interface methods

**Files:**
- Modify: `packages/pages-runtime/src/dockview-backend.ts`

**Interfaces:**
- Consumes: `FloatingFrameBackend` (updated in Task 2), `FrameButtonConfig`, `BackendAttachOptions`
- Produces: Working DockviewBackend implementation with all new methods; error backend updated

- [ ] **Step 1: Add callback arrays and stored options**

In `createDockviewBackend()`, after the existing callback arrays (line ~41), add:

```typescript
const frameCloseCallbacks: Array<(key: string) => void> = [];
const framePinCallbacks: Array<(key: string) => void> = [];
let extraButtons: readonly FrameButtonConfig[] = [];
```

- [ ] **Step 2: Update attach() to accept options and store extraButtons**

Change the `attach` method signature and store extra buttons:

```typescript
attach(el: HTMLElement, contentFactory: ContentFactory, options?: BackendAttachOptions) {
  container = el;
  factory = contentFactory;
  extraButtons = options?.extraButtons ?? [];
  // ... rest unchanged
```

- [ ] **Step 3: Update injectFrameChrome — callbacks instead of DOM events, extra buttons, pin a11y**

Replace the `injectFrameChrome` function:

```typescript
function injectFrameChrome(group: any, frameKey: string): void {
  const el = group.element ?? group.header?.element?.closest?.(".dv-groupview");
  if (!el) return;
  const titlebar = el.querySelector(".dv-floating-titlebar");
  if (!titlebar) return;

  const closeDot = document.createElement("span");
  closeDot.className = "frame-close-dot";
  closeDot.style.cssText = "width:12px;height:12px;border-radius:50%;background:#ff5f57;cursor:pointer;display:inline-block;margin:0 4px;";
  closeDot.addEventListener("pointerdown", (e) => e.stopPropagation());
  closeDot.addEventListener("click", () => {
    for (const cb of frameCloseCallbacks) cb(frameKey);
  });

  const pinBtn = document.createElement("span");
  pinBtn.className = "frame-pin-btn";
  pinBtn.textContent = "\u{1F4CC}";
  pinBtn.style.cssText = "cursor:pointer;margin:0 4px;font-size:12px;opacity:0.5;";
  pinBtn.setAttribute("aria-pressed", "false");
  pinBtn.addEventListener("pointerdown", (e) => e.stopPropagation());
  pinBtn.addEventListener("click", () => {
    for (const cb of framePinCallbacks) cb(frameKey);
  });

  titlebar.prepend(pinBtn);
  titlebar.prepend(closeDot);

  for (const btnConfig of extraButtons) {
    const btn = document.createElement("span");
    btn.className = `frame-extra-btn${btnConfig.className ? ` ${btnConfig.className}` : ""}`;
    btn.textContent = btnConfig.icon;
    btn.title = btnConfig.title;
    btn.style.cssText = "cursor:pointer;margin:0 4px;font-size:12px;";
    btn.addEventListener("pointerdown", (e) => e.stopPropagation());
    btn.addEventListener("click", () => btnConfig.onClick(frameKey));
    titlebar.appendChild(btn);
  }
}
```

- [ ] **Step 4: Implement updatePinState**

Add to the backend object (after `bringToFront`):

```typescript
updatePinState(key: string, pinned: boolean) {
  const group = frameGroups.get(key);
  if (!group) return;
  const el = group.element ?? group.header?.element?.closest?.(".dv-groupview");
  if (!el) return;
  const pinBtn = el.querySelector(".frame-pin-btn") as HTMLElement | null;
  if (pinBtn) {
    pinBtn.style.opacity = pinned ? "1" : "0.5";
    pinBtn.setAttribute("aria-pressed", String(pinned));
    pinBtn.classList.toggle("frame-pin-active", pinned);
  }
},
```

- [ ] **Step 5: Add callback registration and dispose cleanup**

Add to the backend object:

```typescript
onFrameClose(cb: (key: string) => void) { frameCloseCallbacks.push(cb); },
onFramePin(cb: (key: string) => void) { framePinCallbacks.push(cb); },
```

Update `dispose()` to clear new arrays:

```typescript
dispose() {
  if (dockview) { dockview.dispose(); dockview = null; }
  frameGroups.clear();
  contentResults.clear();
  frameMoveCallbacks.length = 0;
  frameResizeCallbacks.length = 0;
  tabDragOutCallbacks.length = 0;
  tabReorderCallbacks.length = 0;
  frameCloseCallbacks.length = 0;
  framePinCallbacks.length = 0;
},
```

- [ ] **Step 6: Update error backend**

Update `createErrorBackend()` to include new methods:

```typescript
function createErrorBackend(): FloatingFrameBackend {
  return {
    attach(container: HTMLElement) {
      const div = document.createElement("div");
      div.className = "pages-floating-workspace-error";
      div.textContent = "Floating workspace failed to load";
      div.style.cssText = "padding:24px;color:#ff5f57;text-align:center;";
      container.appendChild(div);
    },
    detach() {},
    renderFrame() {}, removeFrame() {}, updatePosition() {}, updateSize() {}, bringToFront() {},
    addTab() {}, removeTab() {}, setActiveTab() {},
    onFrameMove() {}, onFrameResize() {}, onTabDragOut() {}, onTabReorder() {},
    onFrameClose() {}, onFramePin() {},
    updatePinState() {},
    dispose() {},
    unwrap() { return null; },
  };
}
```

- [ ] **Step 7: Run type check**

Run: `yarn typecheck`
Expected: No errors in `dockview-backend.ts`. May still have errors in test files (mock update) — resolved in Task 1 Step 4.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/dockview-backend.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#303): implement onFrameClose/onFramePin, updatePinState, extraButtons in DockviewBackend

Chrome buttons now fire typed callbacks instead of DOM events.
updatePinState toggles opacity, aria-pressed, and CSS class.
extraButtons appended to titlebar after built-in buttons.
Error backend updated with no-op implementations.

Refs casehubio/casehub-pages#303"
```

---

### Task 4: Wire function — wireFloatingWorkspace

**Files:**
- Create: `packages/pages-runtime/src/wire-floating-workspace.ts`
- Create: `packages/pages-runtime/src/wire-floating-workspace.test.ts`

**Interfaces:**
- Consumes: `FloatingFrameEngine` from `./floating-frame-engine.js`, `FloatingFrameBackend` from `./floating-frame-backend.js`, `FrameLayout` from `@casehubio/pages-component`
- Produces:
  - `WireHandle`: `{ readonly engine: FloatingFrameEngine; dispose(): void }`
  - `wireFloatingWorkspace(backend: FloatingFrameBackend, container: HTMLElement, savedLayout?: readonly FrameLayout[]): WireHandle`

- [ ] **Step 1: Write failing tests**

Create `packages/pages-runtime/src/wire-floating-workspace.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";
import { wireFloatingWorkspace } from "./wire-floating-workspace.js";
import type { FloatingFrameBackend } from "./floating-frame-backend.js";
import type { FrameTabConfig } from "@casehubio/pages-component";

function makeTab(key: string): FrameTabConfig {
  return { key, label: key, content: { type: "html", props: { content: `<div>${key}</div>` } } };
}

type Callback<T extends unknown[]> = (...args: T) => void;

function mockBackend(): FloatingFrameBackend & {
  _fireMoveCallbacks: (key: string, pos: { x: number; y: number }) => void;
  _fireResizeCallbacks: (key: string, size: { width: number; height: number }) => void;
  _fireCloseCallbacks: (key: string) => void;
  _firePinCallbacks: (key: string) => void;
  _fireTabDragOutCallbacks: (fromFrame: string, tabKey: string, pos: { x: number; y: number }) => void;
  _fireTabReorderCallbacks: (frameKey: string, tabKeys: string[]) => void;
} {
  const moveCbs: Callback<[string, { x: number; y: number }]>[] = [];
  const resizeCbs: Callback<[string, { width: number; height: number }]>[] = [];
  const closeCbs: Callback<[string]>[] = [];
  const pinCbs: Callback<[string]>[] = [];
  const tabDragOutCbs: Callback<[string, string, { x: number; y: number }]>[] = [];
  const tabReorderCbs: Callback<[string, string[]]>[] = [];

  return {
    attach: vi.fn(), detach: vi.fn(),
    renderFrame: vi.fn(), removeFrame: vi.fn(),
    updatePosition: vi.fn(), updateSize: vi.fn(), bringToFront: vi.fn(),
    addTab: vi.fn(), removeTab: vi.fn(), setActiveTab: vi.fn(),
    onFrameMove(cb) { moveCbs.push(cb); },
    onFrameResize(cb) { resizeCbs.push(cb); },
    onTabDragOut(cb) { tabDragOutCbs.push(cb); },
    onTabReorder(cb) { tabReorderCbs.push(cb); },
    onFrameClose(cb) { closeCbs.push(cb); },
    onFramePin(cb) { pinCbs.push(cb); },
    updatePinState: vi.fn(),
    dispose: vi.fn(), unwrap: vi.fn(() => null),
    _fireMoveCallbacks(key, pos) { for (const cb of moveCbs) cb(key, pos); },
    _fireResizeCallbacks(key, size) { for (const cb of resizeCbs) cb(key, size); },
    _fireCloseCallbacks(key) { for (const cb of closeCbs) cb(key); },
    _firePinCallbacks(key) { for (const cb of pinCbs) cb(key); },
    _fireTabDragOutCallbacks(from, tab, pos) { for (const cb of tabDragOutCbs) cb(from, tab, pos); },
    _fireTabReorderCallbacks(fk, tks) { for (const cb of tabReorderCbs) cb(fk, tks); },
  };
}

describe("wireFloatingWorkspace", () => {
  let backend: ReturnType<typeof mockBackend>;
  let container: HTMLElement;

  beforeEach(() => {
    backend = mockBackend();
    container = document.createElement("div");
  });

  it("creates an engine and returns a WireHandle", () => {
    const handle = wireFloatingWorkspace(backend, container);
    expect(handle.engine).toBeDefined();
    expect(handle.engine.frames.size).toBe(0);
  });

  it("restores saved layout", () => {
    const saved = [{
      key: "f1", order: 0, position: { x: 10, y: 20 }, size: { width: 400, height: 300 },
      zIndex: 1, pinned: false, hidden: false, tabs: [makeTab("t1")], activeTabKey: "t1",
    }] as const;
    const handle = wireFloatingWorkspace(backend, container, saved);
    expect(handle.engine.frames.size).toBe(1);
  });

  describe("onFrameMove", () => {
    it("updates engine position and dispatches event", () => {
      const handle = wireFloatingWorkspace(backend, container);
      handle.engine.createFrame({ key: "f1", tabs: [makeTab("t1")] });

      const events: CustomEvent[] = [];
      container.addEventListener("pages-frame-move", ((e: Event) => events.push(e as CustomEvent)));

      backend._fireMoveCallbacks("f1", { x: 200, y: 300 });
      expect(handle.engine.frames.get("f1")!.position).toEqual({ x: 200, y: 300 });
      expect(events).toHaveLength(1);
      expect(events[0]!.detail).toEqual({ frameKey: "f1", position: { x: 200, y: 300 } });
    });
  });

  describe("onFrameResize", () => {
    it("updates engine size and dispatches event", () => {
      const handle = wireFloatingWorkspace(backend, container);
      handle.engine.createFrame({ key: "f1", tabs: [makeTab("t1")] });

      const events: CustomEvent[] = [];
      container.addEventListener("pages-frame-resize", ((e: Event) => events.push(e as CustomEvent)));

      backend._fireResizeCallbacks("f1", { width: 600, height: 500 });
      expect(handle.engine.frames.get("f1")!.size).toEqual({ width: 600, height: 500 });
      expect(events).toHaveLength(1);
      expect(events[0]!.detail).toEqual({ frameKey: "f1", size: { width: 600, height: 500 } });
    });
  });

  describe("onFrameClose", () => {
    it("removes frame from engine and dispatches event", () => {
      const handle = wireFloatingWorkspace(backend, container);
      handle.engine.createFrame({ key: "f1", tabs: [makeTab("t1")] });

      const events: CustomEvent[] = [];
      container.addEventListener("pages-frame-close", ((e: Event) => events.push(e as CustomEvent)));

      backend._fireCloseCallbacks("f1");
      expect(handle.engine.frames.size).toBe(0);
      expect(events).toHaveLength(1);
      expect(events[0]!.detail).toEqual({ frameKey: "f1" });
    });
  });

  describe("onFramePin", () => {
    it("toggles pin on engine, calls updatePinState, and dispatches event", () => {
      const handle = wireFloatingWorkspace(backend, container);
      handle.engine.createFrame({ key: "f1", tabs: [makeTab("t1")] });

      const events: CustomEvent[] = [];
      container.addEventListener("pages-frame-pin", ((e: Event) => events.push(e as CustomEvent)));

      backend._firePinCallbacks("f1");
      expect(handle.engine.frames.get("f1")!.pinned).toBe(true);
      expect(backend.updatePinState).toHaveBeenCalledWith("f1", true);
      expect(events).toHaveLength(1);
      expect(events[0]!.detail).toEqual({ frameKey: "f1", pinned: true });

      backend._firePinCallbacks("f1");
      expect(handle.engine.frames.get("f1")!.pinned).toBe(false);
      expect(backend.updatePinState).toHaveBeenCalledWith("f1", false);
      expect(events).toHaveLength(2);
      expect(events[1]!.detail).toEqual({ frameKey: "f1", pinned: false });
    });
  });

  describe("onTabDragOut", () => {
    it("creates new frame, moves tab, and dispatches event", () => {
      const handle = wireFloatingWorkspace(backend, container);
      handle.engine.createFrame({ key: "f1", tabs: [makeTab("t1"), makeTab("t2")] });

      const events: CustomEvent[] = [];
      container.addEventListener("pages-tab-drag-out", ((e: Event) => events.push(e as CustomEvent)));

      backend._fireTabDragOutCallbacks("f1", "t2", { x: 100, y: 100 });

      expect(handle.engine.frames.get("f1")!.tabs).toHaveLength(1);
      expect(handle.engine.frames.size).toBe(2);
      expect(events).toHaveLength(1);
      expect(events[0]!.detail.tabKey).toBe("t2");
      expect(events[0]!.detail.fromFrame).toBe("f1");
    });

    it("auto-closes source frame when last tab dragged out", () => {
      const handle = wireFloatingWorkspace(backend, container);
      handle.engine.createFrame({ key: "f1", tabs: [makeTab("t1")] });

      backend._fireTabDragOutCallbacks("f1", "t1", { x: 100, y: 100 });

      expect(handle.engine.frames.has("f1")).toBe(false);
    });
  });

  describe("onTabReorder", () => {
    it("dispatches event", () => {
      const handle = wireFloatingWorkspace(backend, container);

      const events: CustomEvent[] = [];
      container.addEventListener("pages-tab-reorder", ((e: Event) => events.push(e as CustomEvent)));

      backend._fireTabReorderCallbacks("f1", ["t2", "t1"]);
      expect(events).toHaveLength(1);
      expect(events[0]!.detail).toEqual({ frameKey: "f1", tabKeys: ["t2", "t1"] });
    });
  });

  describe("dispose", () => {
    it("disposes the engine", () => {
      const handle = wireFloatingWorkspace(backend, container);
      handle.dispose();
      expect(() => handle.engine.createFrame({ key: "f1", tabs: [makeTab("t1")] })).toThrow("Engine is disposed");
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run wire-floating-workspace.test.ts`
Expected: FAIL — cannot find module `./wire-floating-workspace.js`

- [ ] **Step 3: Implement wireFloatingWorkspace**

Create `packages/pages-runtime/src/wire-floating-workspace.ts`:

```typescript
import type { FrameLayout } from "@casehubio/pages-component";
import type { FloatingFrameBackend } from "./floating-frame-backend.js";
import { createFloatingFrameEngine } from "./floating-frame-engine.js";
import type { FloatingFrameEngine } from "./floating-frame-engine.js";

export interface WireHandle {
  readonly engine: FloatingFrameEngine;
  dispose(): void;
}

let nextFrameId = 0;

export function wireFloatingWorkspace(
  backend: FloatingFrameBackend,
  container: HTMLElement,
  savedLayout?: readonly FrameLayout[],
): WireHandle {
  const engine = createFloatingFrameEngine(backend, savedLayout);

  backend.onFrameMove((key, pos) => {
    engine.updatePosition(key, pos);
    container.dispatchEvent(new CustomEvent("pages-frame-move", {
      bubbles: true, composed: true,
      detail: { frameKey: key, position: pos },
    }));
  });

  backend.onFrameResize((key, size) => {
    engine.updateSize(key, size);
    container.dispatchEvent(new CustomEvent("pages-frame-resize", {
      bubbles: true, composed: true,
      detail: { frameKey: key, size },
    }));
  });

  backend.onFrameClose((key) => {
    engine.removeFrame(key);
    container.dispatchEvent(new CustomEvent("pages-frame-close", {
      bubbles: true, composed: true,
      detail: { frameKey: key },
    }));
  });

  backend.onFramePin((key) => {
    engine.togglePin(key);
    const frame = engine.frames.get(key);
    const pinned = frame?.pinned ?? false;
    backend.updatePinState(key, pinned);
    container.dispatchEvent(new CustomEvent("pages-frame-pin", {
      bubbles: true, composed: true,
      detail: { frameKey: key, pinned },
    }));
  });

  backend.onTabDragOut((fromFrame, tabKey, position) => {
    const newKey = `frame-${String(++nextFrameId)}`;
    engine.createFrame({ key: newKey, tabs: [], position, size: { width: 400, height: 300 } });
    engine.moveTab(fromFrame, tabKey, newKey);
    const srcFrame = engine.frames.get(fromFrame);
    if (srcFrame && srcFrame.tabs.length === 0) {
      engine.removeFrame(fromFrame);
    }
    container.dispatchEvent(new CustomEvent("pages-tab-drag-out", {
      bubbles: true, composed: true,
      detail: { tabKey, fromFrame, position },
    }));
  });

  backend.onTabReorder((frameKey, tabKeys) => {
    container.dispatchEvent(new CustomEvent("pages-tab-reorder", {
      bubbles: true, composed: true,
      detail: { frameKey, tabKeys },
    }));
  });

  return {
    engine,
    dispose() {
      engine.dispose();
    },
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run wire-floating-workspace.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/wire-floating-workspace.ts packages/pages-runtime/src/wire-floating-workspace.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#303): wireFloatingWorkspace — reusable engine-backend integration

Subscribes to all backend callbacks, updates engine state,
dispatches CustomEvents on the container. Fixes stale-position
bug via engine.updatePosition/updateSize. Auto-handles close,
pin, tab-drag-out with frame lifecycle management.

Refs casehubio/casehub-pages#303"
```

---

### Task 5: Integrate wire function into activation.ts and site.ts

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/site.ts`
- Modify: `packages/pages-runtime/src/index.ts`

**Interfaces:**
- Consumes: `wireFloatingWorkspace` and `WireHandle` from `./wire-floating-workspace.js`
- Produces: Simplified activation and site event handling

- [ ] **Step 1: Update activation.ts — replace manual wiring with wireFloatingWorkspace**

Add import at the top of `activation.ts` (replace the `createFloatingFrameEngine` import):

```typescript
import { wireFloatingWorkspace } from "./wire-floating-workspace.js";
import type { WireHandle } from "./wire-floating-workspace.js";
```

Remove the now-unused import of `createFloatingFrameEngine`.

Replace the floating-workspace activation block (lines 689–744) with:

```typescript
      const wsRef = options?.floatingWorkspaceRef;

      createDockviewBackend().then((backend) => {
        const overlayContainer = document.createElement("div");
        overlayContainer.style.cssText = "position:absolute;inset:0;pointer-events:none;";
        overlayContainer.dataset.floatingWorkspaceOverlay = "";
        el.appendChild(overlayContainer);

        const defaultFactory: ContentFactory = (tab) => {
          const container = document.createElement("div");
          renderComponent(container, tab.content, {
            permissions: options?.permissions ?? ALLOW_ALL,
            onNode: callback,
          });
          return { element: container };
        };

        backend.attach(overlayContainer, defaultFactory);
        const handle = wireFloatingWorkspace(backend, overlayContainer, wsRef?.stash ?? undefined);

        if (wsRef) {
          wsRef.engine = handle.engine;
          wsRef.stash = undefined;
        }

        if (props.frames && !wsRef?.stash) {
          for (const frameConfig of props.frames) {
            handle.engine.createFrame(frameConfig);
          }
        }
      }).catch((err: unknown) => {
        console.error("Failed to initialize floating workspace backend:", err);
      });
```

- [ ] **Step 2: Update site.ts — remove engine-mutation event handlers**

In `site.ts`, find the floating workspace frame events section (around lines 942–979). Replace it with layout-save-only listeners:

```typescript
  // --- Floating workspace frame events (layout save only — wire function handles state) ---

  for (const eventName of [
    "pages-frame-close", "pages-frame-pin", "pages-frame-move",
    "pages-frame-resize", "pages-tab-drag-out", "pages-tab-reorder",
  ] as const) {
    target.addEventListener(eventName, () => {
      scheduleLayoutSave();
    }, { signal: abortController.signal });
  }
```

This replaces the per-event handlers that were calling `engine.removeFrame()`, `engine.togglePin()`, `engine.createFrame()`/`engine.moveTab()` etc. The wire function now handles all state mutations.

- [ ] **Step 3: Update index.ts — add exports**

Add to `packages/pages-runtime/src/index.ts`:

```typescript
export { wireFloatingWorkspace } from "./wire-floating-workspace.js";
export type { WireHandle } from "./wire-floating-workspace.js";
export type { FrameButtonConfig, BackendAttachOptions } from "./floating-frame-backend.js";
```

- [ ] **Step 4: Run type check and full test suite**

Run: `yarn typecheck && yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: PASS — no type errors, all tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/activation.ts packages/pages-runtime/src/site.ts packages/pages-runtime/src/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#303): integrate wireFloatingWorkspace into activation and site

activation.ts: replace 24 lines of manual callback wiring with
wireFloatingWorkspace() call.
site.ts: replace engine-mutation event handlers with layout-save-
only listeners — wire function handles all state mutations.
Export wireFloatingWorkspace, WireHandle, FrameButtonConfig,
BackendAttachOptions from index.ts.

Refs casehubio/casehub-pages#303"
```

---

### Task 6: Pin drag lock + tab-bar drag fix

**Files:**
- Modify: `packages/pages-runtime/src/dockview-backend.ts`

**Interfaces:**
- Consumes: Dockview `group.locked` API (verify availability), updated `updatePinState` method
- Produces: Pin drag lock in `updatePinState`; tab-bar empty-space drag suppression

- [ ] **Step 1: Investigate Dockview group.locked API**

Check if `group.locked` exists and what it controls. In the Dockview type definitions:

Run: `yarn workspace @casehubio/pages-runtime node -e "const dv = require('dockview-core'); console.log(Object.getOwnPropertyNames(dv.DockviewGroupPanel?.prototype ?? {}).filter(n => n.includes('lock')));"` (or check the `.d.ts` files directly).

Also check the floating overlay's properties for drag-related options.

Read: `node_modules/dockview-core/dist/dockview/dockviewGroupPanelModel.d.ts` for `locked` property.

- [ ] **Step 2: Implement pin drag lock in updatePinState**

Based on Step 1 findings, add drag lock to `updatePinState` in `dockview-backend.ts`.

**If `group.locked` works for floating groups:**

```typescript
updatePinState(key: string, pinned: boolean) {
  const group = frameGroups.get(key);
  if (!group) return;

  // Drag lock via Dockview API
  if ("locked" in group) {
    group.locked = pinned;
  }

  const el = group.element ?? group.header?.element?.closest?.(".dv-groupview");
  if (!el) return;
  const pinBtn = el.querySelector(".frame-pin-btn") as HTMLElement | null;
  if (pinBtn) {
    pinBtn.style.opacity = pinned ? "1" : "0.5";
    pinBtn.setAttribute("aria-pressed", String(pinned));
    pinBtn.classList.toggle("frame-pin-active", pinned);
  }
},
```

**If `group.locked` does NOT prevent floating group drag (fallback):**

```typescript
updatePinState(key: string, pinned: boolean) {
  const group = frameGroups.get(key);
  if (!group) return;
  const el = group.element ?? group.header?.element?.closest?.(".dv-groupview");
  if (!el) return;

  // Drag lock via pointerdown interception
  const titlebar = el.querySelector(".dv-floating-titlebar") as HTMLElement | null;
  if (titlebar) {
    const existingHandler = (titlebar as any).__pinDragLock as ((e: PointerEvent) => void) | undefined;
    if (pinned && !existingHandler) {
      const handler = (e: PointerEvent) => {
        const target = e.target as HTMLElement;
        if (target.closest(".frame-close-dot, .frame-pin-btn, .frame-extra-btn")) return;
        e.stopPropagation();
      };
      titlebar.addEventListener("pointerdown", handler, { capture: true });
      (titlebar as any).__pinDragLock = handler;
    } else if (!pinned && existingHandler) {
      titlebar.removeEventListener("pointerdown", existingHandler, { capture: true });
      delete (titlebar as any).__pinDragLock;
    }
  }

  const pinBtn = el.querySelector(".frame-pin-btn") as HTMLElement | null;
  if (pinBtn) {
    pinBtn.style.opacity = pinned ? "1" : "0.5";
    pinBtn.setAttribute("aria-pressed", String(pinned));
    pinBtn.classList.toggle("frame-pin-active", pinned);
  }
},
```

- [ ] **Step 3: Investigate Dockview tab-bar drag behavior**

Check whether `group.locked` also suppresses tab-bar empty-space drag. If not, check for other config options.

Run the examples gallery (`yarn workspace @casehubio/pages-examples run serve`) and inspect the floating frame DOM to find exact class names for the tab bar container and tabs.

- [ ] **Step 4: Implement tab-bar drag fix**

**If no Dockview config covers it — CSS fallback:**

In `createDockviewBackend()`, after the existing CSS injection block (around line 17-27), append additional CSS to the same style element:

```typescript
if (cssText) {
  const style = document.createElement("style");
  style.setAttribute(CSS_MARKER, "");
  style.textContent = cssText + `
.dv-resize-container > .dv-groupview .dv-tabs-container {
  pointer-events: none;
}
.dv-resize-container > .dv-groupview .dv-tabs-container .dv-tab {
  pointer-events: auto;
}
`;
  document.head.appendChild(style);
}
```

Note: the exact CSS selectors must be verified against Dockview's actual DOM. The `.dv-resize-container` parent scopes this to floating groups only (not docked groups). Adjust selectors based on Step 3 findings.

- [ ] **Step 5: Run full test suite and build**

Run: `yarn typecheck && yarn workspace @casehubio/pages-runtime run test -- --run && yarn build`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/dockview-backend.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#303): pin drag lock + tab-bar empty-space drag fix

Pin drag lock: [group.locked | pointerdown stopPropagation fallback]
prevents pinned frames from being dragged.
Tab-bar fix: CSS override suppresses drag on empty tab-bar area
while preserving tab click/drag interactions.

Refs casehubio/casehub-pages#303"
```

---

### Task 7: Playwright e2e tests

**Files:**
- Modify: `examples/e2e/floating-workspace.spec.ts`

**Interfaces:**
- Consumes: Running examples gallery with floating workspace sample
- Produces: e2e verification of stale-position fix, pin visual state, pin drag lock, tab-bar drag fix

- [ ] **Step 1: Read existing e2e test file**

Read: `examples/e2e/floating-workspace.spec.ts` to understand test patterns, selectors, and page setup.

- [ ] **Step 2: Add position-persists-after-drag test (stale-position fix)**

```typescript
test("position persists after drag", async ({ page }) => {
  // Navigate to floating workspace sample
  // Locate a frame, get its initial position
  // Drag it to a new position
  // Reload the page
  // Verify the frame restored to the dragged position, not the creation position
});
```

Exact selectors depend on the existing test patterns. Follow the conventions in the file.

- [ ] **Step 3: Add pin visual state test**

```typescript
test("pin button reflects toggled state", async ({ page }) => {
  // Find the pin button on a frame
  // Verify initial state: opacity ~0.5, aria-pressed="false"
  // Click pin
  // Verify: opacity 1, aria-pressed="true", has class .frame-pin-active
  // Click pin again
  // Verify: opacity ~0.5, aria-pressed="false", no .frame-pin-active
});
```

- [ ] **Step 4: Add pin drag lock test**

```typescript
test("pinned frame cannot be dragged", async ({ page }) => {
  // Pin a frame
  // Get its position
  // Attempt to drag it via the titlebar
  // Verify position unchanged
});
```

- [ ] **Step 5: Add tab-bar empty area test**

```typescript
test("empty tab-bar area does not initiate drag", async ({ page }) => {
  // Find a frame with tabs
  // Get frame position
  // Click and drag in the empty area to the right of the last tab label
  // Verify frame position unchanged
});
```

- [ ] **Step 6: Run e2e tests**

Run: `yarn build:prod && yarn workspace @casehubio/pages-examples run test`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add examples/e2e/floating-workspace.spec.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "test(#303): e2e tests for position persistence, pin state, drag lock, tab-bar fix

Refs casehubio/casehub-pages#303"
```
