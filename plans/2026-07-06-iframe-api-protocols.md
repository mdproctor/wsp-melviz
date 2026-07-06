# Iframe Component API Protocols — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #122 — Create protocols for iframe component API conventions

**Goal:** Write two protocol files formalising the iframe component postMessage
wire format and lifecycle sequence, and update the protocol indexes.

**Architecture:** Pure documentation — two new protocol markdown files under
`docs/protocols/casehub/`, following the existing frontmatter schema
(PP-YYYYMMDD-hexid format). Both index files updated with new entries.

**Tech Stack:** Markdown with YAML frontmatter

## Global Constraints

- Protocol IDs use format `PP-YYYYMMDD-XXXXXX` (6-char hex suffix)
- Frontmatter fields: `id`, `title`, `type`, `scope`, `applies_to`,
  `cross_repo_consumers`, `severity`, `refs`, `violation_hint`, `created`
- `type: rule`, `scope: repo`, `severity: important` for both protocols
- Protocol content documents the **target** wire format (plain objects),
  noting where current code diverges (#130)

---

### Task 1: Write iframe-message-format.md protocol

**Files:**
- Create: `docs/protocols/casehub/iframe-message-format.md`

**Interfaces:**
- Consumes: spec section "Protocol 1: Iframe Message Format"
- Produces: protocol file referenced by Task 3 (index updates)

- [ ] **Step 1: Generate protocol ID**

```bash
echo "PP-20260706-$(python3 -c 'import secrets; print(secrets.token_hex(3))')"
```

Use the output as the `id` field in frontmatter.

- [ ] **Step 2: Write the protocol file**

Create `docs/protocols/casehub/iframe-message-format.md` with this content:

```markdown
---
id: <generated-id>
title: "Iframe component messages use ComponentMessage envelope with plain-object properties over postMessage"
type: rule
scope: repo
applies_to: "third-party and internal iframe components communicating with the casehub-pages host via postMessage"
cross_repo_consumers: []
severity: important
refs: ["#130"]
violation_hint: "Iframe component message uses Map for properties, uses non-string property keys, or includes non-structured-clone-safe values"
created: 2026-07-06
---

Iframe components communicate with the host page via `postMessage` using
a `ComponentMessage` envelope:

\`\`\`typescript
interface ComponentMessage {
  type: MessageType;
  properties: Record<string, unknown>;
}
\`\`\`

`properties` is a plain object, not a `Map`. All values must be
structured-clone-safe: no `Map`, no class instances, no functions.

> **Current code divergence:** the implementation uses
> `Map<MessageProperty | string, unknown>` for `properties`, which does
> not survive `postMessage` structured clone across iframe boundaries.
> See #130. This protocol documents the correct target format.

## Message types — host → component

| Type | Key properties | Purpose |
|------|---------------|---------|
| `INIT` | `component_id`, plus arbitrary params | Initialise with configuration |
| `DATASET` | `component_id`, `dataSet`, plus params from INIT | Deliver/update dataset |
| `FUNCTION_RESPONSE` | `functionResponse` | Return function call result |

## Message types — component → host

| Type | Key properties | Purpose |
|------|---------------|---------|
| `FILTER` | `component_id`, `filter` | Request cross-filter |
| `FUNCTION_CALL` | `component_id`, `functionCallRequest` | Request server-side function |
| `FIX_CONFIGURATION` | `component_id`, `configurationIssue` | Signal invalid config |
| `CONFIGURATION_OK` | `component_id` | Signal config resolved |
| `READY` | `component_id` | Signal readiness (reserved, not yet implemented) |

## Property keys

String constants from the `MessageProperty` enum, used as plain string
keys in the `properties` object:

| Key | Value | Used in |
|-----|-------|---------|
| `component_id` | `"component_id"` | All messages |
| `dataSet` | `"dataSet"` | DATASET |
| `configurationIssue` | `"configurationIssue"` | FIX_CONFIGURATION |
| `filter` | `"filter"` | FILTER |
| `functionCallRequest` | `"functionCallRequest"` | FUNCTION_CALL |
| `functionResponse` | `"functionResponse"` | FUNCTION_RESPONSE |
| `mode` | `"mode"` | INIT (optional) |

Components must use string values as keys, not enum references — the
host and component may not share the same module.

## Data types

All types carried in message properties are plain objects with only
primitive and array fields:

\`\`\`typescript
interface DataSet {
  columns: Column[];
  data: string[][];
}

interface Column {
  id: string;
  name: string;
  type: ColumnType;
  settings?: ColumnSettings;
}

interface FilterRequest {
  reset: boolean;
  row: number;
  column: number;
}

interface FunctionCallRequest {
  functionName: string;
  parameters: Record<string, unknown>;
}

interface FunctionResponse {
  message: string;
  request: FunctionCallRequest;
  resultType: FunctionResultType;  // "SUCCESS" | "ERROR" | "NOT_FOUND"
  result: unknown;
}
\`\`\`

## component_id

Set by the host on INIT. The component must echo it on every message
sent back to the host. The host uses it to route messages when multiple
iframe components coexist in a dashboard.

## Relationship to other protocols

This protocol covers the **wire format** for iframe-isolated components
communicating via `postMessage`. It is distinct from:

- **pages-event-contract** — covers `CustomEvent`-based inter-component
  events within the host DOM. Iframe components are isolated from this.
- **iframe-component-lifecycle** — covers the **sequence** and
  **state signalling rules** for the same message types defined here.
```

- [ ] **Step 3: Commit**

```bash
git -C <PROJECT> add docs/protocols/casehub/iframe-message-format.md
git -C <PROJECT> commit -m "docs: iframe message format protocol

Refs #122"
```

---

### Task 2: Write iframe-component-lifecycle.md protocol

**Files:**
- Create: `docs/protocols/casehub/iframe-component-lifecycle.md`

**Interfaces:**
- Consumes: spec section "Protocol 2: Iframe Component Lifecycle"
- Produces: protocol file referenced by Task 3 (index updates)

- [ ] **Step 1: Generate protocol ID**

```bash
echo "PP-20260706-$(python3 -c 'import secrets; print(secrets.token_hex(3))')"
```

- [ ] **Step 2: Write the protocol file**

Create `docs/protocols/casehub/iframe-component-lifecycle.md` with this content:

```markdown
---
id: <generated-id>
title: "Iframe components follow INIT → DATASET lifecycle with configuration error signalling"
type: rule
scope: repo
applies_to: "third-party and internal iframe components communicating with the casehub-pages host via postMessage"
cross_repo_consumers: []
severity: important
refs: []
violation_hint: "Iframe component sends messages before receiving INIT, sends FILTER without component_id, or fails to signal FIX_CONFIGURATION on invalid input"
created: 2026-07-06
---

Iframe components follow a host-driven lifecycle. The host initiates
communication; the component responds. The rendering library inside the
iframe is the third party's choice — this protocol covers only the
sequence and state rules for messages crossing the `postMessage` boundary.

## Lifecycle sequence

\`\`\`
Host                          Component
  │                               │
  ├── INIT (params + id) ────────►│
  │                               │  validate params
  │                               │  MAY send FIX_CONFIGURATION
  │                               │
  ├── DATASET (data + params) ───►│
  │                               │  validate dataset
  │                               │  MAY send FIX_CONFIGURATION
  │                               │
  │◄── FILTER ────────────────────┤
  │                               │
  │◄── FUNCTION_CALL ─────────────┤
  ├── FUNCTION_RESPONSE ─────────►│
  │                               │
  ├── DATASET (updated) ─────────►│  after filter or poll
\`\`\`

## Ordering guarantees

- INIT always precedes the first DATASET
- DATASET may arrive multiple times (after filters, polling, user actions)
- Each DATASET re-delivers the INIT params alongside the data
- FUNCTION_RESPONSE correlates to a prior FUNCTION_CALL by `functionName`

## Configuration error flow

1. Component validates params (on INIT) or dataset shape (on DATASET)
2. Invalid → send `FIX_CONFIGURATION` with a human-readable message
3. User corrects configuration in the dashboard editor
4. Host re-sends INIT or DATASET with corrected values
5. Component validates again → if valid, send `CONFIGURATION_OK`
6. `CONFIGURATION_OK` clears any error indicator in the host UI

Components should provide actionable error messages — e.g.,
"Heatmap expects 2 columns: Node ID (TEXT or LABEL) and value (NUMBER)",
not "Invalid configuration".

## Function call correlation

- Component sends `FUNCTION_CALL` with `functionName` and `parameters`
- Host executes server-side and returns `FUNCTION_RESPONSE`
- `resultType` is `SUCCESS`, `ERROR`, or `NOT_FOUND`
- Correlation is by `functionName` — only one in-flight call per
  function name is supported

## READY (reserved, not yet implemented)

`MessageType.READY` exists in the enum but is not implemented. The host
currently treats all components as immediately ready after INIT.

Future use: signal readiness after async setup (loading external
resources, establishing connections).

## Dev mode testing

`@casehubio/pages-iframe-dev` provides a local test harness simulating
the host side. This is an internal testing tool, not a contractual
interface.

- `manifest.dev.json` defines INIT params, dataset, and function responses
- Dev harness sends INIT then DATASET on 100ms delay, simulating the
  production sequence
- Separate entry point (e.g., `index-dev.tsx`) adds the dev harness
  alongside the component

## Relationship to other protocols

- **iframe-message-format** — defines the wire shape of the messages
  whose sequence this protocol governs.
- **pages-event-contract** — covers host-DOM `CustomEvent` communication.
  Iframe components are isolated from this event system.
```

- [ ] **Step 3: Commit**

```bash
git -C <PROJECT> add docs/protocols/casehub/iframe-component-lifecycle.md
git -C <PROJECT> commit -m "docs: iframe component lifecycle protocol

Refs #122"
```

---

### Task 3: Update protocol indexes

**Files:**
- Modify: `docs/protocols/casehub/INDEX.md`
- Modify: `docs/protocols/INDEX.md`

**Interfaces:**
- Consumes: protocol files from Tasks 1 and 2
- Produces: updated indexes (end state)

- [ ] **Step 1: Add entries to casehub/INDEX.md**

Append two rows to the table in `docs/protocols/casehub/INDEX.md`:

```markdown
| [iframe-message-format.md](iframe-message-format.md) | Iframe messages use ComponentMessage envelope with plain-object properties over postMessage | Iframe components via postMessage |
| [iframe-component-lifecycle.md](iframe-component-lifecycle.md) | Iframe components follow INIT → DATASET lifecycle with configuration error signalling | Iframe components via postMessage |
```

- [ ] **Step 2: Add entries to top-level INDEX.md**

Append two rows to the CaseHub Platform table in `docs/protocols/INDEX.md`:

```markdown
| [casehub/iframe-message-format.md](casehub/iframe-message-format.md) | Iframe messages use ComponentMessage envelope with plain-object properties | Iframe components via postMessage |
| [casehub/iframe-component-lifecycle.md](casehub/iframe-component-lifecycle.md) | Iframe components follow INIT → DATASET lifecycle with config error signalling | Iframe components via postMessage |
```

- [ ] **Step 3: Commit**

```bash
git -C <PROJECT> add docs/protocols/INDEX.md docs/protocols/casehub/INDEX.md
git -C <PROJECT> commit -m "docs: add iframe protocols to indexes

Closes #122"
```

---

## Verification

After all three tasks:

```bash
# Confirm all protocol files exist
ls docs/protocols/casehub/iframe-message-format.md docs/protocols/casehub/iframe-component-lifecycle.md

# Confirm indexes reference them
grep iframe docs/protocols/INDEX.md docs/protocols/casehub/INDEX.md

# Confirm frontmatter is valid YAML
python3 -c "
import yaml, pathlib
for f in ['iframe-message-format.md', 'iframe-component-lifecycle.md']:
    text = pathlib.Path(f'docs/protocols/casehub/{f}').read_text()
    fm = text.split('---')[1]
    d = yaml.safe_load(fm)
    assert d['id'].startswith('PP-'), f'{f}: bad id'
    assert d['type'] == 'rule', f'{f}: bad type'
    print(f'{f}: OK ({d[\"id\"]})')
"
```
