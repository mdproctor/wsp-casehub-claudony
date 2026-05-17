# Handover — 2026-05-16

**Head commit (project):** `79ede97` — fix(test): review nits — workerId assertion, @DefaultBean on NoOpWorkloadProvider, #115 link  
**Branch:** `main` — 479 tests passing, zero failures

---

## What happened this session

**Epics closed:** #86 (agent mesh infrastructure — all child issues done)  
**Issues closed:** #92 (engine round-trip test), #112 (reactive=false workaround removal)  
**Issues filed:** #113 (CaseHub.startCase() IO-thread blocker), #114 (style nits), #115 (ClaudonyWorkerContextProvider.buildContext on IO thread)

**Core work — engine integration test:**
- Replaced CDI-event stub `CaseEngineRoundTripTest` with real engine pipeline test: `CONTEXT_CHANGED` → `CaseContextChangedEventHandler.tryProvision()` → `ClaudonyWorkerProvisioner.provision()` (TmuxService mocked) → `WorkflowExecutionCompleted` → `ClaudonyLedgerEventCapture` → `findCompletedWorkers()`
- Added `casehub-testing` + `casehub-engine` as test deps with CDI isolation
- New support beans: `TestResearcherCase` (@ApplicationScoped CaseHub subclass), `NoOpWorkloadProvider` (@DefaultBean)
- `CasehubEnabledProfile` overrides `quarkus.arc.exclude-types` to re-include engine beans — replaces (not extends) the `%test.` prefixed config
- Removed `@TestSecurity` from 3 CDI-only tests (PP-20260513-7c227e)

**Unblocked by upstream fixes:**
- engine#257 — `@DefaultBean` on no-op SPIs (CDI ambiguity resolved)
- qhorus#141 — A2A duplicate route fix (`ExcludedTypeBuildItem` does NOT gate JAX-RS; `@IfBuildProperty` required on resource class itself)

**Known limitation (#113):** `CaseHub.startCase()` not called — `CaseStartedEventHandler` calls `SchedulerService` which blocks on Quartz/JTA from Vert.x IO thread.

**Skill/workflow fixes:**
- `work-start` — hard gate enforced: session must stop (not "all clear") when both repos on main with no epic
- `prompt-snippets.md` in cc-praxis — epic/worktree requirement added explicitly
- `working-style.md` — corrected skill edit direction: cc-praxis first → sync-local

**Garden:** 4 entries — JAX-RS CDI exclusion gap, jar application.properties scan, abstract CDI bean indexing, QuarkusTestProfile config override replaces `%test.` entirely  
**Protocol:** `PP-20260514-engine-spi-noops-defaultbean` in parent repo

---

## Test count

**479 passing, 0 failures.** 4 core + 134 casehub + 341 app.

---

## Open issues

- **#113** — CaseHub.startCase() into the test (engine Quartz/JTA IO-thread fix needed)
- **#114** — style nits (UserTransaction helper, exclude-types ordering, timeouts)
- **#115** — ClaudonyWorkerContextProvider.buildContext() called from IO thread (production bug)
- **#99 epic** (#98, #100, #101, #102) — blocked on Qhorus #131
- **#94** — causal chain (blocked on engine)
- **#105** — MCP endpoint separation

*Full list: `gh issue list --repo casehubio/claudony --state open`*

---

## Immediate next

Pick up any of: #115 (production IO-thread bug, self-contained), or wait for upstream unblocking on #94/#99.

**Before starting:** invoke `work-start` — if no active epic, invoke `/epic begin` first. Never implement on main.

---

## Key references

- Spec: `specs/2026-05-15-casehub-startcase-roundtrip.md`
- Plan: `plans/2026-05-15-casehub-startcase-roundtrip.md`
- Blog: `blog/2026-05-16-mdp01-five-problems-before-assertion.md`
- Garden: `GE-20260516-4bf0dc`, `GE-20260516-8375d5`, `GE-20260516-2805b7`, `GE-20260516-e137f6`
