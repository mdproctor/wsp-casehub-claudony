# Proactive Scaling Mode — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #243 — feat: proactive scaling mode — identity-aware pre-warming of suspended sessions
**Issue group:** #243

**Goal:** Add a proactive scaling mode that resumes recently-suspended pool sessions
before they're requested, reducing acquire latency for returning identities.

**Architecture:** A new `ProactiveConfig` variant in the sealed `ScalingConfig`
hierarchy tells `ScalingScheduler` to maintain a target number of active sessions
by resuming the most-recently-suspended ones. The proactive path bypasses
`ScalingPolicy` and directly calls `manager.resumeSession()`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ

## Global Constraints

- Java `release=21` compiled on Java 26
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all builds
- Use `mvn` not `./mvnw`
- IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing
- All commits reference `Refs #243`
- Test module name for `-pl` is `casehub` (directory name, not artifactId)

---

## Batch 1: ProactiveConfig + ScalingDirection.PREWARM + scheduler logic

### Task 1: ProactiveConfig, PREWARM direction, and proactive evaluation in ScalingScheduler

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingDirection.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ProactiveScalingTest.java`

**Interfaces:**
- Produces: `ScalingConfig.ProactiveConfig(int targetActive, Duration cooldown, Duration scaleInCooldown)` — new sealed variant
- Produces: `ScalingDirection.PREWARM` — new enum value
- Produces: `ScalingScheduler.evaluateProactive(String, AgentSessionManager, ScalingConfig.ProactiveConfig)` — resumes suspended sessions to reach target

- [ ] **Step 1: Write the failing test — proactive resumes suspended sessions**

Create test file `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ProactiveScalingTest.java`:

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

class ProactiveScalingTest {

    private AgentPoolDefinitionRegistry defRegistry;
    private AgentPoolManagerRegistry mgrRegistry;
    private AtomicInteger createCount;
    private ConcurrentHashMap<String, String> convIds;

    @BeforeEach
    void setUp() {
        defRegistry = new AgentPoolDefinitionRegistry();
        mgrRegistry = new AgentPoolManagerRegistry();
        createCount = new AtomicInteger();
        convIds = new ConcurrentHashMap<>();
    }

    private AgentSessionManager createManager(int min, int max) {
        return new AgentSessionManager(
            new AgentSessionManagerConfig(min, max),
            new SessionOperations() {
                @Override public String create(String i, String w) {
                    String id = "s-" + createCount.incrementAndGet();
                    convIds.put(id, "c-" + id);
                    return id;
                }
                @Override public String conversationId(String s) { return convIds.get(s); }
                @Override public void suspend(String s) {}
                @Override public void resume(String s, String c, String w) {}
                @Override public void destroy(String s) {}
                @Override public long memoryBytes(String s) { return 0; }
            }
        );
    }

    @Test
    void proactive_resumesSuspendedSessions() {
        var def = AgentPoolDefinition.builder()
            .agent("reviewer")
            .pool()
                .maxActive(10)
                .scaling(new ScalingConfig.ProactiveConfig(2, null, null))
            .build();
        defRegistry.register(def);

        var manager = createManager(0, 10);
        mgrRegistry.register("reviewer", manager);

        // Create 3 sessions, suspend all of them
        var s1 = manager.acquireSession("r1", "/ws/1", null, WorkingDirPolicy.SHARED_READ);
        var s2 = manager.acquireSession("r2", "/ws/2", null, WorkingDirPolicy.SHARED_READ);
        var s3 = manager.acquireSession("r3", "/ws/3", null, WorkingDirPolicy.SHARED_READ);
        manager.suspendSession(s1.instanceId());
        manager.suspendSession(s2.instanceId());
        manager.suspendSession(s3.instanceId());

        assertThat(manager.activeCount()).isZero();
        assertThat(manager.suspendedCount()).isEqualTo(3);

        var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
        scheduler.tick();

        // Should have resumed 2 sessions (targetActive=2)
        assertThat(manager.activeCount()).isEqualTo(2);
        assertThat(manager.suspendedCount()).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ProactiveScalingTest#proactive_resumesSuspendedSessions`
Expected: Compilation failure — `ProactiveConfig` does not exist.

- [ ] **Step 3: Add ProactiveConfig to ScalingConfig sealed hierarchy**

