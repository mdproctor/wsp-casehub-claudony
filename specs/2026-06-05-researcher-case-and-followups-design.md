# Design: ResearcherCase, Exit Signalling, and Branch Follow-ups

**Branch:** `issue-148-researcher-case`  
**Covers:** #148 (ResearcherCase + exit signalling), #143 (tenancyId tests), #147 (bootstrapCasehubWatchers tests)  
**Date:** 2026-06-06 (revised after code review)

---

## Problem

Claudony has no production case definition. `TestResearcherCase` is test-only and cannot be activated in a running server. There is also no mechanism for a case to auto-complete when the researcher worker (a Claude Code tmux session) exits — cases would stay RUNNING indefinitely.

---

## Architecture: Exit Signalling via `CaseHubRuntime.signal()`

`CaseHubRuntime.signal(caseId, path, value)` is a **direct context patch** using dot-notation paths. `CaseContextImpl.applyAndDiff(path, value)` splits `path` on `.` and creates nested maps. After patching, the engine fires `CONTEXT_CHANGED`, which evaluates goals and completion criteria.

`signal(caseId, "workers.researcher.exited", true)` produces:

```json
{ "workers": { "researcher": { "exited": true } } }
```

…satisfying `.workers.researcher.exited == true` and triggering case completion — with zero changes to Claude's workflow. Multiple workers compose cleanly: each role writes under `workers.<roleName>.exited` without conflict.

### Signal ordering: `drainExitSignal` pattern

`CaseHubRuntime.signal()` calls `CaseHubReactor.signal()` → `eventBus.publish(SIGNAL_RECEIVED, ...)` — it is **not** synchronous; it dispatches to the Vert.x event bus and returns. Placing the signal call in the exit watcher thread immediately after `eventBus.send(WORKER_EXECUTION_FINISHED, ...)` creates a race: `SIGNAL_RECEIVED` may be processed by `SignalReceivedEventHandler` before `WorkflowExecutionCompletedHandler` has updated engine state.

**Fix: `drainExitSignal` — same pattern as `drainCausalContext`.**

`ClaudonyWorkerExecutionManager` stores `pendingExitSignals: ConcurrentHashMap<UUID, String>` (caseId → roleName). When the watcher wins `registry.remove()`:
1. `pendingExitSignals.put(instance.getUuid(), worker.getName())` — stored before the event bus send
2. `eventBus.send(WORKER_EXECUTION_FINISHED, new WorkflowExecutionCompleted(...))` — existing

`ClaudonyLedgerEventCapture.onCaseLifecycleEvent()` drains the signal when it observes `WorkerExecutionCompleted` — which fires **inside the reactive chain of `WorkflowExecutionCompletedHandler`**, after the handler has applied worker output, appended the event log, and notified `WorkerStatusListener`. This guarantees the signal fires after engine state is fully updated. The signal then fires its own `CONTEXT_CHANGED`, which evaluates the goal condition and completes the case.

`ClaudonyWorkerExecutionManager` exposes:
```java
public String drainExitSignal(UUID caseId) {
    return pendingExitSignals.remove(caseId);
}
```

**No constructor change** to `ClaudonyWorkerExecutionManager`. The `pendingExitSignals` map and `drainExitSignal()` are added internally. Existing 11 tests in `ClaudonyWorkerExecutionManagerTest` are unaffected.

### Signal convention — known limitation

The `workers.<roleName>.exited` convention is baked into `ClaudonyLedgerEventCapture`. Case definitions that want different exit semantics (or no auto-completion signal) cannot opt out. Future work: add an `onExitSignal` field to `Capability` in the case definition, allowing case authors to configure or suppress the signal path.

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

Extends `YamlCaseHub` (platform protocol for production case definitions; YAML loaded lazily via `CaseDefinitionYamlMapper.load()`). When CaseHub is disabled (`CaseHubRuntime` absent), the bean exists in CDI but `startCase()` is never called — no breakage.

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

`inputSchema: "{}"` and `outputSchema: "{}"` match `TestResearcherCase`. `inputSchema` is intentionally minimal — Claudony workers receive context via the system prompt from `ClaudonyWorkerContextProvider`, not via the engine's structured input mapping. The populated inputSchema in YAML examples (`{ topic: .topic }`) would map data into the worker's input payload, which Claudony's provisioner ignores. Using `"{}"` avoids confusion and aligns test and production definitions.

### Changes to `ClaudonyWorkerExecutionManager`

Add `private final ConcurrentHashMap<UUID, String> pendingExitSignals = new ConcurrentHashMap<>()`.

In `watcherRunnable`, after `registry.remove(sessionId) != null` wins:

```java
pendingExitSignals.put(instance.getUuid(), worker.getName());  // before send — drain on lifecycle event
eventBus.send(EventBusAddresses.WORKER_EXECUTION_FINISHED,
        new WorkflowExecutionCompleted(instance, worker, idempotencyKey, Map.of()));
```

In `shutdown()`: `pendingExitSignals.clear()` (after `watchers.clear()`, before `sessionToRole.clear()`).

Expose:
```java
public String drainExitSignal(UUID caseId) {
    return pendingExitSignals.remove(caseId);
}
```

### Changes to `ClaudonyLedgerEventCapture`

Inject `ClaudonyWorkerExecutionManager execManager` (alongside existing `ClaudonyReactiveWorkerProvisioner provisioner`) and `Instance<CaseHubRuntime> caseHubRuntime`.

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

### Relationship to `WorkflowExecutionCompleted` + signal ordering — analysis

`WorkflowExecutionCompletedHandler` fires `CaseLifecycleEvent("WorkerExecutionCompleted")` with `actorId = "system"`, `actorRole = "SYSTEM"` — worker name is absent. After the lifecycle event fires, the handler publishes `CONTEXT_CHANGED`. The CDI observer (`ClaudonyLedgerEventCapture`) runs asynchronously; its `signal()` call dispatches `SIGNAL_RECEIVED` to the event bus, which then fires a second `CONTEXT_CHANGED`. The sequence for goal evaluation:

1. First `CONTEXT_CHANGED` (from handler): `.workers.researcher.exited` not yet set → goal not met → case stays RUNNING
2. `SIGNAL_RECEIVED` processed: `applyAndDiff("workers.researcher.exited", true)` → second `CONTEXT_CHANGED`
3. Second `CONTEXT_CHANGED`: goal `.workers.researcher.exited == true` → met → case COMPLETED

`WorkflowExecutionCompleted` was processed at step 0; by the time the case completes at step 3, the engine has already updated all worker state. No stale-event problem.

### CasehubEnabledProfile — exclude `ResearcherCase`

`ResearcherCase` is `@ApplicationScoped` and will be discovered by the engine at startup. `CaseEngineRoundTripTest` uses `TestResearcherCase` explicitly. Both beans would be active in `CasehubEnabledProfile`, creating two active case definitions. The engine does not name-collide (different `namespace:name`) but the test's `.topic != null` trigger would also fire on `ResearcherCase` if started.

Fix: add `io.casehub.claudony.casehub.ResearcherCase` to `CasehubEnabledProfile.quarkus.arc.exclude-types` in `CaseEngineRoundTripTest`. The production YAML (`researcher.yaml`) IS on the test classpath — `claudony-casehub` is a compile dependency of `claudony-app`, so its `src/main/resources/` is visible in tests.

### New tests

**`ResearcherCaseStartupTest`** — default profile (no engine required, just classpath + YAML parsing):

```java
@Inject ResearcherCase researcherCase;

@Test void yamlLoads_withExpectedMetadata() {
    var def = researcherCase.getDefinition();
    assertThat(def).isNotNull();
    assertThat(def.name()).isEqualTo("researcher");
    assertThat(def.namespace()).isEqualTo("io.casehub.claudony");
    assertThat(def.getCapabilities()).anyMatch(c -> "researcher".equals(c.getName()));
    assertThat(def.getBindings()).anyMatch(b ->
        b.getTrigger() != null && b.getTrigger().getContextChange() != null
        && ".topic != null".equals(b.getTrigger().getContextChange().getFilter()));
}
```

**`ResearcherCaseCompletionTest`** — new `ResearcherCaseCasehubProfile` (enables CaseHub, includes `ResearcherCase`, excludes `TestResearcherCase`):

- Starts a case with `Map.of("topic", "test-topic")`
- Mocks `TmuxService` to make `sessionExists()` return false immediately (simulating exit)
- Awaits that `caseInstanceRepository.findByUuid(caseId).getState() == COMPLETED`
- This is the definitive E2E proof that the signal mechanism works

**`ClaudonyWorkerExecutionManagerTest` additions:**

- `workerExit_storesPendingExitSignal` — after watcher detects tmux exit, assert `drainExitSignal(caseId)` returns the role name and a second call returns null (drained)
- `drainExitSignal_unknownCaseId_returnsNull` — sanity test for the drain

**`ClaudonyLedgerEventCaptureTest` additions:**

- `workerExecutionCompleted_withPendingSignal_firesContextSignal` — seed `execManager.pendingExitSignals.put(caseId, "researcher")` (or seed via a test method), fire `CaseLifecycleEvent("WorkerExecutionCompleted")`, assert `caseHubRuntime.signal(caseId, "workers.researcher.exited", true)` was called (mock runtime)
- `workerExecutionCompleted_noPendingSignal_doesNotFireSignal` — drain is empty, signal not called

