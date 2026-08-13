# Workspace Compositor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** (to be created — workspace compositor)
**Issue group:** single issue

**Goal:** Add splits, tabs, and accordion view to the floating workspace via a compositor layer above FloatingFrameEngine.

**Architecture:** New `WorkspaceCompositor` module sits above the existing `FloatingFrameEngine`. Each tab in each region gets its own engine instance via `wireFloatingWorkspace()`. The compositor coordinates cross-tab operations, persistence, and view mode switching. Existing engine, wire function, and backend stay unchanged.

**Tech Stack:** TypeScript 5, Vitest, plain DOM (no Lit — consistent with workbench chrome patterns)

## Global Constraints

- No Lit dependency in pages-runtime — all workbench chrome is plain DOM
- Content-agnostic — compositor manages layout, never content types
- Backward compatible — missing `compositor` field loads legacy `frames` into single-tab single-region
- Max 2 regions — one split, two leaves, no nesting
- Design tokens for all styling: `--pages-neutral-*`, `--pages-accent-*`, `--pages-radius-*`

---

### Task 1: Compositor Types and Core State Management

**Files:**
- Modify: `packages/pages-component/src/model/types.ts`
- Modify: `packages/pages-component/src/model/index.ts`
- Create: `packages/pages-runtime/src/workspace-compositor.ts`
- Test: `packages/pages-runtime/src/workspace-compositor.test.ts`

**Interfaces:**
- Consumes: `FrameLayout` from `@casehubio/pages-component`
- Produces: `CompositorState`, `LeafRegionState`, `SplitRegionState`, `TabState` types; `WorkspaceCompositor` interface with `createTab()`, `closeTab()`, `renameTab()`, `activateTab()`, `reorderTab()`, `moveTabToRegion()`, `splitRegion()`, `collapseRegion()`, `setViewMode()`, `getRegion()`, `root` getter

- [ ] **Step 1: Add compositor types to pages-component**

In `packages/pages-component/src/model/types.ts`, after the `LayoutState` interface, add:

```typescript
export interface CompositorState {
  readonly region: LeafRegionState | SplitRegionState;
}

export interface LeafRegionState {
  readonly type: "leaf";
  readonly id: string;
  readonly tabs: readonly TabState[];
  readonly activeTabId: string;
  readonly viewMode: "tab" | "accordion";
  readonly accordionHeights?: Readonly<Record<string, number>>;
}

export interface SplitRegionState {
  readonly type: "split";
  readonly direction: "h" | "v";
  readonly ratio: number;
  readonly children: [LeafRegionState, LeafRegionState];
}

export interface TabState {
  readonly id: string;
  readonly name: string;
  readonly frames: readonly FrameLayout[];
}
```

Add `compositor?` to `LayoutState`:

```typescript
readonly compositor?: CompositorState;
```

- [ ] **Step 2: Export new types from pages-component**

In `packages/pages-component/src/model/index.ts`, add:

```typescript
export type { CompositorState, LeafRegionState, SplitRegionState, TabState } from "./types.js";
```

- [ ] **Step 3: Write failing tests for compositor core**

