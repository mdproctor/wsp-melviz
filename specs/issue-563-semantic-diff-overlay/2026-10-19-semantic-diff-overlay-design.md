# Semantic Diff Overlay for Step Viewer

**Issue:** #563
**Epic:** #558 — Semantic step review viewer
**Phase:** 4 (Diff — final maturation phase, depends on stable rendering)
**Scope:** Single-file step-level diff (v1). Multi-file application-level diff is v3.

## Overview

Build a diff layer that compares two parsed `ResolvedStep[]` trees and renders
additions, removals, and modifications as visual overlays on the existing step
viewer components. The diff computation lives in yaml-core; the rendering
extends the existing `diffState` property pattern already present in
`step-card`, `swim-lane-view`, and `barrier-bar`.

## Architecture

### Diff Computation (yaml-core)

A new sub-path export `"./step/diff"` in yaml-core's package.json, pointing
to `src/step/diff.ts`. This follows the sub-path export pattern established
by `./step`, `./schema`, `./orchestration` but at a nested level — consumers
who don't need diff don't import it, keeping the core `./step` export clean.

**Public API:**

```typescript
function diffSteps(oldSteps: ResolvedStep[], newSteps: ResolvedStep[]): DiffResult;
```

**Result types:**

All types below are defined in `@casehubio/yaml-core/step/diff` — the
canonical location for diff-related types.

```typescript
interface DiffResult {
  root: DiffNode[];          // merged sequence — includes removed nodes at correct positions
  summary: DiffStat[];
}

interface DiffNode {
  state: DiffState;          // 'added' | 'removed' | 'modified' | 'unchanged'
  step: ResolvedStep;        // new step for added/modified/unchanged; old step for removed
  oldStep?: ResolvedStep;    // old step for 'modified' containers — enables scope metadata access
  fieldChanges: FieldChange[]; // pre-computed field changes (empty for added/removed/unchanged)
  children?: DiffNode[];     // single-scope containers (block, race, unnamed parallel)
  scopes?: Record<string, DiffNode[]>; // multi-scope containers — see §Diff Propagation
}

interface FieldChange {
  field: string;             // dot-path within the step (e.g., "params.threshold", "decorators.retry")
  oldValue: unknown;
  newValue: unknown;
}

type DiffState = 'added' | 'removed' | 'modified' | 'unchanged';

type DiffStatState = Exclude<DiffState, 'unchanged'>;

interface DiffStat {
  label: string;             // e.g., "1 branch added"
  count: number;
  state: DiffStatState;      // only changed states — 'unchanged' is never emitted
}
```

**Type ownership migration:**

`DiffState` and `DiffStat` are currently defined in pages-step-viewer
(step-card.ts and diff-summary.ts respectively). Since `DiffNode`
references both types and lives in yaml-core, and pages-step-viewer
depends on yaml-core (not the reverse), these types must migrate to
yaml-core to avoid a circular dependency:

1. `DiffState` and `DiffStat` canonical definitions move to
   `@casehubio/yaml-core/step/diff` (the new `src/step/diff.ts` module)
2. pages-step-viewer's step-card.ts removes the local `DiffState`
   definition and imports from `@casehubio/yaml-core/step/diff`
3. pages-step-viewer's diff-summary.ts removes the local `DiffStat`
   definition and imports from `@casehubio/yaml-core/step/diff`
4. pages-step-viewer's index.ts re-exports both types from the
   yaml-core import for backward compatibility of the public API

**Breaking changes to migrated types:**

- **`DiffState`**: widened from `'added' | 'removed' | 'modified'` to
  include `'unchanged'`. `renderDiffMarker()` in step-card.ts uses
  `Record<DiffState, string>` which requires a fourth entry. Components
  must treat `'unchanged'` as "no visual treatment" — no badge, no CSS
  match, no background change.
- **`DiffStat`**: field `kind` renamed to `state` (now typed
  `DiffStatState` — excludes `'unchanged'`). The existing `DiffSummary`
  component's `s.kind` references and gallery section 21's
  `{ kind: 'added', ... }` literals must be migrated.

