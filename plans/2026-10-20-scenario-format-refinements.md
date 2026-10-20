# Scenario Format Refinements — TypeScript Side

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #390 — Scenario format refinements — rename, new fields, action types, cleanup
**Issue group:** #390

**Goal:** Apply TypeScript-side refinements to the scenario dispatch protocol: rename command-level `target` to `element`, add step-level `delay`, make speed opt-in, add REST action definition, and consolidate dispatch types into the public types module.

**Architecture:** The dispatch protocol types (`CommandPayload`, `DispatchStep`, `DispatchSequence`, `ExecutorControl`) are currently unexported local interfaces inside `playbook-handler.ts`. This plan extracts them to `playbook/types.ts` as exported types first, then applies field-level refinements. The Java backend changes (HierarchicalParser, ScenarioParser deletion, orchestrator wiring) are out of scope — they live in the backend modules.

**Tech Stack:** TypeScript, Vitest, Lit (pages-aria package)

## Global Constraints

- All types use the `@casehubio/pages-primitives` `AriaTarget` type — no local redefinitions
- `RESERVED_KEYS` and `DECORATOR_KEYS` in `packages/yaml-core/src/step/walker.ts` already include `target`, `delay`, `speed`, `actor` — no walker changes needed
- The existing test at line 715 already validates `speed <= 0` skips inter-step delay — the speed default change must not break this
- Test file uses `'scenario:exec'` topic for legacy commands (not `'playbook:exec'`)

---

## Batch 1: Type extraction and field changes

### Task 1: Extract dispatch protocol types to playbook/types.ts

**Files:**
- Modify: `packages/pages-aria/src/playbook/types.ts` (add exported dispatch types)
- Modify: `packages/pages-aria/src/server/playbook-handler.ts:10-42` (replace local interfaces with imports)

**Interfaces:**
- Produces: exported `CommandPayload`, `DispatchStep`, `DispatchSequence`, `ExecutorControl` types from `playbook/types.ts`

- [ ] **Step 1: Add dispatch protocol types to playbook/types.ts**

Append after the existing `OrchestrationBlock` interface (line 83) in `packages/pages-aria/src/playbook/types.ts`:

```typescript
// --- Dispatch protocol types (executor ↔ orchestrator) ---

export interface CommandPayload {
  id: string;
  action: string;
  element?: AriaTarget;
  value?: string;
  state?: Record<string, unknown>;
  timeout?: number;
}

export interface DispatchStep {
  name: string;
  label?: string;
  actor?: string;
  action: string;
  params: Record<string, unknown>;
  await?: { match?: Record<string, unknown>; timeout?: number; interval?: number; status?: number };
  mode?: string;
  delay?: number;
}

export interface DispatchSequence {
  op: 'dispatch-sequence';
  sessionId: string;
  steps: DispatchStep[];
  speed?: number;
  paused: boolean;
}

export interface ExecutorControl {
  op: 'executor-control';
  sessionId: string;
  command: 'pause' | 'resume' | 'step' | 'speed' | 'stop';
  speed?: number;
}
```

Add the `AriaTarget` import at line 1:
```typescript
import type { AriaTarget } from '@casehubio/pages-primitives';
```

Note: these already include the three field refinements from #390:
- `target` renamed to `element` on `CommandPayload`
- `delay?: number` added to `DispatchStep`
- `speed` made optional on `DispatchSequence`

- [ ] **Step 2: Replace local interfaces in playbook-handler.ts with imports**

Remove the four local interface blocks (lines 10-42) from `playbook-handler.ts` and add an import:

```typescript
import type { CommandPayload, DispatchStep, DispatchSequence, ExecutorControl } from '../playbook/types.js';
```

- [ ] **Step 3: Update legacy event handler for target→element rename**

In `playbook-handler.ts`, the `onLegacyEvent` function (around line 860) reads `cmd.target`. Update to `cmd.element`:

```typescript
    if (cmd.element) {
      flatParams.role = cmd.element.role;
      flatParams.name = cmd.element.name;
    }
```

