# Multi-Tenancy Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add unconditional tenancyId enforcement to Claudony's in-memory stores, CDI events, and cache keys per protocols PP-20260520-439daf and PP-20260520-e6a5f0.

**Architecture:** TenantContext interface + @ApplicationScoped impl in claudony-core delegates to CurrentPrincipal when request scope is active, falls back to TenancyConstants.DEFAULT_TENANT_ID otherwise. SessionRegistry filters all public query methods by tenant. Session record gains tenancyId as last field. CDI events gain tenancyId. System code uses explicit unscoped methods.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api (TenancyConstants, CurrentPrincipal), quarkus-arc (CDI)

**Spec:** `specs/2026-06-26-multitenancy-foundation-design.md` (v3)

## Global Constraints

- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` to run tests
- All commits reference `Refs casehubio/claudony#121`
- PP-20260520-439daf: tenancyId filtering always executes unconditionally — no conditional guards
- PP-20260520-e6a5f0: tenancyId resolved inside data access classes only — callers never touch CurrentPrincipal for tenancy
- `TenancyConstants.DEFAULT_TENANT_ID` is the single source of truth for the sentinel UUID
- Session.tenancyId is `String` (not Optional) — tenancy is never absent
- Constructor injection for SessionRegistry — plain JUnit compatibility required

---

### Task 1: TenantContext interface, DefaultTenantContext, and core pom.xml dependency

**Files:**
- Modify: `core/pom.xml`
- Create: `core/src/main/java/io/casehub/claudony/server/TenantContext.java`
- Create: `core/src/main/java/io/casehub/claudony/server/DefaultTenantContext.java`
- Test: `core/src/test/java/io/casehub/claudony/server/DefaultTenantContextTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.identity.TenancyConstants`, `io.casehub.platform.api.identity.CurrentPrincipal` (from casehub-platform-api)
- Produces: `TenantContext.currentTenantId()` — used by SessionRegistry (Task 2), ClaudonyWorkerStatusListener (Task 5), ClaudonyReactiveCaseChannelProvider (Task 6), SessionResource (Task 7)

- [ ] **Step 1: Add casehub-platform-api dependency to core/pom.xml**

Add between the existing `quarkus-arc` and `quarkus-scheduler` dependencies:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
```

Version is managed by the parent pom's `<dependencyManagement>` via `${casehub.version}`.

- [ ] **Step 2: Write the failing test for DefaultTenantContext**

Create `core/src/test/java/io/casehub/claudony/server/DefaultTenantContextTest.java`:

```java
package io.casehub.claudony.server;

