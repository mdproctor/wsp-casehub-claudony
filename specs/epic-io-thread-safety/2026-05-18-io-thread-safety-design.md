# IO Thread Safety — Reactive SPI Migration

**Issue:** casehubio/claudony#115  
**Epic:** `epic-io-thread-safety`  
**Date:** 2026-05-18  
**Repos affected:** `casehub-engine`, `claudony`

---

## Problem

`CaseContextChangedEventHandler.tryProvision()` runs on the Vert.x IO thread via
`@ConsumeEvent`. It calls `WorkerContextProvider.buildContext()`, which in Claudony's
implementation (`ClaudonyWorkerContextProvider`) does:

1. `CaseLineageQuery.findCompletedWorkers()` — blocking JPA query
2. `CaseChannelProvider.listChannels()` — blocking Qhorus call

Both throw `BlockingOperationNotAllowedException` at runtime. The issue was hidden by
`CaseEngineRoundTripTest` mocking `ClaudonyWorkerContextProvider` entirely.

The same applies to `WorkerProvisioner.provision()` — a blocking tmux `ProcessBuilder`
call on the IO thread.

---

## Decision

**Approach A — Full reactive pipeline.** The engine already ships
`ReactiveWorkerContextProvider`, `ReactiveWorkerProvisioner`, and
`ReactiveCaseChannelProvider` SPIs with `@DefaultBean` no-ops. Claudony implements
the reactive variants; the engine's `tryProvision()` becomes a `Uni<Void>` integrated
into its existing reactive chain. Blocking SPI implementations are deleted.

Parallel stacks (blocking + reactive) are not kept in Claudony. The blocking SPIs are
internal to Claudony and have exactly one consumer each — the context provider — which
is being replaced. Qhorus's dual-stack (ADR-0003) is correct for a foundation layer
serving diverse consumers; it does not apply here.

---

## Section 1 — Engine: `tryProvision()` becomes reactive

**File:** `casehub-engine/runtime/.../handler/CaseContextChangedEventHandler.java`

- Drop `@Inject WorkerContextProvider` and `@Inject WorkerProvisioner`
- Add `@Inject ReactiveWorkerContextProvider` and `@Inject ReactiveWorkerProvisioner`
- `tryProvision()` returns `Uni<Void>`:

```java
private Uni<Void> tryProvision(CaseInstance caseInstance, Capability capability) {
    return reactiveWorkerProvisioner.getCapabilities()
        .flatMap(caps -> {
            if (!caps.contains(capability.getName())) return Uni.createFrom().voidItem();
            Map<String, Object> inputData = caseInstance.getCaseContext()
                .evalObjectTemplate(capability.getInputSchema());
            WorkRequest workRequest = WorkRequest.of(capability.getName(), inputData);
            return reactiveWorkerContextProvider
                .buildContext(null, caseInstance.getUuid(), workRequest)
                .flatMap(ctx -> {
                    ProvisionContext provisionContext = new ProvisionContext(
                        caseInstance.getUuid(), capability.getName(), ctx,
                        PropagationContext.createRoot(), null, null);
                    return reactiveWorkerProvisioner.provision(caps, provisionContext)
                        .replaceWithVoid();
                });
        })
        .onFailure(ProvisioningException.class).invoke(e ->
            LOG.warnf(e, "WorkerProvisioner failed for capability '%s' on case %s",
                capability.getName(), caseInstance.getUuid()));
}
```

- Both `publishWorkerSchedule()` call sites that previously fire-and-forgot now
  `return tryProvision(...)` — provisioning errors propagate into the handler's
  `onFailure()` logging chain.

---

## Section 2 — `CaseLineageQuery` goes reactive

**Files:** `claudony/casehub/src/main/java/io/casehub/claudony/casehub/`

Interface change:
```java
// Before
List<WorkerSummary> findCompletedWorkers(UUID caseId);

// After
Uni<List<WorkerSummary>> findCompletedWorkers(UUID caseId);
```

Implementations:
- `EmptyCaseLineageQuery` → `return Uni.createFrom().item(List.of())`
- `JpaCaseLineageQuery` → switches from blocking Panache to reactive Panache
  (`CaseLedgerEntry.find(...).list()` returning `Uni<List<...>>`), same
  `@LedgerPersistenceUnit` persistence unit

`JpaCaseLineageQueryTest` updated to use `UniAsserter` or `.await().indefinitely()`.

---

## Section 3 — `ClaudonyReactiveCaseChannelProvider`

**Replaces:** `ClaudonyCaseChannelProvider` (deleted)

**Implements:** `ReactiveCaseChannelProvider` from `casehub-qhorus-api`

**Single implementation** — no build-time split. `quarkus.datasource.qhorus.reactive=true`
is set in Claudony's `application.properties`, activating the reactive Qhorus stack.
H2's JDBC limitation does not drive the architecture; H2 is a test database only.

Injects `ReactiveChannelService` and `ReactiveMessageService` directly from
`casehub-qhorus/runtime` (already live, `@IfBuildProperty` activated by the above
property). Does not go through `QhorusMcpTools` — that is the MCP dispatch layer, not
a service API.

**Channel cache:** The existing `ConcurrentHashMap<UUID, Map<String, CaseChannel>>`
is preserved in a shared base class. Cache hits in `openChannel()` return
`Uni.createFrom().item(...)` directly — no service call needed.

**Operations:**
- `openChannel()` — cache miss calls `ReactiveChannelService.create()`; cache hit
  returns immediately
