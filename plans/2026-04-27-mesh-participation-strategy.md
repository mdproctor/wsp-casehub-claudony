# MeshParticipationStrategy SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Define the `MeshParticipationStrategy` SPI with three built-in implementations, wire it into `ClaudonyWorkerContextProvider` so every `WorkerContext` is stamped with the worker's mesh participation level, and bring all documentation into sync with the codebase (including drift from issue #87 which shipped without a doc pass).

**Architecture:** Three plain-Java strategy implementations selected by `claudony.casehub.mesh-participation` config (switch pattern, consistent with `CaseChannelLayout` from #87). `ClaudonyWorkerContextProvider` gains a CDI constructor `(CaseLineageQuery, CaseChannelProvider, CaseHubConfig)` that calls `selectStrategy()`, a package-private 3-arg test constructor, and keeps the existing 2-arg constructor (defaults to Active) for backward compatibility. `strategyFor(workerId, null)` is called at the top of `buildContext()` before any early returns — `"meshParticipation" → participation.name()` appears in every `WorkerContext.properties`, regardless of exit path.

**Tech Stack:** Java 21, Quarkus 3.9.5, SmallRye Config `@ConfigMapping`, JBoss Logging, JUnit 5, AssertJ, Mockito, `@QuarkusTest` + `QuarkusTestProfile`

**GitHub issue:** #88 (part of epic #86)

---

## File Map

| Status | File | Role |
|--------|------|------|
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/MeshParticipationStrategy.java` | SPI interface + `MeshParticipation` enum |
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/ActiveParticipationStrategy.java` | Always returns ACTIVE |
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/ReactiveParticipationStrategy.java` | Always returns REACTIVE |
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/SilentParticipationStrategy.java` | Always returns SILENT |
| Create | `claudony-casehub/src/test/java/dev/claudony/casehub/MeshParticipationStrategyTest.java` | 12 unit tests: happy path, robustness, correctness |
| Create | `claudony-app/src/test/java/dev/claudony/MeshParticipationIntegrationTest.java` | 2 Quarkus integration tests: default ACTIVE, profile SILENT |
| Modify | `claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java` | Add `meshParticipation()` |
| Modify | `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java` | 3 constructors, selectStrategy(), stamp in all buildContext paths |
| Modify | `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java` | 9 new tests: all 3 strategies, all 4 exit paths, bad config |
| Modify | `claudony-casehub/src/test/java/dev/claudony/casehub/WorkerLifecycleSequenceTest.java` | Add meshParticipation assertion to lifecycle sequence |
| Modify | `CLAUDE.md` | Update test count; add new SPI classes to casehub component list |
| Modify | `docs/DESIGN.md` | Add CaseChannelLayout + MeshParticipationStrategy to component tree; update SPI descriptions; add Agent Mesh section |

---

## Task 1: Define `MeshParticipationStrategy` SPI

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/MeshParticipationStrategy.java`

No test needed — pure interface + enum.

- [ ] **Step 1: Create the file**

```java
package dev.claudony.casehub;

import io.casehub.api.model.WorkerContext;

public interface MeshParticipationStrategy {

    /**
     * Returns the participation level for the given worker.
     *
     * @param workerId the worker identifier
     * @param context  the worker context; may be {@code null} if not yet built
     */
    MeshParticipation strategyFor(String workerId, WorkerContext context);

    enum MeshParticipation {
        /** Register on startup, post STATUS, check messages periodically. */
        ACTIVE,
        /** Do not register; only engage when directly addressed. */
        REACTIVE,
        /** No mesh participation. */
        SILENT
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl claudony-casehub -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/MeshParticipationStrategy.java
git commit -m "feat: add MeshParticipationStrategy SPI interface and MeshParticipation enum Refs #88"
```

---

## Task 2: Strategy Implementations + Comprehensive Unit Tests

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/ActiveParticipationStrategy.java`
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/ReactiveParticipationStrategy.java`
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/SilentParticipationStrategy.java`
- Create: `claudony-casehub/src/test/java/dev/claudony/casehub/MeshParticipationStrategyTest.java`

- [ ] **Step 1: Write the failing tests**

12 tests covering happy path, robustness, and correctness:

```java
package dev.claudony.casehub;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class MeshParticipationStrategyTest {

    // ── Happy path ─────────────────────────────────────────────────────────

    @Test
    void active_returnsActive() {
        assertThat(new ActiveParticipationStrategy().strategyFor("worker-1", null))
                .isEqualTo(MeshParticipationStrategy.MeshParticipation.ACTIVE);
    }

    @Test
    void reactive_returnsReactive() {
        assertThat(new ReactiveParticipationStrategy().strategyFor("worker-1", null))
                .isEqualTo(MeshParticipationStrategy.MeshParticipation.REACTIVE);
    }

    @Test
    void silent_returnsSilent() {
        assertThat(new SilentParticipationStrategy().strategyFor("worker-1", null))
                .isEqualTo(MeshParticipationStrategy.MeshParticipation.SILENT);
    }

    @Test
    void threeDistinctValues() {
        assertThat(new ActiveParticipationStrategy().strategyFor("w", null))
                .isNotEqualTo(new ReactiveParticipationStrategy().strategyFor("w", null));
        assertThat(new ReactiveParticipationStrategy().strategyFor("w", null))
                .isNotEqualTo(new SilentParticipationStrategy().strategyFor("w", null));
    }

    // ── Robustness ────────────────────────────────────────────────────────

    @Test
    void allStrategiesAcceptNullWorkerId() {
        assertThatNoException().isThrownBy(() -> {
            new ActiveParticipationStrategy().strategyFor(null, null);
            new ReactiveParticipationStrategy().strategyFor(null, null);
            new SilentParticipationStrategy().strategyFor(null, null);
        });
    }

    @Test
    void allStrategiesAcceptEmptyWorkerId() {
        assertThatNoException().isThrownBy(() -> {
            new ActiveParticipationStrategy().strategyFor("", null);
            new ReactiveParticipationStrategy().strategyFor("", null);
            new SilentParticipationStrategy().strategyFor("", null);
        });
    }

    @Test
    void allStrategiesAcceptNullContext() {
        assertThatNoException().isThrownBy(() -> {
            new ActiveParticipationStrategy().strategyFor("w", null);
            new ReactiveParticipationStrategy().strategyFor("w", null);
            new SilentParticipationStrategy().strategyFor("w", null);
        });
    }

    // ── Correctness ───────────────────────────────────────────────────────

    @Test
    void allStrategiesIgnoreWorkerId() {
        var active = new ActiveParticipationStrategy();
        assertThat(active.strategyFor("alice", null)).isEqualTo(active.strategyFor("bob", null));

        var silent = new SilentParticipationStrategy();
        assertThat(silent.strategyFor("alice", null)).isEqualTo(silent.strategyFor("bob", null));
    }

    @Test
    void resultsAreConsistentAcrossRepeatedCalls() {
        var strategy = new ActiveParticipationStrategy();
        MeshParticipationStrategy.MeshParticipation first = strategy.strategyFor("w", null);
        MeshParticipationStrategy.MeshParticipation second = strategy.strategyFor("w", null);
        assertThat(first).isEqualTo(second);
    }

    @Test
    void participationEnumHasExactlyThreeValues() {
        assertThat(MeshParticipationStrategy.MeshParticipation.values()).hasSize(3);
    }

    @Test
    void allThreeEnumValuesAreDistinct() {
        var values = MeshParticipationStrategy.MeshParticipation.values();
        assertThat(values).doesNotHaveDuplicates();
    }

    @Test
    void enumNameMatchesExpectedStrings() {
        assertThat(MeshParticipationStrategy.MeshParticipation.ACTIVE.name()).isEqualTo("ACTIVE");
        assertThat(MeshParticipationStrategy.MeshParticipation.REACTIVE.name()).isEqualTo("REACTIVE");
        assertThat(MeshParticipationStrategy.MeshParticipation.SILENT.name()).isEqualTo("SILENT");
    }
}
```

- [ ] **Step 2: Run tests — expect compile error**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=MeshParticipationStrategyTest -q 2>&1 | tail -5
```

Expected: COMPILE ERROR (classes don't exist yet)

- [ ] **Step 3: Create `ActiveParticipationStrategy.java`**

```java
package dev.claudony.casehub;

import io.casehub.api.model.WorkerContext;

public class ActiveParticipationStrategy implements MeshParticipationStrategy {

    @Override
    public MeshParticipation strategyFor(String workerId, WorkerContext context) {
        return MeshParticipation.ACTIVE;
    }
}
```

- [ ] **Step 4: Create `ReactiveParticipationStrategy.java`**

```java
package dev.claudony.casehub;

import io.casehub.api.model.WorkerContext;

public class ReactiveParticipationStrategy implements MeshParticipationStrategy {

    @Override
    public MeshParticipation strategyFor(String workerId, WorkerContext context) {
        return MeshParticipation.REACTIVE;
    }
}
```

- [ ] **Step 5: Create `SilentParticipationStrategy.java`**

```java
package dev.claudony.casehub;

import io.casehub.api.model.WorkerContext;

public class SilentParticipationStrategy implements MeshParticipationStrategy {

    @Override
    public MeshParticipation strategyFor(String workerId, WorkerContext context) {
        return MeshParticipation.SILENT;
    }
}
```

- [ ] **Step 6: Run tests — expect all 12 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=MeshParticipationStrategyTest
```

Expected: Tests run: 12, Failures: 0, Errors: 0

- [ ] **Step 7: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/ActiveParticipationStrategy.java \
        claudony-casehub/src/main/java/dev/claudony/casehub/ReactiveParticipationStrategy.java \
        claudony-casehub/src/main/java/dev/claudony/casehub/SilentParticipationStrategy.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/MeshParticipationStrategyTest.java
git commit -m "feat: implement Active/Reactive/Silent participation strategies Refs #88"
```

---

## Task 3: Config + Provider Wiring + Comprehensive Unit Tests

**Files:**
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java`
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java`
- Modify: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java`
- Modify: `claudony-casehub/src/test/java/dev/claudony/casehub/WorkerLifecycleSequenceTest.java`

### Step 3a: Add config property

- [ ] **Step 1: Update `CaseHubConfig`**

```java
package dev.claudony.casehub;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;
import java.util.Map;

@ConfigMapping(prefix = "claudony.casehub")
public interface CaseHubConfig {

    @WithDefault("false")
    boolean enabled();

    @WithName("channel-layout")
    @WithDefault("normative")
    String channelLayout();

    @WithName("mesh-participation")
    @WithDefault("active")
    String meshParticipation();

    Workers workers();

    interface Workers {
        /** Map of capability name to launch command. Key "default" is the fallback. */
        Map<String, String> commands();

        @WithName("default-working-dir")
        @WithDefault("${user.home}/claudony-workspace")
        String defaultWorkingDir();
    }
}
```

### Step 3b: Write new tests first (TDD)

- [ ] **Step 2: Add 9 new tests to `ClaudonyWorkerContextProviderTest`**

The existing 6 tests use the 2-arg constructor (which will default to Active). The new tests use:
- The 3-arg package-private constructor `(CaseLineageQuery, CaseChannelProvider, MeshParticipationStrategy)` for strategy-specific tests
- The CDI constructor `(CaseLineageQuery, CaseChannelProvider, CaseHubConfig)` with a mock config for the bad-config test

Append these to the existing test class (after the last existing `@Test`). Imports to add:
`import io.casehub.api.model.WorkRequest;` (already present), `import static org.mockito.Mockito.when;` (already present).

```java
    // ── Happy path: all 3 strategies stamp correctly ──────────────────────

    @Test
    void buildContext_activeStrategy_stampsMeshParticipationActive() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("task", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).containsEntry("meshParticipation", "ACTIVE");
    }

    @Test
    void buildContext_reactiveStrategy_stampsMeshParticipationReactive() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ReactiveParticipationStrategy());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("task", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).containsEntry("meshParticipation", "REACTIVE");
    }

    @Test
    void buildContext_silentStrategy_stampsMeshParticipationSilent() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new SilentParticipationStrategy());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("task", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).containsEntry("meshParticipation", "SILENT");
    }

    // ── Correctness: stamp present on every exit path ─────────────────────

    @Test
    void buildContext_cleanStart_meshParticipationStamped() {
        var silentProvider = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new SilentParticipationStrategy());

        WorkerContext ctx = silentProvider.buildContext("w1",
                WorkRequest.of("task", Map.of("clean-start", true)));

        assertThat(ctx.properties()).containsEntry("meshParticipation", "SILENT");
        verifyNoInteractions(lineageQuery);
        verifyNoInteractions(channelProvider);
    }

    @Test
    void buildContext_missingCaseId_meshParticipationStamped() {
        var silentProvider = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new SilentParticipationStrategy());

        WorkerContext ctx = silentProvider.buildContext("w1",
                WorkRequest.of("task", Map.of()));

        assertThat(ctx.properties()).containsEntry("meshParticipation", "SILENT");
    }

    @Test
    void buildContext_malformedCaseId_meshParticipationStamped() {
        var silentProvider = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new SilentParticipationStrategy());

        WorkerContext ctx = silentProvider.buildContext("w1",
                WorkRequest.of("task", Map.of("caseId", "not-a-uuid")));

        assertThat(ctx.properties()).containsEntry("meshParticipation", "SILENT");
    }

    @Test
    void buildContext_meshParticipationValueIsEnumNameNotToString() {
        // Verifies .name() is used (not .toString(), which would be identical for enums
        // but this documents the contract explicitly)
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("task", Map.of()));

        assertThat(ctx.properties().get("meshParticipation"))
                .isEqualTo(MeshParticipationStrategy.MeshParticipation.ACTIVE.name());
    }

    // ── Robustness: default constructor uses Active ────────────────────────

    @Test
    void defaultConstructor_usesActiveStrategy() {
        // The 2-arg constructor (used in most tests) must default to Active
        WorkerContext ctx = provider.buildContext("w1",
                WorkRequest.of("task", Map.of()));

        assertThat(ctx.properties()).containsEntry("meshParticipation", "ACTIVE");
    }

    // ── Correctness: bad config throws at construction time ────────────────

    @Test
    void cdiConstructor_badMeshParticipationConfig_throwsIllegalArgumentException() {
        CaseHubConfig config = mock(CaseHubConfig.class);
        when(config.meshParticipation()).thenReturn("bogus");
        when(config.channelLayout()).thenReturn("normative");

        assertThatThrownBy(() -> new ClaudonyWorkerContextProvider(lineageQuery, channelProvider, config))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("bogus");
    }
```

Also add `import static org.mockito.Mockito.mock;` if not already present (it is, since `lineageQuery = mock(...)`), and add `import dev.claudony.casehub.CaseHubConfig;` (same package, not needed).

- [ ] **Step 3: Run new tests — expect compile errors (3-arg constructor doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerContextProviderTest -q 2>&1 | tail -5
```

Expected: COMPILE ERROR

### Step 3c: Refactor the provider

- [ ] **Step 4: Rewrite `ClaudonyWorkerContextProvider.java`**

```java
package dev.claudony.casehub;

import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.CaseChannel;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerSummary;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.api.spi.WorkerContextProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ClaudonyWorkerContextProvider implements WorkerContextProvider {

    private static final Logger log = Logger.getLogger(ClaudonyWorkerContextProvider.class);

    private final CaseLineageQuery lineageQuery;
    private final CaseChannelProvider channelProvider;
    private final MeshParticipationStrategy strategy;

    @Inject
    public ClaudonyWorkerContextProvider(CaseLineageQuery lineageQuery,
                                          CaseChannelProvider channelProvider,
                                          CaseHubConfig config) {
        this(lineageQuery, channelProvider, selectStrategy(config.meshParticipation()));
    }

    ClaudonyWorkerContextProvider(CaseLineageQuery lineageQuery,
                                   CaseChannelProvider channelProvider,
                                   MeshParticipationStrategy strategy) {
        this.lineageQuery = lineageQuery;
        this.channelProvider = channelProvider;
        this.strategy = strategy;
    }

    ClaudonyWorkerContextProvider(CaseLineageQuery lineageQuery,
                                   CaseChannelProvider channelProvider) {
        this(lineageQuery, channelProvider, new ActiveParticipationStrategy());
    }

    @Override
    public WorkerContext buildContext(String workerId, WorkRequest task) {
        MeshParticipationStrategy.MeshParticipation participation =
                strategy.strategyFor(workerId, null);

        var props = new HashMap<String, Object>();
        props.put("meshParticipation", participation.name());

        if (Boolean.TRUE.equals(task.input().get("clean-start"))) {
            props.put("clean-start", true);
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), props);
        }

        String caseIdStr = (String) task.input().get("caseId");
        if (caseIdStr == null || caseIdStr.isBlank()) {
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), props);
        }

        UUID caseId;
        try {
            caseId = UUID.fromString(caseIdStr);
        } catch (IllegalArgumentException e) {
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), props);
        }

        List<WorkerSummary> priorWorkers = lineageQuery.findCompletedWorkers(caseId);

        CaseChannel channel = channelProvider.listChannels(caseId).stream()
                .findFirst()
                .orElse(null);

        return new WorkerContext(task.capability(), caseId, channel, priorWorkers,
                PropagationContext.createRoot(), props);
    }

    private static MeshParticipationStrategy selectStrategy(String name) {
        return switch (name) {
            case "active" -> new ActiveParticipationStrategy();
            case "reactive" -> new ReactiveParticipationStrategy();
            case "silent" -> new SilentParticipationStrategy();
            default -> {
                log.errorf("Unknown mesh-participation '%s' — valid values: active, reactive, silent", name);
                throw new IllegalArgumentException("Unknown mesh participation: " + name);
            }
        };
    }
}
```

- [ ] **Step 5: Run all casehub tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub
```

