# HANDOFF — casehub-pages (slot 177)

Branch: `issue-411-auto-generate-zod-schemas`
Issue: casehubio/casehub-pages#411

## Last Session

Closed #408 (schema-driven YAML completion) — landed on main. Then started #411 (auto-generate Zod schemas from TypeScript interfaces). Completed design spec and implementation plan. No code written yet for #411.

Key finding: `displayer-desugar.ts` lines 374-385 has a dynamic passthrough loop that passes ANY undeclared YAML property through to component props. This is the root cause of type-unsafe YAML.

## Immediate Next Step

Execute the plan at `plans/2026-09-06-auto-generate-zod-schemas.md`. Three batches:

1. **Schema generator first** — run ts-morph generator on current interfaces to produce schemas
2. **Automated gap detection** — run desugarer with schema validation in warn mode against sample YAML. Warnings reveal all missing interface properties automatically. DON'T manually identify gaps.
3. **Fix interfaces from warnings** — add missing properties, regenerate schemas
4. **Replace passthrough** — swap passthrough loop with `schema.safeParse()`

**Important:** The plan as written says "audit interfaces first" (Task 1). Revise the execution order: generator → warn-mode → fix from warnings → regenerate. Automated gap detection replaces manual audit.

## Key Files

- Spec: `specs/issue-411-auto-generate-zod-schemas/2026-09-06-auto-generate-zod-schemas-design.md`
- Plan: `plans/2026-09-06-auto-generate-zod-schemas.md`
- Passthrough: `packages/pages-ui/src/parser/displayer-desugar.ts:374-385`
- Registry: `packages/pages-component/src/model/type-guards.ts:64-134`
