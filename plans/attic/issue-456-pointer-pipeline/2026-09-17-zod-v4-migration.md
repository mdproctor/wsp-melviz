# Zod v4 Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #451 — feat: migrate to Zod v4 across pages packages
**Issue group:** #451

**Goal:** Migrate all 8 pages packages from Zod 3.x to Zod v4, updating
internal API access patterns, deprecated APIs, and the schema generator.

**Architecture:** Mechanical field-name migration of `._def` → `._zod.def`
with adjusted field names (5 renamed, 2 structural changes). All deprecated
v3 APIs replaced with v4 equivalents. Generator script updated to emit v4
code. No codemod — all manual.

**Tech Stack:** TypeScript, Zod v4, Yarn workspaces, Vitest

## Global Constraints

- Phase 1 only — pages packages. No blocks-ui or downstream changes.
- Pre-release project — no backward compat shims, move everything to v4 idioms.
- Manual migration, no codemod.
- All `z.record(val)` → `z.record(z.string(), val)`.
- All `.passthrough()` → `z.looseObject()`.
- All `.merge(other)` → `.extend(other.shape)`.

---

## Batch 1: Version Bump + Generator

### Task 1: Bump Zod to v4 and update generator script

**Files:**
- Modify: `packages/pages-builder/package.json`
- Modify: `packages/pages-code-editor/package.json`
- Modify: `packages/pages-data/package.json`
- Modify: `packages/pages-document/package.json`
- Modify: `packages/pages-lsp/package.json`
- Modify: `packages/pages-schema/package.json`
- Modify: `packages/pages-ui/package.json`
- Modify: `packages/yaml-core/package.json`
- Modify: `packages/pages-schema/scripts/generate-schemas.ts`
- Regenerate: `packages/pages-schema/src/component-schemas.generated.ts`

**Interfaces:**
- Produces: All packages on Zod v4; regenerated schemas with `z.record(z.string(), val)` and `z.looseObject()` instead of `.passthrough()`

- [ ] **Step 1: Bump Zod version in all 8 package.json files**

Change every `"zod": "^3.x"` dependency to `"zod": "^4.0.0"`.

In each of these files, replace the zod version:
- `packages/pages-builder/package.json`: `"zod": "^3.24.2"` → `"zod": "^4.0.0"`
- `packages/pages-code-editor/package.json`: `"zod": "^3.23.0"` → `"zod": "^4.0.0"`
- `packages/pages-data/package.json`: `"zod": "^3.23.0"` → `"zod": "^4.0.0"`
- `packages/pages-document/package.json`: `"zod": "^3.23.0"` → `"zod": "^4.0.0"`
- `packages/pages-lsp/package.json`: `"zod": "^3.24.4"` → `"zod": "^4.0.0"`
- `packages/pages-schema/package.json`: `"zod": "^3.23.0"` → `"zod": "^4.0.0"`
- `packages/pages-ui/package.json`: `"zod": "^3.23.0"` → `"zod": "^4.0.0"`
- `packages/yaml-core/package.json`: `"zod": "^3.23.0"` → `"zod": "^4.0.0"`

- [ ] **Step 2: Install dependencies**

Run: `yarn install`
Expected: resolves Zod v4 across all packages.

- [ ] **Step 3: Update generator script — z.record() template**

In `packages/pages-schema/scripts/generate-schemas.ts`, the `typeToZod` function
emits `z.record(...)` with a single arg (lines 71-75). Update it:

```typescript
// In the Record handling block (around line 67-76):
if (text.startsWith("Record<") || text.startsWith("Readonly<Record<")
    || text.includes("Record<string,")) {
  const typeArgs = type.getAliasTypeArguments();
  if (typeArgs.length === 2) {
    return `z.record(z.string(), ${typeToZod(typeArgs[1], depth + 1)})`;
  }
  const apparentProps = type.getStringIndexType();
  if (apparentProps) return `z.record(z.string(), ${typeToZod(apparentProps, depth + 1)})`;
  return "z.record(z.string(), z.unknown())";
}
```

- [ ] **Step 4: Update generator script — fieldSchemaBlock passthrough**

In the same file, the `fieldSchemaBlock` string template (around line 130-158)
ends with `}).passthrough()`. Replace the entire `fieldSchemaBlock` constant
to use `z.looseObject()` instead:

