# casehub-pages Session Handover — 2026-06-23

## Last Session

Branch `issue-26-quinoa-convention` closed. Established the Quinoa convention: renamed npm scope `@casehub` → `@casehubio`, removed iframe ECharts (#28), published packages at 0.2.0, created CI workflow + template + convention doc. Post-close: created foundational "adopt casehub-pages" issues across host repos and updated their UI issues to build on pages.

## Branch State

Both repos on `main`. Fork and blessed current (`a0af1ea`).

## Cross-Module

**We're blocking** (host repos waiting on us to verify CI publish works):
- drafthouse #75 — needs published packages or local link to start Quinoa integration · S · Low
- claudony #161 — same · S · Low
- devtown #92 — same · XS · Low

## What's Left

- Verify CI publish workflow runs on casehubio/casehub-pages (triggered by the push, should be green)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #12 | Lazy on-demand pagination for datasets | M | High | Next substantive feature |
| #14 | Domain-specific example dashboards | L | Med | Gallery stable |
| #16 | CSP compliance: replace new Function() | M | Med | Security concern in §12 |

## Cross-Repo Issues Created This Session

| Repo | Issue | Title |
|------|-------|-------|
| drafthouse | #75 | Adopt casehub-pages for UI composition via Quinoa |
| drafthouse | #53 updated | Brainstorming UI — depends on #75, build on pages |
| drafthouse | #60 updated | Selection-scoped channels — depends on #75, build on pages |
| drafthouse | #74 closed | Absorbed into #75 |
| claudony | #161 | Adopt casehub-pages for UI composition via Quinoa |
| claudony | #75 updated | Three-panel dashboard — depends on #161, build on pages |
| claudony | #84 updated | Project setup wizard — depends on #161 |
| claudony | #85 updated | Agent onboarding — depends on #161 |
| devtown | #92 | Adopt casehub-pages for UI composition via Quinoa |
| devtown | #85 updated | PR governance dashboard — depends on #92 |

## References

- Blog: `blog/2026-06-23-mdp02-publishable-quinoa-convention.md`
- Convention doc: `docs/quinoa-convention.md`
- Template: `templates/quinoa-host/`
- Previous: `git show HEAD~1:HANDOFF.md`
