# Unified Catalog Portability Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #506 — feat: unified step catalog — portability model and cross-runtime discovery
**Issue group:** #506

**Goal:** Add portability metadata to action definitions, enable client-side merge of Java+TS catalog sources, and validate portability before execution.

**Architecture:** Portability type and validation live in yaml-core/step (foundation). CatalogDataSource abstraction and REST endpoint live in pages-aria (UI+server layer). The `<pages-action-catalog>` component merges multiple sources client-side and renders portability badges.

**Tech Stack:** TypeScript, Lit, Vitest

## Global Constraints

- All types use clean names (no Step prefix) — already enforced on this branch
- Single `ParameterType` enum (STRING, INTEGER, NUMBER, BOOLEAN, ARRAY, OBJECT) — already unified
- ESM imports with explicit `.js` extensions
- Portability values: `'universal' | 'java' | 'ts' | 'both'` — must match platform#483 Java enum

---

## Batch 1: Portability Foundation (yaml-core)

### Task 1: Portability type and Definition field

**Files:**
- Modify: `packages/yaml-core/src/step/types.ts`
- Create: `packages/yaml-core/src/step/portability.ts`
- Create: `packages/yaml-core/src/step/portability.test.ts`
- Modify: `packages/yaml-core/src/step/index.ts`

**Interfaces:**
- Produces: `Portability` type (`'universal' | 'java' | 'ts' | 'both'`), `RuntimeEnvironment` type (`'java' | 'ts'`), `PortabilityViolation` interface, `validatePortability(actions: Definition[], runtime: RuntimeEnvironment): PortabilityViolation[]`, `inferPortability(invoke: InvokeBinding | undefined): Portability`, `isCompatible(portability: Portability, runtime: RuntimeEnvironment): boolean`

- [ ] **Step 1: Write failing tests for portability validation**

Create `packages/yaml-core/src/step/portability.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { validatePortability, inferPortability, isCompatible } from './portability.js';
import type { Definition } from './types.js';
import type { Portability, RuntimeEnvironment } from './portability.js';

describe('isCompatible', () => {
  it('universal is compatible with both runtimes', () => {
    expect(isCompatible('universal', 'java')).toBe(true);
    expect(isCompatible('universal', 'ts')).toBe(true);
  });

  it('both is compatible with both runtimes', () => {
    expect(isCompatible('both', 'java')).toBe(true);
    expect(isCompatible('both', 'ts')).toBe(true);
  });

  it('java is compatible with java only', () => {
    expect(isCompatible('java', 'java')).toBe(true);
    expect(isCompatible('java', 'ts')).toBe(false);
  });

  it('ts is compatible with ts only', () => {
    expect(isCompatible('ts', 'ts')).toBe(true);
    expect(isCompatible('ts', 'java')).toBe(false);
  });
});

describe('inferPortability', () => {
  it('rest binding infers universal', () => {
    expect(inferPortability({ kind: 'rest', method: 'GET', url: '/x', headers: {}, body: {} })).toBe('universal');
  });

  it('graphql binding infers universal', () => {
    expect(inferPortability({ kind: 'graphql', query: '{ x }' })).toBe('universal');
  });

  it('process binding infers universal', () => {
    expect(inferPortability({ kind: 'process', command: 'echo', args: [], output: 'text', env: {}, onError: 'stderr' })).toBe('universal');
  });

  it('mcp binding infers ts', () => {
    expect(inferPortability({ kind: 'mcp', tool: 'x' })).toBe('ts');
  });

  it('script binding infers ts', () => {
    expect(inferPortability({ kind: 'script', runtime: 'node', script: 'x.js', timeout: '30s', env: {} })).toBe('ts');
  });

  it('agent binding infers ts', () => {
    expect(inferPortability({ kind: 'agent', descriptor: 'x', structuredOutput: false })).toBe('ts');
  });

  it('undefined invoke infers ts', () => {
    expect(inferPortability(undefined)).toBe('ts');
  });
});

describe('validatePortability', () => {
  const def = (name: string, portability: Portability): Definition => ({
    name, inputs: {}, outputs: {}, portability,
  });

  it('returns empty for all-compatible actions', () => {
    expect(validatePortability([def('a', 'universal'), def('b', 'ts')], 'ts')).toEqual([]);
  });

  it('returns violations for incompatible actions', () => {
    const violations = validatePortability([def('a', 'java'), def('b', 'ts')], 'ts');
    expect(violations).toHaveLength(1);
    expect(violations[0]!.actionName).toBe('a');
    expect(violations[0]!.actionPortability).toBe('java');
    expect(violations[0]!.runtime).toBe('ts');
  });

  it('returns multiple violations', () => {
    const violations = validatePortability([def('a', 'java'), def('b', 'java')], 'ts');
    expect(violations).toHaveLength(2);
  });

  it('treats missing portability as ts', () => {
    const noPort: Definition = { name: 'x', inputs: {}, outputs: {} };
    const violations = validatePortability([noPort], 'java');
    expect(violations).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/step/portability.test.ts`
