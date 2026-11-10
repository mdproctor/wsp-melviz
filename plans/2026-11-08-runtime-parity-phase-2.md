# Phase 2 — Runtime Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #621 — feat: Phase 2 — runtime parity (behaviour behind the unified schema)
**Issue group:** #621, #608

**Goal:** Add runtime behaviour to the TS YAML walker for 5 Phase 2 gaps: typed error catch scope, ControlStep evaluation, annotation system, simulate block parsing, and test infrastructure.

**Architecture:** All work lands in `packages/yaml-core/src/`. Two existing files are modified (structural-evaluator.ts, walker.ts). Two new directories are created (annotation/, test/). The annotation system uses a two-phase design: Walker resolution parses and stores annotations, StructuralEvaluator evaluation resolves @Inject bindings. Test plugins register as standard CatalogEntry definitions.

**Tech Stack:** TypeScript, Vitest, yaml-core package internals

## Global Constraints

- All paths relative to `packages/yaml-core/src/` unless stated otherwise
- Exports from new directories must be re-exported via barrel `index.ts` files
- All new public types/classes must be re-exported from `packages/yaml-core/src/index.ts`
- Run tests with: `yarn workspace yaml-core test` (or target specific: `yarn workspace yaml-core vitest run src/path/to/file.test.ts`)
- Commit after every task with `feat(#621):` prefix

---

## Batch 1: Runtime Quick Wins — H3 + M4a

After this batch: try-catch blocks inject `${error.type}`, `${error.message}`, `${error.step}`, `${error.detail}` into catch scope. ControlStep (pause/resume) evaluates against ConsumerHandle.

### Task 1: Typed Error Catch Scope

**Files:**
- Create: `step/error-category.ts`
- Modify: `step/structural-evaluator.ts:170-191` (evaluateTryCatchFinally)
- Test: `step/structural-evaluator.test.ts` (add error scope tests)

**Interfaces:**
- Consumes: `VariableResolver.withObjectScope(prefix, source)` from `variable-resolver.ts:54`
- Consumes: `ObjectVariableSource` from `types.ts`
- Consumes: `Result` type with `failureCategory?` and `detail?` from `walker.ts:14-16`
- Produces: `ErrorCategory` type, `ERROR_CATEGORIES` const array

- [ ] **Step 1: Write the failing test for error.type in catch scope**

```typescript
// In structural-evaluator.test.ts — add to existing test file
import { VariableResolver } from '../variable-resolver.js';

it('catch block receives ${error.type} from try failure', async () => {
  // Create a step that fails with category TIMEOUT
  const failAction = { execute: async () => stepFailureWithCategory('timed out', 'TIMEOUT') };
  const captureAction = {
    execute: async (params: Record<string, unknown>) => {
      return stepSuccess({ captured: params['expression'] });
    },
  };

  const failEntry: CatalogEntry = {
    qualifiedName: 'fail-step',
    definition: { name: 'fail-step', inputs: {}, outputs: {} },
    action: failAction,
  };
  const captureEntry: CatalogEntry = {
    qualifiedName: 'capture',
    definition: { name: 'capture', inputs: { expression: { type: 'STRING' as any, required: true } }, outputs: {} },
    action: captureAction,
  };

  const tryCatchStep: TryCatchFinallyStep = {
    kind: 'try-catch-finally',
    name: 'error-test',
    trySteps: [{ kind: 'plugin', name: 'will-fail', entry: failEntry, params: {}, decorators: {}, annotations: [] }],
    catchSteps: [{ kind: 'plugin', name: 'catch-step', entry: captureEntry, params: { expression: '${error.type}' }, decorators: {}, annotations: [] }],
    finallySteps: [],
    decorators: {},
    annotations: [],
  };

  const resolver = new VariableResolver({}, new Set());
  const context = buildContext(resolver);
  const evaluator = new StructuralEvaluator();
  const result = await evaluator.evaluate(tryCatchStep, context);

  expect(result.kind).toBe('success');
  expect(result.output['captured']).toBe('TIMEOUT');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace yaml-core vitest run src/step/structural-evaluator.test.ts -t "catch block receives"`
Expected: FAIL — catch steps currently get unresolved `${error.type}`

- [ ] **Step 3: Create error-category.ts**

```typescript
// step/error-category.ts
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

- [ ] **Step 4: Modify evaluateTryCatchFinally to inject error scope**

In `structural-evaluator.ts`, replace the `evaluateTryCatchFinally` method. Key changes:
1. Track `lastFailedStepName` during try loop
2. Build `ObjectVariableSource` for `error` prefix when try fails
3. Create scoped context for catch steps

```typescript
private async evaluateTryCatchFinally(step: TryCatchFinallyStep, context: Context): Promise<Result> {
  let tryResult: Result = stepSuccess({});
  let lastFailedStepName: string | null = null;

  for (let i = 0; i < step.trySteps.length; i++) {
    const child = step.trySteps[i]!;
    tryResult = await this.evaluate(child, context);
    if (tryResult.kind === 'failure') {
      lastFailedStepName = child.name ?? `root → Step ${i}`;
      break;
    }
  }

  if (tryResult.kind === 'failure' && step.catchSteps.length > 0) {
    const errorSource: ObjectVariableSource = {
      resolve(name: string): unknown {
        if (name === 'type') return tryResult.failureCategory ?? 'UNKNOWN';
        if (name === 'message') return tryResult.message;
        if (name === 'step') return lastFailedStepName;
        if (name.startsWith('detail.') && tryResult.detail) {
          const field = name.substring(7);
          return (tryResult.detail as Record<string, unknown>)[field];
        }
        if (name === 'detail') return tryResult.detail ? JSON.stringify(tryResult.detail) : '';
        return undefined;
      },
      allowContainerReturn() { return false; },
    };
    const scopedResolver = context.resolver.withObjectScope('error', errorSource);
    const catchContext: Context = { ...context, resolver: scopedResolver };

    let catchResult: Result = stepSuccess({});
    for (const child of step.catchSteps) {
      catchResult = await this.evaluate(child, catchContext);
      if (catchResult.kind === 'failure') break;
    }
    tryResult = catchResult;
  }

  for (const child of step.finallySteps) {
    await this.evaluate(child, context);
  }

  return tryResult;
}
```

Add import at top of file:
```typescript
import type { ObjectVariableSource } from '../types.js';
```

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace yaml-core vitest run src/step/structural-evaluator.test.ts -t "catch block receives"`
Expected: PASS

- [ ] **Step 6: Write additional tests — error.message, error.step, error.detail, nested try-catch**

