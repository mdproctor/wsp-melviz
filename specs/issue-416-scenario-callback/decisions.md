# Decisions — Scenario Callback (#416)

## D1: Explicit callbackToken parameter

**Choice:** Accept `callbackToken` as an explicit parameter alongside `callbackUrl` and `dispatchId`. The engine stores it in session state and sends it as `X-Casehub-Callback-Token` header on the callback POST.
**Alternatives:**
- Generic `extraHeaders` map — more flexible but adds surface area; caller must know the header name
- No auth / caller's concern — engine just POSTs with no auth; workers-scenario would need a pre-authenticated URL
**Rationale:** Matches the `WorkerCallbackResource` contract directly. The callback token is a first-class concept in the workers platform — making it explicit keeps the coupling visible and prevents misuse of a generic headers mechanism.
**Trade-offs:** Couples the scenario engine's callback API to the CaseHub callback token convention. If a future caller uses a different auth scheme, they'd need the extraHeaders approach. Acceptable — the primary consumer is workers-scenario.
**Sources:** `WorkerCallbackResource.java` (X-Casehub-Callback-Token header check), `PendingCompletion.java` (callbackToken field)
**Exploration:** quick
**Status:** captured

## D3: Callback output contains all step results keyed by step name

**Choice:** `output = {"stepName1": {result1}, "stepName2": {result2}, ...}` — the full execution trace, keyed by step name (or label if no name).
**Alternatives:**
- Last step result only — simple but loses intermediate results that the caller may need for correlation
- Flat merged map — simple but keys could collide across steps (e.g. two steps both returning "ticketId")
**Rationale:** Workers-scenario dispatches scenarios as case steps. The orchestrating case engine needs to see what each scenario step produced to route subsequent case steps. Keying by step name gives a predictable structure with no collision risk.
**Trade-offs:** Payload grows linearly with step count. Acceptable — scenarios are typically 5-20 steps, each with a small result map.
**Sources:** `PushRequest.StepResult` (stepName + result fields), `WorkerCompletionPayload.output` (Map<String,Object>)
**Exploration:** quick
**Status:** captured

## D2: Completion detection inside ScenarioOrchestrator

**Choice:** Add completion check at the end of `onStepResult()`. When all steps are done (or `on-error:stop` triggers a failure), the orchestrator fires the callback directly via an injected HTTP client. No separate service or CDI event.
**Alternatives:**
- CDI event + separate CallbackService — more decoupled but adds a class for a single-consumer event; over-engineered for this use case
- Push wire observer — listens on `scenario:state` for progress=1.0; fragile, races with session cleanup, and couples to broadcast timing
**Rationale:** The orchestrator already owns session lifecycle (start, stop, step completion). Adding the callback here keeps the "when is a scenario done?" logic in one place. The HTTP POST is fire-and-forget — no complex flow to decouple.
**Trade-offs:** ScenarioOrchestrator gains an HTTP client dependency. Acceptable — the class already coordinates all session concerns.
**Sources:** `ScenarioOrchestrator.java:187-203` (onStepResult method), `ScenarioOrchestrator.java:123-146` (state/progress calculation)
**Exploration:** quick
**Status:** captured
