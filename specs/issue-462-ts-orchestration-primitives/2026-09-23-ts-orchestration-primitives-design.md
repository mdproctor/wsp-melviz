# TS Port of yaml-core Orchestration Primitives

**Issue:** #462
**Date:** 2026-09-23
**Branch:** issue-462-ts-orchestration-primitives

## Summary

Port the Java `io.casehub.yaml.core.orchestration` package to TypeScript in
`packages/yaml-core/src/orchestration/`. All Java blocking methods become
async (Promise-based). Primitives are plain TS objects — no Atomics, no
SharedArrayBuffer, no Web Workers. The cooperative round-robin scheduler
(#461/#463) ensures only one step executes at a time, so no concurrent
access protection is needed.

## Scope

**In scope (#462):**
- All orchestration primitive interfaces and default implementations
- Directive types (LoopDirective, RetryDirective, ComputeBlock)
- Utility types (DurationParser, StepError)
- ConditionEvaluator (wraps existing `isTruthy`)
- Error types (ChannelClosedError, IllegalTransitionError, SemaphoreReentrancyError)
- Runtime callback types (Condition, SpeedMultiplier, RuntimeForEach)
- VariablePrefixRewriter
- Full test coverage

**Out of scope (later issues):**
- Round-robin scheduler, queue management, virtual time (#461/#463)
- YAML parsing of orchestration constructs (#463)
- Trigger evaluation (DataTrigger, TimeTrigger) (#461)
- Simulation data injection

## Architecture

### Module Layout

```
packages/yaml-core/src/
├── orchestration/
│   ├── index.ts                    # barrel export
│   ├── types.ts                    # interfaces: OrcStateMachine, OrcSignal, OrcSemaphore, OrcLatch, OrcChannel, ScenarioScope, StepResultStore
│   ├── callbacks.ts                # StateHandler, TransitionHandler, Condition, SpeedMultiplier, RuntimeForEach
│   ├── errors.ts                   # ChannelClosedError, IllegalTransitionError, SemaphoreReentrancyError
│   ├── state-machine.ts            # DefaultOrcStateMachine
│   ├── signal.ts                   # DefaultOrcSignal
│   ├── semaphore.ts                # DefaultOrcSemaphore
│   ├── latch.ts                    # DefaultOrcLatch
│   ├── channel.ts                  # DefaultOrcChannel
│   ├── scenario-scope.ts           # DefaultScenarioScope
│   ├── step-result-store.ts        # DefaultStepResultStore
│   ├── directives.ts               # LoopDirective, RetryDirective, ComputeBlock (discriminated unions + parse functions)
│   ├── duration-parser.ts          # parseDuration("5s") → milliseconds
│   └── variable-prefix-rewriter.ts # rewrite(input, defaultPrefix, knownPrefixes, forEachVars)
├── condition/
│   └── condition-evaluator.ts      # ConditionEvaluator (wraps isTruthy + delegate)
└── (existing files unchanged)
```

### Export Strategy

New `orchestration/index.ts` barrel re-exports all public types. The package's
root `index.ts` gets an additional export path. `package.json` adds:

```json
"./orchestration": "./src/orchestration/index.ts"
```

Consumers import via `@casehubio/yaml-core/orchestration`.

## Primitive Designs

### OrcStateMachine<S extends string>

Java uses `Enum<S>`. TS uses string literal union types.

```typescript
interface OrcStateMachine<S extends string> {
  currentState(): S;
  transition(from: S, to: S, payload?: unknown): boolean;
  onTransition(from: S, to: S, handler: TransitionHandler): void;
  onEnter(state: S, handler: StateHandler): void;
  onExit(state: S, handler: StateHandler): void;
}
```

**DefaultOrcStateMachine** — builder pattern:
- State stored as a plain variable (no AtomicReference — single-threaded)
- `Map<S, Map<S, Predicate>>` for guarded transitions
- Terminal states block all outgoing transitions
- Builder: `.transition(from, to)`, `.guard(from, to, predicate)`, `.terminal(...states)`

### OrcSignal

```typescript
interface OrcSignal {
  signal(payload?: unknown): void;
  await(): Promise<void>;
  await(timeoutMs: number): Promise<boolean>;
  payload(): unknown;
  isSignalled(): boolean;
}
```

**DefaultOrcSignal** — two modes:
- One-shot (default): first `signal()` resolves all waiters and future `await()` calls resolve immediately
- Repeatable: each `signal()` resolves current waiters, future `await()` calls wait for the next signal

Implementation: queue of `{ resolve, reject }` callbacks. `signal()` drains the queue.

### OrcSemaphore

```typescript
interface OrcSemaphore {
  acquire(): Promise<void>;
  tryAcquire(timeoutMs: number): Promise<boolean>;
  release(): void;
  availablePermits(): number;
}
```

**DefaultOrcSemaphore:**
- Permit counter (plain number) + FIFO queue of pending resolve callbacks
- `acquire()`: if permits > 0, decrement and resolve immediately; else enqueue
- `release()`: if queue non-empty, dequeue and resolve; else increment permits
- Fair ordering (FIFO) matches Java's fair semaphore
- No token-bucket replenishment at primitive level — Java's `semaphore(name, permits, Duration window)` overload uses a ScheduledExecutorService for timed replenishment, which maps to virtual time in TS. The scheduler (#461/#463) will add replenishment as a periodic virtual-time task. The ScenarioScope factory method omits the `window` parameter for now; it will be added when the scheduler provides virtual time support.

### OrcLatch

```typescript
interface OrcLatch {
  countDown(): void;
  await(): Promise<void>;
  await(timeoutMs: number): Promise<boolean>;
  getCount(): number;
}
```

**DefaultOrcLatch:**
- Counter (plain number) + list of pending resolve callbacks
- `countDown()`: decrement; if zero, resolve all waiters
- `await()`: if already zero, resolve immediately; else enqueue

### OrcChannel<T>

```typescript
interface OrcChannel<T> {
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
```

**DefaultOrcChannel:**
- Unbounded: internal array as buffer, senders never block
- Bounded: capacity limit; senders block when full (queued resolve callbacks)
- Receivers block when empty (queued resolve callbacks)
- `close()`: reject all pending receivers with ChannelClosedError
- `close(cause)`: error-close — pending and future receives reject with cause

### ScenarioScope

```typescript
interface ScenarioScope {
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

**DefaultScenarioScope:**
- `Map<string, unknown>` for get-or-create semantics
- `close()`: closes all channels, resolves all signals, zeros all latches, releases all semaphores

Java uses `Class<S> stateType` for enum reflection. TS passes `states: readonly S[]`
(the list of valid state values) since string literal unions aren't introspectable at runtime.

### StepResultStore

```typescript
interface StepResultStore {
  recordSuccess(stepName: string, result: Record<string, unknown>): void;
  recordFailure(stepName: string, error: StepError): void;
  result(stepName: string): Record<string, unknown> | undefined;
  error(stepName: string): StepError | undefined;
  hasCompleted(stepName: string): boolean;
}
```

**DefaultStepResultStore:** three Maps (results, errors, completed flags).

## Directive Types

### LoopDirective

```typescript
type LoopDirective =
  | { type: 'count'; count: number }
  | { type: 'count-until'; count: number; until: string }
  | { type: 'until'; until: string };

function parseLoopDirective(raw: unknown): LoopDirective;
// Number → { type: 'count', count: N }
// { count: N, until: "expr" } → { type: 'count-until', ... }
// { until: "expr" } → { type: 'until', ... }
```

### RetryDirective

```typescript
type RetryDirective =
  | { type: 'simple'; max: number }
  | { type: 'full'; max: number; backoff: string; delayMs: number };

function parseRetryDirective(raw: unknown): RetryDirective;
// Number → { type: 'simple', max: N }
// { max: N, backoff: "exponential", delay: "1s" } → { type: 'full', ... }
```

### ComputeBlock

```typescript
interface ComputeBlock {
  engine: string;
  expression: string;
}

function parseComputeBlock(map: Record<string, unknown>, defaultEngine: string): ComputeBlock;
```

## Utilities

### DurationParser

```typescript
function parseDuration(input: string): number; // returns milliseconds
// "10ms" → 10, "5s" → 5000, "3m" → 180000, "1h" → 3600000
```

### VariablePrefixRewriter

```typescript
function rewriteVariablePrefixes(
  input: string,
  defaultPrefix: string,
  knownPrefixes: Set<string>,
  forEachVars: Set<string>,
): string;
```

### ConditionEvaluator

```typescript
class ConditionEvaluator {
  constructor(expressionDelegate: (expr: string) => boolean);
  evaluate(resolved: string): boolean;
  // tries isTruthy first, falls back to delegate
}
```

## Error Types

```typescript
class ChannelClosedError extends Error {
  constructor(channelName: string, cause?: Error);
}

class IllegalTransitionError extends Error {
  constructor(machineName: string, from: string, to: string);
}

class SemaphoreReentrancyError extends Error {
  constructor(name: string, stepContext: string);
}
```

## Timeout Handling

Java uses `(long timeout, TimeUnit unit)` throwing `InterruptedException`.
TS uses optional `timeoutMs?: number` parameter with semantic return values:

| Method | No timeout | With timeout (expired) |
|--------|-----------|----------------------|
| `await()` | `Promise<void>` (waits forever) | `Promise<boolean>` (false = timed out) |
| `receive()` | `Promise<T>` (waits forever) | `Promise<T \| undefined>` (undefined = timed out) |
| `acquire()` | `Promise<void>` (waits forever) | `Promise<boolean>` (false = timed out) |

Timeouts at the primitive level use real `setTimeout` — the scheduler layer
(#461/#463) will wrap these with virtual-time-aware timeouts. This keeps
primitives simple and testable without a scheduler.

## Testing Strategy

Each primitive gets its own test file alongside its implementation:
`state-machine.test.ts`, `signal.test.ts`, etc.

Tests verify:
- Happy path (acquire/release, send/receive, signal/await)
- Edge cases (timeout expiry, double-close, release without acquire)
- Error conditions (transition to invalid state, receive from closed channel)
- Ordering guarantees (FIFO for semaphore waiters, all-waiters-resolve for latch)
- One-shot vs repeatable signal modes
- ScenarioScope get-or-create semantics and close cascade
- Parse functions for directives (valid input, invalid input, edge cases)

No scheduler needed for primitive tests — they're pure Promise-based
and testable with standard async test patterns.

## References

- Java `io.casehub.yaml.core.orchestration` package (platform repo, commits 1575c16a, a7782d37, 3cb04c9a)
- Java `io.casehub.yaml.core.condition` package (ConditionEvaluator, Truthiness)
- Java `io.casehub.yaml.core.resolver` package (VariablePrefixRewriter)
- Existing TS `@casehubio/yaml-core` package (`packages/yaml-core/`)
- GitHub issue #462
- Design decisions D1–D6 in `decisions.md`
