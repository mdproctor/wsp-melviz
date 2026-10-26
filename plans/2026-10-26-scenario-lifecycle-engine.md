# Scenario Lifecycle Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #498 — Scenario lifecycle state — draft/active/archived versioning with approval workflow
**Issue group:** #498

**Goal:** Replace the hardcoded `ScriptLifecycleState` enum with a configurable lifecycle engine backed by `ScenarioDefinition`, enabling domains to define custom lifecycle states and transitions in YAML.

**Architecture:** The `LifecycleEngine` class wraps a `ScenarioDefinition` and exposes `canTransition`, `validateTransition`, and `availableTransitions` methods. A default lifecycle YAML (draft→active→archived with revise/resurrect) replaces the existing enum. `ScriptRegistry` and `UploadedScriptSource` switch from `ScriptLifecycleState` enum + `OrcStateMachine` to `String` state + `LifecycleEngine` validation.

**Tech Stack:** TypeScript (yaml-core, vitest), Java 21 (Quarkus, JUnit 5, AssertJ), YAML

## Global Constraints

- The default lifecycle YAML must match `ScenarioParser`'s array-valued state format — each state's value is an array of entries
- `initialState` is determined by the first key in the `states` block (no explicit field)
- TS `ScriptDescriptor` interface in `pages-aria/src/controller/library-view.ts` already uses `state: string` — no TS-side changes needed there
- Two copies of `default-lifecycle.yaml` (TS and Java) — cross-runtime consistency test required
- The `LifecycleEngine` is stateless regarding persistence — it validates transitions, caller owns state

---

## Batch 1: TS LifecycleEngine + Default Lifecycle YAML

### Task 1: LifecycleEngine (TypeScript)

**Files:**
- Create: `packages/yaml-core/src/step/scenario/lifecycle/lifecycle-engine.ts`
- Create: `packages/yaml-core/src/step/scenario/lifecycle/default-lifecycle.yaml`
- Create: `packages/yaml-core/src/step/scenario/lifecycle/lifecycle-engine.test.ts`
- Create: `packages/yaml-core/src/step/scenario/lifecycle/index.ts`
- Modify: `packages/yaml-core/src/step/scenario/index.ts`

**Interfaces:**
- Consumes: `ScenarioDefinition`, `StateDefinition`, `ScenarioParser` from `../types.js` and `../parser.js`
- Produces: `LifecycleEngine` class with `canTransition(currentState: string, targetState: string): boolean`, `validateTransition(currentState: string, targetState: string): void`, `availableTransitions(currentState: string): readonly string[]`, `readonly initialState: string`, `readonly states: readonly string[]`; `defaultLifecycleEngine(): LifecycleEngine` factory function

- [ ] **Step 1: Create the default lifecycle YAML**

Create `packages/yaml-core/src/step/scenario/lifecycle/default-lifecycle.yaml`:

```yaml
name: script-lifecycle
states:
  draft:
    - next: active
  active:
    - on:
        archive: archived
        revise: draft
  archived:
    - on:
        resurrect: draft
```

- [ ] **Step 2: Write the failing tests**

Create `packages/yaml-core/src/step/scenario/lifecycle/lifecycle-engine.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { LifecycleEngine, defaultLifecycleEngine } from './lifecycle-engine.js';
import { ScenarioParser } from '../parser.js';

describe('LifecycleEngine', () => {
  describe('default lifecycle', () => {
    const engine = defaultLifecycleEngine();

    it('has draft as initial state', () => {
      expect(engine.initialState).toBe('draft');
    });

    it('lists all states', () => {
      expect(engine.states).toEqual(['draft', 'active', 'archived']);
    });

    it('allows draft → active', () => {
      expect(engine.canTransition('draft', 'active')).toBe(true);
    });

    it('rejects draft → archived (no direct transition)', () => {
      expect(engine.canTransition('draft', 'archived')).toBe(false);
    });

    it('allows active → archived (archive event)', () => {
      expect(engine.canTransition('active', 'archived')).toBe(true);
    });

    it('allows active → draft (revise event)', () => {
      expect(engine.canTransition('active', 'draft')).toBe(true);
    });

    it('allows archived → draft (resurrect event)', () => {
      expect(engine.canTransition('archived', 'draft')).toBe(true);
    });

    it('rejects archived → active (no direct transition)', () => {
      expect(engine.canTransition('archived', 'active')).toBe(false);
    });

    it('validates allowed transition silently', () => {
      expect(() => engine.validateTransition('draft', 'active')).not.toThrow();
    });

    it('validateTransition throws on invalid transition', () => {
      expect(() => engine.validateTransition('draft', 'archived'))
        .toThrow("Cannot transition from 'draft' to 'archived'");
    });

    it('returns available transitions from active', () => {
      expect(engine.availableTransitions('active')).toEqual(['archived', 'draft']);
    });

    it('returns available transitions from draft', () => {
      expect(engine.availableTransitions('draft')).toEqual(['active']);
    });

    it('returns available transitions from archived', () => {
      expect(engine.availableTransitions('archived')).toEqual(['draft']);
    });

    it('throws on unknown state', () => {
      expect(() => engine.availableTransitions('nonexistent'))
        .toThrow("Unknown state: 'nonexistent'");
    });
  });

  describe('custom lifecycle', () => {
    it('supports additional states', () => {
      const def = ScenarioParser.parse('custom-lifecycle', {
        states: {
          draft: [{ next: 'review' }],
          review: [{ on: { approve: 'active', reject: 'draft' } }],
          active: [{ on: { archive: 'archived' } }],
          archived: 'terminal',
        },
      });
      const engine = new LifecycleEngine(def);

      expect(engine.states).toEqual(['draft', 'review', 'active', 'archived']);
      expect(engine.canTransition('draft', 'review')).toBe(true);
      expect(engine.canTransition('draft', 'active')).toBe(false);
      expect(engine.canTransition('review', 'active')).toBe(true);
      expect(engine.canTransition('review', 'draft')).toBe(true);
      expect(engine.availableTransitions('review')).toEqual(['active', 'draft']);
    });
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn vitest run packages/yaml-core/src/step/scenario/lifecycle/lifecycle-engine.test.ts`
Expected: FAIL — module not found