Expected: FAIL — module './portability.js' not found

- [ ] **Step 3: Implement portability module**

Create `packages/yaml-core/src/step/portability.ts`:

```typescript
import type { Definition, InvokeBinding } from './types.js';

export type Portability = 'universal' | 'java' | 'ts' | 'both';
export type RuntimeEnvironment = 'java' | 'ts';

export interface PortabilityViolation {
  actionName: string;
  actionPortability: Portability;
  runtime: RuntimeEnvironment;
  message: string;
}

const UNIVERSAL_BINDINGS = new Set(['rest', 'graphql', 'process']);

export function inferPortability(invoke: InvokeBinding | undefined): Portability {
  if (!invoke) return 'ts';
  return UNIVERSAL_BINDINGS.has(invoke.kind) ? 'universal' : 'ts';
}

export function isCompatible(portability: Portability, runtime: RuntimeEnvironment): boolean {
  if (portability === 'universal' || portability === 'both') return true;
  return portability === runtime;
}

export function validatePortability(
  actions: Definition[],
  runtime: RuntimeEnvironment,
): PortabilityViolation[] {
  const violations: PortabilityViolation[] = [];
  for (const action of actions) {
    const portability = action.portability ?? 'ts';
    if (!isCompatible(portability, runtime)) {
      violations.push({
        actionName: action.name,
        actionPortability: portability,
        runtime,
        message: `Action '${action.name}' requires '${portability}' runtime but current runtime is '${runtime}'`,
      });
    }
  }
  return violations;
}
```

- [ ] **Step 4: Add portability field to Definition interface**

In `packages/yaml-core/src/step/types.ts`, add to the `Definition` interface after `invoke?`:

```typescript
  portability?: import('./portability.js').Portability;
```

Alternatively, to avoid the dynamic import, import `Portability` at the top of `types.ts`:

```typescript
import type { Portability } from './portability.js';
```

And add to Definition:

```typescript
  portability?: Portability;
```

- [ ] **Step 5: Update index.ts re-exports**

In `packages/yaml-core/src/step/index.ts`, add:

```typescript
export type { Portability, RuntimeEnvironment, PortabilityViolation } from './portability.js';
export { inferPortability, isCompatible, validatePortability } from './portability.js';
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/step/portability.test.ts`
Expected: PASS — all tests green

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/src/step/portability.ts packages/yaml-core/src/step/portability.test.ts packages/yaml-core/src/step/types.ts packages/yaml-core/src/step/index.ts
git commit -m "feat(step): add Portability type, validation, and inference Refs #506"
```

### Task 2: DefinitionParser portability inference + PluginRegistry defaults

**Files:**
- Modify: `packages/yaml-core/src/step/definition-parser.ts`
- Modify: `packages/yaml-core/src/step/definition-parser.test.ts`
- Modify: `packages/yaml-core/src/step/plugin-registry.ts`
- Modify: `packages/yaml-core/src/step/plugin-registry.test.ts`

**Interfaces:**
- Consumes: `Portability` type, `inferPortability(invoke)` from Task 1
- Produces: `DefinitionParser.parseAction()` now returns `Definition` with `portability` set. `PluginRegistry.createSource()` sets `portability: 'ts'` on produced definitions.

- [ ] **Step 1: Write failing tests for parser portability**

Add to `packages/yaml-core/src/step/definition-parser.test.ts`:

```typescript
describe('portability', () => {
  it('explicit portability value is preserved', () => {
    const file = DefinitionParser.parse({
      actions: { a: { portability: 'java', inputs: {} } },
    });
    expect(file.actions['a']!.portability).toBe('java');
  });

  it('rest invoke infers universal', () => {
    const file = DefinitionParser.parse({
      actions: { a: { invoke: { rest: { url: '/x' } } } },
    });
    expect(file.actions['a']!.portability).toBe('universal');
  });

  it('mcp invoke infers ts', () => {
    const file = DefinitionParser.parse({
      actions: { a: { invoke: { mcp: 'tool' } } },
    });
    expect(file.actions['a']!.portability).toBe('ts');
  });

  it('no invoke infers ts', () => {
    const file = DefinitionParser.parse({
      actions: { a: { inputs: {} } },
    });
    expect(file.actions['a']!.portability).toBe('ts');
  });

  it('throws on invalid portability value', () => {
    expect(() => DefinitionParser.parse({
      actions: { a: { portability: 'invalid' } },
    })).toThrow(/Invalid portability/);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/step/definition-parser.test.ts`
Expected: FAIL — portability undefined or not validated

- [ ] **Step 3: Update DefinitionParser.parseAction()**

In `packages/yaml-core/src/step/definition-parser.ts`, add import:

```typescript
import { inferPortability } from './portability.js';
import type { Portability } from './portability.js';
```

Update `parseAction()` to read and validate portability:

```typescript
static parseAction(name: string, raw: Record<string, unknown>): Definition {
  const description = raw['description'] as string | undefined;
  const inputs = DefinitionParser.parseParams(raw['inputs'] as Record<string, unknown> | undefined);
  const outputs = DefinitionParser.parseParams(raw['outputs'] as Record<string, unknown> | undefined);
  const invokeRaw = raw['invoke'] as Record<string, unknown> | undefined;
  const invoke = invokeRaw ? DefinitionParser.parseInvoke(invokeRaw) : undefined;

  const VALID_PORTABILITY = new Set(['universal', 'java', 'ts', 'both']);
  const portabilityRaw = raw['portability'] as string | undefined;
  let portability: Portability;
  if (portabilityRaw !== undefined) {
    if (!VALID_PORTABILITY.has(portabilityRaw)) {
      throw new Error(`Invalid portability '${portabilityRaw}' for action '${name}'. Must be one of: universal, java, ts, both`);
    }
    portability = portabilityRaw as Portability;
  } else {
    portability = inferPortability(invoke);
  }

  return {
    name,
    ...(description !== undefined ? { description } : {}),
    inputs,
    outputs,
    ...(invoke !== undefined ? { invoke } : {}),
    portability,
  };
}
```

- [ ] **Step 4: Write failing test for PluginRegistry default portability**

Add to `packages/yaml-core/src/step/plugin-registry.test.ts`:

```typescript
it('createSource sets portability to ts on produced definitions', () => {
  const reg = new PluginRegistry();
  reg.register({ name: 'test', inputs: {}, outputs: {}, execute: async () => stepSuccess({}) });
  const entries = new Map<string, CatalogEntry>();
  reg.createSource().populate(entries);
  expect(entries.get('test')!.definition.portability).toBe('ts');
});
```

- [ ] **Step 5: Update PluginRegistry.createSource()**

In `packages/yaml-core/src/step/plugin-registry.ts`, update the `createSource` method to set `portability: 'ts'` on the produced `Definition`:

```typescript
const definition: Definition = {
  name: plugin.name,
  ...(plugin.description !== undefined ? { description: plugin.description } : {}),
  inputs: plugin.inputs,
  outputs: plugin.outputs,
  portability: 'ts',
};
```

- [ ] **Step 6: Run all tests**

Run: `npx vitest run packages/yaml-core/src/step/definition-parser.test.ts packages/yaml-core/src/step/plugin-registry.test.ts`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/src/step/definition-parser.ts packages/yaml-core/src/step/definition-parser.test.ts packages/yaml-core/src/step/plugin-registry.ts packages/yaml-core/src/step/plugin-registry.test.ts
git commit -m "feat(step): portability inference in parser, default in registry Refs #506"
```

## Batch 2: Catalog Data Sources + REST Endpoint

### Task 3: CatalogDataSource interface and RegistryCatalogSource

**Files:**
- Create: `packages/pages-aria/src/controller/catalog-data-source.ts`
- Create: `packages/pages-aria/src/controller/catalog-data-source.test.ts`
- Modify: `packages/pages-aria/src/controller/index.ts`

**Interfaces:**
- Consumes: `PluginRegistry` from yaml-core/step, `CatalogActionSummary`/`CatalogActionDetail`/`ParameterInfo` from step-catalog.ts
- Produces: `CatalogDataSource` interface (`fetchSummaries(): Promise<CatalogActionSummary[]>`, `fetchDetail(name: string): Promise<CatalogActionDetail | null>`, `priority: number`), `RegistryCatalogSource` class, `RestCatalogSource` class, `GraphqlCatalogSource` class

- [ ] **Step 1: Write failing tests for RegistryCatalogSource**

Create `packages/pages-aria/src/controller/catalog-data-source.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { RegistryCatalogSource } from './catalog-data-source.js';
import { PluginRegistry, stepSuccess } from '@casehubio/yaml-core/step';

describe('RegistryCatalogSource', () => {
  it('fetchSummaries returns registered actions', async () => {
    const reg = new PluginRegistry();
    reg.register({
      name: 'test-action',
      description: 'A test',
      inputs: { x: { type: 'STRING', required: true } },
      outputs: {},
      execute: async () => stepSuccess({}),
    });
    const source = new RegistryCatalogSource(reg);
    const summaries = await source.fetchSummaries();
    expect(summaries).toHaveLength(1);
    expect(summaries[0]!.name).toBe('test-action');
    expect(summaries[0]!.description).toBe('A test');
    expect(summaries[0]!.portability).toBe('ts');
    expect(summaries[0]!.inputCount).toBe(1);
    expect(summaries[0]!.source).toBe('plugin');
  });

  it('fetchDetail returns full parameter info', async () => {
    const reg = new PluginRegistry();
    reg.register({
      name: 'detailed',
      inputs: { id: { type: 'INTEGER', required: true, description: 'The ID' } },
      outputs: { result: { type: 'STRING', required: false } },
      execute: async () => stepSuccess({ result: 'ok' }),
    });
    const source = new RegistryCatalogSource(reg);
    const detail = await source.fetchDetail('detailed');
    expect(detail).not.toBeNull();
    expect(detail!.inputs['id']!.type).toBe('INTEGER');
    expect(detail!.inputs['id']!.required).toBe(true);
    expect(detail!.inputs['id']!.description).toBe('The ID');
    expect(detail!.outputs['result']).toBeDefined();
  });

  it('fetchDetail returns null for unknown action', async () => {
    const reg = new PluginRegistry();
    const source = new RegistryCatalogSource(reg);
    expect(await source.fetchDetail('nope')).toBeNull();
  });

  it('has default priority 0', () => {
    const source = new RegistryCatalogSource(new PluginRegistry());
    expect(source.priority).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-aria/src/controller/catalog-data-source.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement CatalogDataSource interface and RegistryCatalogSource**

Create `packages/pages-aria/src/controller/catalog-data-source.ts`:

```typescript
import type { PluginRegistry } from '@casehubio/yaml-core/step';
import type { CatalogActionSummary, CatalogActionDetail, ParameterInfo } from './step-catalog.js';
import type { Portability } from '@casehubio/yaml-core/step';

export interface CatalogDataSource {
  fetchSummaries(): Promise<CatalogActionSummary[]>;
  fetchDetail(name: string): Promise<CatalogActionDetail | null>;
  readonly priority: number;
}

export class RegistryCatalogSource implements CatalogDataSource {
  constructor(
    private readonly registry: PluginRegistry,
    readonly priority: number = 0,
  ) {}

  async fetchSummaries(): Promise<CatalogActionSummary[]> {
    const entries = new Map();
    this.registry.createSource().populate(entries);
    const summaries: CatalogActionSummary[] = [];
    for (const [, entry] of entries) {
      const def = entry.definition;
      summaries.push({
        name: def.name,
        description: def.description ?? '',
        invokeKind: def.invoke?.kind ?? null,
        source: 'plugin',
        portability: (def.portability ?? 'ts') as Portability,
        inputCount: Object.keys(def.inputs).length,
        outputCount: Object.keys(def.outputs).length,
      });
    }
    return summaries;
  }

  async fetchDetail(name: string): Promise<CatalogActionDetail | null> {
    const entries = new Map();
    this.registry.createSource().populate(entries);
    const entry = entries.get(name);
    if (!entry) return null;
    const def = entry.definition;
    return {
      name: def.name,
      description: def.description ?? '',
      invokeKind: def.invoke?.kind ?? null,
      source: 'plugin',
      portability: (def.portability ?? 'ts') as Portability,
      inputs: RegistryCatalogSource.mapParams(def.inputs),
      outputs: RegistryCatalogSource.mapParams(def.outputs),
      invoke: def.invoke ? { kind: def.invoke.kind, metadata: {} } : null,
    };
  }

  private static mapParams(params: Record<string, { type: string; required: boolean; defaultValue?: string; allowedValues?: string[]; format?: string; description?: string }>): Record<string, ParameterInfo> {
    const result: Record<string, ParameterInfo> = {};
    for (const [name, p] of Object.entries(params)) {
      result[name] = {
        type: p.type,
        required: p.required,
        defaultValue: p.defaultValue ?? null,
        allowedValues: p.allowedValues ?? null,
        format: p.format ?? null,
        description: p.description ?? null,
      };
    }
    return result;
  }
}

export class RestCatalogSource implements CatalogDataSource {
  constructor(
    private readonly baseUrl: string,
    readonly priority: number = 10,
  ) {}

  async fetchSummaries(): Promise<CatalogActionSummary[]> {
    const res = await fetch(`${this.baseUrl}/api/catalog/actions`);
    if (!res.ok) return [];
    return res.json();
  }

  async fetchDetail(name: string): Promise<CatalogActionDetail | null> {
    const res = await fetch(`${this.baseUrl}/api/catalog/actions/${encodeURIComponent(name)}`);
    if (!res.ok) return null;
    return res.json();
  }
}

export class GraphqlCatalogSource implements CatalogDataSource {
  constructor(
    private readonly endpoint: string,
    readonly priority: number = 20,
  ) {}

  async fetchSummaries(): Promise<CatalogActionSummary[]> {
    const query = `{ stepCatalog { actions { name description invokeKind source portability inputCount outputCount } } }`;
    const res = await fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query }),
    });
    if (!res.ok) return [];
    const data = await res.json();
    return data?.data?.stepCatalog?.actions ?? [];
  }

  async fetchDetail(name: string): Promise<CatalogActionDetail | null> {
    const query = `{ stepCatalog { action(name: "${name}") { name description invokeKind source portability inputs { name type required defaultValue allowedValues format description } outputs { name type required defaultValue allowedValues format description } invoke { kind metadata } } } }`;
    const res = await fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query }),
    });
    if (!res.ok) return null;
    const data = await res.json();
    return data?.data?.stepCatalog?.action ?? null;
  }
}
```

- [ ] **Step 4: Update CatalogActionSummary and CatalogActionDetail interfaces**

In `packages/pages-aria/src/controller/step-catalog.ts`, add `portability` to both interfaces:

```typescript
// CatalogActionSummary — add after source field:
portability: string;

