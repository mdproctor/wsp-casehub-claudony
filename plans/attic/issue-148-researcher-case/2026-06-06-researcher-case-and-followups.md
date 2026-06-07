# ResearcherCase, Exit Signalling, and Follow-ups Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a production ResearcherCase that auto-completes when the Claude Code tmux session exits, plus test coverage for tenancyId handling (#143) and bootstrapCasehubWatchers paths (#147).

**Architecture:** `ClaudonyWorkerExecutionManager` stores a `pendingExitSignals` map (caseId → roleName) when the watcher detects exit. `ClaudonyLedgerEventCapture` drains it on `WorkerExecutionCompleted` and fires `CaseHubRuntime.signal("workers.<role>.exited", true)`, patching case context. `ResearcherCase extends YamlCaseHub` carries a goal condition `.workers.researcher.exited == true`; once satisfied, the engine marks the case COMPLETED.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api (signal/context), JUnit 5, Mockito, AssertJ, Awaitility. Maven modules: `casehub/` (claudony-casehub) and `app/` (claudony-app).

**Run all tests:**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/claudony/pom.xml
```
**Run single module:**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -f /Users/mdproctor/claude/casehub/claudony/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/claudony/pom.xml
```
**Run single test class:**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClassName -f /Users/mdproctor/claude/casehub/claudony/pom.xml
```

---

## File Map

| Action | Path | Purpose |
|--------|------|---------|
| MODIFY | `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManager.java` | Add `pendingExitSignals` map + `drainExitSignal()` + put before event send |
| MODIFY | `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManagerTest.java` | Add 2 drain signal tests |
| MODIFY | `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCapture.java` | Add `@Inject` fields for `execManager` + `caseHubRuntime`; drain signal on `WorkerExecutionCompleted` |
| MODIFY | `app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureTest.java` | Add `@InjectMock ClaudonyWorkerExecutionManager`; add 2 signal tests + 2 tenancyId assertions |
| CREATE | `app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureSignalTest.java` | `CasehubEnabledProfile`-scoped tests verifying signal fires with mock runtime |
| CREATE | `casehub/src/main/java/io/casehub/claudony/casehub/ResearcherCase.java` | Production case definition |
| CREATE | `casehub/src/main/resources/casehub/researcher.yaml` | YAML case definition with goal + completion |
| CREATE | `app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java` | Default-profile test: YAML parses, fields correct |
| MODIFY | `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java` | Add `ResearcherCase` to `CasehubEnabledProfile` exclude-types |
| CREATE | `app/src/test/java/io/casehub/claudony/ResearcherCaseCompletionTest.java` | E2E: case reaches COMPLETED after watcher exit |
| CREATE | `app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java` | Extracted loop from `ServerStartup.bootstrapCasehubWatchers()` |
| CREATE | `app/src/test/java/io/casehub/claudony/server/CasehubStartupServiceTest.java` | 3 unit tests: UUID guard, null instance, roleName fallback |
| MODIFY | `app/src/main/java/io/casehub/claudony/server/ServerStartup.java` | Replace `bootstrapCasehubWatchers()` body with `CasehubStartupService` delegation |

---

## Task 1: `pendingExitSignals` in `ClaudonyWorkerExecutionManager`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManager.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManagerTest.java`

- [ ] **Step 1.1: Write failing tests for `drainExitSignal`**

Add to the end of `ClaudonyWorkerExecutionManagerTest`, before the closing `}`:

```java
// ── drainExitSignal ────────────────────────────────────────────────────────

@Test
void workerExit_storesPendingExitSignal() throws Exception {
    var caseId = UUID.randomUUID();
    var sessionId = "sig001";
    var sessionName = SESSION_PREFIX + sessionId;
    seedSession(sessionId, caseId, "researcher");
    when(tmuxService.sessionExists(sessionName)).thenReturn(false);

    manager.watch(sessionId, sessionName, caseInstance(caseId), worker("researcher"));

    // Wait for event bus send — signal is stored before send, so it's there too
    Awaitility.await()
            .atMost(Duration.ofSeconds(2))
            .untilAsserted(() ->
                    verify(eventBus).send(
                            eq(EventBusAddresses.WORKER_EXECUTION_FINISHED),
                            any(WorkflowExecutionCompleted.class)));

    assertThat(manager.drainExitSignal(caseId)).isEqualTo("researcher");
    assertThat(manager.drainExitSignal(caseId)).isNull(); // drained — second call returns null
}

@Test
void drainExitSignal_unknownCaseId_returnsNull() {
    assertThat(manager.drainExitSignal(UUID.randomUUID())).isNull();
}
```

