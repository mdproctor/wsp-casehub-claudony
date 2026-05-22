# CaseHub.startCase() Round-Trip Test Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the CDI-event stub in `CaseEngineRoundTripTest` with a test that exercises the full engine chain: `CaseHub.startCase()` → `ClaudonyWorkerProvisioner.provision()` → ledger capture → `JpaCaseLineageQuery.findCompletedWorkers()`.

**Architecture:** `casehub-testing` provides `@Alternative @Priority(1)` in-memory repos for the engine's `CaseInstanceRepository` and `EventLogRepository`, eliminating the need for an engine JPA schema. A new `TestResearcherCase` CaseHub subclass defines a minimal case that triggers `ClaudonyWorkerProvisioner` via the engine's `tryProvision()` fallback (fires when no static workers match a capability). Completion is driven by publishing `WorkflowExecutionCompleted` directly to the Vert.x event bus, exactly as `WorkResultSubmitter` does internally.

**Tech Stack:** Java 21, Quarkus 3.32.2, `@QuarkusTest`, `@QuarkusTestProfile`, Mockito `@InjectMock`, Awaitility, casehub-testing 0.2-SNAPSHOT, casehub-engine-api

---

## File Map

| Action | File |
|--------|------|
| Modify | `pom.xml` — add `casehub-testing` to `dependencyManagement` |
| Modify | `app/pom.xml` — add `casehub-testing` test dep |
| Modify | `app/src/test/resources/application.properties` — index `casehub-testing` |
| Create | `app/src/test/java/io/casehub/claudony/TestResearcherCase.java` |
| Replace | `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java` |

---

## Task 1: Add casehub-testing to dependency management and app module

**Files:**
- Modify: `pom.xml` (root, `dependencyManagement` section)
- Modify: `app/pom.xml`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1: Add casehub-testing to dependencyManagement in root pom.xml**

In `/Users/mdproctor/claude/casehub/claudony/pom.xml`, add after the `casehub-qhorus-testing` entry (line ~63):

```xml
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-testing</artifactId>
        <version>${casehub.version}</version>
      </dependency>
```

- [ ] **Step 2: Add casehub-testing as a test dependency in app/pom.xml**

In `/Users/mdproctor/claude/casehub/claudony/app/pom.xml`, add after the `casehub-qhorus-testing` dep block:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-testing</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 3: Index casehub-testing in test application.properties**

In `app/src/test/resources/application.properties`, add after the `qhorus-testing` index-dependency lines:

```properties
# Index casehub-testing jar so TestCaseInstanceRepository, TestEventLogRepository,
# TestCaseMetaModelRepository (@Alternative @Priority(1)) are discoverable
quarkus.index-dependency.casehub-testing.group-id=io.casehub
quarkus.index-dependency.casehub-testing.artifact-id=casehub-testing
```

- [ ] **Step 4: Verify Quarkus starts cleanly with the new dependency**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=SmokeTest -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS` — confirms no CDI startup failure from `casehub-testing` being on the classpath.

---

## Task 2: Create TestResearcherCase

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/TestResearcherCase.java`

This is the `CaseHub` subclass used by the round-trip test. It must be a top-level `@ApplicationScoped` class — CDI scanning does not pick up inner classes in `@TestProfile` contexts.

The case definition: one "researcher" capability, one `ContextChangeTrigger` binding that fires when the "topic" key is present in context. **No static workers** — this forces the engine's `tryProvision()` fallback path in `CaseContextChangedEventHandler`, which calls `workerProvisioner.getCapabilities()` and then `provision()` when "researcher" is advertised.

