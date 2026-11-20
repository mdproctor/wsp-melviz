# Scenario Orchestrator — DES Scheduler, YAML Binding, Trigger Evaluation

**Issues:** #461, #463
**Date:** 2026-09-23
**Branch:** issue-461-463-scenario-orchestrator

## Summary

Replace the existing sequential scenario runners with a discrete-event
simulation (DES) scheduler that supports concurrent step queues, virtual
time, orchestration primitives (#462), and trigger-based activation. Add
YAML binding for orchestration constructs at both inline and top-level.

## Scope

**In scope:**
- DES scheduler with virtual clock, step queues, tick loop
- YAML binding for orchestration constructs (concurrent, mutex, barrier,
  signal, await, retry, loop, state machine, channel, trigger)
- Dual-level YAML: inline for small constructs, top-level `orchestration:`
  block for complex/reusable definitions
- DataTrigger and TimeTrigger evaluation as queue activators
- StepExecutor strategy pattern for delivery-type dispatch
- TutorialRunner-shaped public API (play/pause/step/runTo/setSpeed/dispose)
- Delete runner.ts and sectioned-runner.ts
- Comprehensive unit tests (mock executors, injectable tick, speed=infinity)

**Deferred to follow-up issues (out of #463 scope for this spec):**
- `rateLimit: { max: 10, per: 1s }` — token-bucket semaphore requires virtual-time-aware
  replenishment, deferred until scheduler provides virtual time support (per #462 spec)
- `forEach:` runtime expansion — server-side concern, requires RuntimeForEach integration
- `deadline:` propagation on chapters/sections — requires scope lifecycle hooks
- Showcase gallery scenarios — implementation artifact, not a design concern
- GraphQL executor implementation (interface only — wiring is a later issue)
- Simulation data injection (uses the same scheduler but injection logic is separate)
- Visual regression testing of tutorial playback

**Relationship to scenario-handler.ts:**
The `scenario-handler.ts` in `packages/pages-aria/src/server/` is the distributed
execution path — it receives commands from the backend `ScenarioOrchestrator` via
WebSocket push and executes them locally. It has no scenario model, no parser, and
no scheduling — it is a command executor driven by the server. The DES scheduler
replaces the local execution path (`runner.ts`, `sectioned-runner.ts`) used for
tutorials and local demos. The two systems coexist: local scenarios use the scheduler,
server-driven scenarios use `scenario-handler.ts`. They share the `AriaExecutor`
(`command-executor.ts`) for DOM step execution but have independent orchestration.
No changes to `scenario-handler.ts` are required.

## Architecture

### Component Diagram

```
┌────────────────────────────────────────────────────────┐
│                    YAML (.scenario.yaml)                │
│  steps: / sections: / orchestration:                    │
└──────────────────────┬─────────────────────────────────┘
                       │ parse
                       ▼
┌──────────────────────────────────────────────────────┐
│              ScenarioParser (extended)                 │
│  Recognizes orchestration constructs                  │
│  Builds ScenarioModel with OrchestratedStep types     │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│              YamlBinder                               │
│  ScenarioModel → ScenarioScope + StepQueue[]          │
│  Creates primitives from orchestration: block          │
│  Expands inline constructs to anonymous primitives     │
│  Builds queue tree from concurrent: blocks             │
└──────────┬───────────────────────────┬───────────────┘
           │                           │
           ▼                           ▼
┌────────────────────┐  ┌──────────────────────────────┐
│   ScenarioScope    │  │        DES Scheduler          │
│   (from #462)      │  │                               │
│   Primitives:      │  │  Virtual clock                │
│   - semaphore      │  │  Step queues (ready/suspended) │
│   - latch          │  │  Tick loop:                    │
│   - signal         │  │    1. Run ready queues         │
│   - channel        │  │    2. Advance virtual time     │
│   - state machine  │  │    3. Activate triggered queues │
│   - result store   │  │    4. yield (rAF)             │
│                    │  │                               │
│                    │  │  Public API:                   │
│                    │  │  play/pause/step/runTo/        │
│                    │  │  setSpeed/dispose              │
└────────────────────┘  └──────────┬───────────────────┘
                                   │ execute(step)
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
             ┌───────────┐ ┌───────────┐ ┌───────────────┐
             │   Aria     │ │  GraphQL  │ │  Simulated    │
             │  Executor  │ │ Executor  │ │  Executor     │
             │  (DOM ops) │ │ (network) │ │ (data inject) │
             └───────────┘ └───────────┘ └───────────────┘
```

### Module Layout

```
packages/pages-aria/src/scenario/
├── types.ts                    # Extended: OrchestratedStep types, triggers
├── parser.ts                   # Extended: orchestration construct recognition
├── yaml-binder.ts              # NEW: ScenarioModel → ScenarioScope + queues
├── scheduler.ts                # NEW: DES scheduler (virtual time, tick loop)
├── step-queue.ts               # NEW: StepQueue (ready/suspended/blocked)
├── virtual-clock.ts            # NEW: VirtualClock (advance, speed, pause)
├── step-executor.ts            # NEW: StepExecutor interface + registry
├── trigger-evaluator.ts        # NEW: DataTrigger/TimeTrigger → queue activation
├── index.ts                    # Updated barrel
├── runner.ts                   # DELETED
├── sectioned-runner.ts         # DELETED
└── __tests__/
    ├── scheduler.test.ts       # Core scheduler tests
    ├── step-queue.test.ts      # Queue state transitions
    ├── virtual-clock.test.ts   # Time advancement, speed, pause
    ├── yaml-binder.test.ts     # YAML → queue tree construction
    ├── trigger-evaluator.test.ts # Trigger activation
    └── parser-orchestration.test.ts # Orchestration YAML parsing

packages/pages-aria/src/executor/
├── command-executor.ts         # EXISTING: adapted to implement StepExecutor (AriaExecutor)
├── editable-text.ts            # EXISTING: unchanged
├── spotlight.ts                # EXISTING: unchanged
├── visual-feedback.ts          # EXISTING: unchanged
└── index.ts                    # EXISTING: re-exports AriaExecutor
```

## DES Scheduler

### Virtual Clock

```typescript
interface VirtualClock {
  now(): number;              // current virtual time in ms
  advance(deltaMs: number): void;
  speed(): number;
  setSpeed(multiplier: number): void;
  pause(): void;
  resume(): void;
  isPaused(): boolean;
}
```

The clock maintains virtual time as a monotonic counter in milliseconds.
`advance(delta)` adds exactly `delta` to the counter — virtual time
advancement is always deterministic and independent of playback speed.

Speed controls *inter-step pacing* — the real-time delay between ticks:
`await new Promise(r => setTimeout(r, baseDelay / speed))` where
`baseDelay = 300ms`. This matches the existing `sectioned-runner.ts`
behavior (line 235: `setTimeout(r, 300 / rs.speed)`):
- speed=1 → 300ms between steps (human-watchable tutorial pace)
- speed=2 → 150ms
- speed=4 → 75ms
- speed=0.5 → 600ms (slow motion)
- speed=Infinity (test mode): delay resolves immediately via
  `Promise.resolve()`, all steps execute in zero real time

The tick loop also yields to the browser via `rAF` after each tick to
ensure the rendering pipeline can paint. The inter-step delay and rAF
yield serve different purposes: rAF syncs with the display, the delay
paces the tutorial. Both are injectable — tests replace both with
immediate resolution.

`setSpeed(multiplier)` requires `multiplier > 0` or `Infinity`. Speed=0 is
not accepted; use `pause()` instead.

### Step Queue

Each execution path (main, concurrent branches, triggered steps) is a
`StepQueue`:

```typescript
type QueueState = 'ready' | 'suspended' | 'blocked' | 'done';

interface StepQueue {
  id: string;
  state: QueueState;
  steps: OrchestratedStep[];
  position: number;           // current step index
  wakeTime?: number;          // virtual time to wake (for delays)
  blockReason?: Promise<void>; // primitive wait (semaphore, signal, etc.)
  parent?: StepQueue;         // for nested concurrent blocks
  children: StepQueue[];      // child branches
}
```

States:
- **ready** — has steps remaining and no blocking condition
- **suspended** — waiting for a trigger to activate (DataTrigger/TimeTrigger)
- **blocked** — waiting for a primitive (semaphore.acquire, signal.await) or virtual time delay
- **done** — all steps executed

### Tick Loop

```typescript
async function tickLoop() {
  let staleTicks = 0;
  const retryState = new Map<string, number>(); // queue:position → retry count
  const loopState = new Map<string, number>();   // queue:position → remaining iterations

  while (hasActiveQueues()) {
    if (disposed) return;
    const snapshotBefore = queueStateSnapshot();

    // 1. Run all ready queues concurrently — one step each
    const ready = readyQueues();
    await Promise.all(ready.map(async (queue) => {
      const step = queue.steps[queue.position];
      const decorators = step.decorators;
      const posKey = `${queue.id}:${queue.position}`;

      // when: guard — skip step if condition is false
      if (decorators?.when) {
        if (!conditionEvaluator.evaluate(decorators.when)) {
          queue.position++;
          return;
        }
      }

      try {
        await dispatchStep(step, queue);

        // If dispatchStep set queue to blocked (concurrent, semaphore,
        // signal.await), skip all post-dispatch processing — the
        // construct owns the queue's lifecycle from here.
        if (queue.state !== 'ready') return;

        // Determine whether to advance position (loop check first)
        let advancePosition = true;
        if (decorators?.loop) {
          if (evaluateLoopContinuation(decorators.loop, posKey, loopState, conditionEvaluator)) {
            advancePosition = false;
            retryState.delete(posKey);
          } else {
            loopState.delete(posKey);
          }
        }

        if (advancePosition) {
          retryState.delete(posKey);
          queue.position++;
        }

        // Post-step delay (applies to both loop-repeat and advance)
        if (decorators?.delay) {
          queue.wakeTime = clock.now() + parseDuration(decorators.delay);
          queue.state = 'blocked';
          // After delay expires, queue resumes at current position:
          // - if loop repeated: same step (position unchanged)
          // - if advanced: next step (position incremented above)
          return;
        }

        // Check done (only when no delay — delay defers done check
        // to the next tick after unblocking)
        if (advancePosition && queue.position >= queue.steps.length) {
          queue.state = 'done';
          resolveParentIfChildrenDone(queue);
        }
      } catch (err) {
        scope.resultStore().recordFailure(step.name ?? posKey, {
          message: (err as Error).message, stepName: step.name ?? '',
        });
        const retryMax = decorators?.retry?.max ?? 0;
        const retryCount = retryState.get(posKey) ?? 0;
        if (retryCount < retryMax) {
          retryState.set(posKey, retryCount + 1);
        } else {
          retryState.delete(posKey);
          queue.state = 'done';
          emitError(queue, step, err);
        }
      }
    }));

    // 2. Advance virtual time to next wake point
    //    Considers both blocked queue wakeTimes AND suspended TimeTrigger fireTimes
    const nextWake = earliestWakeTime(); // includes trigger.fireTime for TimeTrigger queues
    if (nextWake !== undefined) {
      clock.advance(nextWake - clock.now());
      // Unblock queues whose wakeTime <= clock.now()
      for (const q of blockedQueues()) {
        if (q.wakeTime !== undefined && q.wakeTime <= clock.now()) {
          q.state = 'ready';
          q.wakeTime = undefined;
        }
      }
    }

    // 3. Check trigger conditions — activate suspended queues
    for (const q of suspendedQueues()) {
      if (evaluateTrigger(q.trigger)) {
        q.state = 'ready';
      }
    }

    // 4. Deadlock detection
    //    Only count stale ticks when no DataTrigger-suspended queues exist
    //    (DataTrigger queues wait for external push data — not a deadlock)
    if (queueStateEquals(snapshotBefore, queueStateSnapshot())) {
      const hasDataTriggerQueues = suspendedQueues().some(q => q.trigger?.type === 'data');
      if (!hasDataTriggerQueues) {
        staleTicks++;
        if (staleTicks > 100) {
          emitError(null, null, new Error('Scheduler deadlock: no progress for 100 ticks'));
          return;
        }
      }
    } else {
      staleTicks = 0;
    }

    // 5. Yield to browser + inter-step pacing
    await tick();    // rAF in browser, Promise.resolve() in tests
    await delay();   // setTimeout(baseDelay / speed), immediate in tests
  }
}
```

**`earliestWakeTime()`** returns the minimum of:
- `wakeTime` from blocked queues (virtual-time delays)
- `trigger.fireTime` from suspended TimeTrigger queues

This ensures virtual time advances to reach TimeTrigger fire points even
when no blocked queues exist.

**`evaluateLoopContinuation()`** checks whether the loop should repeat:
- `type: 'count'`: initialises remaining count on first call, decrements
  each iteration. Repeats while remaining > 0.
- `type: 'until'`: evaluates condition via `ConditionEvaluator`. Repeats
  while condition is false. (Inverse of `when:` — `until` is "repeat
  while NOT condition".)
- `type: 'count-until'`: stops when count exhausted OR condition is true.

Loop state (remaining iterations) is tracked per queue-position key in
the scheduler's `loopState` map, not on the step object. Retry count
resets on each new loop iteration.

When a step encounters an orchestration construct (concurrent block,
signal await, semaphore acquire), the scheduler handles it:

- **concurrent:** creates child queues, sets parent queue to blocked
  until all children are done
- **delay/await:** sets `wakeTime` on the queue, transitions to blocked
- **semaphore.acquire:** calls the primitive; if the Promise doesn't
  resolve synchronously, queue transitions to blocked with `blockReason`.
  The Promise's `.then()` callback sets `queue.state = 'ready'` and
  clears `blockReason` — this transition happens asynchronously via the
  microtask queue between ticks. The queue becomes ready and is picked
  up by `readyQueues()` on the next tick. State mutations from Promise
  resolution happen OUTSIDE the tick loop's control flow — this is
  correct DES behavior (events at virtual time T are fully processed
  before time advances, and inter-tick Promise resolutions take effect
  at the start of the next tick).
- **signal.await:** same mechanism as semaphore — queue blocks on the
  Promise, `.then()` transitions to ready between ticks

### Step Dispatch

```typescript
interface StepExecutor {
  canExecute(step: OrchestratedStep): boolean;
  execute(step: OrchestratedStep, context: ExecutionContext): Promise<void>;
}

interface ExecutionContext {
  scope: ScenarioScope;
  clock: VirtualClock;
  eventTarget: EventTarget;
  speed: number;
  conditionEvaluator: ConditionEvaluator;  // from @casehubio/yaml-core/condition (not yet exported — yaml-core barrel update needed)
}
```

The scheduler maintains a `StepExecutor[]` registry. For each step, it
finds the first executor where `canExecute(step)` returns true and calls
`execute()`. The existing `command-executor.ts` becomes `AriaExecutor`.

### Public API

```typescript
interface SchedulerOptions {
  eventTarget: EventTarget;     // required — state events and controller commands
  contentBase?: string;         // base path for template content resolution
  speed?: number;               // initial speed multiplier (default: 1.0)
  startPaused?: boolean;        // start in paused state (default: false)
  executors?: StepExecutor[];   // additional executors (AriaExecutor is always registered)
  onComplete?: (scenarioName: string) => void;
}

function createScheduler(
  scenario: Scenario,           // accepts both flat and sectioned
  options: SchedulerOptions,
): ScenarioRunner;
```

`createScheduler` parses the scenario into queues via YamlBinder, creates
a `ScenarioScope`, registers the built-in `AriaExecutor` plus any
additional executors from `options.executors`, and starts the tick loop.

The scheduler registers a `scenario-command` event listener on
`options.eventTarget` internally (matching the existing
`sectioned-runner.ts` pattern at line 169). This bridges the
`PagesScenarioController`'s transport controls:

```typescript
eventTarget.addEventListener('scenario-command', (e: Event) => {
  const { command, speed, label } = (e as CustomEvent).detail;
  switch (command) {
    case 'pause':  runner.pause(); break;
    case 'resume': runner.play(); break;
    case 'step':   runner.step(); break;
    case 'speed':  if (speed != null) runner.setSpeed(speed); break;
    case 'run-to': if (label) runner.runTo(label); break;
  }
});
```

The listener is removed on `dispose()`.

```typescript
interface ScenarioRunner {
  play(): void;
  pause(): void;
  step(): Promise<void>;      // execute one step, then pause
  runTo(sectionTitle: string): void;
  setSpeed(multiplier: number): void;
  dispose(): void;

  readonly state: 'idle' | 'playing' | 'paused' | 'done';
  readonly outline: OutlineNode[];     // section/step structure for nav UI (from scenario-connection-controller.ts)
  readonly clock: VirtualClock;        // for external time display

  addEventListener(type: string, handler: EventListener): void;
  removeEventListener(type: string, handler: EventListener): void;
}
```

**`runTo(sectionTitle)`** is a **teleport** — it sets position directly
to the target section's first step, pauses the scheduler, resolves the
section's content (template fetch), and emits a `scenario:state` event.
It does NOT execute any intermediate steps. This matches the existing
`sectioned-runner.ts` behavior (lines 262-272) where sections are slides
— users navigate between them instantly without side effects.

For orchestrated scenarios with scope primitives (semaphores, signals,
channels), `runTo()` also **resets the scope** — calls `scope.close()`
and creates a fresh `ScenarioScope`. This avoids stale state from
previous sections (e.g., a semaphore acquired in section 2 would block
section 5 if the scope were preserved). All queues except the target
section's main queue are discarded. Triggered and concurrent queues
are re-created from the target section forward by re-running YamlBinder
on the remaining sections.

`step()` executes exactly one step from the next ready queue and pauses.
For sub-step operations (line-by-line editor typing), the AriaExecutor
handles internal pacing — the scheduler sees it as one step.

**Flat scenarios:** A flat scenario with `steps:` and no `sections:` creates
a single `StepQueue` with all steps. The `outline` is an empty array. The
`runTo()` method is a no-op for flat scenarios (no sections to navigate).
The scheduler handles both formats uniformly — a sectioned scenario is a
flat scenario with implicit queue breaks at section boundaries.

**Dispose:** `dispose()` sets a `disposed` flag, causes the tick loop to
exit at the next check, calls `scope.close()` (which closes all channels,
signals all signals, releases all primitives), removes all event listeners,
and **clears the queue array** (`queues.length = 0`). This breaks the
reference chain `scheduler → queues → blockReason → semaphore._waiters`
and allows GC. Without clearing, `DefaultOrcSemaphore` has no `close()`
method, so `scope.close()` cannot reject pending acquires — the
`_waiters` array retains resolve callbacks indefinitely. Clearing the
queue array is the scheduler's responsibility; the primitive gap is a
#462 concern.

After `dispose()`, pending Promises from primitives may resolve/reject but
their callbacks find the scheduler disposed and no-op.

### Events

Emitted on the runner's EventTarget (same `pages-event` pattern as
existing sectioned-runner):

| Event topic | When | Data |
|---|---|---|
| `scenario:state` | play/pause/step/done transitions | Full `ScenarioState` payload (see below) |
| `scenario:step` | after each step executes | `{ queue, step, virtualTime }` |
| `scenario:section` | entering a new section | `{ sectionIndex, title }` |
| `scenario:queue` | queue state change | `{ queueId, state, reason }` |

The `scenario:state` event emits the existing `ScenarioState` shape from
`scenario-connection-controller.ts` to maintain backward compatibility with
`PagesScenarioController`, `PagesScenarioNarrative`, and `PagesTutorialHost`:

```typescript
interface ScenarioState {
  scenario: string | null;
  chapter: string | null;
  section: string | null;
  step: string | null;
  paused: boolean;
  speed: number;
  progress: number;
  content: { type: string; markdown?: string; path?: string; section?: string } | null;
  slides: string | null;
  outline?: OutlineNode[];
  error?: { step: string; message: string } | null;
}
```

The scheduler computes `progress` from `completedSteps / totalSteps` across all
queues. The `content` field is populated from section content resolution (same
template fetch logic as sectioned-runner). The `outline` is included on the
initial state emission.

## YAML Binding

### Extended Scenario Model

```typescript
// New orchestrated step types added to ScenarioStep union:
| { delivery: 'orchestration'; construct: 'concurrent'; branches: Record<string, ScenarioStep[]> }
| { delivery: 'orchestration'; construct: 'signal'; name: string }
| { delivery: 'orchestration'; construct: 'await'; signal?: string; barrier?: string; timeout?: string }
| { delivery: 'orchestration'; construct: 'delay'; duration: string }

// Step-level decorators (on any delivery type):
interface StepDecorators {
  mutex?: string;
  retry?: RetryDirective;
  loop?: LoopDirective;
  when?: string;          // condition guard — skip step if false
  timeout?: string;       // step-level timeout
  delay?: string;         // post-step virtual-time delay (e.g. "100ms")
}

// OrchestratedStep extends the base step union with optional decorators:
type OrchestratedStep = (ScenarioStep | OrchestrationConstruct) & {
  decorators?: StepDecorators;
};
// The YamlBinder merges inline YAML keys (mutex, retry, loop, when,
// timeout, delay) into the decorators field during binding.

// Top-level orchestration block:
interface OrchestrationBlock {
  machines?: Record<string, StateMachineDefinition>;
  barriers?: Record<string, { count: number }>;
  quorums?: Record<string, { required: number; of: string[] }>;
  channels?: Record<string, { capacity?: number }>;
  signals?: string[];     // pre-declared signal names
}

// State machine definition in YAML:
interface StateMachineDefinition {
  initial: string;
  states: string[];
  transitions: Array<{ from: string; to: string; guard?: string }>;
  terminal?: string[];
}

// Trigger types (#461):
// DataTrigger adapts #461's endpoint-polling concept for the frontend:
// backend DataTrigger polls endpoints → frontend DataTrigger watches channels
// (external data arrives via push sources and is routed to channels)
interface DataTrigger {
  type: 'data';
  channel: string;        // watch for data on this channel
  condition?: string;     // optional filter expression via ConditionEvaluator
}

interface TimeTrigger {
  type: 'time';
  delay: string;          // virtual time delay from queue suspension (e.g. "5s")
  repeat?: boolean;       // fire once or repeatedly
}
```

### YamlBinder

Transforms a parsed `ScenarioModel` into a `ScenarioScope` + root
`StepQueue[]`:

1. Read `orchestration:` block → create named primitives in ScenarioScope
2. Walk the step tree:
   - Regular steps → append to current queue
   - `concurrent:` → create child queues, one per branch
   - `signal:` / `await:` → inline primitive operations
   - Step decorators (mutex, retry, loop) → wrap in orchestration logic
3. Triggered steps → create suspended queues with trigger conditions
4. Return `{ scope, queues, triggers }`

Inline constructs create anonymous entries in ScenarioScope (e.g.
`mutex: db-write` → `scope.semaphore('__anon_mutex_db-write', 1)`).
Top-level constructs use their declared names. The binder validates
that no top-level `orchestration:` name starts with `__anon_` — this
prefix is reserved for inline-generated primitives.

`when:` guards are evaluated via `ConditionEvaluator` from
`@casehubio/yaml-core/condition` (currently not exported from the
yaml-core barrel — a barrel update is needed as a prerequisite).
The evaluator delegates to `isTruthy()` for simple boolean checks and
to a pluggable expression delegate for comparisons:
- `${result.step-name.field} == 'value'` — checks StepResultStore
- `${scope.signal-name.signalled}` — checks signal state

**`when:` semantics:** evaluated inside `dispatchStep()` before calling
the executor. If the condition is false, the step is **skipped** — the
queue advances position and continues to the next step. This is
conditional execution, not "wait until."

**`loop.until` semantics:** evaluated after the step executes. If the
condition is true, the loop ends (advances position). If false, the
step repeats next tick. This is the inverse of `when:` — `until` means
"repeat while condition is false."

`quorum:` binds to `scope.latch(name, required)` where `required < of.length`:
```yaml
orchestration:
  quorums:
    approval: { required: 2, of: [manager, director, vp] }
```

`machine:` binds to `scope.stateMachine(name, states, initial)` with
guarded transitions:
```yaml
orchestration:
  machines:
    order-status:
      initial: pending
      states: [pending, approved, rejected, fulfilled]
      transitions:
        - { from: pending, to: approved }
        - { from: pending, to: rejected }
        - { from: approved, to: fulfilled }
      terminal: [rejected, fulfilled]
```

### YAML Examples

**Sequential with inline orchestration:**
```yaml
scenario: Portfolio Update
steps:
  - click: { role: button, name: Refresh }
    mutex: portfolio-write
    retry: 3
  - await: { signal: data-loaded, timeout: 30s }
  - assert: { role: cell, name: "AAPL", hasText: "150.00" }
```

**Concurrent branches with top-level declarations:**
```yaml
scenario: Multi-Feed Simulation
orchestration:
  barriers:
    all-feeds-ready: { count: 3 }
  channels:
    trades: { capacity: 10 }
    market: {}
    news: {}

steps:
  - concurrent:
      feed-trades:
        - simulated: { dataset: trades, data: { symbol: AAPL, price: 150 } }
          delay: 100ms
        - signal: all-feeds-ready
      feed-market:
        - simulated: { dataset: market, data: { index: SPX, value: 4500 } }
          delay: 50ms
        - signal: all-feeds-ready
      feed-news:
        - simulated: { dataset: news, data: { headline: "Fed holds rates" } }
          delay: 200ms
        - signal: all-feeds-ready
  - await: { barrier: all-feeds-ready }
  - assert: { role: heading, hasText: "3 feeds active" }
```

**Triggered steps:**
```yaml
scenario: Reactive Dashboard
steps:
  - navigate: /dashboard
  - trigger:
      type: data
      channel: trades
    steps:
      - assert: { role: cell, name: "Last Trade" }
      - spotlight: { role: row, name: "AAPL" }
  - trigger:
      type: time
      delay: 5s
    steps:
      - click: { role: button, name: "Auto-refresh" }
```

## Trigger Evaluation

### TriggerEvaluator

```typescript
interface TriggerEvaluator {
  evaluate(trigger: DataTrigger | TimeTrigger, scope: ScenarioScope,
           clock: VirtualClock): boolean;
}
```

- **DataTrigger:** checks `!scope.channel(trigger.channel).isEmpty()` —
  returns true when the channel has data. If `condition` is set, uses
  `ConditionEvaluator` to filter the channel's buffered data.
  Note: trigger evaluation *activates* the queue — it does not consume
  the data. The queue's steps consume via `channel.receive()`. If multiple
  queues trigger on the same channel, all activate, but only one receives
  each message (standard channel semantics — fan-out activation, fan-in
  consumption).
- **TimeTrigger:** checks `clock.now() >= trigger.fireTime` — returns
  true when virtual time has passed the trigger's fire point. `fireTime`
  is computed when the queue transitions to `suspended` (not at bind time):
  `fireTime = clock.now() + parseDuration(trigger.delay)`. This ensures
  the delay is relative to when the trigger becomes active, not to
  scenario start. If `repeat`, reset `fireTime` after each activation:
  `fireTime = clock.now() + parseDuration(trigger.delay)`.

## Testing Strategy

Tests use injectable `tick: () => Promise<void>` that resolves immediately
(no rAF). Virtual clock at `speed=Infinity` means all delays resolve
instantly. Mock executors record calls without DOM interaction.

### Critical Test Cases

From the existing sectioned-runner (must not regress):

| Behavior | Test approach |
|---|---|
| `runTo(section, step)` | Mock executor records step order; verify exactly the expected steps executed |
| Single-step advancement | Call `step()`, verify one step executed, state is paused |
| Speed changes mid-execution | Change speed during play, verify inter-step delays change proportionally |
| Pause/resume at arbitrary points | Pause mid-sequence, verify no more steps execute until resume |
| Line-by-line editor typing | AriaExecutor tests (internal pacing, acceleration phases) |
| Section outline generation | Parse sectioned YAML, verify outline matches section titles |
| Template content resolution | Verify external .md fetch in section content |

New scheduler-specific tests:

| Behavior | Test approach |
|---|---|
| Concurrent branch interleaving | Two branches with mock steps; verify DES ordering (time-ordered, not round-robin) |
| Semaphore limits concurrency | 3 branches, semaphore permits=2; verify only 2 run simultaneously |
| Latch barrier | 3 branches count down; verify main resumes only after all 3 |
| Signal coordination | Branch A signals, Branch B awaits; verify B resumes after A signals |
| Channel send/receive across queues | Producer queue sends, consumer queue receives; verify data flow |
| DataTrigger activation | Suspended queue activates when channel gets data |
| TimeTrigger activation | Suspended queue activates when virtual time reaches threshold |
| Virtual time advancement | Verify clock advances correctly with speed multiplier |
| speed=Infinity | All delays resolve in zero real time |
| Nested concurrent blocks | Concurrent inside concurrent; verify correct queue tree |
| Queue done propagation | Parent unblocks when all children complete |
| Inline mutex as semaphore(1) | Verify step-level mutex creates anonymous semaphore |
| Retry on step failure | Step fails, retries N times per RetryDirective |
| Loop directive | Step repeats N times or until condition |
| Top-level orchestration block | Named primitives created at bind time, referenced from steps |
| Dispose mid-execution | Verify all queues stopped, scope closed, no lingering timers |

## Migration

### Deleted Files
- `runner.ts` — replaced by scheduler
- `sectioned-runner.ts` — replaced by scheduler
- `scenario.test.ts` — replaced by new test files
- `sectioned-runner.test.ts` — replaced by new test files

### Modified Files
- `types.ts` — extended with OrchestratedStep types, triggers, decorators
- `parser.ts` — extended with three new recognition paths in `parseSteps()`:
  1. **Delivery shorthands:** `simulated:` and `graphql:` keys produce
     steps with `delivery: 'simulated'` / `delivery: 'graphql'` directly,
     bypassing `expandAriaShorthand()`. Checked before ARIA fallback.
  2. **Orchestration constructs:** `concurrent:`, `signal:`, `await:`,
     `delay:` keys produce `delivery: 'orchestration'` steps. `await:`
     is used instead of `wait:` to avoid collision with the existing
     ARIA `wait` action in `ARIA_ACTIONS` (line 10 of current parser).
  3. **Decorator extraction:** inline keys matching `StepDecorators`
     fields (`mutex`, `retry`, `loop`, `when`, `timeout`, `delay`) are
     separated from the step body and stored in `step.decorators`.
- `packages/pages-aria/src/executor/command-executor.ts` — adapted to implement `StepExecutor` interface (stays in `src/executor/`)
- `index.ts` — updated exports (see barrel specification below)

### Barrel Exports (`index.ts`)

```typescript
// Functions
export { parseScenario } from './parser.js';
export { createScheduler } from './scheduler.js';
export { isSectioned } from './types.js';

// Types
export type {
  Scenario, FlatScenario, SectionedScenario, ScenarioBase,
  ScenarioStep, OrchestratedStep, TutorialMeta, TutorialSection,
  SectionContent, DataTrigger, TimeTrigger, StepDecorators,
  OrchestrationBlock,
} from './types.js';
export type { ScenarioRunner, SchedulerOptions } from './scheduler.js';
export type { StepExecutor, ExecutionContext } from './step-executor.js';
```

Removed exports: `runScenario` (from deleted `runner.ts`),
`runSectionedScenario` (from deleted `sectioned-runner.ts`),
`TutorialRunner`, `TutorialRunnerOptions`.

### Consumers
The `PagesTutorialHost` component in `packages/pages-aria/src/tutorial/tutorial-host.ts`
currently imports `runSectionedScenario` and `TutorialRunner` from
`../scenario/sectioned-runner.js` (lines 6). It calls `runner.runTo(sectionTitle)`
in `_onPrev()` (line 105) and `_onNext()` (line 112) with section title strings,
and receives `ScenarioState` events via `_trackState()` (line 93).

Migration:
- Change import to `createScheduler` from `../scenario/scheduler.js`
- The `ScenarioRunner` interface preserves the same API shape:
  `play()`, `pause()`, `step()`, `runTo(sectionTitle)`, `setSpeed()`, `dispose()`
- Event payload remains `ScenarioState` on topic `scenario:state` — no change
- The `PagesScenarioController` and `PagesScenarioNarrative` components
  consume events only — no import changes needed

## References

- [packages/yaml-core/src/orchestration/types.ts] — primitive interfaces from #462
- [packages/pages-aria/src/scenario/types.ts] — current scenario model
- [packages/pages-aria/src/scenario/sectioned-runner.ts] — cooperative suspend pattern, runTo, speed control
- [packages/pages-aria/src/executor/command-executor.ts] — aria step execution
- [packages/pages-aria/src/tutorial/tutorial-host.ts] — primary consumer (PagesTutorialHost)
- [packages/pages-aria/src/server/scenario-handler.ts] — server-driven execution (out of scope, coexists)
- [packages/pages-aria/src/controller/scenario-connection-controller.ts] — ScenarioState, OutlineNode types
- [specs/issue-462-ts-orchestration-primitives/decisions.md] — D1-D6 (DES, virtual time, main-as-queue)
- [GitHub #461] — wire DataTrigger and TimeTrigger
- [GitHub #463] — YAML binding for orchestration primitives
- [platform #412] — DES trace validating cooperative model
- Design decisions D1-D5 in `decisions.md`
