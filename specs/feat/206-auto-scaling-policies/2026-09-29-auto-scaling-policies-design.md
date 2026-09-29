# Auto-Scaling Policies for Agent Pools

**Issue:** #206
**Deferred from:** #205 (agent pool management spec)
**Date:** 2026-09-29

## Summary

Add target-tracking and step scaling policies for agent pools. Scaling policies observe pool utilization and produce scaling decisions that a scheduler applies to the pool. Configured in YAML alongside existing pool config. The `ScalingPolicy` interface is also an SPI for custom implementations.

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
    double utilization    // activeCount / (double) maxActive
) {}
```

`ScalingDecision` is the output:

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

### Built-in Implementations

**TargetTrackingPolicy:** Maintains a target utilization. If utilization exceeds the target, scales out. If utilization is significantly below the target, scales in.

```java
public class TargetTrackingPolicy implements ScalingPolicy {

    private final double target;

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        if (snapshot.utilization() > target) {
            int desired = (int) Math.ceil(snapshot.activeCount() / target);
            int add = Math.min(desired - snapshot.activeCount(),
                               snapshot.maxActive() - snapshot.activeCount());
            if (add > 0) return ScalingDecision.scaleOut(add,
                "utilization %.0f%% exceeds target %.0f%%"
                    .formatted(snapshot.utilization() * 100, target * 100));
        }
        if (snapshot.utilization() < target * 0.7 && snapshot.activeCount() > snapshot.minActive()) {
            int desired = Math.max((int) Math.ceil(snapshot.activeCount() * snapshot.utilization() / target),
                                   snapshot.minActive());
            int remove = snapshot.activeCount() - desired;
            if (remove > 0) return ScalingDecision.scaleIn(remove,
                "utilization %.0f%% below scale-in threshold %.0f%%"
                    .formatted(snapshot.utilization() * 100, target * 70));
        }
        return ScalingDecision.none();
    }
}
```

Scale-in triggers at 70% of target (not at the target itself) to create a hysteresis band that prevents oscillation. For example, with `target: 0.7`, scale-out triggers above 70% utilization and scale-in triggers below 49%.

**StepScalingPolicy:** Discrete threshold-based steps. Each step defines a utilization threshold and an adjustment count (positive = scale out, negative = scale in):

```java
public record ScalingStep(double threshold, int adjustment) {}

public class StepScalingPolicy implements ScalingPolicy {

    private final List<ScalingStep> steps;  // sorted by threshold descending

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        for (var step : steps) {
            if (step.adjustment() > 0 && snapshot.utilization() >= step.threshold()) {
                int add = Math.min(step.adjustment(),
                                   snapshot.maxActive() - snapshot.activeCount());
                if (add > 0) return ScalingDecision.scaleOut(add, ...);
            }
            if (step.adjustment() < 0 && snapshot.utilization() <= step.threshold()) {
                int remove = Math.min(-step.adjustment(),
                                      snapshot.activeCount() - snapshot.minActive());
                if (remove > 0) return ScalingDecision.scaleIn(remove, ...);
            }
        }
        return ScalingDecision.none();
    }
}
```

**NoOpScalingPolicy:** The default — returns `ScalingDecision.none()` unconditionally. Applied when no `scaling:` section is present in YAML.

### Scaling Mode

The `mode` property controls how decisions are applied:

- **`proactive`** (default): The scheduler creates or suspends real sessions to match the desired count. Uses `AgentPoolDefinition.AgentConfig` (identity, workingDir, command) to create pre-warmed sessions.
- **`reactive`**: The scheduler adjusts `maxActive` dynamically. Sessions are still created on-demand via `acquireSession()`.

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

    @Scheduled(every = "{claudony.scaling.interval:15s}")
    void tick() {
        for each registered pool:
            1. Build PoolSnapshot from AgentSessionManager.status()
            2. Check cooldown — skip if in cooldown
            3. Call policy.evaluate(snapshot)
            4. If decision != NONE:
                a. Apply decision (proactive: create/suspend sessions; reactive: adjust maxActive)
                b. Record cooldown timestamp
    }
}
```

