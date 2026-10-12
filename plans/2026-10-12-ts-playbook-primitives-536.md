# TS Playbook Primitives Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development). Use Read/Edit for TS source files.
> Platform Java source in ~/claude/casehub/platform is the reference —
> use IntelliJ MCP (`mcp__intellij-index__*`) for Java navigation.

**Focal issue:** #536 — Sync TS playbook primitives: at:, on-complete:, resource:/priority:
**Issue group:** #536

**Goal:** Bring the TypeScript orchestration runtime to parity with platform #562's Java implementation — `OrcNumericPrimitive` interface, threshold conditions, priority semaphores, resource declarations, `at:` decorator, `on-complete:` expansion, and `resource:/priority:` decorator support.

**Architecture:** Extend existing yaml-core orchestration primitives and decorator chain. New interfaces (`OrcNumericPrimitive`, `PriorityOrcSemaphore`) follow the existing pattern of interface + Default implementation. `ThresholdCondition` and `OnCompleteExpander` are new standalone modules. The decorator chain gains two new decorator classes (`AtDecorator`, extended `SemaphoreDecorator`) inserted at the correct positions in the chain ordering.

**Tech Stack:** TypeScript, Vitest

## Global Constraints

- All new types export from `packages/yaml-core/src/orchestration/index.ts`
- Follow existing file-per-primitive pattern (interface + class + tests in same directory)
- Decorator chain ordering must match Java: `at:` inside `timeout:`, `resource:` handled by extended semaphore decorator
- No re-exports across packages

---

## Batch 1: Numeric primitive foundation + Priority enum

### Task 1: OrcNumericPrimitive interface + extend OrcCounter and OrcAccumulator

**Files:**
- Create: `packages/yaml-core/src/orchestration/numeric-primitive.ts`
- Create: `packages/yaml-core/src/orchestration/numeric-primitive.test.ts`
- Modify: `packages/yaml-core/src/orchestration/counter.ts`
- Modify: `packages/yaml-core/src/orchestration/counter.test.ts`
- Modify: `packages/yaml-core/src/orchestration/accumulator.ts`
- Modify: `packages/yaml-core/src/orchestration/accumulator.test.ts`
- Modify: `packages/yaml-core/src/orchestration/index.ts`

**Interfaces:**
- Produces: `OrcNumericPrimitive` interface with `doubleValue(): number`, `onThresholdChange(listener: (value: number) => void): void`, `removeThresholdListener(listener: (value: number) => void): void`
- Produces: `DefaultOrcCounter` now implements `OrcNumericPrimitive`
- Produces: `DefaultOrcAccumulator` now implements `OrcNumericPrimitive`

- [ ] **Step 1: Write OrcNumericPrimitive interface**

Create `packages/yaml-core/src/orchestration/numeric-primitive.ts`:

```typescript
export interface OrcNumericPrimitive {
  doubleValue(): number;
  onThresholdChange(listener: (value: number) => void): void;
  removeThresholdListener(listener: (value: number) => void): void;
}
```

- [ ] **Step 2: Write failing tests for OrcCounter threshold listeners**

Add to `packages/yaml-core/src/orchestration/counter.test.ts`:

```typescript
import type { OrcNumericPrimitive } from './numeric-primitive.js';

describe('OrcNumericPrimitive (counter)', () => {
  it('doubleValue returns current count', () => {
    const c = new DefaultOrcCounter();
    c.add(7);
    expect((c as OrcNumericPrimitive).doubleValue()).toBe(7);
  });

  it('fires threshold listener on value change', () => {
    const c = new DefaultOrcCounter();
    const values: number[] = [];
    (c as OrcNumericPrimitive).onThresholdChange((v) => values.push(v));
    c.increment();
    c.add(4);
    expect(values).toEqual([1, 5]);
  });

  it('removeThresholdListener stops notifications', () => {
    const c = new DefaultOrcCounter();
    const values: number[] = [];
    const listener = (v: number) => values.push(v);
    (c as OrcNumericPrimitive).onThresholdChange(listener);
    c.increment();
    (c as OrcNumericPrimitive).removeThresholdListener(listener);
    c.increment();
    expect(values).toEqual([1]);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run counter.test.ts`
Expected: FAIL — `doubleValue` / `onThresholdChange` not defined

- [ ] **Step 4: Implement OrcNumericPrimitive on DefaultOrcCounter**

Update `packages/yaml-core/src/orchestration/counter.ts`:

```typescript
import type { OrcNumericPrimitive } from './numeric-primitive.js';

export interface OrcCounter {
  increment(): void;
  decrement(): void;
  add(delta: number): void;
  get(): number;
  reset(): void;
}

export class DefaultOrcCounter implements OrcCounter, OrcNumericPrimitive {
  private value = 0;
  private readonly listeners: Set<(value: number) => void> = new Set();

  increment(): void { this.value++; this.notifyListeners(); }
  decrement(): void { this.value--; this.notifyListeners(); }
  add(delta: number): void { this.value += delta; this.notifyListeners(); }
  get(): number { return this.value; }
  reset(): void { this.value = 0; this.notifyListeners(); }

  doubleValue(): number { return this.value; }
  onThresholdChange(listener: (value: number) => void): void { this.listeners.add(listener); }
  removeThresholdListener(listener: (value: number) => void): void { this.listeners.delete(listener); }

  private notifyListeners(): void {
    for (const listener of this.listeners) listener(this.value);
  }
}
```

