# Queue-Depth and External Metrics Enrichment for PoolSnapshot.DemandMetrics

**Issue:** #240
**Deferred from:** #206 (auto-scaling policies spec)
**Date:** 2026-10-01

## Summary

Enrich `PoolSnapshot.DemandMetrics` with acquire latency tracking, an SPI for external metric sources, and a built-in `DemandPressurePolicy` that reacts to exhaustion counts and latency. This enables scaling policies that go beyond fill-ratio-only decisions.

The current `DemandMetrics` has three per-tick counters (`evictions`, `exhaustions`, `acquires`) but no built-in policy uses them — both `TargetTrackingPolicy` and `StepScalingPolicy` only read `fillRatio()`. This issue adds metrics that matter for demand-driven scaling and a policy that consumes them.

## Design

### Enriched DemandMetrics

Add latency fields and an external metrics map to the existing record:

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

Latency is stored as nanoseconds internally (from `System.nanoTime()`) with millisecond convenience accessors. The `externalMetrics` map is immutable (`Map.of()` or `Map.copyOf()`). `ZERO` is updated to include the new fields.

### Latency Tracking in AgentSessionManager

Add two running counters to `AgentSessionManager`, mutated under the existing lock:

```java
private long totalAcquireNanos;
private long maxAcquireNanos;
```

In each `acquireSession()` overload, capture `System.nanoTime()` before and after the body (but inside the lock, so lock contention is included — this is the caller's experienced latency):

```java
public ManagedSession acquireSession(String identity, String workingDir,
                                     String command, WorkingDirPolicy policy) {
    long start = System.nanoTime();
    lock.lock();
    try {
        acquireCount++;
        // ... existing acquire logic ...
        long elapsed = System.nanoTime() - start;
        totalAcquireNanos += elapsed;
        maxAcquireNanos = Math.max(maxAcquireNanos, elapsed);
        return session;
    } finally {
        lock.unlock();
    }
}
```

Note: `start` is captured *before* `lock.lock()` so lock wait time is included in the measurement. This is intentional — from the caller's perspective, lock contention is part of the acquire cost.

Update `snapshotAndResetDemandMetrics()` to include latency and accept external metrics:

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
        return metrics;
    } finally {
        lock.unlock();
    }
}
```

The external metrics map is passed in rather than stored — the `ScalingScheduler` collects external metrics and passes them through. `AgentSessionManager` stays unaware of the SPI.

The no-arg `snapshotAndResetDemandMetrics()` overload is retained as a convenience that delegates with `Map.of()` — used by tests and `claudony-testing` utilities that don't need external metrics.

### DemandMetricsSource SPI

A pull-based interface called by the `ScalingScheduler` during each tick:

```java
public interface DemandMetricsSource {
    Map<String, Double> collect(String poolName);
}
```

Implementations are CDI beans discovered at startup. The scheduler iterates all instances and merges their results into a single map. Key collisions: last-writer-wins (non-deterministic but acceptable — sources should use namespaced keys like `"http.queue_depth"` or `"kafka.lag"`).

Error handling: if a source throws, log a warning and skip it. One failing source must not prevent other sources or the tick from completing.

### ScalingScheduler Changes

The `evaluatePool()` method gains external metrics collection:

```java
private void evaluatePool(String poolName) {
    var manager    = mgrRegistry.get(poolName).orElse(null);
    var definition = defRegistry.get(poolName).orElse(null);
    if (manager == null || definition == null) return;

    var scalingConfig = definition.pool().scaling();
    if (scalingConfig instanceof ScalingConfig.NoScalingConfig) return;

    // Collect external metrics (always, even during cooldown — same as demand metrics)
    var externalMetrics = collectExternalMetrics(poolName);

    var demand = manager.snapshotAndResetDemandMetrics(externalMetrics);
    // ... cooldown check, snapshot construction, policy evaluation (unchanged) ...
}

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

The `ScalingScheduler` constructor gains an injected `Instance<DemandMetricsSource>`:

```java
@Inject
public ScalingScheduler(AgentPoolDefinitionRegistry defRegistry,
                        AgentPoolManagerRegistry mgrRegistry,
                        jakarta.enterprise.inject.Instance<PoolEventEmitter> eventEmitterInstance,
                        jakarta.enterprise.inject.Instance<DemandMetricsSource> metricsSourcesInstance) {
    // ...
    this.metricsSources = metricsSourcesInstance.stream().toList();
}
```

### DemandPressurePolicy

A new built-in policy that reacts to exhaustion count and acquire latency:

```java
public class DemandPressurePolicy implements ScalingPolicy {

    private final int exhaustionThreshold;
    private final long latencyThresholdNanos;

    public DemandPressurePolicy(int exhaustionThreshold, long latencyThresholdMs) {
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

        // Scale out: either signal independently triggers
        if (exhaustionTriggered) {
            return ScalingDecision.scaleOut(1,
                "exhaustions %d > threshold %d"
                    .formatted(demand.exhaustions(), exhaustionThreshold));
        }
        if (latencyTriggered) {
            return ScalingDecision.scaleOut(1,
                "avgAcquireLatency %dms > threshold %dms"
                    .formatted(demand.averageAcquireMs(),
                               latencyThresholdNanos / 1_000_000));
        }

        // Scale in: both signals below hysteresis band (50% of threshold)
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

Scale-out adjustment is always 1 — conservative, incremental. Cooldown prevents rapid successive adjustments. Scale-in requires *both* signals to be quiet (below 50% of their thresholds) to avoid premature scale-in when one signal is still elevated.

### DemandPressureConfig

New sealed variant of `ScalingConfig`:

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

The `ScalingConfig` sealed interface gains this as a new permit. `type()` returns `"demand-pressure"`. `createPolicy()` in `ScalingScheduler` constructs `new DemandPressurePolicy(t.exhaustionThreshold(), t.latencyThresholdMs())`.

### YAML Configuration

```yaml
agent-pools:
  code-reviewer:
    command: "claude --model opus"
    working-dir: /workspace/reviews
    pool:
      min-active: 2
      max-active: 10
      scaling:
        type: demand-pressure
        exhaustion-threshold: 2
        latency-threshold-ms: 500
        cooldown: 60s
        scale-in-cooldown: 300s
```

`AgentPoolYamlParser` gains parsing for the `demand-pressure` type: reads `exhaustion-threshold` (int, required) and `latency-threshold-ms` (long, required) from the scaling section.

`AgentPoolSchema` gains parameter definitions for the new fields.

### Metrics Export

**PoolMetricsRegistrar** additions:

```java
registry.gauge("claudony.pool.acquire_latency_avg_ms", tags, mgr,
    m -> m.lastDemandSnapshot().averageAcquireMs());
registry.gauge("claudony.pool.acquire_latency_max_ms", tags, mgr,
    m -> m.lastDemandSnapshot().maxAcquireMs());
```

This requires `AgentSessionManager` to retain the last demand snapshot (a simple `volatile` field set during `snapshotAndResetDemandMetrics()`). The snapshot is already immutable so no thread-safety concern.

**IoTDBFlatLabelAdapter** gains the new gauge names in its label map.

**PoolEventEmitter** — no changes needed. Scaling decisions already carry the `reason` string which will naturally include the demand-pressure justification.

### Changes to Existing Code

| File | Change |
|------|--------|
| `PoolSnapshot.java` | Add latency fields and externalMetrics map to DemandMetrics |
| `AgentSessionManager.java` | Add totalAcquireNanos/maxAcquireNanos counters, timing in acquireSession(), update snapshotAndResetDemandMetrics() signature, add lastDemandSnapshot field |
| `ScalingConfig.java` | Add `DemandPressureConfig` permit |
| `ScalingScheduler.java` | Inject `Instance<DemandMetricsSource>`, collect external metrics, pass to snapshotAndResetDemandMetrics() |
| `AgentPoolYamlParser.java` | Parse `demand-pressure` type with exhaustion-threshold and latency-threshold-ms |
| `AgentPoolSchema.java` | Add demand-pressure parameters |
| `PoolMetricsRegistrar.java` | Add latency gauges |
| `IoTDBFlatLabelAdapter.java` | Add new gauge labels |

New files:

| File | Purpose |
|------|---------|
| `DemandMetricsSource.java` | SPI interface |
| `DemandPressurePolicy.java` | Built-in policy implementation |

### Data Flow

```
acquireSession() call
  → System.nanoTime() before lock
  → acquire/evict/create under lock
  → elapsed = nanoTime() - start
  → totalAcquireNanos += elapsed, maxAcquireNanos = max(...)
  