- [ ] **Step 4: Implement LifecycleEngine**

Create `packages/yaml-core/src/step/scenario/lifecycle/lifecycle-engine.ts`:

```typescript
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { dirname, resolve } from 'node:path';
import { parse as parseYaml } from 'yaml';
import type { ScenarioDefinition, StateDefinition } from '../types.js';
import { ScenarioParser } from '../parser.js';

export class LifecycleEngine {
  private readonly definition: ScenarioDefinition;

  constructor(definition: ScenarioDefinition) {
    this.definition = definition;
  }

  get initialState(): string {
    return this.definition.initialState;
  }

  get states(): readonly string[] {
    return [...this.definition.states.keys()];
  }

  canTransition(currentState: string, targetState: string): boolean {
    const state = this.definition.states.get(currentState);
    if (!state) return false;
    return this.reachableTargets(state).includes(targetState);
  }

  validateTransition(currentState: string, targetState: string): void {
    if (!this.canTransition(currentState, targetState)) {
      throw new Error(
        `Cannot transition from '${currentState}' to '${targetState}'`
      );
    }
  }

  availableTransitions(currentState: string): readonly string[] {
    const state = this.definition.states.get(currentState);
    if (!state) {
      throw new Error(`Unknown state: '${currentState}'`);
    }
    return this.reachableTargets(state);
  }

  private reachableTargets(state: StateDefinition): string[] {
    const targets: string[] = [];
    if (state.next) {
      targets.push(state.next);
    }
    for (const transition of Object.values(state.events)) {
      if (transition.type === 'simple' || transition.type === 'guarded') {
        if (!targets.includes(transition.target)) {
          targets.push(transition.target);
        }
      } else if (transition.type === 'match-based') {
        for (const c of transition.cases) {
          if (!targets.includes(c.target)) {
            targets.push(c.target);
          }
        }
      }
    }
    return targets;
  }
}

export function defaultLifecycleEngine(): LifecycleEngine {
  const dir = dirname(fileURLToPath(import.meta.url));
  const yamlPath = resolve(dir, 'default-lifecycle.yaml');
  const raw = parseYaml(readFileSync(yamlPath, 'utf-8')) as Record<string, unknown>;
  const name = (raw['name'] as string) ?? 'script-lifecycle';
  const definition = ScenarioParser.parse(name, raw);
  return new LifecycleEngine(definition);
}
```

- [ ] **Step 5: Create the barrel export**

Create `packages/yaml-core/src/step/scenario/lifecycle/index.ts`:

```typescript
export { LifecycleEngine, defaultLifecycleEngine } from './lifecycle-engine.js';
```

- [ ] **Step 6: Re-export from the scenario index**

Add to `packages/yaml-core/src/step/scenario/index.ts`:

```typescript
export { LifecycleEngine, defaultLifecycleEngine } from './lifecycle/index.js';
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn vitest run packages/yaml-core/src/step/scenario/lifecycle/lifecycle-engine.test.ts`
Expected: all 13 tests PASS

- [ ] **Step 8: Run the full scenario test suite for regression**

Run: `yarn vitest run packages/yaml-core/src/step/scenario/`
Expected: all existing scenario tests PASS alongside new lifecycle tests

- [ ] **Step 9: Commit**

