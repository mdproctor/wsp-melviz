# Scenario Lifecycle Engine — Design Spec

**Issue:** casehubio/casehub-pages#498
**Branch:** issue-418-theme-resource
**Date:** 2026-10-26

## Overview

Replace the hardcoded `ScriptLifecycleState` enum (DRAFT, ACTIVE, ARCHIVED)
with a configurable lifecycle engine backed by the existing
`ScenarioDefinition` state machine model. Domains define their own lifecycle
states and transitions in YAML — the default draft/active/archived becomes
a configuration, not the only option.

The platform already provides a generic state machine engine
(`ScenarioDefinition` with states, transitions, events, and guards) for
domain content. Using the same model for script lifecycle management means
the platform eats its own cooking — one state machine implementation for
both execution scenarios and lifecycle management.

## Scope

**In scope:**
- `LifecycleEngine` class (TS in yaml-core, Java in backend/playbook)
- Default lifecycle YAML config (draft/active/archived)
- Replace `ScriptLifecycleState` enum with engine-driven transitions
- Wire `LifecycleEngine` into `ScriptRegistry`
- Re-export lifecycle types from yaml-core

**Out of scope:**
- Versioning (save history, diff, rollback) — separate issue
- Approval workflow (transition guards) — separate issue, builds on engine
- Domain-specific lifecycle configs (FSI, SOC, clinical) — separate issues

## Lifecycle Definition

The lifecycle is a `ScenarioDefinition` instance with empty step arrays.
States represent lifecycle phases; transitions represent allowed state
changes.

### Default Lifecycle YAML

Location: `packages/yaml-core/src/step/scenario/lifecycle/default-lifecycle.yaml`
Mirrored: `backend/playbook/src/main/resources/lifecycle/default-lifecycle.yaml`

```yaml
name: script-lifecycle
states:
  draft:
    next: active
  active:
    events:
      archive: { type: simple, target: archived }
      revise: { type: simple, target: draft }
  archived:
    events:
      resurrect: { type: simple, target: draft }
initial: draft
```

This defines the same three states as the current enum, with explicit
transitions:
- `draft` → `active` (via completion-driven transition)
- `active` → `archived` (via `archive` event)
- `active` → `draft` (via `revise` event)
- `archived` → `draft` (via `resurrect` event)

Domains override this by providing their own YAML with additional states
(e.g., `compliance-review`, `staged`, `suspended`).

## LifecycleEngine

### TS Side

Location: `packages/yaml-core/src/step/scenario/lifecycle/lifecycle-engine.ts`

The engine validates transitions against a `ScenarioDefinition`. It is
stateless regarding persistence — it validates whether a transition is
allowed but does not store which state each entity is in. The caller
owns state persistence.

```typescript
export class LifecycleEngine {
  constructor(definition: ScenarioDefinition);

  canTransition(currentState: string, targetState: string): boolean;

  validateTransition(currentState: string, targetState: string): void;
  // throws if transition is not allowed

  availableTransitions(currentState: string): readonly string[];
  // returns valid target states from the given state

  readonly initialState: string;
  readonly states: readonly string[];
}
```

The engine uses `ScenarioDefinition` directly:
- `canTransition` checks if `targetState` is reachable from `currentState`
  via the state's `next` field or any event transition
- `availableTransitions` collects all reachable states from `next` and
  event transitions
- `validateTransition` calls `canTransition` and throws a descriptive
  error if false

### Java Side

Location: `backend/playbook/src/main/java/io/casehub/pages/playbook/LifecycleEngine.java`

Mirrors the TS API:

```java
public class LifecycleEngine {
    public LifecycleEngine(ScenarioDefinition definition);

    public boolean canTransition(String currentState, String targetState);
    public void validateTransition(String currentState, String targetState);
    public List<String> availableTransitions(String currentState);

    public String initialState();
    public List<String> states();
}
```

### Default Engine Factory

Both sides provide a factory method that loads the default lifecycle YAML
and returns a `LifecycleEngine`:

```typescript
// TS
export function defaultLifecycleEngine(): LifecycleEngine;

// Java
public static LifecycleEngine defaultEngine();
```

## ScriptRegistry Changes

### ScriptLifecycleState — deleted

The `ScriptLifecycleState` enum (`DRAFT`, `ACTIVE`, `ARCHIVED`) is removed.
All references change to `String`.

### ScriptDescriptor

`state` field changes from `ScriptLifecycleState` to `String`. The default
value changes from `ScriptLifecycleState.ACTIVE` to `"active"`.

```java
public record ScriptDescriptor(String name, String description,
                                List<String> labels, List<String> tags,
                                List<ParamDescriptor> params, List<String> calls,
                                ScriptProvenance provenance,
                                String state,
                                List<Map<String, String>> firstStepTargets) {
    public ScriptDescriptor {
        // ...
        if (state == null) state = "active";
    }
}
```

### ScriptRegistry API

