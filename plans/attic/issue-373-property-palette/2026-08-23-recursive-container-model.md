# Recursive Container Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #345 — Recursive Container model — entries as nested Containers
**Issue group:** #345

**Goal:** Enable entries within a Container to own child Containers, making the full tree self-describing and enabling in-tab nesting via a nest button.

**Architecture:** `Entry` gains `childContainer?: Container`. The external `FrameState.childContainers` map is eliminated. Tree-walking helpers, split creation, collapse, and persistence all traverse via `Entry.childContainer`. `Layout` type moves to pages-component to avoid circular package dependency. A new `captureContainerTree()` backend API bridges persistence across the engine/backend boundary.

**Tech Stack:** TypeScript, Vitest, Lit web components, pages-runtime, pages-component

## Global Constraints

- Backward compatible — existing layouts without nesting load unchanged
- Unified depth: `DEFAULT_POLICY.maxDepth = 5`, `SPLIT_POLICY` for split containers
- Content-agnostic — nesting is layout, not content (per `content-agnostic-workbench` protocol)
- All file edits via IntelliJ MCP (`ide_edit_member`, `ide_replace_member`, `ide_insert_member`)
- Test runner: `yarn workspace @casehubio/pages-runtime run test`
- Build check: `yarn build:packages`

---

## Batch 1: Type Foundation — Layout move + Entry extension

After this batch: `Layout` type lives in pages-component, `Entry` has `childContainer`, `Container` interface lives in types.ts, policies are unified. All existing tests pass with no behavior change.

### Task 1: Move Layout type to pages-component and add persistence types

**Files:**
- Modify: `packages/pages-component/src/model/types.ts`
- Modify: `packages/pages-runtime/src/frame-sandbox/types.ts`
- Modify: `packages/pages-runtime/src/frame-sandbox/index.ts`
- Test: `yarn build:packages` (compile check — no new test file needed, this is a type-only move)

**Interfaces:**
- Produces: `Layout` type exported from `@casehubio/pages-component`, `ContainerState` interface, `FrameTabConfig.content` becomes `Component | null`, `FrameTabConfig.children?: ContainerState`, `FrameLayout.containerTree?: ContainerState`

- [ ] **Step 1: Add Layout type and ContainerState to pages-component**

In `packages/pages-component/src/model/types.ts`, add before the `FrameTabConfig` interface:

```typescript
export type Layout = "free" | "tabbed" | "accordion" | "splith" | "splitv" | "content";

export interface ContainerState {
  readonly layout: Layout;
  readonly tabs: readonly FrameTabConfig[];
  readonly layoutState?: unknown;
}
```

- [ ] **Step 2: Update FrameTabConfig**

Change `FrameTabConfig.content` from `readonly content: Component` to `readonly content: Component | null`. Add `readonly children?: ContainerState`.

```typescript
export interface FrameTabConfig {
  readonly key: string;
  readonly label: string;
  readonly icon?: string;
  readonly content: Component | null;
  readonly children?: ContainerState;
}
```

- [ ] **Step 3: Add containerTree to FrameLayout**

Add to `FrameLayout` interface:

```typescript
readonly containerTree?: ContainerState;
```

- [ ] **Step 4: Update pages-runtime types.ts — remove Layout, re-export**

In `packages/pages-runtime/src/frame-sandbox/types.ts`, remove the `Layout` type definition. Add at the top:

```typescript
export type { Layout } from "@casehubio/pages-component";
```

Keep all other types unchanged. The re-export maintains backward compat for consumers importing from pages-runtime.

- [ ] **Step 5: Update index.ts re-exports**

In `packages/pages-runtime/src/frame-sandbox/index.ts`, update the Layout export to come from the re-export in types.ts (no change needed — it already re-exports `Layout` from `./types.js`).

- [ ] **Step 6: Build check**

Run: `yarn build:packages`
Expected: clean build. All imports resolve. No circular dependency.

