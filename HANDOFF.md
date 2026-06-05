# Handover — 2026-06-05

**Head commit (project):** `ee1a45e` — docs(protocols): establish claudony CaseHub worker lifecycle protocols
**Branch:** main (workspace + project)

---

## Last Session

Full implementation of #146 — WorkerExecutionManager tmux exit watcher (critical path item 1). Key design discoveries: shell-based `createSession()` keeps the session alive after the worker exits (`tmux has-session` returns 0 forever); `createWorkerSession()` with `-- sh -c <command>` fixes this. The publish gate is `registry.remove(sessionId) != null` — whichever caller wins this atomic ConcurrentHashMap remove publishes. `terminate()` removes from registry before killing tmux. Recovery on server restart: `@casehub_case_id`/`@casehub_role` tmux session options persisted at provision time; `bootstrapCasehubWatchers()` reads them on startup. 560/560 tests. 7 commits squashed to 3, landed on main.

## Immediate Next Step

Start **Critical Path item 2**: Real CaseDefinition (researcher case). Create a real YAML or Java DSL case definition that exercises the full provision→watch→complete round-trip in production (not `TestResearcherCase` test stub). Open an issue first.

---

## Critical Path to Real-World Examples

*Unchanged — `git show 8889362:HANDOFF.md`*

Item 1 ✅ done (#146, landed 2026-06-05).

---

## What's Left

- **engine#231** — `triggerChannelId`/`triggerCorrelationId` through `ProvisionContext` (gates causedByEntryId) · S · Low
- **#147** — `bootstrapCasehubWatchers()` orchestration test coverage (UUID parse guard, null instance, roleName fallback) · S · Med
- **Unstamped branches** — ~10 completed branches missing `chore: branch closed` stamp · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Real CaseDefinition (researcher case) | S | Low | **Critical path item 2** |
| #143 | CaseLifecycleEvent observer tenancyId updates | S | Low | Remaining engine SNAPSHOT adaptation |
| #121 | Multi-tenancy foundation — tenancyId enforcement | XL | High | Long-horizon; hold until critical path done |

---

## Key references

- Diary: `blog/2026-06-05-mdp01-the-shell-that-outlived-the-worker.md`
- Garden: GE-20260605-e42cc5 (@Blocking on CDI startup observer breaks augmentation), GE-20260605-385d36 (@InjectMock cannot mock @Singleton-scoped beans)
- Protocols: PP-20260605-06af22 (terminate() ordering), PP-20260605-4b6c4e (createWorkerSession() for CaseHub workers)
- Test baseline: 4 core + 170 casehub + 386 app = **560 total, all passing** (2026-06-05, after #146)
- Backup branch: `backup/pre-squash-issue-146-tmux-exit-watcher-20260605` (deletion: 2026-06-19)
