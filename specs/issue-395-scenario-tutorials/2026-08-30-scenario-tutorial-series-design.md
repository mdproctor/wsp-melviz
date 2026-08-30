# Scenario Automation Tutorial Series — Architecture & Infrastructure

> **Issue:** casehubio/casehub-pages#395
> **Date:** 2026-08-30
> **Status:** Draft

## 1. Overview

Build a tutorial system that teaches the scenario automation platform
concepts progressively. Tutorials use the scenario controller's
slide/narrative and YAML viewer capabilities to explain what's happening
at each step.

### 1.1 Scope (this issue)

**Infrastructure:**
- Tutorial descriptor schema and registry (build-time JSON)
- Catalog component with tiled landing and compact list modes
- Learning path manifests
- Client-side parser extension for sectioned scenarios
- Narrative renderer `allowHtml` for inline SVG

**Content:**
- Tutorial 0: Architecture & Concepts (slides-only, no executable code)
- Tutorial 1: Form Automation Basics (hands-on with executable steps)

Tutorials 2–5 (Parameterized Scripts, Data-Driven Table Entry, Script
Composability, Script Library) are follow-up issues — the infrastructure
is designed to support them as content authoring.

### 1.2 Constraints

- Tutorials must work browser-only (no server required) using the
  client-side runner
- The catalog and navigation must work standalone in pages AND when
  aggregated into `casehub/examples` alongside other tutorial families
- Each tutorial builds on concepts from the previous one
- Pre-release — breaking changes acceptable

### 1.3 Key decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Static build-time registry | Simple data contract, fast runtime, easy cross-repo aggregation |
| D2 | Structured scenario YAML with sections | Tutorials are scenarios; reuses existing infrastructure |
| D3 | Two catalog modes (tiled + compact) | Hierarchical browsing + flat filtered search |
| D4 | Separate learning path manifests | Paths evolve without touching tutorials |
| D5 | Inline SVG in markdown | Theme-adaptive via CSS variables, self-contained |
| D6 | Typed labels + free-form tags | Structured filtering by concept, difficulty, area |
| D7 | Catalog component in pages-aria | Same package as scenario controller/library |
| D8 | Infrastructure + Tutorials 0-1 | Validates both slides-only and hands-on modes |

## 2. Tutorial Descriptor Schema

### 2.1 Tutorial YAML format

A tutorial is a scenario YAML with an extended `meta` block and a
`sections` array (replacing the flat `steps` array for structured
tutorials):

```yaml
scenario: form-automation-tutorial
meta:
  title: "Form Automation Basics"
  description: "Learn to automate form entry with ARIA targeting"
  area: scenario-automation
  labels:
    - concept:aria
    - concept:commands
    - difficulty:beginner
  tags:
    - getting-started
    - forms
  estimated: "10 min"
  prerequisites:
    - architecture-concepts
  hero:
    title: "Form Automation Basics"
    subtitle: "Your first steps with ARIA-based browser automation"
    icon: "✎"

sections:
  - title: "What is ARIA targeting?"
    content:
      type: template
      path: content/aria-targeting.md
    steps: []

  - title: "Your first fill command"
    content:
      type: inline
      markdown: |
        The `fill` command targets a form field by its ARIA role and
        accessible name, then types a value into it.
    steps:
      - fill:
          role: textbox
          name: "Full Name"
          value: "Alice Chen"

  - title: "Selecting from dropdowns"
    content:
      type: template
      path: content/select-command.md
    steps:
      - select:
          role: listbox
          name: "Department"
          value: "Engineering"
```

### 2.2 Meta block fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | yes | Display title |
| `description` | string | yes | One-line summary for catalog cards |
| `area` | string | yes | Grouping area (e.g., `scenario-automation`) |
| `labels` | string[] | no | Typed labels: `concept:*`, `difficulty:*`, `capability:*` |
| `tags` | string[] | no | Free-form tags |
| `estimated` | string | no | Estimated duration (e.g., "10 min") |
| `prerequisites` | string[] | no | Tutorial scenario names that should be completed first |
| `hero` | object | no | Hero card content (title, subtitle, icon) |

