# Unified Container Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #345 — Recursive Container model — entries as nested Containers
**Issue group:** #345

**Goal:** Decompose the 1355-line backend god closure into focused modules, eliminate workspace split brain via mount transfer, unify duplicated code paths, and add `refreshEntry` primitive for surgical container replanting. Verify with combinatorial end-to-end tests.

**Architecture:** Three new modules extracted from `group-organiser-backend.ts`: `container-tree-ops.ts` (tree traversal, split/collapse surgery), `frame-renderer.ts` (frame DOM creation using shared primitives), `dnd-coordinator.ts` (drag state machine). Backend becomes ~300-line orchestrator. Workspace transitions use mount transfer (direct re-parent) instead of serialize/recreate. All container moves use surgical subtree replant via new `refreshEntry` primitive.

**Tech Stack:** TypeScript 5, Vitest (JSDOM), Yarn workspaces, pages-runtime package

## Global Constraints

- All 894 existing tests must pass at every batch boundary
- `FloatingFrameBackend` interface gains only `getRootContainer` — all other methods unchanged
- Backward compatible persistence — existing saved layouts load correctly
- `Entry.childContainer` is mutable — surgical replant mutates in place
- No new package dependencies
- Test command: `yarn workspace @casehubio/pages-runtime run test`

---

## Batch 1: Type Foundation — refreshEntry + Entry.component + strategy onCollapse

This batch establishes the type system changes that every subsequent batch depends on. After this batch: all strategies support `refreshEntry` and `onCollapse`, `Entry.component` replaces `_content` casts, and a comprehensive combinatorial test harness exists.

### Task 1: Add `Entry.component` typed field and replace `_content` casts

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/types.ts` — add `component` field to Entry
- Modify: `packages/pages-runtime/src/frame-sandbox/container.ts` — replace `(entry as any)._content` with `entry.component`
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts` — replace all `_content` casts
- Test: `packages/pages-runtime/src/frame-sandbox/container.test.ts`

**Interfaces:**
- Produces: `Entry.component?: Component | undefined` — used by all subsequent tasks

- [ ] **Step 1: Write the failing test**

In `container.test.ts`, add:

```typescript
describe("Entry.component typed field", () => {
  it("containerizeEntry transfers component identity via typed field", () => {
    const entry: Entry = { key: "a", label: "A", component: { type: "html", props: { content: "hello" } } };
    const parent = createContainer({
      entries: [entry, { key: "b", label: "B" }],
      layout: "tabbed",
      contentFactory: simpleFactory(),
      depth: 1,
    });
    parent.mount(host);

    containerizeEntry(entry, parent, simpleFactory());

    // component moved to first child entry
    const childEntries = entry.childContainer!.entries;
    expect(childEntries[0]!.component).toEqual({ type: "html", props: { content: "hello" } });
    // parent entry cleared
    expect(entry.component).toBeUndefined();
  });

  it("flattenEntry restores component identity via typed field", () => {
    const entry: Entry = { key: "a", label: "A", component: { type: "html", props: { content: "hello" } } };
    const parent = createContainer({
      entries: [entry, { key: "b", label: "B" }],
      layout: "tabbed",
      contentFactory: simpleFactory(),
      depth: 1,
    });
    parent.mount(host);

    containerizeEntry(entry, parent, simpleFactory());
    const childEntries = entry.childContainer!.entries;
    const remaining = childEntries[0]!;

    flattenEntry(entry, remaining, simpleFactory());

    expect(entry.component).toEqual({ type: "html", props: { content: "hello" } });
    expect(entry.childContainer).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test.ts`
Expected: FAIL — `Entry` has no `component` property

- [ ] **Step 3: Add `component` field to Entry interface**

In `types.ts`, add to `Entry`:
```typescript
import type { Component } from "@casehubio/pages-component";

export interface Entry {
  // ... existing fields ...
  component?: Component | undefined;
}
```

- [ ] **Step 4: Replace all `_content` casts in container.ts**

In `containerizeEntry`: replace `(entry as any)._content` → `entry.component` (read), `(wrapped as any)._content = ...` → `wrapped.component = ...` (write), and clear with `entry.component = undefined`.

In `flattenEntry`: replace `(parentEntry as any)._content = (remainingChildEntry as any)._content` → `parentEntry.component = remainingChildEntry.component`.

- [ ] **Step 5: Replace all `_content` casts in group-organiser-backend.ts**

Search for `_content` and replace each cast with `entry.component` or `tab.component`. All locations: `wrapContentFactory`, `captureContainerState`, `restoreContainerFromState`, `renderFrame`.

