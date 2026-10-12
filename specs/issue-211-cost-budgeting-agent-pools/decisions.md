## D1: Budget scope

**Choice:** Per-pool budgets with global defaults
**Alternatives:**
- Per-session — finer-grained but harder to manage across a fleet
- Global (all pools) — simplest config but no per-pool differentiation
- Per-pool + per-session — most flexible but doubles config surface
**Rationale:** Pools are already the capacity management unit (min/max/scaling/eviction). Budget is another pool policy.
**Trade-offs:** No per-session limits within a pool — a single session could consume the entire pool budget
**Sources:** Issue #211 description ("budget enforcement as a pool policy"), existing PoolConfig record
**Exploration:** quick
**Status:** captured

## D2: Budget metric

**Choice:** Dollar cost AND token ceiling (whichever hits first)
**Alternatives:**
- Dollar cost only — simpler, but tokens are a useful proxy when pricing changes
- Tokens only — doesn't map to actual spend
**Rationale:** Dollars are what you pay; tokens are what you consume. Both are independently useful constraints. Output tokens cost ~5x input on most models, so token limits catch asymmetric workloads that dollar limits might miss.
**Trade-offs:** Two limits to configure and reason about. Dashboard must show both.
**Sources:** Claude CLI JSON output (provides both `total_cost_usd` and token counts per model)
**Exploration:** quick
**Status:** captured

## D3: Time model

**Choice:** Rolling window
**Alternatives:**
- Calendar period — matches billing cycles but creates burst risk at boundaries
- Lifetime (no reset) — simple but requires manual reset
**Rationale:** Continuous workloads need continuous measurement. Rolling window avoids clock-alignment surprises.
**Trade-offs:** Slightly more complex to implement (sliding window aggregation vs. simple counter reset)
**Sources:** Standard rate-limiting practice; AuthRateLimiter in claudony already uses sliding windows
**Exploration:** quick
**Status:** captured

## D4: Enforcement action

**Choice:** Configurable policy — all three supported: `suspend` | `block-new` | `alert`
**Alternatives:**
- Single hardcoded enforcement — less flexible, forces one-size-fits-all
**Rationale:** Different pools have different risk profiles. A research pool can tolerate alert-only; a production pool needs hard stops. Per-pool override with global default.
**Trade-offs:** Three enforcement paths to implement and test
**Sources:** User direction: "these are all policies... configured in the ops yaml file"
**Exploration:** quick
**Status:** captured

## D5: Cost data source

**Choice:** MCP self-reporting tool
**Alternatives:**
- Session file parsing — depends on internal CLI file format stability
- Pipe-pane output parsing — fragile, format changes break it
- Anthropic API usage endpoint — org-level, not per-session, reporting delays
**Rationale:** Structured, real-time, under our control. Claude sessions know their own cost and can report it via MCP call to Claudony.
**Trade-offs:** Requires Claude to cooperate (system prompt instruction). If the session doesn't call the MCP tool, cost goes untracked.
**Mitigation (review round 1):** Add a configurable "no-report timeout" — if a session hasn't reported cost within N minutes of its last interaction, flag it or apply enforcement. Prevents uncooperative sessions from running indefinitely untracked.
**Sources:** Claude CLI `--output-format json` output structure (total_cost_usd, modelUsage per model)
**Exploration:** quick
**Status:** captured

## D6: Report interval

**Choice:** Configurable policy: `turn` | `periodic(N)` | `completion`
**Alternatives:**
- Single hardcoded interval — less flexible
**Rationale:** Same reasoning as enforcement — different pools have different needs. High-value pools want per-turn tracking; low-risk pools can report at completion.
**Trade-offs:** System prompt for Claude sessions must be dynamically generated based on pool config
**Sources:** User direction: "again this is policy based should be part of the ops deployment yaml"
**Exploration:** quick
**Status:** captured

## D7: Budget state persistence

**Choice:** PostgreSQL (Qhorus datasource)
**Alternatives:**
- In-memory only — restart resets budget counters, risk of overspend
- Best-effort file flush — loses up to one interval on crash
**Rationale:** Budget enforcement is a production guardrail. Restart should not reset the budget counter mid-window. Qhorus datasource already available with Flyway.
**Trade-offs:** Requires a Flyway migration, DB writes on each cost report
**Mitigation (review round 1):** Batch writes — accumulate cost reports in-memory, flush to DB on scheduler tick. Lose at most one tick interval on crash. Acceptable tradeoff for reduced write amplification under per-turn reporting.
**Sources:** Existing Qhorus datasource config, Flyway migration pattern in claudony
**Exploration:** quick
**Status:** captured

## D8: Architecture — where budget enforcement lives

**Choice:** Budget as a parallel concern evaluated by the existing ScalingScheduler
**Alternatives:**
- Independent interceptor — immediate enforcement but more moving parts, potential ordering issues with scaling
- Pool policy engine — correct long-term architecture but premature for two policies
**Rationale:** Smallest change that delivers the feature. Reuses scheduler tick, event emission, and dashboard plumbing. 15s enforcement delay acceptable because cost reports are inherently async.
**Trade-offs:** Enforcement delayed up to one scheduler tick (configurable, default 15s). Budget and scaling evaluated in same loop — must handle interaction (budget vetoes scale-out).
**Mitigation (review round 1):** Add a fast-path CDI observer on cost report events that checks budget immediately and triggers enforcement if threshold crossed. Scheduler remains the backstop for missed reports and periodic re-evaluation.
**Sources:** Existing ScalingScheduler, PoolEventEmitter, ScalingConfig sealed interface
**Exploration:** quick
**Depends on:** D4 (enforcement action)
**Status:** captured

## D9: Runtime mutability

**Choice:** Budget config adjustable at runtime via REST/MCP API (same as scaling and capacity)
**Alternatives:**
- YAML-only (restart required) — simpler but operationally inflexible
**Rationale:** User stated budget should be "configured remotely too". Follows existing pattern — PoolUpdateRequest already supports runtime scaling and capacity changes.
**Trade-offs:** Runtime mutation must invalidate cached budget state and re-evaluate enforcement immediately
**Sources:** Existing PoolUpdateRequest, PoolService.updatePool(), AgentPoolDefinitionRegistry.updateScaling()
**Exploration:** quick
**Depends on:** D8 (architecture)
**Status:** captured

## D10: Token counting scope

**Choice:** Total tokens (input + output) as a single ceiling
**Alternatives:**
- Separate input/output limits — more precise but doubles config surface
- Weighted total — captures cost asymmetry but adds obscure config knob
**Rationale:** Simple to reason about. Users who care about the breakdown see it in the dashboard. Dollar limit already captures the cost asymmetry (output tokens cost more).
**Trade-offs:** A workload heavy on output tokens hits the token limit faster than a cache-read-heavy workload, even if dollar cost is similar
**Sources:** Claude CLI JSON output provides input_tokens, output_tokens, cache_creation_input_tokens separately
**Exploration:** quick
**Depends on:** D2 (budget metric)
**Status:** captured
