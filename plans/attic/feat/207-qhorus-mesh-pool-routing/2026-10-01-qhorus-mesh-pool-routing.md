# Qhorus Mesh Routing for Agent Pools — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #207 — feat: Qhorus mesh routing integration for agent pools
**Issue group:** #207

**Goal:** Bridge pool session lifecycle into Qhorus instance registration so
fleet-managed LLM instances are discoverable and routable through the agent mesh.

**Architecture:** A `SessionLifecycleListener` callback interface in `claudony-casehub`
lets `AgentSessionManager` notify an observer at each lifecycle transition. In
`claudony-app`, `PoolMeshRegistrar` implements this interface and calls Qhorus
`InstanceService` to register/update/deregister pool sessions as mesh instances.
Integration tests use real tmux + `InMemoryInstanceStore` to verify the full chain.

**Tech Stack:** Java 21, Quarkus 3.32.2, Qhorus (embedded), tmux, JUnit 5, AssertJ

## Global Constraints

- Java `release=21` compiled on Java 26
- `casehub-qhorus-testing` is already a test dependency in `claudony-casehub`
- `InMemoryInstanceStore` available from `casehub-qhorus-testing`
- Use `mvn` not `./mvnw`
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all builds
- IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing
- All commits reference `Refs #207`

---

## Batch 1: Listener SPI + AgentSessionManager integration

### Task 1: SessionLifecycleListener interface and AgentSessionManager wiring

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/SessionLifecycleListener.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/SessionLifecycleListenerTest.java`

**Interfaces:**
- Produces: `SessionLifecycleListener` — interface with `onAcquired(ManagedSession, String)`, `onSuspended(ManagedSession, String)`, `onResumed(ManagedSession, String)`, `onDestroyed(String, String)`, and static `NOOP` constant
- Produces: `AgentSessionManager` gains `poolName` field and `SessionLifecycleListener` parameter in constructors

- [ ] **Step 1: Write the failing test — listener receives acquire callback**

Create test file `casehub/src/test/java/io/casehub/claudony/casehub/fleet/SessionLifecycleListenerTest.java`:

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.claudony.testing.fleet.InMemorySessionOperations;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class SessionLifecycleListenerTest {

    @Test
    void acquire_notifiesListener() {
        var events = new ArrayList<String>();
        var listener = new RecordingListener(events);
        var ops = new InMemorySessionOperations();
        var manager = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5),
                ops, listener, "test-pool");

        var session = manager.acquireSession("worker-1", "/tmp");

        assertThat(events).containsExactly("acquired:worker-1:test-pool");
        assertThat(session).isNotNull();
    }

    private record RecordingListener(List<String> events) implements SessionLifecycleListener {
        @Override
        public void onAcquired(ManagedSession session, String poolName) {
            events.add("acquired:" + session.identity() + ":" + poolName);
        }
        @Override
        public void onSuspended(ManagedSession session, String poolName) {
            events.add("suspended:" + session.identity() + ":" + poolName);
        }
        @Override
        public void onResumed(ManagedSession session, String poolName) {
            events.add("resumed:" + session.identity() + ":" + poolName);
        }
        @Override
        public void onDestroyed(String sessionId, String poolName) {
            events.add("destroyed:" + sessionId + ":" + poolName);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=SessionLifecycleListenerTest#acquire_notifiesListener`
Expected: Compilation failure — `SessionLifecycleListener` does not exist, `AgentSessionManager` constructor with listener/poolName does not exist.

- [ ] **Step 3: Create SessionLifecycleListener interface**

Use `ide_create_file` to create `casehub/src/main/java/io/casehub/claudony/casehub/fleet/SessionLifecycleListener.java`:

```java
package io.casehub.claudony.casehub.fleet;

public interface SessionLifecycleListener {

    SessionLifecycleListener NOOP = new SessionLifecycleListener() {};

    default void onAcquired(ManagedSession session, String poolName) {}
    default void onSuspended(ManagedSession session, String poolName) {}
    default void onResumed(ManagedSession session, String poolName) {}
    default void onDestroyed(String sessionId, String poolName) {}
}
```

