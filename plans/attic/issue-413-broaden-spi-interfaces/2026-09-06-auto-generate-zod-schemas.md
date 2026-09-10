# Auto-Generate Zod Schemas Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #411 — feat: auto-generate Zod schemas from TypeScript interfaces at build time
**Issue group:** #411

**Goal:** Replace hand-written Zod schemas with generated ones, fix incomplete TypeScript interfaces, and eliminate the desugarer's dynamic property passthrough.

**Architecture:** A ts-morph script reads `ComponentTypeRegistry` and its referenced interfaces from `pages-component`, generates Zod schemas into `pages-schema`, and the desugarer uses those schemas to validate properties instead of passing through arbitrary keys. The TypeScript interfaces are the single source of truth — schemas are derived, never hand-maintained.

**Tech Stack:** TypeScript, ts-morph, Zod 3.23+, Vitest

## Global Constraints

- Pre-release platform — breaking changes to interfaces and YAML behavior are acceptable
- `ts-morph` version `^24.0.0` (latest stable)
- Generated file has `// AUTO-GENERATED` header and is committed to git
- Function-typed properties are filtered out (can't appear in YAML)
- Branded types (`ColumnId`, `DataSetId`) treated as `z.string()`
- `lookupSchema` from `pages-data` used for the `lookup` field (has transforms)
- `fieldSchemaZod` (recursive JSON Schema) stays hand-written via `z.lazy()`

---

## Batch 1: Interface Audit — fix incomplete TypeScript interfaces

### Task 1: Audit and fix MetricProps and other component interfaces

**Files:**
- Modify: `packages/pages-component/src/model/displayer-types.ts` (add missing properties)
- Modify: `packages/pages-component/src/model/component-props.ts` (if gaps found)
- Test: `packages/pages-component/src/model/component-props.test.ts` (extend)

**Interfaces:**
- Consumes: existing TypeScript interfaces in `displayer-types.ts`
- Produces: complete interfaces where every YAML-usable property is declared

- [ ] **Step 1: Identify gaps in MetricProps**

MetricProps (displayer-types.ts:125-134) is missing `text` and `value` — used in every metric sample YAML. These pass through the desugarer's passthrough loop (displayer-desugar.ts:381-383) but aren't declared.

Check: `examples/samples/` for metric usage patterns:
```bash
find examples/samples -name "*.yaml" -o -name "*.yml" | xargs python3 -c "
import sys, yaml
for f in sys.argv[1:]:
    try:
        with open(f) as fh:
            for doc in yaml.safe_load_all(fh):
                if not doc: continue
                # search for type: metric components
    except: pass
" 2>/dev/null
```

Confirm `text` and `value` are the only missing properties on MetricProps.

- [ ] **Step 2: Write failing test for MetricProps text and value**

Add to `packages/pages-component/src/model/component-props.test.ts`:

```typescript
it("MetricProps has text and value", () => {
  const p: MetricProps = {
    lookup: {} as DataSetLookup,
    text: "Revenue",
    value: "$48,200",
    subtype: "card",
  };
  expect(p.text).toBe("Revenue");
  expect(p.value).toBe("$48,200");
});
```

Run: `yarn workspace @casehubio/pages-component run test`
Expected: FAIL — `text` and `value` not assignable to `MetricProps`

- [ ] **Step 3: Add text and value to MetricProps**

In `packages/pages-component/src/model/displayer-types.ts`, add to `MetricProps`:

```typescript
export interface MetricProps extends DataComponentCommon {
  readonly text?: string;
  readonly value?: string;
  readonly subtype?: "card" | "card2" | "plain-text" | "quota";
  readonly pattern?: string;
  readonly html?: {
    readonly template?: string;
    readonly javascript?: string;
  };
  readonly sparklineData?: readonly number[];
  readonly trend?: "up" | "down" | "flat";
}
```

- [ ] **Step 4: Systematic audit of all other component types**

For each component type in `ComponentTypeRegistry`, compare the interface properties against what the desugarer extracts. The desugarer explicitly extracts properties from named sections (`general`, `chart`, `table`, `meter`, `badge`, `countdown`, `timeline`, `graph`) — those are already in interfaces. Properties that pass through the passthrough loop and ARE in interfaces are fine. Properties that pass through but AREN'T in interfaces are gaps.

Check each type by reading its interface and verifying every property the YAML samples use is declared. Focus on:
- `DataTableProps` — verify `selection`, `selectionKey`, `show_column_picker` if used
- `SelectorProps` — verify completeness
- `MapProps` — verify `mapName`, `colorScheme`
- `EventTimelineProps` — verify `strategyKey`

For any gaps found, add the missing properties to the interface.

- [ ] **Step 5: Run tests and commit**

Run: `yarn workspace @casehubio/pages-component run test`
Expected: All tests PASS

```bash
git add packages/pages-component/
git commit -m "feat(#411): complete MetricProps and other component interfaces

Adds text and value to MetricProps. Audits all component interfaces
for completeness against YAML sample usage.

Refs #411"
```

---

## Batch 2: Schema Generator — ts-morph script

### Task 2: Create the generate-schemas.ts script

**Files:**
- Create: `packages/pages-schema/scripts/generate-schemas.ts`
- Modify: `packages/pages-schema/package.json` (add generate script, tsx + ts-morph deps)
- Test: `packages/pages-schema/src/generator.test.ts`

**Interfaces:**
- Consumes: `ComponentTypeRegistry` from `pages-component/src/model/type-guards.ts`, all referenced interfaces
- Produces: `packages/pages-schema/src/component-schemas.generated.ts` — replaces hand-written `component-schemas.ts`

- [ ] **Step 1: Add ts-morph and tsx dependencies**

Add to `packages/pages-schema/package.json` devDependencies:

```json
"ts-morph": "^24.0.0",
"tsx": "^4.19.0"
```

Add script:
```json
"generate": "tsx scripts/generate-schemas.ts"
```

Run: `yarn install`

- [ ] **Step 2: Write failing test for generator output**

Create `packages/pages-schema/src/generator.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { readFileSync, existsSync } from "fs";
import { resolve } from "path";

describe("schema generator", () => {
  const generatedPath = resolve(__dirname, "component-schemas.generated.ts");

  it("generated file exists", () => {
    expect(existsSync(generatedPath)).toBe(true);
  });

  it("generated file has AUTO-GENERATED header", () => {
    const content = readFileSync(generatedPath, "utf-8");
    expect(content).toContain("AUTO-GENERATED");
  });

  it("generated file exports barChartPropsSchema", () => {
    const content = readFileSync(generatedPath, "utf-8");
    expect(content).toContain("export const barChartPropsSchema");
  });

  it("generated file exports all 55 component schemas", () => {
    const content = readFileSync(generatedPath, "utf-8");
    const exportCount = (content.match(/export const \w+PropsSchema/g) || []).length;
    expect(exportCount).toBeGreaterThanOrEqual(55);
  });

  it("generated file does not contain function types", () => {
    const content = readFileSync(generatedPath, "utf-8");
    expect(content).not.toContain("cellSpan");
    expect(content).not.toContain("renderAfterHeader");
  });

  it("generated schemas parse valid data", async () => {
    const { barChartPropsSchema } = await import("./component-schemas.generated.js");
    expect(() => barChartPropsSchema.parse({
      lookup: { uuid: "ds-1" },
      subtype: "column",
    })).not.toThrow();
  });

  it("MetricProps schema includes text and value", async () => {
    const { metricPropsSchema } = await import("./component-schemas.generated.js");
    const result = metricPropsSchema.parse({
      lookup: { uuid: "ds-1" },
      text: "Revenue",
      value: "$48,200",
    });
    expect(result.text).toBe("Revenue");
  });
});
```

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: FAIL — generated file doesn't exist

- [ ] **Step 3: Implement the generator script**

Create `packages/pages-schema/scripts/generate-schemas.ts`:

```typescript
import { Project, Type, Symbol as MorphSymbol, InterfaceDeclaration } from "ts-morph";
import { writeFileSync } from "fs";
import { resolve } from "path";

const HEADER = `// AUTO-GENERATED by scripts/generate-schemas.ts — DO NOT EDIT
// Re-generate: yarn workspace @casehubio/pages-schema run generate
// Source: packages/pages-component/src/model/
import { z } from "zod";
import { lookupSchema } from "@casehubio/pages-data";
`;

const FILTERED_PROPS = new Set([
  "cellSpan", "mergeRows", "renderAfterHeader",
  "columnRenderers", "rowKey",
]);

function isFunction(type: Type): boolean {
  return type.getCallSignatures().length > 0;
}

function typeToZod(type: Type, prop: MorphSymbol): string {
  const name = prop.getName();
  if (FILTERED_PROPS.has(name)) return "";
  if (isFunction(type)) return "";

  // Branded types
  if (type.getText().includes("DataSetLookup")) return "lookupSchema";
  if (type.getText().includes("ColumnId")) return "z.string()";
  if (type.getText().includes("DataSetId")) return "z.string()";

  // Primitives
  if (type.isString() || type.isStringLiteral()) return "z.string()";
  if (type.isNumber() || type.isNumberLiteral()) return "z.number()";
  if (type.isBoolean() || type.isBooleanLiteral()) return "z.boolean()";

  // String literal union → z.enum
  if (type.isUnion()) {
    const members = type.getUnionTypes().filter(t => !t.isUndefined());
    if (members.every(m => m.isStringLiteral())) {
      const values = members.map(m => m.getLiteralValue() as string);
      return `z.enum([${values.map(v => `"${v}"`).join(", ")}])`;
    }
    if (members.length === 1) return typeToZod(members[0], prop);
    // Mixed union — fall back to z.any()
    return "z.any()";
  }

  // Array
  if (type.isArray()) {
    const elem = type.getArrayElementType();
    if (!elem) return "z.array(z.any())";
    const inner = typeToZod(elem, prop);
    return inner ? `z.array(${inner})` : "z.array(z.any())";
  }

  // Record<string, T>
  if (type.getText().startsWith("Record<") || type.getText().startsWith("Readonly<Record<")) {
    return "z.record(z.unknown())";
  }

  // Object type (inline or interface)
  if (type.isObject() && !type.isArray()) {
    const props = type.getProperties();
    if (props.length === 0) return "z.object({})";
    const fields = props
      .map(p => propToZodField(p))
      .filter(Boolean);
    return `z.object({\n    ${fields.join(",\n    ")},\n  })`;
  }

  return "z.any()";
}

function propToZodField(prop: MorphSymbol): string {
  const name = prop.getName();
  if (FILTERED_PROPS.has(name)) return "";
  
  const decl = prop.getValueDeclaration();
  if (!decl) return "";
  
  const type = prop.getTypeAtLocation(decl);
  if (isFunction(type)) return "";

  const isOptional = prop.isOptional();
  const baseType = isOptional
    ? type.getNonNullableType()
    : type;

  let zodType = typeToZod(baseType, prop);
  if (!zodType) return "";
  if (isOptional) zodType += ".optional()";

  return `${name}: ${zodType}`;
}

function generateInterface(iface: InterfaceDeclaration, allTypes: Map<string, InterfaceDeclaration>): string {
  // Get all properties including inherited
  const props = iface.getType().getProperties();
  const fields = props
    .map(p => propToZodField(p))
    .filter(Boolean);

  return `z.object({\n  ${fields.join(",\n  ")},\n})`;
}

// Main
const project = new Project({
  tsConfigFilePath: resolve(__dirname, "../../pages-component/tsconfig.json"),
});

const typeGuardsFile = project.getSourceFileOrThrow(
  resolve(__dirname, "../../pages-component/src/model/type-guards.ts")
);

const registry = typeGuardsFile.getInterfaceOrThrow("ComponentTypeRegistry");
const registryType = registry.getType();

const output: string[] = [HEADER];
const exports: string[] = [];

for (const prop of registryType.getProperties()) {
  const typeName = prop.getName();
  const camelName = typeName
    .replace(/-([a-z])/g, (_, c) => c.toUpperCase())
    + "PropsSchema";

  const decl = prop.getValueDeclaration();
  if (!decl) continue;
  const propType = prop.getTypeAtLocation(decl);

  const fields = propType.getProperties()
    .map(p => propToZodField(p))
    .filter(Boolean);

  if (fields.length === 0) {
    output.push(`export const ${camelName} = z.object({});`);
  } else {
    output.push(`export const ${camelName} = z.object({
  ${fields.join(",\n  ")},
});`);
  }
  output.push("");
  exports.push(camelName);
}

const outPath = resolve(__dirname, "../src/component-schemas.generated.ts");
writeFileSync(outPath, output.join("\n"), "utf-8");
console.log(`Generated ${exports.length} schemas to ${outPath}`);
```

- [ ] **Step 4: Run the generator**

Run: `yarn workspace @casehubio/pages-schema run generate`
Expected: Generates `component-schemas.generated.ts` with 55+ schemas

- [ ] **Step 5: Update barrel exports to use generated file**

In `packages/pages-schema/src/index.ts`, replace the import of `./component-schemas.js` with `./component-schemas.generated.js`.

- [ ] **Step 6: Update document-schema.ts imports**

In `packages/pages-schema/src/document-schema.ts` and `schema-registry.ts`, update imports from `./component-schemas.js` to `./component-schemas.generated.js`.

- [ ] **Step 7: Delete hand-written component-schemas.ts**

Delete `packages/pages-schema/src/component-schemas.ts` — replaced by the generated file.

- [ ] **Step 8: Run all tests**

Run: `yarn workspace @casehubio/pages-schema run test`
Expected: All tests PASS (including generator tests and existing schema tests)

- [ ] **Step 9: Add staleness check test**

Add to `packages/pages-schema/src/generator.test.ts`:

```typescript
import { execSync } from "child_process";

it("generated file is not stale", () => {
  const current = readFileSync(generatedPath, "utf-8");
  execSync("yarn workspace @casehubio/pages-schema run generate", { stdio: "pipe" });
  const regenerated = readFileSync(generatedPath, "utf-8");
  expect(regenerated).toBe(current);
});
```

- [ ] **Step 10: Commit**

```bash
git add packages/pages-schema/ packages/pages-component/
git commit -m "feat(#411): auto-generate Zod schemas from TypeScript interfaces

ts-morph script reads ComponentTypeRegistry, generates component-schemas.generated.ts.
Replaces hand-written schemas. Staleness test ensures generated file stays current.

Refs #411"
```

---

## Batch 3: Desugarer Validation — replace passthrough with schema.parse()

### Task 3: Replace displayer-desugar passthrough with schema validation

**Files:**
- Modify: `packages/pages-ui/src/parser/displayer-desugar.ts` (replace passthrough loop)
- Modify: `packages/pages-ui/package.json` (add pages-schema dependency)
- Test: `packages/pages-ui/src/parser/displayer-desugar.test.ts` (add validation tests)

**Interfaces:**
- Consumes: `componentSchemaRegistry` from `@casehubio/pages-schema`
- Produces: validated props on desugared components — no undeclared properties pass through

- [ ] **Step 1: Add pages-schema dependency to pages-ui**

Add to `packages/pages-ui/package.json` dependencies:

```json
"@casehubio/pages-schema": "workspace:*"
```

Run: `yarn install`

- [ ] **Step 2: Write failing test for passthrough elimination**

Add to displayer-desugar test file:

```typescript
it("rejects undeclared properties on metric", () => {
  const raw = {
    type: "metric",
    text: "Revenue",
    value: "$48,200",
    undeclaredProp: "should be stripped",
    lookup: { uuid: "ds-1" },
  };
  const result = desugarDisplayer(raw);
  expect(result.props).toBeDefined();
  expect((result.props as Record<string, unknown>).text).toBe("Revenue");
  expect((result.props as Record<string, unknown>).undeclaredProp).toBeUndefined();
});
```

Run: `yarn workspace @casehubio/pages-ui run test -- src/parser/displayer-desugar.test.ts`
Expected: FAIL — `undeclaredProp` passes through

- [ ] **Step 3: Replace passthrough loop with schema validation**

In `packages/pages-ui/src/parser/displayer-desugar.ts`:

Add import at top:
```typescript
import { componentSchemaRegistry } from "@casehubio/pages-schema";
```

Replace lines 374-385 (the passthrough loop):

```typescript
// OLD: passthrough loop
// const handledKeys = new Set([...]);
// for (const [key, value] of Object.entries(raw)) {
//   if (!handledKeys.has(key) && !(key in props)) {
//     props[key] = value;
//   }
// }

// NEW: schema validation — only declared properties survive
const schema = componentSchemaRegistry.get(type);
if (schema) {
  // Merge any remaining raw keys into props before validation
  // (handles flat-format properties like text, value, csvExport)
  const handledKeys = new Set([
    "type", "component", "general", "chart", "axis", "external", "table",
    "data-table", "meter", "badge", "countdown", "timeline", "graph",
    "subtype", "filter", "lookup", "dataSetLookup", "columns", "refresh",
    "extraConfiguration", "dataSet", "visibleWhen", "html", "properties",
  ]);
  for (const [key, value] of Object.entries(raw)) {
    if (!handledKeys.has(key) && !(key in props)) {
      props[key] = value;
    }
  }
  // Validate and strip undeclared keys
  const validated = schema.safeParse(props);
  if (validated.success) {
    Object.keys(props).forEach(k => delete props[k]);
    Object.assign(props, validated.data);
  }
}
```

Note: The passthrough loop is kept but now feeds into `schema.safeParse()` which strips unknown keys. This preserves the flat-format extraction (text, value, etc.) while ensuring only declared properties survive.

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-ui run test`
Expected: All tests PASS

- [ ] **Step 5: Update build chain**

Ensure `pages-schema` builds before `pages-ui` in root `package.json` `build:packages`. It should already be in the right position from #408.

- [ ] **Step 6: Run full build**

Run: `yarn run build:packages`
Expected: All packages build successfully

- [ ] **Step 7: Commit**

```bash
git add packages/pages-ui/ packages/pages-schema/
git commit -m "feat(#411): replace desugarer passthrough with schema validation

displayer-desugar.ts now validates props through componentSchemaRegistry.
Undeclared properties are stripped by Zod's safeParse. All YAML properties
must be declared in TypeScript interfaces to survive desugaring.

Closes #411"
```

---

## References

- [specs/issue-411-auto-generate-zod-schemas/2026-09-06-auto-generate-zod-schemas-design.md] — design spec
- [packages/pages-component/src/model/displayer-types.ts:125-134] — MetricProps (audit target)
- [packages/pages-component/src/model/type-guards.ts:64-134] — ComponentTypeRegistry (generator source)
- [packages/pages-ui/src/parser/displayer-desugar.ts:374-385] — passthrough loop (to be replaced)
- [packages/pages-ui/src/parser/component-desugar.ts] — component routing
- [packages/pages-schema/src/component-schemas.ts] — hand-written schemas (to be replaced)
- [GitHub #408] — schema infrastructure (prerequisite, landed)
- [GitHub #411] — this issue
