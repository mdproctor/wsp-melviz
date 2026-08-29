# Decisions — Script Library & Automation Platform

## D1: Flat catalog with server-side registry

**Choice:** A `ScriptRegistry` service on the backend aggregates scripts from three sources: bundled (classpath), uploaded (server filesystem), and external (registry URLs). Each source contributes `ScriptDescriptor` entries (name, description, labels, params schema, source provenance). The controller fetches the catalog via `GET /scenario/library` with optional label filters. Scripts fetched on-demand by name.
**Alternatives:**
- Client-side aggregation — controller fetches manifests from multiple sources directly, merges in browser. Exposes registry URLs, cross-origin issues, pushes validation to wrong layer.
- Distributed catalog with sync — server pre-syncs all external registries at startup, caches locally. Over-engineered for current stage, adds sync lifecycle and cache invalidation.
**Rationale:** Matches existing REST pattern (outline, state endpoints). Simple aggregation — no new infrastructure. External registries are just another source returning the same descriptor format. Label filtering is server-side (efficient). The `call` command for composability resolves against the same registry.
**Trade-offs:** External registry latency on first browse (not pre-cached). Mitigated by lazy fetch + UI loading state.
**Exploration:** quick
**Status:** captured

## D2: Script composability via `call` command

**Choice:** A step can include `{action: call, script: "setup-project", params: {name: "Acme"}}`. The orchestrator resolves the script from the registry, validates acyclicity against the call graph built at load time, resolves params through the chain (caller context > explicit params > callee defaults), and inlines the callee's steps into the execution plan. Callee inherits the caller's full resolved context. Results from callee steps available to subsequent caller steps via `${stepName.field}`.
**Alternatives:**
- Pre-processor expansion — expand all `call` references into flattened step list before execution. Loses caller/callee boundary, step name collisions, harder to debug.
- Sub-orchestrator — each `call` spawns a nested orchestrator session. Protocol designed for single-orchestrator (D8 in executor protocol). Nesting adds session complexity without built infrastructure.
**Rationale:** Composability is a first-class command, not a pre-processing step. The orchestrator already manages step dispatch and sequencing. Call graph validation at load time uses the registry to resolve all transitive `call` references and detect cycles before execution starts.
**Trade-offs:** Inlined steps share the caller's session namespace — step names must be unique across caller + all callees. Mitigated by automatic name prefixing (`callee-name.step-name`).
**Depends on:** D1 (registry provides script resolution for `call` references)
**Exploration:** quick
**Status:** captured
