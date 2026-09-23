# Scenario Orchestrator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #461 — feat: wire DataTrigger and TimeTrigger evaluation in ScenarioOrchestrator
**Issue group:** #461, #463

**Goal:** Replace the sequential scenario runners with a DES scheduler supporting concurrent step queues, virtual time, orchestration primitives, YAML binding, and trigger evaluation.

**Architecture:** DES scheduler with virtual clock manages step queues (ready/suspended/blocked/done). YAML parser extended to recognize orchestration constructs. YamlBinder transforms parsed model into ScenarioScope + queue tree. StepExecutor strategy pattern dispatches by delivery type. Existing runners deleted, TutorialRunner-shaped API preserved.

**Tech Stack:** TypeScript, vitest, @casehubio/yaml-core/orchestration (from #462)

## Global Constraints

- All imports use `.js` extension suffix
- Tests use vitest with explicit `import { describe, it, expect } from 'vitest'`
- Test files co-located: `<name>.test.ts` alongside `<name>.ts`
- `await:` not `wait:` for orchestration — avoids collision with ARIA `wait` action
- `__anon_` prefix reserved for inline-generated primitives
- speed=0 is invalid — use `pause()` instead
- Injectable `tick` and `delay` functions for testability
- Existing `ScenarioState` event shape preserved for backward compatibility

---

## Batch 1: Foundation — VirtualClock, StepQueue, StepExecutor interface

Self-contained types with no cross-dependencies. After this batch,
the scheduler's building blocks are available and tested.

### Task 1: VirtualClock

**Files:**
- Create: `packages/pages-aria/src/scenario/virtual-clock.ts`
- Test: `packages/pages-aria/src/scenario/virtual-clock.test.ts`

**Interfaces:**
- Produces: `VirtualClock` interface, `DefaultVirtualClock` class
  - `now(): number`
  - `advance(deltaMs: number): void`
  - `speed(): number`
  - `setSpeed(multiplier: number): void`
  - `pause(): void`
  - `resume(): void`
  - `isPaused(): boolean`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultVirtualClock } from './virtual-clock.js';

