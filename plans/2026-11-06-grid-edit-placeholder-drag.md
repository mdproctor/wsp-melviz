# Grid Edit Placeholder Drag — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #618 — Grid edit UX: placeholder-based drag with vertical compaction

**Goal:** Replace the pixel-positioned ghost overlay with a structural CSS Grid placeholder, add vertical compaction, drag handles, and collision resolution modes.

**Architecture:** All grid items use CSS Grid positioning via `applyGridPlacement` (integer `{x, y, w, h}` coordinates). The drag placeholder is a real grid item, not an overlay. Collision resolution is controlled by a `collision` property on the grid layout (`displace` for auto-push, `prompt` for popup).

**Tech Stack:** TypeScript, Vitest, CSS Grid, Playwright

## Global Constraints

- All grid coordinates are 0-based integers on a 12-column grid
- All item positioning uses `applyGridPlacement` — no `position: absolute` on grid items
- `overlaps()` function from `grid-collision.ts` is the single source of truth for overlap detection
- The `pages-grid-edit` custom event contract is unchanged
- All mutations through `_applyEdit('grid', ...)` protocol

---

## Batch 1: Compaction Algorithm (Pure Functions)

### Task 1: Compaction engine with TDD

**Files:**
- Create: `packages/pages-component/src/renderer/grid-compaction.ts`
- Create: `packages/pages-component/src/renderer/grid-compaction.test.ts`
- Modify: `packages/pages-component/src/renderer/grid-collision.ts` — keep `overlaps` and `IdentifiedPlacement`/`PositionedPlacement` types, remove `resolveCollisions`

**Interfaces:**
- Consumes: `overlaps()`, `IdentifiedPlacement`, `PositionedPlacement` from `grid-collision.ts`
- Produces:
  ```typescript
  function compactPlacements(
    items: readonly IdentifiedPlacement[],
    movedId: string,
    newPlacement: PositionedPlacement,
  ): IdentifiedPlacement[]
  ```
  Returns all items with resolved positions after place/push/compact.

- [ ] **Step 1: Write failing tests for compaction**

