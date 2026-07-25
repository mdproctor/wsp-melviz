# TypeScript/Pages Protocols Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #118 — chore: establish TypeScript/pages protocols in garden
**Issue group:** #118

**Goal:** Create 4 project-level protocols and 2 garden-level protocols
formalising casehub-pages conventions for CSS tokens, events, Web
Components, and dataset contracts.

**Architecture:** Documentation-only — protocol files with YAML frontmatter
in two locations: project repo (`docs/protocols/casehub/`) and hortora
garden (`~/.hortora/garden/docs/protocols/web/`, new namespace). Index
files updated in both locations. No code changes.

**Tech Stack:** Markdown with YAML frontmatter. Git commits to two repos
(project and garden).

## Global Constraints

- Project protocol IDs use `PP-YYYYMMDD-xxxxxx` format (hex suffix)
- Garden protocol IDs use kebab-case slugs (e.g., `lit-immutable-collections`)
- Protocol frontmatter fields: `id`, `title`, `type`, `scope`, `applies_to`,
  `severity`, `refs`, `violation_hint`, `created`; optional `cross_repo_consumers`
- `created` date: `2026-07-05` (when the design was approved)
- Garden repo: `~/.hortora/garden` — commit directly to main
- Project repo: `/Users/mdproctor/claude/casehub/pages` — commit to
  `issue-118-ts-pages-protocols` branch

---

### Task 1: Garden `web/` namespace — 2 universal protocols + indexes

Create the first non-Java protocol namespace in the garden. This
establishes the `web/` namespace alongside `casehub/`, adds two
universal Web Component protocols, creates a top-level protocols
INDEX.md, and cross-references from FOUNDATION-INDEX.md.

**Files:**
- Create: `~/.hortora/garden/docs/protocols/web/INDEX.md`
- Create: `~/.hortora/garden/docs/protocols/web/lit-immutable-collections.md`
- Create: `~/.hortora/garden/docs/protocols/web/custom-event-shadow-dom.md`
- Create: `~/.hortora/garden/docs/protocols/INDEX.md`
- Modify: `~/.hortora/garden/docs/protocols/casehub/FOUNDATION-INDEX.md`

**Interfaces:**
- Consumes: nothing (standalone)
- Produces: `web/lit-immutable-collections.md` path (referenced by Task 2's
  `web-component-strategy.md`)

- [ ] **Step 1: Create `web/INDEX.md`**

```markdown
# Web Protocols

Universal conventions for Web Component and browser-side development.
Not CaseHub-specific — these apply to any project using these technologies.

## Protocols

| File | Rule Summary | Applies To |
|------|-------------|------------|
| [lit-immutable-collections.md](lit-immutable-collections.md) | Replace reactive collections on every mutation — never mutate in place | Any Lit project |
| [custom-event-shadow-dom.md](custom-event-shadow-dom.md) | CustomEvents crossing shadow DOM need both `bubbles: true` and `composed: true` | Any Web Component with shadow DOM |
```

- [ ] **Step 2: Create `web/lit-immutable-collections.md`**

```markdown
---
id: lit-immutable-collections
title: "Replace Lit reactive collections on every mutation — never mutate in place"
type: rule
scope: universal
applies_to: "any Lit @state() or @property() holding Set, Map, or Array"
severity: important
refs: []
garden_ref: GE-20260705-7c80f2
violation_hint: "Lit reactive property mutated in place with .add()/.delete()/.push() instead of replaced with a new collection"
created: 2026-07-05
---

Reactive properties (`@state()`, `@property()`) holding mutable collections
(Set, Map, Array) must be replaced on every mutation, never mutated in place.

Lit uses strict reference equality (`===`) to detect property changes. Mutating
a collection in place (`.add()`, `.delete()`, `.push()`, `.splice()`) keeps the
same object reference. Lit sees "same reference → no change" and skips re-render.

This also affects child components. When a parent passes a collection to a child
via property binding (`.items=${this.mySet}`), Lit compares the property value
with `===`. Same reference → child skips re-render too.

## Wrong

```typescript
this.items.add(value);
this.requestUpdate(); // forces THIS component, but children still skip
```

## Right

```typescript
// Set
this.items = new Set([...this.items, value]);