- [ ] **Step 7: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: all existing tests pass unchanged.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/model/types.ts packages/pages-runtime/src/frame-sandbox/types.ts packages/pages-runtime/src/frame-sandbox/index.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor: move Layout type to pages-component, add ContainerState and persistence types Refs #345"
```

### Task 2: Move Container interface to types.ts, extend Entry, unify policies

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/types.ts`
- Modify: `packages/pages-runtime/src/frame-sandbox/container.ts`
- Modify: `packages/pages-runtime/src/frame-sandbox/index.ts`
- Test: `packages/pages-runtime/src/frame-sandbox/container.test.ts` (extend existing)

**Interfaces:**
- Consumes: `Layout` from pages-component (Task 1)
- Produces: `Container` interface in types.ts, `Entry.childContainer?: Container`, `DEFAULT_POLICY.maxDepth = 5`, `SPLIT_POLICY` constant, `ContainerConfig` interface in types.ts

- [ ] **Step 1: Write failing test — Entry.childContainer field exists**

In `packages/pages-runtime/src/frame-sandbox/container.test.ts`, add:

```typescript
it("entry can hold a childContainer", () => {
  const child = createContainer({
    entries: [{ key: "inner", label: "Inner" }],
    layout: "tabbed",
    contentFactory: () => ({ element: document.createElement("div") }),
    depth: 2,
  });
  const entry: Entry = { key: "outer", label: "Outer", childContainer: child };
  expect(entry.childContainer).toBe(child);
  expect(entry.childContainer!.depth).toBe(2);
  child.dispose();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test`
Expected: TypeScript error — `childContainer` does not exist on `Entry`.

- [ ] **Step 3: Move Container and ContainerConfig interfaces to types.ts**

In `packages/pages-runtime/src/frame-sandbox/types.ts`, add the `Container` interface (copy from `container.ts:22-34`) and `ContainerConfig` interface (copy from `container.ts:36-47`):

```typescript
export interface Container {
  readonly entries: readonly Entry[];
  readonly organiser: LayoutStrategy;
  readonly policy: ContainerPolicy;
  readonly depth: number;
  addEntry(entry: Entry, atIndex?: number): void;
  removeEntry(key: string): void;
  replaceChild(oldKey: string, newChild: Entry): void;
  setLayout(type: Layout): void;
  mount(container: HTMLElement): void;
  unmount(): void;
  dispose(): void;
}

export interface ContainerConfig {
  entries: Entry[];
  layout: Layout;
  policy?: ContainerPolicy;
  contentFactory: ContentFactory;
  callbacks?: LayoutCallbacks;
  depth?: number;
  freeLayoutState?: FreeLayoutState;
  onCollapse?: (remainingChild: Entry) => void;
  onAdd?: () => void;
  onLayoutChange?: (type: Layout) => void;
}
```

Add `childContainer` to `Entry`:

```typescript
export interface Entry {
  readonly key: string;
  readonly label: string;
  contentElement?: HTMLElement | undefined;
  contentDispose?: (() => void) | undefined;
  meta?: PerLayoutMeta;
  childContainer?: Container | undefined;
}
```

- [ ] **Step 4: Update container.ts — remove moved interfaces, import from types**

In `packages/pages-runtime/src/frame-sandbox/container.ts`:
- Remove the `Container` interface definition (lines 22-34)
- Remove the `ContainerConfig` interface definition (lines 36-47)
- Update the import to include `Container` and `ContainerConfig` from `./types.js`
- Keep the re-export: `export type { Entry, ContentFactory } from "./types.js"`
- Add re-export: `export type { Container, ContainerConfig } from "./types.js"`

- [ ] **Step 5: Update index.ts re-exports**

In `packages/pages-runtime/src/frame-sandbox/index.ts`, change:
```typescript
export type { Container, ContainerConfig } from "./container";
```
to:
```typescript
export type { Container, ContainerConfig } from "./types.js";
```

