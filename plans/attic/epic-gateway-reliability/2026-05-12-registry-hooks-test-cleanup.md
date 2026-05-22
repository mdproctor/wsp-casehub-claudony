# Registry-Hooks Test Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a negative test proving `register()` doesn't trigger SSE pushes, and remove a cargo-culted `@TestSecurity` annotation.

**Architecture:** Two edits to existing test files only. No production code changes. Both changes are independent and committed together since they're both minor cleanup items under the same issue.

**Tech Stack:** Quarkus 3.32.2, JUnit 5, AssertJ, Java 21 on JDK 26

---

## File Map

| Action | Path |
|--------|------|
| Modify | `app/src/test/java/io/casehub/claudony/server/RegistryHooksStrategyIntegrationTest.java` |
| Modify | `app/src/test/java/io/casehub/claudony/server/HybridDefaultConfigTest.java` |

---

## Key Facts

**`SessionRegistry.register()`** does not call `notifyListeners()` — only `updateStatus()` and `remove()` do. So subscribing to the broadcaster, then calling `register()`, should produce exactly 1 received item: the initial snapshot from `broadcaster.subscribe()`. No second push should arrive.

**`Thread.sleep(200)`** is the established pattern for negative timing assertions in this codebase — see `EventsOnlyStrategyTest.onLifecycleEvent_doesNotPushToOtherCase`. It gives the system 200ms to deliver a push if one were coming, making the "no push" assertion meaningful.

**`@TestSecurity`** on `HybridDefaultConfigTest` has zero effect — the test calls `broadcaster.strategyType()` directly, never touching an HTTP endpoint. Removing it also removes the now-unused import.

**Session constructor** (for reference):
```java
new Session(id, name, workingDir, command, status, createdAt, lastActive,
            expiryPolicy, caseId, roleName)
// caseSession helper already exists in RegistryHooksStrategyIntegrationTest
```

---

## Task 1: Add `register_doesNotTriggerSSEPush`

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/server/RegistryHooksStrategyIntegrationTest.java`

- [ ] **Step 1: Add the test method** after `selectedStrategy_isRegistryHooks()` (line 51):

```java
@Test
void register_doesNotTriggerSSEPush() throws Exception {
    var received = new CopyOnWriteArrayList<String>();

    broadcaster.subscribe("rh-case-3", () -> "data: snap\n\n")
            .subscribe().with(received::add);

    registry.register(caseSession("rh-s3", "rh-case-3"));

    Thread.sleep(200);
    assertThat(received).hasSize(1); // only the initial snapshot — register() does not notify
}
```

Note: `CopyOnWriteArrayList` is already imported. No new imports needed.

- [ ] **Step 2: Run the new test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest="RegistryHooksStrategyIntegrationTest#register_doesNotTriggerSSEPush" -q 2>&1 | tail -5
```

Expected: `Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 3: Run all tests in the class**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=RegistryHooksStrategyIntegrationTest -q 2>&1 | tail -5
```

Expected: `Tests run: 4, Failures: 0, Errors: 0, Skipped: 0`

---

## Task 2: Remove `@TestSecurity` from `HybridDefaultConfigTest`

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/server/HybridDefaultConfigTest.java`

- [ ] **Step 1: Remove the annotation and its import**

Delete line 6: `import io.quarkus.test.security.TestSecurity;`  
Delete line 16: `@TestSecurity(user = "test", roles = "user")`

The file should look like this after the edit:

```java
package io.casehub.claudony.server;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(HybridDefaultConfigTest.HybridProfile.class)
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

---

## Task 3: Full Suite, Review, Commit, Push, Doc Sync

- [ ] **Step 1: Run the full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS` with 479 tests (478 + 1 new), 0 failures.

- [ ] **Step 2: Invoke `superpowers:requesting-code-review`** — any finding Minor or above that isn't fixed must become a GitHub issue before sign-off.

- [ ] **Step 3: Commit** (after review approved, from project repo `/Users/mdproctor/claude/casehub/claudony`)

```bash
git add app/src/test/java/io/casehub/claudony/server/RegistryHooksStrategyIntegrationTest.java \
        app/src/test/java/io/casehub/claudony/server/HybridDefaultConfigTest.java
git commit -m "test(sse): verify register() does not trigger SSE push; remove cargo-culted @TestSecurity

- register_doesNotTriggerSSEPush: documents that SessionRegistry.register()
  intentionally does not fire the registry-hooks change listener
- Remove @TestSecurity from HybridDefaultConfigTest — no HTTP endpoint involved,
  annotation had no effect

Closes #110"
```

- [ ] **Step 4: Push** (from project repo)

```bash
git push origin main
```

- [ ] **Step 5: Invoke `implementation-doc-sync`** — update test count in CLAUDE.md (478 → 479).
