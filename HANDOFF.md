# casehub-pages Session Handover — 2026-06-23

## Last Session

Branch `issue-26-quinoa-convention` closed. Established the Quinoa convention: renamed npm scope `@casehub` → `@casehubio` (GitHub Packages requirement), removed redundant iframe ECharts (#28), added publishing fields, moved pages-viz side-effect import into pages-runtime, bumped to 0.2.0, created CI publish workflow + reference template + convention doc. Workspace `ctx.py` path resolution fixed (stale melviz symlinks).

## Branch State

Both repos on `main`. Fork and blessed current (`a0af1ea`).

## What's Left

- Workspace `WORKSPACE_OK` fix needs no further action (symlinks corrected this session)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #12 | Lazy on-demand pagination for datasets | M | High | Next substantive feature |
| #14 | Domain-specific example dashboards | L | Med | Gallery stable |
| #16 | CSP compliance: replace new Function() | M | Med | Security concern in §12 |

## References

- Blog: `blog/2026-06-23-mdp02-publishable-quinoa-convention.md`
- Convention doc: `docs/quinoa-convention.md`
- Template: `templates/quinoa-host/`
- Spec: `docs/superpowers/specs/2026-06-23-quinoa-convention-design.md`
- Previous: `git show HEAD~1:HANDOFF.md`