- [ ] **Step 4: Add poolName and listener to AgentSessionManager**

Add a `poolName` field and `listener` field to `AgentSessionManager`. The existing 2-arg and 3-arg constructors remain unchanged (they default to `NOOP` listener and empty poolName). Add a new 4-arg constructor:

```java
public AgentSessionManager(AgentSessionManagerConfig config, SessionOperations ops,
                           SessionLifecycleListener listener, String poolName) {
    this(config, ops);
    this.listener = listener;
    this.poolName = poolName != null ? poolName : "";
}
```

Add fields:
```java
private SessionLifecycleListener listener = SessionLifecycleListener.NOOP;
private String poolName = "";
```

Add a `poolName()` accessor:
```java
public String poolName() { return poolName; }
```

Then fire the listener at each lifecycle point:

In `acquireSession()` (all overloads converge to the 4-arg version) — after the session is created/resumed, add:
```java
listener.onAcquired(session, poolName);
```

In `suspendSession()` — after `session.setState(SessionState.SUSPENDED)`:
```java
listener.onSuspended(session, poolName);
```

In `resumeSession()` — after `session.setState(SessionState.ACTIVE)`:
```java
listener.onResumed(session, poolName);
```

In `destroySession()` — before removing from the map:
```java
listener.onDestroyed(instanceId, poolName);
```

In `shutdown()` — in the loop destroying sessions, before `ops.destroy()`:
```java
listener.onDestroyed(entry.getKey(), poolName);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=SessionLifecycleListenerTest#acquire_notifiesListener`
Expected: PASS

- [ ] **Step 6: Write tests for suspend, resume, destroy, and shutdown callbacks**

Add to `SessionLifecycleListenerTest.java`:

```java
@Test
void suspend_notifiesListener() {
    var events = new ArrayList<String>();
    var listener = new RecordingListener(events);
    var ops = new InMemorySessionOperations();
    var manager = new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), ops, listener, "test-pool");

    var session = manager.acquireSession("worker-1", "/tmp");
    events.clear();

    manager.suspendSession(session.instanceId());

    assertThat(events).containsExactly("suspended:worker-1:test-pool");
}

@Test
void resume_notifiesListener() {
    var events = new ArrayList<String>();
    var listener = new RecordingListener(events);
    var ops = new InMemorySessionOperations();
    var manager = new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), ops, listener, "test-pool");

    var session = manager.acquireSession("worker-1", "/tmp");
    manager.suspendSession(session.instanceId());
    events.clear();

    manager.resumeSession(session.instanceId());

    assertThat(events).containsExactly("resumed:worker-1:test-pool");
}

@Test
void destroy_notifiesListener() {
    var events = new ArrayList<String>();
    var listener = new RecordingListener(events);
    var ops = new InMemorySessionOperations();
    var manager = new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), ops, listener, "test-pool");

    var session = manager.acquireSession("worker-1", "/tmp");
    String id = session.instanceId();
    events.clear();

    manager.destroySession(id);

    assertThat(events).containsExactly("destroyed:" + id + ":test-pool");
}

@Test
void shutdown_notifiesListenerForEachSession() {
    var events = new ArrayList<String>();
    var listener = new RecordingListener(events);
    var ops = new InMemorySessionOperations();
    var manager = new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), ops, listener, "test-pool");

    var s1 = manager.acquireSession("worker-1", "/tmp");
    var s2 = manager.acquireSession("worker-2", "/tmp/other");
    events.clear();

    manager.shutdown();

    assertThat(events).hasSize(2);
    assertThat(events).allMatch(e -> e.startsWith("destroyed:") && e.endsWith(":test-pool"));
}

@Test
void noopListener_doesNotThrow() {
    var ops = new InMemorySessionOperations();
    var manager = new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), ops);

    var session = manager.acquireSession("worker-1", "/tmp");
    manager.suspendSession(session.instanceId());
    manager.resumeSession(session.instanceId());
    manager.destroySession(session.instanceId());
    // No exception = NOOP listener works
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=SessionLifecycleListenerTest`
Expected: All 6 tests PASS

