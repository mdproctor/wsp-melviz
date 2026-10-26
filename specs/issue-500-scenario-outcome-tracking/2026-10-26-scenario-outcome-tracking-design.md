# Scenario Outcome Tracking — Design Spec

**Issue:** casehubio/casehub-pages#500
**Branch:** issue-500-scenario-outcome-tracking
**Date:** 2026-10-26

## Overview

Extend the scenario scheduler with structured outcome recording. Currently
scenario execution is fire-and-forget — the scheduler emits progress events
during execution but produces no structured result when done. This spec adds
an outcome data model in yaml-core, a passive collector that instruments the
scheduler, and emission via the existing pages-event mechanism.

The CBR bridge (engine#1190) is out of scope. Engine owns the unified
cross-runtime CBR integration — one bridge that consumes outcomes from both
TS pages and Java platform playbook runtimes. Pages' responsibility ends at
producing a well-structured outcome record and publishing it via push protocol.

The outcome panel UI (history list, detail view, comparison, trending) is
deferred to a separate issue. This spec covers the data foundation.

## Scope

**In scope:**
- Outcome data model (`ExecutionOutcome`, `StepOutcomeEntry`, `OutcomeSummary`)
- `OutcomeCollector` — passive observer class
- Scheduler integration — 3 touch points (~18 lines)
- `SimulationRuntime.allEntries()` — new method for journal collection
- `scenario:outcome` event emission
- Tests for collector and scheduler integration

**Out of scope:**
- CBR case creation/linkage (engine#1190)
- Ledger entry correlation (engine#1190)
- Outcome panel UI (separate issue)
- Outcome persistence/storage (consumer responsibility)

## Data Model

Location: `packages/yaml-core/src/step/outcome.ts`

### StepOutcomeEntry

One entry per step dispatched during execution.

```typescript
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
```

- `name`: step name from the definition, or positional key (`queueId:position`)
  when unnamed
- `queue`: the `StepQueue.id` that dispatched this step
- `kind`: step kind (`plugin`, `parallel`, `delay`, `signal-fire`,
  `await-signal`, `await-barrier`)
- `startedAt` / `completedAt`: epoch milliseconds
- `duration`: `completedAt - startedAt`
- `result`: terminal disposition of this step
- `error`: present only when `result === 'failure'`
- `decorators`: names of decorators applied to this step (e.g., `retry`,
  `loop`, `if`, `simulation`)

### OutcomeSummary

Aggregate step counts for quick consumption.

```typescript
export interface OutcomeSummary {
  readonly stepsExecuted: number;
  readonly stepsFailed: number;
  readonly stepsSkipped: number;
}
```

### ExecutionOutcome

The top-level outcome record emitted when execution ends.

```typescript
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
```

- `scenarioName`: from `scenario.playbook` or `'unnamed'`
- `finalState`: how execution ended
  - `completed`: all queues reached `done` state
  - `failed`: a step failure halted execution (retries exhausted)
  - `deadlocked`: 100 stale ticks with no progress and no data triggers
- `triggerSource`: optional identifier for what initiated this run
- `journal`: present only when a `SimulationRuntime` was active during the
  run; contains all `JournalEntry` records across all overlays

## OutcomeCollector

Location: `packages/yaml-core/src/step/outcome.ts`

A passive observer that accumulates step timing data during execution and
produces an `ExecutionOutcome` on finalization. The collector has no
knowledge of the scheduler's internals — it receives notifications via
two methods and produces the outcome via a third.

```typescript
export class OutcomeCollector {
  constructor(scenarioName: string);
  stepStarted(key: string, kind: string, queue: string): void;
  stepCompleted(key: string, name: string,
                result: 'success' | 'failure' | 'skipped',
                error?: { message: string; failureCategory?: string },
                decorators?: string[]): void;
  finalize(finalState: 'completed' | 'failed' | 'deadlocked',
           journal?: JournalEntry[]): ExecutionOutcome;
}
```

- `stepStarted`: records the start timestamp, step kind, and queue for
  a step identified by `key`
- `stepCompleted`: matches a started step by `key`, records the completion
  timestamp and result, appends to the internal entries array
- `finalize`: computes the summary counts and assembles the
  `ExecutionOutcome` record

The `key` parameter uses the scheduler's existing `posKey`
(`${queueId}:${position}`), which uniquely identifies each step dispatch.

## Scheduler Integration

Location: `packages/pages-aria/src/playbook/scheduler.ts`

Three touch points in the existing `createScheduler()` function:

### 1. Instantiation

At the top of `createScheduler()`, alongside the existing
`simulationRuntime` setup:

```typescript
const collector = new OutcomeCollector(scenario.playbook ?? 'unnamed');
```

### 2. Step dispatch

In the `tickLoop()` step dispatch section (inside the `ready.map()` block),
wrap step execution with collector notifications:

Before `dispatchStep()`:
```typescript
collector.stepStarted(posKey, step.kind, queue.id);
```

After result handling (success or failure):
```typescript
const stepName = ('name' in step ? step.name : null) ?? posKey;
const decoratorNames = 'decorators' in step
  ? Object.keys((step as { decorators: Record<string, unknown> }).decorators)
  : undefined;
collector.stepCompleted(posKey, stepName, resultDisposition, errorDetail, decoratorNames);
```

### 3. Completion — three exit paths

Each exit path emits the outcome event. Journal entries must be collected
*before* `simulationRuntime?.popAll()` — the normal completion path calls
`popAll()` which clears the overlay stack, so `hasActiveOverlay()` would
return `false` after that point.

```typescript
function emitOutcome(finalState: 'completed' | 'failed' | 'deadlocked'): void {
  const journal = simulationRuntime
    ? simulationRuntime.allEntries()
    : undefined;
  const outcome = collector.finalize(finalState,
    journal && journal.length > 0 ? journal : undefined);
  options.eventTarget.dispatchEvent(new CustomEvent('pages-event', {
    detail: { topic: 'scenario:outcome', payload: outcome },
  }));
}
```

On the normal completion path, `emitOutcome('completed')` must be called
before `simulationRuntime?.popAll()`.

Exit paths:
- **Normal completion** (all queues done): `emitOutcome('completed')`
- **Deadlock** (100 stale ticks): `emitOutcome('deadlocked')`
- **Fatal step failure** (retries exhausted): `emitOutcome('failed')`

## SimulationRuntime Extension

Location: `packages/yaml-core/src/step/simulation-runtime.ts`

Add one method to `SimulationRuntime`:

```typescript
allEntries(): readonly JournalEntry[] {
  const result: JournalEntry[] = [];
  for (const overlay of this.overlayStack) {
    result.push(...overlay.journal.entries());
  }
  return result;
}
```

This collects all journal entries across all overlays without filtering by
qualified name. Used by the scheduler at finalization to attach the full
journal to the outcome record.

## Event Contract

Topic: `scenario:outcome`
Channel: `pages-event` CustomEvent on the scheduler's EventTarget

```typescript
new CustomEvent('pages-event', {
  detail: {
    topic: 'scenario:outcome',
    payload: ExecutionOutcome,
  },
})
```

Emitted exactly once per scheduler run, after the final `scenario:state`
event. The push protocol forwards this to subscribers like any other
pages-event — engine's CBR bridge (#1190) will subscribe to this topic.

## Testing Strategy

### Unit tests — OutcomeCollector (`yaml-core`)

- `stepStarted` + `stepCompleted` produces correct `StepOutcomeEntry`
- Multiple steps accumulate in order
- `finalize` computes correct summary counts
- `finalize` with journal entries attaches them
- `finalize` without journal leaves field undefined
- Unmatched `stepCompleted` (no prior `stepStarted`) is a no-op
- Duration calculation correctness

### Integration tests — scheduler outcome emission (`pages-aria`)

- Normal completion emits `scenario:outcome` with `finalState: 'completed'`
- Step failure emits `scenario:outcome` with `finalState: 'failed'`
- Deadlock emits `scenario:outcome` with `finalState: 'deadlocked'`
- Outcome contains correct step count and per-step entries
- Simulation run includes journal entries in outcome
- Non-simulation run has no journal field
- Outcome event fires after the final `scenario:state` event

## References

- `packages/yaml-core/src/step/simulation-runtime.ts` — InvocationJournal, SimulationRuntime, JournalEntry
- `packages/yaml-core/src/step/walker.ts` — Result type, stepSuccess/stepFailure
- `packages/yaml-core/src/step/runner.ts` — Runner interface, DeadlineContext
- `packages/pages-aria/src/playbook/scheduler.ts` — createScheduler, tickLoop, dispatchStep
- `packages/pages-aria/src/playbook/types.ts` — Playbook, SchedulerOptions, SimulationBlock
- `packages/pages-aria/src/playbook/step-queue.ts` — StepQueue state machine
- `packages/yaml-core/src/orchestration/types.ts` — StepResultStore, ScenarioScope
- casehubio/casehub-pages#500 — issue
- casehubio/fsitrading#50 — epic C13 (trading YAML playbooks, CBR feedback loop)
- casehubio/casehub-engine#1190 — generic CBR outcome recording bridge (dependency, out of scope)
