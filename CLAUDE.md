# melviz Workspace

**Name:** melviz
**Project repo:** /Users/mdproctor/claude/melviz
**Workspace type:** public

## Git Remotes

`origin` = `melviz-org/melviz` (blessed repo — do not push here yet)
`fork` = `mdproctor/melviz` (fork — all work pushes here)

## Session Start

Run `add-dir /Users/mdproctor/claude/melviz` and `add-dir /Users/mdproctor/claude/public/melviz` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `work-start`) |
| adr | `adr/` |
| write-content | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md

## Git Discipline

Two git repositories are active in every session:
- **Workspace** (`/Users/mdproctor/claude/public/melviz`) — plans, blog (staging), snapshots, handover
- **Project repo** (`/Users/mdproctor/claude/melviz`) — source code, ADRs (`docs/adr/`), specs

Never rely on CWD for git operations — the session may have started in either repo. Always use explicit paths:
```bash
git -C /Users/mdproctor/claude/public/melviz add <file>    # workspace artifacts
git -C /Users/mdproctor/claude/melviz add <file>      # project artifacts
```
The file path determines the repo: if the file lives under the workspace path, use the workspace; if under the project path, use the project.

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

Per-artifact routing destinations (optional). If absent, all artifacts route to the project repo.

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` — promoted at work end |
| specs      | project     | lands in `docs/specs/` — promoted at work end |
| blog       | workspace   | staged here; published via publish-blog at work end |
| plans      | workspace   | stay in workspace permanently |
| design     | workspace   | epic journal stays in workspace |
| snapshots  | workspace   | stay in workspace permanently |
| handover   | workspace   | |

**Blog directory:** `/Users/mdproctor/claude/public/melviz/blog/`

Valid destinations: `project` · `workspace` · `mdproctor.github.io` · `alternative ~/path/to/repo/`

`mdproctor.github.io` — blog publishing destination, resolved via `~/.claude/blog-routing.yaml`.

To set a global default across all workspaces, add to `~/.claude/CLAUDE.md`:
```markdown
## Routing
**Default destination:** workspace
```
Global valid values: `workspace` or `project` only (no alternative at global level).

---

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Type

type: custom

## Build Commands

This is a hybrid Java/Maven + JavaScript/Yarn monorepo. Build order matters: packages → core → components → webapp.

### Full Build

```bash
# Install dependencies and build everything in correct order (development)
yarn install && yarn build

# Production build — includes full GWT compilation and examples gallery
yarn build:prod
```

`build:prod` differs from `build` in two ways: it runs `mvn -f core/pom.xml clean install -Dfull` (no `-DskipTests`) and also builds the `examples/` gallery.

### Targeted Builds

```bash
# Shared TypeScript packages only (order matters: @casehub/component → @casehub/data → @casehub/ui → @casehub/viz → @casehub/runtime)
yarn build:packages

# Java/GWT core only (skips tests, includes sources profile)
mvn -f core/pom.xml clean install -DskipTests -Psources

# Run Java tests only
mvn -f core/pom.xml test

# React components only (packages must be built first)
yarn build:components

# Final webapp assembly only
yarn build:webapp

# Examples gallery only (webapp must be built first)
yarn build:examples
```

**Do not use `cd core && mvn ...`** — use `mvn -f core/pom.xml ...` instead. `cd` before subsequent commands triggers permission checks that override the allow list.

### Per-Component Builds

```bash
# Build a specific component
yarn workspace @melviz/component-echarts run build

# Run component tests
yarn workspace @melviz/component-echarts run test

# Run specific test file
yarn workspace @melviz/component-echarts run test -- <test-file-pattern>

# Dev mode with hot reload (port 9001)
yarn workspace @melviz/component-echarts run start
```

### Examples Dev Server

```bash
# Serve examples gallery (port 8080) — requires webapp to be built first
yarn workspace @melviz/examples run serve