// CatalogActionDetail — add after source field:
portability: string;
```

- [ ] **Step 5: Update controller/index.ts re-exports**

Add to `packages/pages-aria/src/controller/index.ts`:

```typescript
export type { CatalogDataSource } from './catalog-data-source.js';
export { RegistryCatalogSource, RestCatalogSource, GraphqlCatalogSource } from './catalog-data-source.js';
```

- [ ] **Step 6: Run tests**

Run: `npx vitest run packages/pages-aria/src/controller/catalog-data-source.test.ts`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-aria/src/controller/catalog-data-source.ts packages/pages-aria/src/controller/catalog-data-source.test.ts packages/pages-aria/src/controller/step-catalog.ts packages/pages-aria/src/controller/index.ts
git commit -m "feat(catalog): CatalogDataSource interface with Registry/REST/GraphQL sources Refs #506"
```

### Task 4: REST catalog list handler

**Files:**
- Create: `packages/pages-aria/src/server/catalog-list-handler.ts`
- Create: `packages/pages-aria/src/server/catalog-list-handler.test.ts`

**Interfaces:**
- Consumes: `Catalog`, `CatalogEntry` from yaml-core/step
- Produces: `createCatalogListHandler(catalog: Catalog): { list: () => CatalogActionSummary[], detail: (name: string) => CatalogActionDetail | null }`

