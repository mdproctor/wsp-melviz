# Decisions — Issue #500: Scenario Outcome Tracking

## D1: Scope — recording + emission, no panel

**Choice:** Pages captures structured execution outcomes and emits them via push protocol. CBR linkage is engine's domain (#1190). Outcome panel UI deferred to a separate issue.
**Alternatives:**
- Full scope (recording + emission + panel) — too broad, panel is independent UI work
- Recording only, defer CBR entirely — misses the emission contract that engine needs
**Rationale:** CBR is a unified cross-runtime concern — one bridge for TS pages and Java platform, owned by engine. Pages' job is to produce a well-structured outcome record and publish it. Panel is consumption, not production.
**Trade-offs:** Panel UI delayed; but the data model and emission are the foundation — panel can't exist without them.
**Sources:** Issue #500, fsitrading#50 (epic C13), engine#1190
**Exploration:** quick
**Status:** captured

## D2: Outcome data model lives in yaml-core

**Choice:** `packages/yaml-core/src/step/outcome.ts` — exports `ExecutionOutcome`, `StepOutcomeEntry`, and the `OutcomeCollector` class.
**Alternatives:**
- New `pages-outcome` package — clean separation but adds a dependency for a small type surface
- `pages-aria` alongside scheduler — ties the model to one runtime, Java platform can't share the shape
**Rationale:** yaml-core already owns `Result`, `StepError`, `JournalEntry`, and `ScenarioDefinition`. The outcome model is a natural extension of the step type system. Java platform mirrors yaml-core's type shapes.
**Trade-offs:** yaml-core grows slightly; but outcome types depend on existing yaml-core types, so co-location avoids circular dependencies.
**Sources:** `packages/yaml-core/src/step/simulation-runtime.ts`, `packages/yaml-core/src/step/walker.ts`
**Exploration:** quick
**Status:** captured

## D3: Emission via pages-event CustomEvent

**Choice:** Emit `scenario:outcome` topic via the existing `pages-event` CustomEvent mechanism on the scheduler's EventTarget.
**Alternatives:**
- OutcomeEmitter callback interface — more flexible but adds a new injection point
- Both event + callback — belt and suspenders, two paths to maintain
**Rationale:** Zero new infrastructure. Same channel as `scenario:state`, `scenario:step`, `scenario:queue`. Push protocol already forwards pages-event to subscribers — engine's CBR bridge subscribes like any other consumer.
**Trade-offs:** Tied to DOM EventTarget; but the scheduler already requires one, and tests construct synthetic EventTargets.
**Sources:** `packages/pages-aria/src/playbook/scheduler.ts` (existing event emission pattern)
**Exploration:** quick
**Status:** captured

## D4: Outcome granularity — summary + per-step detail

**Choice:** Top-level summary (duration, step counts, final state, trigger source) plus a per-step entry array with timing and result kind.
**Alternatives:**
- Summary only — lightweight but insufficient for detail view and analysis
- Full trace (+ state machine transitions, barrier events, deadlines, journal) — rich but potentially large, premature
**Rationale:** Enough for history list, detail view, and future comparison without excessive payload size. Per-step entries give the CBR bridge meaningful granularity for case similarity matching.
**Trade-offs:** No state machine transition trace — if needed later, can be added as an optional field without breaking the model.
**Sources:** Issue #500 requirements (execution summary, steps executed/failed, final state, deadlines)
**Exploration:** quick
**Status:** captured

## D5: Journal entries included conditionally

**Choice:** Attach `JournalEntry[]` to the outcome only when simulation was active during the run. Optional field (`journal?: JournalEntry[]`).
**Alternatives:**
- Always exclude — treats journal as simulation-internal, loses analysis data
- Always include — empty array for non-simulation runs, slightly noisier
**Rationale:** Simulation runs need journal data for verifying invocation patterns and data realism. Non-simulation runs don't produce journal entries. Conditional inclusion keeps outcomes lean by default while preserving full simulation context when relevant.
**Trade-offs:** Consumers must check for presence; but `undefined` vs array is a clean discriminator.
**Sources:** `InvocationJournal` in `packages/yaml-core/src/step/simulation-runtime.ts`
**Exploration:** quick
**Status:** captured

## D6: Collection approach — OutcomeCollector inside scheduler

**Choice:** An `OutcomeCollector` class instantiated by the scheduler at run start. Timestamps step dispatches and completions within `tickLoop()`. `collector.finalize()` produces the `ExecutionOutcome` at completion.
**Alternatives:**
- Post-hoc reconstruction from events — listen to `scenario:step`/`scenario:state` events externally and reconstruct. Zero scheduler changes but current events lack step-start timestamps and consistent step names.
**Rationale:** Minimal scheduler changes (3-4 lines in `dispatchStep()` and completion path). Collector is independently testable. Single responsibility: scheduler runs steps, collector observes.
**Trade-offs:** Small coupling between scheduler and collector; but the collector is a passive observer, not a control flow participant.
**Depends on:** D2 (outcome types in yaml-core), D4 (granularity)
**Sources:** `packages/pages-aria/src/playbook/scheduler.ts` (`dispatchStep()`, `tickLoop()` completion path)
**Exploration:** quick
**Status:** captured
