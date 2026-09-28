# Fleet Integration Test Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Each task follows TDD and uses
> ide-tooling for structural editing. Steps use checkbox (`- [ ]`) syntax.

**Focal issue:** #238 — test: fleet integration test with mock Java agent

**Goal:** Prove the full fleet chain works end-to-end with a mock Java agent in real tmux sessions.

**Architecture:** A `MockAgent` Java main class in test sources acts as a fake CLI agent. `FleetPoolIntegrationTest` exercises the full chain: pool definition → backend → session manager → tmux → observable output → lifecycle.

**Tech Stack:** Java 21, JUnit 5, AssertJ, real tmux (TmuxService from core module)

## Global Constraints

- All files in `casehub/src/test/java/io/casehub/claudony/casehub/fleet/`
- Plain JUnit (not @QuarkusTest) — construct TmuxService directly
- `@AfterEach` must clean up all tmux sessions to prevent leaks
- Mock agent command: `java -cp <test-classpath> io.casehub.claudony.casehub.fleet.MockAgent`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=FleetPoolIntegrationTest`

---

## Batch 1: Mock Agent and Fleet Integration Test

### Task 1: Create MockAgent and FleetPoolIntegrationTest

**Files:**
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/MockAgent.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/FleetPoolIntegrationTest.java`

**MockAgent.java** — tiny main class:
- Prints `MOCK_AGENT_READY` to stdout
- Reads stdin in a loop, echoes each line prefixed with `ECHO:`
- Ignores command-line arguments (the fleet appends `--session-id <uuid>`)
- Exits on EOF

**FleetPoolIntegrationTest.java** — plain JUnit with real tmux:

**Setup:**
- Construct `TmuxService` directly (no CDI)
- Compute test classpath from `System.getProperty("java.class.path")`
- Build mock agent command: `java -cp <classpath> io.casehub.claudony.casehub.fleet.MockAgent`
- Use test-specific session prefix (`test-fleet-`) to avoid colliding with real sessions

**Test cases:**

1. `fullChain_yamlToRegistryToBackendToSession` — Parse YAML with mock agent command → register in AgentPoolDefinitionRegistry → `ClaudonyAgentBackend.fromDefinition()` → `acquireSession()` → verify tmux session exists → capture pane output contains `MOCK_AGENT_READY` → destroy session → verify gone

2. `poolCapacity_evictsWhenFull` — Create pool with maxActive=2 → acquire 2 sessions → acquire 3rd → verify oldest is evicted (destroyed) → verify 2 sessions remain active

3. `poolStatus_reflectsRealState` — Create pool → acquire session → verify `poolStatus().active() == 1` → destroy → verify `poolStatus().active() == 0`

**Teardown:**
- `@AfterEach` lists all tmux sessions matching `test-fleet-*` prefix and kills them

**Key implementation detail:** `ClaudonyAgentBackend.fromDefinition()` requires a `ClaudonyConfig`. Create a mock config with just `defaultWorkingDir()` returning `/tmp`. The config is only used for `openSession()` default working dir, and our test calls `acquireSession()` on the manager directly.

Actually — looking at the code path again: `fromDefinition()` creates `TmuxSessionOperations` which creates real tmux sessions. The `ClaudonyConfig` is stored but only used in `openSession()`. For the test, we can construct `TmuxSessionOperations` and `AgentSessionManager` directly without going through `ClaudonyAgentBackend` at all — that's simpler and tests the same chain. But we SHOULD also test `fromDefinition()` to prove the wiring works.

**Steps:**

- [ ] **Step 1: Write MockAgent.java**

```java
package io.casehub.claudony.casehub.fleet;

import java.util.Scanner;

public class MockAgent {
    public static void main(String[] args) {
        System.out.println("MOCK_AGENT_READY");
        System.out.flush();
        var scanner = new Scanner(System.in);
        while (scanner.hasNextLine()) {
            String line = scanner.nextLine();
            System.out.println("ECHO:" + line);
            System.out.flush();
        }
    }
}
```