```bash
git add packages/yaml-core/src/step/scenario/lifecycle/ packages/yaml-core/src/step/scenario/index.ts
git commit -m "feat(yaml-core): LifecycleEngine — configurable state machine for script lifecycle (#498)"
```

---

## Batch 2: Java LifecycleEngine + ScriptDescriptor Migration

### Task 2: Java LifecycleEngine

**Files:**
- Create: `backend/playbook/src/main/java/io/casehub/pages/playbook/LifecycleEngine.java`
- Create: `backend/playbook/src/main/resources/lifecycle/default-lifecycle.yaml`
- Create: `backend/playbook/src/test/java/io/casehub/pages/playbook/LifecycleEngineTest.java`

**Interfaces:**
- Consumes: `ScenarioParser.parse()` from yaml-core (Java), `ScenarioDefinition`, `StateDefinition` from yaml-core types
- Produces: `LifecycleEngine` class with `boolean canTransition(String, String)`, `void validateTransition(String, String)`, `List<String> availableTransitions(String)`, `String initialState()`, `List<String> states()`; `static LifecycleEngine defaultEngine()` factory

- [ ] **Step 1: Copy the default lifecycle YAML to Java resources**

Create `backend/playbook/src/main/resources/lifecycle/default-lifecycle.yaml` — identical content to the TS version:

```yaml
name: script-lifecycle
states:
  draft:
    - next: active
  active:
    - on:
        archive: archived
        revise: draft
  archived:
    - on:
        resurrect: draft
```

- [ ] **Step 2: Write the failing tests**

Create `backend/playbook/src/test/java/io/casehub/pages/playbook/LifecycleEngineTest.java`:

```java
package io.casehub.pages.playbook;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class LifecycleEngineTest {

    final LifecycleEngine engine = LifecycleEngine.defaultEngine();

    @Test
    void initialState_isDraft() {
        assertThat(engine.initialState()).isEqualTo("draft");
    }

    @Test
    void states_returnsAll() {
        assertThat(engine.states()).containsExactly("draft", "active", "archived");
    }

    @Test
    void canTransition_draftToActive() {
        assertThat(engine.canTransition("draft", "active")).isTrue();
    }

    @Test
    void canTransition_draftToArchived_rejected() {
        assertThat(engine.canTransition("draft", "archived")).isFalse();
    }

    @Test
    void canTransition_activeToArchived() {
        assertThat(engine.canTransition("active", "archived")).isTrue();
    }

    @Test
    void canTransition_activeToDraft_revise() {
        assertThat(engine.canTransition("active", "draft")).isTrue();
    }

    @Test
    void canTransition_archivedToDraft_resurrect() {
        assertThat(engine.canTransition("archived", "draft")).isTrue();
    }

    @Test
    void canTransition_archivedToActive_rejected() {
        assertThat(engine.canTransition("archived", "active")).isFalse();
    }

    @Test
    void validateTransition_throwsOnInvalid() {
        assertThatThrownBy(() -> engine.validateTransition("draft", "archived"))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Cannot transition from 'draft' to 'archived'");
    }

    @Test
    void validateTransition_passesOnValid() {
        engine.validateTransition("draft", "active");
    }

    @Test
    void availableTransitions_fromActive() {
        assertThat(engine.availableTransitions("active")).containsExactly("archived", "draft");
    }

    @Test
    void availableTransitions_fromDraft() {
        assertThat(engine.availableTransitions("draft")).containsExactly("active");
    }

    @Test
    void availableTransitions_fromArchived() {
        assertThat(engine.availableTransitions("archived")).containsExactly("draft");
    }

    @Test
    void availableTransitions_unknownState_throws() {
        assertThatThrownBy(() -> engine.availableTransitions("nonexistent"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Unknown state");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl backend/playbook -Dtest=LifecycleEngineTest -f backend/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 4: Implement Java LifecycleEngine**

Create `backend/playbook/src/main/java/io/casehub/pages/playbook/LifecycleEngine.java`:

The Java side needs to parse the lifecycle YAML and build a `ScenarioDefinition`-equivalent model. Since the Java `ScenarioDefinition` lives in the platform yaml-core Maven artifact, we reuse the same parser infrastructure.

Check whether `io.casehub.yaml.core.step.scenario.ScenarioParser` exists in the Maven dependency `casehub-platform-yaml-core`. If it does, use it directly. If not, the Java `LifecycleEngine` must include a minimal YAML parser for the lifecycle format (states → transitions map).

The implementation mirrors the TS API:

```java
package io.casehub.pages.playbook;

import io.casehub.yaml.core.step.scenario.ScenarioDefinition;
import io.casehub.yaml.core.step.scenario.ScenarioParser;
import io.casehub.yaml.core.step.scenario.StateDefinition;
import org.yaml.snakeyaml.Yaml;

