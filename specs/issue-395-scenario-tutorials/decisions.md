## D1: Tutorial registry and aggregation model

**Choice:** Static build-time registry
**Alternatives:**
- Runtime discovery — no build step but multiple network requests, manual index can drift, harder cross-repo aggregation
- ScriptRegistry integration — reuses existing infra but conflates tutorial and script concepts, library browser UX doesn't fit educational content
**Rationale:** Build-time scan of tutorial directories emits tutorial-registry.json. Catalog component loads this JSON array. For casehub/examples aggregation, merge N registry JSONs into one. Simple data contract, fast runtime, works offline, easy validation.
**Trade-offs:** Requires a build step. Adding a tutorial means rebuilding. Acceptable given the existing yarn build pipeline.
**Sources:** Script library spec (ScriptRegistry pattern), existing examples/ build pipeline
**Exploration:** quick
**Status:** captured

## D2: Tutorial authoring format

**Choice:** Structured scenario YAML with sections + optional separate .md content
**Alternatives:**
- Separate markdown + scenario files with manifest — clean separation but more files, more coordination
- Tutorial-specific YAML schema — purpose-built but new schema to design and maintain
**Rationale:** Tutorials are scenarios. Extend the scenario format with a sections array (already defined in the executor protocol spec) and enhanced meta block. Narrative content via show-markdown steps — inline or file-referenced, author's choice. Reuses existing scenario infrastructure.
**Trade-offs:** Requires extending the client-side parser to support sections. Flat step list no longer sufficient for tutorials.
**Sources:** Executor protocol spec §2.1 (three-layer model), existing parser.ts, show-markdown step action
**Exploration:** quick
**Status:** captured

## D3: Tutorial catalog display modes

**Choice:** Two modes — tiled landing view + compact filterable list
**Alternatives:**
- Single list view — simpler but loses the hierarchical area browsing
- Full page per area — heavier, more pages to maintain
**Rationale:** Tiled landing shows areas at root, drill into an area to see tutorials as cards. Compact list provides flat filterable browsing across all tutorials. Both modes share the same registry data.
**Trade-offs:** Two rendering modes to implement and maintain.
**Sources:** Script library's PagesLibraryView (compact list pattern)
**Exploration:** quick
**Status:** captured

## D4: Learning paths — separate manifests

**Choice:** Separate path manifest YAML files
**Alternatives:**
- Inline path metadata in tutorial meta — editing path order means touching every tutorial file
- Defer paths to follow-up — delays a key navigation feature
**Rationale:** A paths/ directory with YAML files listing tutorials in recommended order. Adding or reordering a path doesn't touch tutorial files. Paths can span areas and produce themed/task-based subsets.
**Trade-offs:** Extra files to maintain. Path references can go stale if tutorials are renamed.
**Sources:** User requirement for recommended ordering by theme or task
**Exploration:** quick
**Status:** captured

## D5: Diagram delivery for Tutorial 0

**Choice:** Inline SVG in markdown
**Alternatives:**
- Separate SVG files referenced by img — can't use CSS variables for theming
- Programmatic diagrams — heaviest to build, most flexible
**Rationale:** SVG embedded directly in narrative .md files. Uses CSS variables for light/dark theme adaptation. Self-contained, no external assets to manage.
**Trade-offs:** Larger markdown files. SVG authoring is more manual than programmatic approaches.
**Sources:** Issue #395 (4 SVG diagrams specified), ARIA interaction contract protocol (theming via CSS variables)
**Exploration:** quick
**Status:** captured

## D6: Labeling dimensions

**Choice:** Typed labels (concept, difficulty, area) + free-form tags
**Alternatives:**
- Free-form tags only — simpler but less structured filtering
- Fixed taxonomy — rigid, hard to extend
**Rationale:** Matches the script library's label pattern. concept:aria, concept:params, difficulty:beginner, area:scenario-automation. Free-form tags for ad-hoc grouping. Supports cross-referencing by skill level, technique, or capability.
**Trade-offs:** Label vocabulary needs governance to stay consistent across tutorial sources.
**Sources:** Script library spec §2.1 (labels and tags), user requirement for multi-dimensional filtering
**Exploration:** quick
**Status:** captured
