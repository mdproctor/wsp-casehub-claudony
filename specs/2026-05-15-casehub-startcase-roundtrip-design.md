# Design: Full CaseHub.startCase() Round-Trip Test

**Date:** 2026-05-15  
**Issue:** #92 — Full CaseEngine round-trip E2E  
**Epic:** #86 — Agent mesh infrastructure  
**Status:** Approved

---

## Problem

`CaseEngineRoundTripTest` currently fires CDI events directly, bypassing `CaseHub.startCase()`.
The original intent of #92 was to exercise the full engine chain: `startCase()` → engine evaluates
bindings → `ClaudonyWorkerProvisioner.provision()` → completion driven externally →
`JpaCaseLineageQuery.findCompletedWorkers()` returns a populated `WorkerSummary`.

The previous attempt was abandoned due to CDI ambiguity: `casehub-engine`'s no-op SPI beans
(`NoOpWorkerProvisioner` etc.) were `@ApplicationScoped`, colliding with Claudony's own
implementations when `casehub-testing` was indexed. That upstream issue is now fixed — the
no-ops are `@DefaultBean` (engine #257, protocol `PP-20260514-engine-spi-noops-defaultbean`).

---

## Design

### Changes

**1. `claudony-app/pom.xml`**  
Add `casehub-testing` as a `test` scoped dependency. Provides `TestCaseInstanceRepository`,
`TestCaseMetaModelRepository`, and `TestEventLogRepository` — `@DefaultBean` in-memory
implementations that replace JPA for the engine's own instance and event-log stores in
`@QuarkusTest` without requiring a database schema for engine tables.

**2. `app/src/test/resources/application.properties`**  
Add indexing for `casehub-testing` so Quarkus discovers the in-memory stores:
```properties
quarkus.index-dependency.casehub-testing.group-id=io.casehub
quarkus.index-dependency.casehub-testing.artifact-id=casehub-testing
```

**3. `TestResearcherCase.java`** (new top-level class in test sources)  
`@ApplicationScoped` subclass of `CaseHub`. Defines a minimal case: one "researcher"
capability, one `ContextChangeTrigger(".topic != null")` binding, no static workers.
The absence of static workers forces the engine's `tryProvision()` fallback path in
`CaseContextChangedEventHandler`, which calls `workerProvisioner.getCapabilities()` and
then `provision()` when the capability matches.

Must be top-level (not an inner class) so CDI scanning picks it up correctly within the
`@TestProfile` context.

**4. `CaseEngineRoundTripTest.java`** (replaces current CDI-event version)  
The existing test fires `CaseLifecycleEvent` directly — superseded by the engine round-trip.
Replace entirely.

---

### Test Structure

```
@QuarkusTest
@TestProfile(CaseEngineRoundTripTest.CasehubEnabledProfile.class)
class CaseEngineRoundTripTest
```

**Profile config overrides:**
- `claudony.casehub.enabled=true` — activates `ClaudonyWorkerProvisioner` behaviour
- `claudony.casehub.workers.commands.researcher=claude` — provisioner advertises "researcher"
- `claudony.casehub.workers.commands.default=claude`

**No `@TestSecurity`** — CDI-only test, no HTTP endpoints exercised (PP-20260513-7c227e).

**Mocks:**
- `@InjectMock TmuxService` — `createSession()` stubbed to no-op so no real tmux session
  is created. `ClaudonyWorkerProvisioner.provision()` calls through but returns immediately.

**Injections:**
- `TestResearcherCase` — the `CaseHub` subclass to call `startCase()` on
- `JpaCaseLineageQuery` — the lineage query under test
- `CaseInstanceRepository` — to retrieve the `CaseInstance` for completion driving
- `EventBus` — to publish the completion event directly

---

### Test Flow

```
1. startCase(Map.of("topic", "test")).toCompletableFuture().get(15s)
   → engine persists CaseInstance (in-memory)
   → CaseStartedEventHandler publishes CONTEXT_CHANGED
   → CaseContextChangedEventHandler evaluates ContextChangeTrigger(".topic != null") → fires
   → tryProvision() checks workerProvisioner.getCapabilities() → contains "researcher"
   → ClaudonyWorkerProvisioner.provision() called → tmuxService.createSession() (mocked, no-op)
   → Worker("researcher") returned

2. Awaitility.await(20s): verify(tmuxService, atLeastOnce()).createSession(...)
   → confirms provision() was reached

3. caseInstanceRepository.findByUuid(caseId).await().atMost(5s)
   → retrieve CaseInstance for the completion event

4. eventBus.publish(WORKER_EXECUTION_FINISHED,
       new WorkflowExecutionCompleted(instance, worker, idempotency, output))
   where worker = new Worker("researcher", List.of(cap), ctx -> Map.of())
   (Worker has three constructors: lambda, Serverless Workflow, File — lambda is fine here)
   → WorkflowExecutionCompletedHandler fires CaseLifecycleEvent("WorkerExecutionCompleted", ...)
   → ClaudonyLedgerEventCapture (@ObservesAsync) writes CaseLedgerEntry row

5. Awaitility.await(20s): lineageQuery.findCompletedWorkers(caseId).hasSize(1)
   → confirms full pipeline: engine → provisioner → ledger capture → lineage query

6. Assert WorkerSummary fields:
   - workerName == "researcher"
   - completedAt != null
   - ledgerEntryId != null
```

**Why publish directly vs `WorkResultSubmitter`?**  
`WorkResultSubmitter` looks up workers from `definition.getWorkers()`. `TestResearcherCase`
has no static workers — the provisioner creates them dynamically via `tryProvision()`. Direct
publish to `WORKER_EXECUTION_FINISHED` is the correct boundary for driving completion in this
path.

**`startedAt` behaviour:** The `tryProvision()` path does not fire `WorkerExecutionStarted`,
so `findCompletedWorkers()` falls back to `startedAt = completedAt`. This is correct and is
covered by `CaseLineageQueryIntegrationTest.startedAtFallsBackToCompletedAtWhenNoStartEntry`.

---

### What This Replaces

The current `CaseEngineRoundTripTest` fires CDI events directly and is entirely superseded.
Its comment acknowledging the bypass is resolved. Delete it; the new test is the replacement.

---

### Test Count Impact

Net zero: one test class replaced by one test class. Existing count: 479. New count: 479
(assuming the new test has the same number of `@Test` methods — one round-trip test).

---

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| `casehub-testing` vs mock `WorkerProvisioner` | `casehub-testing` | Correct approach now that `@DefaultBean` fix is in |
| `WorkResultSubmitter` vs direct event bus publish | Direct publish | `WorkResultSubmitter` requires static workers in definition |
| Static workers in `TestResearcherCase` | None | Forces `tryProvision()` path — the real production path for externally provisioned workers |
| `@TestSecurity` | Omitted | CDI-only test, PP-20260513-7c227e |
| `TestResearcherCase` placement | Top-level `@ApplicationScoped` in test sources | Inner class not picked up by CDI scanner in `@TestProfile` context |
