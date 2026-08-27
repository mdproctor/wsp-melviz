# Decisions — Pipeline Examples for Graph Editing Showcase

## D1: Example domain

**Choice:** Data Pipeline Editor (Source, Transform, Filter, Join, Sink)
**Alternatives:**
- State machine (states + transitions) — simpler but fewer constraint types
- Network topology (server, router, switch) — less natural property schemas
**Rationale:** Natural connection constraints (Source: outbound only, Sink: inbound only), containment-free, diverse property schemas, immediately understandable.
**Trade-offs:** No containment/cascade demo — but that's a domain-specific concern for blocks-ui.
**Sources:** User direction
**Exploration:** quick
**Status:** captured

## D2: Stencil visual design

**Choice:** Colored rounded-rects using pages design tokens, distinct icon character per type
**Alternatives:**
- SVG shapes (diamond for filter, circle for join) — more visual variety but harder to implement
- Plain text labels only — too minimal for a showcase
**Rationale:** Consistent shape keeps focus on the editing interactions. Token colors provide automatic theme support.
**Trade-offs:** Less visual variety than mixed shapes.
**Sources:** pages-ui-tokens colour scale (steps 9 for bg, 12 for text)
**Exploration:** quick
**Status:** captured

## D3: Example structure

**Choice:** 3 examples — Basic Pipeline (static), Interactive Editing (mutations), Property Palette (selection + schema forms)
**Alternatives:**
- Single comprehensive demo — too much for one page
- 5+ granular examples — over-segmented for a showcase
**Rationale:** Each example has a clear teaching purpose. Progressive complexity (view → edit → edit+properties).
**Trade-offs:** More files to maintain.
**Sources:** Existing gallery patterns (Tables, Forms categories)
**Exploration:** quick
**Status:** captured

## D4: Technical approach for gallery integration

**Choice:** Side-effect import in casehub-entry.ts registers stencils at bundle load; YAML uses `type: html` for canvas element; TS companion configures model and wiring
**Alternatives:**
- Custom element per demo — encapsulates everything but hides the extension point pattern
- Webpack entry per example — too heavyweight
**Rationale:** Matches existing gallery pattern (Schema Form, Spanning). Stencil registration must happen in the bundle (lit-html templates need compilation). Companion scripts handle model setup which is plain data.
**Trade-offs:** Companion scripts can't use imports at runtime — all graph functions must be on the casehubPages global.
**Sources:** examples/src/app.js (stripTs + new Function), examples/src/casehub-entry.ts (UMD library)
**Exploration:** quick
**Status:** captured
