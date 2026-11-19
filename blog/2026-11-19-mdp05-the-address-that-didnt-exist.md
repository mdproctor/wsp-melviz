---
layout: post
title: "The Address That Didn't Exist"
date: 2026-11-19
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [playbook-editor, step-viewer, yaml-core, addressing, split-view, bidirectional-sync]
series: playbook-editor
---

# The Address That Didn't Exist

Two components, two packages, one invariant they were supposed to share: a hierarchical address scheme for steps inside a playbook. Click a step card in the viewer, it emits an address. The scanner looks up that address to find the YAML line. Except for unnamed parallel lanes, the scanner had no idea what the viewer was talking about.

## Two representations of the same thing

A playbook `parallel:` block has two YAML forms. Named branches use an object — each key is a lane name:

```yaml
- parallel:
    agent:
      - action: do-work
    reviewer:
      - action: review-work
```

Unnamed branches use an array — each element is a single step running in its own lane:

```yaml
- parallel:
    - action: step-1
    - action: step-2
```

The walker in `yaml-core` resolves both into a `ParallelStep`. Named branches produce `NamedBranch[]` with names from the YAML keys. Array branches produce a flat `steps[]` with no branch names. The resolved type faithfully represents both forms.

## The addressing split

`assignStepAddresses` in the step viewer operates on resolved steps. It handles both paths:

```
Named:   0.lane.agent.0    (branch.name)
Unnamed: 0.lane.0.0        (numeric index, .0 for the single step)
```

`StepPositionScanner` in yaml-core operates on raw YAML text. It calls `scanNamedBranches`, which scans for key-colon patterns at the expected indent. Named branches work — `agent:` and `reviewer:` are keys, they get scanned, addresses match. But unnamed branches are list items (`- action: ...`), and `scanNamedBranches` has this line:

```typescript
if (stripped.startsWith('- ')) { i++; continue; }
```

Skip. Next line. The scanner produces the address `0` for the parallel step itself, and nothing for its children. The viewer emits `0.lane.0.0` on click. The scanner has no entry for that address. The cursor doesn't move.

![Address generation mismatch between scanner and step-click](images/parallel-address-mismatch.svg)

## Why it stayed hidden

The split-view playbook editor (#720) landed with named-branch test playbooks — the demo YAML uses `agent:` and `reviewer:` lanes. Click sync worked. The unnamed array form never appeared in the demo, so the mismatch never fired.

This is the pattern that makes addressing bugs insidious. Each component is correct in isolation. The scanner scans all the YAML structures it was designed for. The step-click assigns addresses to all the resolved types it receives. The contract between them — "these two functions produce identical strings for the same step" — exists only as an assumption.

## The fix

Seven lines in `scanNamedBranches`. When the label is `'lane'` (unique to parallel), list items aren't noise to skip — they're unnamed lanes to scan:

```typescript
if (stripped.startsWith('- ')) {
  if (label === 'lane') {
    const laneEnd = findNextSibling(lines, i, nameIndent, stepEnd);
    scanBlock(lines, i, laneEnd, nameIndent,
      `${address}.${label}.${laneIndex}`, results);
    laneIndex++;
    i = laneEnd;
    continue;
  }
  i++; continue;
}
```

`scanBlock` does the heavy lifting. It finds the list item, assigns `0.lane.0.0`, descends into nested control flow if the lane contains an `if` or `try`. The unnamed lane gets the same treatment as any other step block — just addressed under a numeric lane identifier instead of a name.

The `label === 'lane'` guard matters. `scanNamedBranches` is also called for `race`, `select`, and `match`. Race and select have their own array-format mismatches (#726, #727), but their address conventions differ — race uses flat indices (`0.race.0`), select uses channel names (`0.select.events.0`). Each needs its own fix. The guard keeps this change scoped to what it actually fixes.

## The broader pattern

The code review surfaced the same gap in race and select. `scanNamedBranches` was written for the named-key case — object-format YAML where branches are dictionary keys. Three of its four callers pass array-format YAML. The function is correct for its original design. The callers evolved past it.

This is worth watching in any system where two components independently produce identifiers for the same entity. The contract is invisible — no shared type, no shared function, no compile-time check. The only way to catch a mismatch is a test that runs both paths on the same input and compares. For the named case, the existing tests happened to cover it. For the unnamed case, they didn't.

The fix is two tests and seven lines of scanner code. The lesson is that addressing invariants across abstraction boundaries don't announce themselves when they break.
