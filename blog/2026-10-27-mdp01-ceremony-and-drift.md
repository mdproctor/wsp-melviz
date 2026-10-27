---
layout: post
title: "Ceremony and Drift"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [type-unification, yaml, parser, design]
series: issue-508-complete-type-unification
---

# Ceremony and Drift

Type unification landed weeks ago in #506 — the big renames, the portability model, the new `Action`/`Result`/`Walker` vocabulary. The code moved. The ceremony didn't. That's what #508 is for: the cleanup epic, the part where you go through every file and make sure the old world is actually gone.

I expected this to be mechanical. It wasn't.

## The import property that wasn't an alias

The issue body listed `YamlImport.steps` as a rename target — change it to `actions`, done. But when I looked at the interface, both properties already existed side by side:

```typescript
export interface YamlImport {
  module?: string;
  actions?: string;   // new
  steps?: string;     // old
  as: string;
  // ...
}
```

These weren't aliases. They were semantically distinct. `actions` selected which actions to import from a module. `steps` was the old YAML key from before the unification — a different mechanism entirely. The `ImportExpander` had a filter that relied on this distinction:

```typescript
const moduleImports = imports.filter(imp => imp.steps == null);
```

This kept module imports and filtered out old-style step imports. A straight rename — `imp.actions == null` — would break the expander completely: it would filter out every import that specified actions, including the ones it was supposed to expand.

The fix was to remove the filter. The Zod schema already rejects `steps:` in YAML, so old-format imports can't reach the runtime anymore. The migration guard was protecting against something the validation layer had already closed off. With `steps` gone from the interface and the schema enforcing `actions`, the expander processes all imports uniformly — module imports and action imports both expand correctly through the same pipeline.

## Bare arrays and the third document

The other half of the epic is YAML ceremony — the `steps:` key that wraps every playbook's action list:

```yaml
meta:
  title: "Sequential"
steps:
  - click: { role: button, name: "A" }
  - click: { role: button, name: "B" }
```

The design decision from #506 was clear: standalone action lists are bare arrays, no wrapper. The target format uses YAML's multi-document separator to split metadata from actions:

```yaml
playbook: "1.0"
schema: client
---
meta:
  title: "Sequential"
---
- click: { role: button, name: "A" }
- click: { role: button, name: "B" }
```

Three documents: front matter, metadata, bare action array. The parser already used `parseAllDocuments` from the `yaml` library — it just needed to recognise a third document and treat an array there as the action list. The implementation is eight lines in `splitMultiDoc`: detect three documents, check if the third is an array, synthesise the internal `{ steps: array }` shape that `parsePlaybookFromParsed` already knows how to handle.

I kept `steps` as the internal model property deliberately. The ceremony elimination is about the YAML surface — what users write. Internally, a playbook still has `steps`. Renaming the internal model would touch every test, every type guard, every consumer. That's churn with no user-facing payoff. The parser absorbs the format change; the rest of the system doesn't need to know.

## What's left

The Pages-side work is done. All TS type renames were already complete from #506. The import schema drift is fixed. The parser handles both the old 2-doc format and the new 3-doc format. The 26 source playbook files in the platform repo are converted.

What remains is the platform Java side: `StepWalker` has about a hundred references to rename, the Java parser needs the same 3-doc support, and the `resolveMatchCases`/`resolveSelectBranches` methods still read from a `steps` key in YAML maps. That's a separate session — different repo, different language, same principle: the code moved, now make sure the ceremony catches up.

The pattern I keep seeing in these cleanup epics: the interesting work isn't the renaming. It's finding the places where a name change would silently alter semantics — where what looks like drift is actually a load-bearing distinction that the original design encoded in a property name nobody documented.