Expected: All tests pass. Verify the new count is 69 + 12 (strategy) + 9 (provider) + existing increase from lifecycle (Task 3d) = ~90+.

If `buildContext_cleanStart_returnsEmptyPriorWorkers` fails: the test calls `verifyNoInteractions(lineageQuery)` and `verifyNoInteractions(channelProvider)` — both still hold because `strategy.strategyFor()` is a pure call on a plain Java object, not on either mock.

### Step 3d: Add mesh participation assertion to lifecycle sequence test

- [ ] **Step 6: Read `WorkerLifecycleSequenceTest.java` and add one assertion**

Read the file first:
```bash
cat claudony-casehub/src/test/java/dev/claudony/casehub/WorkerLifecycleSequenceTest.java
```

Find the test that calls `buildContext()` (it will be calling `WorkerContextProvider` or `ClaudonyWorkerContextProvider`). Add an assertion that `ctx.properties().containsKey("meshParticipation")` is true — verifying the stamp flows through the full lifecycle sequence.

If `WorkerLifecycleSequenceTest` does not call `buildContext()` at all, add a new test:

```java
@Test
void workerContext_alwaysHasMeshParticipationStamped() {
    // Verifies the full chain: provisioner → context provider → context has participation
    // Uses the default Active strategy via the 2-arg constructor
    var contextProvider = new ClaudonyWorkerContextProvider(
            mock(CaseLineageQuery.class), mock(CaseChannelProvider.class));
    when(contextProvider /* lineageQuery mock */) // adapt based on actual test structure

    WorkerContext ctx = contextProvider.buildContext("w1",
            WorkRequest.of("researcher", Map.of()));

    assertThat(ctx.properties()).containsKey("meshParticipation");
}
```

