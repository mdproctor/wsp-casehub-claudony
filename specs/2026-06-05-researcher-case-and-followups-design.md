# Design: ResearcherCase, Exit Signalling, and Branch Follow-ups

**Branch:** `issue-148-researcher-case`  
**Covers:** #148 (ResearcherCase), #143 (tenancyId tests), #147 (bootstrapCasehubWatchers tests)  
**Date:** 2026-06-05

---

## Problem

Claudony has no production case definition. `TestResearcherCase` is test-only and cannot be activated in a running server. There is also no mechanism for a case to auto-complete when the researcher worker (a Claude Code tmux session) exits — cases would stay RUNNING indefinitely.

---

## Architecture: Exit Signalling via `CaseHubRuntime.signal()`

`CaseHubRuntime.signal(caseId, path, value)` is a **direct context patch** using dot-notation paths. It calls `CaseContext.applyAndDiff(path, value)`, which splits `path` on `.` and creates nested maps. After patching, the engine fires `CONTEXT_CHANGED`, which evaluates goals and completion criteria.

This means `signal(caseId, "workers.researcher.exited", true)` produces:

```json
{ "workers": { "researcher": { "exited": true } } }
```

…which satisfies the JQ goal condition `.workers.researcher.exited == true` and triggers case completion — with zero changes to Claude's workflow.

Multiple workers compose cleanly: each role writes under its own sub-key (`workers.reviewer.exited`, etc.) without conflict.

---

## #148 — ResearcherCase and Exit Signalling

### New classes and resources

**`claudony-casehub/src/main/java/io/casehub/claudony/casehub/ResearcherCase.java`**

```java
@ApplicationScoped
public class ResearcherCase extends YamlCaseHub {
    public ResearcherCase() { super("casehub/researcher.yaml"); }
}
```

Extends `YamlCaseHub` (platform convention for production case definitions). YAML is loaded lazily on first `getDefinition()` call. When CaseHub is disabled, the CDI bean exists but `CaseHubRuntime` is absent, so `startCase()` is never called.

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
      inputSchema: "{ topic: .topic }"
      outputSchema: "{ }"
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

Provision trigger: `.topic != null` in context. Completion: `.workers.researcher.exited == true` (set by exit watcher, see below).

### Exit signalling in `ClaudonyWorkerExecutionManager`

Add `Instance<CaseHubRuntime> caseHubRuntime` to the constructor (injected by CDI). `casehub-engine-api` is already a dependency of `claudony-casehub`.

In `watcherRunnable`, after the atomic `registry.remove()` gate wins and before `break`:

```java
eventBus.send(EventBusAddresses.WORKER_EXECUTION_FINISHED,
        new WorkflowExecutionCompleted(instance, worker, idempotencyKey, Map.of()));

if (!caseHubRuntime.isUnsatisfied()) {
    caseHubRuntime.get().signal(
        instance.getUuid(),
        "workers." + worker.getName() + ".exited",
        true);
}
```

The `isUnsatisfied()` guard means the signal is a no-op when the engine is not on the classpath. The signal is fire-and-forget (event bus publish), consistent with the existing `WorkflowExecutionCompleted` pattern. Order matters: publish completion first so the engine records the worker as done before context changes trigger goal evaluation.

### `TestResearcherCase` — no change

`TestResearcherCase` uses the builder DSL (per protocol: tests build `CaseDefinition` directly, production extends `YamlCaseHub`). It stays as-is, independent of `ResearcherCase`.

### New integration test

`ResearcherCaseStartupTest` (in `claudony-app/src/test/java/`, `CasehubEnabledProfile`):

- Assert `ResearcherCase` is registered with the engine at startup (check `CaseDefinitionRegistry` or use `CaseHubRuntime` to look up the definition)
- Assert YAML loads without exception (`researcherCase.getDefinition()` returns non-null)

### Exit signal test

`ClaudonyWorkerExecutionManagerTest` — add:
- `workerExit_signalsCaseContext` — mock `CaseHubRuntime`, confirm `signal("workers.researcher.exited", true)` called when watcher detects tmux exit
- `workerExit_noSignal_whenRuntimeUnsatisfied` — guard path: signal not called when `caseHubRuntime.isUnsatisfied()`

---

## #143 — tenancyId test additions

Production code (`ClaudonyLedgerEventCapture`) is already correct — it reads `event.tenancyId()` and stores `"default"` when null. The installed engine SNAPSHOT already has `tenancyId` as the second record component.

Two additions to `ClaudonyLedgerEventCaptureTest`:

1. **In `happyPath_singleEvent_writesLedgerEntry`:** add `assertThat(entry.tenancyId).isEqualTo("default")` (fires with `tenancyId=null`, expect stored as `"default"`).

2. **New test `tenancyId_nonNull_storedAsIs`:** fire with `tenancyId="tenant-1"` (second constructor arg), assert `entry.tenancyId == "tenant-1"`.

No production code changes. Closes #143.

---

## #147 — `bootstrapCasehubWatchers` test coverage

### Extraction

Extract the loop body from `ServerStartup.bootstrapCasehubWatchers()` into a new class `CasehubStartupService` (plain Java, constructor injection — no CDI annotations, no `@QuarkusTest` needed):

```java
class CasehubStartupService {
    CasehubStartupService(
        SessionRegistry registry,
        CrossTenantCaseInstanceRepository caseInstanceRepo,
        ClaudonyWorkerExecutionManager execManager) { ... }

    int bootstrapWatchers() { /* moved loop */ }
}
```

`ServerStartup.bootstrapCasehubWatchers()` becomes:

```java
private void bootstrapCasehubWatchers() {
    if (caseInstanceRepo.isUnsatisfied() || workerExecManager.isUnsatisfied()) { ... return; }
    var started = new CasehubStartupService(
            registry, caseInstanceRepo.get(), workerExecManager.get())
        .bootstrapWatchers();
    LOG.infof("Started %d casehub watcher(s) for recovered sessions", started);
}
```

### Tests

`CasehubStartupServiceTest` — 3 plain JUnit unit tests (no Quarkus):

1. **`invalidCaseId_logsWarnAndSkips`** — put a session with `caseId = "not-a-uuid"` in a mock registry; assert `bootstrapWatchers()` returns 0 and `execManager.watch()` is never called.

2. **`nullCaseInstance_logsInfoAndSkips`** — mock `caseInstanceRepo.findByUuid()` returns `null`; assert watcher not started, returns 0.

3. **`absentRoleName_fallsBackToWorker`** — session has no `roleName`; mock repo returns a real `CaseInstance`; assert `execManager.watch()` called with a `Worker` whose name is `"worker"`.

Closes #147.

---

## Platform coherence

- **Right repo:** all changes are in Claudony (`claudony-casehub`, `claudony-app`). No cross-repo concern.
- **Pattern compliance:** `ResearcherCase extends YamlCaseHub` matches the platform protocol for production case definitions.
- **Signal convention:** `workers.<roleName>.exited` is a documented Claudony-specific context key convention. Multi-role cases compose cleanly via nested paths.
- **No Flyway:** no schema changes.
- **No breaking changes** to existing SPIs or tests.