### 2.3 Section structure

Each section has:
- `title` — displayed in the controller outline as a navigation point
- `content` — narrative content, either inline markdown or file reference
  (same format as existing `show-markdown` step)
- `steps` — optional array of executable scenario steps (same format as
  existing flat steps). Empty array for slides-only sections.

Content is rendered by `PagesScenarioNarrative`. Steps are executed by
the scenario runner. The controller outline shows section titles as
clickable navigation points.

### 2.4 Backward compatibility

The parser supports both formats:
- **Flat format** (`scenario` + `steps`): existing scenarios, unchanged
- **Sectioned format** (`scenario` + `sections`): tutorials and
  structured demos

If both `steps` and `sections` are present, `sections` takes precedence
and `steps` is ignored. The `meta` block is always optional — existing
scenarios without `meta` continue to work.

## 3. Tutorial Registry

### 3.1 Build-time scan

A build script scans the `tutorials/` directory, reads each
`tutorial.yaml`, extracts the `meta` block, and emits
`tutorial-registry.json`:

```json
[
  {
    "scenario": "architecture-concepts",
    "title": "Architecture & Concepts",
    "description": "How the scenario engine works — orchestrator, executors, ARIA targeting",
    "area": "scenario-automation",
    "labels": ["concept:architecture", "difficulty:beginner"],
    "tags": ["overview"],
    "estimated": "15 min",
    "prerequisites": [],
    "hero": {
      "title": "Architecture & Concepts",
      "subtitle": "Understanding the automation platform",
      "icon": "◎"
    },
    "path": "tutorials/architecture-concepts/tutorial.yaml",
    "contentType": "slides-only"
  }
]
```

### 3.2 Registry fields

All `meta` block fields plus:
- `scenario` — scenario name from the YAML
- `path` — relative path to the tutorial YAML
- `contentType` — `slides-only` (no executable steps) or `hands-on`
  (has executable steps), derived from section analysis

### 3.3 Aggregation contract

For `casehub/examples` integration:
1. Each tutorial source (pages, helpdesk, etc.) produces its own
   `tutorial-registry.json` as part of its build
2. `casehub/examples` imports these JSON files and merges them with
   a simple array concat script (no conflict resolution — scenario
   names must be globally unique across sources)
3. Each source's registry entries include a `basePath` field (the
   directory root relative to the aggregated output). The aggregator
   prepends this to each entry's `path` field so content resolution
   works from the aggregated root
4. The catalog component receives the merged array — it is
   source-agnostic. It resolves tutorial content paths relative to
   the page's base URL

### 3.4 Build-time validation

The registry build step validates:
- All required meta fields are present
- Labels follow the typed format (`namespace:value`)
- Prerequisites reference valid scenario names within the registry
- Learning path references (§5) point to valid tutorials

## 4. Catalog Component

### 4.1 `<pages-tutorial-catalog>`

A Lit web component in `packages/pages-aria/src/controller/` that
renders the tutorial catalog. Accepts a `TutorialDescriptor[]` array
and supports two display modes.

**Properties:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `registry` | `TutorialDescriptor[]` | `[]` | Tutorial descriptors from registry JSON |
| `paths` | `LearningPath[]` | `[]` | Learning path definitions |
| `mode` | `'tiles' \| 'list'` | `'tiles'` | Display mode |
| `area` | `string \| null` | `null` | Filter to a specific area (drill-down) |
| `labels` | `string[]` | `[]` | Active label filters |
| `activeTutorial` | `string \| null` | `null` | Currently selected tutorial scenario name |

**Events:**

| Event | Detail | Description |
|-------|--------|-------------|
| `tutorial-select` | `{ scenario: string }` | User selected a tutorial |
| `area-select` | `{ area: string }` | User drilled into an area |
| `path-select` | `{ path: string }` | User selected a learning path |

