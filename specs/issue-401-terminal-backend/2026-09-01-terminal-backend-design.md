# casehub-pages-terminal — shared PTY/tmux WebSocket backend

**Issue:** casehubio/casehub-pages#401
**Date:** 2026-09-01

## Problem

The terminal frontend (`pages-component-terminal`) exists but there's no
backend module. Every consumer rolls its own PTY backend, introducing bugs
like echo duplication and output replay issues. Trellis has a working
implementation that should be generalized.

## Architecture

Two Maven modules following the `push` / `push-runtime` split:

### `casehub-pages-terminal` (core, no Quarkus)

Pure Java terminal infrastructure. No CDI, no JAX-RS, no Quarkus dependency.

| Class | Purpose |
|-------|---------|
| `TmuxManager` | Wraps tmux CLI: create/kill session, send-keys, capture-pane, pipe-pane, resize, set/get options, list sessions |
| `FifoRelay` | Reads from named FIFO pipe, relays UTF-8 text to a `Consumer<String>`. Handles multi-byte boundaries, skips initial newline |
| `SessionLogger` | Appends terminal output to per-session log files. Tail-with-offset for history retrieval |
| `TerminalSession` | Record: `name`, `workingDir`, `createdAt`. Extensible — consumers add fields via their own types |

### `casehub-pages-terminal-runtime` (Quarkus integration)

| Class | Purpose |
|-------|---------|
| `TerminalWebSocket` | WebSocket endpoint at `/ws/terminal/{id}/{cols}/{rows}`. FIFO relay per connection, session takeover, pipe-pane lifecycle |
| `TerminalRegistry` | `@ApplicationScoped` session lifecycle: create, destroy, list, get. Bootstraps from existing tmux sessions on startup |
| `TerminalResource` | JAX-RS at `/api/terminals`: CRUD, input, resize |
| `TerminalProducers` | CDI producers for `TmuxManager`, `SessionLogger` (configurable log dir) |

## Echo duplication fix

The bug: `pipe-pane -O` captures terminal output including the shell's
echo of typed characters. The FIFO relay sends both echo and command
output as one stream. When buffering aligns poorly, the echo bytes
concatenate with the first output line (e.g., `lsCLAUDE.md`).

**Fix:** The `forceRedraw` method (resize-1, sleep, resize-back) triggers
a full screen redraw on WebSocket connect. If pipe-pane is already
running from a previous connection, the relay receives the redraw AND
the original output — causing duplication.

Two fixes:
1. **Stop pipe-pane before reconnect** — `stopPipePane()` before
   `pipePaneToFifo()` in `onOpen`. Ensures only one pipe-pane per session.
2. **Delay pipe-pane start** — start pipe-pane AFTER the initial
   `forceRedraw` completes, so the redraw doesn't get captured as
   duplicate output.

## Configuration

```properties
casehub.pages.terminal.prefix=pages-       # tmux session name prefix
casehub.pages.terminal.log-dir=${java.io.tmpdir}/pages-terminal-sessions
casehub.pages.terminal.default-shell=/bin/zsh
```

## Consumer migration (trellis)

Trellis replaces `io.hortora.trellis.terminal.*` with:
1. Depend on `casehub-pages-terminal-runtime`
2. Delete: `TmuxManager`, `FifoRelay`, `SessionLogger`, `TerminalWebSocket`, `TerminalInfo`
3. Keep: `TerminalResource` (extends pages' `TerminalResource` with agent sub-resource)
4. Keep: `TerminalRegistry` subclass (adds slot/repo/issue metadata via extended session type)

## Module structure

```
backend/
  terminal/           # casehub-pages-terminal (core)
    pom.xml
    src/main/java/io/casehub/pages/terminal/
      TmuxManager.java
      FifoRelay.java
      SessionLogger.java
      TerminalSession.java
  terminal-runtime/   # casehub-pages-terminal-runtime (Quarkus)
    pom.xml
    src/main/java/io/casehub/pages/terminal/runtime/
      TerminalWebSocket.java
      TerminalRegistry.java
      TerminalResource.java
      TerminalProducers.java
```

## J2CL compatibility

Core module (`terminal`) uses `ProcessBuilder` for tmux CLI — not
J2CL-compatible. This is intentional: terminal management is inherently
server-side. The J2CL boundary note in CLAUDE.md applies only to push
protocol types, not server infrastructure.

## Test plan

1. `TmuxManager` — unit tests for command construction (mock ProcessBuilder)
2. `FifoRelay` — unit test with piped streams, verify UTF-8 boundary handling
3. `SessionLogger` — unit test tail-with-offset
4. `TerminalRegistry` — unit test session lifecycle (create, get, list, destroy)
5. `TerminalWebSocket` — integration test with mock tmux (verify pipe-pane lifecycle, session takeover)
6. Manual: verify echo duplication is fixed in trellis after migration

## References

- `trellis/sidecar/src/main/java/io/hortora/trellis/terminal/` — source to generalize
- `components/pages-component-terminal/src/PagesTerminal.ts` — frontend component
- `backend/push/` + `backend/push-runtime/` — module split pattern
- casehubio/casehub-pages#401 — issue
