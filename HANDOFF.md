# Handover — 2026-06-01

**Head commit (project):** `1ff0f74` — fix(test): use CrossTenantCaseInstanceRepository in CaseEngineRoundTripTest
**Branch:** `issue-144-round-trip-cross-tenant-fix` (workspace + project)

---

## Last Session

Fixed `CaseEngineRoundTripTest` (#144): engine SNAPSHOT added tenancyId filtering to `findByUuid(UUID, String)` — cases stored with `tenancyId=null` weren't found when queried with `"default"`, causing a null `CaseInstance` to reach `WorkflowExecutionCompletedHandler`, which NPE'd silently and left lineage empty. Fix: switch to `CrossTenantCaseInstanceRepository.findByUuid(UUID)`. All 532 tests now pass. Previous note about `PlanExecutionContext` constructor change was a misdiagnosis — the engine SNAPSHOT is internally consistent.

## Immediate Next Step

`/work end` to close #144 → merge to main, then pick **Critical Path item 1** (WorkerExecutionManager watcher).

---

## Critical Path to Real-World Examples

Two blockers. Everything else is done or non-blocking.

**1. WorkerExecutionManager — monitor tmux exit → publish WorkflowExecutionCompleted** · M · Low
- `ClaudonyReactiveWorkerProvisioner` creates the tmux session but nothing watches for it to end.
- Engine drives workers via Quartz but needs `WorkflowExecutionCompleted` on the Vert.x event bus to complete the cycle. Tests fire this manually; production has no equivalent.
- Fix: virtual thread watcher in the provisioner — poll `tmux has-session` / inspect exit status, publish event on exit.
- Unlocks: the full engine round-trip in production (not just tests).

**2. Real CaseDefinition — at least one concrete case** · S · Low
- `TestResearcherCase` is test scaffolding only; not available at runtime.
- Need a `CaseDefinition` CDI bean in `claudony-casehub` or `claudony-app` — follow the TestResearcherCase pattern, wire a single researcher worker.
- Unlocks: calling `startCase()` with a real case type and watching it provision a Claude session end-to-end.

**Already working (not blocking):**
- Engine round-trip: verified today (532/532)
- Channel delivery: SSE cursor, Qhorus channels, HumanObserver backend
- Ledger capture: working; `causedByEntryId` scaffold in place (fully populates after engine#231)
- Auth, fleet, terminal streaming: done

---

## What's Left

- **#144** close + merge (current branch)
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **parent#117** — PLATFORM.md + claudony.md deep-dive: FleetMessageRelayObserver, test count 532, engine SNAPSHOT note · XS · Low
- **engine#231** — `triggerChannelId`/`triggerCorrelationId` through `ProvisionContext` — gates #140 causedByEntryId population · S · Low (engine work)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | WorkerExecutionManager tmux watcher | M | Low | **Critical path item 1** |
| — | Real CaseDefinition (researcher case) | S | Low | **Critical path item 2** |
| #143 | CaseLifecycleEvent observer tenancyId updates | S | Low | Remaining engine SNAPSHOT adaptation |
| #121 | Multi-tenancy foundation — tenancyId enforcement | XL | High | Long-horizon; don't start until critical path is done |

---

## Key references

- Diary: `blog/2026-05-31-mdp01-the-handler-that-never-fired.md`
- Garden: GE-20260531-929107 (EventSource named-event silent), GE-20260531-6298f4 (RESTEasy SSE Multi<String>), GE-20260531-18fa72 (package-private cross-jar)
- Test baseline: 4 core + 141 casehub + 387 app = **532 total, 532 passing** (2026-06-01, after #144)