- [ ] **Step 1.2: Run tests to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub \
  -Dtest=ClaudonyWorkerExecutionManagerTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -20
```
Expected: FAIL — `drainExitSignal` method does not exist.

- [ ] **Step 1.3: Add `pendingExitSignals` field, `drainExitSignal()`, and `put` in watcher**

In `ClaudonyWorkerExecutionManager.java`:

After the `sessionToRole` field declaration, add:
```java
/** caseId → roleName for exit signals awaiting ledger capture drain. */
private final ConcurrentHashMap<UUID, String> pendingExitSignals = new ConcurrentHashMap<>();
```

Add `drainExitSignal` method after `activeWatcherCount()`:
```java
/** Called by ClaudonyLedgerEventCapture to consume and clear the pending exit signal. */
public String drainExitSignal(UUID caseId) {
    return pendingExitSignals.remove(caseId);
}
```

In `watcherRunnable`, inside the `if (registry.remove(sessionId) != null)` block, insert `pendingExitSignals.put` **before** `eventBus.send`:
```java
if (registry.remove(sessionId) != null) {
    pendingExitSignals.put(instance.getUuid(), worker.getName()); // store before send
    final String idempotencyKey =
            instance.getUuid() + ":" + worker.getName() + ":" + sessionId;
    eventBus.send(EventBusAddresses.WORKER_EXECUTION_FINISHED,
            new WorkflowExecutionCompleted(
                    instance, worker, idempotencyKey, Map.of()));
}
```

- [ ] **Step 1.4: Run tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub \
  -Dtest=ClaudonyWorkerExecutionManagerTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS, 13 tests passing (11 existing + 2 new).

- [ ] **Step 1.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManager.java \
  casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManagerTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "feat(casehub): pendingExitSignals drain in ClaudonyWorkerExecutionManager Refs #148"
```

---

## Task 2: Signal drain in `ClaudonyLedgerEventCapture`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCapture.java`
- Modify: `app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureTest.java`
- Create: `app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureSignalTest.java`

- [ ] **Step 2.1: Write failing tests for signal drain in `ClaudonyLedgerEventCaptureTest`**

Add import and field at top of `ClaudonyLedgerEventCaptureTest`:
```java
import io.casehub.claudony.casehub.ClaudonyWorkerExecutionManager;
import io.quarkus.test.InjectMock;
import static org.mockito.Mockito.when;
import static org.mockito.Mockito.never;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;

// In class body, alongside existing @Inject fields:
@InjectMock
ClaudonyWorkerExecutionManager execManager;
```

Add two tests (before the `findByCaseId` helper method):
```java
@Test
@TestTransaction
void workerExecutionCompleted_noPendingSignal_doesNotFireSignal() {
    UUID caseId = UUID.randomUUID();
    when(execManager.drainExitSignal(caseId)).thenReturn(null);

    lifecycleEvents.fireAsync(new CaseLifecycleEvent(
                    caseId, null, "ExecuteWorker", "WorkerExecutionCompleted",
                    "ACTIVE", "system", "SYSTEM", null))
            .toCompletableFuture().join();

    // No exception = signal guard (isUnsatisfied or null drain) worked correctly.
    // In default profile CaseHubRuntime is unsatisfied, so signal block is never entered.
    var entries = findByCaseId(caseId);
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).eventType).isEqualTo("WorkerExecutionCompleted");
}

@Test
@TestTransaction
void workerExecutionStarted_drainedCausalContext_andWorkerCompleted_bothHandled() {
    UUID caseId = UUID.randomUUID();
    UUID causedBy = UUID.randomUUID();
    provisioner.seedCausalContextForTest(caseId, causedBy);
    when(execManager.drainExitSignal(caseId)).thenReturn(null);

    lifecycleEvents.fireAsync(new CaseLifecycleEvent(
                    caseId, null, "ProvisionWorker", "WorkerStarted", null, null, "System", null))
            .toCompletableFuture().join();
    lifecycleEvents.fireAsync(new CaseLifecycleEvent(
                    caseId, null, "ExecuteWorker", "WorkerExecutionCompleted",
                    "ACTIVE", "system", "SYSTEM", null))
            .toCompletableFuture().join();

    var entries = findByCaseId(caseId);
    assertThat(entries).hasSize(2);
    assertThat(entries.get(0).causedByEntryId).isEqualTo(causedBy);
    assertThat(entries.get(1).eventType).isEqualTo("WorkerExecutionCompleted");
}
```

