# Scenario Outcome Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #500 — feat: scenario outcome tracking — post-execution analysis with CBR linkage

**Goal:** Add structured outcome recording to scenario execution so that
every scheduler run produces an `ExecutionOutcome` event consumable by
engine's CBR bridge.

**Architecture:** Outcome types and `OutcomeCollector` live in yaml-core
(runtime-agnostic, shared with Java platform). The scheduler in pages-aria
instantiates a collector, notifies it at step dispatch boundaries, and
emits a `scenario:outcome` pages-event at each of three exit paths.

**Tech Stack:** TypeScript, Vitest, yaml-core, pages-aria

## Global Constraints

- All types use `readonly` modifiers for immutable interfaces
- `JournalEntry` import from `./simulation-runtime.js` (same package)
- Outcome types re-exported from `packages/yaml-core/src/step/index.ts`
- Tests use Vitest (`describe`, `it`, `expect`, `vi`)
- Existing test helpers `pluginStep()`, `failingPluginStep()`,
  `testOptions()` in `scheduler.test.ts`

---

## Batch 1: Outcome types and collector (yaml-core)

### Task 1: Outcome data model and OutcomeCollector with tests

**Files:**
- Create: `packages/yaml-core/src/step/outcome.ts`
- Create: `packages/yaml-core/src/step/outcome.test.ts`
- Modify: `packages/yaml-core/src/step/index.ts` (add re-exports)

**Interfaces:**
- Consumes: `JournalEntry` from `./simulation-runtime.js`
- Produces: `StepOutcomeEntry`, `OutcomeSummary`, `ExecutionOutcome`
  (interfaces), `OutcomeCollector` (class with `stepStarted()`,
  `stepCompleted()`, `finalize()`)

- [ ] **Step 1: Write the failing tests**