```typescript
it('catch block receives ${error.message}', async () => {
  // Similar setup, verify ${error.message} resolves to 'timed out'
});

it('catch block receives ${error.step} with named step', async () => {
  // Failing step has name: 'will-fail', verify ${error.step} resolves to 'will-fail'
});

it('catch block receives ${error.step} fallback for unnamed step', async () => {
  // Failing step has name: null, verify ${error.step} resolves to 'root → Step 0'
});

it('catch block receives ${error.detail.field} deep access', async () => {
  // Failing step returns stepFailure with detail: { code: 504, reason: 'gateway' }
  // Verify ${error.detail.code} resolves to '504'
});

it('nested try-catch — inner error scope does not leak to outer', async () => {
  // Inner try-catch catches TIMEOUT, outer try-catch catches CHANNEL_CLOSED
  // Verify each catch scope has its own error.type
});
```

- [ ] **Step 7: Run all tests and verify pass**

Run: `yarn workspace yaml-core vitest run src/step/structural-evaluator.test.ts`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/step/error-category.ts packages/yaml-core/src/step/structural-evaluator.ts packages/yaml-core/src/step/structural-evaluator.test.ts
git commit -m "feat(#621): typed error catch scope — inject error.type/message/step/detail into catch blocks"
```

---

### Task 2: ControlStep Runtime Dispatch

**Files:**
- Modify: `step/structural-evaluator.ts:1-6,38-54` (import + dispatch switch)
- Test: `step/structural-evaluator.test.ts` (add control step tests)

**Interfaces:**
- Consumes: `ControlStep` type from `walker.ts:206-213`
- Consumes: `ScenarioScope.consumer(name)` returning `ConsumerHandle` from `orchestration/types.ts:69`
- Consumes: `ConsumerHandle.pause()`, `ConsumerHandle.resume()`, `ConsumerHandle.isPaused()` from `orchestration/consumer-handle.ts`
- Produces: No new exports — internal evaluator change

- [ ] **Step 1: Write the failing test for pause**

```typescript
it('evaluates pause ControlStep via consumer handle', async () => {
  const controlStep: ControlStep = {
    kind: 'control',
    name: 'pause-worker',
    control: 'pause',
    target: 'worker-1',
    decorators: {},
    annotations: [],
  };

  const context = buildContext(new VariableResolver({}, new Set()));
  const evaluator = new StructuralEvaluator();
  const result = await evaluator.evaluate(controlStep, context);

  expect(result.kind).toBe('success');
  expect(result.output['control']).toBe('pause');
  expect(result.output['target']).toBe('worker-1');
  expect(context.scope.consumer('worker-1').isPaused()).toBe(true);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace yaml-core vitest run src/step/structural-evaluator.test.ts -t "evaluates pause"`
Expected: FAIL — dispatch has no case for 'control'

- [ ] **Step 3: Add ControlStep dispatch and evaluateControl method**

In `structural-evaluator.ts`:

1. Add import: `import type { ControlStep } from './walker.js';` (add to existing import)
2. Add case in dispatch switch:
```typescript
case 'control': return this.evaluateControl(step, context);
```
3. Add method:
```typescript
private async evaluateControl(step: ControlStep, context: Context): Promise<Result> {
  const handle = context.scope.consumer(step.target);
  if (step.control === 'pause') {
    handle.pause();
  } else {
    handle.resume();
  }
  return stepSuccess({ control: step.control, target: step.target });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace yaml-core vitest run src/step/structural-evaluator.test.ts -t "evaluates pause"`
Expected: PASS

- [ ] **Step 5: Write resume and unregistered-target tests**

```typescript
it('evaluates resume ControlStep via consumer handle', async () => {
  const context = buildContext(new VariableResolver({}, new Set()));
  // Pre-pause the consumer
  context.scope.consumer('worker-1').pause();
  expect(context.scope.consumer('worker-1').isPaused()).toBe(true);

  const controlStep: ControlStep = {
    kind: 'control', name: 'resume-worker', control: 'resume',
    target: 'worker-1', decorators: {}, annotations: [],
  };

  const evaluator = new StructuralEvaluator();
  const result = await evaluator.evaluate(controlStep, context);

  expect(result.kind).toBe('success');
  expect(context.scope.consumer('worker-1').isPaused()).toBe(false);
});

it('pause on unregistered target creates handle in paused state', async () => {
  const context = buildContext(new VariableResolver({}, new Set()));
  const controlStep: ControlStep = {
    kind: 'control', name: null, control: 'pause',
    target: 'unknown-consumer', decorators: {}, annotations: [],
  };

  const evaluator = new StructuralEvaluator();
  const result = await evaluator.evaluate(controlStep, context);

  expect(result.kind).toBe('success');
  expect(context.scope.consumer('unknown-consumer').isPaused()).toBe(true);
});
```

- [ ] **Step 6: Run all tests**

Run: `yarn workspace yaml-core vitest run src/step/structural-evaluator.test.ts`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/src/step/structural-evaluator.ts packages/yaml-core/src/step/structural-evaluator.test.ts
git commit -m "feat(#621): ControlStep runtime — wire pause/resume to ConsumerHandle"
```

---

## Batch 2: Annotation System Foundation — H1 Parser/Scanner/Extractor/Map

After this batch: raw YAML text can be scanned for `#@Name(args)` annotations and correlated to step positions, producing an `AnnotationMap`.

### Task 3: AnnotationParser + AnnotationMap

**Files:**
- Create: `annotation/annotation-parser.ts`
- Create: `annotation/annotation-parser.test.ts`
- Create: `annotation/annotation-map.ts`
- Create: `annotation/annotation-map.test.ts`
- Create: `annotation/index.ts`

**Interfaces:**
- Produces: `ParsedAnnotation { name: string; args: Record<string, string>; line: number }`
- Produces: `AnnotationParser.parse(rawYaml: string): ParsedAnnotation[]`
- Produces: `Annotation { name: string; args: Record<string, string> }`
- Produces: `AnnotationMap.forStep(address: string): Annotation[]`
- Produces: `AnnotationMap.forDocument(name: string): Annotation | undefined`
- Produces: `AnnotationMap.hasStepAnnotation(address: string, name: string): boolean`
- Produces: `AnnotationMap.allDocumentAnnotations(): Annotation[]`
- Produces: `AnnotationMap.empty(): AnnotationMap`

- [ ] **Step 1: Write failing test for AnnotationParser — single annotation**

```typescript
// annotation/annotation-parser.test.ts
import { AnnotationParser } from './annotation-parser.js';

it('parses a single annotation with positional arg', () => {
  const yaml = `#@Timeout("5s")\nlog: hello`;
  const result = AnnotationParser.parse(yaml);
  expect(result).toHaveLength(1);
  expect(result[0]!.name).toBe('Timeout');
  expect(result[0]!.args).toEqual({ '0': '5s' });
  expect(result[0]!.line).toBe(0);
});
```

- [ ] **Step 2: Run test to verify fail**

Run: `yarn workspace yaml-core vitest run src/annotation/annotation-parser.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement AnnotationParser**

```typescript
// annotation/annotation-parser.ts
export interface ParsedAnnotation {
  name: string;
  args: Record<string, string>;
  line: number;
}

const ANNOTATION_RE = /^\s*#@(\w+)(?:\(([^)]*)\))?\s*$/;

export class AnnotationParser {
  static parse(rawYaml: string): ParsedAnnotation[] {
    const annotations: ParsedAnnotation[] = [];
    const lines = rawYaml.split('\n');
    for (let i = 0; i < lines.length; i++) {
      const match = ANNOTATION_RE.exec(lines[i]!);
      if (!match) continue;
      const name = match[1]!;
      const argsStr = match[2];
      const args = argsStr ? AnnotationParser.parseArgs(argsStr) : {};
      annotations.push({ name, args, line: i });
    }
    return annotations;
  }

  private static parseArgs(argsStr: string): Record<string, string> {
    const args: Record<string, string> = {};
    const tokens = AnnotationParser.tokenizeArgs(argsStr);
    let positionalIndex = 0;
    for (const token of tokens) {
      const colonIdx = token.indexOf(':');
      if (colonIdx > 0 && !token.startsWith('"') && !token.startsWith("'")) {
        const key = token.substring(0, colonIdx).trim();
        const value = AnnotationParser.unquote(token.substring(colonIdx + 1).trim());
        args[key] = value;
      } else {
        args[String(positionalIndex)] = AnnotationParser.unquote(token.trim());
        positionalIndex++;
      }
    }
    return args;
  }

  private static tokenizeArgs(argsStr: string): string[] {
    const tokens: string[] = [];
    let current = '';
    let inQuote: string | null = null;
    for (const ch of argsStr) {
      if (inQuote) {
        current += ch;
        if (ch === inQuote) inQuote = null;
      } else if (ch === '"' || ch === "'") {
        current += ch;
        inQuote = ch;
      } else if (ch === ',') {
        if (current.trim()) tokens.push(current.trim());
        current = '';
      } else {
        current += ch;
      }
    }
    if (current.trim()) tokens.push(current.trim());
    return tokens;
  }

  private static unquote(s: string): string {
    if ((s.startsWith('"') && s.endsWith('"')) || (s.startsWith("'") && s.endsWith("'"))) {
      return s.slice(1, -1);
    }
    return s;
  }
}
```

- [ ] **Step 4: Run test to verify pass**

Run: `yarn workspace yaml-core vitest run src/annotation/annotation-parser.test.ts`
Expected: PASS

- [ ] **Step 5: Add comprehensive parser tests**

```typescript
it('parses annotation with named args', () => {
  const yaml = `#@Retry(max: "3", backoff: "exponential")`;
  const result = AnnotationParser.parse(yaml);
  expect(result[0]!.args).toEqual({ max: '3', backoff: 'exponential' });
});

