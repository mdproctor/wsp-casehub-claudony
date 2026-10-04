# Session Handover — 2026-10-04

## What Happened

Wired model chain resolution into `ClaudonyWorkerProvisioner.setupSession()` (#251) — the final integration step connecting the building blocks from #212 to the provisioning flow.

`setupSession()` now looks up `AgentPoolDefinition` by role name via `AgentPoolDefinitionRegistry`, resolves the model chain through `CliChainResolver.resolve()`, and uses the result to override both the base command and the model config (`withModel()`). `ModelFallbackEvent` fires via CDI when chain resolution falls back from the primary entry.

Added `CLI_PASS_THROUGH` ModelRegistry to `CliChainResolver` — trusts YAML-configured model names since the CLI validates them at runtime. Queried entries return empty (need a real registry).

Branch landed as `f207024` on main. Issue #251 closed.

Created epic #258 with three follow-up issues for completing the fallback story end-to-end.

## Key Decisions

- **CLI_PASS_THROUGH over NoOpModelRegistry:** The platform's `NoOpModelRegistry` (`@DefaultBean`) rejects all Named entries, which would break CLI chain resolution. Rather than fighting CDI bean priority, `CLI_PASS_THROUGH` is a static constant on `CliChainResolver` — the provisioner uses it directly. Clean, no CDI conflicts.

- **Fallback event currently unreachable via provisioner:** With `CLI_PASS_THROUGH`, all Named entries pass validation, so the first entry always wins and `ModelFallbackEvent` never fires through the provisioner. This is by design — the event path activates when `CliCircuitBreaker` is wired in (#259) for runtime fallback.

- **No separate capacity pre-check:** Chain resolution is lightweight (no session creation). Capacity is already enforced inside `AgentSessionManager.acquire()`. A separate pre-check would give clearer error messages but is deferred to #261.

## What's Next

**Epic #258 — complete model chain fallback end-to-end:**

| # | Title | Scale | Priority |
|---|-------|-------|----------|
| #259 | Wire CliCircuitBreaker into provisioner | M | Must-do |
| #260 | Add ModelFallbackEvent observer | S | Should-do |
| #261 | Pool pre-condition exception types | S | Nice-to-have |

#259 is the critical one — it completes the runtime retry path so sessions that fail early (API errors, auth failures) automatically retry with the next chain entry.

## Files Changed

- `ClaudonyWorkerProvisioner.java` — chain resolution wiring, fallback event firing
- `CliChainResolver.java` — `CLI_PASS_THROUGH` ModelRegistry constant
- `ClaudonyWorkerProvisionerTest.java` — 5 new tests, constructor updates
- `WorkerLifecycleSequenceTest.java` — constructor update

## Test Status

All 31 provisioner tests pass. 3 pre-existing failures in `FleetScriptParserTest`/`FleetScriptRunnerTest` (from #247, unrelated).