Create `packages/yaml-core/src/step/outcome.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { OutcomeCollector } from './outcome.js';
import type { ExecutionOutcome, StepOutcomeEntry } from './outcome.js';
import type { JournalEntry } from './simulation-runtime.js';

describe('OutcomeCollector', () => {
  it('produces an outcome with a single successful step', () => {
    const collector = new OutcomeCollector('test-scenario');
    collector.stepStarted('q0:0', 'plugin', 'q0');
    collector.stepCompleted('q0:0', 'click', 'success');

    const outcome = collector.finalize('completed');

    expect(outcome.scenarioName).toBe('test-scenario');
    expect(outcome.finalState).toBe('completed');
    expect(outcome.summary.stepsExecuted).toBe(1);
    expect(outcome.summary.stepsFailed).toBe(0);
    expect(outcome.summary.stepsSkipped).toBe(0);
    expect(outcome.steps).toHaveLength(1);
    expect(outcome.steps[0]!.name).toBe('click');
    expect(outcome.steps[0]!.kind).toBe('plugin');
    expect(outcome.steps[0]!.queue).toBe('q0');
    expect(outcome.steps[0]!.result).toBe('success');
    expect(outcome.steps[0]!.duration).toBeGreaterThanOrEqual(0);
    expect(outcome.journal).toBeUndefined();
  });

  it('accumulates multiple steps in order', () => {
    const collector = new OutcomeCollector('multi');
    collector.stepStarted('q0:0', 'plugin', 'q0');
    collector.stepCompleted('q0:0', 'step-a', 'success');
    collector.stepStarted('q0:1', 'delay', 'q0');
    collector.stepCompleted('q0:1', 'step-b', 'success');
    collector.stepStarted('q1:0', 'plugin', 'q1');
    collector.stepCompleted('q1:0', 'step-c', 'failure',
      { message: 'boom', failureCategory: 'assertion' });

    const outcome = collector.finalize('failed');

    expect(outcome.steps).toHaveLength(3);
    expect(outcome.summary.stepsExecuted).toBe(2);
    expect(outcome.summary.stepsFailed).toBe(1);
    expect(outcome.steps[2]!.result).toBe('failure');
    expect(outcome.steps[2]!.error).toEqual({
      message: 'boom', failureCategory: 'assertion',
    });
  });

  it('counts skipped steps in summary', () => {
    const collector = new OutcomeCollector('skips');
    collector.stepStarted('q0:0', 'plugin', 'q0');
    collector.stepCompleted('q0:0', 'a', 'success');
    collector.stepStarted('q0:1', 'plugin', 'q0');
    collector.stepCompleted('q0:1', 'b', 'skipped');

    const outcome = collector.finalize('completed');

    expect(outcome.summary.stepsExecuted).toBe(1);
    expect(outcome.summary.stepsSkipped).toBe(1);
  });

  it('ignores stepCompleted with no prior stepStarted', () => {
    const collector = new OutcomeCollector('orphan');
    collector.stepCompleted('q0:0', 'orphan-step', 'success');

    const outcome = collector.finalize('completed');

    expect(outcome.steps).toHaveLength(0);
    expect(outcome.summary.stepsExecuted).toBe(0);
  });

  it('attaches journal entries when provided', () => {
    const collector = new OutcomeCollector('sim');
    const journal: JournalEntry[] = [{
      qualifiedName: 'agent.invoke',
      tenancyId: 'default',
      input: { prompt: 'hi' },
      output: { response: 'hello' },
      timestamp: Date.now(),
      dataRealism: 'DOMAIN_PLAUSIBLE',
    }];

    const outcome = collector.finalize('completed', journal);

    expect(outcome.journal).toHaveLength(1);
    expect(outcome.journal![0]!.qualifiedName).toBe('agent.invoke');
  });

  it('leaves journal undefined when no entries provided', () => {
    const collector = new OutcomeCollector('no-sim');
    const outcome = collector.finalize('completed');
    expect(outcome.journal).toBeUndefined();
  });

  it('computes duration from construction to finalize', () => {
    const collector = new OutcomeCollector('duration');
    const outcome = collector.finalize('completed');

    expect(outcome.duration).toBeGreaterThanOrEqual(0);
    expect(outcome.completedAt).toBeGreaterThanOrEqual(outcome.startedAt);
    expect(outcome.duration).toBe(outcome.completedAt - outcome.startedAt);
  });

  it('records decorator names on step entries', () => {
    const collector = new OutcomeCollector('deco');
    collector.stepStarted('q0:0', 'plugin', 'q0');
    collector.stepCompleted('q0:0', 'step-a', 'success', undefined,
      ['retry', 'if']);

    const outcome = collector.finalize('completed');

    expect(outcome.steps[0]!.decorators).toEqual(['retry', 'if']);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/yaml-core/src/step/outcome.test.ts`
Expected: FAIL — `outcome.js` does not exist

- [ ] **Step 3: Write the implementation**

Create `packages/yaml-core/src/step/outcome.ts`:

