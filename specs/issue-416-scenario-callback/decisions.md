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
