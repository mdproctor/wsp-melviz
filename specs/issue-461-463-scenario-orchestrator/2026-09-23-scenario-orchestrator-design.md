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
  signal, wait, retry, loop, state machine, channel, trigger)
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

Speed controls *yield timing* (how long the scheduler waits between ticks
in real time), not virtual time advancement:
- speed=1: yield uses `rAF` (~16ms real time per tick)
- speed=2: yield uses `setTimeout(0)` (ticks run as fast as the browser allows)
- speed=Infinity (test mode): yield resolves immediately, all delays complete
  in zero real time
- speed=0: equivalent to `pause()` — the tick loop suspends

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

  while (hasActiveQueues()) {
    if (disposed) return;
    const snapshotBefore = queueStateSnapshot();

    // 1. Run all ready queues concurrently — one step each
    const ready = readyQueues();
    await Promise.all(ready.map(async (queue) => {
      const step = queue.steps[queue.position];
      try {
        await dispatchStep(step, queue);
        queue.position++;
        if (queue.position >= queue.steps.length) {
          queue.state = 'done';
          resolveParentIfChildrenDone(queue);
        }
      } catch (err) {
        scope.resultStore().recordFailure(step.name ?? `queue:${queue.id}:${queue.position}`, {
          message: (err as Error).message, stepName: step.name ?? '',
        });
        if (step.retry && step.retryCount < step.retry.max) {
          step.retryCount++;
          // don't advance position — retry same step next tick
        } else {
          queue.state = 'done';
          emitError(queue, step, err);
        }
      }
    }));

    // 2. Advance virtual time to next wake point
    const nextWake = earliestWakeTime();
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

    // 4. Deadlock detection — if no state changed, count stale ticks
    if (queueStateEquals(snapshotBefore, queueStateSnapshot())) {
      staleTicks++;
      if (staleTicks > 100) {
        emitError(null, null, new Error('Scheduler deadlock: no progress for 100 ticks'));
        return;
      }
    } else {
      staleTicks = 0;
    }

    // 5. Yield to browser (timing affected by speed)
    await tick(); // rAF in browser, Promise.resolve() in tests
  }
}
```

When a step encounters an orchestration construct (concurrent block,
signal wait, semaphore acquire), the scheduler handles it:

- **concurrent:** creates child queues, sets parent queue to blocked
  until all children are done
- **delay/wait:** sets `wakeTime` on the queue, transitions to blocked
- **semaphore.acquire:** calls the primitive; if the Promise doesn't
  resolve synchronously, queue transitions to blocked with `blockReason`
- **signal.await:** same as semaphore — queue blocks on the Promise

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
  conditionEvaluator: ConditionEvaluator;  // from yaml-core/orchestration
}
```

The scheduler maintains a `StepExecutor[]` registry. For each step, it
finds the first executor where `canExecute(step)` returns true and calls
`execute()`. The existing `command-executor.ts` becomes `AriaExecutor`.

### Public API

```typescript
interface ScenarioRunner {
  play(): void;
  pause(): void;
  step(): Promise<void>;      // execute one step, then pause
  runTo(sectionTitle: string): Promise<void>;
  setSpeed(multiplier: number): void;
  dispose(): void;

  readonly state: 'idle' | 'playing' | 'paused' | 'done';
  readonly outline: OutlineNode[];     // section/step structure for nav UI (from scenario-connection-controller.ts)
  readonly clock: VirtualClock;        // for external time display

  addEventListener(type: string, handler: EventListener): void;
  removeEventListener(type: string, handler: EventListener): void;
}
```

`runTo(sectionTitle)` finds the section by title, resets the scheduler
to execute from that section's first step, and runs in a single tick
burst (skipping rAF yields). This matches the existing `TutorialRunner`
API consumed by `PagesTutorialHost`.

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
signals all signals, releases all primitives), and removes all event listeners.
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
| { delivery: 'orchestration'; construct: 'wait'; signal?: string; barrier?: string; timeout?: string }
| { delivery: 'orchestration'; construct: 'delay'; duration: string }

// Step-level decorators (on any delivery type):
interface StepDecorators {
  mutex?: string;
  retry?: RetryDirective;
  loop?: LoopDirective;
  when?: string;          // condition guard
  timeout?: string;       // step-level timeout
}

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
   - `signal:` / `wait:` → inline primitive operations
   - Step decorators (mutex, retry, loop) → wrap in orchestration logic
3. Triggered steps → create suspended queues with trigger conditions
4. Return `{ scope, queues, triggers }`

Inline constructs create anonymous entries in ScenarioScope (e.g.
`mutex: db-write` → `scope.semaphore('__anon_mutex_db-write', 1)`).
Top-level constructs use their declared names. The binder validates
that no top-level `orchestration:` name starts with `__anon_` — this
prefix is reserved for inline-generated primitives.

`when:` guards are evaluated via `ConditionEvaluator` from yaml-core.
The evaluator delegates to `isTruthy()` for simple boolean checks and
to a pluggable expression delegate for comparisons:
- `${result.step-name.field} == 'value'` — checks StepResultStore
- `${scope.signal-name.signalled}` — checks signal state
The `loop.until` condition uses the same evaluator.

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
  - wait: { signal: data-loaded, timeout: 30s }
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
  - wait: { barrier: all-feeds-ready }
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
- `parser.ts` — extended to recognize orchestration YAML constructs
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
export type { ScenarioRunner } from './scheduler.js';
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
