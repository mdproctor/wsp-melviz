# Floating Workspace UX Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #309 — Epic: Floating workspace UX polish
**Issue group:** #304, #305, #306, #307, #308

**Goal:** Add five UX enhancements to the floating workspace: frame detach/pop-out (#304), organiser toolbar (#305), snap-to-edge (#306), keyboard shortcuts (#307), and frame transition animations (#308).

**Architecture:** All five features build on existing extension points in `pages-runtime`. New behavior is decomposed into companion modules (`frame-detach-handler.ts`, `frame-snap-preview.ts`, `frame-animations.ts`, `frame-keyboard.ts`) composed by the wire function and activation callback. The engine gains snap/detach state methods; the backend gains three additive callbacks. A prerequisite spread refactor eliminates field-stripping fragility in `FrameLayout` construction.

**Tech Stack:** TypeScript 5, Vitest, Dockview Core, Web Animations API, CSS Animations

## Global Constraints

- No new packages — all code in existing `pages-component` and `pages-runtime`
- No Lit dependency in `pages-runtime` — all DOM is plain
- `FloatingFrameBackend` interface additions are additive only (no breaking changes)
- `FrameLayout` new fields are optional (backward-compatible with saved layouts)
- All keyboard shortcuts use `Alt+` modifiers (not `Ctrl+` — browser-intercepted)
- Animations respect `prefers-reduced-motion`
- Existing tests must pass after every task

---

### Task 1: FrameLayout spread refactor (prerequisite)

**Files:**
- Modify: `packages/pages-runtime/src/floating-frame-engine.ts` (lines 92, 102, 113, 133, 148, 158, 165)
- Modify: `packages/pages-runtime/src/frame-organisers.ts` (lines 56-58)
- Test: `packages/pages-runtime/src/floating-frame-engine.test.ts`

**Interfaces:**
- Consumes: existing `FrameLayout` type from `@casehubio/pages-component`
- Produces: no new API — purely internal refactor

- [ ] **Step 1: Verify existing tests pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: all pass

- [ ] **Step 2: Refactor engine construction sites to use spread**

In `floating-frame-engine.ts`, replace each explicit field-listing construction with object spread. Seven sites:

**`showFrame` (line 92):**
```typescript
// Before:
const shown: FrameLayout = { key: frame.key, order: frame.order, position: frame.position, size: frame.size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: false, tabs: frame.tabs, activeTabKey: frame.activeTabKey };
// After:
const shown: FrameLayout = { ...frame, hidden: false };
```

**`addTab` (line 102):**
```typescript
// Before:
const updated: FrameLayout = { key: frame.key, order: frame.order, position: frame.position, size: frame.size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: frame.hidden, tabs: [...frame.tabs, tab], activeTabKey: frame.activeTabKey };
// After:
const updated: FrameLayout = { ...frame, tabs: [...frame.tabs, tab] };
```

**`removeTab` (line 113):**
```typescript
// Before:
const updated: FrameLayout = { key: frame.key, order: frame.order, position: frame.position, size: frame.size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: frame.hidden, tabs: newTabs, activeTabKey };
// After:
const updated: FrameLayout = { ...frame, tabs: newTabs, activeTabKey };
```

**`setActiveTab` (line 133):**
```typescript
// Before:
const updated: FrameLayout = { key: frame.key, order: frame.order, position: frame.position, size: frame.size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: frame.hidden, tabs: frame.tabs, activeTabKey: tabKey };
// After:
const updated: FrameLayout = { ...frame, activeTabKey: tabKey };
```

**`togglePin` (line 148):**
```typescript
// Before:
const updated: FrameLayout = { key: frame.key, order: frame.order, position: frame.position, size: frame.size, zIndex: frame.zIndex, pinned: !frame.pinned, hidden: frame.hidden, tabs: frame.tabs, activeTabKey: frame.activeTabKey };
// After:
const updated: FrameLayout = { ...frame, pinned: !frame.pinned };
```

**`updatePosition` (line 158):**
```typescript
// Before:
frames.set(key, { key: frame.key, order: frame.order, position: pos, size: frame.size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: frame.hidden, tabs: frame.tabs, activeTabKey: frame.activeTabKey });
// After:
frames.set(key, { ...frame, position: pos });
```

**`updateSize` (line 165):**
```typescript
// Before:
frames.set(key, { key: frame.key, order: frame.order, position: frame.position, size, zIndex: frame.zIndex, pinned: frame.pinned, hidden: frame.hidden, tabs: frame.tabs, activeTabKey: frame.activeTabKey });
// After:
frames.set(key, { ...frame, size });
```

- [ ] **Step 3: Refactor `withLayout` in frame-organisers.ts**

In `frame-organisers.ts` (line 56-58):
```typescript
// Before:
function withLayout(f: FrameLayout, pos: { x: number; y: number }, sz: { width: number; height: number }): FrameLayout {
  return { key: f.key, order: f.order, zIndex: f.zIndex, pinned: f.pinned, hidden: f.hidden, tabs: f.tabs, activeTabKey: f.activeTabKey, position: pos, size: sz };
}
// After:
function withLayout(f: FrameLayout, pos: { x: number; y: number }, sz: { width: number; height: number }): FrameLayout {
  return { ...f, position: pos, size: sz };
}
```

- [ ] **Step 4: Run all tests to verify no regressions**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: all pass (behavior-preserving refactor)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/floating-frame-engine.ts packages/pages-runtime/src/frame-organisers.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor: use spread for FrameLayout construction — prevent field stripping

Refs casehubio/casehub-pages#309"
```

---

### Task 2: Type additions and snap pure functions

**Files:**
- Modify: `packages/pages-component/src/model/types.ts`
- Modify: `packages/pages-component/src/model/index.ts`
- Modify: `packages/pages-runtime/src/frame-boundaries.ts`
- Test: `packages/pages-runtime/src/frame-boundaries.test.ts`

**Interfaces:**
- Consumes: nothing new
- Produces:
  - `SnapZone` type: `"left" | "right" | "top" | "bottom" | "top-left" | "top-right" | "bottom-left" | "bottom-right" | "full"`
  - `FrameLayout.detached?: boolean`
  - `FrameLayout.snappedZone?: SnapZone`
  - `snapToZone(dragPosition: Position, containerSize: Size, threshold?: number): SnapZone | null`
  - `zoneToRect(zone: SnapZone, containerSize: Size, gap?: number): { position: Position; size: Size }`

- [ ] **Step 1: Add `SnapZone` type and extend `FrameLayout`**

In `packages/pages-component/src/model/types.ts`, after `FrameConfig` (line 75), add:
```typescript
export type SnapZone = "left" | "right" | "top" | "bottom"
  | "top-left" | "top-right" | "bottom-left" | "bottom-right"
  | "full";
```

Add two fields to `FrameLayout` (after line 92, `activeTabKey`):
```typescript
  readonly detached?: boolean;
  readonly snappedZone?: SnapZone;
```

In `packages/pages-component/src/model/index.ts`, add `SnapZone` to the re-export from `types.js`.

- [ ] **Step 2: Write failing tests for `snapToZone`**

In `packages/pages-runtime/src/frame-boundaries.test.ts`, add:
```typescript
import { snapToZone, zoneToRect } from "./frame-boundaries.js";

describe("snapToZone", () => {
  const container = { width: 1000, height: 800 };

  it("returns null when far from edges", () => {
    expect(snapToZone({ x: 500, y: 400 }, container)).toBeNull();
  });

  it("detects left edge", () => {
    expect(snapToZone({ x: 10, y: 400 }, container)).toBe("left");
  });

  it("detects right edge", () => {
    expect(snapToZone({ x: 990, y: 400 }, container)).toBe("right");
  });

  it("detects top edge", () => {
    expect(snapToZone({ x: 500, y: 10 }, container)).toBe("top");
  });

  it("detects bottom edge", () => {
    expect(snapToZone({ x: 500, y: 790 }, container)).toBe("bottom");
  });

  it("detects top-left corner (corner priority over edge)", () => {
    expect(snapToZone({ x: 10, y: 10 }, container)).toBe("top-left");
  });

  it("detects bottom-right corner", () => {
    expect(snapToZone({ x: 990, y: 790 }, container)).toBe("bottom-right");
  });

  it("respects custom threshold", () => {
    expect(snapToZone({ x: 50, y: 400 }, container, 20)).toBeNull();
    expect(snapToZone({ x: 10, y: 400 }, container, 20)).toBe("left");
  });
});

describe("zoneToRect", () => {
  const container = { width: 1000, height: 800 };

  it("computes left half", () => {
    const r = zoneToRect("left", container);
    expect(r.position).toEqual({ x: 0, y: 0 });
    expect(r.size).toEqual({ width: 496, height: 800 });
  });

  it("computes right half", () => {
    const r = zoneToRect("right", container);
    expect(r.position).toEqual({ x: 504, y: 0 });
    expect(r.size).toEqual({ width: 496, height: 800 });
  });

  it("computes full", () => {
    const r = zoneToRect("full", container);
    expect(r.position).toEqual({ x: 0, y: 0 });
    expect(r.size).toEqual({ width: 1000, height: 800 });
  });

  it("computes top-left quarter", () => {
    const r = zoneToRect("top-left", container);
    expect(r.position).toEqual({ x: 0, y: 0 });
    expect(r.size).toEqual({ width: 496, height: 396 });
  });

  it("respects custom gap", () => {
    const r = zoneToRect("left", container, 16);
    expect(r.size).toEqual({ width: 492, height: 800 });
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-boundaries`
Expected: FAIL — `snapToZone` and `zoneToRect` not exported

- [ ] **Step 4: Implement `snapToZone` and `zoneToRect`**

In `packages/pages-runtime/src/frame-boundaries.ts`, add:
```typescript
import type { SnapZone } from "@casehubio/pages-component";

const DEFAULT_THRESHOLD = 40;

export function snapToZone(
  dragPosition: Position,
  containerSize: Size,
  threshold = DEFAULT_THRESHOLD,
): SnapZone | null {
  const nearLeft = dragPosition.x <= threshold;
  const nearRight = dragPosition.x >= containerSize.width - threshold;
  const nearTop = dragPosition.y <= threshold;
  const nearBottom = dragPosition.y >= containerSize.height - threshold;

  if (nearLeft && nearTop) return "top-left";
  if (nearRight && nearTop) return "top-right";
  if (nearLeft && nearBottom) return "bottom-left";
  if (nearRight && nearBottom) return "bottom-right";
  if (nearLeft) return "left";
  if (nearRight) return "right";
  if (nearTop) return "top";
  if (nearBottom) return "bottom";
  return null;
}

const DEFAULT_GAP = 8;

export function zoneToRect(
  zone: SnapZone,
  containerSize: Size,
  gap = DEFAULT_GAP,
): { position: Position; size: Size } {
  const halfW = Math.floor((containerSize.width - gap) / 2);
  const halfH = Math.floor((containerSize.height - gap) / 2);
  const rightX = containerSize.width - halfW;
  const bottomY = containerSize.height - halfH;

  switch (zone) {
    case "left": return { position: { x: 0, y: 0 }, size: { width: halfW, height: containerSize.height } };
    case "right": return { position: { x: rightX, y: 0 }, size: { width: halfW, height: containerSize.height } };
    case "top": return { position: { x: 0, y: 0 }, size: { width: containerSize.width, height: halfH } };
    case "bottom": return { position: { x: 0, y: bottomY }, size: { width: containerSize.width, height: halfH } };
    case "top-left": return { position: { x: 0, y: 0 }, size: { width: halfW, height: halfH } };
    case "top-right": return { position: { x: rightX, y: 0 }, size: { width: halfW, height: halfH } };
    case "bottom-left": return { position: { x: 0, y: bottomY }, size: { width: halfW, height: halfH } };
    case "bottom-right": return { position: { x: rightX, y: bottomY }, size: { width: halfW, height: halfH } };
    case "full": return { position: { x: 0, y: 0 }, size: { width: containerSize.width, height: containerSize.height } };
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-boundaries`
Expected: all pass

- [ ] **Step 6: Typecheck**

Run: `yarn typecheck`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/types.ts packages/pages-component/src/model/index.ts packages/pages-runtime/src/frame-boundaries.ts packages/pages-runtime/src/frame-boundaries.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#306): add SnapZone type, snapToZone and zoneToRect pure functions

Refs casehubio/casehub-pages#306"
```

---

### Task 3: Engine snap and detach methods

**Files:**
- Modify: `packages/pages-runtime/src/floating-frame-engine.ts`
- Test: `packages/pages-runtime/src/floating-frame-engine.test.ts`

**Interfaces:**
- Consumes: `SnapZone` type, `zoneToRect()` from Task 2
- Produces:
  - `engine.setDetached(key: string, detached: boolean): void`
  - `engine.snapFrame(key: string, zone: SnapZone, canvasSize: { width: number; height: number }): void`
  - `engine.unsnapFrame(key: string): void`
  - `engine.recomputeSnappedFrames(canvasSize: { width: number; height: number }): void`

- [ ] **Step 1: Write failing tests for `setDetached`**

```typescript
describe("setDetached", () => {
  it("marks frame as detached", () => {
    engine.createFrame(makeFrameConfig("f1"));
    engine.setDetached("f1", true);
    expect(engine.frames.get("f1")!.detached).toBe(true);
  });

  it("clears detached flag", () => {
    engine.createFrame(makeFrameConfig("f1"));
    engine.setDetached("f1", true);
    engine.setDetached("f1", false);
    expect(engine.frames.get("f1")!.detached).toBe(false);
  });

  it("is no-op for unknown key", () => {
    engine.setDetached("unknown", true);
    expect(engine.frames.size).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine`
Expected: FAIL — `setDetached` not in engine interface

- [ ] **Step 3: Implement `setDetached`**

Add to `FloatingFrameEngine` interface (after `updateSize`):
```typescript
setDetached(key: string, detached: boolean): void;
```

Add implementation in the engine object:
```typescript
setDetached(key: string, detached: boolean) {
  assertAlive();
  const frame = frames.get(key);
  if (!frame) return;
  frames.set(key, { ...frame, detached });
},
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine`
Expected: PASS

- [ ] **Step 5: Write failing tests for snap methods**

```typescript
describe("snapFrame / unsnapFrame", () => {
  it("sets snappedZone and updates position/size", () => {
    engine.createFrame(makeFrameConfig("f1"));
    engine.snapFrame("f1", "left", { width: 1000, height: 800 });
    const f = engine.frames.get("f1")!;
    expect(f.snappedZone).toBe("left");
    expect(f.position).toEqual({ x: 0, y: 0 });
    expect(f.size.width).toBe(496);
    expect(backend.updatePosition).toHaveBeenCalledWith("f1", { x: 0, y: 0 });
    expect(backend.updateSize).toHaveBeenCalledWith("f1", expect.objectContaining({ width: 496 }));
  });

  it("captures pre-snap state and restores on unsnap", () => {
    engine.createFrame({ ...makeFrameConfig("f1"), position: { x: 100, y: 200 }, size: { width: 300, height: 250 } });
    engine.snapFrame("f1", "right", { width: 1000, height: 800 });
    engine.unsnapFrame("f1");
    const f = engine.frames.get("f1")!;
    expect(f.snappedZone).toBeUndefined();
    expect(f.position).toEqual({ x: 100, y: 200 });
    expect(f.size).toEqual({ width: 300, height: 250 });
  });

  it("unsnapFrame is no-op when not snapped", () => {
    engine.createFrame(makeFrameConfig("f1"));
    const before = engine.frames.get("f1")!;
    engine.unsnapFrame("f1");
    expect(engine.frames.get("f1")!.position).toEqual(before.position);
  });
});

describe("recomputeSnappedFrames", () => {
  it("recomputes position/size for snapped frames on resize", () => {
    engine.createFrame(makeFrameConfig("f1"));
    engine.snapFrame("f1", "left", { width: 1000, height: 800 });
    engine.recomputeSnappedFrames({ width: 1200, height: 900 });
    const f = engine.frames.get("f1")!;
    expect(f.size.width).toBe(596);
    expect(f.size.height).toBe(900);
  });

  it("does not touch unsnapped frames", () => {
    engine.createFrame({ ...makeFrameConfig("f1"), position: { x: 100, y: 100 } });
    engine.recomputeSnappedFrames({ width: 1200, height: 900 });
    expect(engine.frames.get("f1")!.position).toEqual({ x: 100, y: 100 });
  });
});
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine`
Expected: FAIL — `snapFrame`, `unsnapFrame`, `recomputeSnappedFrames` not defined

- [ ] **Step 7: Implement snap methods**

Add to `FloatingFrameEngine` interface:
```typescript
snapFrame(key: string, zone: SnapZone, canvasSize: { width: number; height: number }): void;
unsnapFrame(key: string): void;
recomputeSnappedFrames(canvasSize: { width: number; height: number }): void;
```

Add imports and a `preSnapState` map inside `createFloatingFrameEngine`:
```typescript
import { zoneToRect } from "./frame-boundaries.js";
import type { SnapZone } from "@casehubio/pages-component";

// Inside createFloatingFrameEngine, after `let nextOrder = 0;`:
const preSnapState = new Map<string, { position: { x: number; y: number }; size: { width: number; height: number } }>();
```

Add implementations:
```typescript
snapFrame(key: string, zone: SnapZone, canvasSize: { width: number; height: number }) {
  assertAlive();
  const frame = frames.get(key);
  if (!frame) return;
  if (!frame.snappedZone) {
    preSnapState.set(key, { position: frame.position, size: frame.size });
  }
  const rect = zoneToRect(zone, canvasSize);
  frames.set(key, { ...frame, snappedZone: zone, position: rect.position, size: rect.size });
  backend.updatePosition(key, rect.position);
  backend.updateSize(key, rect.size);
},

unsnapFrame(key: string) {
  assertAlive();
  const frame = frames.get(key);
  if (!frame || !frame.snappedZone) return;
  const saved = preSnapState.get(key);
  preSnapState.delete(key);
  if (saved) {
    frames.set(key, { ...frame, snappedZone: undefined, position: saved.position, size: saved.size });
    backend.updatePosition(key, saved.position);
    backend.updateSize(key, saved.size);
  } else {
    frames.set(key, { ...frame, snappedZone: undefined });
  }
},

recomputeSnappedFrames(canvasSize: { width: number; height: number }) {
  assertAlive();
  for (const [key, frame] of frames) {
    if (!frame.snappedZone) continue;
    const rect = zoneToRect(frame.snappedZone, canvasSize);
    frames.set(key, { ...frame, position: rect.position, size: rect.size });
    backend.updatePosition(key, rect.position);
    backend.updateSize(key, rect.size);
  }
},
```

- [ ] **Step 8: Run all engine tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine`
Expected: all pass

- [ ] **Step 9: Update `mockBackend()` in test file**

If the `mockBackend()` function in tests needs updating for new interface methods (from Task 4), add stubs now. The engine tests mock the backend — new callbacks added in Task 4 must be present:
```typescript
// Add to mockBackend():
onFrameDragMove: vi.fn(), onTitlebarDoubleClick: vi.fn(), getFrameElement: vi.fn(() => null),
```

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/floating-frame-engine.ts packages/pages-runtime/src/floating-frame-engine.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#304,#306): add setDetached, snapFrame, unsnapFrame, recomputeSnappedFrames to engine

Refs casehubio/casehub-pages#304, casehubio/casehub-pages#306"
```

---

### Task 4: Backend interface additions

**Files:**
- Modify: `packages/pages-runtime/src/floating-frame-backend.ts`
- Modify: `packages/pages-runtime/src/dockview-backend.ts`
- Test: `packages/pages-runtime/src/dockview-backend.test.ts`

**Interfaces:**
- Consumes: Dockview `fg.overlay.onDidChange` API
- Produces:
  - `backend.onFrameDragMove(cb: (key: string, pos: { x: number; y: number }) => void): void`
  - `backend.onTitlebarDoubleClick(cb: (key: string) => void): void`
  - `backend.getFrameElement(key: string): HTMLElement | null`

- [ ] **Step 1: Add methods to `FloatingFrameBackend` interface**

In `floating-frame-backend.ts`, after `onFramePin` (line 33):
```typescript
  onFrameDragMove(cb: (key: string, pos: { x: number; y: number }) => void): void;
  onTitlebarDoubleClick(cb: (key: string) => void): void;
  getFrameElement(key: string): HTMLElement | null;
```

- [ ] **Step 2: Write failing tests for dockview-backend**

In `dockview-backend.test.ts`, add test cases for:
- `onTitlebarDoubleClick` fires callback when titlebar is double-clicked
- `getFrameElement` returns the group element for a known key and `null` for unknown
- `onFrameDragMove` callback array exists and fires during drag

- [ ] **Step 3: Implement in dockview-backend.ts**

Add callback arrays alongside existing ones:
```typescript
const frameDragMoveCallbacks: Array<(key: string, pos: { x: number; y: number }) => void> = [];
const titlebarDoubleClickCallbacks: Array<(key: string) => void> = [];
```

In `injectFrameChrome`, add dblclick listener on titlebar:
```typescript
titlebar.addEventListener("dblclick", (e) => {
  if ((e.target as HTMLElement).closest(".frame-close-dot, .frame-pin-btn, .frame-extra-btn")) return;
  for (const cb of titlebarDoubleClickCallbacks) cb(frameKey);
});
```

In the `fg.overlay.onDidChange` subscription (where `onDidChangeEnd` is already subscribed), add continuous drag reporting:
```typescript
// Alongside existing onDidChangeEnd subscription:
fg.overlay.onDidChange(() => {
  const el = fg.element ?? fg.header?.element?.closest?.(".dv-groupview");
  if (!el || !container) return;
  const rect = el.getBoundingClientRect();
  const containerRect = container.getBoundingClientRect();
  const pos = { x: rect.left - containerRect.left, y: rect.top - containerRect.top };
  for (const cb of frameDragMoveCallbacks) cb(frameKey, pos);
});
```

Implement interface methods:
```typescript
onFrameDragMove(cb) { frameDragMoveCallbacks.push(cb); },
onTitlebarDoubleClick(cb) { titlebarDoubleClickCallbacks.push(cb); },
getFrameElement(key: string): HTMLElement | null {
  const group = frameGroups.get(key);
  if (!group) return null;
  return (group.element ?? group.header?.element?.closest?.(".dv-groupview")) as HTMLElement | null;
},
```

Clear callback arrays in `dispose()`:
```typescript
frameDragMoveCallbacks.length = 0;
titlebarDoubleClickCallbacks.length = 0;
```

- [ ] **Step 4: Run backend tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run dockview-backend`
Expected: all pass

- [ ] **Step 5: Typecheck**

Run: `yarn typecheck`
Expected: no errors

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/floating-frame-backend.ts packages/pages-runtime/src/dockview-backend.ts packages/pages-runtime/src/dockview-backend.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#306,#308): add onFrameDragMove, onTitlebarDoubleClick, getFrameElement to backend

Refs casehubio/casehub-pages#306, casehubio/casehub-pages#308"
```

---

### Task 5: Keyboard shortcuts module (#307)

**Files:**
- Create: `packages/pages-runtime/src/frame-keyboard.ts`
- Create: `packages/pages-runtime/src/frame-keyboard.test.ts`

**Interfaces:**
- Consumes: `FloatingFrameEngine` (existing: `focusDirection`, `removeFrame`, `togglePin`, `bringToFront`, `frames`)
- Produces: `createFrameKeyboardHandler(engine: FloatingFrameEngine, container: HTMLElement, signal: AbortSignal): void`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, beforeEach, vi, afterEach } from "vitest";
import { createFrameKeyboardHandler } from "./frame-keyboard.js";
import type { FloatingFrameEngine } from "./floating-frame-engine.js";

function mockEngine(): FloatingFrameEngine {
  return {
    frames: new Map(),
    focusDirection: vi.fn(() => "f2"),
    removeFrame: vi.fn(),
    togglePin: vi.fn(),
    bringToFront: vi.fn(),
    createFrame: vi.fn(),
    hideFrame: vi.fn(), showFrame: vi.fn(),
    addTab: vi.fn(), removeTab: vi.fn(), moveTab: vi.fn(), setActiveTab: vi.fn(),
    updatePosition: vi.fn(), updateSize: vi.fn(),
    setDetached: vi.fn(),
    snapFrame: vi.fn(), unsnapFrame: vi.fn(), recomputeSnappedFrames: vi.fn(),
    applyOrganiser: vi.fn(),
    captureLayout: vi.fn(() => []), restoreLayout: vi.fn(),
    dispose: vi.fn(),
  } as unknown as FloatingFrameEngine;
}

describe("createFrameKeyboardHandler", () => {
  let engine: FloatingFrameEngine;
  let container: HTMLElement;
  let controller: AbortController;

  beforeEach(() => {
    engine = mockEngine();
    container = document.createElement("div");
    controller = new AbortController();
    createFrameKeyboardHandler(engine, container, controller.signal);
  });

  afterEach(() => { controller.abort(); });

  it("Alt+ArrowRight calls focusDirection", () => {
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "ArrowRight", altKey: true }));
    expect(engine.focusDirection).toHaveBeenCalledWith("right");
  });

  it("Alt+W calls removeFrame on focused frame", () => {
    // Focus a frame first via focusDirection
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "ArrowRight", altKey: true }));
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "w", altKey: true }));
    expect(engine.removeFrame).toHaveBeenCalledWith("f2");
  });

  it("Alt+W is no-op when no frame is focused", () => {
    (engine.focusDirection as ReturnType<typeof vi.fn>).mockReturnValue(null);
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "w", altKey: true }));
    expect(engine.removeFrame).not.toHaveBeenCalled();
  });

  it("cleans up on abort", () => {
    controller.abort();
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "ArrowRight", altKey: true }));
    expect(engine.focusDirection).not.toHaveBeenCalled();
  });

  it("ignores shortcuts when in text input", () => {
    const input = document.createElement("input");
    document.body.appendChild(input);
    input.focus();
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "ArrowRight", altKey: true }));
    expect(engine.focusDirection).not.toHaveBeenCalled();
    input.remove();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-keyboard`
Expected: FAIL — module not found

- [ ] **Step 3: Implement `frame-keyboard.ts`**

```typescript
import type { FloatingFrameEngine } from "./floating-frame-engine.js";