Call sites that update for the migration:
step-card.ts:8 (definition → import), step-card.ts:15 (property type),
step-card.ts:212 (Record<DiffState>), barrier-bar.ts:4 (import path),
barrier-bar.ts:9 (property type), swim-lane-view.ts:5 (import path),
swim-lane-view.ts:16-17 (property types), swim-lane-view.ts:148
(Record<DiffState>), diff-summary.ts:4 (definition → import),
index.ts:2 (re-export source), index.ts:31 (re-export source).

### Step Identity Model (Matching)

The diff algorithm walks both step trees and matches steps for comparison.
Matching is **two-pass** within each scope — name matching takes priority
over position matching:

1. **Name pass:** match all named steps (`step: X`) by name within their
   parent container. Named branches in parallel steps and select steps
   also match by branch name. This pass establishes name-based pairings
   regardless of position.
2. **Position pass:** match remaining unnamed steps by position among the
   unmatched slots left after the name pass.
3. **Kind changes** at the same position (unnamed) or with the same name
   (named) are treated as remove + add.
4. **Unnamed parallel children** (no `branches` field — children in flat
   `steps` array) match by position, same as pass 2.
5. **Renames** show as remove + add (no heuristic rename detection in v1).

The two-pass approach prevents mismatches when named and unnamed steps are
interleaved. Example: old `[step:A, unnamed:X, step:B]` → new
`[step:B, unnamed:Y, step:A]`. Name pass matches A↔A and B↔B by name.
Position pass matches X↔Y among the remaining unnamed slots. A naive
single-pass sequential comparison would incorrectly pair A↔B by position.

Each container's child arrays are independent matching scopes:
- `if-else`: `thenSteps` and `elseSteps` are separate scopes
- `parallel`: each `branches[i].steps` is a separate scope; when `branches`
  is undefined, the flat `steps` array is a single scope
- `try-catch-finally`: `trySteps`, `catchSteps`, `finallySteps` are separate
- `match`: each `cases[i].steps` is separate
- `select`: each `branches[i].steps` is separate

Position-based matching for unnamed steps means an insertion at position 2 in
a block of 5 reports 4 modifications. Accepted for v1 — LLMs tend to
regenerate entire sections rather than surgically insert.

### Field-Level Diff

For modified steps, the diff algorithm pre-computes field changes and
includes them in `DiffNode.fieldChanges`. The algorithm compares **all
scalar fields** on the step interface — not just `params` and
`decorators`. Any scalar or optional-scalar field that differs between
old and new produces a `FieldChange` entry.

**Field categories:**

- **Scalar parameter changes:** any scalar value in `params` that differs
  (PluginStep only)
- **Container structural fields:** all scalar fields on container step
  interfaces. Cross-referenced against walker.ts:

  | Step type | Field | Type |
  |-----------|-------|------|
  | IfElseStep | `condition` | `string` |
  | MatchStep | `scrutinee` | `string` |
  | ParallelStep | `advanceAfter` | `number?` |
  | ParallelStep | `cancelRemaining` | `boolean?` |
  | ParallelStep | `collectionTimeout` | `number?` |
  | BarrierStep | `timeout` | `number?` |
  | QuorumStep | `required` | `number` |
  | QuorumStep | `timeout` | `number?` |
  | DelayStep | `duration` | `number` |

- **Derived scalar changes:** for array-type fields, the algorithm emits
  synthetic scalar entries for the array length when it changes. Examples:
  - `{field: "awaitSteps.length", oldValue: 3, newValue: 4}` — barrier count
  - `{field: "ofSteps.length", oldValue: 3, newValue: 4}` — quorum pool size
  - `{field: "steps.length", oldValue: 2, newValue: 3}` — child step count
  This enables the flagship mockup examples (barrier 3→4, quorum 2/3→3/4)
  without requiring a separate `ArrayChange` type.