- [ ] **Step 1: Write the failing test (run the final test first — it will fail because TestResearcherCase doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=CaseEngineRoundTripTest -q 2>&1 | tail -10
```

Expected: compilation failure — `TestResearcherCase` not found.

- [ ] **Step 2: Create TestResearcherCase.java**

```java
package io.casehub.claudony;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.model.Binding;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import jakarta.enterprise.context.ApplicationScoped;

/**
 * Minimal CaseHub subclass for CaseEngineRoundTripTest.
 *
 * No static workers — forces the engine's tryProvision() fallback path,
 * which calls ClaudonyWorkerProvisioner when it advertises the "researcher" capability.
 */
@ApplicationScoped
public class TestResearcherCase extends CaseHub {

    @Override
    public CaseDefinition getDefinition() {
        Capability cap = Capability.builder()
                .name("researcher")
                .inputSchema(".")
                .outputSchema(".")
                .build();

        Binding binding = Binding.builder()
                .name("start-researcher-on-topic")
                .capability(cap)
                .on(new ContextChangeTrigger(".topic != null"))
                .build();

        return CaseDefinition.builder()
                .namespace("io.casehub.claudony.test")
                .name("researcher-round-trip")
                .version("1.0.0")
                .capabilities(cap)
                .bindings(binding)
                .build();
    }
}
```

---

## Task 3: Replace CaseEngineRoundTripTest with the real engine round-trip

**Files:**
- Replace: `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java`

This replaces the CDI-event stub entirely. Key points:
- `@TestProfile(CasehubEnabledProfile.class)` — enables casehub, configures "researcher" capability. Forces a separate Quarkus instance for this test class (no interference with other tests).
- No `@TestSecurity` — this is a CDI-only test, PP-20260513-7c227e.
- `@InjectMock TmuxService` — `createSession()` stubbed to no-op; no real tmux session created.
- Completion driven by publishing `WorkflowExecutionCompleted` directly to `"casehub.worker.finished"` on the Vert.x event bus — the same mechanism `WorkResultSubmitter` uses. `WorkResultSubmitter` cannot be used here because it requires static workers in the case definition.
- `startedAt` will equal `completedAt` in the result (no `WorkerExecutionStarted` event in the `tryProvision()` path) — this is correct and covered by the `startedAtFallsBackToCompletedAtWhenNoStartEntry` test in `CaseLineageQueryIntegrationTest`.

- [ ] **Step 1: Replace CaseEngineRoundTripTest.java with full implementation**

```java
package io.casehub.claudony;

import io.casehub.api.model.Capability;
import io.casehub.api.model.Worker;
import io.casehub.api.model.WorkerSummary;
import io.casehub.claudony.casehub.JpaCaseLineageQuery;
import io.casehub.engine.internal.event.EventBusAddresses;
import io.casehub.engine.internal.event.WorkflowExecutionCompleted;
import io.casehub.engine.internal.model.CaseInstance;
import io.casehub.engine.spi.CaseInstanceRepository;
import io.casehub.claudony.server.TmuxService;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import io.vertx.mutiny.core.eventbus.EventBus;
import jakarta.inject.Inject;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.atLeastOnce;
import static org.mockito.Mockito.doNothing;
import static org.mockito.Mockito.verify;

/**
 * Full CaseEngine round-trip integration test.
 *
 * <p>Exercises: CaseHub.startCase() → engine evaluates ContextChangeTrigger →
 * ClaudonyWorkerProvisioner.provision() (TmuxService mocked) →
 * WorkflowExecutionCompleted published → ClaudonyLedgerEventCapture writes ledger →
 * JpaCaseLineageQuery.findCompletedWorkers() returns populated WorkerSummary.
 *
 * <p>CDI-only test — no HTTP endpoints exercised, no @TestSecurity needed (PP-20260513-7c227e).
 *
 * Closes #92 Refs #86
 */
@QuarkusTest
@TestProfile(CaseEngineRoundTripTest.CasehubEnabledProfile.class)
class CaseEngineRoundTripTest {

    public static class CasehubEnabledProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                    "claudony.casehub.enabled", "true",
                    "claudony.casehub.workers.commands.researcher", "claude",
                    "claudony.casehub.workers.commands.default", "claude"
            );
        }
    }

    @Inject TestResearcherCase researcherCase;
    @Inject JpaCaseLineageQuery lineageQuery;
    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject EventBus eventBus;

    @InjectMock TmuxService tmuxService;

    @Test
    void startCase_engineProvisions_andLineageReturnsCompletedSummary() throws Exception {
        // TmuxService is mocked — provision() can proceed without a real tmux session
        doNothing().when(tmuxService).createSession(anyString(), anyString(), anyString());

        // Start the case with "topic" in context — triggers ContextChangeTrigger(".topic != null")
        UUID caseId = researcherCase.startCase(Map.of("topic", "test-topic"))
                .toCompletableFuture()
                .get(15, TimeUnit.SECONDS);

        assertThat(caseId).as("startCase must return a non-null caseId").isNotNull();

        // Engine evaluates the binding asynchronously — wait for provision() to be called
        Awaitility.await()
                .atMost(Duration.ofSeconds(20))
                .pollInterval(Duration.ofMillis(200))
                .untilAsserted(() ->
                        verify(tmuxService, atLeastOnce())
                                .createSession(anyString(), anyString(), anyString()));

        // Retrieve the live CaseInstance to use in the completion event
        CaseInstance instance = caseInstanceRepository.findByUuid(caseId)
                .await().atMost(Duration.ofSeconds(5));
        assertThat(instance).as("CaseInstance must be present after startCase").isNotNull();

        // Drive completion: publish WorkflowExecutionCompleted to the engine event bus.
        // Worker name must match what ClaudonyWorkerProvisioner.provision() returns
        // (the capability name = "researcher", per the #83 workerId fix).
        Capability cap = new Capability("researcher", ".", ".");
        Worker provisioned = new Worker("researcher", List.of(cap), ctx -> Map.of("output", "done"));

        eventBus.publish(
                EventBusAddresses.WORKER_EXECUTION_FINISHED,
                new WorkflowExecutionCompleted(
                        instance, provisioned, UUID.randomUUID().toString(),
                        Map.of("output", "done")));

        // Wait for ClaudonyLedgerEventCapture (@ObservesAsync) to write the ledger entry
        Awaitility.await()
                .atMost(Duration.ofSeconds(20))
                .pollInterval(Duration.ofMillis(200))
                .untilAsserted(() -> {
                    List<WorkerSummary> workers = lineageQuery.findCompletedWorkers(caseId);
                    assertThat(workers)
                            .as("findCompletedWorkers must return at least one entry")
                            .hasSize(1);
                });

        WorkerSummary summary = lineageQuery.findCompletedWorkers(caseId).get(0);
        assertThat(summary.workerName())
                .as("workerName must be 'researcher'")
                .isEqualTo("researcher");
        assertThat(summary.completedAt())
                .as("completedAt must be populated")
                .isNotNull();
        assertThat(summary.ledgerEntryId())
                .as("ledgerEntryId must be populated")
                .isNotNull();
    }
}
```

- [ ] **Step 2: Run the test — expect failure (the test is written first)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=CaseEngineRoundTripTest -q 2>&1 | tail -20
```

