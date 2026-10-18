# Decorator Structural Keys — Design Spec

**Issue:** casehubio/casehub-pages#571
**Date:** 2026-10-18
**Status:** Draft

## Summary

Promote six decorator keys — `forEach`, `retry`, `timeout`, `when`, `mutex`,
`loop` — to structural keys in both the TypeScript Walker
(`packages/yaml-core/src/step/walker.ts`) and the Java Walker
(`platform/yaml-step-runtime/.../catalog/Walker.java`). When these keys
carry a non-scalar value (object or array), they become structural and
contain child steps directly, eliminating the `block` wrapper.

## YAML Surface

Three value forms, disambiguated by type:

### 1. Object value — structural with parameter

The decorator's scalar parameter becomes a mapping key; its value is
the steps array. Valid YAML: the trailing `:` after the scalar creates
a nested mapping.

```yaml
# forEach with expression and child steps
- forEach: "member in members":
    - fill: ...
    - click: ...

# retry with count and child steps
- retry: 3:
    - fill: ...
    - click: ...

# when with condition and child steps
- when: "${params.enabled}":
    - fill: ...
    - click: ...

# timeout with duration and child steps
- timeout: "30s":
    - fill: ...
    - click: ...

# mutex with lock name and child steps
- mutex: "order-lock":
    - fill: ...
    - click: ...

# loop with mode and child steps
- loop: continuous:
    - fill: ...
    - click: ...
```

Parsed structure (e.g. forEach):
```json
{"forEach": {"member in members": [{"fill": "..."}, {"click": "..."}]}}
```

The Walker extracts the single object key as the decorator parameter and
the key's value as child steps.

### 2. Array value — structural without parameter

The decorator key's value is directly the steps array. No parameter —
uses whatever default the runtime applies.

```yaml
- retry:
    - fill: ...
    - click: ...
```

### 3. Scalar value — decorator (unchanged)

When the value is a scalar alongside an action key, behaviour is
unchanged — the decorator attaches to the action step.

```yaml
- click:
    target: button
  retry: 3
  forEach: "member in members"
```

### Combined decorators

Multiple decorators can combine. One takes the structural role; the rest
remain scalar decorators on the resulting block:

```yaml
- forEach: "member in members":
    - fill: ...
    - click: ...
  retry: 3
  timeout: "30s"
```

Only one decorator can be structural per step — if two decorators both
carry non-scalar values, the Walker errors.

## Resolved Output

The structural form produces the **same output** as the current
`block` + decorator form. No new step kinds, no runtime changes.

```yaml
# New structural form
- forEach: "member in members":
    - fill: ...

# Equivalent existing form
- block:
    - fill: ...
  forEach: "member in members"
```

Both produce:
```
BlockStep {
  kind: 'block',
  steps: [resolved fill step],
  decorators: { forEach: "member in members" }
}
```

## Changes

### 1. Promotable decorator set (both Walkers)

New constant identifying the six promotable decorators:

```
PROMOTABLE_DECORATORS = { 'forEach', 'retry', 'timeout', 'when', 'mutex', 'loop' }
```

### 2. TypeScript Walker — `resolveOne()` classification loop

In the key iteration loop (`walker.ts:238-278`), before the existing
`DECORATOR_KEYS.has(key)` check, add promotable-decorator handling:

```
if PROMOTABLE_DECORATORS.has(key):
  if value is array or non-null object:
    if structuralType already set:
      error: "step has both structural type and structural decorator"
    structuralType = key
    structuralValue = value
  else:
    decorators[key] = value   // scalar — stays as decorator
```

### 3. TypeScript Walker — structural resolution block

After the existing structural type handlers (block, parallel, if, match,
try, select, race, barrier, quorum), add a handler for promotable
decorators:

```
if PROMOTABLE_DECORATORS.has(structuralType):
  if structuralValue is array:
    // Array form — no parameter
    substeps = resolveAtDepth(structuralValue, ...)
    decorators[structuralType] = true
  else:
    // Object form — extract parameter and steps
    entries = Object.entries(structuralValue)
    if entries.length !== 1:
      error: "structural decorator must have exactly one key (the parameter)"
    [param, stepsRaw] = entries[0]
    if stepsRaw is not array:
      error: "structural decorator value must be a steps array"
    substeps = resolveAtDepth(stepsRaw, ...)
    decorators[structuralType] = param
  return BlockStep { steps: substeps, decorators }
```

### 4. Java Walker — mirror changes

Same classification and resolution changes in
`platform/yaml-step-runtime/.../catalog/Walker.java`:

- Add `PROMOTABLE_DECORATORS` set
- In `resolveOne()` key loop: detect non-scalar values for promotable
  decorators → set as structural
- After existing structural handlers: resolve promotable structural
  decorators into `BlockStep` with extracted parameter

### 5. Schema composer — structural forms

`schema-composer.ts` adds `oneOf` entries for each promotable decorator
in structural form, allowing the JSON schema to validate both scalar
(decorator) and object/array (structural) values.

### 6. Validation rules

| Condition | Result |
|-----------|--------|
| Promotable decorator with object value (1 key) | Structural — key is parameter, value is steps |
| Promotable decorator with array value | Structural — no parameter |
| Promotable decorator with scalar value + action key | Decorator (current) |
| Promotable decorator with scalar value, no action key | Decorator-only step (current — empty BlockStep in Java, error in TS unless standalone `delay`) |
| Two promotable decorators both non-scalar | Error: "multiple structural decorators" |
| Structural decorator + `block`/`parallel`/etc | Error: "structural type and structural decorator" |
| Structural decorator + action key | Error: "structural decorator and action key" |
| Object value with != 1 key | Error: "structural decorator must have exactly one key" |

## Scope boundaries

**In scope:**
- TS Walker (`packages/yaml-core/src/step/walker.ts`)
- Java Walker (`platform/yaml-step-runtime/.../catalog/Walker.java`)
- TS Walker tests (`walker.test.ts`)
- Java Walker tests (`WalkerTest.java`)
- Schema composer (`schema-composer.ts`)

**Out of scope:**
- Runtime step execution — no changes needed (same resolved output)
- LSP completions — follow-up issue
- Step viewer — follow-up if needed

## References

- `packages/yaml-core/src/step/walker.ts:250-267` — current structural/decorator classification
- `packages/yaml-core/src/step/walker.ts:298-312` — parallel array/object precedent
- `platform/yaml-step-runtime/.../catalog/Walker.java:99-138` — Java Walker classification
- `packages/yaml-core/src/step/schema-composer.ts:36-59` — structural type schema entries
- casehubio/casehub-pages#571 — issue
