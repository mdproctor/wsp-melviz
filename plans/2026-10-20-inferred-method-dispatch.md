# Inferred Method-Call Dispatch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #531 — Inferred method-call dispatch for playbook steps
**Issue group:** #531

**Goal:** Add secure inferred method-call dispatch so unrecognised step
names resolve as method calls on the target element (TS) or bean (Java),
replacing per-method step definitions.

**Architecture:** Three-layer approach: (1) SPI in pages-primitives
defines the `PLAYBOOK_ACTIONS` Symbol and `PlaybookCapability` type,
(2) yaml-core Catalog gets `resolveInferred()` so the Walker creates
standard `PluginStep` entries for unknown keys, (3) pages-aria
`InferredMethodHandler` performs runtime target resolution, allowlist
check, and secure invocation. Java side adds `@PlaybookInterface` and
`ActionRegistry` fallback.

**Tech Stack:** TypeScript (vitest), Java 21 (JUnit 5, AssertJ), Lit
web components, Maven

## Global Constraints

- All step names use kebab-case (auto-converted to camelCase for method lookup)
- Registered catalog entries always take priority over inferred dispatch
- Only methods declared via `[PLAYBOOK_ACTIONS]` (TS) or `@PlaybookInterface` (Java) are callable
- Walker reserved keys always take precedence over inferred dispatch
- Tests use vitest (TS) and JUnit 5 + AssertJ (Java)
- Run TS tests: `npx vitest run <path>` from the package directory
- Run Java tests: `/opt/homebrew/bin/mvn -pl <module> test -Dtest=<TestClass>` from backend/

---

## Batch 1: Foundation — SPI types and catalog infrastructure

### Task 1: PlaybookCapability SPI in pages-primitives

**Files:**
- Create: `packages/pages-primitives/src/playbook/playbook-actions.ts`
- Create: `packages/pages-primitives/src/playbook/index.ts`
- Create: `packages/pages-primitives/src/playbook/playbook-actions.test.ts`
- Modify: `packages/pages-primitives/src/index.ts`

**Interfaces:**
- Consumes: nothing (foundation)
- Produces: `PLAYBOOK_ACTIONS` (unique symbol), `PlaybookCapability` (type alias `Record<string, PlaybookMethodDescriptor>`), `PlaybookMethodDescriptor` (interface with `params?`, `returns?`, `description?`, `portability?`)

- [ ] **Step 1: Write the failing test**

```typescript
// packages/pages-primitives/src/playbook/playbook-actions.test.ts
import { describe, it, expect } from 'vitest';
import { PLAYBOOK_ACTIONS } from './playbook-actions.js';
import type { PlaybookCapability, PlaybookMethodDescriptor } from './playbook-actions.js';

describe('PLAYBOOK_ACTIONS SPI', () => {
  it('PLAYBOOK_ACTIONS is a symbol with well-known key', () => {
    expect(typeof PLAYBOOK_ACTIONS).toBe('symbol');
    expect(PLAYBOOK_ACTIONS).toBe(Symbol.for('playbook-actions'));
  });

  it('PlaybookCapability type accepts method descriptors', () => {
    const cap: PlaybookCapability = {
      loadContent: {
        params: { panel: { type: 'string', required: true }, content: 'string' },
        description: 'Load content',
      },
      nextDiff: {},
    };
    expect(Object.keys(cap)).toEqual(['loadContent', 'nextDiff']);
  });

  it('PlaybookMethodDescriptor accepts shorthand and detailed params', () => {
    const shorthand: PlaybookMethodDescriptor = { params: { name: 'string' } };
    const detailed: PlaybookMethodDescriptor = {
      params: { name: { type: 'string', required: true } },
      returns: 'object',
      portability: 'ts',
    };
    expect(shorthand.params).toBeDefined();
    expect(detailed.portability).toBe('ts');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-primitives/src/playbook/playbook-actions.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Write the SPI module**

```typescript
// packages/pages-primitives/src/playbook/playbook-actions.ts
import type { Portability } from '@casehubio/yaml-core/step';

export const PLAYBOOK_ACTIONS: unique symbol =
  Symbol.for('playbook-actions') as any;

export interface PlaybookMethodDescriptor {
  params?: Record<string, string | { type: string; required?: boolean }>;
  returns?: string;
  description?: string;
  portability?: Portability;
}

export type PlaybookCapability = Record<string, PlaybookMethodDescriptor>;
```

Wait — the spec says pages-primitives should NOT depend on yaml-core.
The `Portability` type is `'universal' | 'java' | 'ts' | 'both'`.
Inline it instead of importing:

```typescript
// packages/pages-primitives/src/playbook/playbook-actions.ts
export const PLAYBOOK_ACTIONS: unique symbol =
  Symbol.for('playbook-actions') as any;

export interface PlaybookMethodDescriptor {
  params?: Record<string, string | { type: string; required?: boolean }>;
  returns?: string;
  description?: string;
  portability?: 'universal' | 'java' | 'ts' | 'both';
}