- [ ] **Step 8: Verify existing tests still pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentSessionManagerTest`
Expected: All existing tests PASS (they use the 2/3-arg constructor which defaults to NOOP listener)

- [ ] **Step 9: Run diagnostics**

Use `ide_diagnostics` on both new/modified files to verify no compilation errors.

- [ ] **Step 10: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/SessionLifecycleListener.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/SessionLifecycleListenerTest.java
git commit -m "feat(#207): SessionLifecycleListener SPI + AgentSessionManager callback wiring

Refs #207"
```

---

## Batch 2: PoolMeshRegistrar + integration tests

### Task 2: PoolMeshRegistrar and end-to-end integration tests

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/PoolMeshRegistrar.java`
- Test: `app/src/test/java/io/casehub/claudony/server/fleet/PoolMeshIntegrationTest.java`

**Interfaces:**
- Consumes: `SessionLifecycleListener` (from Task 1) — `onAcquired(ManagedSession, String)`, `onSuspended(ManagedSession, String)`, `onResumed(ManagedSession, String)`, `onDestroyed(String, String)`
- Consumes: `InstanceService` (from Qhorus) — `register(String, String, List<String>, String)`, `markOffline(String)`, `deregister(String)`, `findByInstanceId(String)`, `findByCapability(String)`
- Produces: `PoolMeshRegistrar` — `@ApplicationScoped` CDI bean implementing `SessionLifecycleListener`

- [ ] **Step 1: Write the failing integration test — acquire registers Qhorus instance**

Create test file `app/src/test/java/io/casehub/claudony/server/fleet/PoolMeshIntegrationTest.java`
(in `claudony-app` because the test imports `PoolMeshRegistrar` which lives in that module):

```java
package io.casehub.claudony.server.fleet;