- [ ] **Step 6: Update DEFAULT_POLICY and add SPLIT_POLICY**

In `packages/pages-runtime/src/frame-sandbox/types.ts`:

```typescript
export const DEFAULT_POLICY: ContainerPolicy = {
  allowedLayouts: ["free", "tabbed", "accordion"],
  maxDepth: 5,
};

export const SPLIT_POLICY: ContainerPolicy = {
  allowedLayouts: ["free", "tabbed", "accordion", "splith", "splitv"],
  maxDepth: 5,
};
```

Add `SPLIT_POLICY` to the index.ts exports.

- [ ] **Step 7: Write failing test — maxDepth=5 allows deeper nesting**

```typescript
it("DEFAULT_POLICY allows depth 5", () => {
  expect(DEFAULT_POLICY.maxDepth).toBe(5);
});

it("SPLIT_POLICY includes split layouts", () => {
  expect(SPLIT_POLICY.allowedLayouts).toContain("splith");
  expect(SPLIT_POLICY.allowedLayouts).toContain("splitv");
  expect(SPLIT_POLICY.maxDepth).toBe(5);
});
```

- [ ] **Step 8: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test`
Expected: all tests pass.

- [ ] **Step 9: Build check**

Run: `yarn build:packages`
Expected: clean build.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/frame-sandbox/
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor: move Container to types.ts, add Entry.childContainer, unify policies Refs #345"
```

---

## Batch 2: Tree-Walking Refactor — eliminate childContainers map

After this batch: all tree-walking helpers use `Entry.childContainer`, `FrameState.childContainers` is removed, `createSplitContainer` stores children on entries, split collapse reads from entries. All existing split behavior preserved.

### Task 3: Refactor tree-walking helpers and remove childContainers map

