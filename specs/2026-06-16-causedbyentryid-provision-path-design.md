# causedByEntryId in provision path — Design

**Date:** 2026-06-16  
**Issue:** claudony#94 (companion: engine#231 closed, engine#500 open)  
**Branch:** issue-151-engine-compile-scope

---

## Problem

When a Qhorus COMMAND triggers a case context change that causes worker provisioning, there is an auditable causal link: the COMMAND's `MessageLedgerEntry` caused the `WorkerStarted` `CaseLedgerEntry`. The W3C PROV-DM chain should be:

```
MessageLedgerEntry (COMMAND)
  → CaseLedgerEntry (WorkerStarted, causedByEntryId = COMMAND.id)
```

Currently `ClaudonyReactiveWorkerProvisioner.doProvision()` always returns `ProvisionResult.empty()` (null `causedByEntryId`). The scaffold (`causalContext` map, `drainCausalContext()`, the drain in `ClaudonyLedgerEventCapture`) was built waiting for engine#231 to thread `triggerChannelId` and `triggerCorrelationId` through `ProvisionContext`. engine#231 is now closed — `ProvisionContext` carries both fields.

---

## Why the side-channel map is the correct permanent design

The engine's design spec (engine#389, approved) explicitly rejected putting `causedByEntryId` on `CaseLifecycleEvent`:

> "`CaseLifecycleEvent` stays at 7 fields. `causedByEntryId` would be null for every event except `WorkerStarted` and only meaningful to claudony's ledger observer. **Shared events must not carry consumer-specific fields.** Claudony's provisioner stores `causedByEntryId` in an in-memory map on provision; its ledger capture drains the map when it observes `WorkerStarted`."

The `causalContext: ConcurrentHashMap<UUID, UUID>` is the intended, permanent architecture — not a workaround. The Javadoc on `causalContext` must state this explicitly so future maintainers don't conflate the field on `ProvisionResult` with a possible future field on `CaseLifecycleEvent`.

---

## Chain design

### Threading constraint

`SessionOperations.withSession()` (called by `@WithSession("qhorus")`) calls `vertxContext()`, which calls:

```java
VertxContextSafetyToggle.validateContextIfExists(ERROR_MSG, ERROR_MSG);
```

This requires a **safe (isolated) Vert.x sub-context** — a duplicated context. Worker pool threads after `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` do NOT have this. A sequential chain of the form:

```java
.item(() -> setupSession(...))
.runSubscriptionOn(workerPool)
.flatMap(ignored -> causalLinkResolver.resolve(...))   // @WithSession fires from worker thread → FAILS
```

would throw *"Hibernate Reactive Panache requires a safe (isolated) Vert.x sub-context"*.

### Parallel structure — the correct design

`setupSession()` (blocking tmux IO) and `causalLinkResolver.resolve()` (reactive DB lookup) are **independent**. They can run concurrently via `Uni.combine()`. Crucially, `resolve()` is called from the **event loop** where `provision()` is invoked — before any thread switch — so `@WithSession("qhorus")` intercepts in the correct context.

```java
@Override
public Uni<ProvisionResult> provision(Set<String> capabilities, ProvisionContext context) {
    Uni<Void> setup = Uni.createFrom()
        .<Void>item(() -> { setupSession(capabilities, context); return null; })
        .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());

    Uni<Optional<UUID>> causedBy =
        (causalLinkResolver != null
         && context.triggerChannelId() != null
         && context.triggerCorrelationId() != null)
        ? causalLinkResolver.resolve(context.triggerChannelId(), context.triggerCorrelationId())
        : Uni.createFrom().item(Optional.empty());

    return Uni.combine().all().unis(setup, causedBy).asTuple()
        .invoke(tuple -> {
            if (context.caseId() != null) {
                tuple.getItem2().ifPresent(id -> causalContext.put(context.caseId(), id));
            }
        })
        .map(tuple -> new ProvisionResult(tuple.getItem2().orElse(null)))
        .call(result -> signalStarted(capabilities, context))
        .invoke(result -> startWatcher(capabilities, context));
}
```

**Why the guard checks both `causalLinkResolver` and the trigger fields:**

The guard must be at the `Uni` construction site before `Uni.combine()`. If `causalLinkResolver != null` but a trigger field is null, calling `causalLinkResolver.resolve(null, null)` returns a null `Uni` (Mockito default) which NPEs inside `Uni.combine()`. The guard short-circuits on either absence: null resolver (13 existing tests) or null trigger fields (tests with a resolver but no trigger data).

The null checks inside `QhorusCausalLinkResolver.resolve()` become a secondary defensive layer for any direct call path.