// Map
this.lookup = new Map([...this.lookup, [key, value]]);

// Array
this.list = [...this.list, item];
```

## Why `requestUpdate()` is not a workaround

`requestUpdate()` forces the calling component to re-render. But when Lit
re-renders the parent and evaluates `.items=${this.mySet}`, it still compares
with `===`. Same reference → child property unchanged → child skips re-render.
The only fix is a new reference.

## Scope

This applies to any mutable reference type. Primitives (boolean, string, number)
are not affected — reassignment always creates a new value.
```

- [ ] **Step 3: Create `web/custom-event-shadow-dom.md`**

```markdown
---
id: custom-event-shadow-dom
title: "CustomEvents crossing shadow DOM need both bubbles and composed"
type: rule
scope: universal
applies_to: "any Web Component dispatching CustomEvents from shadow DOM"
severity: important
refs: []
violation_hint: "CustomEvent dispatched from shadow DOM with only bubbles:true or only composed:true — event will not reach ancestors outside the shadow root"
created: 2026-07-05
---

CustomEvents intended to cross shadow DOM boundaries must set both
`bubbles: true` and `composed: true`.

| Configuration | Behaviour |
|--------------|-----------|
| `bubbles: false, composed: false` | Event stays on the target element |
| `bubbles: true, composed: false` | Propagates up the DOM but stops at the shadow root boundary |
| `bubbles: false, composed: true` | Crosses the shadow boundary but does not propagate to ancestors |
| `bubbles: true, composed: true` | Crosses shadow boundaries AND propagates up the full DOM tree |

Only the last combination delivers the event to listeners on ancestor elements
outside the shadow root.

## Public vs internal events

Set `composed: true` when the event is part of the component's **public API** —
consumers listen for it on or above the component.

Omit `composed` (or set it to `false`) when the event coordinates **internal
sub-elements** within the same shadow root. Leaking internal events to the
outer DOM creates coupling — consumers start depending on implementation details.

## Pattern

```typescript
// Public event — crosses shadow DOM, reaches ancestors
target.dispatchEvent(new CustomEvent('my-public-event', {
  bubbles: true,
  composed: true,
  detail: payload,
}));

// Internal event — stays within shadow root
target.dispatchEvent(new CustomEvent('internal-sync', {
  bubbles: true,
  composed: false,
  detail: state,
}));
```
```

- [ ] **Step 4: Create top-level `docs/protocols/INDEX.md`**

This file does not exist yet. Create it as the entry point for all garden
protocol namespaces.

```markdown
# Garden Protocols

Standing conventions and rules captured from cross-project work. Each
namespace groups protocols by technology domain.

## Namespaces

| Namespace | Scope | Description |
|-----------|-------|-------------|
| [casehub/](casehub/FOUNDATION-INDEX.md) | Platform | CaseHub platform-wide conventions — API taxonomy, routing strategies |
| [web/](web/INDEX.md) | Universal | Web Component and browser-side conventions — Lit reactivity, shadow DOM events |

## Adding a new protocol

1. Determine scope: does this apply to a specific platform (`casehub/`),
   a technology domain (`web/`), or does it need a new namespace?
2. If a new namespace is needed, create `<namespace>/INDEX.md` and add a
   row to the table above.
3. Write the protocol file with YAML frontmatter (see any existing
   protocol for the format).
4. Add a row to the namespace's INDEX.md.
5. Commit to garden main.
```

- [ ] **Step 5: Update `casehub/FOUNDATION-INDEX.md` with cross-reference**

Add a line at the bottom referencing the new `web/` namespace:

```markdown
## Other Namespaces

- [Web Protocols](../web/INDEX.md) — universal Web Component conventions (Lit, shadow DOM)
```

- [ ] **Step 6: Commit to garden**

```bash
git -C ~/.hortora/garden add docs/protocols/web/INDEX.md docs/protocols/web/lit-immutable-collections.md docs/protocols/web/custom-event-shadow-dom.md docs/protocols/INDEX.md docs/protocols/casehub/FOUNDATION-INDEX.md
git -C ~/.hortora/garden commit -m "feat: web/ protocol namespace — Lit immutable collections, CustomEvent shadow DOM Refs casehubio/casehub-pages#118"
```

