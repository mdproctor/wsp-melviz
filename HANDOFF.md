*Updated: #185 closed — removed from immediate next step.*

# casehub-pages Session Handover — 2026-07-28

## Last Session

Shipped `casehub-pages-ui-static` (#247) — new Maven artifact serving pre-built theme CSS and component ESM bundle via `META-INF/resources/pages/`. Three-tier artifact separation: build-time source (`npm`), runtime design system (`static`), runtime dashboard app (`webapp`). Design review caught 13 issues including `build:bundle` separation from `build` (now protocol PP-20260728-3676e1) and Lit ESM validation gotcha (garden GE-20260728-6d585d). Pushed to upstream.

## Immediate Next Step

Verify CI passes — push to main triggers `maven-publish.yml` which now also builds and publishes `casehub-pages-ui-static`. Once the artifact is on GitHub Maven Packages, add it as a dependency in a consumer app's pom.xml to validate the static artifact integration.

## What's Left

- Verify CI end-to-end: pages → blocks-ui → one consumer app · S · Low
- Hygiene: blog on `issue-192` branch and specs on `issue-144` branch never reached workspace main · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #245 | Compact theme picker with flyout popover | S | Med | |
| #222 | Nested object/array schema support for schema-form | L | High | Recursive rendering |
| #180 | Remove // @ts-nocheck from example files | L | Low | 40 files, mechanical |
