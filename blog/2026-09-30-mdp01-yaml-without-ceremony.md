---
layout: post
title: "YAML Without Ceremony"
date: 2026-09-30
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [yaml, ceremony, walker, format-design, type-unification]
---

# YAML Without Ceremony

The type unification work dropped the `Step` prefix from every TypeScript type in the step module — `StepDefinition` became `Definition`, `StepWalker` became `Walker`. That was the easy part. The harder question was what happened to the YAML format.

Every scenario script started with this:

```yaml
steps:
  - counter-add: { name: "requests", delta: 1 }
  - counter-add: { name: "requests", delta: 3 }
```

That `steps:` line was pure wrapper — the Walker already accepted a bare array. The type unification made the wrapper meaningless: actions are first-class, they don't need a container keyword to distinguish them from anything.

I wanted to remove it entirely. Claude kept insisting it was necessary — first because "a YAML mapping can only have one key" (wrong), then because "you need a container key to separate the action array from configuration" (wrong in the cases that mattered). It took Grok pointing out that `{ wait: "signal", log: { message: "arrived" } }` is perfectly valid YAML — multiple keys in the same mapping — before Claude accepted the point.

The deeper issue was nested ceremony. Inside `select` branches and `match` cases, child actions were wrapped in a `do:` key:

```yaml
- select:
  - wait: fast-signal
    do:
      - log: { message: "Fast branch won" }
```

We eliminated `do:` by treating the child action as a sibling key — strip the configuration keys (`wait`, `pattern`, `guard`), resolve whatever remains using the same logic that handles every other step. Single actions sit inline. Multiple actions use `block:`, which already exists as a structural grouper:

```yaml
- select:
  - wait: fast-signal
    log: { message: "Fast branch won" }
```

The Walker now rejects both `steps` and `do` with a clear error pointing to inline keys or `block`. The only container keys that survive are semantic branch labels — `then`/`else`/`catch`/`finally`/`cases` — which distinguish multiple branches, not wrap a single one.

One thing I hadn't considered: scenario documents that have metadata alongside the action list (`scenario: demo-name` plus the actions). YAML can't hold both a mapping and a sequence at the top level. The front matter pattern handles this — metadata in a mapping, `---` separator, bare action array below. That's how Jekyll, Hugo, and every static site generator already works. It's a parser change on both TS and Java, tracked in the drift epic.

The full audit found drift across both repos — old `Step`-prefixed types in Java (untouched), `steps` keys in the Java Walker, import schema using `steps` instead of `actions`, test descriptions with stale names. A single epic (#508) tracks all of it to prevent the unification from being half-done indefinitely.
