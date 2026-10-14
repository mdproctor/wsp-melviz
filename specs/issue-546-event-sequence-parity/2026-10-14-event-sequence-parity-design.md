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

**Cross-cutting concern:** The AbortSignal-based cancellation system is not
a simple additive extension — it threads through existing async decorators
(WaitDecorator, DelayDecorator, SemaphoreDecorator, TimeoutDecorator) and
requires AbortSignal-aware overloads on orchestration primitives
(OrcSemaphore.acquire, OrcChannel.receive, OrcSignal.await, OrcLatch.await).
This is acknowledged as the main risk area. See "Cancellation Architecture"
section for full scope.

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

### Cancellation Architecture

AbortSignal support is not limited to Context — the orchestration
primitives themselves need AbortSignal-aware overloads to achieve
cooperative cancellation of blocked waits:

```typescript
// OrcSemaphore — new overload
acquire(signal?: AbortSignal): Promise<void>;

// OrcChannel — new overload
receive(signal?: AbortSignal): Promise<T | undefined>;

// OrcSignal — new overload
await(signal?: AbortSignal): Promise<void>;

// OrcLatch — new overload
await(signal?: AbortSignal): Promise<void>;
```

When the AbortSignal fires, the pending promise rejects with an
`AbortError` (standard DOMException name). The decorator catches
`AbortError` and converts it to `stepFailure('Cancelled')`.

**SpawnedTask cancellation:** `DefaultSpawnedTask` gains an
AbortController. The task function receives the controller's signal.
`scope.close()` calls `controller.abort()` before `joinWithTimeout()`.

```typescript
class DefaultSpawnedTask {
  private readonly controller = new AbortController();

  constructor(taskName: string, task: (signal: AbortSignal) => Promise<void>) {
    this.promise = task(this.controller.signal)
      .then(() => { ... })
      .catch((err: Error) => { ... });
  }

  cancel(): void { this.controller.abort(); }
}
```

