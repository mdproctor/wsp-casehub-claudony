# Cost Budgeting and Limits for Agent Pools — Design Spec

**Issue:** casehubio/claudony#211
**Date:** 2026-10-02
**Status:** Draft

---

## Problem

Agent pools manage capacity (min/max sessions, scaling, eviction) but have no cost awareness. A pool of autonomous agents can run up arbitrary API costs with no visibility or guardrails. The issue was deferred from #205 (agent pool management) with the note: "Token consumption tracking is deferred from the initial agent pool implementation. Budget enforcement (stop at $X) can be added as a pool policy without changing the core pool model."

Today, `ManagedSession` tracks memory and idle time. `SessionOperations` has no cost-related operations. The only "budget" is `ClaudonyDispatchBudget` which counts session slots, not dollars. No code in the codebase parses Claude CLI output for cost data, queries the Anthropic billing API, or tracks token consumption.

## Goal

Add per-pool cost budgeting as a first-class pool policy:

1. **Track** — Claude sessions self-report cost and token usage via a new MCP tool
2. **Accumulate** — rolling-window per-pool spend, persisted to PostgreSQL
3. **Enforce** — configurable enforcement when budget is exceeded (suspend, block new acquires, or alert)
4. **Observe** — budget state in the dashboard, SSE events, Micrometer metrics
5. **Configure** — YAML pool definition + runtime API mutation

---

## Architecture

The cost budgeting system has four layers:

```
Claude Session (tmux)
  → MCP call: report_cost(sessionId, costUsd, inputTokens, outputTokens, ...)
    → BudgetTracker.record(poolName, report)
      → in-memory accumulation + periodic DB flush
      → CDI event: CostReportEvent
        → fast-path budget check → enforce if over
    → ScalingScheduler tick (15s)
      → evaluateScaling(pool)
      → evaluateBudget(pool)
        → BudgetTracker.currentSpend(poolName, window)
        → if over → apply enforcement policy
      → emit PoolLifecycleEvent (budget event type)
        → SSE → dashboard
```

Budget evaluation lives in the existing `ScalingScheduler` as a parallel concern — not a `ScalingConfig` variant, but evaluated in the same tick loop. A fast-path CDI observer provides immediate enforcement when cost reports arrive, without waiting for the next scheduler tick.

---

## Component Design

### 1. YAML Surface

**Global defaults** at the top level of the agent-pools YAML:

```yaml
budget-defaults:
  cost-limit: 100.00          # USD ceiling
  token-limit: 50000000       # total tokens (input + output)
  window: 24h                 # rolling window duration
  enforcement: block-new      # suspend | block-new | alert
  report-interval: turn       # turn | periodic(5) | completion
  no-report-timeout: 10m      # flag sessions silent longer than this
```

**Per-pool override** in the `budget:` section:

```yaml
agent-pools:
  code-reviewer:
    working-dir: /workspace/reviews
    command: claude --model opus
    pool:
      min-active: 2
      max-active: 10
      eviction: memory-weighted
      scaling:
        type: target-tracking
        target: 0.7
      budget:
        cost-limit: 50.00       # overrides global
        token-limit: 10000000   # overrides global
        window: 1h              # tighter window for expensive pool
        enforcement: suspend    # hard stop for this pool
        report-interval: turn
        no-report-timeout: 5m

  research-assistant:
    working-dir: /workspace/research
    pool:
      min-active: 0
      max-active: 5
      budget:
        enforcement: alert      # soft limit — don't interrupt research
        report-interval: periodic(10)
```

Per-pool `budget:` fields override individual defaults. Unspecified fields fall back to `budget-defaults`. If no `budget-defaults` and no per-pool `budget:`, budgeting is disabled for that pool (no enforcement, no tracking).

### 2. BudgetConfig — Pool Definition Extension

`BudgetConfig` becomes a new nullable field on `AgentPoolDefinition.PoolConfig`:

```java
public record PoolConfig(
    int minActive, int maxActive,
    EvictionStrategy eviction, ScalingConfig scaling,
    BudgetConfig budget  // nullable — null = no budgeting
) {}

public record BudgetConfig(
    Double costLimit,              // USD ceiling, null = no cost limit
    Long tokenLimit,               // total token ceiling, null = no token limit
    Duration window,               // rolling window duration
    EnforcementPolicy enforcement,
    ReportInterval reportInterval,
    Duration noReportTimeout
) {}

public enum EnforcementPolicy {
    SUSPEND,    // suspend all active sessions + block new acquires
    BLOCK_NEW,  // let running sessions finish, block new acquires
    ALERT       // fire CDI event, scale to min-active, don't hard-stop
}

public sealed interface ReportInterval {
    record Turn() implements ReportInterval {}
    record Periodic(int turns) implements ReportInterval {}
    record Completion() implements ReportInterval {}
}
```

