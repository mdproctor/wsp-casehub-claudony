# Agent Pool Test Utilities Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Each task follows TDD and uses
> ide-tooling for structural editing. Steps use checkbox (`- [ ]`) syntax.

**Focal issue:** #230 — agent pool test utilities for downstream projects

**Goal:** Create a `claudony-testing` Maven module with `InMemorySessionOperations`, `TestPool`, and `TestPoolBuilder` so downstream projects can test agent pools without tmux.

**Architecture:** New Maven module at `testing/` following the `casehub-engine-testing` pattern. Plain Java — no Quarkus dependency. Validated by migrating `AgentSessionManagerTest` to use the new utilities.

**Tech Stack:** Java 21, JUnit 5, AssertJ

## Global Constraints

- Package: `io.casehub.claudony.testing.fleet`
- No Quarkus/CDI dependency — plain Java classes
- Thread-safe: `ConcurrentHashMap` + `AtomicInteger` for all mutable state
- Suspend model: sessions survive suspension (moved to suspended map, not removed) — mirrors #239 tmux-as-persistence design
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing`

---

## Batch 1: Testing Module and InMemorySessionOperations

### Task 1: Create the testing module with InMemorySessionOperations

**Files:**
- Create: `testing/pom.xml`
- Create: `testing/src/main/java/io/casehub/claudony/testing/fleet/InMemorySessionOperations.java`
- Create: `testing/src/test/java/io/casehub/claudony/testing/fleet/InMemorySessionOperationsTest.java`
- Modify: `pom.xml` (add `<module>testing</module>`)

**Interfaces:**
- Consumes: `io.casehub.claudony.casehub.fleet.SessionOperations` (SPI interface from casehub module)
- Produces: `InMemorySessionOperations` — all 6 SPI methods + inspection API (`createCount()`, `suspendCount()`, `resumeCount()`, `destroyCount()`, `activeSessions()`, `suspendedSessions()`, `isActive(String)`, `isSuspended(String)`, `reset()`)

**Steps:**

- [ ] **Step 1: Create `testing/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-claudony-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-claudony-testing</artifactId>
    <name>Claudony :: Testing</name>
    <description>In-memory agent pool test utilities — no tmux required</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-claudony-casehub</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>${assertj.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${compiler-plugin.version}</version>
                <configuration>
                    <release>${maven.compiler.release}</release>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${surefire-plugin.version}</version>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add `<module>testing</module>` to parent POM**

In `pom.xml`, add `testing` after `casehub` and before `app`:

```xml
<modules>
    <module>core</module>
    <module>casehub</module>
    <module>testing</module>
    <module>app</module>
</modules>
```

Order matters: `testing` depends on `casehub`, and `app` may later depend on `testing`.

- [ ] **Step 3: Write the failing tests for InMemorySessionOperations**

```java
package io.casehub.claudony.testing.fleet;

import io.casehub.claudony.casehub.fleet.SessionOperations;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class InMemorySessionOperationsTest {

    private InMemorySessionOperations ops;

    @BeforeEach
    void setUp() {
        ops = new InMemorySessionOperations();
    }

    @Test
    void implementsSessionOperations() {
        assertThat(ops).isInstanceOf(SessionOperations.class);
    }

    @Test
    void create_returnsUniqueSessionIds() {
        String s1 = ops.create("worker-1", "/workspace/a");
        String s2 = ops.create("worker-2", "/workspace/b");
        assertThat(s1).isNotEqualTo(s2);
        assertThat(s1).startsWith("mem-");
        assertThat(ops.createCount()).isEqualTo(2);
    }

    @Test
    void create_storesConversationId() {
        String sessionId = ops.create("worker-1", "/workspace/a");
        assertThat(ops.conversationId(sessionId)).isNotNull();
    }

    @Test
    void create_withCommand_incrementsCounter() {
        String sessionId = ops.create("worker-1", "/workspace/a", "claude --model opus");
        assertThat(sessionId).startsWith("mem-");
        assertThat(ops.createCount()).isEqualTo(1);
    }

    @Test
    void suspend_movesToSuspendedMap() {
        String sessionId = ops.create("worker-1", "/workspace/a");
        assertThat(ops.isActive(sessionId)).isTrue();

        ops.suspend(sessionId);
        assertThat(ops.isActive(sessionId)).isFalse();
        assertThat(ops.isSuspended(sessionId)).isTrue();
        assertThat(ops.suspendCount()).isEqualTo(1);
    }

    @Test
    void resume_movesBackToActive() {
        String sessionId = ops.create("worker-1", "/workspace/a");
        ops.suspend(sessionId);

        ops.resume(sessionId, ops.conversationId(sessionId), "/workspace/a");
        assertThat(ops.isActive(sessionId)).isTrue();
        assertThat(ops.isSuspended(sessionId)).isFalse();
        assertThat(ops.resumeCount()).isEqualTo(1);
    }

    @Test
    void destroy_removesCompletely() {
        String sessionId = ops.create("worker-1", "/workspace/a");
        ops.destroy(sessionId);
        assertThat(ops.isActive(sessionId)).isFalse();
        assertThat(ops.isSuspended(sessionId)).isFalse();
        assertThat(ops.conversationId(sessionId)).isNull();
        assertThat(ops.destroyCount()).isEqualTo(1);
    }

    @Test
    void destroy_suspendedSession() {
        String sessionId = ops.create("worker-1", "/workspace/a");
        ops.suspend(sessionId);
        ops.destroy(sessionId);
        assertThat(ops.isSuspended(sessionId)).isFalse();
        assertThat(ops.destroyCount()).isEqualTo(1);
    }

    @Test
    void memoryBytes_returnsConfigurableDefault() {
        String sessionId = ops.create("worker-1", "/workspace/a");
        assertThat(ops.memoryBytes(sessionId)).isEqualTo(100 * 1024 * 1024L);

        ops.setDefaultMemoryBytes(50 * 1024 * 1024L);
        assertThat(ops.memoryBytes(sessionId)).isEqualTo(50 * 1024 * 1024L);
    }

    @Test
    void activeSessions_returnsCopy() {
        ops.create("worker-1", "/workspace/a");
        ops.create("worker-2", "/workspace/b");
        assertThat(ops.activeSessions()).hasSize(2);
    }

    @Test
    void suspendedSessions_returnsCopy() {
        String s1 = ops.create("worker-1", "/workspace/a");
        ops.suspend(s1);
        assertThat(ops.suspendedSessions()).hasSize(1);
        assertThat(ops.activeSessions()).isEmpty();
    }

    @Test
    void reset_clearsEverything() {
        ops.create("worker-1", "/workspace/a");
        ops.create("worker-2", "/workspace/b");
        ops.reset();
        assertThat(ops.activeSessions()).isEmpty();
        assertThat(ops.createCount()).isZero();
        assertThat(ops.suspendCount()).isZero();
    }

    @Test
    void conversationId_returnsNullForUnknown() {
        assertThat(ops.conversationId("nonexistent")).isNull();
    }

    @Test
    void threadSafety_concurrentCreates() throws Exception {
        var latch = new java.util.concurrent.CountDownLatch(1);
        var futures = new java.util.ArrayList<java.util.concurrent.Future<String>>();
        var executor = java.util.concurrent.Executors.newFixedThreadPool(4);

        for (int i = 0; i < 20; i++) {
            int idx = i;
            futures.add(executor.submit(() -> {
                latch.await();
                return ops.create("worker-" + idx, "/workspace/" + idx);
            }));
        }
        latch.countDown();
        for (var f : futures) f.get();
        executor.shutdown();

        assertThat(ops.createCount()).isEqualTo(20);
        assertThat(ops.activeSessions()).hasSize(20);
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dsurefire.failIfNoSpecifiedTests=false -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: FAIL — `InMemorySessionOperations` class not found

- [ ] **Step 5: Implement InMemorySessionOperations**

```java
package io.casehub.claudony.testing.fleet;

import io.casehub.claudony.casehub.fleet.SessionOperations;

import java.util.Collections;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class InMemorySessionOperations implements SessionOperations {

    private record SessionRecord(String identity, String workingDir) {}

    private final ConcurrentHashMap<String, SessionRecord> active = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, SessionRecord> suspended = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> conversationIds = new ConcurrentHashMap<>();

    private final AtomicInteger counter = new AtomicInteger();
    private final AtomicInteger createCounter = new AtomicInteger();
    private final AtomicInteger suspendCounter = new AtomicInteger();
    private final AtomicInteger resumeCounter = new AtomicInteger();
    private final AtomicInteger destroyCounter = new AtomicInteger();

    private volatile long defaultMemoryBytes = 100L * 1024 * 1024;

    @Override
    public String create(String identity, String workingDir) {
        String sessionId = "mem-" + counter.incrementAndGet();
        String conversationId = UUID.randomUUID().toString();
        active.put(sessionId, new SessionRecord(identity, workingDir));
        conversationIds.put(sessionId, conversationId);
        createCounter.incrementAndGet();
        return sessionId;
    }

    @Override
    public String create(String identity, String workingDir, String command) {
        return create(identity, workingDir);
    }

    @Override
    public String conversationId(String sessionId) {
        return conversationIds.get(sessionId);
    }

    @Override
    public void suspend(String sessionId) {
        var record = active.remove(sessionId);
        if (record != null) {
            suspended.put(sessionId, record);
        }
        suspendCounter.incrementAndGet();
    }

    @Override
    public void resume(String sessionId, String conversationId, String workingDir) {
        var record = suspended.remove(sessionId);
        if (record != null) {
            active.put(sessionId, record);
        }
        resumeCounter.incrementAndGet();
    }

    @Override
    public void destroy(String sessionId) {
        active.remove(sessionId);
        suspended.remove(sessionId);
        conversationIds.remove(sessionId);
        destroyCounter.incrementAndGet();
    }

    @Override
    public long memoryBytes(String sessionId) {
        return defaultMemoryBytes;
    }

    public void setDefaultMemoryBytes(long bytes) {
        this.defaultMemoryBytes = bytes;
    }

    public int createCount() { return createCounter.get(); }
    public int suspendCount() { return suspendCounter.get(); }
    public int resumeCount() { return resumeCounter.get(); }
    public int destroyCount() { return destroyCounter.get(); }

    public Set<String> activeSessions() {
        return Collections.unmodifiableSet(active.keySet());
    }

    public Set<String> suspendedSessions() {
        return Collections.unmodifiableSet(suspended.keySet());
    }

    public boolean isActive(String sessionId) {
        return active.containsKey(sessionId);
    }

    public boolean isSuspended(String sessionId) {
        return suspended.containsKey(sessionId);
    }

    public void reset() {
        active.clear();
        suspended.clear();
        conversationIds.clear();
        createCounter.set(0);
        suspendCounter.set(0);
        resumeCounter.set(0);
        destroyCounter.set(0);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: PASS (all 14 tests)

- [ ] **Step 7: Commit**

```bash
git add testing/ pom.xml
git commit -m "feat(#230): InMemorySessionOperations — in-memory SessionOperations for testing

Sessions survive suspension (moved to suspended map, not removed) mirroring
the tmux-as-persistence model from #239. Thread-safe, inspectable counters.

Refs #230"
```

### Task 2: Add TestPool and TestPoolBuilder

**Files:**
- Create: `testing/src/main/java/io/casehub/claudony/testing/fleet/TestPool.java`
- Create: `testing/src/main/java/io/casehub/claudony/testing/fleet/TestPoolBuilder.java`
- Create: `testing/src/test/java/io/casehub/claudony/testing/fleet/TestPoolBuilderTest.java`

**Interfaces:**
- Consumes: `InMemorySessionOperations` (from Task 1), `AgentSessionManager`, `AgentSessionManagerConfig` (from casehub module)
- Produces: `TestPool(AgentSessionManager manager, InMemorySessionOperations ops)` record, `TestPoolBuilder.create().minActive(N).maxActive(N).build()` → `TestPool`

**Steps:**

- [ ] **Step 1: Write the failing tests for TestPoolBuilder**

```java
package io.casehub.claudony.testing.fleet;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TestPoolBuilderTest {

    @Test
    void build_returnsTestPoolWithManagerAndOps() {
        var pool = TestPoolBuilder.create().build();
        assertThat(pool.manager()).isNotNull();
        assertThat(pool.ops()).isNotNull();
    }

    @Test
    void build_usesDefaultConfig() {
        var pool = TestPoolBuilder.create().build();
        var status = pool.manager().status();
        assertThat(status.min()).isZero();
        assertThat(status.max()).isEqualTo(10);
    }

    @Test
    void build_customMinMax() {
        var pool = TestPoolBuilder.create()
                .minActive(2)
                .maxActive(5)
                .build();
        var status = pool.manager().status();
        assertThat(status.min()).isEqualTo(2);
        assertThat(status.max()).isEqualTo(5);
    }

    @Test
    void pool_managerUsesInMemoryOps() {
        var pool = TestPoolBuilder.create().maxActive(5).build();
        pool.manager().acquireSession("worker-1", "/workspace/a");
        assertThat(pool.ops().createCount()).isEqualTo(1);
        assertThat(pool.ops().activeSessions()).hasSize(1);
    }

    @Test
    void pool_fullLifecycle() {
        var pool = TestPoolBuilder.create().maxActive(2).build();

        var s1 = pool.manager().acquireSession("worker-1", "/workspace/a");
        var s2 = pool.manager().acquireSession("worker-2", "/workspace/b");
        assertThat(pool.manager().activeCount()).isEqualTo(2);

        pool.manager().acquireSession("worker-3", "/workspace/c");
        assertThat(pool.manager().activeCount()).isEqualTo(2);
        assertThat(pool.ops().suspendCount()).isEqualTo(1);
        assertThat(pool.ops().suspendedSessions()).hasSize(1);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: FAIL — `TestPool` and `TestPoolBuilder` not found

- [ ] **Step 3: Implement TestPool**

```java
package io.casehub.claudony.testing.fleet;

import io.casehub.claudony.casehub.fleet.AgentSessionManager;

public record TestPool(AgentSessionManager manager, InMemorySessionOperations ops) {}
```

- [ ] **Step 4: Implement TestPoolBuilder**

```java
package io.casehub.claudony.testing.fleet;

import io.casehub.claudony.casehub.fleet.AgentSessionManager;
import io.casehub.claudony.casehub.fleet.AgentSessionManagerConfig;

public final class TestPoolBuilder {

    private int minActive = 0;
    private int maxActive = 10;

    private TestPoolBuilder() {}

    public static TestPoolBuilder create() {
        return new TestPoolBuilder();
    }

    public TestPoolBuilder minActive(int minActive) {
        this.minActive = minActive;
        return this;
    }

    public TestPoolBuilder maxActive(int maxActive) {
        this.maxActive = maxActive;
        return this;
    }

    public TestPool build() {
        var ops = new InMemorySessionOperations();
        var config = new AgentSessionManagerConfig(minActive, maxActive);
        var manager = new AgentSessionManager(config, ops);
        return new TestPool(manager, ops);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: PASS (all 19 tests — 14 from Task 1 + 5 new)

- [ ] **Step 6: Commit**

```bash
git add testing/src/
git commit -m "feat(#230): TestPool record and TestPoolBuilder — fluent pool construction for tests

TestPoolBuilder.create().maxActive(5).build() → TestPool(manager, ops).
Downstream gets manager to drive and ops to inspect.

Refs #230"
```

## Batch 2: Validate by Migrating AgentSessionManagerTest

### Task 3: Migrate AgentSessionManagerTest to use claudony-testing

**Files:**
- Modify: `casehub/pom.xml` (add test dependency on `claudony-testing`)
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentSessionManagerTest.java`

**Interfaces:**
- Consumes: `InMemorySessionOperations`, `TestPoolBuilder`, `TestPool` (from Tasks 1-2)
- Produces: Validated testing module — all 22 existing tests pass with imported utilities

**Steps:**

- [ ] **Step 1: Add test dependency to casehub module**

In `casehub/pom.xml`, add:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-claudony-testing</artifactId>
    <version>${project.version}</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Migrate AgentSessionManagerTest**

Replace the inline anonymous `SessionOperations` with `InMemorySessionOperations` from the testing module. Replace the `AtomicInteger` counters and `ConcurrentHashMap` fields with `InMemorySessionOperations` inspection methods.

The key changes:
- Remove fields: `createCount`, `suspendCount`, `resumeCount`, `destroyCount`, `conversationIds`
- Add field: `InMemorySessionOperations ops`
- Replace `createManager(int, int)` helper: use `InMemorySessionOperations` + `AgentSessionManager` constructor directly (NOT TestPoolBuilder — some tests need custom memory return values, which TestPoolBuilder doesn't support)
- Replace all `createCount.get()` → `ops.createCount()`, etc.

The full migrated test class:

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.claudony.testing.fleet.InMemorySessionOperations;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AgentSessionManagerTest {

    private InMemorySessionOperations ops;
    private AgentSessionManager manager;

    @BeforeEach
    void setUp() {
        ops = new InMemorySessionOperations();
    }

    private AgentSessionManager createManager(int minActive, int maxActive) {
        return new AgentSessionManager(
                new AgentSessionManagerConfig(minActive, maxActive), ops);
    }

    @Test
    void acquireSession_createsNewWhenNoneExist() {
        manager = createManager(0, 5);
        var session = manager.acquireSession("reviewer", "/workspace/pr-42");
        assertThat(session).isNotNull();
        assertThat(session.identity()).isEqualTo("reviewer");
        assertThat(session.workingDir()).isEqualTo("/workspace/pr-42");
        assertThat(session.state()).isEqualTo(SessionState.ACTIVE);
        assertThat(manager.activeCount()).isEqualTo(1);
        assertThat(ops.createCount()).isEqualTo(1);
    }

    @Test
    void acquireSession_resumesSuspendedMatchingIdentity() {
        manager = createManager(0, 5);
        var session = manager.acquireSession("reviewer", "/workspace/pr-42");
        String originalId = session.instanceId();

        manager.suspendSession(originalId);
        assertThat(manager.activeCount()).isZero();
        assertThat(manager.suspendedCount()).isEqualTo(1);

        var resumed = manager.acquireSession("reviewer", "/workspace/pr-42");
        assertThat(resumed.instanceId()).isEqualTo(originalId);
        assertThat(resumed.state()).isEqualTo(SessionState.ACTIVE);
        assertThat(ops.resumeCount()).isEqualTo(1);
    }

    @Test
    void acquireSession_preservesConversationIdAcrossSuspendResume() {
        manager = createManager(0, 5);
        var session = manager.acquireSession("reviewer", "/workspace/pr-42");
        String originalConvId = session.conversationId();
        assertThat(originalConvId).isNotNull();

        manager.suspendSession(session.instanceId());
        var resumed = manager.acquireSession("reviewer", "/workspace/pr-42");
        assertThat(resumed.conversationId()).isEqualTo(originalConvId);
    }

    @Test
    void acquireSession_exclusivePolicy_throwsOnConflict() {
        manager = createManager(0, 5);
        manager.acquireSession("reviewer", "/workspace/pr-42");

        assertThatThrownBy(() -> manager.acquireSession("coder", "/workspace/pr-42",
                                                        null, WorkingDirPolicy.EXCLUSIVE))
                .isInstanceOf(WorkingDirConflictException.class)
                .hasMessageContaining("/workspace/pr-42")
                .hasMessageContaining("reviewer");
    }

    @Test
    void acquireSession_exclusivePolicy_allowsDifferentWorkingDirs() {
        manager = createManager(0, 5);
        manager.acquireSession("reviewer", "/workspace/pr-42");

        assertThatCode(() -> manager.acquireSession("coder", "/workspace/task-1",
                                                    null, WorkingDirPolicy.EXCLUSIVE))
                .doesNotThrowAnyException();
    }

    @Test
    void acquireSession_sharedReadPolicy_allowsSameWorkingDir() {
        manager = createManager(0, 5);
        manager.acquireSession("reviewer", "/workspace/pr-42",
                               null, WorkingDirPolicy.SHARED_READ);

        assertThatCode(() -> manager.acquireSession("coder", "/workspace/pr-42",
                                                    null, WorkingDirPolicy.SHARED_READ))
                .doesNotThrowAnyException();
        assertThat(manager.activeCount()).isEqualTo(2);
    }

    @Test
    void acquireSession_exclusivePolicy_allowsResumeOfOwnSuspended() {
        manager = createManager(0, 5);
        var session = manager.acquireSession("reviewer", "/workspace/pr-42",
                                             null, WorkingDirPolicy.EXCLUSIVE);
        manager.suspendSession(session.instanceId());

        var resumed = manager.acquireSession("reviewer", "/workspace/pr-42",
                                             null, WorkingDirPolicy.EXCLUSIVE);
        assertThat(resumed.instanceId()).isEqualTo(session.instanceId());
    }

    @Test
    void acquireSession_defaultPolicy_isExclusive() {
        manager = createManager(0, 5);
        manager.acquireSession("reviewer", "/workspace/pr-42");

        assertThatThrownBy(() -> manager.acquireSession("coder", "/workspace/pr-42"))
                .isInstanceOf(WorkingDirConflictException.class);
    }

    @Test
    void acquireSession_branchIsolated_throwsUnsupported() {
        manager = createManager(0, 5);
        assertThatThrownBy(() -> manager.acquireSession("reviewer", "/workspace/pr-42",
                                                        null, WorkingDirPolicy.BRANCH_ISOLATED))
                .isInstanceOf(UnsupportedOperationException.class)
                .hasMessageContaining("BRANCH_ISOLATED");
    }

    @Test
    void activeSessionsForWorkingDir_returnsMatching() {
        manager = createManager(0, 5);
        manager.acquireSession("reviewer", "/workspace/pr-42");
        manager.acquireSession("coder", "/workspace/task-1");

        assertThat(manager.activeSessionsForWorkingDir("/workspace/pr-42")).hasSize(1);
        assertThat(manager.activeSessionsForWorkingDir("/workspace/task-1")).hasSize(1);
        assertThat(manager.activeSessionsForWorkingDir("/workspace/other")).isEmpty();
    }

    @Test
    void acquireSession_doesNotResumeMismatchedIdentity() {
        manager = createManager(0, 5);
        manager.acquireSession("reviewer", "/workspace/pr-42");
        manager.suspendSession("mem-1");

        var newSession = manager.acquireSession("coder", "/workspace/task-1");
        assertThat(newSession.instanceId()).isNotEqualTo("mem-1");
        assertThat(ops.createCount()).isEqualTo(2);
    }

    @Test
    void acquireSession_evictsHighestScoringWhenAtMaxActive() throws Exception {
        manager = createManager(0, 2);
        var s1 = manager.acquireSession("reviewer", "/workspace/pr-1");
        manager.recordInteraction(s1.instanceId(), 50 * 1024 * 1024);

        Thread.sleep(10);
        var s2 = manager.acquireSession("coder", "/workspace/task-1");
        manager.recordInteraction(s2.instanceId(), 50 * 1024 * 1024);

        var s3 = manager.acquireSession("tester", "/workspace/test-1");
        assertThat(manager.activeCount()).isEqualTo(2);
        assertThat(manager.suspendedCount()).isEqualTo(1);
        assertThat(ops.suspendCount()).isEqualTo(1);
    }

    @Test
    void acquireSession_throwsWhenAtMaxAndMinPreventsEviction() {
        manager = createManager(2, 2);
        manager.acquireSession("reviewer", "/workspace/pr-1");
        manager.acquireSession("coder", "/workspace/task-1");

        assertThatThrownBy(() -> manager.acquireSession("tester", "/workspace/test-1"))
                .isInstanceOf(AgentPoolExhaustedException.class);
    }

    @Test
    void suspendSession_movesToSuspended() {
        manager = createManager(0, 5);
        var session = manager.acquireSession("reviewer", "/workspace/pr-42");
        manager.suspendSession(session.instanceId());

        assertThat(manager.activeCount()).isZero();
        assertThat(manager.suspendedCount()).isEqualTo(1);
        assertThat(ops.suspendCount()).isEqualTo(1);
    }

    @Test
    void suspendSession_unknownIdIsNoOp() {
        manager = createManager(0, 5);
        assertThatCode(() -> manager.suspendSession("nonexistent"))
                .doesNotThrowAnyException();
    }

    @Test
    void resumeSession_byInstanceId() {
        manager = createManager(0, 5);
        var session = manager.acquireSession("reviewer", "/workspace/pr-42");
        manager.suspendSession(session.instanceId());

        var resumed = manager.resumeSession(session.instanceId());
        assertThat(resumed).isNotNull();
        assertThat(resumed.state()).isEqualTo(SessionState.ACTIVE);
        assertThat(manager.activeCount()).isEqualTo(1);
        assertThat(manager.suspendedCount()).isZero();
    }

    @Test
    void resumeSession_unknownIdReturnsNull() {
        manager = createManager(0, 5);
        assertThat(manager.resumeSession("nonexistent")).isNull();
    }

    @Test
    void recordInteraction_updatesEvictionMetadata() throws Exception {
        manager = createManager(0, 2);
        var s1 = manager.acquireSession("reviewer", "/workspace/pr-1");
        Thread.sleep(10);
        var s2 = manager.acquireSession("coder", "/workspace/task-1");

        manager.recordInteraction(s1.instanceId(), 10 * 1024 * 1024);
        manager.recordInteraction(s2.instanceId(), 500 * 1024 * 1024);

        var s3 = manager.acquireSession("tester", "/workspace/test-1");

        var s2Managed = manager.getSession(s2.instanceId());
        assertThat(s2Managed.state()).isEqualTo(SessionState.SUSPENDED);
    }

    @Test
    void destroySession_removesCompletely() {
        manager = createManager(0, 5);
        var session = manager.acquireSession("reviewer", "/workspace/pr-42");
        manager.destroySession(session.instanceId());

        assertThat(manager.activeCount()).isZero();
        assertThat(manager.suspendedCount()).isZero();
        assertThat(ops.destroyCount()).isEqualTo(1);
    }

    @Test
    void shutdown_destroysAll() {
        manager = createManager(0, 5);
        manager.acquireSession("reviewer", "/workspace/pr-1");
        manager.acquireSession("coder", "/workspace/task-1");
        var s3 = manager.acquireSession("tester", "/workspace/test-1");
        manager.suspendSession(s3.instanceId());

        manager.shutdown();

        assertThat(manager.activeCount()).isZero();
        assertThat(manager.suspendedCount()).isZero();
        assertThat(ops.destroyCount()).isEqualTo(3);
    }

    @Test
    void status_returnsCorrectCounts() {
        manager = createManager(1, 5);
        manager.acquireSession("reviewer", "/workspace/pr-1");
        manager.acquireSession("coder", "/workspace/task-1");
        var s3 = manager.acquireSession("tester", "/workspace/test-1");
        manager.suspendSession(s3.instanceId());

        var status = manager.status();
        assertThat(status.active()).isEqualTo(2);
        assertThat(status.idle()).isEqualTo(1);
        assertThat(status.total()).isEqualTo(3);
        assertThat(status.health()).isEqualTo(AgentPoolHealth.HEALTHY);
    }
}
```

- [ ] **Step 3: Run all casehub tests to verify migration**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing,casehub -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: PASS (all 19 testing + 331 casehub tests — 22 of which are the migrated AgentSessionManagerTest)

- [ ] **Step 4: Commit**

```bash
git add casehub/pom.xml casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentSessionManagerTest.java
git commit -m "refactor(#230): migrate AgentSessionManagerTest to claudony-testing

Replaces inline anonymous SessionOperations with InMemorySessionOperations.
All 22 tests pass unchanged, validating the testing module.

Closes #230"
```

- [ ] **Step 5: Run full module tests for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: PASS (all modules)

## References

- [2026-09-29-agent-pool-test-utilities-design.md] — design spec
- [SessionOperations.java:4] — SPI interface (6 methods)
- [AgentSessionManagerTest.java:13] — inline mock to extract from
- [AgentSessionManager.java:10] — pool manager consuming SessionOperations
- [AgentSessionManagerConfig.java:3] — config record (minActive, maxActive)
- [engine/testing/pom.xml] — casehub-engine-testing pattern
- [GitHub #230]
- [GitHub #239] — pool suspend model fix (tmux-as-persistence)