it('parses annotation with no args', () => {
  const yaml = `#@Background`;
  const result = AnnotationParser.parse(yaml);
  expect(result[0]!.name).toBe('Background');
  expect(result[0]!.args).toEqual({});
});

it('parses multiple annotations with correct line numbers', () => {
  const yaml = `#@Timeout("5s")\n#@Retry(max: "3")\nlog: hello`;
  const result = AnnotationParser.parse(yaml);
  expect(result).toHaveLength(2);
  expect(result[0]!.line).toBe(0);
  expect(result[1]!.line).toBe(1);
});

it('ignores regular YAML comments', () => {
  const yaml = `# This is a comment\n#@Timeout("5s")\nlog: hello`;
  const result = AnnotationParser.parse(yaml);
  expect(result).toHaveLength(1);
  expect(result[0]!.name).toBe('Timeout');
});

it('handles indented annotations', () => {
  const yaml = `block:\n  #@Timeout("5s")\n  - log: hello`;
  const result = AnnotationParser.parse(yaml);
  expect(result).toHaveLength(1);
});
```

- [ ] **Step 6: Implement AnnotationMap**

```typescript
// annotation/annotation-map.ts
export interface Annotation {
  name: string;
  args: Record<string, string>;
}

export class AnnotationMap {
  private readonly stepAnnotations: Map<string, Annotation[]>;
  private readonly documentAnnotations: Annotation[];

  constructor(stepAnnotations: Map<string, Annotation[]>, documentAnnotations: Annotation[]) {
    this.stepAnnotations = stepAnnotations;
    this.documentAnnotations = documentAnnotations;
  }

  forStep(address: string): Annotation[] {
    return this.stepAnnotations.get(address) ?? [];
  }

  forDocument(name: string): Annotation | undefined {
    return this.documentAnnotations.find(a => a.name === name);
  }

  hasStepAnnotation(address: string, name: string): boolean {
    return this.forStep(address).some(a => a.name === name);
  }

  allDocumentAnnotations(): Annotation[] {
    return [...this.documentAnnotations];
  }

  static empty(): AnnotationMap {
    return new AnnotationMap(new Map(), []);
  }
}
```

- [ ] **Step 7: Write AnnotationMap tests**

```typescript
// annotation/annotation-map.test.ts
it('returns annotations for a step address', () => {
  const map = new AnnotationMap(
    new Map([['0', [{ name: 'Timeout', args: { '0': '5s' } }]]]),
    [],
  );
  expect(map.forStep('0')).toHaveLength(1);
  expect(map.forStep('1')).toHaveLength(0);
});

it('returns document-level annotations', () => {
  const map = new AnnotationMap(new Map(), [{ name: 'Version', args: { '0': '1.0' } }]);
  expect(map.forDocument('Version')?.args['0']).toBe('1.0');
  expect(map.forDocument('Missing')).toBeUndefined();
});

it('empty map returns empty results', () => {
  const map = AnnotationMap.empty();
  expect(map.forStep('0')).toHaveLength(0);
  expect(map.allDocumentAnnotations()).toHaveLength(0);
});
```

- [ ] **Step 8: Create barrel export**

```typescript
// annotation/index.ts
export { AnnotationParser } from './annotation-parser.js';
export type { ParsedAnnotation } from './annotation-parser.js';
export { AnnotationMap } from './annotation-map.js';
export type { Annotation } from './annotation-map.js';
```

- [ ] **Step 9: Run all annotation tests**

Run: `yarn workspace yaml-core vitest run src/annotation/`
Expected: All PASS

- [ ] **Step 10: Commit**

```bash
git add packages/yaml-core/src/annotation/
git commit -m "feat(#621): AnnotationParser + AnnotationMap — parse #@Name(args) from YAML comments"
```

---

### Task 4: StepPositionScanner + AnnotationExtractor

**Files:**
- Create: `annotation/step-position-scanner.ts`
- Create: `annotation/step-position-scanner.test.ts`
- Create: `annotation/annotation-extractor.ts`
- Create: `annotation/annotation-extractor.test.ts`
- Modify: `annotation/index.ts` (add exports)

**Interfaces:**
- Consumes: `ParsedAnnotation` from Task 3
- Consumes: `AnnotationMap`, `Annotation` from Task 3
- Produces: `StepLineInfo { line: number; address: string }`
- Produces: `StepPositionScanner.scan(rawYaml: string): StepLineInfo[]`
- Produces: `AnnotationExtractor.extract(annotations: ParsedAnnotation[], stepPositions: StepLineInfo[]): AnnotationMap`

- [ ] **Step 1: Write failing test for StepPositionScanner**

```typescript
// annotation/step-position-scanner.test.ts
import { StepPositionScanner } from './step-position-scanner.js';