- [ ] **Step 2.2: Confirm tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ClaudonyLedgerEventCaptureTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -20
```
Expected: FAIL — `execManager` cannot be `@InjectMock`-ed because `ClaudonyLedgerEventCapture` does not inject it yet.

- [ ] **Step 2.3: Add `@Inject` fields and signal drain to `ClaudonyLedgerEventCapture`**

In `ClaudonyLedgerEventCapture.java`, add two new imports:
```java
import io.casehub.api.engine.CaseHubRuntime;
import jakarta.enterprise.inject.Instance;
```

Add two new `@Inject` fields (after the existing `@Inject ClaudonyReactiveWorkerProvisioner provisioner`):
```java
@Inject
ClaudonyWorkerExecutionManager execManager;

@Inject
Instance<CaseHubRuntime> caseHubRuntime;
```

In `onCaseLifecycleEvent()`, add after the existing `WorkerStarted` causal-context drain block (after the closing `}` of the `if ("WorkerStarted"...)` check):
```java
if ("WorkerExecutionCompleted".equals(event.eventType())) {
    String roleName = execManager.drainExitSignal(event.caseId());
    if (roleName != null && !caseHubRuntime.isUnsatisfied()) {
        caseHubRuntime.get().signal(
                event.caseId(), "workers." + roleName + ".exited", true);
    }
}
```

- [ ] **Step 2.4: Run ledger capture tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ClaudonyLedgerEventCaptureTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS, all tests passing.

- [ ] **Step 2.5: Create `ClaudonyLedgerEventCaptureSignalTest`**

This test class uses `CasehubEnabledProfile` (where `CaseHubRuntimeImpl` is active) to verify the signal fires when the runtime is available. `@InjectMock CaseHubRuntime` replaces the real bean with a mock.

Create `app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureSignalTest.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.claudony.CaseEngineRoundTripTest;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.mockito.Mockito.*;

/**
 * Verifies that ClaudonyLedgerEventCapture fires a context signal on WorkerExecutionCompleted
 * when CaseHubRuntime is available and execManager returns a pending role.
 *
 * Uses CasehubEnabledProfile so CaseHubRuntimeImpl is in CDI and can be @InjectMock-ed.
 */
@QuarkusTest
@TestProfile(CaseEngineRoundTripTest.CasehubEnabledProfile.class)
class ClaudonyLedgerEventCaptureSignalTest {

    @Inject
    Event<CaseLifecycleEvent> lifecycleEvents;

    @InjectMock
    ClaudonyWorkerExecutionManager execManager;

    @InjectMock
    CaseHubRuntime runtimeMock;

    @Test
    void workerExecutionCompleted_withPendingSignal_firesContextSignal() throws Exception {
        UUID caseId = UUID.randomUUID();
        when(execManager.drainExitSignal(caseId)).thenReturn("researcher");

        lifecycleEvents.fireAsync(new CaseLifecycleEvent(
                        caseId, null, "ExecuteWorker", "WorkerExecutionCompleted",
                        "ACTIVE", "system", "SYSTEM", null))
                .toCompletableFuture().get(5, java.util.concurrent.TimeUnit.SECONDS);

        verify(runtimeMock).signal(caseId, "workers.researcher.exited", true);
    }

    @Test
    void workerExecutionCompleted_noPendingSignal_doesNotFireSignal() throws Exception {
        UUID caseId = UUID.randomUUID();
        when(execManager.drainExitSignal(caseId)).thenReturn(null);

        lifecycleEvents.fireAsync(new CaseLifecycleEvent(
                        caseId, null, "ExecuteWorker", "WorkerExecutionCompleted",
                        "ACTIVE", "system", "SYSTEM", null))
                .toCompletableFuture().get(5, java.util.concurrent.TimeUnit.SECONDS);

        verify(runtimeMock, never()).signal(any(), anyString(), any());
    }
}
```

- [ ] **Step 2.6: Run signal test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ClaudonyLedgerEventCaptureSignalTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -15
```
Expected: BUILD SUCCESS, 2 tests passing.

