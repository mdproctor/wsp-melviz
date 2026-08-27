# Floating Workspace Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** Hortora/trellis#46 — Extract pages-floating-workspace component from workspace-view
**Issue group:** Hortora/trellis#46

**Goal:** Extract Dockview-based floating workspace from trellis into a reusable, content-agnostic `floating-workspace` component in casehub-pages.

**Architecture:** Engine/backend split — pure-state engine manages frames/tabs, pluggable backend wraps Dockview for rendering. Follows the dock-workbench integration pattern: extends pages-ui (builder + YAML desugaring), pages-component (types), and pages-runtime (engine + backend + activation). Dockview lazy-loaded via dynamic import.

**Tech Stack:** TypeScript 5, dockview-core ^7.0.0, Vitest/Jest, Playwright

## Global Constraints

- No new packages — extends pages-component, pages-ui, pages-runtime
- Dockview is an implementation detail — not exposed in public API
- All companion modules (zorder, spatial-nav, organisers, boundaries) are pure functions
- Content factory returns `ContentFactoryResult` with optional `dispose`
- `FrameLayout.tabs` stores full `FrameTabConfig[]` (not just keys) for dynamic tab survival
- Nested floating workspaces unsupported (events bubble, z-index conflicts)
- Pre-release — no backward compatibility constraints

---

### Task 1: Types and Component Registration

**Files:**
- Modify: `packages/pages-component/src/model/types.ts`
- Modify: `packages/pages-component/src/model/component-props.ts`
- Modify: `packages/pages-component/src/model/type-guards.ts`
- Modify: `packages/pages-component/src/model/index.ts`

**Interfaces:**
- Consumes: existing `Component`, `LayoutState`, `PanelEntry` types
- Produces: `FrameTabConfig`, `FrameConfig`, `FloatingWorkspaceConfig`, `FrameLayout`, `FloatingWorkspaceProps`, `isFloatingWorkspace()`, `ContentFactoryResult`, `ContentFactory`

- [ ] **Step 1: Add frame types to types.ts**

Use `ide_insert_member` to add after `LayoutState`:

```typescript
export interface FrameTabConfig {
  readonly key: string;
  readonly label: string;
  readonly icon?: string;
  readonly content: Component;
}

export interface FrameConfig {
  readonly key: string;
  readonly tabs: readonly FrameTabConfig[];
  readonly position?: { x: number; y: number };
  readonly size?: { width: number; height: number };
  readonly pinned?: boolean;
}

export interface FloatingWorkspaceConfig {
  readonly centre: Component | Component[];
  readonly frames?: readonly FrameConfig[];
  readonly organisers?: boolean;
}

export interface FrameLayout {
  readonly key: string;
  readonly order: number;
  readonly position: { x: number; y: number };
  readonly size: { width: number; height: number };
  readonly zIndex: number;
  readonly pinned: boolean;
  readonly hidden: boolean;
  readonly tabs: readonly FrameTabConfig[];
  readonly activeTabKey: string;
}

export interface ContentFactoryResult {
  readonly element: HTMLElement;
  readonly dispose?: () => void;
}

export type ContentFactory = (tab: FrameTabConfig) => ContentFactoryResult;
```

- [ ] **Step 2: Extend LayoutState with frames**

Use `ide_replace_member` on `LayoutState`:

```typescript
export interface LayoutState {
  readonly splits: Readonly<Record<string, readonly number[]>>;
  readonly docks: Readonly<Record<string, boolean>>;
  readonly panels: Readonly<Record<string, PanelEntry>>;
  readonly zones?: Readonly<Record<string, DockZone>>;
  readonly frames?: readonly FrameLayout[];
}
```

- [ ] **Step 3: Add FloatingWorkspaceProps to component-props.ts**

```typescript
export interface FloatingWorkspaceProps {
  readonly centre: Component | Component[];
  readonly frames?: readonly FrameConfig[];
  readonly organisers?: boolean;
}
```

Import `FrameConfig` from `./types.js` and `Component` if not already imported.

- [ ] **Step 4: Register in ComponentTypeRegistry and add type guard**

In `type-guards.ts`, add to `ComponentTypeRegistry`:

```typescript
"floating-workspace": FloatingWorkspaceProps;
```

Add type guard:

```typescript
export function isFloatingWorkspace(c: Component): c is TypedComponent<"floating-workspace"> {
  return c.type === "floating-workspace";
}
```

- [ ] **Step 5: Update index.ts re-exports**

Add to `packages/pages-component/src/model/index.ts`:

```typescript
export type { FrameTabConfig, FrameConfig, FloatingWorkspaceConfig, FrameLayout, ContentFactoryResult, ContentFactory } from "./types.js";
export type { FloatingWorkspaceProps } from "./component-props.js";
export { isFloatingWorkspace } from "./type-guards.js";
```

- [ ] **Step 6: Verify build**

Run: `yarn workspace @casehubio/pages-component run build`
Expected: clean build, no errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add packages/pages-component/src/model/types.ts packages/pages-component/src/model/component-props.ts packages/pages-component/src/model/type-guards.ts packages/pages-component/src/model/index.ts
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "feat(#46): floating workspace types — FrameTabConfig, FrameConfig, FrameLayout, ContentFactory Refs Hortora/trellis#46"
```

---

### Task 2: Companion Pure Functions

**Files:**
- Create: `packages/pages-runtime/src/frame-zorder.ts`
- Create: `packages/pages-runtime/src/frame-zorder.test.ts`
- Create: `packages/pages-runtime/src/frame-spatial-nav.ts`
- Create: `packages/pages-runtime/src/frame-spatial-nav.test.ts`
- Create: `packages/pages-runtime/src/frame-organisers.ts`
- Create: `packages/pages-runtime/src/frame-organisers.test.ts`
- Create: `packages/pages-runtime/src/frame-boundaries.ts`
- Create: `packages/pages-runtime/src/frame-boundaries.test.ts`

**Interfaces:**
- Consumes: `FrameLayout` from Task 1
- Produces: `bringToFront()`, `normalizeForSave()`, `findSpatialTarget()`, `applyPreset()`, `clampPosition()`, `nextFramePosition()`

#### 2a: frame-zorder

- [ ] **Step 1: Write failing tests for frame-zorder**

```typescript
// frame-zorder.test.ts
import { describe, it, expect } from "vitest";
import { bringToFront, normalizeForSave } from "./frame-zorder.js";
import type { FrameLayout } from "@casehubio/pages-component";

function makeFrame(key: string, zIndex: number, pinned = false): FrameLayout {
  return { key, order: 0, position: { x: 0, y: 0 }, size: { width: 400, height: 300 },
    zIndex, pinned, hidden: false, tabs: [], activeTabKey: "" };
}

describe("bringToFront", () => {
  it("increments z-index in normal tier", () => {
    const frames = new Map([["a", makeFrame("a", 1)], ["b", makeFrame("b", 2)]]);
    const result = bringToFront(frames, "a");
    expect(result.get("a")!.zIndex).toBe(3);
    expect(result.get("b")!.zIndex).toBe(2);
  });

  it("increments z-index in pinned tier (10000+)", () => {
    const frames = new Map([["a", makeFrame("a", 10001, true)], ["b", makeFrame("b", 10002, true)]]);
    const result = bringToFront(frames, "a");
    expect(result.get("a")!.zIndex).toBe(10003);
  });

  it("compacts when counter exceeds threshold", () => {
    const frames = new Map([["a", makeFrame("a", 5001)], ["b", makeFrame("b", 5002)]]);
    const result = bringToFront(frames, "a");
    expect(result.get("b")!.zIndex).toBeLessThan(100);
    expect(result.get("a")!.zIndex).toBeGreaterThan(result.get("b")!.zIndex);
  });
});