---

### Task 2: Project protocols — 4 casehub-pages conventions

Create the four project-level protocol files and update both INDEX.md
files in the pages repo.

**Files:**
- Create: `docs/protocols/casehub/css-design-tokens.md`
- Create: `docs/protocols/casehub/pages-event-contract.md`
- Create: `docs/protocols/casehub/web-component-strategy.md`
- Create: `docs/protocols/casehub/dataset-contract.md`
- Modify: `docs/protocols/INDEX.md`
- Modify: `docs/protocols/casehub/INDEX.md`

**Interfaces:**
- Consumes: `web/lit-immutable-collections.md` path (referenced in
  web-component-strategy.md Lit conventions section)
- Produces: nothing (terminal task)

- [ ] **Step 1: Create `css-design-tokens.md`**

Copy the full content of §1 from the spec (`docs/specs/2026-07-05-ts-pages-protocols-design.md`,
lines 55–113) into the protocol file, wrapped with YAML frontmatter:

```yaml
---
id: PP-20260705-a1b2c3
title: "CSS design tokens use --pages- prefix, OKLCH 12-step colour scales, and eleven token categories"
type: rule
scope: repo
applies_to: "all CSS custom property declarations and token definitions in casehub-pages"
cross_repo_consumers: ["casehubio/blocks-ui"]
severity: important
refs: []
violation_hint: "CSS custom property does not use --pages- prefix or uses non-standard category/key naming"
created: 2026-07-05
---
```

Body content: Rules 1–3 (token naming prefix, OKLCH scales with chroma
table, token vocabulary table with all eleven categories), density
variants paragraph. Copy verbatim from the spec — do not summarise.

Generate the hex suffix for the ID: `python3 -c 'import secrets; print(secrets.token_hex(3))'`

- [ ] **Step 2: Create `pages-event-contract.md`**

```yaml
---
id: PP-20260705-<hex>
title: "Inter-component events use single pages-event CustomEvent with topic/payload detail"
type: rule
scope: repo
applies_to: "user/application-level inter-component event communication in casehub-pages"
cross_repo_consumers: []
severity: important
refs: []
violation_hint: "Component dispatches a CustomEvent with a name other than pages-event for application-level communication, or constructs pages-event without bubbles:true and composed:true, or uses a reserved framework event name for application purposes"
created: 2026-07-05
---
```

Body content: `PagesEventDetail` interface, emitter paths section (both
`emitPagesEvent` and `dispatchWireEvent`), topic naming, listener
pattern, full reserved framework event names table (15 entries from spec
lines 158–175), staleness note pointing to `site.ts`. Copy verbatim
from spec §2.

Generate hex suffix separately.

- [ ] **Step 3: Create `web-component-strategy.md`**

```yaml
---
id: PP-20260705-<hex>
title: "Web Components use Lit for interactive UI, vanilla HTMLElement for simple display"
type: rule
scope: repo
applies_to: "all Web Component authoring in casehub-pages"
cross_repo_consumers: []
severity: important
refs: []
violation_hint: "Web Component uses Lit for a static display-only widget, or uses vanilla HTMLElement for a component with reactive state and child property bindings"
created: 2026-07-05
---
```

Body content: decision table (Lit vs vanilla), criteria, Lit conventions
(including reference to garden protocol `web/lit-immutable-collections.md`),
vanilla conventions, vanilla base class hierarchy (`PagesElement` →
`PagesChartElement`/`PagesFormInput`, `PagesContentElement`), element
naming rule. Copy verbatim from spec §3.

Generate hex suffix separately.

- [ ] **Step 4: Create `dataset-contract.md`**

```yaml
---
id: PP-20260705-<hex>
title: "Every named dataset exposes a DatasetContract with name, description, and shape"
type: rule
scope: repo
applies_to: "all dataset definitions in casehub-pages"
cross_repo_consumers: []
severity: important
refs: []
violation_hint: "Dataset is used in YAML or event topics without a corresponding DatasetContract export"
created: 2026-07-05
---
```

