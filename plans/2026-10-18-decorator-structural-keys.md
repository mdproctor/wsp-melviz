# Decorator Structural Keys Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #571 — Promote decorators to structural keys in the Walker
**Issue group:** #571

**Goal:** Allow `forEach`, `retry`, `timeout`, `when`, `mutex`, `loop` to
directly contain child steps when their value is non-scalar, eliminating
the `block` wrapper.

**Architecture:** Value-type disambiguation in the Walker's classification
loop. Object value = structural with parameter (key is parameter, value is
steps array). Array value = structural without parameter. Scalar value =
decorator (unchanged). Output is identical to `block` + decorator — a
`BlockStep` with the decorator in its decorators map. No runtime changes.

**Tech Stack:** TypeScript (yaml-core Walker, schema-composer), Java
(platform yaml-step-runtime Walker)

## Global Constraints

- Pre-release platform — no backward compatibility concerns
- Both TS and Java Walkers must be kept in sync
- Resolved output must be identical to the current `block` + decorator
  form so no runtime changes are needed
- Object-value form parameter is extracted as a string (JS object keys
  are always strings) — the runtime already handles string/number
  coercion for decorator values like `retry`

---

## Batch 1: pages repo — TS Walker + schema

### Task 1: TS Walker structural decorator support

**Files:**
- Modify: `packages/yaml-core/src/step/walker.ts:193-424`
- Test: `packages/yaml-core/src/step/walker.test.ts`

**Interfaces:**
- Produces: `PROMOTABLE_DECORATORS` set (not exported — internal constant)
- Produces: Updated `resolveOne()` that classifies non-scalar promotable
  decorator values as structural and resolves them to `BlockStep`

- [ ] **Step 1: Write failing tests for object-value form**

Add a new `describe` block to `walker.test.ts`:

```typescript
describe('structural decorator forms', () => {
  it('resolves forEach with object value as structural block', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill'), click: makeEntry('click') });
    const steps = [
      { forEach: { 'member in members': [{ fill: {} }, { click: {} }] } },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(2);
      expect(resolved[0].steps[0]!.kind).toBe('plugin');
      expect(resolved[0].decorators['forEach']).toBe('member in members');
    }
  });

  it('resolves retry with object value as structural block', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { retry: { '3': [{ fill: {} }] } },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(1);
      expect(resolved[0].decorators['retry']).toBe('3');
    }
  });

  it('resolves when with object value as structural block', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { when: { '${params.enabled}': [{ fill: {} }] } },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(1);
      expect(resolved[0].decorators['when']).toBe('${params.enabled}');
    }
  });

  it('resolves timeout with object value as structural block', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { timeout: { '30s': [{ fill: {} }] } },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(1);
      expect(resolved[0].decorators['timeout']).toBe('30s');
    }
  });

  it('resolves mutex with object value as structural block', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { mutex: { 'order-lock': [{ fill: {} }] } },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(1);
      expect(resolved[0].decorators['mutex']).toBe('order-lock');
    }
  });

  it('resolves loop with object value as structural block', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { loop: { 'continuous': [{ fill: {} }] }, cancel: 'done', timeout: '1m' },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(1);
      expect(resolved[0].decorators['loop']).toBe('continuous');
      expect(resolved[0].decorators['cancel']).toBe('done');
    }
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run packages/yaml-core/src/step/walker.test.ts`
Expected: FAIL — structural decorator forms not yet handled

- [ ] **Step 3: Add PROMOTABLE_DECORATORS constant**

In `walker.ts`, after `STRUCTURAL_COMPANIONS` (line 202), add:

```typescript
const PROMOTABLE_DECORATORS = new Set(['forEach', 'retry', 'timeout', 'when', 'mutex', 'loop']);
```

- [ ] **Step 4: Add classification logic in resolveOne()**

In `walker.ts`, in the `resolveOne()` classification loop, insert a new
`else if` branch **before** the existing `DECORATOR_KEYS.has(key)` check
(line 267). The new branch must come immediately before it:

Replace:
```typescript
    } else if (DECORATOR_KEYS.has(key) && !(key === 'if' && step['then'] !== undefined)) {
```

With:
```typescript
    } else if (PROMOTABLE_DECORATORS.has(key) && (Array.isArray(value) || (typeof value === 'object' && value !== null))) {
      if (structuralType) {
        throw new Error(`${stepPath}: step has both structural type '${structuralType}' and structural decorator '${key}'`);
      }
      structuralType = key;
      structuralValue = value;
    } else if (DECORATOR_KEYS.has(key) && !(key === 'if' && step['then'] !== undefined)) {
```

- [ ] **Step 5: Add structural resolution handler**

In `walker.ts`, after the quorum handler (line 424) and before the
standalone delay check (line 426), add:

