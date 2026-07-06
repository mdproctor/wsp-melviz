# casehub-pages Session Handover — 2026-07-06

## Last Session

Closed #131: SSEManager extracted from blocks-ui-core to pages-data with named SSE event support. Published `@casehubio/pages-data@0.2.1` to GitHub Packages. Migrated blocks-ui to consume from pages-data (on blocks-ui branch `issue-022-pages-data-table`). Three garden entries submitted (Yarn 4 npmScopes precedence, Vitest fake timers override stubs, Yarn 4 npm whoami broken with GitHub Packages). Also fixed blocks-ui `MockSSESource` to support `addEventListener` for named events.

## Branch State

Both repos on `main`. Pause stack empty.

## What's Left

- PLATFORM.md update — parent#349 filed for pages capability entry update · S · Low
- Minor implementation findings — #120 (null safety, as-any casts, mixin composition test, dist import path) · S · Low
- CLAUDE.md `@casehub/` vs `@casehubio/` prefix fix — #123 filed · XS · Low
- Fleet Monitor gauge overlap — #133 filed (pre-existing, gauge canvas extends beyond grid cell) · M · Med
- Map/postMessage structured clone — #130 filed (ComponentMessage.properties Map doesn't survive postMessage) · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #130 | Fix ComponentMessage.properties Map → Record for postMessage | S | Med | Wire format protocol target |
| #123 | Fix CLAUDE.md @casehub/ → @casehubio/ scope prefix | XS | Low | Quick fix |
| #133 | Fleet Monitor gauge overlap | M | Med | YAML layout or gauge sizing |
| #125 | Event-mode push API epic | L | Med | Epic: #126 EventBroadcaster, #127 Lit EventStreamController, #128 docs |
| #75 | Drag-and-drop panel rearrangement | L | High | Future epic |
| #77 | Floating/popout panels | M | High | Future epic |

## Cross-Module

**We're unblocking:**
- `casehubio/blocks-ui` — blocks-ui#21: 1 of 8 migration items done (SSE); remaining: tokens, mixins, SchemaForm, DatasetContract, BlocksTheme, visual regression
- `casehubio/blocks-ui` — blocks-ui#20: adopt pages-primitives
- `casehub-connectors/notifications` — event-mode push API (#125 epic); SSEManager named events now ready

## References

- Blog: `blog/2026-07-06-mdp01-sse-crosses-the-repo-line.md`
- Previous: `git show HEAD~1:HANDOFF.md`
