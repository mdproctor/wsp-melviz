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
| D7 | Catalog in pages-aria tutorial module | Source-agnostic; decoupled from controller/execution |
| D8 | Infrastructure + Tutorials 0-1 | Validates both slides-only and hands-on modes |

## 2. Tutorial Descriptor Schema

### 2.1 Tutorial YAML format

A tutorial is a scenario YAML with an extended `meta` block and a
`sections` array (replacing the flat `steps` array for structured
tutorials).

**Format relationship:** Tutorial YAML uses the **client-side ARIA
shorthand format** — the same `fill: {role, name, value}` syntax used
by existing flat scenarios and parsed by `parseScenario()` in
`packages/pages-aria/src/scenario/parser.ts`. This is distinct from the
backend hierarchical format (chapters → sections → steps → commands with
explicit `action/target/value` fields) parsed by `HierarchicalParser` in
`backend/scenario`. The sectioned extension here extends the client-side
parser only. The backend parser already supports its own `sections`
keyword — the two share the keyword but differ in step format.
Tutorials are never processed by the backend parser.

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

**Content resolution:** Section content is either inline or a file
reference. Resolution depends on the runtime environment:

- `type: 'inline'` — markdown is embedded in the YAML. No resolution
  needed — the narrative component renders it directly.
- `type: 'template'` — content is in an external markdown file at `path`.
  - **Browser-only mode:** `PagesScenarioNarrative` gains a `contentBase`
    property. When set, template paths resolve via
    `fetch(contentBase + '/' + path)`. The tutorial runner sets
    `contentBase` to the tutorial's content directory (derived from the
    registry entry's `path` field). This uses a static file server
    (webpack-dev-server, Quinoa static serving, etc.).
  - **Server mode:** Falls back to the existing
    `fetch(restBase + '/scenario/content?path=...')` REST endpoint.
  - Resolution strategy: `contentBase` takes precedence when set.
    Absent `contentBase` falls back to the REST endpoint via `restBase`.
    The template cache (`_templateCache`) is shared across both paths.

### 2.4 Backward compatibility and validation

The parser supports both formats:
- **Flat format** (`scenario` + `steps`): existing scenarios, unchanged
- **Sectioned format** (`scenario` + `sections`): tutorials and
  structured demos

If both `steps` and `sections` are present, the parser rejects the
input with an error — the two formats are mutually exclusive. This
mirrors the backend `HierarchicalParser` which throws
`IllegalArgumentException` when `chapters`, `sections`, and `steps`
co-occur at the top level. On a pre-release platform with no existing
content to migrate, silent override is unnecessary complexity that
hides authoring mistakes. The `meta` block is always optional —
existing scenarios without `meta` continue to work.

**Validation rules:**

| Condition | Result |
|-----------|--------|
| `scenario` missing or empty | Error: scenario name required |
| Neither `steps` nor `sections` present | Error: must have `steps` or `sections` |
| Both `steps` and `sections` present | Error: mutually exclusive — use one or the other |
| `sections` is empty array | Valid — produces a `slides-only` tutorial with no content |
| `sections` present without `meta` | Valid — `meta` is always optional |
| Section has no `steps` field | Valid — parser normalizes to `steps: []` (slides-only section) |

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
- `contentType` — `slides-only` or `hands-on`, derived from section
  analysis

**`contentType` derivation rule:** `hands-on` if ANY section has a
non-empty `steps` array (at least one executable step anywhere in the
tutorial). `slides-only` if ALL sections have empty or absent `steps`.
This is deterministic — a tutorial with 6 narrative sections and 1
section containing a single step is `hands-on`.

**Relationship to `ScriptDescriptor`:**

`TutorialDescriptor` (registry) and `ScriptDescriptor` (script library,
`library-view.ts`) serve different domains and are intentionally
separate types:

- **Scripts** are operational automations — browsed by capability,
  executed on demand. `ScriptDescriptor` carries `params`, `calls`,
  `provenance`, and `firstStepTargets` for readiness probing.
- **Tutorials** are instructional content — browsed by concept and
  difficulty, followed as learning exercises. `TutorialDescriptor`
  carries `area`, `estimated`, `prerequisites`, and `hero` for
  catalog presentation.

