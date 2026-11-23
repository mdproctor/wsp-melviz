---
layout: post
title: "Defined Is Not True"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [yaml-core, walker, parallel, boolean-semantics]
series: playbook-editor
---

# Defined Is Not True

Continues from [Parity as a Forcing Function](2026-11-22-mdp01-parity-as-forcing-function.md).

The parity fix for `advance-after` defaulting introduced a check: if `cancel-remaining` is present, default `advance-after` to the branch count. The check was `cancelRemaining !== undefined`. That's correct for `cancel-remaining: true` — the user wants cancellation semantics, so the runtime needs an advancement threshold. It's wrong for `cancel-remaining: false`.

`false` is not `undefined`. It's a defined value that happens to be falsy. The `!== undefined` check treats "I explicitly said no" the same as "I explicitly said yes" — both are "present", both trigger the default.

The consequence: a playbook that says `cancel-remaining: false` gets routed through `evaluateParallelDiscriminator` with `advanceAfter` set to the branch count. The discriminator waits for all branches to complete, then checks whether to cancel the remaining ones. Since `cancelRemaining` is `false`, it doesn't cancel anything. The result is functionally identical to the simple `Promise.allSettled` path — but the user has been shunted through unnecessary machinery for explicitly declining a feature.

The fix is one operator:

```typescript
// Before — defined means present, false counts
cancelRemaining !== undefined
// After — only true triggers the default
cancelRemaining === true
```

This is a category of bug that surfaces anywhere YAML boolean config meets JavaScript truthiness. YAML gives you three states for a boolean key: absent (truly not set), `false` (explicitly off), and `true` (explicitly on). JavaScript's `!== undefined` collapses absent and false into the same bucket — which is usually what you want for optional string or number fields, where any defined value means "the user set this." For booleans, that collapse conflates opting out with not having an opinion.

The test matrix for a YAML boolean key is small enough to be exhaustive:

| `cancel-remaining` | `advanceAfter` default | Path |
|---|---|---|
| absent | `undefined` | `Promise.allSettled` |
| `false` | `undefined` | `Promise.allSettled` |
| `true` | `branchCount` | discriminator |
| `true` + explicit `advance-after: 1` | `1` | discriminator |

The existing tests covered rows 1, 3, and 4. Row 2 was missing — the one where the boolean is defined but false. Adding it turned the check red immediately, confirming the bug wasn't theoretical.

Worth remembering for any config parser that handles YAML booleans: `!== undefined` is a presence check, not a truthiness check. When the distinction between "absent" and "explicitly false" matters — and for feature toggles it usually does — test both.
