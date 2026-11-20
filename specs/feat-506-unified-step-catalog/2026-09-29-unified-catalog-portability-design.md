# Unified Step Catalog — Portability Model and Cross-Runtime Discovery

**Issue:** casehubio/casehub-pages#506
**Branch:** feat/506-unified-step-catalog
**Date:** 2026-09-29
**Depends on:** casehubio/platform#483 (landed)

## Context

The step catalog browser (#501) displays actions from a single runtime. Platform#483 landed a unified step runtime API with `PluginRegistry.register(Definition)` on both Java and TS sides, plus a portability model (universal/java/ts/both). This spec evolves the catalog to discover actions from both runtimes and surface portability metadata.

The TS rename and ParameterType unification (dropping Step prefix, merging LIST→ARRAY) are already committed on this branch.

## 1. Portability Field on Definition

### Type

```typescript
export type Portability = 'universal' | 'java' | 'ts' | 'both';
```

Added to `step/types.ts`. Values match the Java enum from platform#483.

### Definition interface

```typescript
export interface Definition {
  name: string;
  description?: string;
  inputs: Record<string, Parameter>;
  outputs: Record<string, Parameter>;
  invoke?: InvokeBinding;
  portability?: Portability;
}
```

`portability` is optional. Defaults are applied at registration/parse time, not at the type level.

### Default assignment

| Registration path | Default portability |
|---|---|
| `PluginRegistry.register()` | `'ts'` |
| YAML with rest/graphql/process invoke | `'universal'` |
| YAML with mcp/script/agent invoke | `'ts'` |
| YAML with explicit `portability:` key | as specified |
| Java @StepPlugin (via GraphQL) | comes from Java side |

`PluginRegistry.register()` sets `portability` to `'ts'` when unset. `DefinitionParser` infers from invoke binding kind when no explicit value is provided in the YAML.

### DefinitionParser changes

`DefinitionParser.parseDefinition()` reads an optional `portability` string from the YAML raw map. If present, validates against the four allowed values. If absent, infers from the invoke binding kind.

## 2. Catalog Data Sources for the Component

### CatalogDataSource interface

```typescript
export interface CatalogDataSource {
  fetchSummaries(): Promise<CatalogActionSummary[]>;
  fetchDetail(name: string): Promise<CatalogActionDetail | null>;
  readonly priority: number;
}
```

Three implementations:

#### RegistryCatalogSource (standalone browser)

Wraps a `PluginRegistry` instance directly. No network calls. Used when the catalog runs in standalone browser mode with no backend.

```typescript
export class RegistryCatalogSource implements CatalogDataSource {
  constructor(
    private readonly registry: PluginRegistry,
    readonly priority: number = 0,
  ) {}
  // Maps registry entries to CatalogActionSummary/Detail
}
```

#### RestCatalogSource (TS backend)

Fetches from the TS REST endpoint that serves the server-side `PluginRegistry` contents.

```typescript
export class RestCatalogSource implements CatalogDataSource {
  constructor(
    private readonly baseUrl: string,
    readonly priority: number = 10,
  ) {}
  // GET /api/catalog/actions → CatalogActionSummary[]
  // GET /api/catalog/actions/:name → CatalogActionDetail
}
```

#### GraphqlCatalogSource (Java backend)

Fetches from the Java `@McpDomain("step-catalog")` GraphQL endpoint.

```typescript
export class GraphqlCatalogSource implements CatalogDataSource {
  constructor(
    private readonly endpoint: string,
    readonly priority: number = 20,
  ) {}
  // query { stepCatalog { actions { name, description, portability, ... } } }
}
```

### REST endpoint for TS registry

New handler in `packages/pages-aria/src/server/`:

```typescript
export function createCatalogListHandler(
  catalog: Catalog,
): (req: Request) => Promise<Response>
```

Serves the TS-side `PluginRegistry` contents as JSON. Two routes:
- `GET /api/catalog/actions` — list all with summaries
- `GET /api/catalog/actions/:name` — single action detail

### CatalogActionSummary and CatalogActionDetail updates

Both gain a `portability` field:

```typescript
export interface CatalogActionSummary {
  name: string;
  description: string;
  invokeKind: string | null;
  source: string;
  portability: Portability;
  inputCount: number;
  outputCount: number;
}

export interface CatalogActionDetail {
  // ... existing fields ...
  portability: Portability;
}
```

## 3. Component: Client-Side Merge

### Merge semantics

`<pages-action-catalog>` accepts `sources: CatalogDataSource[]`. On load:

1. Fetch summaries from all sources in parallel
2. Sort sources by priority (lower = higher precedence)
3. Deduplicate by action name — first source wins (matches `CompositeCatalog` semantics)
4. Render merged list

Detail fetch: when user clicks an action, fetch from the source that provided the summary entry.

### Configuration modes

| Mode | Sources | Use case |
|---|---|---|
| Standalone | `[RegistryCatalogSource]` | Browser-only, no backend |
| TS backend | `[RestCatalogSource]` | TS server running |
| Full stack | `[RestCatalogSource, GraphqlCatalogSource]` | Both runtimes available |

The component property `sources` is set by the hosting page/controller. `PagesScenarioController` wires the appropriate sources based on available endpoints.

### Portability badges

Each action row shows a colored chip:

| Portability | Color | CSS var |
|---|---|---|
| universal | green | `--pages-success-*` |
| java | orange | `--pages-warning-*` |
| ts | blue | `--pages-accent-*` |
| both | purple | `--pages-info-*` |

New filter chip group for portability alongside the existing source filter chips.

## 4. Pre-flight Portability Validation

### validatePortability function

New export from `step/` module:

```typescript
export type RuntimeEnvironment = 'java' | 'ts';

export interface PortabilityViolation {
  actionName: string;
  actionPortability: Portability;
  runtime: RuntimeEnvironment;
  message: string;
}

export function validatePortability(
  actions: Definition[],
  runtime: RuntimeEnvironment,
): PortabilityViolation[]
```

Compatibility rules:

| Action portability | java runtime | ts runtime |
|---|---|---|
| universal | compatible | compatible |
| java | compatible | VIOLATION |
| ts | VIOLATION | compatible |
| both | compatible | compatible |

### Script-level portability

A script's portability is the most restrictive of its actions. The executor resolves all actions before running and calls `validatePortability()`. Any violation → refuse execution with a clear error listing incompatible actions. No partial execution.

### Catalog browser integration

The "Try it" panel calls `validatePortability()` before executing. If violations exist, shows them inline instead of running. The runtime environment is determined by the available catalog sources (if only TS sources → `'ts'`, if Java source present → user selects).

## Files changed

### New files
- `packages/pages-aria/src/server/catalog-list-handler.ts` — REST endpoint
- `packages/pages-aria/src/controller/catalog-data-source.ts` — CatalogDataSource interface + 3 implementations
- `packages/yaml-core/src/step/portability.ts` — Portability type, validatePortability()

### Modified files
- `packages/yaml-core/src/step/types.ts` — add Portability type, portability field on Definition
- `packages/yaml-core/src/step/definition-parser.ts` — parse portability, infer from invoke binding
- `packages/yaml-core/src/step/plugin-registry.ts` — apply default portability on register()
- `packages/yaml-core/src/step/index.ts` — re-export new types
- `packages/pages-aria/src/controller/step-catalog.ts` — CatalogDataSource integration, portability badges, filter chips
- `packages/pages-aria/src/controller/scenario-controller.ts` — wire catalog sources
- `packages/pages-aria/src/controller/index.ts` — re-export new types

## References

- casehubio/platform#483 — unified step runtime API (Java contract)
- casehubio/casehub-pages#501 — step catalog browser (initial implementation)
- `packages/yaml-core/src/step/catalog.ts` — CompositeCatalog merge semantics
- `packages/yaml-core/src/step/types.ts` — Definition interface
- `packages/yaml-core/src/step/plugin-registry.ts` — PluginRegistry.register()
- `packages/yaml-core/src/step/definition-parser.ts` — YAML definition parsing
- `packages/pages-aria/src/controller/step-catalog.ts` — PagesActionCatalog component
- `packages/pages-aria/src/server/catalog-execute-handler.ts` — existing execution handler