import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class LifecycleEngine {

    private final ScenarioDefinition definition;

    public LifecycleEngine(ScenarioDefinition definition) {
        this.definition = definition;
    }

    public String initialState() {
        return definition.initialState();
    }

    public List<String> states() {
        return List.copyOf(definition.states().keySet());
    }

    public boolean canTransition(String currentState, String targetState) {
        StateDefinition state = definition.states().get(currentState);
        if (state == null) return false;
        return reachableTargets(state).contains(targetState);
    }

    public void validateTransition(String currentState, String targetState) {
        if (!canTransition(currentState, targetState)) {
            throw new IllegalStateException(
                "Cannot transition from '" + currentState + "' to '" + targetState + "'");
        }
    }

    public List<String> availableTransitions(String currentState) {
        StateDefinition state = definition.states().get(currentState);
        if (state == null) {
            throw new IllegalArgumentException("Unknown state: '" + currentState + "'");
        }
        return reachableTargets(state);
    }

    private List<String> reachableTargets(StateDefinition state) {
        List<String> targets = new ArrayList<>();
        if (state.next() != null && !state.next().isEmpty()) {
            targets.add(state.next());
        }
        for (var transition : state.events().values()) {
            for (String target : transition.targets()) {
                if (!targets.contains(target)) {
                    targets.add(target);
                }
            }
        }
        return List.copyOf(targets);
    }

    @SuppressWarnings("unchecked")
    public static LifecycleEngine defaultEngine() {
        try (InputStream is = LifecycleEngine.class.getResourceAsStream("/lifecycle/default-lifecycle.yaml")) {
            Map<String, Object> raw = new Yaml().load(is);
            String name = (String) raw.getOrDefault("name", "script-lifecycle");
            ScenarioDefinition def = ScenarioParser.parse(name, (Map<String, Object>) raw);
            return new LifecycleEngine(def);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load default lifecycle", e);
        }
    }
}
```

**Important:** The exact Java implementation depends on the `ScenarioParser` and `ScenarioDefinition` API in the `casehub-platform-yaml-core` Maven dependency. Before writing, verify the Java API with `ide_find_class` for `ScenarioParser` in `project_and_libraries` scope. If the Java API differs from TS (method names, return types), adapt the implementation to match the Java API while keeping the `LifecycleEngine` public interface identical.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl backend/playbook -Dtest=LifecycleEngineTest -f backend/pom.xml`
Expected: all 13 tests PASS

- [ ] **Step 6: Commit**

```bash
git add backend/playbook/src/main/java/io/casehub/pages/playbook/LifecycleEngine.java backend/playbook/src/main/resources/lifecycle/default-lifecycle.yaml backend/playbook/src/test/java/io/casehub/pages/playbook/LifecycleEngineTest.java
git commit -m "feat(playbook): Java LifecycleEngine — mirrors TS implementation (#498)"
```

### Task 3: ScriptDescriptor enum → String migration

**Files:**
- Delete: `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptLifecycleState.java` (use `ide_refactor_safe_delete`)
- Modify: `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptDescriptor.java:10` — `ScriptLifecycleState state` → `String state`
- Modify: `backend/playbook/src/test/java/io/casehub/pages/playbook/ScriptDescriptorTest.java` — update all `ScriptLifecycleState` references to `String`
- Modify: `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptDescriptorExtractor.java:45-46` — `ScriptLifecycleState` constants → `String` values

**Interfaces:**
- Consumes: nothing new
- Produces: `ScriptDescriptor.state()` returns `String` instead of `ScriptLifecycleState`

- [ ] **Step 1: Update test expectations first**

Modify `ScriptDescriptorTest.java`:
- Change `descriptor_hasState` to assert `desc.state()` equals `"draft"` (String)
- Change `descriptor_defaultsStateToActive_whenNull` to assert equals `"active"`
- Change `descriptor_backwardCompatConstructor_defaultsToActive` to assert equals `"active"`
- Remove all `ScriptLifecycleState` imports

```java
@Test
void descriptor_hasState() {
    var desc = new ScriptDescriptor("test", "desc", List.of(), List.of(),
                                    List.of(), List.of(), ScriptProvenance.UPLOADED,
                                    "draft", List.of());
    assertThat(desc.state()).isEqualTo("draft");
}

@Test
void descriptor_defaultsStateToActive_whenNull() {
    var desc = new ScriptDescriptor("test", "desc", List.of(), List.of(),
                                    List.of(), List.of(), ScriptProvenance.BUNDLED,
                                    (String) null, List.of());
    assertThat(desc.state()).isEqualTo("active");
}

@Test
void descriptor_backwardCompatConstructor_defaultsToActive() {
    var desc = new ScriptDescriptor("test", "desc", List.of(), List.of(),
                                    List.of(), List.of(), ScriptProvenance.BUNDLED, List.of());
    assertThat(desc.state()).isEqualTo("active");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl backend/playbook -Dtest=ScriptDescriptorTest -f backend/pom.xml`
