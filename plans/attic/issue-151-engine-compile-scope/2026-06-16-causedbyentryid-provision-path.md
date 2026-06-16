# causedByEntryId in Provision Path — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a Qhorus COMMAND triggers worker provisioning, set `causedByEntryId` on the resulting `CaseLedgerEntry` so the W3C PROV-DM causal chain is complete.

**Architecture:** `provision()` is restructured from a sequential blocking chain to a parallel `Uni.combine()` that runs `setupSession()` (blocking tmux IO, worker pool) and `QhorusCausalLinkResolver.resolve()` (reactive Qhorus DB lookup, event loop) concurrently. The resolver's result is stored in the existing `causalContext` side-channel map and returned as `ProvisionResult.causedByEntryId`. `ClaudonyLedgerEventCapture` already drains that map on `WorkerStarted` — no changes needed there.

**Tech Stack:** Quarkus 3.32.2, Mutiny (`Uni.combine()`), Hibernate Reactive Panache (`@WithSession("qhorus")`), JUnit 5, Mockito. All changes in the `casehub` Maven module unless noted.

---

## Task 1: Expand provisioner constructor + update all existing call sites

The package-private constructor gains a 9th parameter. Update it and all three call sites before touching any logic — this makes the existing 13 tests compile with no behaviour change.

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/WorkerLifecycleSequenceTest.java`

- [ ] **Step 1.1: Add field and expand both constructors in `ClaudonyReactiveWorkerProvisioner`**

Add the field after `execManager`:
```java
private final ClaudonyWorkerExecutionManager execManager;
private final QhorusCausalLinkResolver causalLinkResolver;  // null in tests; non-null in CDI
```

Update the CDI `@Inject` constructor (add parameter and delegate):
```java
@Inject
public ClaudonyReactiveWorkerProvisioner(
        CaseHubConfig config,
        TmuxService tmux,
        SessionRegistry registry,
        WorkerCommandResolver resolver,
        WorkerSessionMapping sessionMapping,
        Instance<CaseHubRuntime> caseHubRuntime,
        ClaudonyWorkerExecutionManager execManager,
        QhorusCausalLinkResolver causalLinkResolver) {
    this(config.enabled(), tmux, registry, resolver, sessionMapping,
            config.workers().defaultWorkingDir(), caseHubRuntime, execManager, causalLinkResolver);
}
```

Update the package-private constructor:
```java
ClaudonyReactiveWorkerProvisioner(boolean enabled, TmuxService tmux, SessionRegistry registry,
        WorkerCommandResolver resolver, WorkerSessionMapping sessionMapping,
        String defaultWorkingDir, Instance<CaseHubRuntime> caseHubRuntime,
        ClaudonyWorkerExecutionManager execManager,
        QhorusCausalLinkResolver causalLinkResolver) {
    this.enabled = enabled;
    this.tmux = tmux;
    this.registry = registry;
    this.resolver = resolver;
    this.sessionMapping = sessionMapping;
    this.defaultWorkingDir = defaultWorkingDir;
    this.caseHubRuntime = caseHubRuntime;
    this.execManager = execManager;
    this.causalLinkResolver = causalLinkResolver;
}
```

- [ ] **Step 1.2: Update `ClaudonyReactiveWorkerProvisionerTest` — append `null` to 2 constructor call sites**

In `@BeforeEach setUp()`:
```java
provisioner = new ClaudonyReactiveWorkerProvisioner(true, tmux, registry, resolver, sessionMapping, "/tmp/workers", null, null, null);
```

In `provision_disabled_failsWithProvisioningException`:
```java
var disabledProvisioner = new ClaudonyReactiveWorkerProvisioner(
        false, tmux, registry, resolver, sessionMapping, "/tmp", null, null, null);
```

- [ ] **Step 1.3: Update `WorkerLifecycleSequenceTest` — append `null` to 1 constructor call site**

In `@BeforeEach setUp()`:
```java
provisioner = new ClaudonyReactiveWorkerProvisioner(
        true, tmux, registry, resolver, sessionMapping, "/workspace", null, null, null);
```

- [ ] **Step 1.4: Run all casehub tests — confirm they still pass (no behaviour change)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 1.5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java \
        casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java \
        casehub/src/test/java/io/casehub/claudony/casehub/WorkerLifecycleSequenceTest.java
git commit -m "refactor(claudony#94): expand ClaudonyReactiveWorkerProvisioner constructor — 9th param for QhorusCausalLinkResolver"
```

---

## Task 2: Write QhorusCausalLinkResolver unit tests (TDD — write failing first)

All six unit tests for the resolver's conditional logic. Called directly (bypasses CDI proxy and `@WithSession` — correct for unit tests; the integration test verifies the session path). The `messageLedgerRepo` field is set directly because `QhorusCausalLinkResolver` uses package-private field injection, matching the existing Claudony test convention.

