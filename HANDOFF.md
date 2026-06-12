# Handover — 2026-06-13

**Head commit (project):** `d415f2e` — chore: add session permissions (main)
**Branch:** main (project), issue-151-engine-compile-scope (workspace)

---

## Last Session

Two issues landed (#149, #152) plus a blocked start on #151 (engine compile scope).

**#149 — Dev mode validation:** Landed. ResearcherCase registered in CDI, REST endpoint created (`POST /api/casehub/cases/researcher`), dev profile config added. Server starts and endpoint returns 202. Full COMPLETED validation blocked on engine CDI cascade issues (multiple workers provisioned, SNAPSHOT instability).

**#151 — Engine compile scope:** Started but BLOCKED. Engine moved to compile scope, in-memory persistence added. Server CDI deploys but case completion fails — root cause: SNAPSHOT ecosystem drift (tokenise API, em.persist blockade, MockCurrentPrincipal null tenancyId). Left on `issue-151-engine-compile-scope` branch, not closed.

**#152 — tenancyId SNAPSHOT alignment:** Landed and closed. Fixed CI across 20 files: LedgerEntryRepository migration, tokenise(String,ActorType) updates in Qhorus, engine CDI exclude-types, ledger_subject_sequence SQL script. 2 test failures remain (#153, #154).

## Immediate Next Step

Wait for SNAPSHOT ecosystem to stabilize across casehub repos (ledger, platform, qhorus, engine must all be at a consistent commit). Then resume issue-151 (`git checkout issue-151-engine-compile-scope` on project), run `mvn test`, fix remaining CDI issues.

---

## Critical Path to Real-World Examples

| # | Item | Status |
|---|------|--------|
| 1 | Exit watcher | ✅ Done (#146) |
| 2 | ResearcherCase signal chain | ✅ Done (#148) |
| 3 | Dev mode live validation | ⚡ Partial (#149) — endpoint works, COMPLETED blocked by SNAPSHOT |
| 4 | Production JAR — engine to compile scope | ⚡ In-progress (#151) — BLOCKED on SNAPSHOT stability |

---

## What's Left

- **#153** — `ResearcherCaseStartupTest` NPE: `YamlCaseHub.getDefinition()` needs CDI ObjectMapper (SNAPSHOT broke plain JUnit) · S · Low
- **#154** — `ResearcherCaseCompletionTest` timeout: multiple workers created by `signalStarted()` timing race in test context · S · Med
- **#150** — Rename `ResearcherCase` + `researcher` capability to neutral names · S · Low
- **engine#231** — `triggerChannelId`/`triggerCorrelationId` through `ProvisionContext` · S · Low
- **casehubio/parent#186** — claudony.md deep-dive sync · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 151 | Engine compile scope — resume when SNAPSHOT stable | L | High | Blocked on casehub ecosystem SNAPSHOT consistency |
| 150 | Rename ResearcherCase/researcher to neutral names | S | Low | |
| 153 | Fix ResearcherCaseStartupTest — CDI ObjectMapper | S | Low | SNAPSHOT breaking change |
| 154 | Fix ResearcherCaseCompletionTest — signalStarted timing | S | Med | Test timing race in mocked context |
| 121 | Multi-tenancy foundation | XL | High | Hold until critical path done |

---

## Key references

- Diary: `blog/2026-06-12-mdp01-when-the-ledger-enforces-its-own-rules.md`
- Protocols: PP-20260612-d6e7ec (CDI exclude-types sync), PP-20260608-365c92 (binding when-guard)
- Garden (session): GE-20260612-17c161 (em.persist blocked), GE-20260612-8e925b (sql-load-script named PU), GE-20260612-b20b51 (YamlCaseHub CDI ObjectMapper)
- Test baseline: 6 core + 162 casehub + 408 app = **576 total; 3 failures** (MeshResourceInterjectionTest pre-existing, #153, #154)
- Backup: `backup/pre-squash-issue-152-tenancyid-default-fix-20260612` (delete after 2026-06-26)
- Issue-151 project branch: `issue-151-engine-compile-scope` (open, BLOCKED)
