---
layout: post
title: "The Test That Found the Bug"
date: 2026-09-01
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [terminal, testing, tmux]
---

# The Test That Found the Bug

The terminal backend — two new modules generalising trellis's tmux integration
into casehub-pages — had been code-complete for a session already. All the
source was written: `TmuxManager`, `FifoRelay`, `SessionLogger` in the core
module; `TerminalWebSocket`, `TerminalRegistry`, `TerminalResource` in the
Quarkus runtime. The contributor guide was updated. What remained was Maven
verification and close.

The POM fix was trivial — the parent's `<dependencyManagement>` was missing an
entry for `casehub-pages-terminal`, so the runtime module couldn't resolve the
core dependency version. One line, and the build went green.

Then I asked about test coverage.

Claude ran an audit across both modules and reported that `TmuxManager` — the
class that wraps every tmux CLI interaction — had zero tests. That turned out
to be wrong. Another Claude session had already committed eight integration
tests for `TmuxManager` and expanded `TerminalRegistryTest` from two lookups
to six lifecycle tests, including create, duplicate-throws-ISE, destroy with
cleanup, and bootstrap from existing tmux sessions. The audit had scanned the
wrong commit.

What the audit got right: `FifoRelay` was missing multi-byte UTF-8 coverage,
and `SessionLogger.tailLinesWithOffset` — the method that retrieves terminal
history with a page offset — had no tests for the offset parameter at all.

The UTF-8 and newline-handling tests for `FifoRelay` passed immediately. The
`SessionLogger` tests were more interesting. `tailLinesWithOffset` has
non-trivial seek logic: it walks backward through the log file counting
newlines, then walks forward from the end skipping `offset` lines. When the
offset exceeds the number of available lines, the backward walk hits the start
of the file and the forward walk runs out of newlines to skip — but the method
didn't check for this. It returned whatever bytes happened to sit between the
two cursors. For a two-line file with offset 10, that was the string `"li"` —
the first two bytes of "line1".

The fix was a single guard: if `skipLines < offset` after the forward walk,
return empty. The kind of bug that reads as obvious once you see it, but would
have surfaced as corrupted terminal history replay in production — the
`tailLinesWithOffset` method is what feeds reconnecting WebSocket clients their
scrollback content.

Twenty-seven tests now pass across both modules. The WebSocket and REST layers
still lack integration tests — they need Quarkus test infrastructure that isn't
in the POM yet — but the core behavioural contracts are covered.
