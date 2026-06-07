# Handover — 2026-06-07

**Head commit (project):** `eee4575` — fix(casehub): exclude ResearcherCase from production arc
**Branch:** main (workspace + project)

---

## Last Session

Critical path item 2 complete (#148): `ResearcherCase extends YamlCaseHub` with `researcher.yaml` — first production case definition shipping with Claudony. Cases auto-complete when the tmux session exits via `drainExitSignal` → `CaseHubRuntime.signal("workers.researcher.exited", true)` → goal satisfaction. #143 (tenancyId assertions) and #147 (CasehubStartupService extraction + tests) also landed. Late-session fix: `ResearcherCase` must be in `quarkus.arc.exclude-types` in main `application.properties` — engine (`CaseHubRuntimeImpl`) is test-only dep; CDI augmentation fails otherwise. ADR-0010 + 2 protocols + 3 garden entries committed.

## Immediate Next Step

Run a real researcher case end-to-end in a running server:
1. Start server: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server -Dclaudony.casehub.enabled=true`
2. Trigger: `curl -X POST http://localhost:7777/api/cases/researcher -H 'Content-Type: application/json' -d '{"topic":"test"}'`
3. Watch case complete: `curl http://localhost:7777/api/sessions?caseId=<UUID>` shows status COMPLETED

This is Critical Path item 3 — real-world validation that the signal chain works outside tests.

---

## Critical Path to Real-World Examples

Item 1 ✅ exit watcher (#146, 2026-06-05).
Item 2 ✅ ResearcherCase + auto-completion (#148, 2026-06-07).
Item 3 → real server validation (no issue yet).

---

## What's Left

- **engine#231** — `triggerChannelId`/`triggerCorrelationId` through `ProvisionContext` (gates causedByEntryId) · S · Low
- **Unstamped branches** — ~10 completed branches missing `chore: branch closed` stamp · XS · Low
- **casehubio/parent#186** — claudony.md deep-dive sync (filed on parent repo, no action needed here) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Critical path item 3: real server validation | XS | Low | Start a researcher case in prod mode; confirm COMPLETED |
| — | Open issue for real CaseDefinition REST trigger endpoint (if API gap found in item 3) | S | Low | Only if item 3 reveals missing API |
| #143 | ✅ Closed | — | — | |
| #147 | ✅ Closed | — | — | |
| #121 | Multi-tenancy foundation — tenancyId enforcement | XL | High | Hold until critical path done |

---

## Key references

- Diary: `blog/2026-06-07-mdp01-the-signal-that-writes.md`
- ADR: `docs/adr/0010-case-auto-completion-via-exit-watcher-signal.md`
- Protocols: PP-20260607-b829e5 (exit signal ordering), PP-20260607-248d31 (workers.roleName.exited convention)
- Garden: GE-20260607-25a3fe (signal() as context patch), GE-20260607-e27c23 (DefaultWorkerExecutionRecoveryService dep), GE-20260607-609772 (CaseStatusChangedHandler exclusion)
- Test baseline: 4 core + 162 casehub + 407 app ≈ **573+ total, all passing** (2026-06-07, after #148 #143 #147)
- Backup branch: `backup/pre-squash-issue-148-researcher-case-20260607` (deletion: 2026-06-21)