import io.casehub.claudony.casehub.fleet.AgentSessionManager;
import io.casehub.claudony.casehub.fleet.AgentSessionManagerConfig;
import io.casehub.claudony.casehub.fleet.TmuxSessionOperations;
import io.casehub.claudony.server.TmuxService;
import io.casehub.qhorus.persistence.memory.InMemoryInstanceStore;
import io.casehub.qhorus.runtime.instance.InstanceService;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class PoolMeshIntegrationTest {

    private static final String TEST_PREFIX = "test-mesh-";

    private TmuxService tmux;
    private String mockAgentCommand;
    private Path mockAgentScript;
    private final List<String> createdSessions = new ArrayList<>();
    private InMemoryInstanceStore instanceStore;
    private InstanceService instanceService;

    @BeforeEach
    void setUp() throws IOException {
        tmux = new TmuxService();
        mockAgentScript = Files.createTempFile("mock-agent-", ".sh");
        Files.writeString(mockAgentScript, """
                #!/bin/sh
                echo MOCK_AGENT_READY
                while IFS= read -r line; do
                    echo "ECHO:$line"
                done
                """);
        mockAgentScript.toFile().setExecutable(true);
        mockAgentCommand = mockAgentScript.toAbsolutePath().toString();
        instanceStore = new InMemoryInstanceStore();
        instanceService = new InstanceService(instanceStore);
    }

    @AfterEach
    void tearDown() throws Exception {
        for (String sessionId : createdSessions) {
            try { tmux.killSession(sessionId); } catch (Exception ignored) {}
        }
        for (String name : tmux.listSessionNames()) {
            if (name.startsWith(TEST_PREFIX)) {
                try { tmux.killSession(name); } catch (Exception ignored) {}
            }
        }
        Files.deleteIfExists(mockAgentScript);
    }

    @Test
    void acquire_registersAsQhorusInstance() throws Exception {
        var registrar = new PoolMeshRegistrar(instanceService);
        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, mockAgentCommand);
        var manager = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops, registrar, "code-reviewer");

        var session = manager.acquireSession("reviewer-1", "/tmp");
        createdSessions.add(session.instanceId());

        var instance = instanceService.findByInstanceId(session.instanceId());
        assertThat(instance).isPresent();
        assertThat(instance.get().status()).isEqualTo("online");
        assertThat(instance.get().description()).isEqualTo("pool:code-reviewer/reviewer-1");
        assertThat(instance.get().claudonySessionId()).isEqualTo(session.instanceId());

        var caps = instanceService.findCapabilityTagsForInstance(session.instanceId());
        assertThat(caps).containsExactly("pool:code-reviewer");

        manager.destroySession(session.instanceId());
    }

    @Test
    void suspend_marksInstanceOffline() throws Exception {
        var registrar = new PoolMeshRegistrar(instanceService);
        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, mockAgentCommand);
        var manager = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops, registrar, "code-reviewer");

        var session = manager.acquireSession("reviewer-1", "/tmp");
        createdSessions.add(session.instanceId());

        manager.suspendSession(session.instanceId());

        var instance = instanceService.findByInstanceId(session.instanceId());
        assertThat(instance).isPresent();
        assertThat(instance.get().status()).isEqualTo("offline");

        manager.destroySession(session.instanceId());
    }

    @Test
    void resume_marksInstanceOnline() throws Exception {
        var registrar = new PoolMeshRegistrar(instanceService);
        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, mockAgentCommand);
        var manager = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops, registrar, "code-reviewer");

        var session = manager.acquireSession("reviewer-1", "/tmp");
        createdSessions.add(session.instanceId());

        manager.suspendSession(session.instanceId());
        assertThat(instanceService.findByInstanceId(session.instanceId()).get().status())
                .isEqualTo("offline");

        manager.resumeSession(session.instanceId());

        var instance = instanceService.findByInstanceId(session.instanceId());
        assertThat(instance).isPresent();
        assertThat(instance.get().status()).isEqualTo("online");

        manager.destroySession(session.instanceId());
    }

    @Test
    void destroy_deregistersInstance() throws Exception {
        var registrar = new PoolMeshRegistrar(instanceService);
        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, mockAgentCommand);
        var manager = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops, registrar, "code-reviewer");

        var session = manager.acquireSession("reviewer-1", "/tmp");
        createdSessions.add(session.instanceId());

        assertThat(instanceService.findByInstanceId(session.instanceId())).isPresent();

        manager.destroySession(session.instanceId());

        assertThat(instanceService.findByInstanceId(session.instanceId())).isEmpty();
    }

    @Test
    void fullLifecycle_cleanState() throws Exception {
        var registrar = new PoolMeshRegistrar(instanceService);
        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, mockAgentCommand);
        var manager = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops, registrar, "code-reviewer");

        var session = manager.acquireSession("reviewer-1", "/tmp");
        createdSessions.add(session.instanceId());
        String id = session.instanceId();

        // acquire → online
        assertThat(instanceService.findByInstanceId(id).get().status()).isEqualTo("online");

        // suspend → offline
        manager.suspendSession(id);
        assertThat(instanceService.findByInstanceId(id).get().status()).isEqualTo("offline");

        // resume → online
        manager.resumeSession(id);
        assertThat(instanceService.findByInstanceId(id).get().status()).isEqualTo("online");

        // destroy → gone
        manager.destroySession(id);
        assertThat(instanceService.findByInstanceId(id)).isEmpty();
        assertThat(instanceService.listAll()).isEmpty();
    }

    @Test
    void capabilityRouting_findsPoolInstances() throws Exception {
        var registrar = new PoolMeshRegistrar(instanceService);

        // Pool 1: code-reviewer
        var ops1 = new TmuxSessionOperations(tmux, TEST_PREFIX, mockAgentCommand);
        var mgr1 = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops1, registrar, "code-reviewer");
        var s1 = mgr1.acquireSession("reviewer-1", "/tmp");
        createdSessions.add(s1.instanceId());

        // Pool 2: test-runner
        var ops2 = new TmuxSessionOperations(tmux, TEST_PREFIX + "2-", mockAgentCommand);
        var mgr2 = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops2, registrar, "test-runner");
        var s2 = mgr2.acquireSession("runner-1", "/tmp");
        createdSessions.add(s2.instanceId());

        // Routing by capability
        var reviewers = instanceService.findByCapability("pool:code-reviewer");
        assertThat(reviewers).hasSize(1);
        assertThat(reviewers.get(0).instanceId()).isEqualTo(s1.instanceId());

        var runners = instanceService.findByCapability("pool:test-runner");
        assertThat(runners).hasSize(1);
        assertThat(runners.get(0).instanceId()).isEqualTo(s2.instanceId());

        mgr1.destroySession(s1.instanceId());
        mgr2.destroySession(s2.instanceId());
    }

    @Test
    void multipleSessionsInPool_allRegistered() throws Exception {
        var registrar = new PoolMeshRegistrar(instanceService);
        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, mockAgentCommand);
        var manager = new AgentSessionManager(
                new AgentSessionManagerConfig(0, 5), ops, registrar, "code-reviewer");

        var s1 = manager.acquireSession("reviewer-1", "/tmp");
        var s2 = manager.acquireSession("reviewer-2", "/tmp/other");
        createdSessions.add(s1.instanceId());
        createdSessions.add(s2.instanceId());

        var instances = instanceService.findByCapability("pool:code-reviewer");
        assertThat(instances).hasSize(2);
        assertThat(instances).extracting("instanceId")
                .containsExactlyInAnyOrder(s1.instanceId(), s2.instanceId());

        manager.destroySession(s1.instanceId());
        manager.destroySession(s2.instanceId());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PoolMeshIntegrationTest#acquire_registersAsQhorusInstance`
