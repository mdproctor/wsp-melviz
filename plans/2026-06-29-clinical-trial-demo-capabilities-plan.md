# Clinical Trial Demo Capabilities — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add runtime context resolution, conditional visibility, content interpolation, parameterised URLs, action buttons, form submit, and 7 new visualization components to casehub-pages.

**Architecture:** A unified `RuntimeContext` provides filter state, dataset snapshots, page state, and parameters. Components reference context via `#{path}` templates evaluated at runtime. A `PagesContentElement` base class supports non-data-bound Web Components. An `ActionExecutor` handles REST POST operations for action buttons and form submit.

**Tech Stack:** TypeScript 5, Vitest, ECharts (custom series for timeline, graph series for network viz), existing pages-* monorepo packages.

## Global Constraints

- All classes use `Pages*` prefix, tags use `pages-*`, events use `pages-*`, CSS properties use `--pages-*` (from #55 rename)
- Expression evaluator must NOT use `new Function()` — implement a custom parser (CSP compliance, #16)
- All new Web Components use shadow DOM with CSS custom properties for theming
- ARIA attributes required on all new components (accessibility, #15)
- All commits reference `Refs #50` and the specific child issue number
- Tests use Vitest (packages) and Playwright (examples)
- Build order: packages → components → webapp. Run `yarn typecheck` for cross-package verification.

---

## File Structure

### New files

| File | Responsibility |
|------|---------------|
| `packages/pages-component/src/context/types.ts` | `RuntimeContext`, `DataSetSnapshot`, `EscapeMode` types |
| `packages/pages-component/src/context/template-parser.ts` | `resolveTemplate()` — `#{}` interpolation with context-aware escaping |
| `packages/pages-component/src/context/expression-evaluator.ts` | `evaluateExpression()` — boolean condition evaluation |
| `packages/pages-component/src/context/index.ts` | Barrel exports |
| `packages/pages-component/src/model/action-types.ts` | `ActionButtonProps`, `SubmitConfig`, `ActionRequest`, `ActionCallbacks` |
| `packages/pages-runtime/src/context-wiring.ts` | `ContextManager` — state tracking, consumer registry, evaluation cascade |
| `packages/pages-runtime/src/action.ts` | `ActionExecutor` — HTTP action execution with host fetch injection |
| `packages/pages-viz/src/base/PagesContentElement.ts` | Base class for content Web Components (alert, action-button) |
| `packages/pages-viz/src/components/PagesAlert.ts` | Alert banner component |
| `packages/pages-viz/src/components/PagesActionButton.ts` | Action button component |
| `packages/pages-viz/src/components/PagesBadge.ts` | Status badge component |
| `packages/pages-viz/src/components/PagesCountdown.ts` | Countdown timer component |
| `packages/pages-viz/src/charts/PagesTimeline.ts` | Timeline/Gantt chart (ECharts custom series) |
| `packages/pages-viz/src/charts/PagesGraph.ts` | Network graph chart (ECharts graph series) |

### Modified files

| File | Changes |
|------|---------|
| `packages/pages-component/src/model/types.ts` | Add `visibleWhen?: string` to `Component` interface |
| `packages/pages-component/src/model/displayer-types.ts` | Add `BadgeProps`, `CountdownProps`, `TimelineProps`, `GraphProps`, `RowStyleRule`, `ExpandableConfig`; extend `TableProps` |
| `packages/pages-component/src/index.ts` | Export context types |
| `packages/pages-data/src/dataset/manager.ts` | Add `onChanged` callback to `createDataSetManager()` |
| `packages/pages-runtime/src/site.ts` | Integrate `ContextManager`, listen for `pages-action-request`/`pages-action-complete` |
| `packages/pages-runtime/src/activation.ts` | Add `DATA_COMPONENT_TYPES` entries, content Web Component activation path, `visibleWhen` handling |
| `packages/pages-runtime/src/content.ts` | No signature change — called by context wiring for re-renders |
| `packages/pages-runtime/src/data-pipeline.ts` | Parameterised URL resolution, AbortController, tree-table pipeline bypass |
| `packages/pages-ui/src/parser/displayer-desugar.ts` | Add TYPE_MAP entries, extraction blocks for badge/countdown/timeline/graph/rowStyle/expandable/sortable/resizable |
| `packages/pages-ui/src/parser/component-desugar.ts` | Add `alert:`, `action-button:` content shorthand branches, `visibleWhen` extraction |
| `packages/pages-viz/src/custom-elements.ts` | Register 6 new elements |
| `packages/pages-viz/src/components/PagesTable.ts` | Row-level styling, expandable rows |
| `packages/pages-viz/src/form-inputs/PagesFormInput.ts` | Add `submit` support |

---

## Task 1: Context Types, Template Parser, Expression Evaluator

**Files:**
- Create: `packages/pages-component/src/context/types.ts`
- Create: `packages/pages-component/src/context/template-parser.ts`
- Create: `packages/pages-component/src/context/template-parser.test.ts`
- Create: `packages/pages-component/src/context/expression-evaluator.ts`
- Create: `packages/pages-component/src/context/expression-evaluator.test.ts`
- Create: `packages/pages-component/src/context/index.ts`
- Modify: `packages/pages-component/src/index.ts`

**Interfaces:**
- Consumes: Nothing — pure functions, zero dependencies
- Produces:
  - `RuntimeContext` type (used by Tasks 2-19)
  - `DataSetSnapshot` type (used by Tasks 2, 3)
  - `EscapeMode` enum: `'html' | 'markdown' | 'url' | 'none'`
  - `resolveTemplate(template: string, context: RuntimeContext, escape: EscapeMode): string`
  - `hasTemplateVars(str: string): boolean`
  - `allTemplateVarsResolved(template: string, context: RuntimeContext): boolean`
  - `evaluateExpression(expression: string, context: RuntimeContext): boolean`
  - `createRowContext(base: RuntimeContext, row: Record<string, unknown>): RuntimeContext`

- [ ] **Step 1: Write context types**

```typescript
// packages/pages-component/src/context/types.ts
export interface RuntimeContext {
  readonly filter: Record<string, readonly string[]>;
  readonly datasets: Record<string, DataSetSnapshot>;
  readonly page: { readonly name: string; readonly path: string };
  readonly params: Record<string, string>;
  readonly row?: Record<string, unknown>;
}

export interface DataSetSnapshot {
  readonly rowCount: number;
  readonly columns: readonly string[];
  readonly first?: Record<string, string | number | null>;
}

export type EscapeMode = "html" | "markdown" | "url" | "none";

export const EMPTY_CONTEXT: RuntimeContext = {
  filter: {},
  datasets: {},
  page: { name: "", path: "" },
  params: {},
};
```

- [ ] **Step 2: Write failing template parser tests**

Test cases covering:
- Basic interpolation: `"Hello #{params.name}"` → `"Hello Alice"`
- Filter first-element: `"#{filter.ward}"` with `filter: { ward: ["ICU", "CCU"] }` → `"ICU"`
- Empty filter: `"#{filter.ward}"` with `filter: { ward: [] }` → `""`
- Nested path: `"#{datasets.patients.first.name}"` → resolved value
- Dataset rowCount: `"#{datasets.patients.rowCount}"` → `"25"`
- Missing path: `"#{filter.missing}"` → `""`
- HTML escaping: `resolveTemplate("#{filter.name}", ctx, "html")` where name contains `<script>` → escaped
- Markdown escaping: value `"*bold*"` → `"\\*bold\\*"` then HTML-escaped
- URL escaping: value `"hello world"` → `"hello%20world"`
- No-escape mode: raw value passed through
- `hasTemplateVars()`: true for strings containing `#{`, false otherwise
- `allTemplateVarsResolved()`: false when any `#{}` path resolves to undefined/empty
- No templates: `"plain text"` → `"plain text"`
- Multiple templates: `"#{filter.a} and #{filter.b}"` → both resolved

Run: `yarn workspace @casehub/pages-component run test -- --run context/template-parser`
Expected: FAIL — module not found

- [ ] **Step 3: Implement template parser**

Core logic:
1. Regex match `/#\{([^}]+)\}/g` to find all template expressions
2. For each match, resolve the dot-path against the context object
3. For filter values (array), use first element; empty array → `""`
4. Apply escaping based on `EscapeMode`
5. Replace template with resolved (and escaped) value

Markdown escaping: escape `*_`[#~` with backslash prefix, THEN HTML-entity escape `<>"&`.

- [ ] **Step 4: Run template parser tests**

Run: `yarn workspace @casehub/pages-component run test -- --run context/template-parser`
Expected: PASS

- [ ] **Step 5: Write failing expression evaluator tests**

Test cases covering:
- Truthy: `"#{filter.ward}"` with non-empty filter → `true`
- Truthy: `"#{filter.ward}"` with empty/missing filter → `false`
- Equality: `"#{filter.ward} == 'ICU'"` → `true` when first element is `"ICU"`
- Equality numeric coercion: `"#{row.score} == 3"` with `row.score = "3"` → `true` (both parseable as numbers)
- Comparison: `"#{filter.grade} >= 4"` → numeric comparison
- Negation: `"!#{filter.ward}"` → inverts truthy
- AND: `"#{filter.a} && #{filter.b}"` → both must be truthy
- OR: `"#{filter.a} || #{filter.b}"` → either truthy
- Parentheses: `"(#{row.x} == 'a' && #{row.y} == 'b') || #{row.z} == 'c'"`
- Precedence: `"#{a} || #{b} && #{c}"` → AND binds tighter
- String comparison: `"#{row.name} > 'M'"` → lexicographic
- Null literal: `"#{filter.ward} != null"` → true when filter exists
- Row-scoped: `createRowContext(base, { status: "Critical" })` then `"#{row.status} == 'Critical'"` → true

Run: `yarn workspace @casehub/pages-component run test -- --run context/expression-evaluator`
Expected: FAIL

- [ ] **Step 6: Implement expression evaluator**

Recursive descent parser with operator precedence:
1. Tokenizer: splits on operators, parentheses, string literals, numeric literals, `true`/`false`/`null`, and `#{...}` references
2. Parser: `parseOr()` → `parseAnd()` → `parseEquality()` → `parseComparison()` → `parseUnary()` → `parsePrimary()`
3. Each `#{...}` reference resolves against context using the same path resolver as the template parser (no escaping — `EscapeMode.none`)
4. Type coercion: if both operands parse as finite numbers, compare numerically; otherwise compare as strings. Consistent across `==`, `!=`, `>`, `<`, `>=`, `<=`.

`createRowContext()`: returns a new `RuntimeContext` with the `row` property set. Shallow — row values override only the `row` namespace.

- [ ] **Step 7: Run expression evaluator tests**

Run: `yarn workspace @casehub/pages-component run test -- --run context/expression-evaluator`
Expected: PASS

- [ ] **Step 8: Create barrel exports and integrate**

```typescript
// packages/pages-component/src/context/index.ts
export { type RuntimeContext, type DataSetSnapshot, type EscapeMode, EMPTY_CONTEXT } from "./types.js";
export { resolveTemplate, hasTemplateVars, allTemplateVarsResolved } from "./template-parser.js";
export { evaluateExpression, createRowContext } from "./expression-evaluator.js";
```

Add to `packages/pages-component/src/index.ts`:
```typescript
export * from "./context/index.js";
```

- [ ] **Step 9: Run full package typecheck and commit**

Run: `yarn typecheck`
Expected: PASS (no cross-package type errors)

```
git add packages/pages-component/src/context/
git commit -m "feat: add context resolution types, template parser, expression evaluator

Pure functions for #{} runtime template resolution and boolean
expression evaluation. Supports context-aware escaping (HTML, markdown,
URL), operator precedence with parentheses, and row-scoped context.

Refs #50, Refs #47, Refs #48, Refs #49"
```

---

## Task 2: DataSetManager onChanged Callback

**Files:**
- Modify: `packages/pages-data/src/dataset/manager.ts`
- Create: `packages/pages-data/src/dataset/manager-callback.test.ts`

**Interfaces:**
- Consumes: `TypedDataSet`, `DataSetId` from existing pages-data types
- Produces:
  - `DataSetManagerOptions` type: `{ onChanged?: (id: DataSetId, dataset: TypedDataSet) => void }`
  - Updated `createDataSetManager(options?: DataSetManagerOptions): DataSetManager`

- [ ] **Step 1: Write failing tests for onChanged callback**

Test cases:
- `register()` triggers `onChanged` with the registered dataset
- `accumulate()` triggers `onChanged` with the accumulated dataset
- `remove()` does NOT trigger `onChanged`
- Multiple `register()` calls trigger `onChanged` each time
- No callback provided → no error (callback is optional)

Run: `yarn workspace @casehub/pages-data run test -- --run dataset/manager-callback`
Expected: FAIL

- [ ] **Step 2: Implement onChanged callback**

Modify `createDataSetManager()` to accept optional `DataSetManagerOptions`. In `DataSetManagerImpl`:
- Store the callback
- Call `this.options?.onChanged?.(id, dataset)` at the end of `register()` and `accumulate()`

- [ ] **Step 3: Run tests**

Run: `yarn workspace @casehub/pages-data run test -- --run dataset/manager-callback`
Expected: PASS

Run existing manager tests to verify no regressions:
Run: `yarn workspace @casehub/pages-data run test -- --run dataset/manager`
Expected: PASS

- [ ] **Step 4: Commit**

```
git add packages/pages-data/src/dataset/manager.ts packages/pages-data/src/dataset/manager-callback.test.ts
git commit -m "feat: add onChanged callback to DataSetManager

Invoked after register() and accumulate() — enables runtime to build
DataSetSnapshot for the context model without polling.

Refs #50, Refs #47"
```

---

## Task 3: Context Wiring — ContextManager

**Files:**
- Create: `packages/pages-runtime/src/context-wiring.ts`
- Create: `packages/pages-runtime/src/context-wiring.test.ts`

**Interfaces:**
- Consumes:
  - `RuntimeContext`, `DataSetSnapshot`, `resolveTemplate`, `evaluateExpression`, `hasTemplateVars`, `allTemplateVarsResolved`, `createRowContext`, `EscapeMode` from `@casehub/pages-component/context`
  - `TypedDataSet`, `DataSetId` from `@casehub/pages-data`
- Produces:
  - `ContextManager` class:
    - `constructor()`
    - `getContext(): RuntimeContext`
    - `updateFilter(filter: Record<string, readonly string[]>): void`
    - `updateDataset(id: DataSetId, dataset: TypedDataSet): void`
    - `updatePage(name: string, path: string): void`
    - `updateParams(params: Record<string, string>): void`
    - `registerConsumer(consumer: ContextConsumer): void`
    - `deregisterConsumer(element: Element): void`
    - `evaluateAll(): void`
  - `ContextConsumer` interface:
    - `element: Element`
    - `templates: Map<string, { template: string; escapeMode: EscapeMode; lastResolved: string; apply: (value: string) => void }>`
    - `visibleWhen?: { expression: string; lastResult: boolean; onSuspend: () => void; onResume: () => void }`
    - `suspended: boolean`

- [ ] **Step 1: Write failing ContextManager tests**

Test cases:
- `updateFilter()` produces new `RuntimeContext` with updated filter values
- `updateDataset()` builds `DataSetSnapshot` and produces new context
- Consumer with template registered → `evaluateAll()` calls `apply()` with resolved value
- Consumer with `visibleWhen` → `evaluateAll()` calls `onSuspend()`/`onResume()` on transition
- Suspended consumer → only `visibleWhen` re-evaluated, templates skipped
- Cascade: dataset URL change triggers re-fetch callback, which updates dataset, which re-evaluates content consumers
- Cascade depth guard: circular dependency logs warning and halts at depth 10
- `deregisterConsumer()` removes from registry — no further evaluations
- Stale consumer pruning: consumer with `element.isConnected === false` is removed during evaluation

Run: `yarn workspace @casehub/pages-runtime run test -- --run context-wiring`
Expected: FAIL

- [ ] **Step 2: Implement ContextManager**

Core implementation:
1. Holds current `RuntimeContext` (immutable — replaced on each update)
2. Holds `Set<ContextConsumer>` registry
3. On any `update*()` call: build new context, call `evaluateAll()`
4. `evaluateAll()` iterates consumers:
   - If suspended: evaluate only `visibleWhen`. If transitions to truthy → resume.
   - If active with `visibleWhen` that transitions to falsy → suspend.
   - If active: evaluate all templates, compare to `lastResolved`, call `apply()` if changed.
5. Cascade: `evaluateAll()` increments a depth counter. If a consumer's `apply()` triggers an `update*()` call (e.g., dataset URL change → fetch → `updateDataset()`), the re-entrant `evaluateAll()` uses the same depth counter. Halt at 10.
6. `buildSnapshot()`: construct `DataSetSnapshot` from `TypedDataSet` — rowCount, column IDs, first row values.

- [ ] **Step 3: Run tests**

Run: `yarn workspace @casehub/pages-runtime run test -- --run context-wiring`
Expected: PASS

- [ ] **Step 4: Commit**

```
git add packages/pages-runtime/src/context-wiring.ts packages/pages-runtime/src/context-wiring.test.ts
git commit -m "feat: add ContextManager — state tracking, consumer registry, evaluation cascade

Manages RuntimeContext lifecycle: filter/dataset/page/params updates
trigger consumer re-evaluation with cascade depth guard (max 10).
Suspension model for visibleWhen: only the condition is re-evaluated
for suspended consumers.

Refs #50, Refs #47, Refs #48, Refs #49"
```

---

## Task 4: visibleWhen Property + Component Model

**Files:**
- Modify: `packages/pages-component/src/model/types.ts`
- Modify: `packages/pages-runtime/src/activation.ts`
- Create: `packages/pages-runtime/src/visible-when.test.ts`

**Interfaces:**
- Consumes: `ContextManager` from Task 3, `Component` type
- Produces:
  - `visibleWhen?: string` property on `Component` interface
  - Activation callback registers `visibleWhen` consumers with `ContextManager`

- [ ] **Step 1: Add visibleWhen to Component interface**

```typescript
// packages/pages-component/src/model/types.ts — add to Component interface
export interface Component<
  T extends string = string,
  P extends object = Record<string, unknown>,
> {
  readonly type: T;
  readonly id?: string;
  readonly visibleWhen?: string;  // NEW
  readonly props?: Readonly<P>;
  readonly style?: Readonly<Record<string, string>>;
  readonly access?: AccessControl;
  readonly slots?: Readonly<Record<string, readonly Component[]>>;
  readonly items?: readonly GridItem[];
}
```

- [ ] **Step 2: Write failing tests for visibleWhen activation**

Test cases:
- Component with `visibleWhen: "#{filter.ward}"` → hidden initially (filter empty), shown when filter set
- Suspension: hidden component stops refresh timer, suppresses data request
- Resume: shown component re-evaluates templates, re-fetches if URL changed
- Static `visible: false` → component not rendered at activation (hidden attribute set, never re-evaluated)
- `visibleWhen` takes precedence over static `visible`

Run: `yarn workspace @casehub/pages-runtime run test -- --run visible-when`
Expected: FAIL

- [ ] **Step 3: Implement visibleWhen in activation callback**

In `createActivationCallback()`:
1. After creating the component element, check `component.visibleWhen`
2. If present, register a `ContextConsumer` with the `ContextManager`:
   - `visibleWhen.expression = component.visibleWhen`
   - `visibleWhen.onSuspend = () => { el.hidden = true; /* stop refresh timer if applicable */ }`
   - `visibleWhen.onResume = () => { el.hidden = false; /* restart timer, re-evaluate other templates */ }`
3. Also check static `visible` on props: if `props.visible === false`, set `el.hidden = true` at activation

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/pages-runtime run test -- --run visible-when`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add packages/pages-component/src/model/types.ts packages/pages-runtime/src/activation.ts packages/pages-runtime/src/visible-when.test.ts
git commit -m "feat: add visibleWhen conditional visibility with suspension model

Components with visibleWhen hide via hidden attribute when expression
is falsy. Suspended components stop refresh timers, suppress fetches,
and freeze template evaluation. Static visible: false enforced at
activation time.

Refs #50, Refs #47"
```

---

## Task 5: Content Interpolation + Parameterised URLs

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts`
- Modify: `packages/pages-runtime/src/content.ts`
- Modify: `packages/pages-runtime/src/data-pipeline.ts`
- Modify: `packages/pages-runtime/src/site.ts`
- Create: `packages/pages-runtime/src/content-interpolation.test.ts`
- Create: `packages/pages-runtime/src/parameterised-urls.test.ts`

**Interfaces:**
- Consumes: `ContextManager` (Task 3), `resolveTemplate`, `hasTemplateVars`, `allTemplateVarsResolved` (Task 1)
- Produces:
  - Content interpolation: `html:`, `markdown:`, `title:` with `#{}` re-render on context change
  - Parameterised URLs: datasets with `#{}` in URL re-fetch when resolved URL changes
  - Deferred fetch: unresolved variables suppress fetch
  - Request cancellation: `AbortController` per parameterised dataset

- [ ] **Step 1: Write failing content interpolation tests**

Test cases:
- `markdown: { content: "Ward: #{filter.ward}" }` → resolves on activation, re-renders on filter change
- Markdown escaping: filter value `"*ICU*"` → rendered as literal `*ICU*` not italic
- HTML escaping: filter value `"<b>bold</b>"` → rendered as escaped text
- `title: { text: "#{datasets.patients.rowCount} patients" }` → updates when dataset changes
- Plain-DOM consumer: element replaced on re-render, old element content cleared

- [ ] **Step 2: Implement content interpolation reactivity**

In `createActivationCallback()`, when processing `html:`, `markdown:`, `title:` components:
1. Check if props contain `#{}` via `hasTemplateVars()`
2. If yes: resolve templates before initial render, register lightweight consumer with `ContextManager`
3. Consumer's `apply()` callback: clear element content, re-invoke `renderHtml`/`renderMarkdown`/`renderTitle` with resolved props

- [ ] **Step 3: Run content interpolation tests**

Run: `yarn workspace @casehub/pages-runtime run test -- --run content-interpolation`
Expected: PASS

- [ ] **Step 4: Write failing parameterised URL tests**

Test cases:
- Dataset with URL `/api/trials/#{filter.trialId}/patients` → no fetch when `trialId` filter empty
- Filter set → URL resolves → fetch triggered with concrete URL
- Filter changes → new URL → old fetch aborted, new fetch dispatched
- Same URL resolved → no re-fetch
- URL with `encodeURIComponent` escaping: filter value `"hello world"` → `"hello%20world"` in URL
- Multiple parameterised datasets: each tracked independently

- [ ] **Step 5: Implement parameterised URLs**

In data pipeline / site.ts:
1. Before resolving external datasets, check each `ExternalDataSetDef.url` for `#{}` via `hasTemplateVars()`
2. If parameterised: register a URL consumer with `ContextManager`
3. Consumer's `apply()` callback:
   - If `!allTemplateVarsResolved()` → suppress fetch (deferred)
   - If URL changed: abort previous `AbortController`, create new one, dispatch fetch with resolved URL
4. Extend `DataRequest` with optional `signal?: AbortSignal`
5. Pipeline catches `AbortError` and ignores it (no error state on component)

Add `AbortController` map alongside `pendingResolutions` in `DataPipeline`.

- [ ] **Step 6: Run parameterised URL tests**

Run: `yarn workspace @casehub/pages-runtime run test -- --run parameterised-urls`
Expected: PASS

- [ ] **Step 7: Integrate ContextManager into site.ts**

Wire `ContextManager` into `loadSite()`:
1. Create `ContextManager` instance
2. Pass `onChanged` callback to `createDataSetManager()` → calls `contextManager.updateDataset()`
3. On `pages-filter` event → calls `contextManager.updateFilter()`
4. On page navigation → calls `contextManager.updatePage()`
5. Pass `contextManager` to activation callback

- [ ] **Step 8: Run full runtime test suite + typecheck**

Run: `yarn workspace @casehub/pages-runtime run test`
Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 9: Commit**

```
git add packages/pages-runtime/src/
git commit -m "feat: add content interpolation and parameterised dataset URLs

markdown/html/title with #{} re-render on context change with
context-aware escaping. Dataset URLs with #{} re-fetch when resolved
URL changes, with deferred fetch for unresolved variables and
AbortController-based request cancellation.

Refs #50, Refs #48, Refs #49"
```

---

## Task 6: YAML Desugar Pipeline Changes

**Files:**
- Modify: `packages/pages-ui/src/parser/displayer-desugar.ts`
- Modify: `packages/pages-ui/src/parser/component-desugar.ts`
- Modify: `packages/pages-component/src/model/displayer-types.ts`
- Modify: `packages/pages-component/src/model/action-types.ts` (create)
- Create: `packages/pages-ui/src/parser/desugar-new-types.test.ts`

**Interfaces:**
- Consumes: `Component` type (with `visibleWhen`), existing desugar functions
- Produces:
  - `TYPE_MAP` additions: `BADGE`, `COUNTDOWN`, `TIMELINE`, `GRAPH`
  - `alert:`, `action-button:` content shorthand branches in `desugarComponent()`
  - `visibleWhen` extraction in both desugar functions
  - Type-specific settings extraction blocks for badge/countdown/timeline/graph
  - Table extraction fix: `sortable`, `resizable` (pre-existing gap), plus `rowStyle`, `expandable`
  - New props types: `BadgeProps`, `CountdownProps`, `TimelineProps`, `GraphProps`, `AlertProps`, `ActionButtonProps`, `RowStyleRule`, `ExpandableConfig`

- [ ] **Step 1: Define all new prop types**

Add to `displayer-types.ts`:
```typescript
export interface BadgeProps extends DataComponentCommon {
  readonly column?: ColumnId;
  readonly colorMap?: Record<string, string>;
}

export interface CountdownProps extends DataComponentCommon {
  readonly deadlineColumn?: ColumnId;
  readonly format?: "full" | "compact" | "days-only";
  readonly warningThreshold?: string;
  readonly criticalThreshold?: string;
}

export interface TimelineProps extends DataComponentCommon, ChartSettings {
  readonly startColumn?: ColumnId;
  readonly endColumn?: ColumnId;
  readonly labelColumn?: ColumnId;
  readonly categoryColumn?: ColumnId;
}

export interface GraphProps extends DataComponentCommon, ChartSettings {
  readonly layout?: "force" | "circular" | "none";
  readonly sourceColumn?: ColumnId;
  readonly targetColumn?: ColumnId;
  readonly valueColumn?: ColumnId;
  readonly directed?: boolean;
  readonly nodeLabelColumn?: ColumnId;
  readonly nodeColorColumn?: ColumnId;
  readonly nodeColorMap?: Record<string, string>;
  readonly nodeSizeColumn?: ColumnId;
}

export interface RowStyleRule {
  readonly condition: string;
  readonly className?: string;
  readonly style?: Record<string, string>;
}

export interface ExpandableConfig {
  readonly idColumn: ColumnId;
  readonly parentColumn: ColumnId;
  readonly defaultExpanded?: boolean | number;
}
```

Extend `TableProps`:
```typescript
export interface TableProps extends DataComponentCommon {
  readonly pageSize?: number;
  readonly sortable?: boolean;
  readonly resizable?: boolean;
  readonly rowStyle?: readonly RowStyleRule[];
  readonly expandable?: ExpandableConfig;
}
```

Create `action-types.ts` in `pages-component/src/model/`:
```typescript
export interface AlertProps {
  readonly severity: "info" | "warning" | "error" | "success";
  readonly content: string;
  readonly dismissible?: boolean;
}

export interface ActionButtonProps {
  readonly label: string;
  readonly url: string;
  readonly method?: "POST" | "PUT" | "DELETE";
  readonly body?: Record<string, unknown>;
  readonly headers?: Record<string, string>;
  readonly confirm?: string;
  readonly style?: "primary" | "danger" | "secondary";
  readonly disabledWhen?: string;
  readonly onSuccess?: { readonly refresh?: string[]; readonly message?: string };
  readonly onError?: { readonly message?: string };
}

export interface SubmitConfig {
  readonly url: string;
  readonly method?: "POST" | "PUT";
  readonly fieldName?: string;
  readonly clearOnSubmit?: boolean;
  readonly onSuccess?: { readonly refresh?: string[]; readonly message?: string };
  readonly onError?: { readonly message?: string };
}
```

- [ ] **Step 2: Write failing desugar tests**

Test cases:
- `{ displayer: { type: "BADGE", badge: { column: "status", colorMap: { OK: "green" } } } }` → `Component` with type `"badge"` and flattened props
- `{ displayer: { type: "COUNTDOWN", countdown: { deadlineColumn: "deadline" } } }` → correct props
- `{ displayer: { type: "TIMELINE", timeline: { startColumn: "start" } } }` → correct props
- `{ displayer: { type: "GRAPH", graph: { layout: "force", sourceColumn: "from" } } }` → correct props
- `{ alert: { severity: "warning", content: "test" } }` → `Component` with type `"alert"`
- `{ "action-button": { label: "Click", url: "/api/test" } }` → `Component` with type `"action-button"`
- `visibleWhen` extracted from both displayer and component shorthands
- Table: `{ displayer: { table: { sortable: true, resizable: true, rowStyle: [...] } } }` → all props extracted
- Table: `{ displayer: { table: { expandable: { idColumn: "id", parentColumn: "pid" } } } }` → expandable extracted

- [ ] **Step 3: Implement desugar changes**

In `displayer-desugar.ts`:
1. Add to `TYPE_MAP`: `BADGE: "badge"`, `COUNTDOWN: "countdown"`, `TIMELINE: "timeline"`, `GRAPH: "graph"`
2. Add extraction blocks for each type's settings (same pattern as existing `table:` and `meter:` blocks)
3. Fix table extraction: add `sortable`, `resizable` alongside existing `pageSize`; add new `rowStyle`, `expandable`
4. Extract `visibleWhen` from `raw.visibleWhen` and place on output `Component`

In `component-desugar.ts`:
1. Add `alert:` and `action-button:` content shorthand branches (before the `displayer:` check)
2. Extract `visibleWhen` from `raw.visibleWhen`

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehub/pages-ui run test -- --run parser/desugar-new-types`
Expected: PASS

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add packages/pages-component/src/model/ packages/pages-ui/src/parser/
git commit -m "feat: add prop types and desugar mappings for all new components

New types: BadgeProps, CountdownProps, TimelineProps, GraphProps,
AlertProps, ActionButtonProps, SubmitConfig, RowStyleRule, ExpandableConfig.
Desugar: TYPE_MAP entries, settings extraction blocks, alert/action-button
content shorthands, visibleWhen extraction, table sortable/resizable fix.

Refs #50, Refs #37, Refs #38, Refs #39, Refs #40, Refs #41, Refs #42, Refs #43, Refs #46"
```

---

## Task 7: PagesContentElement Base Class

**Files:**
- Create: `packages/pages-viz/src/base/PagesContentElement.ts`
- Create: `packages/pages-viz/src/base/PagesContentElement.test.ts`

**Interfaces:**
- Consumes: Nothing from other tasks (standalone base class)
- Produces:
  - `abstract class PagesContentElement<P extends object>` — used by Tasks 8, 9, 10

```typescript
export abstract class PagesContentElement<P extends object> extends HTMLElement {
  declare readonly shadowRoot: ShadowRoot;
  protected readonly container: HTMLDivElement;
  private _props: P | undefined;

  constructor();
  get props(): P | undefined;
  set props(value: P | undefined);
  connectedCallback(): void;
  disconnectedCallback(): void;
  protected abstract render(container: HTMLDivElement, props: P): void;
}
```

- [ ] **Step 1: Write failing tests**

Test cases:
- Subclass renders when props are set and element is connected
- No render when disconnected
- No render when props undefined
- Props setter triggers re-render
- Shadow DOM created with container div
- `disconnectedCallback` fires (for context deregistration hook)

- [ ] **Step 2: Implement PagesContentElement**

Follow the spec §1.10 exactly. Shadow DOM, props management, connectedCallback/disconnectedCallback lifecycle. No refresh timer, no data request, no resize observer.

- [ ] **Step 3: Run tests and commit**

Run: `yarn workspace @casehub/pages-viz run test -- --run base/PagesContentElement`
Expected: PASS

```
git add packages/pages-viz/src/base/PagesContentElement.ts packages/pages-viz/src/base/PagesContentElement.test.ts
git commit -m "feat: add PagesContentElement base class for content Web Components

Lightweight base for components without dataset binding (alert,
action-button). Shadow DOM + props + lifecycle, no data machinery.

Refs #50, Refs #38, Refs #46"
```

---

## Task 8: ActionExecutor + Action Request Event

**Files:**
- Create: `packages/pages-runtime/src/action.ts`
- Create: `packages/pages-runtime/src/action.test.ts`
- Modify: `packages/pages-runtime/src/site.ts`

**Interfaces:**
- Consumes: `RuntimeContext`, `resolveTemplate` (Task 1), `ActionRequest`, `ActionCallbacks` (Task 6)
- Produces:
  - `ActionExecutor` class:
    - `constructor(fetchFn: typeof fetch, baseUrl: string)`
    - `execute(request: ActionRequest, callbacks: ActionCallbacks, context: RuntimeContext): Promise<ActionResult>`
  - `ActionResult` type: `{ success: boolean; status?: number; error?: string }`
  - `PagesActionRequestDetail` type: `{ config: ActionRequest & { callbacks: ActionCallbacks }; resolve: (result: ActionResult) => void }`
  - `PagesActionCompleteDetail` type: `{ refresh: string[] }`
  - Runtime listens for `pages-action-request`, executes via `ActionExecutor`, dispatches `pages-action-complete`

- [ ] **Step 1: Write failing ActionExecutor tests**

Test cases:
- Successful POST: resolves `#{}` in URL and body, sends request, returns `{ success: true }`
- Failed POST (4xx): returns `{ success: false, status: 400, error: "Bad Request" }`
- Failed POST (5xx): returns `{ success: false, status: 500, error: "..." }`
- Template resolution: URL `"/api/#{filter.id}"` resolved before fetch
- Body template resolution: `{ name: "#{filter.name}" }` → resolved values
- Uses injected `fetch`, not `globalThis.fetch`
- Base URL prepended to relative URLs

- [ ] **Step 2: Implement ActionExecutor**

Core logic:
1. `execute()` resolves all `#{}` templates in URL, body (recursively for objects), headers
2. Prepends `baseUrl` to relative URLs
3. Sends request via injected `fetchFn`
4. Returns `ActionResult` based on response status

- [ ] **Step 3: Wire into site.ts**

Add listener for `pages-action-request`:
1. Runtime catches event, extracts config and resolve callback
2. Calls `actionExecutor.execute(config, callbacks, contextManager.getContext())`
3. Calls `resolve(result)`
4. On success with `refresh`: dispatches `pages-action-complete` event
5. `pages-action-complete` handler: re-fetches listed datasets via data pipeline

- [ ] **Step 4: Run tests and commit**

Run: `yarn workspace @casehub/pages-runtime run test -- --run action`
Expected: PASS

```
git add packages/pages-runtime/src/action.ts packages/pages-runtime/src/action.test.ts packages/pages-runtime/src/site.ts
git commit -m "feat: add ActionExecutor and pages-action-request event bridge

Shared HTTP action infrastructure with host fetch injection, template
resolution, and dataset refresh on success. Runtime listens for
pages-action-request and dispatches pages-action-complete.

Refs #50, Refs #46, Refs #54"
```

---

## Task 9: Action Button Component

**Files:**
- Create: `packages/pages-viz/src/components/PagesActionButton.ts`
- Create: `packages/pages-viz/src/components/PagesActionButton.test.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`
- Modify: `packages/pages-runtime/src/activation.ts`

**Interfaces:**
- Consumes: `PagesContentElement` (Task 7), `ActionButtonProps` (Task 6), `PagesActionRequestDetail` (Task 8)
- Produces: `<pages-action-button>` custom element

- [ ] **Step 1: Write failing tests**

Test cases:
- Renders `<button>` with label text
- Click dispatches `pages-action-request` event with config from props
- Confirm dialog shown when `confirm` prop set, action only proceeds on confirmation
- Loading state: button disabled + spinner during request
- Success: button re-enabled, success message shown briefly
- Error: button re-enabled, error message shown below button
- `disabledWhen` expression evaluated against context → `aria-disabled="true"` when truthy
- `aria-busy="true"` during HTTP request
- Style prop applies CSS class (`pages-btn-primary`, `pages-btn-danger`, `pages-btn-secondary`)

- [ ] **Step 2: Implement PagesActionButton**

Extends `PagesContentElement<ActionButtonProps>`. `render()`:
1. Creates `<button>` with label, style class
2. Click handler: shows native `confirm()` dialog if configured, then dispatches `pages-action-request`
3. Receives result via `resolve` callback → transitions to success/error
4. Error message rendered as `<div class="pages-action-error">` below button

- [ ] **Step 3: Register in custom-elements.ts and activation.ts**

Add to `custom-elements.ts` type map and define element. Add content Web Component activation path in `activation.ts` for type `"action-button"`.

- [ ] **Step 4: Run tests and commit**

```
git add packages/pages-viz/src/components/PagesActionButton.ts packages/pages-viz/src/components/PagesActionButton.test.ts packages/pages-viz/src/custom-elements.ts packages/pages-runtime/src/activation.ts
git commit -m "feat: add action button component — REST POST with confirmation and feedback

<pages-action-button> dispatches pages-action-request, shows loading
state, handles success/error. Supports confirmation dialog, disabledWhen
expression, dataset refresh on success.

Refs #50, Refs #46"
```

---

## Task 10: Form Submit

**Files:**
- Modify: `packages/pages-viz/src/form-inputs/PagesFormInput.ts`
- Create: `packages/pages-viz/src/form-inputs/form-submit.test.ts`

**Interfaces:**
- Consumes: `SubmitConfig` (Task 6), `PagesActionRequestDetail` (Task 8)
- Produces: `submit` prop support on all form inputs

- [ ] **Step 1: Write failing tests**

Test cases:
- Text input with `submit` config: Enter key dispatches `pages-action-request`
- Body constructed as `{ [fieldName ?? field]: inputValue }`
- `clearOnSubmit: true`: field cleared after successful submit
- Error: inline error shown, field value preserved
- `submit` mode operates independently of `dataScope` — no `pages-field-change` dispatched

- [ ] **Step 2: Implement submit in PagesFormInput**

In `PagesFormInput` base class:
1. Check `props.submit` in `connectedCallback`
2. If present, add `keydown` listener for Enter
3. On Enter: construct `ActionRequest` from `SubmitConfig`, dispatch `pages-action-request`
4. Handle result: clear on success (if configured), show error on failure

- [ ] **Step 3: Run tests and commit**

```
git add packages/pages-viz/src/form-inputs/
git commit -m "feat: add form submit — fire-and-forget POST on Enter

Form inputs with submit config POST to URL on Enter, clear on success.
Operates independently of dataScope binding.

Refs #50, Refs #54"
```

---

## Task 11: Alert Banner Component

**Files:**
- Create: `packages/pages-viz/src/components/PagesAlert.ts`
- Create: `packages/pages-viz/src/components/PagesAlert.test.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`
- Modify: `packages/pages-runtime/src/activation.ts`

**Interfaces:**
- Consumes: `PagesContentElement` (Task 7), `AlertProps` (Task 6)
- Produces: `<pages-alert>` custom element

- [ ] **Step 1: Write failing tests**

Test cases:
- Renders banner with severity-based colors (CSS custom properties)
- `role="alert"` for warning/error, `role="status"` for info/success
- Content rendered as text (no HTML interpretation — content comes pre-escaped from template resolver)
- Dismissible: close button present, clicking hides alert
- Dismiss state keyed on resolved content: same content stays dismissed, different content reappears
- Dismiss button has `aria-label="Dismiss alert"`

- [ ] **Step 2: Implement PagesAlert**

Extends `PagesContentElement<AlertProps>`. Shadow DOM styles use `--pages-alert-*` CSS custom properties. `render()`:
1. Create banner div with severity class
2. Set ARIA role based on severity
3. Render content text
4. If dismissible: add close button; track dismissed content in `_dismissedContent: string | null`
5. If `props.content === _dismissedContent` → stay hidden

- [ ] **Step 3: Register and commit**

```
git add packages/pages-viz/src/components/PagesAlert.ts packages/pages-viz/src/components/PagesAlert.test.ts packages/pages-viz/src/custom-elements.ts packages/pages-runtime/src/activation.ts
git commit -m "feat: add alert banner component with dismiss and ARIA support

<pages-alert> renders severity-styled banner. Dismiss state keyed on
resolved content — reappears only when message changes.

Refs #50, Refs #38"
```

---

## Task 12: Badge Component

**Files:**
- Create: `packages/pages-viz/src/components/PagesBadge.ts`
- Create: `packages/pages-viz/src/components/PagesBadge.test.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`

**Interfaces:**
- Consumes: `PagesElement` (existing base), `BadgeProps` (Task 6)
- Produces: `<pages-badge>` custom element

- [ ] **Step 1: Write failing tests**

Test cases:
- Single-row dataset → one badge span with value text
- Multi-row dataset → multiple badge spans
- `colorMap` applied: value `"PENDING"` → background from map
- Missing colorMap entry → fallback palette color
- Badge column defaults to first LABEL column when `column` prop absent

- [ ] **Step 2: Implement and commit**

Extends `PagesElement<BadgeProps>`. `render()` creates `<span class="pages-badge">` per row. CSS custom properties for theming.

```
git add packages/pages-viz/src/components/PagesBadge.ts packages/pages-viz/src/components/PagesBadge.test.ts packages/pages-viz/src/custom-elements.ts
git commit -m "feat: add status badge component with color mapping

<pages-badge> renders styled label tags from dataset column values
with configurable color map.

Refs #50, Refs #39"
```

---

## Task 13: Countdown Component

**Files:**
- Create: `packages/pages-viz/src/components/PagesCountdown.ts`
- Create: `packages/pages-viz/src/components/PagesCountdown.test.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`

**Interfaces:**
- Consumes: `PagesElement` (existing base), `CountdownProps` (Task 6), `parseRefreshTime` from `@casehub/pages-data`
- Produces: `<pages-countdown>` custom element

- [ ] **Step 1: Write failing tests**

Test cases:
- Displays time remaining from deadline in dataset
- Timer updates display (mock `setInterval`)
- Warning threshold: color changes when remaining time < threshold
- Critical threshold: color changes at critical level
- Expired: shows "EXPIRED" with critical styling
- `disconnectedCallback` clears timer
- `aria-live="polite"` on display element
- Screen reader announcements only on threshold crossings

- [ ] **Step 2: Implement and commit**

Extends `PagesElement<CountdownProps>`. Internal `setInterval` at 1s rate. Formats display based on remaining time. Thresholds parsed via `parseRefreshTime()`.

```
git add packages/pages-viz/src/components/PagesCountdown.ts packages/pages-viz/src/components/PagesCountdown.test.ts packages/pages-viz/src/custom-elements.ts
git commit -m "feat: add countdown component with threshold-based color transitions

<pages-countdown> displays live time remaining until a deadline date
from the dataset. Warning/critical thresholds trigger color changes.
ARIA announcements on threshold crossings only.

Refs #50, Refs #43"
```

---

## Task 14: Timeline Component

**Files:**
- Create: `packages/pages-viz/src/charts/PagesTimeline.ts`
- Create: `packages/pages-viz/src/charts/PagesTimeline.test.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`

**Interfaces:**
- Consumes: `PagesChartElement` (existing base), `TimelineProps` (Task 6)
- Produces: `<pages-timeline>` custom element

- [ ] **Step 1: Write failing tests**

Test cases:
- `buildOption()` produces ECharts option with `type: 'custom'` series
- Time x-axis from start/end columns
- Category y-axis from category column
- Rows with null endColumn → diamond milestone markers (not bars)
- Standard ChartSettings (zoom, legend, margin) apply

- [ ] **Step 2: Implement and commit**

Extends `PagesChartElement<TimelineProps>`. `buildOption()` uses ECharts custom series with `renderItem` callback that draws `rect` for duration bars and `diamond` for milestones.

ECharts imports: `use([CanvasRenderer, CustomChart, GridComponent, TooltipComponent, LegendComponent, DataZoomComponent])`.

```
git add packages/pages-viz/src/charts/PagesTimeline.ts packages/pages-viz/src/charts/PagesTimeline.test.ts packages/pages-viz/src/custom-elements.ts
git commit -m "feat: add timeline component — ECharts custom series for duration bars

<pages-timeline> renders horizontal bars on a time axis with category
grouping. Null end dates render as diamond milestone markers.

Refs #50, Refs #37"
```

---

## Task 15: Graph Component

**Files:**
- Create: `packages/pages-viz/src/charts/PagesGraph.ts`
- Create: `packages/pages-viz/src/charts/PagesGraph.test.ts`
- Modify: `packages/pages-viz/src/custom-elements.ts`

**Interfaces:**
- Consumes: `PagesChartElement` (existing base), `GraphProps` (Task 6)
- Produces: `<pages-graph>` custom element

- [ ] **Step 1: Write failing tests**

Test cases:
- `buildOption()` produces ECharts option with `type: 'graph'` series
- Nodes derived from distinct source/target values
- Links built from edge rows
- Layout prop maps to ECharts layout type
- `directed: true` → arrow markers on links
- `nodeLabelColumn` provides display names
- `nodeColorColumn` + `nodeColorMap` → per-node coloring
- `nodeSizeColumn` → proportional node sizing

- [ ] **Step 2: Implement and commit**

Extends `PagesChartElement<GraphProps>`. `buildOption()` builds nodes and links from the edge dataset.

ECharts imports: `use([CanvasRenderer, GraphChart, TooltipComponent, LegendComponent])`.

```
git add packages/pages-viz/src/charts/PagesGraph.ts packages/pages-viz/src/charts/PagesGraph.test.ts packages/pages-viz/src/custom-elements.ts
git commit -m "feat: add graph component — ECharts network visualization

<pages-graph> renders force-directed or circular graph from edge
dataset. Supports node labels, colors, sizes from dataset columns.

Refs #50, Refs #41"
```

---

## Task 16: Row-Level Conditional Styling

**Files:**
- Modify: `packages/pages-viz/src/components/PagesTable.ts`
- Create: `packages/pages-viz/src/components/table-row-style.test.ts`

**Interfaces:**
- Consumes: `evaluateExpression`, `createRowContext` (Task 1), `RowStyleRule` (Task 6), `ContextManager` (Task 3)
- Produces: Row styling in `PagesTable.render()`

- [ ] **Step 1: Write failing tests**

Test cases:
- Row matching `"#{row.status} == 'Critical'"` → `pages-row-danger` class on `<tr>`
- First matching rule wins — subsequent rules not evaluated for that row
- `style:` inline CSS applied when matched
- No match → no class applied
- Row context includes global context: `"#{row.status} == 'Critical' && #{filter.showHighlights}"` works
- Predefined classes render with correct CSS custom properties

- [ ] **Step 2: Implement row styling**

In `PagesTable.render()`:
1. If `props.rowStyle` is defined, evaluate each rule's condition for each row
2. Create row context via `createRowContext(globalContext, rowCells)`
3. First matching rule's `className` or `style` is applied to the `<tr>`
4. Add predefined CSS classes to shadow DOM stylesheet

- [ ] **Step 3: Run tests and commit**

```
git add packages/pages-viz/src/components/PagesTable.ts packages/pages-viz/src/components/table-row-style.test.ts
git commit -m "feat: add row-level conditional styling to tables

Evaluate #{row.*} expressions per row, apply CSS class or inline
style on first match. Predefined classes: pages-row-danger/warning/
success/muted with CSS custom property overrides.

Refs #50, Refs #40"
```

---

## Task 17: Expandable Rows (Tree-Table)

**Files:**
- Modify: `packages/pages-viz/src/components/PagesTable.ts`
- Modify: `packages/pages-runtime/src/data-pipeline.ts`
- Create: `packages/pages-viz/src/components/table-expandable.test.ts`

**Interfaces:**
- Consumes: `ExpandableConfig` (Task 6), `TypedDataSet` from `@casehub/pages-data`
- Produces: Tree-table rendering in `PagesTable`, pipeline bypass for expandable tables

- [ ] **Step 1: Write failing tests**

Test cases:
- Flat dataset with id/parentId columns → root rows shown, children hidden
- Expand toggle click → children appear with indentation
- `defaultExpanded: 1` → first level expanded initially
- `defaultExpanded: true` → all levels expanded
- Collapse → children hidden recursively
- Sorting within level: siblings sorted among siblings
- Pagination by root row count: expanding doesn't push other roots to next page
- Filter: child matches but parent doesn't → parent shown dimmed as context row
- `aria-expanded`, `aria-level`, `aria-setsize`, `aria-posinset` present

- [ ] **Step 2: Implement pipeline bypass**

In `data-pipeline.ts`: when the component's props contain `expandable`, skip `rowOffset`/`rowCount` and `textFilter` — deliver all rows. Add a guard check in `handleDataRequest()`.

- [ ] **Step 3: Implement tree-table rendering**

In `PagesTable.render()`:
1. If `props.expandable`, build tree index from dataset
2. Track expand state per row ID
3. Render only visible rows (roots + expanded children)
4. Add toggle button column
5. Apply indentation via padding-left
6. Internal pagination: count root rows, slice by `pageSize`
7. Set ARIA tree attributes

- [ ] **Step 4: Run tests and commit**

```
git add packages/pages-viz/src/components/PagesTable.ts packages/pages-viz/src/components/table-expandable.test.ts packages/pages-runtime/src/data-pipeline.ts
git commit -m "feat: add expandable rows (tree-table) with pipeline bypass

Self-referencing dataset rendered as tree with expand/collapse.
Pipeline delivers all rows when expandable is configured — component
owns pagination and text search. Sorting per-level, filtering
preserves hierarchy with dimmed context rows.

Refs #50, Refs #42"
```

---

## Task 18: Activation Path for New Components

**Files:**
- Modify: `packages/pages-runtime/src/activation.ts`
- Create: `packages/pages-runtime/src/activation-new-components.test.ts`

**Interfaces:**
- Consumes: All new component types, `ContextManager` (Task 3)
- Produces: Complete activation path for all new component types

- [ ] **Step 1: Write failing tests**

Test cases:
- `type: "badge"` in `DATA_COMPONENT_TYPES` → creates `<pages-badge>`, sets props, registers for data request
- `type: "countdown"` → same pattern
- `type: "timeline"` → same pattern (ECharts chart element)
- `type: "graph"` → same pattern
- `type: "alert"` → creates `<pages-alert>` via content Web Component path, sets props, NO data request
- `type: "action-button"` → creates `<pages-action-button>` via content Web Component path
- Content Web Components registered with `ContextManager` when props contain `#{}` templates

- [ ] **Step 2: Implement activation for new types**

Add to `DATA_COMPONENT_TYPES`: `"badge"`, `"countdown"`, `"timeline"`, `"graph"`.

Add new content Web Component activation branch (after data component branch, before plain-DOM content):
```typescript
const CONTENT_WEB_COMPONENT_TYPES = new Set(["alert", "action-button"]);
if (CONTENT_WEB_COMPONENT_TYPES.has(component.type)) {
  const tagName = `pages-${component.type}`;
  const element = document.createElement(tagName);
  element.props = component.props;
  el.appendChild(element);
  // Register with ContextManager if props contain #{}
}
```

- [ ] **Step 3: Run tests and commit**

```
git add packages/pages-runtime/src/activation.ts packages/pages-runtime/src/activation-new-components.test.ts
git commit -m "feat: wire activation paths for all new component types

Data components (badge, countdown, timeline, graph) added to
DATA_COMPONENT_TYPES. Content Web Components (alert, action-button)
activated via new path without data request.

Refs #50"
```

---

## Task 19: Example Dashboard Update + Playwright Tests

**Files:**
- Modify: `examples/dashboards/Clinical/Patient Tracker.dash.yaml`
- Modify: `examples/dashboards/Clinical/Patient Tracker.ts`
- Modify: `examples/tests/domain-dashboards.spec.ts`

**Interfaces:**
- Consumes: All new components and features
- Produces: Updated Clinical Patient Tracker dashboard exercising new capabilities, Playwright tests

- [ ] **Step 1: Update Clinical Patient Tracker YAML**

Add to existing dashboard:
- `visibleWhen` on Patient Detail table (show only when patient selected via filter)
- Content interpolation in markdown panel: `"#{filter.ward} Ward — #{datasets.patients.rowCount} patients"`
- `alert:` banner with `visibleWhen` on row count
- `badge:` component for patient status
- Row styling on vitals table: critical vitals highlighted
- Action button (mock endpoint — inline handler not needed, just validate rendering)

- [ ] **Step 2: Write Playwright tests**

Test cases:
- New components render without errors
- Badge shows correct status values
- Alert banner appears/disappears based on data
- `visibleWhen` hides/shows components on filter interaction
- Row styling applies correct CSS classes
- Action button renders with correct label

- [ ] **Step 3: Run Playwright tests**

Run: `yarn workspace @casehub/pages-examples run test`
Expected: PASS

- [ ] **Step 4: Commit**

```
git add examples/
git commit -m "feat: update Clinical Patient Tracker with new component capabilities

Exercises visibleWhen, content interpolation, alert banner, badge,
row styling, and action button. Playwright tests verify rendering.

Refs #50"
```

---

## Dependency Graph

```
Task 1 (context types + parser + evaluator)
  ├── Task 2 (manager onChanged)
  │     └── Task 3 (context wiring)
  │           ├── Task 4 (visibleWhen)
  │           ├── Task 5 (content interpolation + parameterised URLs)
  │           ├── Task 16 (row styling)
  │           └── Task 8 (ActionExecutor)
  │                 ├── Task 9 (action button)
  │                 └── Task 10 (form submit)
  ├── Task 6 (YAML desugar) — independent, can run in parallel with Tasks 2-5
  └── Task 7 (PagesContentElement) — independent, can run in parallel

Task 7 (PagesContentElement)
  ├── Task 9 (action button)
  └── Task 11 (alert)

Tasks 12-15 (badge, countdown, timeline, graph) — depend on Task 6 for types
Task 17 (expandable rows) — depends on Task 6 for types
Task 18 (activation) — depends on all component tasks
Task 19 (example dashboard) — depends on all tasks
```

**Parallelism opportunities:**
- Tasks 1, 6, 7 can run in parallel (no dependencies between them)
- After Task 3: Tasks 4, 5, 8, 16 can run in parallel
- After Task 7: Tasks 9, 11 can run in parallel
- Tasks 12-15 can all run in parallel once Task 6 is complete