```typescript
    if (structuralType && PROMOTABLE_DECORATORS.has(structuralType)) {
      let substeps: ResolvedStep[];
      if (Array.isArray(structuralValue)) {
        substeps = Walker.resolveAtDepth(structuralValue as Record<string, unknown>[], catalog, depth + 1, `${stepPath} → ${structuralType}`, seenNames);
        decorators[structuralType] = true;
      } else {
        const entries = Object.entries(structuralValue as Record<string, unknown>);
        if (entries.length !== 1) {
          throw new Error(`${stepPath}: structural '${structuralType}' must have exactly one key (the parameter), got ${entries.length}`);
        }
        const [param, stepsRaw] = entries[0]!;
        if (!Array.isArray(stepsRaw)) {
          throw new Error(`${stepPath}: structural '${structuralType}' parameter '${param}' must map to a steps array`);
        }
        substeps = Walker.resolveAtDepth(stepsRaw as Record<string, unknown>[], catalog, depth + 1, `${stepPath} → ${structuralType}`, seenNames);
        decorators[structuralType] = param;
      }
      return { kind: 'block', name: stepName, steps: substeps, decorators };
    }
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/step/walker.test.ts`
Expected: PASS — all 6 object-value form tests green

- [ ] **Step 7: Write tests for array-value form, errors, and combined decorators**

Append to the `structural decorator forms` describe block:

```typescript
  it('resolves retry with array value as structural block (no parameter)', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill'), click: makeEntry('click') });
    const steps = [
      { retry: [{ fill: {} }, { click: {} }] },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(2);
      expect(resolved[0].decorators['retry']).toBe(true);
    }
  });

  it('combines structural decorator with scalar decorators', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { forEach: { 'member in members': [{ fill: {} }] }, retry: 3, timeout: '30s' },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('block');
    if (resolved[0]!.kind === 'block') {
      expect(resolved[0].steps).toHaveLength(1);
      expect(resolved[0].decorators['forEach']).toBe('member in members');
      expect(resolved[0].decorators['retry']).toBe(3);
      expect(resolved[0].decorators['timeout']).toBe('30s');
    }
  });

  it('preserves step name on structural decorator', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { step: 'my-loop', forEach: { 'item in items': [{ fill: {} }] } },
    ];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.name).toBe('my-loop');
  });

  it('treats scalar decorator value alongside action key as decorator (unchanged)', () => {
    const catalog = makeCatalog({ click: makeEntry('click') });
    const steps = [{ click: { role: 'button' }, retry: 3, forEach: 'item in items' }];
    const resolved = Walker.resolve(steps, catalog);
    expect(resolved[0]!.kind).toBe('plugin');
    expect(resolved[0]!.decorators['retry']).toBe(3);
    expect(resolved[0]!.decorators['forEach']).toBe('item in items');
  });

  it('throws when two promotable decorators both have non-scalar values', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { forEach: { 'a in b': [{ fill: {} }] }, retry: { '3': [{ fill: {} }] } },
    ];
    expect(() => Walker.resolve(steps, catalog)).toThrow(/structural.*structural/i);
  });

  it('throws when structural decorator conflicts with block', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { block: [{ fill: {} }], forEach: { 'a in b': [{ fill: {} }] } },
    ];
    expect(() => Walker.resolve(steps, catalog)).toThrow(/structural.*structural/i);
  });

  it('throws when structural decorator has object with multiple keys', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { forEach: { 'a in b': [{ fill: {} }], 'c in d': [{ fill: {} }] } },
    ];
    expect(() => Walker.resolve(steps, catalog)).toThrow(/exactly one key/i);
  });

  it('throws when structural decorator object value is not an array', () => {
    const catalog = makeCatalog({ fill: makeEntry('fill') });
    const steps = [
      { forEach: { 'a in b': 'not-an-array' } },
    ];
    expect(() => Walker.resolve(steps, catalog)).toThrow(/steps array/i);
  });

  it('throws when structural decorator conflicts with action key', () => {
    const catalog = makeCatalog({ click: makeEntry('click') });
    const steps = [
      { click: {}, forEach: { 'a in b': [{ click: {} }] } },
    ];
    expect(() => Walker.resolve(steps, catalog)).toThrow(/structural.*action/i);
  });
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `npx vitest run packages/yaml-core/src/step/walker.test.ts`
Expected: PASS — all structural decorator tests green

- [ ] **Step 9: Run full test suite**

Run: `npx vitest run packages/yaml-core/`
Expected: PASS — no regressions in existing tests

- [ ] **Step 10: Commit**

```bash
git add packages/yaml-core/src/step/walker.ts packages/yaml-core/src/step/walker.test.ts
git commit -m "feat(#571): promote decorators to structural keys in TS Walker