Use `ide_insert_member` to add inside `ScalingConfig.java`, after `DemandPressureConfig`:

```java
record ProactiveConfig(
        int targetActive,
        Duration cooldown,
        Duration scaleInCooldown
) implements ScalingConfig {
    public ProactiveConfig {
        if (targetActive < 1) {
            throw new IllegalArgumentException("targetActive must be >= 1");
        }
        if (cooldown == null) {cooldown = Duration.ofSeconds(60);}
        if (scaleInCooldown == null) {scaleInCooldown = cooldown;}
    }
}
```

Update the sealed `permits` clause to include `ScalingConfig.ProactiveConfig`.

Update the `type()` default method switch to add:
```java
case ProactiveConfig p -> "proactive";
```

- [ ] **Step 4: Add PREWARM to ScalingDirection**

Use `ide_edit_member` to replace the enum:

```java
public enum ScalingDirection { NONE, OUT, IN, PREWARM }
```

- [ ] **Step 5: Add evaluateProactive to ScalingScheduler**

Use `ide_insert_member` to add after `evaluatePool`:

```java
private void evaluateProactive(String poolName, AgentSessionManager manager,
                                ScalingConfig.ProactiveConfig config) {
    var status = manager.status();
    int activeCount = status.active();
    int suspendedCount = status.idle();

    if (activeCount >= config.targetActive() || suspendedCount == 0) {
        scalingStates.put(poolName, new ScalingState(
                ScalingDecision.none(), java.time.Instant.now(),
                null, null, config));
        return;
    }

    int deficit = Math.min(config.targetActive() - activeCount, suspendedCount);
    deficit = Math.min(deficit, status.max() - activeCount);
    if (deficit <= 0) {
        scalingStates.put(poolName, new ScalingState(
                ScalingDecision.none(), java.time.Instant.now(),
                null, null, config));
        return;
    }

    var suspended = manager.sessions().stream()
            .filter(s -> s.state() == SessionState.SUSPENDED)
            .sorted(java.util.Comparator.comparing(ManagedSession::lastInteraction).reversed())
            .limit(deficit)
            .toList();

    int resumed = 0;
    for (var session : suspended) {
        try {
            manager.resumeSession(session.instanceId());
            resumed++;
        } catch (Exception e) {
            // session may have been destroyed concurrently
        }
    }

    var decision = new ScalingDecision(ScalingDirection.PREWARM, resumed,
            "proactive: resumed " + resumed + " of " + deficit + " to reach target " + config.targetActive());

    Instant now = Instant.now();
    var previousState = scalingStates.get(poolName);
    scalingStates.put(poolName, new ScalingState(
            decision, now,
            previousState != null ? previousState.lastScaleOut() : null,
            previousState != null ? previousState.lastScaleIn() : null,
            config));

    if (eventEmitter != null && resumed > 0) {
        eventEmitter.emitScalingDecision(poolName, decision, status.max(), status.max());
    }
}
```

- [ ] **Step 6: Add branch in evaluatePool for ProactiveConfig**

Use `ide_replace_member` on `evaluatePool` to add the proactive branch. At the top of the method, after the null checks and before the NoScalingConfig check, add:

```java
if (scalingConfig instanceof ScalingConfig.ProactiveConfig proactive) {
    var previousState = scalingStates.get(poolName);
    if (previousState != null && previousState.cooldownRemaining(Instant.now()).compareTo(Duration.ZERO) > 0) {
        return;
    }
    evaluateProactive(poolName, manager, proactive);
    return;
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ProactiveScalingTest#proactive_resumesSuspendedSessions`
Expected: PASS

- [ ] **Step 8: Write remaining tests**

Add to `ProactiveScalingTest.java`:

```java
@Test
void proactive_noActionWhenAtTarget() {
    var def = AgentPoolDefinition.builder()
        .agent("reviewer")
        .pool()
            .maxActive(10)
            .scaling(new ScalingConfig.ProactiveConfig(2, null, null))
        .build();
    defRegistry.register(def);

    var manager = createManager(0, 10);
    mgrRegistry.register("reviewer", manager);

    // Create 2 active sessions — already at target
    manager.acquireSession("r1", "/ws/1", null, WorkingDirPolicy.SHARED_READ);
    manager.acquireSession("r2", "/ws/2", null, WorkingDirPolicy.SHARED_READ);

    var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
    scheduler.tick();

    assertThat(manager.activeCount()).isEqualTo(2);
}

@Test
void proactive_respectsMaxActive() {
    var def = AgentPoolDefinition.builder()
        .agent("reviewer")
        .pool()
            .maxActive(3)
            .scaling(new ScalingConfig.ProactiveConfig(5, null, null))
        .build();
    defRegistry.register(def);

    var manager = createManager(0, 3);
    mgrRegistry.register("reviewer", manager);

    // Create 5 sessions, suspend all
    for (int i = 0; i < 5; i++) {
        var s = manager.acquireSession("r" + i, "/ws/" + i, null, WorkingDirPolicy.SHARED_READ);
        manager.suspendSession(s.instanceId());
    }

    var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
    scheduler.tick();

    // Should only resume 3 (maxActive ceiling)
    assertThat(manager.activeCount()).isEqualTo(3);
}

@Test
void proactive_resumesMostRecentFirst() {
    var def = AgentPoolDefinition.builder()
        .agent("reviewer")
        .pool()
            .maxActive(10)
            .scaling(new ScalingConfig.ProactiveConfig(1, null, null))
        .build();
    defRegistry.register(def);

    var manager = createManager(0, 10);
    mgrRegistry.register("reviewer", manager);

    // Create 3 sessions with different recency
    var s1 = manager.acquireSession("oldest", "/ws/1", null, WorkingDirPolicy.SHARED_READ);
    var s2 = manager.acquireSession("middle", "/ws/2", null, WorkingDirPolicy.SHARED_READ);
    var s3 = manager.acquireSession("newest", "/ws/3", null, WorkingDirPolicy.SHARED_READ);

    // Suspend in order — s3 is suspended last so has most recent lastInteraction
    manager.suspendSession(s1.instanceId());
    manager.suspendSession(s2.instanceId());
    manager.suspendSession(s3.instanceId());

    var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
    scheduler.tick();

    // Should resume s3 (most recent lastInteraction)
    assertThat(manager.activeCount()).isEqualTo(1);
    assertThat(manager.getSession(s3.instanceId()).state()).isEqualTo(SessionState.ACTIVE);
    assertThat(manager.getSession(s1.instanceId()).state()).isEqualTo(SessionState.SUSPENDED);
    assertThat(manager.getSession(s2.instanceId()).state()).isEqualTo(SessionState.SUSPENDED);
}

@Test
void proactive_cooldownPreventsRapidWarming() {
    var def = AgentPoolDefinition.builder()
        .agent("reviewer")
        .pool()
            .maxActive(10)
            .scaling(new ScalingConfig.ProactiveConfig(2, Duration.ofMinutes(5), null))
        .build();
    defRegistry.register(def);

    var manager = createManager(0, 10);
    mgrRegistry.register("reviewer", manager);

    // Create and suspend 3 sessions
    var s1 = manager.acquireSession("r1", "/ws/1", null, WorkingDirPolicy.SHARED_READ);
    var s2 = manager.acquireSession("r2", "/ws/2", null, WorkingDirPolicy.SHARED_READ);
    var s3 = manager.acquireSession("r3", "/ws/3", null, WorkingDirPolicy.SHARED_READ);
    manager.suspendSession(s1.instanceId());
    manager.suspendSession(s2.instanceId());
    manager.suspendSession(s3.instanceId());

    var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);

    // First tick resumes 2
    scheduler.tick();
    assertThat(manager.activeCount()).isEqualTo(2);

    // Suspend them again
    manager.sessions().stream()
        .filter(s -> s.state() == SessionState.ACTIVE)
        .forEach(s -> manager.suspendSession(s.instanceId()));
    assertThat(manager.activeCount()).isZero();

    // Second tick within cooldown should NOT resume
    scheduler.tick();
    assertThat(manager.activeCount()).isZero();
}

@Test
void proactive_emitsScalingEvent() {
    var def = AgentPoolDefinition.builder()
        .agent("reviewer")
        .pool()
            .maxActive(10)
            .scaling(new ScalingConfig.ProactiveConfig(1, null, null))
        .build();
    defRegistry.register(def);

    var manager = createManager(0, 10);
    mgrRegistry.register("reviewer", manager);

    var s1 = manager.acquireSession("r1", "/ws/1", null, WorkingDirPolicy.SHARED_READ);
    manager.suspendSession(s1.instanceId());

    var emittedDecisions = new java.util.ArrayList<ScalingDecision>();
    PoolEventEmitter emitter = new PoolEventEmitter() {
        @Override
        public void emitScalingDecision(String pool, ScalingDecision d, int prevMax, int newMax) {
            emittedDecisions.add(d);
        }
    };

    var scheduler = new ScalingScheduler(defRegistry, mgrRegistry, emitter);
    scheduler.tick();

    assertThat(emittedDecisions).hasSize(1);
    assertThat(emittedDecisions.get(0).direction()).isEqualTo(ScalingDirection.PREWARM);
    assertThat(emittedDecisions.get(0).count()).isEqualTo(1);
}
```