Create `grid-compaction.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { compactPlacements } from "./grid-compaction.js";
import type { IdentifiedPlacement } from "./grid-collision.js";

describe("compactPlacements", () => {
  it("returns items unchanged when no overlap exists", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 2 } },
      { id: "b", placement: { mode: "positioned", x: 6, y: 0, w: 6, h: 2 } },
    ];
    const result = compactPlacements(items, "a", items[0]!.placement);
    expect(result.find(i => i.id === "a")!.placement).toEqual(items[0]!.placement);
    expect(result.find(i => i.id === "b")!.placement).toEqual(items[1]!.placement);
  });

  it("pushes overlapping item down and compacts", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 2 } },
      { id: "b", placement: { mode: "positioned", x: 0, y: 4, w: 6, h: 2 } },
    ];
    const result = compactPlacements(items, "a", { mode: "positioned", x: 0, y: 3, w: 6, h: 2 });
    expect(result.find(i => i.id === "a")!.placement.y).toBe(3);
    expect(result.find(i => i.id === "b")!.placement.y).toBe(5);
  });

  it("compacts gaps upward after displacement", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 0, y: 0, w: 4, h: 1 } },
      { id: "b", placement: { mode: "positioned", x: 0, y: 3, w: 4, h: 1 } },
    ];
    // b is at y=3 with a gap — compact should pull it up to y=1
    const result = compactPlacements(items, "a", items[0]!.placement);
    expect(result.find(i => i.id === "b")!.placement.y).toBe(1);
  });

  it("compacts items that don't overlap in columns independently", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 0, y: 0, w: 4, h: 2 } },
      { id: "b", placement: { mode: "positioned", x: 6, y: 3, w: 4, h: 1 } },
    ];
    const result = compactPlacements(items, "a", items[0]!.placement);
    // b is in columns 6-9, a is in 0-3 — no overlap, b compacts to y=0
    expect(result.find(i => i.id === "b")!.placement.y).toBe(0);
  });

  it("cascades: pushing B causes overlap with C, C is pushed down", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 0, y: 0, w: 4, h: 2 } },
      { id: "b", placement: { mode: "positioned", x: 0, y: 3, w: 4, h: 2 } },
      { id: "c", placement: { mode: "positioned", x: 0, y: 5, w: 4, h: 2 } },
    ];
    const result = compactPlacements(items, "a", { mode: "positioned", x: 0, y: 2, w: 4, h: 2 });
    expect(result.find(i => i.id === "a")!.placement.y).toBe(2);
    expect(result.find(i => i.id === "b")!.placement.y).toBe(4);
    expect(result.find(i => i.id === "c")!.placement.y).toBe(6);
  });

  it("handles spanning items — wide item pushed by narrow item", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 2, y: 0, w: 2, h: 1 } },
      { id: "b", placement: { mode: "positioned", x: 0, y: 1, w: 12, h: 1 } },
    ];
    const result = compactPlacements(items, "a", { mode: "positioned", x: 2, y: 1, w: 2, h: 1 });
    // a moves to y=1, b (full-width) overlaps and is pushed to y=2
    expect(result.find(i => i.id === "a")!.placement.y).toBe(1);
    expect(result.find(i => i.id === "b")!.placement.y).toBe(2);
  });

  it("is idempotent", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 0, y: 0, w: 6, h: 2 } },
      { id: "b", placement: { mode: "positioned", x: 0, y: 1, w: 6, h: 2 } },
    ];
    const placement = { mode: "positioned" as const, x: 0, y: 0, w: 6, h: 2 };
    const first = compactPlacements(items, "a", placement);
    const second = compactPlacements(first, "a", placement);
    expect(second).toEqual(first);
  });

  it("does not compact horizontally — column positions are preserved", () => {
    const items: IdentifiedPlacement[] = [
      { id: "a", placement: { mode: "positioned", x: 4, y: 0, w: 4, h: 1 } },
    ];
    const result = compactPlacements(items, "a", items[0]!.placement);
    expect(result.find(i => i.id === "a")!.placement.x).toBe(4);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-component/src/renderer/grid-compaction.test.ts`
Expected: FAIL — module `./grid-compaction.js` not found

- [ ] **Step 3: Implement compactPlacements**

Create `grid-compaction.ts`:

```typescript
import type { PositionedPlacement, IdentifiedPlacement } from "./grid-collision.js";
import { overlaps } from "./grid-collision.js";

export function compactPlacements(
  items: readonly IdentifiedPlacement[],
  movedId: string,
  newPlacement: PositionedPlacement,
): IdentifiedPlacement[] {
  // 1. Place — apply the new placement for the moved item
  const result: IdentifiedPlacement[] = items.map(item =>
    item.id === movedId
      ? { id: item.id, placement: { ...newPlacement } }
      : { id: item.id, placement: { ...item.placement } },
  );

  const moved = result.find(i => i.id === movedId)!;

  // 2. Push — push items that overlap the moved item below it
  for (const item of result) {
    if (item.id === movedId) continue;
    if (overlaps(moved.placement, item.placement)) {
      (item as { placement: PositionedPlacement }).placement = {
        ...item.placement,
        y: moved.placement.y + moved.placement.h,
      };
    }
  }

  // Sort by y then x for cascading push
  result.sort((a, b) =>
    a.placement.y !== b.placement.y
      ? a.placement.y - b.placement.y
      : a.placement.x - b.placement.x,
  );

  // Cascade: if pushing caused new overlaps, resolve them
  for (let i = 0; i < result.length; i++) {
    for (let j = i + 1; j < result.length; j++) {
      if (overlaps(result[i]!.placement, result[j]!.placement)) {
        (result[j] as { placement: PositionedPlacement }).placement = {
          ...result[j]!.placement,
          y: result[i]!.placement.y + result[i]!.placement.h,
        };
      }
    }
  }

  // 3. Compact — move each item upward to the lowest valid y
  result.sort((a, b) =>
    a.placement.y !== b.placement.y
      ? a.placement.y - b.placement.y
      : a.placement.x - b.placement.x,
  );

  for (const item of result) {
    if (item.id === movedId) continue;
    let bestY = 0;
    while (bestY < item.placement.y) {
      const candidate: PositionedPlacement = { ...item.placement, y: bestY };
      const blocked = result.some(other =>
        other !== item && overlaps(candidate, other.placement),
      );
      if (!blocked) {
        (item as { placement: PositionedPlacement }).placement = candidate;
        break;
      }
      bestY++;
    }
  }

  return result;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-component/src/renderer/grid-compaction.test.ts`
