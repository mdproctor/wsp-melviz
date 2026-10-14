# Event Sequence Pattern Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #546 — Epic: Event sequence pattern parity
**Issue group:** #538, #537, #542, #543, #544, #545

**Goal:** Reach TS runtime parity with the platform's event sequence
pattern features and parallel step extensions.

**Architecture:** Extend the existing Walker → StructuralEvaluator →
DecoratorChain architecture in `packages/yaml-core/src/`. Add AbortSignal
threading for cooperative cancellation. New decorators (from:, cancel:,
background:), new structural step (race:), and extended parallel
policies. All changes in yaml-core; no pages-runtime or pages-aria changes.

**Tech Stack:** TypeScript, vitest, AbortController/AbortSignal (Web API)

## Global Constraints

- All source files in `packages/yaml-core/src/`
- Tests use vitest (`describe`/`it`/`expect`)
- Run tests: `yarn --cwd packages/yaml-core test`
- Run single test: `yarn --cwd packages/yaml-core vitest run src/path/to/file.test.ts`
- Platform decorator chain ordering must be matched exactly
- `race:` is structural (not a decorator) — remove from DECORATOR_KEYS
- CANCELLED failures are non-retryable by default
- `loop: continuous` requires parse-time validation (cancel: or timeout: or background:)
- Use `AbortSignal.any()` for merging abort signals
- Use `ide_insert_member` / `ide_replace_member` for code editing

---

## Batch 1: AbortSignal Foundation + Primitive Overloads

The cross-cutting cancellation infrastructure that all later batches depend on.

### Task 1: AbortSignal-aware orchestration primitive overloads

**Files:**
- Modify: `packages/yaml-core/src/orchestration/types.ts` — add AbortSignal overloads
- Modify: `packages/yaml-core/src/orchestration/channel.ts` — implement receive(signal)
- Modify: `packages/yaml-core/src/orchestration/signal.ts` — implement await(signal)
- Modify: `packages/yaml-core/src/orchestration/semaphore.ts` — implement acquire(signal)
- Modify: `packages/yaml-core/src/orchestration/latch.ts` — implement await(signal)
- Modify: `packages/yaml-core/src/orchestration/spawned-task.ts` — add cancel(), task receives signal
- Test: `packages/yaml-core/src/orchestration/channel.test.ts`
- Test: `packages/yaml-core/src/orchestration/signal.test.ts`
- Test: `packages/yaml-core/src/orchestration/semaphore.test.ts`
- Test: `packages/yaml-core/src/orchestration/latch.test.ts`
- Test: `packages/yaml-core/src/orchestration/spawned-task.test.ts`

**Interfaces:**
- Consumes: existing OrcChannel, OrcSignal, OrcSemaphore, OrcLatch, SpawnedTask interfaces
- Produces: AbortSignal overloads on all four primitive interfaces + SpawnedTask.cancel()

- [ ] **Step 1: Write failing test for channel.receive(signal)**

```typescript
// In channel.test.ts, add to describe('DefaultOrcChannel')
it('receive rejects with AbortError when signal aborts', async () => {
  const ch = new DefaultOrcChannel<number>();
  const controller = new AbortController();
  const p = ch.receive(controller.signal);
  controller.abort();
  await expect(p).rejects.toThrow('aborted');
});

it('receive resolves normally when data arrives before abort', async () => {
  const ch = new DefaultOrcChannel<number>();
  const controller = new AbortController();
  await ch.send(42);
  expect(await ch.receive(controller.signal)).toBe(42);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/yaml-core vitest run src/orchestration/channel.test.ts`
Expected: FAIL — receive() doesn't accept AbortSignal parameter

- [ ] **Step 3: Add AbortSignal overload to OrcChannel interface and implement**

In `types.ts`, add overload to OrcChannel:
```typescript
receive(signal?: AbortSignal): Promise<T>;
receive(timeoutMs: number): Promise<T | undefined>;
```

In `channel.ts`, update `DefaultOrcChannel.receive()` to accept
`signal?: AbortSignal` and register an abort listener that rejects
the pending receive promise with `new DOMException('aborted', 'AbortError')`.

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd packages/yaml-core vitest run src/orchestration/channel.test.ts`
Expected: PASS

- [ ] **Step 5: Write failing tests for signal.await(signal), semaphore.acquire(signal), latch.await(signal)**

```typescript
// signal.test.ts
it('await rejects with AbortError when signal aborts', async () => {
  const sig = new DefaultOrcSignal();
  const controller = new AbortController();
  const p = sig.await(controller.signal);
  controller.abort();
  await expect(p).rejects.toThrow('aborted');
});

// semaphore.test.ts
it('acquire rejects with AbortError when signal aborts', async () => {
  const sem = new DefaultOrcSemaphore(0); // no permits
  const controller = new AbortController();
  const p = sem.acquire(controller.signal);
  controller.abort();
  await expect(p).rejects.toThrow('aborted');
});