Add PROMOTABLE_DECORATORS set (forEach, retry, timeout, when, mutex,
loop). When these keys carry a non-scalar value (object or array),
they become structural — the Walker resolves their child steps and
produces a BlockStep with the decorator attached. Scalar values
remain decorators (unchanged).

Refs #571"
```

### Task 2: Schema composer structural forms

**Files:**
- Modify: `packages/yaml-core/src/step/schema-composer.ts:36-59`

**Interfaces:**
- Consumes: `PROMOTABLE_DECORATORS` concept (list of 6 decorator names)
- Produces: Updated JSON schema `oneOf` entries allowing structural forms

- [ ] **Step 1: Add structural form schema entries**

In `schema-composer.ts`, after the existing `race` oneOf entry (line 59),
add entries for each promotable decorator:

```typescript
    for (const dk of ['forEach', 'retry', 'timeout', 'when', 'mutex', 'loop']) {
      const structuralProps: Record<string, unknown> = {
        [dk]: { oneOf: [{ type: 'object' }, { type: 'array' }] },
        step: { type: 'string' },
      };
      for (const otherDk of DECORATOR_KEYS) {
        if (otherDk !== dk) structuralProps[otherDk] = {};
      }
      oneOf.push({ type: 'object', properties: structuralProps, required: [dk] });
    }
```

- [ ] **Step 2: Run full yaml-core tests**

Run: `npx vitest run packages/yaml-core/`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add packages/yaml-core/src/step/schema-composer.ts
git commit -m "feat(#571): add structural decorator forms to schema composer

Refs #571"
```

## Batch 2: platform repo — Java Walker

### Task 3: Java Walker structural decorator support

**Files:**
- Modify: `platform/yaml-step-runtime/src/main/java/io/casehub/yaml/step/catalog/Walker.java`
- Test: `platform/yaml-step-runtime/src/test/java/io/casehub/yaml/step/catalog/WalkerTest.java`

**Interfaces:**
- Produces: Java `PROMOTABLE_DECORATORS` set (private constant)
- Produces: Updated `resolveOne()` mirroring TS Walker changes

- [ ] **Step 1: Write failing tests**

Add tests to `WalkerTest.java`:

```java
@Test
void resolvesForEachWithObjectValueAsStructuralBlock() {
    var catalog = catalogWith("fill", "click");
    var steps = List.of(
        Map.<String, Object>of("forEach", Map.of("member in members",
            List.of(Map.of("fill", Map.of()), Map.of("click", Map.of()))))
    );
    var resolved = Walker.resolve(steps, catalog);
    assertThat(resolved).hasSize(1);
    assertThat(resolved.get(0)).isInstanceOf(ResolvedStep.BlockStep.class);
    var block = (ResolvedStep.BlockStep) resolved.get(0);
    assertThat(block.steps()).hasSize(2);
    assertThat(block.decorators()).containsEntry("forEach", "member in members");
}

@Test
void resolvesRetryWithArrayValueAsStructuralBlock() {
    var catalog = catalogWith("fill");
    var steps = List.of(
        Map.<String, Object>of("retry", List.of(Map.of("fill", Map.of())))
    );
    var resolved = Walker.resolve(steps, catalog);
    assertThat(resolved.get(0)).isInstanceOf(ResolvedStep.BlockStep.class);
    var block = (ResolvedStep.BlockStep) resolved.get(0);
    assertThat(block.steps()).hasSize(1);
    assertThat(block.decorators()).containsEntry("retry", true);
}

@Test
void combinesStructuralDecoratorWithScalarDecorators() {
    var catalog = catalogWith("fill");
    var steps = List.of(
        Map.<String, Object>of(
            "forEach", Map.of("item in items", List.of(Map.of("fill", Map.of()))),
            "retry", 3,
            "timeout", "30s")
    );
    var resolved = Walker.resolve(steps, catalog);
    var block = (ResolvedStep.BlockStep) resolved.get(0);
    assertThat(block.decorators()).containsEntry("forEach", "item in items");
    assertThat(block.decorators()).containsEntry("retry", 3);
    assertThat(block.decorators()).containsEntry("timeout", "30s");
}

@Test
void scalarDecoratorAlongsideActionKeyUnchanged() {
    var catalog = catalogWith("click");
    var steps = List.of(
        Map.<String, Object>of("click", Map.of(), "retry", 3)
    );
    var resolved = Walker.resolve(steps, catalog);
    assertThat(resolved.get(0)).isInstanceOf(ResolvedStep.PluginStep.class);
    assertThat(resolved.get(0).decorators()).containsEntry("retry", 3);
}

@Test
void throwsWhenTwoPromotableDecoratorsHaveNonScalarValues() {
    var catalog = catalogWith("fill");
    var steps = List.of(
        Map.<String, Object>of(
            "forEach", Map.of("a in b", List.of(Map.of("fill", Map.of()))),
            "retry", Map.of("3", List.of(Map.of("fill", Map.of()))))
    );
    assertThatThrownBy(() -> Walker.resolve(steps, catalog))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("structural");
}

@Test
void throwsWhenStructuralDecoratorObjectHasMultipleKeys() {
    var catalog = catalogWith("fill");
    var steps = List.of(
        Map.<String, Object>of("forEach", Map.of(
            "a in b", List.of(Map.of("fill", Map.of())),
            "c in d", List.of(Map.of("fill", Map.of()))))
    );
    assertThatThrownBy(() -> Walker.resolve(steps, catalog))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("exactly one key");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl yaml-step-runtime -Dtest=WalkerTest -f platform/pom.xml`
