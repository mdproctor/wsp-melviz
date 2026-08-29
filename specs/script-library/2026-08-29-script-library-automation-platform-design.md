# Script Library & Automation Platform

> **Epic:** casehubio/parent#408 — Cross-Platform Scenario Engine
> **Date:** 2026-08-29
> **Status:** Draft

## 1. Overview

The scenario engine currently handles scripted demos — a single scenario
file drives browser automation and service orchestration via the
distributed executor protocol (casehubio/parent#418). Teams now want to
build operational automations on the same engine: onboarding workflows,
data imports, environment setup.

This spec extends the scenario engine into an automation platform:

| Capability | What it adds |
|---|---|
| Script library | Browsable, filterable catalog from multiple sources |
| Parameterizable execution | Typed params, chained resolver |
| Data-driven iteration | `forEach` loops over CSV with typed columns |
| Script composability | `call` command with acyclic enforcement |
| Labels and tags | CaseHub typed labels + free-form tags |
| DOM readiness detection | First-step ARIA probe |
| Paste/upload | Quick import with metadata editing |
| Library browser | Integrated into scenario controller |

Full backward compatibility with existing scenario files and the
distributed executor protocol.

## 2. Script Format Extensions

### 2.1 Script metadata block

Every script gains an optional `meta` block alongside the existing
`scenario` field:

```yaml
scenario: onboard-team-members
meta:
  description: "Onboard new team members with accounts and role assignments"
  labels:
    - domain:hr
    - capability:onboarding
  tags:
    - getting-started
    - team-setup
```

Scripts without `meta` are valid — they appear in the library with name
only (from the `scenario` field). Metadata can be added or edited after
upload.

### 2.2 Parameter declaration

Top-level `params` block declares what a script accepts. Uses a JSON
Schema subset:

```yaml
scenario: create-project
params:
  - name: projectName
    type: string
    required: true
  - name: template
    type: string
    default: "blank"
    enum: [blank, starter, enterprise]
  - name: teamSize
    type: integer
    default: 5
  - name: enableCI
    type: boolean
    default: true
```

Supported types: `string`, `integer`, `boolean`, `decimal`.

Parameters are referenced in step values via `${params.projectName}`.
The `${params.*}` prefix is an alias for `${var.*}` in the shared YAML
core's variable resolver — both resolve through the same chain.

### 2.3 Variable resolution chain

Resolution order (most specific wins):

1. **Caller-supplied parameters** — explicit values passed by the
   invoking script's `call` command or the operator's run action
2. **User/team preferences** — platform preferences API (out of scope
   for this spec; the resolver accepts a `VariableSource` for it)
3. **Script defaults** — `default` values from the `params` declaration
4. **Quarkus MicroProfile Config** — application.properties, env vars

When script A calls script B, B inherits A's full resolved context.
Explicit params in the `call` command override inherited values. B's
own defaults apply only if no value was resolved from the chain.

### 2.4 Data-driven iteration — forEach and when

Control flow constructs align with desired-state YAML via the shared
platform core (`casehub-yaml-core`, D3).

#### forEach on steps

```yaml
data:
  team-members:
    source: members.csv

steps:
  - label: "Create member account"
    forEach:
      as: member
      in: team-members
    target: browser
    commands:
      - action: fill
        target: {role: textbox, name: "Full Name"}
        value: "${each.member.name}"
      - action: fill
        target: {role: textbox, name: "Email"}
        value: "${each.member.email}"
      - action: select
        target: {role: combobox, name: "Role"}
        value: "${each.member.role}"
      - action: click
        target: {role: button, name: "Create"}
```

The step is stamped once per CSV row. Each stamped step gets a unique
name: `create-member-account.alice-chen` (original label slugified +
first column value). The `${each.member.*}` references resolve to the
current row's column values.

#### CSV data sources

The `data` block declares named data sources. This coexists with the
existing `data` section in the executor protocol spec (§3.6), which
holds lookup maps. Disambiguation: if a `data` entry has `source` or
`inline`, it is a CSV data source; otherwise it is a lookup map.

Two forms:

**File reference:**
```yaml
data:
  team-members:
    source: members.csv
```

The file is resolved relative to the script's location (bundled scripts:
classpath, uploaded: server filesystem, registry: relative to
`contentUrl`).

**Inline CSV:**
```yaml
data:
  environments:
    inline: |
      name:string,region:string,tier:integer,production:boolean
      staging,us-east,1,false
      production,eu-west,2,true
```