```typescript
const fieldSchemaBlock = `const fieldSchemaZod: z.ZodType<unknown> = z.lazy(() =>
  z.looseObject({
    type: z.union([z.string(), z.array(z.string())]).optional(),
    format: z.string().optional(),
    title: z.string().optional(),
    description: z.string().optional(),
    enum: z.array(z.string()).optional(),
    pattern: z.string().optional(),
    minimum: z.number().optional(),
    maximum: z.number().optional(),
    exclusiveMinimum: z.number().optional(),
    exclusiveMaximum: z.number().optional(),
    minLength: z.number().optional(),
    maxLength: z.number().optional(),
    minItems: z.number().optional(),
    maxItems: z.number().optional(),
    uniqueItems: z.boolean().optional(),
    multipleOf: z.number().optional(),
    readOnly: z.boolean().optional(),
    properties: z.record(z.string(), fieldSchemaZod).optional(),
    required: z.array(z.string()).optional(),
    items: fieldSchemaZod.optional(),
    const: z.union([z.string(), z.number(), z.boolean(), z.null()]).optional(),
    oneOf: z.array(fieldSchemaZod).optional(),
    $ref: z.string().optional(),
    $defs: z.record(z.string(), fieldSchemaZod).optional(),
    definitions: z.record(z.string(), fieldSchemaZod).optional(),
  }),
);
`;
```

- [ ] **Step 5: Regenerate component-schemas.generated.ts**

Run: `yarn workspace @casehubio/pages-schema run generate`
Expected: `packages/pages-schema/src/component-schemas.generated.ts` regenerated
with `z.record(z.string(), ...)` calls and `z.looseObject()` for fieldSchemaZod.

- [ ] **Step 6: Verify generated output**

Check the regenerated file:
- All `z.record(` calls should have two args: `z.record(z.string(), ...)`
- The `fieldSchemaZod` definition should use `z.looseObject(` instead of `z.object(...).passthrough()`
- No other unexpected changes

- [ ] **Step 7: Commit**

```bash
git add packages/*/package.json yarn.lock packages/pages-schema/scripts/generate-schemas.ts packages/pages-schema/src/component-schemas.generated.ts
git commit -m "feat(zod): bump to v4, update generator script and regenerate schemas

Refs #451"
```

---

## Batch 2: Schema Introspection Files

### Task 2: Update schema-navigation.ts for Zod v4

**Files:**
- Modify: `packages/pages-lsp/src/schema-navigation.ts`
- Modify: `packages/pages-lsp/src/schema-navigation.test.ts`

**Interfaces:**
- Produces: `typeName()`, `getShape()`, `unwrap()`, `navigateSchema()`, `schemaToCompletions()`, `isArrayField()` — same public signatures, updated internals

- [ ] **Step 1: Update test schemas to v4 API**

In `packages/pages-lsp/src/schema-navigation.test.ts`, update all
`z.record(val)` calls to `z.record(z.string(), val)`. Scan the file for
every `z.record(` and add the `z.string()` first argument.

- [ ] **Step 2: Run tests to confirm they fail on ._def access**

Run: `yarn workspace @casehubio/pages-lsp test -- --run schema-navigation`
Expected: FAIL — `._def` no longer exists in Zod v4.

- [ ] **Step 3: Verify v4 internal structure**

Before making changes, add a temporary console.log in the test file to
verify the v4 field names:

```typescript
const s = z.string();
console.log('string def:', JSON.stringify(s._zod.def));
const e = z.enum(["a", "b"]);
console.log('enum def:', JSON.stringify(e._zod.def));
const ne = z.nativeEnum({ A: "a", B: "b" } as const);
console.log('nativeEnum def:', JSON.stringify(ne._zod.def));
const lit = z.literal("foo");
console.log('literal def:', JSON.stringify(lit._zod.def));
```

Run the test once to capture actual field names. Remove the console.logs
after verification. Adjust the field mapping if any names differ from
the design spec.

- [ ] **Step 4: Update typeName() helper**

```typescript
function typeName(schema: z.ZodType): string {
  return (schema as any)._zod.def.type as string ?? '';
}
```

- [ ] **Step 5: Update getShape() helper**