Note: adapt the test to the actual structure of `WorkerLifecycleSequenceTest` — the important assertion is `containsKey("meshParticipation")`.

- [ ] **Step 7: Run all casehub tests again**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub
```

Expected: All tests pass, 0 failures.

- [ ] **Step 8: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java \
        claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/WorkerLifecycleSequenceTest.java
git commit -m "feat: wire MeshParticipationStrategy into ClaudonyWorkerContextProvider — stamps meshParticipation on every context Refs #88"
```

---

## Task 4: Quarkus Integration Tests

**Files:**
- Create: `claudony-app/src/test/java/dev/claudony/MeshParticipationIntegrationTest.java`

These tests verify the full CDI wiring: `CaseHubConfig` → `ClaudonyWorkerContextProvider` constructor → `selectStrategy()` → `buildContext()` → `WorkerContext.properties`. They run the real Quarkus container.

- [ ] **Step 1: Write the integration test file**

The first inner class tests the default config (no profile, no restart). The second uses a `@TestProfile` for "silent" config (one Quarkus restart).

```java
package dev.claudony;

import dev.claudony.casehub.ClaudonyWorkerContextProvider;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class MeshParticipationIntegrationTest {

    @Inject
    ClaudonyWorkerContextProvider provider;

    @Test
    void defaultConfig_stampsActiveParticipation() {
        // Default: claudony.casehub.mesh-participation=active (from @WithDefault)
        WorkerContext ctx = provider.buildContext("integration-worker",
                WorkRequest.of("task", Map.of()));

        assertThat(ctx.properties())
                .containsEntry("meshParticipation", "ACTIVE");
    }

    @Test
    void defaultConfig_meshParticipationKeyAlwaysPresent() {
        // Even with no caseId, the key must be in every context
        WorkerContext ctx = provider.buildContext("integration-worker",
                WorkRequest.of("researcher", Map.of("caseId", "not-a-uuid")));

        assertThat(ctx.properties()).containsKey("meshParticipation");
    }
}
```