CSV header format: `columnName:type` pairs. Supported types: `string`,
`integer`, `boolean`, `decimal`. Type enforcement at parse/load time —
every cell is validated against its declared column type. Type errors
report row number, column name, expected type, and actual value.

#### Named iteration groups

For simple list iteration (no CSV), use the `iterations` block:

```yaml
iterations:
  regions:
    as: region
    in: ["us-east", "eu-west", "ap-south"]
```

Steps reference by name: `forEach: {as: region, in: regions}` or
shorthand `forEach: regions` when the step doesn't need to override `as`.

#### forEach + trigger resolution

When a forEach-expanded step has a `trigger: {after: "step-name"}`,
trigger references are resolved using the same-group pairing rule from
desired-state's dependency alignment: if both the triggering and
triggered steps share the same iteration group, triggers pair by value
(`step.us-east` triggers `other.us-east`). If the referenced step is
not in a forEach group, all stamped copies trigger from the same fixed
step.

#### Conditional steps — when

```yaml
- label: "Enable CI pipeline"
  when: "${params.enableCI}"
  target: browser
  commands:
    - action: click
      target: {role: button, name: "Enable CI"}
```

Truthiness evaluation: `true`, `yes`, `on`, `y`, `1` → include.
`false`, `no`, `off`, `n`, `0` → skip. Case-insensitive.

`when` + `forEach`: condition is evaluated per stamped copy, after
`${each.*}` resolution. This allows conditional exclusion to vary per
iteration value:

```yaml
- label: "Grant admin access"
  forEach: {as: member, in: team-members}
  when: "${each.member.admin}"
  commands:
    - action: click
      target: {role: button, name: "Grant Admin"}
```

### 2.5 Script composability — call command

A step can invoke another script from the library:

```yaml
steps:
  - label: "Set up each team member"
    forEach: {as: member, in: team-members}
    commands:
      - action: call
        script: create-user-account
        params:
          name: "${each.member.name}"
          email: "${each.member.email}"
          role: "${each.member.role}"

  - label: "Configure project settings"
    commands:
      - action: call
        script: configure-project
        params:
          name: "${params.projectName}"
```

#### Resolution and inlining

The orchestrator resolves the script name against the `ScriptRegistry`
(D1). The callee's steps are inlined into the execution plan at the
call site. Step names are prefixed: `create-user-account.fill-name`,
`create-user-account.click-submit`.

The callee inherits the caller's full resolved context. Explicit params
in the `call` command override inherited values. The callee's own
`params` defaults apply only if no value was resolved.

Results from callee steps are available to subsequent caller steps via
`${create-user-account.step-name.field}`.

Inlined callee steps retain their `target` field and participate in
the orchestrator's normal dispatch grouping (§4.9 of the executor
protocol spec). A callee with mixed targets (browser + service) is
split across dispatch sequences like any other multi-target section.

#### Acyclic enforcement

At parse/load time, the system builds a call graph by resolving all
`call` references transitively through the registry. If a cycle is
detected (A → B → C → A), the script is rejected with an error naming
the cycle path. This applies to:

- Scripts loaded from any source (bundled, uploaded, registry)
- Paste/upload — rejected before saving if a cycle would result
- Registry sync — scripts with unresolvable `call` references are
  flagged but not rejected (the target may not be in this registry)

## 3. Script Registry

### 3.1 Architecture

`ScriptRegistry` is a CDI bean that aggregates scripts from three
source types:

| Source | Discovery | Mutability |
|---|---|---|
| Bundled | Classpath scan (`META-INF/scenarios/*.yaml`) | Read-only |
| Uploaded | Server filesystem (`${scenario.library.path}`) | Read-write |
| External | JSON manifest from configured registry URLs | Read-only |

Each source contributes `ScriptDescriptor` entries.

### 3.2 ScriptDescriptor

```java
public record ScriptDescriptor(
    String name,
    String description,
    List<String> labels,        // typed: "domain:helpdesk"
    List<String> tags,          // free-form: "getting-started"
    List<ParamDescriptor> params,
    List<String> calls,         // scripts this one calls
    ScriptProvenance provenance,
    List<AriaTarget> firstStepTargets  // ARIA targets from first step, for client-side readiness probe
) {}

public record ParamDescriptor(
    String name, String type,
    boolean required, Object defaultValue,
    List<Object> enumValues
) {}

public enum ScriptProvenance { BUNDLED, UPLOADED, EXTERNAL }
```

