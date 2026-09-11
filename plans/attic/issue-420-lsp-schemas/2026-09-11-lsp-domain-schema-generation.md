# LSP Domain Schema Generation Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #420 — feat(pages-lsp): ts-morph generation pipeline for domain Zod schemas
**Issue group:** #420

**Goal:** Replace hand-written Zod schemas in `lsp-schemas` (blocks-ui)
with ts-morph-generated schemas for CaseDefinition, Org, HTN, and SWF
domain formats.

**Architecture:** A `generate-domain-schemas.ts` script in
`lsp-schemas/scripts/` uses ts-morph to read TypeScript interfaces from
graph-stencil-* packages, strips index signatures, detects recursive
types, reads discriminator manifests for union recovery, and emits one
`.generated.ts` file per domain format. A vitest staleness test guards
against upstream type drift.

**Tech Stack:** TypeScript, ts-morph, Zod, vitest, tsx

## Global Constraints

- Generated files import only from `zod` — no runtime dependency on ts-morph
- Index signatures (`[k: string]: unknown`) are always stripped
- Function-typed properties are always skipped
- Generated files start with `// AUTO-GENERATED` header
- Existing lsp-schemas tests must pass after each task
- All work is in blocks-ui repo at `/Users/mdproctor/claude/casehub/slots/188/blocks-ui`

---

## Batch 1: Generator Core + CaseDefinition

After this batch: the ts-morph generator exists and produces a correct
CaseDefinition schema with discriminator-driven union recovery. The
hand-written `case-definition.ts` is replaced by `case-definition.generated.ts`.

### Task 1: Generator Infrastructure

**Files:**
- Create: `packages/lsp-schemas/scripts/generate-domain-schemas.ts`
- Create: `packages/lsp-schemas/tsconfig.generator.json`
- Modify: `packages/lsp-schemas/package.json` — add devDeps + scripts
- Test: `packages/lsp-schemas/test/generator-core.test.ts`

**Interfaces:**
- Produces: `typeToZod(type: Type, depth: number, visited: Set<string>): string` — core type mapper
- Produces: `propToZodField(prop: MorphSymbol, depth: number, visited: Set<string>): string` — property mapper with index sig filtering
- Produces: `generateFormatSchema(project: Project, config: FormatConfig): string` — per-format entry point
- Produces: `FormatConfig` type — `{ formatId: string; rootTypeName: string; sourceFile: string; discriminatorManifest?: string }`

- [ ] **Step 1: Add devDependencies**

Add `ts-morph` and `tsx` to `packages/lsp-schemas/package.json` devDependencies:

```bash
yarn workspace @casehubio/lsp-schemas add -D ts-morph tsx
```

Add scripts to `package.json`:

```json
{
  "generate": "tsx scripts/generate-domain-schemas.ts"
}
```

Update `build` script to run generate first:

```json
{
  "build": "yarn generate && tsc -p tsconfig.build.json"
}
```

- [ ] **Step 2: Create tsconfig.generator.json**

Create `packages/lsp-schemas/tsconfig.generator.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2022",
    "paths": {
      "@casehubio/graph-stencil-case": ["../graph-stencil-case/src/index.ts"],
      "@casehubio/graph-stencil-case/*": ["../graph-stencil-case/src/*"],
      "@casehubio/graph-stencil-htn": ["../graph-stencil-htn/src/index.ts"],
      "@casehubio/graph-stencil-org": ["../graph-stencil-org/src/index.ts"]
    }
  },
  "include": [
    "../graph-stencil-case/src/**/*.ts",
    "../graph-stencil-htn/src/**/*.ts",
    "../graph-stencil-org/src/**/*.ts"
  ]
}
```

Note: If `tsconfig.base.json` does not exist at the blocks-ui root, use
inline compiler options instead (check before writing).

- [ ] **Step 3: Write failing test for typeToZod core mapping**