describe('DefaultVirtualClock', () => {
  it('starts at time zero', () => {
    const clock = new DefaultVirtualClock();
    expect(clock.now()).toBe(0);
  });

  it('advances by exact delta', () => {
    const clock = new DefaultVirtualClock();
    clock.advance(100);
    expect(clock.now()).toBe(100);
    clock.advance(50);
    expect(clock.now()).toBe(150);
  });

  it('defaults to speed 1', () => {
    const clock = new DefaultVirtualClock();
    expect(clock.speed()).toBe(1);
  });

  it('setSpeed updates speed', () => {
    const clock = new DefaultVirtualClock();
    clock.setSpeed(2);
    expect(clock.speed()).toBe(2);
  });

  it('setSpeed rejects zero', () => {
    const clock = new DefaultVirtualClock();
    expect(() => clock.setSpeed(0)).toThrow();
  });

  it('setSpeed accepts Infinity', () => {
    const clock = new DefaultVirtualClock();
    clock.setSpeed(Infinity);
    expect(clock.speed()).toBe(Infinity);
  });

  it('pause and resume', () => {
    const clock = new DefaultVirtualClock();
    expect(clock.isPaused()).toBe(false);
    clock.pause();
    expect(clock.isPaused()).toBe(true);
    clock.resume();
    expect(clock.isPaused()).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/scenario/virtual-clock.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: FAIL — module not found

- [ ] **Step 3: Implement virtual-clock.ts**

```typescript
export interface VirtualClock {
  now(): number;
  advance(deltaMs: number): void;
  speed(): number;
  setSpeed(multiplier: number): void;
  pause(): void;
  resume(): void;
  isPaused(): boolean;
}

export class DefaultVirtualClock implements VirtualClock {
  private _time = 0;
  private _speed = 1;
  private _paused = false;

  now(): number { return this._time; }

  advance(deltaMs: number): void {
    this._time += deltaMs;
  }

  speed(): number { return this._speed; }

  setSpeed(multiplier: number): void {
    if (multiplier <= 0 || Number.isNaN(multiplier)) {
      throw new Error(`Speed must be > 0 or Infinity, got ${multiplier}. Use pause() for speed=0.`);
    }
    this._speed = multiplier;
  }

  pause(): void { this._paused = true; }
  resume(): void { this._paused = false; }
  isPaused(): boolean { return this._paused; }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/scenario/virtual-clock.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/scenario/virtual-clock.ts packages/pages-aria/src/scenario/virtual-clock.test.ts
git commit -m "feat(scenario): add VirtualClock — deterministic time for DES scheduler Refs #461"
```

### Task 2: StepQueue

**Files:**
- Create: `packages/pages-aria/src/scenario/step-queue.ts`
- Test: `packages/pages-aria/src/scenario/step-queue.test.ts`

**Interfaces:**
- Produces: `QueueState` type, `StepQueue` class
  - `state: QueueState` ('ready' | 'suspended' | 'blocked' | 'done')
  - `steps: OrchestratedStep[]`, `position: number`
  - `wakeTime?: number`, `blockReason?: Promise<void>`
  - `parent?: StepQueue`, `children: StepQueue[]`
  - `currentStep(): OrchestratedStep | undefined`
  - `advance(): void`
  - `isDone(): boolean`
  - `block(reason?: Promise<void>, wakeTime?: number): void`
  - `unblock(): void`
  - `suspend(trigger): void`
  - `activate(): void`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { StepQueue } from './step-queue.js';

describe('StepQueue', () => {
  const mockSteps = () => [
    { delivery: 'aria' as const, action: 'click', target: { role: 'button', name: 'A' } },
    { delivery: 'aria' as const, action: 'click', target: { role: 'button', name: 'B' } },
  ];

  it('starts ready at position 0', () => {
    const q = new StepQueue('main', mockSteps());
    expect(q.state).toBe('ready');
    expect(q.position).toBe(0);
  });

  it('currentStep returns step at position', () => {
    const q = new StepQueue('main', mockSteps());
    expect(q.currentStep()?.action).toBe('click');
  });

  it('advance increments position', () => {
    const q = new StepQueue('main', mockSteps());
    q.advance();
    expect(q.position).toBe(1);
  });

  it('isDone when position >= steps length', () => {
    const q = new StepQueue('main', mockSteps());
    q.advance();
    q.advance();
    expect(q.isDone()).toBe(true);
  });

  it('block sets state and wakeTime', () => {
    const q = new StepQueue('main', mockSteps());
    q.block(undefined, 500);
    expect(q.state).toBe('blocked');
    expect(q.wakeTime).toBe(500);
  });

  it('unblock sets state to ready', () => {
    const q = new StepQueue('main', mockSteps());
    q.block();
    q.unblock();
    expect(q.state).toBe('ready');
    expect(q.wakeTime).toBeUndefined();
  });

  it('suspend and activate', () => {
    const trigger = { type: 'time' as const, delay: '5s' };
    const q = new StepQueue('main', mockSteps());
    q.suspend(trigger);
    expect(q.state).toBe('suspended');
    expect(q.trigger).toBe(trigger);
    q.activate();
    expect(q.state).toBe('ready');
  });

  it('tracks parent-child relationships', () => {
    const parent = new StepQueue('main', []);
    const child = new StepQueue('branch-a', mockSteps(), parent);
    expect(child.parent).toBe(parent);
    expect(parent.children).toContain(child);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/scenario/step-queue.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: FAIL

- [ ] **Step 3: Implement step-queue.ts**

```typescript
import type { DataTrigger, TimeTrigger } from './types.js';

export type QueueState = 'ready' | 'suspended' | 'blocked' | 'done';

export class StepQueue {
  state: QueueState = 'ready';
  position = 0;
  wakeTime?: number;
  blockReason?: Promise<void>;
  trigger?: DataTrigger | TimeTrigger;
  children: StepQueue[] = [];

  constructor(
    public readonly id: string,
    public readonly steps: unknown[],
    public readonly parent?: StepQueue,
  ) {
    if (parent) parent.children.push(this);
  }

  currentStep(): unknown | undefined {
    return this.position < this.steps.length ? this.steps[this.position] : undefined;
  }

  advance(): void {
    this.position++;
  }

  isDone(): boolean {
    return this.position >= this.steps.length;
  }

  block(reason?: Promise<void>, wakeTime?: number): void {
    this.state = 'blocked';
    this.blockReason = reason;
    this.wakeTime = wakeTime;
  }

  unblock(): void {
    this.state = 'ready';
    this.wakeTime = undefined;
    this.blockReason = undefined;
  }

  suspend(trigger: DataTrigger | TimeTrigger): void {
    this.state = 'suspended';
    this.trigger = trigger;
  }

  activate(): void {
    this.state = 'ready';
    this.trigger = undefined;
  }
}
```

Note: the `steps` type uses `unknown[]` here — it will be refined to `OrchestratedStep[]` when types.ts is extended in Task 5. The queue is structurally independent of step types.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/scenario/step-queue.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/scenario/step-queue.ts packages/pages-aria/src/scenario/step-queue.test.ts
git commit -m "feat(scenario): add StepQueue — state machine for DES scheduler queues Refs #461"
```

### Task 3: StepExecutor interface and AriaExecutor adapter

**Files:**
- Create: `packages/pages-aria/src/scenario/step-executor.ts`
- Modify: `packages/pages-aria/src/executor/command-executor.ts` — wrap as AriaExecutor
- Test: `packages/pages-aria/src/scenario/step-executor.test.ts`

**Interfaces:**
- Consumes: existing `executeStep` from `command-executor.ts`
- Produces: `StepExecutor` interface, `ExecutionContext` interface, `AriaExecutor` class
  - `canExecute(step): boolean`
  - `execute(step, context): Promise<void>`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi } from 'vitest';
import { AriaExecutor } from './step-executor.js';

describe('AriaExecutor', () => {
  it('canExecute returns true for aria delivery', () => {
    const exec = new AriaExecutor(vi.fn());
    expect(exec.canExecute({ delivery: 'aria', action: 'click' } as any)).toBe(true);
  });

  it('canExecute returns false for non-aria delivery', () => {
    const exec = new AriaExecutor(vi.fn());
    expect(exec.canExecute({ delivery: 'graphql' } as any)).toBe(false);
    expect(exec.canExecute({ delivery: 'orchestration' } as any)).toBe(false);
  });

  it('execute delegates to command executor', async () => {
    const commandFn = vi.fn().mockResolvedValue(undefined);
    const exec = new AriaExecutor(commandFn);
    const step = { delivery: 'aria' as const, action: 'click', target: { role: 'button', name: 'OK' } };
    const ctx = { speed: 1, eventTarget: new EventTarget() } as any;
    await exec.execute(step, ctx);
    expect(commandFn).toHaveBeenCalledWith(step, ctx.eventTarget, ctx.speed);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/scenario/step-executor.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: FAIL

- [ ] **Step 3: Implement step-executor.ts**

```typescript
import type { ScenarioScope } from '@casehubio/yaml-core/orchestration';
import type { ConditionEvaluator } from '@casehubio/yaml-core/condition';
import type { VirtualClock } from './virtual-clock.js';

export interface ExecutionContext {
  scope: ScenarioScope;
  clock: VirtualClock;
  eventTarget: EventTarget;
  speed: number;
  conditionEvaluator: ConditionEvaluator;
}

export interface StepExecutor {
  canExecute(step: unknown): boolean;
  execute(step: unknown, context: ExecutionContext): Promise<void>;
}

export class AriaExecutor implements StepExecutor {
  constructor(
    private readonly _commandExecutor: (step: unknown, eventTarget?: EventTarget, speed?: number) => Promise<void>,
  ) {}

  canExecute(step: unknown): boolean {
    return (step as { delivery?: string }).delivery === 'aria';
  }

  execute(step: unknown, context: ExecutionContext): Promise<void> {
    return this._commandExecutor(step, context.eventTarget, context.speed);
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/scenario/step-executor.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/scenario/step-executor.ts packages/pages-aria/src/scenario/step-executor.test.ts
git commit -m "feat(scenario): add StepExecutor interface and AriaExecutor adapter Refs #461"
```

---

## Batch 2: Extended types, parser, YAML binder

Extends the scenario model with orchestration types and teaches the
parser to recognize them. After this batch, orchestration YAML parses
into typed models.

### Task 4: Extended scenario types

**Files:**
- Modify: `packages/pages-aria/src/scenario/types.ts` — add OrchestratedStep, StepDecorators, triggers, OrchestrationBlock

**Interfaces:**
- Produces: `OrchestratedStep`, `StepDecorators`, `DataTrigger`, `TimeTrigger`, `OrchestrationBlock`, `StateMachineDefinition`, orchestration construct step variants

- [ ] **Step 1: Write type definitions** (type-only — test via typecheck)

Add to `packages/pages-aria/src/scenario/types.ts`:

```typescript
import type { RetryDirective, LoopDirective } from '@casehubio/yaml-core/orchestration';

export interface StepDecorators {
  mutex?: string;
  retry?: RetryDirective;
  loop?: LoopDirective;
  when?: string;
  timeout?: string;
  delay?: string;
}

export type OrchestrationConstruct =
  | { delivery: 'orchestration'; construct: 'concurrent'; branches: Record<string, ScenarioStep[]> }
  | { delivery: 'orchestration'; construct: 'signal'; name: string }
  | { delivery: 'orchestration'; construct: 'await'; signal?: string; barrier?: string; timeout?: string }
  | { delivery: 'orchestration'; construct: 'delay'; duration: string };

export type OrchestratedStep = (ScenarioStep | OrchestrationConstruct) & {
  name?: string;
  decorators?: StepDecorators;
};

export interface DataTrigger {
  type: 'data';
  channel: string;
  condition?: string;
}

export interface TimeTrigger {
  type: 'time';
  delay: string;
  repeat?: boolean;
}

export interface StateMachineDefinition {
  initial: string;
  states: string[];
  transitions: Array<{ from: string; to: string; guard?: string }>;
  terminal?: string[];
}

export interface OrchestrationBlock {
  machines?: Record<string, StateMachineDefinition>;
  barriers?: Record<string, { count: number }>;
  quorums?: Record<string, { required: number; of: string[] }>;
  channels?: Record<string, { capacity?: number }>;
  signals?: string[];
}
```

- [ ] **Step 2: Run typecheck**

Run: `npx tsc --noEmit -p packages/pages-aria/tsconfig.json`
Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add packages/pages-aria/src/scenario/types.ts
git commit -m "feat(scenario): add orchestration types — OrchestratedStep, triggers, decorators Refs #463"
```

### Task 5: Parser extension — orchestration construct recognition

**Files:**
- Modify: `packages/pages-aria/src/scenario/parser.ts` — recognize orchestration keys
- Test: `packages/pages-aria/src/scenario/parser-orchestration.test.ts`

**Interfaces:**
- Consumes: types from Task 4
- Produces: `parseScenario()` now returns `OrchestratedStep[]` with decorators and orchestration constructs

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { parseScenario } from './parser.js';

describe('Parser — orchestration constructs', () => {
  it('parses concurrent block with named branches', () => {
    const yaml = `
scenario: test
steps:
  - concurrent:
      branch-a:
        - click: { role: button, name: A }
      branch-b:
        - click: { role: button, name: B }
    `;
    const result = parseScenario(yaml);
    const step = result.steps[0] as any;
    expect(step.delivery).toBe('orchestration');
    expect(step.construct).toBe('concurrent');
    expect(Object.keys(step.branches)).toEqual(['branch-a', 'branch-b']);
  });

  it('parses signal step', () => {
    const yaml = `
scenario: test
steps:
  - signal: go
    `;
    const result = parseScenario(yaml);
    const step = result.steps[0] as any;
    expect(step.delivery).toBe('orchestration');
    expect(step.construct).toBe('signal');
    expect(step.name).toBe('go');
  });

  it('parses await step with signal', () => {
    const yaml = `
scenario: test
steps:
  - await: { signal: data-loaded, timeout: 30s }
    `;
    const result = parseScenario(yaml);
    const step = result.steps[0] as any;
    expect(step.delivery).toBe('orchestration');
    expect(step.construct).toBe('await');
    expect(step.signal).toBe('data-loaded');
    expect(step.timeout).toBe('30s');
  });

  it('parses inline decorators on aria steps', () => {
    const yaml = `
scenario: test
steps:
  - click: { role: button, name: Submit }
    mutex: db-write
    retry: 3
    `;
    const result = parseScenario(yaml);
    const step = result.steps[0] as any;
    expect(step.delivery).toBe('aria');
    expect(step.decorators?.mutex).toBe('db-write');
    expect(step.decorators?.retry).toEqual({ type: 'simple', max: 3 });
  });

  it('parses delay decorator', () => {
    const yaml = `
scenario: test
steps:
  - click: { role: button, name: OK }
    delay: 100ms
    `;
    const result = parseScenario(yaml);
    const step = result.steps[0] as any;
    expect(step.decorators?.delay).toBe('100ms');
  });

  it('parses top-level orchestration block', () => {
    const yaml = `
scenario: test
orchestration:
  barriers:
    all-ready: { count: 3 }
  channels:
    trades: { capacity: 10 }
  signals: [go, stop]
steps:
  - click: { role: button, name: Start }
    `;
    const result = parseScenario(yaml);
    expect(result.orchestration?.barriers?.['all-ready']?.count).toBe(3);
    expect(result.orchestration?.channels?.['trades']?.capacity).toBe(10);
    expect(result.orchestration?.signals).toEqual(['go', 'stop']);
  });

  it('parses triggered steps', () => {
    const yaml = `
scenario: test
steps:
  - trigger:
      type: data
      channel: trades
    steps:
      - click: { role: button, name: Refresh }
    `;
    const result = parseScenario(yaml);
    const step = result.steps[0] as any;
    expect(step.delivery).toBe('orchestration');
    expect(step.construct).toBe('trigger');
    expect(step.trigger.type).toBe('data');
    expect(step.trigger.channel).toBe('trades');
    expect(step.steps).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/scenario/parser-orchestration.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: FAIL

- [ ] **Step 3: Extend parser.ts**

Add three new recognition paths in `parseSteps()`:

1. Check for orchestration keys (`concurrent:`, `signal:`, `await:`, `delay:`, `trigger:`) BEFORE the ARIA shorthand fallback
2. Check for non-ARIA delivery shorthands (`simulated:`, `graphql:`) BEFORE ARIA fallback
3. Extract decorator keys (`mutex`, `retry`, `loop`, `when`, `timeout`, `delay`) from step maps and store in `step.decorators`

Parse `retry:` values using `parseRetryDirective()` from `@casehubio/yaml-core/orchestration`.
Parse `loop:` values using `parseLoopDirective()`.
Parse `orchestration:` block at the scenario level and store on the result.

Add `trigger:` as a new orchestration construct that produces:
```typescript
{ delivery: 'orchestration', construct: 'trigger', trigger: DataTrigger | TimeTrigger, steps: OrchestratedStep[] }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/scenario/parser-orchestration.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: PASS

- [ ] **Step 5: Run existing parser tests to verify no regression**

Run: `npx vitest run src/scenario/scenario.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: ALL PASS (existing ARIA parsing unchanged)

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/src/scenario/parser.ts packages/pages-aria/src/scenario/parser-orchestration.test.ts
git commit -m "feat(scenario): extend parser with orchestration construct recognition Refs #463"
```

### Task 6: YamlBinder — ScenarioModel to ScenarioScope + queue tree

**Files:**
- Create: `packages/pages-aria/src/scenario/yaml-binder.ts`
- Test: `packages/pages-aria/src/scenario/yaml-binder.test.ts`

**Interfaces:**
- Consumes: `ScenarioScope` from yaml-core, types from Task 4, `StepQueue` from Task 2
- Produces: `bindScenario(scenario, scope): BindResult`
  - `BindResult = { queues: StepQueue[], triggers: Map<string, DataTrigger | TimeTrigger> }`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultScenarioScope } from '@casehubio/yaml-core/orchestration';
import { bindScenario } from './yaml-binder.js';

describe('YamlBinder', () => {
  it('creates a single queue for flat sequential steps', () => {
    const scenario = {
      scenario: 'test',
      steps: [
        { delivery: 'aria', action: 'click', target: { role: 'button', name: 'A' } },
        { delivery: 'aria', action: 'click', target: { role: 'button', name: 'B' } },
      ],
    };
    const scope = new DefaultScenarioScope();
    const result = bindScenario(scenario as any, scope);
    expect(result.queues).toHaveLength(1);
    expect(result.queues[0].steps).toHaveLength(2);
    expect(result.queues[0].id).toBe('main');
  });

  it('creates child queues for concurrent block', () => {
    const scenario = {
      scenario: 'test',
      steps: [
        {
          delivery: 'orchestration', construct: 'concurrent',
          branches: {
            'branch-a': [{ delivery: 'aria', action: 'click' }],
            'branch-b': [{ delivery: 'aria', action: 'click' }],
          },
        },
      ],
    };
    const scope = new DefaultScenarioScope();
    const result = bindScenario(scenario as any, scope);
    expect(result.queues).toHaveLength(1);
    const mainQueue = result.queues[0];
    expect(mainQueue.children).toHaveLength(2);
    expect(mainQueue.children[0].id).toBe('branch-a');
    expect(mainQueue.children[1].id).toBe('branch-b');
  });

  it('creates named primitives from orchestration block', () => {
    const scenario = {
      scenario: 'test',
      orchestration: {
        barriers: { 'all-ready': { count: 3 } },
        channels: { trades: { capacity: 10 } },
        signals: ['go'],
      },
      steps: [],
    };
    const scope = new DefaultScenarioScope();
    bindScenario(scenario as any, scope);
    expect(scope.latch('all-ready', 3).getCount()).toBe(3);
    expect(scope.channel('trades').isEmpty()).toBe(true);
    expect(scope.signal('go').isSignalled()).toBe(false);
  });

  it('creates anonymous semaphore for inline mutex', () => {
    const scenario = {
      scenario: 'test',
      steps: [
        { delivery: 'aria', action: 'click', decorators: { mutex: 'db-write' } },
      ],
    };
    const scope = new DefaultScenarioScope();
    bindScenario(scenario as any, scope);
    expect(scope.semaphore('__anon_mutex_db-write', 1).availablePermits()).toBe(1);
  });

  it('creates suspended queue for triggered steps', () => {
    const scenario = {
      scenario: 'test',
      steps: [
        {
          delivery: 'orchestration', construct: 'trigger',
          trigger: { type: 'data', channel: 'trades' },
          steps: [{ delivery: 'aria', action: 'click' }],
        },
      ],
    };
    const scope = new DefaultScenarioScope();
    const result = bindScenario(scenario as any, scope);
    const triggerQueue = result.queues.find(q => q.state === 'suspended');
    expect(triggerQueue).toBeDefined();
    expect(triggerQueue!.trigger).toEqual({ type: 'data', channel: 'trades' });
  });

  it('rejects __anon_ prefix in top-level orchestration names', () => {
    const scenario = {
      scenario: 'test',
      orchestration: { signals: ['__anon_bad'] },
      steps: [],
    };
    const scope = new DefaultScenarioScope();
    expect(() => bindScenario(scenario as any, scope)).toThrow('__anon_');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/scenario/yaml-binder.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: FAIL

- [ ] **Step 3: Implement yaml-binder.ts**

```typescript
import type { ScenarioScope } from '@casehubio/yaml-core/orchestration';
import { StepQueue } from './step-queue.js';
import type { OrchestratedStep, OrchestrationBlock, DataTrigger, TimeTrigger } from './types.js';

export interface BindResult {
  queues: StepQueue[];
  triggers: Map<string, DataTrigger | TimeTrigger>;
}

export function bindScenario(
  scenario: { orchestration?: OrchestrationBlock; steps?: OrchestratedStep[]; sections?: Array<{ steps: OrchestratedStep[] }> },
  scope: ScenarioScope,
): BindResult {
  if (scenario.orchestration) {
    bindOrchestrationBlock(scenario.orchestration, scope);
  }

  const allSteps = scenario.sections
    ? scenario.sections.flatMap(s => s.steps)
    : (scenario.steps ?? []);

  const triggers = new Map<string, DataTrigger | TimeTrigger>();
  const mainQueue = new StepQueue('main', []);
  const allQueues: StepQueue[] = [mainQueue];

  buildQueueTree(allSteps, mainQueue, allQueues, triggers, scope);

  return { queues: allQueues, triggers };
}

function bindOrchestrationBlock(block: OrchestrationBlock, scope: ScenarioScope): void {
  const allNames = [
    ...Object.keys(block.barriers ?? {}),
    ...Object.keys(block.channels ?? {}),
    ...(block.signals ?? []),
    ...Object.keys(block.machines ?? {}),
    ...Object.keys(block.quorums ?? {}),
  ];
  for (const name of allNames) {
    if (name.startsWith('__anon_')) {
      throw new Error(`Orchestration name '${name}' uses reserved '__anon_' prefix`);
    }
  }

  for (const [name, def] of Object.entries(block.barriers ?? {})) {
    scope.latch(name, def.count);
  }
  for (const [name, def] of Object.entries(block.quorums ?? {})) {
    scope.latch(name, def.required);
  }
  for (const [name, def] of Object.entries(block.channels ?? {})) {
    scope.channel(name, def.capacity);
  }
  for (const name of block.signals ?? []) {
    scope.signal(name);
  }
  for (const [name, def] of Object.entries(block.machines ?? {})) {
    const sm = scope.stateMachine(name, def.states, def.initial);
    if (def.terminal) {
      // Terminal states set via builder — but scope.stateMachine already built it.
      // For now, terminal states are informational — the YAML binding creates
      // all-to-all transitions. Selective transitions with guards are a
      // follow-up refinement.
    }
  }
}

function buildQueueTree(
  steps: OrchestratedStep[],
  currentQueue: StepQueue,
  allQueues: StepQueue[],
  triggers: Map<string, DataTrigger | TimeTrigger>,
  scope: ScenarioScope,
): void {
  for (const step of steps) {
    if (isOrchestration(step) && step.construct === 'concurrent') {
      for (const [branchName, branchSteps] of Object.entries(step.branches)) {
        const child = new StepQueue(branchName, branchSteps as unknown[], currentQueue);
        allQueues.push(child);
      }
      (currentQueue.steps as unknown[]).push(step);
    } else if (isOrchestration(step) && step.construct === 'trigger') {
      const triggerQueue = new StepQueue(`trigger-${triggers.size}`, (step as any).steps);
      triggerQueue.suspend((step as any).trigger);
      triggers.set(triggerQueue.id, (step as any).trigger);
      allQueues.push(triggerQueue);
    } else {
      if (step.decorators?.mutex) {
        scope.semaphore(`__anon_mutex_${step.decorators.mutex}`, 1);
      }
      (currentQueue.steps as unknown[]).push(step);
    }
  }
}

function isOrchestration(step: any): step is { delivery: 'orchestration'; construct: string } {
  return step.delivery === 'orchestration';
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/scenario/yaml-binder.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/scenario/yaml-binder.ts packages/pages-aria/src/scenario/yaml-binder.test.ts
git commit -m "feat(scenario): add YamlBinder — ScenarioModel to ScenarioScope + queue tree Refs #463"
```

---

## Batch 3: TriggerEvaluator and DES Scheduler core

The scheduler tick loop — the heart of the system. After this batch,
scenarios execute with concurrency, virtual time, and triggers.

### Task 7: TriggerEvaluator

**Files:**
- Create: `packages/pages-aria/src/scenario/trigger-evaluator.ts`
- Test: `packages/pages-aria/src/scenario/trigger-evaluator.test.ts`

**Interfaces:**
- Consumes: `ScenarioScope` from yaml-core, `VirtualClock` from Task 1
- Produces: `evaluateTrigger(trigger, scope, clock): boolean`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultScenarioScope } from '@casehubio/yaml-core/orchestration';
import { DefaultVirtualClock } from './virtual-clock.js';
import { evaluateTrigger } from './trigger-evaluator.js';
import type { DataTrigger, TimeTrigger } from './types.js';

describe('evaluateTrigger', () => {
  it('DataTrigger fires when channel has data', async () => {
    const scope = new DefaultScenarioScope();
    const clock = new DefaultVirtualClock();
    const ch = scope.channel<number>('trades');
    const trigger: DataTrigger = { type: 'data', channel: 'trades' };

    expect(evaluateTrigger(trigger, scope, clock)).toBe(false);
    await ch.send(42);
    expect(evaluateTrigger(trigger, scope, clock)).toBe(true);
  });

  it('TimeTrigger fires when virtual time passes fireTime', () => {
    const scope = new DefaultScenarioScope();
    const clock = new DefaultVirtualClock();
    const trigger: TimeTrigger & { fireTime: number } = {
      type: 'time', delay: '5s', fireTime: 5000,
    };

    expect(evaluateTrigger(trigger, scope, clock)).toBe(false);
    clock.advance(5000);
    expect(evaluateTrigger(trigger, scope, clock)).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/scenario/trigger-evaluator.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: FAIL

- [ ] **Step 3: Implement trigger-evaluator.ts**

```typescript
import type { ScenarioScope } from '@casehubio/yaml-core/orchestration';
import type { VirtualClock } from './virtual-clock.js';
import type { DataTrigger, TimeTrigger } from './types.js';

export function evaluateTrigger(
  trigger: (DataTrigger | TimeTrigger) & { fireTime?: number },
  scope: ScenarioScope,
  clock: VirtualClock,
): boolean {
  if (trigger.type === 'data') {
    return !scope.channel(trigger.channel).isEmpty();
  }
  if (trigger.type === 'time' && trigger.fireTime !== undefined) {
    return clock.now() >= trigger.fireTime;
  }
  return false;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/scenario/trigger-evaluator.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/scenario/trigger-evaluator.ts packages/pages-aria/src/scenario/trigger-evaluator.test.ts
git commit -m "feat(scenario): add TriggerEvaluator — DataTrigger and TimeTrigger activation Refs #461"
```

### Task 8: DES Scheduler — core tick loop with public API

**Files:**
- Create: `packages/pages-aria/src/scenario/scheduler.ts`
- Test: `packages/pages-aria/src/scenario/scheduler.test.ts`

**Interfaces:**
- Consumes: `VirtualClock` (Task 1), `StepQueue` (Task 2), `StepExecutor` (Task 3), `bindScenario` (Task 6), `evaluateTrigger` (Task 7)
- Produces: `createScheduler(scenario, options): ScenarioRunner`, `ScenarioRunner` interface, `SchedulerOptions` interface

This is the largest task. The test file covers: sequential execution, concurrent branches, pause/resume, step(), runTo(), setSpeed(), dispose(), semaphore coordination, latch barrier, signal coordination, trigger activation, deadlock detection, and retry/loop decorators.

- [ ] **Step 1: Write failing tests** (core subset — sequential, concurrent, pause/resume)

```typescript
import { describe, it, expect, vi } from 'vitest';
import { createScheduler } from './scheduler.js';
import type { StepExecutor } from './step-executor.js';

function mockExecutor(): { executor: StepExecutor; calls: unknown[] } {
  const calls: unknown[] = [];
  const executor: StepExecutor = {
    canExecute: (step: any) => step.delivery === 'aria',
    execute: async (step) => { calls.push(step); },
  };
  return { executor, calls };
}

function immediateOptions(executors: StepExecutor[]) {
  return {
    eventTarget: new EventTarget(),
    speed: Infinity,
    startPaused: true,
    executors,
  };
}

describe('DES Scheduler', () => {
  it('executes flat sequential steps in order', async () => {
    const { executor, calls } = mockExecutor();
    const scenario = {
      scenario: 'test',
      steps: [
        { delivery: 'aria', action: 'click', target: { role: 'button', name: 'A' } },
        { delivery: 'aria', action: 'click', target: { role: 'button', name: 'B' } },
      ],
    };
    const runner = createScheduler(scenario as any, immediateOptions([executor]));
    runner.play();
    await vi.waitFor(() => expect(runner.state).toBe('done'));
    expect(calls).toHaveLength(2);
    expect((calls[0] as any).target.name).toBe('A');
    expect((calls[1] as any).target.name).toBe('B');
  });

  it('pause stops execution, resume continues', async () => {
    const { executor, calls } = mockExecutor();
    const steps = Array.from({ length: 5 }, (_, i) => ({
      delivery: 'aria', action: 'click', target: { role: 'button', name: `btn-${i}` },
    }));
    const runner = createScheduler({ scenario: 'test', steps } as any, immediateOptions([executor]));
    runner.play();
    await vi.waitFor(() => calls.length >= 2);
    runner.pause();
    const countAtPause = calls.length;
    await new Promise(r => setTimeout(r, 50));
    expect(calls.length).toBe(countAtPause);
    runner.play();
    await vi.waitFor(() => runner.state === 'done');
    expect(calls).toHaveLength(5);
  });

  it('step() executes exactly one step', async () => {
    const { executor, calls } = mockExecutor();
    const scenario = {
      scenario: 'test',
      steps: [
        { delivery: 'aria', action: 'click', target: { role: 'button', name: 'A' } },
        { delivery: 'aria', action: 'click', target: { role: 'button', name: 'B' } },
      ],
    };
    const runner = createScheduler(scenario as any, immediateOptions([executor]));
    await runner.step();
    expect(calls).toHaveLength(1);
    expect(runner.state).toBe('paused');
  });

  it('concurrent branches execute with DES interleaving', async () => {
    const { executor, calls } = mockExecutor();
    const scenario = {
      scenario: 'test',
      steps: [{
        delivery: 'orchestration', construct: 'concurrent',
        branches: {
          'a': [
            { delivery: 'aria', action: 'click', target: { role: 'button', name: 'A1' } },
            { delivery: 'aria', action: 'click', target: { role: 'button', name: 'A2' } },
          ],
          'b': [
            { delivery: 'aria', action: 'click', target: { role: 'button', name: 'B1' } },
          ],
        },
      }],
    };
    const runner = createScheduler(scenario as any, immediateOptions([executor]));
    runner.play();
    await vi.waitFor(() => runner.state === 'done');
    expect(calls).toHaveLength(3);
  });

  it('dispose stops execution and cleans up', async () => {
    const { executor, calls } = mockExecutor();
    const steps = Array.from({ length: 100 }, (_, i) => ({
      delivery: 'aria', action: 'click', target: { role: 'button', name: `btn-${i}` },
    }));
    const runner = createScheduler({ scenario: 'test', steps } as any, immediateOptions([executor]));
    runner.play();
    await new Promise(r => setTimeout(r, 10));
    runner.dispose();
    const countAtDispose = calls.length;
    await new Promise(r => setTimeout(r, 50));
    expect(calls.length).toBe(countAtDispose);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/scenario/scheduler.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: FAIL

- [ ] **Step 3: Implement scheduler.ts**

Implement `createScheduler()` with the DES tick loop from the spec (lines 200-322). Key implementation points:

- `createScheduler()` parses the scenario via `bindScenario()`, creates the `DefaultVirtualClock`, registers executors, and returns a `ScenarioRunner`
- Tick loop: run ready queues (one step each via `Promise.all`), advance virtual time, activate triggers, yield via injectable tick/delay
- `play()` starts the tick loop if not already running
- `pause()` sets a flag checked at the start of each tick
- `step()` runs one tick then pauses
- `runTo(sectionTitle)` teleports to the target section (resets scope, re-binds from target forward)
- `dispose()` sets disposed flag, closes scope, clears queues
- `setSpeed()` delegates to VirtualClock
- Events emitted on the EventTarget per spec (scenario:state, scenario:step, scenario:section, scenario:queue)
- dispatchStep handles orchestration constructs: concurrent → create child queues + block parent, signal → scope.signal(name).signal(), await → scope.signal/latch.await() with queue blocking, delay → set wakeTime

The full implementation is ~300 lines. Key patterns:
- Waiter-based blocking: when `semaphore.acquire()` or `signal.await()` returns a pending Promise, store it as `queue.blockReason` and set state to 'blocked'. The Promise's `.then()` sets state back to 'ready'.
- Deadlock detection: count stale ticks (no state changes), bail after 100.
- Retry/loop decorators evaluated per spec (lines 232-278).

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/scenario/scheduler.test.ts --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/scenario/scheduler.ts packages/pages-aria/src/scenario/scheduler.test.ts
git commit -m "feat(scenario): add DES scheduler — tick loop, concurrent queues, virtual time Refs #461"
```

---

## Batch 4: Migration, barrel, consumer wiring

Delete old runners, update exports, wire the consumer component.
After this batch, the full system is integrated.

### Task 9: Delete old runners, update barrel, wire consumer

**Files:**
- Delete: `packages/pages-aria/src/scenario/runner.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/pages-aria/src/scenario/sectioned-runner.ts` (use `ide_refactor_safe_delete`)
- Modify: `packages/pages-aria/src/scenario/index.ts` — new barrel per spec
- Modify: `packages/pages-aria/src/tutorial/tutorial-host.ts` — switch to `createScheduler`

**Interfaces:**
- Consumes: `createScheduler`, `ScenarioRunner` from Task 8

- [ ] **Step 1: Delete old runners**

Use `ide_refactor_safe_delete` for:
- `packages/pages-aria/src/scenario/runner.ts`
- `packages/pages-aria/src/scenario/sectioned-runner.ts`

Check for references first — any remaining references need updating.

- [ ] **Step 2: Update barrel export (index.ts)**

Replace barrel contents per spec lines 796-816:

```typescript
export { parseScenario } from './parser.js';
export { createScheduler } from './scheduler.js';
export { isSectioned } from './types.js';

export type {
  Scenario, FlatScenario, SectionedScenario, ScenarioBase,
  ScenarioStep, OrchestratedStep, TutorialMeta, TutorialSection,
  SectionContent, DataTrigger, TimeTrigger, StepDecorators,
  OrchestrationBlock,
} from './types.js';
export type { ScenarioRunner, SchedulerOptions } from './scheduler.js';
export type { StepExecutor, ExecutionContext } from './step-executor.js';
```

- [ ] **Step 3: Update tutorial-host.ts consumer**

Change import from `runSectionedScenario` to `createScheduler`. The API shape is the same (play/pause/step/runTo/setSpeed/dispose), so the component body changes are minimal — primarily the import and the constructor call.

- [ ] **Step 4: Run typecheck**

Run: `npx tsc --noEmit -p packages/pages-aria/tsconfig.json`
Expected: no errors (may need to fix import references from deleted files)

- [ ] **Step 5: Run full test suite**

Run: `npx vitest run --config packages/pages-aria/vitest.config.ts --root packages/pages-aria`
Expected: ALL PASS (old runner tests deleted, new scheduler tests pass)

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/src/scenario/ packages/pages-aria/src/tutorial/tutorial-host.ts
git commit -m "feat(scenario): replace old runners with DES scheduler, wire consumer Closes #461 Closes #463"
```

---

## References

- [specs/issue-461-463-scenario-orchestrator/2026-09-23-scenario-orchestrator-design.md] — design spec (846 lines, 3-round reviewed)
- [specs/issue-461-463-scenario-orchestrator/decisions.md] — D1-D5 design decisions
- [packages/pages-aria/src/scenario/types.ts] — current scenario model
- [packages/pages-aria/src/scenario/parser.ts] — current parser
- [packages/pages-aria/src/scenario/sectioned-runner.ts] — cooperative suspend, runTo, speed control (being replaced)
- [packages/pages-aria/src/executor/command-executor.ts] — aria step execution (adapted to AriaExecutor)
- [packages/pages-aria/src/tutorial/tutorial-host.ts] — primary consumer
- [packages/yaml-core/src/orchestration/] — primitives from #462
- [GitHub #461] — DataTrigger and TimeTrigger evaluation
- [GitHub #463] — YAML binding for orchestration primitives