### 3.3 REST endpoints

```
GET    /scenario/library              → ScriptDescriptor[]
GET    /scenario/library?labels=domain:hr&tags=onboarding  → filtered
GET    /scenario/library/{name}       → ScriptDescriptor (single)
GET    /scenario/library/{name}/yaml  → raw YAML content
POST   /scenario/library              → upload YAML, returns ScriptDescriptor
PUT    /scenario/library/{name}/meta  → update metadata (description, labels, tags)
DELETE /scenario/library/{name}       → delete uploaded script (bundled/external rejected)
```

Readiness probing is client-side: `ScriptDescriptor.firstStepTargets`
contains the ARIA targets from the first step (extracted at parse time
by the server). The browser's library view runs `findByRole()` against
these targets to compute green/amber/red status locally.

GraphQL equivalents follow the existing controller API pattern (D13):

```graphql
type Query {
    scriptLibrary(labels: [String], tags: [String]): [ScriptDescriptor!]!
    scriptDescriptor(name: String!): ScriptDescriptor
    scriptYaml(name: String!): String
}

type Mutation {
    uploadScript(yaml: String!): ScriptDescriptor!
    updateScriptMeta(name: String!, description: String,
                     labels: [String], tags: [String]): ScriptDescriptor!
    deleteScript(name: String!): Boolean!
}
```

### 3.4 External registry manifest

Configured via:
```properties
casehub.scenario.registries[0].url=https://scripts.example.com/manifest.json
casehub.scenario.registries[0].name=company-scripts
```

The manifest URL returns a JSON array:

```json
[
  {
    "name": "onboard-team",
    "description": "Onboard team members",
    "labels": ["domain:hr", "capability:onboarding"],
    "tags": ["getting-started"],
    "params": [
      {"name": "teamName", "type": "string", "required": true}
    ],
    "contentUrl": "./scripts/onboard-team.yaml",
    "calls": ["create-user", "assign-role"]
  }
]
```

`contentUrl` is resolved relative to the manifest URL. The server
fetches the manifest on first library browse, caches with configurable
TTL (`casehub.scenario.registry.cache-ttl`, default 5 minutes).

### 3.5 Name collision rules

- Uploaded scripts can overwrite other uploaded scripts (same name =
  update).
- Bundled scripts are protected — upload with a bundled name is
  rejected with an error naming the collision.
- External registry scripts are read-only.
- The library browser shows provenance per script (bundled / uploaded /
  registry-name).

## 4. Library Browser

### 4.1 Integration into scenario controller

The library browser is a new view mode in `<scenario-controller>`. A
library icon in the header toggles between outline view (existing) and
library view (new).

### 4.2 Library view layout

```
┌─────────────────────────────────┐
│ ☰  Library        📋 [Upload]  │  ← header with paste/upload button
├─────────────────────────────────┤
│ 🔍 Search...                   │  ← text search across name + description
│ Labels: [domain:hr ×] [+]      │  ← label filter chips
│ Tags: [getting-started ×] [+]  │  ← tag filter chips
├─────────────────────────────────┤
│ 🟢 onboard-team     [▶ Run]   │  ← green = ready
│    Onboard team members         │
│    domain:hr  getting-started   │
│                                 │
│ 🟡 configure-project [▶ Run]  │  ← amber = unknown
│    Set up project structure     │
│    capability:setup             │
│                                 │
│ 🔴 resolve-ticket   [▶ Run]   │  ← red = not ready
│    Close a support ticket       │
│    domain:helpdesk              │
└─────────────────────────────────┘
```

### 4.3 Readiness indicators

| Color | Meaning | Condition |
|---|---|---|
| Green | Ready to run | First step's ARIA targets found in DOM |
| Amber | Unknown | First step has no ARIA targets (navigate, call, service step) |
| Red | Not ready | First step's ARIA targets not found in DOM |

Probes run on library view open and on page navigation events
(`popstate`, `hashchange`). Uses existing `findByRole()` from the
tree walker. The controller debounces probes to avoid flooding during
rapid navigation.

### 4.4 Run action

Clicking "Run" on a script:
1. If the script has required params without defaults, show a parameter
   form (rendered from the `params` schema)
2. Load the YAML from the registry
3. Switch to outline view
4. Start the scenario via `POST /scenario/start`