Both use the shared typed label format (`namespace:value`) and
free-form tags. A tutorial and a script may share the same `scenario`
name — the tutorial teaches the automation that the script executes.
The tutorial catalog and script library are separate browsing contexts
with separate data sources; no name collision mechanism exists.
Tutorials do NOT appear in the script library. Scripts do NOT appear
in the tutorial catalog.

### 3.3 Aggregation contract

For `casehub/examples` integration:
1. Each tutorial source (pages, helpdesk, etc.) produces its own
   `tutorial-registry.json` as part of its build. Registry entries
   use paths relative to the source's own root.
2. `casehub/examples` imports these JSON files and merges them.
   The aggregation script validates that all `scenario` names are
   globally unique across sources — duplicate names fail the build
   with an error identifying both conflicting entries and their
   source registries. No conflict resolution — uniqueness is
   enforced, not negotiated
3. The aggregation script adds a `basePath` field to each source's
   registry entries at merge time. `basePath` is the source's content
   directory relative to the aggregated output root (e.g.,
   `pages/tutorials/` for pages-sourced tutorials). The aggregator
   prepends `basePath` to each entry's `path` field so content
   resolution works from the aggregated root. **`basePath` is not a
   source-level field** — it does not appear in the meta block or
   individual registry files. It is added exclusively by the
   aggregation script.
4. The catalog component receives the merged array — it is
   source-agnostic. It resolves tutorial content paths relative to
   the page's base URL

### 3.4 Build-time validation

The registry build step validates:
- All required meta fields are present
- Labels follow the typed format (`namespace:value`)
- Scenario names are unique across all sources in the merged registry
  (duplicates are an error, not silently last-wins)
- Prerequisites reference valid scenario names within the registry
  (standalone build scope — see below)
- Learning path references (§5) point to valid tutorials
- Learning path ordering is consistent with prerequisites (warning,
  not error — paths may intentionally skip prerequisites for curated
  audiences, e.g., "Task Automation" skips the architecture overview)
- `SectionContent` consistency: `type: 'inline'` requires `markdown`
  and must not have `path`; `type: 'template'` requires `path` and
  must not have `markdown`. Mismatches fail the build.
- Template file existence: for `type: 'template'` sections, the build
  script verifies that `path` resolves to an actual file relative to
  the tutorial's directory. Missing files fail the build with a message
  identifying the tutorial, section title, and unresolved path. This
  catches typos at build time rather than at runtime.

**Prerequisite validation scope:**
- **Standalone build** (single source, e.g. `pages`): prerequisites
  are validated within that source's registry. Unknown prerequisites
  produce warnings, not errors — the prerequisite may exist in another
  source's registry (e.g. `helpdesk-basics` lives in the helpdesk
  registry). This preserves typo detection for within-source refs
  while allowing cross-source prerequisites.
- **Aggregation build** (`casehub/examples`): prerequisites are
  validated against the merged registry. Unknown prerequisites at
  this level are errors — all sources are present, so an unresolved
  prerequisite is either a typo or a missing tutorial.

## 4. Catalog Component

### 4.1 `<pages-tutorial-catalog>`

A Lit web component in `packages/pages-aria/src/tutorial/` that
renders the tutorial catalog. Accepts a `TutorialDescriptor[]` array
and supports two display modes. The `tutorial/` module boundary
separates source-agnostic, data-driven catalog components from the
execution-coupled `controller/` module. The catalog has zero coupling
to push wire, REST endpoints, or execution state — it receives
descriptors and emits selection events.

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
- Hero icon + title + subtitle (when `hero` is present in meta)
- Difficulty chip (beginner/intermediate/advanced)
- Estimated duration
- Concept label chips
- Prerequisites indicator (if any)
- Click to select and launch the tutorial

**Hero fallback:** When `hero` is absent, the card displays `title` and
`description` from the meta block directly. No icon placeholder — the
card layout adapts to a text-only variant without the hero section.
This matches the existing `PagesLibraryView` card pattern (which shows
name + description without hero fields).

**Breadcrumb navigation:** `All Tutorials > [Area Name]`

### 4.3 Compact list view (`mode='list'`)

Flat filterable list across all tutorials (all areas):
- Each row: title, area badge, difficulty chip, concept labels, duration
- Filter bar at top: label chips (click to toggle), text search
- Sort: by area, by difficulty, alphabetical
- Same click-to-select behavior as tiles

### 4.4 Mode toggle