Expected: FAIL — type mismatch

- [ ] **Step 3: Modify ScriptDescriptor record**

Change `ScriptDescriptor.java` line 10: `ScriptLifecycleState state` → `String state`
Change compact constructor default: `if (state == null) state = "active";`
Remove the `ScriptLifecycleState` import.

```java
public record ScriptDescriptor(String name, String description,
                                List<String> labels, List<String> tags,
                                List<ParamDescriptor> params, List<String> calls,
                                ScriptProvenance provenance,
                                String state,
                                List<Map<String, String>> firstStepTargets) {
    public ScriptDescriptor {
        if (labels == null) labels = List.of();
        if (tags == null) tags = List.of();
        if (params == null) params = List.of();
        if (calls == null) calls = List.of();
        if (firstStepTargets == null) firstStepTargets = List.of();
        if (state == null) state = "active";
    }

    public ScriptDescriptor(String name, String description,
                            List<String> labels, List<String> tags,
                            List<ParamDescriptor> params, List<String> calls,
                            ScriptProvenance provenance,
                            List<Map<String, String>> firstStepTargets) {
        this(name, description, labels, tags, params, calls, provenance, null, firstStepTargets);
    }
}
```

- [ ] **Step 4: Update ScriptDescriptorExtractor**

Modify `ScriptDescriptorExtractor.java` lines 45-46 — replace `ScriptLifecycleState` constants with `String`:

```java
// Before:
ScriptLifecycleState state = provenance == ScriptProvenance.UPLOADED
        ? ScriptLifecycleState.DRAFT : ScriptLifecycleState.ACTIVE;

// After:
String state = provenance == ScriptProvenance.UPLOADED
        ? "draft" : "active";
```

Remove the `ScriptLifecycleState` import from this file.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl backend/playbook -Dtest=ScriptDescriptorTest -f backend/pom.xml`
Expected: all tests PASS

- [ ] **Step 6: Delete ScriptLifecycleState.java**

Use `ide_refactor_safe_delete` to remove `ScriptLifecycleState.java`. This will fail if there are remaining references — which is expected since `UploadedScriptSource` and `ScriptRegistry` still use it. Those are updated in Task 4. If safe delete fails, defer deletion to end of Task 4.

- [ ] **Step 7: Commit**

```bash
git add backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptDescriptor.java backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptDescriptorExtractor.java backend/playbook/src/test/java/io/casehub/pages/playbook/ScriptDescriptorTest.java
git commit -m "refactor(playbook): ScriptDescriptor state field — enum to String (#498)"
```

---

## Batch 3: ScriptRegistry + UploadedScriptSource Wiring

### Task 4: Wire LifecycleEngine into UploadedScriptSource and ScriptRegistry

**Files:**
- Modify: `backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/UploadedScriptSource.java` — replace `OrcStateMachine<ScriptLifecycleState>` with String state, rename `transition()` to `setState()`, remove `createStateMachine()`
- Modify: `backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/ScriptRegistry.java` — add `LifecycleEngine` field, replace `activate()`/`archive()`/`revise()` with `transition()`, replace `listActive()` with `listByState()`, update `updateMeta()` guard
- Modify: `backend/playbook-runtime/src/test/java/io/casehub/pages/playbook/runtime/ScriptRegistryTest.java` — update tests for new API
- Modify: `backend/playbook-runtime/src/test/java/io/casehub/pages/playbook/runtime/ScriptLifecycleTest.java` — update from `activate()`/`archive()`/`revise()`/`listActive()` to `transition()`/`listByState()`, `ScriptLifecycleState` constants to `String`, `IllegalTransitionException` to `IllegalStateException`
- Delete: `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptLifecycleState.java` (if not deleted in Task 3)

**Interfaces:**
- Consumes: `LifecycleEngine.defaultEngine()`, `LifecycleEngine.validateTransition()`, `LifecycleEngine.canTransition()` from Task 2; `ScriptDescriptor.state()` returns `String` from Task 3
- Produces: `ScriptRegistry.transition(String name, String targetState): ScriptDescriptor`, `ScriptRegistry.listByState(String state, List<String> labels, List<String> tags): List<ScriptDescriptor>`, `UploadedScriptSource.setState(String name, String state): ScriptDescriptor`, `UploadedScriptSource.stateOf(String name): String`

- [ ] **Step 1: Update ScriptRegistryTest for the new API**

Replace lifecycle-specific tests in `ScriptRegistryTest.java`. Add tests for:
- `transition(name, "active")` succeeds from draft
- `transition(name, "archived")` succeeds from active
- `transition(name, "draft")` succeeds from active (revise)
- `transition(name, "active")` fails from archived (no direct path)
- `listByState("active", ...)` returns only active scripts
- `updateMeta` allowed in initial state ("draft"), rejected in active

```java
@Test
void transition_draftToActive() {
    registry.upload(SAMPLE_YAML);
    var desc = registry.transition("sample-script", "active");
    assertThat(desc.state()).isEqualTo("active");
}