it('identifies step positions from YAML keys', () => {
  const yaml = [
    '- log: hello',
    '- rest-request GET /api:',
    '    body: {}',
  ].join('\n');
  const positions = StepPositionScanner.scan(yaml);
  expect(positions).toHaveLength(2);
  expect(positions[0]!.line).toBe(0);
  expect(positions[0]!.address).toBe('0');
  expect(positions[1]!.line).toBe(1);
  expect(positions[1]!.address).toBe('1');
});
```

- [ ] **Step 2: Run test to verify fail, then implement StepPositionScanner**

The scanner identifies lines that start a new step in a YAML steps array. It looks for `- ` prefixed lines (array items) and tracks depth for hierarchical addressing.

```typescript
// annotation/step-position-scanner.ts
export interface StepLineInfo {
  line: number;
  address: string;
}

export class StepPositionScanner {
  static scan(rawYaml: string): StepLineInfo[] {
    const lines = rawYaml.split('\n');
    const positions: StepLineInfo[] = [];
    let stepIndex = 0;

    for (let i = 0; i < lines.length; i++) {
      const line = lines[i]!;
      const stripped = line.trimStart();
      if (stripped.startsWith('- ') && !stripped.startsWith('- {')) {
        positions.push({ line: i, address: String(stepIndex) });
        stepIndex++;
      }
    }
    return positions;
  }
}
```

- [ ] **Step 3: Write failing test for AnnotationExtractor**

```typescript
// annotation/annotation-extractor.test.ts
import { AnnotationExtractor } from './annotation-extractor.js';
import type { ParsedAnnotation } from './annotation-parser.js';
import type { StepLineInfo } from './step-position-scanner.js';

it('assigns annotation to next step when between steps', () => {
  const annotations: ParsedAnnotation[] = [
    { name: 'Timeout', args: { '0': '5s' }, line: 1 },
  ];
  const positions: StepLineInfo[] = [
    { line: 0, address: '0' },
    { line: 2, address: '1' },
  ];
  const map = AnnotationExtractor.extract(annotations, positions);
  expect(map.forStep('1')).toHaveLength(1);
  expect(map.forStep('1')[0]!.name).toBe('Timeout');
  expect(map.forStep('0')).toHaveLength(0);
});

it('assigns annotation before first step as document-level', () => {
  const annotations: ParsedAnnotation[] = [
    { name: 'Version', args: { '0': '1.0' }, line: 0 },
  ];
  const positions: StepLineInfo[] = [
    { line: 2, address: '0' },
  ];
  const map = AnnotationExtractor.extract(annotations, positions);
  expect(map.forDocument('Version')).toBeDefined();
  expect(map.forStep('0')).toHaveLength(0);
});
```

- [ ] **Step 4: Implement AnnotationExtractor**

```typescript
// annotation/annotation-extractor.ts
import type { ParsedAnnotation } from './annotation-parser.js';
import type { StepLineInfo } from './step-position-scanner.js';
import { AnnotationMap } from './annotation-map.js';
import type { Annotation } from './annotation-map.js';

export class AnnotationExtractor {
  static extract(annotations: ParsedAnnotation[], stepPositions: StepLineInfo[]): AnnotationMap {
    const stepMap = new Map<string, Annotation[]>();
    const documentAnnotations: Annotation[] = [];

    for (const ann of annotations) {
      const annotation: Annotation = { name: ann.name, args: ann.args };
      const targetStep = AnnotationExtractor.findTargetStep(ann.line, stepPositions);
      if (targetStep === null) {
        documentAnnotations.push(annotation);
      } else {
        const existing = stepMap.get(targetStep) ?? [];
        existing.push(annotation);
        stepMap.set(targetStep, existing);
      }
    }

    return new AnnotationMap(stepMap, documentAnnotations);
  }

  private static findTargetStep(annotationLine: number, positions: StepLineInfo[]): string | null {
    if (positions.length === 0) return null;
    if (annotationLine < positions[0]!.line) return null;

    for (let i = positions.length - 1; i >= 0; i--) {
      if (annotationLine < positions[i]!.line) continue;
      if (i + 1 < positions.length) return positions[i + 1]!.address;
      return positions[i]!.address;
    }
    return null;
  }
}
```

- [ ] **Step 5: Run tests, fix AnnotationExtractor logic, add more tests**

Run: `yarn workspace yaml-core vitest run src/annotation/`
Iterate until all pass. Add tests for:
- Multiple annotations on same step
- Annotations on step 0 (right before it)
- Empty annotations list
- Empty positions list

- [ ] **Step 6: Update barrel export**

Add to `annotation/index.ts`:
```typescript
export { StepPositionScanner } from './step-position-scanner.js';
export type { StepLineInfo } from './step-position-scanner.js';
export { AnnotationExtractor } from './annotation-extractor.js';
```

- [ ] **Step 7: Run all annotation tests**

Run: `yarn workspace yaml-core vitest run src/annotation/`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/annotation/
git commit -m "feat(#621): StepPositionScanner + AnnotationExtractor — correlate annotations to steps"
```

---

## Batch 3: Annotation System Runtime — H1 Resolvers + Integration

After this batch: Walker.resolve() with rawYaml populates annotations on steps. DecoratorAnnotationResolver merges annotation-based decorators. InjectionResolver runs at evaluation time.

### Task 5: DecoratorAnnotationResolver + InjectionResolver

**Files:**
- Create: `annotation/decorator-annotation-resolver.ts`
- Create: `annotation/decorator-annotation-resolver.test.ts`
- Create: `annotation/injection-resolver.ts`
- Create: `annotation/injection-resolver.test.ts`
- Modify: `annotation/index.ts` (add exports)

**Interfaces:**
- Consumes: `Annotation` from Task 3
- Consumes: `ServiceRegistry` from `walker.ts:30`
- Produces: `DecoratorAnnotationResolver.mergeDecorators(annotations: Annotation[], decorators: Record<string, unknown>): Record<string, unknown>`
- Produces: `InjectionResolver.resolve(annotations: Annotation[], params: Record<string, unknown>, services: ServiceRegistry): Record<string, unknown>`