**YAML parsing:** `AgentPoolYamlParser` gains `parseBudget(Map<String, Object>)` following the `parseScaling()` pattern. `AgentPoolSchema` gains parameters for `budget.*` fields.

### 3. MCP Tool: `report_cost`

Added to the Claudony MCP endpoint at `/mcp` (alongside the existing 8 session tools).

```java
@Tool(description = "Report cost and token usage for the current session")
public record ReportCostRequest(
    String sessionId,          // tmux session identifier
    double totalCostUsd,       // cumulative cost for this session
    long inputTokens,          // cumulative input tokens
    long outputTokens,         // cumulative output tokens
    long cacheReadTokens,      // cumulative cache read tokens (informational)
    long cacheCreationTokens,  // cumulative cache creation tokens (informational)
    String model               // canonical model name (e.g. "claude-opus-4-6")
) {}
```

**Cumulative, not incremental.** Sessions report total-to-date. `BudgetTracker` computes deltas internally by comparing against the last known value. If a report is missed, the next one is still correct.

**System prompt injection.** `WorkerCommandBuilder` appends a cost-reporting instruction to `--append-system-prompt` based on the pool's `report-interval` config:

- `turn`: "After every turn, call report_cost with your cumulative session cost and token counts."
- `periodic(N)`: "Every N turns, call report_cost with your cumulative session cost and token counts."
- `completion`: "Before your session ends, call report_cost with your final session cost and token counts."

### 4. BudgetTracker — Core Domain Logic

`@ApplicationScoped` in `claudony-casehub`. Accumulates per-pool spend in a rolling window.

```java
@ApplicationScoped
public class BudgetTracker {

    private final ConcurrentHashMap<String, PoolBudgetState> states = new ConcurrentHashMap<>();

    // Record a cost report — computes delta, fires async CDI event
    public BudgetCheckResult record(String poolName, CostReport report);

    // Query current spend within the rolling window
    public PoolBudgetSpend currentSpend(String poolName);

    // Check if pool is over budget
    public BudgetCheckResult checkBudget(String poolName, BudgetConfig config);

    // Flush accumulated entries to DB (called by scheduler)
    public List<CostEntry> flush(String poolName);

    // Reset budget state (for runtime config changes).
    // Clears budgetExceeded flag and triggers immediate re-evaluation.
    // If the pool was budget-locked, this recovers it (restores maxActive).
    public void invalidate(String poolName);

    // Load persisted state on startup
    void bootstrap(Map<String, List<CostEntry>> persistedEntries);
}
```

**Internal records:**

```java
public record CostReport(
    String sessionId, double totalCostUsd,
    long inputTokens, long outputTokens,
    long cacheReadTokens, long cacheCreationTokens,
    String model, Instant timestamp
) {}

record SessionCostSnapshot(double lastCostUsd, long lastTokens, Instant lastReportTime) {}

record CostEntry(
    String poolName, String sessionId,
    double deltaCostUsd, long deltaTokens,
    String model, Instant timestamp
) {}

public record BudgetCheckResult(
    boolean overBudget,
    BudgetDimension violatedDimension,  // COST, TOKENS, or null
    double currentCostUsd, double costLimit,
    long currentTokens, long tokenLimit,
    Duration windowRemaining
) {}

public enum BudgetDimension { COST, TOKENS }
```

**Rolling window:** `PoolBudgetState` holds a time-ordered list of `CostEntry` records and a `ConcurrentHashMap<String, SessionCostSnapshot>` for per-session last-known state (delta computation + `lastReportTime` for no-report timeout checks). Entries older than `now - window` are evicted on each check. Current spend is the sum of remaining entries. Same sliding-window pattern as `AuthRateLimiter`.

**Persistence:**

- In-memory accumulation on each `record()` call
- Batch flush to PostgreSQL on scheduler tick
- On startup, `bootstrap()` loads entries from DB for the current window
- Daily `@Scheduled` cleanup of expired entries

**DB schema (Flyway migration):**