// latch.test.ts
it('await rejects with AbortError when signal aborts', async () => {
  const latch = new DefaultOrcLatch(5); // never reaches 0
  const controller = new AbortController();
  const p = latch.await(controller.signal);
  controller.abort();
  await expect(p).rejects.toThrow('aborted');
});
```

- [ ] **Step 6: Implement AbortSignal overloads for signal, semaphore, latch**

Add `signal?: AbortSignal` parameter to:
- `DefaultOrcSignal.await()` — register abort listener on the waiter promise
- `DefaultOrcSemaphore.acquire()` — register abort listener on the queued resolve
- `DefaultOrcLatch.await()` — register abort listener on the pending resolve

Pattern: if `signal?.aborted` at entry, reject immediately. Otherwise,
register a `signal.addEventListener('abort', reject, { once: true })`
and clean up on normal resolution.

- [ ] **Step 7: Run all orchestration tests**

Run: `yarn --cwd packages/yaml-core vitest run src/orchestration/`
Expected: all PASS

- [ ] **Step 8: Write failing test for SpawnedTask.cancel()**

```typescript
// spawned-task.test.ts
it('cancel aborts the task via AbortSignal', async () => {
  let aborted = false;
  const task = new DefaultSpawnedTask('test', async (signal) => {
    await new Promise((_, reject) => {
      signal.addEventListener('abort', () => { aborted = true; reject(new Error('aborted')); });
    });
  });
  task.cancel();
  await task.join();
  expect(aborted).toBe(true);
  expect(task.isFailed()).toBe(true);
});
```

- [ ] **Step 9: Implement SpawnedTask.cancel()**

Update `SpawnedTask` interface: add `cancel(): void`.

Update `DefaultSpawnedTask`:
- Add private `controller = new AbortController()`
- Change constructor signature: `task: (signal: AbortSignal) => Promise<void>`
- Pass `this.controller.signal` to `task()`
- Implement `cancel(): void { this.controller.abort(); }`

- [ ] **Step 10: Update ScenarioScope.spawn() signature and close() behavior**

In `types.ts`, update `spawn` signature:
```typescript
spawn(name: string, task: (signal: AbortSignal) => Promise<void>): SpawnedTask;
```

In `scenario-scope.ts`, update `DefaultScenarioScope.close()` to call
`task.cancel()` on each spawned task before `joinWithTimeout()`.

- [ ] **Step 11: Run all orchestration tests**

Run: `yarn --cwd packages/yaml-core vitest run src/orchestration/`
Expected: all PASS (may need to update existing spawn() call sites in tests)

- [ ] **Step 12: Commit**

```bash
git add packages/yaml-core/src/orchestration/
git commit -m "feat(#546): add AbortSignal overloads to orchestration primitives

Refs #538 #537 #543 #544"
```

### Task 2: Context.signal + existing decorator AbortSignal checks

**Files:**
- Modify: `packages/yaml-core/src/step/decorator-chain.ts` — add signal to Context, update decorators
- Create: `packages/yaml-core/src/step/abort-utils.ts` — mergeAbortSignals utility
- Test: `packages/yaml-core/src/step/decorator-chain.test.ts`
- Test: `packages/yaml-core/src/step/abort-utils.test.ts` (new)

**Interfaces:**
- Consumes: Context interface, AbortSignal overloads from Task 1
- Produces: `Context.signal?: AbortSignal`, `mergeAbortSignals(parent, child): AbortSignal`

- [ ] **Step 1: Create abort-utils.ts with failing test**

```typescript
// abort-utils.test.ts
import { describe, it, expect } from 'vitest';
import { mergeAbortSignals } from './abort-utils.js';