const INPUT_TAGS = new Set(["INPUT", "TEXTAREA", "SELECT"]);

function isInTextInput(): boolean {
  const active = document.activeElement;
  if (!active) return false;
  if (INPUT_TAGS.has(active.tagName)) return true;
  if ((active as HTMLElement).isContentEditable) return true;
  const root = active.shadowRoot;
  if (root) {
    const inner = root.activeElement;
    if (inner && (INPUT_TAGS.has(inner.tagName) || (inner as HTMLElement).isContentEditable)) return true;
  }
  return false;
}

export function createFrameKeyboardHandler(
  engine: FloatingFrameEngine,
  container: HTMLElement,
  signal: AbortSignal,
): void {
  let focusedKey: string | null = null;

  container.addEventListener("pages-frame-focus", ((e: Event) => {
    focusedKey = (e as CustomEvent<{ frameKey: string }>).detail.frameKey;
  }) as EventListener, { signal });

  function handleKeydown(e: KeyboardEvent): void {
    if (!e.altKey || e.ctrlKey || e.metaKey) return;
    if (isInTextInput()) return;

    const key = e.key;

    if (key === "ArrowUp" || key === "ArrowDown" || key === "ArrowLeft" || key === "ArrowRight") {
      e.preventDefault();
      const dir = key.replace("Arrow", "").toLowerCase() as "up" | "down" | "left" | "right";
      const target = engine.focusDirection(dir);
      if (target) {
        focusedKey = target;
        engine.bringToFront(target);
      }
      return;
    }

    if (key >= "1" && key <= "9") {
      e.preventDefault();
      const index = parseInt(key) - 1;
      const visible = [...engine.frames.values()].filter(f => !f.hidden).sort((a, b) => a.order - b.order);
      if (index < visible.length) {
        focusedKey = visible[index]!.key;
        engine.bringToFront(focusedKey);
      }
      return;
    }

    if (key === "]" || key === "[") {
      e.preventDefault();
      const visible = [...engine.frames.values()].filter(f => !f.hidden).sort((a, b) => a.order - b.order);
      if (visible.length === 0) return;
      const currentIndex = focusedKey ? visible.findIndex(f => f.key === focusedKey) : -1;
      const next = key === "]"
        ? (currentIndex + 1) % visible.length
        : (currentIndex - 1 + visible.length) % visible.length;
      focusedKey = visible[next]!.key;
      engine.bringToFront(focusedKey);
      return;
    }

    if (key === "w" && focusedKey) {
      e.preventDefault();
      engine.removeFrame(focusedKey);
      focusedKey = null;
      return;
    }

    if (key === "p" && focusedKey) {
      e.preventDefault();
      engine.togglePin(focusedKey);
      return;
    }
  }

  document.addEventListener("keydown", handleKeydown, { signal });
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-keyboard`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/frame-keyboard.ts packages/pages-runtime/src/frame-keyboard.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#307): keyboard shortcuts for floating workspace frame management

Alt+Arrow spatial nav, Alt+1-9 index focus, Alt+[/] cycle, Alt+W close, Alt+P pin.

Refs casehubio/casehub-pages#307"
```

---

### Task 6: Organiser toolbar (#305)

**Files:**
- Create: `packages/pages-runtime/src/organiser-toolbar.ts`
- Create: `packages/pages-runtime/src/organiser-toolbar.test.ts`

**Interfaces:**
- Consumes: `FloatingFrameEngine.applyOrganiser(preset, canvasSize)`
- Produces: `createOrganiserToolbar(engine: FloatingFrameEngine, overlayContainer: HTMLElement, parentEl: HTMLElement, signal: AbortSignal): HTMLElement`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";
import { createOrganiserToolbar } from "./organiser-toolbar.js";
import type { FloatingFrameEngine } from "./floating-frame-engine.js";

function mockEngine(): FloatingFrameEngine {
  // same mock as Task 5
  return { /* ... */ applyOrganiser: vi.fn() } as unknown as FloatingFrameEngine;
}

describe("createOrganiserToolbar", () => {
  let engine: FloatingFrameEngine;
  let overlay: HTMLElement;
  let parent: HTMLElement;
  let controller: AbortController;
  let toolbar: HTMLElement;

  beforeEach(() => {
    engine = mockEngine();
    overlay = document.createElement("div");
    Object.defineProperty(overlay, "clientWidth", { value: 1000 });
    Object.defineProperty(overlay, "clientHeight", { value: 800 });
    parent = document.createElement("div");
    controller = new AbortController();
    toolbar = createOrganiserToolbar(engine, overlay, parent, controller.signal);
  });

  it("creates toolbar with 5 preset buttons", () => {
    expect(toolbar.querySelectorAll("button").length).toBe(5);
  });

  it("starts hidden", () => {
    expect(toolbar.style.display).toBe("none");
  });

  it("calls applyOrganiser on button click", () => {
    const gridBtn = toolbar.querySelector("[data-preset='grid']") as HTMLButtonElement;
    gridBtn.click();
    expect(engine.applyOrganiser).toHaveBeenCalledWith("grid", { width: 1000, height: 800 });
  });

  it("shows toolbar when pages-frame-create fires and >1 frame", () => {
    // Simulate 2 frames
    (engine as any).frames = new Map([["f1", { hidden: false }], ["f2", { hidden: false }]]);
    parent.dispatchEvent(new CustomEvent("pages-frame-create", { bubbles: true }));
    expect(toolbar.style.display).toBe("flex");
  });

  it("hides toolbar when <=1 frame remains", () => {
    (engine as any).frames = new Map([["f1", { hidden: false }]]);
    parent.dispatchEvent(new CustomEvent("pages-frame-close", { bubbles: true }));
    expect(toolbar.style.display).toBe("none");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run organiser-toolbar`
Expected: FAIL — module not found

- [ ] **Step 3: Implement `organiser-toolbar.ts`**

```typescript
import type { FloatingFrameEngine } from "./floating-frame-engine.js";
import type { Preset } from "./frame-organisers.js";

const PRESETS: Array<{ key: Preset; icon: string; title: string }> = [
  { key: "side-by-side", icon: "⬜⬜", title: "Side by side" },
  { key: "stacked", icon: "☰", title: "Stacked" },
  { key: "grid", icon: "⊞", title: "Grid" },
  { key: "main-sidebar", icon: "⬜▫", title: "Main + Sidebar" },
  { key: "focus", icon: "◻", title: "Focus" },
];

export function createOrganiserToolbar(
  engine: FloatingFrameEngine,
  overlayContainer: HTMLElement,
  parentEl: HTMLElement,
  signal: AbortSignal,
): HTMLElement {
  const toolbar = document.createElement("div");
  toolbar.className = "pages-organiser-toolbar";
  toolbar.dataset.floatingWorkspaceToolbar = "";
  toolbar.style.cssText = "display:none;padding:4px 8px;gap:4px;align-items:center;background:var(--pages-neutral-2);border-bottom:1px solid var(--pages-neutral-4);";

  let activePreset: string | null = null;

  for (const preset of PRESETS) {
    const btn = document.createElement("button");
    btn.dataset.preset = preset.key;
    btn.title = preset.title;
    btn.textContent = preset.icon;
    btn.style.cssText = "cursor:pointer;border:1px solid var(--pages-neutral-4);border-radius:var(--pages-radius-sm, 4px);background:transparent;padding:2px 6px;color:var(--pages-neutral-9);font-size:12px;";
    btn.addEventListener("click", () => {
      const canvasSize = { width: overlayContainer.clientWidth, height: overlayContainer.clientHeight };
      engine.applyOrganiser(preset.key, canvasSize);
      if (activePreset) {
        toolbar.querySelector(`[data-preset="${activePreset}"]`)?.classList.remove("preset-active");
      }
      btn.classList.add("preset-active");
      btn.style.background = "var(--pages-accent-3)";
      activePreset = preset.key;
      parentEl.dispatchEvent(new CustomEvent("pages-frame-organise", {
        bubbles: true, composed: true, detail: { preset: preset.key },
      }));
    }, { signal });
    toolbar.appendChild(btn);
  }

  function updateVisibility(): void {
    const visibleCount = [...engine.frames.values()].filter(f => !f.hidden).length;
    toolbar.style.display = visibleCount > 1 ? "flex" : "none";
  }

  for (const eventName of ["pages-frame-create", "pages-frame-close", "pages-frame-show", "pages-frame-hide"]) {
    parentEl.addEventListener(eventName, updateVisibility, { signal });
  }

  return toolbar;
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run organiser-toolbar`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/organiser-toolbar.ts packages/pages-runtime/src/organiser-toolbar.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#305): organiser toolbar for floating workspace preset layout

Fill & Arrange toolbar with side-by-side, stacked, grid, main-sidebar, focus presets.

Refs casehubio/casehub-pages#305"
```

---

### Task 7: Frame animations module (#308)

**Files:**
- Create: `packages/pages-runtime/src/frame-animations.ts`
- Create: `packages/pages-runtime/src/frame-animations.test.ts`

**Interfaces:**
- Consumes: `backend.getFrameElement(key)` from Task 4
- Produces:
  - `injectAnimationStyles(): void` — idempotent CSS injection
  - `animateFrameEnter(el: HTMLElement): void`
  - `animateFrameExit(el: HTMLElement): Promise<void>` — resolves after animation or immediately if reduced motion
  - `animateFrameMove(el: HTMLElement, from: { x: number; y: number; w: number; h: number }, to: { x: number; y: number; w: number; h: number }): void`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { injectAnimationStyles, animateFrameEnter, animateFrameExit } from "./frame-animations.js";

describe("injectAnimationStyles", () => {
  it("injects style element with marker", () => {
    injectAnimationStyles();
    const style = document.querySelector("[data-pages-frame-animations]");
    expect(style).not.toBeNull();
    expect(style!.textContent).toContain("frame-enter");
    expect(style!.textContent).toContain("prefers-reduced-motion");
  });

  it("is idempotent", () => {
    injectAnimationStyles();
    injectAnimationStyles();
    expect(document.querySelectorAll("[data-pages-frame-animations]").length).toBe(1);
  });
});

describe("animateFrameEnter", () => {
  it("adds and removes frame-entering class", async () => {
    const el = document.createElement("div");
    animateFrameEnter(el);
    expect(el.classList.contains("frame-entering")).toBe(true);
    el.dispatchEvent(new Event("animationend"));
    expect(el.classList.contains("frame-entering")).toBe(false);
  });
});

describe("animateFrameExit", () => {
  it("adds frame-exiting class and resolves after animationend", async () => {
    const el = document.createElement("div");
    const promise = animateFrameExit(el);
    expect(el.classList.contains("frame-exiting")).toBe(true);
    el.dispatchEvent(new Event("animationend"));
    await promise;
    expect(el.classList.contains("frame-exiting")).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify failure**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-animations`
Expected: FAIL

- [ ] **Step 3: Implement `frame-animations.ts`**

```typescript
const CSS_MARKER = "data-pages-frame-animations";

const ANIMATION_CSS = `
@keyframes frame-enter {
  from { opacity: 0; transform: scale(0.95); }
  to { opacity: 1; transform: scale(1); }
}
@keyframes frame-exit {
  from { opacity: 1; transform: scale(1); }
  to { opacity: 0; transform: scale(0.95); }
}
.frame-entering {
  animation: frame-enter var(--pages-frame-transition-duration, 200ms) ease forwards;
}
.frame-exiting {
  animation: frame-exit var(--pages-frame-transition-duration, 200ms) ease forwards;
  pointer-events: none;
}
@media (prefers-reduced-motion: reduce) {
  .frame-entering, .frame-exiting { animation: none; }
}
`;

let prefersReducedMotion: MediaQueryList | null = null;

function reducedMotion(): boolean {
  if (!prefersReducedMotion) {
    prefersReducedMotion = window.matchMedia("(prefers-reduced-motion: reduce)");
  }
  return prefersReducedMotion.matches;
}

export function injectAnimationStyles(): void {
  if (document.querySelector(`[${CSS_MARKER}]`)) return;
  const style = document.createElement("style");
  style.setAttribute(CSS_MARKER, "");
  style.textContent = ANIMATION_CSS;
  document.head.appendChild(style);
}

export function animateFrameEnter(el: HTMLElement): void {
  if (reducedMotion()) return;
  el.classList.add("frame-entering");
  el.addEventListener("animationend", () => el.classList.remove("frame-entering"), { once: true });
}

export function animateFrameExit(el: HTMLElement): Promise<void> {
  if (reducedMotion()) return Promise.resolve();
  el.classList.add("frame-exiting");
  return new Promise(resolve => {
    el.addEventListener("animationend", () => resolve(), { once: true });
  });
}

export function animateFrameMove(
  el: HTMLElement,
  from: { x: number; y: number; w: number; h: number },
  to: { x: number; y: number; w: number; h: number },
): void {
  if (reducedMotion()) return;
  const duration = parseInt(getComputedStyle(el).getPropertyValue("--pages-frame-transition-duration") || "200");
  el.animate(
    [
      { left: `${from.x}px`, top: `${from.y}px`, width: `${from.w}px`, height: `${from.h}px` },
      { left: `${to.x}px`, top: `${to.y}px`, width: `${to.w}px`, height: `${to.h}px` },
    ],
    { duration, easing: "ease" },
  );
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-animations`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/frame-animations.ts packages/pages-runtime/src/frame-animations.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#308): frame transition animations — enter/exit CSS, Web Animations API

CSS keyframe animations for frame enter/exit, Web Animations API for preset
and snap transitions. Respects prefers-reduced-motion.

Refs casehubio/casehub-pages#308"
```

---

### Task 8: Frame detach handler (#304)

**Files:**
- Create: `packages/pages-runtime/src/frame-detach-handler.ts`
- Create: `packages/pages-runtime/src/frame-detach-handler.test.ts`

**Interfaces:**
- Consumes: `FloatingFrameEngine` (hideFrame, showFrame, setDetached, frames), `ContentFactory`, `EventRelay`, `DetachRegistry`, `copyStyles`
- Produces: `createFrameDetachHandler(engine, container, contentFactory, signal): FrameDetachHandler`
  - `FrameDetachHandler.detach(frameKey: string): void`
  - `FrameDetachHandler.reattach(frameKey: string): void`
  - `FrameDetachHandler.dispose(): void`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, beforeEach, vi, afterEach } from "vitest";
import { createFrameDetachHandler } from "./frame-detach-handler.js";
import type { FloatingFrameEngine } from "./floating-frame-engine.js";
import type { ContentFactory } from "@casehubio/pages-component";

// Mock window.open
const mockChildWindow = {
  document: { title: "", body: { style: { margin: "", width: "", height: "", overflow: "" }, appendChild: vi.fn() }, head: { appendChild: vi.fn() } },
  addEventListener: vi.fn(),
  focus: vi.fn(),
  close: vi.fn(),
  closed: false,
};

describe("createFrameDetachHandler", () => {
  let engine: FloatingFrameEngine;
  let container: HTMLElement;
  let factory: ContentFactory;
  let controller: AbortController;

  beforeEach(() => {
    engine = {
      frames: new Map([["f1", { key: "f1", size: { width: 400, height: 300 }, tabs: [{ key: "t1", label: "Tab 1", content: { type: "html" } }], hidden: false, detached: false }]]),
      hideFrame: vi.fn(),
      showFrame: vi.fn(),
      setDetached: vi.fn(),
    } as unknown as FloatingFrameEngine;
    container = document.createElement("div");
    factory = vi.fn(() => ({ element: document.createElement("div") }));
    controller = new AbortController();
    vi.spyOn(window, "open").mockReturnValue(mockChildWindow as unknown as Window);
  });

  afterEach(() => { controller.abort(); vi.restoreAllMocks(); });

  it("hides frame and opens child window on detach", () => {
    const handler = createFrameDetachHandler(engine, container, factory, controller.signal);
    handler.detach("f1");
    expect(engine.hideFrame).toHaveBeenCalledWith("f1");
    expect(engine.setDetached).toHaveBeenCalledWith("f1", true);
    expect(window.open).toHaveBeenCalled();
  });

  it("shows frame and closes child window on reattach", () => {
    const handler = createFrameDetachHandler(engine, container, factory, controller.signal);
    handler.detach("f1");
    handler.reattach("f1");
    expect(engine.showFrame).toHaveBeenCalledWith("f1");
    expect(engine.setDetached).toHaveBeenCalledWith("f1", false);
    expect(mockChildWindow.close).toHaveBeenCalled();
  });

  it("dispatches pages-frame-detach event", () => {
    const handler = createFrameDetachHandler(engine, container, factory, controller.signal);
    const listener = vi.fn();
    container.addEventListener("pages-frame-detach", listener);
    handler.detach("f1");
    expect(listener).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify failure**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-detach-handler`
Expected: FAIL

- [ ] **Step 3: Implement `frame-detach-handler.ts`**

```typescript
import type { FloatingFrameEngine } from "./floating-frame-engine.js";
import type { ContentFactory } from "@casehubio/pages-component";
import { EventRelay } from "./detach/event-relay.js";
import { DetachRegistry } from "./detach/detach-registry.js";
import { copyStyles } from "./detach/copy-styles.js";

export interface FrameDetachHandler {
  detach(frameKey: string): void;
  reattach(frameKey: string): void;
  dispose(): void;
}

interface DetachedFrame {
  childWindow: Window;
  eventRelay: EventRelay;
  pollTimer: ReturnType<typeof setInterval>;
}

export function createFrameDetachHandler(
  engine: FloatingFrameEngine,
  container: HTMLElement,
  contentFactory: ContentFactory,
  signal: AbortSignal,
): FrameDetachHandler {
  const registry = new DetachRegistry();
  const detachedFrames = new Map<string, DetachedFrame>();

  function detach(frameKey: string): void {
    const frame = engine.frames.get(frameKey);
    if (!frame) return;

    engine.hideFrame(frameKey);
    engine.setDetached(frameKey, true);

    const win = window.open("", "_blank", `width=${frame.size.width},height=${frame.size.height}`);
    if (!win) {
      engine.showFrame(frameKey);
      engine.setDetached(frameKey, false);
      console.warn("Popup blocked — allow popups to detach frames.");
      return;
    }

    copyStyles(document, win.document);
    win.document.title = frame.tabs[0]?.label ?? "Frame";
    win.document.body.style.margin = "0";
    win.document.body.style.width = "100%";
    win.document.body.style.height = "100vh";
    win.document.body.style.overflow = "auto";

    for (const tab of frame.tabs) {
      const result = contentFactory(tab);
      win.document.body.appendChild(win.document.adoptNode(result.element));
    }

    const reattachBtn = win.document.createElement("button");
    reattachBtn.textContent = "⏎ Reattach";
    reattachBtn.style.cssText = "position:fixed;top:8px;right:8px;z-index:99999;cursor:pointer;padding:4px 12px;border:1px solid var(--pages-neutral-4, #ccc);border-radius:4px;background:var(--pages-neutral-2, #f5f5f5);font-size:12px;";
    reattachBtn.addEventListener("click", () => reattach(frameKey));
    win.document.body.appendChild(reattachBtn);

    const eventRelay = new EventRelay(win.document, container);
    eventRelay.start();

    win.addEventListener("beforeunload", () => reattach(frameKey));
    const pollTimer = setInterval(() => {
      if (win.closed) reattach(frameKey);
    }, 500);

    detachedFrames.set(frameKey, { childWindow: win, eventRelay, pollTimer });

    container.dispatchEvent(new CustomEvent("pages-frame-detach", {
      bubbles: true, composed: true, detail: { frameKey },
    }));

    win.focus();
  }

  function reattach(frameKey: string): void {
    const entry = detachedFrames.get(frameKey);
    if (!entry) return;
    detachedFrames.delete(frameKey);

    entry.eventRelay.stop();
    clearInterval(entry.pollTimer);

    if (!entry.childWindow.closed) entry.childWindow.close();

    engine.showFrame(frameKey);
    engine.setDetached(frameKey, false);

    container.dispatchEvent(new CustomEvent("pages-frame-reattach", {
      bubbles: true, composed: true, detail: { frameKey },
    }));
  }

  function dispose(): void {
    for (const [key] of detachedFrames) reattach(key);
  }

  signal.addEventListener("abort", dispose);

  return { detach, reattach, dispose };
}
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-detach-handler`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/frame-detach-handler.ts packages/pages-runtime/src/frame-detach-handler.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#304): frame detach handler — pop-out to child window and reattach

Uses hideFrame/showFrame lifecycle with EventRelay, DetachRegistry, copyStyles.

Refs casehubio/casehub-pages#304"
```

---

### Task 9: Snap preview module (#306)

**Files:**
- Create: `packages/pages-runtime/src/frame-snap-preview.ts`
- Create: `packages/pages-runtime/src/frame-snap-preview.test.ts`

**Interfaces:**
- Consumes: `FloatingFrameEngine` (snapFrame, unsnapFrame, recomputeSnappedFrames, frames), `FloatingFrameBackend` (onFrameDragMove, onTitlebarDoubleClick), `snapToZone`, `zoneToRect`
- Produces: `createSnapPreview(engine, backend, container, signal): void`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";
import { createSnapPreview } from "./frame-snap-preview.js";
import type { FloatingFrameEngine } from "./floating-frame-engine.js";
import type { FloatingFrameBackend } from "./floating-frame-backend.js";

describe("createSnapPreview", () => {
  let engine: FloatingFrameEngine;
  let backend: FloatingFrameBackend;
  let container: HTMLElement;
  let controller: AbortController;
  let dragMoveCb: (key: string, pos: { x: number; y: number }) => void;
  let dblClickCb: (key: string) => void;

  beforeEach(() => {
    engine = {
      frames: new Map([["f1", { key: "f1", snappedZone: undefined, hidden: false }]]),
      snapFrame: vi.fn(),
      unsnapFrame: vi.fn(),
      recomputeSnappedFrames: vi.fn(),
    } as unknown as FloatingFrameEngine;
    backend = {
      onFrameDragMove: vi.fn((cb) => { dragMoveCb = cb; }),
      onTitlebarDoubleClick: vi.fn((cb) => { dblClickCb = cb; }),
    } as unknown as FloatingFrameBackend;
    container = document.createElement("div");
    Object.defineProperty(container, "clientWidth", { value: 1000 });
    Object.defineProperty(container, "clientHeight", { value: 800 });
    controller = new AbortController();
    createSnapPreview(engine, backend, container, controller.signal);
  });

  it("shows preview overlay when dragging near edge", () => {
    dragMoveCb("f1", { x: 10, y: 400 });
    const overlay = container.querySelector("[data-snap-preview]");
    expect(overlay).not.toBeNull();
    expect((overlay as HTMLElement).style.display).not.toBe("none");
  });

  it("hides preview when not near edge", () => {
    dragMoveCb("f1", { x: 10, y: 400 });
    dragMoveCb("f1", { x: 500, y: 400 });
    const overlay = container.querySelector("[data-snap-preview]");
    expect((overlay as HTMLElement).style.display).toBe("none");
  });

  it("double-click snaps to full when not snapped", () => {
    dblClickCb("f1");
    expect(engine.snapFrame).toHaveBeenCalledWith("f1", "full", { width: 1000, height: 800 });
  });

  it("double-click unsnaps when snapped to full", () => {
    (engine.frames as Map<string, any>).set("f1", { key: "f1", snappedZone: "full", hidden: false });
    dblClickCb("f1");
    expect(engine.unsnapFrame).toHaveBeenCalledWith("f1");
  });
});
```

- [ ] **Step 2: Run tests to verify failure**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-snap-preview`
Expected: FAIL

- [ ] **Step 3: Implement `frame-snap-preview.ts`**

```typescript
import type { FloatingFrameEngine } from "./floating-frame-engine.js";
import type { FloatingFrameBackend } from "./floating-frame-backend.js";
import { snapToZone, zoneToRect } from "./frame-boundaries.js";

export function createSnapPreview(
  engine: FloatingFrameEngine,
  backend: FloatingFrameBackend,
  container: HTMLElement,
  signal: AbortSignal,
): void {
  const overlay = document.createElement("div");
  overlay.dataset.snapPreview = "";
  overlay.style.cssText = "position:absolute;background:var(--pages-accent-3, #3b82f6);opacity:0.2;border-radius:var(--pages-radius-sm, 4px);pointer-events:none;display:none;transition:all 100ms ease;";
  container.appendChild(overlay);

  let lastZone: string | null = null;

  backend.onFrameDragMove((key, pos) => {
    const canvasSize = { width: container.clientWidth, height: container.clientHeight };
    const zone = snapToZone(pos, canvasSize);
    if (zone) {
      const rect = zoneToRect(zone, canvasSize);
      overlay.style.left = `${rect.position.x}px`;
      overlay.style.top = `${rect.position.y}px`;
      overlay.style.width = `${rect.size.width}px`;
      overlay.style.height = `${rect.size.height}px`;
      overlay.style.display = "block";
      lastZone = zone;
    } else {
      overlay.style.display = "none";
      lastZone = null;
    }
  });

  backend.onFrameMove((key, pos) => {
    overlay.style.display = "none";
    const canvasSize = { width: container.clientWidth, height: container.clientHeight };
    const zone = snapToZone(pos, canvasSize);
    if (zone) {
      engine.snapFrame(key, zone, canvasSize);
      container.dispatchEvent(new CustomEvent("pages-frame-snap", {
        bubbles: true, composed: true, detail: { frameKey: key, zone },
      }));
    } else {
      const frame = engine.frames.get(key);
      if (frame?.snappedZone) {
        engine.unsnapFrame(key);
        container.dispatchEvent(new CustomEvent("pages-frame-unsnap", {
          bubbles: true, composed: true, detail: { frameKey: key },
        }));
      }
    }
    lastZone = null;
  });

  backend.onTitlebarDoubleClick((key) => {
    const frame = engine.frames.get(key);
    if (!frame) return;
    const canvasSize = { width: container.clientWidth, height: container.clientHeight };
    if (frame.snappedZone === "full") {
      engine.unsnapFrame(key);
      container.dispatchEvent(new CustomEvent("pages-frame-unsnap", {
        bubbles: true, composed: true, detail: { frameKey: key },
      }));
    } else {
      engine.snapFrame(key, "full", canvasSize);
      container.dispatchEvent(new CustomEvent("pages-frame-snap", {
        bubbles: true, composed: true, detail: { frameKey: key, zone: "full" },
      }));
    }
  });

  const resizeObserver = new ResizeObserver(() => {
    const canvasSize = { width: container.clientWidth, height: container.clientHeight };
    engine.recomputeSnappedFrames(canvasSize);
  });
  resizeObserver.observe(container);

  signal.addEventListener("abort", () => {
    resizeObserver.disconnect();
    overlay.remove();
  });
}
```

Note: `backend.onFrameMove` is the existing end-of-drag callback. The snap preview subscribes to it to commit the snap on drop. This requires the snap preview to be wired before the wire function's own `onFrameMove` subscription — or alternatively, the wire function passes the snap commit logic as a hook. During implementation, verify the subscription order works correctly; if not, the snap preview module can subscribe via the container's `pages-frame-move` event instead (which fires after the wire function's handler).

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-snap-preview`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/frame-snap-preview.ts packages/pages-runtime/src/frame-snap-preview.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#306): snap-to-edge preview overlay, double-click maximize, resize recompute

Zone preview during drag, commit snap on drop, double-click titlebar toggles
full/restore. ResizeObserver recomputes snapped frame geometry.

Refs casehubio/casehub-pages#306"
```

---

### Task 10: Integration — wire function, activation, site.ts, exports, event contract

**Files:**
- Modify: `packages/pages-runtime/src/wire-floating-workspace.ts`
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/site.ts`
- Modify: `packages/pages-runtime/src/index.ts`
- Modify: `docs/protocols/casehub/pages-event-contract.md`
- Test: `packages/pages-runtime/src/wire-floating-workspace.test.ts`

**Interfaces:**
- Consumes: all modules from Tasks 5-9
- Produces: updated `wireFloatingWorkspace` with `detachEnabled` option, updated activation callback with toolbar + keyboard wiring

- [ ] **Step 1: Update `wireFloatingWorkspace` to compose modules**

In `wire-floating-workspace.ts`:

Add import:
```typescript
import type { ContentFactory } from "@casehubio/pages-component";
import { createFrameDetachHandler } from "./frame-detach-handler.js";
import { createSnapPreview } from "./frame-snap-preview.js";
import { injectAnimationStyles, animateFrameExit, animateFrameMove } from "./frame-animations.js";
import type { FrameButtonConfig } from "./floating-frame-backend.js";
```

Extend `wireFloatingWorkspace` signature to accept options:
```typescript
export interface WireOptions {
  readonly detachEnabled?: boolean;
  readonly contentFactory?: ContentFactory;
  readonly signal?: AbortSignal;
}

export function wireFloatingWorkspace(
  backend: FloatingFrameBackend,
  container: HTMLElement,
  savedLayout?: readonly FrameLayout[],
  options?: WireOptions,
): WireHandle {
```

After engine creation, compose modules:
```typescript
  // Inject animation styles
  injectAnimationStyles();

  // Snap preview
  if (options?.signal) {
    createSnapPreview(engine, backend, container, options.signal);
  }

  // Detach handler
  let detachHandler: ReturnType<typeof createFrameDetachHandler> | undefined;
  if (options?.detachEnabled !== false && options?.contentFactory && options?.signal) {
    detachHandler = createFrameDetachHandler(engine, container, options.contentFactory, options.signal);
  }
```

Pass detach button as `extraButton` — but `backend.attach()` was already called before `wireFloatingWorkspace`. The detach button needs to be passed to `backend.attach()` in `activation.ts`. Instead, add the button config to the `WireHandle` return:

```typescript
  return {
    engine,
    detachHandler,
    dispose() { engine.dispose(); },
  };
```

- [ ] **Step 2: Update activation callback**

In `activation.ts`, after `backend.attach()` but before `wireFloatingWorkspace()`, prepare the detach button:

```typescript
const detachBtn: FrameButtonConfig = {
  icon: "\u{1F5D7}",
  title: "Pop out to new window",
  className: "frame-detach-btn",
  onClick: (frameKey) => { /* wired after wireFloatingWorkspace */ },
};

backend.attach(overlayContainer, defaultFactory, { extraButtons: [detachBtn] });
const handle = wireFloatingWorkspace(backend, overlayContainer, wsRef?.stash ?? undefined, {
  detachEnabled: true,
  contentFactory: defaultFactory,
  signal: abortController.signal,
});

// Wire detach button's onClick to the handler
if (handle.detachHandler) {
  detachBtn.onClick = (frameKey) => handle.detachHandler!.detach(frameKey);
}
```

After `wireFloatingWorkspace`, add toolbar and keyboard:
```typescript
import { createOrganiserToolbar } from "./organiser-toolbar.js";
import { createFrameKeyboardHandler } from "./frame-keyboard.js";

if (props.organisers !== false) {
  const toolbar = createOrganiserToolbar(handle.engine, overlayContainer, el, abortController.signal);
  el.insertBefore(toolbar, overlayContainer);
}

createFrameKeyboardHandler(handle.engine, overlayContainer, abortController.signal);
```

- [ ] **Step 3: Update site.ts — add layout save for new events**

In `site.ts`, in the event listeners section where `scheduleLayoutSave()` is called for frame events, add:
```typescript
for (const eventName of ["pages-frame-detach", "pages-frame-reattach", "pages-frame-snap", "pages-frame-unsnap"]) {
  target.addEventListener(eventName, () => scheduleLayoutSave(), { signal: abortController.signal });
}
```

- [ ] **Step 4: Update `index.ts` exports**

```typescript
export { createFrameKeyboardHandler } from "./frame-keyboard.js";
export { createOrganiserToolbar } from "./organiser-toolbar.js";
export { createFrameDetachHandler } from "./frame-detach-handler.js";
export type { FrameDetachHandler } from "./frame-detach-handler.js";
export { createSnapPreview } from "./frame-snap-preview.js";
export { injectAnimationStyles, animateFrameEnter, animateFrameExit, animateFrameMove } from "./frame-animations.js";
export { snapToZone, zoneToRect } from "./frame-boundaries.js";
export type { SnapZone } from "@casehubio/pages-component";
```

- [ ] **Step 5: Update event contract protocol**

In `docs/protocols/casehub/pages-event-contract.md`, add to the reserved names table:
```markdown
| `pages-frame-detach` | Frame popped out to child window | Wire function (detach handler) |
| `pages-frame-reattach` | Frame returned from child window | Wire function (reattach handler) |
| `pages-frame-snap` | Frame snapped to edge zone | Snap preview module |
| `pages-frame-unsnap` | Frame unsnapped from zone | Snap preview module |
```

- [ ] **Step 6: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run`
Expected: all pass

- [ ] **Step 7: Typecheck**

Run: `yarn typecheck`
Expected: no errors

- [ ] **Step 8: Build**

Run: `yarn build`
Expected: clean build

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/wire-floating-workspace.ts packages/pages-runtime/src/activation.ts packages/pages-runtime/src/site.ts packages/pages-runtime/src/index.ts docs/protocols/casehub/pages-event-contract.md
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#309): integrate floating workspace UX polish — wire, activation, events

Compose detach handler, snap preview, animations, keyboard shortcuts, and
organiser toolbar into the floating workspace activation pipeline.

Closes casehubio/casehub-pages#304, casehubio/casehub-pages#305,
casehubio/casehub-pages#306, casehubio/casehub-pages#307,
casehubio/casehub-pages#308
Refs casehubio/casehub-pages#309"
```

---

## Task Dependencies

```
Task 1 (spread refactor) ─── prerequisite for all ───┐
Task 2 (types + snap fns) ──────────────────────────┤
Task 3 (engine methods) ← depends on Task 2 ────────┤
Task 4 (backend additions) ─────────────────────────┤
Task 5 (keyboard) ── independent ───────────────────┤
Task 6 (toolbar) ── independent ────────────────────┤
Task 7 (animations) ← depends on Task 4 ───────────┤
Task 8 (detach) ← depends on Task 3 ───────────────┤
Task 9 (snap preview) ← depends on Tasks 2,3,4 ────┤
Task 10 (integration) ← depends on all above ──────┘
```

Tasks 5, 6, 7, 8, 9 are parallelizable after their dependencies are met. Tasks 5 and 6 are fully independent of each other and of Tasks 3-4 (they only need the existing engine interface).
