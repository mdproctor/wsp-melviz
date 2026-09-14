# yaml-core Integration and Interactive Tutorial — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #435 — Interactive YAML tutorial
**Issue group:** #435

**Goal:** Port yaml-core to TypeScript, integrate into pages pipeline/LSP/builder, build 15-step interactive tutorial teaching the unified CaseHub YAML composition language.

**Architecture:** yaml-core is a format-agnostic preprocessing layer (variables, forEach, modules, conditionals) ported from Java to TypeScript as `packages/yaml-core/`. Zod schemas generated from existing JSON Schema fragments compose with format schemas via `z.intersection()`. The builder workbench provides the tutorial's editing surface.

**Tech Stack:** TypeScript, Zod, vitest, json-schema-to-zod, Lit, CodeMirror 6

## Global Constraints

- Zero external dependencies in yaml-core expansion logic (mirrors Java's zero-dep constraint)
- Zod is the only dependency (for generated schemas)
- All expansion logic must support lenient mode (unresolved refs → literal strings + diagnostics)
- Behavioral parity with Java yaml-core verified via conformance tests
- JSON Schema 2020-12 fragments are the schema source of truth
- `z.intersection()` for schema composition (not `.merge()` — format schemas are widened `ZodType`)
- `when:` conditions resolve to boolean strings only (true/false/yes/no/on/off/1/0)
- `pages-aria` does NOT depend on `pages-builder` — cross-package via untyped custom elements

---

## Batch 1: yaml-core TypeScript package

### Task 1: Types, Truthiness, and VariableResolver

**Files:**
- Create: `packages/yaml-core/package.json`
- Create: `packages/yaml-core/tsconfig.json`
- Create: `packages/yaml-core/vitest.config.ts`
- Create: `packages/yaml-core/src/types.ts`
- Create: `packages/yaml-core/src/truthiness.ts`
- Create: `packages/yaml-core/src/variable-resolver.ts`
- Create: `packages/yaml-core/src/index.ts`
- Test: `packages/yaml-core/src/truthiness.test.ts`
- Test: `packages/yaml-core/src/variable-resolver.test.ts`

**Interfaces:**
- Consumes: nothing (first task)
- Produces:
  - `VariableSource`: `(key: string) => string | undefined`
  - `VariableResolver`: class with `resolve(value: unknown): unknown`, `resolveString(template: string, context: string): string`, `withScope(prefix: string, source: VariableSource): VariableResolver`, `withChainedScope(prefix: string, source: VariableSource): VariableResolver`
  - `isTruthy(value: string): boolean` — throws on non-boolean strings
  - `DeferredPrefixHandler`: `(match: string, prefix: string, key: string) => string`
  - `ForEachDirective`: `{ type: 'inline'; as: string; in: string[] } | { type: 'group-ref'; groupName: string }`
  - `YamlModule`, `YamlModuleParameter`, `YamlModuleOutput`, `YamlImport`, `ParameterType`, `IterationGroup`
  - `ExpansionDiagnostic`: `{ severity, message, path, category }`
  - `ExpandResult`: `{ map, diagnostics }`

**Port reference:** Read the Java source at `platform/yaml-core/src/main/java/io/casehub/yaml/core/` for each module. The TypeScript should produce identical behavior — conformance tests verify this.

- [ ] **Step 1: Scaffold package**

Create `packages/yaml-core/package.json`:
```json
{
  "name": "@casehubio/yaml-core",
  "version": "0.2.0",
  "type": "module",
  "main": "src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./expand": "./src/expand.ts",
    "./schema": "./src/schema.ts"
  },
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "devDependencies": {
    "vitest": "workspace:*"
  }
}
```

Create `packages/yaml-core/vitest.config.ts` following the pattern in `packages/pages-document/vitest.config.ts`.

Create `packages/yaml-core/tsconfig.json` extending `packages/pages-tsconfig/tsconfig.json`.

Run: `yarn install` from repo root to wire the new workspace package.

- [ ] **Step 2: Write types.ts**

Read ALL Java types in `platform/yaml-core/src/main/java/io/casehub/yaml/core/` and create TypeScript equivalents. Key types:

```typescript
// packages/yaml-core/src/types.ts

export type ParameterType = 'STRING' | 'LIST' | 'INTEGER' | 'NUMBER' | 'BOOLEAN';

export interface YamlModuleParameter {
  type: ParameterType;
  required: boolean;
  defaultValue?: string;
  minLength?: number;
  maxLength?: number;
  pattern?: string;
  minimum?: number;
  maximum?: number;
  allowedValues?: string[];
  constraintDescription?: string;
}

export interface YamlModuleOutput {
  type: ParameterType;
  value: string;
}

export interface YamlModule {
  name: string;
  parameters: Record<string, YamlModuleParameter>;
  outputs: Record<string, YamlModuleOutput>;
  sections: Record<string, Record<string, unknown>>;
}

export interface YamlImport {
  module: string;
  as: string;
  when?: string;
  parameters: Record<string, string>;
}

export interface IterationGroup {
  in: string[];
}

export type ForEachDirective =
  | { type: 'inline'; as: string; in: string[] }
  | { type: 'group-ref'; groupName: string };

export type VariableSource = (key: string) => string | undefined;

export type DeferredPrefixHandler = (match: string, prefix: string, key: string) => string;

export interface ExpansionDiagnostic {
  severity: 'error' | 'warning';
  message: string;
  path: string[];
  category: 'unresolved-variable' | 'invalid-parameter' | 'circular-module'
           | 'unknown-prefix' | 'expansion-error';
}

export interface ExpandResult {
  map: Record<string, unknown>;
  diagnostics: ExpansionDiagnostic[];
}
```

- [ ] **Step 3: Write failing Truthiness tests**

Read `platform/yaml-core/src/main/java/io/casehub/yaml/core/condition/Truthiness.java` and its test. Write conformance tests:

```typescript
// packages/yaml-core/src/truthiness.test.ts
import { describe, it, expect } from 'vitest';
import { isTruthy } from './truthiness.js';

describe('Truthiness', () => {
  it.each(['true', 'yes', 'on', 'y', '1'])('treats "%s" as truthy', (v) => {
    expect(isTruthy(v)).toBe(true);
  });

  it.each(['TRUE', 'Yes', 'ON', 'Y'])('is case-insensitive for truthy', (v) => {
    expect(isTruthy(v)).toBe(true);
  });

  it.each(['false', 'no', 'off', 'n', '0'])('treats "%s" as falsy', (v) => {
    expect(isTruthy(v)).toBe(false);
  });

  it.each(['FALSE', 'No', 'OFF', 'N'])('is case-insensitive for falsy', (v) => {
    expect(isTruthy(v)).toBe(false);
  });

  it('throws on non-boolean string', () => {
    expect(() => isTruthy('maybe')).toThrow('not a boolean value');
  });

  it('throws on comparison expression', () => {
    expect(() => isTruthy('us != ap')).toThrow('not a boolean value');
  });
});
```

Run: `yarn workspace @casehubio/yaml-core test`
Expected: FAIL — `isTruthy` not defined

- [ ] **Step 4: Implement Truthiness**

```typescript
// packages/yaml-core/src/truthiness.ts
const TRUTHY = new Set(['true', 'yes', 'on', 'y', '1']);
const FALSY = new Set(['false', 'no', 'off', 'n', '0']);

export function isTruthy(value: string): boolean {
  const lower = value.toLowerCase();
  if (TRUTHY.has(lower)) return true;
  if (FALSY.has(lower)) return false;
  throw new Error(
    `Condition resolved to '${value}' which is not a boolean value. ` +
    `Expected: true/false/yes/no/on/off/y/n/1/0`
  );
}
```

Run: `yarn workspace @casehubio/yaml-core test`
Expected: PASS

- [ ] **Step 5: Write failing VariableResolver tests**

Read `platform/yaml-core/src/main/java/io/casehub/yaml/core/resolver/VariableResolver.java` and its test class. Write conformance tests covering:

```typescript
// packages/yaml-core/src/variable-resolver.test.ts
import { describe, it, expect } from 'vitest';
import { VariableResolver } from './variable-resolver.js';
import type { VariableSource } from './types.js';

describe('VariableResolver', () => {
  const source: VariableSource = (key) => ({ region: 'us', env: 'prod' }[key]);

  it('resolves simple variable', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    expect(resolver.resolveString('${var.region}', 'test')).toBe('us');
  });

  it('resolves multiple variables in one string', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    expect(resolver.resolveString('${var.region}-${var.env}', 'test')).toBe('us-prod');
  });

  it('throws on unresolved variable', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    expect(() => resolver.resolveString('${var.missing}', 'test')).toThrow();
  });

  it('preserves deferred prefixes', () => {
    const resolver = new VariableResolver({ var: source }, new Set(['each']));
    expect(resolver.resolveString('${each.item}', 'test')).toBe('${each.item}');
  });

  it('resolves default values', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    expect(resolver.resolveString('${var.missing:-fallback}', 'test')).toBe('fallback');
  });

  it('resolves maps recursively', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    const input = { name: '${var.region}', nested: { value: '${var.env}' } };
    const result = resolver.resolve(input);
    expect(result).toEqual({ name: 'us', nested: { value: 'prod' } });
  });

  it('resolves arrays recursively', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    const input = ['${var.region}', '${var.env}'];
    const result = resolver.resolve(input);
    expect(result).toEqual(['us', 'prod']);
  });

  it('chains scopes with withScope', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    const extended = resolver.withScope('params', (k) => ({ name: 'test' }[k]));
    expect(extended.resolveString('${params.name}', 'test')).toBe('test');
    expect(extended.resolveString('${var.region}', 'test')).toBe('us');
  });

  it('chains scope with withChainedScope', () => {
    const resolver = new VariableResolver({ var: source }, new Set());
    const override: VariableSource = (k) => k === 'region' ? 'eu' : undefined;
    const extended = resolver.withChainedScope('var', override);
    expect(extended.resolveString('${var.region}', 'test')).toBe('eu');
    expect(extended.resolveString('${var.env}', 'test')).toBe('prod');
  });
});
```

Run: `yarn workspace @casehubio/yaml-core test`
Expected: FAIL — `VariableResolver` not defined

- [ ] **Step 6: Implement VariableResolver**

Port `VariableResolver.java` to TypeScript. Key patterns:
- `VAR_PATTERN = /\$\{([^}]+)}/g` — same regex
- `resolve()` dispatches on type: string → resolveString, Map → resolveMap, Array → resolveList, other → passthrough
- `resolveString()` uses `String.replace()` with callback — parse `prefix.key` from match group, look up in `prefixSources`
- Deferred prefixes: if prefix is in `deferredPrefixes` set, return the original match unchanged
- Default values: parse `:-` separator from key, use fallback if source returns undefined
- Immutable pattern: `withScope()` and `withChainedScope()` return new instances

Read the Java source carefully for edge cases: whole-string substitution (when the entire string is one `${...}` and the value resolves to a non-string), deferred prefix handler callback, `forParams()` factory method.

Run: `yarn workspace @casehubio/yaml-core test`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/
git commit -m "feat(#435): yaml-core package — types, Truthiness, VariableResolver with conformance tests Refs #435"
```

### Task 2: ForEachExpander and ModuleExpander

**Files:**
- Create: `packages/yaml-core/src/foreach-expander.ts`
- Create: `packages/yaml-core/src/module-expander.ts`
- Create: `packages/yaml-core/src/parameter-validator.ts`
- Create: `packages/yaml-core/src/csv-parser.ts`
- Test: `packages/yaml-core/src/foreach-expander.test.ts`
- Test: `packages/yaml-core/src/module-expander.test.ts`
- Test: `packages/yaml-core/src/parameter-validator.test.ts`
- Test: `packages/yaml-core/src/csv-parser.test.ts`

**Interfaces:**
- Consumes: `VariableResolver`, `VariableSource`, `ForEachDirective`, `IterationGroup`, `YamlModule`, `YamlImport`, `YamlModuleParameter`, `ParameterType`, `isTruthy` (from Task 1)
- Produces:
  - `ForEachExpander.expand<E>(elements, iterationGroups, resolver, adapter, maxExpansion): ExpansionResult<E>`
  - `ForEachAdapter<E>`: `{ getForEach(element: E): ForEachDirective | null; cloneWithSuffix(element: E, suffix: string): E; resolveVariables(element: E, resolver: VariableResolver): E }`
  - `ModuleExpander.expand(imports, availableModules, existingSections, options?): ExpandedModule`
  - `ParameterValidator.validate(declared, provided): ParameterViolation[]`
  - `CsvParser.parse(csv: string, columns: CsvColumn[]): Record<string, string>[]`

- [ ] **Step 1: Write failing ForEachExpander tests**

Read `platform/yaml-core/src/main/java/io/casehub/yaml/core/foreach/ForEachExpander.java` and its test class. Write conformance tests covering:
- Inline forEach: `{as: 'item', in: ['a', 'b', 'c']}` → 3 copies with stamped IDs
- Group reference forEach: references named iteration group
- Variable resolution in iteration values
- `when:` on a non-forEach element (evaluated with base resolver — element removed if falsy)
- `when:` inside a forEach template (evaluated per-iteration with `${each.*}` in scope)
- `when:` excluding some but not all iterations (e.g., 3 items, 1 excluded → 2 copies)
- Cross-references from non-excluded to excluded elements (should produce error/diagnostic)
- Max expansion limit
- Empty iteration list

Run: `yarn workspace @casehubio/yaml-core test`
Expected: FAIL

- [ ] **Step 2: Implement ForEachExpander**

Port `ForEachExpander.java` to TypeScript. Key patterns:
- Generic `<E>` type with `ForEachAdapter<E>` interface for format-agnostic element handling
- Inline vs group-ref directive dispatch via `ForEachDirective` union type
- ID stamping: `originalId.value` format
- Per-iteration variable scoping via `VariableResolver.withScope('each', ...)`
- `when:` evaluation per element via `isTruthy()` — collect `excludedIds`
- Return `ExpansionResult<E>` with expanded elements map and excluded IDs

Run: `yarn workspace @casehubio/yaml-core test`
Expected: PASS for ForEachExpander tests

- [ ] **Step 3: Write failing ParameterValidator tests**

Read `platform/yaml-core/src/main/java/io/casehub/yaml/core/module/ParameterValidator.java`. Test:
- Required parameter missing → violation
- Type validation: STRING, INTEGER, NUMBER, BOOLEAN, LIST
- Constraint validation: minLength, maxLength, pattern, minimum, maximum, allowedValues
- Default values satisfy required constraints

- [ ] **Step 4: Implement ParameterValidator**

Port the validator. Key patterns:
- LIST type: split on comma, validate each element
- Pattern: `new RegExp(pattern).test(value)`
- Type coercion: INTEGER → `parseInt`, NUMBER → `parseFloat`, BOOLEAN → truthiness check

Run: `yarn workspace @casehubio/yaml-core test`
Expected: PASS

- [ ] **Step 5: Write failing ModuleExpander tests**

Read `platform/yaml-core/src/main/java/io/casehub/yaml/core/module/ModuleExpander.java` and test class. Test:
- Single module import: parameter resolution, section merging
- Multiple imports: sequential, later imports reference earlier outputs via `${module.alias.output}`
- Import-level `when:` conditions stored as metadata
- Module extension (`extends` field)
- Invalid import: unknown module → error
- Invalid parameters: wrong type, missing required → error

- [ ] **Step 6: Implement ModuleExpander**

Port `ModuleExpander.java`. Most complex module. Key patterns:
- Validate imports against available modules
- For each import: resolve parameters via `ParameterValidator`, create `VariableResolver.forParams()`
- Resolve module section content with scoped resolver
- Merge sections into target document: prefix keys with import alias (`alias.key`)
- Resolve outputs: template values with parameter scope
- Module references in parameters: `${module.alias.output}` resolved from earlier imports

- [ ] **Step 7: Write failing CsvParser tests and implement**

Read `platform/yaml-core/src/main/java/io/casehub/yaml/core/data/CsvParser.java`. Small module — test and implement together:
- Parse CSV with typed columns (STRING, INTEGER, NUMBER, BOOLEAN)
- Header row handling
- Quoted fields

- [ ] **Step 8: Commit**

```bash
git add packages/yaml-core/
git commit -m "feat(#435): ForEachExpander, ModuleExpander, ParameterValidator, CsvParser Refs #435"
```

### Task 3: expand() public API and Zod schema generation

**Files:**
- Create: `packages/yaml-core/src/expand.ts`
- Create: `packages/yaml-core/scripts/generate-schemas.mjs`
- Create: `packages/yaml-core/src/schema.generated.ts` (generated)
- Create: `packages/yaml-core/src/schema.ts`
- Modify: `packages/yaml-core/package.json` (add zod dep, generate script)
- Modify: `packages/yaml-core/src/index.ts` (export all)
- Test: `packages/yaml-core/src/expand.test.ts`
- Test: `packages/yaml-core/src/schema.test.ts`

**Interfaces:**
- Consumes: All from Tasks 1-2
- Produces:
  - `expand(map: Record<string, unknown>, options?: { strict?: boolean }): ExpandResult`
  - `yamlCoreDocumentSchema` — Zod schema for document-level yaml-core keys
  - `yamlCoreElementMixin` — Zod mixin for element-level keys (forEach, when)
  - `moduleDefinitionSchema`, `importSchema`, `forEachSchema`, `parameterSchema`, `outputSchema`

- [ ] **Step 1: Write failing expand() integration tests**

```typescript
// packages/yaml-core/src/expand.test.ts
import { describe, it, expect } from 'vitest';
import { expand } from './expand.js';

describe('expand', () => {
  it('passes through documents without yaml-core constructs', () => {
    const input = { pages: [{ name: 'test' }] };
    const result = expand(input);
    expect(result.map).toEqual(input);
    expect(result.diagnostics).toEqual([]);
  });

  it('resolves variables', () => {
    const input = {
      variables: { theme: { color: 'blue' } },
      pages: [{ name: '${theme.color}-dashboard' }],
    };
    const result = expand(input);
    expect(result.map.pages).toEqual([{ name: 'blue-dashboard' }]);
    expect(result.diagnostics).toEqual([]);
  });

  it('expands forEach', () => {
    const input = {
      pages: [{
        name: 'dashboard',
        components: {
          '${item}-metric': {
            forEach: { as: 'item', in: ['cpu', 'mem'] },
            type: 'metric',
            properties: { field: '${each.item}' },
          },
        },
      }],
    };
    const result = expand(input);
    const comps = (result.map.pages as any[])[0].components;
    expect(Object.keys(comps)).toContain('cpu-metric.cpu');
    expect(Object.keys(comps)).toContain('mem-metric.mem');
  });

  it('expands modules', () => {
    const input = {
      modules: {
        greeting: {
          parameters: { name: { type: 'STRING', required: true } },
          sections: {
            pages: { '${params.name}-page': { name: '${params.name}' } },
          },
        },
      },
      imports: [{ module: 'greeting', as: 'hello', parameters: { name: 'world' } }],
      pages: {},
    };
    const result = expand(input);
    expect((result.map.pages as any)['hello.world-page']).toBeDefined();
  });

  it('returns diagnostics in lenient mode for unresolved variables', () => {
    const input = {
      pages: [{ name: '${var.missing}' }],
    };
    const result = expand(input, { strict: false });
    expect(result.diagnostics.length).toBeGreaterThan(0);
    expect(result.diagnostics[0]!.category).toBe('unresolved-variable');
  });

  it('throws in strict mode for unresolved variables', () => {
    const input = {
      pages: [{ name: '${var.missing}' }],
    };
    expect(() => expand(input, { strict: true })).toThrow();
  });
});
```

Run: `yarn workspace @casehubio/yaml-core test`
Expected: FAIL

- [ ] **Step 2: Implement expand()**

The `expand()` function orchestrates the pipeline:
1. Extract yaml-core sections from the input map (`variables`, `modules`, `imports`, `iterations`, `data`)
2. Build iteration groups from CSV data sources (if present)
3. Run ModuleExpander on imports + available modules → merge sections into document
4. Build VariableResolver from `variables:` section with `each` as deferred prefix
5. Resolve variables across the document
6. Run ForEachExpander on sections containing `forEach:` elements
7. Return `ExpandResult { map, diagnostics }`

In lenient mode: catch errors → collect as diagnostics, continue processing. Leave unresolved `${...}` as literal strings.
In strict mode: propagate errors (Java parity).

Remove `variables`, `modules`, `imports`, `iterations`, `data` from the output map — these are yaml-core-only keys consumed during expansion.

- [ ] **Step 3: Run integration tests**

Run: `yarn workspace @casehubio/yaml-core test`
Expected: PASS

- [ ] **Step 4: Vet json-schema-to-zod and generate schemas**

```bash
# Add dev dependency
yarn workspace @casehubio/yaml-core add -D json-schema-to-zod
```

Create `packages/yaml-core/scripts/generate-schemas.mjs`:
- Read each JSON Schema fragment from `../../platform/yaml-core/src/main/resources/schema/`
- Convert to Zod via `json-schema-to-zod`
- Write combined output to `src/schema.generated.ts`
- Verify: generated Zod handles `oneOf`, `$ref`, `$defs`, `enum`, `additionalProperties`

If `json-schema-to-zod` doesn't handle the 2020-12 features correctly, hand-write the 6 schemas (they're small — see `module.schema.json` for the largest at ~135 lines).

- [ ] **Step 5: Write schema.ts — composition schemas**

```typescript
// packages/yaml-core/src/schema.ts
import { z } from 'zod';
// Import generated base schemas from json-schema-to-zod output
import { parameterSchema, outputSchema, importSchema, forEachSchema } from './schema.generated.js';

export const moduleDefinitionSchema = z.object({
  parameters: z.record(parameterSchema).optional(),
  outputs: z.record(outputSchema).optional(),
  sections: z.record(z.record(z.unknown())).optional(),
  extends: z.string().optional(),
});

export const yamlCoreDocumentSchema = z.object({
  variables: z.record(z.record(z.string())).optional(),
  modules: z.record(moduleDefinitionSchema).optional(),
  imports: z.array(importSchema).optional(),
  iterations: z.record(z.object({ in: z.array(z.string()) })).optional(),
  data: z.record(z.unknown()).optional(),
});

export const yamlCoreElementMixin = {
  forEach: forEachSchema.optional(),
  when: z.string().optional(),
};

export { parameterSchema, outputSchema, importSchema, forEachSchema };
```

- [ ] **Step 6: Write schema tests**

Test that the generated/hand-written schemas validate real yaml-core YAML:

```typescript
// packages/yaml-core/src/schema.test.ts
import { describe, it, expect } from 'vitest';
import { yamlCoreDocumentSchema, moduleDefinitionSchema } from './schema.js';

describe('yaml-core schemas', () => {
  it('validates document with modules and imports', () => {
    const doc = {
      modules: {
        greeting: {
          parameters: { name: { type: 'STRING', required: true } },
          sections: { pages: { hello: { name: 'world' } } },
        },
      },
      imports: [{ module: 'greeting', as: 'hi', parameters: { name: 'test' } }],
    };
    expect(yamlCoreDocumentSchema.safeParse(doc).success).toBe(true);
  });

  it('validates document with variables', () => {
    const doc = { variables: { theme: { color: 'blue' } } };
    expect(yamlCoreDocumentSchema.safeParse(doc).success).toBe(true);
  });

  it('validates empty document', () => {
    expect(yamlCoreDocumentSchema.safeParse({}).success).toBe(true);
  });
});
```

- [ ] **Step 7: Commit**

```bash
git add packages/yaml-core/
git commit -m "feat(#435): expand() public API + Zod schema generation from JSON Schema fragments Refs #435"
```

---

## Batch 2: Pages integration — schema + runtime

### Task 4: Schema composition in pages-lsp + componentBase mixin

**Files:**
- Modify: `packages/pages-schema/package.json` (add yaml-core dep)
- Modify: `packages/pages-schema/src/document-schema.ts` (add yamlCoreElementMixin to componentBase)
- Modify: `packages/pages-lsp/package.json` (add yaml-core dep)
- Modify: `packages/pages-lsp/src/formats/page.ts` (z.intersection with yamlCoreDocumentSchema)
- Test: `packages/pages-lsp/src/completion.test.ts` (add yaml-core completion tests)

**Interfaces:**
- Consumes: `yamlCoreDocumentSchema`, `yamlCoreElementMixin`, `forEachSchema` (from Task 3)
- Produces: Updated `pageFormat.documentSchema` with yaml-core keys; updated `componentBase` with forEach/when

- [ ] **Step 1: Write failing completion test**

```typescript
// Add to packages/pages-lsp/src/completion.test.ts
it('offers yaml-core keys at document root', () => {
  const completions = getCompletions('page', '', 0, {});
  const labels = completions.map(c => c.label);
  expect(labels).toContain('modules');
  expect(labels).toContain('imports');
  expect(labels).toContain('variables');
});

it('offers forEach on component elements', () => {
  const completions = getCompletions('page', 'pages.0.components.0', 0, { type: 'title' });
  const labels = completions.map(c => c.label);
  expect(labels).toContain('forEach');
  expect(labels).toContain('when');
});
```

Run: `yarn workspace @casehubio/pages-lsp test`
Expected: FAIL

- [ ] **Step 2: Add yamlCoreElementMixin to componentBase**

In `packages/pages-schema/src/document-schema.ts`, import `yamlCoreElementMixin` and `forEachSchema` from `@casehubio/yaml-core/schema`, then add to `componentBase`:

```typescript
import { yamlCoreElementMixin, forEachSchema } from '@casehubio/yaml-core/schema';

const componentBase = z.object({
  id: z.string().optional(),
  style: z.record(z.string()).optional(),
  visibleWhen: z.string().optional(),
  ...yamlCoreElementMixin,
});
```

- [ ] **Step 3: Compose page format schema with yaml-core**

In `packages/pages-lsp/src/formats/page.ts`:

```typescript
import { yamlCoreDocumentSchema } from '@casehubio/yaml-core/schema';
import { dashboardSchema } from '@casehubio/pages-schema';

export const pageFormat: FormatRegistration = {
  formatId: 'page',
  extensions: ['.page.yaml'],
  contentDetector: (inspector) =>
    inspector.hasKey(['pages']) || inspector.hasKey(['datasets'])
    || inspector.hasKey(['modules']) || inspector.hasKey(['imports']),
  documentSchema: z.intersection(yamlCoreDocumentSchema, dashboardSchema),
  symbolExtractor: pageSymbolExtractor,
};
```

Note: content detector updated to also detect yaml-core-enhanced documents that may have `modules` or `imports` without `pages`.

- [ ] **Step 4: Run completion tests**

Run: `yarn workspace @casehubio/pages-lsp test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/pages-schema/ packages/pages-lsp/
git commit -m "feat(#435): schema composition — yaml-core keys in page format + componentBase mixin Refs #435"
```

### Task 5: Pages runtime integration — expand() in parser pipeline

**Files:**
- Modify: `packages/pages-ui/package.json` (add yaml-core dep)
- Modify: `packages/pages-ui/src/parser/page-parser.ts` (add yaml-core preprocessing)
- Test: `packages/pages-ui/src/parser/page-parser.test.ts` (add yaml-core expansion tests)

**Interfaces:**
- Consumes: `expand()` from `@casehubio/yaml-core/expand`
- Produces: Updated page parser that preprocesses yaml-core constructs before existing parsing logic

- [ ] **Step 1: Write failing parser tests**

```typescript
// Add to page-parser tests
it('expands variables in page YAML', () => {
  const yaml = `
variables:
  app:
    title: "My Dashboard"
pages:
  - name: \${app.title}
    components:
      - type: title
        properties:
          text: \${app.title}
`;
  const result = parsePage(yaml);
  expect(result.pages[0].name).toBe('My Dashboard');
});

it('passes through plain page YAML unchanged', () => {
  const yaml = `
pages:
  - name: test
    components:
      - type: title
`;
  const result = parsePage(yaml);
  expect(result.pages[0].name).toBe('test');
});

it('coexists with page-level properties substitution', () => {
  const yaml = `
variables:
  app:
    env: production
properties:
  title: Dashboard
pages:
  - name: \${title}
    components:
      - type: title
        properties:
          text: \${app.env}
`;
  const result = parsePage(yaml);
  // ${app.env} resolved by yaml-core (dot-separated prefix)
  // ${title} resolved by substituteProperties (no dot)
  expect(result.pages[0].components[0].properties.text).toBe('production');
  expect(result.pages[0].name).toBe('Dashboard');
});
```

Run: `yarn workspace @casehubio/pages-ui test`
Expected: FAIL

- [ ] **Step 2: Add yaml-core preprocessing to page parser**

In `packages/pages-ui/src/parser/page-parser.ts`, add an early preprocessing step:

```typescript
import { expand } from '@casehubio/yaml-core/expand';

function preprocessYamlCore(rawMap: Record<string, unknown>): Record<string, unknown> {
  const hasYamlCore = 'variables' in rawMap || 'modules' in rawMap
                   || 'imports' in rawMap || 'iterations' in rawMap || 'data' in rawMap;
  if (!hasYamlCore) return rawMap;

  const result = expand(rawMap, { strict: false });
  return result.map;
}
```

Call `preprocessYamlCore()` on the raw parsed YAML before the existing page parsing logic. The expanded map is a standard page document.

- [ ] **Step 3: Run parser tests**

Run: `yarn workspace @casehubio/pages-ui test`
Expected: PASS

- [ ] **Step 4: Run full test suite**

Run: `yarn test` (all packages)
Expected: PASS — no regressions

- [ ] **Step 5: Commit**

```bash
git add packages/pages-ui/
git commit -m "feat(#435): yaml-core preprocessing in page parser pipeline Refs #435"
```

---

## Batch 3: Builder integration

### Task 6: Tree view yaml-core nodes + visual preview expansion

**Files:**
- Modify: `packages/pages-document/src/page-document.ts` (add yaml-core accessors)
- Modify: `packages/pages-builder/src/tree/builder-tree.ts` (add yaml-core node types)
- Modify: `packages/pages-builder/src/shell/builder-shell.ts` (add expansion in visual preview)
- Modify: `packages/pages-builder/package.json` (add yaml-core dep)
- Test: `packages/pages-builder/src/tree/builder-tree.test.ts` (add yaml-core tree tests)
- Test: `packages/pages-document/src/page-document.test.ts` (add accessor tests)

**Interfaces:**
- Consumes: `expand()` from `@casehubio/yaml-core/expand`, `PageDocument` accessors
- Produces:
  - `PageDocument.getModules()`, `.getImports()`, `.getVariables()`
  - Extended `TreeNodeType` union with `'module' | 'import' | 'variable'`
  - Updated visual preview with yaml-core expansion

- [ ] **Step 1: Write failing PageDocument accessor tests**

```typescript
// Add to page-document.test.ts
it('returns modules from document', () => {
  const doc = PageDocument.parse('modules:\n  greeting:\n    parameters: {}');
  expect(doc.getModules()).toBeDefined();
  expect(doc.getModules()!['greeting']).toBeDefined();
});

it('returns undefined for document without modules', () => {
  const doc = PageDocument.parse('pages:\n  - name: test');
  expect(doc.getModules()).toBeUndefined();
});
```

- [ ] **Step 2: Implement PageDocument yaml-core accessors**

Add read-only accessors to `packages/pages-document/src/page-document.ts`:

```typescript
getModules(): Record<string, Record<string, unknown>> | undefined {
  const raw = this._doc.getIn(['modules']);
  if (!raw || !isMap(raw)) return undefined;
  return (raw as YAMLMap).toJSON() as Record<string, Record<string, unknown>>;
}

getImports(): Record<string, unknown>[] | undefined {
  const raw = this._doc.getIn(['imports']);
  if (!raw || !isSeq(raw)) return undefined;
  return (raw as YAMLSeq).toJSON() as Record<string, unknown>[];
}

getVariables(): Record<string, unknown> | undefined {
  const raw = this._doc.getIn(['variables']);
  if (!raw || !isMap(raw)) return undefined;
  return (raw as YAMLMap).toJSON() as Record<string, unknown>;
}
```

- [ ] **Step 3: Write failing tree model tests**

```typescript
// Add to builder-tree.test.ts
it('includes modules section in tree', () => {
  const doc = PageDocument.parse('modules:\n  greeting:\n    parameters: {}');
  const tree = buildTreeModel(doc);
  expect(tree.some(n => n.nodeType === 'module')).toBe(true);
});
```

- [ ] **Step 4: Implement tree model builder extensions**

Extend `TreeNodeType` to include `'module' | 'import' | 'variable'`.

Add `buildModuleNodes()`, `buildImportNodes()`, `buildVariableNodes()` functions that produce `TreeNodeInfo` from the raw maps returned by PageDocument accessors.

Update `buildTreeModel()` to call these before existing page nodes.

- [ ] **Step 5: Update builder-shell visual preview**

In `builder-shell.ts`, update `_refreshPreview()` to preprocess with `expand()` before rendering:

```typescript
import { expand } from '@casehubio/yaml-core/expand';

private _refreshPreview(): void {
  // ... existing code ...
  if (this.renderPreview) {
    const yamlText = this._document.toString();
    const parsed = YAML.parse(yamlText) as Record<string, unknown>;
    const expanded = expand(parsed, { strict: false });
    this.renderPreview(container, YAML.stringify(expanded.map));
  }
}
```

- [ ] **Step 6: Run all builder tests**

Run: `yarn workspace @casehubio/pages-builder test && yarn workspace @casehubio/pages-document test`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-document/ packages/pages-builder/
git commit -m "feat(#435): builder tree yaml-core nodes + visual preview expansion Refs #435"
```

---

## Batch 4: Tutorial infrastructure + content

### Task 7: Tutorial infrastructure — types, runner, host dispatch

**Files:**
- Modify: `packages/pages-aria/src/tutorial/types.ts` (add YamlEditorSection, YamlEditorScenario, contentType)
- Create: `packages/pages-aria/src/tutorial/yaml-editor-runner.ts`
- Modify: `packages/pages-aria/src/tutorial/tutorial-host.ts` (dispatch yaml-editor contentType)
- Modify: `scripts/build-tutorial-registry.mjs` (detect yaml-editor contentType)
- Test: `packages/pages-aria/src/tutorial/yaml-editor-runner.test.ts`
- Test: `packages/pages-aria/src/tutorial/tutorial-host.test.ts` (update existing tests)

**Interfaces:**
- Consumes: `TutorialDescriptor`, `LearningPath`, `SectionContent` (from types.ts), `expand()` from `@casehubio/yaml-core/expand`
- Produces:
  - `YamlEditorSection` interface
  - `YamlEditorScenario` interface
  - `runYamlEditorScenario()` function
  - Updated `contentType` union: `'slides-only' | 'hands-on' | 'yaml-editor'`

- [ ] **Step 1: Add YamlEditorSection types**

In `packages/pages-aria/src/tutorial/types.ts`:

```typescript
export interface YamlEditorSection {
  title: string;
  content?: SectionContent;
  initialYaml: string;
  expectedKeys?: string[];
  expectedStructure?: Record<string, unknown>;
  hint?: string;
  solutionYaml?: string;
  buildOnPrevious?: boolean;
}

export interface YamlEditorScenario extends ScenarioBase {
  sections: YamlEditorSection[];
}

// Update contentType
export interface TutorialDescriptor {
  // ... existing fields ...
  contentType: 'slides-only' | 'hands-on' | 'yaml-editor';
}
```

- [ ] **Step 2: Write failing runner tests**

```typescript
// packages/pages-aria/src/tutorial/yaml-editor-runner.test.ts
import { describe, it, expect } from 'vitest';
import { validateYamlStep } from './yaml-editor-runner.js';

describe('validateYamlStep', () => {
  it('passes valid YAML with expected keys', () => {
    const yaml = 'variables:\n  theme:\n    color: blue';
    const result = validateYamlStep(yaml, { expectedKeys: ['variables'] });
    expect(result.valid).toBe(true);
  });

  it('fails on missing expected key', () => {
    const yaml = 'pages:\n  - name: test';
    const result = validateYamlStep(yaml, { expectedKeys: ['variables'] });
    expect(result.valid).toBe(false);
  });

  it('fails on YAML syntax error', () => {
    const yaml = 'invalid: [unclosed';
    const result = validateYamlStep(yaml, {});
    expect(result.valid).toBe(false);
  });

  it('validates expectedStructure against expanded map', () => {
    const yaml = `
variables:
  app:
    title: Dashboard
pages:
  - name: \${app.title}
    components:
      - type: title
`;
    const result = validateYamlStep(yaml, {
      expectedStructure: {
        pages: [{ components: [{ type: 'title' }] }],
      },
    });
    expect(result.valid).toBe(true);
  });
});
```

- [ ] **Step 3: Implement yaml-editor-runner**

Create `packages/pages-aria/src/tutorial/yaml-editor-runner.ts`:
- `validateYamlStep(yaml, section)` — runs the 5-step validation pipeline
- `runYamlEditorScenario(scenario, options)` — manages step progression with builder workbench
- Deep-subset assertion with multiset array matching for `expectedStructure`

- [ ] **Step 4: Update tutorial-host for yaml-editor dispatch**

In `packages/pages-aria/src/tutorial/tutorial-host.ts`, add dispatch:
- Check `contentType === 'yaml-editor'` on the descriptor
- Render `<pages-builder-shell>` (untyped custom element) instead of narrative + controller
- Wire validation callbacks on YAML change

- [ ] **Step 5: Update build-tutorial-registry.mjs**

Add detection logic: if any section has `initialYaml`, contentType is `'yaml-editor'`.

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/pages-aria test`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/pages-aria/ scripts/
git commit -m "feat(#435): yaml-editor tutorial infrastructure — types, runner, host dispatch Refs #435"
```

### Task 8: Tutorial content — 15-step learning path

**Files:**
- Create: `tutorials/yaml-composition/tutorial.yaml`
- Create: `tutorials/yaml-composition/steps/01-yaml-basics.yaml` through `15-full-application.yaml`
- Create: `tutorials/yaml-composition/content/01-intro.md` through `15-recap.md`
- Create: `tutorials/paths/yaml-composition.yaml` (learning path)

**Interfaces:**
- Consumes: `YamlEditorSection`, `YamlEditorScenario` types, builder workbench, yaml-core expansion
- Produces: Complete 15-step tutorial with YAML examples, validation, and narrative content

- [ ] **Step 1: Create tutorial descriptor**

`tutorials/yaml-composition/tutorial.yaml`:
```yaml
scenario: yaml-composition
meta:
  title: "CaseHub YAML Composition"
  description: "Learn the CaseHub YAML composition language — variables, modules, forEach, conditionals — by building progressively complex pages"
  area: yaml-composition
  labels:
    - concept:variables
    - concept:modules
    - concept:forEach
    - concept:conditionals
    - difficulty:beginner
  tags:
    - yaml
    - composition
    - getting-started
  estimated: "45 min"
  prerequisites: []
  hero:
    title: "CaseHub YAML Composition"
    subtitle: "Build complex pages from reusable, parameterized modules"
    icon: "🧩"
```

- [ ] **Step 2: Author steps 1-5 (basics + variables)**

Create `tutorials/yaml-composition/steps/01-yaml-basics.yaml` through `05-variable-scoping.yaml`.

Each step file contains:
```yaml
title: "YAML Basics"
content:
  type: template
  path: content/01-intro.md
initialYaml: |
  pages:
    - name: My First Page
      components:
        - type: title
          properties:
            text: Hello World
expectedKeys: [pages]
expectedStructure:
  pages:
    - components:
        - type: title
hint: "Start with a simple page containing one title component"
solutionYaml: |
  pages:
    - name: My First Page
      components:
        - type: title
          properties:
            text: Hello World
```

Create corresponding `tutorials/yaml-composition/content/01-intro.md` through `05-variable-scoping.md` with narrative explaining each concept. Each markdown file teaches the concept with examples and explains what the user should do.

- [ ] **Step 3: Author steps 6-10 (forEach + modules)**

Create steps 6-10 covering forEach basics, forEach with data, modules intro, module parameters, module imports. Each builds on previous concepts — use `buildOnPrevious: true` where appropriate.

Representative step (step 6 — forEach basics):
```yaml
title: "ForEach Basics"
content:
  type: template
  path: content/06-foreach-basics.md
initialYaml: |
  pages:
    - name: Metrics Dashboard
      components:
        - type: metric
          properties:
            field: cpu
            title: CPU Usage
        - type: metric
          properties:
            field: memory
            title: Memory Usage
        - type: metric
          properties:
            field: disk
            title: Disk Usage
expectedKeys: [pages]
expectedStructure:
  pages:
    - components:
        - type: metric
        - type: metric
        - type: metric
hint: "Replace the three repeated metric components with a single component using forEach: {as: field, in: [cpu, memory, disk]}"
solutionYaml: |
  pages:
    - name: Metrics Dashboard
      components:
        - type: metric
          forEach: {as: field, in: [cpu, memory, disk]}
          properties:
            field: ${each.field}
            title: ${each.field} Usage
buildOnPrevious: false
```

Representative step (step 8 — modules intro):
```yaml
title: "Modules — Reusable Blocks"
content:
  type: template
  path: content/08-modules-intro.md
initialYaml: |
  pages:
    - name: Sales Dashboard
      components:
        - type: metric
          forEach: {as: field, in: [revenue, orders, conversion]}
          properties:
            field: ${each.field}
            dataset: sales-data
    - name: Ops Dashboard
      components:
        - type: metric
          forEach: {as: field, in: [uptime, latency, errors]}
          properties:
            field: ${each.field}
            dataset: ops-data
expectedKeys: [modules, imports]
expectedStructure:
  pages:
    - name: Sales Dashboard
    - name: Ops Dashboard
hint: "Extract the repeated forEach+metric pattern into a module with 'dataset' and 'metrics' parameters"
solutionYaml: |
  modules:
    metric-row:
      parameters:
        dataset: {type: STRING, required: true}
        metrics: {type: LIST, required: true}
      sections:
        pages:
          ${params.dataset}-dashboard:
            name: ${params.dataset} Dashboard
            components:
              - type: metric
                forEach: {as: field, in: ${params.metrics}}
                properties:
                  field: ${each.field}
                  dataset: ${params.dataset}
  imports:
    - module: metric-row
      as: sales
      parameters: {dataset: sales-data, metrics: "revenue,orders,conversion"}
    - module: metric-row
      as: ops
      parameters: {dataset: ops-data, metrics: "uptime,latency,errors"}
buildOnPrevious: false
```

- [ ] **Step 4: Author steps 11-15 (outputs + conditionals + composition)**

Create steps 11-15 covering module outputs, conditionals, composition, iteration groups, full application. Step 15 brings everything together.

Representative step (step 12 — conditionals):
```yaml
title: "Conditionals — when: Guards"
content:
  type: template
  path: content/12-conditionals.md
initialYaml: |
  variables:
    features:
      showMetrics: "true"
      showTable: "false"
  pages:
    - name: Dashboard
      components:
        - type: metric
          properties:
            field: revenue
        - type: data-table
          properties:
            dataset: sales
expectedKeys: [variables]
hint: "Add when: conditions to show/hide components based on the feature flags in variables"
solutionYaml: |
  variables:
    features:
      showMetrics: "true"
      showTable: "false"
  pages:
    - name: Dashboard
      components:
        - type: metric
          when: "${features.showMetrics}"
          properties:
            field: revenue
        - type: data-table
          when: "${features.showTable}"
          properties:
            dataset: sales
buildOnPrevious: false
```

- [ ] **Step 5: Create learning path**

`tutorials/paths/yaml-composition.yaml`:
```yaml
path: yaml-composition
title: "CaseHub YAML Composition Language"
description: "Master variables, modules, forEach, and conditionals"
labels:
  - difficulty:beginner
tutorials:
  - yaml-composition
```

- [ ] **Step 6: Build and verify registry**

Run: `node scripts/build-tutorial-registry.mjs`
Verify: `dist/tutorial-registry.json` includes the new tutorial with `contentType: 'yaml-editor'`.

- [ ] **Step 7: Manual test**

Start dev server: `yarn dev`
Navigate to tutorials page. Verify:
- Tutorial appears in catalog with correct metadata
- Clicking opens the builder workbench
- Each step loads initialYaml in the editor
- Editing YAML updates the tree view and visual preview
- Validation passes when expected keys/structure are present
- "Next" advances only after validation passes

- [ ] **Step 8: Commit**

```bash
git add tutorials/
git commit -m "feat(#435): 15-step YAML composition tutorial — learning path complete Refs #435"
```

---

## Out of scope (tracked separately)

- **Issue 9: blocks-ui schema composition** (case, swf, htn, org) — same one-line `z.intersection()` pattern as Task 4, applied in `blocks-ui/packages/lsp-schemas/`. Separate repo, separate session.
- **Issues 10-12: follow-up issues** (LSP semantic completions, expanded diagrams, J2CL evaluation) — filed as GitHub issues per spec §8.

## References

- [2026-09-14-yaml-core-integration-tutorial-design.md] — design spec this plan implements
- [decisions.md] — 7 decisions (D2+D5 revised)
- [platform/yaml-core/src/main/java/io/casehub/yaml/core/] — Java source for TS port
- [platform/yaml-core/src/main/resources/schema/*.schema.json] — JSON Schema fragments
- [packages/pages-lsp/src/schema-navigation.ts] — LSP completion engine
- [packages/pages-lsp/src/formats/page.ts] — page format registration
- [packages/pages-builder/src/shell/builder-shell.ts] — builder workbench
- [packages/pages-builder/src/tree/builder-tree.ts] — tree view component
- [packages/pages-document/src/page-document.ts] — CST-backed document model
- [packages/pages-aria/src/tutorial/tutorial-host.ts] — tutorial runner
- [packages/pages-aria/src/tutorial/types.ts] — tutorial type definitions
- [casehubio/platform#247] — yaml-core extraction issue
- [casehubio/casehub-pages#344] — J2CL strategy (deferred)
- [casehubio/casehub-pages#428] — visual YAML builder