Also create a second test class for the silent profile (must be a separate class because `@TestProfile` applies to the whole class and triggers a Quarkus restart):

```java
package dev.claudony;

import dev.claudony.casehub.ClaudonyWorkerContextProvider;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestProfile(MeshParticipationSilentProfileTest.SilentProfile.class)
class MeshParticipationSilentProfileTest {

    public static class SilentProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("claudony.casehub.mesh-participation", "silent");
        }
    }

    @Inject
    ClaudonyWorkerContextProvider provider;

    @Test
    void silentConfig_stampsParticipationSilent() {
        WorkerContext ctx = provider.buildContext("integration-worker",
                WorkRequest.of("task", Map.of()));

        assertThat(ctx.properties())
                .containsEntry("meshParticipation", "SILENT");
    }
}
```

Both classes go in `claudony-app/src/test/java/dev/claudony/`.

- [ ] **Step 2: Run the integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app \
  -Dtest=MeshParticipationIntegrationTest,MeshParticipationSilentProfileTest
```

Expected: 3 tests, 0 failures. The profile test restarts Quarkus once — this is normal and expected.

If the injection fails with "Unsatisfied dependency" for `ClaudonyWorkerContextProvider`, check that `claudony-casehub` is on the classpath (it's a `claudony-app` dependency) and that `CaseHubConfig` is present in the test config. The default `application.properties` in `src/test/resources` already configures Qhorus and lineage defaults.

- [ ] **Step 3: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass, 0 failures.

- [ ] **Step 4: Commit**

```bash
git add claudony-app/src/test/java/dev/claudony/MeshParticipationIntegrationTest.java \
        claudony-app/src/test/java/dev/claudony/MeshParticipationSilentProfileTest.java