import io.casehub.platform.api.identity.TenancyConstants;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultTenantContextTest {

    @Test
    void returnsDefaultTenantId_whenNoPrincipalResolvable() {
        var ctx = new DefaultTenantContext();
        assertThat(ctx.currentTenantId()).isEqualTo(TenancyConstants.DEFAULT_TENANT_ID);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=DefaultTenantContextTest`

Expected: FAIL — `DefaultTenantContext` class does not exist.

- [ ] **Step 4: Create TenantContext interface**

Create `core/src/main/java/io/casehub/claudony/server/TenantContext.java`:

```java
package io.casehub.claudony.server;

public interface TenantContext {
    String currentTenantId();
}
```

- [ ] **Step 5: Create DefaultTenantContext implementation**

Create `core/src/main/java/io/casehub/claudony/server/DefaultTenantContext.java`:

```java
package io.casehub.claudony.server;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.arc.Arc;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

@ApplicationScoped
public class DefaultTenantContext implements TenantContext {

    private final Instance<CurrentPrincipal> principal;

    @Inject
    public DefaultTenantContext(Instance<CurrentPrincipal> principal) {
        this.principal = principal;
    }

    DefaultTenantContext() {
        this.principal = null;
    }

    @Override
    public String currentTenantId() {
        if (principal != null && principal.isResolvable()
                && Arc.container() != null
                && Arc.container().requestContext().isActive()) {
            return principal.get().tenancyId();
        }
        return TenancyConstants.DEFAULT_TENANT_ID;
    }
}
```

Note: the no-arg constructor and `Arc.container() != null` guard enable plain JUnit usage (no CDI container). The `@Inject` constructor is the CDI path.

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=DefaultTenantContextTest`

Expected: PASS

- [ ] **Step 7: Commit**

```
git add core/pom.xml \
  core/src/main/java/io/casehub/claudony/server/TenantContext.java \
  core/src/main/java/io/casehub/claudony/server/DefaultTenantContext.java \
  core/src/test/java/io/casehub/claudony/server/DefaultTenantContextTest.java
git commit -m "feat(#121): add TenantContext interface and DefaultTenantContext with CurrentPrincipal delegation

Refs casehubio/claudony#121"
```

---

### Task 2: Session.tenancyId field and SessionRegistry tenant filtering

**Files:**
- Modify: `core/src/main/java/io/casehub/claudony/server/model/Session.java`
- Modify: `core/src/main/java/io/casehub/claudony/server/SessionRegistry.java`
- Create: `core/src/test/java/io/casehub/claudony/server/MutableTenantContext.java`
- Create: `core/src/test/java/io/casehub/claudony/server/SessionRegistryTenantFilterTest.java`
- Modify: `core/src/test/java/io/casehub/claudony/server/SessionRegistryTest.java`

**Interfaces:**
- Consumes: `TenantContext.currentTenantId()` (Task 1)
- Produces: `Session.tenancyId()`, `SessionRegistry.all()` (filtered), `SessionRegistry.find(id)` (filtered), `SessionRegistry.findByCaseId(caseId)` (filtered), `SessionRegistry.allUnscoped()`, `SessionRegistry.findUnscoped(id)`, `SessionRegistry.existsByName(name)` — used by all subsequent tasks

- [ ] **Step 1: Create MutableTenantContext test support class**

Create `core/src/test/java/io/casehub/claudony/server/MutableTenantContext.java`:

```java
package io.casehub.claudony.server;

import io.casehub.platform.api.identity.TenancyConstants;

public class MutableTenantContext implements TenantContext {

    private String tenantId = TenancyConstants.DEFAULT_TENANT_ID;

    public void setTenantId(String id) { this.tenantId = id; }

    public void resetForTest() { this.tenantId = TenancyConstants.DEFAULT_TENANT_ID; }

    @Override
    public String currentTenantId() { return tenantId; }
}
```

- [ ] **Step 2: Write failing tests for SessionRegistry tenant filtering**

Create `core/src/test/java/io/casehub/claudony/server/SessionRegistryTenantFilterTest.java`:

```java
package io.casehub.claudony.server;

import io.casehub.claudony.server.model.Session;
import io.casehub.claudony.server.model.SessionStatus;
import io.casehub.platform.api.identity.TenancyConstants;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class SessionRegistryTenantFilterTest {

    static final String TENANT_A = TenancyConstants.DEFAULT_TENANT_ID;
    static final String TENANT_B = "00000000-0000-0000-0000-000000000002";

    private MutableTenantContext tenantCtx;
    private SessionRegistry registry;

    @BeforeEach
    void setUp() {
        tenantCtx = new MutableTenantContext();
        registry = new SessionRegistry(tenantCtx);
    }

    private Session session(String id, String caseId, String tenancyId) {
        return new Session(id, "name-" + id, "/tmp", "cmd", SessionStatus.IDLE,
                Instant.now(), Instant.now(), Optional.empty(),
                Optional.ofNullable(caseId), Optional.empty(), tenancyId);
    }

    @Test
    void all_returnsOnlySessionsMatchingCurrentTenant() {
        registry.register(session("s1", null, TENANT_A));
        registry.register(session("s2", null, TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.all()).extracting(Session::id).containsExactly("s1");

        tenantCtx.setTenantId(TENANT_B);
        assertThat(registry.all()).extracting(Session::id).containsExactly("s2");
    }

    @Test
    void find_returnsEmpty_forOtherTenantSession() {
        registry.register(session("s1", null, TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.find("s1")).isEmpty();
    }

    @Test
    void find_returnsSession_forMatchingTenant() {
        registry.register(session("s1", null, TENANT_A));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.find("s1")).isPresent();
    }

    @Test
    void findByCaseId_filtersByTenant() {
        registry.register(session("s1", "case-1", TENANT_A));
        registry.register(session("s2", "case-1", TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.findByCaseId("case-1")).extracting(Session::id).containsExactly("s1");
    }

    @Test
    void allUnscoped_returnsAllSessionsRegardlessOfTenant() {
        registry.register(session("s1", null, TENANT_A));
        registry.register(session("s2", null, TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.allUnscoped()).hasSize(2);
    }

    @Test
    void findUnscoped_returnsSession_regardlessOfTenant() {
        registry.register(session("s1", null, TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.findUnscoped("s1")).isPresent();
    }

    @Test
    void existsByName_findsNamesAcrossAllTenants() {
        registry.register(session("s1", null, TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.existsByName("name-s1")).isTrue();
        assertThat(registry.existsByName("nonexistent")).isFalse();
    }

    @Test
    void remove_worksOnAnySession_regardlessOfTenant() {
        registry.register(session("s1", null, TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        assertThat(registry.remove("s1")).isNotNull();
    }

    @Test
    void updateStatus_worksOnAnySession_regardlessOfTenant() {
        registry.register(session("s1", null, TENANT_B));

        tenantCtx.setTenantId(TENANT_A);
        registry.updateStatus("s1", SessionStatus.ACTIVE);
        assertThat(registry.findUnscoped("s1").get().status()).isEqualTo(SessionStatus.ACTIVE);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=SessionRegistryTenantFilterTest`

Expected: FAIL — `Session` constructor doesn't accept `tenancyId`, `SessionRegistry` has no constructor taking `TenantContext`.

- [ ] **Step 4: Add tenancyId to Session record**

Modify `core/src/main/java/io/casehub/claudony/server/model/Session.java`:

```java
package io.casehub.claudony.server.model;

import java.time.Instant;
import java.util.Optional;

public record Session(
        String id,
        String name,
        String workingDir,
        String command,
        SessionStatus status,
        Instant createdAt,
        Instant lastActive,
        Optional<String> expiryPolicy,
        Optional<String> caseId,
        Optional<String> roleName,
        String tenancyId) {

    public Session withStatus(SessionStatus newStatus) {
        return new Session(id, name, workingDir, command, newStatus, createdAt, Instant.now(),
                expiryPolicy, caseId, roleName, tenancyId);
    }

    public Session withLastActive() {
        return new Session(id, name, workingDir, command, status, createdAt, Instant.now(),
                expiryPolicy, caseId, roleName, tenancyId);
    }
}
```

- [ ] **Step 5: Add tenant filtering to SessionRegistry**

Modify `core/src/main/java/io/casehub/claudony/server/SessionRegistry.java`:

```java
package io.casehub.claudony.server;

import io.casehub.claudony.server.model.Session;
import io.casehub.claudony.server.model.SessionStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;

@ApplicationScoped
public class SessionRegistry {

    private final Map<String, Session> sessions = new ConcurrentHashMap<>();
    private final List<Consumer<String>> changeListeners = new CopyOnWriteArrayList<>();
    private final TenantContext tenantContext;

    @Inject
    public SessionRegistry(TenantContext tenantContext) {
        this.tenantContext = tenantContext;
    }

    public void register(Session session) {
        sessions.put(session.id(), session);
    }

    public Optional<Session> find(String id) {
        return Optional.ofNullable(sessions.get(id))
                .filter(s -> s.tenancyId().equals(tenantContext.currentTenantId()));
    }

    public Collection<Session> all() {
        String tenant = tenantContext.currentTenantId();
        return sessions.values().stream()
                .filter(s -> s.tenancyId().equals(tenant))
                .toList();
    }

    public List<Session> findByCaseId(String caseId) {
        String tenant = tenantContext.currentTenantId();
        return sessions.values().stream()
                .filter(s -> s.tenancyId().equals(tenant))
                .filter(s -> s.caseId().map(caseId::equals).orElse(false))
                .sorted(Comparator.comparing(Session::createdAt))
                .toList();
    }

    public Optional<Session> findUnscoped(String id) {
        return Optional.ofNullable(sessions.get(id));
    }

    public Collection<Session> allUnscoped() {
        return Collections.unmodifiableCollection(sessions.values());
    }

    public boolean existsByName(String name) {
        return sessions.values().stream().anyMatch(s -> s.name().equals(name));
    }

    public Session remove(String id) {
        notifyListeners(id);
        return sessions.remove(id);
    }

    public void updateStatus(String id, SessionStatus status) {
        sessions.computeIfPresent(id, (k, s) -> s.withStatus(status));
        notifyListeners(id);
    }

    public void touch(String id) {
        sessions.computeIfPresent(id, (k, s) -> s.withLastActive());
    }

    public void addChangeListener(Consumer<String> listener) {
        changeListeners.add(listener);
    }

    private void notifyListeners(String sessionId) {
        var session = sessions.get(sessionId);
        if (session != null) {
            session.caseId().ifPresent(caseId ->
                    changeListeners.forEach(l -> l.accept(caseId)));
        }
    }
}
```

- [ ] **Step 6: Run new tenant filter tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=SessionRegistryTenantFilterTest`

Expected: PASS — all 9 tenant filter tests pass.

- [ ] **Step 7: Fix existing SessionRegistryTest**

Modify `core/src/test/java/io/casehub/claudony/server/SessionRegistryTest.java` — update `setUp()` and `session()`:

Change `setUp()` from:
```java
@BeforeEach
void setUp() { registry = new SessionRegistry(); }
```
to:
```java
@BeforeEach
void setUp() { registry = new SessionRegistry(() -> TenancyConstants.DEFAULT_TENANT_ID); }
```

Add import:
```java
import io.casehub.platform.api.identity.TenancyConstants;
```

Change `session()` from:
```java
private Session session(String id, String caseId) {
    return new Session(id, "name-" + id, "/tmp", "cmd", SessionStatus.IDLE,
            Instant.now(), Instant.now(), Optional.empty(),
            Optional.ofNullable(caseId), Optional.empty());
}
```
to:
```java
private Session session(String id, String caseId) {
    return new Session(id, "name-" + id, "/tmp", "cmd", SessionStatus.IDLE,
            Instant.now(), Instant.now(), Optional.empty(),
            Optional.ofNullable(caseId), Optional.empty(), TenancyConstants.DEFAULT_TENANT_ID);
}
```

- [ ] **Step 8: Run all core tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core`

Expected: PASS — all core module tests pass.

- [ ] **Step 9: Commit**

```
git add core/src/main/java/io/casehub/claudony/server/model/Session.java \
  core/src/main/java/io/casehub/claudony/server/SessionRegistry.java \
  core/src/test/java/io/casehub/claudony/server/MutableTenantContext.java \
  core/src/test/java/io/casehub/claudony/server/SessionRegistryTenantFilterTest.java \
  core/src/test/java/io/casehub/claudony/server/SessionRegistryTest.java
git commit -m "feat(#121): add tenancyId to Session, tenant filtering to SessionRegistry

Session record gains tenancyId as last field (non-optional).
SessionRegistry filters all(), find(), findByCaseId() by tenant.
System operations: allUnscoped(), findUnscoped(), existsByName().

Refs casehubio/claudony#121"
```

---

### Task 3: CDI events gain tenancyId + SessionIdleScheduler uses allUnscoped()

**Files:**
- Modify: `core/src/main/java/io/casehub/claudony/server/WorkerCaseLifecycleEvent.java`
- Modify: `core/src/main/java/io/casehub/claudony/server/CaseChannelCreatedEvent.java`
- Modify: `core/src/main/java/io/casehub/claudony/server/expiry/SessionIdleScheduler.java`

**Interfaces:**
- Consumes: `SessionRegistry.allUnscoped()` (Task 2)
- Produces: `WorkerCaseLifecycleEvent(String caseId, String tenancyId)`, `CaseChannelCreatedEvent(UUID channelId, String channelName, String tenancyId)` — consumed by casehub module (Task 5) and app module observers

- [ ] **Step 1: Add tenancyId to WorkerCaseLifecycleEvent**

Modify `core/src/main/java/io/casehub/claudony/server/WorkerCaseLifecycleEvent.java`:

```java
package io.casehub.claudony.server;

public record WorkerCaseLifecycleEvent(String caseId, String tenancyId) {}
```

- [ ] **Step 2: Add tenancyId to CaseChannelCreatedEvent**

Modify `core/src/main/java/io/casehub/claudony/server/CaseChannelCreatedEvent.java`:

```java
package io.casehub.claudony.server;

import java.util.UUID;

public record CaseChannelCreatedEvent(UUID channelId, String channelName, String tenancyId) {}
```

- [ ] **Step 3: Change SessionIdleScheduler to use allUnscoped()**

In `core/src/main/java/io/casehub/claudony/server/expiry/SessionIdleScheduler.java`, change line 32 from:

```java
for (var session : registry.all()) {
```
to:
```java
for (var session : registry.allUnscoped()) {
```

- [ ] **Step 4: Run core tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core`

Expected: PASS — core tests pass. The event record changes will cause compile errors in casehub and app modules, but core tests are self-contained.

- [ ] **Step 5: Commit**

```
git add core/src/main/java/io/casehub/claudony/server/WorkerCaseLifecycleEvent.java \
  core/src/main/java/io/casehub/claudony/server/CaseChannelCreatedEvent.java \
  core/src/main/java/io/casehub/claudony/server/expiry/SessionIdleScheduler.java
git commit -m "feat(#121): add tenancyId to CDI events, use allUnscoped() in expiry scheduler

WorkerCaseLifecycleEvent and CaseChannelCreatedEvent gain tenancyId field.
SessionIdleScheduler uses allUnscoped() — must expire across all tenants.

Refs casehubio/claudony#121"
```

---

### Task 4: casehub module — provisioner tenancyId, causal context key, drainCausalContext signature

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCapture.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManager.java`

**Interfaces:**
- Consumes: `Session(... tenancyId)` (Task 2), `ProvisionContext.tenancyId()`, `SessionRegistry.findUnscoped()` (Task 2)
- Produces: `drainCausalContext(String tenancyId, UUID caseId)` — called by `ClaudonyLedgerEventCapture`, `seedCausalContextForTest(String tenancyId, UUID caseId, UUID entryId)` — called by tests

- [ ] **Step 1: Update ClaudonyReactiveWorkerProvisioner**

In `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`:

**a)** Change causal context map key (line 37):

From:
```java
private final ConcurrentHashMap<UUID, UUID> causalContext = new ConcurrentHashMap<>();
```
To:
```java
private record CausalKey(String tenancyId, UUID caseId) {}
private final ConcurrentHashMap<CausalKey, UUID> causalContext = new ConcurrentHashMap<>();
```

**b)** Update `provision()` — causal context put (line 106):