# Dev mode with file watching
yarn workspace @melviz/examples run dev
```

## Architecture Overview

### Monorepo Structure

- **`core/`** — Java/Maven GWT webapp; compiles Java to JavaScript
- **`packages/`** — Shared TypeScript libraries (`@casehub/component`, `@casehub/data`, `@casehub/ui`, `@casehub/viz`, `@casehub/runtime`, `@melviz/component-api`, `@melviz/component-echarts-base`, `@melviz/component-dev`, `webpack-base`, `tsconfig`)
- **`components/`** — Independent React microfrontend visualization components
- **`webapp/`** — Webpack orchestrator; copies GWT output + component bundles into `dist/`
- **`examples/`** — Interactive dashboard examples gallery; depends on `@melviz/webapp`

### Java Core (GWT + Errai CDI)

The Java core uses **GWT** to compile Java to JavaScript and **Errai** for CDI-style dependency injection on the client. Key modules:

- `melviz-base/melviz-dataset` — Core `DataSet` model, `DataSetManager`, `DataSetOpEngine` (filtering/grouping)
- `melviz-base/melviz-json` — JSON utility layer
- `melviz-shared/melviz-displayer-api` — `DisplayerSettings` and chart settings builders
- `melviz-shared/melviz-navigation-api` — `NavTree` for page navigation structure
- `melviz-client/melviz-displayer-client` — Abstract displayer framework and dataset lookup logic
- `melviz-client/melviz-renderers/melviz-renderer-default` — Table, selector, metric displayers
- `melviz-client/melviz-renderers/melviz-renderer-echarts` — ECharts displayer bridge
- `melviz-webapp-parent/melviz-webapp-shared` — JSON marshallers and `RuntimeModel` (the wire format carrying `NavTree` + `LayoutTemplate` list + dataset definitions)
- `melviz-webapp-parent/melviz-webapp` — `RuntimeEntryPoint` (GWT `@EntryPoint`), `Router`, `PlaceManager`, `RuntimeClientLoader`

The GWT entry point (`RuntimeEntryPoint`) initialises the Errai IoC container, injects js-yaml, then calls `Router.doRoute()` to determine which screen to show based on `setup.js` configuration and `postMessage` content.

### Microfrontend Component Architecture

Each React component in `components/` runs in an `<iframe>` and communicates with the GWT core through `window.postMessage`. The `@melviz/component-api` package provides the TypeScript bridge.

**Component lifecycle:**
```typescript
const controller = new ComponentApi().getComponentController();
controller.setOnInit((params) => { /* configure from params */ });
controller.setOnDataSet((dataset, params) => { /* render */ });
controller.ready();                         // tells core the component is ready
controller.filter(filterRequest);           // send filter back to core
```

**Registered components** (copied into `webapp/dist/melviz/component/<name>/` by webpack):
- `echarts` — Apache ECharts charts
- `llm-prompter` — LLM prompt engineering UI
- `svg-heatmap` — SVG-based heatmaps

> Note: `melviz-component-map` exists in `components/` but is **not** registered in `webapp/webpack.config.js` and is not bundled into the webapp.

### Data Flow

```
setup.js / postMessage YAML
        ↓
  RuntimeClientLoader  (parses YAML via js-yaml, builds RuntimeModel)
        ↓
  DataSetOpEngine      (applies JSONata transformations, filters, groups)
        ↓
  DisplayerSettings    (maps YAML config to displayer properties)
        ↓
  ExternalComponentDisplayer  (serialises DataSet → postMessage → iframe)
        ↓
  React Component (ComponentController.setOnDataSet callback)
        ↓
  controller.filter()  →  back to DataSetOpEngine
```

### YAML Dashboard Format

Dashboards are YAML documents with `pages` → `components` structure. The webapp accepts them via:

1. **`setup.js`** — configure `melviz.dashboards` array and optional `melviz.samplesUrl` for static deployments
2. **`postMessage`** — send YAML string to `window` for dynamic embedding

```javascript
window.postMessage(`pages:
  - components:
    - markdown: "# Hello"
`, null)
```

The `melviz.mode` can be `"EDITOR"` (live editing) or `"CLIENT"` (readonly rendering).

### Runtime Modes

`MelvizRuntimeMode` (in `melviz-webapp-shared`):
- **`CLIENT`** — loads dashboards from configured YAML files; no editing
- **`EDITOR`** — enables the layout editor for authoring dashboards

## Key Technologies

- **Java 17** / **Maven** — core build (note: `core/README.md` says Java 21 but `pom.xml` targets 17)
- **GWT** — compiles Java client code to JavaScript
- **Errai** — CDI-style IoC and marshalling for GWT client
- **Yarn 4.10.3** with workspaces
- **TypeScript 4.6.2** / **React 17** / **Webpack 5**
- **Jest + ts-jest** — component unit tests
- **Apache ECharts** — charting library
- **JSONata** — data transformation DSL used inside the GWT core

## Project Artifacts

Paths that are project content (not workspace noise). Skills use this to avoid
filtering or dropping commits that touch these paths.

| Path | What it is |
|------|------------|
| `CLAUDE.md` | Project conventions (build, test, naming) |
| `docs/superpowers/specs/` | Design specs |

## Work Tracking

Issue tracking: enabled
GitHub repo: mdproctor/melviz
Changelog: GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — when the user says "implement", "start coding",
  "execute the plan", "let's build", or similar: check if an active issue or epic
  exists. If not, run issue-workflow Phase 1 to create one **before writing any code**.
- **Before writing any code** — check if an issue exists for what's about to be
  implemented. If not, draft one and assess epic placement (issue-workflow Phase 2)
  before starting. Also check if the work spans multiple concerns.
- **Before any commit** — run issue-workflow Phase 3 (via git-commit) to confirm
  issue linkage and check for split candidates. This is a fallback — the issue
  should already exist from before implementation began.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
  If the user explicitly says to skip ("commit as is", "no issue"), ask once to confirm
  before proceeding — it must be a deliberate choice, not a default.
