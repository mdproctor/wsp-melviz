# Event Sequence Pattern Parity — TS Runtime + Playbooks

**Epic:** #546
**Date:** 2026-10-14
**Branch:** issue-546-event-sequence-parity

## Summary

Reach TS runtime parity with the platform's event sequence pattern features
(platform #600) and parallel step extensions (platform #392). Add
corresponding playbook YAML examples and scenario page updates.

All changes land in `packages/yaml-core/src/` — walker, structural
evaluator, decorator chain, orchestration primitives, and schema composer.
Playbooks land in `playbooks/`. No changes to packages/pages-runtime or
packages/pages-aria.

## Scope

**In scope (6 child issues):**
- #538 — `from:` basic decorator (channel read, filter, fan-in)
- #537 — `loop: continuous`, `cancel:` decorator, `background:` decorator,
  `forEach: { collect: all }`, conditional retry, primitive variables
- #542 — `from:` hardening (until, channel close, named consumers)
- #543 — `race:` structural step
- #544 — Parallel extensions (advanceAfter, cancelRemaining, collectionTimeout,
  polymorphic parsing)
- #545 — Event sequence composition playbooks (4 temporal patterns + scenario
  page updates)

**Out of scope:**
- DES scheduler changes (scheduler in pages-aria, not yaml-core)
- Platform-side YAML examples (platform #609)
- Simulation data injection

## Architecture

### Extension Points

Each feature maps to a specific extension point in the existing architecture:

| Feature | Layer | Extension point |
|---------|-------|----------------|
| `from:` | DecoratorChain | New `FromDecorator` class |
| `cancel:` | DecoratorChain | New `CancelDecorator` class |
| `background:` | DecoratorChain | New `BackgroundDecorator` class |
| `loop: continuous` | Directives | New `LoopDirective` variant |
| `forEach: { collect: all }` | DecoratorChain | Extend `ForEachDecorator` |
| Conditional retry | DecoratorChain | Extend `RetryDecorator` |
| Primitive variables | VariableResolver | Extend variable resolution |
| `race:` | Walker + StructuralEvaluator | New `RaceStep` type + `evaluateRace()` |
| Parallel extensions | Walker + StructuralEvaluator | Extend `ParallelStep` + `evaluateParallel()` |
| `from:` hardening | DecoratorChain + ScenarioScope | Extend `FromDecorator` + `ConsumerHandle` |
| Playbooks | playbooks/ | New YAML files + manifest entries |

### DecoratorChain Ordering

Match platform ordering exactly. New decorators insert at specific positions:

```
if (when guard)
  → forEach
    → background       ← NEW
      → cancel          ← NEW
        → loop
          → on-error
            → from      ← NEW
              → timeout
                → at
                  → wait
                    → retry
                      → resource/semaphore/mutex
                        → delay
                          → signal/publish/transition/transform
                            → on-success/on-failure
                              → CoreExecution
```

The `DecoratorChain.build()` method wraps from inside out, so the build
order is reversed: on-success/on-failure first, `if` last.

### AbortSignal Threading

`Context` gains an optional `signal?: AbortSignal`:

```typescript
export interface Context {
  readonly params: Record<string, unknown>;
  readonly services: ServiceRegistry;
  readonly scope: ScenarioScope;
  readonly stepName: string;
  readonly resolver: VariableResolver;
  readonly signal?: AbortSignal;   // cooperative cancellation
}
```

Decorators that perform async waits check `signal?.aborted` before blocking:
- `WaitDecorator` — before `signal.await()`
- `DelayDecorator` — before `sleep()`
- `SemaphoreDecorator` — before `sem.acquire()`
- `TimeoutDecorator` — before `Promise.race()`
- `FromDecorator` — before `channel.receive()`
- `LoopDecorator` — at the top of each iteration

When `signal.aborted` is true, the decorator returns
`stepFailure('Cancelled')` immediately.

## Feature Designs

### 1. from: Decorator (#538)

Three YAML forms:

```yaml
# Simple — read from named channel
- myAction: { ... }
  from: sensor-readings

# Full — with alias and filter
- myAction: { ... }
  from: { channel: sensor-readings, as: reading, filter: "${from.value > 50}" }

# Fan-in — merge from multiple channels
- myAction: { ... }
  from: { channels: [sensors, alerts], as: event }
```

**FromDecorator implementation:**

```typescript
class FromDecorator implements DecoratedExecution {
  constructor(
    private readonly next: DecoratedExecution,
    private readonly raw: unknown,
  ) {}

  async execute(context: Context): Promise<Result> {
    if (context.signal?.aborted) return stepFailure('Cancelled');

    const spec = this.parseSpec();
    const channel = this.resolveChannel(spec, context);
    const item = await channel.receive();

    if (item === undefined) {
      // Channel closed — null sentinel
      return stepSuccess({ '__from.closed': true });
    }

    if (spec.filter) {
      const resolved = context.resolver.resolveString(spec.filter, 'from-filter');
      if (!isTruthy(resolved)) {
        return stepSuccess({}); // filtered out — skip
      }
    }

    const alias = spec.as ?? 'from';
    const scoped = pushFromContext(context.resolver, alias, item);
    return this.next.execute(withResolver(context, scoped));
  }
}
```

For fan-in (`channels: [a, b]`), the decorator uses `Promise.race` across
all channel receives — first channel with data wins.

Variable injection uses the same `ObjectVariableSource` pattern as
`ForEachDecorator`:

```typescript
function pushFromContext(
  resolver: VariableResolver, alias: string, item: unknown,
): VariableResolver {
  const source: ObjectVariableSource = {
    resolve(name: string): unknown {
      if (name === alias) return item;
      if (typeof item === 'object' && item !== null) {
        const prefix = alias + '.';
        if (name.startsWith(prefix))
          return (item as Record<string, unknown>)[name.substring(prefix.length)];
      }
      return undefined;
    },
    allowContainerReturn() { return true; },
  };
  return resolver.withObjectScope(alias, source);
}
```

**Walker changes:** Add `'from'` to `RESERVED_KEYS` and `DECORATOR_KEYS`.

**Schema changes:** Add `from:` to decorator properties in SchemaComposer
with oneOf: string | object (channel + as + filter) | object (channels + as).

### 2. loop: continuous, cancel:, background: (#537)

#### loop: continuous

New variant in the LoopDirective union:

```typescript
type LoopDirective =
  | { type: 'count'; count: number }
  | { type: 'count-until'; count: number; until: string }
  | { type: 'until'; until: string }
  | { type: 'continuous' };            // NEW
```

`parseLoopDirective` recognizes the string `'continuous'`:

```typescript
if (raw === 'continuous') return { type: 'continuous' };
```

`LoopDecorator` handles continuous as infinite iteration with AbortSignal
check and safety cap:

```typescript
const maxIterations = directive.type === 'continuous' ? 100_000 : directive.count;
for (let i = 0; i < maxIterations; i++) {
  if (context.signal?.aborted) return stepFailure('Cancelled');
  // ... existing loop body
}
```

#### cancel: decorator

Position in chain: between `background:` and `loop:`.

```typescript
class CancelDecorator implements DecoratedExecution {
  constructor(
    private readonly next: DecoratedExecution,
    private readonly signalName: string,
  ) {}

  async execute(context: Context): Promise<Result> {
    const controller = new AbortController();
    const signal = context.scope.signal(this.signalName);

    const onSignal = () => controller.abort();
    // If already signalled, abort immediately
    if (signal.isSignalled()) {
      return stepFailure('Cancelled');
    }
    signal.await().then(onSignal);

    const merged = mergeAbortSignals(context.signal, controller.signal);
    try {
      return await this.next.execute({ ...context, signal: merged });
    } finally {
      // Cleanup: prevent the signal listener from firing after completion
      controller.abort(); // no-op if already aborted
    }
  }
}
```

`mergeAbortSignals` creates a new AbortController that aborts when either
input signal aborts:

```typescript
function mergeAbortSignals(
  parent: AbortSignal | undefined,
  child: AbortSignal,
): AbortSignal {
  if (!parent) return child;
  const merged = new AbortController();
  parent.addEventListener('abort', () => merged.abort(), { once: true });
  child.addEventListener('abort', () => merged.abort(), { once: true });
  return merged.signal;
}
```

**Walker changes:** Add `'cancel'` to `RESERVED_KEYS` and `DECORATOR_KEYS`.

#### background: decorator

Position in chain: between `forEach:` and `cancel:`.

```typescript
class BackgroundDecorator implements DecoratedExecution {
  constructor(
    private readonly next: DecoratedExecution,
  ) {}

  async execute(context: Context): Promise<Result> {
    const handle = context.scope.spawn(async (signal) => {
      await this.next.execute({ ...context, signal });
    });
    return stepSuccess({ '__background.handle': handle.id });
  }
}
```

The spawned task receives an AbortSignal from `scope.spawn()`. On
`scope.close()`, all spawned tasks are cancelled.

**Walker changes:** Add `'background'` to `RESERVED_KEYS` and
`DECORATOR_KEYS`.

#### forEach: { collect: all }

Extend `ForEachDecorator` to check `spec['collect'] === 'all'`:

```typescript
if (spec['collect'] === 'all') {
  const results: unknown[] = [];
  for (const item of items) {
    const r = await this.next.execute(withResolver(ctx, scoped));
    if (r.kind === 'success') results.push(r.output);
  }
  return stepSuccess({ '__forEach.collected': results });
}
```

#### Conditional retry

Extend `RetryDirective` with optional failure category filter:

```typescript
type RetryDirective =
  | { type: 'simple'; max: number }
  | { type: 'full'; max: number; backoff: string; delayMs: number }
  | { type: 'conditional'; max: number; on?: string[]; except?: string[];
      backoff?: string; delayMs?: number };
```

`RetryDecorator` checks the failure category before retrying:

```typescript
if (directive.type === 'conditional' && lastResult.kind === 'failure') {
  const category = lastResult.executionMetadata?.failureCategory as string;
  if (directive.on && !directive.on.includes(category)) break;
  if (directive.except && directive.except.includes(category)) break;
}
```

#### Primitive variable resolution

Extend `VariableResolver` to resolve `flag.*`, `counter.*`, `gauge.*`,
`signal.*`, `accumulator.*` prefixes against the ScenarioScope:

```typescript
// In VariableResolver or a new PrimitiveVariableSource
resolve(name: string): unknown {
  if (name.startsWith('flag.')) return scope.flag(name.substring(5)).isSet();
  if (name.startsWith('counter.')) return scope.counter(name.substring(8)).longValue();
  if (name.startsWith('gauge.')) return scope.gauge(name.substring(6)).doubleValue();
  if (name.startsWith('signal.')) return scope.signal(name.substring(7)).isSignalled();
  if (name.startsWith('accumulator.')) return scope.accumulator(name.substring(12)).doubleValue();
  return undefined;
}
```

### 3. from: Hardening (#542)

#### from: until

Extend the from: spec with an `until` field:

```yaml
- myAction: { ... }
  from: { channel: events, until: "${flag.shutdown}" }
```

The `FromDecorator` evaluates the until condition before each receive:

```typescript
if (spec.until) {
  const resolved = context.resolver.resolveString(spec.until, 'from-until');
  if (isTruthy(resolved)) {
    return stepSuccess({ '__from.terminated': true });
  }
}
const item = await channel.receive();
```

#### Channel close detection

`FromDecorator` already handles `undefined` from `channel.receive()` as
the null sentinel (stream-ended). For error-close, `channel.receive()`
throws `ChannelClosedError` — the decorator catches it and returns failure:

```typescript
try {
  const item = await channel.receive();
  if (item === undefined) return stepSuccess({ '__from.closed': true });
  // ... process item
} catch (e) {
  if (e instanceof ChannelClosedError) {
    return stepFailure(`Channel '${spec.channel}' closed with error: ${e.message}`);
  }
  throw e;
}
```

#### Named consumers

New `ConsumerHandle` primitive in `orchestration/`:

```typescript
interface ConsumerHandle {
  readonly name: string;
  stop(): void;
  start(): void;
  pause(): void;
  resume(): Promise<void>;
  isStopped(): boolean;
  isPaused(): boolean;
}
```

`ScenarioScope` gains `consumer(name)`:

```typescript
consumer(name: string): ConsumerHandle;
```

`FromDecorator` registers a consumer when `spec.name` is present:

```typescript
if (spec.name) {
  const consumer = context.scope.consumer(spec.name);
  // consumer.stop() aborts the from: loop via AbortController
  // consumer.pause() suspends channel reads via promise gate
}
```

**New catalog actions:** `stop`, `start`, `pause`, `resume` — each takes
a `name` param and calls the corresponding method on the consumer handle
retrieved from scope.

### 4. race: Structural Step (#543)

New step type in the `ResolvedStep` union:

```typescript
export interface RaceStep {
  kind: 'race';
  name: string | null;
  steps: ResolvedStep[];
  decorators: Record<string, unknown>;
}
```

**Walker changes:** Add `race:` structural handling in `resolveOne()`,
analogous to `block`/`parallel`:

```typescript
if (structuralType === 'race') {
  const substeps = Walker.resolveAtDepth(
    structuralValue as Record<string, unknown>[],
    catalog, depth + 1, `${stepPath} → race`, seenNames,
  );
  return { kind: 'race', name: stepName, steps: substeps, decorators };
}
```

Remove `'race'` from `DECORATOR_KEYS` (it's structural, not a decorator).

**StructuralEvaluator changes:** New `evaluateRace()`:

```typescript
private async evaluateRace(step: RaceStep, context: Context): Promise<Result> {
  const controllers = step.steps.map(() => new AbortController());
  const promises = step.steps.map((child, i) => {
    const branchCtx: Context = {
      ...context,
      signal: mergeAbortSignals(context.signal, controllers[i]!.signal),
    };
    return this.evaluate(child, branchCtx).then(result => ({ index: i, result }));
  });

  const winner = await Promise.race(promises);

  // Cancel all other branches
  for (let i = 0; i < controllers.length; i++) {
    if (i !== winner.index) controllers[i]!.abort();
  }

  return winner.result;
}
```

**Schema changes:** Add `race:` as a structural variant in SchemaComposer.

**Playbook:** `step-workflows/race.playbook.yaml` — two branches with
different delays, first to complete wins.

### 5. Parallel Extensions (#544)

#### Extended ParallelStep

```typescript
export interface ParallelStep {
  kind: 'parallel';
  name: string | null;
  branches: NamedBranch[];         // changed from steps: ResolvedStep[]
  advanceAfter?: number;           // NEW — discriminator threshold
  cancelRemaining?: boolean;       // NEW — cancel after advancement
  collectionTimeout?: number;      // NEW — timeout for remaining (ms)
  decorators: Record<string, unknown>;
}

export interface NamedBranch {
  name: string | null;
  steps: ResolvedStep[];
}
```

#### Polymorphic parsing

Walker handles both forms:

```yaml
# List form — unnamed branches (current)
parallel:
  - step-a: { ... }
  - step-b: { ... }

# Map form — named branches (new)
parallel:
  branch-a:
    - step-a1: { ... }
  branch-b:
    - step-b1: { ... }
```

```typescript
if (structuralType === 'parallel') {
  const raw = structuralValue;
  let branches: NamedBranch[];

  if (Array.isArray(raw)) {
    // List form: each element is a single unnamed branch step
    const substeps = Walker.resolveAtDepth(raw, catalog, depth + 1, ...);
    branches = substeps.map((s, i) => ({ name: s.name ?? null, steps: [s] }));
  } else if (typeof raw === 'object' && raw !== null) {
    // Map form: keys are branch names, values are step arrays
    branches = Object.entries(raw).map(([name, steps]) => ({
      name,
      steps: Walker.resolveAtDepth(steps as Record<string, unknown>[], catalog, depth + 1, ...),
    }));
  }

  const advanceAfter = step['advance-after'] as number | undefined;
  const cancelRemaining = step['cancel-remaining'] as boolean | undefined;
  const collectionTimeoutRaw = step['collection-timeout'] as string | undefined;
  const collectionTimeout = collectionTimeoutRaw
    ? parseDurationMs(collectionTimeoutRaw) : undefined;

  return {
    kind: 'parallel', name: stepName, branches,
    ...(advanceAfter !== undefined ? { advanceAfter } : {}),
    ...(cancelRemaining !== undefined ? { cancelRemaining } : {}),
    ...(collectionTimeout !== undefined ? { collectionTimeout } : {}),
    decorators,
  };
}
```

Add `'advance-after'`, `'cancel-remaining'`, `'collection-timeout'` to
`RESERVED_KEYS`.

#### ParallelCompletionTracker

Two-latch pattern for policy-aware evaluation:

```typescript
class ParallelCompletionTracker {
  private readonly advancementLatch: OrcLatch;
  private readonly completionLatch: OrcLatch;
  private readonly controllers: AbortController[];

  constructor(
    branchCount: number,
    advanceAfter: number,
    cancelRemaining: boolean,
    scope: ScenarioScope,
  ) {
    this.advancementLatch = scope.latch('__parallel_advance', advanceAfter);
    this.completionLatch = scope.latch('__parallel_complete', branchCount);
    this.controllers = Array.from({ length: branchCount }, () => new AbortController());
  }
}
```

#### Policy-aware evaluateParallel

```typescript
private async evaluateParallel(step: ParallelStep, context: Context): Promise<Result> {
  if (!step.advanceAfter) {
    // Barrier mode (current behavior) — wait for all
    return this.evaluateParallelBarrier(step, context);
  }
  // Discriminator mode
  return this.evaluateParallelDiscriminator(step, context);
}
```

**Barrier mode:** Same as current `Promise.allSettled` but adapted for
`NamedBranch[]`.

**Discriminator mode:** Launch all branches, await advancement latch.
If `cancelRemaining`, abort remaining branches. If `collectionTimeout`,
race the completion latch against a timeout. Collect partial results.

### 6. Composition Playbooks (#545)

Four new playbooks in `playbooks/composition/`:

| File | Title | Pattern |
|------|-------|---------|
| `timed-absence.playbook.yaml` | Timed Absence | A then !B[window] — block with timeout on from: |
| `delayed-sequence.playbook.yaml` | Delayed Sequence | Delay between sequential from: reads |
| `heartbeat-monitor.playbook.yaml` | Heartbeat Monitor | loop: continuous + delay + from: with timeout |
| `recursive-pattern.playbook.yaml` | Recursive Pattern | ((A then B) or (A then C))* via nested block/race/loop |

All marked `runnable: false` initially (require runtime features from
#538, #537, #542, #543 to execute).

**Manifest updates:** Add all 4 to `playbooks/manifest.json` under the
Composition category.

**Scenario page updates:** Add temporal patterns to the Act 4 (Composition)
dropdown picker in the scenario examples.

## Implementation Order

Follows the dependency graph from the epic:

```
#538 (from: basic) ──→ #542 (from: hardening) ──→ #545 (playbooks)
#537 (loop/cancel/bg) ─────────────────────────→ #545
                        #543 (race:) ──────────→ #545
#544 (parallel ext) — independent
```

Batch 1 (foundation, can be parallel):
- #538 — from: decorator (walker + decorator chain + schema)
- #537 — loop continuous, cancel, background (directives + decorator chain)
- #544 — parallel extensions (walker + evaluator + schema)

Batch 2 (depends on batch 1):
- #542 — from: hardening (extends #538's FromDecorator)
- #543 — race: structural step (walker + evaluator + schema)

Batch 3 (depends on all above):
- #545 — composition playbooks + scenario page updates

## Testing Strategy

Each feature gets tests alongside its implementation file:

| Feature | Test file | Key cases |
|---------|-----------|-----------|
| FromDecorator | decorator-chain.test.ts | Simple read, filter, fan-in, closed channel, aborted signal |
| CancelDecorator | decorator-chain.test.ts | Signal before start, signal during execution, normal completion cleanup |
| BackgroundDecorator | decorator-chain.test.ts | Spawn-and-continue, scope close cancels background |
| loop: continuous | directives.test.ts | Parse 'continuous', safety cap, AbortSignal termination |
| forEach collect | decorator-chain.test.ts | Collect all results into array |
| Conditional retry | decorator-chain.test.ts | Retry on matching category, skip on excluded |
| Primitive variables | variable-resolver.test.ts | flag.*, counter.*, gauge.*, signal.*, accumulator.* |
| RaceStep | structural-evaluator.test.ts | First wins, losers cancelled, nested race |
| ParallelStep extensions | structural-evaluator.test.ts | Discriminator, race-as-policy, barrier (regression), map form |
| ParallelCompletionTracker | parallel-completion-tracker.test.ts | Advance latch, cancel remaining, collection timeout |
| ConsumerHandle | consumer-handle.test.ts | Stop, start, pause, resume lifecycle |
| Playbooks | Validated by schema (non-runnable) | YAML parses without error |

## File Changes Summary

### New files
- `packages/yaml-core/src/step/from-decorator.ts` — FromDecorator
- `packages/yaml-core/src/step/cancel-decorator.ts` — CancelDecorator
- `packages/yaml-core/src/step/background-decorator.ts` — BackgroundDecorator
- `packages/yaml-core/src/step/abort-utils.ts` — mergeAbortSignals utility
- `packages/yaml-core/src/step/parallel-completion-tracker.ts` — two-latch tracker
- `packages/yaml-core/src/orchestration/consumer-handle.ts` — ConsumerHandle
- `playbooks/composition/timed-absence.playbook.yaml`
- `playbooks/composition/delayed-sequence.playbook.yaml`
- `playbooks/composition/heartbeat-monitor.playbook.yaml`
- `playbooks/composition/recursive-pattern.playbook.yaml`
- `playbooks/step-workflows/race.playbook.yaml`

### Modified files
- `packages/yaml-core/src/step/walker.ts` — RaceStep type, parallel map form, new reserved/decorator keys
- `packages/yaml-core/src/step/structural-evaluator.ts` — evaluateRace(), policy-aware evaluateParallel()
- `packages/yaml-core/src/step/decorator-chain.ts` — new decorator insertion points, AbortSignal checks on existing decorators
- `packages/yaml-core/src/orchestration/directives.ts` — continuous LoopDirective, conditional RetryDirective
- `packages/yaml-core/src/orchestration/types.ts` — ConsumerHandle, consumer() on ScenarioScope
- `packages/yaml-core/src/step/schema-composer.ts` — race:, from:, cancel:, background:, parallel policy schemas
- `playbooks/manifest.json` — 5 new entries (4 composition + 1 race)

## References

- [packages/yaml-core/src/step/walker.ts] — current walker with ResolvedStep union
- [packages/yaml-core/src/step/structural-evaluator.ts] — current evaluator dispatch
- [packages/yaml-core/src/step/decorator-chain.ts] — current decorator chain and ordering
- [packages/yaml-core/src/orchestration/directives.ts] — LoopDirective, RetryDirective
- [packages/yaml-core/src/orchestration/types.ts] — ScenarioScope, OrcChannel
- [packages/yaml-core/src/orchestration/channel.ts] — DefaultOrcChannel
- [specs/issue-462-ts-orchestration-primitives/] — orchestration primitives design
- [specs/issue-461-463-scenario-orchestrator/] — DES scheduler design
- [playbooks/manifest.json] — playbook manifest structure
- [platform #600] — epic: Event sequence patterns
- [platform #392] — discriminator coordination pattern
- [platform #563] — loop continuous, cancel, background
- [platform #564] — from: decorator
- [platform #601] — from: hardening (until, channel close)
- [platform #602] — named consumers
- [platform #603] — race: structural step
- [decisions.md] — D1-D8 design decisions
