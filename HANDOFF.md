# casehub-pages Session Handover — 2026-06-30

## Last Session

Closed branch `issue-56-websocket-robustness` covering 5 issues: WebSocket auth via query-param tokens (#58), transparent relay proxy (#57), incremental reconnect via seq/since (#56), single-subscriber fallback routing (#61), duplicate subscribe guard + append column-count validation (#62). Squashed to 6 commits, pushed to fork + blessed. Also corrected README and CLAUDE.md to TypeScript-first positioning (#63 filed for broader web architecture doc).

## Branch State

Both repos on `main`. Fork and blessed current.

## What's Left

*None — all trailing obligations resolved.*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #72 | WebSocket test coverage gaps (auth+reconnect interaction, replace/remove seq, relay query params) | XS | Low | Batch of 5 minor test additions |
| #73 | Add WebSocket section to CASEHUB-PAGES.md LLM integration guide | S | Low | Document auth, relay, seq, wire protocol |
| #63 | Web Architecture document (PLATFORM.md equivalent for frontend tier) | S | Med | New doc |
| #70 | Error propagation for push sources (WebSocket permanent close → component notification) | S | Med | Gap: DataSetEventListener has no error channel |
| #71 | WebSocket subscription lifecycle — unsubscribe on component unmount | S | Med | Currently all subs live until site disposal |
| #12 | Lazy on-demand pagination for datasets | M | High | |
| #15 | Accessibility: ARIA, keyboard, screen reader | L | High | Deployment gate |
| #16 | CSP compliance: replace new Function() | M | Med | §12 risk |
| #21 | Optional Quarkus backend MVP | XL | High | Gates #22, #23 |

## Cross-Module

**We're blocking:**
- `casehubio/clinical` — can consume all new components + metric lookups with `groupBy(null, col("field"))` per #59 fix
- `casehubio/aml` — can now build pages app with Quinoa convention; WebSocket auth available if needed

## References

- Spec: `docs/superpowers/specs/2026-06-29-websocket-robustness-design.md`
- Blog: `blog/2026-06-30-mdp01-five-issues-one-connection.md`
- Review: `~/adr/casehub-pages/websocket-robustness-*/`
- Previous: `git show HEAD~1:HANDOFF.md`