- **Decorator changes:** additions, removals, and value modifications in
  `decorators: Record<string, unknown>`. A step with changed decorators is
  `modified`; the specific changes appear as field changes (e.g.,
  `{field: "decorators.retry", oldValue: 2, newValue: 3}`)
- **Structural metadata:** `name` and `kind` changes (all step types),
  `entry.qualifiedName` changes (PluginStep)
- **Invoke spec (shallow):** for InvokeStep, the algorithm performs a
  shallow first-level comparison of `invokeSpec: Record<string, unknown>`.
  Scalar first-level values (URL, method) produce FieldChange entries.
  Nested object values (body, headers) are compared by reference —
  any change produces a single `{field: "invokeSpec.{key}"}` entry
  without drilling into the nested structure.

v1 emits all field changes (no filtering). v2 will introduce collapsing
for noise fields based on observed real-world patterns.

## Component Extensions

### Existing Components (diff-aware, need `fieldChanges` addition)

These already accept `diffState` with full CSS treatment:

| Component | Existing Support |
|-----------|-----------------|
| `step-card` | `diffState`, diff-marker badge, background/border |
| `swim-lane-view` | `diffState` (existing `branchDiffStates` removed — superseded by `diffNode`) |
| `barrier-bar` | `diffState`, diff-marker badge |
| `diff-summary` | Renders `DiffStat[]` (field `kind` → `state` rename) |

**Addition:** All components gain `fieldChanges: FieldChange[]` property
and render inline `old → new` spans using a shared template.

### Components to Extend

These need `diffState: DiffState` property, corresponding CSS, and
`diffNode: DiffNode` for container-level diff propagation:

| Component | Diff Semantics | Scope keys |
|-----------|---------------|------------|
| `if-else-view` | Per-branch diff (then/else highlighted independently) | `then`, `else` |
| `match-view` | Per-case diff (each case highlighted independently) | `case-{index}` |
| `try-catch-view` | Per-block diff (try/catch/finally independent) | `try`, `catch`, `finally` |
| `step-block-view` | Whole-block diff state | uses `children` (single-scope) |
| `race-view` | Per-branch diff | uses `children` (single-scope) |
| `select-view` | Per-subscription diff | `{branchName}` |
| `invoke-card` | Added/removed/modified as leaf | — (leaf, no children) |
| `state-machine-view` | Added/removed/modified as leaf | — (leaf, no children) |
| `error-step` | Added/removed/modified as leaf | — (leaf, no children) |

### New Component: `<pages-delay-step>`

The `delay` step kind is currently rendered as inline HTML in `step-dispatch.ts`
with no component — diff state has no attachment point. Extract to a
`<pages-delay-step>` Lit element with `step: DelayStep`, `diffState: DiffState`,
and `fieldChanges: FieldChange[]` properties. Rendering matches the current
inline HTML (timer icon + duration badge).

### Diff CSS System

Three visual states. The raw Catppuccin values below are the mockup
reference — implementation must define these as `--pages-diff-*` CSS
custom properties under the `--pages-` token namespace (per
css-design-tokens protocol), mapped from `--pages-success-*`,
`--pages-danger-*`, and `--pages-warning-*` semantic scales:

| State | Background | Border | Badge |
|-------|-----------|--------|-------|
| Added | `--pages-diff-added-bg` | `--pages-diff-added-border` | `+ NEW` green |
| Removed | `--pages-diff-removed-bg` | `--pages-diff-removed-border` | `- REMOVED` red, line-through, 0.7 opacity |
| Modified | `--pages-diff-modified-bg` | `--pages-diff-modified-border` | `MODIFIED` amber |
| Unchanged | No visual treatment | No visual treatment | No badge |

**Migration:** Existing diff CSS in `step-card.ts`, `swim-lane-view.ts`,
and `barrier-bar.ts` uses hardcoded Catppuccin `rgba()` values. This issue
migrates all existing hardcoded diff CSS to the new `--pages-diff-*` token
system. Same work surface — prevents two parallel CSS systems diverging
when themes change.