@Test
void transition_activeToArchived() {
    registry.upload(SAMPLE_YAML);
    registry.transition("sample-script", "active");
    var desc = registry.transition("sample-script", "archived");
    assertThat(desc.state()).isEqualTo("archived");
}

@Test
void transition_activeToDraft_revise() {
    registry.upload(SAMPLE_YAML);
    registry.transition("sample-script", "active");
    var desc = registry.transition("sample-script", "draft");
    assertThat(desc.state()).isEqualTo("draft");
}

@Test
void transition_archivedToActive_rejected() {
    registry.upload(SAMPLE_YAML);
    registry.transition("sample-script", "active");
    registry.transition("sample-script", "archived");
    assertThatThrownBy(() -> registry.transition("sample-script", "active"))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void transition_rejectsNonUploaded() {
    assertThatThrownBy(() -> registry.transition("helpdesk-intake", "active"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("not an uploaded script");
}

@Test
void listByState_filtersCorrectly() {
    registry.upload(SAMPLE_YAML);
    registry.transition("sample-script", "active");
    var active = registry.listByState("active", List.of(), List.of());
    assertThat(active).extracting("name").contains("sample-script");
    var drafts = registry.listByState("draft", List.of(), List.of());
    assertThat(drafts).extracting("name").doesNotContain("sample-script");
}

@Test
void updateMeta_allowedInDraft() {
    registry.upload(SAMPLE_YAML);
    var updated = registry.updateMeta("sample-script",
            new ScriptMeta("Updated", List.of(), List.of()));
    assertThat(updated.description()).isEqualTo("Updated");
}

@Test
void updateMeta_rejectedInActive() {
    registry.upload(SAMPLE_YAML);
    registry.transition("sample-script", "active");
    assertThatThrownBy(() -> registry.updateMeta("sample-script",
            new ScriptMeta("Updated", List.of(), List.of())))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("must be draft");
}
```

Also remove the old `updateMeta_updatesUploadedOnly` test (it didn't activate the script, so the script starts in draft/active default — verify whether the existing test is passing because of the default state).

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl backend/playbook-runtime -Dtest=ScriptRegistryTest -f backend/pom.xml`
Expected: FAIL — `transition()` and `listByState()` methods don't exist

- [ ] **Step 3: Modify UploadedScriptSource**

Replace the `OrcStateMachine` approach with simple String state tracking:

1. Remove the `stateMachines` field and `createStateMachine()` method
2. Remove all `OrcStateMachine` and `ScriptLifecycleState` imports
3. Add a `Map<String, String> scriptStates` field
4. Rename `transition()` to `setState()` — it now takes `(String name, String state)` and updates state without validation (validation is the engine's job)
5. Change `stateOf()` to return `String`
6. Update `upload()` to initialize state as the engine's initial state
7. Update `scanDirectory()` to initialize state

Key changes to `UploadedScriptSource`:

```java
// Remove:
private final Map<String, OrcStateMachine<ScriptLifecycleState>> stateMachines = new LinkedHashMap<>();

// Add:
private final Map<String, String> scriptStates = new LinkedHashMap<>();

// setState replaces transition:
public ScriptDescriptor setState(String name, String state) {
    if (!descriptors.containsKey(name)) {
        throw new IllegalArgumentException("Not an uploaded script: " + name);
    }
    scriptStates.put(name, state);
    ScriptDescriptor existing = descriptors.get(name);
    ScriptDescriptor updated = new ScriptDescriptor(existing.name(), existing.description(),
            existing.labels(), existing.tags(), existing.params(), existing.calls(),
            existing.provenance(), state, existing.firstStepTargets());
    descriptors.put(name, updated);
    return updated;
}

// stateOf returns String:
public String stateOf(String name) {
    return scriptStates.get(name);
}
```

Update `upload()`: replace `stateMachines.put(...)` with `scriptStates.put(desc.name(), "draft")`.
Update `delete()`: replace `stateMachines.remove(name)` with `scriptStates.remove(name)`.
Update `scanDirectory()`: replace `stateMachines.put(...)` with `scriptStates.put(desc.name(), desc.state())`.

- [ ] **Step 4: Modify ScriptRegistry**

1. Add `LifecycleEngine` field initialized via `LifecycleEngine.defaultEngine()`
2. Replace `activate()`, `archive()`, `revise()` with single `transition(String name, String targetState)`
3. Replace `listActive()` with `listByState(String state, List<String> labels, List<String> tags)`
4. Update `updateMeta()` guard to use `lifecycleEngine.initialState()`

```java
private final LifecycleEngine lifecycleEngine = LifecycleEngine.defaultEngine();

public ScriptDescriptor transition(String name, String targetState) {
    if (!uploaded.contains(name)) {
        throw new IllegalArgumentException(
            "Cannot transition '" + name + "': not an uploaded script");
    }
    String current = uploaded.stateOf(name);
    lifecycleEngine.validateTransition(current, targetState);
    return uploaded.setState(name, targetState);
}

public List<ScriptDescriptor> listByState(String state,
        List<String> labels, List<String> tags) {
    return allDescriptors()
            .filter(d -> state.equals(d.state()))
            .filter(d -> matchesLabels(d, labels))
            .filter(d -> matchesTags(d, tags))
            .toList();
}
```

Update `updateMeta()`:
```java
public ScriptDescriptor updateMeta(String name, ScriptMeta meta) {
    if (!uploaded.contains(name)) {
        throw new IllegalArgumentException(
                "Cannot update metadata for '" + name + "': not an uploaded script");
    }
    String state = uploaded.stateOf(name);
    if (state != null && !state.equals(lifecycleEngine.initialState())) {
        throw new IllegalStateException(
                "Cannot update metadata for '" + name + "': script is " + state
                + ", must be " + lifecycleEngine.initialState());
    }
    return uploaded.updateMeta(name, meta);
}
```

Remove `activate()`, `archive()`, `revise()`, `listActive()`.

- [ ] **Step 5: Update ScriptLifecycleTest.java**

This test file exercises the full lifecycle flow. Update every test to use the new API:

- Replace `registry.activate("test-script")` → `registry.transition("test-script", "active")`
- Replace `registry.archive("test-script")` → `registry.transition("test-script", "archived")`
- Replace `registry.revise("test-script")` → `registry.transition("test-script", "draft")`
- Replace `registry.listActive(...)` → `registry.listByState("active", ...)`
- Replace all `ScriptLifecycleState.DRAFT` → `"draft"`, `.ACTIVE` → `"active"`, `.ARCHIVED` → `"archived"`
- Replace `IllegalTransitionException.class` → `IllegalStateException.class` in `assertThatThrownBy`
- Remove `import io.casehub.pages.playbook.ScriptLifecycleState`
- Remove `import io.casehub.yaml.core.orchestration.IllegalTransitionException`

```java
@Test
void uploaded_startsAsDraft() {
    ScriptDescriptor desc = registry.upload(SAMPLE_YAML);
    assertThat(desc.state()).isEqualTo("draft");
}

@Test
void transition_movesToActive() {
    registry.upload(SAMPLE_YAML);
    ScriptDescriptor activated = registry.transition("test-script", "active");
    assertThat(activated.state()).isEqualTo("active");
}

@Test
void transition_movesToArchived() {
    registry.upload(SAMPLE_YAML);
    registry.transition("test-script", "active");
    ScriptDescriptor archived = registry.transition("test-script", "archived");
    assertThat(archived.state()).isEqualTo("archived");
}

@Test
void transition_revise_movesActiveBackToDraft() {
    registry.upload(SAMPLE_YAML);
    registry.transition("test-script", "active");
    ScriptDescriptor revised = registry.transition("test-script", "draft");
    assertThat(revised.state()).isEqualTo("draft");
}

@Test
void transition_fromArchived_toActive_throws() {
    registry.upload(SAMPLE_YAML);
    registry.transition("test-script", "active");
    registry.transition("test-script", "archived");
    assertThatThrownBy(() -> registry.transition("test-script", "active"))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void transition_archive_fromDraft_throws() {
    registry.upload(SAMPLE_YAML);
    assertThatThrownBy(() -> registry.transition("test-script", "archived"))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void transition_nonUploaded_throws() {
    assertThatThrownBy(() -> registry.transition("nonexistent", "active"))
            .isInstanceOf(IllegalArgumentException.class);
}

@Test
void listByState_excludesDraftAndArchived() {
    registry.upload(SAMPLE_YAML);
    assertThat(registry.listByState("active", List.of(), List.of())).isEmpty();

    registry.transition("test-script", "active");
    assertThat(registry.listByState("active", List.of(), List.of())).hasSize(1);

    registry.transition("test-script", "archived");
    assertThat(registry.listByState("active", List.of(), List.of())).isEmpty();
}

@Test
void updateMeta_onlyAllowedOnDraft() {
    registry.upload(SAMPLE_YAML);
    registry.transition("test-script", "active");
    assertThatThrownBy(() -> registry.updateMeta("test-script",
            new ScriptMeta("new desc", List.of(), List.of())))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void get_reflectsCurrentState() {
    registry.upload(SAMPLE_YAML);
    assertThat(registry.get("test-script").orElseThrow().state())
            .isEqualTo("draft");

    registry.transition("test-script", "active");
    assertThat(registry.get("test-script").orElseThrow().state())
            .isEqualTo("active");
}
```

- [ ] **Step 6: Delete ScriptLifecycleState.java**

If not deleted in Task 3, use `ide_refactor_safe_delete` now. All references should be gone.

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl backend/playbook-runtime -f backend/pom.xml`
Expected: all tests PASS (both ScriptRegistryTest and ScriptLifecycleTest)

Also run the full playbook module tests:
Run: `mvn test -pl backend/playbook -f backend/pom.xml`
Expected: PASS (ScriptDescriptorTest and ScriptDescriptorExtractorTest updated in Task 3)

- [ ] **Step 8: Commit**

```bash
git add backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptLifecycleState.java backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/UploadedScriptSource.java backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/ScriptRegistry.java backend/playbook-runtime/src/test/java/io/casehub/pages/playbook/runtime/ScriptRegistryTest.java backend/playbook-runtime/src/test/java/io/casehub/pages/playbook/runtime/ScriptLifecycleTest.java
git commit -m "refactor(playbook): wire LifecycleEngine into ScriptRegistry, remove enum (#498)"
```

---

## Batch 4: Cross-Runtime Consistency

### Task 5: Cross-runtime YAML consistency test

**Files:**
- Create: `backend/playbook/src/test/java/io/casehub/pages/playbook/LifecycleYamlConsistencyTest.java`

**Interfaces:**
- Consumes: both `default-lifecycle.yaml` files (TS at `packages/yaml-core/src/step/scenario/lifecycle/default-lifecycle.yaml`, Java at classpath `/lifecycle/default-lifecycle.yaml`)
- Produces: test that fails if the two YAML files diverge

- [ ] **Step 1: Write the consistency test**

Create `backend/playbook/src/test/java/io/casehub/pages/playbook/LifecycleYamlConsistencyTest.java`:

```java
package io.casehub.pages.playbook;

import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class LifecycleYamlConsistencyTest {

    @Test
    void defaultLifecycleYaml_isIdenticalAcrossRuntimes() throws IOException {
        String javaYaml;
        try (InputStream is = getClass().getResourceAsStream("/lifecycle/default-lifecycle.yaml")) {
            assertThat(is).as("Java lifecycle YAML must exist on classpath").isNotNull();
            javaYaml = new String(is.readAllBytes());
        }

        Path tsYaml = Path.of("packages/yaml-core/src/step/scenario/lifecycle/default-lifecycle.yaml");
        assertThat(tsYaml).as("TS lifecycle YAML must exist").exists();
        String tsContent = Files.readString(tsYaml);

        assertThat(javaYaml).as("Java and TS lifecycle YAML must be identical").isEqualTo(tsContent);
    }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn test -pl backend/playbook -Dtest=LifecycleYamlConsistencyTest -f backend/pom.xml`
Expected: PASS (both files were created with identical content)

- [ ] **Step 3: Commit**

```bash
git add backend/playbook/src/test/java/io/casehub/pages/playbook/LifecycleYamlConsistencyTest.java
git commit -m "test(playbook): cross-runtime lifecycle YAML consistency check (#498)"
```

---

## References

- `specs/issue-418-theme-resource/2026-10-26-scenario-lifecycle-engine-design.md` — design spec
- `specs/issue-418-theme-resource/decisions.md` — D1-D4 design decisions
- `packages/yaml-core/src/step/scenario/types.ts:45` — `ScenarioDefinition` interface
- `packages/yaml-core/src/step/scenario/parser.ts:7` — `ScenarioParser` class
- `packages/yaml-core/src/step/scenario/index.ts` — scenario barrel exports
- `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptLifecycleState.java:3` — enum to delete
- `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptDescriptor.java:6` — record to modify
- `backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/ScriptRegistry.java:11` — registry to rewire
- `backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/UploadedScriptSource.java:19` — source to simplify
- `backend/playbook/src/main/java/io/casehub/pages/playbook/ScriptDescriptorExtractor.java:45` — state assignment to update
- `backend/playbook-runtime/src/test/java/io/casehub/pages/playbook/runtime/ScriptLifecycleTest.java` — lifecycle test to migrate
- `packages/pages-aria/src/controller/library-view.ts:6` — TS `ScriptDescriptor` (already `state: string`, no change)
- GitHub #498 — focal issue