```sql
CREATE TABLE budget_cost_entry (
    id          BIGSERIAL PRIMARY KEY,
    pool_name   VARCHAR(255) NOT NULL,
    session_id  VARCHAR(255) NOT NULL,
    delta_cost  DOUBLE PRECISION NOT NULL,
    delta_tokens BIGINT NOT NULL,
    model       VARCHAR(255),
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_budget_pool_time ON budget_cost_entry(pool_name, recorded_at);
```

### 5. Enforcement Integration with ScalingScheduler

`ScalingScheduler` gains a budget evaluation step that runs **before** scaling and proactive evaluation. This ordering prevents proactive scaling from resuming sessions that budget enforcement just suspended.

```java
void evaluatePool(String poolName) {
    // 1. NEW: evaluate budget FIRST (gates all subsequent decisions)
    BudgetConfig budget = defRegistry.get(poolName).pool().budget();
    boolean budgetExceeded = false;
    if (budget != null) {
        BudgetCheckResult result = budgetTracker.checkBudget(poolName, budget);
        if (result.overBudget()) {
            budgetExceeded = true;
            budgetTracker.setBudgetExceeded(poolName, true);
            applyEnforcement(poolName, budget.enforcement(), result);
        } else if (budgetTracker.isBudgetExceeded(poolName)) {
            // Window rolled — recover
            budgetTracker.setBudgetExceeded(poolName, false);
            recoverFromBudget(poolName);
        }
        budgetTracker.flush(poolName);
        checkNoReportTimeouts(poolName, budget);
    }

    // 2. Existing: evaluate scaling/proactive — skip if budget exceeded
    if (!budgetExceeded) {
        ScalingDecision scalingDecision = evaluateScaling(poolName);
        // apply scaling decision, emit events
    }
}
```

