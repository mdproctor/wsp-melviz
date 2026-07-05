# casehub-pages Session Handover — 2026-07-05

## Last Session

Design tokens + push protocol maturation batch: four issues (#101, #105, #106, #107) designed, adversarial-reviewed (7 rounds, 17 issues, $23.48), implemented via subagent-driven development (9 tasks, 8 per-task reviews + final branch review), and merged to main. Created `@casehubio/pages-ui-tokens` package (OKLCH 12-step colour scales, full non-colour vocabulary). Matured push protocol with general correlation layer (required `id` + ack/error on all ops), prefix wildcard matching in `TopicRegistry`, `EventStore` SPI + `InMemoryEventStore` for per-topic event replay, and Promise-based `EventConnection` with seq tracking and reconnect replay.

## Branch State

Both repos on `main`. Pause stack empty.

## What's Left

- PLATFORM.md update — casehubio/parent#346 filed for pages capability entry + repo map update · S · Low
- npm publish for 0.3.0 — version bump committed, packages not yet published to GitHub Packages registry · XS · Low
- DraftHouse adoption of `casehub-pages-push` — replace manual JSON formatting with typed PushMessage builders + TopicRegistry + EventStore · S · Med
- Connectors chat-demo adoption — replace Map-based ObjectMapper serialization with PushMessage builders · S · Low
- DraftHouse/Connectors push protocol adoption — #115 filed · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #112 | blocks-ui migration from --blocks-* to --pages-* tokens | S | Low | Pages now owns canonical tokens |
| #113 | Durable EventStore implementations (JDBC, Redis) | M | Med | InMemoryEventStore is the default; apps bring their own |
| #114 | Nested wildcard patterns (debate:room:*:summary) | S | Med | Only if trailing-* proves insufficient |
| #75 | Drag-and-drop panel rearrangement | L | High | Future epic |
| #77 | Floating/popout panels | M | High | Future epic |

## Cross-Module

**We're unblocking:**
- `casehubio/blocks-ui` — can switch from `--blocks-*` to `--pages-*` token imports (#112)
- `casehubio/connectors` — chat demo can adopt typed push protocol SDK
- `casehubio/drafthouse` — can adopt typed push protocol SDK + EventStore for replay

## References

- Design spec: `docs/superpowers/specs/2026-07-05-tokens-and-push-protocol-maturation-design.md`
- Previous: `git show HEAD~1:HANDOFF.md`