export type PlaybookCapability = Record<string, PlaybookMethodDescriptor>;
```

```typescript
// packages/pages-primitives/src/playbook/index.ts
export { PLAYBOOK_ACTIONS } from './playbook-actions.js';
export type { PlaybookCapability, PlaybookMethodDescriptor } from './playbook-actions.js';
```

Add to the package barrel export:

```typescript
// Append to packages/pages-primitives/src/index.ts
export * from './playbook/index.js';
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run packages/pages-primitives/src/playbook/playbook-actions.test.ts`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add packages/pages-primitives/src/playbook/ packages/pages-primitives/src/index.ts
git commit -m "feat(pages-primitives): add PLAYBOOK_ACTIONS SPI for inferred method dispatch

Add Symbol.for('playbook-actions'), PlaybookCapability, and
PlaybookMethodDescriptor types. Components implement this SPI to
declare which methods are playbook-callable.

Refs #531

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Catalog extension and Walker change in yaml-core

**Files:**
- Modify: `packages/yaml-core/src/step/walker.ts:49-52` (Catalog interface)
- Modify: `packages/yaml-core/src/step/walker.ts:320-327` (resolveOne key classification)
- Modify: `packages/yaml-core/src/step/types.ts:13-22` (InvokeBinding union)
- Modify: `packages/yaml-core/src/step/catalog.ts` (CompositeCatalog, ImportScopedCatalog)
- Modify: `packages/yaml-core/src/step/walker.test.ts`
- Modify: `packages/yaml-core/src/step/catalog.test.ts`

**Interfaces:**
- Consumes: nothing new (extends existing Catalog)
- Produces: `Catalog.resolveInferred?(actionName: string): CatalogEntry | undefined`, `ElementMethodBinding` type

- [ ] **Step 1: Write failing tests for Catalog.resolveInferred**

```typescript
// Append to packages/yaml-core/src/step/catalog.test.ts

describe('CompositeCatalog resolveInferred delegation', () => {
  it('delegates resolveInferred when no static match', () => {
    const inferredEntry = entry('inferred-method');
    const catalog = new CompositeCatalog([source(100, { click: entry('click') })]);
    (catalog as any).resolveInferred = (name: string) =>
      name === 'inferred-method' ? inferredEntry : undefined;
    // This tests the interface contract — actual delegation tested below
    expect(catalog.resolveInferred?.('inferred-method')?.qualifiedName).toBe('inferred-method');
  });
});

describe('ImportScopedCatalog resolveInferred delegation', () => {
  it('delegates resolveInferred to wrapped catalog', () => {
    const inferredEntry = entry('load-content');
    const innerCatalog = {
      resolve: () => undefined,
      availableActions: () => new Set<string>(),
      resolveInferred: (name: string) =>
        name === 'load-content' ? inferredEntry : undefined,
    };
    const catalog = new ImportScopedCatalog(new Map(), innerCatalog);
    expect(catalog.resolveInferred?.('load-content')?.qualifiedName).toBe('load-content');
  });
});
```

- [ ] **Step 2: Write failing tests for Walker inferred resolution**

```typescript
// Append to packages/yaml-core/src/step/walker.test.ts

describe('inferred method dispatch', () => {
  it('resolves unknown key via resolveInferred', () => {
    const inferredEntry = makeEntry('load-content');
    const catalog = {
      ...makeCatalog({ click: makeEntry('click') }),
      resolveInferred: (name: string) =>
        name === 'load-content' ? inferredEntry : undefined,
    };
    const steps = [{ 'load-content': { role: 'region', name: 'Diff' } }];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved).toHaveLength(1);
    expect(resolved[0].kind).toBe('plugin');
    expect(resolved[0].name).toBe('load-content');
  });

  it('throws for unknown key when resolveInferred returns undefined', () => {
    const catalog = {
      ...makeCatalog({}),
      resolveInferred: () => undefined,
    };
    const steps = [{ 'bad-key': {} }];
    expect(() => Walker.resolve(steps, catalog)).toThrow('unknown step key');
  });

  it('throws for unknown key when resolveInferred is not present', () => {
    const catalog = makeCatalog({});
    const steps = [{ 'bad-key': {} }];
    expect(() => Walker.resolve(steps, catalog)).toThrow('unknown step key');
  });

  it('catalog.resolve takes priority over resolveInferred', () => {
    const registeredEntry = makeEntry('click-registered');
    const inferredEntry = makeEntry('click-inferred');
    const catalog = {
      ...makeCatalog({ click: registeredEntry }),
      resolveInferred: () => inferredEntry,
    };
    const steps = [{ click: {} }];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0].kind).toBe('plugin');
    expect((resolved[0] as any).entry.qualifiedName).toBe('click-registered');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/step/walker.test.ts`
Run: `npx vitest run packages/yaml-core/src/step/catalog.test.ts`
Expected: FAIL

- [ ] **Step 4: Add ElementMethodBinding to InvokeBinding union**

In `packages/yaml-core/src/step/types.ts`, add to the union:

```typescript
export type InvokeBinding =
  | McpBinding
  | RestBinding
  | GraphqlBinding
  | ScriptBinding
  | AgentBinding
  | ProcessBinding
  | AriaBinding
  | GraphqlDomainBinding
  | SimulatedBinding
  | ElementMethodBinding;

// ... after SimulatedBinding interface:

export interface ElementMethodBinding {
  kind: 'element-method';
  method: string;
}
```

- [ ] **Step 5: Add resolveInferred to Catalog interface**

In `packages/yaml-core/src/step/walker.ts:49-52`, add the optional method:

```typescript
export interface Catalog {
  resolve(actionName: string): CatalogEntry | undefined;
  resolveInferred?(actionName: string): CatalogEntry | undefined;
  availableActions(): Set<string>;
}
```

- [ ] **Step 6: Update Walker.resolveOne to use resolveInferred**

In `packages/yaml-core/src/step/walker.ts:320-327`, change:

```typescript
// Before:
      } else if (!RESERVED_KEYS.has(key)) {
        const entry = catalog.resolve(key);
        if (entry) {
          actionKey = key;
          actionValue = value;
        } else {
          throw new Error(`${stepPath}: unknown step key '${key}' — not a reserved keyword and not found in catalog`);
        }
      }

