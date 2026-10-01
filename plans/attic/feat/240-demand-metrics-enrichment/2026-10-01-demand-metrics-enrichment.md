# Demand Metrics Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #240 — queue-depth and external metrics enrichment for PoolSnapshot.DemandMetrics
**Issue group:** #240

**Goal:** Enrich DemandMetrics with acquire latency, an external metrics SPI, and a demand-pressure scaling policy.

**Architecture:** Add latency tracking fields to `DemandMetrics`, a `DemandMetricsSource` SPI for external metric injection, and a `DemandPressurePolicy` that triggers on exhaustion count or latency thresholds. All changes are within `casehub/fleet` — no cross-module impact.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, AssertJ, JUnit 5, Micrometer

## Global Constraints

- Java release target: 21
- All records are immutable — no mutable fields on records
- `DemandMetrics.ZERO` must be updated whenever fields change
- Existing tests must continue to compile using the no-arg `snapshotAndResetDemandMetrics()` overload
- All new code in `casehub/src/main/java/io/casehub/claudony/casehub/fleet/`
- All new tests in `casehub/src/test/java/io/casehub/claudony/casehub/fleet/`
- Test command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`

---

## Batch 1: Enriched DemandMetrics and latency tracking

### Task 1: Enrich DemandMetrics record with latency fields and external metrics map

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSnapshot.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolSnapshotTest.java`

**Interfaces:**
- Produces: `DemandMetrics(int evictions, int exhaustions, int acquires, long averageAcquireNanos, long maxAcquireNanos, Map<String, Double> externalMetrics)`, `DemandMetrics.ZERO`, `averageAcquireMs()`, `maxAcquireMs()`

- [ ] **Step 1: Write failing tests for enriched DemandMetrics**

Add to `PoolSnapshotTest.java`:

```java
@Test
void demandMetricsZeroIncludesLatencyAndExternalFields() {
    var zero = PoolSnapshot.DemandMetrics.ZERO;
    assertThat(zero.averageAcquireNanos()).isZero();
    assertThat(zero.maxAcquireNanos()).isZero();
    assertThat(zero.externalMetrics()).isEmpty();
}

@Test
void demandMetricsAcquireMsConvertsFromNanos() {
    var metrics = new PoolSnapshot.DemandMetrics(0, 0, 0,
        5_000_000L, 12_000_000L, Map.of());
    assertThat(metrics.averageAcquireMs()).isEqualTo(5L);
    assertThat(metrics.maxAcquireMs()).isEqualTo(12L);
}

@Test
void demandMetricsExternalMetricsImmutable() {
    var mutable = new java.util.HashMap<String, Double>();
    mutable.put("http.queue_depth", 3.0);
    var metrics = new PoolSnapshot.DemandMetrics(0, 0, 0, 0L, 0L,
        Map.copyOf(mutable));
    mutable.put("injected", 1.0);
    assertThat(metrics.externalMetrics()).doesNotContainKey("injected");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=PoolSnapshotTest`
Expected: compilation errors — `DemandMetrics` constructor has wrong arity

- [ ] **Step 3: Update DemandMetrics record**

Use `ide_replace_member` to replace the `DemandMetrics` record inside `PoolSnapshot`:

```java
public record DemandMetrics(
    int evictions,
    int exhaustions,
    int acquires,
    long averageAcquireNanos,
    long maxAcquireNanos,
    Map<String, Double> externalMetrics
) {
    public static final DemandMetrics ZERO =
        new DemandMetrics(0, 0, 0, 0L, 0L, Map.of());

    public long averageAcquireMs() {
        return averageAcquireNanos / 1_000_000;
    }

    public long maxAcquireMs() {
        return maxAcquireNanos / 1_000_000;
    }
}
```

Add `import java.util.Map;` to `PoolSnapshot.java` if not present.

- [ ] **Step 4: Fix existing code that constructs DemandMetrics**

Two callers construct `DemandMetrics` directly:

1. `AgentSessionManager.snapshotAndResetDemandMetrics()` (line 140) — will be updated in Task 2.
2. Existing test code using `new PoolSnapshot.DemandMetrics(evictions, exhaustions, acquires)` — update to 6-arg constructor with `0L, 0L, Map.of()` appended.