### 4.2 Tiled landing view (`mode='tiles'`)

**Root level (no area filter):**
Cards grouped by area. Each area card shows:
- Area name
- Tutorial count
- Label chips (aggregated from tutorials in the area)
- Click to drill into the area

**Area level (area filter set):**
Tutorial cards within the selected area:
- Hero icon + title + subtitle
- Difficulty chip (beginner/intermediate/advanced)
- Estimated duration
- Concept label chips
- Prerequisites indicator (if any)
- Click to select and launch the tutorial

**Breadcrumb navigation:** `All Tutorials > [Area Name]`

### 4.3 Compact list view (`mode='list'`)

Flat filterable list across all tutorials (all areas):
- Each row: title, area badge, difficulty chip, concept labels, duration
- Filter bar at top: label chips (click to toggle), text search
- Sort: by area, by difficulty, alphabetical
- Same click-to-select behavior as tiles

### 4.4 Mode toggle

A segmented control in the catalog header switches between tiles and
list modes. The active mode is stored in local storage so it persists
across sessions. URL hash parameter `#mode=list` allows direct linking.

### 4.5 Styling

Uses design tokens throughout (`--pages-*` CSS custom properties).
Light/dark theme via token values. Card styles follow the existing
pattern from `PagesLibraryView`.

## 5. Learning Paths

### 5.1 Path manifest format

```yaml
# tutorials/paths/fundamentals.yaml
path: scenario-fundamentals
title: "Scenario Engine Fundamentals"
description: "Complete introduction to the automation platform"
labels:
  - difficulty:beginner
tutorials:
  - architecture-concepts
  - form-automation-tutorial
  - parameterized-scripts
  - data-driven-table-entry
  - script-composability
  - script-library-registries
```

### 5.2 Path data model

```typescript
interface LearningPath {
  path: string;
  title: string;
  description: string;
  labels: string[];
  tutorials: string[];   // scenario names in order
}
```

### 5.3 Path navigation

When a user selects a learning path:
- The catalog switches to path mode: ordered list with prev/next
  navigation
- Completed tutorials are marked (via local storage)
- Progress is shown (3/6 complete)
- The user can exit the path and return to browse mode at any time

### 5.4 Multiple paths

A tutorial can appear in multiple paths. Paths can span areas.
Example paths:
- "Scenario Engine Fundamentals" — all tutorials in order
- "Task Automation" — Tutorials 1, 2, 3, 4 (skip architecture)
- "Platform Admin" — Tutorials 0, 5 (architecture + registry)

## 6. Client-Side Parser Extension

### 6.1 Sectioned format support

Extend `parseScenario()` in `packages/pages-aria/src/scenario/parser.ts`
to handle the `sections` array:

```typescript
export interface TutorialMeta {
  title: string;
  description: string;
  area: string;
  labels?: string[];
  tags?: string[];
  estimated?: string;
  prerequisites?: string[];
  hero?: { title: string; subtitle?: string; icon?: string };
}

export interface SectionContent {
  type: 'inline' | 'template';
  markdown?: string;
  path?: string;
  section?: string;
}

export interface TutorialSection {
  title: string;
  content?: SectionContent;
  steps: ScenarioStep[];
}

export interface SectionedScenario extends Scenario {
  meta?: TutorialMeta;
  sections: TutorialSection[];
}
```

The parser detects `sections` vs `steps` at the top level and returns
the appropriate type. The `Scenario` type gains an optional `meta`
field. `SectionedScenario` extends it with `sections`.

### 6.2 Scenario runner extension

The runner needs to handle sectioned scenarios. In browser-only mode
(no server), the runner dispatches events on the shared `EventTarget`
that the controller and narrative component already listen to:

1. Build an outline from sections: `OutlineNode[]` with section titles
   as top-level nodes and steps nested under each