```typescript
function getShape(schema: z.ZodType): Record<string, z.ZodType> | null {
  const def = (schema as any)._zod.def;
  if (typeof def.shape === 'function') return (def.shape as () => Record<string, z.ZodType>)();
  if (typeof def.shape === 'object' && def.shape) return def.shape as Record<string, z.ZodType>;
  return null;
}
```

- [ ] **Step 6: Update unwrap() helper**

```typescript
export function unwrap(schema: z.ZodType): z.ZodType {
  const tn = typeName(schema);
  if (tn === 'optional' || tn === 'default' || tn === 'nullable') {
    return unwrap((schema as any)._zod.def.innerType);
  }
  if (tn === 'lazy') {
    return unwrap((schema as any)._zod.def.getter());
  }
  return schema;
}
```

- [ ] **Step 7: Update navigateObjectKey() — array element field rename**

```typescript
function navigateObjectKey(shape: Record<string, z.ZodType>, key: string): z.ZodType | null {
  if (!(key in shape)) return null;
  let field = unwrap(shape[key] as z.ZodType);
  if (typeName(field) === 'array') {
    field = unwrap((field as any)._zod.def.element);
  }
  return field;
}
```

- [ ] **Step 8: Update navigateSchema() — all type name strings and field accesses**

Update every `typeName` comparison and `._def` access in `navigateSchema()`:

- `'ZodObject'` → `'object'`
- `'ZodArray'` → `'array'`, and `(current._def as { type: z.ZodType }).type` → `(current as any)._zod.def.element`
- `'ZodRecord'` → `'record'`, and `(current._def as { valueType: z.ZodType }).valueType` → `(current as any)._zod.def.valueType`
- `'ZodDiscriminatedUnion'` → remove the separate branch. Instead, check inside the `'union'` case: if `def.discriminator` exists, handle as DU.
- `'ZodUnion'` → `'union'`, but now must also handle discriminated unions here.
- `'ZodIntersection'` → `'intersection'`

For the combined union/DU handling:

```typescript
} else if (tn === 'union') {
  const def = (current as any)._zod.def;
  if (def.discriminator) {
    // Discriminated union path
    if (key === def.discriminator) {
      const literals = def.options.map((opt: z.ZodType) => {
        const optShape = getShape(unwrap(opt));
        if (!optShape || !(def.discriminator in optShape)) return null;
        const discField = unwrap(optShape[def.discriminator] as z.ZodType);
        const discDef = (discField as any)._zod.def;
        return discDef.values?.[0] ?? String(discDef.value);
      }).filter(Boolean) as string[];
      return z.enum(literals as [string, ...string[]]);
    }
    const typeValue = siblings?.[def.discriminator];
    if (!typeValue) return null;
    // Linear scan to find matching branch
    const branch = def.options.find((opt: z.ZodType) => {
      const optShape = getShape(unwrap(opt));
      if (!optShape || !(def.discriminator in optShape)) return false;
      const discField = unwrap(optShape[def.discriminator] as z.ZodType);
      const discDef = (discField as any)._zod.def;
      const litValue = discDef.values?.[0] ?? discDef.value;
      return String(litValue) === typeValue;
    });
    if (!branch) return null;
    const branchObj = unwrap(branch);
    const branchShape = getShape(branchObj);
    if (!branchShape) return null;
    const result = navigateObjectKey(branchShape, key);
    if (!result) return null;
    current = result;
  } else {
    // Regular union path
    let found: z.ZodType | null = null;
    for (const option of def.options) {
      const result = navigateSchema(option, [key], siblings);
      if (result) { found = result; break; }
    }
    if (!found) return null;
    current = found;
  }
}
```

- [ ] **Step 9: Update describeType() — type name strings and enum entries**

```typescript
function describeType(schema: z.ZodType): string | undefined {
  const tn = typeName(schema);
  if (tn === 'string') return 'string';
  if (tn === 'number') return 'number';
  if (tn === 'boolean') return 'boolean';
  if (tn === 'enum') {
    const entries = (schema as any)._zod.def.entries;
    if (Array.isArray(entries)) return entries.join(' | ');
    return Object.values(entries).filter((v: unknown) => typeof v === 'string').join(' | ');
  }
  if (tn === 'array') return 'array';
  if (tn === 'object') return 'object';
  if (tn === 'record') return 'record';
  return undefined;
}
```