ScalingScheduler.tick() [every 15s]
  → collectExternalMetrics(poolName) → merged Map<String, Double>
  → manager.snapshotAndResetDemandMetrics(externalMetrics)
      → returns DemandMetrics(evictions, exhaustions, acquires,
                              avgNanos, maxNanos, externalMetrics)
      → resets all counters
  → PoolSnapshot(status + demand)
  → DemandPressurePolicy.evaluate(snapshot)
      → exhaustions > threshold OR avgLatency > threshold → scaleOut(1)
      → both below 50% of threshold → scaleIn(1)
  → manager.adjustMaxActive(newMax)
```

### Testing Strategy

**Unit tests:**

- `DemandPressurePolicyTest` — exhaustion-only trigger, latency-only trigger, both trigger (verify single scaleOut), neither trigger (none), hysteresis band (above 50% stays none, below 50% both → scaleIn), boundary values (exactly at threshold), minActive floor prevents scale-in
- `DemandMetricsSourceTest` — verify the SPI contract with a test implementation
- `PoolSnapshotTest` — add tests for new DemandMetrics fields, averageAcquireMs/maxAcquireMs convenience methods, ZERO constant includes new fields, externalMetrics immutability
- `ScalingConfigTest` — add DemandPressureConfig construction, validation (negative threshold, zero latency), type() returns "demand-pressure"
- `AgentSessionManagerTest` — latency tracking: verify snapshotAndResetDemandMetrics includes non-zero latency after acquires, verify reset clears latency counters, verify average is computed correctly across multiple acquires, verify max tracks the worst case
- `StepScalingPolicyTest` / `TargetTrackingPolicyTest` — regression: verify they still work with enriched DemandMetrics (they only read fillRatio, so this is a compatibility check)

**Integration tests:**

- `ScalingSchedulerTest` — add test with DemandPressureConfig, verify external metrics collection with mock DemandMetricsSource, verify failing source is logged and skipped
- `AgentPoolYamlParserTest` — parse demand-pressure YAML variant, round-trip validation
- `FleetPoolIntegrationTest` — end-to-end: configure demand-pressure pool, exhaust it, verify scaleOut fires

## Scope

**In scope:**
- Enriched `DemandMetrics` with latency fields and external metrics map
- Latency tracking in `AgentSessionManager`
- `DemandMetricsSource` SPI
- `DemandPressurePolicy` built-in implementation
- `DemandPressureConfig` sealed variant
- YAML parsing and schema for `demand-pressure` type
- Micrometer gauge export for latency
- IoTDB label map update
- Unit and integration tests for all new types

**Out of scope:**
- Concrete `DemandMetricsSource` implementations (HTTP polling, Prometheus scraping, etc.)
- Dashboard display of demand-pressure metrics (#242)
- REST API for runtime scaling config changes (#241)
- Percentile latency tracking (p99) — insufficient sample size at 15s tick interval

## References

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSnapshot.java` — current DemandMetrics record
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java:137-148` — snapshotAndResetDemandMetrics pattern
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java` — sealed hierarchy
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java` — tick loop and policy dispatch
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/TargetTrackingPolicy.java` — hysteresis band pattern (70% for scale-in)
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolMetricsRegistrar.java` — Micrometer gauge registration
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/IoTDBFlatLabelAdapter.java` — IoTDB export
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EvictionPolicy.java` — SPI pattern reference
- `specs/feat/206-auto-scaling-policies/2026-09-29-auto-scaling-policies-design.md` — parent spec, DemandMetrics origin
- Issue #206 — auto-scaling policies (deferred #240)
- Issue #240 — this issue