2. Fire `pages-event` with `op: 'event', topic: 'scenario:state'` on
   the `eventTarget` — the same format the controller consumes from
   push wire. Include `scenario`, `section` (title), `step` (name),
   `paused`, `progress`, and `content` (the section's narrative)
3. For each section: fire a state update with the section's content,
   then iterate its steps, firing state updates per step
4. Between sections: pause (if in step mode) or continue (if playing)
5. Fire `scenario-narrative` custom event with the section's `content`
   payload — the narrative component listens for this directly

### 6.3 Controller outline integration

The scenario controller's outline tree renders section titles as
navigation points. Each section's steps are nested under it. The
`runTo` feature works at the section level — clicking a section
title in the outline runs to that section.

## 7. Narrative Renderer Enhancement

### 7.1 `allowHtml` property

Add a boolean property to `PagesScenarioNarrative`:

```typescript
@property({ type: Boolean }) allowHtml = false;
```

When `true`, skip the HTML escaping step in `_renderMarkdown()`.
This enables inline SVG and other HTML in markdown content.

Default is `false` — existing behavior preserved. Tutorials set
`allowHtml` on the narrative component they mount.

**Security note:** This is for trusted authored content only
(tutorials, specs). User-generated content must never use this mode.

### 7.2 SVG theming

Inline SVGs use CSS custom properties for colors:

```svg
<svg viewBox="0 0 400 200" xmlns="http://www.w3.org/2000/svg">
  <rect fill="var(--pages-neutral-3)" stroke="var(--pages-accent-7)" .../>
  <text fill="var(--pages-neutral-12)" ...>Orchestrator</text>
</svg>
```

This adapts automatically to light/dark themes without JS.

## 8. Directory Structure

```
tutorials/
  architecture-concepts/
    tutorial.yaml           # slides-only scenario YAML
    content/
      intro.md              # narrative with inline SVG
      orchestrator.md
      compilation.md
      aria-targeting.md
      registry.md
  form-automation/
    tutorial.yaml           # hands-on scenario YAML with sections
    content/
      intro.md
      fill-explained.md
      select-explained.md
      recap.md
  paths/
    fundamentals.yaml       # learning path manifest
```

Build output:
```
dist/
  tutorial-registry.json    # build-time generated
```

## 9. Tutorial 0: Architecture & Concepts

Slides-only tutorial introducing the automation platform. No executable
steps — pure narrative with SVG architecture diagrams.

### 9.1 Sections

1. **What is the scenario engine?** — An automation platform, not just
   a demo tool. Operational workflows, onboarding, data imports.
2. **Distributed executor architecture** — Orchestrator ↔ browser
   executor ↔ service executors. Star topology via push wire.
   SVG: orchestrator topology diagram.
3. **Push wire protocol** — How commands flow from orchestrator to
   executors and results flow back. Event-based, not request/response.
4. **ARIA-based targeting** — Role + name + index + within scoping.
   How the browser executor finds elements. Same coordinate system as
   screen readers. SVG: DOM tree with role/name/index matching.
5. **Compilation pipeline** — YAML → params → forEach → when → call
   resolution → execution plan. SVG: pipeline stages diagram.
6. **Script library** — Bundled, uploaded, external registry sources.
   SVG: script registry architecture (3 source types aggregating).
7. **Readiness probes** — How the browser checks if a script can run
   on the current page. First-step ARIA target detection.

### 9.2 Meta

```yaml
scenario: architecture-concepts
meta:
  title: "Architecture & Concepts"
  description: "How the scenario engine works — orchestrator, executors, ARIA targeting, compilation"
  area: scenario-automation
  labels:
    - concept:architecture
    - concept:executor-protocol
    - concept:aria
    - concept:compilation
    - difficulty:beginner
  tags:
    - overview
    - getting-started
  estimated: "15 min"
  prerequisites: []
  hero:
    title: "Architecture & Concepts"
    subtitle: "Understanding the automation platform before writing your first script"
    icon: "◎"
```

## 10. Tutorial 1: Form Automation Basics

First hands-on tutorial. Users step through a scenario that fills a
form, with the YAML viewer showing each command as it executes.

### 10.1 Target application

An embedded form (inline HTML component, same pattern as existing
Form Automation example): New Team Member form with Full Name, Email,
Department (select), Role fields, and Submit/Reset buttons. All
elements have ARIA roles and accessible names.

### 10.2 Sections

1. **Introduction** — What you'll learn. The form is visible, the
   controller is in compact overlay mode.
2. **Scenario YAML structure** — `scenario`, `steps`, `commands`.
   Show the tutorial's own YAML source in the YAML viewer fly-out.
3. **ARIA commands: fill** — Explain `fill` targeting by `{role, name}`.
   Execute a fill step on the Full Name field. Visual feedback
   highlights the field.
4. **ARIA commands: select** — Explain `select` for dropdowns.
   Execute a select step on the Department field.
5. **ARIA commands: click** — Explain `click` for buttons. Execute
   a click on Submit.
6. **The controller UI** — Explain the outline tree, transport
   controls, step-by-step execution, speed slider.
7. **Recap** — Summary of concepts covered. Link to Tutorial 2.

### 10.3 Meta

```yaml
scenario: form-automation-tutorial
meta:
  title: "Form Automation Basics"
  description: "Automate form entry with fill, select, and click commands"
  area: scenario-automation
  labels:
    - concept:aria
    - concept:commands
    - concept:fill
    - concept:select
    - concept:click
    - difficulty:beginner
  tags:
    - getting-started
    - forms
    - hands-on
  estimated: "10 min"
  prerequisites:
    - architecture-concepts
  hero:
    title: "Form Automation Basics"
    subtitle: "Your first steps with ARIA-based browser automation"
    icon: "✎"
```

## 11. Integration with casehub/examples

### 11.1 Aggregation model

`casehub/examples` maintains its own `tutorials/` directory and build
step. It copies or references tutorial content from source repos
(pages, helpdesk, etc.) and merges their `tutorial-registry.json`
files into a combined registry.

The `<pages-tutorial-catalog>` component is imported from
`@casehubio/pages-aria` and receives the merged registry. No code
changes needed in the catalog component — it is source-agnostic.

### 11.2 Cross-repo learning paths

Learning paths in `casehub/examples` can reference tutorials from any
source repo. The path manifest uses scenario names as keys; the build
step validates that all referenced tutorials exist in the merged
registry.

### 11.3 Filtering across sources

Label dimensions are shared across all tutorial sources:
- `concept:*` — technical concept being taught
- `difficulty:*` — beginner / intermediate / advanced
- `capability:*` — platform capability demonstrated
- `area:*` — inherited from each tutorial's area field

The catalog's filter bar works across the aggregate — a user can filter
by `difficulty:beginner` and see tutorials from all sources.

## References

- Executor protocol spec: `docs/specs/issue-408-scenario-engine/2026-08-20-distributed-executor-protocol-design.md`
- Controller UI spec: `docs/specs/issue-408-scenario-engine/2026-08-21-scenario-controller-ui-design.md`
- Compact overlay spec: `docs/specs/issue-408-scenario-engine/2026-08-22-controller-compact-overlay-design.md`
- YAML viewer spec: `docs/specs/issue-408-scenario-engine/2026-08-23-yaml-flyout-viewer-design.md`
- Visual feedback spec: `docs/specs/issue-408-scenario-engine/2026-08-23-visual-feedback-design.md`
- Script library spec: `docs/specs/script-library/2026-08-29-script-library-automation-platform-design.md`
- ARIA interaction contract: `docs/protocols/casehub/aria-interaction-contract.md`
- Existing scenario parser: `packages/pages-aria/src/scenario/parser.ts`
- Existing scenario runner: `packages/pages-aria/src/scenario/runner.ts`
- Existing narrative component: `packages/pages-aria/src/controller/scenario-narrative.ts`
- Existing library view: `packages/pages-aria/src/controller/library-view.ts`
- Existing examples: `examples/samples/Scenario Automation/`
