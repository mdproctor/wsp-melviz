# Phase 2 — Runtime Parity Design

**Issue:** casehubio/casehub-pages#621
**Parent:** casehubio/casehub-pages#608 (TS/Java YAML runtime parity audit)
**Date:** 2026-11-08
**Scope:** yaml-core runtime behaviour for 5 Phase 2 gaps: H3, M4a, H1, M6, H4

---

## Overview

Phase 1 (#620) achieved schema parity — the TS Walker accepts the same YAML vocabulary as Java. Phase 2 adds runtime behaviour behind the unified schema: typed error handling, control step evaluation, annotation processing, test simulation parsing, and test infrastructure.

**Implementation order:** H3 → M4a → H1 → M6 → H4 (D1). Quick wins first, annotations mid-sequence, test infrastructure last as capstone.

**Package:** All work lands in `packages/yaml-core/src/`.

---

## 1. Typed Error Catch Scope (H3)

**Gap:** `StructuralEvaluator.evaluateTryCatchFinally()` runs catch steps with the same context as try steps. Java injects `${error.type}`, `${error.message}`, `${error.step}`, `${error.detail}` via `YamlErrorMapper`.

**Approach (D2):** Derive error scope from `Result.failure` via `ObjectVariableSource`. No new error hierarchy — the TS runtime already propagates failures as `Result` values with `message`, `failureCategory`, and `detail`.

### 1.1 ErrorCategory Constants

New file: `step/error-category.ts`

```typescript
export const ERROR_CATEGORIES = [
  'UNKNOWN_STEP', 'PARAMETER_VIOLATION', 'INVALID_STRUCTURE',
  'EXPRESSION_PARSE', 'DUPLICATE_STEP', 'STEP_ACTION_FAILED',
  'TIMEOUT', 'RETRY_EXHAUSTED', 'GUARD_REJECTED',
  'EXPRESSION_EVAL', 'CHANNEL_CLOSED', 'DEADLINE_EXCEEDED',
  'CORRELATION_TIMEOUT', 'SEMAPHORE_REENTRANCY',
  'ILLEGAL_TRANSITION', 'RATE_LIMIT', 'SESSION_LIMIT',
  'NOT_FOUND', 'AUTH_DENIED', 'SIMULATION', 'INFRASTRUCTURE',
] as const;

export type ErrorCategory = typeof ERROR_CATEGORIES[number];
```

### 1.2 evaluateTryCatchFinally Changes

Modified file: `step/structural-evaluator.ts`

Current code runs catch steps with the original context:
```typescript
if (tryResult.kind === 'failure' && step.catchSteps.length > 0) {
  for (const child of step.catchSteps) {
    catchResult = await this.evaluate(child, context);  // same context
  }
}
```

After change:
1. Track the failing step name during try evaluation
2. When `tryResult.kind === 'failure'`, build an `ObjectVariableSource`:
   - `type` → `tryResult.failureCategory ?? 'UNKNOWN'`
   - `message` → `tryResult.message`
   - `step` → name of the step that failed (tracked during try loop)
   - Deep drill into `tryResult.detail` for `detail.*` paths
3. Create scoped resolver: `context.resolver.withObjectScope('error', errorSource)`
4. Build catch context: `{ ...context, resolver: scopedResolver }`

The try evaluation loop is modified to track `lastFailedStepName` alongside `tryResult`.

### 1.3 Test Strategy

- Unit test: try-catch with a step that returns `stepFailureWithCategory('msg', 'TIMEOUT')`, verify `${error.type}` resolves to `TIMEOUT` in catch steps
- Unit test: verify `${error.detail.someField}` deep access
- Unit test: verify `${error.step}` carries the failing step's name
- Edge case: nested try-catch — inner catch error scope doesn't leak to outer

---

## 2. ControlStep Runtime (M4a)

**Gap:** `StructuralEvaluator.dispatch()` has no `case 'control'` — ControlStep (parsed since Phase 1) silently falls through with undefined behaviour.

**Approach:** Wire to ScenarioScope's lifecycle engine.

### 2.1 StructuralEvaluator Changes

Modified file: `step/structural-evaluator.ts`

Add to dispatch switch:
```typescript
case 'control': return this.evaluateControl(step, context);
```

Add import of `ControlStep` type. Implement `evaluateControl()`:
- For `pause`: `context.scope.lifecycleEngine().pause(step.target)`
- For `resume`: `context.scope.lifecycleEngine().resume(step.target)`
- If lifecycle engine doesn't know the target: `stepFailureWithCategory(msg, 'NOT_FOUND')`
- On success: `stepSuccess({ control: step.control, target: step.target })`

### 2.2 Test Strategy

- Unit test: pause a named consumer, verify lifecycle engine state
- Unit test: resume a paused consumer, verify reactivation
- Unit test: pause unknown target, verify NOT_FOUND failure

---

## 3. Annotation System (H1)

**Gap:** `annotations: unknown[]` exists on all step types but is always empty. No parser, extractor, map, or injection resolver.

**Approach (D3, D4):** Pre-parse raw YAML text for `#@Name(args)` patterns, build positional map, merge annotation-based decorators into the existing decorator map before `DecoratorChain.build()`.

### 3.1 AnnotationParser

New file: `annotation/annotation-parser.ts`

**Grammar:** Line starts with optional whitespace, then `#@`, then identifier, then optional `(args)`.

Args parsing:
- Comma-separated
- Named: `key: "value"` or `key: 'value'`
- Positional: `"value"` or `'value'` or bare value

```typescript
export interface ParsedAnnotation {
  name: string;
  args: Record<string, string>;
  line: number;
}

export class AnnotationParser {
  static parse(rawYaml: string): ParsedAnnotation[];
}
```

### 3.2 AnnotationExtractor

New file: `annotation/annotation-extractor.ts`

Takes `ParsedAnnotation[]` and the parsed YAML steps array. Correlates annotations to steps using line-number-based positioning.

**Addressing scheme** (matching Java):
- Annotations before the first step → document-level
- Annotations between step N-1 and step N → attached to step N
- Step addresses are hierarchical: `0`, `1`, `1.0` (first child of step 1), `1.0.2` (third child of first child of step 1)

```typescript
export class AnnotationExtractor {
  static extract(
    annotations: ParsedAnnotation[],
    stepLineNumbers: StepLineInfo[],
  ): AnnotationMap;
}
```

`StepLineInfo` carries each step's line number and address in the hierarchy. The Walker needs to produce this during resolution (see §3.6).

### 3.3 AnnotationMap

New file: `annotation/annotation-map.ts`

```typescript
export interface Annotation {
  name: string;
  args: Record<string, string>;
}

export class AnnotationMap {
  forStep(address: string): Annotation[];
  forDocument(name: string): Annotation | undefined;
  hasStepAnnotation(address: string, name: string): boolean;
  allDocumentAnnotations(): Annotation[];

  static empty(): AnnotationMap;
}
```

Immutable. Constructed by `AnnotationExtractor`.

### 3.4 InjectionResolver

New file: `annotation/injection-resolver.ts`

Processes `@Inject(TypeName, qualifier: "value")` annotations. Looks up the named type in the `ServiceRegistry` and injects into the step's params.

```typescript
export class InjectionResolver {
  static resolve(
    annotations: Annotation[],
    params: Record<string, unknown>,
    services: ServiceRegistry,
  ): Record<string, unknown>;
}
```

Returns augmented params with injected values. Injection failures produce clear error messages naming the missing type.

### 3.5 DecoratorAnnotationResolver

New file: `annotation/decorator-annotation-resolver.ts`

Maps the 12 migrated decorator annotations to decorator map entries:

| Annotation | Decorator Map Entry |
|------------|-------------------|
| `#@Timeout("5s")` | `{ timeout: "5s" }` |
| `#@Retry(max: "3")` | `{ retry: { max: 3 } }` |
| `#@Retry(max: "3", backoff: "exponential")` | `{ retry: { max: 3, backoff: "exponential" } }` |
| `#@Delay("1s")` | `{ delay: "1s" }` |
| `#@Priority("HIGH")` | `{ priority: "HIGH" }` |
| `#@Resource("pool-name")` | `{ resource: "pool-name" }` |
| `#@Background` | `{ background: true }` |
| `#@OnError("fallback-step")` | `{ on-error: "fallback-step" }` |
| `#@Trigger("signal-name")` | `{ trigger: "signal-name" }` |
| `#@Semaphore("sem-name")` | `{ semaphore: "sem-name" }` |
| `#@At("metric >= 5")` | `{ at: "metric >= 5" }` |
| `#@Cancel("signal-name")` | `{ cancel: "signal-name" }` |
| `#@Mutex("lock-name")` | `{ mutex: "lock-name" }` |

**Precedence:** Inline decorator values override annotation values.

```typescript
export class DecoratorAnnotationResolver {
  static mergeDecorators(
    annotations: Annotation[],
    decorators: Record<string, unknown>,
  ): Record<string, unknown>;
}
```

### 3.6 Walker Integration

Modified file: `step/walker.ts`

`Walker.resolve()` gains an optional `rawYaml?: string` parameter:

```typescript
static resolve(
  steps: Record<string, unknown>[],
  catalog: Catalog,
  rawYaml?: string,
): ResolvedStep[];
```

When `rawYaml` is provided:
1. `AnnotationParser.parse(rawYaml)` → `ParsedAnnotation[]`
2. Pass annotation map through `resolveAtDepth()` via an internal context object
3. In `resolveOne()`, look up annotations for the current step address
4. `DecoratorAnnotationResolver.mergeDecorators(stepAnnotations, decorators)` → merged decorators
5. Populate `annotations` field on the resolved step with the `Annotation[]` data

The hierarchical address is tracked through the recursion — each `resolveAtDepth()` call passes a base address, and each step gets `baseAddress.index`.

Threading through match cases and select branches: `resolveMatchCases` and similar helpers pass the parent step's address so child steps get correctly addressed (e.g., `1.0` for the first step inside the first case of match at position 1).

### 3.7 New Directory Structure

```
packages/yaml-core/src/annotation/
├── index.ts
├── annotation-parser.ts
├── annotation-parser.test.ts
├── annotation-extractor.ts
├── annotation-extractor.test.ts
├── annotation-map.ts
├── annotation-map.test.ts
├── injection-resolver.ts
├── injection-resolver.test.ts
├── decorator-annotation-resolver.ts
└── decorator-annotation-resolver.test.ts
```

### 3.8 Test Strategy

- AnnotationParser: parse single/multiple annotations, named/positional args, edge cases (no args, nested parens in strings, multiline)
- AnnotationExtractor: document-level vs step-level, hierarchical addressing, annotations on nested steps (inside blocks, if-else, match)
- AnnotationMap: lookup, empty map, has-annotation queries
- InjectionResolver: successful injection, missing type error, qualifier matching
- DecoratorAnnotationResolver: each of 12 decorators, precedence (inline wins), unknown annotation passthrough
- Integration: Walker.resolve with rawYaml produces steps with populated annotations fields and merged decorators

---

## 4. SimulateBlockParser (M6)

**Gap:** `SimulationRuntime` and `SimulationOverlay` exist but no parser for inline `simulate:` blocks in test YAML.

### 4.1 Parser

New file: `step/simulate-block-parser.ts`

```typescript
export class SimulateBlockParser {
  static parse(raw: Record<string, unknown>): SimulationConfig;
}
```

**Alias expansion table:**

| Alias | Expanded Qualified Name | Field Mapping |
|-------|------------------------|---------------|
| `agent` | `agent-provider.invoke` | `prompt` → `key`, `response` → `output` |
| `rest` | `rest-request` | — |
| `graphql` | `graphql` | — |
| `mcp` | `mcp-call` | — |
| `process` | `process-exec` | — |

**Input format** (test YAML `simulate:` block):
```yaml
simulate:
  agent:
    strategy: sequential
    corpus:
      - { prompt: "hello", response: "world" }
  rest-request:
    strategy: key-lookup
    corpus:
      - { input: { url: "/api" }, output: { status: 200 } }
```

**Output:** `SimulationConfig` (existing type from simulation-runtime.ts) with:
- Aliases expanded to qualified names
- Corpus field-mapped per alias table
- Strategy strings passed through

### 4.2 Test Strategy

- Unit test: each alias expansion
- Unit test: direct qualified names (no alias) pass through
- Unit test: field mapping for agent alias (prompt→key, response→output)
- Unit test: mixed aliases and direct names in same block

---

## 5. Test Infrastructure (H4)

**Gap:** No `.test.yaml` discovery, execution, or test plugins.

### 5.1 YamlTestParser

New file: `test/yaml-test-parser.ts`

```typescript
export interface TestCase {
  name: string;
  steps: Record<string, unknown>[];
}

export interface TestFile {
  name: string;
  simulate?: Record<string, unknown>;
  setup?: Record<string, unknown>[];
  teardown?: Record<string, unknown>[];
  vars?: Record<string, unknown>;
  params?: Record<string, unknown>;
  tests: TestCase[];
}

export class YamlTestParser {
  static parse(raw: Record<string, unknown>): TestFile;
}
```

Handles single-test shorthand (`steps:` at top level) by wrapping in a single-element `tests` array.

### 5.2 YamlTestDiscovery

New file: `test/yaml-test-discovery.ts`

```typescript
export class YamlTestDiscovery {
  static scan(dir: string): string[];  // returns paths to *.test.yaml files
}
```

Recursively scans a directory for `*.test.yaml` files.

### 5.3 Test Plugins

#### assert

New file: `test/plugins/assert-plugin.ts`

Definition: `{ name: 'assert', inputs: { expression: { type: 'STRING', required: true } }, outputs: {} }`

Action: Resolves the expression parameter via the context's VariableResolver, then checks truthiness. Returns `stepSuccess({})` or `stepFailureWithCategory('Assertion failed: ...', 'STEP_ACTION_FAILED')`.

#### assertThat

New file: `test/plugins/assert-that-plugin.ts`

Definition: `{ name: 'assertThat', inputs: { expression: { type: 'STRING', required: true } }, outputs: {}, absorbSiblings: true }`

Action: Resolves the expression, then checks absorbed sibling matchers. 12 matchers:

| Matcher | Assertion |
|---------|-----------|
| `equals` | strict equality |
| `contains` | substring or array element |
| `matches` | regex match |
| `hasSize` | array/string length |
| `startsWith` | string prefix |
| `endsWith` | string suffix |
| `greaterThan` | numeric > |
| `greaterThanOrEqual` | numeric >= |
| `lessThan` | numeric < |
| `not-null` | not null/undefined (boolean flag) |
| `not-empty` | not empty string/array (boolean flag) |
| `empty` | is empty (boolean flag) |

Multiple matchers on the same step are AND-combined.

#### verify-simulation

New file: `test/plugins/verify-simulation-plugin.ts`

Definition: `{ name: 'verify-simulation', inputs: { action: { type: 'STRING', required: true } }, outputs: {}, absorbSiblings: true }`

Action: Looks up the `InvocationJournal` from the active `SimulationOverlay` and checks assertions:

| Assertion Key | Check |
|--------------|-------|
| `called` | exact call count |
| `calledAtLeast` | minimum call count |
| `calledAtMost` | maximum call count |
| `neverCalled` | zero calls (boolean flag) |
| `inOrder` | calls occurred in declared order |
| `exhaustive` | all corpus entries were consumed |

#### run-playbook

New file: `test/plugins/run-playbook-plugin.ts`

Definition: `{ name: 'run-playbook', inputs: { playbook: { type: 'STRING', required: true } }, outputs: {} }`

Action: Reads the referenced playbook YAML, compiles it via `Walker.resolve()`, executes it via `StructuralEvaluator`, and returns the final result.

#### validate-playbook

New file: `test/plugins/validate-playbook-plugin.ts`

Definition: `{ name: 'validate-playbook', inputs: { playbook: { type: 'STRING', required: true } }, outputs: { stepCount: {}, stepNames: {}, poolDeclarations: {} } }`

Action: Reads and compiles the playbook without executing. Returns structural metadata: step count, step names, pool declarations.

#### TestCatalog

New file: `test/plugins/test-catalog.ts`

Composes all 5 test plugins into a `Catalog` implementation. Used by `YamlTestExecutor` to create a `CompositeCatalog` with the application's main catalog.

### 5.4 YamlTestExecutor

New file: `test/yaml-test-executor.ts`

```typescript
export interface TestResult {
  name: string;
  status: 'pass' | 'fail' | 'error';
  message?: string;
  duration: number;
}

export interface TestSuiteResult {
  name: string;
  results: TestResult[];
  totalDuration: number;
}

export class YamlTestExecutor {
  constructor(
    private readonly catalog: Catalog,
    private readonly services: ServiceRegistry,
  ) {}

  async run(testFile: TestFile): Promise<TestSuiteResult>;
}
```

**Lifecycle per test case:**
1. Parse `simulate` block via `SimulateBlockParser.parse()` → push `SimulationOverlay`
2. Build `VariableResolver` from `vars` and `params`
3. Run `setup` steps (if any) via `StructuralEvaluator`
4. Run test `steps` via `StructuralEvaluator`
5. Run `teardown` steps (if any, even on failure) via `StructuralEvaluator`
6. Pop `SimulationOverlay`
7. Produce `TestResult`

**Test isolation:** Each test case gets its own `SimulationOverlay`, `VariableResolver`, and `ScenarioScope`. Tests within a suite do not share state.

### 5.5 Vitest Adapter

New file: `test/vitest-adapter.ts`

```typescript
export function describeYamlTests(
  dir: string,
  catalog: Catalog,
  services: ServiceRegistry,
): void;
```

Discovers `.test.yaml` files in `dir`, parses each, and generates Vitest `describe`/`it` blocks. Each test case becomes a Vitest test case with the yaml test's name.

### 5.6 New Directory Structure

```
packages/yaml-core/src/test/
├── index.ts
├── yaml-test-parser.ts
├── yaml-test-parser.test.ts
├── yaml-test-discovery.ts
├── yaml-test-discovery.test.ts
├── yaml-test-executor.ts
├── yaml-test-executor.test.ts
├── vitest-adapter.ts
├── vitest-adapter.test.ts
└── plugins/
    ├── test-catalog.ts
    ├── assert-plugin.ts
    ├── assert-plugin.test.ts
    ├── assert-that-plugin.ts
    ├── assert-that-plugin.test.ts
    ├── verify-simulation-plugin.ts
    ├── verify-simulation-plugin.test.ts
    ├── run-playbook-plugin.ts
    ├── run-playbook-plugin.test.ts
    ├── validate-playbook-plugin.ts
    └── validate-playbook-plugin.test.ts
```

### 5.7 Test Strategy

- YamlTestParser: single-test shorthand, multi-test, all optional fields, missing required fields
- YamlTestDiscovery: nested directories, non-test yaml files filtered
- Each plugin: success case, failure case, edge cases per matcher/assertion
- YamlTestExecutor: full lifecycle, setup failure aborts, teardown runs on failure, simulation overlay cleanup
- Vitest adapter: integration test with a real `.test.yaml` file

---

## File Summary

### New Files (26)

| File | Component |
|------|-----------|
| `step/error-category.ts` | H3 — ErrorCategory constants |
| `annotation/index.ts` | H1 — barrel export |
| `annotation/annotation-parser.ts` | H1 — #@ comment parser |
| `annotation/annotation-parser.test.ts` | H1 |
| `annotation/annotation-extractor.ts` | H1 — position correlation |
| `annotation/annotation-extractor.test.ts` | H1 |
| `annotation/annotation-map.ts` | H1 — immutable annotation store |
| `annotation/annotation-map.test.ts` | H1 |
| `annotation/injection-resolver.ts` | H1 — @Inject DI |
| `annotation/injection-resolver.test.ts` | H1 |
| `annotation/decorator-annotation-resolver.ts` | H1 — 12 decorator mappings |
| `annotation/decorator-annotation-resolver.test.ts` | H1 |
| `step/simulate-block-parser.ts` | M6 — test simulate parsing |
| `step/simulate-block-parser.test.ts` | M6 |
| `test/index.ts` | H4 — barrel export |
| `test/yaml-test-parser.ts` | H4 — .test.yaml parsing |
| `test/yaml-test-parser.test.ts` | H4 |
| `test/yaml-test-discovery.ts` | H4 — file scanning |
| `test/yaml-test-discovery.test.ts` | H4 |
| `test/yaml-test-executor.ts` | H4 — test lifecycle |
| `test/yaml-test-executor.test.ts` | H4 |
| `test/vitest-adapter.ts` | H4 — Vitest integration |
| `test/vitest-adapter.test.ts` | H4 |
| `test/plugins/test-catalog.ts` | H4 — plugin composition |
| `test/plugins/assert-plugin.ts` | H4 |
| `test/plugins/assert-that-plugin.ts` | H4 |
| `test/plugins/verify-simulation-plugin.ts` | H4 |
| `test/plugins/run-playbook-plugin.ts` | H4 |
| `test/plugins/validate-playbook-plugin.ts` | H4 |
| `test/plugins/*.test.ts` | H4 (5 test files) |

### Modified Files (2)

| File | Change |
|------|--------|
| `step/structural-evaluator.ts` | H3: error scope in catch; M4a: control step dispatch |
| `step/walker.ts` | H1: optional rawYaml param, annotation integration |

---

## References

- `docs/specs/2026-11-07-ts-platform-parity-audit.md` — Phase 2 gap definitions (H3, H4, M6, H1, M4a)
- `step/structural-evaluator.ts:170-191` — current evaluateTryCatchFinally (no error scope)
- `step/structural-evaluator.ts:39-53` — dispatch switch (no control case)
- `step/walker.ts:67-81` — ResolvedStep union (annotations field exists, always empty)
- `step/walker.ts:266-271` — Walker.resolve entry point
- `step/decorator-chain.ts:594-678` — DecoratorChain.build (unchanged by annotation design)
- `variable-resolver.ts:49-57` — withScope/withObjectScope (used for error scope injection)
- `step/simulation-runtime.ts` — SimulationConfig, SimulationOverlay, InvocationJournal (consumed by M6, H4)
- `step/types.ts:112-120` — Definition with absorbSiblings (used by assertThat)
- `orchestration/errors.ts` — existing typed errors (ChannelClosedError, IllegalTransitionError, SemaphoreReentrancyError)
- `step/scenario/lifecycle/` — lifecycle engine (used by M4a)
- casehubio/casehub-pages#608 — parent audit issue