Expected: FAIL — compilation errors or test failures

- [ ] **Step 3: Add PROMOTABLE_DECORATORS constant**

In `Walker.java`, after `REMOVED_KEYS` (line 29), add:

```java
private static final Set<String> PROMOTABLE_DECORATORS = Set.of(
        "forEach", "retry", "timeout", "when", "mutex", "loop");
```

- [ ] **Step 4: Add classification logic**

In `Walker.java`, in the `resolveOne()` key loop, add a new branch
**before** the `RESERVED_KEYS.contains(key)` check (line 118):

```java
            } else if (PROMOTABLE_DECORATORS.contains(key)
                       && (e.getValue() instanceof List || e.getValue() instanceof Map)) {
                if (structuralType != null) {
                    throw new IllegalArgumentException(
                            path + " → Step " + index + ": step has both structural type '"
                            + structuralType + "' and structural decorator '" + key + "'");
                }
                structuralType = key;
                structuralValue = e.getValue();
            } else if (RESERVED_KEYS.contains(key)) {
```

- [ ] **Step 5: Add structural resolution handler**

In `Walker.java`, after the `resume` handler (line 317) and before the
`matchedAction` check (line 318), add:

```java
        if (structuralType != null && PROMOTABLE_DECORATORS.contains(structuralType)) {
            if (structuralValue instanceof List) {
                @SuppressWarnings("unchecked")
                var stepList = (List<Map<String, Object>>) structuralValue;
                var resolved = resolve(stepList, registry, depth + 1,
                        stepPath + " → " + structuralType, seenNames);
                decorators.put(structuralType, true);
                return new ResolvedStep.BlockStep(stepName, resolved, decorators);
            }
            @SuppressWarnings("unchecked")
            var paramMap = (Map<String, Object>) structuralValue;
            if (paramMap.size() != 1) {
                throw new IllegalArgumentException(
                        stepPath + ": structural '" + structuralType
                        + "' must have exactly one key (the parameter), got " + paramMap.size());
            }
            var entry = paramMap.entrySet().iterator().next();
            String param = String.valueOf(entry.getKey());
            if (!(entry.getValue() instanceof List)) {
                throw new IllegalArgumentException(
                        stepPath + ": structural '" + structuralType
                        + "' parameter '" + param + "' must map to a steps array");
            }
            @SuppressWarnings("unchecked")
            var stepList = (List<Map<String, Object>>) entry.getValue();
            var resolved = resolve(stepList, registry, depth + 1,
                    stepPath + " → " + structuralType, seenNames);
            decorators.put(structuralType, param);
            return new ResolvedStep.BlockStep(stepName, resolved, decorators);
        }
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl yaml-step-runtime -Dtest=WalkerTest -f platform/pom.xml`
Expected: PASS

- [ ] **Step 7: Run full module tests**

Run: `mvn test -pl yaml-step-runtime -f platform/pom.xml`
Expected: PASS — no regressions

- [ ] **Step 8: Commit**

```bash
git -C platform add yaml-step-runtime/src/
git -C platform commit -m "feat(#571): promote decorators to structural keys in Java Walker

Mirror TS Walker changes: PROMOTABLE_DECORATORS set, classification
of non-scalar values as structural, resolution to BlockStep.

Refs casehubio/casehub-pages#571"
```

## References

- [2026-10-18-decorator-structural-keys-design.md] — design spec
- [packages/yaml-core/src/step/walker.ts:250-267] — TS Walker classification loop
- [packages/yaml-core/src/step/walker.ts:293-424] — TS Walker structural handlers
- [platform/yaml-step-runtime/.../Walker.java:99-331] — Java Walker
- [packages/yaml-core/src/step/schema-composer.ts] — JSON schema generation
- [packages/yaml-core/src/step/walker.test.ts] — existing TS Walker tests
- [GitHub #571] — focal issue