Expected: test failure — `startCase()` doesn't yet work end-to-end (will diagnose from output).

---

## Task 4: Diagnose and fix any failures

The test may fail for one of these predictable reasons. Diagnose from the output and apply the matching fix.

**Failure A — CDI ambiguity on startup (DeploymentException)**

Symptom: `Unsatisfied or ambiguous dependencies for type WorkerProvisioner`

Fix: The engine no-op wasn't updated to `@DefaultBean` in the installed jar. Verify the jar:
```bash
cd /tmp/noopcheck && javap -verbose io/casehub/engine/internal/worker/NoOpWorkerProvisioner.class \
  | grep -i DefaultBean
```
If missing: re-install the engine (`mvn install -DskipTests -q -pl runtime -f ~/claude/casehub/engine/pom.xml`).

**Failure B — Awaitility timeout waiting for createSession()**

Symptom: `ConditionTimeoutException` on the first `await()`.

Diagnosis — add a temporary log or check the engine logs above the timeout for errors from `CaseContextChangedEventHandler` or `tryProvision()`. Common causes:
- `claudony.casehub.enabled` not active in the test profile → check `CasehubEnabledProfile` config keys
- `WorkerProvisioner.getCapabilities()` not returning "researcher" → check that `claudony.casehub.workers.commands.researcher=claude` is set

