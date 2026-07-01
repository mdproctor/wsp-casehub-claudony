# Design: ResearcherCase, Exit Signalling, and Branch Follow-ups

**Branch:** `issue-148-researcher-case`  
**Covers:** #148 (ResearcherCase + exit signalling), #143 (tenancyId tests), #147 (bootstrapCasehubWatchers tests)  
**Date:** 2026-06-06 (rev 3)

---

## Problem

Claudony has no production case definition. `TestResearcherCase` is test-only and cannot be activated in a running server. There is also no mechanism for a case to auto-complete when the researcher worker (a Claude Code tmux session) exits — cases would stay RUNNING indefinitely.

---

## Architecture: Exit Signalling via `CaseHubRuntime.signal()`

`CaseHubRuntime.signal(caseId, path, value)` is a **direct context patch** using dot-notation paths. `CaseContextImpl.applyAndDiff(path, value)` (confirmed from source) splits `path` on `.` and creates nested maps, then fires `CONTEXT_CHANGED`. This triggers goal evaluation.

`signal(caseId, "workers.researcher.exited", true)` produces `{ "workers": { "researcher": { "exited": true } } }`, satisfying `.workers.researcher.exited == true` — with zero changes to Claude's workflow. Multiple workers compose cleanly under `workers.<roleName>.exited`.

### Signal ordering: `drainExitSignal` pattern

**Correctness guarantee:** The goal `.workers.researcher.exited == true` can only be satisfied by the signal's `CONTEXT_CHANGED` — there is no other code path that sets this key. The first `CONTEXT_CHANGED` (fired by `WorkflowExecutionCompletedHandler` after applying worker output) evaluates the goal and finds it false. The second `CONTEXT_CHANGED` (fired by `SignalReceivedEventHandler` after `drainExitSignal`) finds it true and completes the case. The signal can race with the handler's remaining work, but since the condition cannot be true until the signal fires, no race produces an incorrect completion.

**Engine contract dependency:** This assumes `WorkflowExecutionCompletedHandler` fires the CDI lifecycle event only after committing its own engine state updates (event log, context, worker status). Claudony depends on this ordering being upheld by the engine. If the handler fires early (mid-update), the signal could race with uncommitted state.

**Implementation:** `ClaudonyWorkerExecutionManager` stores `pendingExitSignals: ConcurrentHashMap<UUID, String>` (caseId → roleName). When the watcher wins `registry.remove()`:

```java
pendingExitSignals.put(instance.getUuid(), worker.getName());
eventBus.send(EventBusAddresses.WORKER_EXECUTION_FINISHED,
        new WorkflowExecutionCompleted(instance, worker, idempotencyKey, Map.of()));
```

`ClaudonyLedgerEventCapture` drains the signal on `WorkerExecutionCompleted` — which fires as a CDI async event after `WorkflowExecutionCompletedHandler` commits engine state.

`ClaudonyWorkerExecutionManager` exposes:
```java
public String drainExitSignal(UUID caseId) {
    return pendingExitSignals.remove(caseId);
}
```

**No constructor change.** The `pendingExitSignals` map and `drainExitSignal()` are internal additions. Existing 11 tests in `ClaudonyWorkerExecutionManagerTest` are unaffected.

**`shutdown()` — no explicit clear.** `pendingExitSignals` is not cleared in `shutdown()`. The map is GC'd with the bean. Clearing it would race with in-flight event bus messages: if the event was already queued but `clear()` runs before `drainExitSignal()` is called, the signal is lost and the case stays RUNNING. Omitting the clear eliminates this race. Note: if the server shuts down while a `WorkflowExecutionCompleted` event is in-flight on the Vert.x event bus, the signal may be lost and the case stays RUNNING. Recovery requires restart (watcher bootstrap handles session recovery, but context signal must be re-fired manually).