- [ ] **Step 10: Update schemaToCompletions() — all type names and field accesses**

Update every branch in `schemaToCompletions()`:

- `'ZodObject'` → `'object'`
- `'ZodEnum'` → `'enum'` with `._zod.def.entries` (handle both array and object)
- `'ZodNativeEnum'` → merged into `'enum'` case (same type tag in v4)
- `'ZodLiteral'` → `'literal'` with `._zod.def.values[0]` (array in v4)
- `'ZodBoolean'` → `'boolean'`
- `'ZodDiscriminatedUnion'` → merged into `'union'` case (check `def.discriminator`)
- `'ZodUnion'` → `'union'` (handle both regular and DU)
- `'ZodRecord'` → `'record'` with `._zod.def.valueType`
- `'ZodIntersection'` → `'intersection'` with `._zod.def.left` / `._zod.def.right`

For ZodEnum/ZodNativeEnum merge:
```typescript
if (tn === 'enum') {
  const entries = (unwrapped as any)._zod.def.entries;
  if (Array.isArray(entries)) {
    return entries.map((v: string) => ({ label: v, type: 'enum' as const }));
  }
  // Native enum — entries is an object
  const values = Object.values(entries).filter((v: unknown): v is string => typeof v === 'string');
  return values.map((v) => ({ label: v, type: 'enum' as const }));
}
```

For ZodLiteral:
```typescript
if (tn === 'literal') {
  const values = (unwrapped as any)._zod.def.values as unknown[];
  return [{
    label: String(values[0]),
    type: 'enum' as const,
  }];
}
```

For combined union/DU in completions:
```typescript
if (tn === 'union') {
  const def = (unwrapped as any)._zod.def;
  if (def.discriminator) {
    // Discriminated union completions
    const discKey = def.discriminator;
    const typeValues = def.options.map((opt: z.ZodType) => {
      const optShape = getShape(unwrap(opt));
      if (!optShape || !(discKey in optShape)) return null;
      const discField = unwrap(optShape[discKey] as z.ZodType);
      const discDef = (discField as any)._zod.def;
      return discDef.values?.[0] ?? String(discDef.value);
    }).filter(Boolean) as string[];
    const firstBranch = def.options[0];
    const branchCompletions = firstBranch ? schemaToCompletions(firstBranch) : [];
    const commonKeys = branchCompletions.filter((c: CompletionEntry) => c.label !== discKey);
    return [
      {
        label: discKey,
        detail: typeValues.join(' | '),
        type: 'property' as const,
        apply: discKey + ': ',
      },
      ...commonKeys,
    ];
  }
  // Regular union
  const options = def.options as z.ZodType[];
  if (siblings && Object.keys(siblings).length > 0) {
    const siblingKeys = new Set(Object.keys(siblings));
    const matching = options.filter((opt: z.ZodType) => {
      const optShape = getShape(unwrap(opt));
      if (!optShape) return false;
      return [...siblingKeys].some(k => k in optShape);
    });
    if (matching.length === 1) {
      return schemaToCompletions(matching[0]!, siblings);
    }
  }
  const allCompletions: CompletionEntry[] = [];
  for (const option of options) {
    allCompletions.push(...schemaToCompletions(option, siblings));
  }
  const seen = new Set<string>();
  return allCompletions.filter((c) => {
    if (seen.has(c.label)) return false;
    seen.add(c.label);
    return true;
  });
}
```

- [ ] **Step 11: Run tests**

Run: `yarn workspace @casehubio/pages-lsp test -- --run schema-navigation`
Expected: all tests PASS. If any fail, debug using the field mapping table
and the console.log output from Step 3.

- [ ] **Step 12: Commit**

```bash
git add packages/pages-lsp/src/schema-navigation.ts packages/pages-lsp/src/schema-navigation.test.ts
git commit -m "feat(zod): migrate schema-navigation.ts to v4 internal API

Update ._def → ._zod.def with field name adjustments:
- typeName → def.type (lowercase values)
- ._def.type (array) → def.element
- ._def.values (enum) → def.entries
- ._def.value (literal) → def.values (array)
- ._def.optionsMap → linear scan of def.options
- Merge ZodDiscriminatedUnion into union case (shared type tag)
- Merge ZodNativeEnum into enum case (shared type tag)

Refs #451"
```