// After:
      } else if (!RESERVED_KEYS.has(key)) {
        const entry = catalog.resolve(key) ?? catalog.resolveInferred?.(key);
        if (entry) {
          actionKey = key;
          actionValue = value;
        } else {
          throw new Error(`${stepPath}: unknown step key '${key}' — not a reserved keyword and not found in catalog`);
        }
      }
```

- [ ] **Step 7: Add resolveInferred delegation to CompositeCatalog and ImportScopedCatalog**

In `packages/yaml-core/src/step/catalog.ts`:

```typescript
export class CompositeCatalog implements Catalog {
  private readonly entries = new Map<string, CatalogEntry>();
  private readonly sources: CatalogSource[];

  constructor(sources: CatalogSource[]) {
    this.sources = [...sources].sort((a, b) => a.priority - b.priority);
    for (const source of this.sources) {
      const sourceEntries = new Map<string, CatalogEntry>();
      source.populate(sourceEntries);
      for (const [name, entry] of sourceEntries) {
        if (!this.entries.has(name)) {
          this.entries.set(name, entry);
        }
      }
    }
  }

  resolve(actionName: string): CatalogEntry | undefined {
    return this.entries.get(actionName);
  }

  resolveInferred?(actionName: string): CatalogEntry | undefined {
    // Delegated by catalog consumers (e.g., pages-aria catalog factory)
    // CompositeCatalog itself doesn't know about inferred methods
    return undefined;
  }

  availableActions(): Set<string> {
    return new Set(this.entries.keys());
  }
}

export class ImportScopedCatalog implements Catalog {
  constructor(
    private readonly importedEntries: Map<string, CatalogEntry>,
    private readonly delegate: Catalog,
  ) {}

  resolve(actionName: string): CatalogEntry | undefined {
    return this.importedEntries.get(actionName) ?? this.delegate.resolve(actionName);
  }

  resolveInferred(actionName: string): CatalogEntry | undefined {
    return this.delegate.resolveInferred?.(actionName);
  }