Create `packages/pages-runtime/src/workspace-compositor.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { createWorkspaceCompositor } from "./workspace-compositor.js";
import type { WorkspaceCompositor } from "./workspace-compositor.js";

describe("WorkspaceCompositor", () => {
  function makeCompositor(): WorkspaceCompositor {
    return createWorkspaceCompositor();
  }

  describe("initial state", () => {
    it("starts with a single leaf region with one tab", () => {
      const c = makeCompositor();
      expect(c.root.type).toBe("leaf");
      const leaf = c.root as { type: "leaf"; tabs: { id: string; name: string }[] };
      expect(leaf.tabs).toHaveLength(1);
      expect(leaf.tabs[0]!.name).toBe("Tab 1");
      expect(leaf.viewMode).toBe("tab");
    });
  });

  describe("tab lifecycle", () => {
    it("creates a new tab in a leaf region", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      const leaf = c.root as { tabs: { id: string; name: string }[] };
      expect(leaf.tabs).toHaveLength(2);
      expect(leaf.tabs[1]!.name).toBe("Tab 2");
    });

    it("closes a tab", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      const tabId = (c.root as { tabs: { id: string }[] }).tabs[1]!.id;
      c.closeTab(regionId, tabId);
      expect((c.root as { tabs: { id: string }[] }).tabs).toHaveLength(1);
    });

    it("renames a tab", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      const tabId = (c.root as { tabs: { id: string }[] }).tabs[0]!.id;
      c.renameTab(tabId, "My Workspace");
      expect((c.root as { tabs: { name: string }[] }).tabs[0]!.name).toBe("My Workspace");
    });

    it("activates a tab", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      const tab2Id = (c.root as { tabs: { id: string }[] }).tabs[1]!.id;
      c.activateTab(regionId, tab2Id);
      expect((c.root as { activeTabId: string }).activeTabId).toBe(tab2Id);
    });

    it("reorders tabs", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      c.createTab(regionId);
      const tabs = (c.root as { tabs: { id: string }[] }).tabs;
      c.reorderTab(regionId, tabs[2]!.id, 0);
      const reordered = (c.root as { tabs: { id: string }[] }).tabs;
      expect(reordered[0]!.id).toBe(tabs[2]!.id);
    });
  });

  describe("view mode", () => {
    it("toggles between tab and accordion view", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      expect((c.root as { viewMode: string }).viewMode).toBe("tab");
      c.setViewMode(regionId, "accordion");
      expect((c.root as { viewMode: string }).viewMode).toBe("accordion");
    });

    it("preserves accordion heights across view mode toggles", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      const tabId = (c.root as { tabs: { id: string }[] }).tabs[0]!.id;
      c.setViewMode(regionId, "accordion");
      c.setAccordionHeight(regionId, tabId, 400);
      c.setViewMode(regionId, "tab");
      c.setViewMode(regionId, "accordion");
      expect((c.root as { accordionHeights: Record<string, number> }).accordionHeights[tabId]).toBe(400);
    });
  });

  describe("split and collapse", () => {
    it("splits a leaf region into two", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      const tabId = (c.root as { tabs: { id: string }[] }).tabs[1]!.id;
      c.splitRegion(regionId, tabId, "v");
      expect(c.root.type).toBe("split");
      const split = c.root as { direction: string; ratio: number; children: { tabs: { id: string }[] }[] };
      expect(split.direction).toBe("v");
      expect(split.ratio).toBe(0.5);
      expect(split.children[0]!.tabs).toHaveLength(1);
      expect(split.children[1]!.tabs).toHaveLength(1);
      expect(split.children[1]!.tabs[0]!.id).toBe(tabId);
    });

    it("collapses when last tab in a region is closed", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      const tabId = (c.root as { tabs: { id: string }[] }).tabs[1]!.id;
      c.splitRegion(regionId, tabId, "h");
      const split = c.root as { children: { id: string; tabs: { id: string }[] }[] };
      const rightRegionId = split.children[1]!.id;
      const rightTabId = split.children[1]!.tabs[0]!.id;
      c.closeTab(rightRegionId, rightTabId);
      expect(c.root.type).toBe("leaf");
    });

    it("updates split ratio", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      const tabId = (c.root as { tabs: { id: string }[] }).tabs[1]!.id;
      c.splitRegion(regionId, tabId, "v");
      c.setSplitRatio(0.3);
      expect((c.root as { ratio: number }).ratio).toBe(0.3);
    });
  });

  describe("cross-region tab move", () => {
    it("moves a tab from one region to another", () => {
      const c = makeCompositor();
      const regionId = (c.root as { id: string }).id;
      c.createTab(regionId);
      c.createTab(regionId);
      const tabToMove = (c.root as { tabs: { id: string }[] }).tabs[2]!.id;
      c.splitRegion(regionId, tabToMove, "v");
      const split = c.root as { children: { id: string; tabs: { id: string }[] }[] };
      const leftId = split.children[0]!.id;
      const rightId = split.children[1]!.id;
      const leftTab = split.children[0]!.tabs[1]!.id;
      c.moveTabToRegion(leftTab, leftId, rightId);
      const updated = c.root as { children: { tabs: { id: string }[] }[] };
      expect(updated.children[0]!.tabs).toHaveLength(1);
      expect(updated.children[1]!.tabs).toHaveLength(2);
    });
  });
});
```

- [ ] **Step 4: Run tests — verify they fail**

```bash
GH_PACKAGES_TOKEN=$(gh auth token) yarn workspace @casehubio/pages-runtime run test src/workspace-compositor.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 5: Implement compositor core**

Create `packages/pages-runtime/src/workspace-compositor.ts`:

```typescript
import type {
  LeafRegionState,
  SplitRegionState,
  TabState,
} from "@casehubio/pages-component";

type Region = LeafRegionState | SplitRegionState;

export interface WorkspaceCompositor {
  readonly root: Region;
  createTab(regionId: string): string;
  closeTab(regionId: string, tabId: string): void;
  renameTab(tabId: string, name: string): void;
  activateTab(regionId: string, tabId: string): void;
  reorderTab(regionId: string, tabId: string, newIndex: number): void;
  moveTabToRegion(tabId: string, fromRegionId: string, toRegionId: string): void;
  splitRegion(regionId: string, tabId: string, direction: "h" | "v"): void;
  collapseRegion(regionId: string): void;
  setSplitRatio(ratio: number): void;
  setViewMode(regionId: string, mode: "tab" | "accordion"): void;
  setAccordionHeight(regionId: string, tabId: string, height: number): void;
  getLeafRegion(regionId: string): LeafRegionState | null;
  dispose(): void;
}

let tabCounter = 0;
let regionCounter = 0;

function nextTabId(): string { return `tab-${++tabCounter}`; }
function nextRegionId(): string { return `region-${++regionCounter}`; }

function makeTab(name: string): TabState {
  return { id: nextTabId(), name };
}

function makeLeaf(tabs: TabState[], activeTabId?: string): LeafRegionState {
  return {
    type: "leaf",
    id: nextRegionId(),
    tabs,
    activeTabId: activeTabId ?? tabs[0]?.id ?? "",
    viewMode: "tab",
  };
}