### Task 3: Update zod-to-fieldschema.ts for Zod v4

**Files:**
- Modify: `packages/pages-document/src/zod-to-fieldschema.ts`
- Modify: `packages/pages-document/src/zod-to-fieldschema.test.ts`

**Interfaces:**
- Consumes: Same `z.ZodType` schemas from pages-schema (now v4)
- Produces: `zodToFieldSchema(schema: z.ZodType): FieldSchema` — same signature

- [ ] **Step 1: Update test schemas to v4 API**

In `packages/pages-document/src/zod-to-fieldschema.test.ts`, update any
`z.record(val)` calls to `z.record(z.string(), val)`.

- [ ] **Step 2: Run tests to confirm they fail**

Run: `yarn workspace @casehubio/pages-document test -- --run zod-to-fieldschema`
Expected: FAIL — `._def` no longer exists.

- [ ] **Step 3: Update typeName(), getShape(), unwrap() helpers**

Same pattern as schema-navigation.ts — update to use `._zod.def` with v4
field names. These are duplicated functions with the same v3 patterns:

```typescript
function typeName(schema: z.ZodType): string {
  return (schema as any)._zod.def.type as string ?? '';
}

function unwrap(schema: z.ZodType, seen = new WeakSet<z.ZodType>()): z.ZodType {
  if (seen.has(schema)) return schema;
  seen.add(schema);
  const tn = typeName(schema);
  if (tn === 'optional' || tn === 'default' || tn === 'nullable') {
    return unwrap((schema as any)._zod.def.innerType, seen);
  }
  if (tn === 'lazy') {
    return unwrap((schema as any)._zod.def.getter(), seen);
  }
  return schema;
}

function getShape(schema: z.ZodType): Record<string, z.ZodType> | null {
  const def = (schema as any)._zod.def;
  if (typeof def.shape === 'function') return (def.shape as () => Record<string, z.ZodType>)();
  if (typeof def.shape === 'object' && def.shape) return def.shape as Record<string, z.ZodType>;
  return null;
}
```

- [ ] **Step 4: Update isOptional()**

```typescript
function isOptional(schema: z.ZodType): boolean {
  const tn = typeName(schema);
  return tn === 'optional' || tn === 'default' || tn === 'nullable';
}
```

- [ ] **Step 5: Update convertInner() — all type name strings and field accesses**

Update every branch:
- `'ZodString'` → `'string'`
- `'ZodNumber'` → `'number'`, with `(unwrapped as any)._zod.def.checks`
- `'ZodBoolean'` → `'boolean'`
- `'ZodEnum'` → `'enum'`, with `(unwrapped as any)._zod.def.entries` (handle both array and object for native enum)
- `'ZodLiteral'` → `'literal'`, with `(unwrapped as any)._zod.def.values[0]`
- `'ZodArray'` → `'array'`, with `(unwrapped as any)._zod.def.element`
- `'ZodRecord'` → `'record'`
- `'ZodObject'` → `'object'`
- `'ZodUnion'` → `'union'`, with `(unwrapped as any)._zod.def.options`
- `'ZodIntersection'` → `'intersection'`, with `._zod.def.left` / `._zod.def.right`
- `'ZodNativeEnum'` → merge into `'enum'` case
- `'ZodAny'` / `'ZodUnknown'` → `'any'` / `'unknown'`

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/pages-document test -- --run zod-to-fieldschema`
Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/pages-document/src/zod-to-fieldschema.ts packages/pages-document/src/zod-to-fieldschema.test.ts
git commit -m "feat(zod): migrate zod-to-fieldschema.ts to v4 internal API

Refs #451"
```

### Task 4: Update hover.ts for Zod v4

**Files:**
- Modify: `packages/pages-lsp/src/hover.ts`
- Modify: `packages/pages-lsp/src/hover.test.ts`

**Interfaces:**
- Consumes: `buildYamlContext`, `navigateSchema`, `unwrap` from schema-navigation (already v4)
- Produces: `handleHover()` — same signature

- [ ] **Step 1: Update test schemas to v4 API**

In `packages/pages-lsp/src/hover.test.ts`, update any `z.record(val)` calls
to `z.record(z.string(), val)`.

- [ ] **Step 2: Update typeName() and describeSchema() in hover.ts**