Guard evaluation table:

| `causalLinkResolver` | `triggerChannelId` | `triggerCorrelationId` | Result |
|---|---|---|---|
| `null` | any | any | short-circuit → `Optional.empty()` |
| non-null | `null` | any | short-circuit → `Optional.empty()` |
| non-null | non-null | `null` | short-circuit → `Optional.empty()` |
| non-null | non-null | non-null | call `resolve()` |

Each operation in its natural context:

| Step | Nature | Operator |
|------|--------|----------|
| `setupSession()` | blocking IO — tmux + registry | `.item()` + `runSubscriptionOn(workerPool)` |
| `causalLinkResolver.resolve()` | reactive IO — Qhorus DB | `Uni.combine()` (event loop) |
| `signalStarted()` | async — event bus | `.call()` |
| `startWatcher()` | sync — virtual thread | `.invoke()` |

**Why `.invoke()` not `.map()` for `causalContext.put()`:** `.map()` must be a pure transformation. Side effects belong in `.invoke()`.

**`caseId` null guard:** `ConcurrentHashMap.put(null, ...)` throws NPE. The guard on `context.caseId() != null` is explicit.

**If `setupSession()` fails:** `Uni.combine()` propagates the failure; `causalContext` is not populated and no `ProvisionResult` is built. Correct.

### `setupSession()` — renamed from `doProvision()`

```java
private void setupSession(Set<String> capabilities, ProvisionContext context) {
    // all blocking work: disabled guard, tmux.createWorkerSession(), tmux.setSessionOption(),
    // registry.register(), sessionMapping.register()
    // throws ProvisioningException on failure (propagated as Uni failure)
}
```

Returns `void`. The causal resolution and `ProvisionResult` construction move out.

### QhorusCausalLinkResolver — new CDI bean

New `@ApplicationScoped` class in `casehub/` module, package `io.casehub.claudony.casehub`:

```java
@ApplicationScoped
class QhorusCausalLinkResolver {

    @Inject Instance<ReactiveMessageLedgerEntryRepository> messageLedgerRepo;

    @WithSession("qhorus")
    Uni<Optional<UUID>> resolve(String channelIdStr, String correlationId) {
        if (channelIdStr == null || correlationId == null || messageLedgerRepo.isUnsatisfied())
            return Uni.createFrom().item(Optional.empty());
        UUID channelId;
        try { channelId = UUID.fromString(channelIdStr); }
        catch (IllegalArgumentException e) { return Uni.createFrom().item(Optional.empty()); }
        return messageLedgerRepo.get()
            .findLatestByCorrelationId(channelId, correlationId, null)
            .map(opt -> opt.map(e -> e.id));
    }
}
```

**Why a separate CDI bean:** `@WithSession("qhorus")` intercepts only via the CDI proxy. A direct/private method call on `ClaudonyReactiveWorkerProvisioner` bypasses interception. A separate `@ApplicationScoped` bean ensures the proxy wraps `resolve()`.

**Why called from the event loop:** `causalLinkResolver.resolve()` is called at the point where `provision()` is invoked — on the Vert.x event loop. The CDI proxy intercepts here; `@WithSession("qhorus")` fires and captures the event loop's safe Vert.x duplicated context. The session is bound into that context. When `Uni.combine()` subscribes to the `causedBy` Uni, the reactive Panache query runs within that session.

**`Instance<>` injection:** `ReactiveMessageLedgerEntryRepository` is gated by `@IfBuildProperty(casehub.qhorus.reactive.enabled=true)`. Using `Instance<>` gracefully handles the absent case.

### Tenancy

`findLatestByCorrelationId` is called with `null` tenancyId (→ `DEFAULT_TENANT_ID`). Correct for single-tenant Claudony. Multi-tenant requires `triggerTenancyId` on `ProvisionContext` — tracked in engine#500.

### Constructor change — impact on existing tests

The package-private constructor gains a 9th parameter `QhorusCausalLinkResolver causalLinkResolver`:

```java
ClaudonyReactiveWorkerProvisioner(boolean enabled, TmuxService tmux, SessionRegistry registry,
        WorkerCommandResolver resolver, WorkerSessionMapping sessionMapping,
        String defaultWorkingDir, Instance<CaseHubRuntime> caseHubRuntime,
        ClaudonyWorkerExecutionManager execManager,
        QhorusCausalLinkResolver causalLinkResolver) { ... }
```

Call sites that must pass `null`:

| File | Location | Fix |
|------|----------|-----|
| `ClaudonyReactiveWorkerProvisionerTest.java` | `@BeforeEach` (affects all 13 tests) | append `null` |
| `ClaudonyReactiveWorkerProvisionerTest.java` | `provision_disabled_failsWithProvisioningException` | append `null` |
| `WorkerLifecycleSequenceTest.java` | `@BeforeEach` (affects 3+ tests) | append `null` |

`null` is safe: the extended guard `(causalLinkResolver != null && ...)` short-circuits before any dereference.

### causalContext Javadoc update

```java
// Permanent side-channel: CaseLifecycleEvent deliberately has no causedByEntryId field
// (shared events must not carry consumer-specific fields — see engine#389 design spec).
// This map bridges ProvisionResult.causedByEntryId → CaseLedgerEntry.causedByEntryId.
// Keyed by caseId; drained when WorkerStarted fires. Safe for concurrent access;
// one provisioning per case at a time is the architectural invariant.
private final ConcurrentHashMap<UUID, UUID> causalContext = new ConcurrentHashMap<>();
```

---

## Test plan

### QhorusCausalLinkResolverTest — unit tests (plain JUnit, no Quarkus)

Mock `Instance<ReactiveMessageLedgerEntryRepository>` directly — bypasses CDI proxy and `@WithSession` (covered by the integration test). Tests the resolver's own conditional logic.

| Test | Assertion |
|------|-----------|
| `resolve_nullChannelId_returnsEmpty` | `Optional.empty()` |
| `resolve_nullCorrelationId_returnsEmpty` | `Optional.empty()` |
| `resolve_repoUnsatisfied_returnsEmpty` | `Optional.empty()` |
| `resolve_invalidChannelIdUuid_returnsEmpty` | `Optional.empty()` |
| `resolve_entryFound_returnsEntryId` | `Optional.of(entry.id)` |
| `resolve_entryNotFound_returnsEmpty` | `Optional.empty()` |

### QhorusCausalLinkResolverIntegrationTest — required `@QuarkusTest` in `app/`

**This test is mandatory.** It validates the core assumption of the design: `@WithSession("qhorus")` intercepts correctly when `resolve()` is called from the Vert.x event loop context.

