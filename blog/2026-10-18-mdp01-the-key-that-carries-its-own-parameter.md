---
layout: post
title: "The Key That Carries Its Own Parameter"
date: 2026-10-18
entry_type: article
subtype: diary
projects: [casehubio/casehub-pages]
tags: [walker, yaml, dsl, structural-keys, decorators]
---

# The Key That Carries Its Own Parameter

Decorators in the Walker — `forEach`, `retry`, `timeout`, `when`, `mutex`, `loop` — have always been scalar values attached to a step. To retry a group of steps, you wrap them in `block:` and hang the decorator off the side:

```yaml
- block:
    - fill: ...
    - click: ...
  retry: 3
```

`block` is an implementation detail. Authors don't think in blocks. They think "retry these steps." The YAML should say that directly.

## The obvious answer that isn't valid

The first instinct was `- retry: 3:` on a single line — the trailing colon turning the scalar into a nested mapping key. YAML rejects it. The parser sees `retry: 3` as a complete key-value pair, then chokes on the second colon.

The second attempt was the nested-key form:

```yaml
- retry:
    3:
      - fill: ...
      - click: ...
```

Valid YAML, correct parse — `{retry: {3: [{fill: ...}]}}`. The Walker extracts the single key as the parameter and the value as the steps array. It works. But it's two lines of ceremony for what should be one thought.

## The key IS the parameter

The breakthrough: put the decorator and its parameter together as a single YAML mapping key.

```yaml
- retry 3:
    - fill: ...
    - click: ...
```

`retry 3` is a plain scalar key. The colon starts its value — the steps array. The YAML parser produces `{"retry 3": [{fill: ...}, {click: ...}]}`. The Walker splits the key at the first space: decorator = `retry`, parameter = `3`.

This reads like natural language. Compare:

```yaml
# Old: implementation detail leaks through
- block:
    - fill: ...
    - click: ...
  forEach: "member in members"

# New: says what it means
- forEach member in members:
    - fill: ...
    - click: ...
```

The parameter lives in the key. The steps are the value. One line of intent, then the body.

## Three forms, one output

The Walker now accepts three forms for structural decorators. All produce the same resolved output — a `BlockStep` with the decorator in its decorators map:

| Form | YAML | When to use |
|------|------|------------|
| Combined key | `retry 3:` | Primary — cleanest |
| Nested object | `retry:` → `3:` → `[steps]` | Alternative — valid but verbose |
| Array value | `retry:` → `[steps]` | Parameterless — default count |

The disambiguation is value-type based. Scalar value alongside an action key → decorator (unchanged). Array value → structural without parameter. Object with exactly one key whose value is an array → structural with parameter. The combined-key form normalises into the object form before hitting the resolution handler, so there's one code path for both.

## Beyond decorators

The combined-key form turned out to work for all structural keywords, not just the six promotable decorators. For `block`, `parallel`, and `race`, the parameter after the space becomes the step name:

```yaml
- block setup:
    - fill: ...
    - click: ...

- parallel fast-path:
    - fill: ...
    - click: ...
```

No separate `step: setup` sibling key needed. The structural keyword carries its own identity.

## The disambiguation that matters

One subtlety cost us a round of fixes. Existing playbooks use object values as decorator *configuration* — `retry: {max: 3}`, `loop: {count: 3}`. These are not structural forms; they're config objects where no value is a steps array. The tightened check: an object value only triggers structural mode when it has exactly one key and that key's value is an array. Multi-key objects and objects with non-array values fall through as decorator configuration. The distinction is whether the object contains steps or settings.

Both the TypeScript Walker (`packages/yaml-core/src/step/walker.ts`) and the Java Walker (`platform/yaml-step-runtime/.../catalog/Walker.java`) implement the same rules. The resolved output is identical to the old `block:` + decorator form — the runtime doesn't need to change.
