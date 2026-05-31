# Handover — 2026-05-31

**Head commit (project):** `2f0dbf0` — docs: update test baseline 525→532, Qhorus tool count 58→59, engine SNAPSHOT tenancyId notes
**Branch:** `main` — #140 and #125 closed and merged

---

## Last Session

Closed branch `issue-140-125-causal-sse`:
- **#140** — `causedByEntryId` wiring scaffold: `ConcurrentHashMap` bridge in `ClaudonyReactiveWorkerProvisioner` + drain in `ClaudonyLedgerEventCapture` on `WorkerStarted`. Data flows once engine#231 ships `triggerChannelId`/`triggerCorrelationId` through `ProvisionContext`.
- **#125** — SSE reconnect cursor: `?after=<id>` + `_eventId` embedded in `/api/mesh/events` JSON payload; `SseMeshStrategy` fixed (silent `addEventListener('mesh-update')` bug — server sends unnamed events), reconnects with cursor on error, content-equality check prevents flicker.
- Adapted to engine SNAPSHOT: `CaseLifecycleEvent` 8-field (`tenancyId`), `CaseLedgerEntry.tenancyId` non-null, `findByUuid(UUID, tenancyId)`. `CaseEngineRoundTripTest` still fails — engine has an in-flight `PlanExecutionContext` change not yet published.

## Immediate Next Step

`/work` — both repos on `main`, pause stack empty. Engine SNAPSHOT needs stabilising before `CaseEngineRoundTripTest` goes green; pick from What's Left.

---

## What's Left

- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **parent#117** — PLATFORM.md + claudony.md deep-dive: FleetMessageRelayObserver, test count 532, engine SNAPSHOT note · XS · Low
- **engine#231** — thread `triggerChannelId`/`triggerCorrelationId` through `ProvisionContext` (engine work, not Claudony) — gates #140 completing causedByEntryId population · S · Low (once engine ships)
- **engine SNAPSHOT stabilisation** — `PlanExecutionContext` constructor change blocking `CaseEngineRoundTripTest`; fix lands when engine publishes consistent SNAPSHOT · XS · Low (engine work)

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Garden: GE-20260531-929107 (EventSource named-event silent), GE-20260531-6298f4 (RESTEasy SSE Multi<String> wrapping), GE-20260531-18fa72 (package-private cross-jar)
- Diary: `blog/2026-05-31-mdp01-the-handler-that-never-fired.md`
- Test baseline: 4 core + 141 casehub + 387 app = 532 total, 531 passing (2026-05-31, after #140/#125)
- Backup branch: `backup/pre-squash-main-20260531` (deletion: 2026-06-14)