### Inline Value Changes

A shared `renderFieldChanges(changes: FieldChange[])` Lit template renders
field name labels with `old → new` for each changed field:

```html
<span class="field-name">threshold</span>: <span class="old-value">5</span> → <span class="new-value">10</span>
```

- `field-name`: muted text, derived from the field path (see display
  name rules below)
- `old-value`: red text, strikethrough
- `new-value`: green text

**Display name rules:** The label shown in `field-name` uses the last
segment of the dot-path by default (e.g., `params.threshold` →
`threshold`, `decorators.retry` → `retry`). Synthetic derived fields
use a display name override table to match what the component renders:

| Field path | Display name | Rationale |
|------------|-------------|-----------|
| `awaitSteps.length` | `count` | barrier-bar renders `count: N` |
| `ofSteps.length` | `of` | barrier-bar renders `N of M` |
| `steps.length` | `steps` | child step count |

All other fields use the last dot-path segment as-is.

Field name labels prevent ambiguity when a step has multiple field
changes — the user sees which field each `old → new` span represents.

## Rendering Pipeline

### `<pages-step-viewer>` Component (new)

This component is the top-level integration point for step rendering,
created as part of this issue. (The parent spec's component hierarchy
envisions `<pages-step-viewer>` with a `<diff-overlay>` child component;
this spec evolves that to a property-based approach — diff state flows
as properties on existing components rather than through a wrapper
element. Properties are simpler and avoid an extra DOM layer.)

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `steps` | `ResolvedStep[]` | Steps to render (non-diff mode) |
| `oldSteps` | `ResolvedStep[]` | Previous version (triggers diff mode) |
| `newSteps` | `ResolvedStep[]` | Current version (used with `oldSteps`) |
| `diffResult` | `DiffResult` | Pre-computed diff (takes precedence) |

**Rendering logic:**

1. If `diffResult` is provided, use it directly
2. If `oldSteps` + `newSteps` are provided, call `diffSteps(oldSteps, newSteps)`
3. If only `steps` is provided, render without diff (backward-compatible)

In diff mode:
- Renders `<diff-summary>` with `DiffResult.summary`
- Iterates `DiffResult.root` through `renderDiffContent()` (see below)

In non-diff mode:
- Iterates `steps` through the existing `renderStepContent()`

### Diff Propagation

The existing `renderStepContent(step: ResolvedStep)` dispatch function
takes only `ResolvedStep` and has no mechanism for threading diff state.
Rather than modifying its signature (which would couple the base dispatch
to diff concerns), a new parallel dispatch function handles diff rendering:

```typescript
function renderDiffContent(node: DiffNode): TemplateResult;
```

`renderDiffContent` unwraps the `DiffNode` and creates the same component
elements as `renderStepContent`, but sets diff-specific properties:

- **Leaf components** (step-card, invoke-card, barrier-bar, pages-delay-step,
  state-machine-view, error-step): sets `diffState` and `fieldChanges`
- **Container components** (swim-lane-view, if-else-view, match-view,
  try-catch-view, step-block-view, race-view, select-view): sets
  `diffState`, `fieldChanges`, and `diffNode`

Container components gain a `diffNode?: DiffNode` property. When set,
they iterate their children from the DiffNode structure instead of from
the step's child arrays:

- **Single-scope containers** (block, race): iterate `diffNode.children`
  through `renderDiffContent()`
- **Multi-scope containers** (if-else, parallel, try-catch, match, select):
  iterate `diffNode.scopes[scopeKey]` through `renderDiffContent()` for
  each scope

When `diffNode` is not set, containers iterate the step's child arrays
through `renderStepContent()` as before — backward-compatible.

**Multi-scope DiffNode structure by container type:**