- [ ] **Step 4: Update speed default and handling**

In `playbook-handler.ts`, change the speed initialization (line 709):

```typescript
  let speed = 0;
```

In `onDispatch` (around line 794), handle optional speed — use 0 (no delay) when omitted:

```typescript
    speed = detail.speed ?? 0;
```

- [ ] **Step 5: Add step-level delay support to executeSequence**

In `executeSequence()`, replace the inter-step delay block (lines 778-781) with logic that prefers `step.delay` when set:

```typescript
        if (stepQueue.length > 0 && !paused) {
          const stepDelay = step.delay;
          const speedDelay = (speed > 0 && speed < 1000) ? Math.max(10, 1000 / speed) : 0;
          const effectiveDelay = stepDelay ?? speedDelay;
          if (effectiveDelay > 0) {
            await new Promise<void>((resolve) => setTimeout(resolve, effectiveDelay));
          }
        }
```

- [ ] **Step 6: Update existing tests for target→element rename**

In `packages/pages-aria/src/server/playbook-handler.test.ts`, update legacy command tests that use `target:` to use `element:`:

Line 57-58 (click command):
```typescript
      element: { role: 'button', name: 'Submit' },
```

Line 88-89 (fill command):
```typescript
      element: { role: 'textbox', name: 'Name' },
```

Line 117 (target not found):
```typescript
      element: { role: 'button', name: 'Nonexistent' },
```

Line 138-139 (unknown action):
```typescript
      element: { role: 'button', name: 'X' },
```

- [ ] **Step 7: Add test for step-level delay override**

Add a new test in the "PlaybookHandler sequence protocol" describe block:

```typescript
  it('step-level delay overrides speed-based delay', async () => {
    vi.useFakeTimers();
    const conn = mockConnection();
    createPlaybookHandler(conn, eventTarget);

    eventTarget.dispatchEvent(new CustomEvent('scenario-dispatch', {
      detail: {
        op: 'dispatch-sequence',
        sessionId: 's-delay',
        steps: [
          { name: 'step-1', label: 'First', action: 'ready', params: {}, delay: 50 },
          { name: 'step-2', label: 'Second', action: 'ready', params: {} },
        ],
        speed: 1,
        paused: false,
      },
    }));

    // step-1 executes immediately, then 50ms delay (not 1000ms from speed=1)
    await vi.advanceTimersByTimeAsync(10);
    let results = conn.sent.filter(
      (m) => (m as Record<string, unknown>).op === 'step-result');
    expect(results).toHaveLength(1);

    // After 50ms delay, step-2 should execute
    await vi.advanceTimersByTimeAsync(50);
    results = conn.sent.filter(
      (m) => (m as Record<string, unknown>).op === 'step-result');
    expect(results).toHaveLength(2);

    vi.useRealTimers();
  });
```

- [ ] **Step 8: Add test for speed omitted (no delay)**

Add a test verifying that omitting speed from DispatchSequence means no delay:

```typescript
  it('omitted speed means no inter-step delay', async () => {
    const conn = mockConnection();
    createPlaybookHandler(conn, eventTarget);

    const start = performance.now();
    eventTarget.dispatchEvent(new CustomEvent('scenario-dispatch', {
      detail: {
        op: 'dispatch-sequence',
        sessionId: 's-nospeed',
        steps: [
          { name: 'step-1', label: 'First', action: 'ready', params: {} },
          { name: 'step-2', label: 'Second', action: 'ready', params: {} },
        ],
        paused: false,
      },
    }));

    await vi.waitFor(() => {
      const results = conn.sent.filter(
        (m) => (m as Record<string, unknown>).op === 'step-result');
      expect(results).toHaveLength(2);
    });

    const elapsed = performance.now() - start;
    expect(elapsed).toBeLessThan(500);
  });
```

- [ ] **Step 9: Run all tests and verify**

Run: `npx vitest run packages/pages-aria/src/server/playbook-handler.test.ts`
Expected: all existing and new tests pass.

- [ ] **Step 10: Commit**

