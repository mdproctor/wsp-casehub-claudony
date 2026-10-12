## D1: Usage pattern for pre-warming

**Choice:** Recency-based — resume the N most-recently-suspended sessions
**Alternatives:**
- Frequency-based — track acquire frequency per identity over a window. Better for bursty patterns but more state to maintain.
- Time-of-day — learn which identities are active at which times. Most sophisticated but overkill for MVP.
**Rationale:** Simple, predictable, handles the common case where the same identities return. Suspended sessions already have recency data (`lastInteraction`).
**Trade-offs:** Doesn't handle burst patterns where an identity uses the pool intensely then stops. Acceptable for MVP — frequency-based can be added later as a refinement.
**Sources:** AgentSessionManager.java (ManagedSession.lastInteraction), issue #243
**Exploration:** quick
**Status:** captured

## D2: Integration with scaling system

**Choice:** New `ProactiveConfig` variant in the sealed `ScalingConfig` hierarchy, with the scheduler resuming sessions directly
**Alternatives:**
- Separate `ProactiveWarmingScheduler` — clean separation but coordination overhead with two tick loops on the same pools.
- `ScalingPolicy` + post-hook — overloads ScaleOut meaning (raise ceiling vs fill it), two-phase fragility.
**Rationale:** The scheduler already has tick loop, cooldown, and pool traversal infrastructure. Proactive warming is another kind of scaling action — it belongs in the same evaluation loop. The sealed hierarchy makes the new variant type-safe.
**Trade-offs:** Proactive warming is conceptually different from capacity scaling (direct action vs ceiling adjustment). Mixing them in one scheduler adds a branch in `evaluatePool()`. Acceptable — the scheduler is the natural home for periodic pool actions.
**Sources:** ScalingScheduler.java, ScalingConfig.java (sealed hierarchy), ScalingPolicy.java
**Exploration:** quick
**Depends on:** D1 (recency-based pattern)
**Status:** captured