| Container | Scope keys | Mapped from |
|-----------|-----------|-------------|
| `if-else-view` | `then`, `else` | `thenSteps`, `elseSteps` |
| `try-catch-view` | `try`, `catch`, `finally` | `trySteps`, `catchSteps`, `finallySteps` |
| `swim-lane-view` (named branches) | branch names (e.g., `fast`, `slow`) | `branches[i].name` |
| `swim-lane-view` (unnamed) | uses `children` (single-scope) | flat `steps` array |
| `match-view` | `case-0`, `case-1`, … | `cases[i].steps` |
| `select-view` | branch names | `branches[i].name` |

When `ParallelStep.branches` is undefined (unnamed parallel — children
in flat `steps` array), the DiffNode uses `children` (single-scope), not
`scopes`. swim-lane-view must handle both: named branches read from
`diffNode.scopes`, unnamed read from `diffNode.children`.

**Removed scope metadata via `oldStep`:**

When a scope is removed (exists in old but not in new), the container
component needs the old scope's metadata to render its header. The
`DiffNode.oldStep` field provides access to the old step:

- **match-view**: for removed `case-N`, reads `oldStep.cases[N].pattern`
  and `oldStep.cases[N].guard` to render the case header
- **select-view**: for removed branches, reads `oldStep.branches` to find
  the branch's `type` ('subscribe'/'wait') and `name`
- **if-else-view, try-catch-view**: NOT affected — scope keys are fixed
  labels, not data-driven metadata
- **swim-lane-view**: NOT affected — scope keys are branch names, which
  are sufficient for rendering the lane header

### Merged Sequence Output

`DiffResult.root` is the merged sequence — removed nodes appear at their
correct interleaved positions relative to unchanged and added nodes. The
same applies to `DiffNode.children` and `DiffNode.scopes[key]` at every
level of the tree. The renderer iterates the sequence in order; no
interleaving logic is needed at render time.

**Ordering rule for removed nodes:** The merged sequence follows the new
tree's order. Removed nodes are inserted after the last new-tree node
whose old-tree position preceded the removed node's old-tree position.
If no such node exists, removed nodes appear at the beginning of the
sequence. This rule is deterministic, simple to implement, and produces
intuitive results for the common case (sequential removals without
reordering).

For removed steps, the `DiffNode` contains the old step with
`state: 'removed'` — the component renders it with red background,
strikethrough, and `- REMOVED` badge.

### Layered API

The diff function `diffSteps()` is independently callable from yaml-core.
Consumers that need the diff result for multiple purposes (e.g., gallery
needs `DiffStat[]` for the summary bar AND the full `DiffResult` for the
viewer) can call it directly and pass the pre-computed result.

The viewer accepts either:
- `oldSteps` + `newSteps` (convenience — diffs internally)
- A pre-computed `DiffResult` (when the caller already has it)

## Gallery Integration

**Replaces existing section 21.** The current gallery section 21 demonstrates
a diff overlay with imperatively-applied `diffState` and `branchDiffStates`
— a protocol violation (gallery-renders-from-yaml-only). This issue
replaces it with a proper YAML-driven implementation:

- Two side-by-side CodeMirror YAML editors labeled "Old" and "New"
- Pre-loaded with example YAML showing a barrier change (matching mockup
  section 10 — adding a 4th branch, bumping barrier count 3→4)
- Unified diff rendering below, updating live as either YAML is edited
- Calls `diffSteps()` on parsed YAML — renders exclusively from real YAML
  (protocol: gallery-renders-from-yaml-only)

## Diff Summary Bar

Rendered by the existing `<diff-summary>` component at the top of the
diff view. Shows aggregate statistics:

```
● 1 branch added  ● 2 steps modified  ● 1 barrier count changed
```

Each stat has a colored dot matching its diff state (green/amber/red).
The `DiffStat[]` array is computed by `diffSteps()` as part of the
`DiffResult`.

**Label generation:** The diff algorithm derives labels from the parent
container kind and the nature of the change:

| Context | Label pattern |
|---------|--------------|
| Parallel container children | `N branch(es) added/removed/modified` |
| Block/sequence children | `N step(s) added/removed/modified` |
| Barrier `awaitSteps.length` change | `barrier count changed` |
| Quorum `required` or `ofSteps.length` change | `quorum threshold changed` |
| If-else / match / try-catch children | `N step(s) added/removed/modified` |
| Decorator changes | `N decorator(s) changed` |
| Select subscription changes | `N subscription(s) added/removed/modified` |

Labels are pluralized based on count. Stats with count 0 are omitted.
DiffStat entries are only emitted for changed states (added, removed,
modified). Unchanged items are never counted in the summary — enforced
at the type level by `DiffStatState = Exclude<DiffState, 'unchanged'>`.

## Testing Strategy

### Unit Tests (yaml-core)

- `diffSteps` with identical trees → all unchanged, empty summary
- Named step added/removed/modified → correct DiffNode states
- Unnamed step position matching → correct pairing
- Kind change at same position → remove + add pair
- Named step with kind change → remove + add pair
- Nested containers (parallel with blocks inside) → correct tree shape
- Multi-scope containers (if-else, try-catch) → correct scope keys
- Decorator changes → field-level diff entries
- Parameter changes → field-level diff entries
- Container structural fields (condition, scrutinee, duration, timeout, required) → FieldChange
- InvokeStep.invokeSpec shallow diff → first-level scalar FieldChange entries
- Two-pass matching: named steps matched by name before unnamed by position
- Interleaved named/unnamed steps → correct pairings (name pass, then position pass)
- Array-length derived changes (awaitSteps.length) → synthetic FieldChange
- Empty old tree (all added) / empty new tree (all removed)
- Removed steps appear at correct interleaved positions in merged sequence
- Removed named steps follow ordering rule (after last preceding new-tree node)
- Modified container DiffNode carries `oldStep` for metadata access
- Unnamed parallel children (no branches) → position matching, uses `children` not `scopes`
- DiffStat label generation matches expected vocabulary
- DiffStat never emits entries with `'unchanged'` state
- Select branches match by branch name (same as parallel)

### Component Tests (pages-step-viewer)

- Each extended component renders correct CSS classes for each DiffState
- `fieldChanges` render inline old→new spans
- Removed steps render with strikethrough and badge
- `renderDiffContent` dispatch creates correct elements with diff properties
- Container components with `diffNode` iterate diff children correctly
- Container components without `diffNode` iterate step children (backward compat)
- Viewer with only `steps` (no oldSteps) renders without diff — backward compat
- Viewer with `oldSteps` + `newSteps` produces correct diff rendering
- New `<pages-delay-step>` component renders with diff state
- match-view renders removed case headers using `oldStep` metadata
- select-view renders removed branch headers using `oldStep` metadata
- swim-lane-view with unnamed parallel uses `diffNode.children`
- `fieldChanges` render with field name labels (not bare values)

### Gallery Visual Tests

- Diff section renders correctly with pre-loaded example YAML
- Editing either YAML pane updates the diff view
- Matches mockup section 10 visual output

## References

- docs/specs/2026-10-17-step-review-viewer-design.md — D3 (unified diagram), D10 (single-file v1), §Diff Architecture
- docs/mockups/step-review-viewer.html — sections 10 (barrier diff), 16 (agent coordination diff)
- packages/yaml-core/src/step/walker.ts — ResolvedStep discriminated union, NamedBranch
- packages/pages-step-viewer/src/step-card.ts — existing DiffState type (migrates to yaml-core), diffState property
- packages/pages-step-viewer/src/swim-lane-view.ts — branchDiffStates (removed, superseded by diffNode)
- packages/pages-step-viewer/src/diff-summary.ts — existing DiffStat (migrates to yaml-core), DiffSummary component
- packages/pages-step-viewer/src/barrier-bar.ts — existing diffState support
- docs/protocols/casehub/gallery-renders-from-yaml-only.md — gallery data integrity
- docs/protocols/casehub/web-component-strategy.md — Lit conventions, pages- prefix
- docs/protocols/casehub/css-design-tokens.md — --pages- token naming