Only 2 `._def` accesses:

```typescript
function typeName(schema: z.ZodType): string {
  return (schema as any)._zod.def.type as string ?? '';
}

function describeSchema(schema: z.ZodType): string {
  const unwrapped = unwrap(schema);
  const tn = typeName(unwrapped);
  const desc = schema.description;
  const parts: string[] = [];
  if (desc) parts.push(desc);
  if (tn === 'string') parts.push('Type: `string`');
  else if (tn === 'number') parts.push('Type: `number`');
  else if (tn === 'boolean') parts.push('Type: `boolean`');
  else if (tn === 'enum') {
    const entries = (unwrapped as any)._zod.def.entries;
    const values = Array.isArray(entries)
      ? entries
      : Object.values(entries).filter((v: unknown) => typeof v === 'string');
    parts.push('Values: ' + (values as string[]).map(v => '`' + v + '`').join(', '));
  }
  else if (tn === 'array') parts.push('Type: `array`');
  else if (tn === 'object') parts.push('Type: `object`');
  return parts.join('\n\n') || 'No description available';
}
```

- [ ] **Step 3: Run tests**

Run: `yarn workspace @casehubio/pages-lsp test -- --run hover`
Expected: all tests PASS.

- [ ] **Step 4: Commit**

```bash
git add packages/pages-lsp/src/hover.ts packages/pages-lsp/src/hover.test.ts
git commit -m "feat(zod): migrate hover.ts to v4 internal API

Refs #451"
```

---

## Batch 3: Mechanical API Updates + Full Verification

### Task 5: Fix z.record() calls across all source files

**Files:**
- Modify: `packages/pages-schema/src/document-schema.ts`
- Modify: `packages/pages-schema/src/base-schemas.ts`
- Modify: `packages/pages-schema/src/component-schemas.ts`
- Modify: `packages/pages-data/src/dataset/external/schema.ts`
- Modify: `packages/pages-ui/src/parser/page-schema.ts`
- Modify: `packages/yaml-core/src/schema.ts`

**Interfaces:**
- Produces: All `z.record()` calls use the two-arg form

- [ ] **Step 1: Update document-schema.ts**

Replace every `z.record(x)` with `z.record(z.string(), x)`:
- Line 30: `z.record(z.string())` → `z.record(z.string(), z.string())`
- Line 109: `z.record(z.unknown())` → `z.record(z.string(), z.unknown())`
- Line 114: `z.record(z.unknown())` → `z.record(z.string(), z.unknown())`
- Line 122: `z.record(z.string())` → `z.record(z.string(), z.string())`
- Line 129: `z.record(z.string())` → `z.record(z.string(), z.string())`

- [ ] **Step 2: Update base-schemas.ts**

- Line 6: `z.record(z.string())` → `z.record(z.string(), z.string())`
- Line 64: `z.record(z.unknown())` → `z.record(z.string(), z.unknown())`

- [ ] **Step 3: Update component-schemas.ts**

All `z.record(x)` calls → `z.record(z.string(), x)`. This file has 14
occurrences — update all of them.

- [ ] **Step 4: Update pages-data external/schema.ts**

- Line 33: `z.record(z.string())` → `z.record(z.string(), z.string())`
- Line 34: `z.record(z.string())` → `z.record(z.string(), z.string())`
- Line 35: `z.record(z.string())` → `z.record(z.string(), z.string())`

- [ ] **Step 5: Update page-schema.ts**

- Line 14: `z.record(z.unknown())` → `z.record(z.string(), z.unknown())`
- Line 15: `z.record(z.string())` → `z.record(z.string(), z.string())`

- [ ] **Step 6: Update yaml-core/schema.ts**

All `z.record(x)` calls → `z.record(z.string(), x)`:
- Line 27, 51, 52, 53, 58, 59, 61, 62

- [ ] **Step 7: Commit**

```bash
git add packages/pages-schema/src/document-schema.ts packages/pages-schema/src/base-schemas.ts packages/pages-schema/src/component-schemas.ts packages/pages-data/src/dataset/external/schema.ts packages/pages-ui/src/parser/page-schema.ts packages/yaml-core/src/schema.ts
git commit -m "feat(zod): update z.record() to two-arg form across all packages

Refs #451"
```

### Task 6: Clean up deprecated APIs and update test files