**Files:**
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts`
- Test: `packages/pages-runtime/src/group-organiser-backend.test.ts` (extend existing)

**Interfaces:**
- Consumes: `Entry.childContainer` (Task 2), `SPLIT_POLICY` (Task 2)
- Produces: Refactored `findLeafContainer`, `findContainerWithTab`, `forEachLeafContainer`, `findParentEntry` (renamed from `findParentSplitEntry`) — all without `childMap` param. `FrameState` without `childContainers`. `createSplitContainer` using `entry.childContainer`. `createLeafContainer` using `DEFAULT_POLICY`.

- [ ] **Step 1: Write failing tests for refactored helpers**

In `packages/pages-runtime/src/group-organiser-backend.test.ts`, add tests that verify the helpers work with `entry.childContainer` instead of external map. These tests may need to be written after the helper signatures change, so start by writing the test expectations:

```typescript
describe("tree-walking with entry.childContainer", () => {
  it("findLeafContainer finds leaf in mixed container", () => {
    // A tabbed container with 3 entries: 1 nested (has childContainer), 2 leaves
    // findLeafContainer should return the parent container (it has leaf entries)
    const innerContainer = createContainer({
      entries: [{ key: "deep", label: "Deep" }],
      layout: "tabbed",
      contentFactory: () => ({ element: document.createElement("div") }),
      depth: 2,
    });
    const entries: Entry[] = [
      { key: "a", label: "A" },
      { key: "b", label: "B", childContainer: innerContainer },
      { key: "c", label: "C" },
    ];
    const container = createContainer({
      entries,
      layout: "tabbed",
      contentFactory: () => ({ element: document.createElement("div") }),
    });
    const leaf = findLeafContainer(container);
    expect(leaf).toBe(container); // mixed container IS a leaf target
    container.dispose();
  });
});
```

- [ ] **Step 2: Refactor the four tree-walking helpers**

In `group-organiser-backend.ts`, update each helper:

**findLeafContainer** — remove `childMap` param, walk `entry.childContainer`:
```typescript
function findLeafContainer(
  container: Container,
  predicate?: (c: Container) => boolean,
): Container | null {
  for (const entry of container.entries) {
    if (entry.childContainer) {
      const found = findLeafContainer(entry.childContainer, predicate);
      if (found) return found;
    }
  }
  const hasLeafEntries = container.entries.some(e => !e.childContainer);
  if (hasLeafEntries && (!predicate || predicate(container))) return container;
  return null;
}
```

**findContainerWithTab** — remove `childMap` param:
```typescript
function findContainerWithTab(
  container: Container,
  tabKey: string,
): Container | null {
  if (container.entries.some(e => e.key === tabKey)) return container;
  for (const entry of container.entries) {
    if (entry.childContainer) {
      const found = findContainerWithTab(entry.childContainer, tabKey);
      if (found) return found;
    }
  }
  return null;
}
```

**forEachLeafContainer** — remove `childMap` param:
```typescript
function forEachLeafContainer(
  container: Container,
  callback: (container: Container, paneKey?: string) => void,
  paneKey?: string,
): void {
  for (const entry of container.entries) {
    if (entry.childContainer) {
      forEachLeafContainer(entry.childContainer, callback, entry.key);
    }
  }
  const hasLeafEntries = container.entries.some(e => !e.childContainer);
  if (hasLeafEntries) callback(container, paneKey);
}
```

**findParentSplitEntry → findParentEntry** — remove `childMap` param, rename:
```typescript
function findParentEntry(
  root: Container,
  targetContainer: Container,
): { parent: Container; entryKey: string } | null {
  for (const entry of root.entries) {
    if (entry.childContainer === targetContainer) return { parent: root, entryKey: entry.key };
    if (entry.childContainer) {
      const found = findParentEntry(entry.childContainer, targetContainer);
      if (found) return found;
    }
  }
  return null;
}
```

- [ ] **Step 3: Remove childContainers from FrameState**

Change `FrameState` interface — remove `childContainers: Map<string, Container>`:
```typescript
interface FrameState {
  readonly key: string;
  position: { x: number; y: number };
  size: { width: number; height: number };
  frameEl: HTMLElement;
  rootContainer: Container;
  tabContentEl: HTMLElement;
}
```

- [ ] **Step 4: Update createSplitContainer — store children on entries**

```typescript
function createSplitContainer(
  frameKey: string,
  direction: "splith" | "splitv",
  childEntries: Array<{ key: string; child: Container }>,
): Container {
  const entries: Entry[] = childEntries.map(({ key, child }) => ({
    key,
    label: key,
    childContainer: child,
  }));

  return createContainer({
    entries,
    layout: direction,
    contentFactory: (entry: Entry) => {
      if (entry.childContainer) {
        const el = document.createElement("div");
        el.style.cssText = "display:flex;flex-direction:column;height:100%;";
        entry.childContainer.mount(el);
        return { element: el, dispose: () => entry.childContainer!.dispose() };
      }
      return { element: document.createElement("div") };
    },
    policy: SPLIT_POLICY,
    onCollapse: (remainingEntry) => {
      const state = frames.get(frameKey);
      if (!state) return;
      const remainingChild = remainingEntry.childContainer;
      if (remainingChild) {
        remainingChild.unmount();
        remainingEntry.childContainer = undefined;
        while (state.tabContentEl.firstChild) {
          state.tabContentEl.removeChild(state.tabContentEl.firstChild);
        }
        state.rootContainer = remainingChild;
        remainingChild.mount(state.tabContentEl);
      }
    },
  });
}
```

Note: remove the `state` parameter — it was only needed for `state.childContainers`.

- [ ] **Step 5: Update createLeafContainer — use DEFAULT_POLICY**

```typescript
function createLeafContainer(frameKey: string, entries: Entry[]): Container {
  const callbacks = createTabCallbacksForFrame(frameKey);
  return createContainer({
    entries,
    layout: "tabbed" as Layout,
    contentFactory: wrapContentFactory(frameKey),
    callbacks,
    policy: DEFAULT_POLICY,
    onAdd: () => { addChildToFrame(frameKey); },
    onLayoutChange: (type) => {
      for (const cb of layoutChangeCbs) cb(frameKey, type);
    },
  });
}
```

- [ ] **Step 6: Update all ~15 call sites**

Remove `state.childContainers` from every call to the tree-walking helpers. Each call site changes from `fn(state.rootContainer, state.childContainers, ...)` to `fn(state.rootContainer, ...)`. Also remove `childContainers: new Map()` from FrameState construction.

Update `splitFrame` to not pass `state` to `createSplitContainer` (it no longer needs it).

- [ ] **Step 7: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose`
Expected: all tests pass. Existing split behavior preserved.

