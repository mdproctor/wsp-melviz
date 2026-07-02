# casehub-pages Session Handover — 2026-07-02

## Last Session

Completed the backend data module (#21): two new Maven modules — `data` (SPI, REST relay + query, SSRF-protected proxy, DTOs) and `data-sql` (SQL push-down via Quarkus Agroal, parameterized SqlQueryBuilder, ResultSetMapper). 50 tests green. Also cleared four trailing items from #88 (prod-profile exclusion test, layoutSaveDelayMs tests, PLATFORM.md updates). Did a full gap analysis against the original GWT DashBuilder (~80% parity), created 6 new issues for uncovered gaps, and organized all 15 remaining issues under epic #95.

## Branch State

Branch `issue-21-data-module-backend` open in both repos. Main has 1 unpushed commit (`e786bc4` — trailing tests from #88). Pause stack empty.

## What's Left

- PLATFORM.md update — layout serialization + backend modules applied in casehub-parent (committed, not pushed) · XS · Low
- Frontend `RemoteDataService` for `/api/dataset/query` (#89) — connects SQL push-down to frontend pipeline · M · Med
- Named datasource resolution in SqlDataProvider — MVP uses default datasource only · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #86 | Publish 0.3.0 with workbench primitives + terminal | S | Low | CI green, unblocked |
| #89 | Frontend RemoteDataService for /query | M | Med | Blocked by #21 branch close |
| #90 | Server-side data caching (Caffeine) | S | Low | |
| #95 | Epic: GWT migration remaining gaps | — | — | Tracks all 15 open issues |

## Cross-Module

**We're unblocking:**
- `casehubio/connectors` — chat demo needs `casehub-pages-auth` for user identity

## References

- Design spec: `docs/superpowers/specs/2026-07-02-data-module-backend-design.md`
- Blog: `blog/2026-07-02-mdp01-completing-the-data-module.md`
- Epic: casehubio/casehub-pages#95
- Previous: `git show HEAD~1:HANDOFF.md`
