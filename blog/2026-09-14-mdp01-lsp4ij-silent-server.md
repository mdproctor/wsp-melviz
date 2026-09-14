---
layout: post
title: "Why Your LSP Server Runs But IntelliJ Ignores It"
date: 2026-09-14
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [intellij, lsp, lsp4ij, debugging]
series: issue-437-lsp4ij-completions
---

# Why Your LSP Server Runs But IntelliJ Ignores It

The IntelliJ plugin starts the LSP server. The server appears in the Language Servers panel, status: running. Ctrl+Space in a `.page.yaml` file gives nothing — no LSP items, just the YAML plugin's generic `{}` suggestions. The IntelliJ log has zero `textDocument` entries. The server is alive but deaf.

I knew the server worked. 165 tests pass. Direct stdio testing returns correct completions for every scenario — union narrowing, sibling filtering, enum values. The problem was between LSP4IJ and the server, somewhere in the handshake or the file mapping.

## The stale bundle trap

The first root cause was embarrassingly mechanical. The plugin extracts its bundled server to `/tmp/casehub-lsp/server-node.bundle.cjs` and checks `if (Files.exists(targetFile)) return targetFile`. Once cached, the bundle is never re-extracted — even when the plugin is rebuilt with a completely different server. During iterative development, every code change to the LSP server was invisible. The running process was a ghost from three iterations ago.

The fix: always re-extract. The bundle is 1.2MB; extraction takes milliseconds. There is no performance argument for caching something that's loaded once per IntelliJ session.

## The quiet protocol mismatch

The LSP specification says `textDocumentSync` accepts either a bare number (`TextDocumentSyncKind.Full`, value `1`) or an object (`{ openClose: true, change: 1 }`). Both are valid. Every `vscode-languageserver` example uses the number form. It works perfectly with VS Code and CodeMirror clients.

LSP4IJ may only parse the object form. Without `openClose: true`, it doesn't know the server wants `didOpen`/`didClose` notifications — and silently doesn't send them. No error, no warning, just zero document events. Claude and I verified this by sending a raw `initialize` → `didOpen` → `completion` sequence to the server via stdio. The server responded correctly with 55 completion items. The protocol worked; the capability advertisement didn't.

## The file pattern gap

The plugin's `fileNamePatternMapping` covered `*.page.yaml` only. The spec (D9) explicitly calls for `*.dash.yaml` transition support during the rename period. Missing it meant files with the old extension got no LSP intelligence at all.

## CompletionWeigher — the safe filtering approach

The YAML plugin's `{}` and `[]` completions clutter the popup alongside the LSP items. The obvious approach — a `CompletionContributor` that filters them out — doesn't work with LSP4IJ. Three approaches were tried in the original debugging session: `runRemainingContributors` blocks LSP4IJ's async channel, `stopHere()` kills all contributors including LSP4IJ's, and a custom CaseHubYAML language dialect broke the completion pathway entirely.

`CompletionWeigher` is the safe alternative. It only affects sort order — runs after all contributors finish, can't block or interfere with any pipeline. YAML structural items get weight -1, everything else gets 0. They sink to the bottom without being removed.

## What's left

These fixes address every likely root cause we could identify from the code. The actual verification needs IntelliJ — the stderr logging (`[casehub-lsp] initialize`, `didOpen`, `completion`) will show in the Language Servers console exactly where the handshake stalls, if it still does. Build integration for the plugin's Gradle build is tracked separately in #438.
