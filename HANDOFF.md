# HANDOFF — casehub-pages

## Last Session: 2026-11-19

### What was done

**Dual-mode datasets (#719):** Built `fallback-source.ts` — a DataSource combinator that tries a primary source (REST), falls back to a secondary (inline) on error. Updated the data pipeline to allow `url + content` together on dataset definitions. All app samples (helpdesk, contacts, tasks) now carry both — one YAML works in gallery (content fallback) and with backend (REST data). Removed apps-management and platform from Client tab (server-only via "Apps Within Apps").

**Playbook split-view editor (#720):** Built the web component foundation:
- `StepPositionScanner` enhanced for hierarchical nesting (`"1.then.0"`, `"2.catch.1"`) with `findByLine`/`findByAddress` helpers
- `EditableTextBridge.onCursorChange()` callback added
- Step viewer click events with `data-step-address` attributes and `step-click` CustomEvent
- `<playbook-split-view>` container with bidirectional sync, gallery demo

**Form pages fleshed out:** Helpdesk, contacts, tasks, apps-management — all empty grid stubs replaced with working `components:` form fields and `schema-form:` create forms.

**Apps-demo catalogue fix:** Converted from unsupported `rows:` layout to `grid:` format.

### What's next

**Queue (4 items):**

| # | Title | Scale | Notes |
|---|-------|-------|-------|
| #722 | Match step address mismatch | S | Scanner vs step-click use different address formats for match arms |
| #723 | Extract demoCatalog | XS | DRY — duplicated between split-view and gallery |
| #724 | Parallel lane address inconsistency | S | Same mismatch as #722 but for parallel lanes |
| #725 | IntelliJ JCEF preview | M | FileEditorProvider with JCEF step viewer — plan exists at `plans/2026-11-19-playbook-split-view.md` (Batch 4) |

### Key files

- `packages/pages-data/src/datasource/sources/fallback-source.ts` — dual-mode combinator
- `packages/yaml-core/src/annotation/step-position-scanner.ts` — hierarchical position scanner
- `packages/pages-step-viewer/src/playbook-split-view.ts` — split-view container
- `specs/wip-helpdesk-form-pages/2026-11-19-playbook-split-view-design.md` — design spec
- `plans/2026-11-19-playbook-split-view.md` — implementation plan (Batch 4 remaining)