- [ ] **Step 1: Write failing test for DecoratorAnnotationResolver**

```typescript
// annotation/decorator-annotation-resolver.test.ts
import { DecoratorAnnotationResolver } from './decorator-annotation-resolver.js';
import type { Annotation } from './annotation-map.js';

it('merges @Timeout annotation into decorators', () => {
  const annotations: Annotation[] = [{ name: 'Timeout', args: { '0': '5s' } }];
  const decorators: Record<string, unknown> = {};
  const merged = DecoratorAnnotationResolver.mergeDecorators(annotations, decorators);
  expect(merged['timeout']).toBe('5s');
});

it('inline decorator takes precedence over annotation', () => {
  const annotations: Annotation[] = [{ name: 'Timeout', args: { '0': '5s' } }];
  const decorators: Record<string, unknown> = { timeout: '10s' };
  const merged = DecoratorAnnotationResolver.mergeDecorators(annotations, decorators);
  expect(merged['timeout']).toBe('10s');
});
```

- [ ] **Step 2: Implement DecoratorAnnotationResolver**

```typescript
// annotation/decorator-annotation-resolver.ts
import type { Annotation } from './annotation-map.js';

const DECORATOR_ANNOTATION_MAP: Record<string, (args: Record<string, string>) => [string, unknown]> = {
  Timeout: (args) => ['timeout', args['0'] ?? args['value']],
  Retry: (args) => {
    const spec: Record<string, unknown> = {};
    if (args['max']) spec['max'] = Number(args['max']);
    if (args['backoff']) spec['backoff'] = args['backoff'];
    if (args['0']) spec['max'] = Number(args['0']);
    return ['retry', Object.keys(spec).length === 1 && spec['max'] ? spec['max'] : spec];
  },
  Delay: (args) => ['delay', args['0'] ?? args['value']],
  Priority: (args) => ['priority', args['0'] ?? args['value']],
  Resource: (args) => ['resource', args['0'] ?? args['value']],
  Background: () => ['background', true],
  OnError: (args) => ['on-error', args['0'] ?? args['value']],
  Trigger: (args) => ['trigger', args['0'] ?? args['value']],
  Semaphore: (args) => ['semaphore', args['0'] ?? args['value']],
  At: (args) => ['at', args['0'] ?? args['value']],
  Cancel: (args) => ['cancel', args['0'] ?? args['value']],
  Mutex: (args) => ['mutex', args['0'] ?? args['value']],
};

export class DecoratorAnnotationResolver {
  static mergeDecorators(
    annotations: Annotation[],
    decorators: Record<string, unknown>,
  ): Record<string, unknown> {
    const merged = { ...decorators };
    for (const ann of annotations) {
      const mapper = DECORATOR_ANNOTATION_MAP[ann.name];
      if (!mapper) continue;
      const [key, value] = mapper(ann.args);
      if (!(key in merged)) {
        merged[key] = value;
      }
    }
    return merged;
  }
}
```

- [ ] **Step 3: Run tests, add comprehensive tests for all 13 decorators**

- [ ] **Step 4: Write failing test for InjectionResolver**

```typescript
// annotation/injection-resolver.test.ts
import { InjectionResolver } from './injection-resolver.js';
import { MapServiceRegistry } from '../step/walker.js';
import type { Annotation } from './annotation-map.js';

it('resolves @Inject annotation from ServiceRegistry', () => {
  const annotations: Annotation[] = [
    { name: 'Inject', args: { '0': 'DatabaseConnection', qualifier: 'primary' } },
  ];
  const params: Record<string, unknown> = { query: 'SELECT 1' };
  const registry = new MapServiceRegistry();
  const mockDb = { name: 'primary-db' };
  registry.register({ name: 'DatabaseConnection' }, mockDb);

  const result = InjectionResolver.resolve(annotations, params, registry);
  expect(result['DatabaseConnection']).toBe(mockDb);
  expect(result['query']).toBe('SELECT 1');
});

it('throws clear error when injected type not found', () => {
  const annotations: Annotation[] = [
    { name: 'Inject', args: { '0': 'MissingService' } },
  ];
  expect(() => InjectionResolver.resolve(annotations, {}, new MapServiceRegistry()))
    .toThrow(/MissingService/);
});
```

- [ ] **Step 5: Implement InjectionResolver**

```typescript
// annotation/injection-resolver.ts
import type { Annotation } from './annotation-map.js';
import type { ServiceRegistry } from '../step/walker.js';

export class InjectionResolver {
  static resolve(
    annotations: Annotation[],
    params: Record<string, unknown>,
    services: ServiceRegistry,
  ): Record<string, unknown> {
    const augmented = { ...params };
    for (const ann of annotations) {
      if (ann.name !== 'Inject') continue;
      const typeName = ann.args['0'];
      if (!typeName) continue;
      try {
        const instance = services.lookup({ name: typeName });
        augmented[typeName] = instance;
      } catch {
        throw new Error(`@Inject failed: service '${typeName}' not found in registry`);
      }
    }
    return augmented;
  }
}
```