From:
```java
tuple.getItem2().ifPresent(id -> causalContext.put(context.caseId(), id));
```
To:
```java
tuple.getItem2().ifPresent(id -> causalContext.put(new CausalKey(context.tenancyId(), context.caseId()), id));
```

**c)** Update `drainCausalContext()` signature (line 171):

From:
```java
UUID drainCausalContext(UUID caseId) {
    return causalContext.remove(caseId);
}
```
To:
```java
UUID drainCausalContext(String tenancyId, UUID caseId) {
    return causalContext.remove(new CausalKey(tenancyId, caseId));
}
```

**d)** Update `seedCausalContextForTest()` (line 176):

From:
```java
void seedCausalContextForTest(UUID caseId, UUID entryId) {
    causalContext.put(caseId, entryId);
}
```
To:
```java
void seedCausalContextForTest(String tenancyId, UUID caseId, UUID entryId) {
    causalContext.put(new CausalKey(tenancyId, caseId), entryId);
}
```

**e)** Update `setupSession()` — Session construction (line 207-210) and tmux option (after line 201):

Add after `tmux.setSessionOption(sessionName, "@casehub_role", roleName);`:
```java
tmux.setSessionOption(sessionName, "@casehub_tenant_id", context.tenancyId());
```

Change Session construction from:
```java
var session = new Session(sessionId, sessionName, effectiveWorkingDir, enrichedCommand,
        SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty(),
        Optional.ofNullable(context.caseId()).map(UUID::toString),
        Optional.of(roleName));
```
To:
```java
var session = new Session(sessionId, sessionName, effectiveWorkingDir, enrichedCommand,
        SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty(),
        Optional.ofNullable(context.caseId()).map(UUID::toString),
        Optional.of(roleName), context.tenancyId());
```