Body content: `DatasetContract<T>` interface, explanation of `shape` as
concrete example value (not schema, not runtime-introspected). Copy
verbatim from spec §4.

Generate hex suffix separately.

- [ ] **Step 5: Update `docs/protocols/casehub/INDEX.md`**

Replace the file with:

```markdown
# CaseHub Protocols — casehub-pages

| File | Rule Summary | Applies To |
|------|-------------|------------|
| [version-alignment-with-parent.md](version-alignment-with-parent.md) | All module versions must align with casehub-parent | All package.json and pom.xml version declarations |
| [css-design-tokens.md](css-design-tokens.md) | CSS tokens use `--pages-` prefix, OKLCH 12-step scales, eleven categories | All CSS custom property declarations |
| [pages-event-contract.md](pages-event-contract.md) | Application events use single `pages-event` CustomEvent with topic/payload | Inter-component event communication |
| [web-component-strategy.md](web-component-strategy.md) | Lit for interactive UI, vanilla for simple display | All Web Component authoring |
| [dataset-contract.md](dataset-contract.md) | Named datasets expose `DatasetContract` with name, description, shape | All dataset definitions |
```

- [ ] **Step 6: Update `docs/protocols/INDEX.md`**

Replace the file with:

```markdown
# Protocols — casehub-pages

## CaseHub Platform

| File | Rule Summary | Applies To |
|------|-------------|------------|
| [casehub/version-alignment-with-parent.md](casehub/version-alignment-with-parent.md) | All module versions must align with casehub-parent | All package.json and pom.xml version declarations |
| [casehub/css-design-tokens.md](casehub/css-design-tokens.md) | CSS tokens use `--pages-` prefix, OKLCH 12-step scales, eleven categories | All CSS custom property declarations |
| [casehub/pages-event-contract.md](casehub/pages-event-contract.md) | Application events use single `pages-event` CustomEvent with topic/payload | Inter-component event communication |
| [casehub/web-component-strategy.md](casehub/web-component-strategy.md) | Lit for interactive UI, vanilla for simple display | All Web Component authoring |
| [casehub/dataset-contract.md](casehub/dataset-contract.md) | Named datasets expose `DatasetContract` with name, description, shape | All dataset definitions |

See [casehub/INDEX.md](casehub/INDEX.md) for the full listing.
```

- [ ] **Step 7: Commit to project branch**

```bash
git -C /Users/mdproctor/claude/casehub/pages add docs/protocols/casehub/css-design-tokens.md docs/protocols/casehub/pages-event-contract.md docs/protocols/casehub/web-component-strategy.md docs/protocols/casehub/dataset-contract.md docs/protocols/INDEX.md docs/protocols/casehub/INDEX.md
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: pages protocols — CSS tokens, event contract, web component strategy, dataset contract Refs #118"
```

---

## Self-Review

**Spec coverage:**
- §1 css-design-tokens → Task 2 Step 1 ✅
- §2 pages-event-contract → Task 2 Step 2 ✅
- §3 web-component-strategy → Task 2 Step 3 ✅
- §4 dataset-contract → Task 2 Step 4 ✅
- §5 lit-immutable-collections (garden) → Task 1 Step 2 ✅
- §6 custom-event-shadow-dom (garden) → Task 1 Step 3 ✅
- Garden INDEX.md creation → Task 1 Step 4 ✅
- FOUNDATION-INDEX.md cross-ref → Task 1 Step 5 ✅
- Project INDEX.md updates → Task 2 Steps 5-6 ✅
- Protocol file format with `cross_repo_consumers` → Task 2 Steps 1-4 ✅
- `web/` namespace rationale (in INDEX.md description) → Task 1 Steps 1, 4 ✅
- Out-of-scope issues already filed: #121, #122, #123 ✅

**Placeholder scan:** No TBD/TODO. Hex suffixes are generated at write
time (shown as `<hex>` in the plan template but the step explicitly says
to generate). All file paths are exact.

**Type consistency:** Garden protocol path `web/lit-immutable-collections.md`
referenced consistently in Task 1 Step 2 (creates it) and Task 2 Step 3
(references it from web-component-strategy.md). ✅
