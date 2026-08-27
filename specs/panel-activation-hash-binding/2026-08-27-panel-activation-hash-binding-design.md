# Panel Activation & Hash Binding — Design Spec

**Date:** 2026-08-27
**Status:** Draft
**Scope:** `@casehubio/pages-runtime`, `@casehubio/pages-ui`

## 1. Overview

Workbench consumers (trellis, claudony) need two capabilities that casehub-pages does not yet expose:

1. **Programmatic dock panel activation** — call `site.activateDockPanel("backlog")` to show a specific dock panel without simulating a dock-bar button click.
2. **Hash-based panel deep linking** — `#/page/dashboard?panel=backlog,properties` activates specific dock panels on load and browser navigation.

Both features build on the existing `pages-dock-toggle` event system and the `ZoneLayoutEngine` topology. No new rendering path or component type is introduced.

### Constraints

- ZoneLayoutEngine remains pure data — no DOM coupling (PP-20260810-72779a)
- Workbench components remain content-agnostic (PP-20260810-cdcc8f)
- Inter-component events follow the pages-event contract — `pages-dock-toggle` is the existing reserved event for dock panel visibility (PP-20260705-bac842)
- The popstate/pushState separation pattern (GE-20260625-fa01da) must be preserved — DOM navigation and URL sync are separate concerns

## 2. Pre-requisite: Fix popstate dock state restoration

### Problem

`restoreFromUrl` in `site.ts` (line 349) updates the `dockState` map on popstate but never dispatches `pages-dock-toggle` events. Dock panels don't actually show or hide on browser back/forward navigation. This is a pre-existing bug that affects both the current `dock=` URL param and the proposed `panel=` param.

### Fix

After updating `dockState` entries in `restoreFromUrl`, compare the new state against the previous state and dispatch `pages-dock-toggle` for each panel whose visibility changed.

**File:** `packages/pages-runtime/src/site.ts` — `restoreFromUrl` function (line 319)

```typescript
// Current code (line 349-352):
if (link.dock) {
  for (const [id, state] of Object.entries(link.dock)) {
    dockState.set(id, state === "open");
  }
}

// Proposed:
if (link.dock) {
  for (const [id, state] of Object.entries(link.dock)) {
    const visible = state === "open";
    const wasVisible = dockState.get(id);
    dockState.set(id, visible);
    if (visible !== wasVisible) {
      target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
        bubbles: true, composed: true,
        detail: { panelId: id, visible },
      }));
    }
  }
}
```

This dispatches `pages-dock-toggle` only for changed entries, avoiding redundant DOM work. The existing dock-toggle handler in `site.ts` handles exclusive mode, cascading visibility, and dock-bar button state updates.

## 3. Feature 1: `activateDockPanel` — LiveSite method

### API

Add `activateDockPanel` to the `LiveSite` interface:

**File:** `packages/pages-runtime/src/site.ts` (line 123)

```typescript
export interface LiveSite extends Site {
  navigate(path: string): void;
  setTheme(mode: "light" | "dark"): void;
  dispose(): void;
  readonly layout: LayoutState;
  activateDockPanel(key: string): boolean;  // new
}
```

### Semantics

- **Show, not toggle.** `activateDockPanel("backlog")` always makes the panel visible. To hide, consumers dispatch `pages-dock-toggle` with `visible: false` directly (existing mechanism).
- **Exclusive mode.** The existing dock-toggle handler enforces zone-group exclusivity — showing panel B in the same zone automatically hides panel A. No new logic needed.
- **Deferred rendering.** Panels wrapped in `deferred` components render on first activation via the existing `pages-deferred-render` dispatch in the dock-toggle handler. No new logic needed.
- **URL sync.** The existing dock-toggle handler calls `syncUrl("replaceState")` after toggling. The URL updates automatically.
- **Unknown key.** Returns `false` if the key is not found in the zone engine's `zoneMap`. No error thrown — callers can check the return value.

### Implementation

A closure method inside `loadSite()`, with direct access to `zoneEngine`, `target`, and `dockState`:

**File:** `packages/pages-runtime/src/site.ts` — inside `loadSite()` closure