describe('mergeAbortSignals', () => {
  it('returns child when parent is undefined', () => {
    const controller = new AbortController();
    expect(mergeAbortSignals(undefined, controller.signal)).toBe(controller.signal);
  });

  it('aborts when parent aborts', () => {
    const parent = new AbortController();
    const child = new AbortController();
    const merged = mergeAbortSignals(parent.signal, child.signal);
    parent.abort();
    expect(merged.aborted).toBe(true);
  });

  it('aborts when child aborts', () => {
    const parent = new AbortController();
    const child = new AbortController();
    const merged = mergeAbortSignals(parent.signal, child.signal);
    child.abort();
    expect(merged.aborted).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/yaml-core vitest run src/step/abort-utils.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement mergeAbortSignals**

```typescript
// abort-utils.ts
export function mergeAbortSignals(
  parent: AbortSignal | undefined,
  child: AbortSignal,
): AbortSignal {
  if (!parent) return child;
  return AbortSignal.any([parent, child]);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd packages/yaml-core vitest run src/step/abort-utils.test.ts`
Expected: PASS

- [ ] **Step 5: Add signal to Context interface**

In `decorator-chain.ts`, add to `Context`:
```typescript
readonly signal?: AbortSignal;
```

- [ ] **Step 6: Write failing tests for decorator AbortSignal checks**

```typescript
// In decorator-chain.test.ts, add:
it('WaitDecorator returns failure when signal is already aborted', async () => {
  const controller = new AbortController();
  controller.abort();
  // ... set up context with signal: controller.signal
  // ... invoke WaitDecorator.execute()
  // expect: stepFailure('Cancelled')
});
```

Add similar tests for DelayDecorator, SemaphoreDecorator, TimeoutDecorator.

- [ ] **Step 7: Update existing decorators to check signal.aborted**

In each of WaitDecorator, DelayDecorator, SemaphoreDecorator,
TimeoutDecorator — add at the start of `execute()`:
```typescript
if (context.signal?.aborted) return stepFailure('Cancelled');
```

For WaitDecorator and SemaphoreDecorator, pass `context.signal` to
the primitive's AbortSignal-aware overload. Catch `AbortError` and
return `stepFailure('Cancelled')`.

- [ ] **Step 8: Run all step tests**

Run: `yarn --cwd packages/yaml-core vitest run src/step/`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git add packages/yaml-core/src/step/abort-utils.ts packages/yaml-core/src/step/abort-utils.test.ts packages/yaml-core/src/step/decorator-chain.ts packages/yaml-core/src/step/decorator-chain.test.ts
git commit -m "feat(#546): add AbortSignal to Context and existing decorators

Refs #538 #537 #543 #544"
```

### Task 3: LoopDirective.continuous + conditional retry + failureCategory

**Files:**
- Modify: `packages/yaml-core/src/orchestration/directives.ts` — continuous variant + conditional retry
- Modify: `packages/yaml-core/src/step/walker.ts` — Result failureCategory, loop validation
- Modify: `packages/yaml-core/src/step/decorator-chain.ts` — LoopDecorator continuous, RetryDecorator conditional + CANCELLED skip
- Test: `packages/yaml-core/src/orchestration/directives.test.ts`
- Test: `packages/yaml-core/src/step/decorator-chain.test.ts`
- Test: `packages/yaml-core/src/step/walker.test.ts`

**Interfaces:**
- Consumes: LoopDirective, RetryDirective, Context.signal from Task 2
- Produces: `LoopDirective.continuous`, `RetryDirective.conditional`, `Result.failureCategory`

- [ ] **Step 1: Write failing test for parseLoopDirective('continuous')**

```typescript
// directives.test.ts
it('parses continuous string', () => {
  expect(parseLoopDirective('continuous')).toEqual({ type: 'continuous' });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/yaml-core vitest run src/orchestration/directives.test.ts`
Expected: FAIL

- [ ] **Step 3: Add continuous variant to LoopDirective and parseLoopDirective**

In `directives.ts`, extend the type:
```typescript
| { type: 'continuous' }
```

Add to `parseLoopDirective`:
```typescript
if (raw === 'continuous') return { type: 'continuous' };
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write failing test for LoopDecorator continuous with AbortSignal**

```typescript
// decorator-chain.test.ts
it('loop continuous iterates until AbortSignal fires', async () => {
  const controller = new AbortController();
  let count = 0;
  // action that increments count and aborts after 5
  const action = { execute: async () => {
    count++;
    if (count >= 5) controller.abort();
    return stepSuccess({});
  }};
  const chain = DecoratorChain.build({ loop: 'continuous' }, action);
  const result = await chain.execute({ ...baseContext, signal: controller.signal });
  expect(count).toBe(5);
  expect(result.kind).toBe('failure');
});
```

- [ ] **Step 6: Implement LoopDecorator continuous**

In `decorator-chain.ts`, update `LoopDecorator.execute()`:
```typescript
if (directive.type === 'continuous') {
  while (true) {
    if (context.signal?.aborted) return stepFailure('Cancelled');
    lastResult = await this.next.execute(context);
    if (lastResult.kind === 'failure') return lastResult;
  }
}
```

- [ ] **Step 7: Write failing test for parse-time validation**

```typescript
// walker.test.ts
it('rejects loop: continuous without cancel: or timeout:', () => {
  expect(() => Walker.resolve([{ someAction: {}, loop: 'continuous' }], catalog))
    .toThrow('loop: continuous requires cancel:, timeout:, or background:');
});

it('allows loop: continuous with cancel:', () => {
  expect(() => Walker.resolve(
    [{ someAction: {}, loop: 'continuous', cancel: 'shutdown' }], catalog,
  )).not.toThrow();
});
```

- [ ] **Step 8: Implement parse-time validation in Walker**

In `walker.ts`, after decorator extraction in `resolveOne()`:
```typescript
if (decorators['loop'] === 'continuous'
    && !decorators['cancel'] && !decorators['timeout']
    && !decorators['background']) {
  throw new Error(`${stepPath}: loop: continuous requires cancel:, timeout:, or background:`);
}
```

Add `'cancel'`, `'background'` to `RESERVED_KEYS` and `DECORATOR_KEYS`.

- [ ] **Step 9: Add failureCategory to Result + conditional retry**

Add `failureCategory?: string` to the failure variant of `Result` in `walker.ts`.

Add `stepFailureWithCategory(message, category)` helper.

Extend `RetryDirective`:
```typescript
| { type: 'conditional'; max: number; on?: string[]; except?: string[];
    backoff?: string; delayMs?: number }
```

Update `RetryDecorator.execute()`:
- Skip CANCELLED failures by default (all retry modes)
- For conditional type: check `on`/`except` against `failureCategory`

Update `TimeoutDecorator`: set `failureCategory: 'TIMEOUT'` on failure.
Update `SemaphoreDecorator`: set `failureCategory: 'TIMEOUT'` on deadline failure.
Update `WaitDecorator`: set `failureCategory: 'TIMEOUT'` on deadline failure.

- [ ] **Step 10: Run all tests**

Run: `yarn --cwd packages/yaml-core vitest run`
Expected: all PASS

- [ ] **Step 11: Commit**

```bash
git add packages/yaml-core/src/orchestration/directives.ts packages/yaml-core/src/orchestration/directives.test.ts packages/yaml-core/src/step/walker.ts packages/yaml-core/src/step/walker.test.ts packages/yaml-core/src/step/decorator-chain.ts packages/yaml-core/src/step/decorator-chain.test.ts
git commit -m "feat(#537): loop: continuous, conditional retry, failureCategory

Refs #537 #546"
```

---

## Batch 2: New Decorators (from:, cancel:, background:)

### Task 4: from: decorator (#538)

**Files:**
- Create: `packages/yaml-core/src/step/from-decorator.ts` — FromDecorator + pushFromContext
- Create: `packages/yaml-core/src/step/from-decorator.test.ts`
- Modify: `packages/yaml-core/src/step/walker.ts` — add 'from' to RESERVED_KEYS, DECORATOR_KEYS
- Modify: `packages/yaml-core/src/step/decorator-chain.ts` — wire FromDecorator into chain

**Interfaces:**
- Consumes: Context (with signal), OrcChannel.receive(signal), VariableResolver.withObjectScope
- Produces: `FromDecorator` class, `pushFromContext()` helper

- [ ] **Step 1: Write failing tests for FromDecorator**

```typescript
// from-decorator.test.ts
import { describe, it, expect } from 'vitest';
import { FromDecorator, pushFromContext } from './from-decorator.js';
import { stepSuccess } from './walker.js';

describe('FromDecorator', () => {
  it('reads from channel and injects as ${from.*} variables', async () => {
    // Set up: channel with one item { value: 42, name: 'sensor' }
    // Action that reads ${from.value} via resolver
    // Assert: action receives the channel data
  });

  it('simple form: from: channel-name', async () => {
    // from: 'sensor-readings' reads from channel 'sensor-readings'
  });

  it('full form with filter', async () => {
    // from: { channel: 'readings', filter: '${from.value}' }
    // truthy filter passes, falsy filter skips
  });

  it('full form with alias', async () => {
    // from: { channel: 'readings', as: 'reading' }
    // variables injected as ${reading.*} not ${from.*}
  });

  it('fan-in: first channel with data wins', async () => {
    // from: { channels: ['a', 'b'] }
    // send to channel 'b', verify data from 'b' arrives
  });

  it('fan-in cancels losing channel receives', async () => {
    // verify no ghost consumers on losing channels
  });

  it('returns closed sentinel on ChannelClosedError', async () => {
    // channel.close() before receive → success with __from.closed
  });

  it('returns failure on AbortSignal', async () => {
    // signal already aborted → stepFailure('Cancelled')
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement FromDecorator**

Create `from-decorator.ts` with:
- `FromDecorator` class implementing `DecoratedExecution`
- `parseFromSpec(raw)` — parses string | { channel, as?, filter? } | { channels, as? }
- `pushFromContext(resolver, alias, item)` — ObjectVariableSource pattern
- `receiveFanIn(channelNames, context)` — Promise.race with per-channel AbortControllers

Key: inject data BEFORE filter evaluation (review fix R1-02).

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Wire into Walker and DecoratorChain**

In `walker.ts`: add `'from'` to `RESERVED_KEYS` and `DECORATOR_KEYS`.

In `decorator-chain.ts`, in `DecoratorChain.build()`, add after
`on-error` and before `timeout`:
```typescript
if (decorators['from'] != null) {
  execution = new FromDecorator(execution, decorators['from']);
}
```

Import `FromDecorator` from `./from-decorator.js`.

- [ ] **Step 6: Write walker integration test**

```typescript
// walker.test.ts
it('from: is treated as a decorator, not a structural key', () => {
  const steps = Walker.resolve([{ someAction: {}, from: 'my-channel' }], catalog);
  expect(steps[0]!.kind).toBe('plugin');
  expect(steps[0]!.decorators['from']).toBe('my-channel');
});
```

- [ ] **Step 7: Run all tests**

Run: `yarn --cwd packages/yaml-core vitest run`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/step/from-decorator.ts packages/yaml-core/src/step/from-decorator.test.ts packages/yaml-core/src/step/walker.ts packages/yaml-core/src/step/decorator-chain.ts
git commit -m "feat(#538): from: decorator — channel read, filter, fan-in

Refs #538 #546"
```

### Task 5: cancel: and background: decorators (#537)

**Files:**
- Create: `packages/yaml-core/src/step/cancel-decorator.ts`
- Create: `packages/yaml-core/src/step/cancel-decorator.test.ts`
- Create: `packages/yaml-core/src/step/background-decorator.ts`
- Create: `packages/yaml-core/src/step/background-decorator.test.ts`
- Modify: `packages/yaml-core/src/step/decorator-chain.ts` — wire cancel + background

**Interfaces:**
- Consumes: Context.signal, OrcSignal.await(), mergeAbortSignals, scope.spawn()
- Produces: `CancelDecorator`, `BackgroundDecorator`

- [ ] **Step 1: Write failing tests for CancelDecorator**

```typescript
// cancel-decorator.test.ts
describe('CancelDecorator', () => {
  it('cancels inner execution when named signal fires', async () => {
    // Set up scope with signal 'shutdown'
    // CancelDecorator wraps a long-running action
    // Fire signal → action receives AbortSignal abort
  });

  it('returns failure immediately if signal already fired', async () => {
    // Signal already signalled → stepFailure('Cancelled')
  });

  it('cleans up signal listener on normal completion', async () => {
    // Action completes normally, signal never fires
    // No orphaned listeners
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement CancelDecorator**

```typescript
// cancel-decorator.ts
export class CancelDecorator implements DecoratedExecution {
  constructor(
    private readonly next: DecoratedExecution,
    private readonly signalName: string,
  ) {}

  async execute(context: Context): Promise<Result> {
    const orcSignal = context.scope.signal(this.signalName);
    if (orcSignal.isSignalled()) return stepFailure('Cancelled');

    const controller = new AbortController();
    orcSignal.await().then(() => controller.abort()).catch(() => {});

    const merged = mergeAbortSignals(context.signal, controller.signal);
    try {
      return await this.next.execute({ ...context, signal: merged });
    } finally {
      controller.abort();
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write failing tests for BackgroundDecorator**

```typescript
// background-decorator.test.ts
describe('BackgroundDecorator', () => {
  it('resolves immediately with handle name', async () => {
    let started = false;
    const action = { execute: async () => { started = true; return stepSuccess({}); }};
    // Wrap with BackgroundDecorator
    const result = await decorator.execute(context);
    expect(result.kind).toBe('success');
    // Background task runs asynchronously
  });

  it('scope.close() cancels background task', async () => {
    // Long-running background action, close scope, verify aborted
  });
});
```

- [ ] **Step 6: Implement BackgroundDecorator**

```typescript
// background-decorator.ts
export class BackgroundDecorator implements DecoratedExecution {
  constructor(private readonly next: DecoratedExecution) {}

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

- [ ] **Step 7: Wire into DecoratorChain**

In `decorator-chain.ts`, `DecoratorChain.build()`:

After `forEach`:
```typescript
if (decorators['background'] === true) {
  execution = new BackgroundDecorator(execution);
}
```

After `background`:
```typescript
if (decorators['cancel'] != null) {
  execution = new CancelDecorator(execution, String(decorators['cancel']));
}
```

Chain order (inside→out): ... → on-error → from → timeout → ... → loop → cancel → background → forEach → if

- [ ] **Step 8: Run all tests**

Run: `yarn --cwd packages/yaml-core vitest run`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git add packages/yaml-core/src/step/cancel-decorator.ts packages/yaml-core/src/step/cancel-decorator.test.ts packages/yaml-core/src/step/background-decorator.ts packages/yaml-core/src/step/background-decorator.test.ts packages/yaml-core/src/step/decorator-chain.ts
git commit -m "feat(#537): cancel: and background: decorators

Refs #537 #546"
```

### Task 6: forEach collect:all + primitive variable resolution (#537)

**Files:**
- Modify: `packages/yaml-core/src/step/decorator-chain.ts` — ForEachDecorator collect
- Modify: `packages/yaml-core/src/variable-resolver.ts` — primitive variable source
- Test: `packages/yaml-core/src/step/decorator-chain.test.ts`
- Test: `packages/yaml-core/src/variable-resolver.test.ts`

**Interfaces:**
- Consumes: ForEachDecorator, VariableResolver, ScenarioScope primitives
- Produces: `collect: 'all'` mode, `flag.*`/`counter.*`/`gauge.*`/`signal.*`/`accumulator.*` resolution

- [ ] **Step 1: Write failing test for forEach collect: all**

```typescript
// decorator-chain.test.ts
it('forEach collect: all gathers all results', async () => {
  const items = [{ id: 1 }, { id: 2 }, { id: 3 }];
  const action = { execute: async (params: Record<string, unknown>) => {
    return stepSuccess({ doubled: (params as any).value.id * 2 });
  }};
  const chain = DecoratorChain.build({
    forEach: { in: items, as: 'item', collect: 'all' },
  }, action);
  const result = await chain.execute(baseContext);
  expect(result.kind).toBe('success');
  expect((result as any).output['__forEach.collected']).toHaveLength(3);
});

it('forEach collect: all returns failure when all items fail', async () => {
  const action = { execute: async () => stepFailure('bad') };
  const chain = DecoratorChain.build({
    forEach: { in: [1, 2], as: 'item', collect: 'all' },
  }, action);
  const result = await chain.execute(baseContext);
  expect(result.kind).toBe('failure');
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement collect: all in ForEachDecorator**

Add collect check after parallel check in `ForEachDecorator.execute()`:
```typescript
if (spec['collect'] === 'all') {
  const results: unknown[] = [];
  let failures = 0;
  for (let i = 0; i < items.length; i++) {
    const scoped = pushEachContext(ctx.resolver, as, items[i], i);
    const r = await this.next.execute(withResolver(ctx, scoped));
    if (r.kind === 'success') results.push(r.output);
    else failures++;
  }
  if (failures === items.length) return stepFailure('forEach collect: all items failed');
  return stepSuccess({ '__forEach.collected': results });
}
```

- [ ] **Step 4: Write failing test for primitive variable resolution**

```typescript
// variable-resolver.test.ts
it('resolves flag.* prefix against scope', () => {
  // scope.flag('shutdown').set()
  // resolver.resolveString('${flag.shutdown}') → 'true'
});

it('resolves counter.* prefix against scope', () => {
  // scope.counter('workers').increment()
  // resolver.resolveString('${counter.workers}') → '1'
});
```

- [ ] **Step 5: Implement primitive variable source**

Create a `PrimitiveVariableSource` that registers prefix sources for
`flag`, `counter`, `gauge`, `signal`, `accumulator` against the
ScenarioScope. Wire into VariableResolver when scope is available.

- [ ] **Step 6: Run all tests**

Run: `yarn --cwd packages/yaml-core vitest run`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/src/step/decorator-chain.ts packages/yaml-core/src/step/decorator-chain.test.ts packages/yaml-core/src/variable-resolver.ts packages/yaml-core/src/variable-resolver.test.ts
git commit -m "feat(#537): forEach collect:all + primitive variable resolution

Refs #537 #546"
```

---

## Batch 3: Structural Steps (race:, parallel extensions)

### Task 7: race: structural step (#543)

**Files:**
- Modify: `packages/yaml-core/src/step/walker.ts` — RaceStep type, structural parsing, remove from DECORATOR_KEYS
- Modify: `packages/yaml-core/src/step/structural-evaluator.ts` — evaluateRace()
- Test: `packages/yaml-core/src/step/walker.test.ts`
- Test: `packages/yaml-core/src/step/structural-evaluator.test.ts`

**Interfaces:**
- Consumes: Walker.resolveOne, StructuralEvaluator.dispatch, mergeAbortSignals, Context.signal
- Produces: `RaceStep` type, `evaluateRace()` method

- [ ] **Step 1: Write failing test for Walker race: parsing**

```typescript
// walker.test.ts
it('parses race: as structural step with substeps', () => {
  const steps = Walker.resolve([{
    race: [
      { actionA: {} },
      { actionB: {} },
    ],
  }], catalog);
  expect(steps).toHaveLength(1);
  expect(steps[0]!.kind).toBe('race');
  expect((steps[0] as RaceStep).steps).toHaveLength(2);
});

it('race: is not treated as a decorator', () => {
  // Verify race is removed from DECORATOR_KEYS
  const steps = Walker.resolve([{ race: [{ actionA: {} }] }], catalog);
  expect(steps[0]!.decorators).not.toHaveProperty('race');
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Add RaceStep type and walker parsing**

In `walker.ts`:

Add `RaceStep` to the type union:
```typescript
export interface RaceStep {
  kind: 'race';
  name: string | null;
  steps: ResolvedStep[];
  decorators: Record<string, unknown>;
}
```

Add `| RaceStep` to `ResolvedStep` union.

Remove `'race'` from `DECORATOR_KEYS`.

Add structural handling in `resolveOne()` after `select`:
```typescript
if (structuralType === 'race') {
  const substeps = Walker.resolveAtDepth(
    structuralValue as Record<string, unknown>[],
    catalog, depth + 1, `${stepPath} → race`, seenNames,
  );
  return { kind: 'race', name: stepName, steps: substeps, decorators };
}
```

Add `'race'` to the structural detection: `key === 'race'` alongside
`block`, `parallel`, `try`, `select`.

- [ ] **Step 4: Run walker tests**

- [ ] **Step 5: Write failing test for evaluateRace**

```typescript
// structural-evaluator.test.ts
it('race: first branch to complete wins', async () => {
  // Branch A: delay then success
  // Branch B: immediate success
  // Result: Branch B wins
});

it('race: losing branches are cancelled', async () => {
  // Branch A: waits on signal (never fires)
  // Branch B: immediate success
  // Result: Branch B wins, Branch A receives abort
});

it('race: handles branch exceptions', async () => {
  // Branch A: throws error
  // Branch B: succeeds
  // Result: first to settle wins (A's error)
});
```

- [ ] **Step 6: Implement evaluateRace**

In `structural-evaluator.ts`:

Add `'race'` to dispatch switch.

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

  const winner = await Promise.race(promises);
  for (let i = 0; i < controllers.length; i++) {
    if (i !== winner.index) controllers[i]!.abort();
  }
  return winner.result;
}
```

Import `mergeAbortSignals` from `./abort-utils.js`.

- [ ] **Step 7: Run all tests**

Run: `yarn --cwd packages/yaml-core vitest run`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/step/walker.ts packages/yaml-core/src/step/walker.test.ts packages/yaml-core/src/step/structural-evaluator.ts packages/yaml-core/src/step/structural-evaluator.test.ts
git commit -m "feat(#543): race: structural step — first-to-complete with cancellation

Refs #543 #546"
```

### Task 8: Parallel extensions (#544)

**Files:**
- Modify: `packages/yaml-core/src/step/walker.ts` — ParallelStep extensions, map form, new reserved keys
- Create: `packages/yaml-core/src/step/parallel-completion-tracker.ts`
- Create: `packages/yaml-core/src/step/parallel-completion-tracker.test.ts`
- Modify: `packages/yaml-core/src/step/structural-evaluator.ts` — policy-aware evaluateParallel
- Test: `packages/yaml-core/src/step/walker.test.ts`
- Test: `packages/yaml-core/src/step/structural-evaluator.test.ts`

**Interfaces:**
- Consumes: ParallelStep, OrcLatch, AbortController, mergeAbortSignals
- Produces: Extended `ParallelStep` (branches, advanceAfter, cancelRemaining, collectionTimeout), `ParallelCompletionTracker`, `NamedBranch`

- [ ] **Step 1: Write failing test for parallel map form parsing**

```typescript
// walker.test.ts
it('parses parallel: map form into named branches', () => {
  const steps = Walker.resolve([{
    parallel: {
      'branch-a': [{ actionA: {} }],
      'branch-b': [{ actionB: {} }],
    },
  }], catalog);
  const p = steps[0] as ParallelStep;
  expect(p.branches).toHaveLength(2);
  expect(p.branches![0]!.name).toBe('branch-a');
  expect(p.steps).toHaveLength(2); // flat view preserved
});

it('parses parallel: list form preserving backward compat', () => {
  const steps = Walker.resolve([{
    parallel: [{ actionA: {} }, { actionB: {} }],
  }], catalog);
  const p = steps[0] as ParallelStep;
  expect(p.steps).toHaveLength(2);
  expect(p.branches).toBeUndefined();
});

it('parses advanceAfter, cancelRemaining, collectionTimeout', () => {
  const steps = Walker.resolve([{
    parallel: { a: [{ actionA: {} }], b: [{ actionB: {} }] },
    'advance-after': 1,
    'cancel-remaining': true,
    'collection-timeout': '5s',
  }], catalog);
  const p = steps[0] as ParallelStep;
  expect(p.advanceAfter).toBe(1);
  expect(p.cancelRemaining).toBe(true);
  expect(p.collectionTimeout).toBe(5000);
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Extend ParallelStep and walker parsing**

In `walker.ts`:

Add `NamedBranch` interface:
```typescript
export interface NamedBranch {
  name: string;
  steps: ResolvedStep[];
}
```

Extend `ParallelStep`:
```typescript
export interface ParallelStep {
  kind: 'parallel';
  name: string | null;
  steps: ResolvedStep[];
  branches?: NamedBranch[];
  advanceAfter?: number;
  cancelRemaining?: boolean;
  collectionTimeout?: number;
  decorators: Record<string, unknown>;
}
```

Add `'advance-after'`, `'cancel-remaining'`, `'collection-timeout'` to `RESERVED_KEYS`.

Update the `parallel` structural handler to detect array vs object form
and extract policy keys.

- [ ] **Step 4: Run walker tests**

- [ ] **Step 5: Write failing test for ParallelCompletionTracker**

```typescript
// parallel-completion-tracker.test.ts
describe('ParallelCompletionTracker', () => {
  it('barrier mode: resolves when all branches complete', async () => {
    // 3 branches, no advanceAfter → wait for all
  });

  it('discriminator mode: resolves after N branches complete', async () => {
    // 3 branches, advanceAfter: 1 → resolves after first
  });

  it('cancelRemaining: aborts remaining after advancement', async () => {
    // 3 branches, advanceAfter: 1, cancelRemaining: true
    // First completes → other two receive abort signal
  });

  it('collectionTimeout: times out remaining after advancement', async () => {
    // 3 branches, advanceAfter: 1, collectionTimeout: 100ms
    // First completes, others timeout after 100ms
  });

  it('unique latch names per step (nesting safe)', async () => {
    // Two trackers with different step names don't collide
  });
});
```

- [ ] **Step 6: Implement ParallelCompletionTracker**

```typescript
// parallel-completion-tracker.ts
export class ParallelCompletionTracker {
  private readonly advancementLatch: OrcLatch;
  private readonly completionLatch: OrcLatch;
  private readonly controllers: AbortController[];

  constructor(
    stepName: string,
    branchCount: number,
    advanceAfter: number,
    scope: ScenarioScope,
  ) {
    this.advancementLatch = scope.latch(`__parallel_advance_${stepName}`, advanceAfter);
    this.completionLatch = scope.latch(`__parallel_complete_${stepName}`, branchCount);
    this.controllers = Array.from({ length: branchCount }, () => new AbortController());
  }

  signalFor(index: number): AbortSignal { return this.controllers[index]!.signal; }
  branchDone(index: number): void {
    this.advancementLatch.countDown();
    this.completionLatch.countDown();
  }
  async awaitAdvancement(): Promise<void> { await this.advancementLatch.await(); }
  cancelRemaining(advancedIndices: Set<number>): void {
    for (let i = 0; i < this.controllers.length; i++) {
      if (!advancedIndices.has(i)) this.controllers[i]!.abort();
    }
  }
  async awaitCompletion(timeoutMs?: number): Promise<boolean> {
    if (timeoutMs) return this.completionLatch.await(timeoutMs);
    await this.completionLatch.await();
    return true;
  }
}
```

- [ ] **Step 7: Implement policy-aware evaluateParallel**

In `structural-evaluator.ts`, update `evaluateParallel()`:

```typescript
private async evaluateParallel(step: ParallelStep, context: Context): Promise<Result> {
  if (!step.advanceAfter) {
    return this.evaluateParallelBarrier(step, context);
  }
  return this.evaluateParallelDiscriminator(step, context);
}
```

Implement `evaluateParallelBarrier` (refactored current code) and
`evaluateParallelDiscriminator` (using ParallelCompletionTracker).

- [ ] **Step 8: Run all tests**

Run: `yarn --cwd packages/yaml-core vitest run`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git add packages/yaml-core/src/step/walker.ts packages/yaml-core/src/step/walker.test.ts packages/yaml-core/src/step/parallel-completion-tracker.ts packages/yaml-core/src/step/parallel-completion-tracker.test.ts packages/yaml-core/src/step/structural-evaluator.ts packages/yaml-core/src/step/structural-evaluator.test.ts
git commit -m "feat(#544): parallel extensions — discriminator, cancelRemaining, collectionTimeout

Refs #544 #546"
```

---

## Batch 4: from: Hardening + Schema (#542)

### Task 9: from: hardening — until, channel close, named consumers (#542)

**Files:**
- Modify: `packages/yaml-core/src/step/from-decorator.ts` — until, ChannelClosedError, ConsumerHandle
- Create: `packages/yaml-core/src/orchestration/consumer-handle.ts`
- Create: `packages/yaml-core/src/orchestration/consumer-handle.test.ts`
- Modify: `packages/yaml-core/src/orchestration/types.ts` — consumer() on ScenarioScope
- Modify: `packages/yaml-core/src/step/from-decorator.test.ts` — hardening tests

**Interfaces:**
- Consumes: FromDecorator from Task 4, ScenarioScope, ChannelClosedError
- Produces: `ConsumerHandle` interface, `consumer()` on scope, `until` field on from: spec

- [ ] **Step 1: Write failing tests for from: until**

```typescript
// from-decorator.test.ts
it('from: until terminates when condition is true', async () => {
  // from: { channel: 'events', until: '${flag.shutdown}' }
  // Set flag after 2 reads → terminates with __from.terminated
});
```

- [ ] **Step 2: Implement from: until**

In FromDecorator, before receive:
```typescript
if (spec.until) {
  const resolved = context.resolver.resolveString(spec.until, 'from-until');
  if (isTruthy(resolved)) return stepSuccess({ '__from.terminated': true });
}
```

- [ ] **Step 3: Write failing tests for ConsumerHandle**

```typescript
// consumer-handle.test.ts
it('stop() sets isStopped and aborts the controller', () => {
  const handle = new DefaultConsumerHandle('enricher');
  expect(handle.isStopped()).toBe(false);
  handle.stop();
  expect(handle.isStopped()).toBe(true);
  expect(handle.signal.aborted).toBe(true);
});
```

- [ ] **Step 4: Implement ConsumerHandle**

```typescript
// consumer-handle.ts
export interface ConsumerHandle {
  readonly name: string;
  stop(): void;
  isStopped(): boolean;
  readonly signal: AbortSignal;
}

export class DefaultConsumerHandle implements ConsumerHandle {
  private readonly controller = new AbortController();
  readonly name: string;
  constructor(name: string) { this.name = name; }
  stop(): void { this.controller.abort(); }
  isStopped(): boolean { return this.controller.signal.aborted; }
  get signal(): AbortSignal { return this.controller.signal; }
}
```

- [ ] **Step 5: Wire consumer() into ScenarioScope**

Add `consumer(name: string): ConsumerHandle` to ScenarioScope interface.
Implement in DefaultScenarioScope with get-or-create semantics.

- [ ] **Step 6: Wire named consumer into FromDecorator**

When `spec.name` is present, register a consumer handle and merge its
signal into the context for cancellation.

- [ ] **Step 7: Run all tests**

Run: `yarn --cwd packages/yaml-core vitest run`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/src/step/from-decorator.ts packages/yaml-core/src/step/from-decorator.test.ts packages/yaml-core/src/orchestration/consumer-handle.ts packages/yaml-core/src/orchestration/consumer-handle.test.ts packages/yaml-core/src/orchestration/types.ts
git commit -m "feat(#542): from: hardening — until, channel close, named consumers

Refs #542 #546"
```

### Task 10: Schema composer updates + stop action

**Files:**
- Modify: `packages/yaml-core/src/step/schema-composer.ts` — race:, from:, cancel:, background:, parallel policy
- Modify: `packages/yaml-core/src/step/schema-composer.test.ts`

**Interfaces:**
- Consumes: SchemaComposer, all new types from prior tasks
- Produces: Updated JSON Schema with all new structural types and decorator keys

- [ ] **Step 1: Write failing test for schema additions**

```typescript
// schema-composer.test.ts
it('includes race: as a structural variant', () => {
  const schema = SchemaComposer.compose(catalog);
  const oneOf = schema.oneOf as any[];
  const raceVariant = oneOf.find(v => v.required?.includes('race'));
  expect(raceVariant).toBeDefined();
});

it('includes from: in decorator keys', () => {
  const schema = SchemaComposer.compose(catalog);
  expect(schema.properties).toHaveProperty('from');
});

it('does not include race: in decorator keys', () => {
  // race is structural, not a decorator
});
```

- [ ] **Step 2: Implement schema updates**

In `schema-composer.ts`:
- Remove `'race'` from `DECORATOR_KEYS` array
- Add `'from'`, `'cancel'`, `'background'`, `'advance-after'`, `'cancel-remaining'`, `'collection-timeout'` to `DECORATOR_KEYS`
- Add race: structural variant to `oneOf`:
  ```typescript
  oneOf.push({
    type: 'object',
    properties: { race: { type: 'array' }, step: { type: 'string' } },
    required: ['race'],
  });
  ```
- Update parallel: variant to support both array and object form

- [ ] **Step 3: Run tests**

- [ ] **Step 4: Commit**

```bash
git add packages/yaml-core/src/step/schema-composer.ts packages/yaml-core/src/step/schema-composer.test.ts
git commit -m "feat(#546): schema composer updates — race:, from:, parallel extensions

Refs #546"
```

---

## Batch 5: Playbooks + Scenario Pages (#545)

### Task 11: Playbook YAML files + manifest

**Files:**
- Create: `playbooks/step-workflows/race.playbook.yaml`
- Create: `playbooks/flow-control/from-until.playbook.yaml`
- Create: `playbooks/flow-control/from-channel-close.playbook.yaml`
- Create: `playbooks/flow-control/pause-resume.playbook.yaml`
- Create: `playbooks/concurrency-patterns/named-consumer.playbook.yaml`
- Create: `playbooks/composition/timed-absence.playbook.yaml`
- Create: `playbooks/composition/delayed-sequence.playbook.yaml`
- Create: `playbooks/composition/heartbeat-monitor.playbook.yaml`
- Create: `playbooks/composition/recursive-pattern.playbook.yaml`
- Modify: `playbooks/manifest.json` — 9 new entries

**Interfaces:**
- Consumes: all runtime features from Batches 1-4
- Produces: 9 playbook YAML files + manifest entries

- [ ] **Step 1: Write race.playbook.yaml**

```yaml
playbook: "1.0"
schema: client
---
meta:
  title: "Race"
  description: "First-to-complete over arbitrary blocks — two branches execute concurrently, the first to finish wins, the loser is cancelled."
  tags:
    - race
    - first-to-complete
    - cancellation
    - step-workflows
---
- race:
  - block:
    - delay: 200ms
    - log: { message: "Slow branch finished" }
  - log: { message: "Fast branch finished" }
```

- [ ] **Step 2: Write from: hardening playbooks (4 files)**

Create `from-until.playbook.yaml`, `from-channel-close.playbook.yaml`,
`pause-resume.playbook.yaml`, `named-consumer.playbook.yaml` with
appropriate YAML structures demonstrating each feature.

- [ ] **Step 3: Write composition playbooks (4 files)**

Create `timed-absence.playbook.yaml`, `delayed-sequence.playbook.yaml`,
`heartbeat-monitor.playbook.yaml`, `recursive-pattern.playbook.yaml`.

Each demonstrates a temporal pattern composed from existing primitives:
- Timed Absence: block + timeout on from:
- Delayed Sequence: sequential from: reads with delay
- Heartbeat Monitor: loop: continuous + delay + from: with timeout
- Recursive Pattern: nested block/race/loop

- [ ] **Step 4: Update manifest.json**

Add all 9 entries to their respective categories. All marked
`"runnable": false`.

- [ ] **Step 5: Validate playbook YAML parses**

Run the schema validator against each new playbook to ensure
well-formed YAML:
```bash
yarn --cwd packages/yaml-core vitest run src/playbook.test.ts
```

- [ ] **Step 6: Commit**

```bash
git add playbooks/ 
git commit -m "feat(#545): 9 event sequence playbooks — race, from-hardening, temporal composition

Refs #545 #546"
```

### Task 12: Scenario page updates + deferred issue

**Files:**
- Modify: scenario example pages (Act 2, 3, 4 dropdown pickers)
- Create: GitHub issue for deferred consumer lifecycle (pause/resume/start)

**Interfaces:**
- Consumes: manifest.json entries from Task 11
- Produces: Updated scenario pages, tracking issue

- [ ] **Step 1: Identify scenario pages to update**

Find the scenario example pages that have dropdown pickers:
```bash
find examples/samples/Scenarios -name "*.page.yaml" -o -name "*.ts"
```

- [ ] **Step 2: Add from-hardening playbooks to Act 2 (Step Modifiers)**

Update the relevant scenario page dropdown to include from-until,
from-channel-close entries.

- [ ] **Step 3: Add race to Act 3 (Control Flow)**

Update the control flow scenario page dropdown to include the race
playbook.

- [ ] **Step 4: Add temporal patterns to Act 4 (Composition)**

Update the composition scenario page dropdown to include
timed-absence, delayed-sequence, heartbeat-monitor, recursive-pattern.

- [ ] **Step 5: Create GitHub issue for deferred consumer lifecycle**

```bash
gh issue create --repo casehubio/casehub-pages \
  --title "Deferred: Consumer lifecycle pause/resume/start (#542 follow-up)" \
  --body "Platform #602 includes pause/resume/start on ConsumerHandle. Pages #542 implemented stop-only — pause/resume deferred as unnecessary for browser runtime. Track here if YAML scenarios surface a need for these operations. See spec: specs/issue-546-event-sequence-parity/2026-10-14-event-sequence-parity-design.md §3." \
  --label "deferred"
```

- [ ] **Step 6: Run full test suite**

```bash
yarn --cwd packages/yaml-core test
```

- [ ] **Step 7: Commit**

```bash
git add examples/ 
git commit -m "feat(#545): scenario page updates — from-hardening, race, temporal patterns

Refs #545 #546"
```

---

## References

- [specs/issue-546-event-sequence-parity/2026-10-14-event-sequence-parity-design.md] — design spec
- [specs/issue-546-event-sequence-parity/decisions.md] — D1-D8 design decisions
- [packages/yaml-core/src/step/walker.ts] — walker with ResolvedStep union
- [packages/yaml-core/src/step/structural-evaluator.ts] — evaluator dispatch
- [packages/yaml-core/src/step/decorator-chain.ts] — decorator chain and ordering
- [packages/yaml-core/src/orchestration/types.ts] — ScenarioScope, primitive interfaces
- [packages/yaml-core/src/orchestration/channel.ts] — DefaultOrcChannel
- [packages/yaml-core/src/orchestration/spawned-task.ts] — SpawnedTask interface
- [packages/yaml-core/src/step/schema-composer.ts] — JSON schema generation
- [packages/yaml-core/src/variable-resolver.ts] — variable resolution
- [playbooks/manifest.json] — playbook manifest structure
- [GitHub #546] — epic
- [GitHub #538, #537, #542, #543, #544, #545] — child issues