**Budget-exceeded flag:** `BudgetTracker` maintains a `budgetExceeded` boolean per pool in `PoolBudgetState`. Both the fast-path observer and the scheduler check and set this flag, making enforcement idempotent. The scheduler skips scaling/proactive evaluation when the flag is set. `AgentSessionManager.acquireSession()` checks this flag (via a `BudgetTracker` injection or a callback) and rejects with `AgentPoolExhaustedException` when the pool is budget-locked — this is the primary enforcement mechanism, not `adjustMaxActive()`. The `adjustMaxActive()` call is best-effort (it's clamped to `minActive` by design), so a pool with `minActive=2` and `adjustMaxActive(0)` would silently keep capacity at 2 without the flag check. The flag check in `acquireSession()` is what actually blocks new sessions.

**Session-to-pool reverse index:** `AgentPoolManagerRegistry` maintains a `ConcurrentHashMap<String, String>` mapping sessionId → poolName, populated on `acquireSession()` and cleaned on `destroySession()`. The MCP `report_cost` tool uses this to resolve which pool a reporting session belongs to.

**Enforcement actions:**

| Policy | Action |
|--------|--------|
| `SUSPEND` | Suspend all active sessions first, THEN `adjustMaxActive(0)`. Order matters: `adjustMaxActive()` clamps to `activeCount()`, so suspending first ensures it can reach 0. |
| `BLOCK_NEW` | `adjustMaxActive(0)` — running sessions continue, no new acquires |
| `ALERT` | `adjustMaxActive(minActive)` — scale down to minimum |

All three emit a budget event via `PoolEventEmitter`.

**Shared enforcement method:** `BudgetEnforcer` (new package-private class in `claudony-casehub`) contains the `applyEnforcement()` logic. Both `BudgetEnforcementObserver` and `ScalingScheduler` delegate to it — no duplicated enforcement code.

**Fast-path CDI observer (immediate enforcement):**

```java
@ApplicationScoped
public class BudgetEnforcementObserver {
    @Inject BudgetTracker tracker;
    @Inject AgentPoolDefinitionRegistry defRegistry;
    @Inject BudgetEnforcer enforcer;

    void onCostReport(@ObservesAsync CostReportEvent event) {
        BudgetConfig config = defRegistry.get(event.poolName()).pool().budget();
        if (config == null) return;
        if (tracker.isBudgetExceeded(event.poolName())) return; // already enforced
        BudgetCheckResult result = tracker.checkBudget(event.poolName(), config);
        if (result.overBudget()) {
            tracker.setBudgetExceeded(event.poolName(), true);
            enforcer.apply(event.poolName(), config.enforcement(), result);
        }
    }
}
```

The `isBudgetExceeded()` check makes enforcement idempotent — if the scheduler already enforced, the observer skips. If the observer enforced first, the scheduler skips scaling.

**No-report timeout:** On each scheduler tick, for pools with `noReportTimeout`, check all active sessions via `BudgetTracker`'s per-session `SessionCostSnapshot.lastReportTime()`. If `now - lastReportTime > noReportTimeout`, fire a `NoReportTimeoutEvent` and apply the pool's enforcement policy. Sessions that have never reported use their `ManagedSession.lastInteraction` as the baseline.

**Recovery:** When `checkBudget()` returns `overBudget = false` (window rolled), the scheduler clears the `budgetExceeded` flag and restores `maxActive` to its configured value. Suspended sessions are NOT automatically resumed — proactive scaling (which now runs after budget check) or the operator handles that.

### 6. Runtime Mutation

`PoolUpdateRequest` gains budget fields (all optional):

```java
public record PoolUpdateRequest(
    // existing...
    Integer minActive, Integer maxActive,
    String scalingType, Double targetFillRatio, /* ... */
    // NEW
    Double costLimit,
    Long tokenLimit,
    String window,           // duration string
    String enforcement,      // "suspend" | "block-new" | "alert"
    String reportInterval,   // "turn" | "periodic(5)" | "completion"
    String noReportTimeout   // duration string
) {}
```

`PoolService.updatePool()` handles budget updates: parse fields → build `BudgetConfig` → `defRegistry.updateBudget()` → `budgetTracker.invalidate()`.

### 7. Observability

**Pool event types — budget events:**

```json
{"type": "budget", "event": "EXCEEDED", "poolName": "code-reviewer",
 "dimension": "COST", "current": 51.23, "limit": 50.00,
 "enforcement": "suspend", "timestamp": "..."}

{"type": "budget", "event": "RECOVERED", "poolName": "code-reviewer",
 "currentCost": 42.10, "costLimit": 50.00,
 "currentTokens": 8200000, "tokenLimit": 10000000, "timestamp": "..."}

{"type": "budget", "event": "COST_REPORT", "poolName": "code-reviewer",
 "sessionId": "...", "deltaCost": 0.37, "deltaTokens": 12450,
 "model": "claude-opus-4-6", "timestamp": "..."}

{"type": "budget", "event": "NO_REPORT_TIMEOUT", "poolName": "code-reviewer",
 "sessionId": "...", "silentMinutes": 10, "timestamp": "..."}
```

**Dashboard additions to `claudony-pool-panel.ts`:**

1. **Budget KPI cards** — Cost fill bar (`$42.10 / $50.00`), Token fill bar (`8.2M / 10M`), Window countdown, Status badge (`OK` | `EXCEEDED` | `NO_REPORT`)
2. **Budget section** — current config, per-session cost table, edit controls
3. **Event log** — budget events with `$` type badge, EXCEEDED in red

**Micrometer metrics:**

| Metric | Type | Description |
|--------|------|-------------|
| `claudony.pool.budget.cost_usd` | gauge | Current spend in window |
| `claudony.pool.budget.cost_limit_usd` | gauge | Configured limit |
| `claudony.pool.budget.tokens` | gauge | Current tokens in window |
| `claudony.pool.budget.token_limit` | gauge | Configured limit |
| `claudony.pool.budget.exceeded.total` | counter | Times budget exceeded |
| `claudony.pool.budget.reports.total` | counter | Cost reports received |
| `claudony.pool.budget.no_report_timeouts.total` | counter | No-report timeouts |

Registered by `PoolMetricsRegistrar` and pushed to IoTDB when enabled.

---

## Testing

### Unit tests (claudony-casehub)

| Test class | Covers |
|---|---|
| `BudgetTrackerTest` | Rolling window expiry, spend accumulation, delta from cumulative reports, concurrent reports, flush, invalidate, bootstrap |
| `BudgetConfigTest` | Parse from map, defaults, null = disabled, enforcement enum, report interval variants |
| `BudgetEnforcementObserverTest` | Fast-path: over-budget triggers enforcement, under-budget no-op, null config skips, COST vs TOKENS dimension, async decoupling from MCP thread |
| `BudgetEnforcerTest` | SUSPEND order (suspend before adjustMax), BLOCK_NEW, ALERT, idempotent re-enforcement |
| `AgentPoolYamlParserTest` (additions) | Budget parsing, budget-defaults merging, per-pool override, missing = null, partial override inherits |
| `AgentPoolManagerRegistryTest` (additions) | Reverse index: populated on acquire, cleaned on destroy, concurrent acquires, destroy-during-report race |

### Unit tests (claudony-app)

| Test class | Covers |
|---|---|
| `PoolServiceTest` (additions) | Budget update via PoolUpdateRequest, validation, budget in PoolDetailResponse |
| `PoolEventBusTest` (additions) | Budget event types |

### Integration tests (claudony-app)

| Test class | Covers |
|---|---|
| `BudgetIntegrationTest` | Full chain: MCP report_cost → record → CDI → fast-path → enforcement. PostgreSQL persistence round-trip. |
| `ClaudonyPoolApiTest` (additions) | Budget fields in updatePool and pool detail |
| `McpServerIntegrationTest` (additions) | report_cost tool at `/mcp` |

### Frontend tests

| Test | Type | Covers |
|---|---|---|
| vitest: budget KPI rendering | Unit | Fill bars, status badge, window countdown |
| vitest: budget editing | Unit | Limit inputs, enforcement dropdown, save |
| Playwright: budget display | E2E | Budget section renders, SSE updates, exceeded state |

**Estimated:** ~30-35 new Java tests, ~4 vitest, ~2 E2E assertions.

---

## Files Changed

### New files

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/BudgetConfig.java`
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/BudgetTracker.java`
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/CostReport.java`
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/CostEntry.java`
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/BudgetCheckResult.java`
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/BudgetEnforcementObserver.java`
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/BudgetEnforcer.java`
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/CostReportEvent.java`
- `app/src/main/java/io/casehub/claudony/agent/McpCostReportTool.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/BudgetPersistence.java`
- `app/src/main/resources/db/migration/qhorus/V*__budget_cost_entry.sql`
- Test files for all of the above

### Modified files

- `casehub/.../fleet/AgentPoolDefinition.java` — add `budget` to `PoolConfig`
- `casehub/.../fleet/AgentPoolYamlParser.java` — parse `budget:` and `budget-defaults:`
- `casehub/.../fleet/AgentPoolSchema.java` — add budget parameters
- `casehub/.../fleet/AgentPoolManagerRegistry.java` — add sessionId→poolName reverse index
- `casehub/.../fleet/ScalingScheduler.java` — budget evaluation + no-report timeout + flush
- `casehub/.../fleet/PoolEventEmitter.java` — budget event methods
- `casehub/.../WorkerCommandBuilder.java` — cost-reporting system prompt
- `app/.../fleet/PoolService.java` — budget in detail response + updatePool
- `app/.../fleet/PoolUpdateRequest.java` — budget fields
- `app/.../fleet/PoolDetail.java` — `BudgetView`
- `app/.../fleet/PoolResource.java` — delegate budget to PoolService
- `app/.../fleet/PoolMetricsRegistrar.java` — budget metrics
- `app/.../agent/McpServer.java` — register report_cost tool
- `app/src/main/webui/src/components/claudony-pool-panel.ts` — budget UI
- `CLAUDE.md` — test count, MCP tool list

---

## What This Does NOT Cover

- **Per-session budgets within a pool** — the pool is the budget boundary. Individual session limits can be added later as a finer-grained policy.
- **Anthropic API billing reconciliation** — MCP self-reporting is the data source. Cross-referencing with the Anthropic billing API for accuracy is a follow-on.
- **Cost prediction / forecasting** — the system tracks actuals, not projections. Predicting "this pool will exceed budget in 2 hours" is a follow-on.
- **Multi-currency / multi-provider cost normalization** — costs are in USD as reported by the Claude CLI. Non-Claude backends would need their own cost reporting.

---

## References

- Issue #205 — original pool management spec (deferred cost tracking)
- Issue #241/#242 — scaling API and dashboard (PoolService, PoolUpdateRequest, SSE patterns)
- Issue #235 — YAML alignment (AgentPoolSchema, StepValidator)
- `ScalingScheduler` — existing scheduler tick loop
- `PoolEventEmitter` — existing event emission
- `AgentPoolYamlParser` — existing YAML parsing
- `WorkerCommandBuilder` — system prompt construction
- `BudgetTracker` pattern — `AuthRateLimiter` sliding window
- `PoolMetricsRegistrar` — existing Micrometer registration
- Claude CLI `--output-format json` — `total_cost_usd`, `modelUsage` structure
- `document-workbench` (casehub dependency) — `WorkspaceProgressPayload.cost` upstream type