```typescript
import type { JournalEntry } from './simulation-runtime.js';

export interface StepOutcomeEntry {
  readonly name: string;
  readonly queue: string;
  readonly kind: string;
  readonly startedAt: number;
  readonly completedAt: number;
  readonly duration: number;
  readonly result: 'success' | 'failure' | 'skipped';
  readonly error?: { readonly message: string; readonly failureCategory?: string };
  readonly decorators?: readonly string[];
}

export interface OutcomeSummary {
  readonly stepsExecuted: number;
  readonly stepsFailed: number;
  readonly stepsSkipped: number;
}

export interface ExecutionOutcome {
  readonly scenarioName: string;
  readonly startedAt: number;
  readonly completedAt: number;
  readonly duration: number;
  readonly finalState: 'completed' | 'failed' | 'deadlocked';
  readonly triggerSource?: string;
  readonly summary: OutcomeSummary;
  readonly steps: readonly StepOutcomeEntry[];
  readonly journal?: readonly JournalEntry[];
}

export class OutcomeCollector {
  private readonly scenarioName: string;
  private readonly startedAt: number;
  private readonly stepEntries: StepOutcomeEntry[] = [];
  private readonly activeSteps = new Map<string, {
    startedAt: number; kind: string; queue: string;
  }>();

  constructor(scenarioName: string) {
    this.scenarioName = scenarioName;
    this.startedAt = Date.now();
  }

  stepStarted(key: string, kind: string, queue: string): void {
    this.activeSteps.set(key, { startedAt: Date.now(), kind, queue });
  }

  stepCompleted(
    key: string,
    name: string,
    result: 'success' | 'failure' | 'skipped',
    error?: { message: string; failureCategory?: string },
    decorators?: string[],
  ): void {
    const active = this.activeSteps.get(key);
    if (!active) return;
    this.activeSteps.delete(key);
    const completedAt = Date.now();
    this.stepEntries.push({
      name,
      queue: active.queue,
      kind: active.kind,
      startedAt: active.startedAt,
      completedAt,
      duration: completedAt - active.startedAt,
      result,
      error,
      decorators,
    });
  }

  finalize(
    finalState: 'completed' | 'failed' | 'deadlocked',
    journal?: JournalEntry[],
  ): ExecutionOutcome {
    const completedAt = Date.now();
    return {
      scenarioName: this.scenarioName,
      startedAt: this.startedAt,
      completedAt,
      duration: completedAt - this.startedAt,
      finalState,
      summary: {
        stepsExecuted: this.stepEntries.filter(s => s.result === 'success').length,
        stepsFailed: this.stepEntries.filter(s => s.result === 'failure').length,
        stepsSkipped: this.stepEntries.filter(s => s.result === 'skipped').length,
      },
      steps: this.stepEntries,
      journal: journal && journal.length > 0 ? journal : undefined,
    };
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/yaml-core/src/step/outcome.test.ts`
Expected: all 7 tests PASS

- [ ] **Step 5: Add re-exports to yaml-core index**

Add to `packages/yaml-core/src/step/index.ts` at the end, before the
`ChainedServiceRegistry` export:

```typescript
export type { StepOutcomeEntry, OutcomeSummary, ExecutionOutcome } from './outcome.js';
export { OutcomeCollector } from './outcome.js';
```

- [ ] **Step 6: Add `allEntries()` to SimulationRuntime**

Add to `SimulationRuntime` class in
`packages/yaml-core/src/step/simulation-runtime.ts`, after the existing
`allEntriesFor()` method:

```typescript
allEntries(): readonly JournalEntry[] {
  const result: JournalEntry[] = [];
  for (const overlay of this.overlayStack) {
    result.push(...overlay.journal.entries());
  }
  return result;
}
```

- [ ] **Step 7: Add test for `allEntries()`**

Add to `packages/yaml-core/src/step/simulation-runtime.test.ts` inside
the `SimulationRuntime` describe block:

```typescript
it('allEntries collects entries across all overlays', () => {
  const runtime = new SimulationRuntime();
  const config1: SimulationConfig = { strategies: {}, corpus: [], capture: [] };
  const config2: SimulationConfig = { strategies: {}, corpus: [], capture: [] };

  runtime.pushOverlay(config1);
  runtime.recordJournal('a.op', '', {}, {}, 'DOMAIN_PLAUSIBLE');
  runtime.pushOverlay(config2);
  runtime.recordJournal('b.op', '', {}, {}, 'RECORDED_REAL');

  const all = runtime.allEntries();
  expect(all).toHaveLength(2);
  expect(all[0]!.qualifiedName).toBe('a.op');
  expect(all[1]!.qualifiedName).toBe('b.op');
});
```

- [ ] **Step 8: Run all yaml-core step tests**

Run: `yarn vitest run packages/yaml-core/src/step/`
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git add packages/yaml-core/src/step/outcome.ts \
       packages/yaml-core/src/step/outcome.test.ts \
       packages/yaml-core/src/step/index.ts \
       packages/yaml-core/src/step/simulation-runtime.ts \
       packages/yaml-core/src/step/simulation-runtime.test.ts
git commit -m "feat(yaml-core): add outcome data model and collector for scenario execution tracking

