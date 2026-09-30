# Auto-Scaling Policies Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #206 — feat: auto-scaling policies for agent pools
**Issue group:** #206

**Goal:** Add target-tracking and step scaling policies that reactively adjust pool `maxActive` based on utilization, configured via YAML.

**Architecture:** `ScalingPolicy` is a pure-function SPI (`evaluate(PoolSnapshot) → ScalingDecision`). Built-in implementations (`TargetTrackingPolicy`, `StepScalingPolicy`) adjust `effectiveMaxActive` on the `AgentSessionManager`. A `ScalingScheduler` ticks policies periodically with per-pool cooldown tracking. Configuration is YAML-driven via a sealed `ScalingConfig` hierarchy. Custom policies are resolved via CDI `@Named` beans.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ

## Global Constraints

- All new types go in `casehub/src/main/java/io/casehub/claudony/casehub/fleet/`
- All new tests go in `casehub/src/test/java/io/casehub/claudony/casehub/fleet/`
- Test pattern: plain JUnit 5 + AssertJ (no Quarkus context for unit tests)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
- `ScalingPolicy` follows the same pure-function SPI pattern as `EvictionPolicy`
- `effectiveMaxActive` can never exceed `config.maxActive()` (hard ceiling) or drop below `config.minActive()` (floor)

---

## Batch 1: Core SPI Types

### Task 1: ScalingDecision, ScalingDirection, PoolSnapshot, DemandMetrics

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingDirection.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingDecision.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSnapshot.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolSnapshotTest.java`

**Interfaces:**
- Produces: `ScalingDirection.NONE`, `ScalingDirection.OUT`, `ScalingDirection.IN`
- Produces: `ScalingDecision(ScalingDirection, int, String)`, `ScalingDecision.none()`, `ScalingDecision.scaleOut(int, String)`, `ScalingDecision.scaleIn(int, String)`
- Produces: `PoolSnapshot(int activeCount, int suspendedCount, int minActive, int maxActive, PoolSnapshot.DemandMetrics demand)`, `PoolSnapshot.fillRatio()` → `double`
- Produces: `PoolSnapshot.DemandMetrics(int evictions, int exhaustions, int acquires)`, `DemandMetrics.ZERO`

- [ ] **Step 1: Write PoolSnapshot tests**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class PoolSnapshotTest {

    @Test
    void fillRatioCalculatedFromActiveAndMax() {
        var snapshot = new PoolSnapshot(7, 2, 0, 10, PoolSnapshot.DemandMetrics.ZERO);
        assertThat(snapshot.fillRatio()).isCloseTo(0.7, within(0.001));
    }

    @Test
    void fillRatioZeroWhenMaxIsZero() {
        var snapshot = new PoolSnapshot(0, 0, 0, 0, PoolSnapshot.DemandMetrics.ZERO);
        assertThat(snapshot.fillRatio()).isCloseTo(0.0, within(0.001));
    }

    @Test
    void fillRatioOneWhenFull() {
        var snapshot = new PoolSnapshot(5, 0, 0, 5, PoolSnapshot.DemandMetrics.ZERO);
        assertThat(snapshot.fillRatio()).isCloseTo(1.0, within(0.001));
    }

    @Test
    void fillRatioZeroWhenEmpty() {
        var snapshot = new PoolSnapshot(0, 3, 0, 10, PoolSnapshot.DemandMetrics.ZERO);
        assertThat(snapshot.fillRatio()).isCloseTo(0.0, within(0.001));
    }

    @Test
    void demandMetricsZeroConstant() {
        var zero = PoolSnapshot.DemandMetrics.ZERO;
        assertThat(zero.evictions()).isZero();
        assertThat(zero.exhaustions()).isZero();
        assertThat(zero.acquires()).isZero();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=PoolSnapshotTest`
Expected: FAIL — classes not found

- [ ] **Step 3: Create ScalingDirection enum**

```java
package io.casehub.claudony.casehub.fleet;

public enum ScalingDirection { NONE, OUT, IN }
```

- [ ] **Step 4: Create ScalingDecision record**

```java
package io.casehub.claudony.casehub.fleet;

public record ScalingDecision(ScalingDirection direction, int count, String reason) {

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
```

- [ ] **Step 5: Create PoolSnapshot record**

```java
package io.casehub.claudony.casehub.fleet;

public record PoolSnapshot(
    int activeCount,
    int suspendedCount,
    int minActive,
    int maxActive,
    DemandMetrics demand
) {
    public double fillRatio() {
        return maxActive > 0 ? activeCount / (double) maxActive : 0.0;
    }

    public record DemandMetrics(int evictions, int exhaustions, int acquires) {
        public static final DemandMetrics ZERO = new DemandMetrics(0, 0, 0);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=PoolSnapshotTest`
Expected: PASS (5 tests)

- [ ] **Step 7: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingDirection.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingDecision.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSnapshot.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolSnapshotTest.java
git commit -m "feat(#206): add ScalingDirection, ScalingDecision, PoolSnapshot records"
```

### Task 2: ScalingPolicy SPI and NoOpScalingPolicy

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingPolicy.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/NoOpScalingPolicy.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/NoOpScalingPolicyTest.java`

**Interfaces:**
- Consumes: `PoolSnapshot`, `ScalingDecision`
- Produces: `ScalingPolicy.evaluate(PoolSnapshot) → ScalingDecision`
- Produces: `NoOpScalingPolicy` (plain class, constructed directly — not CDI-managed)