```typescript
function activateDockPanel(key: string): boolean {
  if (!zoneEngine || !zoneEngine.zoneMap.has(key)) return false;
  target.dispatchEvent(new CustomEvent("pages-dock-toggle", {
    bubbles: true, composed: true,
    detail: { panelId: key, visible: true },
  }));
  return true;
}
```

Add to the returned `site` object alongside `navigate`, `setTheme`, `dispose`.

### Consumer usage

```typescript
import { loadSite } from "@casehubio/pages-runtime";

const site = await loadSite(container, config, options);

// Programmatic activation
site.activateDockPanel("backlog");

// Check if panel exists
if (!site.activateDockPanel("unknown-key")) {
  console.warn("Panel not found");
}
```

### Re-export

`activateDockPanel` is a method on `LiveSite`, not a standalone export. The `LiveSite` type is already exported from `packages/pages-runtime/src/index.ts`. No new export needed — consumers access it via the `loadSite()` return value.

## 4. Feature 2: Hash panel binding

### DeepLink extension

Add `panel` field to the `DeepLink` interface:

**File:** `packages/pages-ui/src/model/page-types.ts` (line 27)

```typescript
export interface DeepLink {
  readonly page: string;
  readonly filters?: Readonly<Record<string, readonly string[]>>;
  readonly sort?: Readonly<Record<string, { readonly columnId: string; readonly order: SortOrder }>>;
  readonly pagination?: Readonly<Record<string, number>>;
  readonly textFilter?: Readonly<Record<string, string>>;
  readonly dock?: Readonly<Record<string, "open" | "closed">>;
  readonly panel?: readonly string[];  // new — activation intent
}
```

### Semantic distinction: `panel` vs `dock`

| Aspect | `dock=` | `panel=` |
|--------|---------|----------|
| Semantics | Total state declaration — all panels' open/closed | Activation intent — "show these panels" |
| Format | `dock=nav:open,files:closed,props:closed` | `panel=nav,props` |
| Resilience | Fragile to panel additions (unlisted panels have no declared state) | Additive — works even if new panels are added to the workbench |
| Use case | Internal URL state sync (generated by the runtime) | Human-authored deep links |

When both are present for the same panel key, `panel` wins — activation intent overrides saved state.

### URL serialization

**File:** `packages/pages-runtime/src/url.ts` — `serializeToUrl` (line 19)

Add after the existing `dock` serialization block:

```typescript
if (link.panel && link.panel.length > 0) {
  params.push(`panel=${link.panel.map(encodeURIComponent).join(",")}`);
}
```

### URL parsing

**File:** `packages/pages-runtime/src/url.ts` — `parseFromUrl` (line 79)

Two changes:

**4a. Handle hashes without `#/page/` prefix.** Currently the parser returns `{ page: "" }` for any hash not starting with `#/page/`. Extend it to parse query params from `#?params` format:

```typescript
export function parseFromUrl(hash: string): DeepLink {
  if (!hash) return { page: "" };

  let page = "";
  let queryStr = "";

  if (hash.startsWith("#/page/")) {
    const withoutPrefix = hash.substring("#/page/".length);
    const qIndex = withoutPrefix.indexOf("?");
    page = qIndex === -1
      ? decodePagePath(withoutPrefix)
      : decodePagePath(withoutPrefix.substring(0, qIndex));
    if (qIndex !== -1) queryStr = withoutPrefix.substring(qIndex + 1);
  } else if (hash.startsWith("#?")) {
    queryStr = hash.substring(2);
  } else {
    return { page: "" };
  }

  // ... existing query param parsing (filters, sort, pagination, textFilter, dock) ...
  // ... moved inside the `if (queryStr)` block ...

  // New: parse panel param
  let panel: string[] | undefined;
  // (inside the queryStr parsing block)
  const panelStr = params.get("panel");
  if (panelStr) {
    panel = panelStr.split(",").map(decodeURIComponent).filter(Boolean);
  }

  return {
    page,
    ...(filters ? { filters } : {}),
    ...(sort ? { sort } : {}),
    ...(pagination ? { pagination } : {}),
    ...(textFilter ? { textFilter } : {}),
    ...(dock ? { dock } : {}),
    ...(panel ? { panel } : {}),
  };
}
```

### Activation on load and popstate

**File:** `packages/pages-runtime/src/site.ts`

