# Proactive Scaling Mode — Identity-Aware Pre-Warming

**Issue:** casehubio/claudony#243
**Date:** 2026-10-02
**Status:** Approved

## Problem

Current scaling policies (target-tracking, step, demand-pressure) are all reactive —
they adjust the `maxActive` capacity ceiling based on load metrics but never proactively
create or resume sessions. When demand arrives, session acquisition still pays the full
cold-start cost (tmux session creation, agent bootstrap). For identities that return
frequently, this latency is unnecessary — their suspended sessions could be resumed
before the next request arrives.

Anonymous proactive creation is ruled out by the identity-correlated pool model (D10
from #205). But suspended sessions already have identity — they can be resumed
proactively without violating the model.

## Solution

A new `ProactiveConfig` variant in the sealed `ScalingConfig` hierarchy. When
configured, the `ScalingScheduler` resumes recently-suspended sessions to maintain
a target number of active sessions, using recency as the selection criterion.

## Design

### Component 1: `ScalingConfig.ProactiveConfig`

New sealed variant in `ScalingConfig`:

```java
record ProactiveConfig(
    int targetActive,
    Duration cooldown,
    Duration scaleInCooldown
) implements ScalingConfig {
    public ProactiveConfig {
        if (targetActive < 1) throw new IllegalArgumentException("targetActive must be >= 1");
        if (cooldown == null) cooldown = Duration.ofSeconds(60);
        if (scaleInCooldown == null) scaleInCooldown = cooldown;
    }
}
```

`targetActive` is the desired number of active sessions to maintain. When the actual
active count falls below this, the scheduler resumes suspended sessions to fill the gap.

Added to the sealed `permits` clause and the `type()` switch — type string is `"proactive"`.

### Component 2: Proactive evaluation in `ScalingScheduler`

The proactive path does NOT go through `ScalingPolicy`. It's a direct action in
`ScalingScheduler.evaluatePool()`:

```
if config is ProactiveConfig:
    evaluateProactive(poolName, manager, config)
    return   // skip reactive policy evaluation
```

`evaluateProactive()` logic:

1. Get `activeCount` from `manager.status()`
2. If `activeCount >= targetActive`, emit `ScalingDecision.none()` and return
3. Calculate `deficit = targetActive - activeCount`
4. Clamp deficit to available suspended sessions and capacity ceiling
   (`deficit = min(deficit, suspendedCount, maxActive - activeCount)`)
5. Get suspended sessions sorted by `lastInteraction` descending (most recent first)
6. Resume `deficit` sessions via `manager.resumeSession(id)`
7. Emit a scaling event with direction `PREWARM`, count = resumed count
8. Record `ScalingState` with the decision and cooldown

### Component 3: `ScalingDirection.PREWARM`

New enum value in `ScalingDirection`. Used for event emission and state tracking.
Does not participate in `adjustMaxActive()` — proactive warming doesn't change the
capacity ceiling.

### Component 4: `PoolUpdateRequest` and `PoolService` support

The existing pool update API (`ClaudonyPoolApi.updatePool()`) already supports
changing scaling config. `PoolService.parseScalingConfig()` needs a new `"proactive"`
case that reads `targetActive` from the request.

`PoolDetail.ScalingConfigView` needs a `targetActive` field to surface the config
in the dashboard.

### Interaction with existing scaling

Proactive and reactive scaling are **mutually exclusive per pool** — a pool uses
either a reactive policy (target-tracking, step, demand-pressure) or proactive
warming, not both. The sealed config hierarchy enforces this naturally: each pool
has one `ScalingConfig`.

Proactive warming respects the `maxActive` ceiling: it will never resume sessions
past `maxActive`. If the pool needs more capacity than `maxActive` allows, the
operator must raise `maxActive` separately.

### Configuration

Via YAML pool definition:
```yaml
agent-pools:
  code-reviewer:
    working-dir: /workspace
    command: "claude --model opus"
    pool:
      min-active: 0
      max-active: 10
      scaling:
        type: proactive
        target-active: 3
        cooldown: 30s
```

Via REST API (existing `updatePool` endpoint):
```json
{
  "scalingType": "proactive",
  "targetActive": 3,
  "cooldown": "30s"
}
```

## Integration tests

Plain JUnit with real tmux (same pattern as `FleetPoolIntegrationTest`). Tests:

1. **proactive_resumesSuspendedSessions** — start with 3 suspended sessions,
   configure `targetActive=2`, trigger tick, verify 2 most-recently-suspended
   are now active.

2. **proactive_noActionWhenAtTarget** — active count already equals target,
   verify no sessions resumed.

3. **proactive_respectsMaxActive** — `targetActive=5` but `maxActive=3`,
   verify only 3 sessions active after tick.

4. **proactive_resumesMostRecentFirst** — 3 suspended sessions with different
   `lastInteraction` times, verify the most recent is resumed first.

5. **proactive_cooldownPreventsRapidWarming** — verify tick respects cooldown
   between warming bursts.

## What changes

| File | Change |
|------|--------|
| `ScalingConfig.java` | Add `ProactiveConfig` to sealed permits + type switch |
| `ScalingDirection.java` | Add `PREWARM` value |
| `ScalingScheduler.java` | Add `evaluateProactive()` method, branch in `evaluatePool()` |
| `AgentPoolYamlParser.java` | Parse `proactive` scaling type with `target-active` |
| `PoolService.java` | Parse `"proactive"` in `parseScalingConfig()` |
| `PoolUpdateRequest.java` | Add `targetActive` field |
| `ScalingConfigView.java` | Add `targetActive` field |
| `PoolDetail.java` | Surface `targetActive` in scaling view |
| `CLAUDE.md` | Update test count, add proactive scaling to config properties |

## What doesn't change

- `ScalingPolicy` interface — proactive warming bypasses it
- Existing reactive policies (target-tracking, step, demand-pressure)
- `AgentSessionManager` — already has `resumeSession()` and `sessions()`
- `SessionLifecycleListener` / `PoolMeshRegistrar` — resume triggers `onResumed()` automatically

## References

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java` — sealed hierarchy
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java` — tick loop
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingPolicy.java` — reactive policy interface (unchanged)
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java` — resumeSession(), sessions()
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ManagedSession.java` — lastInteraction()
- [GitHub #243] — proactive scaling issue
- [GitHub #206] — auto-scaling policies spec (deferred proactive)
- [GitHub #205] — D10: identity-correlated pool model constraint