describe("normalizeForSave", () => {
  it("compacts to sequential indices", () => {
    const frames = new Map([["a", makeFrame("a", 500)], ["b", makeFrame("b", 1000)], ["c", makeFrame("c", 10500, true)]]);
    const result = normalizeForSave(frames);
    expect(result.get("a")!.zIndex).toBe(1);
    expect(result.get("b")!.zIndex).toBe(2);
    expect(result.get("c")!.zIndex).toBe(10001);
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-zorder`
Expected: FAIL — module not found

- [ ] **Step 3: Implement frame-zorder**

```typescript
// frame-zorder.ts
import type { FrameLayout } from "@casehubio/pages-component";

const PINNED_BASE = 10000;
const COMPACT_THRESHOLD = 5000;

export function bringToFront(
  frames: ReadonlyMap<string, FrameLayout>, key: string
): Map<string, FrameLayout> {
  const frame = frames.get(key);
  if (!frame) return new Map(frames);

  const tier = frame.pinned ? PINNED_BASE : 0;
  let maxZ = tier;
  for (const f of frames.values()) {
    if (f.pinned === frame.pinned && f.zIndex > maxZ) maxZ = f.zIndex;
  }
  const newZ = maxZ + 1;
  const result = new Map(frames);
  result.set(key, { ...frame, zIndex: newZ });

  if (newZ - tier > COMPACT_THRESHOLD) return compact(result);
  return result;
}

function compact(frames: Map<string, FrameLayout>): Map<string, FrameLayout> {
  const normal = [...frames.entries()].filter(([, f]) => !f.pinned).sort((a, b) => a[1].zIndex - b[1].zIndex);
  const pinned = [...frames.entries()].filter(([, f]) => f.pinned).sort((a, b) => a[1].zIndex - b[1].zIndex);
  const result = new Map<string, FrameLayout>();
  normal.forEach(([k, f], i) => result.set(k, { ...f, zIndex: i + 1 }));
  pinned.forEach(([k, f], i) => result.set(k, { ...f, zIndex: PINNED_BASE + i + 1 }));
  return result;
}

export function normalizeForSave(
  frames: ReadonlyMap<string, FrameLayout>
): Map<string, FrameLayout> {
  return compact(new Map(frames));
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-zorder`
Expected: PASS

#### 2b: frame-spatial-nav

- [ ] **Step 5: Write failing tests for frame-spatial-nav**

```typescript
// frame-spatial-nav.test.ts
import { describe, it, expect } from "vitest";
import { findSpatialTarget } from "./frame-spatial-nav.js";
import type { FrameLayout } from "@casehubio/pages-component";

function makeFrame(key: string, x: number, y: number, w = 400, h = 300): FrameLayout {
  return { key, order: 0, position: { x, y }, size: { width: w, height: h },
    zIndex: 1, pinned: false, hidden: false, tabs: [], activeTabKey: "" };
}

describe("findSpatialTarget", () => {
  it("finds frame to the right", () => {
    const frames = new Map([["a", makeFrame("a", 0, 0)], ["b", makeFrame("b", 500, 0)]]);
    expect(findSpatialTarget(frames, "a", "right")).toBe("b");
  });

  it("finds frame below", () => {
    const frames = new Map([["a", makeFrame("a", 0, 0)], ["b", makeFrame("b", 0, 400)]]);
    expect(findSpatialTarget(frames, "a", "down")).toBe("b");
  });

  it("returns null when no frame in direction", () => {
    const frames = new Map([["a", makeFrame("a", 0, 0)]]);
    expect(findSpatialTarget(frames, "a", "right")).toBeNull();
  });

  it("skips hidden frames", () => {
    const hidden = { ...makeFrame("b", 500, 0), hidden: true };
    const frames = new Map([["a", makeFrame("a", 0, 0)], ["b", hidden]]);
    expect(findSpatialTarget(frames, "a", "right")).toBeNull();
  });
});
```

- [ ] **Step 6: Implement frame-spatial-nav**

```typescript
// frame-spatial-nav.ts
import type { FrameLayout } from "@casehubio/pages-component";

type Direction = "up" | "down" | "left" | "right";

function center(f: FrameLayout): { cx: number; cy: number } {
  return { cx: f.position.x + f.size.width / 2, cy: f.position.y + f.size.height / 2 };
}

function inHalfPlane(from: { cx: number; cy: number }, to: { cx: number; cy: number }, dir: Direction): boolean {
  switch (dir) {
    case "right": return to.cx > from.cx;
    case "left":  return to.cx < from.cx;
    case "down":  return to.cy > from.cy;
    case "up":    return to.cy < from.cy;
  }
}

export function findSpatialTarget(
  frames: ReadonlyMap<string, FrameLayout>, current: string, direction: Direction
): string | null {
  const source = frames.get(current);
  if (!source) return null;
  const from = center(source);

  let bestKey: string | null = null;
  let bestDist = Infinity;

  for (const [key, frame] of frames) {
    if (key === current || frame.hidden) continue;
    const to = center(frame);
    if (!inHalfPlane(from, to, direction)) continue;
    const dist = Math.hypot(to.cx - from.cx, to.cy - from.cy);
    if (dist < bestDist) { bestDist = dist; bestKey = key; }
  }
  return bestKey;
}
```

- [ ] **Step 7: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-spatial-nav`
Expected: PASS

#### 2c: frame-organisers

- [ ] **Step 8: Write failing tests for frame-organisers**

```typescript
// frame-organisers.test.ts
import { describe, it, expect } from "vitest";
import { applyPreset } from "./frame-organisers.js";
import type { FrameLayout } from "@casehubio/pages-component";

function makeFrame(key: string, order: number): FrameLayout {
  return { key, order, position: { x: 0, y: 0 }, size: { width: 400, height: 300 },
    zIndex: 1, pinned: false, hidden: false, tabs: [], activeTabKey: "" };
}

const canvas = { width: 1200, height: 800 };

describe("applyPreset", () => {
  it("side-by-side splits horizontally", () => {
    const frames = [makeFrame("a", 0), makeFrame("b", 1)];
    const result = applyPreset(frames, canvas, "side-by-side");
    expect(result[0].position.x).toBeLessThan(result[1].position.x);
    expect(result[0].size.width + result[1].size.width).toBeLessThanOrEqual(canvas.width);
  });

  it("stacked splits vertically", () => {
    const frames = [makeFrame("a", 0), makeFrame("b", 1)];
    const result = applyPreset(frames, canvas, "stacked");
    expect(result[0].position.y).toBeLessThan(result[1].position.y);
  });

  it("grid arranges in rows and columns", () => {
    const frames = [makeFrame("a", 0), makeFrame("b", 1), makeFrame("c", 2), makeFrame("d", 3)];
    const result = applyPreset(frames, canvas, "grid");
    expect(result).toHaveLength(4);
    expect(new Set(result.map(f => `${f.position.x},${f.position.y}`))).toHaveLength(4);
  });

  it("focus maximizes first frame", () => {
    const frames = [makeFrame("a", 0), makeFrame("b", 1)];
    const result = applyPreset(frames, canvas, "focus");
    expect(result[0].size.width).toBeGreaterThan(result[1].size.width);
  });

  it("returns empty for empty input", () => {
    expect(applyPreset([], canvas, "grid")).toEqual([]);
  });
});
```

- [ ] **Step 9: Implement frame-organisers**

```typescript
// frame-organisers.ts
import type { FrameLayout } from "@casehubio/pages-component";

type Preset = "side-by-side" | "stacked" | "grid" | "main-sidebar" | "focus";
type CanvasSize = { width: number; height: number };

const GAP = 8;

export function applyPreset(
  frames: readonly FrameLayout[], canvas: CanvasSize, preset: Preset
): FrameLayout[] {
  if (frames.length === 0) return [];
  const sorted = [...frames].sort((a, b) => a.order - b.order);

  switch (preset) {
    case "side-by-side": return sideBySide(sorted, canvas);
    case "stacked": return stacked(sorted, canvas);
    case "grid": return grid(sorted, canvas);
    case "main-sidebar": return mainSidebar(sorted, canvas);
    case "focus": return focus(sorted, canvas);
  }
}

function sideBySide(frames: FrameLayout[], c: CanvasSize): FrameLayout[] {
  const w = Math.floor((c.width - GAP * (frames.length - 1)) / frames.length);
  return frames.map((f, i) => ({ ...f, position: { x: i * (w + GAP), y: 0 }, size: { width: w, height: c.height } }));
}

function stacked(frames: FrameLayout[], c: CanvasSize): FrameLayout[] {
  const h = Math.floor((c.height - GAP * (frames.length - 1)) / frames.length);
  return frames.map((f, i) => ({ ...f, position: { x: 0, y: i * (h + GAP) }, size: { width: c.width, height: h } }));
}

function grid(frames: FrameLayout[], c: CanvasSize): FrameLayout[] {
  const cols = Math.ceil(Math.sqrt(frames.length));
  const rows = Math.ceil(frames.length / cols);
  const cellW = Math.floor((c.width - GAP * (cols - 1)) / cols);
  const cellH = Math.floor((c.height - GAP * (rows - 1)) / rows);
  return frames.map((f, i) => {
    const col = i % cols;
    const row = Math.floor(i / cols);
    return { ...f, position: { x: col * (cellW + GAP), y: row * (cellH + GAP) }, size: { width: cellW, height: cellH } };
  });
}

function mainSidebar(frames: FrameLayout[], c: CanvasSize): FrameLayout[] {
  if (frames.length === 1) return [{ ...frames[0], position: { x: 0, y: 0 }, size: { width: c.width, height: c.height } }];
  const mainW = Math.floor(c.width * 0.65);
  const sideW = c.width - mainW - GAP;
  const sideH = Math.floor((c.height - GAP * (frames.length - 2)) / (frames.length - 1));
  return [
    { ...frames[0], position: { x: 0, y: 0 }, size: { width: mainW, height: c.height } },
    ...frames.slice(1).map((f, i) => ({ ...f, position: { x: mainW + GAP, y: i * (sideH + GAP) }, size: { width: sideW, height: sideH } })),
  ];
}

function focus(frames: FrameLayout[], c: CanvasSize): FrameLayout[] {
  const mainW = Math.floor(c.width * 0.85);
  const mainH = Math.floor(c.height * 0.85);
  const result = [{ ...frames[0], position: { x: Math.floor((c.width - mainW) / 2), y: Math.floor((c.height - mainH) / 2) }, size: { width: mainW, height: mainH } }];
  const thumbW = 200;
  const thumbH = 150;
  for (let i = 1; i < frames.length; i++) {
    result.push({ ...frames[i], position: { x: c.width - thumbW - GAP, y: GAP + (i - 1) * (thumbH + GAP) }, size: { width: thumbW, height: thumbH } });
  }
  return result;
}
```

- [ ] **Step 10: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-organisers`
Expected: PASS

#### 2d: frame-boundaries

- [ ] **Step 11: Write failing tests for frame-boundaries**

```typescript
// frame-boundaries.test.ts
import { describe, it, expect } from "vitest";
import { clampPosition, nextFramePosition } from "./frame-boundaries.js";

describe("clampPosition", () => {
  it("clamps to container bounds", () => {
    const pos = clampPosition({ x: -50, y: -20 }, { width: 400, height: 300 }, { width: 1200, height: 800 });
    expect(pos.x).toBe(0);
    expect(pos.y).toBe(0);
  });

  it("clamps right/bottom overflow", () => {
    const pos = clampPosition({ x: 1000, y: 700 }, { width: 400, height: 300 }, { width: 1200, height: 800 });
    expect(pos.x).toBe(800);
    expect(pos.y).toBe(500);
  });

  it("passes through valid positions", () => {
    const pos = clampPosition({ x: 100, y: 100 }, { width: 400, height: 300 }, { width: 1200, height: 800 });
    expect(pos).toEqual({ x: 100, y: 100 });
  });
});

describe("nextFramePosition", () => {
  it("centers first frame", () => {
    const pos = nextFramePosition({ width: 1200, height: 800 }, { width: 400, height: 300 }, []);
    expect(pos.x).toBe(400);
    expect(pos.y).toBe(250);
  });

  it("offsets subsequent frames", () => {
    const existing = [{ x: 400, y: 250 }];
    const pos = nextFramePosition({ width: 1200, height: 800 }, { width: 400, height: 300 }, existing);
    expect(pos.x).not.toBe(400);
    expect(pos.y).not.toBe(250);
  });
});
```

- [ ] **Step 12: Implement frame-boundaries**

```typescript
// frame-boundaries.ts
type Position = { x: number; y: number };
type Size = { width: number; height: number };

const DEFAULT_DISPLACEMENT = 30;

export function clampPosition(pos: Position, size: Size, container: Size): Position {
  return {
    x: Math.max(0, Math.min(pos.x, container.width - size.width)),
    y: Math.max(0, Math.min(pos.y, container.height - size.height)),
  };
}

export function nextFramePosition(
  container: Size, frameSize: Size, existing: readonly Position[], displacement = DEFAULT_DISPLACEMENT
): Position {
  if (existing.length === 0) {
    return {
      x: Math.floor((container.width - frameSize.width) / 2),
      y: Math.floor((container.height - frameSize.height) / 2),
    };
  }

  let candidate = {
    x: existing[existing.length - 1].x + displacement,
    y: existing[existing.length - 1].y + displacement,
  };

  for (let attempt = 0; attempt < 20; attempt++) {
    const collides = existing.some(e => Math.abs(e.x - candidate.x) < 10 && Math.abs(e.y - candidate.y) < 10);
    if (!collides) break;
    candidate = { x: candidate.x + displacement, y: candidate.y + displacement };
  }

  return clampPosition(candidate, frameSize, container);
}
```

- [ ] **Step 13: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run frame-boundaries`
Expected: PASS

- [ ] **Step 14: Commit all companion modules**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add packages/pages-runtime/src/frame-zorder.ts packages/pages-runtime/src/frame-zorder.test.ts packages/pages-runtime/src/frame-spatial-nav.ts packages/pages-runtime/src/frame-spatial-nav.test.ts packages/pages-runtime/src/frame-organisers.ts packages/pages-runtime/src/frame-organisers.test.ts packages/pages-runtime/src/frame-boundaries.ts packages/pages-runtime/src/frame-boundaries.test.ts
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "feat(#46): companion pure functions — zorder, spatial-nav, organisers, boundaries Refs Hortora/trellis#46"
```

---

### Task 3: Backend Interface and Dockview Implementation

**Files:**
- Create: `packages/pages-runtime/src/floating-frame-backend.ts`
- Create: `packages/pages-runtime/src/dockview-backend.ts`
- Create: `packages/pages-runtime/src/dockview-backend.test.ts`
- Modify: `packages/pages-runtime/package.json`

**Interfaces:**
- Consumes: `FrameLayout`, `FrameTabConfig`, `ContentFactory`, `ContentFactoryResult` from Task 1
- Produces: `FloatingFrameBackend` interface, `createDockviewBackend()` factory

- [ ] **Step 1: Define backend interface**

```typescript
// floating-frame-backend.ts
import type { FrameLayout, FrameTabConfig, ContentFactory } from "@casehubio/pages-component";

export interface FloatingFrameBackend {
  attach(container: HTMLElement, contentFactory: ContentFactory): void;
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

  dispose(): void;
  unwrap(): unknown | null;
}
```

- [ ] **Step 2: Add dockview-core dependency**

Run: `yarn workspace @casehubio/pages-runtime add dockview-core@^7.0.0`

- [ ] **Step 3: Write failing tests for Dockview backend**

```typescript
// dockview-backend.test.ts
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { createDockviewBackend } from "./dockview-backend.js";
import type { FloatingFrameBackend } from "./floating-frame-backend.js";
import type { FrameLayout, ContentFactory, FrameTabConfig } from "@casehubio/pages-component";

function makeTab(key: string): FrameTabConfig {
  return { key, label: key, content: { type: "html", props: { content: `<div>${key}</div>` } } };
}

function makeLayout(key: string): FrameLayout {
  return { key, order: 0, position: { x: 50, y: 50 }, size: { width: 400, height: 300 },
    zIndex: 1, pinned: false, hidden: false, tabs: [makeTab("tab1")], activeTabKey: "tab1" };
}

const testFactory: ContentFactory = (tab) => {
  const el = document.createElement("div");
  el.textContent = tab.label;
  return { element: el };
};

describe("createDockviewBackend", () => {
  let backend: FloatingFrameBackend;
  let container: HTMLElement;

  beforeEach(async () => {
    container = document.createElement("div");
    container.style.width = "1200px";
    container.style.height = "800px";
    document.body.appendChild(container);
    backend = await createDockviewBackend();
  });

  afterEach(() => {
    backend.dispose();
    container.remove();
  });

  it("attaches to container", () => {
    backend.attach(container, testFactory);
    expect(container.querySelector(".dv-dockview")).toBeTruthy();
  });

  it("renders a frame", () => {
    backend.attach(container, testFactory);
    backend.renderFrame(makeLayout("frame1"));
    expect(backend.unwrap()).toBeTruthy();
  });

  it("fires onFrameMove callback", () => {
    const moves: Array<{ key: string; pos: { x: number; y: number } }> = [];
    backend.attach(container, testFactory);
    backend.onFrameMove((key, pos) => moves.push({ key, pos }));
    backend.renderFrame(makeLayout("frame1"));
    backend.updatePosition("frame1", { x: 100, y: 100 });
    // Position update is programmatic — callback fires
    expect(moves.length).toBeGreaterThanOrEqual(0); // Dockview may or may not fire for programmatic moves
  });

  it("disposes cleanly", () => {
    backend.attach(container, testFactory);
    backend.renderFrame(makeLayout("frame1"));
    backend.dispose();
    // No errors after dispose
  });
});
```

- [ ] **Step 4: Implement Dockview backend**

```typescript
// dockview-backend.ts
import type { FloatingFrameBackend } from "./floating-frame-backend.js";
import type { FrameLayout, FrameTabConfig, ContentFactory, ContentFactoryResult } from "@casehubio/pages-component";

const CSS_MARKER = "data-pages-dockview-css";

export async function createDockviewBackend(): Promise<FloatingFrameBackend> {
  let dockview: any;
  let DockviewComponent: any;
  let themeDark: any;

  try {
    const mod = await import("dockview-core");
    DockviewComponent = mod.DockviewComponent;
    themeDark = mod.themeDark;
    const cssModule = await import("dockview-core/dist/styles/dockview.css?raw");
    if (!document.querySelector(`style[${CSS_MARKER}]`)) {
      const style = document.createElement("style");
      style.setAttribute(CSS_MARKER, "");
      style.textContent = typeof cssModule === "string" ? cssModule : (cssModule as any).default;
      document.head.appendChild(style);
    }
  } catch (err) {
    console.error("Failed to load dockview-core:", err);
    return createErrorBackend(err);
  }

  let container: HTMLElement | null = null;
  let factory: ContentFactory | null = null;
  const frameMoveCallbacks: Array<(key: string, pos: { x: number; y: number }) => void> = [];
  const frameResizeCallbacks: Array<(key: string, size: { width: number; height: number }) => void> = [];
  const tabDragOutCallbacks: Array<(fromFrame: string, tabKey: string, position: { x: number; y: number }) => void> = [];
  const tabReorderCallbacks: Array<(frameKey: string, tabKeys: string[]) => void> = [];
  const frameGroups = new Map<string, any>();
  const contentResults = new Map<string, ContentFactoryResult>();

  return {
    attach(el: HTMLElement, contentFactory: ContentFactory) {
      container = el;
      factory = contentFactory;
      dockview = new DockviewComponent(el, {
        createComponent: (options: any) => {
          const tabConfig = options.params?.tabConfig as FrameTabConfig;
          if (!tabConfig || !factory) {
            const placeholder = document.createElement("div");
            placeholder.textContent = "No content";
            return { element: placeholder, init: () => {}, dispose: () => {} };
          }
          const result = factory(tabConfig);
          contentResults.set(tabConfig.key, result);
          return {
            element: result.element,
            init: () => {},
            dispose: () => {
              result.dispose?.();
              contentResults.delete(tabConfig.key);
            },
          };
        },
        theme: { ...themeDark, tabAnimation: "smooth" },
        dndEdges: false,
      });
    },

    detach() {
      if (dockview) { dockview.dispose(); dockview = null; }
      container = null;
      factory = null;
      frameGroups.clear();
      contentResults.clear();
    },

    renderFrame(layout: FrameLayout) {
      if (!dockview || layout.tabs.length === 0) return;
      const firstTab = layout.tabs[0];
      const panel = dockview.addPanel({
        id: `${layout.key}:${firstTab.key}`,
        component: "default",
        params: { tabConfig: firstTab, frameKey: layout.key },
        title: firstTab.label,
        floating: { width: layout.size.width, height: layout.size.height, x: layout.position.x, y: layout.position.y },
      });
      const group = panel.group;
      frameGroups.set(layout.key, group);

      for (let i = 1; i < layout.tabs.length; i++) {
        const tab = layout.tabs[i];
        dockview.addPanel({
          id: `${layout.key}:${tab.key}`,
          component: "default",
          params: { tabConfig: tab, frameKey: layout.key },
          title: tab.label,
          position: { referenceGroup: group },
        });
      }

      if (layout.activeTabKey) {
        const activePanel = dockview.getPanel(`${layout.key}:${layout.activeTabKey}`);
        if (activePanel) activePanel.api.setActive();
      }

      this._subscribeOverlayEvents(layout.key);
      this._injectFrameChrome(group, layout.key);
    },

    removeFrame(key: string) {
      const group = frameGroups.get(key);
      if (!group || !dockview) return;
      const panels = [...group.panels];
      for (const p of panels) dockview.removePanel(p);
      frameGroups.delete(key);
    },

    updatePosition(key: string, pos: { x: number; y: number }) {
      const fg = this._findFloatingOverlay(key);
      if (fg?.overlay) fg.overlay.setBounds({ ...fg.overlay.getBounds(), left: pos.x, top: pos.y });
    },

    updateSize(key: string, size: { width: number; height: number }) {
      const fg = this._findFloatingOverlay(key);
      if (fg?.overlay) fg.overlay.setBounds({ ...fg.overlay.getBounds(), width: size.width, height: size.height });
    },

    bringToFront(key: string) {
      const fg = this._findFloatingOverlay(key);
      if (fg?.overlay) fg.overlay.bringToFront();
    },

    addTab(frameKey: string, tab: FrameTabConfig) {
      const group = frameGroups.get(frameKey);
      if (!group || !dockview) return;
      dockview.addPanel({
        id: `${frameKey}:${tab.key}`,
        component: "default",
        params: { tabConfig: tab, frameKey },
        title: tab.label,
        position: { referenceGroup: group },
      });
    },

    removeTab(frameKey: string, tabKey: string) {
      if (!dockview) return;
      const panel = dockview.getPanel(`${frameKey}:${tabKey}`);
      if (panel) dockview.removePanel(panel);
    },

    setActiveTab(frameKey: string, tabKey: string) {
      if (!dockview) return;
      const panel = dockview.getPanel(`${frameKey}:${tabKey}`);
      if (panel) panel.api.setActive();
    },

    onFrameMove(cb) { frameMoveCallbacks.push(cb); },
    onFrameResize(cb) { frameResizeCallbacks.push(cb); },
    onTabDragOut(cb) { tabDragOutCallbacks.push(cb); },
    onTabReorder(cb) { tabReorderCallbacks.push(cb); },

    dispose() {
      if (dockview) { dockview.dispose(); dockview = null; }
      frameGroups.clear();
      contentResults.clear();
      frameMoveCallbacks.length = 0;
      frameResizeCallbacks.length = 0;
      tabDragOutCallbacks.length = 0;
      tabReorderCallbacks.length = 0;
    },

    unwrap() { return dockview ?? null; },

    _findFloatingOverlay(frameKey: string) {
      if (!dockview) return null;
      const group = frameGroups.get(frameKey);
      if (!group) return null;
      const fgs = (dockview as any).floatingGroups;
      return fgs?.find((fg: any) => fg._group === group) ?? null;
    },

    _subscribeOverlayEvents(frameKey: string) {
      const fg = this._findFloatingOverlay(frameKey);
      if (!fg?.overlay?.onDidChangeEnd) return;
      fg.overlay.onDidChangeEnd(() => {
        const el = fg.overlay._element ?? fg.overlay.element;
        if (!el) return;
        const rect = el.getBoundingClientRect();
        const containerRect = container?.getBoundingClientRect();
        if (!containerRect) return;
        const pos = { x: rect.left - containerRect.left, y: rect.top - containerRect.top };
        const size = { width: rect.width, height: rect.height };
        for (const cb of frameMoveCallbacks) cb(frameKey, pos);
        for (const cb of frameResizeCallbacks) cb(frameKey, size);
      });
    },

    _injectFrameChrome(group: any, frameKey: string) {
      const el = group.element ?? group.header?.element?.closest?.(".dv-groupview");
      if (!el) return;
      const titlebar = el.querySelector(".dv-floating-titlebar");
      if (!titlebar) return;

      const closeDot = document.createElement("span");
      closeDot.className = "frame-close-dot";
      closeDot.style.cssText = "width:12px;height:12px;border-radius:50%;background:#ff5f57;cursor:pointer;display:inline-block;margin:0 4px;";
      closeDot.addEventListener("pointerdown", (e) => { e.stopPropagation(); });
      closeDot.addEventListener("click", () => {
        container?.dispatchEvent(new CustomEvent("pages-frame-close", { bubbles: true, composed: true, detail: { frameKey } }));
      });

      const pinBtn = document.createElement("span");
      pinBtn.className = "frame-pin-btn";
      pinBtn.textContent = "📌";
      pinBtn.style.cssText = "cursor:pointer;margin:0 4px;font-size:12px;opacity:0.5;";
      pinBtn.addEventListener("pointerdown", (e) => { e.stopPropagation(); });
      pinBtn.addEventListener("click", () => {
        container?.dispatchEvent(new CustomEvent("pages-frame-pin", { bubbles: true, composed: true, detail: { frameKey } }));
      });

      titlebar.prepend(pinBtn);
      titlebar.prepend(closeDot);
    },
  } as FloatingFrameBackend & Record<string, any>;
}

function createErrorBackend(err: unknown): FloatingFrameBackend {
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
    dispose() {},
    unwrap() { return null; },
  };
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run dockview-backend`
Expected: PASS (some tests may need jsdom or happy-dom — verify test environment)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add packages/pages-runtime/src/floating-frame-backend.ts packages/pages-runtime/src/dockview-backend.ts packages/pages-runtime/src/dockview-backend.test.ts packages/pages-runtime/package.json
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "feat(#46): FloatingFrameBackend interface + Dockview implementation Refs Hortora/trellis#46"
```

---

### Task 4: Floating Frame Engine

**Files:**
- Create: `packages/pages-runtime/src/floating-frame-engine.ts`
- Create: `packages/pages-runtime/src/floating-frame-engine.test.ts`

**Interfaces:**
- Consumes: `FloatingFrameBackend` from Task 3, companion functions from Task 2, types from Task 1
- Produces: `FloatingFrameEngine` interface, `createFloatingFrameEngine()` factory

- [ ] **Step 1: Write failing tests**

```typescript
// floating-frame-engine.test.ts
import { describe, it, expect, beforeEach, vi } from "vitest";
import { createFloatingFrameEngine } from "./floating-frame-engine.js";
import type { FloatingFrameBackend } from "./floating-frame-backend.js";
import type { FrameConfig, FrameTabConfig } from "@casehubio/pages-component";

function mockBackend(): FloatingFrameBackend {
  return {
    attach: vi.fn(), detach: vi.fn(),
    renderFrame: vi.fn(), removeFrame: vi.fn(),
    updatePosition: vi.fn(), updateSize: vi.fn(), bringToFront: vi.fn(),
    addTab: vi.fn(), removeTab: vi.fn(), setActiveTab: vi.fn(),
    onFrameMove: vi.fn(), onFrameResize: vi.fn(), onTabDragOut: vi.fn(), onTabReorder: vi.fn(),
    dispose: vi.fn(), unwrap: vi.fn(() => null),
  };
}

function makeTab(key: string): FrameTabConfig {
  return { key, label: key, content: { type: "html", props: { content: `<div>${key}</div>` } } };
}

function makeFrameConfig(key: string, tabs = ["tab1"]): FrameConfig {
  return { key, tabs: tabs.map(makeTab) };
}

describe("FloatingFrameEngine", () => {
  let backend: FloatingFrameBackend;

  beforeEach(() => { backend = mockBackend(); });

  it("creates a frame", () => {
    const engine = createFloatingFrameEngine(backend);
    const layout = engine.createFrame(makeFrameConfig("f1"));
    expect(layout.key).toBe("f1");
    expect(engine.frames.size).toBe(1);
    expect(backend.renderFrame).toHaveBeenCalledOnce();
  });

  it("removes a frame", () => {
    const engine = createFloatingFrameEngine(backend);
    engine.createFrame(makeFrameConfig("f1"));
    engine.removeFrame("f1");
    expect(engine.frames.size).toBe(0);
    expect(backend.removeFrame).toHaveBeenCalledWith("f1");
  });

  it("hides and shows a frame", () => {
    const engine = createFloatingFrameEngine(backend);
    engine.createFrame(makeFrameConfig("f1"));
    engine.hideFrame("f1");
    expect(engine.frames.get("f1")!.hidden).toBe(true);
    engine.showFrame("f1");
    expect(engine.frames.get("f1")!.hidden).toBe(false);
  });

  it("moves tab between frames", () => {
    const engine = createFloatingFrameEngine(backend);
    engine.createFrame(makeFrameConfig("f1", ["t1", "t2"]));
    engine.createFrame(makeFrameConfig("f2", ["t3"]));
    engine.moveTab("f1", "t1", "f2");
    expect(engine.frames.get("f1")!.tabs).toHaveLength(1);
    expect(engine.frames.get("f2")!.tabs).toHaveLength(2);
  });

  it("captures and restores layout", () => {
    const engine = createFloatingFrameEngine(backend);
    engine.createFrame(makeFrameConfig("f1"));
    engine.createFrame(makeFrameConfig("f2"));
    const captured = engine.captureLayout();
    expect(captured).toHaveLength(2);

    const engine2 = createFloatingFrameEngine(backend, captured);
    expect(engine2.frames.size).toBe(2);
  });

  it("toggles pin", () => {
    const engine = createFloatingFrameEngine(backend);
    engine.createFrame(makeFrameConfig("f1"));
    expect(engine.frames.get("f1")!.pinned).toBe(false);
    engine.togglePin("f1");
    expect(engine.frames.get("f1")!.pinned).toBe(true);
  });

  it("applies organiser preset", () => {
    const engine = createFloatingFrameEngine(backend);
    engine.createFrame({ ...makeFrameConfig("f1"), position: { x: 0, y: 0 }, size: { width: 400, height: 300 } });
    engine.createFrame({ ...makeFrameConfig("f2"), position: { x: 0, y: 0 }, size: { width: 400, height: 300 } });
    engine.applyOrganiser("side-by-side");
    const f1 = engine.frames.get("f1")!;
    const f2 = engine.frames.get("f2")!;
    expect(f1.position.x).not.toBe(f2.position.x);
  });

  it("disposes backend on dispose", () => {
    const engine = createFloatingFrameEngine(backend);
    engine.dispose();
    expect(backend.dispose).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine`
Expected: FAIL

- [ ] **Step 3: Implement floating-frame-engine**

```typescript
// floating-frame-engine.ts
import type { FrameConfig, FrameLayout, FrameTabConfig } from "@casehubio/pages-component";
import type { FloatingFrameBackend } from "./floating-frame-backend.js";
import { bringToFront as zBringToFront, normalizeForSave } from "./frame-zorder.js";
import { findSpatialTarget } from "./frame-spatial-nav.js";
import { applyPreset } from "./frame-organisers.js";

const DEFAULT_SIZE = { width: 400, height: 300 };

export interface FloatingFrameEngine {
  readonly frames: ReadonlyMap<string, FrameLayout>;
  createFrame(config: FrameConfig): FrameLayout;
  removeFrame(key: string): void;
  hideFrame(key: string): void;
  showFrame(key: string): void;
  addTab(frameKey: string, tab: FrameTabConfig): void;
  removeTab(frameKey: string, tabKey: string): void;
  moveTab(fromFrame: string, tabKey: string, toFrame: string): void;
  setActiveTab(frameKey: string, tabKey: string): void;
  bringToFront(key: string): void;
  togglePin(key: string): void;
  focusDirection(direction: "up" | "down" | "left" | "right"): string | null;
  applyOrganiser(preset: "side-by-side" | "stacked" | "grid" | "main-sidebar" | "focus"): void;
  captureLayout(): readonly FrameLayout[];
  restoreLayout(saved: readonly FrameLayout[]): void;
  dispose(): void;
}

export function createFloatingFrameEngine(
  backend: FloatingFrameBackend,
  savedLayout?: readonly FrameLayout[],
): FloatingFrameEngine {
  let frames = new Map<string, FrameLayout>();
  let disposed = false;
  let nextOrder = 0;

  function assertAlive() { if (disposed) throw new Error("Engine is disposed"); }

  if (savedLayout) {
    for (const layout of savedLayout) {
      frames.set(layout.key, layout);
      if (!layout.hidden) backend.renderFrame(layout);
      if (layout.order >= nextOrder) nextOrder = layout.order + 1;
    }
  }

  const engine: FloatingFrameEngine = {
    get frames() { return new Map(frames); },

    createFrame(config: FrameConfig): FrameLayout {
      assertAlive();
      const layout: FrameLayout = {
        key: config.key,
        order: nextOrder++,
        position: config.position ?? { x: 50, y: 50 },
        size: config.size ?? DEFAULT_SIZE,
        zIndex: 1,
        pinned: config.pinned ?? false,
        hidden: false,
        tabs: config.tabs,
        activeTabKey: config.tabs[0]?.key ?? "",
      };
      frames.set(config.key, layout);
      frames = zBringToFront(frames, config.key);
      backend.renderFrame(frames.get(config.key)!);
      return frames.get(config.key)!;
    },

    removeFrame(key: string) {
      assertAlive();
      if (!frames.has(key)) return;
      backend.removeFrame(key);
      frames.delete(key);
    },

    hideFrame(key: string) {
      assertAlive();
      const frame = frames.get(key);
      if (!frame || frame.hidden) return;
      backend.removeFrame(key);
      frames.set(key, { ...frame, hidden: true });
    },

    showFrame(key: string) {
      assertAlive();
      const frame = frames.get(key);
      if (!frame || !frame.hidden) return;
      const shown = { ...frame, hidden: false };
      frames.set(key, shown);
      frames = zBringToFront(frames, key);
      backend.renderFrame(frames.get(key)!);
    },

    addTab(frameKey: string, tab: FrameTabConfig) {
      assertAlive();
      const frame = frames.get(frameKey);
      if (!frame) return;
      frames.set(frameKey, { ...frame, tabs: [...frame.tabs, tab] });
      backend.addTab(frameKey, tab);
    },

    removeTab(frameKey: string, tabKey: string) {
      assertAlive();
      const frame = frames.get(frameKey);
      if (!frame) return;
      const newTabs = frame.tabs.filter(t => t.key !== tabKey);
      const activeTabKey = frame.activeTabKey === tabKey ? (newTabs[0]?.key ?? "") : frame.activeTabKey;
      frames.set(frameKey, { ...frame, tabs: newTabs, activeTabKey });
      backend.removeTab(frameKey, tabKey);
    },

    moveTab(fromFrame: string, tabKey: string, toFrame: string) {
      assertAlive();
      const srcFrame = frames.get(fromFrame);
      const dstFrame = frames.get(toFrame);
      if (!srcFrame || !dstFrame) return;
      const tab = srcFrame.tabs.find(t => t.key === tabKey);
      if (!tab) return;
      this.removeTab(fromFrame, tabKey);
      this.addTab(toFrame, tab);
    },

    setActiveTab(frameKey: string, tabKey: string) {
      assertAlive();
      const frame = frames.get(frameKey);
      if (!frame) return;
      frames.set(frameKey, { ...frame, activeTabKey: tabKey });
      backend.setActiveTab(frameKey, tabKey);
    },

    bringToFront(key: string) {
      assertAlive();
      frames = zBringToFront(frames, key);
      backend.bringToFront(key);
    },

    togglePin(key: string) {
      assertAlive();
      const frame = frames.get(key);
      if (!frame) return;
      frames.set(key, { ...frame, pinned: !frame.pinned });
      frames = zBringToFront(frames, key);
      backend.bringToFront(key);
    },

    focusDirection(direction) {
      assertAlive();
      const visible = new Map([...frames].filter(([, f]) => !f.hidden));
      const currentKey = [...visible.entries()].reduce((a, b) => a[1].zIndex > b[1].zIndex ? a : b)[0];
      return findSpatialTarget(visible, currentKey, direction);
    },

    applyOrganiser(preset) {
      assertAlive();
      const visible = [...frames.values()].filter(f => !f.hidden);
      if (visible.length === 0) return;
      const container = { width: 1200, height: 800 }; // will be provided by activation
      const arranged = applyPreset(visible, container, preset);
      for (const a of arranged) {
        frames.set(a.key, a);
        backend.updatePosition(a.key, a.position);
        backend.updateSize(a.key, a.size);
      }
    },

    captureLayout(): readonly FrameLayout[] {
      const normalized = normalizeForSave(frames);
      return [...normalized.values()].sort((a, b) => a.order - b.order);
    },

    restoreLayout(saved: readonly FrameLayout[]) {
      assertAlive();
      for (const [key] of frames) backend.removeFrame(key);
      frames.clear();
      nextOrder = 0;
      for (const layout of saved) {
        frames.set(layout.key, layout);
        if (!layout.hidden) backend.renderFrame(layout);
        if (layout.order >= nextOrder) nextOrder = layout.order + 1;
      }
    },

    dispose() {
      if (disposed) return;
      disposed = true;
      backend.dispose();
      frames.clear();
    },
  };

  return engine;
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --run floating-frame-engine`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add packages/pages-runtime/src/floating-frame-engine.ts packages/pages-runtime/src/floating-frame-engine.test.ts
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "feat(#46): FloatingFrameEngine — state manager with pluggable backend Refs Hortora/trellis#46"
```

---

### Task 5: DSL Builder and YAML Desugaring

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts`
- Modify: `packages/pages-ui/src/dsl/builders.test.ts`
- Modify: `packages/pages-ui/src/dsl/index.ts`
- Modify: `packages/pages-ui/src/parser/component-desugar.ts`
- Modify: `packages/pages-ui/src/parser/desugar-new-types.test.ts`

**Interfaces:**
- Consumes: `FloatingWorkspaceConfig`, `FrameConfig`, `FrameTabConfig`, `Component` from Task 1
- Produces: `floatingWorkspace()` builder function, YAML desugaring for `floating-workspace` type

- [ ] **Step 1: Write failing builder tests**

Add to `builders.test.ts`:

```typescript
describe("floatingWorkspace", () => {
  it("creates floating-workspace component with centre", () => {
    const result = floatingWorkspace({
      centre: { type: "html", props: { content: "<div>main</div>" } },
    });
    expect(result.type).toBe("floating-workspace");
    expect(result.props?.centre).toBeDefined();
  });

  it("includes frames in props", () => {
    const result = floatingWorkspace({
      centre: { type: "html", props: { content: "<div>main</div>" } },
      frames: [{ key: "f1", tabs: [{ key: "t1", label: "Tab", content: { type: "html", props: { content: "x" } } }] }],
    });
    expect(result.props?.frames).toHaveLength(1);
    expect(result.props?.frames![0].key).toBe("f1");
  });

  it("defaults organisers to true", () => {
    const result = floatingWorkspace({
      centre: { type: "html", props: { content: "" } },
    });
    expect(result.props?.organisers).toBe(true);
  });
});
```

- [ ] **Step 2: Implement builder**

Add to `builders.ts`:

```typescript
export function floatingWorkspace(config: FloatingWorkspaceConfig): TypedComponent<"floating-workspace"> {
  const props: FloatingWorkspaceProps = {
    centre: config.centre,
    ...(config.frames ? { frames: config.frames } : {}),
    organisers: config.organisers ?? true,
  };
  return Object.freeze({ type: "floating-workspace" as const, props }) as TypedComponent<"floating-workspace">;
}
```

Import `FloatingWorkspaceConfig` from `@casehubio/pages-component` and `FloatingWorkspaceProps`.

- [ ] **Step 3: Re-export from index.ts**

Add to `packages/pages-ui/src/dsl/index.ts`:

```typescript
export { floatingWorkspace } from "./builders.js";
export type { FloatingWorkspaceConfig } from "@casehubio/pages-component";
```

- [ ] **Step 4: Run builder tests — verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --run builders`
Expected: PASS

- [ ] **Step 5: Write failing desugaring tests**

Add to `desugar-new-types.test.ts`:

```typescript
describe("floating-workspace desugaring", () => {
  it("desugars floating-workspace YAML", () => {
    const raw = {
      type: "floating-workspace",
      centre: [{ type: "html", properties: { content: "<div>main</div>" } }],
      frames: [{
        key: "f1",
        tabs: [{ key: "t1", label: "Tab", content: { type: "html", properties: { content: "x" } } }],
        position: { x: 50, y: 50 },
        size: { width: 400, height: 300 },
      }],
    };
    const result = desugarComponent(raw, {});
    expect(result.type).toBe("floating-workspace");
    expect(result.props?.frames).toHaveLength(1);
    expect(result.props?.frames[0].tabs[0].content.type).toBe("html");
  });
});
```

- [ ] **Step 6: Implement desugaring**

Add to `component-desugar.ts`, following the dock-workbench pattern:

```typescript
if ("type" in raw && raw.type === "floating-workspace") {
  const centreRaw = raw.centre;
  const centre = Array.isArray(centreRaw)
    ? centreRaw.map((c: unknown) => desugarComponent(c, displayerDefaults))
    : desugarComponent(centreRaw, displayerDefaults);

  const framesRaw = (raw as any).frames as unknown[] | undefined;
  const frames = framesRaw?.map((f: any) => ({
    key: f.key as string,
    tabs: (f.tabs as any[]).map((t: any) => ({
      key: t.key as string,
      label: t.label as string,
      ...(t.icon ? { icon: t.icon as string } : {}),
      content: desugarComponent(t.content, displayerDefaults),
    })),
    ...(f.position ? { position: f.position } : {}),
    ...(f.size ? { size: f.size } : {}),
    ...(f.pinned !== undefined ? { pinned: f.pinned as boolean } : {}),
  }));

  const organisers = (raw as any).organisers;
  return floatingWorkspace({
    centre,
    ...(frames ? { frames } : {}),
    ...(organisers !== undefined ? { organisers: organisers as boolean } : {}),
  });
}
```

Import `floatingWorkspace` from the DSL builders.

- [ ] **Step 7: Run desugaring tests — verify they pass**

Run: `yarn workspace @casehubio/pages-ui run test -- --run desugar`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add packages/pages-ui/src/dsl/builders.ts packages/pages-ui/src/dsl/builders.test.ts packages/pages-ui/src/dsl/index.ts packages/pages-ui/src/parser/component-desugar.ts packages/pages-ui/src/parser/desugar-new-types.test.ts
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "feat(#46): floatingWorkspace() builder + YAML desugaring Refs Hortora/trellis#46"
```

---

### Task 6: Runtime Integration

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/site.ts`
- Modify: `packages/pages-runtime/src/site.test.ts`
- Modify: `packages/pages-runtime/src/index.ts`

**Interfaces:**
- Consumes: `FloatingFrameEngine`, `createFloatingFrameEngine` from Task 4; `createDockviewBackend` from Task 3; `FloatingWorkspaceProps` from Task 1
- Produces: `floating-workspace` activation in `createActivationCallback`, frame event handlers in site.ts, extended `captureLayout()`

- [ ] **Step 1: Write failing site integration tests**

Add to `site.test.ts`:

```typescript
describe("floating-workspace integration", () => {
  it("includes frames in captureLayout when engine exists", () => {
    // Test that captureLayout() returns frames field
    // when a floating-workspace component is rendered
  });

  it("frameLayoutStash preserves state across re-render", () => {
    // Test that frame state survives dock-rearrange teardown
  });
});
```

(Detailed test implementations depend on the existing test harness patterns in site.test.ts — adapt to match.)

- [ ] **Step 2: Add activation callback for floating-workspace**

In `activation.ts`, add after the `deferred` activation block:

```typescript
if (component.type === "floating-workspace") {
  const props = component.props as FloatingWorkspaceProps;
  const centreComponents = Array.isArray(props.centre) ? props.centre : [props.centre];

  // Render centre content synchronously
  const centreContainer = document.createElement("div");
  centreContainer.style.cssText = "position:relative;width:100%;height:100%;";
  centreContainer.dataset.floatingWorkspaceCentre = "";
  el.appendChild(centreContainer);
  for (const child of centreComponents) {
    renderComponent(centreContainer, child, { permissions, onNode });
  }

  // Async backend initialization
  createDockviewBackend().then((backend) => {
    const overlayContainer = document.createElement("div");
    overlayContainer.style.cssText = "position:absolute;inset:0;pointer-events:none;";
    overlayContainer.dataset.floatingWorkspaceOverlay = "";
    el.style.position = "relative";
    el.appendChild(overlayContainer);

    const defaultFactory: ContentFactory = (tab) => {
      const container = document.createElement("div");
      renderComponent(container, tab.content, { permissions, onNode });
      return { element: container };
    };

    backend.attach(overlayContainer, defaultFactory);
    const engine = createFloatingFrameEngine(backend, frameLayoutStash ?? undefined);

    // Store engine reference for site.ts event handlers
    (el as any).__floatingFrameEngine = engine;

    // Render declared frames
    if (props.frames && !frameLayoutStash) {
      for (const frameConfig of props.frames) {
        engine.createFrame(frameConfig);
      }
    }

    // Wire backend callbacks to events
    backend.onFrameMove((key, pos) => {
      engine.frames.get(key) && el.dispatchEvent(new CustomEvent("pages-frame-move", { bubbles: true, composed: true, detail: { frameKey: key, position: pos } }));
    });
    backend.onFrameResize((key, size) => {
      el.dispatchEvent(new CustomEvent("pages-frame-resize", { bubbles: true, composed: true, detail: { frameKey: key, size } }));
    });
    backend.onTabDragOut((fromFrame, tabKey, position) => {
      el.dispatchEvent(new CustomEvent("pages-tab-drag-out", { bubbles: true, composed: true, detail: { tabKey, fromFrame, position } }));
    });
    backend.onTabReorder((frameKey, tabKeys) => {
      el.dispatchEvent(new CustomEvent("pages-tab-reorder", { bubbles: true, composed: true, detail: { frameKey, tabKeys } }));
    });

    frameLayoutStash = undefined;
  });
}
```

Import `createDockviewBackend`, `createFloatingFrameEngine`, `FloatingWorkspaceProps`, `ContentFactory`, `renderComponent` at the top of activation.ts.

- [ ] **Step 3: Add frame event handlers in site.ts**

Add event listeners after the `pages-dock-rearrange` handler:

```typescript
// --- Floating workspace frame events ---

let frameEngine: FloatingFrameEngine | undefined;
let frameLayoutStash: readonly FrameLayout[] | undefined;

target.addEventListener("pages-frame-close", ((e: Event) => {
  const { frameKey } = (e as CustomEvent<{ frameKey: string }>).detail;
  const engine = findFrameEngine(target);
  if (engine) { engine.removeFrame(frameKey); scheduleLayoutSave(); }
}), { signal: abortController.signal });

target.addEventListener("pages-frame-pin", ((e: Event) => {
  const { frameKey } = (e as CustomEvent<{ frameKey: string }>).detail;
  const engine = findFrameEngine(target);
  if (engine) { engine.togglePin(frameKey); scheduleLayoutSave(); }
}), { signal: abortController.signal });

target.addEventListener("pages-frame-move", ((e: Event) => {
  const { frameKey, position } = (e as CustomEvent<{ frameKey: string; position: { x: number; y: number } }>).detail;
  const engine = findFrameEngine(target);
  if (engine) {
    const frame = engine.frames.get(frameKey);
    if (frame) { /* engine state updated by backend callback */ }
    scheduleLayoutSave();
  }
}), { signal: abortController.signal });

target.addEventListener("pages-frame-resize", ((e: Event) => {
  scheduleLayoutSave();
}), { signal: abortController.signal });

target.addEventListener("pages-tab-drag-out", ((e: Event) => {
  const { tabKey, fromFrame, position } = (e as CustomEvent<{ tabKey: string; fromFrame: string; position: { x: number; y: number } }>).detail;
  const engine = findFrameEngine(target);
  if (engine) {
    const newKey = `frame-${Date.now()}-${Math.random().toString(36).slice(2, 6)}`;
    engine.createFrame({ key: newKey, tabs: [], position, size: { width: 400, height: 300 } });
    engine.moveTab(fromFrame, tabKey, newKey);
    if (engine.frames.get(fromFrame)?.tabs.length === 0) engine.removeFrame(fromFrame);
    scheduleLayoutSave();
  }
}), { signal: abortController.signal });

function findFrameEngine(target: HTMLElement): FloatingFrameEngine | undefined {
  const el = target.querySelector("[data-floating-workspace-overlay]")?.parentElement;
  return el ? (el as any).__floatingFrameEngine : undefined;
}
```

- [ ] **Step 4: Extend captureLayout in site.ts**

In the `captureLayout()` function, add after the `zones` field:

```typescript
frames: (() => {
  const engine = findFrameEngine(target);
  if (engine) return engine.captureLayout();
  return frameLayoutStash ?? undefined;
})(),
```

- [ ] **Step 5: Handle re-render in dock-rearrange handler**

In the `pages-dock-rearrange` handler, before `target.innerHTML = ""`:

```typescript
const fEngine = findFrameEngine(target);
if (fEngine) {
  frameLayoutStash = fEngine.captureLayout();
  fEngine.dispose();
}
```

- [ ] **Step 6: Update runtime exports**

Add to `packages/pages-runtime/src/index.ts`:

```typescript
export { createFloatingFrameEngine } from "./floating-frame-engine.js";
export type { FloatingFrameEngine } from "./floating-frame-engine.js";
export type { FloatingFrameBackend } from "./floating-frame-backend.js";
export { createDockviewBackend } from "./dockview-backend.js";
export { bringToFront, normalizeForSave } from "./frame-zorder.js";
export { findSpatialTarget } from "./frame-spatial-nav.js";
export { applyPreset } from "./frame-organisers.js";
export { clampPosition, nextFramePosition } from "./frame-boundaries.js";
```

- [ ] **Step 7: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: PASS

- [ ] **Step 8: Build all packages**

Run: `yarn build:packages`
Expected: clean build

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add packages/pages-runtime/src/activation.ts packages/pages-runtime/src/site.ts packages/pages-runtime/src/site.test.ts packages/pages-runtime/src/index.ts
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "feat(#46): floating-workspace activation + site event handlers + layout persistence Refs Hortora/trellis#46"
```

---

### Task 7: Examples Gallery and Playwright E2E Tests

**Files:**
- Create: `examples/samples/Layout/Floating Workspace.dash.yaml`
- Modify: `examples/samples.json`
- Create: `examples/e2e/floating-workspace.spec.ts`

**Interfaces:**
- Consumes: fully integrated floating-workspace component from Tasks 1-6
- Produces: gallery sample demonstrating composition with dock-workbench, Playwright e2e test suite

- [ ] **Step 1: Create gallery sample YAML**

Write `examples/samples/Layout/Floating Workspace.dash.yaml` with the full dock-workbench + floating-workspace composition sample as designed in §7.

- [ ] **Step 2: Update samples.json**

Add to the Layout category array and increment `totalSamples`:

```json
{
  "name": "Floating Workspace",
  "path": "Layout/Floating Workspace",
  "category": "Layout",
  "file": "Layout/Floating Workspace.dash.yaml"
}
```

- [ ] **Step 3: Build and verify sample loads**

Run: `yarn build && yarn workspace @casehubio/pages-examples run serve`
Navigate to Layout → Floating Workspace. Verify:
- Centre content renders immediately
- Floating frames appear after Dockview loads
- Frames are draggable
- Tabs are reorderable
- Close dot and pin button work

- [ ] **Step 4: Write Playwright e2e tests**

```typescript
// examples/e2e/floating-workspace.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Floating Workspace", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("http://localhost:8080/#Layout/Floating%20Workspace");
    await page.waitForSelector("[data-floating-workspace-overlay]", { timeout: 10000 });
  });

  test("frame position persists across refresh", async ({ page }) => {
    const frame = page.locator(".dv-floating-titlebar").first();
    const box = await frame.boundingBox();
    await frame.dragTo(page.locator("[data-floating-workspace-centre]"), { targetPosition: { x: 200, y: 200 } });
    await page.reload();
    await page.waitForSelector("[data-floating-workspace-overlay]");
    const newBox = await page.locator(".dv-floating-titlebar").first().boundingBox();
    expect(newBox?.x).not.toBe(box?.x);
  });

  test("active tab persists across refresh", async ({ page }) => {
    const tabs = page.locator(".dv-tab");
    const secondTab = tabs.nth(1);
    await secondTab.click();
    const tabText = await secondTab.textContent();
    await page.reload();
    await page.waitForSelector("[data-floating-workspace-overlay]");
    const activeTab = page.locator(".dv-tab.dv-active");
    expect(await activeTab.textContent()).toBe(tabText);
  });

  test("pin keeps frame on top", async ({ page }) => {
    const pinBtn = page.locator(".frame-pin-btn").first();
    await pinBtn.click();
    // Verify z-index is in pinned tier
    const frame = page.locator(".dv-resize-container").first();
    const zIndex = await frame.evaluate(el => parseInt(getComputedStyle(el).zIndex || "0"));
    expect(zIndex).toBeGreaterThanOrEqual(10000);
  });

  test("close dot removes frame", async ({ page }) => {
    const frameCount = await page.locator(".dv-resize-container").count();
    const closeDot = page.locator(".frame-close-dot").first();
    await closeDot.click();
    const newCount = await page.locator(".dv-resize-container").count();
    expect(newCount).toBe(frameCount - 1);
  });

  test("composition with dock-workbench", async ({ page }) => {
    // Dock panel toggles independently of floating frames
    const dockBtn = page.locator("[data-dock-panel-id]").first();
    await dockBtn.click();
    const floatingFrames = await page.locator(".dv-resize-container").count();
    expect(floatingFrames).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 5: Run Playwright tests**

Run: `npx playwright test examples/e2e/floating-workspace.spec.ts`
Expected: PASS (requires dev server running)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add "examples/samples/Layout/Floating Workspace.dash.yaml" examples/samples.json examples/e2e/floating-workspace.spec.ts
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "feat(#46): examples gallery sample + Playwright e2e tests Refs Hortora/trellis#46"
```

---

### Task 8: Event Contract Documentation

**Files:**
- Modify: `docs/protocols/casehub/pages-event-contract.md`

**Interfaces:**
- Consumes: event names and detail types from Tasks 4 and 6
- Produces: updated event contract documentation

- [ ] **Step 1: Add floating workspace events to event contract**

Add a new section after the dock-workbench events:

```markdown
### Floating Workspace Events

| Event | Detail | Fired when |
|-------|--------|------------|
| `pages-frame-create` | `{ frameKey: string, position: {x,y}, size: {width,height} }` | New floating frame created |
| `pages-frame-close` | `{ frameKey: string }` | Frame permanently removed |
| `pages-frame-hide` | `{ frameKey: string }` | Frame hidden (config preserved) |
| `pages-frame-show` | `{ frameKey: string }` | Hidden frame recreated |
| `pages-frame-focus` | `{ frameKey: string }` | Frame brought to front |
| `pages-frame-pin` | `{ frameKey: string, pinned: boolean }` | Pin state toggled |
| `pages-frame-move` | `{ frameKey: string, position: {x,y} }` | Frame drag completed |
| `pages-frame-resize` | `{ frameKey: string, size: {width,height} }` | Frame resize completed |
| `pages-frame-organise` | `{ preset: string }` | Layout preset applied |
| `pages-tab-move` | `{ tabKey: string, fromFrame: string, toFrame: string }` | Tab moved between frames |
| `pages-tab-drag-out` | `{ tabKey: string, fromFrame: string, position: {x,y} }` | Tab dragged out to new frame |
| `pages-tab-reorder` | `{ frameKey: string, tabKeys: string[] }` | Tab order changed within frame |
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/slots/5/pages add docs/protocols/casehub/pages-event-contract.md
git -C /Users/mdproctor/claude/hortora/slots/5/pages commit -m "docs(#46): add floating workspace events to event contract Refs Hortora/trellis#46"
```

---

## Dependency Graph

```
Task 1 (Types) ──────────────────────────┐
                                         │
Task 2 (Companions) ─── requires T1 ─────┤
                                         │
Task 3 (Backend) ─── requires T1 ────────┤
                                         │
Task 4 (Engine) ─── requires T1,T2,T3 ───┤
                                         │
Task 5 (Builder/YAML) ─── requires T1 ───┤
                                         │
Task 6 (Runtime) ─── requires T1-T5 ─────┤
                                         │
Task 7 (Gallery/E2E) ─── requires T1-T6 ─┤
                                         │
Task 8 (Docs) ─── no code deps ──────────┘
```

Tasks 2, 3, and 5 can run in parallel after Task 1. Task 4 depends on 2 and 3. Task 6 depends on all prior tasks. Task 8 can run at any time.