- [ ] **Step 2.7: Run all casehub module tests to confirm nothing broken**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 2.8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCapture.java \
  app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureTest.java \
  app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureSignalTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "feat(casehub): drain exit signal in ClaudonyLedgerEventCapture Refs #148"
```

---

## Task 3: `ResearcherCase` + YAML + startup test

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/ResearcherCase.java`
- Create: `casehub/src/main/resources/casehub/researcher.yaml`
- Create: `app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java`

- [ ] **Step 3.1: Write failing startup test**

Create `app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import io.casehub.api.model.ContextChangeTrigger;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Verifies the production YAML case definition parses correctly.
 * No engine required — runs in the default test profile.
 */
@QuarkusTest
class ResearcherCaseStartupTest {

    @Inject
    ResearcherCase researcherCase;

    @Test
    void yamlLoads_withExpectedMetadata() {
        var def = researcherCase.getDefinition();

        assertThat(def).isNotNull();
        assertThat(def.name()).isEqualTo("researcher");
        assertThat(def.namespace()).isEqualTo("claudony");
        assertThat(def.getCapabilities())
                .anyMatch(c -> "researcher".equals(c.getName()));
        assertThat(def.getBindings())
                .anyMatch(b ->
                        b.getOn() instanceof ContextChangeTrigger ctx
                        && ctx.getFilter() instanceof JQExpressionEvaluator jq
                        && ".topic != null".equals(jq.expression()));
    }
}
```

