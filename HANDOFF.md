# HANDOFF — casehub-pages

## Last Session: 2026-11-24

Both #619 and #671 landed on main. Queue repopulated with CDI boilerplate reduction issues.

**Landed:**
- **#619** — `fixedWidth` auto-sizing for grid panels. Gap-aware formula, YAML round-trip, edit bridge preservation. 5 commits.
- **#671** — 7 hover bar action button tests (split, delete, merge, add-empty dispatch, bubbling). 2 commits.

**Filed:** #743 (full sizing model — widthMode, heightMode, fixedWidth, fixedHeight). M/Med.

## Immediate Next Step

Queue has #744 (S/Med) then #745 (S/Med). Use `work start`.

- **#744** — Auto-provide PlaybookConfig, RestDispatcher, GraphQLDispatcher as `@DefaultBean` producers in `casehub-pages-playbook-runtime`. Eliminates identical CDI boilerplate across Pages apps.
- **#745** — Auto-provide push infrastructure beans (SessionSender, PushEndpoint, JsonWriter, ConnectionRegistry) as `@DefaultBean` producers in `casehub-pages-push-runtime`.

Both block the zero-Java goal for casehubio/examples#212.

## Open Issues (by category)

**Queued:** #744 (CDI playbook defaults, S/Med), #745 (CDI push defaults, S/Med)
**New:** #743 (full sizing model, M/Med)
**Mid-size:** #646, #719, #725
**Large:** #669, #616, #92, #94
**Epics:** #426, #95
