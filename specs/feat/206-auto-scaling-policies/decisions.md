## D1: Metric source

**Choice:** Utilization-only (activeCount / maxActive)
**Alternatives:**
- Queue depth — requires new request queuing infrastructure that doesn't exist yet (YAGNI)
- Idle-time — closer to eviction than scaling; EvictionPolicy already handles this concern
- External metrics (CaseHub work queue) — requires integration abstraction with no other use today
**Rationale:** Utilization is universally meaningful, already available via AgentSessionManager.activeCount(), and maps naturally to both target-tracking and step scaling policies
**Trade-offs:** Can't react to demand before it manifests as utilization pressure. Queue-depth would catch demand spikes earlier.
**Sources:** AgentSessionManager.java:159 (activeCount), issue #206
**Exploration:** quick
**Status:** captured

## D2: Scaling action mode

**Choice:** Configurable mode — `proactive` (default) or `reactive`, set per-pool in YAML
**Alternatives:**
- Proactive-only — simpler but ceiling-adjustment is useful for resource limiting without pre-warming
- Reactive-only — simpler but defeats the latency benefit of scaling ahead of demand
**Rationale:** Both modes serve genuine use cases (latency optimization vs resource capping). Config option with sensible default keeps simple case simple.
**Trade-offs:** Two code paths to maintain. Reactive mode is slightly awkward with target-tracking (adjusting ceiling without creating sessions).
**Sources:** AgentPoolDefinition.java (AgentConfig carries identity/workingDir/command for pre-warming)
**Exploration:** quick
**Status:** captured

## D3: Cooldown mechanism

**Choice:** Single `cooldown` property with optional `scale-in-cooldown` override
**Alternatives:**
- Separate scale-out-cooldown and scale-in-cooldown (always two values) — more explicit but verbose for the common case
- No cooldown (rely on policy to self-limit) — dangerous, causes oscillation
**Rationale:** Most users want one cooldown value. Asymmetric cooldowns are a tuning concern, not a primary design concern. Optional override keeps the simple case to one property.
**Trade-offs:** Users who need fine-grained control must learn the override property exists.
**Sources:** AWS Auto Scaling cooldown model
**Exploration:** quick
**Status:** captured

## D4: SPI shape and scheduling

**Choice:** Policy-as-function with external scheduler (Approach A)
**Alternatives:**
- Stateful policy with internal cooldowns — self-contained but forces every SPI implementor to handle cooldowns correctly, harder to test
- Reactive event-driven (no scheduler) — immediate response but adds latency to hot path, can't scale during quiet periods
**Rationale:** Pure function SPI (evaluate(PoolSnapshot) → ScalingDecision) is trivially testable and matches the existing EvictionPolicy pattern. Cooldown logic lives once in the scheduler, not re-implemented per policy. Clean separation: policy decides, scheduler orchestrates, manager executes.
**Trade-offs:** Requires a scheduler thread. Scaling response time is bounded by tick interval (not instant).
**Sources:** EvictionPolicy.java (existing pure-function SPI pattern), AgentSessionManager.java (orchestration pattern)
**Exploration:** quick
**Status:** captured
