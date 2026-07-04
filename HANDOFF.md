# casehub-pages Session Handover — 2026-07-05

## Last Session

Push protocol infrastructure batch: four issues (#89, #98, #99, #100) designed, reviewed, implemented, and merged in one branch. Created `casehub-pages-push` Java module (typed wire protocol SDK), `EventConnection` API for event-only WebSocket, shared wire utilities with listen/unlisten ops, and capability discovery endpoint. #101 (design tokens) deferred — blocks-ui's token system is still in flux.

## Branch State

Both repos on `main`. Pause stack empty.

## What's Left

- PLATFORM.md update — needs pages capability entry updated for push module + capability discovery; also layout serialization wording from prior session · S · Low
- npm publish for 0.3.0 — version bump committed, packages not yet published to GitHub Packages registry · XS · Low
- DraftHouse adoption of `casehub-pages-push` — replace manual JSON formatting in DebateWebSocket/WebSocketEventBus with typed PushMessage builders · S · Med
- Connectors chat-demo adoption — replace Map-based ObjectMapper serialization with PushMessage builders · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #101 | Adopt 12-step design token system | S | Med | Deferred — waiting for blocks-ui to stabilise tokens |
| #105 | Wildcard topic matching (debate:*) | S | Med | Filed during design review |
| #106 | Server-side event replay with seq | M | High | Filed during design review — requires event persistence |
| #107 | Error/ack ops for listen/unlisten | S | Low | Filed during design review |
| #75 | Drag-and-drop panel rearrangement | L | High | Future epic |
| #77 | Floating/popout panels | M | High | Future epic |

## Cross-Module

**We're unblocking:**
- `casehubio/connectors` — chat demo needs `casehub-pages-auth` for user identity + `pages-auth-success` event for post-login actions
- `casehubio/drafthouse` — can now adopt typed push protocol SDK (`casehub-pages-push`) and EventConnection API instead of manual JSON formatting

## References

- Design spec: `docs/superpowers/specs/2026-07-04-push-protocol-and-capabilities-design.md`
- Implementation plan: `docs/superpowers/plans/2026-07-04-push-protocol-and-capabilities.md`
- Previous: `git show HEAD~1:HANDOFF.md`