### 4.5 Paste/upload

The upload button in the library header accepts:
- **Paste** — opens a text area for pasting YAML content
- **File upload** — standard file picker for `.yaml` / `.yml` files

On submit:
1. Parse the YAML, extract `scenario` field as the name
2. Validate: syntax, param extraction, `call` references (acyclic check)
3. Save to server via `POST /scenario/library`
4. Script appears in the library immediately

Metadata editing (description, labels, tags) is available from a
context menu on each script in the library view. Opens an inline
editor that PUTs to `/scenario/library/{name}/meta`.

## 5. Scenario YAML — Complete Schema

Combining existing format (executor protocol spec) with new extensions:

```yaml
# --- Identity ---
scenario: help-desk-demo              # required, unique name
meta:                                  # optional
  description: "IT help desk demo"
  labels: [domain:helpdesk]
  tags: [demo, getting-started]

# --- Parameters ---
params:                                # optional
  - name: agentName
    type: string
    default: "Alice"

# --- Data sources ---
data:                                  # optional
  tickets:
    source: tickets.csv
  priorities:
    inline: |
      name:string,level:integer
      low,1
      medium,2
      high,3

# --- Iteration groups ---
iterations:                            # optional
  regions:
    as: region
    in: ["us-east", "eu-west"]

# --- Execution ---
speed: 1.0                             # optional, default 1.0
on-error: pause                        # optional: pause | continue | stop

# --- Steps (flat) ---
steps:
  - name: step-id                      # optional, for trigger/result references
    label: "Human-readable label"      # required
    target: browser                    # which executor
    actor: demo-admin                  # optional identity
    trigger:                           # optional scheduling
      after: previous-step
      delay: 2000
    forEach:                           # optional iteration
      as: ticket
      in: tickets
    when: "${each.ticket.active}"     # optional conditional (boolean column)
    commands:                          # one or more commands
      - action: click
        target: {role: button, name: Submit}
      - action: call
        script: verify-ticket
        params: {ticketId: "${each.ticket.id}"}

# --- Or hierarchical ---
chapters:
  - label: "Chapter 1"
    sections:
      - label: "Section 1"
        steps:
          - label: "Step 1"
            target: browser
            forEach: regions
            commands:
              - action: navigate
                value: "#dashboard/${each.region}"
```

## 6. Compilation Pipeline

When a script is loaded (for execution or call-graph validation):

```
1. Parse YAML
2. Extract params schema → validate caller-supplied values
3. Resolve variables: ${params.*} and ${var.*} via chained resolver
4. Load data sources (CSV files, inline CSV) → parse and type-check
5. Expand forEach → stamp steps per iteration value
6. Evaluate when → exclude conditional steps
7. Build call graph → resolve all call references transitively
8. Validate acyclicity → reject if call graph has cycles
9. Inline callee steps → prefix names, merge into execution plan
10. Flatten to execution plan → dispatch via orchestrator
```

Steps 2-6 use shared primitives from `casehub-platform-yaml-core`:
- Step 3: `VariableResolver` with `VariableSource.chain()` wiring the
  caller > preferences > defaults > config resolution order
- Step 4: `CsvParser.parse()` → `CsvDataSource` with typed columns
- Steps 5-6: `ForEachExpander.expand()` with a `ScenarioStepAdapter`
  implementing `ForEachAdapter<ScenarioStep>`. The expander takes
  `Map<String, E>` — the adapter converts the ordered step list to a
  `LinkedHashMap` (keyed by step name/label) for expansion and back
  to a list afterward, preserving step order. `when` evaluation uses
  `Truthiness.isTruthy()` internally via the expander.

Steps 7-9 are scenario-specific (the registry and call graph are
scenario concepts). Acyclicity is validated before inlining (step 8
before step 9) — a cycle in the call graph would cause infinite
expansion if inlined first.

## 7. Relationship to Existing Architecture

### 7.1 What changes

| Component | Current | After |
|---|---|---|
| Scenario format | `scenario` + `steps` or `chapters` | + `meta`, `params`, `data`, `iterations`, `forEach`, `when`, `call` |
| ScenarioParser (Java) | Parses flat + hierarchical | + param extraction, data block, forEach/when expansion, call resolution |
| scenario-handler.ts | Executes ARIA commands | Unchanged — call/forEach/when are resolved server-side before dispatch |
| ScenarioControlResource | `/scenario/start`, `/pause`, etc. | + `/scenario/library/*` endpoints |
| `<scenario-controller>` | Outline + transport + compact | + library view, paste/upload, readiness indicators |
| PushRequest/PushMessage | Existing ops | Unchanged — library is REST/GraphQL, not push wire |