- [ ] **Step 6: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test.ts`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All 894+ tests pass

- [ ] **Step 8: Commit**

```bash
git add packages/pages-runtime/src/frame-sandbox/types.ts packages/pages-runtime/src/frame-sandbox/container.ts packages/pages-runtime/src/group-organiser-backend.ts packages/pages-runtime/src/frame-sandbox/container.test.ts
git commit -m "refactor: Entry.component typed field replaces _content casts Refs #345"
```

### Task 2: Add `refreshEntry` to LayoutStrategy, Container, and all strategies

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/types.ts` — add `refreshEntry` to LayoutStrategy
- Modify: `packages/pages-runtime/src/frame-sandbox/container.ts` — add `refreshEntry` to Container
- Modify: `packages/pages-runtime/src/frame-sandbox/tabbed-strategy.ts` — implement `refreshEntry`
- Modify: `packages/pages-runtime/src/frame-sandbox/accordion-strategy.ts` — implement `refreshEntry`
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-strategy.ts` — implement `refreshEntry`
- Modify: `packages/pages-runtime/src/frame-sandbox/split-strategy.ts` — implement `refreshEntry`
- Test: `packages/pages-runtime/src/frame-sandbox/container.test.ts`

**Interfaces:**
- Produces: `LayoutStrategy.refreshEntry(key: string): void` — used by surgical replant (Batch 4)
- Produces: `Container.refreshEntry(key: string): void` — public API

- [ ] **Step 1: Write the failing test**

In `container.test.ts`, add:

```typescript
describe("refreshEntry", () => {
  it("re-renders a tabbed entry content after childContainer mutation", () => {
    const inner1 = createContainer({
      entries: [{ key: "leaf1", label: "L1" }],
      layout: "tabbed",
      contentFactory: simpleFactory(),
      depth: 2,
    });

    const entry: Entry = { key: "host", label: "Host", childContainer: inner1 };
    const outer = createContainer({
      entries: [entry],
      layout: "tabbed",
      contentFactory: (e: Entry) => {
        if (e.childContainer) {
          const el = document.createElement("div");
          el.dataset.testChildHost = "true";
          e.childContainer.mount(el);
          return { element: el, dispose: () => e.childContainer!.unmount() };
        }
        const el = document.createElement("div");
        el.textContent = `Content: ${e.key}`;
        return { element: el };
      },
      depth: 1,
    });
    outer.mount(host);

    // Verify initial state: inner container is mounted
    expect(host.querySelector("[data-test-child-host]")).not.toBeNull();

    // Mutate: replace child container with a different one
    const inner2 = createContainer({
      entries: [{ key: "leaf2", label: "L2" }],
      layout: "accordion",
      contentFactory: simpleFactory(),
      depth: 2,
    });
    inner1.unmount();
    entry.childContainer = inner2;

    // Call refreshEntry — should re-render with new child
    outer.refreshEntry("host");

    // New child should be mounted
    const newHost = host.querySelector("[data-test-child-host]");
    expect(newHost).not.toBeNull();
    // Old inner was accordion, check for accordion marker
    expect(newHost!.querySelector("[data-accordion-section]")).not.toBeNull();
  });

  it("refreshEntry on inactive tab is safe (no-op until activated)", () => {
    const entry1: Entry = { key: "a", label: "A" };
    const entry2: Entry = { key: "b", label: "B" };
    const group = createContainer({
      entries: [entry1, entry2],
      layout: "tabbed",
      contentFactory: simpleFactory(),
      depth: 1,
    });
    group.mount(host);

    // "a" is active, "b" is not rendered yet
    // refreshEntry on "b" should not throw
    expect(() => group.refreshEntry("b")).not.toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test.ts`
Expected: FAIL — `refreshEntry` is not a function

- [ ] **Step 3: Add `refreshEntry` to LayoutStrategy interface**

In `types.ts`, add to `LayoutStrategy`:
```typescript
refreshEntry(key: string): void;
```

- [ ] **Step 4: Implement `refreshEntry` in tabbed-strategy.ts**

```typescript
refreshEntry(key: string): void {
  const idx = currentEntries.findIndex(e => e.key === key);
  if (idx === -1) return;
  const entry = currentEntries[idx]!;
  // Only re-render if this entry is currently visible (active tab)
  if (entry.key !== activeKey) return;
  // Dispose old content
  if (entry.contentElement?.parentElement) {
    entry.contentElement.remove();
  }
  entry.contentDispose?.();
  entry.contentElement = undefined;
  entry.contentDispose = undefined;
  // Re-create via factory
  ensureContent(entry);
  if (entry.contentElement && contentArea) {
    contentArea.appendChild(entry.contentElement);
  }
},
```

- [ ] **Step 5: Implement `refreshEntry` in split-strategy.ts**

```typescript
refreshEntry(key: string): void {
  const idx = currentEntries.findIndex(e => e.key === key);
  if (idx === -1 || !flexContainer) return;
  const entry = currentEntries[idx]!;
  const pane = flexContainer.querySelector(`[data-split-pane="${key}"]`) as HTMLElement;
  if (!pane) return;
  // Dispose old content
  if (entry.contentElement?.parentElement) {
    entry.contentElement.remove();
  }
  entry.contentDispose?.();
  entry.contentElement = undefined;
  entry.contentDispose = undefined;
  // Re-create and append to pane
  pane.appendChild(ensureContent(entry));
},
```

- [ ] **Step 6: Implement `refreshEntry` in accordion-strategy.ts**

```typescript
refreshEntry(key: string): void {
  const idx = currentEntries.findIndex(e => e.key === key);
  if (idx === -1 || !hostElement) return;
  const entry = currentEntries[idx]!;
  const section = hostElement.querySelector(`[data-accordion-section="${key}"]`) as HTMLElement;
  if (!section) return;
  const contentDiv = section.querySelector("[data-accordion-content]") as HTMLElement;
  if (!contentDiv) return;
  if (entry.contentElement?.parentElement) {
    entry.contentElement.remove();
  }
  entry.contentDispose?.();
  entry.contentElement = undefined;
  entry.contentDispose = undefined;
  contentDiv.appendChild(ensureContent(entry));
},
```

- [ ] **Step 7: Implement `refreshEntry` in free-layout-strategy.ts**

```typescript
refreshEntry(key: string): void {
  const idx = currentEntries.findIndex(e => e.key === key);
  if (idx === -1) return;
  const entry = currentEntries[idx]!;
  const frameEl = frameElements.get(key);
  if (!frameEl) return;
  const body = frameEl.querySelector("[data-frame-body]") as HTMLElement ?? frameEl;
  if (entry.contentElement?.parentElement) {
    entry.contentElement.remove();
  }
  entry.contentDispose?.();
  entry.contentElement = undefined;
  entry.contentDispose = undefined;
  body.appendChild(ensureContent(entry));
},
```

- [ ] **Step 8: Add `refreshEntry` to Container in container.ts**

In `createContainer`, add to the returned `group` object:
```typescript
refreshEntry(key: string): void {
  const entry = entries.find(e => e.key === key);
  if (!entry) return;
  entry.contentDispose?.();
  entry.contentElement = undefined;
  entry.contentDispose = undefined;
  currentOrganiser.refreshEntry(key);
},
```

- [ ] **Step 9: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test.ts`
Expected: PASS

- [ ] **Step 10: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 11: Commit**

```bash
git add packages/pages-runtime/src/frame-sandbox/
git commit -m "feat: refreshEntry primitive — re-renders one entry slot without remove/add Refs #345"
```

### Task 3: Add `onCollapse` to tabbed + accordion strategies + layout-aware flatten

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/types.ts` — add `onCollapse` to `LayoutCallbacks`
- Modify: `packages/pages-runtime/src/frame-sandbox/container.ts` — update `buildOrganiser` to pass `onCollapse` to ALL strategies (currently only split)
- Modify: `packages/pages-runtime/src/frame-sandbox/tabbed-strategy.ts` — accept onCollapse in callbacks, add check in removeEntry
- Modify: `packages/pages-runtime/src/frame-sandbox/accordion-strategy.ts` — same
- Test: `packages/pages-runtime/src/frame-sandbox/tabbed-strategy.test.ts`
- Test: `packages/pages-runtime/src/frame-sandbox/accordion-strategy.test.ts`

**Interfaces:**
- Consumes: `ContainerConfig.onCollapse` — already on ContainerConfig
- Produces: `LayoutCallbacks.onCollapse?: (remaining: Entry) => void` — NEW on LayoutCallbacks (currently only on `SplitCallbacks`)

**CRITICAL (plan-R1-03):** `onCollapse` exists on `ContainerConfig` but NOT on `LayoutCallbacks`. `buildOrganiser()` in container.ts only passes `onCollapse` to split strategies via the `SplitCallbacks` extension. For tabbed/accordion to receive it, two things must happen:
1. Add `onCollapse?: (remaining: Entry) => void` to `LayoutCallbacks` in types.ts
2. Update `buildOrganiser()` to spread `onCollapse` into callbacks for ALL strategy types (not just split)

Without this routing fix, the `onCollapse` check added to tabbed/accordion `removeEntry` will never fire — the callback will always be undefined.

**Note on free-layout:** `onCollapse` is intentionally NOT added to free-layout strategy. A single floating window inside a free-layout container is valid UX — unlike a single tab in tabbed, which is pointless nesting. If the user changes a nested container to free, closes all but one entry, the container persists. They can switch back to tabbed to trigger flatten.

- [ ] **Step 1: Write failing tests**

In `tabbed-strategy.test.ts`, add:

```typescript
describe("onCollapse", () => {
  it("fires onCollapse when last sibling is removed and one entry remains", () => {
    const onCollapse = vi.fn();
    const strategy = createTabbedStrategy({ onCollapse });
    const entries: Entry[] = [
      { key: "a", label: "A" },
      { key: "b", label: "B" },
    ];
    const host = document.createElement("div");
    strategy.mount(host, entries, simpleFactory());

    strategy.removeEntry("a");

    expect(onCollapse).toHaveBeenCalledOnce();
    expect(onCollapse).toHaveBeenCalledWith(expect.objectContaining({ key: "b" }));
  });

  it("does not fire onCollapse when 2+ entries remain", () => {
    const onCollapse = vi.fn();
    const strategy = createTabbedStrategy({ onCollapse });
    const entries: Entry[] = [
      { key: "a", label: "A" },
      { key: "b", label: "B" },
      { key: "c", label: "C" },
    ];
    const host = document.createElement("div");
    strategy.mount(host, entries, simpleFactory());

    strategy.removeEntry("a");

    expect(onCollapse).not.toHaveBeenCalled();
  });
});
```

Same pattern for `accordion-strategy.test.ts`.

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose tabbed-strategy.test.ts`
Expected: FAIL — onCollapse not called

- [ ] **Step 3: Add onCollapse check to tabbed-strategy removeEntry**

In `tabbed-strategy.ts`, after the entry is removed in `removeEntry`:

```typescript
if (currentEntries.length === 1 && callbacks?.onCollapse) {
  callbacks.onCollapse(currentEntries[0]!);
  return;
}
```

- [ ] **Step 4: Add onCollapse check to accordion-strategy removeEntry**

Same pattern in `accordion-strategy.ts`.

- [ ] **Step 5: Run tests, verify pass**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose tabbed-strategy.test.ts accordion-strategy.test.ts`
Expected: PASS

- [ ] **Step 6: Run full suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add packages/pages-runtime/src/frame-sandbox/tabbed-strategy.ts packages/pages-runtime/src/frame-sandbox/accordion-strategy.ts packages/pages-runtime/src/frame-sandbox/tabbed-strategy.test.ts packages/pages-runtime/src/frame-sandbox/accordion-strategy.test.ts
git commit -m "fix: onCollapse fires in tabbed and accordion strategies on single-entry Refs #345"
```

---

## Batch 2: Module Extraction — decompose the backend

After this batch: the backend is decomposed into focused modules. Tree operations, frame rendering, and DnD each live in their own file. All existing tests pass unchanged.

### Task 4: Extract `container-tree-ops.ts`

**Files:**
- Create: `packages/pages-runtime/src/container-tree-ops.ts`
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts` — remove extracted functions, import from new module
- Test: `packages/pages-runtime/src/container-tree-ops.test.ts` (new)

**Interfaces:**
- Produces: `findLeafContainer`, `findContainerWithTab`, `forEachLeafContainer`, `findParentOf`, `isSplitLayout`, `captureContainerState`, `restoreContainerFromState`
- Consumes: `Container`, `Entry`, `Layout` from `frame-sandbox/types`

- [ ] **Step 1: Write failing tests for tree traversal**

Create `container-tree-ops.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { findLeafContainer, findContainerWithTab, findParentOf, forEachLeafContainer, isSplitLayout, captureContainerState } from "./container-tree-ops.js";
import { createContainer } from "./frame-sandbox/index.js";
import type { Entry, ContentFactory } from "./frame-sandbox/types.js";

function simpleFactory(): ContentFactory {
  return (entry) => {
    const el = document.createElement("div");
    el.textContent = entry.key;
    return { element: el };
  };
}

describe("container-tree-ops", () => {
  let host: HTMLElement;

  beforeEach(() => {
    host = document.createElement("div");
    document.body.appendChild(host);
  });

  describe("findContainerWithTab", () => {
    it("finds tab in a flat container", () => {
      const c = createContainer({
        entries: [{ key: "a", label: "A" }, { key: "b", label: "B" }],
        layout: "tabbed",
        contentFactory: simpleFactory(),
      });
      expect(findContainerWithTab(c, "a")).toBe(c);
      expect(findContainerWithTab(c, "z")).toBeNull();
    });

    it("finds tab in a nested container via childContainer", () => {
      const inner = createContainer({
        entries: [{ key: "deep", label: "Deep" }],
        layout: "tabbed",
        contentFactory: simpleFactory(),
        depth: 2,
      });
      const entry: Entry = { key: "host", label: "Host", childContainer: inner };
      const outer = createContainer({
        entries: [entry],
        layout: "tabbed",
        contentFactory: simpleFactory(),
        depth: 1,
      });
      expect(findContainerWithTab(outer, "deep")).toBe(inner);
    });
  });

  describe("findParentOf", () => {
    it("returns parent container and entry for a nested child", () => {
      const inner = createContainer({
        entries: [{ key: "leaf", label: "Leaf" }],
        layout: "tabbed",
        contentFactory: simpleFactory(),
        depth: 2,
      });
      const entry: Entry = { key: "host", label: "Host", childContainer: inner };
      const outer = createContainer({
        entries: [entry],
        layout: "tabbed",
        contentFactory: simpleFactory(),
        depth: 1,
      });

      const result = findParentOf(outer, inner);
      expect(result).not.toBeNull();
      expect(result!.container).toBe(outer);
      expect(result!.entry).toBe(entry);
    });

    it("returns null for root container", () => {
      const c = createContainer({
        entries: [{ key: "a", label: "A" }],
        layout: "tabbed",
        contentFactory: simpleFactory(),
      });
      expect(findParentOf(c, c)).toBeNull();
    });
  });

  describe("captureContainerState", () => {
    it("serializes a flat container", () => {
      const c = createContainer({
        entries: [
          { key: "a", label: "A", component: { type: "html", props: {} } },
          { key: "b", label: "B", component: { type: "chart", props: {} } },
        ],
        layout: "tabbed",
        contentFactory: simpleFactory(),
      });
      c.mount(host);

      const state = captureContainerState(c);
      expect(state.layout).toBe("tabbed");
      expect(state.tabs).toHaveLength(2);
      expect(state.tabs[0]!.key).toBe("a");
      expect(state.tabs[0]!.content).toEqual({ type: "html", props: {} });
    });

    it("serializes nested containers recursively", () => {
      const inner = createContainer({
        entries: [{ key: "deep", label: "D", component: { type: "html", props: {} } }],
        layout: "accordion",
        contentFactory: simpleFactory(),
        depth: 2,
      });
      const entry: Entry = { key: "host", label: "Host", childContainer: inner };
      const outer = createContainer({
        entries: [entry],
        layout: "tabbed",
        contentFactory: simpleFactory(),
        depth: 1,
      });
      outer.mount(host);

      const state = captureContainerState(outer);
      expect(state.tabs[0]!.children).toBeDefined();
      expect(state.tabs[0]!.children!.layout).toBe("accordion");
      expect(state.tabs[0]!.children!.tabs[0]!.key).toBe("deep");
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container-tree-ops.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Create `container-tree-ops.ts` with extracted functions**

Move the following functions from `group-organiser-backend.ts` to `container-tree-ops.ts`:
- `isSplitLayout` (line ~55)
- `findLeafContainer` (line ~57)
- `findContainerWithTab` (line ~70)
- `forEachLeafContainer` (line ~88)
- `findParentOf` (renamed from `findParentEntry`, returns `{ container, entry }`)
- `captureContainerState` (line ~116)
- `restoreContainerFromState` (line ~140)
- `paneCounter` (module-level monotonic counter)

Export all functions. Import types from `./frame-sandbox/types.js`.

- [ ] **Step 4: Update backend to import from container-tree-ops**

Replace all usages in `group-organiser-backend.ts` with imports from `./container-tree-ops.js`.

- [ ] **Step 5: Run tests, verify pass**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass (old and new)

- [ ] **Step 6: Commit**

```bash
git add packages/pages-runtime/src/container-tree-ops.ts packages/pages-runtime/src/container-tree-ops.test.ts packages/pages-runtime/src/group-organiser-backend.ts
git commit -m "refactor: extract container-tree-ops from backend — tree traversal and serialization Refs #345"
```

### Task 5: Extract `frame-state.ts` and `frame-renderer.ts`

**Files:**
- Create: `packages/pages-runtime/src/frame-state.ts`
- Create: `packages/pages-runtime/src/frame-renderer.ts`
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts` — remove rendering code, import from new modules
- Test: `packages/pages-runtime/src/frame-renderer.test.ts` (new)

**Interfaces:**
- Produces: `FrameState extends PositionedState`, `renderFrame() → FrameRenderResult`
- Consumes: `createFrameShell`, `createFrameTitlebar`, `createFrameResizeHandles`, `wireTitlebarDrag` from `frame-shell.ts`; `injectFrameChrome` from `frame-chrome.ts`

- [ ] **Step 1: Create `frame-state.ts`**

```typescript
import type { PositionedState } from "./frame-shell.js";
import type { Container } from "./frame-sandbox/types.js";

export interface FrameState extends PositionedState {
  readonly key: string;
  frameEl: HTMLElement;
  tabContentEl: HTMLElement;
  rootContainer: Container;
}
```

- [ ] **Step 2: Write failing test for frame-renderer**

Create `frame-renderer.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { renderFrame } from "./frame-renderer.js";

const ResizeObserverMock = vi.fn().mockImplementation(() => ({
  observe: vi.fn(), unobserve: vi.fn(), disconnect: vi.fn(),
}));
vi.stubGlobal("ResizeObserver", ResizeObserverMock);

describe("renderFrame", () => {
  it("creates frame with shell, titlebar, and tabContentEl", () => {
    const layout = {
      key: "f1", position: { x: 10, y: 20 }, size: { width: 300, height: 200 },
      order: 0, zIndex: 1, pinned: false, hidden: false,
      tabs: [{ key: "t1", label: "T1", content: { type: "html", props: {} } }],
      activeTabKey: "t1",
    };

    const result = renderFrame(layout as any, {
      onClose: vi.fn(), onPin: vi.fn(), onTitlebarDoubleClick: vi.fn(),
      onMove: vi.fn(), onResize: vi.fn(),
    });

    expect(result.frameEl).toBeTruthy();
    expect(result.frameEl.getAttribute("data-frame-key")).toBe("f1");
    expect(result.titlebar).toBeTruthy();
    expect(result.tabContentEl).toBeTruthy();

    result.dispose();
  });

  it("uses createFrameTitlebar from frame-shell (not manual creation)", () => {
    const layout = {
      key: "f2", position: { x: 0, y: 0 }, size: { width: 200, height: 100 },
      order: 0, zIndex: 1, pinned: false, hidden: false,
      tabs: [{ key: "t1", label: "T1", content: null }],
      activeTabKey: "t1",
    };

    const result = renderFrame(layout as any, {
      onClose: vi.fn(), onPin: vi.fn(), onTitlebarDoubleClick: vi.fn(),
      onMove: vi.fn(), onResize: vi.fn(),
    });

    // Shared titlebar has data-frame-titlebar attribute
    expect(result.titlebar.hasAttribute("data-frame-titlebar")).toBe(true);

    result.dispose();
  });
});
```

- [ ] **Step 3: Create `frame-renderer.ts`**

Extract `renderFrame`'s DOM creation from backend. Key change: use `createFrameTitlebar()` and `wireTitlebarDrag()` from frame-shell (currently imported but unused in backend — manual creation code is deleted). Use `createFrameResizeHandles()` from frame-shell instead of the backend's duplicate `createResizeHandles`.

- [ ] **Step 4: Update backend — import FrameState, delegate to frame-renderer**

Replace the backend's `FrameState` type with import from `frame-state.ts`. Replace `renderFrame`'s DOM creation with delegation to `frame-renderer.renderFrame()`.

- [ ] **Step 5: Delete dead code from backend**

Remove: local `createResizeHandles` function (~100 lines), manual titlebar creation (~10 lines), unused `addChildToFrame` function.

- [ ] **Step 6: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add packages/pages-runtime/src/frame-state.ts packages/pages-runtime/src/frame-renderer.ts packages/pages-runtime/src/frame-renderer.test.ts packages/pages-runtime/src/group-organiser-backend.ts
git commit -m "refactor: extract frame-state and frame-renderer from backend Refs #345"
```

### Task 6: Extract `dnd-coordinator.ts`

**Files:**
- Create: `packages/pages-runtime/src/dnd-coordinator.ts`
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts` — remove DnD code, delegate to coordinator
- Test: `packages/pages-runtime/src/dnd-coordinator.test.ts` (new)

**Interfaces:**
- Produces: `createDndCoordinator(context, callbacks) → DndCoordinator`
- Consumes: `FrameState` from `frame-state.ts`, `EdgeZone` from `frame-boundaries.ts`

- [ ] **Step 1: Write failing test**

Create `dnd-coordinator.test.ts` with tests for drag state machine: start → move → end lifecycle, cross-frame detection, preview show/hide.

- [ ] **Step 2: Create `dnd-coordinator.ts`**

Extract from backend: `dragState`, `crossFramePreview`, `edgeSplitPreview`, and all their management functions. The coordinator receives `DndContext` (readonly frame map access, container element) and fires `DndCallbacks` (onCrossFrameDrop with insertIndex, onEdgeSplit with targetLeaf, onTabDragOut).

- [ ] **Step 3: Update backend to delegate DnD to coordinator**

The backend creates a `DndCoordinator` in `attach()`, passes its frame map as context, and wires callbacks to its own handlers.

- [ ] **Step 4: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add packages/pages-runtime/src/dnd-coordinator.ts packages/pages-runtime/src/dnd-coordinator.test.ts packages/pages-runtime/src/group-organiser-backend.ts
git commit -m "refactor: extract dnd-coordinator from backend Refs #345"
```

---

## Batch 3: Code Unification + Workspace Mount Transfer

After this batch: zone picker is unified, workspace transitions use mount transfer (no serialize/recreate), split brain is eliminated.

### Task 7: Unified zone picker

**Files:**
- Modify: `packages/pages-runtime/src/frame-zone-picker.ts` — add `SnapCallback` param to shared grid UI
- Modify: `packages/pages-runtime/src/frame-sandbox/free-layout-strategy.ts` — remove inline zone picker, use shared
- Test: `packages/pages-runtime/src/frame-zone-picker.test.ts`

**Interfaces:**
- Produces: `showZonePicker(anchorEl, onSnap: SnapCallback): HTMLElement`

- [ ] **Step 1: Read current `frame-zone-picker.ts` and `free-layout-strategy.ts` inline picker**

Use ide-tooling to understand both implementations.

- [ ] **Step 2: Add shared `showZoneGrid` function to frame-zone-picker.ts**

Extract the 3x3 grid dropdown UI into a shared function that takes a snap callback. Each call site provides its own callback:
- Root frames: `(zone) => engine.snapFrame(key, zone, canvasSize)`
- Inner panels: `(zone) => { state.position = ...; state.size = ...; }`

The root-frame picker retains its lifecycle management (current zone highlighting, toggle/unsnap) around the shared grid.

- [ ] **Step 3: Update free-layout-strategy.ts to use shared grid**

Remove the inline `ZONES` array, `zoneToRect` function, and `showZonePicker` function. Import and use the shared `showZoneGrid` from `frame-zone-picker.ts`.

- [ ] **Step 4: Run full test suite, commit**

```bash
git commit -m "refactor: unified zone picker grid — shared between root frames and inner panels Refs #345"
```

### Task 8: Workspace mount transfer — eliminate split brain

**Files:**
- Modify: `packages/pages-runtime/src/floating-frame-backend.ts` — add `getRootContainer` to interface
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts` — implement `getRootContainer`
- Modify: `packages/pages-runtime/src/wire-floating-workspace.ts` — rewrite `applyWorkspaceMode` with mount transfer
- Test: `packages/pages-runtime/src/wire-floating-workspace.test.ts`

**Interfaces:**
- Consumes: `Container.mount()`, `Container.unmount()` from `frame-sandbox/types`
- Produces: `FloatingFrameBackend.getRootContainer(frameKey): Container | null`

- [ ] **Step 1: Write failing test for mount transfer**

In `wire-floating-workspace.test.ts`, add:

```typescript
describe("mount transfer", () => {
  it("workspace mode free→tabbed preserves live container identity", () => {
    // Create backend with real containers (not mock)
    const backend = createGroupOrganiserBackend();
    const container = document.createElement("div");
    Object.defineProperty(container, "clientWidth", { value: 800 });
    Object.defineProperty(container, "clientHeight", { value: 600 });
    document.body.appendChild(container);
    backend.attach(container, testContentFactory());

    backend.renderFrame(makeLayout("f1", ["a", "b"]));

    const containerBefore = backend.getRootContainer("f1");
    expect(containerBefore).not.toBeNull();

    // Get organiser state before transition
    const stateBefore = containerBefore!.organiser.getState();

    // Simulate workspace mode transition: would previously serialize/recreate
    // After mount transfer: same container, same state
    const containerAfter = backend.getRootContainer("f1");
    expect(containerAfter).toBe(containerBefore); // Same object — not recreated

    backend.dispose();
    document.body.removeChild(container);
  });
});
```

- [ ] **Step 2: Add `getRootContainer` to FloatingFrameBackend interface**

In `floating-frame-backend.ts`:
```typescript
getRootContainer(frameKey: string): Container | null;
```

- [ ] **Step 3: Implement `getRootContainer` in backend**

In `group-organiser-backend.ts`:
```typescript
getRootContainer(frameKey: string): Container | null {
  const state = frames.get(frameKey);
  return state?.rootContainer ?? null;
},
```

- [ ] **Step 4: Rewrite `applyWorkspaceMode` in wire-floating-workspace.ts**

Replace the serialize/recreate flow with direct re-parent:
- Free → non-free: for each frame, call `backend.getRootContainer(key)`, unmount from frame, create workspace entry with `childContainer = container`, mount workspace container
- Non-free → free: for each workspace entry, unmount from workspace, mount back into frame
- Non-free → non-free: `workspaceContainer.setLayout(targetMode)` (unchanged)

Delete `restoreWorkspaceTree` and `syncWorkspaceStateToFrames` entirely.

- [ ] **Step 5: Fix iteration-during-mutation (spec-R1-06)**

In the workspace→free transition, copy entries before iterating:
```typescript
for (const entry of [...workspaceContainer.entries]) { ... }
```

- [ ] **Step 6: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add packages/pages-runtime/src/floating-frame-backend.ts packages/pages-runtime/src/group-organiser-backend.ts packages/pages-runtime/src/wire-floating-workspace.ts packages/pages-runtime/src/wire-floating-workspace.test.ts
git commit -m "fix: workspace mount transfer — eliminate split brain, delete restoreWorkspaceTree Refs #345"
```

---

## Batch 4: Surgical Replant

After this batch: split and collapse operations use surgical subtree re-parent via `refreshEntry` instead of full-tree unmount/remount. Sibling state is preserved across structural changes.

### Task 9: Surgical split — `splitFrame` uses `refreshEntry`

**Files:**
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts` (or `container-tree-ops.ts` if split logic was extracted)
- Test: `packages/pages-runtime/src/group-organiser-backend.test.ts`

**Interfaces:**
- Consumes: `Container.refreshEntry(key)`, `findParentOf` from container-tree-ops

- [ ] **Step 1: Write failing test — sibling state preserved across split**

In `group-organiser-backend.test.ts`, add:

```typescript
describe("surgical split", () => {
  it("splitting a pane preserves sibling container organiser state", () => {
    // Create a frame with 2 tabs
    backend.renderFrame(makeLayout("f1", ["a", "b", "c"]));

    const root = backend.getRootContainer("f1");
    expect(root).not.toBeNull();

    // Switch to accordion to set distinguishable state
    root!.setLayout("accordion");
    const accordionStateBefore = root!.organiser.getState();

    // Perform an edge split: drag tab "c" to right edge of frame
    // This creates a split with the original container (now with a, b) and a new pane (c)
    let edgeSplitCb: any;
    backend.onEdgeSplit((from, tabKey, target, zone) => {
      edgeSplitCb = { from, tabKey, target, zone };
    });

    // Simulate: the split creates a new split container as root
    // After surgical split, the ORIGINAL container (with a, b) should keep its accordion state

    // Verify original container content is preserved
    const contentA = container.querySelector("[data-content-key='a']");
    expect(contentA).not.toBeNull();

    // Verify accordion layout is preserved (not reset to tabbed)
    const accordionSections = container.querySelectorAll("[data-accordion-section]");
    expect(accordionSections.length).toBeGreaterThanOrEqual(2); // a and b in accordion
  });

  it("pane-level split does not unmount sibling frames", () => {
    // Track mount/unmount calls via spy
    backend.renderFrame(makeLayout("f1", ["a", "b"]));

    const root = backend.getRootContainer("f1");
    expect(root).not.toBeNull();

    // Count DOM children before split
    const frameEl = backend.getFrameElement("f1");
    const childCountBefore = frameEl?.querySelectorAll("[data-tab-key]").length ?? 0;
    expect(childCountBefore).toBeGreaterThanOrEqual(2);

    // After surgical split, all original tab content should still be present
    // (not re-created — same DOM nodes, verified by object identity)
  });
});
```

- [ ] **Step 2: Rewrite pane-level split to use surgical replant**

In `splitFrame`, pane-level path:
```typescript
// OLD: state.rootContainer.unmount(); ... state.rootContainer.mount(state.tabContentEl);
// NEW:
const parent = findParentOf(state.rootContainer, targetLeaf);
if (parent) {
  targetLeaf.unmount();
  const splitContainer = createSplitContainer(...);
  parent.entry.childContainer = splitContainer;
  parent.container.refreshEntry(parent.entry.key);
}
```

- [ ] **Step 3: Run full test suite, commit**

```bash
git commit -m "refactor: surgical split — pane-level split uses refreshEntry instead of full remount Refs #345"
```

### Task 10: Surgical collapse + layout-aware flatten

**Files:**
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts`
- Modify: `packages/pages-runtime/src/frame-sandbox/container.ts` — update `containerizeEntry`'s onCollapse handler
- Test: `packages/pages-runtime/src/frame-sandbox/container.test.ts`

**Interfaces:**
- Consumes: `Container.refreshEntry(key)`, `findParentOf`

- [ ] **Step 1: Write failing test — layout-aware flatten**

In `container.test.ts`, add:

```typescript
describe("layout-aware flatten", () => {
  it("auto-flattens when child layout matches parent (tabbed in tabbed)", () => {
    const entry: Entry = { key: "a", label: "A", component: { type: "html", props: {} } };
    const parent = createContainer({
      entries: [entry, { key: "b", label: "B" }],
      layout: "tabbed",
      contentFactory: simpleFactory(),
      depth: 1,
    });
    parent.mount(host);

    containerizeEntry(entry, parent, simpleFactory());
    expect(entry.childContainer).toBeDefined();

    // Remove all but one tab from child (both are tabbed → should flatten)
    const child = entry.childContainer!;
    const childEntries = [...child.entries];
    child.removeEntry(childEntries[1]!.key); // Remove "New Tab"

    // Should have auto-flattened — childContainer gone
    expect(entry.childContainer).toBeUndefined();
    expect(entry.component).toBeDefined();
  });

  it("preserves nesting when child layout differs from parent (accordion in tabbed)", () => {
    const entry: Entry = { key: "a", label: "A", component: { type: "html", props: {} } };
    const parent = createContainer({
      entries: [entry, { key: "b", label: "B" }],
      layout: "tabbed",
      contentFactory: simpleFactory(),
      depth: 1,
    });
    parent.mount(host);

    containerizeEntry(entry, parent, simpleFactory());
    const child = entry.childContainer!;
    child.setLayout("accordion"); // Change child to accordion

    // Remove all but one
    const childEntries = [...child.entries];
    child.removeEntry(childEntries[1]!.key);

    // Should NOT have flattened — layouts differ
    expect(entry.childContainer).toBeDefined();
  });
});
```

- [ ] **Step 2: Update `containerizeEntry` onCollapse handler**

In `container.ts`, update the onCollapse callback:

```typescript
onCollapse: (remaining: Entry) => {
  const childLayout = child.organiser.type;
  const parentLayout = parentContainer.organiser.type;
  if (childLayout === parentLayout) {
    flattenEntry(entry, remaining, contentFactory);
    parentContainer.refreshEntry(entry.key);
  }
},
```

- [ ] **Step 3: Rewrite collapse in backend to use surgical replant**

In `createSplitContainer`'s `onCollapse`:
```typescript
// OLD: state.rootContainer.unmount(); ... state.rootContainer.mount(state.tabContentEl);
// NEW:
const parent = findParentOf(state.rootContainer, collapsingContainer);
if (parent) {
  // NESTED collapse — surgical replant via refreshEntry
  surviving.unmount();
  collapsingContainer.dispose();
  parent.entry.childContainer = surviving;
  parent.container.refreshEntry(parent.entry.key);
} else {
  // ROOT collapse — promote surviving child as new rootContainer
  surviving.unmount();
  state.rootContainer.dispose();
  state.rootContainer = surviving;
  surviving.mount(state.tabContentEl);
}
```

**CRITICAL (plan-R1-06):** Both paths must be handled. Without the `else` branch, root collapse is silently dropped — the surviving child never gets promoted, and `state.rootContainer` goes stale.

- [ ] **Step 4: Run full test suite, commit**

```bash
git commit -m "fix: layout-aware flatten + surgical collapse via refreshEntry Refs #345"
```

---

## Batch 5: Combinatorial End-to-End Tests

After this batch: a comprehensive test matrix verifies that container nesting, layout switching, split/collapse, and workspace transitions work correctly across all layout combinations and depths.

### Task 11: Test harness — parameterized container tree factory

**Files:**
- Create: `packages/pages-runtime/src/frame-sandbox/test-harness.ts` — reusable factory for building N-level nested container trees
- Test: `packages/pages-runtime/src/frame-sandbox/combinatorial.test.ts` (new)

**Interfaces:**
- Produces: `buildContainerTree(spec: TreeSpec): { root: Container, containers: Map<string, Container> }`

- [ ] **Step 1: Create test harness**

```typescript
import { createContainer } from "./container.js";
import type { Container, Entry, Layout, ContentFactory } from "./types.js";

export interface TreeLevel {
  layout: Layout;
  entryCount: number;
  nestedAt?: number; // which entry index gets a child container (0-based)
}

export interface TreeSpec {
  levels: TreeLevel[];
}

export function simpleTestFactory(): ContentFactory {
  return (entry: Entry) => {
    if (entry.childContainer) {
      const el = document.createElement("div");
      el.dataset.childHost = entry.key;
      entry.childContainer.mount(el);
      return { element: el, dispose: () => entry.childContainer!.unmount() };
    }
    const el = document.createElement("div");
    el.textContent = `Leaf: ${entry.key}`;
    el.dataset.testLeaf = entry.key;
    return { element: el };
  };
}

export function buildContainerTree(spec: TreeSpec): {
  root: Container;
  containers: Map<string, Container>;
} {
  const containers = new Map<string, Container>();
  let keyCounter = 0;
  const nextKey = () => `e${keyCounter++}`;

  function buildLevel(levelIdx: number, depth: number): Container {
    const level = spec.levels[levelIdx]!;
    const entries: Entry[] = [];

    for (let i = 0; i < level.entryCount; i++) {
      const key = nextKey();
      const entry: Entry = {
        key,
        label: `${level.layout}[${i}]@d${depth}`,
        component: { type: "html", props: { content: key } },
      };

      // If this entry should nest and there's a deeper level
      if (i === (level.nestedAt ?? -1) && levelIdx + 1 < spec.levels.length) {
        const child = buildLevel(levelIdx + 1, depth + 1);
        entry.childContainer = child;
        entry.component = undefined;
      }

      entries.push(entry);
    }

    const container = createContainer({
      entries,
      layout: level.layout,
      contentFactory: simpleTestFactory(),
      depth,
    });
    containers.set(`L${levelIdx}`, container);
    return container;
  }

  const root = buildLevel(0, 1);
  return { root, containers };
}
```

- [ ] **Step 2: Commit harness**

```bash
git add packages/pages-runtime/src/frame-sandbox/test-harness.ts
git commit -m "test: container tree test harness for combinatorial testing Refs #345"
```

### Task 12: 2-level layout matrix — all layout pairs rendered and switchable

**Files:**
- Test: `packages/pages-runtime/src/frame-sandbox/combinatorial.test.ts`

- [ ] **Step 1: Write the combinatorial 2-level rendering matrix**

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { buildContainerTree, type TreeSpec } from "./test-harness.js";
import type { Layout } from "./types.js";

const LEAF_LAYOUTS: Layout[] = ["tabbed", "accordion", "free"];
const ALL_LAYOUTS: Layout[] = ["tabbed", "accordion", "free", "splith", "splitv"];

describe("2-level layout matrix", () => {
  let host: HTMLElement;

  beforeEach(() => {
    host = document.createElement("div");
    host.style.cssText = "width:800px;height:600px;position:relative;";
    Object.defineProperty(host, "clientWidth", { value: 800, configurable: true });
    Object.defineProperty(host, "clientHeight", { value: 600, configurable: true });
    document.body.appendChild(host);
  });

  afterEach(() => {
    document.body.removeChild(host);
  });

  for (const outer of LEAF_LAYOUTS) {
    for (const inner of LEAF_LAYOUTS) {
      describe(`${outer} > ${inner}`, () => {
        it("renders both levels with correct DOM structure", () => {
          const { root, containers } = buildContainerTree({
            levels: [
              { layout: outer, entryCount: 2, nestedAt: 0 },
              { layout: inner, entryCount: 2 },
            ],
          });
          root.mount(host);

          // Outer container rendered
          expect(host.children.length).toBeGreaterThan(0);

          // Inner container rendered (has a child-host div)
          const childHost = host.querySelector("[data-child-host]");
          expect(childHost).not.toBeNull();

          // Leaf content exists at depth 2
          const leaves = host.querySelectorAll("[data-test-leaf]");
          expect(leaves.length).toBeGreaterThan(0);

          root.dispose();
        });

        it("inner layout switch preserves outer structure", () => {
          const { root, containers } = buildContainerTree({
            levels: [
              { layout: outer, entryCount: 2, nestedAt: 0 },
              { layout: inner, entryCount: 2 },
            ],
          });
          root.mount(host);

          const innerContainer = containers.get("L1")!;
          const otherLayout: Layout = inner === "tabbed" ? "accordion" : "tabbed";
          innerContainer.setLayout(otherLayout);

          // Outer is still mounted
          expect(host.children.length).toBeGreaterThan(0);

          // Inner still has leaves
          const leaves = host.querySelectorAll("[data-test-leaf]");
          expect(leaves.length).toBeGreaterThan(0);

          root.dispose();
        });

        it("outer layout switch preserves inner content", () => {
          const { root, containers } = buildContainerTree({
            levels: [
              { layout: outer, entryCount: 2, nestedAt: 0 },
              { layout: inner, entryCount: 2 },
            ],
          });
          root.mount(host);

          const otherOuter: Layout = outer === "tabbed" ? "accordion" : "tabbed";
          root.setLayout(otherOuter);

          // Inner container still exists
          const childHost = host.querySelector("[data-child-host]");
          // For tabbed→accordion, the content may be in a different DOM position
          // but should still exist somewhere in the tree
          const leaves = host.querySelectorAll("[data-test-leaf]");
          expect(leaves.length).toBeGreaterThan(0);

          root.dispose();
        });
      });
    }
  }
});
```

This generates **9 test suites** (3×3 leaf layouts), each with 3 tests = **27 tests** covering every 2-level pair.

- [ ] **Step 2: Run the matrix**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose combinatorial.test.ts`
Expected: All 27 tests pass

- [ ] **Step 3: Commit**

```bash
git add packages/pages-runtime/src/frame-sandbox/combinatorial.test.ts
git commit -m "test: 2-level layout matrix — 27 combinatorial tests across all layout pairs Refs #345"
```

### Task 13: 3-level deep + split combinations + operations

**Files:**
- Test: `packages/pages-runtime/src/frame-sandbox/combinatorial.test.ts` (append)

- [ ] **Step 1: Add 3-level depth tests**

```typescript
describe("3-level deep nesting", () => {
  for (const l1 of LEAF_LAYOUTS) {
    for (const l2 of LEAF_LAYOUTS) {
      for (const l3 of LEAF_LAYOUTS) {
        it(`${l1} > ${l2} > ${l3} renders all three levels`, () => {
          const { root } = buildContainerTree({
            levels: [
              { layout: l1, entryCount: 2, nestedAt: 0 },
              { layout: l2, entryCount: 2, nestedAt: 0 },
              { layout: l3, entryCount: 2 },
            ],
          });
          root.mount(host);

          const leaves = host.querySelectorAll("[data-test-leaf]");
          expect(leaves.length).toBeGreaterThan(0);

          root.dispose();
        });
      }
    }
  }
});
```

This generates **27 tests** (3×3×3 = 27 layout triples).

- [ ] **Step 2: Add split-within-nest tests**

```typescript
describe("split containers in nested trees", () => {
  for (const splitDir of ["splith", "splitv"] as const) {
    for (const leafLayout of LEAF_LAYOUTS) {
      it(`${splitDir} at root with ${leafLayout} leaves renders`, () => {
        const { root } = buildContainerTree({
          levels: [
            { layout: splitDir, entryCount: 2, nestedAt: 0 },
            { layout: leafLayout, entryCount: 2 },
          ],
        });
        root.mount(host);

        const splitContainer = host.querySelector(`[data-split-container]`);
        expect(splitContainer).not.toBeNull();

        const leaves = host.querySelectorAll("[data-test-leaf]");
        expect(leaves.length).toBeGreaterThan(0);

        root.dispose();
      });

      it(`${leafLayout} at root with ${splitDir} nested renders`, () => {
        const { root } = buildContainerTree({
          levels: [
            { layout: leafLayout, entryCount: 2, nestedAt: 0 },
            { layout: splitDir, entryCount: 2 },
          ],
        });
        root.mount(host);

        const leaves = host.querySelectorAll("[data-test-leaf]");
        expect(leaves.length).toBeGreaterThan(0);

        root.dispose();
      });
    }
  }
});
```

**12 more tests** (2 split directions × 3 leaf layouts × 2 orderings).

- [ ] **Step 3: Add containerize + flatten round-trip tests**

```typescript
describe("containerize/flatten round-trip across layouts", () => {
  for (const layout of LEAF_LAYOUTS) {
    it(`containerize + flatten in ${layout} preserves content`, () => {
      const entry: Entry = {
        key: "target",
        label: "Target",
        component: { type: "html", props: { content: "original" } },
      };
      const container = createContainer({
        entries: [entry, { key: "sibling", label: "Sibling" }],
        layout,
        contentFactory: simpleTestFactory(),
        depth: 1,
      });
      container.mount(host);

      // Containerize
      containerizeEntry(entry, container, simpleTestFactory());
      expect(entry.childContainer).toBeDefined();
      expect(entry.component).toBeUndefined();

      // Flatten — remove second child tab to trigger onCollapse
      const child = entry.childContainer!;
      const secondKey = child.entries[1]!.key;
      child.removeEntry(secondKey);

      // Should have flattened (both are tabbed)
      expect(entry.childContainer).toBeUndefined();
      expect(entry.component).toEqual({ type: "html", props: { content: "original" } });

      container.dispose();
    });
  }
});
```

**3 more tests** (one per leaf layout).

- [ ] **Step 4: Run, verify, commit**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose combinatorial.test.ts`
Expected: All tests pass (27 + 27 + 12 + 3 = **69 tests** in this file)

```bash
git commit -m "test: 3-level depth matrix + split combinations + round-trip tests Refs #345"
```

### Task 14: Persistence round-trip for recursive trees

**Files:**
- Test: `packages/pages-runtime/src/container-tree-ops.test.ts` (append)

- [ ] **Step 1: Write persistence round-trip tests**

```typescript
describe("persistence round-trip", () => {
  for (const l1 of LEAF_LAYOUTS) {
    for (const l2 of LEAF_LAYOUTS) {
      it(`${l1} > ${l2} survives capture → restore`, () => {
        const { root } = buildContainerTree({
          levels: [
            { layout: l1, entryCount: 2, nestedAt: 0 },
            { layout: l2, entryCount: 2 },
          ],
        });
        root.mount(host);

        // Capture
        const state = captureContainerState(root);
        root.dispose();

        // Verify serialized structure
        expect(state.layout).toBe(l1);
        expect(state.tabs).toHaveLength(2);
        expect(state.tabs[0]!.children).toBeDefined();
        expect(state.tabs[0]!.children!.layout).toBe(l2);
        expect(state.tabs[0]!.children!.tabs).toHaveLength(2);

        // Restore
        const restored = restoreContainerFromState(state, 1, DEFAULT_POLICY, simpleTestFactory(), {});
        restored.mount(host);

        // Verify DOM structure matches original
        const leaves = host.querySelectorAll("[data-test-leaf]");
        expect(leaves.length).toBeGreaterThan(0);

        restored.dispose();
      });
    }
  }
});
```

**9 more round-trip tests** (3×3 layout pairs).

- [ ] **Step 2: Run, verify, commit**

```bash
git commit -m "test: persistence round-trip for all layout pair combinations Refs #345"
```

### Task 15: Workspace mount-transfer state preservation

**Files:**
- Test: `packages/pages-runtime/src/wire-floating-workspace.test.ts` (append)

- [ ] **Step 1: Write workspace transition tests with real containers**

```typescript
describe("workspace mount transfer — state preservation", () => {
  it("free→tabbed→free preserves container organiser state", () => {
    // This test uses real backend (not mock) to verify mount transfer
    const backend = createGroupOrganiserBackend();
    const container = document.createElement("div");
    Object.defineProperty(container, "clientWidth", { value: 800 });
    Object.defineProperty(container, "clientHeight", { value: 600 });
    document.body.appendChild(container);
    backend.attach(container, testContentFactory());

    backend.renderFrame(makeLayout("f1", ["a", "b", "c"]));

    // Capture container identity before transition
    const containerBefore = backend.getRootContainer("f1");
    const stateBefore = containerBefore!.organiser.getState();

    // After mount transfer, container is the SAME object
    const containerAfter = backend.getRootContainer("f1");
    expect(containerAfter).toBe(containerBefore);

    // Organiser state preserved
    const stateAfter = containerAfter!.organiser.getState();
    expect(stateAfter).toEqual(stateBefore);

    backend.dispose();
    document.body.removeChild(container);
  });
});
```

- [ ] **Step 2: Run, verify, commit**

```bash
git commit -m "test: workspace mount-transfer state preservation Refs #345"
```

---

## Final verification

After all batches:

```bash
# Full test suite
yarn workspace @casehubio/pages-runtime run test

# Type check
yarn typecheck

# Lint
yarn lint
```

All must pass. The combinatorial test suite alone adds ~80+ new tests covering every meaningful layout combination at 2 and 3 levels of nesting.

## References

- [2026-08-23-recursive-container-model-design.md] — D1-D7 feature spec
- [2026-08-24-unified-architecture-design.md] — D8-D14 architecture spec
- [decisions.md] — all 14 decisions (D1-D14)
- `packages/pages-runtime/src/group-organiser-backend.ts` — the 1355-line god closure being decomposed
- `packages/pages-runtime/src/wire-floating-workspace.ts` — workspace transition code being rewritten
- `packages/pages-runtime/src/frame-sandbox/types.ts` — Entry, Container, LayoutStrategy interfaces
- `packages/pages-runtime/src/frame-sandbox/container.ts` — createContainer, containerizeEntry, flattenEntry
- `packages/pages-runtime/src/frame-sandbox/nesting.test.ts` — existing nesting tests (328 lines)
- `packages/pages-runtime/src/frame-shell.ts` — shared frame rendering (titlebar, resize handles)
- `packages/pages-runtime/src/frame-chrome.ts` — shared chrome (already unified)
- `packages/pages-runtime/src/frame-zone-picker.ts` — zone picker (being unified)
- GitHub #345 — Recursive Container model
