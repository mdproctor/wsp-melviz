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

## D3: Shared YAML primitives via platform core

**Choice:** Extract `VariableResolver`, `ForEachExpander` (generified), CSV typed data source parser, `when` evaluator, `isTruthy()`, and named iteration groups into a shared platform module (`casehub-yaml-core`). Both desired-state and scenario automation compose these primitives into their own domain-specific compilation pipelines. Eidos is building this in platform now. Desired-state migrates to it after delivery.
**Alternatives:**
- Consistent-but-separate — each domain implements its own primitives with same syntax. Risks drift over time. Bug fixes happen twice.
- Full shared YAML framework — one compilation pipeline with SPIs. Host structures (graph nodes vs ordered steps) are too different for a shared framework. Premature.
**Rationale:** Consistency enforced by shared code, not discipline. CSV typed data sources land in both systems for free. The extraction boundary is clean — primitives, not framework. J2CL-compatible (no reflection, no CDI, no concurrency in core).
**Trade-offs:** Coordination dependency — scenario automation waits for platform delivery. Mitigated by Eidos building it now.
**Exploration:** quick
**Status:** captured

## D4: Control flow constructs — forEach, when, CSV data

**Choice:** Scenario automation adopts desired-state's control flow syntax: `forEach` on steps (with `as` + `in`), `when` conditionals, `${each.*}` interpolation, named `iterations` groups, and top-level `data` block for CSV sources. CSV first row declares `columnName:type` pairs (string, integer, boolean, decimal). Type enforcement at parse/load time. `forEach` + `when` can combine — condition evaluated per iteration value.
**Alternatives:**
- JSONata-based iteration — more powerful but requires learning JSONata. Inconsistent with desired-state.
- JSON array inline — verbose in YAML, no natural tabular feel.
**Rationale:** Direct alignment with desired-state. Teams familiar with one YAML surface recognize the other. Shared platform core (D3) means identical behavior, not just identical syntax.
**Trade-offs:** CSV is a less expressive data format than JSON — no nested structures. Sufficient for tabular iteration which is the primary use case.
**Depends on:** D3 (shared YAML core provides the primitives)
**Exploration:** quick
**Status:** captured

## D5: Script metadata — typed labels + free-form tags

**Choice:** Scripts carry both CaseHub typed labels (`namespace:value` pairs like `domain:helpdesk`, `capability:onboarding`) and free-form string tags (`["getting-started", "quick"]`). Typed labels enable faceted filtering in the library browser. Tags provide user-defined categories. Both are set in the YAML `meta` block and editable after upload.
**Alternatives:**
- Typed labels only — structured but rigid. Users can't add ad-hoc categories without defining label namespaces.
- Free-form tags only — flexible but unstructured. No faceted filtering, no platform consistency.
**Rationale:** Typed labels align with platform conventions (issue labels, work items). Tags cover the long tail of user-defined categories. Two filtering dimensions in the UI.
**Trade-offs:** Two metadata systems to manage. Mitigated by clear purpose separation — labels are platform-structured, tags are user-defined.
**Exploration:** quick
**Status:** captured

## D6: Registry manifest — JSON descriptor array

**Choice:** External registries serve a JSON array of `ScriptDescriptor` objects at the configured URL. Each descriptor includes: name, description, labels, tags, params schema, contentUrl (relative URL to fetch the actual YAML), and calls (list of scripts this one calls, for acyclic validation). The server fetches the manifest on first library browse, caches with TTL.
**Alternatives:**
- YAML index file — consistent with YAML tooling but JSON is more natural for HTTP APIs.
- OpenAPI-style discovery — most formal but heaviest to implement and consume.
**Rationale:** JSON is the standard HTTP response format. Simple, cacheable, no new format. The `calls` field enables the server to validate acyclicity across registry boundaries without fetching every script body.
**Trade-offs:** Manifest must be kept in sync with actual scripts. `contentUrl` relative resolution requires a well-defined base URL convention.
**Depends on:** D1 (registry aggregation)
**Exploration:** quick
**Status:** captured

## D7: Paste/upload — minimal metadata, edit later