**Interrupt path — no leak window.** `pendingExitSignals.put()` and `eventBus.send()` are both non-blocking, non-interruptible operations. The thread interrupt flag does not prevent either from executing — they always run as a sequential pair before any interrupt check can take effect. The watcher's interrupt checks occur only at the top of the poll loop and inside blocking operations (`Thread.sleep()`, `sessionExists()`). No meaningful gap exists between put() and send() where an interrupt could cause put() to succeed and send() to fail. The watcher's `finally` block does not touch `pendingExitSignals`.

### Signal convention — known limitation

The `workers.<roleName>.exited` convention is baked into `ClaudonyLedgerEventCapture`. Case definitions that want different exit semantics cannot opt out. Future work: add `onExitSignal` to `Capability`, allowing case authors to configure or suppress the signal path.

---

## #148 — ResearcherCase and Exit Signalling

### New files

**`claudony-casehub/src/main/java/io/casehub/claudony/casehub/ResearcherCase.java`**

```java
@ApplicationScoped
public class ResearcherCase extends YamlCaseHub {
    public ResearcherCase() { super("casehub/researcher.yaml"); }
}
```

Extends `YamlCaseHub` (platform protocol; YAML loaded lazily via `CaseDefinitionYamlMapper.load()`). When `CaseHubRuntime` is absent, the bean exists in CDI but `startCase()` is never called.

**`claudony-casehub/src/main/resources/casehub/researcher.yaml`**

```yaml
dsl: "0.1"
namespace: io.casehub.claudony
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

`inputSchema: "{}"` matches `TestResearcherCase`. Note: Claudony workers receive context via the system prompt from `ClaudonyWorkerContextProvider`, not via the engine's input mapping. `inputSchema` is intentionally minimal — a populated schema would map context to input data that the provisioner ignores.

### Changes to `ClaudonyWorkerExecutionManager`

Add `private final ConcurrentHashMap<UUID, String> pendingExitSignals = new ConcurrentHashMap<>()`.

In `watcherRunnable`, after `registry.remove(sessionId) != null` wins:
```java
pendingExitSignals.put(instance.getUuid(), worker.getName());
eventBus.send(EventBusAddresses.WORKER_EXECUTION_FINISHED,
        new WorkflowExecutionCompleted(instance, worker, idempotencyKey, Map.of()));
```

Expose `drainExitSignal(UUID)` (public, no annotation). No changes to constructor or `shutdown()`.

### Changes to `ClaudonyLedgerEventCapture`

Add two `@Inject` fields — `ClaudonyWorkerExecutionManager execManager` and `Instance<CaseHubRuntime> caseHubRuntime` — matching the existing field injection style in `ClaudonyLedgerEventCapture` (which already uses `@Inject` fields for `em`, `provisioner`, etc.).

In `onCaseLifecycleEvent()`, add after the existing `WorkerStarted` causal-context drain:

```java
if ("WorkerExecutionCompleted".equals(event.eventType())) {
    String roleName = execManager.drainExitSignal(event.caseId());
    if (roleName != null && !caseHubRuntime.isUnsatisfied()) {
        caseHubRuntime.get().signal(
            event.caseId(), "workers." + roleName + ".exited", true);
    }
}
```

No circular dependency: `ClaudonyWorkerExecutionManager` does not inject `ClaudonyLedgerEventCapture`.

### `CasehubEnabledProfile` — add `ResearcherCase` to exclude-types

`CaseEngineRoundTripTest.CasehubEnabledProfile` must add `io.casehub.claudony.casehub.ResearcherCase` to its `quarkus.arc.exclude-types` string. The production `ResearcherCase` and `TestResearcherCase` would otherwise both be active — they have different `namespace:name` identifiers but the `.topic != null` binding would fire on both. The YAML is on the test classpath since `claudony-casehub` is a compile dep of `claudony-app`.

### New tests

**`ResearcherCaseStartupTest`** — default profile (no engine required):

Inject `ResearcherCase researcherCase`. Assert on `researcherCase.getDefinition()`:

```java
var def = researcherCase.getDefinition();
assertThat(def.name()).isEqualTo("researcher");
assertThat(def.namespace()).isEqualTo("io.casehub.claudony");
assertThat(def.getCapabilities()).anyMatch(c -> "researcher".equals(c.getName()));
// Binding API (verified against decompiled jar):
// Binding.getOn() returns Trigger; ContextChangeTrigger.getFilter() returns ExpressionEvaluator;
// JQExpressionEvaluator.expression() returns the expression String.
assertThat(def.getBindings()).anyMatch(b ->
    b.getOn() instanceof ContextChangeTrigger ctx
    && ctx.getFilter() instanceof JQExpressionEvaluator jq
    && ".topic != null".equals(jq.expression()));