```bash
git add packages/pages-aria/src/playbook/types.ts packages/pages-aria/src/server/playbook-handler.ts packages/pages-aria/src/server/playbook-handler.test.ts
git commit -m "feat(pages-aria): extract dispatch types, rename target→element, add delay/speed refinements

Refs #390"
```

---

## Batch 2: REST action definition

### Task 2: Add REST action to scenario definitions

**Files:**
- Modify: `packages/pages-aria/src/playbook/invoke/definitions.ts` (add REST_ACTION)

**Interfaces:**
- Consumes: `Definition` type from `@casehubio/yaml-core/step`
- Produces: `action: rest` registered in `loadScenarioDefinitions()`

- [ ] **Step 1: Write test for REST action presence**

Create test file `packages/pages-aria/src/playbook/invoke/definitions.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { loadScenarioDefinitions } from './definitions.js';

describe('loadScenarioDefinitions', () => {
  it('includes rest action with correct inputs', () => {
    const files = loadScenarioDefinitions();
    const allActions = Object.assign({}, ...files.map(f => f.actions));
    const rest = allActions['rest'];
    expect(rest).toBeDefined();
    expect(rest.name).toBe('rest');
    expect(rest.inputs.method).toEqual({ type: 'STRING', required: true });
    expect(rest.inputs.url).toEqual({ type: 'STRING', required: true });
    expect(rest.inputs.headers).toBeDefined();
    expect(rest.inputs.body).toBeDefined();
    expect(rest.inputs.expectedStatus).toBeDefined();
    expect(rest.invoke).toEqual({ kind: 'rest' });
    expect(rest.portability).toBe('universal');
  });

  it('includes graphql action', () => {
    const files = loadScenarioDefinitions();
    const allActions = Object.assign({}, ...files.map(f => f.actions));
    expect(allActions['graphql']).toBeDefined();
    expect(allActions['graphql'].invoke).toEqual({ kind: 'graphql-domain' });
  });

  it('includes all ARIA actions', () => {
    const files = loadScenarioDefinitions();
    const allActions = Object.assign({}, ...files.map(f => f.actions));
    expect(allActions['click']).toBeDefined();
    expect(allActions['fill']).toBeDefined();
    expect(allActions['navigate']).toBeDefined();
    expect(allActions['spotlight']).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-aria/src/playbook/invoke/definitions.test.ts`
Expected: FAIL — `rest` action not found

- [ ] **Step 3: Add REST_ACTION definition**

In `packages/pages-aria/src/playbook/invoke/definitions.ts`, add after `SIMULATED_ACTION` (line 47):

```typescript
const REST_ACTION: Definition = {
  name: 'rest', inputs: {
    method: { type: 'STRING', required: true },
    url: { type: 'STRING', required: true },
    headers: { type: 'OBJECT', required: false },
    body: { type: 'OBJECT', required: false },
    expectedStatus: { type: 'INTEGER', required: false },
  }, outputs: {}, invoke: { kind: 'rest' }, portability: 'universal',
};
```

Register it in `loadScenarioDefinitions()` (after line 53):

```typescript
  actions['rest'] = REST_ACTION;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-aria/src/playbook/invoke/definitions.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/playbook/invoke/definitions.ts packages/pages-aria/src/playbook/invoke/definitions.test.ts
git commit -m "feat(pages-aria): add REST action definition to scenario catalog

Refs #390"
```

---

## References

- [GitHub #390](https://github.com/casehubio/casehub-pages/issues/390) — focal issue
- [packages/pages-aria/src/server/playbook-handler.ts] — dispatch handler with local protocol types
- [packages/pages-aria/src/server/playbook-handler.test.ts] — existing handler tests
- [packages/pages-aria/src/playbook/types.ts] — public type module (target for type extraction)
- [packages/pages-aria/src/playbook/invoke/definitions.ts] — action definitions catalog
- [packages/yaml-core/src/step/walker.ts:191-211] — RESERVED_KEYS/DECORATOR_KEYS (already include target, delay, speed)
- [casehubio/parent#409, casehubio/parent#408] — parent format specification