Expected: all PASS

- [ ] **Step 5: Remove resolveCollisions from grid-collision.ts**

Keep `overlaps`, `PositionedPlacement`, and `IdentifiedPlacement` in `grid-collision.ts`. Remove the `resolveCollisions` function. Update imports in `grid-edit-controller.ts` to import from `grid-compaction.ts` instead (this will be wired in Task 3).

- [ ] **Step 6: Run existing grid-collision tests**

Run: `npx vitest run packages/pages-component/src/renderer/grid-collision.test.ts`
Expected: `overlaps` tests PASS, `resolveCollisions` tests FAIL (function removed). Remove the `resolveCollisions` describe block from `grid-collision.test.ts` — those behaviors are now covered by `grid-compaction.test.ts`.

- [ ] **Step 7: Commit**

```bash
git add packages/pages-component/src/renderer/grid-compaction.ts \
       packages/pages-component/src/renderer/grid-compaction.test.ts \
       packages/pages-component/src/renderer/grid-collision.ts \
       packages/pages-component/src/renderer/grid-collision.test.ts
git commit -m "feat(#618): compaction algorithm — place/push/compact with TDD"
```

---

## Batch 2: Placeholder Drag and Drag Handle

### Task 2: Replace ghost with structural placeholder in grid-edit-controller

**Files:**
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts` — replace ghost with placeholder div in CSS Grid
- Modify: `packages/pages-component/src/renderer/grid-overlay.ts` — remove `showGhost`, `hideGhost`, `createGridOverlay` (background grid); keep `createEditButton`, `removeEditButton`, `createInsertionZones`, `removeInsertionZones`
- Modify: `packages/pages-component/src/renderer/grid-drag.ts` — remove `cellRectToPixel`, `cellToPixel`; keep `snapToCell`, `computeGridGeometry`, `clampPlacement`, `hitTestHandle`, `DRAG_THRESHOLD`
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.test.ts` — update tests for placeholder behavior

**Interfaces:**
- Consumes: `compactPlacements()` from `grid-compaction.ts`, `applyGridPlacement()` from `grid.ts`
- Produces: same `GridEditController` interface (unchanged public API)

- [ ] **Step 1: Write failing test — placeholder appears during drag**

Add to `grid-edit-controller.test.ts`:

```typescript
it("inserts a placeholder div into the grid during drag", () => {
  // Set up grid with items, enter edit mode, simulate pointerdown + pointermove
  // Assert: gridEl contains a child with [data-grid-placeholder]
  // Assert: placeholder has grid-column and grid-row set (not position: absolute)
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-component/src/renderer/grid-edit-controller.test.ts`
Expected: FAIL — no placeholder element created

- [ ] **Step 3: Add placeholder creation to grid-edit-controller**

In `GridEditControllerImpl`, add:
- A `placeholder: HTMLDivElement | null = null` field
- In `handlePointerMove`, when drag starts: create a `<div data-grid-placeholder>` styled with dashed border and semi-transparent background. Append it to `gridEl`. Apply `applyGridPlacement(placeholder, newPlacement)` instead of calling `showGhost`.
- On each subsequent move: update placeholder placement via `applyGridPlacement`.
- Replace `resolveCollisions` call with `compactPlacements` from `grid-compaction.ts`.
- Remove all `showGhost`/`hideGhost` calls.