git commit -m "test: add Quarkus integration tests for MeshParticipationStrategy config wiring Refs #88"
```

---

## Task 5: Documentation — Fix Drift from #87 and #88

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/DESIGN.md`

### Step 5a: Update `CLAUDE.md`

- [ ] **Step 1: Run the test suite and capture the count**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep "Tests run:" | tail -1
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | \
  awk '/\[INFO\] Tests run:/{split($0,a,"Tests run: "); split(a[2],b,","); sum+=b[1]} END{print sum " total"}'
```

Use the actual numbers. After #87 the count was 69 casehub + 283 app = 352. After #88 tasks 1-4 it will be higher.

- [ ] **Step 2: Update the test count line in `CLAUDE.md`**

Find:
```
**334 tests passing** (as of 2026-04-27, all modules): 51 in `claudony-casehub` + 283 in `claudony-app`. Zero failures, zero errors.
```

Replace with the actual new counts from step 1. Format:
```
**N tests passing** (as of 2026-04-27, all modules): X in `claudony-casehub` + Y in `claudony-app`. Zero failures, zero errors.
```

- [ ] **Step 3: Update the `claudony-casehub` component list in `CLAUDE.md`**

Find the project structure section. The `claudony-casehub` tree currently shows:
```
claudony-casehub/src/main/java/dev/claudony/casehub/
├── config/ClaudonyConfig.java          — all config properties
└── server/
    ├── model/                          — Session, SessionStatus, SessionExpiredEvent
    ├── TmuxService.java                — ProcessBuilder wrappers for tmux commands
    ├── SessionRegistry.java            — in-memory ConcurrentHashMap session store
    └── expiry/                         — ExpiryPolicy SPI + implementations + scheduler
