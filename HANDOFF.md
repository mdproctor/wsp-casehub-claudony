# Session Handover — 2026-10-04

## What Happened

Completed epic #258 (model chain fallback end-to-end) — all three child issues landed as a single squashed commit `856f64c` on main.

**#259 — CliCircuitBreaker wired into provisioner:** Replaced static `CliChainResolver.resolve()` in `ClaudonyWorkerProvisioner.setupSession()` with `CliCircuitBreaker.tryChain()`. Sessions that exit within the grace period (API auth failure, model unavailable at runtime) now automatically retry the next model in the chain. Added `paneExitCode()` helper for tmux pane dead status inspection. Default grace period is 30 seconds; per-model overrides via pool YAML `grace-period` field.

**#260 — ModelFallbackEvent observer:** Created `ModelFallbackEventObserver` — `@ApplicationScoped` CDI bean that `@Observes ModelFallbackEvent`, logs at WARN level with pool name, requested/resolved models, and fallback depth. Thread-safe counter via `AtomicInteger` for ops dashboards.

**#261 — Pool pre-condition exception types:** Added `PoolAtCapacityException` (pool full, minActive prevents eviction) and `BudgetExceededException` (budgetLocked blocks acquisition) as subtypes of `AgentPoolExhaustedException`. Backward compatible — existing catch blocks still work.

## Key Decisions

- **Circuit breaker replaces static resolver for model chain path:** The provisioner no longer calls `CliChainResolver.resolve()` when a model chain is present. The circuit breaker combines model selection + session creation + retry in one call. The static resolver class remains available for other uses.

- **Package-private `circuitBreakerSleeper` field for test injection:** Rather than adding another constructor parameter (already 13), a package-private `Consumer<Duration>` field defaults to `Thread.sleep()` and is overridden to a no-op in tests. Tests also mock `tmux.sessionExists()` to return true so the alive check passes immediately.

- **Exception subtypes extend the base:** `PoolAtCapacityException` and `BudgetExceededException` extend `AgentPoolExhaustedException` rather than replacing it. Existing `catch (AgentPoolExhaustedException)` blocks continue to work.

## What's Next

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #262 | Clean up failed circuit breaker sessions on retry | S | Med |
| #263 | Fix 3 pre-existing FleetScript test failures | S | Low |

**#262** builds on #258 — the circuit breaker creates tmux sessions that may fail and retry, but failed sessions remain tracked as ACTIVE in `AgentSessionManager` with dead processes. Needs a cleanup mechanism (destroy callback or post-tryChain cleanup).

**#263** is independent — `FleetScriptParserTest.throwsOnEmptyNodes`, `throwsOnMissingVariable`, and `FleetScriptRunnerTest.emptyNodesThrows` expect `IllegalArgumentException` but get `UncheckedIOException`. Pre-existing on main since #247.

## Files Changed

- `ClaudonyWorkerProvisioner.java` — circuit breaker wiring, `paneExitCode()`, `fireFallbackEvent()` (renamed from `fireFallbackEventIfNeeded`)
- `AgentSessionManager.java` — `BudgetExceededException` and `PoolAtCapacityException` at throw sites
- `BudgetExceededException.java` — new
- `PoolAtCapacityException.java` — new
- `ModelFallbackEventObserver.java` — new
- `ModelFallbackEventObserverTest.java` — new (3 tests)
- `ClaudonyWorkerProvisionerTest.java` — setUp adds `sessionExists` mock, model chain tests add sleeper override
- `AgentSessionManagerTest.java` — 4 new tests for specific exception types
- `AgentSessionManagerWithTestPoolTest.java` — updated assertion to `PoolAtCapacityException`

## Test Status

All tests pass except 3 pre-existing FleetScript failures (tracked as #263).