The tick interval is configurable (`claudony.scaling.interval`, default 15s). This is the global evaluation frequency, separate from per-pool cooldowns.

### ScalingConfig

A new record that lives inside `AgentPoolDefinition.PoolConfig`:

```java
public record ScalingConfig(
    ScalingType type,       // TARGET_TRACKING, STEP, NONE
    ScalingMode mode,       // PROACTIVE, REACTIVE
    double target,          // for target-tracking: target utilization (0.0-1.0)
    List<ScalingStep> steps, // for step: threshold/adjustment pairs
    Duration cooldown,
    Duration scaleInCooldown  // null = same as cooldown
) {}
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
        mode: proactive
        cooldown: 60s
        scale-in-cooldown: 300s
```

Step scaling variant:

```yaml
      scaling:
        type: step
        mode: proactive
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

### Changes to Existing Code

**`AgentPoolDefinition.PoolConfig`:** Add `ScalingConfig scaling` field (nullable, defaults to null = no scaling).

**`AgentPoolYamlParser`:** Parse the `scaling:` nested section. Validate scaling config (target in 0.0-1.0, cooldown > 0, steps non-empty for step type). Construct the appropriate `ScalingPolicy` implementation.

**`AgentPoolSchema`:** Add `pool.scaling.*` parameters to the schema definition.

**`AgentSessionManager`:** Add `adjustMaxActive(int newMax)` for reactive mode — replaces the `AgentSessionManagerConfig` record with a new instance (records stay immutable). Add a `scaleOut(int count)` method for proactive mode that creates sessions using the pool's `AgentConfig`. Add a `scaleIn(int count)` method that suspends the N sessions with the highest eviction scores. Both methods are lock-guarded and respect `minActive`/`maxActive` bounds.

**`AgentPoolDefinitionRegistry`:** Track the `ScalingPolicy` per registered pool so the scheduler can look them up.

### Testing Strategy

- **Unit tests for policies:** `TargetTrackingPolicyTest`, `StepScalingPolicyTest` — pure function tests with various `PoolSnapshot` inputs. Test hysteresis band, boundary conditions, min/max clamping.
- **Unit tests for ScalingConfig:** YAML parsing, validation, defaults.
- **Unit test for ScalingScheduler:** Cooldown enforcement, tick-driven evaluation, mode dispatch. Mock `AgentSessionManager`.
- **Integration test:** `FleetPoolIntegrationTest` addition — end-to-end: configure a pool with target-tracking, drive utilization above target, verify sessions are created.

### Data Flow

```
YAML config
  → AgentPoolYamlParser
  → AgentPoolDefinition(agent, pool(min, max, eviction, scaling))
  → AgentPoolDefinitionRegistry.register()
  → ScalingScheduler.tick() [every 15s]
      → PoolSnapshot from AgentSessionManager.status()
      → ScalingPolicy.evaluate(snapshot) → ScalingDecision
      → if not in cooldown:
          → proactive: manager.acquireSession() / manager.suspendSession()
          → reactive: manager.adjustMaxActive()
      → record cooldown
```

## Scope

**In scope:**
- `ScalingPolicy` SPI (interface + `NoOpScalingPolicy` default)
- `TargetTrackingPolicy` and `StepScalingPolicy` built-in implementations
- `PoolSnapshot`, `ScalingDecision`, `ScalingDirection` records
- `ScalingConfig` record and YAML parsing
- `ScalingScheduler` with per-pool cooldown tracking
- `ScalingMode` enum (`PROACTIVE`, `REACTIVE`)
- Schema updates for `pool.scaling.*`
- Unit tests for all new types
- Integration test extension

**Out of scope:**
- Queue-depth or external metrics (future — `PoolSnapshot` can grow)
- REST API for runtime scaling config changes (follow-on)
- Dashboard scaling status display (follow-on)

## References

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EvictionPolicy.java` — existing pure-function SPI pattern
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java` — pool orchestration
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java` — pool definition model
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java` — YAML config parsing
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java` — schema validation
- Issue #205 — original agent pool management spec (scaling deferred from here)
- Issue #229 — EvictionPolicy SPI (same pattern reused)
