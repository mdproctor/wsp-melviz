## D1: Layout pipeline extensibility approach

**Choice:** Two hooks — `_computeLayout` and `_postLayout` — replacing the single hardcoded `computeElkLayout` call in `_fullRender`
**Alternatives:**
- `_computeLayout` only — SWF is fully served but org still needs `_fullRender` override for post-layout processing (engine.postLayout, edge styling)
- Override-friendly `_fullRender` (make fields `protected` instead of fixing the pipeline) — doesn't reduce duplication, just removes the `(this as any)` casts
**Rationale:** Both SWF and org override the entire `_fullRender` (~65 lines each, 18 total `(this as any)` casts) just to swap the layout call and add post-processing. Two focused hooks eliminate both overrides entirely. The pipeline becomes: `_adaptYaml → _computeLayout → toReactFlowGraph → _postLayout → store`
**Trade-offs:** Two hooks instead of one is slightly more API surface, but `_postLayout` is a no-op default so the cost is near-zero for diagrams that don't need it
**Sources:** SWF `_fullRender` override (swf-diagram.ts:145-169), org `_fullRender` override (blocks-org-diagram.ts:266-330)
**Exploration:** quick
**Status:** captured

## D2: `_computeLayout` return type

**Choice:** Return `LayoutResult { layout: ElkLayoutResult; direction?: Direction }` instead of bare `ElkLayoutResult`
**Alternatives:**
- Return `ElkLayoutResult` only — `_fullRender` reads direction from `_layoutOptions()`, but org's fallback chain means the effective direction differs from what `_layoutOptions()` returns
- Store effective direction as a field set by `_computeLayout` — works but implicit coupling between a return value and a side-effect
**Rationale:** Org diagram's auto-detection tries radial (no direction), then ELK layered (DOWN), then force fallback. The effective direction depends on which strategy succeeded, which only `_computeLayout` knows. Returning it in the result keeps the contract explicit.
**Trade-offs:** Slightly richer return type; default implementation trivially wraps `{ layout, direction: options.direction }`
**Sources:** org-diagram auto-detection (blocks-org-diagram.ts:280-302), `toReactFlowGraph` direction parameter usage
**Exploration:** quick
**Status:** captured

## D3: Template approach — composable methods, not monolithic shell

**Choice:** Composable protected template methods (`_handleCanvasEvent`, `_renderCanvas`, `_renderDialogs`) that each diagram's `render()` composes freely
**Alternatives:**
- Monolithic `_renderDiagramShell()` with slot-like extension points — more deduplication but org-diagram can't use it (different canvas component, bottom panels, tooltips)
- No template methods — only `_handleCanvasEvent` event dispatcher — misses the canvas rendering deduplication
**Rationale:** The three diagrams share structural patterns (toolbar/palette/canvas/properties) but diverge too much in detail for a single shell template. Org uses `graph-canvas-core` instead of `pages-graph-canvas`, adds bottom panels and tooltips. Composable methods let each diagram take what it needs.
**Trade-offs:** Less deduplication than a monolithic shell — each diagram still has its own layout structure. But no diagram is forced to override or work around a rigid template.
**Sources:** casehub-diagram render (casehub-diagram.ts:927-1007), swf-diagram render (swf-diagram.ts:328-405), org-diagram render (blocks-org-diagram.ts:501-636)
**Exploration:** quick
**Status:** captured

## D4: Event dispatcher design — switch with super delegation

**Choice:** `_handleCanvasEvent` uses switch/case for common events; subclasses call `super._handleCanvasEvent(e)` then handle domain-specific events
**Alternatives:**
- Map-based dispatch (topic → handler map) — more flexible but harder to override individual topics
- Individual listener methods bound in render — current pattern; requires copy-pasting the inline handler across every diagram
**Rationale:** switch/case is readable, covers the 5 common events (`graph:node:click`, `graph:edge:click`, `graph:selection:change`, `graph:pane:click`, `graph:connect:end-on-empty`), and super-call delegation is the standard Lit mixin pattern. New events (e.g. `graph:edge:mouseenter`) are added once in the mixin and immediately available to all diagrams.
**Trade-offs:** Slightly less flexible than a map for dynamic registration, but diagrams don't need dynamic event registration — their events are known at class definition time
**Sources:** pages-event-contract protocol (colon-separated topics), GraphCanvas.ts event emissions (18 topics)
**Exploration:** quick
**Status:** captured