**Files:**
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/QhorusCausalLinkResolverTest.java`

- [ ] **Step 2.1: Create the test file**

```java
package io.casehub.claudony.casehub;

import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.ReactiveMessageLedgerEntryRepository;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class QhorusCausalLinkResolverTest {

    private ReactiveMessageLedgerEntryRepository repo;
    private QhorusCausalLinkResolver resolver;

    @SuppressWarnings("unchecked")
    @BeforeEach
    void setUp() {
        repo = mock(ReactiveMessageLedgerEntryRepository.class);
        Instance<ReactiveMessageLedgerEntryRepository> instance = mock(Instance.class);
        when(instance.isUnsatisfied()).thenReturn(false);
        when(instance.get()).thenReturn(repo);
        resolver = new QhorusCausalLinkResolver();
        resolver.messageLedgerRepo = instance;
    }

    @Test
    void resolve_nullChannelId_returnsEmpty() {
        Optional<UUID> result = resolver.resolve(null, "corr-id").await().indefinitely();

        assertThat(result).isEmpty();
        verifyNoInteractions(repo);
    }

    @Test
    void resolve_nullCorrelationId_returnsEmpty() {
        Optional<UUID> result = resolver.resolve("channel-uuid", null).await().indefinitely();

        assertThat(result).isEmpty();
        verifyNoInteractions(repo);
    }

    @Test
    @SuppressWarnings("unchecked")
    void resolve_repoUnsatisfied_returnsEmpty() {
        Instance<ReactiveMessageLedgerEntryRepository> unsatisfied = mock(Instance.class);
        when(unsatisfied.isUnsatisfied()).thenReturn(true);
        resolver.messageLedgerRepo = unsatisfied;

        Optional<UUID> result = resolver.resolve("channel-uuid", "corr-id").await().indefinitely();

        assertThat(result).isEmpty();
    }

    @Test
    void resolve_invalidChannelIdUuid_returnsEmpty() {
        Optional<UUID> result = resolver.resolve("not-a-uuid", "corr-id").await().indefinitely();

        assertThat(result).isEmpty();
        verifyNoInteractions(repo);
    }

    @Test
    void resolve_entryFound_returnsEntryId() {
        UUID channelId = UUID.randomUUID();
        MessageLedgerEntry entry = new MessageLedgerEntry();
        entry.id = UUID.randomUUID();
        when(repo.findLatestByCorrelationId(channelId, "corr-id", null))
            .thenReturn(Uni.createFrom().item(Optional.of(entry)));

        Optional<UUID> result = resolver.resolve(channelId.toString(), "corr-id")
            .await().indefinitely();

        assertThat(result).contains(entry.id);
    }

    @Test
    void resolve_entryNotFound_returnsEmpty() {
        UUID channelId = UUID.randomUUID();
        when(repo.findLatestByCorrelationId(channelId, "corr-id", null))
            .thenReturn(Uni.createFrom().item(Optional.empty()));

        Optional<UUID> result = resolver.resolve(channelId.toString(), "corr-id")
            .await().indefinitely();

        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2.2: Run the new tests — confirm they fail (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=QhorusCausalLinkResolverTest
```

Expected: COMPILATION FAILURE — `QhorusCausalLinkResolver` does not exist.

---

## Task 3: Implement QhorusCausalLinkResolver

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/QhorusCausalLinkResolver.java`

- [ ] **Step 3.1: Create the class**

```java
package io.casehub.claudony.casehub;

import io.casehub.qhorus.runtime.ledger.ReactiveMessageLedgerEntryRepository;
import io.quarkus.hibernate.reactive.panache.common.WithSession;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.Optional;
import java.util.UUID;

/**
 * Resolves the Qhorus MessageLedgerEntry UUID that caused a provisioning event.
 *
 * Called from the event loop (before runSubscriptionOn(workerPool)) so that
 * @WithSession("qhorus") intercepts with the correct Vert.x safe sub-context.
 */
@ApplicationScoped
class QhorusCausalLinkResolver {

    @Inject
    Instance<ReactiveMessageLedgerEntryRepository> messageLedgerRepo;

    @WithSession("qhorus")
    Uni<Optional<UUID>> resolve(String channelIdStr, String correlationId) {
        if (channelIdStr == null || correlationId == null || messageLedgerRepo.isUnsatisfied()) {
            return Uni.createFrom().item(Optional.empty());
        }
        UUID channelId;
        try {
            channelId = UUID.fromString(channelIdStr);
        } catch (IllegalArgumentException e) {
            return Uni.createFrom().item(Optional.empty());
        }
        return messageLedgerRepo.get()
            .findLatestByCorrelationId(channelId, correlationId, null)
            .map(opt -> opt.map(e -> e.id));
    }
}
```

- [ ] **Step 3.2: Run the unit tests — confirm they all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=QhorusCausalLinkResolverTest
```

Expected: BUILD SUCCESS, 6 tests pass.

- [ ] **Step 3.3: Run all casehub tests — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 3.4: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/QhorusCausalLinkResolver.java \
        casehub/src/test/java/io/casehub/claudony/casehub/QhorusCausalLinkResolverTest.java
git commit -m "feat(claudony#94): add QhorusCausalLinkResolver — reactive Qhorus ledger lookup for causedByEntryId"
```

---

## Task 4: Write new provisioner causal context tests (TDD — write failing first)

Two tests that verify `provision()` populates `causalContext` and returns `ProvisionResult.causedByEntryId`. Each constructs its own provisioner with a mocked resolver — they cannot use the `@BeforeEach` provisioner (which has `null` for the resolver).

**Files:**
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java`

- [ ] **Step 4.1: Add the two new tests to `ClaudonyReactiveWorkerProvisionerTest`**

Add these two methods at the end of the class (before the `provisionContext` helper):

```java
@Test
void provision_withTriggerFields_storesCausalContextAndReturnsEntryId() throws Exception {
    UUID entryId = UUID.randomUUID();
    QhorusCausalLinkResolver mockResolver = mock(QhorusCausalLinkResolver.class);
    when(mockResolver.resolve("ch-123", "corr-456"))
        .thenReturn(Uni.createFrom().item(Optional.of(entryId)));
    var prov = new ClaudonyReactiveWorkerProvisioner(
        true, tmux, registry, resolver, sessionMapping, "/tmp/workers", null, null, mockResolver);
    UUID caseId = UUID.randomUUID();
    var ctx = new ProvisionContext(caseId, "code-reviewer", null, null, "ch-123", "corr-456");

    ProvisionResult result = prov.provision(Set.of("code-reviewer"), ctx)
        .await().indefinitely();

    assertThat(result.causedByEntryId()).isEqualTo(entryId);
    assertThat(prov.drainCausalContext(caseId)).isEqualTo(entryId);
    assertThat(prov.drainCausalContext(caseId)).isNull(); // drained — second call returns null
}

@Test
void provision_withNullTriggerFields_guardShortCircuits() throws Exception {
    QhorusCausalLinkResolver mockResolver = mock(QhorusCausalLinkResolver.class);
    var prov = new ClaudonyReactiveWorkerProvisioner(
        true, tmux, registry, resolver, sessionMapping, "/tmp/workers", null, null, mockResolver);
    UUID caseId = UUID.randomUUID();
    var ctx = new ProvisionContext(caseId, "code-reviewer", null, null, null, null);

    ProvisionResult result = prov.provision(Set.of("code-reviewer"), ctx)
        .await().indefinitely();

    assertThat(result.causedByEntryId()).isNull();
    assertThat(prov.drainCausalContext(caseId)).isNull();
    // null trigger fields → guard short-circuits before resolver is called
    verifyNoInteractions(mockResolver);
}
```

Also add the import at the top of the file (if not already present):
```java
import java.util.Optional;
```

- [ ] **Step 4.2: Run just the two new tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub \
  -Dtest="ClaudonyReactiveWorkerProvisionerTest#provision_withTriggerFields_storesCausalContextAndReturnsEntryId+provision_withNullTriggerFields_guardShortCircuits"
```

Expected: FAIL — `provision()` still returns `ProvisionResult.empty()` and never calls the resolver.

---

## Task 5: Restructure `provision()` with `Uni.combine()` + extract `setupSession()`

This is the core logic change. `doProvision()` is renamed to `setupSession()` (returns `void`). `provision()` is restructured into a parallel `Uni.combine()` chain.

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`

- [ ] **Step 5.1: Update the `causalContext` field Javadoc**

Replace the existing comment on `causalContext`:
```java
// Permanent side-channel: CaseLifecycleEvent deliberately has no causedByEntryId field
// (shared events must not carry consumer-specific fields — see engine#389 design spec).
// This map bridges ProvisionResult.causedByEntryId → CaseLedgerEntry.causedByEntryId.
// Keyed by caseId; drained when WorkerStarted fires. Safe for concurrent access;
// one provisioning per case at a time is the architectural invariant.
private final ConcurrentHashMap<UUID, UUID> causalContext = new ConcurrentHashMap<>();
```

- [ ] **Step 5.2: Restructure `provision()` with `Uni.combine()` and the extended guard**

Replace the entire `provision()` method:
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

- [ ] **Step 5.3: Rename `doProvision()` → `setupSession()` and change return type to `void`**

Replace `doProvision()` entirely. Remove the TODO comment and the final `return ProvisionResult.empty()` line. Change return type to `void`:

```java
private void setupSession(Set<String> capabilities, ProvisionContext context) {
    if (!enabled) {
        throw new ProvisioningException(
                "CaseHub integration is disabled — set claudony.casehub.enabled=true");
    }
    String sessionId = UUID.randomUUID().toString();
    String roleName = context.taskType() != null
            ? context.taskType()
            : capabilities.stream().findFirst().orElse("worker");
    String command = resolver.resolve(capabilities);
    String sessionName = SESSION_PREFIX + sessionId;

    try {
        tmux.createWorkerSession(sessionName, defaultWorkingDir, command);
        if (context.caseId() != null) {
            tmux.setSessionOption(sessionName, "@casehub_case_id", context.caseId().toString());
            tmux.setSessionOption(sessionName, "@casehub_role", roleName);
        }
    } catch (IOException | InterruptedException e) {
        throw new ProvisioningException("Failed to create tmux session for worker " + sessionId, e);
    }

    var session = new Session(sessionId, sessionName, defaultWorkingDir, command,
            SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty(),
            Optional.ofNullable(context.caseId()).map(UUID::toString),
            Optional.of(roleName));
    registry.register(session);
    sessionMapping.register(roleName, context.caseId(), sessionId);
}
```

- [ ] **Step 5.4: Run all two new tests — confirm they now pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub \
  -Dtest="ClaudonyReactiveWorkerProvisionerTest#provision_withTriggerFields_storesCausalContextAndReturnsEntryId+provision_withNullTriggerFields_guardShortCircuits"
```

Expected: BUILD SUCCESS, both tests pass.

- [ ] **Step 5.5: Run all casehub tests — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub
```

Expected: BUILD SUCCESS, all tests pass (including all 13 existing provisioner tests).

- [ ] **Step 5.6: Run app module tests — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app
```

Expected: BUILD SUCCESS. Known failing: `ResearcherCaseCompletionTest` (engine SNAPSHOT, tracked in #154), `MeshResourceInterjectionTest.postMessage_eventType_isValid` (qhorus#271). All others pass.

- [ ] **Step 5.7: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java \
        casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java
git commit -m "feat(claudony#94): wire causedByEntryId into provision path via Uni.combine()

Restructures provision() with parallel Uni.combine(): setupSession() (blocking tmux IO on
worker pool) and QhorusCausalLinkResolver.resolve() (reactive Qhorus DB, event loop) run
concurrently. Extended guard short-circuits when resolver is null OR trigger fields are null.
causalContext map stores (caseId → entryId); ClaudonyLedgerEventCapture drains it on WorkerStarted.

Refs claudony#94, engine#231"
```

---

## Task 6: Integration test — blocked on qhorus#280

The `QhorusCausalLinkResolverIntegrationTest` validates the `@WithSession("qhorus")` + event loop context assumption against a real H2 Qhorus datasource. It cannot be implemented until `MessageLedgerEntryTestFactory` is moved from `casehub-qhorus/runtime/src/test/java/` to the `casehub-qhorus-testing` artifact (qhorus#280).

**Do not implement this task until qhorus#280 is merged and a new `casehub-qhorus-testing` SNAPSHOT containing the factory is available.**

When qhorus#280 ships:

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/casehub/QhorusCausalLinkResolverIntegrationTest.java`

The test seeds a `MessageLedgerEntry` via blocking JPA and asserts the reactive lookup returns the correct UUID:

```java
package io.casehub.claudony.casehub;

import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryTestFactory;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.vertx.RunOnVertxContext;
import io.quarkus.test.vertx.UniAsserter;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class QhorusCausalLinkResolverIntegrationTest {

    @Inject QhorusCausalLinkResolver resolver;
    @Inject LedgerEntryRepository ledgerEntryRepository;

    private UUID channelId;
    private String correlationId;
    private UUID expectedEntryId;

    @BeforeEach
    void seed() {
        channelId = UUID.randomUUID();
        correlationId = "corr-" + UUID.randomUUID();
        MessageLedgerEntry entry = MessageLedgerEntryTestFactory.entry(
            channelId, 1L, "COMMAND", channelId, correlationId);
        // subjectId = channelId: the JPQL query filters on subjectId, not channelId column
        QuarkusTransaction.requiringNew().run(
            () -> ledgerEntryRepository.save(entry, TenancyConstants.DEFAULT_TENANT_ID));
        expectedEntryId = entry.id; // assigned by @PrePersist during save
    }

    @Test
    @RunOnVertxContext
    void resolve_missingEntry_returnsEmptyWithoutSessionError(UniAsserter asserter) {
        // Validates @WithSession("qhorus") intercepts from event loop — no context error thrown
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
}
```

Run with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=QhorusCausalLinkResolverIntegrationTest`

Commit with: `git commit -m "test(claudony#94): add QhorusCausalLinkResolverIntegrationTest — @WithSession event loop verification"`
