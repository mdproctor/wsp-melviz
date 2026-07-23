# casehub-pages Workspace

**Name:** casehub-pages
**Project repo:** /Users/mdproctor/claude/casehub/pages
**Workspace type:** public

## Git Remotes

`origin` = `mdproctor/casehub-pages` (fork — all work pushes here)
`upstream` = `casehubio/casehub-pages` (blessed repo)

## Session Start

Run `add-dir /Users/mdproctor/claude/casehub/pages` and `add-dir /Users/mdproctor/claude/public/casehub/pages` before any other work.

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
- **Workspace** (`/Users/mdproctor/claude/public/casehub/pages`) — plans, blog (staging), snapshots, handover
- **Project repo** (`/Users/mdproctor/claude/casehub/pages`) — source code, ADRs (`docs/adr/`), specs

Never rely on CWD for git operations — the session may have started in either repo. Always use explicit paths:
```bash
git -C /Users/mdproctor/claude/public/casehub/pages add <file>    # workspace artifacts
git -C /Users/mdproctor/claude/casehub/pages add <file>      # project artifacts
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

**Blog directory:** `/Users/mdproctor/claude/public/casehub/pages/blog/`

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

This is a TypeScript monorepo managed with Yarn workspaces. Build order matters: packages → components → webapp.

### Full Build

```bash
# Install dependencies and build everything in correct order (development)
yarn install && yarn build

# Production build — includes examples gallery
yarn build:prod
```

### Targeted Builds

```bash
# Shared TypeScript packages only
yarn build:packages

# Iframe components only (packages must be built first)
yarn build:components

# Final webapp assembly only
yarn build:webapp

# Examples gallery only (webapp must be built first)
yarn build:examples
```

### Per-Component Builds

```bash
# Build a specific component
yarn workspace @casehubio/pages-component-echarts run build

# Run component tests
yarn workspace @casehubio/pages-component-echarts run test

# Dev mode with hot reload (port 9001)
yarn workspace @casehubio/pages-component-echarts run start
```

### Type Checking & Linting

```bash
# Incremental cross-package type check (uses tsc --build with project references)
yarn typecheck

# ESLint with strict-type-checked rules
yarn lint
```

### Examples Dev Server

```bash
# Serve examples gallery (port 8080) — requires webapp to be built first
yarn workspace @casehubio/pages-examples run serve

# Dev mode with file watching
yarn workspace @casehubio/pages-examples run dev
```

## Architecture Overview

### Monorepo Structure

- **`packages/`** — Core TypeScript libraries for dashboard rendering
- **`components/`** — Iframe-isolated React microfrontend visualization components
- **`webapp/`** — Webpack orchestrator; assembles final application bundle
- **`examples/`** — Interactive dashboard examples gallery
- **`_legacy/`** — Former Java/GWT core (reference only, not built)

### Package Overview

**Core Packages** (`packages/`):
- `@casehubio/pages-ui-tokens` — OKLCH 12-step design tokens: colour scales, spacing, typography, elevation, motion, radius. Theme generation and injection. Must build before `pages-viz`.
- `@casehubio/pages-data` — DataSet model, operations engine, external data extraction, JSONata. Push wire protocol (`EventConnection`, `PushSource`, `WebSocketSource`). General-purpose `SSEManager` (connection pooling, named event support, reconnection). Group extraction (`extractGroupBoundaries`, `extractGroupTree`). `SourceConnector` — data source lifecycle primitive (connect/disconnect/replace/refresh with stale-source guard).
- `@casehubio/pages-ui` — YAML parser, DashBuilder backward compat, component model
- `@casehubio/pages-viz` — Web Component chart/grid-table/metric wrappers (ECharts). `PagesGridTable` (`pages-grid-table`, type `grid-table`) — lightweight information grid with togglable column/row headers (cross-matrix support) and per-column cell display modes (text, boolean, color, badge, number). Builder: `gridTable()`.
- `@casehubio/pages-component` — CSS grid layout renderer, interactive containers, `DataSourceController` (Declaration + VizTarget, delegates lifecycle to `SourceConnector`), `createStandaloneConnector` (wires controller + connector + DataSetManager for non-pipeline use)
- `@casehubio/pages-primitives` — Lit-dependent UI primitives: a11y mixins (LiveRegionMixin, FocusTrapMixin, RovingTabindexMixin, KeyboardShortcutMixin). Depends on `lit`. Migrated from blocks-ui-core in blocks-ui#48.
- `@casehubio/pages-table` — `PagesDataTable` (`pages-data-table`, type `data-table`) — interactive data table: CSS Grid rendering, virtual scroll, sorting, filtering, column visibility (`hiddenColumns`), multi-mode selection, tree rows, row-detail expansion, CSV export, ARIA grid, keyboard navigation, native `groupBy` (interleaved group headers). Builder: `dataTable()`. Depends on `lit`. Migrated from blocks-ui in blocks-ui#48.
- `@casehubio/pages-runtime` — Site orchestrator: `loadSite()` API, navigation, data pipeline, layout serialization (`LayoutStore`, `createLocalLayoutStore`)

**Iframe Component API** (`packages/`):
- `@casehubio/pages-iframe-api` — Component controller for iframe-isolated components
- `@casehubio/pages-iframe-dev` — Development utilities for component testing
- `@casehubio/pages-echarts-base` — Reusable ECharts wrapper library

**Standalone Components** (`components/`):
- `@casehubio/pages-component-echarts` — Apache ECharts visualizations
- `@casehubio/pages-component-llm-prompter` — LLM prompt engineering UI
- `@casehubio/pages-component-svg-heatmap` — SVG-based heatmaps

**Backend (Java)** (`backend/`):
- `casehub-pages-push` — Typed wire protocol SDK: `PushMessage` (server→client builders), `PushRequest` (sealed client→server parser with ack/error correlation), `TopicRegistry` (wildcard-aware connection tracking), `EventStore` SPI + `InMemoryEventStore` (bounded per-topic event replay). jackson-core only, no Quarkus.
- `casehub-pages-push-runtime` — CDI producers for EventBroadcaster, TopicRegistry, EventStore. @DefaultBean InMemoryEventStore with configurable capacity. Consumer provides SessionSender. Quarkus Arc, no JPA.

### Data Flow

```
YAML → @casehubio/pages-ui (parse) → @casehubio/pages-data (resolve)
  → @casehubio/pages-component (layout) → @casehubio/pages-viz (render)
  → pages-filter/pages-sort events → back to data layer
```

## Key Technologies

- **Yarn 4.10.3** with workspaces
- **TypeScript 5** / **React 17** / **Webpack 5**
- **Vitest / Jest** — testing
- **ESLint** with `@typescript-eslint/strict-type-checked` — linting
- **Apache ECharts** — charting
- **JSONata** — data transformation

## Project Artifacts

| Path | What it is |
|------|------------|
| `CLAUDE.md` | Project conventions (build, test, naming) |
| `docs/protocols/` | Standing project rules and conventions |
| `docs/superpowers/specs/` | Design specs |

## Work Tracking

Issue tracking: enabled
GitHub repo: casehubio/casehub-pages
Changelog: GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — check if an active issue or epic exists. If not, create one before writing any code.
- **Before writing any code** — check if an issue exists. If not, draft one before starting.
- **Before any commit** — confirm issue linkage.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