**Cancelled branch result semantics:** When a branch is cancelled (race
loser, cancel: signal fired), the result is `stepFailure('Cancelled')`
with `failureCategory: 'CANCELLED'`. The on-error and on-failure
decorators receive this failure normally. YAML authors who want to
distinguish cancellation from real failures can use conditional retry
with `except: CANCELLED` (from D5's conditional retry). The result
store records the step as failed with category CANCELLED — auditable
but clearly distinguished from operational failures.

**Scope lifecycle alternative (considered and rejected):**
`childScope().close()` was considered as a cancellation mechanism
(close losing scopes instead of aborting). This handles channel.receive
and signal.await (which close/signal on scope.close) but does NOT
unblock semaphore.acquire or latch.await. Since parallel extensions
use semaphores for coordination, scope-close is insufficient.
AbortSignal provides uniform cancellation across all primitive types.

### race: vs parallel: policy relationship

`race:` is a structural step type, distinct from `parallel:` with
`advanceAfter: 1, cancelRemaining: true`. The distinction:

- **`race:`** — first-to-complete over arbitrary step sequences
  (blocks, nested structures). Branches are always unnamed. No
  result collection from losing branches.
- **`parallel: { advanceAfter: 1, cancelRemaining: true }`** —
  race-as-policy on named branches. Supports partial result
  collection before cancellation. Named branches allow the
  discriminator to report which branch won.

`race:` is syntactic sugar for the common case. Remove from
`DECORATOR_KEYS` (it's structural only), keep in `RESERVED_KEYS`.

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
    let item: unknown;

    try {
      item = await this.receiveItem(spec, context);
    } catch (e) {
      if (e instanceof ChannelClosedError) {
        return stepSuccess({ '__from.closed': true });
      }
      throw e;
    }

    // Inject received data BEFORE filter evaluation
    const alias = spec.as ?? 'from';
    const scoped = pushFromContext(context.resolver, alias, item);
    const scopedCtx = withResolver(context, scoped);

    if (spec.filter) {
      const resolved = scoped.resolveString(spec.filter, 'from-filter');
      if (!isTruthy(resolved)) {
        return stepSuccess({}); // filtered out — skip
      }
    }

    return this.next.execute(scopedCtx);
  }

  private async receiveItem(
    spec: FromSpec, context: Context,
  ): Promise<unknown> {
    if (spec.channels) {
      // Fan-in: race with cancellation of losers
      return this.receiveFanIn(spec.channels, context);
    }
    return context.scope
      .channel<unknown>(spec.channel)
      .receive(context.signal);  // pass AbortSignal
  }

  private async receiveFanIn(
    channelNames: string[], context: Context,
  ): Promise<unknown> {
    // Use per-channel AbortControllers to cancel losers
    const controllers = channelNames.map(() => new AbortController());
    const promises = channelNames.map((name, i) => {
      const merged = mergeAbortSignals(context.signal, controllers[i]!.signal);
      return context.scope.channel<unknown>(name)
        .receive(merged)
        .then(item => ({ index: i, item }));
    });

    const winner = await Promise.race(promises);
    // Cancel losing channel receives
    for (let i = 0; i < controllers.length; i++) {
      if (i !== winner.index) controllers[i]!.abort();
    }
    return winner.item;
  }
}
```

**Filter expressions:** Filters use truthiness checks via `isTruthy()`,
not comparison operators. `${from.value}` resolves to the value; `isTruthy`
checks if it's non-empty/non-zero. For comparison-based filtering, use
the ConditionEvaluator (which wraps `isTruthy` + a pluggable delegate).

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
check:

```typescript
if (directive.type === 'continuous') {
  while (true) {
    if (context.signal?.aborted) return stepFailure('Cancelled');
    lastResult = await this.next.execute(context);
    if (lastResult.kind === 'failure') return lastResult;
  }
}
```

**Parse-time validation:** Walker rejects `loop: continuous` unless it
has a termination guarantee — either on the same step or from a parent
scope:

- Same-step: `cancel:` or `timeout:` decorator present
- Same-step: `background: true` (scope-bounded lifetime)
- Parent scope: Walker tracks whether the current resolution depth is
  inside a block/race/parallel with a timeout decorator (passed as a
  `hasParentTimeout` flag through `resolveAtDepth`)

```typescript
// In Walker.resolveOne(), after resolving decorators:
if (decorators['loop'] === 'continuous'
    && !decorators['cancel'] && !decorators['timeout']
    && !decorators['background'] && !hasParentTimeout) {
  throw new Error(`${stepPath}: loop: continuous requires cancel:, timeout:, or background:`);
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
    const orcSignal = context.scope.signal(this.signalName);

    // If already signalled, abort immediately
    if (orcSignal.isSignalled()) {
      return stepFailure('Cancelled');
    }
    orcSignal.await().then(() => controller.abort());

    const merged = mergeAbortSignals(context.signal, controller.signal);
    try {
      return await this.next.execute({ ...context, signal: merged });
    } finally {
      controller.abort(); // cleanup: prevent signal listener from firing after completion
    }
  }
}
```

**Signal/cancel interaction (deliberate):** `cancel:` uses OrcSignal
intentionally — the same signal can both cancel one step's inner
execution and unblock another step's `wait:`. This matches platform
semantics where a "shutdown" signal might cancel a consumer loop AND
trigger a cleanup step. YAML authors who want cancellation-only events
should use a dedicated signal name (convention: `cancel-*` prefix).

`mergeAbortSignals` uses `AbortSignal.any()` (available in Node 20+,
all modern browsers) for correct lifecycle management:

```typescript
function mergeAbortSignals(
  parent: AbortSignal | undefined,
  child: AbortSignal,
): AbortSignal {
  if (!parent) return child;
  return AbortSignal.any([parent, child]);
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
    const handle = context.scope.spawn(
      context.stepName,
      async (signal: AbortSignal) => {
        await this.next.execute({ ...context, signal });
      },
    );
    return stepSuccess({ '__background.handle': handle.name() });
  }
}
```

**SpawnedTask interface changes:** `spawn()` signature changes from
`(name: string, task: () => Promise<void>)` to
`(name: string, task: (signal: AbortSignal) => Promise<void>)`.
The `SpawnedTask` interface gains `cancel(): void`. These are
breaking changes to the orchestration primitive API, scoped to this epic.

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

Extend `Result` failure variant with an optional `failureCategory`:

```typescript
| { kind: 'failure'; message: string; failureCategory?: string }
```

Standard categories: `TIMEOUT`, `TRANSIENT`, `RATE_LIMITED`, `PERMANENT`,
`CANCELLED`, `UNKNOWN`. Executors set the category when returning failure.
If unset, defaults to `UNKNOWN`.

**Existing code paths that need category annotations:**
- `TimeoutDecorator` → `TIMEOUT`
- `SemaphoreDecorator` deadline failure → `TIMEOUT`
- `WaitDecorator` deadline failure → `TIMEOUT`
- AbortSignal cancellation path → `CANCELLED`

`RetryDecorator` checks the failure category before retrying. CANCELLED
is non-retryable by default — cancellation is a deliberate control flow
decision, not a transient failure:

```typescript
// All retry modes skip CANCELLED by default
if (lastResult.kind === 'failure') {
  const category = lastResult.failureCategory ?? 'UNKNOWN';
  if (category === 'CANCELLED') break;
}
// Conditional retry adds further filtering
if (directive.type === 'conditional' && lastResult.kind === 'failure') {
  const category = lastResult.failureCategory ?? 'UNKNOWN';
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

**Simplified ConsumerHandle** — the browser runtime does not need the
full platform consumer lifecycle (pause/resume/start/stop). The same
behavior is achievable through `cancel:` + re-establishing a new `from:`.
The TS implementation provides only stop:

```typescript
interface ConsumerHandle {
  readonly name: string;
  stop(): void;
  isStopped(): boolean;
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
}
```

**Built-in action:** `stop` — registered as a built-in catalog entry.
Takes a `name` param and calls `stop()` on the consumer handle retrieved
from scope. `pause`/`resume`/`start` are deferred — if a YAML scenario
needs them, the author can use `cancel: signal` + a new `from:` block
to restart consumption.

This departs from platform parity on the pause/resume API surface.
The platform has distributed actors with long-running processes where
pause/resume matters. The browser runtime has ephemeral scopes where
stop + restart is equivalent and simpler.

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
    return this.evaluate(child, branchCtx)
      .then(result => ({ index: i, result }))
      .catch(err => ({
        index: i,
        result: stepFailure(err instanceof Error ? err.message : String(err)),
      }));
  });

  // First branch to settle (success or failure) wins
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
  steps: ResolvedStep[];           // preserved for backward compat (list form)
  branches?: NamedBranch[];        // NEW — populated for map form only
  advanceAfter?: number;           // NEW — discriminator threshold
  cancelRemaining?: boolean;       // NEW — cancel after advancement
  collectionTimeout?: number;      // NEW — timeout for remaining (ms)
  decorators: Record<string, unknown>;
}

