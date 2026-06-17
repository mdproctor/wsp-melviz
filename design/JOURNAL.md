# Design Journal — issue-19-dashbuilder-demos

### Lazy tab rendering (#32) — 2026-06-17

Implemented lazy rendering for one-at-a-time containers (tabs, pills, sidebar, carousel, stack). Only the active slot's children are instantiated in the DOM. On tab switch, the swap function tears down the old slot and renders the new one.

Key design decisions:
- **Swap function as single owner** — each wire function builds one swap function that owns slot state, DOM lifecycle, button chrome, and event dispatch. Click handlers and `activateSlot` both delegate to it via a `slotSwapRegistry` WeakMap.
- **`slot-swap.ts` shared module** — consolidated `dispatchSlotChange` from interactive.ts and activate-slot.ts. Eliminates pre-existing duplication.
- **Registry eviction via `isConnected`** — the `casehub-slot-change` handler scans the registry and removes entries whose element is disconnected. O(1) per entry.
- **Accordion stays eager** — all sections start expanded, so lazy rendering would change UX.

### NavTree page deduplication — 2026-06-17

Discovered that `parsePage` included pages BOTH as top-level `content` slot entries AND as slot values inside navigation containers, causing N-fold DOM duplication (6 copies of each component in the Kitchensink).

Fix: `collectNavTreePageNames` extracts all page names from navTree groups. These are filtered from the root `content` slot. `patchSlotReferences` recursively replaces unresolved page references in nav slots with their resolved versions.

### Navigation type expansion (#27) — 2026-06-17

Added tree, menu, tiles as navigation types. Parser slot-target replacement now preserves the original nav component type instead of hardcoding "tabs". Renderer treats them as lazy one-at-a-time containers (same behavior as tabs — distinct visual styles are a follow-up).

### Cross-filter verification — 2026-06-17

Found that the filter flow (selector → `casehub-filter` → listening component re-query) required three fixes to work end-to-end:
1. Event listener ordering — `renderComponent` must run AFTER `addEventListener` (garden entry GE-20260617-0b0dba)
2. `@casehub/viz` side-effect import — custom elements must be registered for `connectedCallback` to fire
3. `yamlLoad` call restored in the YAML string path

Integration test verifies the full flow: navigate to lazy tab → data loads → selector filters table → navigate away destroys content.
