---
layout: post
title: "The Package Sync That Wasn't"
date: 2026-09-14
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [lsp, intellij, blocks-ui, symlinks]
---

# The Package Sync That Wasn't

I started with three issues queued on a single branch — sync the `.casehub-packages` directory in blocks-ui, port LSP fixes to the CaseHub YAML plugin, and register the domain schema formats. The branch existed but had zero implementation. A scaffolding session had created it and written the handover.

The first thing that went wrong was also the most instructive. The blocks-ui repo in this slot was sitting on `slot-190-intellij-plugin`, a branch stamped as closed. The `lsp-schemas` package — the one that holds domain format registrations and the bundled LSP server — had no source directory on that branch. It existed on main but couldn't be seen because the worktree clone can't check out main (the parent repo holds it). Creating a new branch from main brought the source tree back, and suddenly `.casehub-packages` was already in sync. Issue #445 resolved itself.

Except "in sync" was a misleading frame. I'd spent time diffing source trees between the pages repo and `.casehub-packages`, finding zero differences, and wondering what was stale. Claude spotted the answer: `.casehub-packages/packages/pages-lsp` is a symlink to `pages/packages/pages-lsp`, not a vendored copy. They're literally the same directory. The entire concept of "syncing" was meaningless — there's nothing to sync when both paths resolve to the same inode.

Issue #446 was straightforward porting work. The pages IntelliJ plugin had accumulated fixes during earlier LSP4IJ debugging — TextDocumentSync needing an object form instead of a bare enum, serverInfo in the initialize response, textEdit ranges in completion items, Node.js fallback paths for macOS, and the stale bundle cache. The blocks-ui plugin was missing all of them. Six changes, a new CompletionWeigher class for all CaseHub YAML formats, and 60 tests still green.

The interesting failure came during the build. After applying the fixes, `tsc` compiled fine and tests passed. Then I ran the full build-and-bundle pipeline, and five test suites broke with `ReferenceError: exports is not defined in ES module scope`. The pages-lsp dist had been built by a prior session using CJS output — `Object.defineProperty(exports, ...)` — while the package.json declares `"type": "module"`. The mismatch was always there, but something about running the build changed how vitest resolved the module. Rebuilding pages-lsp from the pages repo context (where the tsconfig inherits `module: "ESNext"`) produced proper ESM output and the tests came back green.

Issue #447 — the domain format registrations — was already done. The lsp-schemas package on main had complete registrations for CaseDefinition, SWF, HTN, and Org, with generated Zod schemas, content detectors, symbol extractors, and staleness tests. The issue described a problem that had been solved in an earlier branch.

Three issues closed with two commits in blocks-ui and zero code changes in pages. The work was entirely operational — getting the right branch, finding the right build context, porting existing fixes across a plugin boundary. The kind of session where the interesting part isn't what was built, but what was discovered about how things are wired.