- [ ] **Step 6: Run all tests, update barrel export**

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/src/annotation/
git commit -m "feat(#621): DecoratorAnnotationResolver + InjectionResolver — annotation-to-decorator mapping and @Inject DI"
```

---

### Task 6: Walker + StructuralEvaluator Annotation Integration

**Files:**
- Modify: `step/walker.ts:266-271` (resolve signature + annotation threading)
- Modify: `step/structural-evaluator.ts:24-36` (evaluate — injection before dispatch)
- Create: `step/walker.test.ts` (add annotation integration tests)
- Modify: `annotation/index.ts` (final exports)

**Interfaces:**
- Consumes: `AnnotationParser`, `StepPositionScanner`, `AnnotationExtractor`, `AnnotationMap` from Tasks 3-4
- Consumes: `DecoratorAnnotationResolver`, `InjectionResolver` from Task 5
- Produces: `Walker.resolve(steps, catalog, rawYaml?)` — enhanced signature

- [ ] **Step 1: Write failing integration test — Walker with rawYaml produces annotated steps**

```typescript
// In walker.test.ts
it('Walker.resolve with rawYaml populates annotations and merges decorators', () => {
  const rawYaml = [
    '#@Timeout("5s")',
    '- log: hello',
    '- log: world',
  ].join('\n');

  const steps = [
    { log: 'hello' },
    { log: 'world' },
  ];

  const catalog = createTestCatalog(); // helper that registers 'log'
  const resolved = Walker.resolve(steps, catalog, rawYaml);

  expect(resolved[0]!.annotations).toHaveLength(1);
  expect(resolved[0]!.decorators['timeout']).toBe('5s');
  expect(resolved[1]!.annotations).toHaveLength(0);
});
```

- [ ] **Step 2: Run test to verify fail**

- [ ] **Step 3: Modify Walker.resolve to accept rawYaml and thread annotations**

Add optional `rawYaml` parameter to `Walker.resolve()`. When provided:
1. Parse annotations via `AnnotationParser.parse(rawYaml)`
2. Scan step positions via `StepPositionScanner.scan(rawYaml)`
3. Extract annotation map via `AnnotationExtractor.extract()`
4. Pass annotation map to `resolveAtDepth()` via an internal context parameter
5. In `resolveOne()`, look up step annotations by address
6. Call `DecoratorAnnotationResolver.mergeDecorators()` on the decorators
7. Store annotations on the resolved step

The internal context parameter is threaded through `resolveAtDepth` and `resolveOne` to avoid changing public signatures beyond the `rawYaml` addition.

- [ ] **Step 4: Run test to verify pass**

- [ ] **Step 5: Modify StructuralEvaluator.evaluate to run InjectionResolver**

In `structural-evaluator.ts`, before `this.dispatch(step, context)`, check for `@Inject` annotations:

```typescript
async evaluate(step: ResolvedStep, context: Context): Promise<Result> {
  let processedStep = step;
  if (step.annotations.length > 0 && 'params' in step) {
    const annotations = step.annotations as Annotation[];
    const hasInject = annotations.some(a => a.name === 'Inject');
    if (hasInject) {
      const augmentedParams = InjectionResolver.resolve(annotations, (step as any).params, context.services);
      processedStep = { ...step, params: augmentedParams } as ResolvedStep;
    }
  }
  const result = await this.dispatch(processedStep, context);
  // ... rest unchanged
}
```

- [ ] **Step 6: Write test for InjectionResolver in StructuralEvaluator**

- [ ] **Step 7: Run all yaml-core tests**

Run: `yarn workspace yaml-core vitest run`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/step/walker.ts packages/yaml-core/src/step/walker.test.ts packages/yaml-core/src/step/structural-evaluator.ts packages/yaml-core/src/annotation/
git commit -m "feat(#621): Walker + StructuralEvaluator annotation integration — rawYaml param, decorator merge, @Inject DI"
```

---

## Batch 4: Test Infrastructure — M6 + H4

After this batch: `.test.yaml` files can be discovered, parsed, and executed with simulation support, test assertions, and Vitest integration.

### Task 7: SimulateBlockParser + YamlTestParser + YamlTestDiscovery

**Files:**
- Create: `test/simulate-block-parser.ts`
- Create: `test/simulate-block-parser.test.ts`
- Create: `test/yaml-test-parser.ts`
- Create: `test/yaml-test-parser.test.ts`
- Create: `test/yaml-test-discovery.ts`
- Create: `test/yaml-test-discovery.test.ts`
- Create: `test/index.ts`

**Interfaces:**
- Consumes: `SimulationConfig` from `step/simulation-runtime.ts:24-29`
- Produces: `SimulateBlockParser.parse(raw: Record<string, unknown>): SimulationConfig`
- Produces: `TestFile`, `TestCase` types
- Produces: `YamlTestParser.parse(raw: Record<string, unknown>): TestFile`
- Produces: `YamlTestDiscovery.scan(dir: string): string[]`

- [ ] **Step 1: Write failing test for SimulateBlockParser — agent alias**

```typescript
// test/simulate-block-parser.test.ts
import { SimulateBlockParser } from './simulate-block-parser.js';

it('expands agent alias to agent-provider.invoke with field mapping', () => {
  const raw = {
    agent: {
      strategy: 'sequential',
      corpus: [{ prompt: 'hello', response: 'world' }],
    },
  };
  const config = SimulateBlockParser.parse(raw);
  expect(config.strategies['agent-provider.invoke']).toBe('sequential');
  const corpus = config.corpus[0]!;
  expect('inline' in corpus).toBe(true);
  if ('inline' in corpus) {
    const entries = corpus.inline['agent-provider.invoke']!;
    expect(entries[0]!.input).toEqual({ key: 'hello' });
    expect(entries[0]!.output).toBe('world');
  }
});
```

- [ ] **Step 2: Implement SimulateBlockParser**

```typescript
// test/simulate-block-parser.ts
import type { SimulationConfig, CorpusDatum, CorpusEntry } from '../step/simulation-runtime.js';

const ALIAS_TABLE: Record<string, { qualifiedName: string; mapFields?: (entry: Record<string, unknown>) => CorpusDatum }> = {
  agent: {
    qualifiedName: 'agent-provider.invoke',
    mapFields: (entry) => ({
      input: { key: entry['prompt'] },
      output: entry['response'] as unknown,
    }),
  },
  rest: { qualifiedName: 'rest-request' },
  graphql: { qualifiedName: 'graphql' },
  mcp: { qualifiedName: 'mcp-call' },
  process: { qualifiedName: 'process-exec' },
};

export class SimulateBlockParser {
  static parse(raw: Record<string, unknown>): SimulationConfig {
    const strategies: Record<string, string> = {};
    const corpusEntries: Record<string, CorpusDatum[]> = {};

    for (const [key, value] of Object.entries(raw)) {
      if (key === 'capture') continue;
      const alias = ALIAS_TABLE[key];
      const qualifiedName = alias ? alias.qualifiedName : key;
      const spec = value as Record<string, unknown>;

      strategies[qualifiedName] = spec['strategy'] as string ?? 'sequential';

      if (Array.isArray(spec['corpus'])) {
        const mapFn = alias?.mapFields;
        corpusEntries[qualifiedName] = (spec['corpus'] as Record<string, unknown>[]).map(entry => {
          if (mapFn) return mapFn(entry);
          return { input: entry['input'], output: entry['output'] } as CorpusDatum;
        });
      }
    }

    const corpus: CorpusEntry[] = Object.keys(corpusEntries).length > 0
      ? [{ inline: corpusEntries }]
      : [];

    return {
      strategies,
      corpus,
      capture: Array.isArray(raw['capture']) ? raw['capture'] as string[] : [],
    };
  }
}
```

- [ ] **Step 3: Add tests for direct names, mixed aliases, capture key**

- [ ] **Step 4: Implement YamlTestParser**