export interface NamedBranch {
  name: string;
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
    // List form: preserve as steps (backward compat)
    const substeps = Walker.resolveAtDepth(raw, catalog, depth + 1, ...);
    steps = substeps;
    branches = undefined;
  } else if (typeof raw === 'object' && raw !== null) {
    // Map form: keys are branch names, values are step arrays
    const namedBranches = Object.entries(raw).map(([name, stepsRaw]) => ({
      name,
      steps: Walker.resolveAtDepth(stepsRaw as Record<string, unknown>[], catalog, depth + 1, ...),
    }));
    branches = namedBranches;
    steps = namedBranches.flatMap(b => b.steps); // flat view for backward compat
  }

  const advanceAfter = step['advance-after'] as number | undefined;
  const cancelRemaining = step['cancel-remaining'] as boolean | undefined;
  const collectionTimeoutRaw = step['collection-timeout'] as string | undefined;
  const collectionTimeout = collectionTimeoutRaw
    ? parseDurationMs(collectionTimeoutRaw) : undefined;

  return {
    kind: 'parallel', name: stepName, steps,
    ...(branches ? { branches } : {}),
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
    stepName: string,
    branchCount: number,
    advanceAfter: number,
    cancelRemaining: boolean,
    scope: ScenarioScope,
  ) {
    // Unique latch names per parallel step to support nesting
    this.advancementLatch = scope.latch(`__parallel_advance_${stepName}`, advanceAfter);
    this.completionLatch = scope.latch(`__parallel_complete_${stepName}`, branchCount);
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

### Playbooks from #542 (from: hardening)

Four new playbooks:

| Category | File | Title |
|----------|------|-------|
| flow-control | `flow-control/from-until.playbook.yaml` | From Until |
| flow-control | `flow-control/from-channel-close.playbook.yaml` | From Channel Close |
| flow-control | `flow-control/pause-resume.playbook.yaml` | Pause / Resume |
| concurrency-patterns | `concurrency-patterns/named-consumer.playbook.yaml` | Named Consumer |

### Playbooks from #545 (composition)

Four new playbooks in `playbooks/composition/`:

| File | Title | Pattern |
|------|-------|---------|
| `timed-absence.playbook.yaml` | Timed Absence | A then !B[window] — block with timeout on from: |
| `delayed-sequence.playbook.yaml` | Delayed Sequence | Delay between sequential from: reads |
| `heartbeat-monitor.playbook.yaml` | Heartbeat Monitor | loop: continuous + delay + from: with timeout |
| `recursive-pattern.playbook.yaml` | Recursive Pattern | ((A then B) or (A then C))* via nested block/race/loop |

### Playbook from #543 (race)

| Category | File | Title |
|----------|------|-------|
| step-workflows | `step-workflows/race.playbook.yaml` | Race |

### select.playbook.yaml update (#543)

Already updated in prior work (#541) — description says "For
first-to-complete over arbitrary concurrent blocks, use race: instead."
No further changes needed.

**Total: 9 new playbooks** (4 from #542, 4 from #545, 1 from #543).
All marked `runnable: false` initially (require runtime features to
execute).

**Manifest updates:** Add all 9 to `playbooks/manifest.json` under
their respective categories.

**Scenario page updates:** Add playbooks to scenario page dropdowns —
from: hardening playbooks to Act 2 (Step Modifiers), race to Act 3
(Control Flow), temporal patterns to Act 4 (Composition).

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
| ConsumerHandle | consumer-handle.test.ts | Stop, isStopped, scope close cancels |
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
- `playbooks/flow-control/from-until.playbook.yaml`
- `playbooks/flow-control/from-channel-close.playbook.yaml`
- `playbooks/flow-control/pause-resume.playbook.yaml`
- `playbooks/concurrency-patterns/named-consumer.playbook.yaml`

### Modified files
- `packages/yaml-core/src/step/walker.ts` — RaceStep type, parallel map form, new reserved/decorator keys
- `packages/yaml-core/src/step/structural-evaluator.ts` — evaluateRace(), policy-aware evaluateParallel()
- `packages/yaml-core/src/step/decorator-chain.ts` — new decorator insertion points, AbortSignal checks on existing decorators
- `packages/yaml-core/src/orchestration/directives.ts` — continuous LoopDirective, conditional RetryDirective
- `packages/yaml-core/src/orchestration/types.ts` — ConsumerHandle, consumer() on ScenarioScope
- `packages/yaml-core/src/step/schema-composer.ts` — race:, from:, cancel:, background:, parallel policy schemas
- `packages/yaml-core/src/orchestration/types.ts` — SpawnedTask.cancel(), spawn() signature change
- `playbooks/manifest.json` — 9 new entries (4 composition + 4 from-hardening + 1 race)

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