### 7.2 What stays

- Distributed executor protocol (dispatch-sequence, executor-control,
  step-result, executor-register)
- Push wire transport for executor communication
- ARIA command execution (click, fill, navigate, assert, wait)
- Visual feedback (highlights, typing animation)
- YAML viewer with syntax highlighting
- Compact/full controller modes
- Narrative component

### 7.3 New dependencies

| Module | Depends on | Why |
|---|---|---|
| `casehub-pages-scenario` | `casehub-platform-yaml-core` | Shared forEach, when, variable resolver, CSV parser |
| `casehub-pages-scenario` | `casehub-pages-push` | Existing — push wire protocol |
| `pages-aria` (TS) | — | Library browser is UI-only, no new TS dependencies |

### 7.4 Platform YAML Core API surface (`io.casehub.yaml.core`)

The scenario compilation pipeline consumes these types from `casehub-platform-yaml-core`:

| Type | Package | Purpose |
|---|---|---|
| `VariableResolver` | `resolver` | `${prefix.name}` resolution with pluggable `VariableSource` chain, `withEachContext()` for simple iteration, `withEachRowContext()` for CSV row iteration, `withScope()` for caller context inheritance |
| `VariableSource` | `resolver` | `@FunctionalInterface` — `resolve(name) → String`. Static `chain()` composes ordered sources |
| `ForEachExpander` | `foreach` | Generic `<E> expand(elements, groups, resolver, adapter, maxExpansion)`. Domain provides `ForEachAdapter<E>` |
| `ForEachAdapter<E>` | `foreach` | `stamp()`, `getForEach()`, `getId()`, `getWhen()` — scenario provides `ScenarioStepAdapter` |
| `IterationGroup` | `foreach` | `record(as, in)` — named iteration group |
| `ExpansionResult<E>` | `foreach` | `record(elements, excludedIds)` — expansion output |
| `CsvParser` | `data` | `parse(name, csvContent) → CsvDataSource` — header parsing, type validation |
| `CsvDataSource` | `data` | `record(name, columns, rows)` — parsed CSV with typed rows |
| `CsvColumn` | `data` | `record(name, type)` |
| `CsvColumnType` | `data` | `STRING, INTEGER, BOOLEAN, DECIMAL` with `parse()` method |
| `Truthiness` | `condition` | `isTruthy(String) → boolean` — shared `when` evaluation |

The scenario `ScenarioCompiler` wires these as:
```java
var paramSource = VariableSource.chain(
    callerParams::get,          // caller-supplied
    preferences::get,           // platform preferences
    scriptDefaults::get,        // params[].default
    config::getOptionalValue    // MicroProfile Config
);
var resolver = new VariableResolver(
    Map.of("params", paramSource, "var", paramSource),
    Set.of("step")              // deferred — resolved at execution time
);
```

## 8. Non-Goals

1. **Script version history** — overwrite is destructive. Versioning
   deferred to a future phase.
2. **Script editor UI** — authoring is done in a text editor or via
   paste. No visual script builder.
3. **Multi-scenario concurrent execution** — one scenario at a time
   per orchestrator (existing constraint).
4. **Nested orchestrators** — `call` inlines into the current session,
   not a sub-orchestrator.
5. **Cross-registry call resolution** — scripts in one registry can
   `call` scripts in another only if both are configured on the same
   server. No federated resolution.

## References

- Distributed executor protocol spec (casehubio/parent#418)
- Cross-platform scenario engine spec (casehub-life)
- ARIA interaction contract (PP-20260817-a11y01)
- `casehub-platform-yaml-core` — shared YAML primitives:
  `VariableResolver`, `ForEachExpander`, `CsvParser`, `Truthiness`
  in `/Users/mdproctor/claude/casehub/platform/yaml-core/`
- Desired-state YAML (migration source): `ForEachExpander.java`,
  `VariableResolver.java` in `casehub-desiredstate/yaml/runtime/`
- Existing scenario specs: `docs/specs/issue-408-scenario-engine/`
  (D8-D27)
- `scenario-handler.ts` — browser executor
- `ScenarioParser.java` — server-side YAML parser
- `ScenarioConnectionController` — controller push wire management
