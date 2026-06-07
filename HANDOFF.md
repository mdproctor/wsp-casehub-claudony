# Handover — 2026-06-07

**Head commit (project):** `eee4575` — fix(casehub): exclude ResearcherCase from production arc
**Branch:** main (workspace + project)

---

## Last Session

Critical path item 2 complete (#148): `ResearcherCase extends YamlCaseHub` with `researcher.yaml` — first production case definition shipping with Claudony. Cases auto-complete when the tmux session exits via `drainExitSignal` → `CaseHubRuntime.signal("workers.researcher.exited", true)` → goal satisfaction. #143 (tenancyId assertions) and #147 (CasehubStartupService extraction + tests) also landed. Late-session fix: `ResearcherCase` must be in `quarkus.arc.exclude-types` in main `application.properties` — engine (`CaseHubRuntimeImpl`) is test-only dep; CDI augmentation fails otherwise. ADR-0010 + 2 protocols + 3 garden entries committed.

## Immediate Next Step

Open an issue for Critical Path item 3 then start a branch. Three sub-tasks:
- **3a** — `%dev.quarkus.arc.exclude-types` without `ResearcherCase` (so it registers in dev mode)
- **3b** — `POST /api/cases/researcher` REST endpoint → calls `researcherCase.startCase(body)`
- **3c** — `%dev.claudony.casehub.workers.commands.researcher=claude` in application.properties

Then validate: `mvn quarkus:dev -Dclaudony.mode=server` → trigger endpoint → see COMPLETED in dashboard.

---

## Critical Path to Real-World Examples

| # | Item | Status |
|---|------|--------|
| 1 | Exit watcher — detect when tmux session ends | ✅ Done (#146) |
| 2 | ResearcherCase — signal chain proven in tests | ✅ Done (#148) |
| 3 | Dev mode validation — run it live, not just in tests | → Next (~2-3h) |
| 4 | Production JAR — move `casehub-engine` to compile scope | Deferred (design needed) |

**Note on item 3:** `casehub-engine` is test-scope — it IS on the classpath in `mvn quarkus:dev` (dev mode includes test deps), so the engine works there. The remaining gaps are: `ResearcherCase` excluded from dev CDI (our production CDI fix), no REST trigger endpoint, and missing dev profile config.

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
