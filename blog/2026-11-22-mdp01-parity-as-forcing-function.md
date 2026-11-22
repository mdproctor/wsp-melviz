---
layout: post
title: "Parity as a Forcing Function"
date: 2026-11-22
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [yaml-core, step-viewer, playbook-parity, addressing, walker, scanner]
series: playbook-editor
---

# Parity as a Forcing Function

Continues from [The Address That Didn't Exist](2026-11-19-mdp05-the-address-that-didnt-exist.md).

The match address mismatch (#722) was the last of four control flow types where `StepPositionScanner` and `assignStepAddresses` disagreed on how to name steps. The prior session fixed parallel, race, and select. Match was different because the scanner was looking for a YAML format (`arms:`) that the walker doesn't even accept — it expects `cases:`. The scanner would silently produce zero child addresses. The viewer would emit `0.match.0.0`; the scanner would have nothing to look up.

The fix was a dedicated `scanMatchCases` function that finds the `cases:` container, iterates its list items, skips metadata keys (`when`, `pattern`, `guard`), and produces numeric addresses matching step-click's scheme. Default cases get their nested step list scanned separately because the indentation differs — the steps live two levels deeper under the `default:` key.

The `demoCatalog` extraction (#723) was straightforward — the same catalog stub was copy-pasted into `playbook-split-view.ts` and `gallery.html`. A shared module with proper `Catalog` typing replaced both. Worth noting only because it's the kind of duplication that persists forever when both copies work.

The interesting batch was the four playbook parity fixes (#734–738). These came from an audit comparing the TS walker against platform's Java parser, and each one exposes the same pattern: the TS port accepted things the platform rejects, or silently skipped defaults the platform applies.

`parallel: {steps: [...]}` was silently treated as a named branch called "steps" (#734). The object-form parallel parser iterates `Object.entries()` and treats every key as a branch name — no key is special. Adding a three-line guard before the iteration catches the `steps` key and throws with a migration message.

`steps:` as a document-level wrapper (#736) was quietly unwrapped in `parseAndResolve` via a fallback chain: `Array.isArray(doc) ? doc : doc.steps ?? doc`. Platform rejects this outright — the canonical form is a bare list. The fix replaces the silent unwrap with an error.

`advance-after` defaulting (#737) is subtler. Platform defaults it to the branch count when `cancel-remaining` or `collection-timeout` is present. TS left it `undefined`. The difference is invisible until a playbook relies on the default — then platform completes when all branches finish, while TS never advances because the threshold is unset. The fix is one line: `advanceAfterRaw ?? (cancelRemaining !== undefined || collectionTimeout !== undefined ? branchCount : undefined)`.

Removing `concurrent` from `RESERVED_KEYS` (#738) is cleanup. The keyword was reserved but platform doesn't use it. `concurrency` (the decorator that limits parallel thread count) is the real keyword — it's already there. `concurrent` remains wired as a structural alias for `parallel` in the walker, but there's no reason to block someone from naming an action `concurrent`.

The pattern across all four: the TS port was written from the YAML spec, not from the platform's parser. It got the shape right and missed the guardrails. Each fix is small — three lines, one line, one line, one deletion — but without the parity audit they'd surface as runtime mismatches in production playbooks.