- [ ] **Step 3.2: Confirm test fails (class missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ResearcherCaseStartupTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -15
```
Expected: FAIL — `ResearcherCase` not found.

- [ ] **Step 3.3: Create `ResearcherCase.java`**

Create `casehub/src/main/java/io/casehub/claudony/casehub/ResearcherCase.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.engine.YamlCaseHub;
import jakarta.enterprise.context.ApplicationScoped;

/**
 * Production researcher case definition. Loaded from classpath YAML.
 * When CaseHub is disabled (CaseHubRuntime absent), this bean exists in CDI
 * but startCase() is never called — no effect on non-CaseHub deployments.
 */
@ApplicationScoped
public class ResearcherCase extends YamlCaseHub {

    public ResearcherCase() {
        super("casehub/researcher.yaml");
    }
}
```

- [ ] **Step 3.4: Create `researcher.yaml`**

Create `casehub/src/main/resources/casehub/researcher.yaml`:

```yaml
dsl: "0.1"
namespace: claudony
name: researcher
version: "1.0.0"
title: "Researcher Case"
spec:
  capabilities:
    - name: researcher
      description: "Claude Code research worker started as a tmux session via Claudony"
      inputSchema: "{}"
      outputSchema: "{}"
  bindings:
    - name: start-researcher-on-topic
      capability: researcher
      on:
        contextChange:
          filter: ".topic != null"
  goals:
    - name: research-complete
      condition: ".workers.researcher.exited == true"
      kind: success
  completion:
    success:
      allOf:
        - research-complete
```

- [ ] **Step 3.5: Run startup test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ResearcherCaseStartupTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS, 1 test passing.

- [ ] **Step 3.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  casehub/src/main/java/io/casehub/claudony/casehub/ResearcherCase.java \
  casehub/src/main/resources/casehub/researcher.yaml \
  app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "feat(casehub): add ResearcherCase production case definition Refs #148"
```

---

## Task 4: Exclude `ResearcherCase` from `CasehubEnabledProfile`

`ResearcherCase` is now a production CDI bean. Without exclusion, both `ResearcherCase` and `TestResearcherCase` would be active in `CaseEngineRoundTripTest`, with duplicate `.topic != null` bindings.

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java`

- [ ] **Step 4.1: Add `ResearcherCase` to `CasehubEnabledProfile` exclude-types**

In `CaseEngineRoundTripTest.java`, find the `quarkus.arc.exclude-types` value in `CasehubEnabledProfile.getConfigOverrides()`. It currently ends with `"io.casehub.work.core.strategy.RoundRobinStrategy"`. Append `,` and the new entry:

The final value should end with:
```java
+ "io.casehub.engine.internal.orchestration.WorkOrchestrator,"
+ "io.casehub.work.core.strategy.RoundRobinStrategy,"
+ "io.casehub.claudony.casehub.ResearcherCase"
```

- [ ] **Step 4.2: Run `CaseEngineRoundTripTest` to confirm it still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=CaseEngineRoundTripTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 4.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "test(casehub): exclude ResearcherCase from CasehubEnabledProfile Refs #148"
```

---

## Task 5: `ResearcherCaseCompletionTest` — E2E proof

This test verifies the full chain: case start → provision → watcher exit → signal → COMPLETED. It uses a new profile that includes `SignalReceivedEventHandler` and `CaseStatusChangedHandler` (both excluded in `CasehubEnabledProfile`).

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/ResearcherCaseCompletionTest.java`

- [ ] **Step 5.1: Create `ResearcherCaseCompletionTest.java`**

```java
package io.casehub.claudony;

import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.Worker;
import io.casehub.claudony.casehub.ClaudonyWorkerExecutionManager;
import io.casehub.claudony.casehub.ResearcherCase;
import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TmuxService;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.event.Event;
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
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

/**
 * E2E proof: ResearcherCase auto-completes when the tmux session exits.
 *
 * Chain: startCase → provision → watch → exit detected → pendingExitSignals.put →
 *   WorkflowExecutionCompleted → WorkerExecutionCompleted lifecycle event →
 *   ClaudonyLedgerEventCapture.drainExitSignal → CaseHubRuntime.signal →
 *   context patched → CONTEXT_CHANGED → goal satisfied → case COMPLETED.
 */
@QuarkusTest
@TestProfile(ResearcherCaseCompletionTest.ResearcherCaseCasehubProfile.class)
class ResearcherCaseCompletionTest {

    /**
     * Same base as CasehubEnabledProfile (CaseEngineRoundTripTest) with two differences:
     * 1. SignalReceivedEventHandler NOT excluded — processes workers.<role>.exited signal.
     * 2. CaseStatusChangedHandler NOT excluded — updates case state to COMPLETED.
     * 3. TestResearcherCase excluded — prevents dual registration with ResearcherCase.
     * 4. ResearcherCase NOT excluded — it is the case definition under test.
     */
    public static class ResearcherCaseCasehubProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                    "claudony.casehub.enabled", "true",
                    "claudony.casehub.workers.commands.researcher", "claude",
                    "claudony.casehub.workers.commands.default", "claude",
                    "claudony.mode", "agent",
                    "quarkus.index-dependency.casehub-engine.group-id", "io.casehub",
                    "quarkus.index-dependency.casehub-engine.artifact-id", "casehub-engine",
                    "quarkus.arc.exclude-types",
                    "io.casehub.ledger.repository.CaseLedgerEntryRepository,"
                    + "io.casehub.ledger.service.CaseLedgerEventCapture,"
                    + "io.casehub.ledger.service.WorkerDecisionEventCapture,"
                    + "io.casehub.persistence.memory.InMemoryCaseInstanceRepository,"
                    + "io.casehub.persistence.memory.InMemoryCaseMetaModelRepository,"
                    + "io.casehub.persistence.memory.InMemoryEventLogRepository,"
                    + "io.casehub.testing.WorkResultSubmitter,"
                    + "io.casehub.engine.internal.engine.handler.MilestoneActivatedEventHandler,"
                    + "io.casehub.engine.internal.engine.handler.MilestoneCompletedEventHandler,"
                    + "io.casehub.engine.internal.engine.handler.WorkerScheduleEventHandler,"
                    + "io.casehub.engine.internal.engine.recovery.DefaultWorkerExecutionRecoveryService,"
                    + "io.casehub.engine.internal.orchestration.WorkOrchestrator,"
                    + "io.casehub.work.core.strategy.RoundRobinStrategy,"
                    + "io.casehub.claudony.TestResearcherCase"
                    // NOT excluded: SignalReceivedEventHandler, CaseStatusChangedHandler, ResearcherCase
            );
        }
    }

    @Inject ResearcherCase researcherCase;
    @Inject SessionRegistry sessionRegistry;
    @Inject CrossTenantCaseInstanceRepository caseInstanceRepository;
    @Inject ClaudonyWorkerExecutionManager execManager;
    @Inject Event<CaseLifecycleEvent> lifecycleEvents;

    @InjectMock TmuxService tmuxService;

    @Test
    void researcherCase_completesWhenWorkerSessionExits() throws Exception {
        // Start a case with topic in context
        UUID caseId = researcherCase.startCase(Map.of("topic", "test-topic"))
                .toCompletableFuture()
                .get(10, TimeUnit.SECONDS);

        // Wait for provision — ClaudonyReactiveWorkerProvisioner calls createWorkerSession
        Awaitility.await()
                .atMost(Duration.ofSeconds(10))
                .pollInterval(Duration.ofMillis(200))
                .untilAsserted(() ->
                        verify(tmuxService, atLeastOnce())
                                .createWorkerSession(anyString(), anyString(), anyString()));

        // WorkOrchestrator excluded — start watcher manually (same pattern as CaseEngineRoundTripTest)
        var session = sessionRegistry.findByCaseId(caseId.toString()).get(0);
        CaseInstance instance = caseInstanceRepository.findByUuid(caseId)
                .await().atMost(Duration.ofSeconds(5));
        var cap = new Capability("researcher", "{}", "{}");
        var worker = new Worker("researcher", List.of(cap), ctx -> Map.of());

        // Fire WorkerExecutionStarted so lineage resolves correctly (engine#390: actorId="system" on Completed)
        lifecycleEvents.fireAsync(new CaseLifecycleEvent(
                caseId, null, "ExecuteWorker", "WorkerExecutionStarted", "ACTIVE",
                "researcher", "WORKER", null)).toCompletableFuture().get(5, TimeUnit.SECONDS);

        // Simulate tmux session exit — watcher detects false, fires completion + stores signal
        when(tmuxService.sessionExists(anyString())).thenReturn(false);
        execManager.watch(session.id(), session.name(), instance, worker);

        // Assert case reaches COMPLETED:
        // exit → WorkflowExecutionCompleted → WorkerExecutionCompleted lifecycle →
        // drainExitSignal → signal → context patched → goal met → COMPLETED
        Awaitility.await()
                .atMost(Duration.ofSeconds(15))
                .pollInterval(Duration.ofMillis(300))
                .untilAsserted(() -> {
                    CaseInstance updated = caseInstanceRepository.findByUuid(caseId)
                            .await().atMost(Duration.ofSeconds(5));
                    assertThat(updated.getState())
                            .as("case state after researcher exit")
                            .isEqualTo(CaseStatus.COMPLETED);
                });
    }
}
```

- [ ] **Step 5.2: Run the completion test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ResearcherCaseCompletionTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -20
```
Expected: BUILD SUCCESS, 1 test passing.

