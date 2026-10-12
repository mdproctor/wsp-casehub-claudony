## D1: DemandMetrics enrichment structure

**Choice:** Flat record enrichment — add new fields directly to `DemandMetrics` (latency fields + `Map<String, Double> externalMetrics`)
**Alternatives:**
- Nested sub-records — unnecessary structure for ~4 new fields
- Separate snapshot types on PoolSnapshot — adds API surface without benefit since policies already get the full snapshot
**Rationale:** DemandMetrics is already a per-tick snapshot and the natural place for all demand signals. The record stays immutable. The external metrics map is open-ended without sealed type explosion.
**Trade-offs:** The record constructor grows from 3 to ~6 parameters — acceptable for a data record, and the ZERO constant absorbs it.
**Sources:** `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSnapshot.java`, #206 spec (DemandMetrics design)
**Exploration:** quick
**Status:** captured

## D2: External metrics SPI design

**Choice:** Pull-based, per-tick — `DemandMetricsSource.collect(poolName)` called by `ScalingScheduler` during each tick, results merged into external metrics map
**Alternatives:**
- Push-based event-driven — unnecessary complexity; scaling tick is 15s so polling is fine
- Reactive streams — overkill for periodic metric collection at this cadence
**Rationale:** Matches the existing tick-based evaluation model. Sources are stateless from the scheduler's perspective. The scheduler owns lifecycle and handles failures gracefully (log + skip). No threading complexity.
**Trade-offs:** Sources that produce data faster than 15s lose intermediate values. Acceptable — scaling decisions don't need sub-second resolution.
**Sources:** `ScalingScheduler.java` tick loop, `EvictionPolicy.java` (same pure-function SPI pattern)
**Exploration:** quick
**Status:** captured

## D3: Demand-pressure policy design

**Choice:** Threshold-based with independent triggers — exhaustionThreshold (int) and latencyThresholdMs (long), either crossing triggers scale-out. Scale-in when both below hysteresis band (50% of threshold).
**Alternatives:**
- Weighted composite score — forces signals to reinforce each other; exhaustions alone justify scaling without waiting for latency confirmation
- AND logic — ignores exhaustions until latency also degrades, defeating the exhaustion signal
**Rationale:** Each signal independently indicates stress. Exhaustions = hard failures (even one per tick matters). High latency = degradation without failures. Independent triggers catch both fast (exhaustion spike) and slow (latency creep) pressure patterns. Simple to configure and reason about.
**Trade-offs:** Independent OR triggers are more aggressive than AND — may scale out more eagerly. Mitigated by cooldown and hysteresis band for scale-in.
**Sources:** `TargetTrackingPolicy.java` (hysteresis band pattern at 70%), `StepScalingPolicy.java` (threshold-based pattern)
**Exploration:** quick
**Depends on:** D1 (latency and exhaustion fields must exist in DemandMetrics)
**Status:** captured

## D4: Latency tracking mechanism in AgentSessionManager

**Choice:** Inline timing with running aggregates — `System.nanoTime()` at entry/exit of `acquireSession()`, update `totalAcquireNanos` and `maxAcquireNanos` under the existing lock. `snapshotAndResetDemandMetrics()` computes average and resets.
**Alternatives:**
- Ring buffer of individual latencies — p99 over 15s with single-digit acquires is statistically meaningless; avg and max are sufficient
- Micrometer Timer integration — unnecessary indirection; policy needs data in DemandMetrics, not in the meter registry
**Rationale:** No new data structures. The lock already exists and all counter mutations happen inside it. Zero allocation overhead — just two long fields. Matches existing snapshot-and-reset pattern.
**Trade-offs:** Timing includes lock acquisition wait time, not just the acquire operation itself. Acceptable — lock contention IS part of the acquire experience from the caller's perspective.
**Sources:** `AgentSessionManager.java:137-148` (snapshotAndResetDemandMetrics pattern)
**Exploration:** quick
**Depends on:** D1 (latency fields in DemandMetrics)
**Status:** captured

## D5: DemandPressurePolicy as ScalingConfig variant

**Choice:** New sealed variant `DemandPressureConfig(int exhaustionThreshold, long latencyThresholdMs, Duration cooldown, Duration scaleInCooldown)` added to the `ScalingConfig` sealed interface. YAML type name `demand-pressure` maps directly to it.
**Alternatives:**
- Use `CustomScalingConfig` with a `@Named` CDI bean — config (thresholds) has nowhere to live; `CustomScalingConfig` only carries beanName and cooldowns, breaking the self-contained YAML model
**Rationale:** Follows the exact same pattern as `TargetTrackingConfig` and `StepConfig`. Sealed hierarchy makes the type system enforce all variants are handled in `createPolicy()`. YAML parsing is a straightforward switch extension.
**Trade-offs:** Adds a new permit to the sealed interface — requires updating the exhaustive switch in `ScalingConfig.type()` and `ScalingScheduler.createPolicy()`. This is the correct cost for a built-in policy type.
**Sources:** `ScalingConfig.java` (sealed hierarchy), `ScalingScheduler.java:127-137` (createPolicy switch)
**Exploration:** quick
**Depends on:** D3 (policy design determines what config fields are needed)
**Status:** captured