```typescript
// test/yaml-test-parser.ts
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
  static parse(raw: Record<string, unknown>): TestFile {
    const name = (raw['name'] as string) ?? 'unnamed';

    if (raw['steps'] && !raw['tests']) {
      return {
        name,
        simulate: raw['simulate'] as Record<string, unknown> | undefined,
        setup: raw['setup'] as Record<string, unknown>[] | undefined,
        teardown: raw['teardown'] as Record<string, unknown>[] | undefined,
        vars: raw['vars'] as Record<string, unknown> | undefined,
        params: raw['params'] as Record<string, unknown> | undefined,
        tests: [{ name, steps: raw['steps'] as Record<string, unknown>[] }],
      };
    }

    const rawTests = (raw['tests'] as Record<string, unknown>[]) ?? [];
    const tests = rawTests.map(t => ({
      name: (t['name'] as string) ?? 'unnamed',
      steps: (t['steps'] as Record<string, unknown>[]) ?? [],
    }));

    return {
      name,
      simulate: raw['simulate'] as Record<string, unknown> | undefined,
      setup: raw['setup'] as Record<string, unknown>[] | undefined,
      teardown: raw['teardown'] as Record<string, unknown>[] | undefined,
      vars: raw['vars'] as Record<string, unknown> | undefined,
      params: raw['params'] as Record<string, unknown> | undefined,
      tests,
    };
  }
}
```

- [ ] **Step 5: Write YamlTestParser tests — multi-test, single-test shorthand, optional fields**

- [ ] **Step 6: Implement YamlTestDiscovery**

```typescript
// test/yaml-test-discovery.ts
import { readdirSync, statSync } from 'node:fs';
import { join } from 'node:path';

export class YamlTestDiscovery {
  static scan(dir: string): string[] {
    const results: string[] = [];
    YamlTestDiscovery.walk(dir, results);
    return results.sort();
  }

  private static walk(dir: string, results: string[]): void {
    for (const entry of readdirSync(dir)) {
      const full = join(dir, entry);
      if (statSync(full).isDirectory()) {
        YamlTestDiscovery.walk(full, results);
      } else if (entry.endsWith('.test.yaml') || entry.endsWith('.test.yml')) {
        results.push(full);
      }
    }
  }
}
```

- [ ] **Step 7: Write YamlTestDiscovery test with temp directory**

- [ ] **Step 8: Create barrel export, run all tests, commit**

```bash
git add packages/yaml-core/src/test/
git commit -m "feat(#621): SimulateBlockParser + YamlTestParser + YamlTestDiscovery"
```

---

### Task 8: Test Plugins + TestCatalog

**Files:**
- Create: `test/plugins/assert-plugin.ts`
- Create: `test/plugins/assert-plugin.test.ts`
- Create: `test/plugins/assert-that-plugin.ts`
- Create: `test/plugins/assert-that-plugin.test.ts`
- Create: `test/plugins/verify-simulation-plugin.ts`
- Create: `test/plugins/verify-simulation-plugin.test.ts`
- Create: `test/plugins/run-playbook-plugin.ts`
- Create: `test/plugins/run-playbook-plugin.test.ts`
- Create: `test/plugins/validate-playbook-plugin.ts`
- Create: `test/plugins/validate-playbook-plugin.test.ts`
- Create: `test/plugins/test-catalog.ts`

**Interfaces:**
- Consumes: `Action`, `Result`, `CatalogEntry`, `Catalog`, `ServiceRegistry` from `walker.ts`
- Consumes: `Definition` from `types.ts`
- Consumes: `SimulationRuntime`, `InvocationJournal` from `simulation-runtime.ts`
- Consumes: `isTruthy` from `truthiness.ts`
- Produces: `TestCatalog` implementing `Catalog` with 5 test plugins

- [ ] **Step 1: Write failing test for assert plugin**

```typescript
// test/plugins/assert-plugin.test.ts
it('assert passes for truthy expression', async () => {
  const plugin = createAssertPlugin();
  const result = await plugin.action.execute({ expression: 'true' }, new MapServiceRegistry());
  expect(result.kind).toBe('success');
});

it('assert fails for falsy expression', async () => {
  const plugin = createAssertPlugin();
  const result = await plugin.action.execute({ expression: '' }, new MapServiceRegistry());
  expect(result.kind).toBe('failure');
  expect(result.failureCategory).toBe('STEP_ACTION_FAILED');
});
```

- [ ] **Step 2: Implement assert plugin**

```typescript
// test/plugins/assert-plugin.ts
import type { Action, Result, CatalogEntry, ServiceRegistry } from '../../step/walker.js';
import { stepSuccess, stepFailureWithCategory } from '../../step/walker.js';
import { isTruthy } from '../../truthiness.js';
import type { Definition } from '../../step/types.js';

const assertDefinition: Definition = {
  name: 'assert',
  inputs: { expression: { type: 'STRING' as any, required: true } },
  outputs: {},
};

const assertAction: Action = {
  async execute(params: Record<string, unknown>): Promise<Result> {
    const expr = String(params['expression'] ?? params['_subject'] ?? '');
    if (isTruthy(expr)) return stepSuccess({});
    return stepFailureWithCategory(`Assertion failed: '${expr}' is falsy`, 'STEP_ACTION_FAILED');
  },
};

export function createAssertPlugin(): CatalogEntry {
  return { qualifiedName: 'assert', definition: assertDefinition, action: assertAction };
}
```

- [ ] **Step 3: Implement assertThat plugin with 12 matchers**