Placeholder styles:
```typescript
placeholder.style.border = "2px dashed var(--pages-accent-7, #3b82f6)";
placeholder.style.background = "var(--pages-accent-3, rgba(59, 130, 246, 0.15))";
placeholder.style.borderRadius = "var(--pages-radius-md, 6px)";
placeholder.style.pointerEvents = "none";
```

- [ ] **Step 4: Remove ghost overlay and pixel math**

In `grid-overlay.ts`:
- Remove `createGridOverlay`, `removeGridOverlay`, `showGhost`, `hideGhost`
- Keep `createEditButton`, `removeEditButton`, `createInsertionZones`, `removeInsertionZones`

In `grid-drag.ts`:
- Remove `cellRectToPixel` and `cellToPixel`
- Keep `snapToCell`, `computeGridGeometry`, `clampPlacement`, `hitTestHandle`, `DRAG_THRESHOLD`

In `grid-edit-controller.ts`:
- Remove `overlay` field and all references to it
- Remove `position: relative` from `enter()` (no longer needed — no absolute-positioned children)
- Update `cancelDrag` to remove placeholder instead of calling `hideGhost`
- Update `handlePointerUp` to remove placeholder and apply final placements
- Update imports: remove `createGridOverlay`, `removeGridOverlay`, `showGhost`, `hideGhost`

- [ ] **Step 5: Update insertion zones to use CSS Grid instead of absolute positioning**

In `grid-overlay.ts`, change `createZoneButton` to use `applyGridPlacement` instead of `cellRectToPixel`:

```typescript
function createZoneButton(gridEl: HTMLElement, x: number, y: number): HTMLElement {
  const btn = gridEl.ownerDocument.createElement("button");
  btn.textContent = "+";
  btn.dataset.insertionZone = "true";
  btn.dataset.x = String(x);
  btn.dataset.y = String(y);
  applyGridPlacement(btn, { mode: "positioned", x, y, w: 1, h: 1 });
  // ... styling (no position: absolute, no pixel coords)
  gridEl.appendChild(btn);
  return btn;
}
```

Remove the `geometry` parameter from `createInsertionZones` — it's no longer needed.

- [ ] **Step 6: Add CSS transitions for smooth reflow**

In `enter()`, add a transition to all grid items so they animate when displaced:

```typescript
el.style.transition = "grid-column 200ms ease, grid-row 200ms ease";
```

Remove transitions in `exit()`.

- [ ] **Step 7: Run all grid tests**

Run: `npx vitest run packages/pages-component/src/renderer/grid-edit-controller.test.ts packages/pages-component/src/renderer/grid-overlay.test.ts packages/pages-component/src/renderer/grid-drag.test.ts`
Expected: PASS (update any tests that reference removed functions)

- [ ] **Step 8: Commit**

```bash
git add packages/pages-component/src/renderer/
git commit -m "feat(#618): structural placeholder replaces ghost overlay, compaction wired"
```

### Task 3: Drag handle — title bar only

**Files:**
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts` — restrict drag initiation to top 24px
- Modify: `packages/pages-component/src/renderer/grid-drag.ts` — add `isDragHandle` hit test

**Interfaces:**
- Consumes: `hitTestHandle()` from `grid-drag.ts`
- Produces: `isDragHandle(pointerY: number, itemRect: DOMRect): boolean` in `grid-drag.ts`

- [ ] **Step 1: Write failing test — drag only from title bar**

Add to `grid-edit-controller.test.ts`:

```typescript
it("does not start drag when pointer is below the title bar zone", () => {
  // pointerdown at y = itemRect.top + 50 (below 24px zone)
  // pointermove with distance > DRAG_THRESHOLD
  // Assert: dragging is false, no placeholder created
});