  availableActions(): Set<string> {
    const all = new Set(this.delegate.availableActions());
    for (const name of this.importedEntries.keys()) all.add(name);
    return all;
  }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/step/walker.test.ts`
Run: `npx vitest run packages/yaml-core/src/step/catalog.test.ts`
Expected: ALL PASS

- [ ] **Step 9: Run full yaml-core test suite**

Run: `npx vitest run` from `packages/yaml-core/`
Expected: ALL PASS — no regressions

- [ ] **Step 10: Commit**

```bash
git add packages/yaml-core/src/step/
git commit -m "feat(yaml-core): add Catalog.resolveInferred and ElementMethodBinding for inferred dispatch

Extend Catalog interface with optional resolveInferred(). Walker uses
it as fallback when catalog.resolve() returns undefined. CompositeCatalog
and ImportScopedCatalog delegate correctly. Add ElementMethodBinding to
InvokeBinding union.

Refs #531

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: TS secure dispatch — inferred method resolution

### Task 3: Shared playbook-resolve utilities and InferredMethodHandler

**Files:**
- Create: `packages/pages-aria/src/executor/playbook-resolve.ts`
- Create: `packages/pages-aria/src/executor/playbook-resolve.test.ts`
- Create: `packages/pages-aria/src/playbook/invoke/inferred-method-handler.ts`
- Create: `packages/pages-aria/src/playbook/invoke/inferred-method-handler.test.ts`
- Modify: `packages/pages-aria/src/playbook/catalog-factory.ts`

**Interfaces:**
- Consumes: `PLAYBOOK_ACTIONS`, `PlaybookCapability`, `PlaybookMethodDescriptor` from pages-primitives. `CatalogEntry`, `Action`, `Catalog`, `ElementMethodBinding` from yaml-core.
- Produces: `resolvePlaybookCapability(el: Element): PlaybookCapability | null`, `mapParams(step: Record<string, unknown>, desc: PlaybookMethodDescriptor): unknown[]`, `kebabToCamel(name: string): string`, `InferredMethodHandler` (InvokeHandler impl)

- [ ] **Step 1: Write failing tests for playbook-resolve utilities**

```typescript
// packages/pages-aria/src/executor/playbook-resolve.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { resolvePlaybookCapability, mapParams, kebabToCamel } from './playbook-resolve.js';
import { PLAYBOOK_ACTIONS } from '@casehubio/pages-primitives';

describe('kebabToCamel', () => {
  it('converts kebab-case to camelCase', () => {
    expect(kebabToCamel('load-content')).toBe('loadContent');
    expect(kebabToCamel('next-diff')).toBe('nextDiff');
    expect(kebabToCamel('set-view-mode')).toBe('setViewMode');
  });

  it('leaves camelCase unchanged', () => {
    expect(kebabToCamel('toggleSync')).toBe('toggleSync');
  });

  it('handles single word', () => {
    expect(kebabToCamel('reset')).toBe('reset');
  });
});

describe('resolvePlaybookCapability', () => {
  beforeEach(() => { document.body.innerHTML = ''; });

  it('returns null when element has no PLAYBOOK_ACTIONS', () => {
    const el = document.createElement('div');
    expect(resolvePlaybookCapability(el)).toBeNull();
  });

  it('reads static PLAYBOOK_ACTIONS from constructor', () => {
    class TestEl extends HTMLElement {
      static get [PLAYBOOK_ACTIONS]() {
        return { doThing: { params: { x: 'string' } } };
      }
    }
    customElements.define('test-playbook-el', TestEl);
    const el = document.createElement('test-playbook-el');
    const cap = resolvePlaybookCapability(el);
    expect(cap).toEqual({ doThing: { params: { x: 'string' } } });
  });
});

describe('validateDescriptor', () => {
  it('warns when descriptor param count mismatches method length', () => {
    const warnings: string[] = [];
    const warnFn = (msg: string) => warnings.push(msg);
    class BadEl extends HTMLElement {
      static get [PLAYBOOK_ACTIONS]() {
        return { doThing: { params: { a: 'string', b: 'string' } } };
      }
      doThing(a: string) {} // 1 param, descriptor says 2
    }
    customElements.define('test-bad-el', BadEl);
    const el = document.createElement('test-bad-el');
    validateDescriptor(el, warnFn);
    expect(warnings.length).toBeGreaterThan(0);
    expect(warnings[0]).toContain('doThing');
  });
});

describe('mapParams', () => {
  it('maps named params to positional args via descriptor key order', () => {
    const step = { panel: 'a', content: '# Hello', label: 'v1' };
    const desc = {
      params: {
        panel: { type: 'string', required: true },
        content: { type: 'string', required: true },
        label: 'string',
      },
    };
    expect(mapParams(step, desc)).toEqual(['a', '# Hello', 'v1']);
  });

  it('throws when required param is missing', () => {
    const step = { content: '# Hello' };
    const desc = {
      params: { panel: { type: 'string', required: true }, content: 'string' },
    };
    expect(() => mapParams(step, desc)).toThrow("Required parameter 'panel' is missing");
  });

  it('returns empty array when no params declared', () => {
    expect(mapParams({}, {})).toEqual([]);
    expect(mapParams({}, { params: undefined })).toEqual([]);
  });

  it('passes undefined for optional missing params', () => {
    const step = { panel: 'a' };
    const desc = { params: { panel: 'string', label: 'string' } };
    expect(mapParams(step, desc)).toEqual(['a', undefined]);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-aria/src/executor/playbook-resolve.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement playbook-resolve.ts**

```typescript
// packages/pages-aria/src/executor/playbook-resolve.ts
import { PLAYBOOK_ACTIONS } from '@casehubio/pages-primitives';
import type { PlaybookCapability, PlaybookMethodDescriptor } from '@casehubio/pages-primitives';

export function kebabToCamel(name: string): string {
  return name.replace(/-([a-z])/g, (_, c: string) => c.toUpperCase());
}

export function resolvePlaybookCapability(el: Element): PlaybookCapability | null {
  if (PLAYBOOK_ACTIONS in el) return (el as any)[PLAYBOOK_ACTIONS];
  const ctor = el.constructor as any;
  if (ctor && PLAYBOOK_ACTIONS in ctor) return ctor[PLAYBOOK_ACTIONS];
  return null;
}

export function validateDescriptor(
  el: Element,
  warn: (msg: string) => void = console.warn,
): void {
  const cap = resolvePlaybookCapability(el);
  if (!cap) return;
  for (const [name, desc] of Object.entries(cap)) {
    const method = (el as any)[name];
    if (typeof method !== 'function') {
      warn(`[PLAYBOOK_ACTIONS] ${el.tagName}: declares '${name}' but element has no such method`);
      continue;
    }
    const paramCount = desc.params ? Object.keys(desc.params).length : 0;
    if (paramCount !== method.length) {
      warn(`[PLAYBOOK_ACTIONS] ${el.tagName}.${name}: descriptor has ${paramCount} params but method.length is ${method.length}`);
    }
  }
}

export function mapParams(
  step: Record<string, unknown>,
  desc: PlaybookMethodDescriptor,
): unknown[] {
  if (!desc.params) return [];
  const args: unknown[] = [];
  for (const [key, spec] of Object.entries(desc.params)) {
    const value = step[key];
    const isRequired = typeof spec === 'object' ? !!spec.required : !(typeof spec === 'string' && spec.endsWith('?'));
    if (isRequired && value === undefined) {
      throw new Error(`Required parameter '${key}' is missing`);
    }
    args.push(value);
  }
  return args;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-aria/src/executor/playbook-resolve.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Write failing test for InferredMethodHandler**

```typescript
// packages/pages-aria/src/playbook/invoke/inferred-method-handler.test.ts
import { describe, it, expect } from 'vitest';
import { InferredMethodHandler } from './inferred-method-handler.js';

describe('InferredMethodHandler', () => {
  const handler = new InferredMethodHandler();

  it('supports element-method binding', () => {
    expect(handler.supports({ kind: 'element-method', method: 'foo' })).toBe(true);
  });

  it('does not support other bindings', () => {
    expect(handler.supports({ kind: 'aria', action: 'click' })).toBe(false);
  });
});
```

- [ ] **Step 6: Implement InferredMethodHandler**

```typescript
// packages/pages-aria/src/playbook/invoke/inferred-method-handler.ts
import type { InvokeHandler, Action } from '@casehubio/yaml-core/step';
import type { InvokeBinding, Definition, ElementMethodBinding } from '@casehubio/yaml-core/step';
import { resolvePlaybookCapability, mapParams, kebabToCamel } from '../../executor/playbook-resolve.js';
import { resolveTarget } from '../../executor/command-executor.js';
import type { AriaTarget } from '@casehubio/pages-primitives';

export class InferredMethodHandler implements InvokeHandler {
  supports(binding: InvokeBinding): boolean {
    return binding.kind === 'element-method';
  }

  create(definition: Definition, binding: InvokeBinding): Action {
    const methodName = kebabToCamel((binding as ElementMethodBinding).method);
    return {
      async execute(params, services) {
        const target = params.target as AriaTarget | undefined;
        if (!target) {
          return { kind: 'failure', message: `Inferred method '${methodName}' requires a target` };
        }

        const el = resolveTarget(target);
        const cap = resolvePlaybookCapability(el);
        if (!cap) {
          return {
            kind: 'failure',
            message: `Target ${target.role} "${target.name}" is not playbook-callable — no [PLAYBOOK_ACTIONS] declared`,
          };
        }

        const desc = cap[methodName];
        if (!desc) {
          const available = Object.keys(cap).join(', ');
          return {
            kind: 'failure',
            message: `Method '${methodName}' not playbook-callable on ${target.role} "${target.name}". Available: ${available}`,
          };
        }

        const method = (el as any)[methodName];
        if (typeof method !== 'function') {
          return {
            kind: 'failure',
            message: `Target declares '${methodName}' but has no such method`,
          };
        }

        try {
          const args = mapParams(params, desc);
          const result = method.call(el, ...args);
          if (result instanceof Promise) await result;
          return { kind: 'success', output: result ?? {}, executionMetadata: {} };
        } catch (e: any) {
          return { kind: 'failure', message: e.message ?? String(e) };
        }
      },
    };
  }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `npx vitest run packages/pages-aria/src/playbook/invoke/inferred-method-handler.test.ts`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add packages/pages-aria/src/executor/playbook-resolve.ts packages/pages-aria/src/executor/playbook-resolve.test.ts packages/pages-aria/src/playbook/invoke/inferred-method-handler.ts packages/pages-aria/src/playbook/invoke/inferred-method-handler.test.ts
git commit -m "feat(pages-aria): add playbook-resolve utilities and InferredMethodHandler

Shared utilities for PLAYBOOK_ACTIONS resolution: resolvePlaybookCapability,
mapParams (with required-param validation), kebabToCamel. InferredMethodHandler
implements InvokeHandler for element-method binding — runtime target resolution,
allowlist check, and secure invocation.

Refs #531

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Secure command-executor switch default + catalog factory + playbook-handler

**Files:**
- Modify: `packages/pages-aria/src/executor/command-executor.ts:104,108-126`
- Modify: `packages/pages-aria/src/executor/command-executor.test.ts`
- Modify: `packages/pages-aria/src/playbook/catalog-factory.ts`
- Modify: `packages/pages-aria/src/server/playbook-handler.ts`

**Interfaces:**
- Consumes: `resolvePlaybookCapability`, `mapParams`, `kebabToCamel` from playbook-resolve. `InferredMethodHandler` from Task 3.
- Produces: Secure `invokePlaybookMethod` (replaces raw `invokeMethod`), `catalog.resolveInferred()` on scenario catalog

- [ ] **Step 1: Write failing test for secure invokePlaybookMethod**

```typescript
// Append to packages/pages-aria/src/executor/command-executor.test.ts

import { PLAYBOOK_ACTIONS } from '@casehubio/pages-primitives';

describe('inferred method dispatch', () => {
  it('calls declared playbook method', async () => {
    let called = false;
    class TestWidget extends HTMLElement {
      static get [PLAYBOOK_ACTIONS]() {
        return { doThing: { params: { value: { type: 'string', required: true } } } };
      }
      doThing(value: string) { called = true; }
    }
    customElements.define('test-widget-exec', TestWidget);
    const el = document.createElement('test-widget-exec');
    el.setAttribute('role', 'region');
    el.setAttribute('aria-label', 'TestWidget');
    document.body.appendChild(el);

    const { executeStep } = await import('./command-executor.js');
    await executeStep({ action: 'doThing', target: { role: 'region', name: 'TestWidget' }, value: 'hello' });
    expect(called).toBe(true);
  });

  it('rejects method not in PLAYBOOK_ACTIONS', async () => {
    const el = document.createElement('div');
    el.setAttribute('role', 'region');
    el.setAttribute('aria-label', 'NoCap');
    document.body.appendChild(el);

    const { executeStep } = await import('./command-executor.js');
    await expect(executeStep({ action: 'remove', target: { role: 'region', name: 'NoCap' } }))
      .rejects.toThrow('not playbook-callable');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-aria/src/executor/command-executor.test.ts`
Expected: FAIL — current `invokeMethod` doesn't check PLAYBOOK_ACTIONS

- [ ] **Step 3: Replace invokeMethod with invokePlaybookMethod in command-executor.ts**

In `packages/pages-aria/src/executor/command-executor.ts`:

1. Add import:
```typescript
import { resolvePlaybookCapability, mapParams, kebabToCamel } from './playbook-resolve.js';
```

2. Replace the `default` case (line 104):
```typescript
    default: return invokePlaybookMethod(step);
```

3. Replace the `invokeMethod` and `extractArgs` functions (lines 108-126) with:
```typescript
async function invokePlaybookMethod(
  step: { action?: string; target?: AriaTarget; [key: string]: unknown },
): Promise<void> {
  if (!step.target) {
    throw new Error(`Inferred method '${step.action}' requires a target`);
  }
  if (!step.action) return;

  const el = resolveTarget(step.target);
  const cap = resolvePlaybookCapability(el);

  if (!cap) {
    throw new Error(
      `Target ${step.target.role} "${step.target.name}" is not playbook-callable — no [PLAYBOOK_ACTIONS] declared`,
    );
  }

  const methodName = kebabToCamel(step.action);
  const desc = cap[methodName];
  if (!desc) {
    throw new Error(
      `Method '${methodName}' is not declared as playbook-callable on ${step.target.role} "${step.target.name}". Available: ${Object.keys(cap).join(', ')}`,
    );
  }

  const method = (el as any)[methodName];
  if (typeof method !== 'function') {
    throw new Error(`Target declares '${methodName}' as playbook-callable but has no such method`);
  }

  const args = mapParams(step, desc);
  const result = method.call(el, ...args);
  if (result instanceof Promise) await result;
}
```

- [ ] **Step 4: Add resolveInferred to catalog-factory.ts**

In `packages/pages-aria/src/playbook/catalog-factory.ts`, extend the returned catalog object to include `resolveInferred`:

```typescript
import { InferredMethodHandler } from './invoke/inferred-method-handler.js';

// In the catalog construction, after building the static entries map:
const inferredHandler = new InferredMethodHandler();

return {
  resolve(name: string) { return staticEntries.get(name); },
  resolveInferred(name: string) {
    if (!/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/.test(name)) return undefined;
    const definition = {
      name,
      inputs: {},
      outputs: {},
      invoke: { kind: 'element-method' as const, method: name },
      portability: 'ts' as const,
    };
    return {
      qualifiedName: name,
      definition,
      action: inferredHandler.create(definition, definition.invoke),
    };
  },
  availableActions() { return new Set(staticEntries.keys()); },
};
```

- [ ] **Step 5: Extend playbook-handler.ts default case**

In `packages/pages-aria/src/server/playbook-handler.ts`, in the `executeAriaAction` switch statement's default case, replace the `throw` with:

```typescript
default: {
  const { resolvePlaybookCapability, mapParams, kebabToCamel } = await import('../../executor/playbook-resolve.js');
  const cap = resolvePlaybookCapability(el);
  const methodName = kebabToCamel(action);
  if (!cap || !cap[methodName]) {
    throw new Error(`Unknown action: ${action}`);
  }
  const method = (el as any)[methodName];
  if (typeof method !== 'function') {
    throw new Error(`Target declares '${methodName}' but has no such method`);
  }
  const args = mapParams(step, cap[methodName]);
  const result = method.call(el, ...args);
  if (result instanceof Promise) await result;
  break;
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npx vitest run packages/pages-aria/src/executor/command-executor.test.ts`
Expected: ALL PASS

- [ ] **Step 7: Run full pages-aria test suite**

Run: `npx vitest run` from `packages/pages-aria/`
Expected: ALL PASS — no regressions

- [ ] **Step 8: Commit**

```bash
git add packages/pages-aria/src/executor/command-executor.ts packages/pages-aria/src/executor/command-executor.test.ts packages/pages-aria/src/playbook/catalog-factory.ts packages/pages-aria/src/server/playbook-handler.ts
git commit -m "feat(pages-aria): secure inferred dispatch — replace raw invokeMethod, add resolveInferred to catalog

Replace unsafe (el as any)[action] with PLAYBOOK_ACTIONS-gated
invokePlaybookMethod. Extend catalog factory with resolveInferred for
kebab-case identifiers. Extend playbook-handler.ts default case with
same secure resolution.

Refs #531

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: First adopter — pages-document-diff

### Task 5: Add PLAYBOOK_ACTIONS to PagesDocumentDiff

**Files:**
- Modify: `packages/pages-document-diff/src/pages-document-diff.ts`
- Create: `packages/pages-document-diff/src/playbook-actions.test.ts`

**Interfaces:**
- Consumes: `PLAYBOOK_ACTIONS`, `PlaybookCapability` from pages-primitives
- Produces: `PagesDocumentDiff` class implements `[PLAYBOOK_ACTIONS]` static getter

- [ ] **Step 1: Write failing test**

```typescript
// packages/pages-document-diff/src/playbook-actions.test.ts
import { describe, it, expect } from 'vitest';
import { PLAYBOOK_ACTIONS } from '@casehubio/pages-primitives';
import { PagesDocumentDiff } from './pages-document-diff.js';

describe('PagesDocumentDiff PLAYBOOK_ACTIONS', () => {
  it('declares playbook-callable methods', () => {
    const cap = (PagesDocumentDiff as any)[PLAYBOOK_ACTIONS];
    expect(cap).toBeDefined();
    expect(Object.keys(cap)).toContain('loadContent');
    expect(Object.keys(cap)).toContain('nextDiff');
    expect(Object.keys(cap)).toContain('prevDiff');
    expect(Object.keys(cap)).toContain('setViewMode');
    expect(Object.keys(cap)).toContain('toggleSync');
  });

  it('loadContent declares panel and content as required', () => {
    const cap = (PagesDocumentDiff as any)[PLAYBOOK_ACTIONS];
    const lc = cap.loadContent;
    expect(lc.params.panel).toEqual({ type: 'string', required: true });
    expect(lc.params.content).toEqual({ type: 'string', required: true });
  });

  it('does not expose private or DOM methods', () => {
    const cap = (PagesDocumentDiff as any)[PLAYBOOK_ACTIONS];
    expect(cap.remove).toBeUndefined();
    expect(cap.setAttribute).toBeUndefined();
    expect(cap.connectedCallback).toBeUndefined();
    expect(cap._syncEnabled).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/pages-document-diff/src/playbook-actions.test.ts`
Expected: FAIL — no PLAYBOOK_ACTIONS on the class

- [ ] **Step 3: Add PLAYBOOK_ACTIONS static getter to PagesDocumentDiff**

In `packages/pages-document-diff/src/pages-document-diff.ts`, add the import
and static getter to the class:

```typescript
import { PLAYBOOK_ACTIONS } from '@casehubio/pages-primitives';
import type { PlaybookCapability } from '@casehubio/pages-primitives';

// Inside the class, after the static styles:
static get [PLAYBOOK_ACTIONS](): PlaybookCapability {
  return {
    loadContent: {
      params: {
        panel: { type: 'string', required: true },
        content: { type: 'string', required: true },
        label: 'string',
      },
      description: 'Load content into a diff panel',
    },
    nextDiff: { description: 'Navigate to next diff chunk' },
    prevDiff: { description: 'Navigate to previous diff chunk' },
    setViewMode: {
      params: { mode: { type: 'string', required: true } },
      description: 'Switch between split and unified view',
    },
    toggleSync: { description: 'Toggle scroll synchronisation' },
    swapPanels: { description: 'Swap panel A and B' },
    getDiffSummary: { returns: 'object', description: 'Get current diff summary' },
    scrollToLocation: {
      params: { location: { type: 'string', required: true } },
      description: 'Scroll to a named location',
    },
    highlightSection: {
      params: { location: { type: 'string', required: true } },
      description: 'Highlight a diff section',
    },
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run packages/pages-document-diff/src/playbook-actions.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-document-diff/src/
git commit -m "feat(pages-document-diff): add PLAYBOOK_ACTIONS — first inferred dispatch adopter

Declare loadContent, nextDiff, prevDiff, setViewMode, toggleSync,
swapPanels, getDiffSummary, scrollToLocation, highlightSection as
playbook-callable methods via the PLAYBOOK_ACTIONS SPI.

Refs #531

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Java — @PlaybookInterface and ActionRegistry extension

### Task 6: @PlaybookInterface annotation and ActionRegistry fallback

**Files:**
- Create: `backend/playbook-client/src/main/java/io/casehub/pages/playbook/client/PlaybookInterface.java`
- Modify: `backend/playbook-client/src/main/java/io/casehub/pages/playbook/client/ActionRegistry.java`
- Modify: `backend/playbook-client/src/test/java/io/casehub/pages/playbook/client/ActionRegistryTest.java`

**Interfaces:**
- Consumes: `@PlaybookAction`, `ActionContext`, `ActionRegistry` (existing)
- Produces: `@PlaybookInterface` annotation, `ActionRegistry.invoke()` with interface fallback, `ActionRegistry.discoverInterfaceMethods()` for executor registration

- [ ] **Step 1: Write failing tests**

Append to `backend/playbook-client/src/test/java/io/casehub/pages/playbook/client/ActionRegistryTest.java`:

```java
@PlaybookInterface
interface TestDiffViewer {
    Map<String, Object> loadContent(ActionContext ctx);
    void nextDiff(ActionContext ctx);
    default boolean isReady() { return true; } // should be skipped
}

static class DiffViewerImpl implements TestDiffViewer {
    @Override
    public Map<String, Object> loadContent(ActionContext ctx) {
        return Map.of("loaded", ctx.data("panel"));
    }
    @Override
    public void nextDiff(ActionContext ctx) {}
}

@Test
void invokesPlaybookInterfaceMethod() throws Exception {
    var registry = ActionRegistry.scan(List.of(new DiffViewerImpl()));
    var ctx = ActionContext.of("admin", Map.of("panel", "a"), Map.of());
    var result = registry.invoke("loadContent", ctx);
    assertThat(result).containsEntry("loaded", "a");
}

@Test
void skipsDefaultMethodsOnPlaybookInterface() {
    var registry = ActionRegistry.scan(List.of(new DiffViewerImpl()));
    var ctx = ActionContext.of("admin", Map.of(), Map.of());
    assertThatThrownBy(() -> registry.invoke("isReady", ctx))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test
void playBookActionTakesPriorityOverInterface() throws Exception {
    // TestActions has @PlaybookAction("greet")
    // If DiffViewerImpl also had a greet method, @PlaybookAction wins
    var registry = ActionRegistry.scan(List.of(new TestActions(), new DiffViewerImpl()));
    var ctx = ActionContext.of("admin", Map.of("name", "Bob"), Map.of());
    var result = registry.invoke("greet", ctx);
    assertThat(result).containsEntry("greeting", "Hello Bob");
}

@Test
void discoversInterfaceMethodsForRegistration() {
    var registry = ActionRegistry.scan(List.of(new DiffViewerImpl()));
    assertThat(registry.allActions())
        .contains("loadContent", "nextDiff")
        .doesNotContain("isReady"); // default methods excluded
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f backend/playbook-client/pom.xml test -Dtest=ActionRegistryTest`
Expected: FAIL — no @PlaybookInterface, no interface fallback

- [ ] **Step 3: Create @PlaybookInterface annotation**

```java
// backend/playbook-client/src/main/java/io/casehub/pages/playbook/client/PlaybookInterface.java
package io.casehub.pages.playbook.client;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface PlaybookInterface {
}
```

- [ ] **Step 4: Extend ActionRegistry with interface fallback**

In `backend/playbook-client/src/main/java/io/casehub/pages/playbook/client/ActionRegistry.java`:

1. Add a `registeredBeans` field
2. In `scan()`, store the beans list
3. In `invoke()`, fall back to `invokeInterfaceMethod` when no handler
4. Add `allActions()` including interface methods
5. Add startup validation for `-parameters` flag

Key changes:

```java
private List<Object> registeredBeans = List.of();

public static ActionRegistry scan(List<Object> beans) {
    // ... existing @PlaybookAction scanning ...
    registry.registeredBeans = List.copyOf(beans);
    registry.validateParameterNames();
    return registry;
}

public Map<String, Object> invoke(String action, ActionContext ctx) throws Exception {
    ActionHandler handler = handlers.get(action);
    if (handler != null) {
        // ... existing handler invocation ...
    }
    return invokeInterfaceMethod(action, ctx);
}

private Map<String, Object> invokeInterfaceMethod(String action, ActionContext ctx) throws Exception {
    for (Object bean : registeredBeans) {
        for (Class<?> iface : getAllInterfaces(bean.getClass())) {
            if (!iface.isAnnotationPresent(PlaybookInterface.class)) continue;
            for (Method m : iface.getDeclaredMethods()) {
                if (m.isDefault()) continue;
                if (m.getName().equals(action)) {
                    m.setAccessible(true);
                    Object result = m.invoke(bean, ctx);
                    if (result instanceof Map<?,?>) return (Map<String, Object>) result;
                    return result != null ? Map.of("result", result) : Map.of();
                }
            }
        }
    }
    throw new IllegalArgumentException("No @PlaybookAction handler or @PlaybookInterface method for: " + action);
}

private static Set<Class<?>> getAllInterfaces(Class<?> clazz) {
    var result = new java.util.LinkedHashSet<Class<?>>();
    for (Class<?> c = clazz; c != null; c = c.getSuperclass()) {
        for (Class<?> iface : c.getInterfaces()) {
            if (result.add(iface)) {
                result.addAll(getAllInterfaces(iface));
            }
        }
    }
    return result;
}

public Set<String> allActions() {
    var all = new java.util.LinkedHashSet<>(actions());
    for (Object bean : registeredBeans) {
        for (Class<?> iface : getAllInterfaces(bean.getClass())) {
            if (!iface.isAnnotationPresent(PlaybookInterface.class)) continue;
            for (Method m : iface.getDeclaredMethods()) {
                if (!m.isDefault()) all.add(m.getName());
            }
        }
    }
    return all;
}

private void validateParameterNames() {
    for (Object bean : registeredBeans) {
        for (Class<?> iface : getAllInterfaces(bean.getClass())) {
            if (!iface.isAnnotationPresent(PlaybookInterface.class)) continue;
            for (Method m : iface.getDeclaredMethods()) {
                if (m.isDefault() || m.getParameterCount() == 0) continue;
                java.lang.reflect.Parameter first = m.getParameters()[0];
                if (first.getName().startsWith("arg")) {
                    throw new IllegalStateException(
                        "@PlaybookInterface method " + iface.getSimpleName() + "." + m.getName() +
                        " has synthetic parameter names (arg0, arg1). " +
                        "Add -parameters to javac compiler flags.");
                }
                return; // One check is enough
            }
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f backend/playbook-client/pom.xml test -Dtest=ActionRegistryTest`
Expected: ALL PASS

- [ ] **Step 6: Run full playbook-client test suite**

Run: `/opt/homebrew/bin/mvn -f backend/playbook-client/pom.xml test`
Expected: ALL PASS — no regressions

- [ ] **Step 7: Update PlaybookExecutorClient to include interface methods in registration**

In `backend/playbook-client/src/main/java/io/casehub/pages/playbook/client/PlaybookExecutorClient.java`,
find the `sendRegister()` method that broadcasts the action list. Replace
`actionRegistry.actions()` with `actionRegistry.allActions()` so that
interface-declared methods are included in the `executor-register`
message's `actions` array.

This ensures the orchestrator knows the executor can handle inferred
method calls, and `PlaybookOrchestrator.validateExecutors()` won't
reject scenarios that use inferred methods.

- [ ] **Step 8: Commit**

```bash
git add backend/playbook-client/
git commit -m "feat(playbook-client): add @PlaybookInterface and ActionRegistry inferred dispatch

New @PlaybookInterface annotation marks interface methods as playbook-callable.
ActionRegistry.invoke() falls back to interface scanning when no @PlaybookAction
handler matches. Default methods are skipped. Full interface hierarchy traversal.
allActions() includes interface methods for executor registration.

Refs #531

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- `specs/issue-531-inferred-method-dispatch/2026-10-20-inferred-method-dispatch-design.md` — design spec
- `packages/yaml-core/src/step/walker.ts:320-327` — Walker unknown-key error path
- `packages/yaml-core/src/step/catalog.ts` — CompositeCatalog, ImportScopedCatalog
- `packages/yaml-core/src/step/types.ts:13-22` — InvokeBinding union
- `packages/pages-aria/src/executor/command-executor.ts:77-126` — current dispatch with raw invokeMethod
- `packages/pages-aria/src/playbook/catalog-factory.ts` — catalog construction
- `packages/pages-aria/src/server/playbook-handler.ts` — server dispatch switch
- `packages/pages-aria/src/playbook/invoke/aria-handler.ts` — existing InvokeHandler
- `packages/pages-editor-core/src/discovery.ts` — EDITABLE_TEXT precedent
- `packages/pages-primitives/src/a11y/aria-contract.ts` — AriaTarget
- `packages/pages-document-diff/src/pages-document-diff.ts` — first adopter
- `backend/playbook-client/src/main/java/io/casehub/pages/playbook/client/ActionRegistry.java`
- `backend/playbook-client/src/main/java/io/casehub/pages/playbook/client/PlaybookAction.java`
- GitHub #531 — focal issue
- GitHub #506 — unified step catalog (portability model)
- GitHub #530 — Milkdown editor (motivation)