Use `ide_find_references` on the `DemandMetrics` constructor to find all callers. Update each to the new 6-arg signature.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=PoolSnapshotTest`
Expected: all 8 tests PASS (5 existing + 3 new)

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSnapshot.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolSnapshotTest.java
git commit -m "feat(#240): enrich DemandMetrics with latency fields and external metrics map

Refs #240"
```

### Task 2: Add latency tracking to AgentSessionManager

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentSessionManagerTest.java`

**Interfaces:**
- Consumes: `DemandMetrics` 6-arg constructor from Task 1
- Produces: `snapshotAndResetDemandMetrics(Map<String, Double> externalMetrics)`, `snapshotAndResetDemandMetrics()` (no-arg backward-compat overload), `lastDemandSnapshot()` accessor

- [ ] **Step 1: Write failing tests for latency tracking**

Add to `AgentSessionManagerTest.java`:

```java
@Test
void snapshotDemandMetrics_includesLatencyAfterAcquires() {
    var manager = createManager(0, 5);
    manager.acquireSession("a", "/tmp/a");
    manager.acquireSession("b", "/tmp/b");
    var metrics = manager.snapshotAndResetDemandMetrics();
    assertThat(metrics.averageAcquireNanos()).isGreaterThan(0);
    assertThat(metrics.maxAcquireNanos()).isGreaterThanOrEqualTo(
        metrics.averageAcquireNanos());
}

@Test
void snapshotDemandMetrics_resetsLatencyCounters() {
    var manager = createManager(0, 5);
    manager.acquireSession("a", "/tmp/a");
    manager.snapshotAndResetDemandMetrics();
    var after = manager.snapshotAndResetDemandMetrics();
    assertThat(after.averageAcquireNanos()).isZero();
    assertThat(after.maxAcquireNanos()).isZero();
}

@Test
void snapshotDemandMetrics_acceptsExternalMetrics() {
    var manager = createManager(0, 5);
    manager.acquireSession("a", "/tmp/a");
    var ext = Map.of("http.queue_depth", 5.0);
    var metrics = manager.snapshotAndResetDemandMetrics(ext);
    assertThat(metrics.externalMetrics()).containsEntry("http.queue_depth", 5.0);
}

@Test
void snapshotDemandMetrics_noArgOverloadReturnsEmptyExternalMetrics() {
    var manager = createManager(0, 5);
    manager.acquireSession("a", "/tmp/a");
    var metrics = manager.snapshotAndResetDemandMetrics();
    assertThat(metrics.externalMetrics()).isEmpty();
}