**Failure C — Awaitility timeout waiting for findCompletedWorkers()**

Symptom: `ConditionTimeoutException` on the second `await()`.

Diagnosis — the ledger entry may not have been written. Check logs for `ClaudonyLedgerEventCapture` DEBUG output (`Ledger entry written: ...`). If missing, `WorkflowExecutionCompleted` may not have been received by the handler. Check event bus codec registration.

**Failure D — WorkerSummary.workerName() mismatch**

Symptom: `expected "researcher" but was "default"` or similar.

Fix: `ClaudonyWorkerProvisioner` may be using a different name. Check `WorkerCommandResolver` and `WorkerSessionMapping` to confirm the capability name is used as the worker name.

- [ ] **Run test and diagnose output**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=CaseEngineRoundTripTest 2>&1 | grep -E "ERROR|WARN|FAIL|Exception|ConditionTimeout|Ledger entry" | head -30
```

- [ ] **Apply fix matching the failure mode above, then re-run**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=CaseEngineRoundTripTest -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS` and `Tests run: 1, Failures: 0, Errors: 0`.

---

## Task 5: Run full test suite and commit

- [ ] **Step 1: Run all app tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`. If count changed from 479, investigate — no regressions allowed.

- [ ] **Step 2: Commit**

```bash
git add \
  pom.xml \
  app/pom.xml \
  app/src/test/resources/application.properties \
  app/src/test/java/io/casehub/claudony/TestResearcherCase.java \
  app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java

git commit -m "$(cat <<'EOF'
test(casehub): full CaseHub.startCase() → provision → lineage round-trip

Replaces the CDI-event stub with a real engine integration test:
- CaseHub.startCase() triggers the engine's ContextChangeTrigger evaluation
- Engine's tryProvision() fallback calls ClaudonyWorkerProvisioner.provision()
  (TmuxService mocked — no real tmux session created)
- WorkflowExecutionCompleted drives completion via the Vert.x event bus
- ClaudonyLedgerEventCapture captures the lifecycle event asynchronously
- JpaCaseLineageQuery.findCompletedWorkers() asserts a populated WorkerSummary

casehub-testing provides @Alternative @Priority(1) in-memory repos for the
engine's CaseInstance and EventLog stores (no engine JPA schema required).
Engine no-op SPIs are now @DefaultBean (engine#257) so casehub-testing indexes
cleanly alongside Claudony's own SPI implementations.

Closes #92 Refs #86
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- ✅ Add casehub-testing dep (Task 1)
- ✅ Add index-dependency config (Task 1)
- ✅ TestResearcherCase top-level `@ApplicationScoped` (Task 2)
- ✅ Replace CaseEngineRoundTripTest with engine round-trip (Task 3)
- ✅ No `@TestSecurity` (PP-20260513-7c227e) (Task 3)
- ✅ `@InjectMock TmuxService` (Task 3)
- ✅ Drive completion via direct `eventBus.publish` (Task 3)
- ✅ Assert `completedAt` and `ledgerEntryId` non-null (Task 3)
- ✅ `workerName == "researcher"` assertion (Task 3)

**Placeholder scan:** None found. All code blocks are complete. All commands are exact.

**Type consistency:**
- `WorkflowExecutionCompleted(CaseInstance, Worker, String, Map<String, Object>)` — matches record definition
- `Worker("researcher", List.of(cap), ctx -> Map.of("output", "done"))` — matches `Worker(String, List<Capability>, Function<CaseContext, Map<String, Object>>)` constructor
- `Capability("researcher", ".", ".")` — matches `Capability(String name, String inputSchema, String outputSchema)` constructor
- `EventBusAddresses.WORKER_EXECUTION_FINISHED = "casehub.worker.finished"` — confirmed from source
- `lineageQuery.findCompletedWorkers(caseId)` — returns `List<WorkerSummary>`, confirmed
- `summary.workerName()`, `summary.completedAt()`, `summary.ledgerEntryId()` — confirmed from `WorkerSummary` record used in `CaseLineageQueryIntegrationTest`
