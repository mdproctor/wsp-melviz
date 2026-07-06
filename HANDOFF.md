# casehub-pages Session Handover — 2026-07-06

## Last Session

Established TypeScript/pages protocols (#118). Created 4 project-level protocols (CSS design tokens, event contract, web component strategy, dataset contract) and 2 universal garden protocols (Lit immutable collections, CustomEvent shadow DOM) in a new `web/` namespace — the first non-Java protocol namespace in the garden. Spec went through 4-round adversarial design review (17 issues, 16 verified, 1 accepted, $14.56). Also audited token adoption across the codebase, filed #124 (visual alignment cleanup), and created #125 epic (event-mode push API — EventBroadcaster, Lit EventStreamController, documentation) after analysing how pages' push architecture could serve the platform notification use case.

## Branch State

Both repos on `main`. Pause stack empty.

## What's Left

- PLATFORM.md update — parent#349 filed for pages capability entry update · S · Low
- npm publish for 0.2.0 — #121 filed · XS · Low
- Minor implementation findings — #120 (null safety, as-any casts, mixin composition test, dist import path) · S · Low
- CLAUDE.md `@casehub/` vs `@casehubio/` prefix fix — #123 filed · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #124 | Align examples and auth widgets with OKLCH token system | M | Med | Examples gallery entirely off-token; auth widgets use hardcoded #007bff |
| #125 | Event-mode push API epic | — | — | Epic: #126 EventBroadcaster (S/Low), #127 Lit EventStreamController (M/Med), #128 docs (XS/Low) |
| #122 | Iframe component API protocols | S | Low | Future — if patterns emerge |
| #119 | Trie-based TopicRegistry wildcard index | S | Med | Only if pattern count warrants |
| #75 | Drag-and-drop panel rearrangement | L | High | Future epic |
| #77 | Floating/popout panels | M | High | Future epic |

## Cross-Module

**We're unblocking:**
- `casehubio/blocks-ui` — blocks-ui#21: migrate from `--blocks-*` to `--pages-*` tokens; protocol now authoritative at `docs/protocols/casehub/css-design-tokens.md`
- `casehubio/blocks-ui` — blocks-ui#20: adopt `<pages-filter-chips>` and `<pages-scope-selector>` from pages-primitives
- `casehub-connectors/notifications` — event-mode push API (#125 epic) addresses their SSE vs WebSocket concerns; `createEventConnection` already works for rich domain events

## References

- Spec: `docs/specs/2026-07-05-ts-pages-protocols-design.md`
- Review workspace: `~/adr/casehub-pages/ts-pages-protocols-20260705-224610/`
- Blog: `blog/2026-07-06-mdp01-writing-rules-from-the-garden.md`
- Previous: `git show HEAD~1:HANDOFF.md`