Each matcher is a function `(actual: unknown, expected: unknown) => boolean`. The plugin iterates absorbed params (keys not in the definition's inputs), looks up each as a matcher, and AND-combines results.

```typescript
// test/plugins/assert-that-plugin.ts
const MATCHERS: Record<string, (actual: unknown, expected: unknown) => boolean> = {
  equals: (a, e) => a === e || String(a) === String(e),
  contains: (a, e) => String(a).includes(String(e)),
  matches: (a, e) => new RegExp(String(e)).test(String(a)),
  hasSize: (a, e) => {
    const len = Array.isArray(a) ? a.length : String(a).length;
    return len === Number(e);
  },
  startsWith: (a, e) => String(a).startsWith(String(e)),
  endsWith: (a, e) => String(a).endsWith(String(e)),
  greaterThan: (a, e) => Number(a) > Number(e),
  greaterThanOrEqual: (a, e) => Number(a) >= Number(e),
  lessThan: (a, e) => Number(a) < Number(e),
  'not-null': (a) => a !== null && a !== undefined,
  'not-empty': (a) => a !== '' && (!Array.isArray(a) || a.length > 0),
  empty: (a) => a === '' || (Array.isArray(a) && a.length === 0),
};
```

Definition uses `absorbSiblings: true`.

- [ ] **Step 4: Write assertThat tests for each matcher**

- [ ] **Step 5: Implement verify-simulation plugin**

Looks up `SimulationRuntime` from `ServiceRegistry`, gets `activeJournal()`, counts entries for the specified action name.

- [ ] **Step 6: Write verify-simulation tests**

- [ ] **Step 7: Implement run-playbook and validate-playbook plugins**

`run-playbook` looks up `Catalog` and `StructuralEvaluator` from ServiceRegistry, reads the playbook file (via a `FileReader` service), resolves and executes.

`validate-playbook` does the same but only resolves (no execution), returning metadata.

- [ ] **Step 8: Write tests for run-playbook and validate-playbook**

- [ ] **Step 9: Implement TestCatalog**

```typescript
// test/plugins/test-catalog.ts
import type { Catalog, CatalogEntry } from '../../step/walker.js';
import { createAssertPlugin } from './assert-plugin.js';
import { createAssertThatPlugin } from './assert-that-plugin.js';
import { createVerifySimulationPlugin } from './verify-simulation-plugin.js';
import { createRunPlaybookPlugin } from './run-playbook-plugin.js';
import { createValidatePlaybookPlugin } from './validate-playbook-plugin.js';

export class TestCatalog implements Catalog {
  private readonly entries: Map<string, CatalogEntry>;

  constructor() {
    this.entries = new Map([
      ['assert', createAssertPlugin()],
      ['assertThat', createAssertThatPlugin()],
      ['verify-simulation', createVerifySimulationPlugin()],
      ['run-playbook', createRunPlaybookPlugin()],
      ['validate-playbook', createValidatePlaybookPlugin()],
    ]);
  }

  resolve(actionName: string): CatalogEntry | undefined {
    return this.entries.get(actionName);
  }

  availableActions(): Set<string> {
    return new Set(this.entries.keys());
  }
}
```

- [ ] **Step 10: Run all tests, commit**

```bash
git add packages/yaml-core/src/test/plugins/
git commit -m "feat(#621): test plugins — assert, assertThat (12 matchers), verify-simulation, run-playbook, validate-playbook"
```

---

### Task 9: YamlTestExecutor + Vitest Adapter

**Files:**
- Create: `test/yaml-test-executor.ts`
- Create: `test/yaml-test-executor.test.ts`
- Create: `test/vitest-adapter.ts`
- Create: `test/vitest-adapter.test.ts`
- Modify: `test/index.ts` (final exports)

**Interfaces:**
- Consumes: `TestFile`, `TestCase` from Task 7
- Consumes: `SimulateBlockParser` from Task 7
- Consumes: `TestCatalog` from Task 8
- Consumes: `Walker`, `Catalog`, `ServiceRegistry`, `StructuralEvaluator`
- Consumes: `SimulationRuntime` from `simulation-runtime.ts`
- Consumes: `VariableResolver` from `variable-resolver.ts`
- Produces: `TestResult`, `TestSuiteResult` types
- Produces: `YamlTestExecutor.run(testFile: TestFile): Promise<TestSuiteResult>`
- Produces: `describeYamlTests(dir, catalog, services): void`

- [ ] **Step 1: Write failing test for YamlTestExecutor — basic passing test**

```typescript
// test/yaml-test-executor.test.ts
it('executes a simple passing test case', async () => {
  const testFile: TestFile = {
    name: 'simple test',
    tests: [{ name: 'passes', steps: [{ assert: 'true' }] }],
  };

  const executor = new YamlTestExecutor(testCatalog, services);
  const result = await executor.run(testFile);

  expect(result.results).toHaveLength(1);
  expect(result.results[0]!.status).toBe('pass');
});
```

- [ ] **Step 2: Implement YamlTestExecutor**

The executor orchestrates the full lifecycle:
1. Create `CompositeCatalog` from TestCatalog + application catalog
2. For each test case:
   - Parse simulate → push SimulationOverlay
   - Build VariableResolver from vars/params
   - Resolve steps via Walker.resolve()
   - Run setup → test → teardown via StructuralEvaluator
   - Pop SimulationOverlay
   - Produce TestResult

Register `SimulationRuntime`, `Catalog`, and `StructuralEvaluator` in the test ServiceRegistry.

- [ ] **Step 3: Run test to verify pass**

- [ ] **Step 4: Write tests for failure case, setup/teardown, simulation, test isolation**

- [ ] **Step 5: Implement Vitest adapter**

```typescript
// test/vitest-adapter.ts
import { readFileSync } from 'node:fs';
import { parse as parseYaml } from 'yaml';
import { YamlTestDiscovery } from './yaml-test-discovery.js';
import { YamlTestParser } from './yaml-test-parser.js';
import { YamlTestExecutor } from './yaml-test-executor.js';
import type { Catalog, ServiceRegistry } from '../step/walker.js';

export function describeYamlTests(
  dir: string,
  catalog: Catalog,
  services: ServiceRegistry,
): void {
  const files = YamlTestDiscovery.scan(dir);
  const executor = new YamlTestExecutor(catalog, services);

  for (const filePath of files) {
    const raw = parseYaml(readFileSync(filePath, 'utf-8')) as Record<string, unknown>;
    const testFile = YamlTestParser.parse(raw);

    describe(testFile.name, () => {
      for (const testCase of testFile.tests) {
        it(testCase.name, async () => {
          const singleFile = { ...testFile, tests: [testCase] };
          const result = await executor.run(singleFile);
          const testResult = result.results[0]!;
          if (testResult.status !== 'pass') {
            throw new Error(testResult.message ?? `Test '${testCase.name}' failed`);
          }
        });
      }
    });
  }
}
```

- [ ] **Step 6: Write Vitest adapter integration test with a real .test.yaml fixture**

Create a test fixture at `test/fixtures/example.test.yaml` and verify the adapter generates the correct test structure.

- [ ] **Step 7: Update barrel export, run all yaml-core tests**

Run: `yarn workspace yaml-core vitest run`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/test/
git commit -m "feat(#621): YamlTestExecutor + Vitest adapter — .test.yaml discovery, execution, and Vitest integration"
```

---

## References

- [2026-11-08-runtime-parity-phase-2-design.md] — design spec this plan implements
- [step/structural-evaluator.ts:170-191] — evaluateTryCatchFinally (H3 target)
- [step/structural-evaluator.ts:39-53] — dispatch switch (M4a target)
- [step/walker.ts:266-271] — Walker.resolve entry point (H1 target)
- [step/walker.ts:67-81] — ResolvedStep union with annotations field
- [step/decorator-chain.ts:594-678] — DecoratorChain.build (unchanged)
- [variable-resolver.ts:49-57] — withScope/withObjectScope (H3 error scope)
- [step/simulation-runtime.ts:24-29] — SimulationConfig type (M6 target)
- [orchestration/consumer-handle.ts:5-6] — ConsumerHandle.pause/resume (M4a target)
- [orchestration/types.ts:69] — ScenarioScope.consumer() (M4a target)
- [step/types.ts:112-120] — Definition with absorbSiblings (H4 assertThat)
- [GitHub #621] — focal issue
- [GitHub #608] — parent audit issue
