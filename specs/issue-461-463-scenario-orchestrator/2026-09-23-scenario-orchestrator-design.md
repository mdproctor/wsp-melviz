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

**Out of scope:**
- GraphQL executor implementation (interface only — wiring is a later issue)
- Simulation data injection (uses the same scheduler but injection logic is separate)
- Visual regression testing of tutorial playback

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
├── command-executor.ts         # EXISTING: aria step execution (becomes AriaExecutor)
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
`advance(delta)` adds `delta * speed` to the counter. At `speed=Infinity`
(test mode), all delays resolve instantly — the scheduler advances virtual
time to the next wake point without real waiting.

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
  while (hasActiveQueues()) {
    // 1. Run all ready queues — one step each
    for (const queue of readyQueues()) {
      const step = queue.steps[queue.position];
      await dispatchStep(step, queue);
      queue.position++;
      if (queue.position >= queue.steps.length) {
        queue.state = 'done';
        resolveParentIfChildrenDone(queue);
      }
    }

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

    // 4. Yield to browser
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
  runTo(sectionIndex: number, stepIndex?: number): Promise<void>;
  setSpeed(multiplier: number): void;
  dispose(): void;

  readonly state: 'idle' | 'playing' | 'paused' | 'done';
  readonly outline: TutorialOutline;   // section/step structure for nav UI
  readonly clock: VirtualClock;        // for external time display

  addEventListener(type: string, handler: EventListener): void;
  removeEventListener(type: string, handler: EventListener): void;
}
```

`runTo(sectionIndex, stepIndex)` executes all steps up to the target
position in a single tick burst (skipping rAF yields between steps).
This preserves the sectioned-runner's navigation behavior.

`step()` executes exactly one step from the next ready queue and pauses.
For sub-step operations (line-by-line editor typing), the AriaExecutor
handles internal pacing — the scheduler sees it as one step.

### Events

Emitted on the runner's EventTarget (same `pages-event` pattern as
existing sectioned-runner):

| Event topic | When | Data |
|---|---|---|
| `scenario:state` | play/pause/step/done transitions | `{ state, section, step }` |
| `scenario:step` | after each step executes | `{ queue, step, virtualTime }` |
| `scenario:section` | entering a new section | `{ sectionIndex, title }` |
| `scenario:queue` | queue state change | `{ queueId, state, reason }` |

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
  channels?: Record<string, { capacity?: number }>;
  signals?: string[];     // pre-declared signal names
}

// Trigger types (#461):
interface DataTrigger {
  type: 'data';
  channel: string;        // wait for data on this channel
  condition?: string;     // optional filter expression
}

interface TimeTrigger {
  type: 'time';
  delay: string;          // virtual time delay (e.g. "5s")
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
Top-level constructs use their declared names.

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
    - feed-trades:
        - simulated: { dataset: trades, data: { symbol: AAPL, price: 150 } }
          delay: 100ms
        - signal: all-feeds-ready
    - feed-market:
        - simulated: { dataset: market, data: { index: SPX, value: 4500 } }
          delay: 50ms
        - signal: all-feeds-ready
    - feed-news:
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

- **DataTrigger:** checks `scope.channel(trigger.channel).isEmpty()` —
  returns true when the channel has data. If `condition` is set, uses
  `ConditionEvaluator` to filter.
- **TimeTrigger:** checks `clock.now() >= trigger.fireTime` — returns
  true when virtual time has passed the trigger's delay. `fireTime` is
  computed at bind time: `clock.now() + parseDuration(trigger.delay)`.
  If `repeat`, reset `fireTime` after each activation.

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
- `command-executor.ts` — adapted to implement `StepExecutor` interface
- `index.ts` — updated exports

### Consumers
The `ScenarioPlayer` component in `packages/pages-aria/src/components/`
currently imports from `sectioned-runner.ts`. It needs updating to use the
new `ScenarioRunner` interface from `scheduler.ts`. Same API shape
(play/pause/step/setSpeed/dispose), different import path.

## References

- [packages/yaml-core/src/orchestration/types.ts] — primitive interfaces from #462
- [packages/pages-aria/src/scenario/types.ts] — current scenario model
- [packages/pages-aria/src/scenario/sectioned-runner.ts] — cooperative suspend pattern, runTo, speed control
- [packages/pages-aria/src/scenario/command-executor.ts] — aria step execution
- [specs/issue-462-ts-orchestration-primitives/decisions.md] — D1-D6 (DES, virtual time, main-as-queue)
- [GitHub #461] — wire DataTrigger and TimeTrigger
- [GitHub #463] — YAML binding for orchestration primitives
- [platform #412] — DES trace validating cooperative model
- Design decisions D1-D5 in `decisions.md`
