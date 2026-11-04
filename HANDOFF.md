# HANDOFF — casehub-pages

## Status

On branch `issue-609-forms-validation`. Scaffolded, no implementation started. Ready for brainstorm → plan → execute.

## What this branch covers

Epic #609 — Forms Validation. Three child issues:

| # | Issue | Scale | Blocked by |
|---|-------|-------|------------|
| #610 | Server-side JSON Schema constraint enforcement | S/Med | — |
| #611 | Structured validation error response + field-level error mapping | S/Med | — |
| #612 | Validate-on-blur for form inputs | S/Low | #611 |

#610 and #611 are independent. #612 depends on #611's field-level error display.

## Context

The JSON Apps forms infrastructure has client-side validation (`PagesSchemaForm.validate()` + `validateField()`) but the server only enforces types via `CollectionSchema.validate()`. Client validation is bypassed by direct API calls. Server errors are opaque strings — no field-level mapping back to the form.

## Key files

- `packages/pages-component/src/model/field-validation.ts` — client-side constraint checking (reference implementation)
- `packages/pages-viz/src/form-inputs/PagesSchemaForm.ts` — form component, `validate()` on submit
- `packages/pages-viz/src/form-inputs/PagesFormInput.ts` — abstract input base class
- `backend/data/src/main/java/io/casehub/pages/data/CollectionSchema.java` — server validation (type-only)
- `backend/data/src/main/java/io/casehub/pages/data/SchemaViolationException.java` — current exception
- `packages/pages-runtime/src/adapters/rest-adapter.ts` — save pipeline, opaque error handling

## Previous work

Epic #592 (JSON Apps Runtime) completed — all 12 issues shipped. This epic builds on that foundation.