- `listChannels()` — calls `ReactiveChannelService.listAll()` filtered by
  `"case-" + caseId` prefix in Java; once qhorus#161 ships, replace with
  `findByNamePrefix("case-" + caseId)` for a single indexed query
- `postToChannel()` — calls `ReactiveMessageService.send()`
- `closeChannel()` — `Uni.createFrom().voidItem()` (Qhorus channels are persistent)

**Tests:** `quarkus-reactive-h2-client` added to test classpath to enable reactive
Qhorus stack against H2. If incompatible with Hibernate Reactive Panache, affected
tests move to Docker/PostgreSQL — an acceptable trade for a clean architecture.

**Deferred:**
- qhorus#161 — `ReactiveChannelService.findByNamePrefix()` for efficient listing
- claudony#116 — verify reactive PostgreSQL path end-to-end
- claudony#117 — `ClaudonyChannelBackend` for conversation panel display (inbound)
- claudony#118 — multi-node fleet channel delivery (CLUSTER scope)

---

## Section 4 — `ClaudonyReactiveWorkerContextProvider`

**Replaces:** `ClaudonyWorkerContextProvider` (deleted)

**Implements:** `ReactiveWorkerContextProvider` from `casehub-engine-api`

**Fast paths** (clean-start, null caseId) return `Uni.createFrom().item(...)` directly.

**Main path** — lineage and channel listing run in parallel (independent queries):

```java
return Uni.combine().all()
    .unis(lineageQuery.findCompletedWorkers(caseId),
          channelProvider.listChannels(caseId))
    .asTuple()
    .map(tuple -> assemble(workerId, caseId, task,
                           tuple.getItem1(), tuple.getItem2()));
```

`assemble()` extracts the existing synchronous assembly logic — pure, non-blocking,
unit-testable without a Mutiny harness.

Dependencies: reactive `CaseLineageQuery`, `ReactiveCaseChannelProvider`,
`MeshParticipationStrategy`, `CaseChannelLayout`, `CaseHubConfig` (unchanged).

Package-private constructors for tests updated to accept `ReactiveCaseChannelProvider`.

---

## Section 5 — `ClaudonyReactiveWorkerProvisioner`

**Replaces:** `ClaudonyWorkerProvisioner` (deleted)

**Implements:** `ReactiveWorkerProvisioner` from `casehub-engine-api`

Tmux `ProcessBuilder` and session registry operations are inherently blocking. Virtual-
thread offload is the correct mechanism:

```java
public Uni<Worker> provision(Set<String> capabilities, ProvisionContext context) {
    return Uni.createFrom()
              .item(() -> doProvision(capabilities, context))
              .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
}
```

`doProvision()` is the current `provision()` logic as a synchronous helper — easy to
test without Mutiny. `terminate()` and `getCapabilities()` follow the same offload
pattern.

**Dead code removed:** The current provisioner injects `ClaudonyWorkerContextProvider`
but never calls it. The engine calls the context provider separately; the result arrives
via `ProvisionContext.workerContext()`. This injection is removed.

---

## Section 6 — Cleanup and test updates

**Deleted:**
- `ClaudonyWorkerContextProvider`
- `ClaudonyCaseChannelProvider`
- `ClaudonyWorkerProvisioner`

**Test classes updated:**
- `ClaudonyWorkerContextProviderTest` → `ClaudonyReactiveWorkerContextProviderTest`
- `ClaudonyCaseChannelProviderTest` → `ClaudonyReactiveCaseChannelProviderTest`
- `ClaudonyWorkerProvisionerTest` → `ClaudonyReactiveWorkerProvisionerTest`
- `WorkerLifecycleSequenceTest` — updated for reactive types
- `MeshParticipationIntegrationTest`, `SystemPromptIntegrationTest` — updated
- `CaseEngineRoundTripTest` — **`@InjectMock ClaudonyWorkerContextProvider` removed**;
  real `ClaudonyReactiveWorkerContextProvider` exercised through the engine pipeline

**Configuration:**
- `application.properties` — `quarkus.datasource.qhorus.reactive=true`
- `pom.xml` (test scope) — `quarkus-reactive-h2-client` extension

---

## Acceptance criteria

- `CaseEngineRoundTripTest` removes `@InjectMock ClaudonyWorkerContextProvider` — the
  real implementation is exercised
- No `BlockingOperationNotAllowedException` in test or production
- All 479+ tests pass with zero failures

---

## Deferred concerns captured as issues

| Issue | Repo | Description |
|-------|------|-------------|
| #116 | claudony | Verify reactive Qhorus stack for PostgreSQL deployments |
| #117 | claudony | `ClaudonyChannelBackend` — conversation panel inbound display |
| #118 | claudony | Multi-node fleet channel message delivery (CLUSTER scope) |
| #141 | qhorus | Gate `quarkus-hibernate-reactive-panache` behind build-time flag; investigate `quarkus-reactive-h2-client` |
| #161 | qhorus | `ReactiveChannelService.findByNamePrefix()` for efficient per-case channel listing |

---

## Platform docs updated this session

- `casehub-parent/docs/PLATFORM.md` — `ChannelBackend` + `MessageObserver` capability rows; three new boundary rules
- `casehub-qhorus/docs/messaging-architecture.md` — topology guidance: LOCAL vs CLUSTER vs multi-node fleet

---

## Out of scope

- `CaseHub.startCase()` into the round-trip test — blocked on engine Quartz/JTA IO-thread fix (#113)
- `ClaudonyChannelBackend` implementation — tracked as #117
- Reactive PostgreSQL validation — tracked as #116