A segmented control in the catalog header switches between tiles and
list modes. The active mode is stored in local storage
(`tutorial:catalog-mode` → `tiles` | `list`) so it persists across
sessions. URL hash parameter `#mode=list` allows direct linking and
overrides the stored preference for that page load.

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

**Completion criteria:** A tutorial is marked complete when the runner
reaches the end of the final section. For hands-on tutorials, all
steps in the final section must execute. For slides-only tutorials,
the final section's content must be rendered (the runner reaches the
end-of-scenario state). Completion is persisted to local storage:

- Key: `tutorial:completed:{scenario-name}` → `true`
- Path progress: count of entries in the path's `tutorials` array
  that have a `true` value in local storage

**Local storage graceful degradation:** If `localStorage` throws
(disabled, quota exceeded, or private browsing mode), completion
tracking degrades silently — tutorials work normally, progress is
not persisted. The catalog shows 0% path progress. No error is
surfaced. Cross-origin: `localStorage` is per-origin, so tutorials
aggregated from different sources on the same origin share completion
state correctly.

**Tutorial rename handling:** The scenario name is the stable key.
If a tutorial is renamed (scenario name changes), the old completion
entry is orphaned (harmless stale data, not leaked). The new name
starts as incomplete. No migration mechanism — the scenario name is
intended to be stable across content revisions.

**Prerequisite behavior:** Prerequisites are advisory, not blocking.
When a tutorial has incomplete prerequisites, the catalog card shows
a prerequisite indicator. Clicking the card shows a dismissible
notice ("Recommended: complete [prerequisite titles] first") but does
not block — the user can proceed immediately. This avoids frustrating
users who are familiar with the material but haven't completed the
formal tutorial sequence.

### 5.4 Multiple paths

A tutorial can appear in multiple paths. Paths can span areas.
Example paths:
- "Scenario Engine Fundamentals" — all tutorials in order
- "Task Automation" — Tutorials 1, 2, 3, 4 (skip architecture)
- "Platform Admin" — Tutorials 0, 5 (architecture + registry)

## 6. Client-Side Parser Extension

### 6.1 Sectioned format support

Extend `parseScenario()` in `packages/pages-aria/src/scenario/parser.ts`
to handle the `sections` array.

**Type hierarchy — discriminated union:**

The existing `Scenario` type has a required `steps` field. A sectioned
scenario has steps inside sections, not at the top level. Rather than
making `steps` optional (which weakens the base type) or inheriting a
contradictory required field, the types use a discriminated union —
matching the backend `HierarchicalParser`'s mutual exclusivity of
`chapters`, `sections`, and `steps`:

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
  steps: ScenarioStep[];  // always present after parsing — YAML may omit, parser normalizes to []
}

export interface ScenarioBase {
  scenario: string;
  meta?: TutorialMeta;
}

export interface FlatScenario extends ScenarioBase {
  steps: ScenarioStep[];
}

export interface SectionedScenario extends ScenarioBase {
  sections: TutorialSection[];
}

export type Scenario = FlatScenario | SectionedScenario;

export function isSectioned(s: Scenario): s is SectionedScenario {
  return 'sections' in s;
}
```

The parser detects `sections` vs `steps` at the top level and returns
the appropriate variant. Callers that only handle flat scenarios
narrow with `if (!isSectioned(s))`. This is a breaking change to the
`Scenario` type — call sites that access `scenario.steps` directly
must narrow first. The migration is mechanical.

### 6.2 Scenario runner extension

The runner needs to handle sectioned scenarios. In browser-only mode
(no server), the runner acts as a client-side orchestrator, dispatching
events on the shared `EventTarget` that the controller and narrative
component already listen to.

**Runner API:**

```typescript
export interface TutorialRunnerOptions {
  eventTarget: EventTarget;
  contentBase?: string;  // base URL for template content resolution
  speed?: number;        // default: 1.0
  startPaused?: boolean; // default: true
}

export interface TutorialRunner {
  play(): void;
  pause(): void;
  step(): void;
  runTo(sectionTitle: string): void;
  setSpeed(speed: number): void;
  dispose(): void;
}

