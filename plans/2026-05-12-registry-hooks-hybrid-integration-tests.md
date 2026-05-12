# Registry-Hooks and Hybrid-Default Integration Tests Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add two `@QuarkusTest` integration test classes that verify `CaseEventBroadcaster` wires `RegistryHooksStrategy` correctly (registry mutations trigger SSE pushes) and selects `HybridStrategy` when configured as `hybrid`.

**Architecture:** Two separate test classes, each with an inner `@QuarkusTestProfile` that overrides `claudony.case-worker-update`. Each profile forces a fresh Quarkus app context. All production code exists — these are pure test additions with no implementation changes. Follows the `MeshParticipationSilentProfileTest` pattern exactly.

**Tech Stack:** Quarkus 3.32.2, JUnit 5, AssertJ, SmallRye Mutiny, `@QuarkusTestProfile`, Java 21 on JDK 26

---

## File Map

| Action | Path | Purpose |
|--------|------|---------|
| Create | `app/src/test/java/io/casehub/claudony/server/RegistryHooksStrategyIntegrationTest.java` | 3 tests: strategy type, updateStatus push, remove push |
| Create | `app/src/test/java/io/casehub/claudony/server/HybridDefaultConfigTest.java` | 1 test: strategy type is hybrid when config is hybrid |

No production files are modified.

---

## Key Facts Before You Start

**`CaseEventBroadcaster.strategyType()`** is package-private — tests in package `io.casehub.claudony.server` can call it directly.

**`SessionRegistry.notifyListeners(id)`** is called by `updateStatus()` and `remove()` before the session is removed. For `remove()`, the session is still in the map when listeners fire — so `caseId` is resolvable.

**`@AfterEach` cleanup** calls `registry.remove()` on all remaining sessions. With registry-hooks active, each cleanup removal fires the change listener and pushes to any active SSE subscribers. This is harmless — the latch is already satisfied by the time cleanup runs, and the extra push just increments the received list beyond the asserted size.

**Test `application.properties`** sets `%test.claudony.case-worker-update=events-only`. The `@QuarkusTestProfile` config overrides take precedence over this, so both new profiles work as expected.

**Session constructor:**
```java
new Session(
    String id,
    String name,
    String workingDir,
    String command,
    SessionStatus status,    // ACTIVE | WAITING | IDLE
    Instant createdAt,
    Instant lastActive,
    Optional<String> expiryPolicy,
    Optional<String> caseId,
    Optional<String> roleName
)
```

---

## Task 1: `RegistryHooksStrategyIntegrationTest`

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/server/RegistryHooksStrategyIntegrationTest.java`

- [ ] **Step 1: Create the test class**

```java
package io.casehub.claudony.server;

import io.casehub.claudony.server.model.Session;
import io.casehub.claudony.server.model.SessionStatus;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(RegistryHooksStrategyIntegrationTest.RegistryHooksProfile.class)
@TestSecurity(user = "test", roles = "user")
class RegistryHooksStrategyIntegrationTest {

    public static class RegistryHooksProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("claudony.case-worker-update", "registry-hooks");
        }
    }

    @Inject SessionRegistry registry;
    @Inject CaseEventBroadcaster broadcaster;

    @AfterEach
    void cleanup() {
        registry.all().stream().map(Session::id).toList().forEach(registry::remove);
    }

    private Session caseSession(String id, String caseId) {
        return new Session(id, "name-" + id, "/tmp", "cmd", SessionStatus.IDLE,
                Instant.now(), Instant.now(), Optional.empty(),
                Optional.of(caseId), Optional.of("worker"));
    }

    @Test
    void selectedStrategy_isRegistryHooks() {
        assertThat(broadcaster.strategyType()).isEqualTo("registry-hooks");
    }

    @Test
    void updateStatus_triggersSSEPush() throws Exception {
        registry.register(caseSession("rh-s1", "rh-case-1"));

        var received = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(2); // initial snapshot + registry-triggered push

        broadcaster.subscribe("rh-case-1", () -> "data: snap\n\n")
                .subscribe().with(e -> { received.add(e); latch.countDown(); });

        registry.updateStatus("rh-s1", SessionStatus.ACTIVE);

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received).hasSize(2);
    }

    @Test
    void remove_triggersSSEPush() throws Exception {
        registry.register(caseSession("rh-s2", "rh-case-2"));

        var received = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(2); // initial snapshot + registry-triggered push

        broadcaster.subscribe("rh-case-2", () -> "data: snap\n\n")
                .subscribe().with(e -> { received.add(e); latch.countDown(); });

        registry.remove("rh-s2");

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received).hasSize(2);
    }
}
```

- [ ] **Step 2: Run the test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=RegistryHooksStrategyIntegrationTest -q 2>&1 | tail -5
```

Expected: `Tests run: 3, Failures: 0, Errors: 0, Skipped: 0`

If `latch.await` times out, the change listener wiring in `CaseEventBroadcaster.init()` is not firing — check that the profile config override is taking effect by verifying `selectedStrategy_isRegistryHooks` passes first.

---

## Task 2: `HybridDefaultConfigTest`

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/server/HybridDefaultConfigTest.java`

- [ ] **Step 1: Create the test class**

```java
package io.casehub.claudony.server;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(HybridDefaultConfigTest.HybridProfile.class)
@TestSecurity(user = "test", roles = "user")
class HybridDefaultConfigTest {

    public static class HybridProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("claudony.case-worker-update", "hybrid");
        }
    }

    @Inject CaseEventBroadcaster broadcaster;

    @Test
    void selectedStrategy_isHybrid_whenConfiguredAsHybrid() {
        assertThat(broadcaster.strategyType()).isEqualTo("hybrid");
    }
}
```

- [ ] **Step 2: Run the test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=HybridDefaultConfigTest -q 2>&1 | tail -5
```

Expected: `Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`

Note: `HybridStrategy` starts a background heartbeat thread. It is not cancelled between tests since there is only one test in this class and the app context is torn down after. No interference.

---

## Task 3: Full Suite Verification and Commit

- [ ] **Step 1: Run the full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS` with 478 tests passing (474 + 4 new), 0 failures.

- [ ] **Step 2: Invoke `superpowers:requesting-code-review`** before committing.

- [ ] **Step 3: Commit** (after review is approved)

```bash
git add app/src/test/java/io/casehub/claudony/server/RegistryHooksStrategyIntegrationTest.java \
        app/src/test/java/io/casehub/claudony/server/HybridDefaultConfigTest.java
git commit -m "test(sse): add integration tests for registry-hooks wiring and hybrid config default

- RegistryHooksStrategyIntegrationTest: verifies registry.updateStatus() and
  registry.remove() trigger SSE pushes via change listener wiring
- HybridDefaultConfigTest: verifies broadcaster selects HybridStrategy when
  claudony.case-worker-update=hybrid

Closes #108"
```

- [ ] **Step 4: Push**

```bash
git push origin main
```

- [ ] **Step 5: Invoke `implementation-doc-sync`** to check if CLAUDE.md or DESIGN.md need updating.