export function createWorkspaceCompositor(): WorkspaceCompositor {
  const firstTab = makeTab("Tab 1");
  let root: Region = makeLeaf([firstTab], firstTab.id);
  let tabNameCounter = 1;

  function findLeaf(regionId: string): LeafRegionState | null {
    if (root.type === "leaf") return root.id === regionId ? root : null;
    return root.children.find(c => c.id === regionId) ?? null;
  }

  function replaceLeaf(regionId: string, updater: (leaf: LeafRegionState) => Region): void {
    if (root.type === "leaf" && root.id === regionId) {
      root = updater(root);
      return;
    }
    if (root.type === "split") {
      const idx = root.children.findIndex(c => c.id === regionId);
      if (idx >= 0) {
        const result = updater(root.children[idx]!);
        if (result.type === "leaf") {
          const newChildren = [...root.children] as [LeafRegionState, LeafRegionState];
          newChildren[idx] = result;
          root = { ...root, children: newChildren };
        }
      }
    }
  }

  const compositor: WorkspaceCompositor = {
    get root() { return root; },

    createTab(regionId: string): string {
      const tab = makeTab(`Tab ${++tabNameCounter}`);
      replaceLeaf(regionId, leaf => ({
        ...leaf,
        tabs: [...leaf.tabs, tab],
      }));
      return tab.id;
    },

    closeTab(regionId: string, tabId: string) {
      const leaf = findLeaf(regionId);
      if (!leaf) return;
      const remaining = leaf.tabs.filter(t => t.id !== tabId);
      if (remaining.length === 0 && root.type === "split") {
        compositor.collapseRegion(regionId);
        return;
      }
      const activeTabId = leaf.activeTabId === tabId
        ? (remaining[0]?.id ?? "")
        : leaf.activeTabId;
      replaceLeaf(regionId, () => ({ ...leaf, tabs: remaining, activeTabId }));
    },

    renameTab(tabId: string, name: string) {
      function updateTabs(leaf: LeafRegionState): LeafRegionState {
        return {
          ...leaf,
          tabs: leaf.tabs.map(t => t.id === tabId ? { ...t, name } : t),
        };
      }
      if (root.type === "leaf") {
        root = updateTabs(root);
      } else {
        root = {
          ...root,
          children: root.children.map(updateTabs) as [LeafRegionState, LeafRegionState],
        };
      }
    },

    activateTab(regionId: string, tabId: string) {
      replaceLeaf(regionId, leaf => ({ ...leaf, activeTabId: tabId }));
    },

    reorderTab(regionId: string, tabId: string, newIndex: number) {
      replaceLeaf(regionId, leaf => {
        const tabs = [...leaf.tabs];
        const oldIdx = tabs.findIndex(t => t.id === tabId);
        if (oldIdx < 0) return leaf;
        const [moved] = tabs.splice(oldIdx, 1);
        tabs.splice(newIndex, 0, moved!);
        return { ...leaf, tabs };
      });
    },

    moveTabToRegion(tabId: string, fromRegionId: string, toRegionId: string) {
      if (root.type !== "split") return;
      const from = findLeaf(fromRegionId);
      const to = findLeaf(toRegionId);
      if (!from || !to) return;
      const tab = from.tabs.find(t => t.id === tabId);
      if (!tab) return;
      const remainingFrom = from.tabs.filter(t => t.id !== tabId);
      if (remainingFrom.length === 0) {
        compositor.collapseRegion(fromRegionId);
        return;
      }
      const activeFrom = from.activeTabId === tabId
        ? (remainingFrom[0]?.id ?? "")
        : from.activeTabId;
      const newFrom: LeafRegionState = { ...from, tabs: remainingFrom, activeTabId: activeFrom };
      const newTo: LeafRegionState = { ...to, tabs: [...to.tabs, tab] };
      root = {
        ...root,
        children: root.children.map(c => {
          if (c.id === fromRegionId) return newFrom;
          if (c.id === toRegionId) return newTo;
          return c;
        }) as [LeafRegionState, LeafRegionState],
      };
    },

    splitRegion(regionId: string, tabId: string, direction: "h" | "v") {
      if (root.type !== "leaf" || root.id !== regionId) return;
      const tab = root.tabs.find(t => t.id === tabId);
      if (!tab) return;
      const remaining = root.tabs.filter(t => t.id !== tabId);
      if (remaining.length === 0) return;
      const activeTabId = root.activeTabId === tabId
        ? (remaining[0]?.id ?? "")
        : root.activeTabId;
      const leftLeaf: LeafRegionState = {
        ...root,
        tabs: remaining,
        activeTabId,
      };
      const rightLeaf = makeLeaf([tab], tab.id);
      root = {
        type: "split",
        direction,
        ratio: 0.5,
        children: [leftLeaf, rightLeaf],
      };
    },

    collapseRegion(regionId: string) {
      if (root.type !== "split") return;
      const survivor = root.children.find(c => c.id !== regionId);
      if (survivor) root = survivor;
    },

    setSplitRatio(ratio: number) {
      if (root.type !== "split") return;
      root = { ...root, ratio: Math.max(0.1, Math.min(0.9, ratio)) };
    },

    setViewMode(regionId: string, mode: "tab" | "accordion") {
      replaceLeaf(regionId, leaf => ({ ...leaf, viewMode: mode }));
    },

    setAccordionHeight(regionId: string, tabId: string, height: number) {
      replaceLeaf(regionId, leaf => ({
        ...leaf,
        accordionHeights: { ...leaf.accordionHeights, [tabId]: height },
      }));
    },

    getLeafRegion(regionId: string): LeafRegionState | null {
      return findLeaf(regionId);
    },

    dispose() {
      // cleanup will be added when engine map is integrated
    },
  };

  return compositor;
}
```

- [ ] **Step 6: Run tests — verify they pass**

```bash
GH_PACKAGES_TOKEN=$(gh auth token) yarn workspace @casehubio/pages-runtime run test src/workspace-compositor.test.ts
```

Expected: all tests pass.

- [ ] **Step 7: Build to verify types compile**

```bash
GH_PACKAGES_TOKEN=$(gh auth token) yarn build:packages
```

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/types.ts packages/pages-component/src/model/index.ts packages/pages-runtime/src/workspace-compositor.ts packages/pages-runtime/src/workspace-compositor.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: compositor types and core state management  Refs #<N>"
```