Expected: Compilation failure — `PoolMeshRegistrar` does not exist.

- [ ] **Step 3: Create PoolMeshRegistrar**

Use `ide_create_file` to create `app/src/main/java/io/casehub/claudony/server/fleet/PoolMeshRegistrar.java`:

```java
package io.casehub.claudony.server.fleet;

import io.casehub.claudony.casehub.fleet.ManagedSession;
import io.casehub.claudony.casehub.fleet.SessionLifecycleListener;
import io.casehub.qhorus.runtime.instance.InstanceService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;

@ApplicationScoped
public class PoolMeshRegistrar implements SessionLifecycleListener {

    private final InstanceService instanceService;

    @Inject
    public PoolMeshRegistrar(InstanceService instanceService) {
        this.instanceService = instanceService;
    }

    @Override
    public void onAcquired(ManagedSession session, String poolName) {
        instanceService.register(
                session.instanceId(),
                "pool:" + poolName + "/" + session.identity(),
                List.of("pool:" + poolName),
                session.instanceId());
    }

    @Override
    public void onSuspended(ManagedSession session, String poolName) {
        instanceService.markOffline(session.instanceId());
    }

    @Override
    public void onResumed(ManagedSession session, String poolName) {
        instanceService.register(
                session.instanceId(),
                "pool:" + poolName + "/" + session.identity(),
                List.of("pool:" + poolName),
                session.instanceId());
    }

    @Override
    public void onDestroyed(String sessionId, String poolName) {
        instanceService.deregister(sessionId);
    }
}
```

- [ ] **Step 4: Run all integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PoolMeshIntegrationTest`
Expected: All 7 tests PASS

- [ ] **Step 5: Run existing FleetPoolIntegrationTest to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=FleetPoolIntegrationTest`
Expected: All 11 existing tests PASS (they use 2/3-arg constructors, no listener)