**Choice:** On paste/upload, auto-detect `scenario` name from the YAML and save immediately to the server-side library. Metadata editing (description, labels, tags) is a separate action available from the library browser. Params are auto-extracted from `${params.*}` usage in the YAML for display but not validated until first run.
**Alternatives:**
- Guided form — on paste, present name/description/labels for confirmation. Structured but adds friction to the paste flow.
- Full editor — YAML editor with live preview + metadata sidebar. Capable but heavy for a quick paste.
**Rationale:** Fastest path from clipboard to library. The paste action is an import, not an authoring session. Metadata can be refined later. Auto-detection of name from the `scenario:` field means the script is immediately usable.
**Trade-offs:** Scripts land in the library without description or labels. Mitigated by auto-extracted params and the ability to edit metadata afterward.
**Exploration:** quick
**Status:** captured

## D8: Readiness probe — first-step ARIA target check

**Choice:** The library browser probes whether each script can start at the current browser state by checking if the first step's ARIA targets exist in the DOM. Uses the existing `findByRole()` from the tree walker. Result displayed as a green (ready) / amber (unknown — no ARIA targets in first step) / red (not ready — targets not found) indicator per script in the library browser. Probe runs on library open and on page navigation events.
**Alternatives:**
- Full preflight scan — walk all steps. Misleading since later steps depend on earlier mutations.
- Declarative requires — new metadata field. Adds authoring burden without clear advantage over the ARIA probe.
**Rationale:** The first step is the only reliable guarantee. After that, each step may mutate state. The probe reuses existing tree walker infrastructure (no new mechanism). ARIA targets are the canonical element-targeting contract (PP-20260817-a11y01).
**Trade-offs:** Steps without ARIA targets (e.g., `navigate`, `call`, service-targeted steps) produce amber (unknown) rather than green. Acceptable — amber means "can't verify, try it."
**Exploration:** quick
**Status:** captured

## D9: Library browser integrated into scenario controller

**Choice:** The library browser is a new view mode within the existing `<scenario-controller>` component. The controller gains a library icon that toggles between outline view (current) and library view. Library view shows a filterable, searchable list of scripts from the registry with labels, tags, readiness indicators, and a run button. Selecting a script loads it and switches to outline view. Paste/upload accessible via a button in the library view header.
**Alternatives:**
- Separate `<scenario-library>` component — decoupled but requires external wiring. The browse → select → run flow spans two components.
- Standalone page + embedded — both a `/scenario/library` page and an embeddable component. More surface area than needed initially.
**Rationale:** Browse → select → run is one flow. Integrating into the controller keeps it cohesive. The controller already manages connection, state, and transport — adding script selection is a natural extension. Paste/upload in the library view is where the author expects it.
**Trade-offs:** Controller component grows in scope. Acceptable — the component's purpose is "scenario operations" broadly, not just transport controls.
**Depends on:** D1 (registry API), D5 (labels/tags for filtering), D8 (readiness indicator)
**Exploration:** quick
**Status:** captured

## D10: Name collision — overwrite uploaded, protect bundled

**Choice:** Uploaded scripts can overwrite other uploaded scripts (same name = update). Bundled scripts are protected — upload with a bundled name is rejected with an error naming the collision. External registry scripts are read-only (never overwritten by upload). The library browser shows provenance (bundled / uploaded / registry-name) per script.
**Alternatives:**
- Namespace by source — each source gets a prefix (bundled/foo, uploaded/foo). No collisions but more complex naming, breaks `call` references that assume flat namespace.
- Always reject duplicates — no two scripts share a name. Forces rename on every update.
**Rationale:** Uploaded scripts are user-owned — the user who uploaded can update. Bundled scripts are distribution-owned — protecting them prevents accidental override of curated content. Same-name upload is the natural "save new version" gesture.
**Trade-offs:** No version history for uploaded scripts — overwrite is destructive. Acceptable for current stage. Versioning can be added later without changing the collision semantics.
**Depends on:** D1 (registry), D7 (paste/upload)
**Exploration:** quick
**Status:** captured
