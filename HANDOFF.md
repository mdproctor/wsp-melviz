*Updated: #246, #236 closed — removed from backlog.*

# casehub-pages Session Handover — 2026-07-28

## Last Session

Implemented Maven SNAPSHOT cross-repo dependency management (ADR-0001, #246). Migrated 12 repos: pages and blocks-ui produce `casehub-pages-npm` / `casehub-blocks-ui-npm` Maven artifacts via `yarn pack`, all 9 consumer repos consume via `maven-dependency-plugin:unpack` + Yarn `portal:` resolutions. Eliminated three different CI workarounds (sed hacks, sibling checkouts, broken builds). Updated `ui-architecture.md` in parent, cleaned stale references across 5 repos, removed 5 vestigial `.npmrc` files. Garden entries GE-20260728-93e8db (yarn pack gotcha) and GE-20260728-4f59e3 (portal technique).

## Immediate Next Step

Verify CI passes on casehub-pages first (push to main triggers `maven-publish.yml` which now builds `npm-packages/` and `webapp/`). Once pages artifacts are on GitHub Maven Packages, trigger blocks-ui CI to verify it resolves pages-npm correctly. Then spot-check one consumer repo (openclaw is simplest).

## What's Left

- Verify CI end-to-end: pages → blocks-ui → one consumer app · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #245 | Compact theme picker with flyout popover | S | Med | |
| #222 | Nested object/array schema support for schema-form | L | High | Recursive rendering |
| #180 | Remove // @ts-nocheck from example files | L | Low | 40 files, mechanical |