---

### Task 2: Compositor Persistence

**Files:**
- Create: `packages/pages-runtime/src/compositor-persistence.ts`
- Test: `packages/pages-runtime/src/compositor-persistence.test.ts`

**Interfaces:**
- Consumes: `WorkspaceCompositor` from Task 1, `CompositorState`, `LayoutState`, `FrameLayout` from `@casehubio/pages-component`
- Produces: `captureCompositorState(compositor, engineMap): CompositorState`, `restoreCompositorState(state): WorkspaceCompositor`, `migrateFromLegacyFrames(frames): CompositorState`

- [ ] **Step 1: Write failing tests**

Create `packages/pages-runtime/src/compositor-persistence.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { captureCompositorState, restoreCompositorState, migrateFromLegacyFrames } from "./compositor-persistence.js";
import { createWorkspaceCompositor } from "./workspace-compositor.js";
import type { FrameLayout, LeafRegionState, CompositorState } from "@casehubio/pages-component";

const stubFrame: FrameLayout = {
  key: "f1", order: 0, position: { x: 0, y: 0 }, size: { width: 400, height: 300 },
  zIndex: 1, pinned: false, hidden: false, tabs: [], activeTabKey: "",
};

describe("compositor-persistence", () => {
  describe("captureCompositorState", () => {
    it("captures single-region single-tab state", () => {
      const c = createWorkspaceCompositor();
      const regionId = (c.root as LeafRegionState).id;
      const tabId = (c.root as LeafRegionState).tabs[0]!.id;
      const engineMap = new Map([[tabId, { captureLayout: () => [stubFrame] }]]);
      const state = captureCompositorState(c, engineMap);
      expect(state.region.type).toBe("leaf");
      const leaf = state.region as LeafRegionState & { tabs: { frames: FrameLayout[] }[] };
      expect(leaf.tabs[0]!.frames).toEqual([stubFrame]);
    });
  });

  describe("restoreCompositorState", () => {
    it("round-trips through capture and restore", () => {
      const c = createWorkspaceCompositor();
      const regionId = (c.root as LeafRegionState).id;
      const tabId = (c.root as LeafRegionState).tabs[0]!.id;
      const engineMap = new Map([[tabId, { captureLayout: () => [stubFrame] }]]);
      const state = captureCompositorState(c, engineMap);
      const restored = restoreCompositorState(state);
      expect(restored.root.type).toBe("leaf");
      const leaf = restored.root as LeafRegionState;
      expect(leaf.tabs).toHaveLength(1);
      expect(leaf.viewMode).toBe("tab");
    });
  });

  describe("migrateFromLegacyFrames", () => {
    it("wraps legacy frames in single-tab single-region", () => {
      const state = migrateFromLegacyFrames([stubFrame]);
      expect(state.region.type).toBe("leaf");
      const leaf = state.region as LeafRegionState & { tabs: { frames: FrameLayout[] }[] };
      expect(leaf.tabs).toHaveLength(1);
      expect(leaf.tabs[0]!.name).toBe("Tab 1");
      expect(leaf.tabs[0]!.frames).toEqual([stubFrame]);
    });

    it("handles empty frames array", () => {
      const state = migrateFromLegacyFrames([]);
      const leaf = state.region as LeafRegionState & { tabs: { frames: FrameLayout[] }[] };
      expect(leaf.tabs[0]!.frames).toEqual([]);
    });
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
GH_PACKAGES_TOKEN=$(gh auth token) yarn workspace @casehubio/pages-runtime run test src/compositor-persistence.test.ts
```

- [ ] **Step 3: Implement persistence module**

Create `packages/pages-runtime/src/compositor-persistence.ts`:

```typescript
import type {
  CompositorState,
  LeafRegionState,
  SplitRegionState,
  TabState,
  FrameLayout,
} from "@casehubio/pages-component";
import { createWorkspaceCompositor } from "./workspace-compositor.js";
import type { WorkspaceCompositor } from "./workspace-compositor.js";

interface EngineCapture {
  captureLayout(): readonly FrameLayout[];
}

export function captureCompositorState(
  compositor: WorkspaceCompositor,
  engineMap: Map<string, EngineCapture>,
): CompositorState {
  function captureLeaf(leaf: LeafRegionState): LeafRegionState {
    const tabs: TabState[] = leaf.tabs.map(tab => {
      const engine = engineMap.get(tab.id);
      const frames = engine ? engine.captureLayout() : [];
      return { ...tab, frames };
    });
    return { ...leaf, tabs };
  }

  if (compositor.root.type === "leaf") {
    return { region: captureLeaf(compositor.root) };
  }
  return {
    region: {
      ...compositor.root,
      children: compositor.root.children.map(captureLeaf) as [LeafRegionState, LeafRegionState],
    },
  };
}

export function restoreCompositorState(state: CompositorState): WorkspaceCompositor {
  const compositor = createWorkspaceCompositor(state);
  return compositor;
}

export function migrateFromLegacyFrames(frames: readonly FrameLayout[]): CompositorState {
  const tab: TabState = { id: "tab-legacy-1", name: "Tab 1", frames };
  const region: LeafRegionState = {
    type: "leaf",
    id: "region-legacy-1",
    tabs: [tab],
    activeTabId: tab.id,
    viewMode: "tab",
  };
  return { region };
}
```

Note: `createWorkspaceCompositor` needs an optional `initialState?: CompositorState` parameter. Update the factory in `workspace-compositor.ts` to accept and apply it — set `root = initialState.region` and reconstruct tab/region counters from existing IDs.

- [ ] **Step 4: Update createWorkspaceCompositor to accept initial state**

In `workspace-compositor.ts`, change the factory signature:

```typescript
export function createWorkspaceCompositor(initialState?: CompositorState): WorkspaceCompositor {
```

When `initialState` is provided, set `root = initialState.region` and derive `tabNameCounter` from the highest existing tab number.

- [ ] **Step 5: Run tests — verify they pass**

```bash
GH_PACKAGES_TOKEN=$(gh auth token) yarn workspace @casehubio/pages-runtime run test src/compositor-persistence.test.ts
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/compositor-persistence.ts packages/pages-runtime/src/compositor-persistence.test.ts packages/pages-runtime/src/workspace-compositor.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: compositor persistence — capture, restore, legacy migration  Refs #<N>"
```

---

### Task 3: Compositor Renderer — Tab Bar, Tab View, Accordion

**Files:**
- Create: `packages/pages-runtime/src/compositor-renderer.ts`
- Test: `packages/pages-runtime/src/compositor-renderer.test.ts`

**Interfaces:**
- Consumes: `WorkspaceCompositor` from Task 1
- Produces: `renderCompositor(compositor, container, callbacks): CompositorDOM`, `CompositorDOM.update()`, `CompositorDOM.dispose()`

`callbacks` is an interface the activation code provides:

```typescript
interface CompositorCallbacks {
  onTabCreate(regionId: string): void;
  onTabClose(regionId: string, tabId: string): void;
  onTabActivate(regionId: string, tabId: string): void;
  onTabRename(tabId: string, newName: string): void;
  onViewModeToggle(regionId: string): void;
  onAccordionResize(regionId: string, tabId: string, height: number): void;
  onSplitResize(ratio: number): void;
  getTabContainer(tabId: string): HTMLElement;
}
```

- [ ] **Step 1: Write failing tests for tab bar rendering**