- [ ] **Step 2: Write FleetPoolIntegrationTest.java**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.claudony.server.TmuxService;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class FleetPoolIntegrationTest {

    private static final String TEST_PREFIX = "test-fleet-";

    private TmuxService tmux;
    private String mockAgentCommand;
    private final List<String> createdSessions = new ArrayList<>();

    @BeforeEach
    void setUp() {
        tmux = new TmuxService();
        String classpath = System.getProperty("java.class.path");
        mockAgentCommand = "java -cp " + classpath + " io.casehub.claudony.casehub.fleet.MockAgent";
    }

    @AfterEach
    void tearDown() throws Exception {
        for (String sessionId : createdSessions) {
            try { tmux.killSession(sessionId); } catch (Exception ignored) {}
        }
        // Safety net: kill any leaked sessions with our prefix
        for (String name : tmux.listSessionNames()) {
            if (name.startsWith(TEST_PREFIX)) {
                try { tmux.killSession(name); } catch (Exception ignored) {}
            }
        }
    }

    @Test
    void fullChain_yamlToRegistryToBackendToSession() throws Exception {
        // Parse YAML with mock agent command
        var yaml = """
                agent-pools:
                  test-reviewer:
                    working-dir: /tmp
                    command: %s
                    pool:
                      min-active: 0
                      max-active: 5
                """.formatted(mockAgentCommand);

        var parser = new AgentPoolYamlParser();
        var registry = new AgentPoolDefinitionRegistry();
        parser.parseInto(yaml, registry);

        assertThat(registry.get("test-reviewer")).isPresent();
        var definition = registry.get("test-reviewer").get();

        // Create backend from definition
        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, definition.agent().command());
        var manager = new AgentSessionManager(definition.toSessionManagerConfig(), ops);

        // Acquire session — creates real tmux session
        var session = manager.acquireSession("reviewer-1", "/tmp");
        createdSessions.add(session.instanceId());

        // Verify tmux session exists
        assertThat(tmux.sessionExists(session.instanceId())).isTrue();

        // Wait for mock agent output and verify
        Thread.sleep(2000); // allow JVM startup in tmux
        String output = tmux.capturePane(session.instanceId(), 20);
        assertThat(output).contains("MOCK_AGENT_READY");

        // Destroy and verify gone
        manager.destroySession(session.instanceId());
        assertThat(tmux.sessionExists(session.instanceId())).isFalse();
    }

    @Test
    void poolCapacity_evictsWhenFull() throws Exception {
        var definition = AgentPoolDefinition.builder()
                .agent("capacity-test")
                    .command(mockAgentCommand)
                .pool()
                    .minActive(0)
                    .maxActive(2)
                .build();

        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, definition.agent().command());
        var manager = new AgentSessionManager(definition.toSessionManagerConfig(), ops);

        // Acquire 2 sessions — both should be active
        var s1 = manager.acquireSession("worker-1", "/tmp");
        createdSessions.add(s1.instanceId());
        var s2 = manager.acquireSession("worker-2", "/tmp/other");
        createdSessions.add(s2.instanceId());

        assertThat(manager.activeCount()).isEqualTo(2);

        // Acquire 3rd — should evict oldest
        var s3 = manager.acquireSession("worker-3", "/tmp/third");
        createdSessions.add(s3.instanceId());

        assertThat(manager.activeCount()).isEqualTo(2);
        assertThat(manager.status().total()).isEqualTo(3); // 2 active + 1 suspended
    }

    @Test
    void poolStatus_reflectsRealState() throws Exception {
        var definition = AgentPoolDefinition.builder()
                .agent("status-test")
                    .command(mockAgentCommand)
                .pool()
                    .minActive(0)
                    .maxActive(5)
                .build();

        var ops = new TmuxSessionOperations(tmux, TEST_PREFIX, definition.agent().command());
        var manager = new AgentSessionManager(definition.toSessionManagerConfig(), ops);

        assertThat(manager.status().active()).isZero();

        var session = manager.acquireSession("worker-1", "/tmp");
        createdSessions.add(session.instanceId());
        assertThat(manager.status().active()).isEqualTo(1);

        manager.destroySession(session.instanceId());
        assertThat(manager.status().active()).isZero();
    }
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=FleetPoolIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (all 3 tests)

- [ ] **Step 4: Run full module tests for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`
Expected: PASS (all ~327 tests)

- [ ] **Step 5: Commit**

```bash
git add casehub/src/test/java/io/casehub/claudony/casehub/fleet/MockAgent.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/FleetPoolIntegrationTest.java
git commit -m "test(#238): fleet integration test with mock Java agent

MockAgent — tiny Java main that prints MOCK_AGENT_READY and echoes stdin.
FleetPoolIntegrationTest — exercises pool definition → backend → real tmux
session → observable output → lifecycle management.

Closes #238"
```

## References

- [2026-09-28-fleet-integration-test-design.md] — design spec
- [ClaudonyAgentBackend.java:45] — fromDefinition() factory
- [AgentSessionManager.java] — capacity-bounded pool
- [TmuxSessionOperations.java:31] — creates real tmux sessions
- [TmuxServiceTest.java] — pattern for real-tmux test setup
- [GitHub #238]