**If the test times out waiting for COMPLETED:** the engine may need `CaseStatusChangedHandler` or `GoalReachedEventHandler` explicitly active. Check if they appear in the `app/src/test/resources/application.properties` default exclude-types and add them to `ResearcherCaseCasehubProfile.quarkus.arc.exclude-types` removal (i.e., do NOT list them in that profile's exclusions). Also confirm `CaseContextChangedEventHandler` is not in the exclusions.

- [ ] **Step 5.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/test/java/io/casehub/claudony/ResearcherCaseCompletionTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "test(casehub): ResearcherCaseCompletionTest — E2E proof case reaches COMPLETED Refs #148"
```

---

## Task 6: tenancyId assertions (#143)

`ClaudonyLedgerEventCapture` already stores `tenancyId` correctly. This task adds the missing test assertions.

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureTest.java`

- [ ] **Step 6.1: Add `assertThat(entry.tenancyId)` to `happyPath_singleEvent_writesLedgerEntry`**

Find the existing `happyPath_singleEvent_writesLedgerEntry` test. After the existing assertions, add:
```java
assertThat(entry.tenancyId).isEqualTo("default"); // null tenancyId stored as "default"
```

- [ ] **Step 6.2: Add new `tenancyId_nonNull_storedAsIs` test**

Add before the `findByCaseId` helper:
```java
@Test
@TestTransaction
void tenancyId_nonNull_storedAsIs() {
    UUID caseId = UUID.randomUUID();

    lifecycleEvents.fireAsync(new CaseLifecycleEvent(
                    caseId, "tenant-1", "StartCase", "CaseStarted", "RUNNING", null, "System", null))
            .toCompletableFuture().join();

    List<CaseLedgerEntry> entries = findByCaseId(caseId);
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).tenancyId).isEqualTo("tenant-1");
}
```

- [ ] **Step 6.3: Run ledger capture tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ClaudonyLedgerEventCaptureTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS, all tests passing (existing 15 + 3 new = 18).

- [ ] **Step 6.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/test/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCaptureTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "test(casehub): add tenancyId assertions to ClaudonyLedgerEventCaptureTest Closes #143"
```

---

## Task 7: `CasehubStartupService` extraction (#147)

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java`
- Create: `app/src/test/java/io/casehub/claudony/server/CasehubStartupServiceTest.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/ServerStartup.java`

- [ ] **Step 7.1: Write failing unit tests**

Create `app/src/test/java/io/casehub/claudony/server/CasehubStartupServiceTest.java`:

```java
package io.casehub.claudony.server;

import io.casehub.claudony.casehub.ClaudonyWorkerExecutionManager;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class CasehubStartupServiceTest {

    private SessionRegistry registry;
    private CrossTenantCaseInstanceRepository caseInstanceRepo;
    private ClaudonyWorkerExecutionManager execManager;
    private CasehubStartupService service;

    @BeforeEach
    void setUp() {
        registry = new SessionRegistry();
        caseInstanceRepo = mock(CrossTenantCaseInstanceRepository.class);
        execManager = mock(ClaudonyWorkerExecutionManager.class);
        service = new CasehubStartupService(registry, caseInstanceRepo, execManager);
    }

    @Test
    void invalidCaseId_logsWarnAndSkips() {
        // Session has a non-UUID caseId string
        registry.register(session("s1", "not-a-uuid", "researcher"));

        int started = service.bootstrapWatchers();

        assertThat(started).isEqualTo(0);
        verify(execManager, never()).watch(any(), any(), any(), any());
    }

    @Test
    void nullCaseInstance_logsInfoAndSkips() {
        UUID caseId = UUID.randomUUID();
        registry.register(session("s2", caseId.toString(), "researcher"));
        when(caseInstanceRepo.findByUuid(caseId))
                .thenReturn(Uni.createFrom().item((CaseInstance) null));

        int started = service.bootstrapWatchers();

        assertThat(started).isEqualTo(0);
        verify(execManager, never()).watch(any(), any(), any(), any());
    }

    @Test
    void absentRoleName_fallsBackToWorker() {
        UUID caseId = UUID.randomUUID();
        // Session with no roleName (empty Optional)
        registry.register(sessionNoRole("s3", caseId.toString()));
        CaseInstance inst = new CaseInstance();
        inst.setUuid(caseId);
        when(caseInstanceRepo.findByUuid(caseId))
                .thenReturn(Uni.createFrom().item(inst));

        int started = service.bootstrapWatchers();

        assertThat(started).isEqualTo(1);
        verify(execManager).watch(
                eq("s3"),
                argThat(name -> name.contains("s3")),
                eq(inst),
                argThat(w -> "worker".equals(w.getName())));
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    private io.casehub.claudony.server.model.Session session(
            String id, String caseId, String role) {
        return new io.casehub.claudony.server.model.Session(
                id,
                io.casehub.claudony.casehub.ClaudonyReactiveWorkerProvisioner.SESSION_PREFIX + id,
                "/tmp", "claude",
                io.casehub.claudony.server.model.SessionStatus.IDLE,
                Instant.now(), Instant.now(),
                Optional.empty(),
                Optional.of(caseId),
                Optional.of(role));
    }

    private io.casehub.claudony.server.model.Session sessionNoRole(String id, String caseId) {
        return new io.casehub.claudony.server.model.Session(
                id,
                io.casehub.claudony.casehub.ClaudonyReactiveWorkerProvisioner.SESSION_PREFIX + id,
                "/tmp", "claude",
                io.casehub.claudony.server.model.SessionStatus.IDLE,
                Instant.now(), Instant.now(),
                Optional.empty(),
                Optional.of(caseId),
                Optional.empty()); // no roleName
    }
}
```

- [ ] **Step 7.2: Confirm tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=CasehubStartupServiceTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: FAIL — `CasehubStartupService` does not exist.

- [ ] **Step 7.3: Create `CasehubStartupService.java`**

Create `app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java`:

```java
package io.casehub.claudony.server;

import io.casehub.api.model.Capability;
import io.casehub.api.model.Worker;
import io.casehub.claudony.casehub.ClaudonyReactiveWorkerProvisioner;
import io.casehub.claudony.casehub.ClaudonyWorkerExecutionManager;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.List;
import java.util.UUID;

/**
 * Extracted from ServerStartup.bootstrapCasehubWatchers().
 * Plain Java — no CDI annotations, so unit tests instantiate directly.
 */
class CasehubStartupService {

    private static final Logger LOG = Logger.getLogger(CasehubStartupService.class);

    private final SessionRegistry registry;
    private final CrossTenantCaseInstanceRepository caseInstanceRepo;
    private final ClaudonyWorkerExecutionManager execManager;

    CasehubStartupService(
            SessionRegistry registry,
            CrossTenantCaseInstanceRepository caseInstanceRepo,
            ClaudonyWorkerExecutionManager execManager) {
        this.registry = registry;
        this.caseInstanceRepo = caseInstanceRepo;
        this.execManager = execManager;
    }

    int bootstrapWatchers() {
        int started = 0;
        for (var session : registry.all()) {
            if (session.caseId().isEmpty()) continue;
            UUID caseId;
            try {
                caseId = UUID.fromString(session.caseId().get());
            } catch (IllegalArgumentException e) {
                LOG.warnf("Invalid caseId in registry for session %s — skipping", session.id());
                continue;
            }
            try {
                CaseInstance instance = caseInstanceRepo.findByUuid(caseId)
                        .await().atMost(Duration.ofSeconds(5));
                if (instance == null) {
                    LOG.infof("No CaseInstance for caseId %s — skipping recovery watcher", caseId);
                    continue;
                }
                var roleName = session.roleName().orElse("worker");
                var worker = new Worker(roleName, List.of(), ctx -> java.util.Map.of());
                execManager.watch(session.id(), session.name(), instance, worker);
                started++;
            } catch (Exception e) {
                LOG.errorf(e, "Failed to recover watcher for caseId %s session %s — skipping",
                        caseId, session.id());
            }
        }
        return started;
    }
}
```

- [ ] **Step 7.4: Run unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=CasehubStartupServiceTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS, 3 tests passing.

- [ ] **Step 7.5: Update `ServerStartup.bootstrapCasehubWatchers()`**

Replace the body of `ServerStartup.bootstrapCasehubWatchers()` with delegation to `CasehubStartupService`. The existing `caseInstanceRepo` and `workerExecManager` `Instance<>` fields and guards stay:

```java
void bootstrapCasehubWatchers() {
    if (caseInstanceRepo.isUnsatisfied()) {
        LOG.debug("CrossTenantCaseInstanceRepository not available — skipping casehub watcher recovery");
        return;
    }
    if (workerExecManager.isUnsatisfied()) {
        LOG.debug("ClaudonyWorkerExecutionManager not available — skipping casehub watcher recovery");
        return;
    }
    int started = new CasehubStartupService(
                    registry, caseInstanceRepo.get(), workerExecManager.get())
            .bootstrapWatchers();
    LOG.infof("Started %d casehub watcher(s) for recovered sessions", started);
}
```

- [ ] **Step 7.6: Run `ServerStartupTest` to confirm nothing broken**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=ServerStartupTest \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 7.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java \
  app/src/test/java/io/casehub/claudony/server/CasehubStartupServiceTest.java \
  app/src/main/java/io/casehub/claudony/server/ServerStartup.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "refactor(server): extract bootstrapCasehubWatchers to CasehubStartupService + unit tests Closes #147"
```

---

## Task 8: Full test suite + verification

- [ ] **Step 8.1: Run complete test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml 2>&1 | tail -20
```
Expected: BUILD SUCCESS. Update the test baseline count in `CLAUDE.md` (currently 560).

- [ ] **Step 8.2: Update test count in `CLAUDE.md`**

Find the line: `**Baseline (as of 2026-06-05, after #146 tmux exit watcher):** 4 in claudony-core + 170 in claudony-casehub + 386 in claudony-app = **560 total, all passing**.`

Update with the new date and new count from the test run output.

- [ ] **Step 8.3: Update test descriptions in `CLAUDE.md`**

Add entries for the new test classes in the relevant sections of `CLAUDE.md`:
- Under `claudony-casehub` tests: `ClaudonyWorkerExecutionManagerTest` — note 2 new drain tests
- Under `claudony-app` tests: `ResearcherCaseStartupTest`, `ResearcherCaseCompletionTest`, `ClaudonyLedgerEventCaptureSignalTest`, `CasehubStartupServiceTest`

- [ ] **Step 8.4: Commit CLAUDE.md**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "docs(claude): update test baseline and descriptions after #148 #143 #147"
```