Create `packages/pages-runtime/src/compositor-renderer.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { renderCompositor } from "./compositor-renderer.js";
import type { LeafRegionState } from "@casehubio/pages-component";

function stubCallbacks() {
  return {
    onTabCreate: vi.fn(),
    onTabClose: vi.fn(),
    onTabActivate: vi.fn(),
    onTabRename: vi.fn(),
    onViewModeToggle: vi.fn(),
    onAccordionResize: vi.fn(),
    onSplitResize: vi.fn(),
    getTabContainer: vi.fn(() => document.createElement("div")),
  };
}

describe("compositor-renderer", () => {
  it("renders a tab bar with one tab for a single-region compositor", () => {
    const container = document.createElement("div");
    const region: LeafRegionState = {
      type: "leaf", id: "r1", tabs: [{ id: "t1", name: "Tab 1", frames: [] }],
      activeTabId: "t1", viewMode: "tab",
    };
    const dom = renderCompositor(region, container, stubCallbacks());
    const tabBar = container.querySelector("[data-compositor-tabbar]");
    expect(tabBar).not.toBeNull();
    const headers = container.querySelectorAll("[data-tab-id]");
    expect(headers).toHaveLength(1);
    dom.dispose();
  });

  it("shows add button and view toggle in tab bar", () => {
    const container = document.createElement("div");
    const region: LeafRegionState = {
      type: "leaf", id: "r1", tabs: [{ id: "t1", name: "Tab 1", frames: [] }],
      activeTabId: "t1", viewMode: "tab",
    };
    const dom = renderCompositor(region, container, stubCallbacks());
    expect(container.querySelector("[data-tab-add]")).not.toBeNull();
    expect(container.querySelector("[data-view-toggle]")).not.toBeNull();
    dom.dispose();
  });

  it("calls onTabCreate when add button is clicked", () => {
    const container = document.createElement("div");
    const region: LeafRegionState = {
      type: "leaf", id: "r1", tabs: [{ id: "t1", name: "Tab 1", frames: [] }],
      activeTabId: "t1", viewMode: "tab",
    };
    const cbs = stubCallbacks();
    const dom = renderCompositor(region, container, cbs);
    (container.querySelector("[data-tab-add]") as HTMLElement).click();
    expect(cbs.onTabCreate).toHaveBeenCalledWith("r1");
    dom.dispose();
  });

  it("renders accordion sections when viewMode is accordion", () => {
    const container = document.createElement("div");
    const region: LeafRegionState = {
      type: "leaf", id: "r1",
      tabs: [
        { id: "t1", name: "Tab 1", frames: [] },
        { id: "t2", name: "Tab 2", frames: [] },
      ],
      activeTabId: "t1", viewMode: "accordion",
    };
    const dom = renderCompositor(region, container, stubCallbacks());
    const sections = container.querySelectorAll("[data-accordion-section]");
    expect(sections).toHaveLength(2);
    dom.dispose();
  });

  it("renders split with two regions and a resize handle", () => {
    const container = document.createElement("div");
    const split: SplitRegionState = {
      type: "split", direction: "v", ratio: 0.5,
      children: [
        { type: "leaf", id: "r1", tabs: [{ id: "t1", name: "Tab 1", frames: [] }], activeTabId: "t1", viewMode: "tab" },
        { type: "leaf", id: "r2", tabs: [{ id: "t2", name: "Tab 2", frames: [] }], activeTabId: "t2", viewMode: "tab" },
      ],
    };
    const dom = renderCompositor(split, container, stubCallbacks());
    const regions = container.querySelectorAll("[data-region-id]");
    expect(regions).toHaveLength(2);
    expect(container.querySelector("[data-split-handle]")).not.toBeNull();
    dom.dispose();
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement renderer**

Create `packages/pages-runtime/src/compositor-renderer.ts`. This module creates plain DOM for:

- **Single leaf region:** tab bar (headers + add + view toggle) + content area
- **Split region:** two leaf regions side by side (or stacked) with a resize handle between them
- **Tab view content:** single container per region, shows only active tab's workspace container
- **Accordion view content:** flex column with collapsible sections, resize handles between sections

The renderer returns a `CompositorDOM` object with `update(root)` (re-renders from new state) and `dispose()`.

Implementation details: ~200 lines of plain DOM construction. Each tab header is a `<button>` with `draggable="true"`. Tab close is a nested `<span>`. Add button and view toggle are sibling buttons. Accordion sections use `flex-direction: column` with `overflow-y: auto` on the region container.

**Event dispatching (F4):** Every callback invocation must also dispatch the corresponding compositor event on the container element (`bubbles: true, composed: true`). Example: when `onTabCreate` fires, also dispatch `new CustomEvent("pages-compositor-tab-create", { bubbles: true, composed: true, detail: { regionId, tabId, name } })`. This enables `scheduleLayoutSave()` in site.ts to listen for compositor events and auto-save layout state. All 9 compositor events from the spec's event contract must be dispatched by either the renderer (UI actions) or the compositor core (split/collapse).

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/compositor-renderer.ts packages/pages-runtime/src/compositor-renderer.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: compositor renderer — tab bar, accordion, split layout  Refs #<N>"
```

---

### Task 4: Tab Drag — Reorder, Edge Detection, Split Creation

**Files:**
- Create: `packages/pages-runtime/src/compositor-drag.ts`
- Test: `packages/pages-runtime/src/compositor-drag.test.ts`

**Interfaces:**
- Consumes: `WorkspaceCompositor` from Task 1, `CompositorDOM` from Task 3
- Produces: `setupCompositorDrag(compositor, dom, container, callbacks, signal): void`

- [ ] **Step 1: Write failing tests for edge zone detection**

Create `packages/pages-runtime/src/compositor-drag.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { detectEdgeZone } from "./compositor-drag.js";

describe("compositor-drag", () => {
  describe("detectEdgeZone", () => {
    const containerRect = { x: 0, y: 0, width: 800, height: 600 };

    it("returns 'left' when near left edge", () => {
      expect(detectEdgeZone({ x: 15, y: 300 }, containerRect, 40)).toBe("left");
    });

    it("returns 'right' when near right edge", () => {
      expect(detectEdgeZone({ x: 785, y: 300 }, containerRect, 40)).toBe("right");
    });

    it("returns 'top' when near top edge", () => {
      expect(detectEdgeZone({ x: 400, y: 15 }, containerRect, 40)).toBe("top");
    });

    it("returns 'bottom' when near bottom edge", () => {
      expect(detectEdgeZone({ x: 400, y: 585 }, containerRect, 40)).toBe("bottom");
    });

    it("returns null when in centre", () => {
      expect(detectEdgeZone({ x: 400, y: 300 }, containerRect, 40)).toBeNull();
    });
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement drag module**

Create `packages/pages-runtime/src/compositor-drag.ts`:

```typescript
export type EdgeZone = "left" | "right" | "top" | "bottom";

