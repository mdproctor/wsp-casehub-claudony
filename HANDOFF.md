# Handover — 2026-06-16

**Head commit (project):** `4f2ba73` — chore: pre-push hook classification-based (main)
**Branch:** main (project and workspace)

---

## Last Session

Three issues landed (#151, #153, #94) plus a pre-push hook improvement to cc-praxis.

**#151 — Engine compile scope / production case orchestration:** Closed. Engine moved to compile scope. `ClaudonyWorkerExecutionManager.startWatcherForSession()` added — provisions workers and starts the watcher. `@WithSession("qhorus")` added to `listChannels()` (required by engine event loop). CDI exclude-types aligned (`WorkerScheduleEventHandler` removed from exclusions to unblock the watcher chain). SNAPSHOT API changes absorbed (ChannelCreateRequest, terminate(String,String), Worker sealed interface, DispatchResult 13-field). Live JAR test confirmed the provision chain fires: CaseStarted → WorkerStarted → WorkflowExecutionCompleted → signal. COMPLETED blocked by engine#493 (see below).

**#153 — ResearcherCaseStartupTest:** Closed. Bypass CDI ObjectMapper by loading YAML via `CaseDefinitionYamlMapper.load(InputStream)` — no CDI context required in the plain JUnit path.

**#94 — causedByEntryId at provisioning time:** Closed. `QhorusCausalLinkResolver` added — `@ApplicationScoped` bean with `@WithSession("qhorus")`, resolves Qhorus `MessageLedgerEntry` UUID from `triggerChannelId`/`triggerCorrelationId`. Wired via `Uni.combine()` parallel chain in `provision()`: blocking setup on worker pool + reactive DB lookup on event loop run simultaneously. Critical constraint: `@WithSession` must be intercepted on the event loop, not after `runSubscriptionOn(workerPool)` — the Uni is constructed on the event loop, capturing the right context. `.emitOn(workerPool)` added before `startWatcher()` to prevent event-loop deadlock (watcher calls `.await()`).

Two new protocols captured:
- `PP-20260616-fc862e` — causalContext ConcurrentHashMap is the permanent side-channel; CaseLifecycleEvent must never carry causedByEntryId (engine#389 design constraint)
- `PP-20260616-d32bc3` — @WithSession calls must be pre-constructed on event loop, not chained after runSubscriptionOn(workerPool)

**Pre-push hook:** cc-praxis git-squash hook updated — removed `^fix\(ci\):` from coarse filter (too broad, fires on substantial SNAPSHOT API fixes). Hook now passes silently when all commits are KEEP-worthy. claudony `.githooks/pre-push` updated from "always block" to classification-based. Synced and committed.

---

## Blocked / Open Upstream

| Issue | Repo | What it blocks |
|-------|------|----------------|
| **engine#493** | casehub-engine | `SignalReceivedEventHandler` does not fire `CaseContextChangedEvent` after signal — ResearcherCase stays at RUNNING after exited signal. `ResearcherCaseCompletionTest` is a known failing test. |
| **qhorus#280** | casehub-qhorus | `MessageLedgerEntryTestFactory` needs to move to `casehub-qhorus-testing` — then `QhorusCausalLinkResolverTest` can add a real Panache integration test instead of the hand-written stub. |
| **engine#500** | casehub-engine | `triggerTenancyId` missing from `ProvisionContext` — multi-tenant accuracy for causal link resolution. |
| **parent#260** | casehub-parent | Sync claudony deep-dive for `QhorusCausalLinkResolver` and engine compile scope change. |

---

## Test Baseline

**586 total, 584 passing.** Known failing:
- `MeshResourceInterjectionTest.postMessage_eventType_isValid` — Qhorus SNAPSHOT: EVENT dispatch now requires gateway registration (pre-existing)
- `ResearcherCaseCompletionTest` — blocked by engine#493

---

## Next Session

Main branch is clean. Next priorities:
1. Watch for engine#493 fix — once landed, `ResearcherCaseCompletionTest` should pass
2. Watch for qhorus#280 — then promote the QhorusCausalLinkResolver integration test
3. Consider closing #154 (TestCompletionCase investigation is parked — root cause is engine#493, not Claudony)
