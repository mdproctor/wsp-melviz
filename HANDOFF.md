# HANDOFF — casehub-pages

## Status

On main. #598 landed and closed. Epic #592 (JSON Apps Runtime) has 3 remaining issues in the queue.

## What just shipped

#598 — Apps management view. `AppManagementResource` at `/api/apps` with list, detail, and lifecycle (start/stop) endpoints. Extended `_apps` schema with `enabled`, `displayName`, `autoSeed`, `maxDocuments` fields. Lifecycle gating in `PageResource` and `AppCatalogueResource` — disabled apps return 404, data preserved in SQLite. `apps-management.page.yaml` with list, settings (schema-form), and lifecycle (action-button) tabs. 4 commits, 35 tests.

Key decisions: settings as _apps document fields (not PreferenceStore — lifecycle state is app data, not user preference); gating at resource layer (store stays unaware of lifecycle); dynamic apps only (static classpath apps always enabled); cache eviction on stop (prevents stale-page-with-404-data UX).

## Queue

| # | Issue | Scale | Notes |
|---|-------|-------|-------|
| #605 | Gallery server-connected rendering | S/Low | Wire gallery to render apps with real backend data |
| #606 | Template param escape upgrade | XS/Low | XSS protection for `#{params.x}` in dynamic pages |
| #607 | Wire extendDataSetScope | XS/Low | Dynamic pages declaring their own datasets |

## References

- `backend/data-json/src/main/java/io/casehub/pages/data/json/AppManagementResource.java` — new resource
- `backend/data-json/src/main/java/io/casehub/pages/data/json/AppRegistry.java` — extended schema, isAppEnabled
- `examples/samples/Domain Showcases/apps-management/apps-management.page.yaml` — management page
- Design spec: `specs/issue-598-apps-management-view/2026-11-04-apps-management-view-design.md`
- Blog: `blog/2026-11-04-mdp01-app-lifecycle-without-the-platform.md`
- Epic: casehubio/casehub-pages#592