---

## #143 — tenancyId test additions

`ClaudonyLedgerEventCapture` already reads `event.tenancyId()` and stores `"default"` when null. Installed engine SNAPSHOT has `tenancyId` as second record component. `CrossTenantCaseInstanceRepository.findByUuid(UUID)` is single-arg — confirmed from decompiled SNAPSHOT class; existing code is not stale.

Two additions to `ClaudonyLedgerEventCaptureTest`:

1. **In `happyPath_singleEvent_writesLedgerEntry`:** add `assertThat(entry.tenancyId).isEqualTo("default")`. `CaseLedgerEntry.tenancyId` is a public field — direct field access, matching `ClaudonyLedgerEventCapture`'s own `entry.tenancyId = ...` assignment.

2. **New test `tenancyId_nonNull_storedAsIs`:** fire with second constructor arg `"tenant-1"`, assert `entry.tenancyId.equals("tenant-1")`.

No production code changes. Closes #143.

---

## #147 — `bootstrapCasehubWatchers` test coverage

### `CrossTenantCaseInstanceRepository.findByUuid` — confirmed single-arg

`CrossTenantCaseInstanceRepository.findByUuid(UUID): Uni<CaseInstance>` — single parameter. The `CaseInstanceRepository.findByUuid(UUID, String)` two-arg variant is a different interface. Existing `bootstrapCasehubWatchers()` code is correct.

### Extraction

Extract loop body from `ServerStartup.bootstrapCasehubWatchers()` into `CasehubStartupService` in package `io.casehub.claudony.server` (alongside `ServerStartup`):

```java
class CasehubStartupService {
    private final SessionRegistry registry;
    private final CrossTenantCaseInstanceRepository caseInstanceRepo;
    private final ClaudonyWorkerExecutionManager execManager;

    CasehubStartupService(
            SessionRegistry registry,
            CrossTenantCaseInstanceRepository caseInstanceRepo,
            ClaudonyWorkerExecutionManager execManager) { ... }

    int bootstrapWatchers() { /* moved loop from ServerStartup */ }
}
```

Plain Java, no CDI annotations — tests instantiate directly without Quarkus context.

`ServerStartup.bootstrapCasehubWatchers()` becomes:

```java
private void bootstrapCasehubWatchers() {
    if (caseInstanceRepo.isUnsatisfied() || workerExecManager.isUnsatisfied()) {
        LOG.debug("...");
        return;
    }
    int started = new CasehubStartupService(
            registry, caseInstanceRepo.get(), workerExecManager.get())
        .bootstrapWatchers();
    LOG.infof("Started %d casehub watcher(s) for recovered sessions", started);
}
```

### Tests — reactive mock types

`CasehubStartupServiceTest` — 3 plain JUnit/Mockito unit tests. `CrossTenantCaseInstanceRepository.findByUuid()` returns `Uni<CaseInstance>` — mocks must return `Uni`, not bare values:

1. **`invalidCaseId_logsWarnAndSkips`** — session with `caseId = Optional.of("not-a-uuid")`; mock `findByUuid` never called; assert `bootstrapWatchers()` returns 0, `execManager.watch()` never called.

2. **`nullCaseInstance_logsInfoAndSkips`** — valid UUID caseId; `when(caseInstanceRepo.findByUuid(any())).thenReturn(Uni.createFrom().item((CaseInstance) null))`; assert watcher not started, returns 0.

3. **`absentRoleName_fallsBackToWorker`** — session with valid UUID and no `roleName`; `when(caseInstanceRepo.findByUuid(any())).thenReturn(Uni.createFrom().item(mockCaseInstance))`; assert `execManager.watch()` called with `Worker` whose `getName()` returns `"worker"`.

### `WorkerExitRecoveryIntegrationTest` — unaffected

`ClaudonyWorkerExecutionManager` constructor does not change. `@Inject ClaudonyWorkerExecutionManager` in the default profile continues to work. `drainExitSignal()` in the default profile (no `CaseHubRuntime`) returns null (empty map) — the `isUnsatisfied()` guard in `ClaudonyLedgerEventCapture` ensures no signal call. Confirmed safe.

---

## Platform coherence

- **Right repo:** all changes in Claudony (`claudony-casehub`, `claudony-app`). No cross-repo concern.
- **Pattern compliance:** `ResearcherCase extends YamlCaseHub` follows platform protocol.
- **Signal convention:** `workers.<roleName>.exited` uses dot-notation nested paths supported by `CaseContextImpl.applyAndDiff()` (confirmed from source). Limitation documented — no per-capability opt-out today.
- **No Flyway:** no schema changes.
- **`CrossTenantCaseInstanceRepository.findByUuid(UUID)`:** single-arg — confirmed clean.