- [ ] **Step 1: Write failing tests**

Create `packages/pages-aria/src/server/catalog-list-handler.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { createCatalogListHandler } from './catalog-list-handler.js';
import type { Catalog, CatalogEntry, Action } from '@casehubio/yaml-core/step';
import { stepSuccess } from '@casehubio/yaml-core/step';

function mockCatalog(entries: Record<string, CatalogEntry>): Catalog {
  return {
    resolve: (name) => entries[name],
    availableActions: () => new Set(Object.keys(entries)),
  };
}

function entry(name: string, portability = 'ts' as string): CatalogEntry {
  const action: Action = { execute: async () => stepSuccess({}) };
  return {
    qualifiedName: name,
    definition: {
      name,
      description: `desc-${name}`,
      inputs: { x: { type: 'STRING', required: true } },
      outputs: {},
      portability: portability as 'ts',
    },
    action,
  };
}

describe('createCatalogListHandler', () => {
  it('list returns all actions as summaries', () => {
    const handler = createCatalogListHandler(mockCatalog({ a: entry('a'), b: entry('b', 'universal') }));
    const summaries = handler.list();
    expect(summaries).toHaveLength(2);
    expect(summaries.find(s => s.name === 'a')!.portability).toBe('ts');
    expect(summaries.find(s => s.name === 'b')!.portability).toBe('universal');
  });

  it('detail returns full info for known action', () => {
    const handler = createCatalogListHandler(mockCatalog({ a: entry('a') }));
    const detail = handler.detail('a');
    expect(detail).not.toBeNull();
    expect(detail!.name).toBe('a');
    expect(detail!.inputs['x']!.type).toBe('STRING');
  });

  it('detail returns null for unknown action', () => {
    const handler = createCatalogListHandler(mockCatalog({}));
    expect(handler.detail('nope')).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-aria/src/server/catalog-list-handler.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement catalog list handler**

Create `packages/pages-aria/src/server/catalog-list-handler.ts`:

```typescript
import type { Catalog } from '@casehubio/yaml-core/step';
import type { CatalogActionSummary, CatalogActionDetail, ParameterInfo } from '../controller/step-catalog.js';