- [ ] **Step 6: Run diagnostics**

Use `ide_diagnostics` on `PoolMeshRegistrar.java`.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/fleet/PoolMeshRegistrar.java \
       app/src/test/java/io/casehub/claudony/server/fleet/PoolMeshIntegrationTest.java
git commit -m "feat(#207): PoolMeshRegistrar bridges pool lifecycle to Qhorus instances

Pool sessions are now registered as Qhorus instances on acquire, marked
offline on suspend, re-registered on resume, and deregistered on destroy.
7 integration tests verify the full chain with real tmux + InMemoryInstanceStore.

Refs #207"
```

---

## Batch 3: CDI wiring + CLAUDE.md update

### Task 3: Wire PoolMeshRegistrar into pool construction and update docs

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolManagerRegistry.java`
- Modify: `CLAUDE.md` (test count and architecture notes)

**Interfaces:**
- Consumes: `PoolMeshRegistrar` (from Task 2) — `@ApplicationScoped` bean
- Consumes: `AgentPoolManagerRegistry.register()` — existing pool registration
- Consumes: `AgentSessionManager` 4-arg constructor (from Task 1) — `(config, ops, listener, poolName)`

- [ ] **Step 1: Examine AgentPoolManagerRegistry to understand pool construction**

Use `ide_file_structure` on `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolManagerRegistry.java` and read the `register()` method to understand where `AgentSessionManager` is created. Determine where the listener should be injected.

Look for who calls `AgentPoolManagerRegistry.register()` — use `ide_find_references` on the `register` method. The caller constructs the `AgentSessionManager` and registers it. The listener needs to flow from CDI injection at the call site to the `AgentSessionManager` constructor.

- [ ] **Step 2: Wire the listener**

The wiring depends on who constructs `AgentSessionManager` instances. Two patterns:

**Pattern A** — if `AgentPoolManagerRegistry` constructs `AgentSessionManager` internally:
Add a `setLifecycleListener(SessionLifecycleListener)` method to `AgentPoolManagerRegistry`.
Callers inject `PoolMeshRegistrar` and call `setLifecycleListener()` before registering pools.

**Pattern B** — if callers pass pre-built `AgentSessionManager` to `register()`:
Callers construct `AgentSessionManager` with the 4-arg constructor and the injected `PoolMeshRegistrar`.

Examine the actual code and apply the appropriate pattern. The goal: every `AgentSessionManager`
created through the registry gets `PoolMeshRegistrar` as its listener and the pool name
from the `AgentPoolDefinition`.

- [ ] **Step 3: Verify existing pool tests still pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolManagerRegistryTest`
Expected: PASS

- [ ] **Step 4: Verify full module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: All tests PASS

- [ ] **Step 5: Update CLAUDE.md**

Add `PoolMeshRegistrar` to the project structure section under `claudony-app`.
Update test count. Add `SessionLifecycleListener` to the `claudony-casehub` section.
Add note about pool→mesh integration to Architecture Notes.

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolManagerRegistry.java \
       CLAUDE.md
git commit -m "feat(#207): wire PoolMeshRegistrar into pool construction + docs

Pool sessions now automatically register as Qhorus mesh instances.
Closes #207"
```

## References

- [2026-10-01-qhorus-mesh-pool-routing-design.md] — design spec this plan implements
- [casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java] — pool session manager
- [casehub/src/main/java/io/casehub/claudony/casehub/fleet/SessionOperations.java] — SPI pattern reference
- [casehub/src/test/java/io/casehub/claudony/casehub/fleet/FleetPoolIntegrationTest.java] — existing test pattern
- [qhorus/runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java] — Qhorus registration
- [qhorus/api/src/main/java/io/casehub/qhorus/api/instance/Instance.java] — instance model
- [app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java] — pool service (unchanged)
- [GitHub #207] — feat: Qhorus mesh routing integration for agent pools