- [ ] **Step 2: Update ClaudonyLedgerEventCapture — drainCausalContext call**

In `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCapture.java`, line 85:

From:
```java
UUID causedBy = provisioner.drainCausalContext(event.caseId());
```
To:
```java
UUID causedBy = provisioner.drainCausalContext(tenancyId, event.caseId());
```

(The `tenancyId` local variable is already available from line 58.)

- [ ] **Step 3: Update ClaudonyWorkerExecutionManager — findUnscoped in watcherRunnable**

In `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManager.java`, line 181:

From:
```java
if (registry.find(sessionId).isEmpty()) break;
```
To:
```java
if (registry.findUnscoped(sessionId).isEmpty()) break;
```

- [ ] **Step 4: Fix casehub test compilation — update all test Session constructions and seedCausalContextForTest calls**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl casehub`

Fix every compile error by adding `TenancyConstants.DEFAULT_TENANT_ID` as the last argument to `new Session(...)` calls and updating `seedCausalContextForTest` calls to include tenancyId. This is a mechanical migration — every test file that constructs a Session in the casehub module needs the extra argument. Every `seedCausalContextForTest(caseId, entryId)` becomes `seedCausalContextForTest(TenancyConstants.DEFAULT_TENANT_ID, caseId, entryId)`.

Files to fix (from earlier scan):
- `ClaudonyWorkerExecutionManagerTest.java` — Session construction + `new SessionRegistry()` → `new SessionRegistry(() -> TenancyConstants.DEFAULT_TENANT_ID)`
- `ClaudonyWorkerStatusListenerTest.java` — Session construction
- `WorkerLifecycleSequenceTest.java` — `new SessionRegistry()` → `new SessionRegistry(() -> TenancyConstants.DEFAULT_TENANT_ID)`

- [ ] **Step 5: Run casehub tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`