Create `packages/lsp-schemas/test/generator-core.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { Project, SyntaxKind } from 'ts-morph';

import { typeToZod, propToZodField } from '../scripts/generate-domain-schemas.js';

describe('typeToZod', () => {
  const project = new Project({ useInMemoryFileSystem: true });

  function zodForType(typeCode: string): string {
    const file = project.createSourceFile(
      'test.ts',
      `export type T = ${typeCode};`,
      { overwrite: true },
    );
    const typeAlias = file.getTypeAliasOrThrow('T');
    return typeToZod(typeAlias.getType(), 0, new Set());
  }

  it('maps string to z.string()', () => {
    expect(zodForType('string')).toBe('z.string()');
  });

  it('maps number to z.number()', () => {
    expect(zodForType('number')).toBe('z.number()');
  });

  it('maps boolean to z.boolean()', () => {
    expect(zodForType('boolean')).toBe('z.boolean()');
  });

  it('maps string literal union to z.enum()', () => {
    const result = zodForType("'a' | 'b' | 'c'");
    expect(result).toBe('z.enum(["a", "b", "c"])');
  });

  it('maps string array to z.array(z.string())', () => {
    expect(zodForType('string[]')).toBe('z.array(z.string())');
  });

  it('maps Record<string, T> to z.record()', () => {
    expect(zodForType('Record<string, number>')).toBe('z.record(z.number())');
  });

  it('maps unknown to z.unknown()', () => {
    expect(zodForType('unknown')).toBe('z.unknown()');
  });
});

describe('propToZodField — index signature stripping', () => {
  const project = new Project({ useInMemoryFileSystem: true });

  it('strips index signature properties', () => {
    const file = project.createSourceFile(
      'idx.ts',
      `export interface Foo {
        name: string;
        [k: string]: unknown;
      }`,
      { overwrite: true },
    );
    const iface = file.getInterfaceOrThrow('Foo');
    const props = iface.getType().getProperties();
    const fields = props
      .map(p => propToZodField(p, 0, new Set()))
      .filter(Boolean);
    expect(fields).toHaveLength(1);
    expect(fields[0]).toContain('name');
  });
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --run test/generator-core.test.ts`
Expected: FAIL — `typeToZod` and `propToZodField` are not exported

- [ ] **Step 5: Implement generator infrastructure**

Create `packages/lsp-schemas/scripts/generate-domain-schemas.ts`:

```typescript
import { Project, Type, Symbol as MorphSymbol, SyntaxKind } from 'ts-morph';
import { writeFileSync, readFileSync, existsSync } from 'fs';
import { resolve, dirname } from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

export interface FormatConfig {
  formatId: string;
  rootTypeName: string;
  sourceFile: string;
  outputFile: string;
  discriminatorManifest?: string;
}

export function typeToZod(type: Type, depth: number, visited: Set<string>): string {
  if (depth > 8) return 'z.unknown()';

  const symbol = type.getSymbol() || type.getAliasSymbol();
  const symbolId = symbol?.getFullyQualifiedName() ?? '';

  if (symbolId && visited.has(symbolId)) {
    return `z.lazy(() => ${symbolId.split('.').pop()}Schema)`;
  }

  const text = type.getText();

  if (type.isString() || type.isStringLiteral()) return 'z.string()';
  if (type.isNumber() || type.isNumberLiteral()) return 'z.number()';
  if (type.isBoolean() || type.isBooleanLiteral()) return 'z.boolean()';
  if (type.isNull()) return 'z.null()';
  if (type.isUndefined()) return 'z.undefined()';
  if (text === 'unknown') return 'z.unknown()';

  if (type.isUnion()) {
    const members = type.getUnionTypes().filter(t => !t.isUndefined());
    if (members.length === 0) return 'z.undefined()';
    if (members.every(m => m.isStringLiteral())) {
      const values = members.map(m => JSON.stringify(m.getLiteralValue()));
      return `z.enum([${values.join(', ')}])`;
    }
    if (members.every(m => m.isBooleanLiteral())) return 'z.boolean()';
    if (members.length === 1) return typeToZod(members[0], depth + 1, visited);
    const zodMembers = members.map(m => typeToZod(m, depth + 1, visited));
    return `z.union([${zodMembers.join(', ')}])`;
  }

  if (type.isArray()) {
    const elem = type.getArrayElementType();
    if (!elem) return 'z.array(z.unknown())';
    return `z.array(${typeToZod(elem, depth + 1, visited)})`;
  }

  if (text.startsWith('readonly ') && text.endsWith('[]')) {
    const inner = type.getTypeArguments()[0];
    if (inner) return `z.array(${typeToZod(inner, depth + 1, visited)})`;
    return 'z.array(z.unknown())';
  }

  if (text.startsWith('Record<') || text.includes('Record<string,')) {
    const typeArgs = type.getAliasTypeArguments();
    if (typeArgs.length === 2) {
      return `z.record(${typeToZod(typeArgs[1], depth + 1, visited)})`;
    }
    const indexType = type.getStringIndexType();
    if (indexType) return `z.record(${typeToZod(indexType, depth + 1, visited)})`;
    return 'z.record(z.unknown())';
  }

  if (type.isObject() && !type.isArray()) {
    if (symbolId) visited.add(symbolId);
    const props = type.getProperties();
    if (props.length === 0) return 'z.object({})';
    const fields = props
      .map(p => propToZodField(p, depth + 1, visited))
      .filter(Boolean);
    if (symbolId) visited.delete(symbolId);
    if (fields.length === 0) return 'z.object({})';
    const indent = '  '.repeat(depth + 2);
    const closingIndent = '  '.repeat(depth + 1);
    return `z.object({\n${indent}${fields.join(`,\n${indent}`)},\n${closingIndent}})`;
  }

  return 'z.unknown()';
}

function isFunction(type: Type): boolean {
  return type.getCallSignatures().length > 0;
}

function isIndexSignature(prop: MorphSymbol): boolean {
  const decl = prop.getValueDeclaration();
  if (!decl) return false;
  return decl.getKind() === SyntaxKind.IndexSignature;
}

export function propToZodField(
  prop: MorphSymbol,
  depth: number,
  visited: Set<string>,
): string {
  const name = prop.getName();
  if (isIndexSignature(prop)) return '';

  const decl = prop.getValueDeclaration();
  if (!decl) return '';

  const type = prop.getTypeAtLocation(decl);
  if (isFunction(type)) return '';

  const isOptional = prop.isOptional();
  const baseType = isOptional ? type.getNonNullableType() : type;
  let zodType = typeToZod(baseType, depth, visited);
  if (!zodType) return '';
  if (isOptional) zodType += '.optional()';

  const safeName = /^[a-zA-Z_$][a-zA-Z0-9_$]*$/.test(name) ? name : `"${name}"`;
  return `${safeName}: ${zodType}`;
}

const FORMATS: FormatConfig[] = [
  // Will be populated in Task 2+
];

function generateHeader(sourceDesc: string): string {
  return `// AUTO-GENERATED by scripts/generate-domain-schemas.ts — DO NOT EDIT
// Re-generate: yarn workspace @casehubio/lsp-schemas run generate
// Source: ${sourceDesc}
import { z } from "zod";
`;
}

if (import.meta.url === `file://${process.argv[1]}`) {
  const project = new Project({
    tsConfigFilePath: resolve(__dirname, '../tsconfig.generator.json'),
  });

  for (const config of FORMATS) {
    const sourceFile = project.getSourceFileOrThrow(
      resolve(__dirname, config.sourceFile),
    );
    const rootType = sourceFile.getTypeAliasOrThrow(config.rootTypeName).getType();
    const visited = new Set<string>();
    const zodCode = typeToZod(rootType, 0, visited);
    const output = `${generateHeader(config.sourceFile)}\nexport const ${config.formatId}DocumentSchema = ${zodCode};\n`;
    const outPath = resolve(__dirname, config.outputFile);
    writeFileSync(outPath, output, 'utf-8');
    console.log(`Generated ${config.formatId} schema to ${outPath}`);
  }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --run test/generator-core.test.ts`
Expected: PASS

- [ ] **Step 7: Run existing tests to verify no regressions**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: All existing tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui add packages/lsp-schemas/scripts/ packages/lsp-schemas/tsconfig.generator.json packages/lsp-schemas/package.json packages/lsp-schemas/test/generator-core.test.ts
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui commit -m "feat(lsp-schemas): generator infrastructure with typeToZod and index sig stripping Refs casehubio/casehub-pages#420"
```

---

### Task 2: CaseDefinition Generation + Discriminator Manifest