```

**`ResearcherCaseCompletionTest`** — new `ResearcherCaseCasehubProfile`:

Profile config: same as `CasehubEnabledProfile` with two differences:
1. `TestResearcherCase` added to `quarkus.arc.exclude-types` (`io.casehub.claudony.TestResearcherCase`)
2. `SignalReceivedEventHandler` **removed** from `quarkus.arc.exclude-types` — it is excluded in `CasehubEnabledProfile` but is required here to process the `workers.researcher.exited` signal that completes the case. `WorkOrchestrator` remains excluded.

Test pattern (follows `CaseEngineRoundTripTest`):

```java
@InjectMock TmuxService tmuxService;
@Inject ResearcherCase researcherCase;
@Inject SessionRegistry sessionRegistry;
@Inject CrossTenantCaseInstanceRepository caseInstanceRepository;
@Inject ClaudonyWorkerExecutionManager execManager;
@Inject Event<CaseLifecycleEvent> lifecycleEvents;

// Start case
UUID caseId = researcherCase.startCase(Map.of("topic", "test-topic"))
    .toCompletableFuture().get(10, TimeUnit.SECONDS);

// Wait for provision (createWorkerSession called by ClaudonyReactiveWorkerProvisioner)
Awaitility.await().atMost(Duration.ofSeconds(10))
    .untilAsserted(() -> verify(tmuxService, atLeastOnce())
        .createWorkerSession(anyString(), anyString(), anyString()));

// WorkOrchestrator excluded — start watcher manually (same as CaseEngineRoundTripTest)
var session = sessionRegistry.findByCaseId(caseId.toString()).get(0);
var instance = caseInstanceRepository.findByUuid(caseId).await().atMost(Duration.ofSeconds(5));
var cap = new Capability("researcher", "{}", "{}");
var worker = new Worker("researcher", List.of(cap), ctx -> Map.of());

// Fire WorkerExecutionStarted so lineage resolves correctly
lifecycleEvents.fireAsync(new CaseLifecycleEvent(
    caseId, null, "ExecuteWorker", "WorkerExecutionStarted", "ACTIVE",
    "researcher", "WORKER", null)).toCompletableFuture().get(5, TimeUnit.SECONDS);

// Trigger watcher exit detection
when(tmuxService.sessionExists(anyString())).thenReturn(false);
execManager.watch(session.id(), session.name(), instance, worker);

// Assert case reaches COMPLETED (CaseInstance.getState() returns CaseStatus)
Awaitility.await()
    .atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(200))
    .untilAsserted(() -> {
        CaseInstance updated = caseInstanceRepository.findByUuid(caseId)
            .await().atMost(Duration.ofSeconds(5));
        assertThat(updated.getState()).isEqualTo(CaseStatus.COMPLETED);
    });