- [ ] **Step 5: Run counter tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run counter.test.ts`
Expected: PASS

- [ ] **Step 6: Write failing tests for OrcAccumulator threshold listeners**

Add to `packages/yaml-core/src/orchestration/accumulator.test.ts`:

```typescript
import type { OrcNumericPrimitive } from './numeric-primitive.js';

describe('OrcNumericPrimitive (accumulator)', () => {
  it('doubleValue returns current accumulated value', () => {
    const a = new DefaultOrcAccumulator((x, y) => x + y, 0);
    a.accumulate(3);
    a.accumulate(4);
    expect((a as OrcNumericPrimitive).doubleValue()).toBe(7);
  });

  it('fires threshold listener on accumulate', () => {
    const a = new DefaultOrcAccumulator((x, y) => x + y, 0);
    const values: number[] = [];
    (a as OrcNumericPrimitive).onThresholdChange((v) => values.push(v));
    a.accumulate(5);
    a.accumulate(3);
    expect(values).toEqual([5, 8]);
  });
});
```

- [ ] **Step 7: Implement OrcNumericPrimitive on DefaultOrcAccumulator**

Update `packages/yaml-core/src/orchestration/accumulator.ts`:

```typescript
import type { OrcNumericPrimitive } from './numeric-primitive.js';

export interface OrcAccumulator {
  accumulate(value: number): void;
  get(): number;
  reset(): void;
}

export class DefaultOrcAccumulator implements OrcAccumulator, OrcNumericPrimitive {
  private value: number;
  private readonly listeners: Set<(value: number) => void> = new Set();

  constructor(
    private readonly op: (a: number, b: number) => number,
    private readonly identity: number,
  ) {
    this.value = identity;
  }

  accumulate(value: number): void { this.value = this.op(this.value, value); this.notifyListeners(); }
  get(): number { return this.value; }
  reset(): void { this.value = this.identity; this.notifyListeners(); }

  doubleValue(): number { return this.value; }
  onThresholdChange(listener: (value: number) => void): void { this.listeners.add(listener); }
  removeThresholdListener(listener: (value: number) => void): void { this.listeners.delete(listener); }

  private notifyListeners(): void {
    for (const listener of this.listeners) listener(this.value);
  }
}
```

- [ ] **Step 8: Run accumulator tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run accumulator.test.ts`
Expected: PASS

- [ ] **Step 9: Export OrcNumericPrimitive from index.ts**

Add to `packages/yaml-core/src/orchestration/index.ts`:

```typescript
export type { OrcNumericPrimitive } from './numeric-primitive.js';
```

- [ ] **Step 10: Commit**

```bash
git add packages/yaml-core/src/orchestration/numeric-primitive.ts packages/yaml-core/src/orchestration/numeric-primitive.test.ts packages/yaml-core/src/orchestration/counter.ts packages/yaml-core/src/orchestration/counter.test.ts packages/yaml-core/src/orchestration/accumulator.ts packages/yaml-core/src/orchestration/accumulator.test.ts packages/yaml-core/src/orchestration/index.ts
git commit -m "feat(#536): OrcNumericPrimitive interface — threshold listeners on counter and accumulator"
```

### Task 2: ThresholdCondition parser

**Files:**
- Create: `packages/yaml-core/src/orchestration/threshold-condition.ts`
- Create: `packages/yaml-core/src/orchestration/threshold-condition.test.ts`
- Modify: `packages/yaml-core/src/orchestration/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `ThresholdCondition` with `parse(input: string): ThresholdCondition`, `test(value: number): boolean`, fields `operator`, `threshold`, `metricName`

- [ ] **Step 1: Write failing tests for ThresholdCondition**

Create `packages/yaml-core/src/orchestration/threshold-condition.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { ThresholdCondition } from './threshold-condition.js';