@Test
void lastDemandSnapshot_returnsLatestSnapshot() {
    var manager = createManager(0, 5);
    manager.acquireSession("a", "/tmp/a");
    var snapshot = manager.snapshotAndResetDemandMetrics();
    assertThat(manager.lastDemandSnapshot()).isSameAs(snapshot);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentSessionManagerTest`
Expected: compilation errors — new method signatures don't exist yet

- [ ] **Step 3: Add latency fields and update snapshotAndResetDemandMetrics**

Use `ide_edit_member` to add fields after `exhaustionCount`:

```java
private long totalAcquireNanos;
private long maxAcquireNanos;
private volatile PoolSnapshot.DemandMetrics lastDemandSnapshot = PoolSnapshot.DemandMetrics.ZERO;
```

Use `ide_replace_member` to replace the 4-arg `acquireSession` method. Add `long start = System.nanoTime();` before `lock.lock()`. Before each `return` statement inside the try block, add:

```java
long elapsed = System.nanoTime() - start;
totalAcquireNanos += elapsed;
maxAcquireNanos = Math.max(maxAcquireNanos, elapsed);
```

Use `ide_replace_member` to replace `snapshotAndResetDemandMetrics`:

```java
public PoolSnapshot.DemandMetrics snapshotAndResetDemandMetrics(
        Map<String, Double> externalMetrics) {
    lock.lock();
    try {
        long avgNanos = acquireCount > 0 ? totalAcquireNanos / acquireCount : 0;
        var metrics = new PoolSnapshot.DemandMetrics(
            evictionCount, exhaustionCount, acquireCount,
            avgNanos, maxAcquireNanos,
            externalMetrics != null ? externalMetrics : Map.of());
        evictionCount = 0;
        exhaustionCount = 0;
        acquireCount = 0;
        totalAcquireNanos = 0;
        maxAcquireNanos = 0;
        lastDemandSnapshot = metrics;
        return metrics;
    } finally {
        lock.unlock();
    }
}

public PoolSnapshot.DemandMetrics snapshotAndResetDemandMetrics() {
    return snapshotAndResetDemandMetrics(Map.of());
}

public PoolSnapshot.DemandMetrics lastDemandSnapshot() {
    return lastDemandSnapshot;
}
```

Add `import java.util.Map;` if not present.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentSessionManagerTest`
Expected: all tests PASS (existing + 5 new)

- [ ] **Step 5: Run full module tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`
Expected: all tests PASS — existing tests use no-arg overload unchanged

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentSessionManagerTest.java
git commit -m "feat(#240): add latency tracking to AgentSessionManager

Capture System.nanoTime() around acquireSession() including lock wait.
Add snapshotAndResetDemandMetrics(Map) overload for external metrics.
Retain no-arg overload for backward compatibility.

Refs #240"
```

---

## Batch 2: DemandMetricsSource SPI and DemandPressurePolicy

### Task 3: Create DemandMetricsSource SPI and wire into ScalingScheduler

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/DemandMetricsSource.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingSchedulerTest.java`

**Interfaces:**
- Consumes: `snapshotAndResetDemandMetrics(Map<String, Double>)` from Task 2
- Produces: `DemandMetricsSource` interface, `ScalingScheduler` external metrics collection

- [ ] **Step 1: Write failing test for external metrics in ScalingScheduler**

Add to `ScalingSchedulerTest.java`:

```java
@Test
void tickCollectsExternalMetricsAndPassesToSnapshot() {
    var def = AgentPoolDefinition.builder()
        .agent("test").pool().maxActive(10)
        .scaling(new ScalingConfig.TargetTrackingConfig(0.7, null, null))
        .build();
    defRegistry.register(def);

    var manager = createManager(0, 10);
    mgrRegistry.register("test", manager);
    manager.acquireSession("test", "/ws/1", null, WorkingDirPolicy.SHARED_READ);

    DemandMetricsSource source = poolName ->
        Map.of("http.queue_depth", 3.0);

    var scheduler = new ScalingScheduler(defRegistry, mgrRegistry,
        null, List.of(source));
    scheduler.tick();

    var lastSnapshot = manager.lastDemandSnapshot();
    assertThat(lastSnapshot.externalMetrics())
        .containsEntry("http.queue_depth", 3.0);
}

@Test
void tickHandlesFailingMetricsSourceGracefully() {
    var def = AgentPoolDefinition.builder()
        .agent("test").pool().maxActive(10)
        .scaling(new ScalingConfig.TargetTrackingConfig(0.7, null, null))
        .build();
    defRegistry.register(def);

    var manager = createManager(0, 10);
    mgrRegistry.register("test", manager);
    manager.acquireSession("test", "/ws/1", null, WorkingDirPolicy.SHARED_READ);

    DemandMetricsSource failingSource = poolName -> {
        throw new RuntimeException("source down");
    };
    DemandMetricsSource goodSource = poolName ->
        Map.of("healthy", 1.0);

    var scheduler = new ScalingScheduler(defRegistry, mgrRegistry,
        null, List.of(failingSource, goodSource));
    scheduler.tick();

    var lastSnapshot = manager.lastDemandSnapshot();
    assertThat(lastSnapshot.externalMetrics())
        .containsEntry("healthy", 1.0)
        .doesNotContainKey("broken");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ScalingSchedulerTest`
Expected: compilation errors — `DemandMetricsSource` doesn't exist, constructor signature wrong

- [ ] **Step 3: Create DemandMetricsSource interface**

Create new file `casehub/src/main/java/io/casehub/claudony/casehub/fleet/DemandMetricsSource.java`:

```java
package io.casehub.claudony.casehub.fleet;

import java.util.Map;

public interface DemandMetricsSource {
    Map<String, Double> collect(String poolName);
}
```

- [ ] **Step 4: Update ScalingScheduler to collect external metrics**

Add a `metricsSources` field:

```java
private final List<DemandMetricsSource> metricsSources;
```

Add a test-friendly constructor that accepts sources directly:

```java
ScalingScheduler(AgentPoolDefinitionRegistry defRegistry,
                 AgentPoolManagerRegistry mgrRegistry,
                 PoolEventEmitter eventEmitter,
                 List<DemandMetricsSource> metricsSources) {
    this.defRegistry  = defRegistry;
    this.mgrRegistry  = mgrRegistry;
    this.eventEmitter = eventEmitter;
    this.metricsSources = metricsSources != null ? metricsSources : List.of();
}
```

Update the CDI constructor to accept `Instance<DemandMetricsSource>`:

```java
@Inject
public ScalingScheduler(AgentPoolDefinitionRegistry defRegistry,
                        AgentPoolManagerRegistry mgrRegistry,
                        jakarta.enterprise.inject.Instance<PoolEventEmitter> eventEmitterInstance,
                        jakarta.enterprise.inject.Instance<DemandMetricsSource> metricsSourcesInstance) {
    this.defRegistry  = defRegistry;
    this.mgrRegistry  = mgrRegistry;
    this.eventEmitter = eventEmitterInstance.isUnsatisfied() ? null : eventEmitterInstance.get();
    this.metricsSources = metricsSourcesInstance.stream().toList();
}
```

Update the existing 2-arg test constructor to set `metricsSources = List.of()`.

Add `collectExternalMetrics` method:

```java
private Map<String, Double> collectExternalMetrics(String poolName) {
    if (metricsSources.isEmpty()) return Map.of();
    var merged = new java.util.HashMap<String, Double>();
    for (var source : metricsSources) {
        try {
            var metrics = source.collect(poolName);
            if (metrics != null) merged.putAll(metrics);
        } catch (Exception e) {
            LOG.warning("DemandMetricsSource failed for pool '"
                + poolName + "': " + e.getMessage());
        }
    }
    return Map.copyOf(merged);
}
```

Update `evaluatePool()`: change `manager.snapshotAndResetDemandMetrics()` to:

```java
var externalMetrics = collectExternalMetrics(poolName);
var demand = manager.snapshotAndResetDemandMetrics(externalMetrics);
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ScalingSchedulerTest`
Expected: all tests PASS (7 existing + 2 new)

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/DemandMetricsSource.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingSchedulerTest.java
git commit -m "feat(#240): add DemandMetricsSource SPI and wire into ScalingScheduler

Pull-based interface called per tick. Failing sources are logged and
skipped. External metrics passed through to DemandMetrics.

Refs #240"
```

### Task 4: DemandPressurePolicy and DemandPressureConfig

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/DemandPressurePolicy.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java` (createPolicy switch)
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/DemandPressurePolicyTest.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingConfigTest.java`

**Interfaces:**
- Consumes: `DemandMetrics.exhaustions()`, `DemandMetrics.averageAcquireNanos()`, `PoolSnapshot.maxActive()`, `PoolSnapshot.minActive()` from Task 1
- Produces: `DemandPressurePolicy`, `ScalingConfig.DemandPressureConfig`

- [ ] **Step 1: Write failing tests for DemandPressurePolicy**

Create `casehub/src/test/java/io/casehub/claudony/casehub/fleet/DemandPressurePolicyTest.java`:

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DemandPressurePolicyTest {

    private final DemandPressurePolicy policy =
        new DemandPressurePolicy(2, 500);

    private PoolSnapshot snap(int active, int max, int min,
            int exhaustions, long avgNanos) {
        return new PoolSnapshot(active, 0, min, max,
            new PoolSnapshot.DemandMetrics(0, exhaustions, 0,
                avgNanos, avgNanos, Map.of()));
    }

    @Test
    void scalesOutWhenExhaustionsExceedThreshold() {
        var decision = policy.evaluate(snap(10, 10, 2, 3, 0));
        assertThat(decision.direction()).isEqualTo(ScalingDirection.OUT);
        assertThat(decision.count()).isEqualTo(1);
    }

    @Test
    void scalesOutWhenLatencyExceedsThreshold() {
        var decision = policy.evaluate(
            snap(5, 10, 2, 0, 600_000_000L));
        assertThat(decision.direction()).isEqualTo(ScalingDirection.OUT);
        assertThat(decision.count()).isEqualTo(1);
    }

    @Test
    void exhaustionTakesPriorityOverLatency() {
        var decision = policy.evaluate(
            snap(10, 10, 2, 5, 600_000_000L));
        assertThat(decision.direction()).isEqualTo(ScalingDirection.OUT);
        assertThat(decision.reason()).contains("exhaustions");
    }

    @Test
    void noActionWhenBelowThresholdsButAboveHysteresis() {
        var decision = policy.evaluate(
            snap(5, 10, 2, 1, 300_000_000L));
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
    }

    @Test
    void scalesInWhenBothBelowHysteresisBand() {
        var decision = policy.evaluate(
            snap(5, 10, 2, 0, 100_000_000L));
        assertThat(decision.direction()).isEqualTo(ScalingDirection.IN);
        assertThat(decision.count()).isEqualTo(1);
    }

    @Test
    void noScaleInWhenAtMinActive() {
        var decision = policy.evaluate(
            snap(2, 2, 2, 0, 0));
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
    }

    @Test
    void exactlyAtExhaustionThresholdDoesNotTrigger() {
        var decision = policy.evaluate(snap(5, 10, 2, 2, 0));
        assertThat(decision.direction()).isNotEqualTo(ScalingDirection.OUT);
    }

    @Test
    void rejectsNegativeExhaustionThreshold() {
        assertThatThrownBy(() -> new DemandPressurePolicy(-1, 500))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsZeroLatencyThreshold() {
        assertThatThrownBy(() -> new DemandPressurePolicy(2, 0))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Write failing tests for DemandPressureConfig**

Add to `ScalingConfigTest.java`:

```java
@Test
void demandPressureConfigValidatesFields() {
    var config = new ScalingConfig.DemandPressureConfig(2, 500, null, null);
    assertThat(config.exhaustionThreshold()).isEqualTo(2);
    assertThat(config.latencyThresholdMs()).isEqualTo(500);
    assertThat(config.cooldown()).isEqualTo(Duration.ofSeconds(60));
}

@Test
void demandPressureConfigType() {
    var config = new ScalingConfig.DemandPressureConfig(1, 100, null, null);
    assertThat(config.type()).isEqualTo("demand-pressure");
}

@Test
void demandPressureConfigRejectsNegativeExhaustionThreshold() {
    assertThatThrownBy(() ->
        new ScalingConfig.DemandPressureConfig(-1, 500, null, null))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test
void demandPressureConfigRejectsZeroLatencyThreshold() {
    assertThatThrownBy(() ->
        new ScalingConfig.DemandPressureConfig(2, 0, null, null))
        .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=DemandPressurePolicyTest,ScalingConfigTest`
Expected: compilation errors — classes don't exist

- [ ] **Step 4: Add DemandPressureConfig to ScalingConfig sealed hierarchy**

Use `ide_edit_member` to add `DemandPressureConfig` as a new permit on the `ScalingConfig` sealed interface. Update the `permits` clause:

```java
public sealed interface ScalingConfig
    permits ScalingConfig.TargetTrackingConfig,
            ScalingConfig.StepConfig,
            ScalingConfig.CustomScalingConfig,
            ScalingConfig.NoScalingConfig,
            ScalingConfig.DemandPressureConfig {
```

Use `ide_insert_member` to add the new record inside `ScalingConfig`:

```java
record DemandPressureConfig(
    int exhaustionThreshold,
    long latencyThresholdMs,
    Duration cooldown,
    Duration scaleInCooldown
) implements ScalingConfig {
    public DemandPressureConfig {
        if (exhaustionThreshold < 0)
            throw new IllegalArgumentException(
                "exhaustionThreshold must be non-negative");
        if (latencyThresholdMs <= 0)
            throw new IllegalArgumentException(
                "latencyThresholdMs must be positive");
        if (cooldown == null) cooldown = Duration.ofSeconds(60);
        if (scaleInCooldown == null) scaleInCooldown = cooldown;
    }
}
```

Update the `type()` switch in `ScalingConfig` to add the new case:

```java
case DemandPressureConfig d -> "demand-pressure";
```

- [ ] **Step 5: Create DemandPressurePolicy**

Create `casehub/src/main/java/io/casehub/claudony/casehub/fleet/DemandPressurePolicy.java`:

```java
package io.casehub.claudony.casehub.fleet;

public class DemandPressurePolicy implements ScalingPolicy {

    private final int exhaustionThreshold;
    private final long latencyThresholdNanos;

    public DemandPressurePolicy(int exhaustionThreshold,
                                long latencyThresholdMs) {
        if (exhaustionThreshold < 0)
            throw new IllegalArgumentException(
                "exhaustionThreshold must be non-negative");
        if (latencyThresholdMs <= 0)
            throw new IllegalArgumentException(
                "latencyThresholdMs must be positive");
        this.exhaustionThreshold = exhaustionThreshold;
        this.latencyThresholdNanos = latencyThresholdMs * 1_000_000;
    }

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        var demand = snapshot.demand();
        boolean exhaustionTriggered =
            demand.exhaustions() > exhaustionThreshold;
        boolean latencyTriggered =
            demand.averageAcquireNanos() > latencyThresholdNanos;

        if (exhaustionTriggered) {
            return ScalingDecision.scaleOut(1,
                "exhaustions %d > threshold %d"
                    .formatted(demand.exhaustions(),
                               exhaustionThreshold));
        }
        if (latencyTriggered) {
            return ScalingDecision.scaleOut(1,
                "avgAcquireLatency %dms > threshold %dms"
                    .formatted(demand.averageAcquireMs(),
                               latencyThresholdNanos / 1_000_000));
        }

        boolean exhaustionQuiet =
            demand.exhaustions() <= exhaustionThreshold / 2;
        boolean latencyQuiet =
            demand.averageAcquireNanos() <= latencyThresholdNanos / 2;

        if (exhaustionQuiet && latencyQuiet
                && snapshot.maxActive() > snapshot.minActive()) {
            return ScalingDecision.scaleIn(1,
                "demand pressure below hysteresis band");
        }

        return ScalingDecision.none();
    }
}
```

- [ ] **Step 6: Wire DemandPressureConfig into ScalingScheduler.createPolicy()**

Use `ide_replace_member` to update `createPolicy` — add the case:

```java
case ScalingConfig.DemandPressureConfig d ->
    new DemandPressurePolicy(d.exhaustionThreshold(),
                             d.latencyThresholdMs());
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=DemandPressurePolicyTest,ScalingConfigTest`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/DemandPressurePolicy.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/DemandPressurePolicyTest.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingConfigTest.java
git commit -m "feat(#240): add DemandPressurePolicy and DemandPressureConfig

Independent OR triggers for scale-out (exhaustions OR latency).
AND for scale-in (both below 50% hysteresis band).
New sealed variant on ScalingConfig.

Refs #240"
```

---

## Batch 3: YAML parsing, schema, metrics export, and integration test

### Task 5: YAML parsing and schema for demand-pressure type

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParserTest.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolSchemaTest.java`

**Interfaces:**
- Consumes: `ScalingConfig.DemandPressureConfig` from Task 4

- [ ] **Step 1: Write failing test for YAML parsing**

Add to `AgentPoolYamlParserTest.java`:

```java
@Test
void parsesDemandPressureScalingType() {
    var yaml = """
        agent-pools:
          reviewer:
            command: claude
            pool:
              max-active: 10
              scaling:
                type: demand-pressure
                exhaustion-threshold: 2
                latency-threshold-ms: 500
                cooldown: 60s
                scale-in-cooldown: 300s
        """;
    var defs = new AgentPoolYamlParser().parse(yaml);
    var scaling = defs.get(0).pool().scaling();
    assertThat(scaling).isInstanceOf(ScalingConfig.DemandPressureConfig.class);
    var dp = (ScalingConfig.DemandPressureConfig) scaling;
    assertThat(dp.exhaustionThreshold()).isEqualTo(2);
    assertThat(dp.latencyThresholdMs()).isEqualTo(500);
    assertThat(dp.cooldown()).isEqualTo(Duration.ofSeconds(60));
    assertThat(dp.scaleInCooldown()).isEqualTo(Duration.ofSeconds(300));
}

@Test
void demandPressureMissingThresholdThrows() {
    var yaml = """
        agent-pools:
          reviewer:
            command: claude
            pool:
              max-active: 10
              scaling:
                type: demand-pressure
                exhaustion-threshold: 2
        """;
    assertThatThrownBy(() -> new AgentPoolYamlParser().parse(yaml))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("latency-threshold-ms");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentPoolYamlParserTest`
Expected: FAIL — parser doesn't handle `demand-pressure` type

- [ ] **Step 3: Add demand-pressure case to parseScaling**

Use `ide_replace_member` to update `parseScaling`. Add a new case before the `default`:

```java
case "demand-pressure" -> {
    var exhaustionThreshold = scalingMap.get("exhaustion-threshold");
    if (exhaustionThreshold == null) {
        throw new IllegalArgumentException(
            "demand-pressure scaling requires 'exhaustion-threshold'");
    }
    var latencyThresholdMs = scalingMap.get("latency-threshold-ms");
    if (latencyThresholdMs == null) {
        throw new IllegalArgumentException(
            "demand-pressure scaling requires 'latency-threshold-ms'");
    }
    yield new ScalingConfig.DemandPressureConfig(
        exhaustionThreshold instanceof Number n
            ? n.intValue()
            : Integer.parseInt(exhaustionThreshold.toString()),
        latencyThresholdMs instanceof Number n
            ? n.longValue()
            : Long.parseLong(latencyThresholdMs.toString()),
        cooldown, scaleInCooldown);
}
```

- [ ] **Step 4: Add schema parameters**

Use `ide_edit_member` to add to the `AgentPoolSchema` static initializer, before the `DEFINITION` assignment:

```java
inputs.put("pool.scaling.exhaustion-threshold", new StepParameter(
    StepParameterType.INTEGER, false, null, null, null,
    "Exhaustion count per tick that triggers scale-out (demand-pressure)"));

inputs.put("pool.scaling.latency-threshold-ms", new StepParameter(
    StepParameterType.INTEGER, false, null, null, null,
    "Average acquire latency in ms that triggers scale-out (demand-pressure)"));
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentPoolYamlParserTest,AgentPoolSchemaTest`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParserTest.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolSchemaTest.java
git commit -m "feat(#240): parse demand-pressure scaling type from YAML

Adds exhaustion-threshold and latency-threshold-ms to YAML config
and schema definition.

Refs #240"
```

### Task 6: Metrics export and full regression test

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolMetricsRegistrar.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/IoTDBFlatLabelAdapter.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolMetricsRegistrarTest.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/IoTDBFlatLabelAdapterTest.java`

**Interfaces:**
- Consumes: `AgentSessionManager.lastDemandSnapshot()` from Task 2

- [ ] **Step 1: Write failing test for latency gauges**

Add to `PoolMetricsRegistrarTest.java`:

```java
@Test
void registersAcquireLatencyGauges() {
    registrar.registerPool("test");
    var avgGauge = registry.find("claudony.pool.acquire_latency_avg_ms")
        .tag("pool", "test").gauge();
    var maxGauge = registry.find("claudony.pool.acquire_latency_max_ms")
        .tag("pool", "test").gauge();
    assertThat(avgGauge).isNotNull();
    assertThat(maxGauge).isNotNull();
}
```

- [ ] **Step 2: Write failing test for IoTDB latency columns**

Add to `IoTDBFlatLabelAdapterTest.java`:

```java
@Test
void tickIncludesLatencyColumns() {
    // Register gauges for latency
    registry.gauge("claudony.pool.acquire_latency_avg_ms",
        io.micrometer.core.instrument.Tags.of("pool", "test"),
        1, x -> 5.0);
    registry.gauge("claudony.pool.acquire_latency_max_ms",
        io.micrometer.core.instrument.Tags.of("pool", "test"),
        1, x -> 12.0);
    // Also register base gauges needed for the tick
    registry.gauge("claudony.pool.active",
        io.micrometer.core.instrument.Tags.of("pool", "test"),
        1, x -> 3.0);
    registry.gauge("claudony.pool.idle",
        io.micrometer.core.instrument.Tags.of("pool", "test"),
        1, x -> 1.0);
    registry.gauge("claudony.pool.max",
        io.micrometer.core.instrument.Tags.of("pool", "test"),
        1, x -> 10.0);
    registry.gauge("claudony.pool.fill_ratio",
        io.micrometer.core.instrument.Tags.of("pool", "test"),
        1, x -> 0.3);

    adapter.tick();

    assertThat(executedSql).hasSize(1);
    assertThat(executedSql.get(0))
        .contains("acquire_latency_avg_ms")
        .contains("acquire_latency_max_ms");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=PoolMetricsRegistrarTest,IoTDBFlatLabelAdapterTest`
Expected: FAIL — gauges not registered, columns not in SQL

- [ ] **Step 4: Add latency gauges to PoolMetricsRegistrar**

Use `ide_edit_member` to add to `registerPool()` after the existing gauges:

```java
registry.gauge("claudony.pool.acquire_latency_avg_ms", tags, mgr,
    m -> m.lastDemandSnapshot().averageAcquireMs());
registry.gauge("claudony.pool.acquire_latency_max_ms", tags, mgr,
    m -> m.lastDemandSnapshot().maxAcquireMs());
```

- [ ] **Step 5: Add latency columns to IoTDBFlatLabelAdapter**

Use `ide_replace_member` to update the `tick()` method — add the new gauge reads and SQL columns:

```java
double acquireLatencyAvg = gaugeValue("claudony.pool.acquire_latency_avg_ms", pool);
double acquireLatencyMax = gaugeValue("claudony.pool.acquire_latency_max_ms", pool);
```

Update the SQL template to include the new columns:

```java
var sql = String.format(
    "INSERT INTO pool_metrics(pool, active, idle, max_active, fill_ratio, " +
    "acquires, evictions, exhaustions, acquire_latency_avg_ms, acquire_latency_max_ms) " +
    "VALUES('%s', %d, %d, %d, %.4f, %d, %d, %d, %d, %d)",
    pool, (int) active, (int) idle, (int) maxActive, fillRatio,
    (long) acquires, (long) evictions, (long) exhaustions,
    (long) acquireLatencyAvg, (long) acquireLatencyMax);
```

- [ ] **Step 6: Run all tests to verify everything passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`
Expected: all tests PASS — full module regression green

- [ ] **Step 7: Run app module tests for cross-module regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolMetricsRegistrar.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/IoTDBFlatLabelAdapter.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolMetricsRegistrarTest.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/IoTDBFlatLabelAdapterTest.java
git commit -m "feat(#240): export latency metrics to Micrometer and IoTDB

Add acquire_latency_avg_ms and acquire_latency_max_ms gauges to
PoolMetricsRegistrar. Add columns to IoTDBFlatLabelAdapter SQL.

Closes #240"
```

---

## References

- [2026-10-01-demand-metrics-enrichment-design.md] — design spec this plan implements
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSnapshot.java` — DemandMetrics record
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java:137-148` — snapshotAndResetDemandMetrics
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java` — sealed hierarchy
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java` — tick loop
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/TargetTrackingPolicy.java` — hysteresis pattern
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/StepScalingPolicy.java` — step threshold pattern
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java:104-144` — parseScaling switch
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java` — schema definition
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolMetricsRegistrar.java` — Micrometer gauges
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/IoTDBFlatLabelAdapter.java` — IoTDB export
- `specs/feat/206-auto-scaling-policies/2026-09-29-auto-scaling-policies-design.md` — parent spec
- GitHub #240 — focal issue
- GitHub #206 — parent issue (auto-scaling policies)