**Prerequisite (qhorus#280):** `MessageLedgerEntryTestFactory` must be moved from `casehub-qhorus/runtime/src/test/java/` to the `casehub-qhorus-testing` module before this test can be written. The factory has no test-framework dependencies — it is a pure domain factory using only `LedgerEntryType`, `ActorType`, and `TenancyConstants`, all of which are already compile-scope dependencies of `casehub-qhorus-testing`. The integration test blocks on qhorus#280.

**Seeding requirements for the round-trip test:**

`findLatestByCorrelationId` filters by four fields:
```sql
subjectId = ?1 AND correlationId = ?2 AND tenancyId = ?3 AND messageType IN ('COMMAND','HANDOFF')
```

Required fields (non-obvious ones called out):

- **`subjectId` = channelId UUID** — the JPQL query uses the inherited `LedgerEntry.subjectId`, not `MessageLedgerEntry.channelId`. Both hold the same channel UUID; set both to the same value.
- **`messageType` = `"COMMAND"` or `"HANDOFF"`** — any other type is excluded by the `IN` clause.
- **`tenancyId` = `TenancyConstants.DEFAULT_TENANT_ID`** — `resolve()` passes `null` → normalised to DEFAULT.
- **`sequenceNumber`** — `nullable = false`; must be set (e.g., `1`).
- **`id`** — assigned by `@PrePersist` on `LedgerEntry`; do not set manually. **Read `entry.id` after `save()` to capture the UUID for the assertion.**

Use `MessageLedgerEntryTestFactory.entry(channelId, 1L, "COMMAND", channelId, correlationId)` as the seeding template (available after qhorus#280).

Seeding and assertion pattern (use `@RunOnVertxContext + UniAsserter`, same pattern as `ClaudonyReactiveCaseChannelProviderPostgresIT`):

```java
private UUID channelId;
private String correlationId;
private UUID expectedEntryId;

@BeforeEach
void seed() {
    channelId = UUID.randomUUID();
    correlationId = "corr-" + UUID.randomUUID();
    MessageLedgerEntry entry = MessageLedgerEntryTestFactory.entry(
        channelId, 1L, "COMMAND", channelId, correlationId);
    QuarkusTransaction.requiringNew().run(
        () -> ledgerEntryRepository.save(entry, TenancyConstants.DEFAULT_TENANT_ID));
    expectedEntryId = entry.id; // assigned by @PrePersist during save
}

@Test
@RunOnVertxContext
void resolve_missingEntry_returnsEmptyWithoutSessionError(UniAsserter asserter) {
    asserter.assertThat(
        () -> resolver.resolve(UUID.randomUUID().toString(), "no-such-corr"),
        opt -> assertThat(opt).isEmpty());
}

@Test
@RunOnVertxContext
void resolve_seededEntry_returnsEntryId(UniAsserter asserter) {
    asserter.assertThat(
        () -> resolver.resolve(channelId.toString(), correlationId),
        opt -> assertThat(opt).contains(expectedEntryId));
}
```

| Test | What it proves |
|------|----------------|
| `resolve_missingEntry_returnsEmptyWithoutSessionError` | `@WithSession` intercepts in event loop — no context error |
| `resolve_seededEntry_returnsEntryId` | full round-trip: H2 lookup returns the seeded entry's auto-assigned UUID |

### ClaudonyReactiveWorkerProvisionerTest — additions to existing file

The two new causal context tests require a provisioner constructed with a **mocked resolver** — they cannot use the `@BeforeEach` provisioner (null resolver). Each test creates its own provisioner inline:

```java
@Test
void provision_withTriggerFields_storesCausalContextAndReturnsEntryId() throws Exception {
    UUID entryId = UUID.randomUUID();
    QhorusCausalLinkResolver mockResolver = mock(QhorusCausalLinkResolver.class);
    when(mockResolver.resolve("channel-uuid", "corr-id"))
        .thenReturn(Uni.createFrom().item(Optional.of(entryId)));
    var prov = new ClaudonyReactiveWorkerProvisioner(
        true, tmux, registry, resolver, sessionMapping, "/tmp/workers", null, null, mockResolver);
    UUID caseId = UUID.randomUUID();
    var ctx = new ProvisionContext(caseId, "code-reviewer", null, null, "channel-uuid", "corr-id");

    ProvisionResult result = prov.provision(Set.of("code-reviewer"), ctx).await().indefinitely();

    assertThat(result.causedByEntryId()).isEqualTo(entryId);
    assertThat(prov.drainCausalContext(caseId)).isEqualTo(entryId);
}

@Test
void provision_withNullTriggerFields_noCausalContext() throws Exception {
    QhorusCausalLinkResolver mockResolver = mock(QhorusCausalLinkResolver.class);
    var prov = new ClaudonyReactiveWorkerProvisioner(
        true, tmux, registry, resolver, sessionMapping, "/tmp/workers", null, null, mockResolver);
    UUID caseId = UUID.randomUUID();

    ProvisionResult result = prov.provision(Set.of("code-reviewer"),
        new ProvisionContext(caseId, "code-reviewer", null, null, null, null))
        .await().indefinitely();

    assertThat(result.causedByEntryId()).isNull();
    assertThat(prov.drainCausalContext(caseId)).isNull();
    // null trigger fields → guard short-circuits before causalLinkResolver.resolve() is called
    verifyNoInteractions(mockResolver);
}
```

The existing `provision_withNullTriggerFields_returnsEmptyProvisionResult` (null resolver, null trigger fields) remains valid — the guard short-circuits on `causalLinkResolver != null` being false.

**Update all 13 existing tests + `provision_disabled`:** append `null` as 9th constructor argument.

### ClaudonyLedgerEventCaptureTest — no changes

Existing drain tests remain valid. `seedCausalContextForTest()` continues to serve the drain path tests.

---

## Out of scope / deferred

- `triggerTenancyId` on `ProvisionContext` — engine#500; single-tenant DEFAULT is correct until then
- Multi-tenant `causedByEntryId` accuracy — follows from engine#500
- `MessageLedgerEntryTestFactory` move to `casehub-qhorus-testing` — qhorus#280 (prerequisite for integration test)

---

## Files changed

| File | Change |
|------|--------|
| `casehub/src/main/java/io/casehub/claudony/casehub/QhorusCausalLinkResolver.java` | new |
| `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java` | `doProvision()` → `setupSession()` (void); inject `QhorusCausalLinkResolver`; restructure `provision()` with `Uni.combine()` + extended guard; update `causalContext` Javadoc |
| `casehub/src/test/java/io/casehub/claudony/casehub/QhorusCausalLinkResolverTest.java` | new |
| `app/src/test/java/io/casehub/claudony/casehub/QhorusCausalLinkResolverIntegrationTest.java` | new (required, blocks on qhorus#280) |
| `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java` | 3 constructor call sites updated; 2 new causal context tests with per-test provisioner+mock |
| `casehub/src/test/java/io/casehub/claudony/casehub/WorkerLifecycleSequenceTest.java` | 1 constructor call site updated |