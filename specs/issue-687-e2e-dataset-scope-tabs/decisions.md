## D1: Virtual page padding default

**Choice:** Virtual pages set `style: { padding: "0" }` in desugar
**Alternatives:**
- Distinguish virtual vs real pages in render.ts via a marker prop — cleaner separation but adds a new concept to the Component model for one CSS property; YAGNI
- Leave as-is and document — 16px padding is visually wrong for edge-to-edge content like data tables
**Rationale:** The desugar already creates virtual pages and knows they're synthetic. Setting `style: { padding: "0" }` uses the existing style-override mechanism in render.ts (component.style runs after page defaults). No new concepts, no new syntax, minimal change.
**Trade-offs:** If someone wants padding on a virtual page, there's no YAML syntax for it — they'd need to add padding to their content component's style. This is unlikely since the tab container already provides the visual boundary.
**Sources:** component-desugar.ts:460-466, render.ts:60-80, types.ts:8 (Component.style), render.test.ts:598 (style override test)
**Exploration:** quick
**Status:** captured