**Initial load** (after dock zone initialization, ~line 1297):

```typescript
if (initialLink.panel) {
  for (const key of initialLink.panel) {
    activateDockPanel(key);
  }
}
```

Panel activation runs after `initDockZoneGroup` so that the dock bars are wired and the DOM is ready. Panel entries override any dock state set by `initDockZoneGroup` for the same key.

**Popstate handler** (line 1342):

```typescript
window.addEventListener("popstate", () => {
  const link = parseFromUrl(location.hash);
  if (link.page !== currentPage) navigateInternal(link.page);
  clearPageFilters(filterState, currentPage);
  componentViewState.clear();
  restoreFromUrl(location.hash, filterState, componentViewState);
  // New: activate panels from deep link
  if (link.panel) {
    for (const key of link.panel) {
      activateDockPanel(key);
    }
  }
  pipeline.deliverAll();
}, { signal: abortController.signal });
```

Panel activation runs after `restoreFromUrl` (which handles dock state). If the same panel key appears in both `dock=` and `panel=`, the `activateDockPanel` call overrides — it always dispatches with `visible: true`.

### Multi-panel and zone exclusivity

Processing order is left-to-right in the comma-separated list. If two keys are in the same exclusive zone, the last one wins — this is inherent to exclusive dock bar semantics. Example:

```
?panel=backlog,cases
```

If both `backlog` and `cases` are in `left-top`:
1. `activateDockPanel("backlog")` → backlog shown in left-top
2. `activateDockPanel("cases")` → exclusive mode hides backlog, cases shown in left-top

This is correct behavior — the zone can only show one panel. Deep link authors should only list one panel per zone.

## 5. Files changed

| File | Change |
|------|--------|
| `packages/pages-ui/src/model/page-types.ts` | Add `panel?: readonly string[]` to `DeepLink` |
| `packages/pages-runtime/src/url.ts` | Extend `parseFromUrl` for `#?` prefix and `panel` param; extend `serializeToUrl` for `panel` |
| `packages/pages-runtime/src/site.ts` | Add `activateDockPanel` to `LiveSite` interface and `loadSite()` closure; fix `restoreFromUrl` dock dispatch; add panel activation to initial load and popstate handler |

No new files. No new packages. No new component types.

## 6. Testing strategy

### Unit tests

- `url.test.ts`: `parseFromUrl` handles `#/page/dashboard?panel=backlog`, `#/page/dashboard?panel=backlog,properties`, `#?panel=backlog` (panel-only), `#/page/dashboard?dock=nav:open&panel=backlog` (both params), empty `panel=`, unknown keys in panel list
- `url.test.ts`: `serializeToUrl` round-trips panel entries
- `zone-layout-engine.test.ts`: No changes needed — engine stays pure data

### Integration tests (Playwright)

- `activateDockPanel` shows the panel and hides others in the same zone
- `activateDockPanel` with unknown key returns false and doesn't throw
- Deep link with `panel=backlog` opens the backlog panel on page load
- Deep link with `panel=backlog,properties` opens panels in different zones
- Browser back/forward correctly toggles dock panels (pre-requisite bug fix)
- `panel=` overrides `dock=` for the same key
- Panel-only deep link `#?panel=backlog` works (no page path)

## References

- `packages/pages-runtime/src/site.ts` — LiveSite interface (line 123), loadSite closure, dock-toggle handler, restoreFromUrl (line 319), initDockZoneGroup (line 1252), popstate handler (line 1342)
- `packages/pages-runtime/src/url.ts` — DeepLink (defined in page-types.ts), serializeToUrl (line 19), parseFromUrl (line 79)
- `packages/pages-runtime/src/zone-layout-engine.ts` — ZoneLayoutEngine interface, zoneMap
- `packages/pages-ui/src/model/page-types.ts` — DeepLink interface (line 27)
- PP-20260810-72779a — workbench integration pattern (extend existing packages)
- PP-20260810-cdcc8f — content-agnostic workbench
- PP-20260705-bac842 — pages-event contract (pages-dock-toggle reserved)
- GE-20260625-fa01da — pushState/popstate separation pattern
- GE-20260809-aee002 — resolveDockZone zone naming
- GE-20260804-befd45 — dockWorkbench decomposition into 3 primitives