describe('ThresholdCondition', () => {
  it('parses >=N metric', () => {
    const tc = ThresholdCondition.parse('>=14 supply');
    expect(tc.operator).toBe('>=');
    expect(tc.threshold).toBe(14);
    expect(tc.metricName).toBe('supply');
  });

  it('defaults to >= when no operator', () => {
    const tc = ThresholdCondition.parse('10 workers');
    expect(tc.operator).toBe('>=');
    expect(tc.threshold).toBe(10);
    expect(tc.metricName).toBe('workers');
  });

  it('parses < operator', () => {
    const tc = ThresholdCondition.parse('<5 health');
    expect(tc.operator).toBe('<');
    expect(tc.threshold).toBe(5);
  });

  it('parses > operator', () => {
    const tc = ThresholdCondition.parse('>100 score');
    expect(tc.operator).toBe('>');
    expect(tc.threshold).toBe(100);
  });

  it('parses <= operator', () => {
    const tc = ThresholdCondition.parse('<=0 energy');
    expect(tc.operator).toBe('<=');
    expect(tc.threshold).toBe(0);
  });

  it('parses decimal thresholds', () => {
    const tc = ThresholdCondition.parse('>=3.5 rating');
    expect(tc.threshold).toBe(3.5);
  });

  it('throws on invalid input', () => {
    expect(() => ThresholdCondition.parse('invalid')).toThrow('Invalid threshold condition');
  });

  it('test returns true when condition met', () => {
    expect(ThresholdCondition.parse('>=14 supply').test(14)).toBe(true);
    expect(ThresholdCondition.parse('>=14 supply').test(20)).toBe(true);
    expect(ThresholdCondition.parse('>10 x').test(11)).toBe(true);
    expect(ThresholdCondition.parse('<=5 x').test(5)).toBe(true);
    expect(ThresholdCondition.parse('<5 x').test(4)).toBe(true);
  });

  it('test returns false when condition not met', () => {
    expect(ThresholdCondition.parse('>=14 supply').test(13)).toBe(false);
    expect(ThresholdCondition.parse('>10 x').test(10)).toBe(false);
    expect(ThresholdCondition.parse('<=5 x').test(6)).toBe(false);
    expect(ThresholdCondition.parse('<5 x').test(5)).toBe(false);
  });

  it('trims whitespace', () => {
    const tc = ThresholdCondition.parse('  >= 14  supply  ');
    expect(tc.operator).toBe('>=');
    expect(tc.threshold).toBe(14);
    expect(tc.metricName).toBe('supply');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run threshold-condition.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement ThresholdCondition**

Create `packages/yaml-core/src/orchestration/threshold-condition.ts`:

```typescript
const PATTERN = /^(>=|>|<=|<)?\s*(\d+(?:\.\d+)?)\s+(\w+)$/;

export class ThresholdCondition {
  private constructor(
    readonly operator: string,
    readonly threshold: number,
    readonly metricName: string,
  ) {}

  static parse(input: string): ThresholdCondition {
    const m = input.trim().match(PATTERN);
    if (!m) throw new Error(`Invalid threshold condition: '${input}'`);
    const op = m[1] ?? '>=';
    const threshold = parseFloat(m[2]!);
    const metric = m[3]!;
    return new ThresholdCondition(op, threshold, metric);
  }

  test(value: number): boolean {
    switch (this.operator) {
      case '>=': return value >= this.threshold;
      case '>':  return value > this.threshold;
      case '<=': return value <= this.threshold;
      case '<':  return value < this.threshold;
      default: throw new Error(`Unknown operator: ${this.operator}`);
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run threshold-condition.test.ts`
Expected: PASS

- [ ] **Step 5: Export from index.ts and commit**

Add to `packages/yaml-core/src/orchestration/index.ts`:
```typescript
export { ThresholdCondition } from './threshold-condition.js';
```

```bash
git add packages/yaml-core/src/orchestration/threshold-condition.ts packages/yaml-core/src/orchestration/threshold-condition.test.ts packages/yaml-core/src/orchestration/index.ts
git commit -m "feat(#536): ThresholdCondition parser — regex-based threshold syntax for at: decorator"
```

### Task 3: Priority enum + PriorityOrcSemaphore

**Files:**
- Create: `packages/yaml-core/src/orchestration/priority.ts`
- Create: `packages/yaml-core/src/orchestration/priority-semaphore.ts`
- Create: `packages/yaml-core/src/orchestration/priority-semaphore.test.ts`
- Modify: `packages/yaml-core/src/orchestration/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `Priority` enum (BACKGROUND=0, NORMAL=1, HIGH=2), `PriorityOrcSemaphore` interface with `acquire(priority: Priority): Promise<void>`, `tryAcquire(priority: Priority, timeoutMs: number): Promise<boolean>`, `release(): void`, `availablePermits(): number`

- [ ] **Step 1: Write Priority enum**

Create `packages/yaml-core/src/orchestration/priority.ts`:

```typescript
export enum Priority {
  BACKGROUND = 0,
  NORMAL = 1,
  HIGH = 2,
}

export function parsePriority(raw: string): Priority {
  switch (raw.toLowerCase()) {
    case 'background': return Priority.BACKGROUND;
    case 'normal': return Priority.NORMAL;
    case 'high': return Priority.HIGH;
    default: throw new Error(`Unknown priority: '${raw}'`);
  }
}
```

- [ ] **Step 2: Write failing tests for PriorityOrcSemaphore**

Create `packages/yaml-core/src/orchestration/priority-semaphore.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { DefaultPriorityOrcSemaphore } from './priority-semaphore.js';
import { Priority } from './priority.js';

describe('DefaultPriorityOrcSemaphore', () => {
  it('acquire succeeds immediately when permits available', async () => {
    const sem = new DefaultPriorityOrcSemaphore(2);
    await sem.acquire(Priority.NORMAL);
    expect(sem.availablePermits()).toBe(1);
  });

  it('acquire blocks when no permits, unblocks on release', async () => {
    const sem = new DefaultPriorityOrcSemaphore(1);
    await sem.acquire(Priority.NORMAL);
    let acquired = false;
    const p = sem.acquire(Priority.NORMAL).then(() => { acquired = true; });
    expect(acquired).toBe(false);
    sem.release();
    await p;
    expect(acquired).toBe(true);
  });

  it('higher priority acquires before lower priority', async () => {
    const sem = new DefaultPriorityOrcSemaphore(1);
    await sem.acquire(Priority.NORMAL);
    const order: string[] = [];
    const pBg = sem.acquire(Priority.BACKGROUND).then(() => order.push('bg'));
    const pHi = sem.acquire(Priority.HIGH).then(() => order.push('high'));
    sem.release();
    sem.release();
    await Promise.all([pBg, pHi]);
    expect(order).toEqual(['high', 'bg']);
  });

  it('tryAcquire returns false on timeout', async () => {
    const sem = new DefaultPriorityOrcSemaphore(0);
    expect(await sem.tryAcquire(Priority.NORMAL, 10)).toBe(false);
  });

  it('tryAcquire returns true when permit becomes available', async () => {
    const sem = new DefaultPriorityOrcSemaphore(0);
    const p = sem.tryAcquire(Priority.NORMAL, 1000);
    sem.release();
    expect(await p).toBe(true);
  });

  it('rejects permits < 1', () => {
    expect(() => new DefaultPriorityOrcSemaphore(0)).toThrow('permits must be >= 1');
  });

  it('release without prior acquire increases permits', () => {
    const sem = new DefaultPriorityOrcSemaphore(1);
    sem.release();
    expect(sem.availablePermits()).toBe(2);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run priority-semaphore.test.ts`
Expected: FAIL — module not found

- [ ] **Step 4: Implement PriorityOrcSemaphore**

Create `packages/yaml-core/src/orchestration/priority-semaphore.ts`:

```typescript
import { Priority } from './priority.js';

export interface PriorityOrcSemaphore {
  acquire(priority: Priority): Promise<void>;
  tryAcquire(priority: Priority, timeoutMs: number): Promise<boolean>;
  release(): void;
  availablePermits(): number;
}

interface PriorityWaiter {
  priority: Priority;
  resolve: (acquired: boolean) => void;
  timer?: ReturnType<typeof setTimeout>;
}

export class DefaultPriorityOrcSemaphore implements PriorityOrcSemaphore {
  private _permits: number;
  private _waiters: PriorityWaiter[] = [];

  constructor(permits: number) {
    if (permits < 1) throw new Error('permits must be >= 1');
    this._permits = permits;
  }

  acquire(priority: Priority): Promise<void> {
    if (this._permits > 0) {
      this._permits--;
      return Promise.resolve();
    }
    return new Promise<boolean>((resolve) => {
      this.insertWaiter({ priority, resolve });
    }).then(() => undefined);
  }

  tryAcquire(priority: Priority, timeoutMs: number): Promise<boolean> {
    if (this._permits > 0) {
      this._permits--;
      return Promise.resolve(true);
    }
    return new Promise<boolean>((resolve) => {
      const waiter: PriorityWaiter = { priority, resolve };
      waiter.timer = setTimeout(() => {
        const idx = this._waiters.indexOf(waiter);
        if (idx >= 0) this._waiters.splice(idx, 1);
        resolve(false);
      }, timeoutMs);
      this.insertWaiter(waiter);
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

  private insertWaiter(waiter: PriorityWaiter): void {
    let i = 0;
    while (i < this._waiters.length && this._waiters[i]!.priority >= waiter.priority) i++;
    this._waiters.splice(i, 0, waiter);
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run priority-semaphore.test.ts`
Expected: PASS

- [ ] **Step 6: Export from index.ts and commit**

Add to `packages/yaml-core/src/orchestration/index.ts`:
```typescript
export { Priority, parsePriority } from './priority.js';
export type { PriorityOrcSemaphore } from './priority-semaphore.js';
export { DefaultPriorityOrcSemaphore } from './priority-semaphore.js';
```

```bash
git add packages/yaml-core/src/orchestration/priority.ts packages/yaml-core/src/orchestration/priority-semaphore.ts packages/yaml-core/src/orchestration/priority-semaphore.test.ts packages/yaml-core/src/orchestration/index.ts
git commit -m "feat(#536): Priority enum + PriorityOrcSemaphore — priority-ordered permit acquisition"
```

## Batch 2: ResourceDeclaration + ScenarioScope extensions

### Task 4: ResourceDeclaration type + ScenarioScope additions

**Files:**
- Create: `packages/yaml-core/src/orchestration/resource-declaration.ts`
- Create: `packages/yaml-core/src/orchestration/resource-declaration.test.ts`
- Modify: `packages/yaml-core/src/orchestration/types.ts` (add `numericPrimitive` and `prioritySemaphore` to `ScenarioScope`)
- Modify: `packages/yaml-core/src/orchestration/scenario-scope.ts` (implement new methods)
- Modify: `packages/yaml-core/src/orchestration/scenario-scope.test.ts`
- Modify: `packages/yaml-core/src/orchestration/index.ts`

**Interfaces:**
- Consumes: `OrcNumericPrimitive` (Task 1), `PriorityOrcSemaphore` / `DefaultPriorityOrcSemaphore` (Task 3), `Priority` (Task 3)
- Produces: `ResourceDeclaration` type, `ScenarioScope.numericPrimitive(name)`, `ScenarioScope.prioritySemaphore(name, permits)`

- [ ] **Step 1: Write ResourceDeclaration**

Create `packages/yaml-core/src/orchestration/resource-declaration.ts`:

```typescript
export interface ResourceDeclaration {
  readonly concurrency: number;
}

export function parseResourceDeclaration(raw: unknown): ResourceDeclaration {
  if (typeof raw === 'number') {
    if (raw < 1) throw new Error(`concurrency must be >= 1, got: ${raw}`);
    return { concurrency: raw };
  }
  if (typeof raw === 'object' && raw !== null) {
    const map = raw as Record<string, unknown>;
    const concurrency = typeof map['concurrency'] === 'number' ? map['concurrency'] : 1;
    if (concurrency < 1) throw new Error(`concurrency must be >= 1, got: ${concurrency}`);
    return { concurrency };
  }
  return { concurrency: 1 };
}
```

- [ ] **Step 2: Write failing test for ResourceDeclaration**

Create `packages/yaml-core/src/orchestration/resource-declaration.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { parseResourceDeclaration } from './resource-declaration.js';

describe('parseResourceDeclaration', () => {
  it('parses number as concurrency', () => {
    expect(parseResourceDeclaration(3)).toEqual({ concurrency: 3 });
  });

  it('parses map with concurrency', () => {
    expect(parseResourceDeclaration({ concurrency: 5 })).toEqual({ concurrency: 5 });
  });

  it('defaults concurrency to 1', () => {
    expect(parseResourceDeclaration({})).toEqual({ concurrency: 1 });
    expect(parseResourceDeclaration('anything')).toEqual({ concurrency: 1 });
  });

  it('rejects concurrency < 1', () => {
    expect(() => parseResourceDeclaration(0)).toThrow('concurrency must be >= 1');
    expect(() => parseResourceDeclaration({ concurrency: -1 })).toThrow('concurrency must be >= 1');
  });
});
```

- [ ] **Step 3: Run and verify tests pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run resource-declaration.test.ts`
Expected: PASS

- [ ] **Step 4: Add numericPrimitive and prioritySemaphore to ScenarioScope interface**

Add to `packages/yaml-core/src/orchestration/types.ts` — within the `ScenarioScope` interface:

```typescript
numericPrimitive(name: string): import('./numeric-primitive.js').OrcNumericPrimitive;
prioritySemaphore(name: string, permits: number): import('./priority-semaphore.js').PriorityOrcSemaphore;
```

- [ ] **Step 5: Write failing tests for ScenarioScope new methods**

Add to `packages/yaml-core/src/orchestration/scenario-scope.test.ts`:

```typescript
import type { OrcNumericPrimitive } from './numeric-primitive.js';

describe('numericPrimitive', () => {
  it('returns counter as OrcNumericPrimitive', () => {
    const scope = new DefaultScenarioScope();
    scope.counter('supply');
    const np = scope.numericPrimitive('supply');
    expect(np.doubleValue()).toBe(0);
  });

  it('throws for non-numeric primitive', () => {
    const scope = new DefaultScenarioScope();
    scope.flag('ready');
    expect(() => scope.numericPrimitive('ready')).toThrow();
  });
});

describe('prioritySemaphore', () => {
  it('creates and returns a PriorityOrcSemaphore', () => {
    const scope = new DefaultScenarioScope();
    const sem = scope.prioritySemaphore('pool', 3);
    expect(sem.availablePermits()).toBe(3);
  });

  it('returns same instance on subsequent calls', () => {
    const scope = new DefaultScenarioScope();
    const sem1 = scope.prioritySemaphore('pool', 3);
    const sem2 = scope.prioritySemaphore('pool', 3);
    expect(sem1).toBe(sem2);
  });
});
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run scenario-scope.test.ts`
Expected: FAIL — methods don't exist

- [ ] **Step 7: Implement numericPrimitive and prioritySemaphore on DefaultScenarioScope**

Add imports and methods to `packages/yaml-core/src/orchestration/scenario-scope.ts`:

```typescript
import type { OrcNumericPrimitive } from './numeric-primitive.js';
import { DefaultPriorityOrcSemaphore } from './priority-semaphore.js';
import type { PriorityOrcSemaphore } from './priority-semaphore.js';

// Add methods to DefaultScenarioScope class:

numericPrimitive(name: string): OrcNumericPrimitive {
  const existing = this._primitives.get(name);
  if (existing !== undefined) {
    if (typeof (existing as Record<string, unknown>).doubleValue === 'function') {
      return existing as OrcNumericPrimitive;
    }
    throw new Error(`Primitive '${name}' is not a numeric primitive`);
  }
  throw new Error(`No primitive named '${name}' exists`);
}

prioritySemaphore(name: string, permits: number): PriorityOrcSemaphore {
  return this._getOrCreate(name, () => new DefaultPriorityOrcSemaphore(permits));
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run scenario-scope.test.ts`
Expected: PASS

- [ ] **Step 9: Export ResourceDeclaration from index.ts and commit**

Add to `packages/yaml-core/src/orchestration/index.ts`:
```typescript
export type { ResourceDeclaration } from './resource-declaration.js';
export { parseResourceDeclaration } from './resource-declaration.js';
```

```bash
git add packages/yaml-core/src/orchestration/resource-declaration.ts packages/yaml-core/src/orchestration/resource-declaration.test.ts packages/yaml-core/src/orchestration/types.ts packages/yaml-core/src/orchestration/scenario-scope.ts packages/yaml-core/src/orchestration/scenario-scope.test.ts packages/yaml-core/src/orchestration/index.ts
git commit -m "feat(#536): ResourceDeclaration + ScenarioScope.numericPrimitive/prioritySemaphore"
```

## Batch 3: Decorator chain extensions — at: and resource:/priority:

### Task 5: AtDecorator + resource:/priority: in SemaphoreDecorator

**Files:**
- Modify: `packages/yaml-core/src/step/decorator-chain.ts`
- Modify: `packages/yaml-core/src/step/decorator-chain.test.ts`

**Interfaces:**
- Consumes: `ThresholdCondition` (Task 2), `OrcNumericPrimitive` (Task 1), `PriorityOrcSemaphore` (Task 3), `Priority` / `parsePriority` (Task 3), `parseResourceDeclaration` (Task 4), `ScenarioScope.numericPrimitive()` (Task 4), `ScenarioScope.prioritySemaphore()` (Task 4)
- Produces: `at:` decorator in chain (between `timeout:` and `retry:`/`semaphore:`), `resource:` key handled in semaphore section with `priority:` support

- [ ] **Step 1: Write failing tests for AtDecorator**

Add to `packages/yaml-core/src/step/decorator-chain.test.ts`:

```typescript
describe('at decorator', () => {
  it('blocks until threshold met then executes', async () => {
    const scope = new DefaultScenarioScope();
    const counter = scope.counter('supply');
    const action = mockAction(stepSuccess({ v: 1 }));
    const chain = DecoratorChain.build({ at: '>=14 supply' }, action);
    const ctx = mockContext({ scope });
    const resultPromise = chain.execute(ctx);
    counter.add(14);
    const result = await resultPromise;
    expect(result.kind).toBe('success');
    expect(action.execute).toHaveBeenCalled();
  });

  it('executes immediately when threshold already met', async () => {
    const scope = new DefaultScenarioScope();
    const counter = scope.counter('supply');
    counter.add(20);
    const action = mockAction(stepSuccess({}));
    const chain = DecoratorChain.build({ at: '>=14 supply' }, action);
    const result = await chain.execute(mockContext({ scope }));
    expect(result.kind).toBe('success');
    expect(action.execute).toHaveBeenCalled();
  });

  it('supports list of threshold conditions (all must be met)', async () => {
    const scope = new DefaultScenarioScope();
    const supply = scope.counter('supply');
    const workers = scope.counter('workers');
    const action = mockAction(stepSuccess({}));
    const chain = DecoratorChain.build({ at: ['>=10 supply', '>=5 workers'] }, action);
    const resultPromise = chain.execute(mockContext({ scope }));
    supply.add(10);
    workers.add(5);
    const result = await resultPromise;
    expect(result.kind).toBe('success');
  });

  it('guard mode checks immediately and skips if not met', async () => {
    const scope = new DefaultScenarioScope();
    scope.counter('supply');
    const action = mockAction(stepSuccess({}));
    const chain = DecoratorChain.build({ at: { metric: '>=14 supply', mode: 'guard' } }, action);
    const result = await chain.execute(mockContext({ scope }));
    expect(result.kind).toBe('success');
    expect(action.execute).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run decorator-chain.test.ts`
Expected: FAIL — at: decorator not handled

- [ ] **Step 3: Implement AtDecorator**

Add to `packages/yaml-core/src/step/decorator-chain.ts`:

```typescript
import { ThresholdCondition } from '../orchestration/threshold-condition.js';
import type { OrcNumericPrimitive } from '../orchestration/numeric-primitive.js';

class AtDecorator implements DecoratedExecution {
  constructor(
    private readonly next: DecoratedExecution,
    private readonly raw: unknown,
  ) {}
  async execute(context: Context): Promise<Result> {
    if (typeof this.raw === 'object' && this.raw !== null && !Array.isArray(this.raw)) {
      const map = this.raw as Record<string, unknown>;
      if (map['mode'] === 'guard') {
        return this.executeGuard(map, context);
      }
    }
    const conditions = this.parseConditions();
    await this.waitForConditions(conditions, context);
    return this.next.execute(context);
  }

  private async executeGuard(map: Record<string, unknown>, context: Context): Promise<Result> {
    const condStr = String(map['metric']);
    const cond = ThresholdCondition.parse(condStr);
    const prim = context.scope.numericPrimitive(cond.metricName);
    if (!cond.test(prim.doubleValue())) {
      return stepSuccess({});
    }
    return this.next.execute(context);
  }

  private parseConditions(): ThresholdCondition[] {
    if (typeof this.raw === 'string') return [ThresholdCondition.parse(this.raw)];
    if (Array.isArray(this.raw)) return this.raw.map(s => ThresholdCondition.parse(String(s)));
    if (typeof this.raw === 'object' && this.raw !== null) {
      const map = this.raw as Record<string, unknown>;
      const metric = map['metric'];
      if (typeof metric === 'string') return [ThresholdCondition.parse(metric)];
    }
    throw new Error(`Invalid at: specification: ${JSON.stringify(this.raw)}`);
  }

  private waitForConditions(conditions: ThresholdCondition[], context: Context): Promise<void> {
    const met = new Set<number>();
    for (let i = 0; i < conditions.length; i++) {
      const prim = context.scope.numericPrimitive(conditions[i]!.metricName);
      if (conditions[i]!.test(prim.doubleValue())) met.add(i);
    }
    if (met.size === conditions.length) return Promise.resolve();

    return new Promise<void>((resolve) => {
      const listeners: Array<{ prim: OrcNumericPrimitive; listener: (v: number) => void }> = [];
      const cleanup = () => { for (const { prim, listener } of listeners) prim.removeThresholdListener(listener); };

      for (let i = 0; i < conditions.length; i++) {
        if (met.has(i)) continue;
        const cond = conditions[i]!;
        const prim = context.scope.numericPrimitive(cond.metricName);
        const listener = (value: number) => {
          if (cond.test(value)) {
            met.add(i);
            if (met.size === conditions.length) { cleanup(); resolve(); }
          }
        };
        listeners.push({ prim, listener });
        prim.onThresholdChange(listener);
      }
    });
  }
}
```

Insert into `DecoratorChain.build()` — after `timeout:` and before `retry:`/`semaphore:`:

```typescript
if (decorators['at'] != null) {
  execution = new AtDecorator(execution, decorators['at']);
}
```

- [ ] **Step 4: Run at: decorator tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run decorator-chain.test.ts`
Expected: PASS

- [ ] **Step 5: Write failing tests for resource:/priority: support**

Add to `packages/yaml-core/src/step/decorator-chain.test.ts`:

```typescript
import { Priority } from '../orchestration/priority.js';

describe('resource decorator with priority', () => {
  it('acquires priority semaphore with default normal priority', async () => {
    const scope = new DefaultScenarioScope();
    const sem = scope.prioritySemaphore('api-pool', 2);
    const action = mockAction(stepSuccess({}));
    const chain = DecoratorChain.build({ resource: 'api-pool' }, action);
    await chain.execute(mockContext({ scope }));
    expect(sem.availablePermits()).toBe(2);
    expect(action.execute).toHaveBeenCalled();
  });

  it('acquires with specified priority', async () => {
    const scope = new DefaultScenarioScope();
    const sem = scope.prioritySemaphore('api-pool', 1);
    const action = mockAction(stepSuccess({}));
    const chain = DecoratorChain.build({ resource: 'api-pool', priority: 'high' }, action);
    await chain.execute(mockContext({ scope }));
    expect(sem.availablePermits()).toBe(1);
  });

  it('releases priority semaphore on error', async () => {
    const scope = new DefaultScenarioScope();
    const sem = scope.prioritySemaphore('err-pool', 1);
    const action: Action = { execute: () => { throw new Error('boom'); } };
    const chain = DecoratorChain.build({ resource: 'err-pool' }, action);
    const result = await chain.execute(mockContext({ scope }));
    expect(result.kind).toBe('failure');
    expect(sem.availablePermits()).toBe(1);
  });
});
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run decorator-chain.test.ts`
Expected: FAIL — resource: not handled

- [ ] **Step 7: Extend SemaphoreDecorator section for resource:/priority:**

Add to `packages/yaml-core/src/step/decorator-chain.ts`, inside `DecoratorChain.build()`, extending the semaphore handling section:

```typescript
import { parsePriority, Priority } from '../orchestration/priority.js';

// In DecoratorChain.build(), add resource: handling BEFORE the existing semaphore/mutex block:
const resourceRaw = decorators['resource'];
if (resourceRaw != null) {
  const resourceName = typeof resourceRaw === 'string' ? resourceRaw : String((resourceRaw as Record<string, unknown>)['name']);
  const priorityRaw = decorators['priority'] as string | undefined;
  const priority = priorityRaw != null ? parsePriority(priorityRaw) : Priority.NORMAL;
  execution = new PrioritySemaphoreDecorator(execution, resourceName, priority);
}
```

Add the `PrioritySemaphoreDecorator` class:

```typescript
class PrioritySemaphoreDecorator implements DecoratedExecution {
  constructor(
    private readonly next: DecoratedExecution,
    private readonly name: string,
    private readonly priority: Priority,
  ) {}
  async execute(context: Context): Promise<Result> {
    const sem = context.scope.prioritySemaphore(this.name, 1);
    const remaining = context.scope.remainingTime();
    if (remaining != null) {
      const acquired = await sem.tryAcquire(this.priority, remaining);
      if (!acquired) {
        return stepFailure(`Resource '${this.name}' acquire exceeded deadline`);
      }
    } else {
      await sem.acquire(this.priority);
    }
    try {
      return await this.next.execute(context);
    } catch (e) {
      return stepFailure(e instanceof Error ? e.message : String(e));
    } finally {
      sem.release();
    }
  }
}
```

- [ ] **Step 8: Run all decorator chain tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run decorator-chain.test.ts`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add packages/yaml-core/src/step/decorator-chain.ts packages/yaml-core/src/step/decorator-chain.test.ts
git commit -m "feat(#536): at: decorator + resource:/priority: — threshold-gated execution and priority resource contention"
```

## Batch 4: OnCompleteExpander

### Task 6: OnCompleteExpander + SourceLocationMap

**Files:**
- Create: `packages/yaml-core/src/step/on-complete-expander.ts`
- Create: `packages/yaml-core/src/step/on-complete-expander.test.ts`
- Modify: `packages/yaml-core/src/step/index.ts`

**Interfaces:**
- Consumes: nothing (standalone parse-phase transform)
- Produces: `OnCompleteExpander.expand(steps): ExpandedSteps`, `SourceLocationMap` with `isGenerated(name)`, `originalContext(name)`

- [ ] **Step 1: Write failing tests for OnCompleteExpander**

Create `packages/yaml-core/src/step/on-complete-expander.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { OnCompleteExpander } from './on-complete-expander.js';

describe('OnCompleteExpander', () => {
  it('passes through steps without on-complete', () => {
    const steps = [
      { action: 'deploy', name: 'step1' },
      { action: 'notify', name: 'step2' },
    ];
    const result = OnCompleteExpander.expand(steps);
    expect(result.steps).toEqual(steps);
    expect(result.locations.isGenerated('step1')).toBe(false);
  });

  it('desugars on-complete to signal/wait pair', () => {
    const steps = [
      {
        action: 'deploy',
        name: 'deploy-step',
        'on-complete': [{ action: 'notify' }],
      },
    ];
    const result = OnCompleteExpander.expand(steps);
    expect(result.steps).toHaveLength(2);
    expect(result.steps[0]!['signal']).toBe('deploy-step_done');
    expect(result.steps[0]!['on-complete']).toBeUndefined();
    expect(result.steps[1]!['wait']).toBe('deploy-step_done');
    expect(result.steps[1]!['action']).toBe('notify');
  });

  it('generates name when step has no name', () => {
    const steps = [
      {
        action: 'deploy',
        'on-complete': [{ action: 'notify' }],
      },
    ];
    const result = OnCompleteExpander.expand(steps);
    expect(result.steps[0]!['name']).toMatch(/^__oc_/);
    expect(result.locations.isGenerated(result.steps[0]!['name'] as string)).toBe(true);
  });

  it('preserves existing signal — merges as list', () => {
    const steps = [
      {
        action: 'deploy',
        name: 'dep',
        signal: 'existing-signal',
        'on-complete': [{ action: 'notify' }],
      },
    ];
    const result = OnCompleteExpander.expand(steps);
    expect(result.steps[0]!['signal']).toEqual(['existing-signal', 'dep_done']);
  });

  it('handles nested on-complete', () => {
    const steps = [
      {
        action: 'step-a',
        name: 'a',
        'on-complete': [
          {
            action: 'step-b',
            name: 'b',
            'on-complete': [{ action: 'step-c' }],
          },
        ],
      },
    ];
    const result = OnCompleteExpander.expand(steps);
    expect(result.steps).toHaveLength(3);
    expect(result.steps[0]!['action']).toBe('step-a');
    expect(result.steps[1]!['action']).toBe('step-b');
    expect(result.steps[1]!['wait']).toBe('a_done');
    expect(result.steps[2]!['action']).toBe('step-c');
    expect(result.steps[2]!['wait']).toBe('b_done');
  });

  it('SourceLocationMap.originalContext returns context for generated names', () => {
    const steps = [
      {
        action: 'deploy',
        'on-complete': [{ action: 'notify' }],
      },
    ];
    const result = OnCompleteExpander.expand(steps);
    const genName = result.steps[0]!['name'] as string;
    expect(result.locations.originalContext(genName)).toContain('on-complete');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run on-complete-expander.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement OnCompleteExpander**

Create `packages/yaml-core/src/step/on-complete-expander.ts`:

```typescript
export class SourceLocationMap {
  private readonly generatedToOriginal = new Map<string, string>();

  record(generatedName: string, parentContext: string): void {
    this.generatedToOriginal.set(generatedName, parentContext);
  }

  isGenerated(name: string): boolean {
    return this.generatedToOriginal.has(name);
  }

  originalContext(generatedName: string): string | undefined {
    return this.generatedToOriginal.get(generatedName);
  }
}

export interface ExpandedSteps {
  steps: Record<string, unknown>[];
  locations: SourceLocationMap;
}

export class OnCompleteExpander {
  private constructor() {}

  static expand(steps: Record<string, unknown>[]): ExpandedSteps {
    let counter = 0;
    const locations = new SourceLocationMap();
    const result: Record<string, unknown>[] = [];

    function expandRecursive(input: Record<string, unknown>[], output: Record<string, unknown>[]): void {
      for (const step of input) {
        const onComplete = step['on-complete'];
        if (onComplete == null) { output.push(step); continue; }

        const expanded = { ...step };
        delete expanded['on-complete'];

        counter++;
        let name: string;
        if (expanded['name'] != null) {
          name = String(expanded['name']);
        } else {
          name = `__oc_${counter}`;
          expanded['name'] = name;
          locations.record(name, `on-complete block #${counter}`);
        }

        const signalName = `${name}_done`;
        const existingSignal = expanded['signal'];
        if (existingSignal != null) {
          const signals: string[] = [];
          if (Array.isArray(existingSignal)) {
            existingSignal.forEach(s => signals.push(String(s)));
          } else {
            signals.push(String(existingSignal));
          }
          signals.push(signalName);
          expanded['signal'] = signals;
        } else {
          expanded['signal'] = signalName;
        }

        output.push(expanded);

        const continuations = onComplete as Record<string, unknown>[];
        const continuationSteps: Record<string, unknown>[] = [];
        for (const cont of continuations) {
          continuationSteps.push({ ...cont, wait: signalName });
        }

        expandRecursive(continuationSteps, output);
      }
    }

    expandRecursive(steps, result);
    return { steps: result, locations };
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run on-complete-expander.test.ts`
Expected: PASS

- [ ] **Step 5: Export from step/index.ts and commit**

Add to `packages/yaml-core/src/step/index.ts`:
```typescript
export { OnCompleteExpander, SourceLocationMap } from './on-complete-expander.js';
export type { ExpandedSteps } from './on-complete-expander.js';
```

```bash
git add packages/yaml-core/src/step/on-complete-expander.ts packages/yaml-core/src/step/on-complete-expander.test.ts packages/yaml-core/src/step/index.ts
git commit -m "feat(#536): OnCompleteExpander — desugar on-complete: to signal:/wait: pairs"
```

### Task 7: Run full test suite and verify no regressions

**Files:** none — verification only

- [ ] **Step 1: Run full yaml-core test suite**

Run: `yarn workspace @nicecasehub/yaml-core test -- --run`
Expected: All tests pass including existing tests

- [ ] **Step 2: Run typecheck**

Run: `yarn workspace @nicecasehub/yaml-core tsc --noEmit`
Expected: No type errors

- [ ] **Step 3: Commit any adjustments, then final commit**

If fixes were needed, commit them. Otherwise:

```bash
git log --oneline -6
```

Verify the commit sequence matches the plan.

## References

- casehubio/platform#562 — Java implementation (source of truth for API contracts)
- `packages/yaml-core/src/orchestration/counter.ts` — existing OrcCounter pattern
- `packages/yaml-core/src/orchestration/semaphore.ts` — existing OrcSemaphore pattern
- `packages/yaml-core/src/step/decorator-chain.ts` — decorator chain build ordering
- `packages/yaml-core/src/orchestration/types.ts` — ScenarioScope interface
- `packages/yaml-core/src/orchestration/scenario-scope.ts` — DefaultScenarioScope
- GitHub #536 — focal issue
