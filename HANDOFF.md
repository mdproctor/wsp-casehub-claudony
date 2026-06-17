# Handover — 2026-06-17

**Head commit (project):** `ffaafbf` — docs: update CLAUDE.md (main)
**Branch:** main (project and workspace)

---

## Last Session

CI fix session following the branch close for issue #151 / #94.

**What shipped:**

CI was failing with 11 test errors across four root causes after the branch merge. All new failures fixed in two pushes:

1. **Worker.builder().function()** — engine SNAPSHOT removed the lambda constructor; three call sites updated
2. **QuartzRetryService CDI exclusion** — new engine SNAPSHOT bean with unsatisfied deps; added to all test profile exclude-types
3. **SystemPromptIntegrationTest / SilentProfileTest** — `@WithSession("qhorus")` fires from plain JUnit thread; fixed with `@RunOnVertxContext` + `UniAsserter`
4. **CaseEngineRoundTripTest — provision loop** — the deeper fix took two pushes:
   - First fix: `sessionExists()=true` before `startCase()`, fast poll (200ms); passed locally but failed in CI with "Expected size: 1 but was: 186"
   - 186 ledger entries revealed the signal() API mismatch: remote CI jar has `void signal()`, local has `CompletionStage<Void>`. catch(Throwable) silently swallowed the NoSuchMethodError, signal never sent, when-guard never cleared → 93 provision cycles
   - Fix: `CaseHubRuntimeCompat.signal()` — reflection-based shim handles both return types
   - Fix: Re-included `SignalReceivedEventHandler` in `CasehubEnabledProfile` (was excluded for engine#493); without it, `casehub.signal.received` has no Vert.x event bus handler → `NO_HANDLERS,-1`
   - Added `NoOpWorkerExecutionRecoveryService` `@DefaultBean` for SignalReceivedEventHandler's new dep
   - `.gitignore` anchored `io/` → `/io/` (was hiding source paths from git)
5. **Two new protocols:** PP-20260617-52285f (signal compat shim required) and PP-20260617-10cf10 (SignalReceivedEventHandler must stay in CasehubEnabledProfile)

**CI status:** Green on `ffaafbf`. 6 pre-existing `MeshResourceInterjectionTest` failures (Qhorus SNAPSHOT issue #155) + `ResearcherCaseCompletionTest` (engine#493) remain at baseline.

---

## Known Open Issues

| Issue | Repo | Status |
|-------|------|--------|
| **#155** | casehubio/claudony | MeshResourceInterjectionTest 6/9 failing — Qhorus SNAPSHOT `updateLastActivity` JPQL positional-param bug |
| **engine#493** | casehub-engine | SignalReceivedEventHandler doesn't fire CONTEXT_CHANGED after signal; ResearcherCaseCompletionTest blocked |
| **qhorus#280** | casehub-qhorus | MessageLedgerEntryTestFactory needs to move to qhorus-testing |
| **engine#500** | casehub-engine | triggerTenancyId missing from ProvisionContext |
| **parent#260** | casehubio/parent | Sync claudony deep-dive for QhorusCausalLinkResolver and engine compile scope |

---

## Next Session

Main branch is clean. CI at baseline. When Qhorus SNAPSHOT gets a new build that fixes `updateLastActivity`, run `MeshResourceInterjectionTest` and close #155 if green.