Expected: PASS

- [ ] **Step 6: Commit**

```
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java \
  casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCapture.java \
  casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManager.java \
  casehub/src/test/
git commit -m "feat(#121): tenancyId in provisioner, causal context, and watcher

Provisioner stamps tenancyId on Session and tmux session option.
Causal context keyed by (tenancyId, caseId).
Watcher uses findUnscoped() — virtual thread has no request scope.

Refs casehubio/claudony#121"
```

---

### Task 5: casehub module — ClaudonyWorkerStatusListener uses findUnscoped() + event tenancyId

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerStatusListener.java`

**Interfaces:**
- Consumes: `SessionRegistry.findUnscoped()` (Task 2), `TenantContext` (Task 1), `WorkerCaseLifecycleEvent(caseId, tenancyId)` (Task 3)
- Produces: Updated event fire sites with tenancyId

- [ ] **Step 1: Update ClaudonyWorkerStatusListener**

Modify `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerStatusListener.java`:

Add `TenantContext` to constructor injection. Change all `registry.find()` calls to `registry.findUnscoped()`. Resolve tenancyId from session for events, with fallback to `TenancyConstants.DEFAULT_TENANT_ID`.

```java
package io.casehub.claudony.casehub;

import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TenantContext;
import io.casehub.claudony.server.TmuxService;
import io.casehub.claudony.server.WorkerCaseLifecycleEvent;
import io.casehub.claudony.server.model.SessionStatus;
import io.casehub.api.model.WorkResult;
import io.casehub.api.model.WorkStatus;
import io.casehub.api.spi.WorkerStatusListener;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import java.io.IOException;
import java.util.Map;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ClaudonyWorkerStatusListener implements WorkerStatusListener {

    private static final Logger LOG = Logger.getLogger(ClaudonyWorkerStatusListener.class);
    static final String SESSION_PREFIX = "claudony-worker-";

    private final SessionRegistry registry;
    private final TmuxService tmux;
    private final Event<Object> events;
    private final WorkerSessionMapping sessionMapping;
    private final TenantContext tenantContext;

    @Inject
    public ClaudonyWorkerStatusListener(SessionRegistry registry, TmuxService tmux,
                                         Event<Object> events, WorkerSessionMapping sessionMapping,
                                         TenantContext tenantContext) {
        this.registry = registry;
        this.tmux = tmux;
        this.events = events;
        this.sessionMapping = sessionMapping;
        this.tenantContext = tenantContext;
    }

    @Override
    public void onWorkerStarted(String roleName, Map<String, String> sessionMeta) {
        String caseId = sessionMeta != null ? sessionMeta.get("caseId") : null;
        String sessionId = caseId != null
                ? sessionMapping.findByCase(caseId, roleName)
                        .orElseGet(() -> sessionMapping.findByRole(roleName).orElse(null))
                : sessionMapping.findByRole(roleName).orElse(null);
        if (sessionId != null) {
            registry.updateStatus(sessionId, SessionStatus.ACTIVE);
        }
        if (caseId != null) {
            String tenancyId = sessionId != null
                    ? registry.findUnscoped(sessionId).map(s -> s.tenancyId()).orElse(TenancyConstants.DEFAULT_TENANT_ID)
                    : TenancyConstants.DEFAULT_TENANT_ID;
            events.fire(new WorkerCaseLifecycleEvent(caseId, tenancyId));
        }
        LOG.debugf("Worker started: role=%s sessionId=%s", roleName, sessionId);
    }

    @Override
    public void onWorkerCompleted(String roleName, WorkResult result) {
        String sessionId = result.caseId() != null
                ? sessionMapping.findByCase(result.caseId().toString(), roleName)
                        .orElseGet(() -> sessionMapping.findByRole(roleName).orElse(null))
                : sessionMapping.findByRole(roleName).orElse(null);
        if (sessionId == null) {
            LOG.warnf("No session found for worker role: %s", roleName);
            return;
        }
        String tenancyId = registry.findUnscoped(sessionId).map(s -> s.tenancyId()).orElse(TenancyConstants.DEFAULT_TENANT_ID);
        if (result.status() == WorkStatus.FAULTED) {
            try {
                tmux.killSession(SESSION_PREFIX + sessionId);
            } catch (IOException | InterruptedException e) {
                LOG.warnf("Could not kill faulted session for worker %s (session %s): %s",
                        roleName, sessionId, e.getMessage());
            }
            registry.remove(sessionId);
            sessionMapping.remove(roleName);
        } else {
            registry.findUnscoped(sessionId).ifPresent(session ->
                    registry.updateStatus(sessionId, SessionStatus.IDLE));
        }
        if (result.caseId() != null) {
            events.fire(new WorkerCaseLifecycleEvent(result.caseId().toString(), tenancyId));
        }
        LOG.debugf("Worker completed: role=%s sessionId=%s status=%s caseId=%s",
                roleName, sessionId, result.status(), result.caseId());
    }

    @Override
    public void onWorkerStalled(String workerId) {
        LOG.warnf("Worker stalled: %s", workerId);
        events.fire(new WorkerStalledEvent(workerId));
        sessionMapping.findByRole(workerId)
                .flatMap(registry::findUnscoped)
                .ifPresent(session -> session.caseId()
                        .ifPresent(caseId -> events.fire(new WorkerCaseLifecycleEvent(caseId, session.tenancyId()))));
    }

    public record WorkerStalledEvent(String workerId) {}
}
```

- [ ] **Step 2: Fix ClaudonyWorkerStatusListenerTest**

Update the test constructor call to include `TenantContext`. Add `() -> TenancyConstants.DEFAULT_TENANT_ID` as the new last parameter. Update any event assertions to expect the two-arg `WorkerCaseLifecycleEvent`.

- [ ] **Step 3: Run casehub tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`