- [ ] **Step 8: Build check**

Run: `yarn build:packages`
Expected: clean build.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/group-organiser-backend.ts packages/pages-runtime/src/group-organiser-backend.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "refactor: tree-walking via Entry.childContainer, eliminate childContainers map Refs #345"
```

---

## Batch 3: Nesting Operations — containerize, flatten, nest button

After this batch: users can nest entries via a content-area button, auto-flatten works on collapse. The core nesting UX is functional.

### Task 4: Implement containerizeEntry and flattenEntry

**Files:**
- Modify: `packages/pages-runtime/src/frame-sandbox/container.ts`
- Test: `packages/pages-runtime/src/frame-sandbox/container.test.ts`

**Interfaces:**
- Consumes: `Entry.childContainer` (Task 2), `createContainer` (existing), `ContentFactory` (existing)
- Produces: `containerizeEntry(entry, container, contentFactory): void`, `flattenEntry(parentEntry, remainingChildEntry, contentFactory): void`

- [ ] **Step 1: Write failing test — containerizeEntry converts leaf to non-leaf**

```typescript
describe("containerizeEntry", () => {
  it("converts leaf entry to non-leaf with childContainer", () => {
    const factory: ContentFactory = () => ({ element: document.createElement("div") });
    const entry: Entry = { key: "tab1", label: "Tab 1" };
    (entry as any)._content = { type: "html", props: { content: "<p>Hello</p>" } };

    const parent = createContainer({
      entries: [entry],
      layout: "tabbed",
      contentFactory: factory,
    });

    containerizeEntry(entry, parent, factory);

    expect(entry.childContainer).toBeDefined();
    expect(entry.contentElement).toBeUndefined();
    expect(entry.childContainer!.entries.length).toBe(2);
    expect((entry.childContainer!.entries[0] as any)._content).toEqual(
      { type: "html", props: { content: "<p>Hello</p>" } }
    );

    parent.dispose();
  });

  it("respects maxDepth — throws at limit", () => {
    const factory: ContentFactory = () => ({ element: document.createElement("div") });
    const entry: Entry = { key: "tab1", label: "Tab 1" };
    const parent = createContainer({
      entries: [entry],
      layout: "tabbed",
      contentFactory: factory,
      depth: 5,
      policy: { allowedLayouts: ["tabbed"], maxDepth: 5 },
    });

    expect(() => containerizeEntry(entry, parent, factory)).toThrow(/maximum nesting depth/);
    parent.dispose();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test`
Expected: FAIL — `containerizeEntry` is not defined.

- [ ] **Step 3: Implement containerizeEntry**

In `packages/pages-runtime/src/frame-sandbox/container.ts`:

```typescript
export function containerizeEntry(
  entry: Entry,
  parentContainer: Container,
  contentFactory: ContentFactory,
): void {
  if (entry.childContainer) return;
  if (parentContainer.depth + 1 > parentContainer.policy.maxDepth) {
    throw new Error(
      `Cannot nest at depth ${parentContainer.depth + 1} — maximum nesting depth is ${parentContainer.policy.maxDepth}`
    );
  }

  if (entry.contentDispose) entry.contentDispose();
  entry.contentElement = undefined;
  entry.contentDispose = undefined;

  const wrappedKey = `entry-${String(Date.now())}-${String(Math.random().toString(36).slice(2, 6))}`;
  const wrapped: Entry = { key: wrappedKey, label: entry.label };
  (wrapped as any)._content = (entry as any)._content;

  const emptyKey = `entry-${String(Date.now())}-${String(Math.random().toString(36).slice(2, 6))}`;
  const empty: Entry = { key: emptyKey, label: "Tab 2" };

  const child = createContainer({
    entries: [wrapped, empty],
    layout: "tabbed",
    contentFactory,
    depth: parentContainer.depth + 1,
    policy: parentContainer.policy,
    onCollapse: (remaining) => flattenEntry(entry, remaining, contentFactory),
  });

  (entry as any)._content = undefined;
  entry.childContainer = child;
}
```

- [ ] **Step 4: Write failing test — flattenEntry converts non-leaf back to leaf**

```typescript
describe("flattenEntry", () => {
  it("restores leaf state from remaining child", () => {
    const factory: ContentFactory = () => ({ element: document.createElement("div") });
    const entry: Entry = { key: "tab1", label: "Tab 1" };
    (entry as any)._content = { type: "html", props: { content: "<p>Hello</p>" } };

    const parent = createContainer({
      entries: [entry],
      layout: "tabbed",
      contentFactory: factory,
    });

    containerizeEntry(entry, parent, factory);
    expect(entry.childContainer).toBeDefined();

    const remaining = entry.childContainer!.entries[0]!;
    flattenEntry(entry, remaining, factory);

    expect(entry.childContainer).toBeUndefined();
    expect((entry as any)._content).toEqual(
      { type: "html", props: { content: "<p>Hello</p>" } }
    );

    parent.dispose();
  });
});
```

- [ ] **Step 5: Implement flattenEntry**

```typescript
export function flattenEntry(
  parentEntry: Entry,
  remainingChildEntry: Entry,
  contentFactory: ContentFactory,
): void {
  if (!parentEntry.childContainer) return;

  parentEntry.childContainer.unmount();
  (parentEntry as any)._content = (remainingChildEntry as any)._content;

  if (remainingChildEntry.contentDispose) remainingChildEntry.contentDispose();
  remainingChildEntry.contentElement = undefined;
  remainingChildEntry.contentDispose = undefined;

  parentEntry.childContainer.dispose();
  parentEntry.childContainer = undefined;
}
```

- [ ] **Step 6: Export from container.ts and index.ts**

Add `containerizeEntry` and `flattenEntry` to the exports in `container.ts` and `index.ts`.

- [ ] **Step 7: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose container.test`
Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/frame-sandbox/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: containerizeEntry and flattenEntry for recursive nesting Refs #345"
```

### Task 5: Add nest button to content area

**Files:**
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts`
- Test: `packages/pages-runtime/src/group-organiser-backend.test.ts`

**Interfaces:**
- Consumes: `containerizeEntry`, `flattenEntry` (Task 4), `Container.depth`, `Container.policy` (Task 2)
- Produces: Nest button injected into leaf tab content areas, wired to `containerizeEntry`

- [ ] **Step 1: Write failing test — nest button appears in leaf content**

```typescript
describe("nest button", () => {
  it("injects nest button into leaf tab content area", () => {
    // Create a frame with a tabbed container, verify the nest button exists
    // in the content area
    // This test will need the full backend setup — adapt from existing renderFrame tests
  });
});
```

- [ ] **Step 2: Implement nest button injection**

In `group-organiser-backend.ts`, after the content is mounted for each leaf entry, inject a nest button overlay:

```typescript
function injectNestButton(
  contentArea: HTMLElement,
  entry: Entry,
  container: Container,
  frameKey: string,
): HTMLElement | null {
  if (entry.childContainer) return null;
  if (container.depth >= container.policy.maxDepth) return null;

  const btn = document.createElement("button");
  btn.setAttribute("data-nest-button", "");
  btn.setAttribute("role", "button");
  btn.setAttribute("aria-label", "Nest content into tabbed container");
  btn.textContent = "⊞";
  btn.title = "Nest";
  btn.style.cssText =
    "position:absolute;bottom:8px;right:8px;z-index:10;" +
    "padding:4px 8px;border:1px solid var(--pages-border-1,#333);" +
    "background:var(--pages-surface-2,#222);color:var(--pages-text-2,#aaa);" +
    "border-radius:4px;cursor:pointer;font-size:14px;opacity:0.5;" +
    "transition:opacity 0.15s ease;";
  btn.addEventListener("mouseenter", () => { btn.style.opacity = "1"; });
  btn.addEventListener("mouseleave", () => { btn.style.opacity = "0.5"; });
  btn.addEventListener("click", (e) => {
    e.stopPropagation();
    containerizeEntry(entry, container, wrapContentFactory(frameKey));
    btn.remove();
  });

  contentArea.style.position = "relative";
  contentArea.appendChild(btn);
  return btn;
}
```

Wire this into the content factory flow — call `injectNestButton` after each leaf entry's content is mounted.

- [ ] **Step 3: Run tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose`
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/group-organiser-backend.ts packages/pages-runtime/src/group-organiser-backend.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: nest button in content area triggers containerizeEntry Refs #345"
```

---

## Batch 4: Persistence — capture and restore Container trees

After this batch: nested and split Container trees are persisted and restored correctly. Full round-trip works.

### Task 6: Add captureContainerTree to backend, wire to engine

**Files:**
- Modify: `packages/pages-runtime/src/floating-frame-backend.ts`
- Modify: `packages/pages-runtime/src/group-organiser-backend.ts`
- Modify: `packages/pages-runtime/src/floating-frame-engine.ts`
- Test: `packages/pages-runtime/src/group-organiser-backend.test.ts`

**Interfaces:**
- Consumes: `ContainerState` (Task 1), `Entry.childContainer` (Task 2), `FrameLayout.containerTree` (Task 1)
- Produces: `FloatingFrameBackend.captureContainerTree(frameKey): ContainerState | undefined`, engine `captureLayout()` includes `containerTree`, engine `restoreLayout()` reads `containerTree`

- [ ] **Step 1: Write failing test — captureContainerTree serializes tree**

```typescript
describe("captureContainerTree", () => {
  it("returns undefined for flat single-container frame", () => {
    // Create a frame with flat tabs, call captureContainerTree
    // Expected: undefined (no nesting or splits)
  });

  it("serializes split tree", () => {
    // Create a frame, split it, call captureContainerTree
    // Expected: ContainerState with layout: "splith", two children
  });

  it("serializes nested entry tree", () => {
    // Create a frame, containerize one entry, call captureContainerTree
    // Expected: ContainerState with layout: "tabbed", one child has children
  });
});
```

- [ ] **Step 2: Add captureContainerTree to FloatingFrameBackend interface**

In `packages/pages-runtime/src/floating-frame-backend.ts`:

```typescript
captureContainerTree(frameKey: string): ContainerState | undefined;
```

Add the `ContainerState` import from `@casehubio/pages-component`.

- [ ] **Step 3: Implement captureContainerTree in group-organiser-backend**

```typescript
function captureContainerTreeFromContainer(container: Container): ContainerState {
  const tabs: FrameTabConfig[] = container.entries.map(entry => {
    if (entry.childContainer) {
      return {
        key: entry.key,
        label: entry.label,
        content: null,
        children: captureContainerTreeFromContainer(entry.childContainer),
      };
    }
    return {
      key: entry.key,
      label: entry.label,
      content: (entry as any)._content ?? { type: "html", props: {} },
    };
  });

  return {
    layout: container.organiser.type,
    tabs,
    layoutState: container.organiser.getState(),
  };
}
```

Expose via the backend API:
```typescript
captureContainerTree(frameKey: string): ContainerState | undefined {
  const state = frames.get(frameKey);
  if (!state) return undefined;
  const hasNesting = state.rootContainer.entries.some(e => e.childContainer);
  const isSplit = isSplitLayout(state.rootContainer.organiser.type);
  if (!hasNesting && !isSplit) return undefined;
  return captureContainerTreeFromContainer(state.rootContainer);
}
```

- [ ] **Step 4: Wire captureLayout in the engine**

In `packages/pages-runtime/src/floating-frame-engine.ts`, update `captureLayout()`:

```typescript
captureLayout(): readonly FrameLayout[] {
  const normalized = normalizeForSave(frames);
  return [...normalized.values()]
    .sort((a, b) => a.order - b.order)
    .map(layout => ({
      ...layout,
      containerTree: backend.captureContainerTree(layout.key),
    }));
}
```

- [ ] **Step 5: Wire restoreLayout in the backend**

Update `renderFrame` in `group-organiser-backend.ts` to accept and restore `containerTree`:

When `layout.containerTree` is present, build the Container tree recursively from the serialized state instead of creating a flat tabbed container.

- [ ] **Step 6: Write round-trip test**

```typescript
describe("persistence round-trip", () => {
  it("nested tree survives capture and restore", () => {
    // Create frame → containerize entry → captureLayout → restoreLayout → verify tree
  });

  it("flat layout backward compat — no containerTree", () => {
    // Restore a FrameLayout without containerTree → verify flat tabs work
  });
});
```

- [ ] **Step 7: Run all tests**

Run: `yarn workspace @casehubio/pages-runtime run test -- --reporter=verbose`
Expected: all tests pass.

- [ ] **Step 8: Build check**

Run: `yarn build:packages`
Expected: clean build.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-runtime/src/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: captureContainerTree persistence bridge for recursive Container trees Refs #345"
```

---

## Batch 5: Integration verification

After this batch: full type check passes, all tests green, build succeeds.

### Task 7: Full integration test and type check

**Files:**
- Test: all existing test suites
- Build: full build

**Interfaces:**
- Consumes: everything from Batches 1-4

- [ ] **Step 1: Run full type check**

Run: `yarn typecheck`
Expected: clean.

- [ ] **Step 2: Run full test suite**

Run: `yarn workspace @casehubio/pages-runtime run test`
Expected: all pass.

- [ ] **Step 3: Run full build**

Run: `yarn build`
Expected: clean build.

- [ ] **Step 4: Run lint**

Run: `yarn lint`
Expected: clean or only pre-existing warnings.

- [ ] **Step 5: Commit any fixes**

If any issues surfaced, fix and commit with:
```bash
git -C /Users/mdproctor/claude/casehub/pages commit -m "fix: address integration issues from recursive container model Refs #345"
```

---

## References

- [2026-08-23-recursive-container-model-design.md] — design spec this plan implements
- [decisions.md] — D1-D7 design decisions
- `packages/pages-runtime/src/frame-sandbox/types.ts:1-81` — Entry, Layout, ContainerPolicy types
- `packages/pages-runtime/src/frame-sandbox/container.ts:1-280` — Container, createContainer
- `packages/pages-runtime/src/frame-sandbox/split-strategy.ts:1-172` — split layout with collapse
- `packages/pages-runtime/src/group-organiser-backend.ts:39-128` — FrameState, tree-walking helpers
- `packages/pages-runtime/src/floating-frame-backend.ts:1-55` — FloatingFrameBackend interface
- `packages/pages-runtime/src/floating-frame-engine.ts:300-328` — captureLayout/restoreLayout
- `packages/pages-component/src/model/types.ts:61-118` — FrameTabConfig, FrameLayout
- `docs/protocols/casehub/content-agnostic-workbench.md` — content-agnostic constraint
- GitHub #345 — Recursive Container model
- GitHub #312 — Container tree migration (predecessor)