**Files:**
- Create: `packages/lsp-schemas/discriminators/case-definition.json`
- Create: `packages/lsp-schemas/src/schemas/case-definition.generated.ts` (generated output)
- Modify: `packages/lsp-schemas/scripts/generate-domain-schemas.ts` — add CaseDefinition config + manifest reading
- Modify: `packages/lsp-schemas/src/formats/case-definition.ts` — switch import to generated schema
- Modify: `packages/lsp-schemas/src/index.ts` — update re-export if needed
- Test: `packages/lsp-schemas/test/case-definition.test.ts` (existing — must still pass)

**Interfaces:**
- Consumes: `typeToZod`, `propToZodField`, `FormatConfig` from Task 1
- Produces: `caseDefinitionDocumentSchema` (generated Zod schema, same export name as hand-written)

- [ ] **Step 1: Write failing test for CaseDefinition generation**

Add to `packages/lsp-schemas/test/generator-core.test.ts`:

```typescript
describe('CaseDefinition generation', () => {
  it('generated schema parses a valid case definition', async () => {
    const { caseDefinitionDocumentSchema } = await import(
      '../src/schemas/case-definition.generated.js'
    );
    const result = caseDefinitionDocumentSchema.safeParse({
      dsl: '1.0',
      namespace: 'test',
      name: 'TestCase',
      spec: {
        capabilities: [{ name: 'cap1' }],
        bindings: [{ name: 'b1', capability: 'cap1' }],
        workers: [{ name: 'w1', capabilities: ['cap1'] }],
      },
    });
    expect(result.success).toBe(true);
  });

  it('generated schema does not allow index signature keys', async () => {
    const { caseDefinitionDocumentSchema } = await import(
      '../src/schemas/case-definition.generated.js'
    );
    const result = caseDefinitionDocumentSchema.strict().safeParse({
      dsl: '1.0',
      spec: { capabilities: [] },
      randomUnknownKey: 'should fail',
    });
    expect(result.success).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --run test/generator-core.test.ts`
Expected: FAIL — `case-definition.generated.ts` does not exist

- [ ] **Step 3: Create discriminator manifest**

Create `packages/lsp-schemas/discriminators/case-definition.json`:

```json
{
  "spec.bindings[]": {
    "strategy": "key-presence",
    "discriminatorKeys": ["subCase", "humanTask", "trigger"],
    "sharedKeys": ["name", "guard"]
  }
}
```

- [ ] **Step 4: Add CaseDefinition to FORMATS config and implement manifest reading**

Update `packages/lsp-schemas/scripts/generate-domain-schemas.ts`:

1. Add discriminator manifest reading logic
2. Add CaseDefinition to the FORMATS array:

```typescript
{
  formatId: 'caseDefinition',
  rootTypeName: 'CaseHub',
  sourceFile: '../../graph-stencil-case/src/types/generated/case-definition.ts',
  outputFile: '../src/schemas/case-definition.generated.ts',
  discriminatorManifest: '../discriminators/case-definition.json',
}
```

3. Implement the generation logic that walks the `CaseHub` root type,
   applies `typeToZod` with `stripIndexSignatures`, and reads the
   discriminator manifest to emit `z.union()` at annotated paths.

4. Run the generator:

```bash
yarn workspace @casehubio/lsp-schemas run generate
```

5. Inspect the generated output — verify it looks correct vs the
   hand-written `src/schemas/case-definition.ts`.

- [ ] **Step 5: Update format registration import**

Modify `packages/lsp-schemas/src/formats/case-definition.ts` line 3:

```typescript
// Before
import { caseDefinitionDocumentSchema } from '../schemas/case-definition.js';
// After
import { caseDefinitionDocumentSchema } from '../schemas/case-definition.generated.js';
```

