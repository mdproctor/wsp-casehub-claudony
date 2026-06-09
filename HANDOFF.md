# Handover — 2026-06-09

**Head commit (project):** `806f22e` — fix(claudony#152): fail fast on null tenancyId; fix LedgerEntry parent field shadowing
**Branch:** `issue-152-tenancyid-default-fix` (project), main (workspace)

---

## Last Session (2026-06-09, platform session cross-repo)

Fixed claudony#152: `ClaudonyLedgerEventCapture` was silently falling back to `"default"` when `CaseLifecycleEvent.tenancyId()` was null — protocol violation PP-20260520-439daf. Fix: fail fast with error log, drop the event. Also discovered and fixed: `CaseLedgerEntry.tenancyId` shadows `LedgerEntry.tenancyId`; production code now sets both fields explicitly (child field + parent via cast). Filed engine#460 for the root cause (duplicate field declaration in `CaseLedgerEntry`).

Tests: updated all events in `ClaudonyLedgerEventCaptureTest` to supply `"tenant-1"`; added `nullTenancyId_eventDropped_noLedgerEntryWritten`. Test failures remain due to pre-existing `PropertyValueException` from the engine#460 field shadowing issue — those failures pre-date claudony#152 and are not caused by it.

Prerequisite for claudony#121 (full multi-tenancy foundation).

## Immediate Next Step

Merge `issue-152-tenancyid-default-fix` branch (claudony#152). Then start Critical Path item 3 for the ResearcherCase dev-mode validation.

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
