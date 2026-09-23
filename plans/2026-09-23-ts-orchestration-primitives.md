# TS Orchestration Primitives Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #462 — feat: TS port of yaml-core orchestration primitives — parity with platform#386
**Issue group:** #462

**Goal:** Port the Java `io.casehub.yaml.core.orchestration` package to TypeScript as Promise-based primitives in `packages/yaml-core/src/orchestration/`.

**Architecture:** All Java blocking methods become async (Promise-based). Primitives are plain TS objects — cooperative DES scheduler (#461/#463) ensures single-step execution, so no concurrent-access protection needed. Discriminated unions replace Java sealed interfaces.

**Tech Stack:** TypeScript, vitest, zod (existing dep)

## Global Constraints

- All imports use `.js` extension suffix (existing project convention)
- Tests use vitest with explicit `import { describe, it, expect } from 'vitest'`
- Test files co-located: `<name>.test.ts` alongside `<name>.ts`
- No Atomics, SharedArrayBuffer, or Web Workers
- Timeout methods use optional `timeoutMs?: number` with `setTimeout` — scheduler wraps later
- Package export path: `@casehubio/yaml-core/orchestration`

---

## Batch 1: Foundation — errors, callbacks, duration parser, directives

Standalone types with no dependencies on each other. After this batch,
all shared types that primitives depend on are available.

### Task 1: Error types and callback types

**Files:**
- Create: `packages/yaml-core/src/orchestration/errors.ts`
- Create: `packages/yaml-core/src/orchestration/callbacks.ts`
- Test: `packages/yaml-core/src/orchestration/errors.test.ts`

**Interfaces:**
- Produces: `ChannelClosedError`, `IllegalTransitionError`, `SemaphoreReentrancyError` (used by channel, state-machine, semaphore)
- Produces: `StateHandler`, `TransitionHandler`, `Condition`, `SpeedMultiplier`, `RuntimeForEach` (used by state-machine, scenario-scope)

- [ ] **Step 1: Write error tests**

```typescript
import { describe, it, expect } from 'vitest';
import { ChannelClosedError, IllegalTransitionError, SemaphoreReentrancyError } from './errors.js';

describe('ChannelClosedError', () => {
  it('includes channel name in message', () => {
    const err = new ChannelClosedError('trades');
    expect(err.message).toContain('trades');
    expect(err.name).toBe('ChannelClosedError');
    expect(err).toBeInstanceOf(Error);
  });

  it('chains cause when provided', () => {
    const cause = new Error('upstream');
    const err = new ChannelClosedError('trades', cause);
    expect(err.cause).toBe(cause);
  });
});

describe('IllegalTransitionError', () => {
  it('includes machine name and states', () => {
    const err = new IllegalTransitionError('workflow', 'idle', 'complete');
    expect(err.message).toContain('workflow');
    expect(err.message).toContain('idle');
    expect(err.message).toContain('complete');
    expect(err.name).toBe('IllegalTransitionError');
  });
});

describe('SemaphoreReentrancyError', () => {
  it('includes name and step context', () => {
    const err = new SemaphoreReentrancyError('db-write', 'step-3');
    expect(err.message).toContain('db-write');
    expect(err.message).toContain('step-3');
    expect(err.name).toBe('SemaphoreReentrancyError');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/errors.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement errors.ts**

```typescript
export class ChannelClosedError extends Error {
  constructor(public readonly channelName: string, cause?: Error) {
    super(`Channel '${channelName}' is closed`);
    this.name = 'ChannelClosedError';
    if (cause) this.cause = cause;
  }
}

export class IllegalTransitionError extends Error {
  constructor(
    public readonly machineName: string,
    public readonly from: string,
    public readonly to: string,
  ) {
    super(`Illegal transition in '${machineName}': ${from} → ${to}`);
    this.name = 'IllegalTransitionError';
  }
}

export class SemaphoreReentrancyError extends Error {
  constructor(
    public readonly semaphoreName: string,
    public readonly stepContext: string,
  ) {
    super(`Semaphore '${semaphoreName}' reentrancy detected in '${stepContext}'`);
    this.name = 'SemaphoreReentrancyError';
  }
}
```

- [ ] **Step 4: Implement callbacks.ts** (no tests — type-only)

```typescript
export type StateHandler = () => void;

export type TransitionHandler = (payload: unknown) => void;

export type Condition = () => boolean;

export type SpeedMultiplier = () => number;

export type RuntimeForEach = () => unknown[];
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/errors.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/yaml-core/src/orchestration/errors.ts packages/yaml-core/src/orchestration/errors.test.ts packages/yaml-core/src/orchestration/callbacks.ts
git commit -m "feat(yaml-core): add orchestration error types and callback types Refs #462"
```

### Task 2: Duration parser

**Files:**
- Create: `packages/yaml-core/src/orchestration/duration-parser.ts`
- Test: `packages/yaml-core/src/orchestration/duration-parser.test.ts`

**Interfaces:**
- Produces: `parseDuration(input: string): number` (returns ms — used by directives parse functions)

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { parseDuration } from './duration-parser.js';

describe('parseDuration', () => {
  it('parses milliseconds', () => {
    expect(parseDuration('10ms')).toBe(10);
    expect(parseDuration('0ms')).toBe(0);
    expect(parseDuration('500ms')).toBe(500);
  });

  it('parses seconds', () => {
    expect(parseDuration('5s')).toBe(5000);
    expect(parseDuration('1s')).toBe(1000);
    expect(parseDuration('0.5s')).toBe(500);
  });

  it('parses minutes', () => {
    expect(parseDuration('3m')).toBe(180_000);
    expect(parseDuration('1m')).toBe(60_000);
  });

  it('parses hours', () => {
    expect(parseDuration('1h')).toBe(3_600_000);
    expect(parseDuration('2h')).toBe(7_200_000);
  });

  it('throws on invalid format', () => {
    expect(() => parseDuration('')).toThrow();
    expect(() => parseDuration('10')).toThrow();
    expect(() => parseDuration('abc')).toThrow();
    expect(() => parseDuration('10x')).toThrow();
  });

  it('throws on negative values', () => {
    expect(() => parseDuration('-5s')).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/duration-parser.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement duration-parser.ts**

```typescript
const UNITS: Record<string, number> = {
  ms: 1,
  s: 1000,
  m: 60_000,
  h: 3_600_000,
};

const DURATION_RE = /^(\d+(?:\.\d+)?)(ms|s|m|h)$/;

export function parseDuration(input: string): number {
  const match = DURATION_RE.exec(input);
  if (!match) {
    throw new Error(`Invalid duration: '${input}'. Expected format: <number><unit> where unit is ms|s|m|h`);
  }
  const value = parseFloat(match[1]);
  if (value < 0) {
    throw new Error(`Duration must not be negative: '${input}'`);
  }
  return Math.round(value * UNITS[match[2]]);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/duration-parser.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/duration-parser.ts packages/yaml-core/src/orchestration/duration-parser.test.ts
git commit -m "feat(yaml-core): add duration parser — ms/s/m/h to milliseconds Refs #462"
```

### Task 3: Directives — LoopDirective, RetryDirective, ComputeBlock, StepError

**Files:**
- Create: `packages/yaml-core/src/orchestration/directives.ts`
- Test: `packages/yaml-core/src/orchestration/directives.test.ts`

**Interfaces:**
- Consumes: `parseDuration` from `./duration-parser.js`
- Produces: `LoopDirective`, `parseLoopDirective(raw: unknown): LoopDirective`
- Produces: `RetryDirective`, `parseRetryDirective(raw: unknown): RetryDirective`
- Produces: `ComputeBlock`, `parseComputeBlock(map: Record<string, unknown>, defaultEngine: string): ComputeBlock`
- Produces: `StepError` (interface with `message`, `exceptionClass`, `stackTrace`)

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { parseLoopDirective, parseRetryDirective, parseComputeBlock } from './directives.js';
import type { StepError } from './directives.js';

describe('parseLoopDirective', () => {
  it('parses number as count', () => {
    expect(parseLoopDirective(5)).toEqual({ type: 'count', count: 5 });
  });

  it('parses object with count and until as count-until', () => {
    expect(parseLoopDirective({ count: 3, until: '${done}' })).toEqual({
      type: 'count-until', count: 3, until: '${done}',
    });
  });

  it('parses object with until only', () => {
    expect(parseLoopDirective({ until: '${complete}' })).toEqual({
      type: 'until', until: '${complete}',
    });
  });

  it('passes through LoopDirective objects', () => {
    const existing = { type: 'count' as const, count: 2 };
    expect(parseLoopDirective(existing)).toEqual(existing);
  });

  it('throws on invalid input', () => {
    expect(() => parseLoopDirective('bad')).toThrow();
    expect(() => parseLoopDirective(null)).toThrow();
    expect(() => parseLoopDirective({})).toThrow();
  });
});

describe('parseRetryDirective', () => {
  it('parses number as simple', () => {
    expect(parseRetryDirective(3)).toEqual({ type: 'simple', max: 3 });
  });

  it('parses object with backoff and delay as full', () => {
    expect(parseRetryDirective({ max: 3, backoff: 'exponential', delay: '1s' })).toEqual({
      type: 'full', max: 3, backoff: 'exponential', delayMs: 1000,
    });
  });

  it('parses object with max only as simple', () => {
    expect(parseRetryDirective({ max: 5 })).toEqual({ type: 'simple', max: 5 });
  });

  it('throws on invalid input', () => {
    expect(() => parseRetryDirective('bad')).toThrow();
    expect(() => parseRetryDirective(null)).toThrow();
  });
});

describe('parseComputeBlock', () => {
  it('parses engine and expression', () => {
    expect(parseComputeBlock({ engine: 'js', expression: '1 + 1' }, 'mvel')).toEqual({
      engine: 'js', expression: '1 + 1',
    });
  });

  it('uses default engine when not specified', () => {
    expect(parseComputeBlock({ expression: 'x > 0' }, 'mvel')).toEqual({
      engine: 'mvel', expression: 'x > 0',
    });
  });

  it('throws when expression is missing', () => {
    expect(() => parseComputeBlock({}, 'mvel')).toThrow();
  });
});

describe('StepError', () => {
  it('has required fields', () => {
    const err: StepError = { message: 'failed', exceptionClass: 'Error', stackTrace: 'at ...' };
    expect(err.message).toBe('failed');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/directives.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement directives.ts**

```typescript
import { parseDuration } from './duration-parser.js';

export interface StepError {
  readonly message: string;
  readonly exceptionClass: string;
  readonly stackTrace: string;
}

export type LoopDirective =
  | { type: 'count'; count: number }
  | { type: 'count-until'; count: number; until: string }
  | { type: 'until'; until: string };

export function parseLoopDirective(raw: unknown): LoopDirective {
  if (raw == null) throw new Error('LoopDirective cannot be null');
  if (typeof raw === 'number') return { type: 'count', count: raw };
  if (typeof raw === 'object' && !Array.isArray(raw)) {
    const map = raw as Record<string, unknown>;
    if ('type' in map) return map as LoopDirective;
    const count = map['count'] as number | undefined;
    const until = map['until'] as string | undefined;
    if (count != null && until != null) return { type: 'count-until', count, until };
    if (until != null) return { type: 'until', until };
    if (count != null) return { type: 'count', count };
  }
  throw new Error(`Invalid LoopDirective: expected number or {count?, until?}, got ${typeof raw}`);
}

export type RetryDirective =
  | { type: 'simple'; max: number }
  | { type: 'full'; max: number; backoff: string; delayMs: number };

export function parseRetryDirective(raw: unknown): RetryDirective {
  if (raw == null) throw new Error('RetryDirective cannot be null');
  if (typeof raw === 'number') return { type: 'simple', max: raw };
  if (typeof raw === 'object' && !Array.isArray(raw)) {
    const map = raw as Record<string, unknown>;
    if ('type' in map) return map as RetryDirective;
    const max = map['max'] as number | undefined;
    if (max == null) throw new Error('RetryDirective requires max');
    const backoff = map['backoff'] as string | undefined;
    const delay = map['delay'] as string | undefined;
    if (backoff && delay) return { type: 'full', max, backoff, delayMs: parseDuration(delay) };
    return { type: 'simple', max };
  }
  throw new Error(`Invalid RetryDirective: expected number or {max, backoff?, delay?}, got ${typeof raw}`);
}

export interface ComputeBlock {
  readonly engine: string;
  readonly expression: string;
}

export function parseComputeBlock(map: Record<string, unknown>, defaultEngine: string): ComputeBlock {
  const expression = map['expression'] as string | undefined;
  if (!expression) throw new Error('ComputeBlock requires an expression');
  const engine = (map['engine'] as string | undefined) ?? defaultEngine;
  return { engine, expression };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/directives.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/directives.ts packages/yaml-core/src/orchestration/directives.test.ts
git commit -m "feat(yaml-core): add LoopDirective, RetryDirective, ComputeBlock, StepError Refs #462"
```

---

## Batch 2: Core primitives — signal, latch, semaphore

Independent primitives with no cross-dependencies. Each is self-contained
with its own interface and default implementation.

### Task 4: OrcSignal

**Files:**
- Create: `packages/yaml-core/src/orchestration/types.ts`
- Create: `packages/yaml-core/src/orchestration/signal.ts`
- Test: `packages/yaml-core/src/orchestration/signal.test.ts`

**Interfaces:**
- Consumes: `TransitionHandler`, `StateHandler` from `./callbacks.js`
- Produces: `OrcSignal` interface (used by ScenarioScope)
- Produces: `DefaultOrcSignal` class

Note: `types.ts` is created here with ALL interfaces (OrcSignal, OrcSemaphore,
OrcLatch, OrcChannel, OrcStateMachine, ScenarioScope, StepResultStore) so
subsequent tasks can import from it. Only OrcSignal is implemented in this task.

- [ ] **Step 1: Create types.ts with all interfaces**

```typescript
import type { StateHandler, TransitionHandler } from './callbacks.js';

export interface OrcSignal {
  signal(payload?: unknown): void;
  await(): Promise<void>;
  await(timeoutMs: number): Promise<boolean>;
  payload(): unknown;
  isSignalled(): boolean;
}

export interface OrcSemaphore {
  acquire(): Promise<void>;
  tryAcquire(timeoutMs: number): Promise<boolean>;
  release(): void;
  availablePermits(): number;
}

export interface OrcLatch {
  countDown(): void;
  await(): Promise<void>;
  await(timeoutMs: number): Promise<boolean>;
  getCount(): number;
}

export interface OrcChannel<T> {
  send(value: T): Promise<void>;
  send(value: T, timeoutMs: number): Promise<boolean>;
  receive(): Promise<T>;
  receive(timeoutMs: number): Promise<T | undefined>;
  isEmpty(): boolean;
  close(): void;
  close(cause: Error): void;
  isErrorClosed(): boolean;
  closeError(): Error | undefined;
}

export interface OrcStateMachine<S extends string> {
  currentState(): S;
  transition(from: S, to: S, payload?: unknown): boolean;
  onTransition(from: S, to: S, handler: TransitionHandler): void;
  onEnter(state: S, handler: StateHandler): void;
  onExit(state: S, handler: StateHandler): void;
}

export interface StepResultStore {
  recordSuccess(stepName: string, result: Record<string, unknown>): void;
  recordFailure(stepName: string, error: import('./directives.js').StepError): void;
  result(stepName: string): Record<string, unknown> | undefined;
  error(stepName: string): import('./directives.js').StepError | undefined;
  hasCompleted(stepName: string): boolean;
}

export interface ScenarioScope {
  semaphore(name: string, permits: number): OrcSemaphore;
  latch(name: string, count: number): OrcLatch;
  signal(name: string): OrcSignal;
  channel<T>(name: string, capacity?: number): OrcChannel<T>;
  stateMachine<S extends string>(name: string, states: readonly S[], initial: S): OrcStateMachine<S>;
  primitive<T>(name: string, type: new (...args: unknown[]) => T): T;
  resultStore(): StepResultStore;
  close(): void;
}
```

- [ ] **Step 2: Write failing signal tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultOrcSignal } from './signal.js';

describe('DefaultOrcSignal', () => {
  describe('one-shot mode', () => {
    it('resolves waiters on signal', async () => {
      const s = new DefaultOrcSignal();
      const p = s.await();
      s.signal('data');
      await p;
      expect(s.payload()).toBe('data');
    });

    it('resolves immediately if already signalled', async () => {
      const s = new DefaultOrcSignal();
      s.signal();
      await s.await();
      expect(s.isSignalled()).toBe(true);
    });

    it('resolves multiple waiters', async () => {
      const s = new DefaultOrcSignal();
      const results: string[] = [];
      const p1 = s.await().then(() => results.push('a'));
      const p2 = s.await().then(() => results.push('b'));
      s.signal();
      await Promise.all([p1, p2]);
      expect(results).toHaveLength(2);
    });

    it('timeout returns false when not signalled', async () => {
      const s = new DefaultOrcSignal();
      const result = s.await(10);
      expect(await result).toBe(false);
    });

    it('timeout returns true when signalled before expiry', async () => {
      const s = new DefaultOrcSignal();
      const p = s.await(1000);
      s.signal();
      expect(await p).toBe(true);
    });
  });

  describe('repeatable mode', () => {
    it('resets after signal — next await waits again', async () => {
      const s = new DefaultOrcSignal(true);
      const p1 = s.await();
      s.signal('first');
      await p1;
      expect(s.isSignalled()).toBe(false);

      const p2 = s.await();
      s.signal('second');
      await p2;
      expect(s.payload()).toBe('second');
    });
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/signal.test.ts`
Expected: FAIL

- [ ] **Step 4: Implement signal.ts**

```typescript
import type { OrcSignal } from './types.js';

interface Waiter {
  resolve: (timedOut: boolean) => void;
  timer?: ReturnType<typeof setTimeout>;
}

export class DefaultOrcSignal implements OrcSignal {
  private _signalled = false;
  private _payload: unknown = undefined;
  private readonly _repeatable: boolean;
  private _waiters: Waiter[] = [];

  constructor(repeatable = false) {
    this._repeatable = repeatable;
  }

  signal(payload?: unknown): void {
    this._payload = payload;
    this._signalled = true;
    const waiters = this._waiters;
    this._waiters = [];
    for (const w of waiters) {
      if (w.timer) clearTimeout(w.timer);
      w.resolve(true);
    }
    if (this._repeatable) {
      this._signalled = false;
    }
  }

  await(): Promise<void>;
  await(timeoutMs: number): Promise<boolean>;
  await(timeoutMs?: number): Promise<void | boolean> {
    if (this._signalled && !this._repeatable) {
      return timeoutMs !== undefined ? Promise.resolve(true) : Promise.resolve();
    }
    return new Promise<boolean>((resolve) => {
      const waiter: Waiter = {
        resolve: (val) => resolve(val),
      };
      if (timeoutMs !== undefined) {
        waiter.timer = setTimeout(() => {
          const idx = this._waiters.indexOf(waiter);
          if (idx >= 0) this._waiters.splice(idx, 1);
          resolve(false);
        }, timeoutMs);
      }
      this._waiters.push(waiter);
    }).then((result) => (timeoutMs !== undefined ? result : undefined)) as Promise<void | boolean>;
  }

  payload(): unknown {
    return this._payload;
  }

  isSignalled(): boolean {
    return this._signalled;
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/signal.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/yaml-core/src/orchestration/types.ts packages/yaml-core/src/orchestration/signal.ts packages/yaml-core/src/orchestration/signal.test.ts
git commit -m "feat(yaml-core): add OrcSignal — one-shot and repeatable modes Refs #462"
```

### Task 5: OrcLatch

**Files:**
- Create: `packages/yaml-core/src/orchestration/latch.ts`
- Test: `packages/yaml-core/src/orchestration/latch.test.ts`

**Interfaces:**
- Consumes: `OrcLatch` from `./types.js`
- Produces: `DefaultOrcLatch` class

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultOrcLatch } from './latch.js';

describe('DefaultOrcLatch', () => {
  it('resolves when count reaches zero', async () => {
    const latch = new DefaultOrcLatch(2);
    const p = latch.await();
    latch.countDown();
    expect(latch.getCount()).toBe(1);
    latch.countDown();
    expect(latch.getCount()).toBe(0);
    await p;
  });

  it('resolves immediately if count is already zero', async () => {
    const latch = new DefaultOrcLatch(0);
    await latch.await();
  });

  it('resolves all waiters at once', async () => {
    const latch = new DefaultOrcLatch(1);
    const results: number[] = [];
    const p1 = latch.await().then(() => results.push(1));
    const p2 = latch.await().then(() => results.push(2));
    latch.countDown();
    await Promise.all([p1, p2]);
    expect(results).toHaveLength(2);
  });

  it('ignores countDown below zero', () => {
    const latch = new DefaultOrcLatch(1);
    latch.countDown();
    latch.countDown();
    expect(latch.getCount()).toBe(0);
  });

  it('timeout returns false when latch not released', async () => {
    const latch = new DefaultOrcLatch(1);
    expect(await latch.await(10)).toBe(false);
  });

  it('timeout returns true when released before expiry', async () => {
    const latch = new DefaultOrcLatch(1);
    const p = latch.await(1000);
    latch.countDown();
    expect(await p).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/latch.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement latch.ts**

```typescript
import type { OrcLatch } from './types.js';

interface Waiter {
  resolve: (timedOut: boolean) => void;
  timer?: ReturnType<typeof setTimeout>;
}

export class DefaultOrcLatch implements OrcLatch {
  private _count: number;
  private _waiters: Waiter[] = [];

  constructor(count: number) {
    this._count = Math.max(0, count);
  }

  countDown(): void {
    if (this._count <= 0) return;
    this._count--;
    if (this._count === 0) {
      const waiters = this._waiters;
      this._waiters = [];
      for (const w of waiters) {
        if (w.timer) clearTimeout(w.timer);
        w.resolve(true);
      }
    }
  }

  await(): Promise<void>;
  await(timeoutMs: number): Promise<boolean>;
  await(timeoutMs?: number): Promise<void | boolean> {
    if (this._count === 0) {
      return timeoutMs !== undefined ? Promise.resolve(true) : Promise.resolve();
    }
    return new Promise<boolean>((resolve) => {
      const waiter: Waiter = { resolve };
      if (timeoutMs !== undefined) {
        waiter.timer = setTimeout(() => {
          const idx = this._waiters.indexOf(waiter);
          if (idx >= 0) this._waiters.splice(idx, 1);
          resolve(false);
        }, timeoutMs);
      }
      this._waiters.push(waiter);
    }).then((result) => (timeoutMs !== undefined ? result : undefined)) as Promise<void | boolean>;
  }

  getCount(): number {
    return this._count;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/latch.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/latch.ts packages/yaml-core/src/orchestration/latch.test.ts
git commit -m "feat(yaml-core): add OrcLatch — countdown barrier with await Refs #462"
```

### Task 6: OrcSemaphore

**Files:**
- Create: `packages/yaml-core/src/orchestration/semaphore.ts`
- Test: `packages/yaml-core/src/orchestration/semaphore.test.ts`

**Interfaces:**
- Consumes: `OrcSemaphore` from `./types.js`
- Produces: `DefaultOrcSemaphore` class

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultOrcSemaphore } from './semaphore.js';

describe('DefaultOrcSemaphore', () => {
  it('acquire succeeds immediately when permits available', async () => {
    const sem = new DefaultOrcSemaphore(2);
    await sem.acquire();
    expect(sem.availablePermits()).toBe(1);
    await sem.acquire();
    expect(sem.availablePermits()).toBe(0);
  });

  it('acquire blocks when no permits, unblocks on release', async () => {
    const sem = new DefaultOrcSemaphore(1);
    await sem.acquire();
    let acquired = false;
    const p = sem.acquire().then(() => { acquired = true; });
    expect(acquired).toBe(false);
    sem.release();
    await p;
    expect(acquired).toBe(true);
  });

  it('FIFO ordering — first waiter gets next permit', async () => {
    const sem = new DefaultOrcSemaphore(1);
    await sem.acquire();
    const order: number[] = [];
    const p1 = sem.acquire().then(() => order.push(1));
    const p2 = sem.acquire().then(() => order.push(2));
    sem.release();
    sem.release();
    await Promise.all([p1, p2]);
    expect(order).toEqual([1, 2]);
  });

  it('release without prior acquire increases permits', () => {
    const sem = new DefaultOrcSemaphore(1);
    sem.release();
    expect(sem.availablePermits()).toBe(2);
  });

  it('tryAcquire returns false on timeout', async () => {
    const sem = new DefaultOrcSemaphore(0);
    expect(await sem.tryAcquire(10)).toBe(false);
  });

  it('tryAcquire returns true when permit becomes available', async () => {
    const sem = new DefaultOrcSemaphore(0);
    const p = sem.tryAcquire(1000);
    sem.release();
    expect(await p).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/semaphore.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement semaphore.ts**

```typescript
import type { OrcSemaphore } from './types.js';

interface Waiter {
  resolve: (acquired: boolean) => void;
  timer?: ReturnType<typeof setTimeout>;
}

export class DefaultOrcSemaphore implements OrcSemaphore {
  private _permits: number;
  private _waiters: Waiter[] = [];

  constructor(permits: number) {
    this._permits = permits;
  }

  acquire(): Promise<void> {
    if (this._permits > 0) {
      this._permits--;
      return Promise.resolve();
    }
    return new Promise<boolean>((resolve) => {
      this._waiters.push({ resolve });
    }).then(() => undefined);
  }

  tryAcquire(timeoutMs: number): Promise<boolean> {
    if (this._permits > 0) {
      this._permits--;
      return Promise.resolve(true);
    }
    return new Promise<boolean>((resolve) => {
      const waiter: Waiter = { resolve };
      waiter.timer = setTimeout(() => {
        const idx = this._waiters.indexOf(waiter);
        if (idx >= 0) this._waiters.splice(idx, 1);
        resolve(false);
      }, timeoutMs);
      this._waiters.push(waiter);
    });
  }

  release(): void {
    const waiter = this._waiters.shift();
    if (waiter) {
      if (waiter.timer) clearTimeout(waiter.timer);
      waiter.resolve(true);
    } else {
      this._permits++;
    }
  }

  availablePermits(): number {
    return this._permits;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/semaphore.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/semaphore.ts packages/yaml-core/src/orchestration/semaphore.test.ts
git commit -m "feat(yaml-core): add OrcSemaphore — FIFO permit queue Refs #462"
```

---

## Batch 3: Channel, state machine, result store

More complex primitives. Channel has bounded/unbounded modes and
error-close. State machine has builder pattern and guarded transitions.

### Task 7: OrcChannel

**Files:**
- Create: `packages/yaml-core/src/orchestration/channel.ts`
- Test: `packages/yaml-core/src/orchestration/channel.test.ts`

**Interfaces:**
- Consumes: `OrcChannel` from `./types.js`, `ChannelClosedError` from `./errors.js`
- Produces: `DefaultOrcChannel` class

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultOrcChannel } from './channel.js';
import { ChannelClosedError } from './errors.js';

describe('DefaultOrcChannel', () => {
  describe('unbounded', () => {
    it('send then receive', async () => {
      const ch = new DefaultOrcChannel<number>();
      await ch.send(42);
      expect(await ch.receive()).toBe(42);
    });

    it('receive blocks until send', async () => {
      const ch = new DefaultOrcChannel<string>();
      let received: string | undefined;
      const p = ch.receive().then((v) => { received = v; });
      expect(received).toBeUndefined();
      await ch.send('hello');
      await p;
      expect(received).toBe('hello');
    });

    it('FIFO ordering', async () => {
      const ch = new DefaultOrcChannel<number>();
      await ch.send(1);
      await ch.send(2);
      await ch.send(3);
      expect(await ch.receive()).toBe(1);
      expect(await ch.receive()).toBe(2);
      expect(await ch.receive()).toBe(3);
    });

    it('isEmpty reflects buffer state', async () => {
      const ch = new DefaultOrcChannel<number>();
      expect(ch.isEmpty()).toBe(true);
      await ch.send(1);
      expect(ch.isEmpty()).toBe(false);
      await ch.receive();
      expect(ch.isEmpty()).toBe(true);
    });
  });

  describe('bounded', () => {
    it('send blocks when buffer full', async () => {
      const ch = new DefaultOrcChannel<number>(1);
      await ch.send(1);
      let sent = false;
      const p = ch.send(2).then(() => { sent = true; });
      expect(sent).toBe(false);
      await ch.receive();
      await p;
      expect(sent).toBe(true);
    });

    it('send with timeout returns false when full', async () => {
      const ch = new DefaultOrcChannel<number>(1);
      await ch.send(1);
      expect(await ch.send(2, 10)).toBe(false);
    });
  });

  describe('close', () => {
    it('close rejects pending receivers', async () => {
      const ch = new DefaultOrcChannel<number>();
      const p = ch.receive();
      ch.close();
      await expect(p).rejects.toThrow(ChannelClosedError);
    });

    it('receive after close throws', async () => {
      const ch = new DefaultOrcChannel<number>();
      ch.close();
      await expect(ch.receive()).rejects.toThrow(ChannelClosedError);
    });

    it('error close stores cause', () => {
      const ch = new DefaultOrcChannel<number>();
      const cause = new Error('upstream');
      ch.close(cause);
      expect(ch.isErrorClosed()).toBe(true);
      expect(ch.closeError()).toBe(cause);
    });

    it('receive with timeout returns undefined on timeout', async () => {
      const ch = new DefaultOrcChannel<number>();
      expect(await ch.receive(10)).toBeUndefined();
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/channel.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement channel.ts**

```typescript
import type { OrcChannel } from './types.js';
import { ChannelClosedError } from './errors.js';

interface RecvWaiter<T> {
  resolve: (value: T) => void;
  reject: (error: Error) => void;
  timer?: ReturnType<typeof setTimeout>;
}

interface SendWaiter<T> {
  value: T;
  resolve: (sent: boolean) => void;
  timer?: ReturnType<typeof setTimeout>;
}

export class DefaultOrcChannel<T> implements OrcChannel<T> {
  private _buffer: T[] = [];
  private _recvWaiters: RecvWaiter<T>[] = [];
  private _sendWaiters: SendWaiter<T>[] = [];
  private _closed = false;
  private _closeError?: Error;
  private readonly _capacity: number | undefined;

  constructor(capacity?: number) {
    this._capacity = capacity;
  }

  send(value: T): Promise<void>;
  send(value: T, timeoutMs: number): Promise<boolean>;
  send(value: T, timeoutMs?: number): Promise<void | boolean> {
    if (this._closed) {
      const err = new ChannelClosedError('channel', this._closeError);
      return timeoutMs !== undefined ? Promise.resolve(false) : Promise.reject(err);
    }
    const receiver = this._recvWaiters.shift();
    if (receiver) {
      if (receiver.timer) clearTimeout(receiver.timer);
      receiver.resolve(value);
      return timeoutMs !== undefined ? Promise.resolve(true) : Promise.resolve();
    }
    if (this._capacity === undefined || this._buffer.length < this._capacity) {
      this._buffer.push(value);
      return timeoutMs !== undefined ? Promise.resolve(true) : Promise.resolve();
    }
    return new Promise<boolean>((resolve) => {
      const waiter: SendWaiter<T> = { value, resolve };
      if (timeoutMs !== undefined) {
        waiter.timer = setTimeout(() => {
          const idx = this._sendWaiters.indexOf(waiter);
          if (idx >= 0) this._sendWaiters.splice(idx, 1);
          resolve(false);
        }, timeoutMs);
      }
      this._sendWaiters.push(waiter);
    }).then((result) => (timeoutMs !== undefined ? result : undefined)) as Promise<void | boolean>;
  }

  receive(): Promise<T>;
  receive(timeoutMs: number): Promise<T | undefined>;
  receive(timeoutMs?: number): Promise<T | undefined> {
    if (this._buffer.length > 0) {
      const value = this._buffer.shift()!;
      const sender = this._sendWaiters.shift();
      if (sender) {
        if (sender.timer) clearTimeout(sender.timer);
        this._buffer.push(sender.value);
        sender.resolve(true);
      }
      return Promise.resolve(value);
    }
    if (this._closed) {
      const err = new ChannelClosedError('channel', this._closeError);
      return timeoutMs !== undefined ? Promise.resolve(undefined) : Promise.reject(err);
    }
    return new Promise<T>((resolve, reject) => {
      const waiter: RecvWaiter<T> = { resolve, reject };
      if (timeoutMs !== undefined) {
        waiter.timer = setTimeout(() => {
          const idx = this._recvWaiters.indexOf(waiter);
          if (idx >= 0) this._recvWaiters.splice(idx, 1);
          resolve(undefined as T);
        }, timeoutMs);
      }
      this._recvWaiters.push(waiter);
    });
  }

  isEmpty(): boolean {
    return this._buffer.length === 0;
  }

  close(): void;
  close(cause: Error): void;
  close(cause?: Error): void {
    this._closed = true;
    if (cause) this._closeError = cause;
    const err = new ChannelClosedError('channel', cause);
    for (const w of this._recvWaiters) {
      if (w.timer) clearTimeout(w.timer);
      w.reject(err);
    }
    this._recvWaiters = [];
    for (const w of this._sendWaiters) {
      if (w.timer) clearTimeout(w.timer);
      w.resolve(false);
    }
    this._sendWaiters = [];
  }

  isErrorClosed(): boolean {
    return this._closed && this._closeError !== undefined;
  }

  closeError(): Error | undefined {
    return this._closeError;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/channel.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/channel.ts packages/yaml-core/src/orchestration/channel.test.ts
git commit -m "feat(yaml-core): add OrcChannel — bounded/unbounded with close semantics Refs #462"
```

### Task 8: OrcStateMachine

**Files:**
- Create: `packages/yaml-core/src/orchestration/state-machine.ts`
- Test: `packages/yaml-core/src/orchestration/state-machine.test.ts`

**Interfaces:**
- Consumes: `OrcStateMachine` from `./types.js`, `StateHandler`, `TransitionHandler` from `./callbacks.js`, `IllegalTransitionError` from `./errors.js`
- Produces: `DefaultOrcStateMachine` class, `StateMachineBuilder` class

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { StateMachineBuilder } from './state-machine.js';
import { IllegalTransitionError } from './errors.js';

describe('DefaultOrcStateMachine', () => {
  const buildSimple = () =>
    new StateMachineBuilder<'idle' | 'running' | 'done'>('test', 'idle')
      .transition('idle', 'running')
      .transition('running', 'done')
      .terminal('done')
      .build();

  it('starts in initial state', () => {
    const sm = buildSimple();
    expect(sm.currentState()).toBe('idle');
  });

  it('transitions between valid states', () => {
    const sm = buildSimple();
    expect(sm.transition('idle', 'running')).toBe(true);
    expect(sm.currentState()).toBe('running');
  });

  it('rejects transition from wrong current state', () => {
    const sm = buildSimple();
    expect(sm.transition('running', 'done')).toBe(false);
    expect(sm.currentState()).toBe('idle');
  });

  it('rejects undeclared transition', () => {
    const sm = buildSimple();
    expect(() => sm.transition('idle', 'done')).toThrow(IllegalTransitionError);
  });

  it('rejects transition from terminal state', () => {
    const sm = buildSimple();
    sm.transition('idle', 'running');
    sm.transition('running', 'done');
    expect(() => sm.transition('done', 'idle')).toThrow(IllegalTransitionError);
  });

  it('fires onEnter and onExit handlers', () => {
    const log: string[] = [];
    const sm = new StateMachineBuilder<'a' | 'b'>('test', 'a')
      .transition('a', 'b')
      .build();
    sm.onExit('a', () => log.push('exit-a'));
    sm.onEnter('b', () => log.push('enter-b'));
    sm.transition('a', 'b');
    expect(log).toEqual(['exit-a', 'enter-b']);
  });

  it('fires onTransition handler with payload', () => {
    let captured: unknown;
    const sm = new StateMachineBuilder<'a' | 'b'>('test', 'a')
      .transition('a', 'b')
      .build();
    sm.onTransition('a', 'b', (payload) => { captured = payload; });
    sm.transition('a', 'b', 'my-data');
    expect(captured).toBe('my-data');
  });

  it('guard blocks transition when predicate returns false', () => {
    const sm = new StateMachineBuilder<'a' | 'b'>('test', 'a')
      .transition('a', 'b')
      .guard('a', 'b', () => false)
      .build();
    expect(sm.transition('a', 'b')).toBe(false);
    expect(sm.currentState()).toBe('a');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/state-machine.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement state-machine.ts**

```typescript
import type { OrcStateMachine } from './types.js';
import type { StateHandler, TransitionHandler } from './callbacks.js';
import { IllegalTransitionError } from './errors.js';

export class DefaultOrcStateMachine<S extends string> implements OrcStateMachine<S> {
  private _state: S;
  private readonly _name: string;
  private readonly _transitions: Map<S, Set<S>>;
  private readonly _guards: Map<string, (payload: unknown) => boolean>;
  private readonly _terminalStates: Set<S>;
  private readonly _enterHandlers = new Map<S, StateHandler[]>();
  private readonly _exitHandlers = new Map<S, StateHandler[]>();
  private readonly _transitionHandlers = new Map<string, TransitionHandler[]>();

  constructor(
    name: string,
    initial: S,
    transitions: Map<S, Set<S>>,
    guards: Map<string, (payload: unknown) => boolean>,
    terminalStates: Set<S>,
  ) {
    this._name = name;
    this._state = initial;
    this._transitions = transitions;
    this._guards = guards;
    this._terminalStates = terminalStates;
  }

  currentState(): S {
    return this._state;
  }

  transition(from: S, to: S, payload?: unknown): boolean {
    if (this._state !== from) return false;
    if (this._terminalStates.has(from)) {
      throw new IllegalTransitionError(this._name, from, to);
    }
    const targets = this._transitions.get(from);
    if (!targets || !targets.has(to)) {
      throw new IllegalTransitionError(this._name, from, to);
    }
    const guardKey = `${from}->${to}`;
    const guard = this._guards.get(guardKey);
    if (guard && !guard(payload)) return false;
    const exitHandlers = this._exitHandlers.get(from) ?? [];
    for (const h of exitHandlers) h();
    this._state = to;
    const transHandlers = this._transitionHandlers.get(guardKey) ?? [];
    for (const h of transHandlers) h(payload);
    const enterHandlers = this._enterHandlers.get(to) ?? [];
    for (const h of enterHandlers) h();
    return true;
  }

  onTransition(from: S, to: S, handler: TransitionHandler): void {
    const key = `${from}->${to}`;
    const list = this._transitionHandlers.get(key) ?? [];
    list.push(handler);
    this._transitionHandlers.set(key, list);
  }

  onEnter(state: S, handler: StateHandler): void {
    const list = this._enterHandlers.get(state) ?? [];
    list.push(handler);
    this._enterHandlers.set(state, list);
  }

  onExit(state: S, handler: StateHandler): void {
    const list = this._exitHandlers.get(state) ?? [];
    list.push(handler);
    this._exitHandlers.set(state, list);
  }
}

export class StateMachineBuilder<S extends string> {
  private readonly _name: string;
  private readonly _initial: S;
  private readonly _transitions = new Map<S, Set<S>>();
  private readonly _guards = new Map<string, (payload: unknown) => boolean>();
  private readonly _terminalStates = new Set<S>();

  constructor(name: string, initial: S) {
    this._name = name;
    this._initial = initial;
  }

  transition(from: S, to: S): this {
    let targets = this._transitions.get(from);
    if (!targets) {
      targets = new Set();
      this._transitions.set(from, targets);
    }
    targets.add(to);
    return this;
  }

  guard(from: S, to: S, predicate: (payload: unknown) => boolean): this {
    this._guards.set(`${from}->${to}`, predicate);
    return this;
  }

  terminal(...states: S[]): this {
    for (const s of states) this._terminalStates.add(s);
    return this;
  }

  build(): DefaultOrcStateMachine<S> {
    return new DefaultOrcStateMachine(
      this._name, this._initial, this._transitions, this._guards, this._terminalStates,
    );
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/state-machine.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/state-machine.ts packages/yaml-core/src/orchestration/state-machine.test.ts
git commit -m "feat(yaml-core): add OrcStateMachine — builder pattern with guards and handlers Refs #462"
```

### Task 9: StepResultStore

**Files:**
- Create: `packages/yaml-core/src/orchestration/step-result-store.ts`
- Test: `packages/yaml-core/src/orchestration/step-result-store.test.ts`

**Interfaces:**
- Consumes: `StepResultStore` from `./types.js`, `StepError` from `./directives.js`
- Produces: `DefaultStepResultStore` class

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultStepResultStore } from './step-result-store.js';

describe('DefaultStepResultStore', () => {
  it('records and retrieves success', () => {
    const store = new DefaultStepResultStore();
    store.recordSuccess('step-1', { count: 42 });
    expect(store.result('step-1')).toEqual({ count: 42 });
    expect(store.hasCompleted('step-1')).toBe(true);
    expect(store.error('step-1')).toBeUndefined();
  });

  it('records and retrieves failure', () => {
    const store = new DefaultStepResultStore();
    const err = { message: 'boom', exceptionClass: 'Error', stackTrace: 'at ...' };
    store.recordFailure('step-2', err);
    expect(store.error('step-2')).toEqual(err);
    expect(store.hasCompleted('step-2')).toBe(true);
    expect(store.result('step-2')).toBeUndefined();
  });

  it('returns undefined for unknown steps', () => {
    const store = new DefaultStepResultStore();
    expect(store.result('nope')).toBeUndefined();
    expect(store.error('nope')).toBeUndefined();
    expect(store.hasCompleted('nope')).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/step-result-store.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement step-result-store.ts**

```typescript
import type { StepResultStore } from './types.js';
import type { StepError } from './directives.js';

export class DefaultStepResultStore implements StepResultStore {
  private readonly _results = new Map<string, Record<string, unknown>>();
  private readonly _errors = new Map<string, StepError>();
  private readonly _completed = new Set<string>();

  recordSuccess(stepName: string, result: Record<string, unknown>): void {
    this._results.set(stepName, result);
    this._completed.add(stepName);
  }

  recordFailure(stepName: string, error: StepError): void {
    this._errors.set(stepName, error);
    this._completed.add(stepName);
  }

  result(stepName: string): Record<string, unknown> | undefined {
    return this._results.get(stepName);
  }

  error(stepName: string): StepError | undefined {
    return this._errors.get(stepName);
  }

  hasCompleted(stepName: string): boolean {
    return this._completed.has(stepName);
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/step-result-store.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/step-result-store.ts packages/yaml-core/src/orchestration/step-result-store.test.ts
git commit -m "feat(yaml-core): add StepResultStore — success/failure tracking Refs #462"
```

---

## Batch 4: ScenarioScope, utilities, barrel export, package wiring

The integration layer — ScenarioScope ties all primitives together.
Utilities and condition evaluator complete the port. Barrel export
and package.json wiring make everything consumable.

### Task 10: ScenarioScope

**Files:**
- Create: `packages/yaml-core/src/orchestration/scenario-scope.ts`
- Test: `packages/yaml-core/src/orchestration/scenario-scope.test.ts`

**Interfaces:**
- Consumes: `ScenarioScope` from `./types.js`, all Default* implementations
- Produces: `DefaultScenarioScope` class

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultScenarioScope } from './scenario-scope.js';

describe('DefaultScenarioScope', () => {
  it('creates semaphore by name', () => {
    const scope = new DefaultScenarioScope();
    const sem = scope.semaphore('db', 3);
    expect(sem.availablePermits()).toBe(3);
  });

  it('get-or-create — same name returns same instance', () => {
    const scope = new DefaultScenarioScope();
    const s1 = scope.signal('go');
    const s2 = scope.signal('go');
    expect(s1).toBe(s2);
  });

  it('creates latch by name', () => {
    const scope = new DefaultScenarioScope();
    const latch = scope.latch('barrier', 2);
    expect(latch.getCount()).toBe(2);
  });

  it('creates channel by name', () => {
    const scope = new DefaultScenarioScope();
    const ch = scope.channel<number>('trades');
    expect(ch.isEmpty()).toBe(true);
  });

  it('creates bounded channel', async () => {
    const scope = new DefaultScenarioScope();
    const ch = scope.channel<number>('bounded', 1);
    await ch.send(1);
    expect(ch.isEmpty()).toBe(false);
  });

  it('creates state machine by name', () => {
    const scope = new DefaultScenarioScope();
    const sm = scope.stateMachine('wf', ['idle', 'active', 'done'] as const, 'idle');
    expect(sm.currentState()).toBe('idle');
  });

  it('resultStore returns singleton', () => {
    const scope = new DefaultScenarioScope();
    expect(scope.resultStore()).toBe(scope.resultStore());
  });

  it('close cascades — channels closed, signals fired', async () => {
    const scope = new DefaultScenarioScope();
    const ch = scope.channel<number>('test');
    const sig = scope.signal('gate');
    scope.close();
    await expect(ch.receive()).rejects.toThrow();
    expect(sig.isSignalled()).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/orchestration/scenario-scope.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement scenario-scope.ts**

```typescript
import type {
  ScenarioScope, OrcSemaphore, OrcLatch, OrcSignal, OrcChannel,
  OrcStateMachine, StepResultStore,
} from './types.js';
import { DefaultOrcSemaphore } from './semaphore.js';
import { DefaultOrcLatch } from './latch.js';
import { DefaultOrcSignal } from './signal.js';
import { DefaultOrcChannel } from './channel.js';
import { StateMachineBuilder } from './state-machine.js';
import { DefaultStepResultStore } from './step-result-store.js';

export class DefaultScenarioScope implements ScenarioScope {
  private readonly _primitives = new Map<string, unknown>();
  private _resultStore?: DefaultStepResultStore;

  semaphore(name: string, permits: number): OrcSemaphore {
    return this._getOrCreate(name, () => new DefaultOrcSemaphore(permits));
  }

  latch(name: string, count: number): OrcLatch {
    return this._getOrCreate(name, () => new DefaultOrcLatch(count));
  }

  signal(name: string): OrcSignal {
    return this._getOrCreate(name, () => new DefaultOrcSignal());
  }

  channel<T>(name: string, capacity?: number): OrcChannel<T> {
    return this._getOrCreate(name, () => new DefaultOrcChannel<T>(capacity));
  }

  stateMachine<S extends string>(
    name: string, states: readonly S[], initial: S,
  ): OrcStateMachine<S> {
    return this._getOrCreate(name, () => {
      const builder = new StateMachineBuilder<S>(name, initial);
      for (let i = 0; i < states.length - 1; i++) {
        for (let j = i + 1; j < states.length; j++) {
          builder.transition(states[i], states[j]);
          builder.transition(states[j], states[i]);
        }
      }
      return builder.build();
    });
  }

  primitive<T>(name: string, type: new (...args: unknown[]) => T): T {
    return this._getOrCreate(name, () => new type());
  }

  resultStore(): StepResultStore {
    if (!this._resultStore) this._resultStore = new DefaultStepResultStore();
    return this._resultStore;
  }

  close(): void {
    for (const [, prim] of this._primitives) {
      if (prim && typeof prim === 'object') {
        if ('close' in prim && typeof (prim as { close: unknown }).close === 'function') {
          (prim as { close(): void }).close();
        }
        if ('signal' in prim && typeof (prim as { signal: unknown }).signal === 'function') {
          (prim as OrcSignal).signal();
        }
      }
    }
    this._primitives.clear();
  }

  private _getOrCreate<T>(name: string, factory: () => T): T {
    let existing = this._primitives.get(name) as T | undefined;
    if (existing === undefined) {
      existing = factory();
      this._primitives.set(name, existing);
    }
    return existing;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/orchestration/scenario-scope.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/yaml-core/src/orchestration/scenario-scope.ts packages/yaml-core/src/orchestration/scenario-scope.test.ts
git commit -m "feat(yaml-core): add ScenarioScope — factory container with close cascade Refs #462"
```

### Task 11: ConditionEvaluator and VariablePrefixRewriter

**Files:**
- Create: `packages/yaml-core/src/condition/condition-evaluator.ts`
- Create: `packages/yaml-core/src/orchestration/variable-prefix-rewriter.ts`
- Test: `packages/yaml-core/src/condition/condition-evaluator.test.ts`
- Test: `packages/yaml-core/src/orchestration/variable-prefix-rewriter.test.ts`

**Interfaces:**
- Consumes: `isTruthy` from `../truthiness.js`
- Produces: `ConditionEvaluator` class
- Produces: `rewriteVariablePrefixes(input, defaultPrefix, knownPrefixes, forEachVars): string`

- [ ] **Step 1: Write failing condition evaluator tests**

```typescript
import { describe, it, expect } from 'vitest';
import { ConditionEvaluator } from './condition-evaluator.js';

describe('ConditionEvaluator', () => {
  it('evaluates truthy strings via isTruthy', () => {
    const eval_ = new ConditionEvaluator(() => false);
    expect(eval_.evaluate('true')).toBe(true);
    expect(eval_.evaluate('false')).toBe(false);
    expect(eval_.evaluate('yes')).toBe(true);
    expect(eval_.evaluate('no')).toBe(false);
  });

  it('falls back to delegate for non-boolean strings', () => {
    const eval_ = new ConditionEvaluator((expr) => expr === 'x > 5');
    expect(eval_.evaluate('x > 5')).toBe(true);
    expect(eval_.evaluate('x < 5')).toBe(false);
  });
});
```

- [ ] **Step 2: Write failing rewriter tests**

```typescript
import { describe, it, expect } from 'vitest';
import { rewriteVariablePrefixes } from './variable-prefix-rewriter.js';

describe('rewriteVariablePrefixes', () => {
  const known = new Set(['env', 'ctx']);
  const forEach = new Set(['item']);

  it('leaves known-prefixed vars alone', () => {
    expect(rewriteVariablePrefixes('${env.HOST}', 'step', known, forEach)).toBe('${env.HOST}');
  });

  it('adds each. prefix to forEach vars', () => {
    expect(rewriteVariablePrefixes('${item.name}', 'step', known, forEach)).toBe('${each.item.name}');
  });

  it('adds default prefix to unqualified vars', () => {
    expect(rewriteVariablePrefixes('${count}', 'step', known, forEach)).toBe('${step.count}');
  });

  it('preserves default value syntax', () => {
    expect(rewriteVariablePrefixes('${host:-localhost}', 'step', known, forEach)).toBe('${step.host:-localhost}');
  });

  it('handles multiple vars in one string', () => {
    const input = '${env.HOST}:${port}/${item.db}';
    expect(rewriteVariablePrefixes(input, 'step', known, forEach)).toBe('${env.HOST}:${step.port}/${each.item.db}');
  });

  it('returns input unchanged if no vars', () => {
    expect(rewriteVariablePrefixes('plain text', 'step', known, forEach)).toBe('plain text');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/condition/condition-evaluator.test.ts packages/yaml-core/src/orchestration/variable-prefix-rewriter.test.ts`
Expected: FAIL

- [ ] **Step 4: Implement condition-evaluator.ts**

```typescript
import { isTruthy } from '../truthiness.js';

export class ConditionEvaluator {
  private readonly _delegate: (expr: string) => boolean;

  constructor(expressionDelegate: (expr: string) => boolean) {
    this._delegate = expressionDelegate;
  }

  evaluate(resolved: string): boolean {
    try {
      return isTruthy(resolved);
    } catch {
      return this._delegate(resolved);
    }
  }
}
```

- [ ] **Step 5: Implement variable-prefix-rewriter.ts**

```typescript
const VAR_RE = /\$\{([^}]+)}/g;

export function rewriteVariablePrefixes(
  input: string,
  defaultPrefix: string,
  knownPrefixes: Set<string>,
  forEachVars: Set<string>,
): string {
  return input.replace(VAR_RE, (match, content: string) => {
    const defaultSep = content.indexOf(':-');
    const varPart = defaultSep >= 0 ? content.substring(0, defaultSep) : content;
    const defaultVal = defaultSep >= 0 ? content.substring(defaultSep) : '';
    const dot = varPart.indexOf('.');
    if (dot >= 0) {
      const prefix = varPart.substring(0, dot);
      if (knownPrefixes.has(prefix)) return match;
      if (forEachVars.has(prefix)) return `\${each.${varPart}${defaultVal}}`;
    } else {
      if (forEachVars.has(varPart)) return `\${each.${varPart}${defaultVal}}`;
    }
    return `\${${defaultPrefix}.${varPart}${defaultVal}}`;
  });
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/condition/condition-evaluator.test.ts packages/yaml-core/src/orchestration/variable-prefix-rewriter.test.ts`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/src/condition/condition-evaluator.ts packages/yaml-core/src/condition/condition-evaluator.test.ts packages/yaml-core/src/orchestration/variable-prefix-rewriter.ts packages/yaml-core/src/orchestration/variable-prefix-rewriter.test.ts
git commit -m "feat(yaml-core): add ConditionEvaluator and VariablePrefixRewriter Refs #462"
```

### Task 12: Barrel export and package.json wiring

**Files:**
- Create: `packages/yaml-core/src/orchestration/index.ts`
- Modify: `packages/yaml-core/package.json`

**Interfaces:**
- Consumes: all types from previous tasks
- Produces: `@casehubio/yaml-core/orchestration` export path

- [ ] **Step 1: Create orchestration/index.ts**

```typescript
export type {
  OrcSignal, OrcSemaphore, OrcLatch, OrcChannel, OrcStateMachine,
  ScenarioScope, StepResultStore,
} from './types.js';

export type { StateHandler, TransitionHandler, Condition, SpeedMultiplier, RuntimeForEach } from './callbacks.js';

export { ChannelClosedError, IllegalTransitionError, SemaphoreReentrancyError } from './errors.js';

export type { StepError, LoopDirective, RetryDirective, ComputeBlock } from './directives.js';
export { parseLoopDirective, parseRetryDirective, parseComputeBlock } from './directives.js';

export { parseDuration } from './duration-parser.js';

export { DefaultOrcSignal } from './signal.js';
export { DefaultOrcLatch } from './latch.js';
export { DefaultOrcSemaphore } from './semaphore.js';
export { DefaultOrcChannel } from './channel.js';
export { DefaultOrcStateMachine, StateMachineBuilder } from './state-machine.js';
export { DefaultScenarioScope } from './scenario-scope.js';
export { DefaultStepResultStore } from './step-result-store.js';

export { rewriteVariablePrefixes } from './variable-prefix-rewriter.js';
```

- [ ] **Step 2: Add export path to package.json**

Add to the `exports` field in `packages/yaml-core/package.json`:

```json
"./orchestration": "./src/orchestration/index.ts"
```

- [ ] **Step 3: Add condition evaluator export to package.json**

Add to the `exports` field:

```json
"./condition": "./src/condition/condition-evaluator.ts"
```

- [ ] **Step 4: Run full test suite**

Run: `npx vitest run --project yaml-core`
Expected: ALL PASS

If vitest doesn't support `--project`, run: `npx vitest run --config packages/yaml-core/vitest.config.ts`

- [ ] **Step 5: Run typecheck**

Run: `npx tsc --noEmit -p packages/yaml-core/tsconfig.json`
Expected: no errors

- [ ] **Step 6: Commit**

```bash
git add packages/yaml-core/src/orchestration/index.ts packages/yaml-core/package.json
git commit -m "feat(yaml-core): barrel export and package wiring for orchestration module Refs #462"
```

---

## References

- [specs/issue-462-ts-orchestration-primitives/2026-09-23-ts-orchestration-primitives-design.md] — design spec this plan implements
- [specs/issue-462-ts-orchestration-primitives/decisions.md] — D1-D6 design decisions
- [packages/yaml-core/src/truthiness.ts] — existing isTruthy function consumed by ConditionEvaluator
- [packages/yaml-core/src/types.ts] — existing type conventions (discriminated unions, VariableSource)
- [packages/yaml-core/src/index.ts] — existing barrel export pattern
- [packages/yaml-core/vitest.config.ts] — test configuration
- [GitHub #462] — focal issue
- [platform #386, #391, #412] — Java source commits and browser parity analysis