```

That's the core module tree. Find the `claudony-casehub` section (lines starting with `CaseHubConfig`, `WorkerCommandResolver`, etc.) and add the new classes introduced in #87 and #88. Current list ends at:
```
└── ClaudonyWorkerStatusListener.java   — WorkerStatusListener SPI: lifecycle → SessionRegistry
```

Add after that line:
```
├── WorkerSessionMapping.java           — role↔session bridge: caseId:role→sessionId + role→sessionId fallback
├── JpaCaseLineageQuery.java            — @Alternative @Priority(1): queries case_ledger_entry via qhorus PU
├── CaseChannelLayout.java              — SPI: controls which channels open per case; ChannelSpec record
├── NormativeChannelLayout.java         — default: work/observe/oversight channels (APPEND semantic)
├── SimpleLayout.java                   — 2-channel variant: work/observe only (no oversight)
├── MeshParticipationStrategy.java      — SPI: controls how a worker engages with Qhorus mesh; MeshParticipation enum
├── ActiveParticipationStrategy.java    — default: register + STATUS + check_messages periodically
├── ReactiveParticipationStrategy.java  — engage only when directly addressed
└── SilentParticipationStrategy.java    — no mesh participation
```

- [ ] **Step 4: Update `claudony.casehub.*` config properties in `CLAUDE.md`**

Find the configuration block:
```properties
claudony.casehub.enabled=true
claudony.casehub.workers.commands.default=claude
```

Add the two new properties below `enabled`:
```properties
claudony.casehub.channel-layout=normative      # normative | simple
claudony.casehub.mesh-participation=active     # active | reactive | silent
```

### Step 5b: Update `docs/DESIGN.md`

- [ ] **Step 5: Update the `claudony-casehub` component tree in `DESIGN.md`**

Find the section starting:
```
claudony-casehub — dev.claudony.casehub
├── CaseHubConfig                   — @ConfigMapping for claudony.casehub.*
```

The current tree ends at `ClaudonyWorkerStatusListener`. Add the new classes:

```
├── WorkerSessionMapping            — role↔session bridge (caseId:role→sessionId; role→sessionId fallback)
├── JpaCaseLineageQuery             — @Alternative @Priority(1) — queries case_ledger_entry via qhorus PU
├── CaseChannelLayout               — SPI: List<ChannelSpec> channelsFor(caseId, definition)
│                                     ChannelSpec: purpose, ChannelSemantic, allowedTypes, description
├── NormativeChannelLayout          — default layout: work (APPEND, all types) / observe (APPEND, EVENT) /
│                                     oversight (APPEND, QUERY+COMMAND)
├── SimpleLayout                    — 2-channel layout: work + observe only (no oversight)
├── MeshParticipationStrategy       — SPI: MeshParticipation strategyFor(workerId, context)
│                                     MeshParticipation enum: ACTIVE | REACTIVE | SILENT
├── ActiveParticipationStrategy     — fleet default: register + STATUS + periodic check_messages
├── ReactiveParticipationStrategy   — engage only when directly addressed; skip registration
└── SilentParticipationStrategy     — no mesh participation
```

- [ ] **Step 6: Update the CaseHub SPIs table in `DESIGN.md`**

Find the table under "CaseHub SPIs — Shipped":
```
| `CaseChannelProvider` | `ClaudonyCaseChannelProvider` | Qhorus client — opens channels named `case-{caseId}/{purpose}` |
| `WorkerContextProvider` | `ClaudonyWorkerContextProvider` | Builds Claude startup prompt with task description, lineage, and channel name |
```

Replace with:
```
| `CaseChannelProvider` | `ClaudonyCaseChannelProvider` | Opens all channels defined by `CaseChannelLayout` on first touch (init-on-first-touch cache). Default: `NormativeChannelLayout` — `work`/`observe`/`oversight` channels, all APPEND. Config: `claudony.casehub.channel-layout=normative\|simple` |
| `WorkerContextProvider` | `ClaudonyWorkerContextProvider` | Builds worker context: task description, lineage, channel, and `meshParticipation` property stamped from `MeshParticipationStrategy`. Config: `claudony.casehub.mesh-participation=active\|reactive\|silent` |
```

- [ ] **Step 7: Add an Agent Mesh section to `DESIGN.md`**

Find the `### Guard Rails` section near the end of the Ecosystem Integration section. Insert a new section before it:

```markdown
### Agent Mesh — Shipped (partial, epic #86)

Two SPIs control how Claudony-managed Claude agents engage with the Qhorus mesh:

**`CaseChannelLayout`** — defines *what channels* open when a case starts. Built-in:
- `NormativeChannelLayout` (default): `case-{id}/work` (all types), `case-{id}/observe` (EVENT only), `case-{id}/oversight` (QUERY+COMMAND)
- `SimpleLayout`: `case-{id}/work` + `case-{id}/observe` only (no human oversight channel)

**`MeshParticipationStrategy`** — defines *how* a worker engages at startup. Built-in:
- `ActiveParticipationStrategy` (default): register → announce → check messages periodically
- `ReactiveParticipationStrategy`: no registration; engage only when directly addressed
- `SilentParticipationStrategy`: no mesh participation

Both are selected via config and stamped into every `WorkerContext.properties`:
```properties
claudony.casehub.channel-layout=normative      # normative | simple
claudony.casehub.mesh-participation=active     # active | reactive | silent
```

`WorkerContext.properties["meshParticipation"]` is always present — "ACTIVE", "REACTIVE", or "SILENT". The system prompt template (epic #86, issue #89) reads this to decide whether to include mesh startup instructions.

**Outstanding:** System prompt template (#89), `MeshParticipationStrategy.strategyFor()` currently receives `null` for `context` since the context isn't yet built at strategy-call time.
```