export function detectEdgeZone(
  pos: { x: number; y: number },
  container: { x: number; y: number; width: number; height: number },
  threshold: number,
): EdgeZone | null {
  const relX = pos.x - container.x;
  const relY = pos.y - container.y;
  if (relX < threshold) return "left";
  if (relX > container.width - threshold) return "right";
  if (relY < threshold) return "top";
  if (relY > container.height - threshold) return "bottom";
  return null;
}

export function edgeToDirection(zone: EdgeZone): "h" | "v" {
  return zone === "top" || zone === "bottom" ? "h" : "v";
}
```

The full `setupCompositorDrag()` function wires `dragstart`, `dragover`, `drop` events on the tab bar and container. It handles:
- Tab reorder within a region (drop on another tab header)
- Edge drop zone detection (drop near container edge → split creation)
- Cross-region tab move (drop on another region's tab bar)

Drag priority: edge drop zone > cross-region tab bar > tab reorder.

Preview overlay during drag: a semi-transparent div (`position: absolute`, `pointer-events: none`) positioned to cover the edge zone. Created on `dragenter`, removed on `dragleave`/`drop`.

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/compositor-drag.ts packages/pages-runtime/src/compositor-drag.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: compositor drag — edge detection, tab reorder, split creation  Refs #<N>"
```

---

### Task 5: Cross-Tab Frame Transfer with State Snapshot

**Files:**
- Create: `packages/pages-runtime/src/compositor-transfer.ts`
- Test: `packages/pages-runtime/src/compositor-transfer.test.ts`

**Interfaces:**
- Consumes: `FloatingFrameEngine` from `./floating-frame-engine.js`, `FrameLayout`, `FrameTabConfig`
- Produces: `TransferSnapshot`, `captureTransferSnapshot(frameEl: HTMLElement): TransferSnapshot`, `applyTransferSnapshot(frameEl: HTMLElement, snapshot: TransferSnapshot): void`, `transferFrame(sourceEngine, targetEngine, frameKey, dropPosition, contentFactory): void`

- [ ] **Step 1: Write failing tests**

Create `packages/pages-runtime/src/compositor-transfer.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { captureTransferSnapshot, applyTransferSnapshot } from "./compositor-transfer.js";

describe("compositor-transfer", () => {
  describe("captureTransferSnapshot", () => {
    it("captures scroll positions", () => {
      const el = document.createElement("div");
      Object.defineProperty(el, "scrollTop", { value: 150, writable: true });
      Object.defineProperty(el, "scrollLeft", { value: 30, writable: true });
      const snapshot = captureTransferSnapshot(el);
      expect(snapshot.scrollTop).toBe(150);
      expect(snapshot.scrollLeft).toBe(30);
    });
  });

  describe("applyTransferSnapshot", () => {
    it("restores scroll positions", () => {
      const el = document.createElement("div");
      Object.defineProperty(el, "scrollTop", { value: 0, writable: true });
      Object.defineProperty(el, "scrollLeft", { value: 0, writable: true });
      applyTransferSnapshot(el, { scrollTop: 150, scrollLeft: 30, inputValues: [] });
      expect(el.scrollTop).toBe(150);
      expect(el.scrollLeft).toBe(30);
    });
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement transfer module**

Create `packages/pages-runtime/src/compositor-transfer.ts`:

```typescript
export interface TransferSnapshot {
  readonly scrollPositions: readonly { selector: string; top: number; left: number }[];
  readonly focusedSelector: string | null;
  readonly inputValues: readonly { selector: string; value: string }[];
  readonly selectionRange: { selector: string; start: number; end: number } | null;
}

export function captureTransferSnapshot(el: HTMLElement): TransferSnapshot {
  const inputValues: { selector: string; value: string }[] = [];
  el.querySelectorAll("input, textarea, select").forEach((input, i) => {
    const selector = `[data-transfer-idx="${i}"]`;
    input.setAttribute("data-transfer-idx", String(i));
    inputValues.push({ selector, value: (input as HTMLInputElement).value });
  });
  return { scrollTop: el.scrollTop, scrollLeft: el.scrollLeft, inputValues };
}

export function applyTransferSnapshot(el: HTMLElement, snapshot: TransferSnapshot): void {
  el.scrollTop = snapshot.scrollTop;
  el.scrollLeft = snapshot.scrollLeft;
  for (const { selector, value } of snapshot.inputValues) {
    const input = el.querySelector(selector) as HTMLInputElement | null;
    if (input) input.value = value;
  }
}

export function transferFrame(
  sourceEngine: FloatingFrameEngine,
  targetEngine: FloatingFrameEngine,
  frameKey: string,
  dropPosition: { x: number; y: number },
): void {
  const frame = sourceEngine.frames.get(frameKey);
  if (!frame) return;
  // Reattach if detached (R2: close child window before transfer)
  if (frame.detached) {
    sourceEngine.setDetached(frameKey, false);
    sourceEngine.showFrame(frameKey);
  }
  const tabs: FrameTabConfig[] = [...frame.tabs];
  sourceEngine.removeFrame(frameKey);
  targetEngine.createFrame({ key: frameKey, tabs, position: dropPosition });
}
```

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/compositor-transfer.ts packages/pages-runtime/src/compositor-transfer.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: cross-tab frame transfer with state snapshot  Refs #<N>"
```