it("starts drag when pointer is within the title bar zone", () => {
  // pointerdown at y = itemRect.top + 10 (within 24px zone)
  // pointermove with distance > DRAG_THRESHOLD
  // Assert: placeholder exists
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run packages/pages-component/src/renderer/grid-edit-controller.test.ts`

- [ ] **Step 3: Add isDragHandle to grid-drag.ts**

```typescript
const DRAG_HANDLE_HEIGHT = 24;

export function isDragHandle(
  pointerY: number,
  itemRect: DOMRect,
): boolean {
  return pointerY - itemRect.top < DRAG_HANDLE_HEIGHT;
}
```

- [ ] **Step 4: Wire into handlePointerDown**

In `grid-edit-controller.ts` `handlePointerDown`:
- After finding the target item, check `isDragHandle(e.clientY, target.rect)`
- If false AND the result from `hitTestHandle` is `"move"`, return early (don't start drag)
- If `hitTestHandle` returns a resize handle, allow it (resize handles are on edges, not restricted to title bar)

- [ ] **Step 5: Add cursor styling for drag handle zone**

In `enter()`, for each grid item element:
```typescript
el.style.cursor = "default";
```

In `handlePointerMove` (non-drag hover), update cursor based on position:
```typescript
if (isDragHandle(e.clientY, rect)) {
  el.style.cursor = "grab";
} else {
  el.style.cursor = "default";
}
```

- [ ] **Step 6: Run tests**

Run: `npx vitest run packages/pages-component/src/renderer/grid-edit-controller.test.ts packages/pages-component/src/renderer/grid-drag.test.ts`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-component/src/renderer/grid-edit-controller.ts \
       packages/pages-component/src/renderer/grid-drag.ts \
       packages/pages-component/src/renderer/grid-edit-controller.test.ts
git commit -m "feat(#618): drag handle — restrict drag to title bar zone (24px)"
```

---

## Batch 3: Collision Resolution Modes

### Task 4: `collision` property in types and rendering

**Files:**
- Modify: `packages/pages-component/src/model/types.ts` — add `collision` to `Component` props
- Modify: `packages/pages-component/src/renderer/render.ts` — read `collision` from component props
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts` — accept `collision` mode

**Interfaces:**
- Produces: `collision?: 'displace' | 'prompt'` on grid component props, threaded to `GridEditControllerImpl`

- [ ] **Step 1: Add collision property to type model**

In `types.ts`, the `collision` property lives on the component's `props` object (same as `columns`). No type change needed to `Component` itself — it's generic over `P extends object`. The `isGrid` check in `render.ts` already reads `props.columns`; add `props.collision` alongside it.

- [ ] **Step 2: Thread collision mode through to controller**

In `render.ts`, where `attachGridEditController` is called, pass `collision`:
```typescript
const collision = component.props?.collision ?? "prompt";
attachGridEditController(el, component.items, columns, rowHeight, collision);
```

Update `attachGridEditController` signature to accept `collision: 'displace' | 'prompt'`.
Store on `GridEditControllerImpl` as `private readonly collision`.

- [ ] **Step 3: Run type checks**

Run: `npx tsc --noEmit -p packages/pages-component/tsconfig.json`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add packages/pages-component/src/model/types.ts \
       packages/pages-component/src/renderer/render.ts \
       packages/pages-component/src/renderer/grid-edit-controller.ts
git commit -m "feat(#618): collision property on grid layout — displace | prompt"
```

### Task 5: Collision popup for prompt mode

**Files:**
- Create: `packages/pages-component/src/renderer/grid-collision-popup.ts`
- Create: `packages/pages-component/src/renderer/grid-collision-popup.test.ts`
- Modify: `packages/pages-component/src/renderer/grid-edit-controller.ts` — call popup on occupied-cell drop

**Interfaces:**
- Consumes: `PositionedPlacement`, `IdentifiedPlacement` from `grid-collision.ts`
- Produces:
  ```typescript
  type CollisionAction = "displace" | "swap" | "replace" | "split-h" | "split-v" | "cancel";

  function showCollisionPopup(
    anchor: HTMLElement,
    targetPlacement: PositionedPlacement,
  ): Promise<CollisionAction>

  function computeSplitH(
    target: PositionedPlacement,
    incoming: PositionedPlacement,
  ): { target: PositionedPlacement; incoming: PositionedPlacement }

  function computeSplitV(
    target: PositionedPlacement,
    incoming: PositionedPlacement,
  ): { target: PositionedPlacement; incoming: PositionedPlacement }

  function validSplitDirections(
    target: PositionedPlacement,
  ): { h: boolean; v: boolean }
  ```

- [ ] **Step 1: Write failing tests for split mechanics**

Create `grid-collision-popup.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { computeSplitH, computeSplitV, validSplitDirections } from "./grid-collision-popup.js";

describe("validSplitDirections", () => {
  it("allows both directions for a 4x2 cell", () => {
    const p = { mode: "positioned" as const, x: 0, y: 0, w: 4, h: 2 };
    expect(validSplitDirections(p)).toEqual({ h: true, v: true });
  });

  it("allows only horizontal for a 1x2 cell", () => {
    const p = { mode: "positioned" as const, x: 0, y: 0, w: 1, h: 2 };
    expect(validSplitDirections(p)).toEqual({ h: true, v: false });
  });

  it("allows only vertical for a 4x1 cell", () => {
    const p = { mode: "positioned" as const, x: 0, y: 0, w: 4, h: 1 };
    expect(validSplitDirections(p)).toEqual({ h: true, v: true });
    // h is always true because the grid can grow rows
  });

  it("allows horizontal only for a 1x1 cell", () => {
    const p = { mode: "positioned" as const, x: 0, y: 0, w: 1, h: 1 };
    expect(validSplitDirections(p)).toEqual({ h: true, v: false });
    // h always valid (grid grows rows); v invalid (w=1 can't split)
  });
});

describe("computeSplitH", () => {
  it("splits a 4x2 horizontally — top half and bottom half", () => {
    const target = { mode: "positioned" as const, x: 2, y: 1, w: 4, h: 2 };
    const incoming = { mode: "positioned" as const, x: 0, y: 0, w: 4, h: 1 };
    const result = computeSplitH(target, incoming);
    expect(result.target).toEqual({ mode: "positioned", x: 2, y: 1, w: 4, h: 1 });
    expect(result.incoming).toEqual({ mode: "positioned", x: 2, y: 2, w: 4, h: 1 });
  });

  it("splits a 6x1 horizontally — target keeps row, incoming goes below", () => {
    const target = { mode: "positioned" as const, x: 0, y: 0, w: 6, h: 1 };
    const incoming = { mode: "positioned" as const, x: 0, y: 0, w: 6, h: 1 };
    const result = computeSplitH(target, incoming);
    expect(result.target).toEqual({ mode: "positioned", x: 0, y: 0, w: 6, h: 1 });
    expect(result.incoming).toEqual({ mode: "positioned", x: 0, y: 1, w: 6, h: 1 });
  });
});

describe("computeSplitV", () => {
  it("splits a 4x2 vertically — left half and right half", () => {
    const target = { mode: "positioned" as const, x: 2, y: 1, w: 4, h: 2 };
    const incoming = { mode: "positioned" as const, x: 0, y: 0, w: 2, h: 2 };
    const result = computeSplitV(target, incoming);
    expect(result.target).toEqual({ mode: "positioned", x: 2, y: 1, w: 2, h: 2 });
    expect(result.incoming).toEqual({ mode: "positioned", x: 4, y: 1, w: 2, h: 2 });
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run packages/pages-component/src/renderer/grid-collision-popup.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement split computation functions**

Create `grid-collision-popup.ts`:

```typescript
import type { PositionedPlacement } from "./grid-collision.js";

export type CollisionAction = "displace" | "swap" | "replace" | "split-h" | "split-v" | "cancel";

export function validSplitDirections(
  target: PositionedPlacement,
): { h: boolean; v: boolean } {
  return {
    h: true,
    v: target.w > 1,
  };
}

export function computeSplitH(
  target: PositionedPlacement,
  incoming: PositionedPlacement,
): { target: PositionedPlacement; incoming: PositionedPlacement } {
  const halfH = Math.max(1, Math.floor(target.h / 2));
  return {
    target: { ...target, h: halfH },
    incoming: { mode: "positioned", x: target.x, y: target.y + halfH, w: target.w, h: Math.max(1, target.h - halfH) },
  };
}

export function computeSplitV(
  target: PositionedPlacement,
  incoming: PositionedPlacement,
): { target: PositionedPlacement; incoming: PositionedPlacement } {
  const halfW = Math.max(1, Math.floor(target.w / 2));
  return {
    target: { ...target, w: halfW },
    incoming: { mode: "positioned", x: target.x + halfW, y: target.y, w: Math.max(1, target.w - halfW), h: target.h },
  };
}

export function showCollisionPopup(
  gridEl: HTMLElement,
  anchorRect: DOMRect,
  targetPlacement: PositionedPlacement,
): Promise<CollisionAction> {
  return new Promise((resolve) => {
    const popup = gridEl.ownerDocument.createElement("div");
    popup.dataset.collisionPopup = "";
    popup.style.position = "fixed";
    popup.style.left = `${String(anchorRect.left + anchorRect.width / 2)}px`;
    popup.style.top = `${String(anchorRect.top)}px`;
    popup.style.transform = "translate(-50%, -100%)";
    popup.style.zIndex = "100";
    popup.style.padding = "8px";
    popup.style.background = "var(--pages-surface-2, #27272a)";
    popup.style.border = "1px solid var(--pages-neutral-5, #a1a1aa)";
    popup.style.borderRadius = "var(--pages-radius-md, 6px)";
    popup.style.display = "flex";
    popup.style.gap = "4px";
    popup.style.boxShadow = "var(--pages-shadow-3, 0 4px 12px rgba(0,0,0,.2))";

    const actions: { label: string; action: CollisionAction; enabled: boolean }[] = [
      { label: "Displace", action: "displace", enabled: true },
      { label: "Swap", action: "swap", enabled: true },
      { label: "Replace", action: "replace", enabled: true },
    ];

    const dirs = validSplitDirections(targetPlacement);
    if (dirs.h) actions.push({ label: "Split ↕", action: "split-h", enabled: true });
    if (dirs.v) actions.push({ label: "Split ↔", action: "split-v", enabled: true });

    for (const { label, action, enabled } of actions) {
      if (!enabled) continue;
      const btn = gridEl.ownerDocument.createElement("button");
      btn.textContent = label;
      btn.style.padding = "4px 8px";
      btn.style.fontSize = "12px";
      btn.style.cursor = "pointer";
      btn.style.border = "1px solid var(--pages-neutral-5, #a1a1aa)";
      btn.style.borderRadius = "var(--pages-radius-sm, 4px)";
      btn.style.background = "var(--pages-surface-1, #18181b)";
      btn.style.color = "var(--pages-text, #fafafa)";
      btn.addEventListener("click", () => {
        popup.remove();
        resolve(action);
      });
      popup.appendChild(btn);
    }

    const dismiss = (e: PointerEvent) => {
      if (!popup.contains(e.target as Node)) {
        popup.remove();
        gridEl.ownerDocument.removeEventListener("pointerdown", dismiss);
        resolve("cancel");
      }
    };
    gridEl.ownerDocument.addEventListener("pointerdown", dismiss);
    gridEl.ownerDocument.body.appendChild(popup);
  });
}
```

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-component/src/renderer/grid-collision-popup.test.ts`
Expected: PASS

- [ ] **Step 5: Wire popup into handlePointerUp**

In `grid-edit-controller.ts` `handlePointerUp`:

```typescript
// After computing newPlacement and resolved items:
const targetItem = items.find(i =>
  i.id !== this.dragItemId && overlaps(newPlacement, i.placement)
);

if (targetItem && this.collision === "prompt") {
  const el = this.gridEl.querySelector(`[data-component-id="${targetItem.id}"]`) as HTMLElement;
  if (el) {
    const action = await showCollisionPopup(this.gridEl, el.getBoundingClientRect(), targetItem.placement);
    // Apply action: displace (already computed), swap, replace, split-h, split-v, cancel
    if (action === "cancel") { this.cancelDrag(); return; }
    // ... apply the selected action to produce final resolvedItems
  }
}
```

Make `handlePointerUp` async.

- [ ] **Step 6: Run all tests**

Run: `npx vitest run packages/pages-component/src/renderer/`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-component/src/renderer/
git commit -m "feat(#618): collision popup with displace/swap/replace/split options"
```

---

## Batch 4: Gallery Samples and Visual Verification

### Task 6: Update gallery samples and visual test

**Files:**
- Modify: `examples/samples/Layout/Grid Builder.page.yaml` — add `collision: displace` property
- Create: `examples/samples/Layout/Grid Collision Modes.page.yaml` — new sample showing both modes
- Modify: `examples/samples/Layout/Grid Builder.yaml` — seed data for builder
- Modify: `packages/pages-component/src/renderer/grid-overlay.test.ts` — remove tests for deleted functions
- Modify: `packages/pages-component/src/renderer/grid-drag.test.ts` — remove tests for deleted functions

**Interfaces:**
- Consumes: all prior tasks

- [ ] **Step 1: Clean up test files for removed functions**

Update `grid-overlay.test.ts`: remove tests for `createGridOverlay`, `showGhost`, `hideGhost`.
Update `grid-drag.test.ts`: remove tests for `cellRectToPixel`, `cellToPixel`.

- [ ] **Step 2: Run full test suite**

Run: `npx vitest run packages/pages-component/`
Expected: PASS

- [ ] **Step 3: Update Grid Builder sample with collision mode**

In `examples/samples/Layout/Grid Builder.page.yaml`, add `collision: displace` to the grid component.

- [ ] **Step 4: Create Grid Collision Modes sample**

Create `examples/samples/Layout/Grid Collision Modes.page.yaml` with two grids side by side:
- Left grid: `collision: displace` with 4 positioned panels
- Right grid: `collision: prompt` with 4 positioned panels
- Description text explaining the two modes

- [ ] **Step 5: Rebuild examples**

Run: `npm run build --prefix examples` (or the appropriate build command)

- [ ] **Step 6: Visual verification in Playwright**

Open the gallery in headed Playwright, navigate to Grid Builder. Enter edit mode, drag a panel. Verify:
- Placeholder appears at the snap position (dashed border, semi-transparent)
- Other panels animate smoothly to new positions
- Drag only works from the title bar zone
- Drop on empty cell works without popup
- Drop on occupied cell shows popup (prompt mode) or auto-displaces (displace mode)

Take screenshot for evidence.

- [ ] **Step 7: Commit**

```bash
git add examples/ packages/pages-component/
git commit -m "feat(#618): gallery samples for collision modes, visual verification"
```

## References

- [2026-11-06-grid-edit-placeholder-drag-design.md] — design spec this plan implements
- `packages/pages-component/src/renderer/grid.ts` — `applyGridPlacement`
- `packages/pages-component/src/renderer/grid-edit-controller.ts` — current controller (436 lines)
- `packages/pages-component/src/renderer/grid-collision.ts` — current `resolveCollisions` (58 lines)
- `packages/pages-component/src/renderer/grid-overlay.ts` — ghost overlay (179 lines)
- `packages/pages-component/src/renderer/grid-drag.ts` — pixel math + snap (122 lines)
- `packages/pages-component/src/model/types.ts:20-27` — `GridPlacement` type
- `packages/pages-component/src/renderer/render.ts:104-147` — grid rendering
- [PP-20260916-b6f3e8] — all-mutations-through-applyedit protocol
- [GitHub #618] — focal issue
