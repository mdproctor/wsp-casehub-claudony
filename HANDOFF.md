# Handover — 2026-05-26

**Head commit (project):** `435805e` — chore(#129): replace ChannelView/InstanceView with canonical api types
**Branch:** `issue-113-start-case-entry-point` — open, needs PR

---

## Last Session

*(Previous 2026-05-25: S/XS batch — #120 openChannel race, #127 cursor accumulation, #114 test cleanup.)*

**Parent session (2026-05-26) made two commits on `issue-113-start-case-entry-point`:**

- **#113** (commit `edbc82e`): `CaseEngineRoundTripTest` now calls `researcherCase.startCase()` as the true engine entry point. `CaseStartedEventHandler` and `SchedulerService` un-excluded from test profile; `quarkus.quartz.store-type=ram` added. Requires engine#367 (`blocking=true`) to be merged and published before the test fully runs.
- **#129** (commit `435805e`): `MeshResource` uses `ChannelDetail` and `InstanceInfo` from `casehub-qhorus-api` instead of `QhorusDashboardService.ChannelView`/`InstanceView`. Explicit `casehub-qhorus-api` dep added to `app/pom.xml`. Coordinated with qhorus branch `issue-201-canonical-dashboard-types` (qhorus#201 — also closed).

Both issues closed. IntelliJ confirms zero errors.

## Immediate Next Step

**PR branch `issue-113-start-case-entry-point`** — 3 files changed across 2 commits. Test (#113) will fully pass once engine#367 is merged and published.

---

## What's Left

- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#131** — ChannelEventBus-driven true push (replace 500ms tick) · M · Med
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- Open branch: `issue-113-start-case-entry-point` (project repo — needs PR)
- Coordinated: qhorus `issue-201-canonical-dashboard-types` (qhorus#201)
- Engine dependency: `issue-367-350-332-engine-xs-batch` must merge first for #113 test to pass
- Garden: GE-20260524-c1e573 (Quarkus exclude-types silently fails for extension beans)
- Blog: `2026-05-24-mdp01-dependency-that-didnt-exist.md`
- Test baseline: 4 core + 135 casehub + 371 app = 510 (2026-05-25)