- [ ] **Step 8: Verify `DESIGN.md` has no remaining stale references**

Check for any references to the old single-channel `openChannel` behaviour (now replaced by init-on-first-touch) or missing references to new classes. Grep for potential gaps:

```bash
grep -n "openChannel\|single channel\|WorkerSessionMapping\|CaseChannelLayout\|MeshParticipation" docs/DESIGN.md
```

If `WorkerSessionMapping` isn't mentioned anywhere, add a one-liner in the CaseHub SPIs section under the Lineage note:

```
**Worker↔Session mapping:** `WorkerSessionMapping` bridges CaseHub role names to Claudony tmux session UUIDs — two maps: `caseId:role→sessionId` (precise) and `role→sessionId` (fallback). Limitation: concurrent same-role workers across cases (tracked upstream, #93).
```

- [ ] **Step 9: Commit documentation updates**

```bash
git add CLAUDE.md docs/DESIGN.md
git commit -m "docs: sync DESIGN.md and CLAUDE.md with #87 and #88 — CaseChannelLayout, MeshParticipationStrategy, updated test count Refs #88"
```

---

## Self-Review

**Spec coverage:**

| Requirement | Task |
|---|---|
| `MeshParticipationStrategy` interface + `MeshParticipation` enum | Task 1 |
| `strategyFor(workerId, context)` with null-context documented | Task 1 |
| `ActiveParticipationStrategy` returns ACTIVE | Task 2 |
| `ReactiveParticipationStrategy` returns REACTIVE | Task 2 |
| `SilentParticipationStrategy` returns SILENT | Task 2 |
| 12 unit tests: happy path, robustness, correctness | Task 2 |
| Config `claudony.casehub.mesh-participation` with default "active" | Task 3a |
| `buildContext()` stamps meshParticipation on ALL exit paths (clean-start, no caseId, malformed caseId, full path) | Task 3c |
| 9 new unit tests for provider (all strategies, all paths, bad config) | Task 3b |
| Lifecycle sequence test asserts meshParticipation present | Task 3d |
| Integration test: default config → ACTIVE | Task 4 |
| Integration test: `mesh-participation=silent` profile → SILENT | Task 4 |
| `CLAUDE.md` test count updated to actual post-#88 count | Task 5a |
| `CLAUDE.md` casehub component list includes all new classes | Task 5a |
| `CLAUDE.md` config block includes channel-layout and mesh-participation | Task 5a |
| `DESIGN.md` casehub component tree updated | Task 5b |
| `DESIGN.md` CaseHub SPIs table updated for both providers | Task 5b |
| `DESIGN.md` Agent Mesh section added | Task 5b |
| `DESIGN.md` WorkerSessionMapping mentioned | Task 5b |

**Gaps noted and addressed:**
- Original plan missing REACTIVE strategy in provider tests → added (test 2 of 9)
- Original plan missing clean-start stamp test → added (test 4 of 9)
- Original plan missing malformed-caseId stamp test → added (test 5 of 9)
- Original plan missing bad-config test → added (test 9 of 9)
- Original plan missing integration tests → Task 4 added
- Original plan missing documentation → Task 5 added
- `docs/DESIGN.md` had no mention of `CaseChannelLayout`, `MeshParticipationStrategy`, `WorkerSessionMapping`, `JpaCaseLineageQuery` in component tree → Task 5b fixes
- `CLAUDE.md` test count stale at 334 (actual 352+ post-#87) → Task 5a fixes

**Placeholder scan:** No TBD or "implement later". All code blocks are complete. Task 3d has conditional logic based on actual file content — the subagent must read `WorkerLifecycleSequenceTest.java` before editing.

**Type consistency:**
- `MeshParticipationStrategy.MeshParticipation` enum defined in Task 1, used identically throughout Tasks 2, 3, 4
- `strategyFor(String, WorkerContext)` signature matches in all 3 implementations (Task 2) and call site (Task 3c: `strategy.strategyFor(workerId, null)`)
- 3-arg package-private constructor defined in Task 3c matches usage in Task 3b tests
- CDI constructor `(CaseLineageQuery, CaseChannelProvider, CaseHubConfig)` defined in Task 3c matches integration test injection in Task 4
- `props.put("meshParticipation", participation.name())` — key "meshParticipation" matches all test assertions
