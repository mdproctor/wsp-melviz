## D1: PTY backend technology

**Choice:** Keep tmux — fix the pipe-pane relay buffering for the echo issue
**Alternatives:**
- pty4j (JetBrains PTY) — direct PTY control, no tmux dependency, but no session persistence across server restarts
- SPI with tmux default — abstraction layer for future backends, but YAGNI
**Rationale:** Tmux is proven, provides session persistence, detach/reattach, and consumers already use it. The echo duplication is a relay buffering issue, not a tmux design flaw.
**Trade-offs:** Requires tmux installed on the host. Acceptable — all target environments (dev machines, server containers) have tmux.
**Sources:** trellis TerminalWebSocket.java, TmuxManager.java, FifoRelay.java
**Exploration:** quick
**Status:** captured

## D2: Generalization scope

**Choice:** Core terminal only — TmuxManager, FifoRelay, SessionLogger, WebSocket endpoint, basic TerminalRegistry, REST CRUD. No trellis-specific metadata (slot, repo, issue, agent). Consumers extend via their own types.
**Alternatives:**
- Full trellis port — includes all metadata, trellis becomes thin wrapper. Over-couples pages to trellis's domain model.
- Minimal (WebSocket + TmuxManager only) — too low-level, consumers re-implement registry/logger/REST each time.
**Rationale:** Core terminal covers the shared infrastructure every consumer needs. Trellis-specific concerns (agent management, slot/repo metadata) stay in trellis as extensions.
**Trade-offs:** Trellis needs a small adapter layer for its metadata. Acceptable — it's a record extension, not a code rewrite.
**Sources:** trellis TerminalResource.java (agent coupling), TerminalInfo.java (trellis metadata)
**Exploration:** quick
**Status:** captured