Three hardcoded methods collapse into one generic method:

| Before | After |
|--------|-------|
| `activate(name)` | `transition(name, "active")` |
| `archive(name)` | `transition(name, "archived")` |
| `revise(name)` | `transition(name, "draft")` |
| `listActive(labels, tags)` | `listByState("active", labels, tags)` |

```java
public ScriptDescriptor transition(String name, String targetState) {
    if (!uploaded.contains(name)) {
        throw new IllegalArgumentException(
            "Cannot transition '" + name + "': not an uploaded script");
    }
    String current = uploaded.stateOf(name);
    lifecycleEngine.validateTransition(current, targetState);
    return uploaded.setState(name, targetState);
}

public List<ScriptDescriptor> listByState(String state,
        List<String> labels, List<String> tags) {
    return allDescriptors()
            .filter(d -> state.equals(d.state()))
            .filter(d -> matchesLabels(d, labels))
            .filter(d -> matchesTags(d, tags))
            .toList();
}
```

The `updateMeta()` guard changes from hardcoded `state != DRAFT` to a
configurable check. In the default lifecycle, only `draft` allows editing.
This can be expressed as a property on the state definition in the lifecycle
YAML (future extension) or as a convention (initial state allows editing).
For now, the guard checks `current.equals(lifecycleEngine.initialState())`.

### UploadedScriptSource

The `transition(name, state)` method is renamed to `setState(name, state)`
to reflect that it's a persistence operation, not a validation operation.
Validation is the engine's responsibility.

## TS-Side Integration

The TS `ScriptDescriptor` interface in `pages-aria/src/controller/library-view.ts`
already uses `state: string` — no change needed.

The `LifecycleEngine` is exported from yaml-core for client-side use:
determining which transitions are available for UI button states, validating
before sending requests to the server.

## File Layout

### New files

```
packages/yaml-core/src/step/scenario/lifecycle/
  default-lifecycle.yaml
  lifecycle-engine.ts
  lifecycle-engine.test.ts
  index.ts

backend/playbook/
  src/main/java/io/casehub/pages/playbook/LifecycleEngine.java
  src/main/resources/lifecycle/default-lifecycle.yaml
  src/test/java/io/casehub/pages/playbook/LifecycleEngineTest.java
```

### Modified files

```
backend/playbook/
  src/main/java/io/casehub/pages/playbook/ScriptLifecycleState.java  — DELETED
  src/main/java/io/casehub/pages/playbook/ScriptDescriptor.java      — state: String
  src/test/java/io/casehub/pages/playbook/ScriptDescriptorTest.java   — updated

backend/playbook-runtime/
  src/main/java/.../ScriptRegistry.java          — transition(), listByState()
  src/main/java/.../UploadedScriptSource.java     — setState()
  src/test/java/.../ScriptRegistryTest.java       — updated

packages/yaml-core/src/step/scenario/index.ts    — re-export lifecycle
```

## Testing Strategy

### TS unit tests — LifecycleEngine

- Loads default lifecycle from YAML
- `canTransition('draft', 'active')` → true
- `canTransition('draft', 'archived')` → false (no direct transition)
- `canTransition('active', 'draft')` → true (revise)
- `canTransition('active', 'archived')` → true (archive)
- `canTransition('archived', 'draft')` → true (resurrect)
- `canTransition('archived', 'active')` → false
- `validateTransition` throws on invalid transition
- `availableTransitions('active')` → `['archived', 'draft']`
- `availableTransitions('draft')` → `['active']`
- `initialState` → `'draft'`
- Custom lifecycle YAML with additional states works

### Java unit tests — LifecycleEngine

- Same coverage as TS tests
- Default engine factory loads correctly

### Java integration tests — ScriptRegistry

- `transition(name, 'active')` succeeds from draft
- `transition(name, 'archived')` succeeds from active
- `transition(name, 'draft')` succeeds from active (revise)
- `transition(name, 'active')` fails from archived (no direct path)
- `listByState('active', ...)` returns only active scripts
- `updateMeta` allowed in initial state, rejected in others

### Cross-runtime consistency

- Java test loads both YAML files (TS and Java copies), parses both,
  asserts identical structure

## References

- `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptLifecycleState.java` — existing enum (to be deleted)
- `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptDescriptor.java` — state field
- `backend/playbook-runtime/src/main/java/.../ScriptRegistry.java` — existing activate/archive/revise
- `packages/yaml-core/src/step/scenario/types.ts` — ScenarioDefinition, StateDefinition
- `packages/yaml-core/src/step/scenario/parser.ts` — ScenarioParser
- `packages/yaml-core/src/step/scenario/validator.ts` — ScenarioValidator
- `packages/pages-aria/src/controller/library-view.ts` — TS ScriptDescriptor interface
- casehubio/casehub-pages#498 — issue
- casehubio/fsitrading#50 — epic C13 (compliance use case)