- [ ] **Step 9: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ProactiveScalingTest`
Expected: All 6 tests PASS

- [ ] **Step 10: Run existing scaling tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ScalingSchedulerTest`
Expected: All existing tests PASS

- [ ] **Step 11: Run diagnostics**

Use `ide_diagnostics` on `ScalingConfig.java`, `ScalingDirection.java`, `ScalingScheduler.java`.

- [ ] **Step 12: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingDirection.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/ProactiveScalingTest.java
git commit -m "feat(#243): ProactiveConfig + PREWARM direction + scheduler evaluation

Adds identity-aware proactive scaling: resumes most-recently-suspended
sessions to maintain a target active count. 6 tests cover resume,
no-action, max ceiling, recency ordering, cooldown, and event emission.

Refs #243"
```

---

## Batch 2: YAML parsing + REST API + CLAUDE.md

### Task 2: YAML parser, PoolService, REST API, and docs

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PoolUpdateRequest.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/ScalingConfigView.java`
- Modify: `CLAUDE.md`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ProactiveYamlParserTest.java`

**Interfaces:**
- Consumes: `ScalingConfig.ProactiveConfig(int, Duration, Duration)` from Task 1
- Produces: YAML `type: proactive` parsing with `target-active` field
- Produces: REST `"proactive"` case in `PoolService.parseScalingConfig()`
- Produces: `PoolUpdateRequest.targetActive()` field
- Produces: `ScalingConfigView.targetActive()` field

- [ ] **Step 1: Write the failing test — YAML parser handles proactive type**

Create test file `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ProactiveYamlParserTest.java`:

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ProactiveYamlParserTest {

    @Test
    void parseProactiveScaling() {
        var yaml = """
                agent-pools:
                  warm-pool:
                    working-dir: /tmp
                    pool:
                      min-active: 0
                      max-active: 10
                      scaling:
                        type: proactive
                        target-active: 3
                        cooldown: 30s
                """;

        var parser = new AgentPoolYamlParser();
        var defs = parser.parse(yaml);

        assertThat(defs).hasSize(1);
        var scaling = defs.get(0).pool().scaling();
        assertThat(scaling).isInstanceOf(ScalingConfig.ProactiveConfig.class);
        var proactive = (ScalingConfig.ProactiveConfig) scaling;
        assertThat(proactive.targetActive()).isEqualTo(3);
        assertThat(proactive.cooldown().getSeconds()).isEqualTo(30);
    }

    @Test
    void parseProactiveScaling_defaultCooldown() {
        var yaml = """
                agent-pools:
                  warm-pool:
                    working-dir: /tmp
                    pool:
                      min-active: 0
                      max-active: 10
                      scaling:
                        type: proactive
                        target-active: 5
                """;

        var parser = new AgentPoolYamlParser();
        var defs = parser.parse(yaml);

        var proactive = (ScalingConfig.ProactiveConfig) defs.get(0).pool().scaling();
        assertThat(proactive.targetActive()).isEqualTo(5);
        assertThat(proactive.cooldown().getSeconds()).isEqualTo(60);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ProactiveYamlParserTest`
Expected: Failure — parser doesn't handle `"proactive"` type.

- [ ] **Step 3: Add proactive case to AgentPoolYamlParser.parseScaling()**

In the `switch (type)` block in `parseScaling()`, add before the `default` case:

```java
case "proactive" -> {
    var targetActive = scalingMap.get("target-active");
    if (targetActive == null) {
        throw new IllegalArgumentException("proactive scaling requires a 'target-active' field");
    }
    int target = targetActive instanceof Number n ? n.intValue() : Integer.parseInt(targetActive.toString());
    yield new ScalingConfig.ProactiveConfig(target, cooldown, scaleInCooldown);
}
```

- [ ] **Step 4: Run YAML parser tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ProactiveYamlParserTest`
Expected: PASS

- [ ] **Step 5: Add targetActive to PoolUpdateRequest**

Use `ide_edit_member` to replace the record:

```java
public record PoolUpdateRequest(
    Integer minActive,
    Integer maxActive,
    String scalingType,
    Double targetFillRatio,
    List<ScalingStepInput> steps,
    Integer exhaustionThreshold,
    Long latencyThresholdMs,
    Integer targetActive,
    String cooldown,
    String scaleInCooldown
) {}
```

- [ ] **Step 6: Add targetActive to ScalingConfigView**

Use `ide_edit_member` to replace the record:

```java
public record ScalingConfigView(
    Double targetFillRatio,
    List<ScalingStepView> steps,
    Integer exhaustionThreshold,
    Long latencyThresholdMs,
    String beanName,
    Integer targetActive,
    String cooldown,
    String scaleInCooldown
) {
    public record ScalingStepView(double threshold, int adjustment) {}
}
```

- [ ] **Step 7: Add proactive case to PoolService.parseScalingConfig()**

In the `switch (u.scalingType())` block, add before `"none"`:

```java
case "proactive" -> {
    if (u.targetActive() == null) {
        throw new IllegalArgumentException("targetActive required for proactive scaling");
    }
    yield new ScalingConfig.ProactiveConfig(u.targetActive(), cooldown, scaleInCooldown);
}
```

- [ ] **Step 8: Add targetActive to PoolService.buildScalingConfigView()**

In the `switch (config)` block, add before `NoScalingConfig`:

```java
case ScalingConfig.ProactiveConfig p -> new ScalingConfigView(
        null, null, null, null, null, p.targetActive(),
        formatDuration(p.cooldown()), formatDuration(p.scaleInCooldown()));
```

Update the other cases to pass `null` for the new `targetActive` parameter
(6th argument). For example, `TargetTrackingConfig`:
```java
case ScalingConfig.TargetTrackingConfig t -> new ScalingConfigView(
        t.targetFillRatio(), null, null, null, null, null,
        formatDuration(t.cooldown()), formatDuration(t.scaleInCooldown()));
```

Do the same for `StepConfig`, `DemandPressureConfig`, `CustomScalingConfig`,
and `NoScalingConfig` — add `null` as the 6th argument (targetActive).

- [ ] **Step 9: Run existing PoolServiceTest to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --also-make -Dtest=PoolServiceTest -Dsurefire.failIfNoSpecifiedTests=false -Denforcer.skip=true`
Expected: All existing tests PASS

- [ ] **Step 10: Update CLAUDE.md**

Add to configuration properties:
```properties
# Proactive scaling (pool pre-warming)
# scaling.type=proactive target-active=3 cooldown=30s
```

Update test count: add ProactiveScalingTest (6) + ProactiveYamlParserTest (2).

Add `ProactiveConfig` to `ScalingConfig` description in project structure.

- [ ] **Step 11: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolUpdateRequest.java \
       app/src/main/java/io/casehub/claudony/server/fleet/ScalingConfigView.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/ProactiveYamlParserTest.java \
       CLAUDE.md
git commit -m "feat(#243): YAML + REST API support for proactive scaling

Parses 'type: proactive' with 'target-active' in YAML definitions.
PoolService handles proactive updates via REST. ScalingConfigView
surfaces targetActive in dashboard API.

Closes #243"
```

## References

- [2026-10-02-proactive-scaling-design.md] — design spec this plan implements
- [casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java] — sealed hierarchy
- [casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java] — tick loop
- [casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingPolicy.java] — reactive policy (unchanged)
- [casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java] — YAML parsing
- [casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingSchedulerTest.java] — existing test pattern
- [app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java] — REST scaling config parsing
- [GitHub #243] — proactive scaling issue
- [GitHub #206] — auto-scaling policies spec (deferred proactive)