Expected: PASS

- [ ] **Step 4: Commit**

```
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerStatusListener.java \
  casehub/src/test/
git commit -m "feat(#121): ClaudonyWorkerStatusListener uses findUnscoped() with tenancyId on events

All 3 methods use findUnscoped() — engine CDI observer context has
no request scope. Events carry tenancyId from session (fallback to
DEFAULT_TENANT_ID when session not found).

Refs casehubio/claudony#121"
```

---

### Task 6: casehub module — ClaudonyReactiveCaseChannelProvider cache key + event tenancyId

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`

**Interfaces:**
- Consumes: `TenantContext.currentTenantId()` (Task 1), `CaseChannelCreatedEvent(channelId, channelName, tenancyId)` (Task 3)
- Produces: Updated cache key, event fire site with tenancyId

- [ ] **Step 1: Update ClaudonyReactiveCaseChannelProvider**

**a)** Add `TenantContext` field and constructor injection. Add to both the CDI constructor and the package-private test constructor.

**b)** Change cache key from `UUID` to record:

From:
```java
private final ConcurrentHashMap<UUID, Uni<Map<String, CaseChannel>>> layoutCache = new ConcurrentHashMap<>();
```
To:
```java
private record CacheKey(String tenancyId, UUID caseId) {}
private final ConcurrentHashMap<CacheKey, Uni<Map<String, CaseChannel>>> layoutCache = new ConcurrentHashMap<>();
```

**c)** Update `openChannel()` — cache key construction:

From:
```java
return layoutCache.computeIfAbsent(caseId,
        id -> initializeLayout(id)
                .onFailure().invoke(err -> layoutCache.remove(id))
```
To:
```java
var key = new CacheKey(tenantContext.currentTenantId(), caseId);
return layoutCache.computeIfAbsent(key,
        k -> initializeLayout(caseId)
                .onFailure().invoke(err -> layoutCache.remove(k))
```

**d)** Update `createQhorusChannel()` event fire — add tenancyId:

From:
```java
channelCreatedEvent.fire(
        new io.casehub.claudony.server.CaseChannelCreatedEvent(detail.id, detail.name));
```
To:
```java
channelCreatedEvent.fire(
        new io.casehub.claudony.server.CaseChannelCreatedEvent(detail.id, detail.name, tenantContext.currentTenantId()));
```

- [ ] **Step 2: Fix casehub tests**

Update `ClaudonyReactiveCaseChannelProviderTest` — pass a `TenantContext` (lambda or mock) to the test constructor. Update event assertions to expect 3-arg `CaseChannelCreatedEvent`.

- [ ] **Step 3: Run casehub tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`

Expected: PASS

- [ ] **Step 4: Commit**

```
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java \
  casehub/src/test/
git commit -m "feat(#121): tenant-scoped cache key and event tenancyId in channel provider

Layout cache keyed by (tenancyId, caseId).
CaseChannelCreatedEvent carries tenancyId from TenantContext.

Refs casehubio/claudony#121"
```

---

### Task 7: app module — ServerStartup bootstrap, CasehubStartupService, SessionResource, and all app test migrations

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/ServerStartup.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/SessionResource.java`
- Modify: all app test files that construct `Session` (15 files) or observe the changed CDI events

**Interfaces:**
- Consumes: `Session(... tenancyId)` (Task 2), `SessionRegistry.allUnscoped()` (Task 2), `SessionRegistry.existsByName()` (Task 2), `TenantContext.currentTenantId()` (Task 1), `WorkerCaseLifecycleEvent(caseId, tenancyId)` (Task 3), `CaseChannelCreatedEvent(channelId, channelName, tenancyId)` (Task 3)

- [ ] **Step 1: Update ServerStartup.bootstrapRegistry()**

In `app/src/main/java/io/casehub/claudony/server/ServerStartup.java`:

Add import: `import io.casehub.platform.api.identity.TenancyConstants;`

In `bootstrapRegistry()`, after the `roleName` read (line 84), add:

```java
Optional<String> tenancyId = Optional.empty();
try {
    tenancyId = tmux.getSessionOption(name, "@casehub_tenant_id");
} catch (IOException | InterruptedException e) {
    LOG.warnf("Could not read tenant option for session %s: %s", name, e.getMessage());
}
```

Update Session construction (line 91-94) — add tenancyId:

From:
```java
registry.register(new Session(
        UUID.randomUUID().toString(), name,
        "unknown", config.claudeCommand(),
        SessionStatus.IDLE, now, now, Optional.empty(), caseId, roleName));
```
To:
```java
registry.register(new Session(
        UUID.randomUUID().toString(), name,
        "unknown", config.claudeCommand(),
        SessionStatus.IDLE, now, now, Optional.empty(), caseId, roleName,
        tenancyId.orElse(TenancyConstants.DEFAULT_TENANT_ID)));
```

- [ ] **Step 2: Update CasehubStartupService.bootstrapWatchers()**

In `app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java`, line 39:

From:
```java
for (var session : registry.all()) {
```
To:
```java
for (var session : registry.allUnscoped()) {
```

- [ ] **Step 3: Update SessionResource**

In `app/src/main/java/io/casehub/claudony/server/SessionResource.java`:

**a)** Add `TenantContext` injection:

```java
@Inject TenantContext tenantContext;
```

Add import: `import io.casehub.claudony.server.TenantContext;` and `import io.casehub.platform.api.identity.TenancyConstants;`

**b)** Update `create()` — duplicate check (line 194):

From:
```java
var existing = registry.all().stream()
        .filter(s -> s.name().equals(name))
        .findFirst();
```
To:
```java
var existingByName = registry.existsByName(name);
```

And adjust the conditional:
```java
if (existingByName) {
    if (!overwrite) {
        return Response.status(409)
                .entity("{\"error\":\"Session '" + name + "' already exists\"}")
                .build();
    }
    var existingSession = registry.allUnscoped().stream()
            .filter(s -> s.name().equals(name))
            .findFirst().orElse(null);
    if (existingSession != null) {
        try {
            tmux.killSession(name);
            registry.remove(existingSession.id());
            LOG.infof("Overwrote existing session '%s'", name);
        } catch (IOException | InterruptedException e) {
            LOG.warnf("Could not clean up existing session '%s': %s", name, e.getMessage());
        }
    }
}
```

**c)** Update `create()` — Session construction (line 219):

From:
```java
var session = new Session(id, name, workingDir, command, SessionStatus.IDLE, now, now,
                          Optional.ofNullable(req.expiryPolicy()), Optional.empty(), Optional.empty());
```
To:
```java
var session = new Session(id, name, workingDir, command, SessionStatus.IDLE, now, now,
                          Optional.ofNullable(req.expiryPolicy()), Optional.empty(), Optional.empty(),
                          tenantContext.currentTenantId());
```

**d)** Update `rename()` — duplicate check (line 258):

From:
```java
boolean duplicate = registry.all().stream().anyMatch(s -> s.name().equals(newTmuxName));
```
To:
```java
boolean duplicate = registry.existsByName(newTmuxName);
```

**e)** Update `rename()` — Session construction (line 272-274):

From:
```java
var renamed = new Session(id, newTmuxName, session.workingDir(),
        session.command(), session.status(), session.createdAt(), Instant.now(),
        session.expiryPolicy(), session.caseId(), session.roleName());
```
To:
```java
var renamed = new Session(id, newTmuxName, session.workingDir(),
        session.command(), session.status(), session.createdAt(), Instant.now(),
        session.expiryPolicy(), session.caseId(), session.roleName(), session.tenancyId());
```

- [ ] **Step 4: Fix all app test compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl app --also-make`

Fix every compile error by adding `TenancyConstants.DEFAULT_TENANT_ID` as the last argument to every `new Session(...)` call across all 15 app test files. Also fix:
- `CasehubStartupServiceTest.java` — `new SessionRegistry()` → `new SessionRegistry(() -> TenancyConstants.DEFAULT_TENANT_ID)`
- Event construction sites that reference `WorkerCaseLifecycleEvent` or `CaseChannelCreatedEvent` — add tenancyId argument
- `CaseEventBroadcasterTest.java` — update `WorkerCaseLifecycleEvent` construction
- `ChannelFleetBroadcasterTest.java` — update `CaseChannelCreatedEvent` construction (add `TenancyConstants.DEFAULT_TENANT_ID` third arg)
- `ChannelInitialisedObserverTest.java` — if it fires `CaseChannelCreatedEvent`, add tenancyId

This is a mechanical migration. Every `new Session(` needs `TenancyConstants.DEFAULT_TENANT_ID` appended. Every event construction needs its tenancyId arg.

- [ ] **Step 5: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

Expected: PASS — all 576+ tests pass (test count will increase with new SessionRegistryTenantFilterTest and DefaultTenantContextTest).

- [ ] **Step 6: Commit**

```
git add app/src/main/java/io/casehub/claudony/server/ServerStartup.java \
  app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java \
  app/src/main/java/io/casehub/claudony/server/SessionResource.java \
  app/src/test/
git commit -m "feat(#121): app module tenancy — bootstrap, startup service, session resource, all tests

ServerStartup reads @casehub_tenant_id from tmux on bootstrap.
CasehubStartupService uses allUnscoped() for watcher recovery.
SessionResource uses existsByName() for tmux namespace checks,
tenantContext.currentTenantId() for new Session construction.
All 15 app test files updated with tenancyId arg.

Refs casehubio/claudony#121"
```

---

### Task 8: Final verification and CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` (test count update)

- [ ] **Step 1: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

Expected: PASS — all tests green. Count the new total.

- [ ] **Step 2: Update CLAUDE.md test baseline**

Update the `## Test Count and Status` section with the new test count and description of what changed.

- [ ] **Step 3: Commit**

```
git add CLAUDE.md
git commit -m "docs(#121): update test baseline after multi-tenancy foundation

Refs casehubio/claudony#121"
```