- [ ] **Step 6: Run tests to verify generated schema works**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: ALL tests PASS (both new generator tests and existing case-definition tests)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui add packages/lsp-schemas/
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui commit -m "feat(lsp-schemas): generate CaseDefinition schema with discriminator manifest Refs casehubio/casehub-pages#420"
```

---

## Batch 2: Remaining Formats + Staleness Tests

After this batch: all four domain formats use generated schemas and a
staleness test guards against upstream type drift.

### Task 3: Org Generation

**Files:**
- Create: `packages/lsp-schemas/src/schemas/org.generated.ts` (generated output)
- Modify: `packages/lsp-schemas/scripts/generate-domain-schemas.ts` — add Org config
- Modify: `packages/lsp-schemas/src/formats/org.ts` — switch import
- Test: `packages/lsp-schemas/test/org.test.ts` (existing — must still pass)

**Interfaces:**
- Consumes: `generateFormatSchema`, `FormatConfig` from Task 1
- Produces: `orgDocumentSchema` (generated)

- [ ] **Step 1: Write failing test for Org generation**

Add to `packages/lsp-schemas/test/generator-core.test.ts`:

```typescript
describe('Org generation', () => {
  it('generated schema parses a valid org document', async () => {
    const { orgDocumentSchema } = await import(
      '../src/schemas/org.generated.js'
    );
    const result = orgDocumentSchema.safeParse({
      organization: {
        units: [{
          unitId: 'u1',
          name: 'Engineering',
          tenancyId: 't1',
          members: [{ agentId: 'a1' }],
          capabilities: [{ name: 'coding' }],
          goals: [{ name: 'ship' }],
          constraints: [{ name: 'budget' }],
        }],
        relationships: [{
          sourceAgentId: 'a1',
          targetAgentId: 'a2',
          kind: 'SUPERVISES',
          tenancyId: 't1',
        }],
      },
    });
    expect(result.success).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --run test/generator-core.test.ts`
Expected: FAIL — `org.generated.ts` does not exist

- [ ] **Step 3: Add Org to FORMATS and generate**

Add to FORMATS in `generate-domain-schemas.ts`:

```typescript
{
  formatId: 'org',
  rootTypeName: 'OrgStructureYaml',
  sourceFile: '../../graph-stencil-org/src/types.ts',
  outputFile: '../src/schemas/org.generated.ts',
}
```

Run: `yarn workspace @casehubio/lsp-schemas run generate`

- [ ] **Step 4: Update format registration import**

Modify `packages/lsp-schemas/src/formats/org.ts`:

```typescript
// Before
import { orgDocumentSchema } from '../schemas/org.js';
// After
import { orgDocumentSchema } from '../schemas/org.generated.js';
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui add packages/lsp-schemas/
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui commit -m "feat(lsp-schemas): generate Org schema from OrgStructureYaml Refs casehubio/casehub-pages#420"
```

---

### Task 4: HTN Generation

**Files:**
- Create: `packages/lsp-schemas/src/schemas/htn.generated.ts` (generated output)
- Create: `packages/graph-stencil-htn/src/types/htn-yaml.ts` — YAML definition interfaces (if they don't exist)
- Modify: `packages/lsp-schemas/scripts/generate-domain-schemas.ts` — add HTN config
- Modify: `packages/lsp-schemas/src/formats/htn.ts` — switch import
- Test: `packages/lsp-schemas/test/htn.test.ts` (existing — must still pass)

**Interfaces:**
- Consumes: `generateFormatSchema`, `FormatConfig` from Task 1
- Produces: `htnDocumentSchema` (generated)

**Note:** The graph-stencil-htn types (`TaskNodeSnapshot`, `DagPlanSnapshot`)
are runtime snapshot types, NOT the YAML definition format. The HTN YAML
format defines `dsl`, `namespace`, `name`, `spec.decomposition.root` with
recursive `task → methods → tasks` structure. If `HtnDocumentYaml`
interfaces do not exist, they must be authored first — derive them from
the existing hand-written Zod schema in `src/schemas/htn.ts`.

- [ ] **Step 1: Check if HTN YAML definition types exist**

Search `packages/graph-stencil-htn/src/` for an interface matching the
YAML format (`dsl`, `namespace`, `spec.decomposition`). If found, use it.
If not, proceed to Step 2.

- [ ] **Step 2: Author HTN YAML definition interfaces (if needed)**

Create `packages/graph-stencil-htn/src/types/htn-yaml.ts`:

```typescript
export interface HtnMethodYaml {
  guard?: string;
  guardLabel?: string;
  strategy?: string;
  estimatedCost?: number;
  estimatedDuration?: string;
  tasks: HtnTaskYaml[];
}

export interface HtnTaskYaml {
  name: string;
  capability?: string;
  definitionRef?: string;
  methods?: HtnMethodYaml[];
}

export interface HtnDocumentYaml {
  dsl?: string;
  namespace?: string;
  name?: string;
  spec: {
    decomposition: {
      root: HtnTaskYaml;
    };
  };
}
```

Export from `packages/graph-stencil-htn/src/types/index.ts`:

```typescript
export type { HtnDocumentYaml, HtnTaskYaml, HtnMethodYaml } from './htn-yaml.js';
```

- [ ] **Step 3: Write failing test for HTN generation**

Add to `packages/lsp-schemas/test/generator-core.test.ts`:

```typescript
describe('HTN generation', () => {
  it('generated schema parses a valid HTN document', async () => {
    const { htnDocumentSchema } = await import(
      '../src/schemas/htn.generated.js'
    );
    const result = htnDocumentSchema.safeParse({
      dsl: '1.0',
      namespace: 'test',
      name: 'TestHTN',
      spec: {
        decomposition: {
          root: {
            name: 'root-task',
            methods: [{
              guard: '.status == "active"',
              tasks: [{ name: 'leaf-task', capability: 'cap1' }],
            }],
          },
        },
      },
    });
    expect(result.success).toBe(true);
  });
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --run test/generator-core.test.ts`
Expected: FAIL

- [ ] **Step 5: Add HTN to FORMATS and generate**

Add to FORMATS:

```typescript
{
  formatId: 'htn',
  rootTypeName: 'HtnDocumentYaml',
  sourceFile: '../../graph-stencil-htn/src/types/htn-yaml.ts',
  outputFile: '../src/schemas/htn.generated.ts',
}
```

Run: `yarn workspace @casehubio/lsp-schemas run generate`

Note: HTN has mutual recursion (`HtnTaskYaml` ↔ `HtnMethodYaml`). The
generator's cycle detection (`visited` set) must emit `z.lazy()` at the
cycle-breaking point. Verify the generated output has `z.lazy()` wrappers.

- [ ] **Step 6: Update format registration import**

Modify `packages/lsp-schemas/src/formats/htn.ts`:

```typescript
// Before
import { htnDocumentSchema } from '../schemas/htn.js';
// After
import { htnDocumentSchema } from '../schemas/htn.generated.js';
```

- [ ] **Step 7: Run tests**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui add packages/lsp-schemas/ packages/graph-stencil-htn/
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui commit -m "feat(lsp-schemas): generate HTN schema with recursive type support Refs casehubio/casehub-pages#420"
```

---

### Task 5: SWF Generation + Staleness Tests

**Files:**
- Create: `packages/lsp-schemas/src/schemas/swf.generated.ts` (generated output)
- Modify: `packages/lsp-schemas/scripts/generate-domain-schemas.ts` — add SWF config
- Modify: `packages/lsp-schemas/src/formats/swf.ts` — switch import
- Test: `packages/lsp-schemas/test/swf.test.ts` (existing — must still pass)
- Test: `packages/lsp-schemas/test/staleness.test.ts` — new staleness test

**Interfaces:**
- Consumes: `generateFormatSchema`, `FormatConfig` from Task 1
- Produces: `swfDocumentSchema` (generated), staleness test

**Note:** The SWF hand-written schema is intentionally loose (mostly
`z.unknown()`). The `@openworkflowspec/sdk` package provides the type
definitions. If the SDK is not installed, install it first. If it lacks
TypeScript type definitions for the document format, create minimal YAML
definition interfaces in `graph-stencil-swf/src/` similar to the HTN
approach.

- [ ] **Step 1: Resolve SWF source types**

Check if `@openworkflowspec/sdk` is installed and has document type
definitions. If not, either install it or create minimal SWF YAML
definition interfaces:

```typescript
// packages/graph-stencil-swf/src/swf-yaml.ts (if SDK lacks types)
export interface SwfDocumentInfo {
  dsl?: string;
  namespace?: string;
  name?: string;
  version?: string;
}

export interface SwfDocumentYaml {
  document?: SwfDocumentInfo;
  input?: unknown;
  output?: unknown;
  do: Record<string, unknown>[];
  use?: unknown;
  timeout?: unknown;
  schedule?: unknown;
}
```

- [ ] **Step 2: Write failing test for SWF generation**

Add to `packages/lsp-schemas/test/generator-core.test.ts`:

```typescript
describe('SWF generation', () => {
  it('generated schema parses a valid SWF document', async () => {
    const { swfDocumentSchema } = await import(
      '../src/schemas/swf.generated.js'
    );
    const result = swfDocumentSchema.safeParse({
      document: { dsl: '1.0', name: 'test-workflow' },
      do: [{ callTask: { call: 'http', with: { uri: 'https://example.com' } } }],
    });
    expect(result.success).toBe(true);
  });
});
```

- [ ] **Step 3: Add SWF to FORMATS and generate**

Add to FORMATS and run: `yarn workspace @casehubio/lsp-schemas run generate`

- [ ] **Step 4: Update format registration import**

Modify `packages/lsp-schemas/src/formats/swf.ts`:

```typescript
// Before
import { swfDocumentSchema } from '../schemas/swf.js';
// After
import { swfDocumentSchema } from '../schemas/swf.generated.js';
```

- [ ] **Step 5: Write staleness test**

Create `packages/lsp-schemas/test/staleness.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { readFileSync, existsSync } from 'fs';
import { execSync } from 'child_process';
import { resolve } from 'path';

const schemasDir = resolve(__dirname, '../src/schemas');

const GENERATED_FILES = [
  'case-definition.generated.ts',
  'org.generated.ts',
  'htn.generated.ts',
  'swf.generated.ts',
];

describe('schema staleness', () => {
  for (const file of GENERATED_FILES) {
    it(`${file} exists`, () => {
      expect(existsSync(resolve(schemasDir, file))).toBe(true);
    });

    it(`${file} has AUTO-GENERATED header`, () => {
      const content = readFileSync(resolve(schemasDir, file), 'utf-8');
      expect(content).toContain('AUTO-GENERATED');
    });
  }

  it('generated files are not stale', () => {
    const before = new Map<string, string>();
    for (const file of GENERATED_FILES) {
      before.set(file, readFileSync(resolve(schemasDir, file), 'utf-8'));
    }
    execSync('yarn workspace @casehubio/lsp-schemas run generate', {
      stdio: 'pipe',
    });
    for (const file of GENERATED_FILES) {
      const after = readFileSync(resolve(schemasDir, file), 'utf-8');
      expect(after).toBe(before.get(file));
    }
  });
});
```

- [ ] **Step 6: Run full test suite**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: ALL PASS — generator tests, per-format tests, multi-format tests, staleness tests

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui add packages/lsp-schemas/ packages/graph-stencil-swf/
git -C /Users/mdproctor/claude/casehub/slots/188/blocks-ui commit -m "feat(lsp-schemas): generate SWF schema + staleness tests for all formats Refs casehubio/casehub-pages#420"
```

---

## References

- [specs/issue-420-lsp-schemas/2026-09-11-lsp-domain-schema-generation-design.md] — design spec
- [packages/pages-schema/scripts/generate-schemas.ts] — component generator (template)
- [packages/pages-schema/src/generator.test.ts] — component staleness test pattern
- [packages/lsp-schemas/src/schemas/case-definition.ts] — hand-written CaseDefinition schema
- [packages/lsp-schemas/src/schemas/htn.ts] — hand-written HTN schema
- [packages/lsp-schemas/src/schemas/swf.ts] — hand-written SWF schema
- [packages/lsp-schemas/src/schemas/org.ts] — hand-written Org schema
- [packages/graph-stencil-case/src/types/generated/case-definition.ts] — CaseDefinition source types
- [packages/graph-stencil-org/src/types.ts:72] — OrgStructureYaml interface
- [packages/graph-stencil-htn/src/types/] — HTN runtime snapshot types
- [GE-20260907-156200] — ts-morph stale dist gotcha
- [GE-20260907-d0ccd0] — split tsconfig technique
- [GitHub casehubio/casehub-pages#420] — focal issue
- [GitHub casehubio/casehub-pages#411] — component schema generator (prior art)
