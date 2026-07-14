# casehub-pages Session Handover — 2026-07-14

## Last Session

Closed #148 and #134 — DataSourceController URL auto-routing and pipeline refresh redesign. `createSourceFactory` routes by URL scheme (ws/sse/rest). Pipeline `refreshDataSet` now re-fetches from source instead of serving stale cache; `deliverDataSet` handles cache re-delivery. Added `pages-refresh-request` event and stale-while-revalidate with manager-level TTL (60s default). Adversarial design review (4 rounds, 13 issues) caught infinite recursion before implementation.

## Branch State

Both repos on main. Pause stack empty.

## Immediate Next Step

Pick up #180 (remove @ts-nocheck — L/Low, mechanical), #142 (Scenario Engine), or one of the new deferred issues (#181-#185).

## What's Left

- #180 — remove // @ts-nocheck from 40 example files (full type conformance) · L · Low
- #181 — Manager-level eviction for unreferenced datasets · S · Med
- #182 — Per-dataset TTL configuration beyond refreshTime · S · Med
- #183 — DataSourceController pipeline integration · M · High
- #185 — totalPath implementation in restSource · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #180 | Remove @ts-nocheck — full type conformance for examples | L | Low | ModelMesh is reference impl |
| #159 | pages-schema-form — JSON Schema forms | L | High | Unblocks Developer Registration |
| #142 | Scenario Engine — composition, triggers, demo UI | L | High | Plan ready |
| #181 | Manager-level eviction (unreferenced datasets) | S | Med | Deferred from #134 |
| #182 | Per-dataset TTL beyond refreshTime | S | Med | Deferred from #134 |
| #185 | totalPath in restSource | S | Low | Deferred from #148 |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