export function runSectionedScenario(
  scenario: SectionedScenario,
  options: TutorialRunnerOptions,
): TutorialRunner;
```

**Runner behavior:**

1. Build an outline from sections: `OutlineNode[]` with section titles
   as top-level nodes and steps nested under each
2. For each section:
   a. Fire `pages-event` with `op: 'event', topic: 'scenario:state'`
      on the `eventTarget` — the same format the controller consumes
      from push wire. Include `scenario`, `section` (title), `step`
      (null initially), `paused`, `progress`, and `content` (the
      section's narrative payload)
   b. For slides-only sections (no steps): pause regardless of play
      mode. User must explicitly advance (play/step/outline click)
   c. For sections with steps: execute each step in sequence using the
      existing `executeStep()` function, firing state updates per step
      with current step name and progress
   d. Between sections: pause (if in step mode) or continue (if
      playing)

**ScenarioState mapping:**

The `ScenarioState` interface (`scenario-connection-controller.ts`)
has `scenario`, `chapter`, `section`, `step`, `paused`, `speed`,
`progress`, `content`, and `slides` fields. The existing controller
breadcrumb renders `[chapter] → [section] → [step]` (null segments
filtered). The sectioned runner populates all fields:

| ScenarioState field | Tutorial source |
|---------------------|-----------------|
| `scenario` | Tutorial's `scenario` field |
| `chapter` | Tutorial's `meta.title` (tutorial title as chapter-level label) |
| `section` | Current section's `title` |
| `step` | Current step's generated name (same as flat scenarios); `null` for slides-only sections |
| `paused` | Runner execution state |
| `speed` | Runner speed multiplier |
| `progress` | Hands-on: steps completed / total steps across all sections. Slides-only (total steps = 0): sections visited / total sections. Avoids 0/0 NaN. |
| `content` | Current section's `content` object (templates pre-resolved to inline by runner — see template pre-resolution below) |
| `slides` | `null` (tutorials don't use reveal.js slides) |

Breadcrumb renders as: `Form Automation Basics → Your first fill
command → fill-textbox-Full Name`. `chapter` is never null for
tutorials — it always carries the tutorial title, providing context
in the status bar.

**Naming relationship to executor protocol:** The tutorial format
uses `title` for sections (content-authoring convention). The backend
`ScenarioSection` uses `label` (protocol convention). Both refer to
the human-readable name displayed in navigation. The client-side
runner maps `title` → `section` in `ScenarioState`. If tutorials are
ever server-parsed, `HierarchicalParser` can accept `title` as an
alias for `label` — but this is out of scope since tutorials are
browser-only (§1.2).

**Event path:** The sectioned runner uses ONLY the `pages-event` with
topic `scenario:state` path. It does NOT fire `scenario-narrative`
custom events. The `scenario-narrative` event is used exclusively by
`scenario-handler.ts` for server-dispatched `show-markdown` commands.
The `ScenarioConnectionController` extracts content from the state
payload and the narrative component renders from
`this._conn.state.content` — one event path, no duplication.

**Pause/resume mechanism:** The runner maintains internal `paused` and
`speed` state. Between sections and between steps, the runner checks
`paused`. If paused, execution suspends on a Promise that resolves
when `play()` or `step()` is called:

```typescript
if (paused) {
  await new Promise<void>(resolve => { resumeResolve = resolve; });
}
```

`step()` resumes and immediately re-pauses via `queueMicrotask`:

```typescript
step(): void {
  paused = false;
  if (resumeResolve) { resumeResolve(); resumeResolve = null; }
  queueMicrotask(() => { paused = true; });
}
```

This is identical to the server-side handler pattern in
`scenario-handler.ts` (lines 862-863). The default start state is
`paused: true` — the user sees the first section's content and
outline, then advances with play or step.

**Transport commands:** The runner listens for `scenario-control`
custom events on the eventTarget (same event format as
`scenario-handler.ts` uses for `pause`/`resume`/`step`/`speed`
commands). This reuses the existing control protocol without REST.

**Template pre-resolution:** Before starting section iteration, the
runner pre-resolves all `type: 'template'` sections by fetching
their content via `fetch(contentBase + '/' + path)`. If any fetch
fails (404, network error), `runSectionedScenario` throws with a
message identifying the unresolved path and section title. This is
a fail-fast startup error — the tutorial does not partially load.
Build-time validation (§3.4) catches most path errors; runtime
pre-resolution is the second line of defense for paths valid at
build time but missing at serve time (e.g., deploy artifact issues).

**Dispose contract:** `dispose()` performs four cleanup actions,
mirroring the server-side handler's dispose
(`scenario-handler.ts:905-919`):

1. Remove the `scenario-control` event listener from the eventTarget
2. Resolve any pending pause Promise (prevents memory leak — the
   suspended async function completes and releases references)
3. Set an internal `disposed` flag checked by the execution loop —
   the next `await` point exits cleanly rather than continuing to
   the next section
4. Fire a final state event with `scenario: null` to clear the
   controller's display (outline, breadcrumb, transport controls
   reset to their empty state)

After `dispose()`, calling `play()`, `step()`, or `runTo()` is a
no-op (the runner checks the disposed flag).

**Controller integration in browser-only mode:**

| Concern | Server mode | Browser-only mode |
|---------|-------------|-------------------|
| Outline | `GET /scenario/outline` | Built client-side from `SectionedScenario.sections` |
| Transport commands | REST POST (`/pause`, `/resume`, `/step`) | `scenario-control` events on eventTarget |
| State updates | Push wire → `pages-event` | Runner fires `pages-event` directly |
| Content resolution | `/scenario/content?path=...` | `fetch(contentBase + '/' + path)` |

**Required component modifications for browser-only mode:**

`ScenarioConnectionController` and `PagesScenarioController` both
require modifications to support eventTarget-only operation (no
`connection`, no `baseUrl`):

1. **`ScenarioConnectionController.hostConnected()`** — currently
   gates event listener registration on `conn && target` (line 66 of
   `scenario-connection-controller.ts`). Without a connection, the
   `pages-event` listener is never registered. Fix: register the
   `pages-event` listener when `eventTarget` is provided, even
   without a connection. The `conn.listen()` call (server-side topic
   subscription) is skipped; only the DOM event listener is needed
   for the runner's locally-dispatched state events.

2. **`PagesScenarioController.render()`** — currently returns an
   error div when `!this.connection && !this.baseUrl` (line 276 of
   `scenario-controller.ts`). Fix: accept `eventTarget`-only
   configuration as valid. The guard becomes:
   `!this.connection && !this.baseUrl && !this.eventTarget`.

3. **`PagesScenarioController` keyboard shortcuts** — currently call
   `this._conn.sendCommand(...)` (lines 208-212) which issues REST
   `fetch()`. Fix: in browser-only mode (no `baseUrl`), dispatch
   `scenario-control` events on the `eventTarget` instead, matching
   the same event format the runner listens for.

4. **`PagesScenarioController._fetchOutline()`** — currently fetches
   from `GET /scenario/outline` (line 249). Fix: in browser-only
   mode, the outline is provided directly via a new `outline`
   property, populated by the tutorial host page from the parsed
   `SectionedScenario.sections`.

5. **`PagesScenarioController._renderNode()` and
   `_isBeforeCurrent()`** — current highlighting logic matches
   leaf nodes against `state.step` (line 404:
   `node.label === this._conn?.state.step`). For slides-only
   sections where `step` is `null`, no node is ever highlighted.
   `_isBeforeCurrent` uses `labels.indexOf(state.step ?? '')`
   which returns -1 when step is null, so no completion markers
   appear either. Fix: when `state.step` is `null` and
   `state.section` is set, match section-level nodes (outline
   headings) against `state.section` instead of leaf nodes
   against `state.step`. A section node is "current" when its
   label equals `state.section` and `state.step` is null. A
   section node is "completed" when it precedes the current
   section in the outline. `_isBeforeCurrent` falls back to
   section-level comparison using `state.section` when
   `state.step` is null.

### 6.3 Controller outline integration

The scenario controller's outline tree renders section titles as
navigation points. Each section's steps are nested under it. The
`runTo` feature works at the section level — clicking a section
title in the outline runs to that section.

**Outline population in browser-only mode:**

The runner includes the outline in the initial `scenario:state` event
payload. The `ScenarioState` type gains an optional `outline` field:

```typescript
export interface ScenarioState {
  // ... existing fields ...
  outline?: OutlineNode[];  // present in initial state, absent in updates
}
```

The controller's `_onStateChange` checks for this field:

```typescript
private _onStateChange(s: ScenarioState): void {
  if (s.outline) {
    this._outline = s.outline;
  } else if (s.scenario && this._outline.length === 0) {
    void this._fetchOutline();  // server mode fallback
  }
  if (!s.scenario) this._outline = [];
  // ...
}
```

The outline is sent once (in the first state event) — subsequent
state updates omit it. This avoids a new event type while keeping
the outline delivery lazy.

**RunTo in browser-only mode:**

The controller detects browser-only mode when `eventTarget` is set
but no `connection` or `baseUrl` is configured. In this mode, outline
click handlers dispatch `scenario-control` events instead of REST
calls:

```typescript
@click=${() => {
  if (this._browserOnly) {
    this.eventTarget?.dispatchEvent(new CustomEvent('scenario-control', {
      detail: { command: 'run-to', target: node.label }
    }));
  } else {
    void this._conn.sendCommand('/run-to', { label: node.label });
  }
}}
```

The runner listens for `scenario-control` with `command: 'run-to'`
and navigates to the target section. Navigation is bidirectional:

- **Forward** (target after current position): skip intervening
  sections and their steps without executing them. The runner
  repositions to the target section's initial state and pauses.
- **Backward** (target before current position): reset the runner's
  position to the target section. Fire a state event with the
  target section's content. No steps are re-executed — the runner
  repositions and pauses at the target section's initial state.
  Step completion state for sections after the target is reset
  (the user can re-execute them).

In both directions, the runner pauses at the target section
regardless of the current play/pause state. This gives the user
a stable reading position after navigation.

## 7. Narrative Renderer Enhancement

### 7.1 SVG sanitization mode

Add a property to `PagesScenarioNarrative` that enables inline SVG
in rendered markdown while maintaining security:

```typescript
@property({ type: String }) htmlMode: 'escape' | 'sanitized' = 'escape';
```

**Modes:**
- `'escape'` (default) — all HTML is escaped. Existing behavior
  preserved. Non-tutorial scenarios use this mode.
- `'sanitized'` — HTML is parsed and sanitized. Only SVG elements
  and a restricted set of attributes pass through. All other HTML
  (including `<script>`, `<iframe>`, event handler attributes) is
  stripped.

**Allowed elements (sanitizer allowlist):**
- SVG structural: `svg`, `g`, `defs`, `use`, `symbol`, `clipPath`,
  `mask`, `pattern`
- SVG shapes: `rect`, `circle`, `ellipse`, `line`, `polyline`,
  `polygon`, `path`
- SVG text: `text`, `tspan`
- SVG paint: `linearGradient`, `radialGradient`, `stop`, `marker`

**Allowed attributes:**
- Geometry: `viewBox`, `xmlns`, `width`, `height`, `x`, `y`, `cx`,
  `cy`, `r`, `rx`, `ry`, `x1`, `y1`, `x2`, `y2`, `d`, `points`,
  `transform`
- Presentation: `fill`, `stroke`, `stroke-width`, `opacity`,
  `font-size`, `font-family`, `text-anchor`, `dominant-baseline`,
  `stroke-dasharray`, `stroke-linecap`
- Style: `style` attribute permitted, but only CSS custom property
  references (`var(--pages-*)`) and safe CSS properties. Values
  containing `url()`, `expression()`, `javascript:`, or
  `-moz-binding` are stripped.
- Identity: `id`, `class`, `aria-label`

**Blocked (stripped silently):**
- Elements: `<script>`, `<iframe>`, `<object>`, `<embed>`, `<form>`,
  `<input>`, `<link>`, `<meta>`
- Attributes: event handlers (`on*`), `href`/`xlink:href` with
  `javascript:` protocol
- Style values: `url()`, `expression()`, `-moz-binding`

**Who sets it:** The tutorial runner sets `htmlMode = 'sanitized'`
on the narrative component when running a tutorial. Regular scenario
execution never sets this — the default `'escape'` mode applies.
This is set programmatically by the runner, not declaratively in
YAML — the YAML content has no ability to control its own rendering
mode.

**Implementation:** A small inline sanitizer (~50 LOC) using
`DOMParser` and element/attribute allowlists. No external dependency.
The sanitizer parses the markdown output with `DOMParser`, walks the
DOM tree, removes disallowed elements and attributes, and serializes
back to HTML. This runs after markdown-to-HTML conversion but before
`innerHTML` assignment.

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

The `area` meta field is a direct filter dimension in the catalog, not
a label namespace. The catalog's filter bar supports `area` as a
built-in grouping/filter alongside label-based filtering — no `area:*`
label synthesis is needed. A user can filter by `difficulty:beginner`
and see tutorials from all sources, or drill into a specific area via
the tiled view.

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
