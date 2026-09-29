# Auto-Scaling Policies for Agent Pools

**Issue:** #206
**Deferred from:** #205 (agent pool management spec, D5)
**Date:** 2026-09-29

## Summary

Add target-tracking and step scaling policies for agent pools. Scaling policies observe pool fill ratio and demand-pressure metrics, producing scaling decisions that adjust the pool's `maxActive` ceiling. Configured in YAML alongside existing pool config. The `ScalingPolicy` interface is also an SPI for custom implementations.

All scaling is reactive — policies adjust `maxActive` based on pool state; sessions are still created on-demand via `acquireSession()`. This aligns with the identity-correlated session model (D10 from #205): sessions are bound to specific identities and cannot be pre-created anonymously.

## Design

### ScalingPolicy SPI

A pure function that evaluates pool state and returns a scaling decision:

```java
public interface ScalingPolicy {
    ScalingDecision evaluate(PoolSnapshot snapshot);
}
```

`PoolSnapshot` is an immutable record of pool state at evaluation time:

```java
public record PoolSnapshot(
    int activeCount,
    int suspendedCount,
    int minActive,
    int maxActive,
    DemandMetrics demand
) {
    /** Fraction of capacity ceiling occupied by active sessions. */
    public double fillRatio() {
        return maxActive > 0 ? activeCount / (double) maxActive : 0.0;
    }

    /** Demand-pressure counters accumulated since the last evaluation tick. */
    public record DemandMetrics(
        int evictions,     // sessions evicted to make room for acquires
        int exhaustions,   // acquires rejected (pool exhausted, minActive reached)
        int acquires       // total acquireSession() calls
    ) {
        public static final DemandMetrics ZERO = new DemandMetrics(0, 0, 0);
    }
}
```

`fillRatio()` is a derived method — not a constructor parameter — so it cannot be inconsistent with `activeCount` and `maxActive`. `DemandMetrics` provides the actual demand signal: how many evictions and exhaustions occurred between ticks. Policies can use `fillRatio()` for ceiling-proximity tracking and `DemandMetrics` for demand-pressure response.

For reactive scaling (adjusting `maxActive`), `fillRatio` has the correct feedback direction: increasing `maxActive` decreases `fillRatio` (denominator grows, numerator unchanged), creating the negative feedback loop required for convergence.

`ScalingDecision` is the output — `count` represents the magnitude of `maxActive` adjustment:

```java
public record ScalingDecision(
    ScalingDirection direction,
    int count,
    String reason
) {
    public static ScalingDecision none() {
        return new ScalingDecision(ScalingDirection.NONE, 0, "no action needed");
    }

    public static ScalingDecision scaleOut(int count, String reason) {
        return new ScalingDecision(ScalingDirection.OUT, count, reason);
    }

    public static ScalingDecision scaleIn(int count, String reason) {
        return new ScalingDecision(ScalingDirection.IN, count, reason);
    }
}

public enum ScalingDirection { NONE, OUT, IN }
```

`scaleOut(n)` means increase `maxActive` by `n`. `scaleIn(n)` means decrease `maxActive` by `n`.

### Built-in Implementations

**TargetTrackingPolicy:** Maintains a target fill ratio by adjusting `maxActive`. When fill ratio exceeds the target, increases `maxActive` so `activeCount / maxActive` approaches the target. When fill ratio drops significantly below the target, decreases `maxActive` toward the target.

```java
public class TargetTrackingPolicy implements ScalingPolicy {

    private final double targetFillRatio;

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        double fillRatio = snapshot.fillRatio();

        if (fillRatio > targetFillRatio) {
            int desiredMax = (int) Math.ceil(snapshot.activeCount() / targetFillRatio);
            int increase = desiredMax - snapshot.maxActive();
            if (increase > 0) return ScalingDecision.scaleOut(increase,
                "fillRatio %.0f%% exceeds target %.0f%%"
                    .formatted(fillRatio * 100, targetFillRatio * 100));
        }

        double scaleInThreshold = targetFillRatio * 0.7;
        if (fillRatio < scaleInThreshold && snapshot.maxActive() > snapshot.minActive()) {
            int desiredMax = Math.max(
                (int) Math.ceil(snapshot.activeCount() / targetFillRatio),
                snapshot.minActive());
            int decrease = snapshot.maxActive() - desiredMax;
            if (decrease > 0) return ScalingDecision.scaleIn(decrease,
                "fillRatio %.0f%% below threshold %.0f%%"
                    .formatted(fillRatio * 100, scaleInThreshold * 100));
        }

        return ScalingDecision.none();
    }
}
```

Scale-in triggers at 70% of the target fill ratio (not at the target itself) to create a hysteresis band that prevents oscillation. For example, with `target: 0.7`, scale-out triggers above 70% fill ratio and scale-in triggers below 49%.

The key difference from the session-count model: decisions adjust `maxActive`, not session count. Increasing `maxActive` **decreases** fill ratio because the denominator grows while `activeCount` stays constant. No positive feedback loop.

**Traced example (scale-out):** target=0.7, activeCount=8, maxActive=10, fillRatio=0.8
- `desiredMax = ceil(8 / 0.7) = 12`, increase = 12 - 10 = 2
- After: maxActive=12, activeCount=8 (unchanged), fillRatio = 8/12 = 0.67 — below target, stable ✓

**Traced example (scale-in):** target=0.7, activeCount=3, maxActive=10, fillRatio=0.3
- scaleInThreshold = 0.49, fillRatio 0.3 < 0.49
- `desiredMax = max(ceil(3 / 0.7), minActive) = max(5, 2) = 5`, decrease = 10 - 5 = 5
- After: maxActive=5, fillRatio = 3/5 = 0.6 — above threshold, stable ✓

**Traced example (empty pool):** target=0.7, activeCount=0, maxActive=10, fillRatio=0.0
- `desiredMax = max(ceil(0 / 0.7), 2) = max(0, 2) = 2`, decrease = 10 - 2 = 8
- After: maxActive=2 (= minActive), fillRatio = 0.0 — still below threshold but at minActive floor ✓

**StepScalingPolicy:** Discrete threshold-based steps. Each step defines a fill ratio threshold and a `maxActive` adjustment (positive = increase, negative = decrease):

```java
public record ScalingStep(double threshold, int adjustment) {}

public class StepScalingPolicy implements ScalingPolicy {

    private final List<ScalingStep> steps;  // sorted by threshold descending

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        double fillRatio = snapshot.fillRatio();
        for (var step : steps) {
            if (step.adjustment() > 0 && fillRatio >= step.threshold()) {
                return ScalingDecision.scaleOut(step.adjustment(),
                    "fillRatio %.0f%% >= step threshold %.0f%%"
                        .formatted(fillRatio * 100, step.threshold() * 100));
            }
            if (step.adjustment() < 0 && fillRatio <= step.threshold()) {
                return ScalingDecision.scaleIn(-step.adjustment(),
                    "fillRatio %.0f%% <= step threshold %.0f%%"
                        .formatted(fillRatio * 100, step.threshold() * 100));
            }
        }
        return ScalingDecision.none();
    }
}
```

Step validation rules (enforced at parse time):
- All thresholds must be in (0.0, 1.0) exclusive
- No duplicate thresholds
- Scale-out steps (adjustment > 0) must have thresholds strictly above all scale-in thresholds (adjustment < 0) — prevents overlap zones where both actions apply
- No zero adjustments

**NoOpScalingPolicy:** The default — returns `ScalingDecision.none()` unconditionally. Applied when no `scaling:` section is present in YAML.

### Cooldown

Cooldown prevents oscillation by enforcing a minimum time between successive scaling actions:

- `cooldown` — applies to both scale-out and scale-in (default: 60s)
- `scale-in-cooldown` — optional override for scale-in only (default: same as `cooldown`)

Cooldown state is tracked in the `ScalingScheduler`, not in the policy. The policy is stateless.

### ScalingScheduler

A `@Scheduled` bean that periodically evaluates each pool's scaling policy:

```java
@ApplicationScoped
public class ScalingScheduler {

    @Inject AgentPoolDefinitionRegistry definitionRegistry;
    @Inject AgentPoolManagerRegistry managerRegistry;

    @Scheduled(every = "{claudony.scaling.interval:15s}")
    void tick() {
        for (String poolName : managerRegistry.poolNames()) {
            evaluatePool(poolName);
        }
    }

    private void evaluatePool(String poolName) {
        var manager = managerRegistry.get(poolName).orElse(null);
        var definition = definitionRegistry.get(poolName).orElse(null);
        if (manager == null || definition == null) return;

        var scalingConfig = definition.pool().scaling();
        if (scalingConfig instanceof NoScalingConfig) return;
        if (inCooldown(poolName, scalingConfig)) return;

        var demand = manager.snapshotAndResetDemandMetrics();
        var status = manager.status();
        var snapshot = new PoolSnapshot(
            status.active(), status.idle(), status.min(), status.max(), demand);

        var policy = policyFor(scalingConfig);
        var decision = policy.evaluate(snapshot);

        if (decision.direction() != ScalingDirection.NONE) {
            int currentMax = status.max();
            int newMax = switch (decision.direction()) {
                case OUT -> currentMax + decision.count();
                case IN  -> currentMax - decision.count();
                case NONE -> currentMax;
            };
            manager.adjustMaxActive(newMax);
            recordCooldown(poolName, decision.direction(), scalingConfig);
        }
    }
}
```

The tick interval is configurable (`claudony.scaling.interval`, default 15s). This is the global evaluation frequency, separate from per-pool cooldowns.

Pools are evaluated serially within a tick. Quarkus `@Scheduled` defaults to `BLOCKED` concurrency — a new tick cannot start while the previous one is still running. Since `adjustMaxActive()` is a lightweight in-memory operation (no tmux process creation), serial evaluation within a single tick is sufficient for the expected pool count (single-digit).

### ScalingConfig

A sealed interface hierarchy — each variant carries only its relevant fields:

```java
public sealed interface ScalingConfig
    permits TargetTrackingConfig, StepConfig, NoScalingConfig {
    Duration cooldown();
    Duration scaleInCooldown();
}

public record TargetTrackingConfig(
    double targetFillRatio,
    Duration cooldown,
    Duration scaleInCooldown
) implements ScalingConfig {
    public TargetTrackingConfig {
        if (targetFillRatio <= 0.0 || targetFillRatio > 1.0)
            throw new IllegalArgumentException("targetFillRatio must be in (0.0, 1.0]");
        if (cooldown == null) cooldown = Duration.ofSeconds(60);
        if (scaleInCooldown == null) scaleInCooldown = cooldown;
    }
}

public record StepConfig(
    List<ScalingStep> steps,
    Duration cooldown,
    Duration scaleInCooldown
) implements ScalingConfig {
    public StepConfig {
        Objects.requireNonNull(steps);
        if (steps.isEmpty()) throw new IllegalArgumentException("steps must not be empty");
        validateSteps(steps);
        if (cooldown == null) cooldown = Duration.ofSeconds(60);
        if (scaleInCooldown == null) scaleInCooldown = cooldown;
    }

    private static void validateSteps(List<ScalingStep> steps) {
        double lowestScaleOut = steps.stream()
            .filter(s -> s.adjustment() > 0).mapToDouble(ScalingStep::threshold)
            .min().orElse(Double.MAX_VALUE);
        double highestScaleIn = steps.stream()
            .filter(s -> s.adjustment() < 0).mapToDouble(ScalingStep::threshold)
            .max().orElse(-1.0);
        if (highestScaleIn >= lowestScaleOut)
            throw new IllegalArgumentException(
                "scale-in thresholds must be strictly below all scale-out thresholds");
        var dupes = steps.stream().map(ScalingStep::threshold)
            .collect(java.util.stream.Collectors.groupingBy(t -> t, java.util.stream.Collectors.counting()))
            .entrySet().stream().filter(e -> e.getValue() > 1).toList();
        if (!dupes.isEmpty())
            throw new IllegalArgumentException("duplicate thresholds: " + dupes);
        for (var step : steps) {
            if (step.threshold() <= 0.0 || step.threshold() >= 1.0)
                throw new IllegalArgumentException("threshold must be in (0.0, 1.0): " + step.threshold());
            if (step.adjustment() == 0)
                throw new IllegalArgumentException("zero adjustment is meaningless");
        }
    }
}

public record NoScalingConfig() implements ScalingConfig {
    public static final NoScalingConfig INSTANCE = new NoScalingConfig();
    @Override public Duration cooldown() { return Duration.ZERO; }
    @Override public Duration scaleInCooldown() { return Duration.ZERO; }
}
```

### YAML Configuration

```yaml
agent-pools:
  code-reviewer:
    command: "claude --model opus"
    working-dir: /workspace/reviews
    pool:
      min-active: 2
      max-active: 10
      eviction: memory-weighted
      scaling:
        type: target-tracking
        target: 0.7
        cooldown: 60s
        scale-in-cooldown: 300s
```

Step scaling variant:

```yaml
      scaling:
        type: step
        cooldown: 60s
        steps:
          - threshold: 0.8
            adjustment: 2
          - threshold: 0.9
            adjustment: 4
          - threshold: 0.3
            adjustment: -1
```

No `scaling:` section = `NoOpScalingPolicy` (current behaviour, fully backward compatible).

Custom policy via CDI bean:

```yaml
      scaling:
        type: demand-pressure
        cooldown: 60s
```

Built-in type names (`target-tracking`, `step`, `none`) are resolved directly. Unknown type names are looked up as CDI `@Named` beans implementing `ScalingPolicy`. If no matching bean is found, the parser throws `IllegalArgumentException`.

### Changes to Existing Code

**`AgentPoolDefinition.PoolConfig`:** Add `ScalingConfig scaling` field (defaults to `NoScalingConfig.INSTANCE` if null).

**`AgentPoolYamlParser`:** Parse the `scaling:` nested section. Construct the appropriate `ScalingConfig` sealed subtype based on the `type:` field. For unrecognised type names, record the name for CDI bean lookup at runtime.

**`AgentPoolSchema`:** Add `pool.scaling.*` parameters to the schema definition.

**`AgentSessionManager`:**
- Change `config` field from `private final` to `private volatile` — enables safe reads from `status()` without acquiring the lock, while allowing atomic replacement by `adjustMaxActive()`. The config record is immutable; only the reference is reassigned. Read sites (`status()`, `activeCount()` boundary checks) see a consistent snapshot via volatile semantics. Write sites (`adjustMaxActive()`, `acquireSession()` eviction path) acquire the lock.
- Add demand-pressure counters: `evictionCount`, `exhaustionCount`, `acquireCount` (plain `int`, accessed only under the lock).
- Increment `acquireCount` at entry to `acquireSession()` (inside the lock).
- Increment `evictionCount` in `evictOne()` on successful eviction.
- Increment `exhaustionCount` in `evictOne()` when throwing `AgentPoolExhaustedException`.
- Add `adjustMaxActive(int newMax)`: acquires the lock, clamps `newMax` to `max(newMax, activeCount(), minActive())` — never goes below current occupancy or configured floor. Replaces `this.config` with a new `AgentSessionManagerConfig` instance. No sessions are suspended — the new ceiling only affects future `acquireSession()` capacity decisions.
- Add `snapshotAndResetDemandMetrics()`: acquires the lock, returns a `PoolSnapshot.DemandMetrics` snapshot of current counters, resets all counters to zero.

**`AgentPoolManagerRegistry` (new):** A `@ApplicationScoped` CDI bean mapping pool names to `AgentSessionManager` instances. `ClaudonyAgentBackend` registers its manager during startup via `managerRegistry.register(poolName, sessionManager)`. The `ScalingScheduler` injects this registry to reach each pool's manager.

### Testing Strategy

- **Unit tests for policies:** `TargetTrackingPolicyTest`, `StepScalingPolicyTest` — pure function tests with various `PoolSnapshot` inputs. Test hysteresis band, boundary conditions, minActive clamping. Verify no positive feedback loops by tracing multi-tick sequences.
- **Unit tests for ScalingConfig:** Sealed type construction, YAML parsing round-trip, validation rules (step overlap, threshold ranges, fill ratio bounds). Verify `NoScalingConfig.INSTANCE` identity.
- **Unit tests for demand metrics:** Verify counter increment/reset lifecycle across multiple acquires and evictions. Verify `snapshotAndResetDemandMetrics()` atomicity.
- **Unit test for ScalingScheduler:** Cooldown enforcement, tick-driven evaluation, `adjustMaxActive` dispatch. Mock `AgentSessionManager`.
- **Integration test:** `FleetPoolIntegrationTest` addition — end-to-end: configure a pool with target-tracking, drive fill ratio above target via concurrent acquires, verify `maxActive` increases.

### Data Flow

```
YAML config
  → AgentPoolYamlParser
  → AgentPoolDefinition(agent, pool(min, max, eviction, scaling))
  → AgentPoolDefinitionRegistry.register()
  → ClaudonyAgentBackend startup
      → AgentPoolManagerRegistry.register(poolName, sessionManager)
  → ScalingScheduler.tick() [every 15s]
      → manager.snapshotAndResetDemandMetrics() → DemandMetrics
      → PoolSnapshot from AgentSessionManager.status() + DemandMetrics
      → ScalingPolicy.evaluate(snapshot) → ScalingDecision
      → if not in cooldown and decision != NONE:
          → manager.adjustMaxActive(currentMax ± decision.count())
      → record cooldown
```

## Scope

**In scope:**
- `ScalingPolicy` SPI (interface + `NoOpScalingPolicy` default)
- `TargetTrackingPolicy` and `StepScalingPolicy` built-in implementations
- `PoolSnapshot` (with `fillRatio()` derived method and `DemandMetrics`), `ScalingDecision`, `ScalingDirection` records
- `ScalingConfig` sealed hierarchy (`TargetTrackingConfig`, `StepConfig`, `NoScalingConfig`)
- `AgentPoolManagerRegistry` for scheduler-to-manager wiring
- `ScalingScheduler` with per-pool cooldown tracking
- YAML parsing and schema updates for `pool.scaling.*`
- Demand-pressure counters in `AgentSessionManager`
- CDI `@Named` bean lookup for custom `ScalingPolicy` types
- Unit tests for all new types
- Integration test extension

**Out of scope (GitHub issues filed):**
- Queue-depth or external metrics enrichment for `PoolSnapshot.DemandMetrics` (#240)
- REST API for runtime scaling config changes (#241)
- Dashboard scaling status display (#242)
- Proactive scaling mode — identity-aware pre-warming of suspended sessions (#243)

## References

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EvictionPolicy.java` — existing pure-function SPI pattern
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java` — pool orchestration
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java` — pool definition model
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java` — YAML config parsing
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java` — schema validation
- Issue #205 — original agent pool management spec (D5: scaling deferred, D10: identity-correlated sessions)
- Issue #229 — EvictionPolicy SPI (same pattern reused)
- `docs/specs/issue-205-llm-fleet-manager/decisions.md` — D5, D8, D9, D10 design decisions