Adds ExecutionOutcome, StepOutcomeEntry, OutcomeSummary interfaces and
OutcomeCollector class. Adds SimulationRuntime.allEntries() for
cross-overlay journal collection.

Refs #500"
```

## Batch 2: Scheduler integration (pages-aria)

### Task 2: Wire OutcomeCollector into scheduler and emit scenario:outcome

**Files:**
- Modify: `packages/pages-aria/src/playbook/scheduler.ts`
- Modify: `packages/pages-aria/src/playbook/scheduler.test.ts`

**Interfaces:**
- Consumes: `OutcomeCollector` from `@casehubio/yaml-core/step`,
  `OutcomeCollector.stepStarted(key, kind, queue)`,
  `OutcomeCollector.stepCompleted(key, name, result, error?, decorators?)`,
  `OutcomeCollector.finalize(finalState, journal?)` → `ExecutionOutcome`,
  `SimulationRuntime.allEntries()` → `readonly JournalEntry[]`
- Produces: `scenario:outcome` pages-event CustomEvent with
  `ExecutionOutcome` payload

- [ ] **Step 1: Write integration tests for outcome emission**

Add to `packages/pages-aria/src/playbook/scheduler.test.ts`. Use the
existing `pluginStep()`, `failingPluginStep()`, and `testOptions()`
helpers:

```typescript
describe('outcome emission', () => {
  it('emits scenario:outcome with completed on normal finish', async () => {
    const calls: unknown[] = [];
    const scenario = {
      playbook: 'test-playbook',
      steps: [pluginStep('click', calls)],
    };
    const opts = testOptions();
    const outcomes: any[] = [];
    opts.eventTarget.addEventListener('pages-event', (e: Event) => {
      const detail = (e as CustomEvent).detail;
      if (detail.topic === 'scenario:outcome') outcomes.push(detail.payload);
    });

    const runner = createScheduler(scenario, opts);
    runner.play();
    await vi.waitFor(() => { expect(runner.state).toBe('done'); });

    expect(outcomes).toHaveLength(1);
    expect(outcomes[0].scenarioName).toBe('test-playbook');
    expect(outcomes[0].finalState).toBe('completed');
    expect(outcomes[0].summary.stepsExecuted).toBe(1);
    expect(outcomes[0].summary.stepsFailed).toBe(0);
    expect(outcomes[0].steps).toHaveLength(1);
    expect(outcomes[0].journal).toBeUndefined();
  });

  it('emits scenario:outcome with failed on step failure', async () => {
    const scenario = {
      playbook: 'fail-playbook',
      steps: [failingPluginStep('bad-step', 'something broke')],
    };
    const opts = testOptions();
    const outcomes: any[] = [];
    opts.eventTarget.addEventListener('pages-event', (e: Event) => {
      const detail = (e as CustomEvent).detail;
      if (detail.topic === 'scenario:outcome') outcomes.push(detail.payload);
    });

    const runner = createScheduler(scenario, opts);
    runner.play();
    await vi.waitFor(() => { expect(runner.state).toBe('done'); });

    expect(outcomes).toHaveLength(1);
    expect(outcomes[0].finalState).toBe('failed');
    expect(outcomes[0].summary.stepsFailed).toBe(1);
    expect(outcomes[0].steps[0].error.message).toBe('something broke');
  });

  it('includes journal entries when simulation is active', async () => {
    const scenario = {
      playbook: 'sim-playbook',
      simulation: {
        strategies: { 'domain.op': 'sequential' },
        corpus: [{ inline: { 'domain.op': [{ input: {}, output: { v: 1 } }] } }],
      },
      steps: [{
        kind: 'plugin' as const,
        name: null,
        params: { domain: 'domain', operation: 'op' },
        decorators: {},
        entry: {
          qualifiedName: 'graphql-domain',
          definition: { name: 'graphql-domain', inputs: {}, outputs: {} },
          action: {
            async execute(p: Record<string, unknown>, services: ServiceRegistry) {
              const rt = services.lookup<SimulationRuntime>({ name: 'SimulationRuntime' });
              const resolved = rt.resolve('domain.op', p.params ?? {});
              if (resolved) rt.recordJournal('domain.op', '', p.params ?? {}, resolved.output, resolved.realism);
              return stepSuccess(resolved?.output as Record<string, unknown> ?? {});
            },
          },
        },
      }],
    };
    const opts = testOptions();
    const outcomes: any[] = [];
    opts.eventTarget.addEventListener('pages-event', (e: Event) => {
      const detail = (e as CustomEvent).detail;
      if (detail.topic === 'scenario:outcome') outcomes.push(detail.payload);
    });

    const runner = createScheduler(scenario, opts);
    runner.play();
    await vi.waitFor(() => { expect(runner.state).toBe('done'); });

    expect(outcomes).toHaveLength(1);
    expect(outcomes[0].journal).toBeDefined();
    expect(outcomes[0].journal.length).toBeGreaterThanOrEqual(1);
  });

  it('outcome event fires exactly once per run', async () => {
    const calls: unknown[] = [];
    const scenario = {
      playbook: 'multi-step',
      steps: [
        pluginStep('a', calls),
        pluginStep('b', calls),
        pluginStep('c', calls),
      ],
    };
    const opts = testOptions();
    let outcomeCount = 0;
    opts.eventTarget.addEventListener('pages-event', (e: Event) => {
      if ((e as CustomEvent).detail.topic === 'scenario:outcome') outcomeCount++;
    });

    const runner = createScheduler(scenario, opts);
    runner.play();
    await vi.waitFor(() => { expect(runner.state).toBe('done'); });

    expect(outcomeCount).toBe(1);
  });

  it('uses unnamed when playbook has no name', async () => {
    const calls: unknown[] = [];
    const scenario = { steps: [pluginStep('click', calls)] };
    const opts = testOptions();
    const outcomes: any[] = [];
    opts.eventTarget.addEventListener('pages-event', (e: Event) => {
      const detail = (e as CustomEvent).detail;
      if (detail.topic === 'scenario:outcome') outcomes.push(detail.payload);
    });

    const runner = createScheduler(scenario, opts);
    runner.play();
    await vi.waitFor(() => { expect(runner.state).toBe('done'); });

    expect(outcomes[0].scenarioName).toBe('unnamed');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/pages-aria/src/playbook/scheduler.test.ts`
Expected: FAIL — no `scenario:outcome` events emitted

- [ ] **Step 3: Add import and instantiation to scheduler**

In `packages/pages-aria/src/playbook/scheduler.ts`:

Add import at the top with existing imports:
```typescript
import { OutcomeCollector } from '@casehubio/yaml-core/step';
```

Inside `createScheduler()`, after the `simulationRuntime` setup
(after line ~67 `let runnerState`), add:
```typescript
const collector = new OutcomeCollector(scenario.playbook ?? 'unnamed');
```

- [ ] **Step 4: Add emitOutcome helper function**

Inside `createScheduler()`, after the existing `emitQueueEvent()`
function (after line ~113), add:

```typescript
function emitOutcome(finalState: 'completed' | 'failed' | 'deadlocked'): void {
  const journal = simulationRuntime
    ? simulationRuntime.allEntries()
    : undefined;
  const outcome = collector.finalize(finalState,
    journal && journal.length > 0 ? journal as JournalEntry[] : undefined);
  options.eventTarget.dispatchEvent(new CustomEvent('pages-event', {
    detail: { topic: 'scenario:outcome', payload: outcome },
  }));
}
```

Add to the import from yaml-core:
```typescript
import type { JournalEntry } from '@casehubio/yaml-core/step';
```

- [ ] **Step 5: Instrument step dispatch in tickLoop**

In the `tickLoop()` function, inside the `ready.map()` block, find:
```typescript
const result = await dispatchStep(step, queue);
emitStepEvent(queue, step);
```

Add before `dispatchStep()`:
```typescript
collector.stepStarted(posKey, step.kind, queue.id);
```

After the existing failure handling block (the `if (result && result.kind === 'failure')` block that ends with `return;`), the step was successful. Before the `let advancePosition` line, add:
```typescript
const completedStepName = ('name' in step ? step.name : null) ?? posKey;
const decoratorNames = 'decorators' in step
  ? Object.keys((step as { decorators: Record<string, unknown> }).decorators).filter(k => (step as any).decorators[k] != null)
  : undefined;
collector.stepCompleted(posKey, completedStepName, 'success', undefined,
  decoratorNames && decoratorNames.length > 0 ? decoratorNames : undefined);
```

In the failure path (inside the `else` branch where `retryCount >= retryMax`), after `resolveParentIfChildrenDone(queue);` and before `return;`, add:
```typescript
const failDecNames = 'decorators' in step
  ? Object.keys((step as { decorators: Record<string, unknown> }).decorators).filter(k => (step as any).decorators[k] != null)
  : undefined;
collector.stepCompleted(posKey, stepName, 'failure',
  { message: result.message, failureCategory: result.failureCategory },
  failDecNames && failDecNames.length > 0 ? failDecNames : undefined);
```

- [ ] **Step 6: Add outcome emission at the three exit paths**

**Normal completion** — in `tickLoop()`, find:
```typescript
runnerState = 'done';
simulationRuntime?.popAll();
emitState();
```
Change to:
```typescript
runnerState = 'done';
emitOutcome('completed');
simulationRuntime?.popAll();
emitState();
```

**Deadlock** — in the deadlock detection block, find:
```typescript
if (staleTicks > 100) {
  runnerState = 'done';
```
After `runnerState = 'done';`, add:
```typescript
emitOutcome('deadlocked');
```

**Fatal step failure** — this is trickier. The failure path inside the
step dispatch doesn't immediately end the scheduler — it marks the queue
as `done` and continues. The scheduler ends when `hasActiveQueues()`
returns false. The `emitOutcome('failed')` call should happen when the
main while loop exits due to a failure having occurred.

To handle this: add a `let hadFailure = false;` variable near `let staleTicks`. In the failure path (where `queue.state = 'done'` after retries exhausted), set `hadFailure = true;`.

Then change the normal completion to:
```typescript
runnerState = 'done';
emitOutcome(hadFailure ? 'failed' : 'completed');
simulationRuntime?.popAll();
emitState();
```

This consolidates the completed/failed exit into one — the `hadFailure`
flag distinguishes them. The deadlock exit remains separate (it returns
early from tickLoop).

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn vitest run packages/pages-aria/src/playbook/scheduler.test.ts`
Expected: all tests PASS (both existing and new outcome tests)

- [ ] **Step 8: Run full test suite**

Run: `yarn vitest run packages/yaml-core/src/step/ packages/pages-aria/src/playbook/`
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-aria/src/playbook/scheduler.ts \
       packages/pages-aria/src/playbook/scheduler.test.ts
git commit -m "feat(scheduler): emit scenario:outcome event with execution results

Integrates OutcomeCollector into the scheduler. Emits a scenario:outcome
pages-event at each exit path (completed, failed, deadlocked) with
per-step timing, summary counts, and optional simulation journal.

Refs #500"
```

## References

- [2026-10-26-scenario-outcome-tracking-design.md] — design spec this plan implements
- `packages/yaml-core/src/step/simulation-runtime.ts` — InvocationJournal, SimulationRuntime
- `packages/yaml-core/src/step/simulation-runtime.test.ts` — existing test patterns
- `packages/yaml-core/src/step/walker.ts` — Result type
- `packages/yaml-core/src/step/index.ts` — barrel exports
- `packages/pages-aria/src/playbook/scheduler.ts:44-440` — createScheduler, tickLoop, dispatchStep
- `packages/pages-aria/src/playbook/scheduler.test.ts:1-58` — test helpers and patterns
- `packages/pages-aria/src/playbook/types.ts` — Playbook, SimulationBlock types
- GitHub #500 — focal issue