- [ ] **Step 1: Write NoOpScalingPolicy test**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class NoOpScalingPolicyTest {

    private final ScalingPolicy policy = new NoOpScalingPolicy();

    @Test
    void alwaysReturnsNone() {
        var snapshot = new PoolSnapshot(5, 2, 0, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
        assertThat(decision.count()).isZero();
    }

    @Test
    void returnsNoneEvenWhenFull() {
        var snapshot = new PoolSnapshot(10, 0, 0, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=NoOpScalingPolicyTest`
Expected: FAIL — ScalingPolicy not found

- [ ] **Step 3: Create ScalingPolicy interface**

```java
package io.casehub.claudony.casehub.fleet;

public interface ScalingPolicy {
    ScalingDecision evaluate(PoolSnapshot snapshot);
}
```

- [ ] **Step 4: Create NoOpScalingPolicy**

```java
package io.casehub.claudony.casehub.fleet;

/** No-op scaling — always returns NONE. Constructed directly, not CDI-managed. */
public class NoOpScalingPolicy implements ScalingPolicy {

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        return ScalingDecision.none();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=NoOpScalingPolicyTest`
Expected: PASS (2 tests)

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingPolicy.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/NoOpScalingPolicy.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/NoOpScalingPolicyTest.java
git commit -m "feat(#206): add ScalingPolicy SPI and NoOpScalingPolicy default"
```

---

## Batch 2: Built-in Policy Implementations

### Task 3: TargetTrackingPolicy

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/TargetTrackingPolicy.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/TargetTrackingPolicyTest.java`

**Interfaces:**
- Consumes: `ScalingPolicy`, `PoolSnapshot`, `ScalingDecision`, `ScalingDirection`
- Produces: `TargetTrackingPolicy(double targetFillRatio)` implementing `ScalingPolicy`

- [ ] **Step 1: Write TargetTrackingPolicy tests**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TargetTrackingPolicyTest {

    @Test
    void scalesOutWhenFillRatioExceedsTarget() {
        var policy = new TargetTrackingPolicy(0.7);
        // activeCount=8, maxActive=10 → fillRatio=0.8 > 0.7
        var snapshot = new PoolSnapshot(8, 0, 2, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.OUT);
        // desiredMax = ceil(8 / 0.7) = 12, increase = 12 - 10 = 2
        assertThat(decision.count()).isEqualTo(2);
    }

    @Test
    void scalesInWhenFillRatioBelowHysteresisBand() {
        var policy = new TargetTrackingPolicy(0.7);
        // activeCount=3, maxActive=10 → fillRatio=0.3, threshold=0.49
        var snapshot = new PoolSnapshot(3, 0, 2, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.IN);
        // desiredMax = max(ceil(3 / 0.7), 2) = max(5, 2) = 5, decrease = 10 - 5 = 5
        assertThat(decision.count()).isEqualTo(5);
    }

    @Test
    void noActionWhenInHysteresisBand() {
        var policy = new TargetTrackingPolicy(0.7);
        // activeCount=6, maxActive=10 → fillRatio=0.6, between 0.49 and 0.7
        var snapshot = new PoolSnapshot(6, 0, 2, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
    }

    @Test
    void scaleInRespectsMinActive() {
        var policy = new TargetTrackingPolicy(0.7);
        // activeCount=0, maxActive=10, minActive=5 → fillRatio=0.0
        // desiredMax = max(ceil(0/0.7), 5) = 5, decrease = 10 - 5 = 5
        var snapshot = new PoolSnapshot(0, 0, 5, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.IN);
        assertThat(decision.count()).isEqualTo(5);
    }

    @Test
    void noScaleInWhenAlreadyAtMinActive() {
        var policy = new TargetTrackingPolicy(0.7);
        // maxActive=5, minActive=5 → can't reduce
        var snapshot = new PoolSnapshot(0, 0, 5, 5, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
    }

    @Test
    void noScaleOutWhenExactlyAtTarget() {
        var policy = new TargetTrackingPolicy(0.7);
        // activeCount=7, maxActive=10 → fillRatio=0.7, exactly at target
        var snapshot = new PoolSnapshot(7, 0, 0, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
    }

    @Test
    void emptyPoolScalesInToMinActive() {
        var policy = new TargetTrackingPolicy(0.7);
        // activeCount=0, maxActive=10, minActive=2
        var snapshot = new PoolSnapshot(0, 0, 2, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.IN);
        // desiredMax = max(ceil(0/0.7), 2) = 2, decrease = 10 - 2 = 8
        assertThat(decision.count()).isEqualTo(8);
    }

    @Test
    void scaleOutStabilizesAfterOneStep() {
        var policy = new TargetTrackingPolicy(0.7);
        // Before: active=8, max=10 → fill=0.8 → scaleOut(2)
        // After: active=8, max=12 → fill=0.67 → in hysteresis band (0.49 < 0.67 < 0.7)
        var after = new PoolSnapshot(8, 0, 2, 12, PoolSnapshot.DemandMetrics.ZERO);
        assertThat(policy.evaluate(after).direction()).isEqualTo(ScalingDirection.NONE);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=TargetTrackingPolicyTest`
Expected: FAIL — TargetTrackingPolicy not found

- [ ] **Step 3: Implement TargetTrackingPolicy**

```java
package io.casehub.claudony.casehub.fleet;

public class TargetTrackingPolicy implements ScalingPolicy {

    private final double targetFillRatio;

    public TargetTrackingPolicy(double targetFillRatio) {
        if (targetFillRatio <= 0.0 || targetFillRatio > 1.0) {
            throw new IllegalArgumentException("targetFillRatio must be in (0.0, 1.0]");
        }
        this.targetFillRatio = targetFillRatio;
    }

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        double fillRatio = snapshot.fillRatio();

        if (fillRatio > targetFillRatio) {
            int desiredMax = (int) Math.ceil(snapshot.activeCount() / targetFillRatio);
            int increase = desiredMax - snapshot.maxActive();
            if (increase > 0) {
                return ScalingDecision.scaleOut(increase,
                    "fillRatio %.0f%% exceeds target %.0f%%"
                        .formatted(fillRatio * 100, targetFillRatio * 100));
            }
        }

        double scaleInThreshold = targetFillRatio * 0.7;
        if (fillRatio < scaleInThreshold && snapshot.maxActive() > snapshot.minActive()) {
            int desiredMax = Math.max(
                (int) Math.ceil(snapshot.activeCount() / targetFillRatio),
                snapshot.minActive());
            int decrease = snapshot.maxActive() - desiredMax;
            if (decrease > 0) {
                return ScalingDecision.scaleIn(decrease,
                    "fillRatio %.0f%% below threshold %.0f%%"
                        .formatted(fillRatio * 100, scaleInThreshold * 100));
            }
        }

        return ScalingDecision.none();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=TargetTrackingPolicyTest`
Expected: PASS (8 tests)

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/TargetTrackingPolicy.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/TargetTrackingPolicyTest.java
git commit -m "feat(#206): add TargetTrackingPolicy with hysteresis band"
```

### Task 4: ScalingStep and StepScalingPolicy

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingStep.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/StepScalingPolicy.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/StepScalingPolicyTest.java`

**Interfaces:**
- Consumes: `ScalingPolicy`, `PoolSnapshot`, `ScalingDecision`, `ScalingDirection`
- Produces: `ScalingStep(double threshold, int adjustment)`
- Produces: `StepScalingPolicy(List<ScalingStep>)` implementing `ScalingPolicy`

- [ ] **Step 1: Write StepScalingPolicy tests**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class StepScalingPolicyTest {

    @Test
    void scalesOutAtHighestMatchingThreshold() {
        var policy = new StepScalingPolicy(List.of(
            new ScalingStep(0.9, 4),
            new ScalingStep(0.8, 2)
        ));
        // fillRatio = 0.95 → matches 0.9 step (sorted descending, checked first)
        var snapshot = new PoolSnapshot(19, 0, 0, 20, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.OUT);
        assertThat(decision.count()).isEqualTo(4);
    }

    @Test
    void scalesOutAtLowerThresholdWhenHigherNotMet() {
        var policy = new StepScalingPolicy(List.of(
            new ScalingStep(0.9, 4),
            new ScalingStep(0.8, 2)
        ));
        // fillRatio = 0.85 → matches 0.8 step only
        var snapshot = new PoolSnapshot(17, 0, 0, 20, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.OUT);
        assertThat(decision.count()).isEqualTo(2);
    }

    @Test
    void scalesInAtNegativeAdjustmentThreshold() {
        var policy = new StepScalingPolicy(List.of(
            new ScalingStep(0.8, 2),
            new ScalingStep(0.3, -1)
        ));
        // fillRatio = 0.2 → matches 0.3 scale-in step
        var snapshot = new PoolSnapshot(2, 0, 0, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.IN);
        assertThat(decision.count()).isEqualTo(1);
    }

    @Test
    void noActionWhenNoThresholdMatched() {
        var policy = new StepScalingPolicy(List.of(
            new ScalingStep(0.8, 2),
            new ScalingStep(0.3, -1)
        ));
        // fillRatio = 0.5 → between thresholds
        var snapshot = new PoolSnapshot(5, 0, 0, 10, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.NONE);
    }

    @Test
    void emptyStepsListThrows() {
        assertThatThrownBy(() -> new StepScalingPolicy(List.of()))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void overlappingThresholdsThrow() {
        // scale-in threshold 0.5 is not strictly below scale-out threshold 0.5
        assertThatThrownBy(() -> new StepScalingPolicy(List.of(
            new ScalingStep(0.5, 2),
            new ScalingStep(0.5, -1)
        ))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void scaleInThresholdAboveScaleOutThrows() {
        assertThatThrownBy(() -> new StepScalingPolicy(List.of(
            new ScalingStep(0.4, 2),
            new ScalingStep(0.6, -1)
        ))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void zeroAdjustmentThrows() {
        assertThatThrownBy(() -> new StepScalingPolicy(List.of(
            new ScalingStep(0.8, 0)
        ))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void thresholdOutOfRangeThrows() {
        assertThatThrownBy(() -> new StepScalingPolicy(List.of(
            new ScalingStep(0.0, 2)
        ))).isInstanceOf(IllegalArgumentException.class);

        assertThatThrownBy(() -> new StepScalingPolicy(List.of(
            new ScalingStep(1.0, 2)
        ))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void graduatedScaleInMatchesMostAggressiveStep() {
        var policy = new StepScalingPolicy(List.of(
            new ScalingStep(0.8, 2),
            new ScalingStep(0.3, -1),
            new ScalingStep(0.1, -5)
        ));
        // fillRatio = 0.05 → below both 0.3 and 0.1 → most aggressive (-5) should win
        var snapshot = new PoolSnapshot(1, 0, 0, 20, PoolSnapshot.DemandMetrics.ZERO);
        var decision = policy.evaluate(snapshot);
        assertThat(decision.direction()).isEqualTo(ScalingDirection.IN);
        assertThat(decision.count()).isEqualTo(5);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=StepScalingPolicyTest`
Expected: FAIL — classes not found

- [ ] **Step 3: Create ScalingStep record**

```java
package io.casehub.claudony.casehub.fleet;

public record ScalingStep(double threshold, int adjustment) {}
```

- [ ] **Step 4: Implement StepScalingPolicy**

```java
package io.casehub.claudony.casehub.fleet;

import java.util.Comparator;
import java.util.List;
import java.util.stream.Collectors;

public class StepScalingPolicy implements ScalingPolicy {

    private final List<ScalingStep> steps;

    public StepScalingPolicy(List<ScalingStep> steps) {
        if (steps == null || steps.isEmpty()) {
            throw new IllegalArgumentException("steps must not be empty");
        }
        validateSteps(steps);
        this.steps = steps.stream()
            .sorted(Comparator.comparingDouble(ScalingStep::threshold).reversed())
            .toList();
    }

    @Override
    public ScalingDecision evaluate(PoolSnapshot snapshot) {
        double fillRatio = snapshot.fillRatio();
        ScalingStep matchedScaleIn = null;
        for (var step : steps) {
            if (step.adjustment() > 0 && fillRatio >= step.threshold()) {
                return ScalingDecision.scaleOut(step.adjustment(),
                    "fillRatio %.0f%% >= step threshold %.0f%%"
                        .formatted(fillRatio * 100, step.threshold() * 100));
            }
            if (step.adjustment() < 0 && fillRatio <= step.threshold()) {
                matchedScaleIn = step;
            }
        }
        if (matchedScaleIn != null) {
            return ScalingDecision.scaleIn(-matchedScaleIn.adjustment(),
                "fillRatio %.0f%% <= step threshold %.0f%%"
                    .formatted(fillRatio * 100, matchedScaleIn.threshold() * 100));
        }
        return ScalingDecision.none();
    }

    private static void validateSteps(List<ScalingStep> steps) {
        for (var step : steps) {
            if (step.threshold() <= 0.0 || step.threshold() >= 1.0) {
                throw new IllegalArgumentException(
                    "threshold must be in (0.0, 1.0): " + step.threshold());
            }
            if (step.adjustment() == 0) {
                throw new IllegalArgumentException("zero adjustment is meaningless");
            }
        }
        var dupes = steps.stream().map(ScalingStep::threshold)
            .collect(Collectors.groupingBy(t -> t, Collectors.counting()))
            .entrySet().stream().filter(e -> e.getValue() > 1).toList();
        if (!dupes.isEmpty()) {
            throw new IllegalArgumentException("duplicate thresholds: " + dupes);
        }
        double lowestScaleOut = steps.stream()
            .filter(s -> s.adjustment() > 0).mapToDouble(ScalingStep::threshold)
            .min().orElse(Double.MAX_VALUE);
        double highestScaleIn = steps.stream()
            .filter(s -> s.adjustment() < 0).mapToDouble(ScalingStep::threshold)
            .max().orElse(-1.0);
        if (highestScaleIn >= lowestScaleOut) {
            throw new IllegalArgumentException(
                "scale-in thresholds must be strictly below all scale-out thresholds");
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=StepScalingPolicyTest`
Expected: PASS (10 tests)

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingStep.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/StepScalingPolicy.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/StepScalingPolicyTest.java
git commit -m "feat(#206): add StepScalingPolicy with validation rules"
```

---

## Batch 3: ScalingConfig Sealed Hierarchy and AgentSessionManager Changes

### Task 5: ScalingConfig sealed hierarchy

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingConfigTest.java`

**Interfaces:**
- Consumes: `ScalingStep`
- Produces: `ScalingConfig` sealed interface (`cooldown()`, `scaleInCooldown()`)
- Produces: `TargetTrackingConfig(double targetFillRatio, Duration cooldown, Duration scaleInCooldown)`
- Produces: `StepConfig(List<ScalingStep> steps, Duration cooldown, Duration scaleInCooldown)`
- Produces: `CustomScalingConfig(String beanName, Duration cooldown, Duration scaleInCooldown)`
- Produces: `NoScalingConfig.INSTANCE`

- [ ] **Step 1: Write ScalingConfig tests**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ScalingConfigTest {

    @Test
    void targetTrackingDefaults() {
        var config = new ScalingConfig.TargetTrackingConfig(0.7, null, null);
        assertThat(config.targetFillRatio()).isEqualTo(0.7);
        assertThat(config.cooldown()).isEqualTo(Duration.ofSeconds(60));
        assertThat(config.scaleInCooldown()).isEqualTo(Duration.ofSeconds(60));
    }

    @Test
    void targetTrackingCustomCooldowns() {
        var config = new ScalingConfig.TargetTrackingConfig(
            0.8, Duration.ofSeconds(30), Duration.ofSeconds(300));
        assertThat(config.cooldown()).isEqualTo(Duration.ofSeconds(30));
        assertThat(config.scaleInCooldown()).isEqualTo(Duration.ofSeconds(300));
    }

    @Test
    void targetTrackingInvalidFillRatio() {
        assertThatThrownBy(() -> new ScalingConfig.TargetTrackingConfig(0.0, null, null))
            .isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> new ScalingConfig.TargetTrackingConfig(1.1, null, null))
            .isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> new ScalingConfig.TargetTrackingConfig(-0.1, null, null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void stepConfigDefaults() {
        var config = new ScalingConfig.StepConfig(
            List.of(new ScalingStep(0.8, 2)), null, null);
        assertThat(config.cooldown()).isEqualTo(Duration.ofSeconds(60));
        assertThat(config.scaleInCooldown()).isEqualTo(Duration.ofSeconds(60));
    }

    @Test
    void stepConfigEmptyStepsThrows() {
        assertThatThrownBy(() -> new ScalingConfig.StepConfig(List.of(), null, null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void stepConfigValidatesStepsAtConstructionTime() {
        assertThatThrownBy(() -> new ScalingConfig.StepConfig(
            List.of(new ScalingStep(0.8, 0)), null, null))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("zero adjustment");
    }

    @Test
    void customScalingConfigBlankBeanNameThrows() {
        assertThatThrownBy(() -> new ScalingConfig.CustomScalingConfig("", null, null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void noScalingConfigSingleton() {
        assertThat(ScalingConfig.NoScalingConfig.INSTANCE.cooldown()).isEqualTo(Duration.ZERO);
        assertThat(ScalingConfig.NoScalingConfig.INSTANCE.scaleInCooldown()).isEqualTo(Duration.ZERO);
    }

    @Test
    void sealedHierarchyPatternMatching() {
        ScalingConfig config = new ScalingConfig.TargetTrackingConfig(0.7, null, null);
        String type = switch (config) {
            case ScalingConfig.TargetTrackingConfig t -> "target";
            case ScalingConfig.StepConfig s -> "step";
            case ScalingConfig.CustomScalingConfig c -> "custom";
            case ScalingConfig.NoScalingConfig n -> "none";
        };
        assertThat(type).isEqualTo("target");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ScalingConfigTest`
Expected: FAIL — ScalingConfig not found

- [ ] **Step 3: Create ScalingConfig sealed hierarchy**

```java
package io.casehub.claudony.casehub.fleet;

import java.time.Duration;
import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

public sealed interface ScalingConfig
    permits ScalingConfig.TargetTrackingConfig,
            ScalingConfig.StepConfig,
            ScalingConfig.CustomScalingConfig,
            ScalingConfig.NoScalingConfig {

    Duration cooldown();
    Duration scaleInCooldown();

    record TargetTrackingConfig(
        double targetFillRatio,
        Duration cooldown,
        Duration scaleInCooldown
    ) implements ScalingConfig {
        public TargetTrackingConfig {
            if (targetFillRatio <= 0.0 || targetFillRatio > 1.0)
                throw new IllegalArgumentException(
                    "targetFillRatio must be in (0.0, 1.0]");
            if (cooldown == null) cooldown = Duration.ofSeconds(60);
            if (scaleInCooldown == null) scaleInCooldown = cooldown;
        }
    }

    record StepConfig(
        List<ScalingStep> steps,
        Duration cooldown,
        Duration scaleInCooldown
    ) implements ScalingConfig {
        public StepConfig {
            Objects.requireNonNull(steps);
            if (steps.isEmpty())
                throw new IllegalArgumentException("steps must not be empty");
            validateSteps(steps);
            if (cooldown == null) cooldown = Duration.ofSeconds(60);
            if (scaleInCooldown == null) scaleInCooldown = cooldown;
        }

        private static void validateSteps(List<ScalingStep> steps) {
            for (var step : steps) {
                if (step.threshold() <= 0.0 || step.threshold() >= 1.0)
                    throw new IllegalArgumentException(
                        "threshold must be in (0.0, 1.0): " + step.threshold());
                if (step.adjustment() == 0)
                    throw new IllegalArgumentException("zero adjustment is meaningless");
            }
            var dupes = steps.stream().map(ScalingStep::threshold)
                .collect(Collectors.groupingBy(t -> t, Collectors.counting()))
                .entrySet().stream().filter(e -> e.getValue() > 1).toList();
            if (!dupes.isEmpty())
                throw new IllegalArgumentException("duplicate thresholds: " + dupes);
            double lowestScaleOut = steps.stream()
                .filter(s -> s.adjustment() > 0).mapToDouble(ScalingStep::threshold)
                .min().orElse(Double.MAX_VALUE);
            double highestScaleIn = steps.stream()
                .filter(s -> s.adjustment() < 0).mapToDouble(ScalingStep::threshold)
                .max().orElse(-1.0);
            if (highestScaleIn >= lowestScaleOut)
                throw new IllegalArgumentException(
                    "scale-in thresholds must be strictly below all scale-out thresholds");
        }
    }

    record CustomScalingConfig(
        String beanName,
        Duration cooldown,
        Duration scaleInCooldown
    ) implements ScalingConfig {
        public CustomScalingConfig {
            Objects.requireNonNull(beanName);
            if (beanName.isBlank())
                throw new IllegalArgumentException("beanName must not be blank");
            if (cooldown == null) cooldown = Duration.ofSeconds(60);
            if (scaleInCooldown == null) scaleInCooldown = cooldown;
        }
    }

    record NoScalingConfig() implements ScalingConfig {
        public static final NoScalingConfig INSTANCE = new NoScalingConfig();
        @Override public Duration cooldown() { return Duration.ZERO; }
        @Override public Duration scaleInCooldown() { return Duration.ZERO; }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ScalingConfigTest`
Expected: PASS (9 tests)

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingConfigTest.java
git commit -m "feat(#206): add ScalingConfig sealed hierarchy"
```

### Task 6: AgentSessionManager — effectiveMaxActive, demand counters, adjustMaxActive

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentSessionManagerTest.java`

**Interfaces:**
- Consumes: `PoolSnapshot.DemandMetrics`
- Produces: `AgentSessionManager.adjustMaxActive(int newMax) → int`
- Produces: `AgentSessionManager.snapshotAndResetDemandMetrics() → PoolSnapshot.DemandMetrics`

- [ ] **Step 1: Write tests for new manager methods**

Add to `AgentSessionManagerTest.java`:

```java
@Test
void adjustMaxActive_clampsToHardCeiling() {
    manager = createManager(0, 10);
    int actual = manager.adjustMaxActive(15);
    assertThat(actual).isEqualTo(10); // hard ceiling
}

@Test
void adjustMaxActive_clampsToMinActive() {
    manager = createManager(3, 10);
    int actual = manager.adjustMaxActive(1);
    assertThat(actual).isEqualTo(3); // floor
}

@Test
void adjustMaxActive_setsEffectiveMax() {
    manager = createManager(0, 10);
    manager.adjustMaxActive(7);
    var status = manager.status();
    assertThat(status.max()).isEqualTo(7); // status reports effectiveMaxActive
}

@Test
void adjustMaxActive_clampsToActiveCount() {
    manager = createManager(0, 10);
    manager.acquireSession("a", "/ws/1");
    manager.acquireSession("b", "/ws/2");
    manager.acquireSession("c", "/ws/3");
    manager.acquireSession("d", "/ws/4");
    // 4 active sessions — cannot lower effectiveMax below 4
    int actual = manager.adjustMaxActive(2);
    assertThat(actual).isEqualTo(4); // clamped to activeCount
}

@Test
void adjustMaxActive_affectsAcquireCapacity() {
    manager = createManager(0, 10);
    manager.adjustMaxActive(2);
    manager.acquireSession("a", "/ws/1");
    manager.acquireSession("b", "/ws/2");
    // at effectiveMax=2, next acquire should evict or throw
    // with minActive=0, it should evict
    var s3 = manager.acquireSession("c", "/ws/3");
    assertThat(s3).isNotNull();
    assertThat(suspendCount.get()).isEqualTo(1); // one evicted
}

@Test
void snapshotAndResetDemandMetrics_returnsCountsAndResets() {
    manager = createManager(0, 2);
    manager.acquireSession("a", "/ws/1");
    manager.acquireSession("b", "/ws/2");
    // this acquire triggers eviction
    manager.acquireSession("c", "/ws/3");

    var metrics = manager.snapshotAndResetDemandMetrics();
    assertThat(metrics.acquires()).isEqualTo(3);
    assertThat(metrics.evictions()).isEqualTo(1);
    assertThat(metrics.exhaustions()).isZero();

    // after reset, counters are zero
    var metricsAfterReset = manager.snapshotAndResetDemandMetrics();
    assertThat(metricsAfterReset.acquires()).isZero();
    assertThat(metricsAfterReset.evictions()).isZero();
}

@Test
void snapshotDemandMetrics_countsExhaustions() {
    manager = createManager(2, 2); // minActive = maxActive, can't evict
    manager.acquireSession("a", "/ws/1");
    manager.acquireSession("b", "/ws/2");

    try {
        manager.acquireSession("c", "/ws/3");
    } catch (AgentPoolExhaustedException e) {
        // expected
    }

    var metrics = manager.snapshotAndResetDemandMetrics();
    assertThat(metrics.acquires()).isEqualTo(3);
    assertThat(metrics.exhaustions()).isEqualTo(1);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentSessionManagerTest`
Expected: FAIL — `adjustMaxActive` and `snapshotAndResetDemandMetrics` not found

- [ ] **Step 3: Add fields to AgentSessionManager**

Add to `AgentSessionManager`:
- `private volatile int effectiveMaxActive` — initialized from `config.maxActive()` in constructor
- `private int acquireCount`, `private int evictionCount`, `private int exhaustionCount` — all zero, accessed under lock

- [ ] **Step 4: Modify acquireSession to use effectiveMaxActive**

Change `activeCount() >= config.maxActive()` to `activeCount() >= effectiveMaxActive` in both `acquireSession()` and `resumeSession()`. Add `acquireCount++` at entry to `acquireSession()` (inside the lock).

- [ ] **Step 5: Modify evictOne to track counters**

Add `evictionCount++` after successful eviction. Add `exhaustionCount++` before throwing `AgentPoolExhaustedException`.

- [ ] **Step 6: Implement adjustMaxActive**

```java
public int adjustMaxActive(int newMax) {
    lock.lock();
    try {
        int clamped = Math.min(newMax, config.maxActive());
        clamped = Math.max(clamped, config.minActive());
        clamped = Math.max(clamped, activeCount());
        effectiveMaxActive = clamped;
        return clamped;
    } finally {
        lock.unlock();
    }
}
```

- [ ] **Step 7: Implement snapshotAndResetDemandMetrics**

```java
public PoolSnapshot.DemandMetrics snapshotAndResetDemandMetrics() {
    lock.lock();
    try {
        var metrics = new PoolSnapshot.DemandMetrics(evictionCount, exhaustionCount, acquireCount);
        evictionCount = 0;
        exhaustionCount = 0;
        acquireCount = 0;
        return metrics;
    } finally {
        lock.unlock();
    }
}
```

- [ ] **Step 8: Modify status() to report effectiveMaxActive**

Change `status()` to use `effectiveMaxActive` instead of `config.maxActive()`.

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentSessionManagerTest`
Expected: PASS (all existing + 7 new tests)

- [ ] **Step 10: Run full casehub module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: PASS (all ~342 tests)

- [ ] **Step 11: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentSessionManagerTest.java
git commit -m "feat(#206): add effectiveMaxActive, demand counters, adjustMaxActive to AgentSessionManager"
```

---

## Batch 4: YAML Parsing and PoolConfig Integration

### Task 7: PoolConfig scaling field, YAML parsing, schema

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java` — add `ScalingConfig scaling` to `PoolConfig`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java` — parse `scaling:` section
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java` — add scaling parameters
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParserTest.java` — scaling YAML tests
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionTest.java` — verify PoolConfig with scaling

**Interfaces:**
- Consumes: `ScalingConfig`, `ScalingConfig.TargetTrackingConfig`, `ScalingConfig.StepConfig`, `ScalingConfig.CustomScalingConfig`, `ScalingConfig.NoScalingConfig`, `ScalingStep`
- Produces: `PoolConfig.scaling()` → `ScalingConfig` (defaults to `NoScalingConfig.INSTANCE`)

- [ ] **Step 1: Write YAML parsing tests for scaling section**

Add to `AgentPoolYamlParserTest.java`:

```java
@Test
void targetTrackingScaling() {
    var yaml = """
            agent-pools:
              reviewer:
                pool:
                  max-active: 10
                  scaling:
                    type: target-tracking
                    target: 0.7
                    cooldown: 60s
                    scale-in-cooldown: 300s
            """;

    var def = parser.parse(yaml).getFirst();
    var scaling = def.pool().scaling();
    assertThat(scaling).isInstanceOf(ScalingConfig.TargetTrackingConfig.class);
    var tt = (ScalingConfig.TargetTrackingConfig) scaling;
    assertThat(tt.targetFillRatio()).isEqualTo(0.7);
    assertThat(tt.cooldown()).isEqualTo(java.time.Duration.ofSeconds(60));
    assertThat(tt.scaleInCooldown()).isEqualTo(java.time.Duration.ofSeconds(300));
}

@Test
void stepScaling() {
    var yaml = """
            agent-pools:
              reviewer:
                pool:
                  scaling:
                    type: step
                    cooldown: 30s
                    steps:
                      - threshold: 0.8
                        adjustment: 2
                      - threshold: 0.3
                        adjustment: -1
            """;

    var def = parser.parse(yaml).getFirst();
    var scaling = def.pool().scaling();
    assertThat(scaling).isInstanceOf(ScalingConfig.StepConfig.class);
    var step = (ScalingConfig.StepConfig) scaling;
    assertThat(step.steps()).hasSize(2);
    assertThat(step.cooldown()).isEqualTo(java.time.Duration.ofSeconds(30));
}

@Test
void noScalingSectionDefaultsToNoOp() {
    var yaml = """
            agent-pools:
              reviewer:
                pool:
                  max-active: 5
            """;

    var def = parser.parse(yaml).getFirst();
    assertThat(def.pool().scaling()).isInstanceOf(ScalingConfig.NoScalingConfig.class);
}

@Test
void customScalingType() {
    var yaml = """
            agent-pools:
              reviewer:
                pool:
                  scaling:
                    type: demand-pressure
                    cooldown: 45s
            """;

    var def = parser.parse(yaml).getFirst();
    var scaling = def.pool().scaling();
    assertThat(scaling).isInstanceOf(ScalingConfig.CustomScalingConfig.class);
    var custom = (ScalingConfig.CustomScalingConfig) scaling;
    assertThat(custom.beanName()).isEqualTo("demand-pressure");
    assertThat(custom.cooldown()).isEqualTo(java.time.Duration.ofSeconds(45));
}

@Test
void scalingCooldownDefaultsTo60s() {
    var yaml = """
            agent-pools:
              reviewer:
                pool:
                  scaling:
                    type: target-tracking
                    target: 0.7
            """;

    var def = parser.parse(yaml).getFirst();
    var scaling = (ScalingConfig.TargetTrackingConfig) def.pool().scaling();
    assertThat(scaling.cooldown()).isEqualTo(java.time.Duration.ofSeconds(60));
    assertThat(scaling.scaleInCooldown()).isEqualTo(java.time.Duration.ofSeconds(60));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolYamlParserTest`
Expected: FAIL — `scaling()` method not found on `PoolConfig`

- [ ] **Step 3: Add `ScalingConfig scaling` to PoolConfig record**

Modify `AgentPoolDefinition.PoolConfig`:
```java
public record PoolConfig(int minActive, int maxActive, EvictionStrategy eviction, ScalingConfig scaling) {
    public PoolConfig {
        if (minActive < 0) throw new IllegalArgumentException("minActive must be >= 0");
        if (maxActive < 1) throw new IllegalArgumentException("maxActive must be >= 1");
        if (maxActive < minActive) throw new IllegalArgumentException("maxActive must be >= minActive");
        if (eviction == null) eviction = EvictionStrategy.MEMORY_WEIGHTED;
        if (scaling == null) scaling = ScalingConfig.NoScalingConfig.INSTANCE;
    }
}
```

Update the `PoolBuilder` to carry a `ScalingConfig scaling` field and expose a `scaling(ScalingConfig)` setter. Update `build()` to pass `scaling` to the `PoolConfig` constructor.

Fix all existing call sites that construct `PoolConfig` with 3 args — add the 4th `scaling` parameter (or rely on the `null` default via compact constructor).

- [ ] **Step 4: Add scaling YAML parsing to AgentPoolYamlParser**

Add a `parseScaling(Map<String, Object> scalingMap)` method to `AgentPoolYamlParser`:
- If `scalingMap` is null → return `NoScalingConfig.INSTANCE`
- Read `type` string. Parse `cooldown` and `scale-in-cooldown` as `Duration`.
- `"target-tracking"` → read `target` double → `TargetTrackingConfig`
- `"step"` → read `steps` list → `StepConfig`
- `"none"` → `NoScalingConfig.INSTANCE`
- Anything else → `CustomScalingConfig(type, cooldown, scaleInCooldown)`

Call `parseScaling()` from `toDefinition()` with the `scaling` sub-map from the `pool` section.

- [ ] **Step 5: Update AgentPoolSchema**

Add schema parameters for `pool.scaling.type`, `pool.scaling.target`, `pool.scaling.cooldown`, `pool.scaling.scale-in-cooldown`, `pool.scaling.steps`. Mark all as optional.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolYamlParserTest`
Expected: PASS (all existing + 5 new)

- [ ] **Step 7: Run full casehub module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: PASS (all tests — PoolConfig 4th param is handled by null default)

- [ ] **Step 8: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParserTest.java
git commit -m "feat(#206): add scaling config to PoolConfig, YAML parsing, schema"
```

---

## Batch 5: ScalingScheduler and AgentPoolManagerRegistry

### Task 8: AgentPoolManagerRegistry and ScalingScheduler

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolManagerRegistry.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolManagerRegistryTest.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingSchedulerTest.java`

**Interfaces:**
- Consumes: `AgentPoolDefinitionRegistry`, `AgentSessionManager`, `ScalingPolicy`, `ScalingConfig`, `PoolSnapshot`, `ScalingDecision`, `ScalingDirection`
- Produces: `AgentPoolManagerRegistry.register(String poolName, AgentSessionManager manager)`, `.get(String) → Optional<AgentSessionManager>`, `.poolNames() → Set<String>`
- Produces: `ScalingScheduler.tick()` — `@Scheduled` method

- [ ] **Step 1: Write AgentPoolManagerRegistry test**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AgentPoolManagerRegistryTest {

    @Test
    void registerAndGet() {
        var registry = new AgentPoolManagerRegistry();
        var manager = new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), new StubOps());
        registry.register("reviewer", manager);
        assertThat(registry.get("reviewer")).isPresent().hasValue(manager);
    }

    @Test
    void getMissingReturnsEmpty() {
        var registry = new AgentPoolManagerRegistry();
        assertThat(registry.get("missing")).isEmpty();
    }

    @Test
    void poolNamesReturnsAll() {
        var registry = new AgentPoolManagerRegistry();
        registry.register("a", new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), new StubOps()));
        registry.register("b", new AgentSessionManager(
            new AgentSessionManagerConfig(0, 5), new StubOps()));
        assertThat(registry.poolNames()).containsExactlyInAnyOrder("a", "b");
    }

    // Minimal stub — only needs to compile, not be functional
    private static class StubOps implements SessionOperations {
        @Override public String create(String i, String w) { return "s"; }
        @Override public String conversationId(String s) { return "c"; }
        @Override public void suspend(String s) {}
        @Override public void resume(String s, String c, String w) {}
        @Override public void destroy(String s) {}
        @Override public long memoryBytes(String s) { return 0; }
    }
}
```

- [ ] **Step 2: Write ScalingScheduler test**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;
import static org.assertj.core.api.Assertions.assertThat;

class ScalingSchedulerTest {

    private AgentPoolDefinitionRegistry defRegistry;
    private AgentPoolManagerRegistry mgrRegistry;
    private AtomicInteger createCount;
    private AgentSessionManager manager;

    @BeforeEach
    void setUp() {
        defRegistry = new AgentPoolDefinitionRegistry();
        mgrRegistry = new AgentPoolManagerRegistry();
        createCount = new AtomicInteger();
    }

    private AgentSessionManager createManager(int min, int max) {
        return new AgentSessionManager(
            new AgentSessionManagerConfig(min, max),
            new SessionOperations() {
                @Override public String create(String i, String w) {
                    return "s-" + createCount.incrementAndGet();
                }
                @Override public String conversationId(String s) { return "c-" + s; }
                @Override public void suspend(String s) {}
                @Override public void resume(String s, String c, String w) {}
                @Override public void destroy(String s) {}
                @Override public long memoryBytes(String s) { return 0; }
            }
        );
    }

    @Test
    void tickEvaluatesPolicyAndAdjustsMax() {
        var def = AgentPoolDefinition.builder()
            .agent("reviewer")
            .pool()
                .maxActive(20)
                .scaling(new ScalingConfig.TargetTrackingConfig(0.7, null, null))
            .build();
        defRegistry.register(def);

        manager = createManager(0, 20);
        // Reduce effectiveMaxActive to 10, then fill to 80%
        manager.adjustMaxActive(10);
        mgrRegistry.register("reviewer", manager);

        for (int i = 0; i < 8; i++) {
            manager.acquireSession("reviewer", "/ws/" + i, null, WorkingDirPolicy.SHARED_READ);
        }

        var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
        scheduler.tick();

        // effectiveMaxActive should have increased from 10 (hard ceiling is 20)
        assertThat(manager.status().max()).isGreaterThan(10);
    }

    @Test
    void tickSkipsPoolsWithNoScaling() {
        var def = AgentPoolDefinition.builder()
            .agent("reviewer")
            .pool().maxActive(10)
            .build(); // no scaling config → NoScalingConfig
        defRegistry.register(def);

        manager = createManager(0, 10);
        mgrRegistry.register("reviewer", manager);

        var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
        scheduler.tick();

        assertThat(manager.status().max()).isEqualTo(10); // unchanged
    }

    @Test
    void cooldownPreventsConsecutiveScaling() {
        var def = AgentPoolDefinition.builder()
            .agent("reviewer")
            .pool()
                .maxActive(30)
                .scaling(new ScalingConfig.TargetTrackingConfig(
                    0.7, Duration.ofSeconds(300), null))
            .build();
        defRegistry.register(def);

        manager = createManager(0, 30);
        manager.adjustMaxActive(20); // start with effectiveMax=20
        mgrRegistry.register("reviewer", manager);

        for (int i = 0; i < 16; i++) {
            manager.acquireSession("reviewer", "/ws/" + i, null, WorkingDirPolicy.SHARED_READ);
        }

        var scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
        scheduler.tick(); // first tick scales — fill=16/20=0.8 > 0.7
        int maxAfterFirst = manager.status().max();
        assertThat(maxAfterFirst).isGreaterThan(20);

        scheduler.tick(); // second tick should be in cooldown
        assertThat(manager.status().max()).isEqualTo(maxAfterFirst); // unchanged
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolManagerRegistryTest,ScalingSchedulerTest`
Expected: FAIL — classes not found

- [ ] **Step 4: Create AgentPoolManagerRegistry**

```java
package io.casehub.claudony.casehub.fleet;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class AgentPoolManagerRegistry {

    private final Map<String, AgentSessionManager> managers = new ConcurrentHashMap<>();

    public void register(String poolName, AgentSessionManager manager) {
        managers.put(poolName, manager);
    }

    public Optional<AgentSessionManager> get(String poolName) {
        return Optional.ofNullable(managers.get(poolName));
    }

    public Set<String> poolNames() {
        return managers.keySet();
    }
}
```

- [ ] **Step 5: Create ScalingScheduler**

```java
package io.casehub.claudony.casehub.fleet;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkus.scheduler.Scheduled;

import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ScalingScheduler {

    private static final java.util.logging.Logger LOG =
        java.util.logging.Logger.getLogger(ScalingScheduler.class.getName());

    private final AgentPoolDefinitionRegistry defRegistry;
    private final AgentPoolManagerRegistry mgrRegistry;
    private final Map<String, Instant> lastScaleOut = new ConcurrentHashMap<>();
    private final Map<String, Instant> lastScaleIn = new ConcurrentHashMap<>();
    private final Map<String, ScalingPolicy> policyCache = new ConcurrentHashMap<>();

    @Inject
    public ScalingScheduler(AgentPoolDefinitionRegistry defRegistry,
                            AgentPoolManagerRegistry mgrRegistry) {
        this.defRegistry = defRegistry;
        this.mgrRegistry = mgrRegistry;
    }

    @Scheduled(every = "${claudony.scaling.interval:15s}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void tick() {
        for (String poolName : mgrRegistry.poolNames()) {
            evaluatePool(poolName);
        }
    }

    private void evaluatePool(String poolName) {
        var manager = mgrRegistry.get(poolName).orElse(null);
        var definition = defRegistry.get(poolName).orElse(null);
        if (manager == null || definition == null) return;

        var scalingConfig = definition.pool().scaling();
        if (scalingConfig instanceof ScalingConfig.NoScalingConfig) return;

        var demand = manager.snapshotAndResetDemandMetrics();

        if (inCooldown(poolName, scalingConfig)) return;

        var status = manager.status();
        var snapshot = new PoolSnapshot(
            status.active(), status.idle(), status.min(), status.max(), demand);

        var policy = policyCache.computeIfAbsent(poolName, k -> policyFor(scalingConfig));
        if (policy == null) return;

        var decision = policy.evaluate(snapshot);

        if (decision.direction() != ScalingDirection.NONE) {
            int currentMax = status.max();
            int newMax = switch (decision.direction()) {
                case OUT -> currentMax + decision.count();
                case IN  -> currentMax - decision.count();
                case NONE -> currentMax;
            };
            int actualMax = manager.adjustMaxActive(newMax);
            if (actualMax != currentMax) {
                recordCooldown(poolName, decision.direction());
            }
        }
    }

    private ScalingPolicy policyFor(ScalingConfig config) {
        return switch (config) {
            case ScalingConfig.TargetTrackingConfig t ->
                new TargetTrackingPolicy(t.targetFillRatio());
            case ScalingConfig.StepConfig s ->
                new StepScalingPolicy(s.steps());
            case ScalingConfig.CustomScalingConfig c ->
                resolveCustomPolicy(c.beanName());
            case ScalingConfig.NoScalingConfig n ->
                new NoOpScalingPolicy();
        };
    }

    private ScalingPolicy resolveCustomPolicy(String beanName) {
        try {
            var instance = jakarta.enterprise.inject.spi.CDI.current()
                .select(ScalingPolicy.class, new jakarta.enterprise.util.NamedLiteral(beanName));
            if (instance.isResolvable()) return instance.get();
            LOG.warning("Custom ScalingPolicy bean '" + beanName + "' not found — scaling disabled for this pool");
        } catch (Exception e) {
            LOG.warning("Failed to resolve custom ScalingPolicy bean '" + beanName + "': " + e.getMessage());
        }
        return null;
    }

    private boolean inCooldown(String poolName, ScalingConfig config) {
        Instant now = Instant.now();
        var outTime = lastScaleOut.get(poolName);
        if (outTime != null && Duration.between(outTime, now).compareTo(config.cooldown()) < 0) {
            return true;
        }
        var inTime = lastScaleIn.get(poolName);
        return inTime != null && Duration.between(inTime, now).compareTo(config.scaleInCooldown()) < 0;
    }

    private void recordCooldown(String poolName, ScalingDirection direction) {
        if (direction == ScalingDirection.OUT) {
            lastScaleOut.put(poolName, Instant.now());
        } else if (direction == ScalingDirection.IN) {
            lastScaleIn.put(poolName, Instant.now());
        }
    }
}
```

- [ ] **Step 6: Add `scaling(ScalingConfig)` to PoolBuilder**

In `AgentPoolDefinition.PoolBuilder`, add:
```java
public PoolBuilder scaling(ScalingConfig scaling) {
    parent.scaling = scaling;
    return this;
}
```

And the `scaling` field to `Builder` with a corresponding pass-through to `PoolConfig`.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolManagerRegistryTest,ScalingSchedulerTest`
Expected: PASS (6 tests total)

- [ ] **Step 8: Run full casehub module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolManagerRegistry.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java \
       casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolManagerRegistryTest.java \
       casehub/src/test/java/io/casehub/claudony/casehub/fleet/ScalingSchedulerTest.java
git commit -m "feat(#206): add ScalingScheduler with per-pool cooldown tracking"
```

- [ ] **Step 10: Run all tests across all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS (~823+ tests)

- [ ] **Step 11: Commit any fixups**

If any cross-module test failures surfaced, fix and commit here.

---

## Batch 6: Wiring

### Task 9: Wire ClaudonyAgentBackend to AgentPoolManagerRegistry

**Files:**
- Modify: `app/src/main/java/dev/claudony/server/ServerStartup.java` (or the class that creates `AgentSessionManager`)
- Test: Verify via existing integration tests that the scheduler can reach the manager

**Interfaces:**
- Consumes: `AgentPoolManagerRegistry.register(String, AgentSessionManager)`, `AgentPoolDefinitionRegistry.names()`

> **Note:** `ClaudonyAgentBackend` creates `AgentSessionManager` from `AgentPoolConfig` (a `@ConfigMapping` interface) — it does not currently know its pool name. The pool name must come from the `AgentPoolDefinitionRegistry`. If only one pool is registered, use that name. If the definition registry is empty (legacy config-only path), use a default name like `"default"`. This wiring is in the `app` module, not `casehub`.

- [ ] **Step 1: Inject AgentPoolManagerRegistry into the startup class**

Inject `AgentPoolManagerRegistry` alongside the existing `AgentPoolDefinitionRegistry`.

- [ ] **Step 2: After creating the AgentSessionManager, register it**

```java
String poolName = defRegistry.names().stream().findFirst().orElse("default");
managerRegistry.register(poolName, sessionManager);
```

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS — no regressions. The `ScalingScheduler` can now find the manager via `AgentPoolManagerRegistry`.

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/dev/claudony/server/ServerStartup.java
git commit -m "feat(#206): wire AgentSessionManager into AgentPoolManagerRegistry for scaling"
```

## References

- [2026-09-29-auto-scaling-policies-design.md] — design spec this plan implements
- [AgentSessionManager.java:1-216] — pool orchestration (effectiveMaxActive, demand counters)
- [EvictionPolicy.java:1-8] — existing pure-function SPI pattern
- [AgentPoolDefinition.java:1-158] — pool definition model (PoolConfig, PoolBuilder)
- [AgentPoolYamlParser.java:1-97] — YAML config parsing
- [AgentPoolSchema.java:1-59] — schema validation
- [AgentPoolDefinitionRegistry.java:1-45] — definition registry pattern
- [DefaultEvictionPolicyTest.java] — test style reference
- [AgentSessionManagerTest.java] — test setup pattern reference
- [GitHub #206] — focal issue
- [GitHub #205] — parent spec (D5: scaling deferred, D10: identity-correlated sessions)
- [GitHub #229] — EvictionPolicy SPI (same pattern)