**Files:**
- Modify: `packages/pages-schema/src/component-schemas.ts` (`.merge()`, `.passthrough()`)
- Modify: `packages/pages-schema/src/generator.test.ts` (`.strict()`)
- Modify: `packages/pages-lsp/src/completion.test.ts` (`z.record()`)
- Modify: `packages/pages-code-editor/src/schema-completion.test.ts` (`z.record()`)
- Modify: `packages/pages-lsp/src/schema-registry.test.ts` (`z.record()`)
- Modify: `packages/pages-lsp/src/diagnostics.test.ts` (`z.record()`)
- Modify: `packages/pages-lsp/src/server.test.ts` (`z.record()`)
- Modify: `packages/pages-data/src/dataset/lookup-parser.test.ts`
- Modify: `packages/pages-data/src/dataset/schema-export.test.ts`
- Modify: `packages/pages-schema/src/base-schemas.test.ts`

**Interfaces:**
- Produces: All deprecated APIs replaced; all test files use v4 API

- [ ] **Step 1: Clean up .merge() in component-schemas.ts**

Replace all 5 `.merge()` calls with `.extend()`:

```typescript
// Line 21: const chartDataBase = dataComponentCommonSchema.merge(chartSettingsSchema);
const chartDataBase = dataComponentCommonSchema.extend(chartSettingsSchema.shape);

// Lines 106, 116, 134, 141: same pattern
// e.g. dataComponentCommonSchema.merge(chartSettingsSchema).extend({...})
// → dataComponentCommonSchema.extend(chartSettingsSchema.shape).extend({...})
```

- [ ] **Step 2: Clean up .passthrough() in component-schemas.ts**

Replace any `.passthrough()` calls with `z.looseObject()`:
- Line 164: `z.object({ mode: z.string() }).passthrough()` → `z.looseObject({ mode: z.string() })`
- Line 172: `z.object({ fn: z.string() }).passthrough()` → `z.looseObject({ fn: z.string() })`
- Line 285: `}).passthrough().optional()` → use `z.looseObject({...}).optional()`
- Line 342: `}).passthrough()` → `z.looseObject({...})`

- [ ] **Step 3: Clean up .strict() in generator.test.ts**

```typescript
// Line 69: const result = metricPropsSchema.strict().safeParse({
// → use z.strictObject or just remove .strict() since we're testing parse not strictness
const result = metricPropsSchema.safeParse({
```

- [ ] **Step 4: Update all remaining test files with z.record() two-arg**

For each test file, replace every `z.record(val)` with `z.record(z.string(), val)`:
- `packages/pages-lsp/src/completion.test.ts`
- `packages/pages-code-editor/src/schema-completion.test.ts`
- `packages/pages-lsp/src/schema-registry.test.ts`
- `packages/pages-lsp/src/diagnostics.test.ts`
- `packages/pages-lsp/src/server.test.ts`
- `packages/pages-data/src/dataset/lookup-parser.test.ts`
- `packages/pages-data/src/dataset/schema-export.test.ts`
- `packages/pages-schema/src/base-schemas.test.ts`

- [ ] **Step 5: Run full test suite**

Run: `yarn test`
Expected: all tests PASS across all packages.

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: no type errors.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(zod): clean up deprecated APIs and update test files for v4

- .merge() → .extend(other.shape)
- .passthrough() → z.looseObject()
- .strict() → removed from test
- z.record() → two-arg form in all test files

Refs #451"
```

---

## References

- [2026-09-17-zod-v4-migration-design.md](/Users/mdproctor/claude/public/casehub/pages/specs/issue-451-zod-v4-migration/2026-09-17-zod-v4-migration-design.md) — design spec
- `packages/pages-lsp/src/schema-navigation.ts` — primary introspection (19 `._def` accesses)
- `packages/pages-document/src/zod-to-fieldschema.ts` — FieldSchema converter (10 accesses)
- `packages/pages-lsp/src/hover.ts` — hover provider (2 accesses)
- `packages/pages-schema/scripts/generate-schemas.ts` — schema code generator
- `packages/pages-schema/src/document-schema.ts` — discriminated union usage
- [Zod v4 migration guide](https://zod.dev/v4/changelog)
- [Zod v4 core docs](https://zod.dev/packages/core)
- GitHub #451