```

`NoOpWorkerExecutionManager` is `@DefaultBean @ApplicationScoped` and yields automatically to `ClaudonyWorkerExecutionManager` — no conflict, no exclusion needed.

**`ClaudonyWorkerExecutionManagerTest` additions:**

- `workerExit_storesPendingExitSignal` — trigger the exit path via mock `TmuxService`, assert `drainExitSignal(caseId)` returns the role name, second call returns null
- `drainExitSignal_unknownCaseId_returnsNull` — sanity guard

**`ClaudonyLedgerEventCaptureTest` additions:**

Mock `ClaudonyWorkerExecutionManager` is already injected via `@InjectMock` (same as existing `provisioner`). Corrected seeding — use Mockito stub, not direct field access (`pendingExitSignals` is private):

- `workerExecutionCompleted_withPendingSignal_firesContextSignal`: `when(execManager.drainExitSignal(caseId)).thenReturn("researcher")`; fire `CaseLifecycleEvent("WorkerExecutionCompleted")`; assert `caseHubRuntime.signal(caseId, "workers.researcher.exited", true)` called (mock runtime injected via `@InjectMock`)
- `workerExecutionCompleted_noPendingSignal_doesNotFireSignal`: drain returns null; assert signal not called

---

## #143 — tenancyId test additions

`ClaudonyLedgerEventCapture` already handles `tenancyId` correctly. `CrossTenantCaseInstanceRepository.findByUuid(UUID)` is single-arg (confirmed from decompiled SNAPSHOT class). No production code changes.

Two additions to `ClaudonyLedgerEventCaptureTest`:

1. **In `happyPath_singleEvent_writesLedgerEntry`:** add `assertThat(entry.tenancyId).isEqualTo("default")`. `CaseLedgerEntry.tenancyId` is a public field — direct access matches `ClaudonyLedgerEventCapture`'s own `entry.tenancyId = ...` assignment style.

2. **New test `tenancyId_nonNull_storedAsIs`:** fire with second constructor arg `"tenant-1"`, assert `entry.tenancyId.equals("tenant-1")`.

Closes #143.

---

## #147 — `bootstrapCasehubWatchers` test coverage

### Extraction to `CasehubStartupService`

Package: `io.casehub.claudony.server`. Plain Java, no CDI annotations — tests instantiate directly.

```java
class CasehubStartupService {
    private final SessionRegistry registry;
    private final CrossTenantCaseInstanceRepository caseInstanceRepo;
    private final ClaudonyWorkerExecutionManager execManager;

    CasehubStartupService(SessionRegistry registry,
                          CrossTenantCaseInstanceRepository caseInstanceRepo,
                          ClaudonyWorkerExecutionManager execManager) { ... }

    int bootstrapWatchers() { /* moved loop from ServerStartup */ }
}
```

`ServerStartup.bootstrapCasehubWatchers()` becomes a thin wrapper that constructs `CasehubStartupService` after the `Instance<>` guard.

### Tests — correct reactive mock types

`CasehubStartupServiceTest` — 3 plain JUnit/Mockito unit tests. `CrossTenantCaseInstanceRepository.findByUuid(UUID)` returns `Uni<CaseInstance>` — mocks must return `Uni`:

1. **`invalidCaseId_logsWarnAndSkips`** — session with `caseId = Optional.of("not-a-uuid")`; assert returns 0, `execManager.watch()` never called.

2. **`nullCaseInstance_logsInfoAndSkips`** — valid UUID; `when(caseInstanceRepo.findByUuid(any())).thenReturn(Uni.createFrom().item((CaseInstance) null))`; assert watcher not started, returns 0.

3. **`absentRoleName_fallsBackToWorker`** — no `roleName` on session; `when(caseInstanceRepo.findByUuid(any())).thenReturn(Uni.createFrom().item(mockCaseInstance))`; assert `execManager.watch()` called with `Worker` whose `getName()` returns `"worker"`.

### `WorkerExitRecoveryIntegrationTest` — unaffected

No constructor change to `ClaudonyWorkerExecutionManager`. `@Inject` in default profile works as before. `drainExitSignal()` returns null (empty map) in tests where no exit was detected — `caseHubRuntime.isUnsatisfied()` guard in `ClaudonyLedgerEventCapture` ensures no signal call anyway.

Closes #147.

---

## Platform coherence

- **Right repo:** all changes in Claudony (`claudony-casehub`, `claudony-app`). No cross-repo concern.
- **Pattern compliance:** `ResearcherCase extends YamlCaseHub` follows platform protocol.
- **Signal convention:** `workers.<roleName>.exited` uses dot-notation paths confirmed by `CaseContextImpl.applyAndDiff()` source. Limitation acknowledged.
- **No Flyway:** no schema changes.
- **`CrossTenantCaseInstanceRepository.findByUuid(UUID)`:** single-arg confirmed — existing code clean.