## D1: YAML surface for structural decorators

**Choice:** Non-scalar value disambiguation — decorator key with object value `key: param: [steps]` or array value `key: [steps]` becomes structural
**Alternatives:**
- `then` companion — clean but overloads `then` which is currently paired only with `if`; requires validation changes in both Walkers
- New companion keyword (e.g. `do:`) — already in REMOVED_KEYS, would need un-removal
**Rationale:** Matches existing `parallel` pattern (array = unnamed steps, object = named branches). `forEach: "member in members":` is valid YAML and reads naturally. The `:` after the scalar value is the syntactic marker that turns decorator into structural.
**Trade-offs:** Authors must remember the trailing `:` — forgetting it makes the step a decorator instead of structural, which is a silent semantic difference. Linting/LSP can catch this.
**Sources:** walker.ts:250-267 (structural/decorator classification), walker.ts:298-312 (parallel array/object disambiguation), Walker.java:106-138 (Java Walker classification)
**Exploration:** quick
**Status:** captured

## D2: Output format for structural decorators

**Choice:** BlockStep with decorator in decorators map — identical output to `block: [steps] + decorator: param`
**Alternatives:**
- New step kinds per decorator (ForEachStep, RetryStep, etc.) — requires new interfaces, new runtime handlers, breaks existing decorator chain execution
**Rationale:** Pure syntactic sugar. The resolved output is the same as today's `block` + decorator form. Runtime changes: zero. The runtime already knows how to execute decorators on BlockSteps.
**Trade-offs:** The step viewer and any tooling that inspects `kind: 'block'` won't distinguish between explicit `block:` and promoted decorator. This is acceptable — the YAML source is the authoring surface, not the resolved tree.
**Sources:** walker.ts:293-296 (block resolution), Walker.java:210-215 (Java block resolution)
**Exploration:** quick
**Status:** captured