export interface CatalogListHandler {
  list(): CatalogActionSummary[];
  detail(name: string): CatalogActionDetail | null;
}

export function createCatalogListHandler(catalog: Catalog): CatalogListHandler {
  function mapParams(params: Record<string, { type: string; required: boolean; defaultValue?: string; allowedValues?: string[]; format?: string; description?: string }>): Record<string, ParameterInfo> {
    const result: Record<string, ParameterInfo> = {};
    for (const [name, p] of Object.entries(params)) {
      result[name] = {
        type: p.type,
        required: p.required,
        defaultValue: p.defaultValue ?? null,
        allowedValues: p.allowedValues ?? null,
        format: p.format ?? null,
        description: p.description ?? null,
      };
    }
    return result;
  }

  return {
    list(): CatalogActionSummary[] {
      const summaries: CatalogActionSummary[] = [];
      for (const name of catalog.availableActions()) {
        const entry = catalog.resolve(name);
        if (!entry) continue;
        const def = entry.definition;
        summaries.push({
          name: def.name,
          description: def.description ?? '',
          invokeKind: def.invoke?.kind ?? null,
          source: 'registry',
          portability: (def.portability ?? 'ts') as string,
          inputCount: Object.keys(def.inputs).length,
          outputCount: Object.keys(def.outputs).length,
        });
      }
      return summaries;
    },

    detail(name: string): CatalogActionDetail | null {
      const entry = catalog.resolve(name);
      if (!entry) return null;
      const def = entry.definition;
      return {
        name: def.name,
        description: def.description ?? '',
        invokeKind: def.invoke?.kind ?? null,
        source: 'registry',
        portability: (def.portability ?? 'ts') as string,
        inputs: mapParams(def.inputs),
        outputs: mapParams(def.outputs),
        invoke: def.invoke ? { kind: def.invoke.kind, metadata: {} } : null,
      };
    },
  };
}
```

- [ ] **Step 4: Run tests**

Run: `npx vitest run packages/pages-aria/src/server/catalog-list-handler.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-aria/src/server/catalog-list-handler.ts packages/pages-aria/src/server/catalog-list-handler.test.ts
git commit -m "feat(server): catalog list handler — REST endpoint for registry contents Refs #506"
```

## Batch 3: Component Evolution — Client-Side Merge + Badges

### Task 5: PagesActionCatalog — multi-source merge + portability badges

**Files:**
- Modify: `packages/pages-aria/src/controller/step-catalog.ts`
- Modify: `packages/pages-aria/src/controller/step-catalog.test.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`

**Interfaces:**
- Consumes: `CatalogDataSource` from Task 3, `Portability` from Task 1
- Produces: Updated `<pages-action-catalog>` component with `sources` property accepting `CatalogDataSource[]`, portability filter chips, portability badge rendering

- [ ] **Step 1: Add portability badge CSS to PagesActionCatalog**

In `packages/pages-aria/src/controller/step-catalog.ts`, add to the `static styles`:

```css
.portability-badge {
  font-size: 9px; padding: 1px 5px;
  border-radius: var(--pages-radius-sm, 4px);
  font-weight: 500; text-transform: uppercase;
}
.portability-universal { background: var(--pages-success-3, #dcfce7); color: var(--pages-success-11, #166534); }
.portability-java { background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-11, #92400e); }
.portability-ts { background: var(--pages-accent-3, #e8eaf6); color: var(--pages-accent-11, #283593); }
.portability-both { background: var(--pages-info-3, #e0e7ff); color: var(--pages-info-11, #3730a3); }
```

- [ ] **Step 2: Add `sources` property and multi-source fetch logic**

Add a `sources` property to PagesActionCatalog:

```typescript
@property({ attribute: false })
sources: CatalogDataSource[] = [];
```

Replace the existing single-fetch `connectedCallback` with multi-source merge:

```typescript
override connectedCallback() {
  super.connectedCallback();
  this._loadFromSources();
}

private async _loadFromSources(): Promise<void> {
  if (this.sources.length === 0) return;
  const sorted = [...this.sources].sort((a, b) => a.priority - b.priority);
  const results = await Promise.all(sorted.map(s => s.fetchSummaries().catch(() => [])));
  const seen = new Set<string>();
  const merged: (CatalogActionSummary & { _sourceIndex: number })[] = [];
  for (let i = 0; i < results.length; i++) {
    for (const summary of results[i]!) {
      if (!seen.has(summary.name)) {
        seen.add(summary.name);
        merged.push({ ...summary, _sourceIndex: i });
      }
    }
  }
  this._actions = merged;
  this._sourceMap = new Map(merged.map(a => [a.name, sorted[a._sourceIndex]!]));
  this.requestUpdate();
}
```

- [ ] **Step 3: Add portability badge to action row rendering**

In the `_renderActionRow` method, add the badge after the action name:

```typescript
private _renderPortabilityBadge(portability: string): TemplateResult {
  const cls = `portability-badge portability-${portability}`;
  return html`<span class="${cls}">${portability}</span>`;
}
```

Call it in the action row template alongside the existing source chip.

- [ ] **Step 4: Add portability filter chips**

Add a portability filter state and chips alongside existing source filters. Follow the same pattern as the existing `_sourceFilters`:

```typescript
@state() private _portabilityFilters = new Set<string>();
```

Render filter chips for each portability value found in the current action list. Clicking a chip toggles it. Filtered actions must match both source and portability filters (AND logic).

- [ ] **Step 5: Update detail fetch to use source map**

When user clicks an action, fetch detail from the source that provided the summary:

```typescript
private async _loadDetail(name: string): Promise<void> {
  const source = this._sourceMap.get(name);
  if (!source) return;
  this._selectedDetail = await source.fetchDetail(name);
  this.requestUpdate();
}
```

- [ ] **Step 6: Update scenario-controller.ts to wire sources**

In `packages/pages-aria/src/controller/scenario-controller.ts`, when creating the catalog view, pass `sources` instead of direct data:

```typescript
import { RegistryCatalogSource } from './catalog-data-source.js';

// When wiring catalog view:
catalogElement.sources = [new RegistryCatalogSource(this._pluginRegistry)];
```

- [ ] **Step 7: Write tests for multi-source merge**

Add to `packages/pages-aria/src/controller/step-catalog.test.ts` (or catalog-data-source.test.ts if DOM tests aren't feasible):

Test that when two sources return overlapping action names, the lower-priority source wins (first source). Test portability badge rendering if DOM testing is available.

- [ ] **Step 8: Run full test suite**

Run: `npx vitest run packages/yaml-core/src/step/ packages/pages-aria/src/server/ packages/pages-aria/src/controller/catalog-data-source.test.ts`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add packages/pages-aria/src/controller/step-catalog.ts packages/pages-aria/src/controller/step-catalog.test.ts packages/pages-aria/src/controller/scenario-controller.ts
git commit -m "feat(catalog): multi-source merge and portability badges Refs #506"
```

## Batch 4: Pre-flight Validation Integration

### Task 6: Catalog browser pre-flight portability check

**Files:**
- Modify: `packages/pages-aria/src/controller/step-catalog.ts`
- Modify: `packages/pages-aria/src/server/catalog-execute-handler.ts`
- Modify: `packages/pages-aria/src/server/catalog-execute-handler.test.ts`

**Interfaces:**
- Consumes: `validatePortability()`, `PortabilityViolation` from Task 1, `CatalogDataSource` from Task 3
- Produces: Updated "Try it" panel with portability pre-check, updated execute handler with validation gate

- [ ] **Step 1: Write failing tests for execute handler portability validation**

Add to `packages/pages-aria/src/server/catalog-execute-handler.test.ts`:

```typescript
it('rejects execution of incompatible action', async () => {
  const javaDef = {
    name: 'java-only',
    inputs: {},
    outputs: {},
    portability: 'java' as const,
  };
  const action = { execute: async () => stepSuccess({}) };
  const catalog: Catalog = {
    resolve: (name) => name === 'java-only' ? { qualifiedName: 'java-only', definition: javaDef, action } : undefined,
    availableActions: () => new Set(['java-only']),
  };
  const handler = createCatalogExecuteHandler(catalog, 'ts');
  const result = await handler({ actionName: 'java-only', params: {} });
  expect(result.kind).toBe('failure');
  expect((result as { kind: 'failure'; message: string }).message).toContain('portability');
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/pages-aria/src/server/catalog-execute-handler.test.ts`
Expected: FAIL — no portability check in handler yet

- [ ] **Step 3: Update execute handler with portability validation**

In `packages/pages-aria/src/server/catalog-execute-handler.ts`, add an optional `runtime` parameter and portability check:

```typescript
import type { Catalog, Result } from '@casehubio/yaml-core/step';
import { stepFailure, MapServiceRegistry } from '@casehubio/yaml-core/step';
import { isCompatible } from '@casehubio/yaml-core/step';
import type { RuntimeEnvironment } from '@casehubio/yaml-core/step';

export function createCatalogExecuteHandler(
  catalog: Catalog,
  runtime: RuntimeEnvironment = 'ts',
): (req: CatalogExecuteRequest) => Promise<Result> {
  return async (req: CatalogExecuteRequest): Promise<Result> => {
    const entry = catalog.resolve(req.actionName);
    if (!entry) {
      return stepFailure(`Action '${req.actionName}' not found in catalog`);
    }

    const portability = entry.definition.portability ?? 'ts';
    if (!isCompatible(portability, runtime)) {
      return stepFailure(
        `Action '${req.actionName}' requires '${portability}' runtime but current runtime is '${runtime}'`,
      );
    }

    const services = new MapServiceRegistry();
    try {
      return await entry.action.execute(req.params, services);
    } catch (err) {
      return stepFailure(
        `Execution failed: ${err instanceof Error ? err.message : String(err)}`,
      );
    }
  };
}
```

- [ ] **Step 4: Add portability check to Try It panel**

In the `<pages-action-catalog>` component's "Try it" execution method, check portability before executing. If incompatible, show the violation message inline instead of running.

- [ ] **Step 5: Run all tests**

Run: `npx vitest run packages/yaml-core/src/step/ packages/pages-aria/src/server/ packages/pages-aria/src/controller/catalog-data-source.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/src/controller/step-catalog.ts packages/pages-aria/src/server/catalog-execute-handler.ts packages/pages-aria/src/server/catalog-execute-handler.test.ts
git commit -m "feat(catalog): pre-flight portability validation before execution Refs #506"
```

## References

- [2026-09-29-unified-catalog-portability-design.md] — design spec this plan implements
- `packages/yaml-core/src/step/types.ts` — Definition interface (portability field added)
- `packages/yaml-core/src/step/definition-parser.ts` — YAML parsing with portability inference
- `packages/yaml-core/src/step/plugin-registry.ts` — PluginRegistry.register() with defaults
- `packages/yaml-core/src/step/catalog.ts` — CompositeCatalog merge semantics
- `packages/yaml-core/src/step/index.ts` — barrel re-exports
- `packages/pages-aria/src/controller/step-catalog.ts` — PagesActionCatalog component
- `packages/pages-aria/src/server/catalog-execute-handler.ts` — execution handler
- `packages/pages-aria/src/controller/scenario-controller.ts` — controller wiring
- casehubio/platform#483 — Java API contract
- casehubio/casehub-pages#506 — focal issue