---

### Task 6: Activation and Site Integration

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts` (lines ~693-748)
- Modify: `packages/pages-runtime/src/site.ts` (lines ~1130-1167)
- Modify: `packages/pages-runtime/src/index.ts`
- Modify: `docs/protocols/casehub/pages-event-contract.md`

**Interfaces:**
- Consumes: `WorkspaceCompositor` from Task 1, `CompositorDOM` from Task 3, `captureCompositorState` / `restoreCompositorState` / `migrateFromLegacyFrames` from Task 2, `setupCompositorDrag` from Task 4, `transferFrame` from Task 5
- Produces: Working compositor integration — the final wiring that makes everything run

- [ ] **Step 1: Update activation.ts — replace single engine with compositor**

In `packages/pages-runtime/src/activation.ts`, the floating workspace activation block (lines ~693-748) currently creates one `wireFloatingWorkspace()` call. Replace with:

1. Import `createWorkspaceCompositor`, `renderCompositor`, `captureCompositorState`, `restoreCompositorState`, `migrateFromLegacyFrames`
2. Check if `seedLayout?.compositor` exists → `restoreCompositorState(seedLayout.compositor)`, else if `seedLayout?.frames` exists → restore from `migrateFromLegacyFrames(seedLayout.frames)`, else → `createWorkspaceCompositor()` (default single tab)
3. For each tab in each leaf region, create `wireFloatingWorkspace()` + restore the tab's frames
4. Store the `tabId → WireHandle` map on the compositor ref
5. Wire `CompositorCallbacks` to the compositor methods
6. On tab creation: create new `wireFloatingWorkspace()`, wire keyboard handler and organiser toolbar
7. On tab close: dispose the tab's engine

- [ ] **Step 2: Update site.ts — captureLayout delegates to compositor**

In `packages/pages-runtime/src/site.ts`:

1. Change `floatingWorkspaceRef` to carry `compositor?: WorkspaceCompositor` and `engineMap?: Map<string, WireHandle>` alongside the existing `engine` field
2. In `captureLayout()` (~line 1130): if `floatingWorkspaceRef.compositor` exists, call `captureCompositorState(compositor, engineMap)` and include `compositor` field in the returned `LayoutState`
3. In seed layout loading (~line 1156): if `seedLayout.compositor` exists, set `floatingWorkspaceRef.compositorStash = seedLayout.compositor`
4. Add compositor event names to `scheduleLayoutSave()` triggers: `pages-compositor-tab-create`, `pages-compositor-tab-close`, `pages-compositor-split`, `pages-compositor-collapse`, `pages-compositor-view-mode`

- [ ] **Step 3: Update index.ts — export compositor modules**

Add to `packages/pages-runtime/src/index.ts`:

```typescript
export { createWorkspaceCompositor } from "./workspace-compositor.js";
export type { WorkspaceCompositor } from "./workspace-compositor.js";
export { captureCompositorState, restoreCompositorState, migrateFromLegacyFrames } from "./compositor-persistence.js";
export { renderCompositor } from "./compositor-renderer.js";
export type { CompositorCallbacks, CompositorDOM } from "./compositor-renderer.js";
export { detectEdgeZone } from "./compositor-drag.js";
export { transferFrame, captureTransferSnapshot, applyTransferSnapshot } from "./compositor-transfer.js";
```

- [ ] **Step 4: Update event contract protocol**

Add to `docs/protocols/casehub/pages-event-contract.md` reserved names table:

```markdown
| `pages-compositor-tab-create` | New tab created in region | Compositor renderer |
| `pages-compositor-tab-close` | Tab closed | Compositor renderer |
| `pages-compositor-tab-rename` | Tab renamed | Compositor renderer |
| `pages-compositor-tab-activate` | Tab switched | Compositor renderer |
| `pages-compositor-tab-move` | Tab moved cross-region | Compositor drag |
| `pages-compositor-split` | Split created | Compositor drag |
| `pages-compositor-collapse` | Split collapsed (last tab closed) | Compositor |
| `pages-compositor-view-mode` | View mode toggled | Compositor renderer |
| `pages-frame-transfer` | Frame moved cross-tab | Compositor transfer |
```

- [ ] **Step 5: Run full test suite**

```bash
GH_PACKAGES_TOKEN=$(gh auth token) yarn workspace @casehubio/pages-runtime run test
```

Expected: all tests pass (including the pre-existing dark mode failure which is unrelated).

- [ ] **Step 6: Build everything**

```bash
GH_PACKAGES_TOKEN=$(gh auth token) yarn build
```

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/activation.ts packages/pages-runtime/src/site.ts packages/pages-runtime/src/index.ts docs/protocols/casehub/pages-event-contract.md
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: compositor activation and site integration  Refs #<N>"
```
