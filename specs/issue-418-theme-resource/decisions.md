# Decisions — Issue #498: Scenario Lifecycle State

## D1: Configurable lifecycle engine, not fixed states

**Choice:** Build a configurable lifecycle engine where domains define their own lifecycle states and transitions in YAML. The existing draft/active/archived becomes the default configuration, not the only option.
**Alternatives:**
- Fixed lifecycle + extension points — keep DRAFT/ACTIVE/ARCHIVED universal, add hooks at transitions. Simpler but domains can't add states (e.g., compliance-review, staged).
- Scope down to versioning only — skip lifecycle redesign, just add version history. Misses the architectural opportunity.
**Rationale:** The platform builds a generic state machine engine (ScenarioDefinition) for domain content but hardcodes its own lifecycle. Different domains genuinely need different lifecycles — FSI needs compliance gates, SOC needs staged deployment, clinical trials need regulatory holds. A configurable engine lets each domain define its own.
**Trade-offs:** More complex than a simple enum. Domains must author lifecycle YAML configs. Migration from the existing hardcoded enum.
**Sources:** `ScriptLifecycleState.java` (existing enum), `ScenarioDefinition` (existing state machine model), fsitrading#50 (compliance use case)
**Exploration:** quick
**Status:** captured

## D2: Reuse ScenarioDefinition for lifecycle state machine

**Choice:** The lifecycle engine uses `ScenarioDefinition` as its state machine model. A lifecycle is defined as a ScenarioDefinition instance with empty step arrays — states represent lifecycle phases, transitions represent allowed state changes, guards represent approval requirements.
**Alternatives:**
- New lightweight LifecycleDefinition model — avoids ScenarioDefinition's execution baggage (steps, deadlines, orchestration). But creates a parallel model to maintain.
**Rationale:** The platform should eat its own cooking. ScenarioDefinition already has states, transitions, events, and guards. The parser, validator, and compiler already exist. Using them for lifecycle state management proves the model and avoids maintaining two state machine implementations.
**Trade-offs:** StateDefinition carries fields unused by lifecycle (steps, deadline, onFailure). These are empty/null — minor noise, not a correctness issue.
**Depends on:** D1 (configurable engine)
**Sources:** `packages/yaml-core/src/step/scenario/types.ts` (ScenarioDefinition, StateDefinition), `packages/yaml-core/src/step/scenario/` (parser, validator, compiler)
**Exploration:** quick
**Status:** captured

## D3: Scope — lifecycle engine + default config + ScriptRegistry wiring

**Choice:** Build the lifecycle engine, provide the default draft/active/archived YAML config, replace `ScriptLifecycleState` enum with engine-driven transitions, wire into ScriptRegistry. Versioning and approval workflow are separate issues.
**Alternatives:**
- Full scope (engine + versioning + approval) — too large, conflates infrastructure with features
- Engine only, no wiring — proves the model but delivers no user-visible change
**Rationale:** The engine + default config + wiring is the minimal deliverable that replaces the hardcoded enum with something configurable. Versioning is orthogonal (version history doesn't depend on which lifecycle model is used). Approval workflow requires guard infrastructure that builds on the engine.
**Trade-offs:** Versioning and approval deferred — need separate issues.
**Depends on:** D1 (configurable engine), D2 (reuse ScenarioDefinition)
**Sources:** Issue #498, `ScriptRegistry.java` (existing wiring)
**Exploration:** quick
**Status:** captured

## D4: Default lifecycle YAML lives in yaml-core

**Choice:** Ship the default lifecycle as a YAML file in `packages/yaml-core/src/step/scenario/lifecycle/`. Java mirrors the same file in `backend/playbook/src/main/resources/lifecycle/`.
**Alternatives:**
- Backend playbook resources only — Java owns the config, TS receives it via API. Single source of truth but TS can't validate offline.
- Both sides independently — simple but risks drift.
**Rationale:** yaml-core is the shared type system for both runtimes. The lifecycle definition is a platform concern, not a server concern. TS needs it for client-side validation (can this transition happen?) without a server round-trip.
**Trade-offs:** Two copies of the same YAML — must stay in sync. Mitigated by tests that compare them.
**Depends on:** D2 (reuse ScenarioDefinition)
**Sources:** `packages/yaml-core/src/step/scenario/` (existing scenario infrastructure)
**Exploration:** quick
**Status:** captured
