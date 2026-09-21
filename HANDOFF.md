# Handoff — #205 LLM Fleet Manager

**Branch:** `issue-205-llm-fleet-manager`
**Slot:** 202
**Date:** 2026-09-21
**Repos:** claudony (primary), platform, engine

## What Was Done

### Session 1 — Design and planning (prior session)

7 brainstorming decisions captured. Spec and plan written. See previous HANDOFF for details.

### Session 2 — Spec reconciliation + Batch 1–3 implementation

**Reconciled plan with spec.** The spec (revised by another session) made pool lifecycle internal to `ClaudonyAgentBackend` — no `PoolManager` platform SPI. The plan's Batch 1 (platform SPI) was dropped entirely. Reconciled sequence:

1. Deploy platform agent stack (pom.xml deps only)
2. ClaudonyAgentBackend with internal AgentPool
3. Observability REST endpoints
4. Engine convergence (deferred — highest risk)

**Implemented with TDD (17 new tests, 222 total casehub module green):**

| Commit | What |
|--------|------|
| `079d9d5` | AgentPool — capacity-bounded session pool (11 tests) |
| `ce601e2` | ClaudonyAgentBackend — CLI sessions as AgentBackend key "claudony" (6 tests) |
| `c26e378` | Deploy agent-router, agent-gate, agent-langchain4j deps + @HandWrittenEndpoint fixes |
| `53d6ba5` | AgentPoolResource — GET /api/agent-pools observability endpoint |

**Key design decisions:**
- `AgentPool` is a standalone class (not CDI) for pure unit testability — injected into `ClaudonyAgentBackend`
- Pool is "warm start" — sessions are destroyed on release and replaced (not reused)
- Config via constructor for now; will move to `@ConfigMapping` when wired to Quarkus
- `ClaudonyAgentBackend` `invoke()` and `openSession()` are stubs — pool session factory not yet wired to `TmuxService`

### Blocker: Pre-existing CDI deployment failures in app module

The app module `@QuarkusTest` tests fail with 83 CDI deployment errors — **pre-existing, not caused by agent deps**. Confirmed by:
1. No agent-specific CDI errors (BackendInstanceCoordinator, RouterBeans etc. resolved fine)
2. The branch already had compile errors from #204 `@McpDomain` annotation processor (fixed with `@HandWrittenEndpoint`)
3. The engine SNAPSHOTs in .m2 have evolved past what this slot's `quarkus.arc.exclude-types` covers

**Impact:** Cannot run integration tests that verify RoutingAgentProvider displaces NoOpAgentProvider, or that ClaudonyAgentBackend is auto-registered in BackendInstanceRegistry. Unit tests (222 green) validate the core logic.

**Fix needed:** Refresh engine SNAPSHOTs by installing compatible versions from the engine repo, then update the `quarkus.arc.exclude-types` list in `app/src/test/resources/application.properties`. This is a slot infrastructure task, not a #205 task.

## What's Next

1. **Wire pool session factory to TmuxService** — `AgentPool` currently uses a stub factory. Need to connect it to `TmuxService.createWorkerSession()`.
2. **Implement `ClaudonyAgentBackend.openSession()`** — acquire from pool, return an `AgentSession` that releases on close.
3. **Wire `ClaudonyWorkerProvisioner` to `AgentProvider`** — replace direct `TmuxService.createWorkerSession()` calls with `agentProvider.openSession()`.
4. **Fix CDI deployment issue** — refresh engine SNAPSHOTs, update exclude-types. Blocks integration tests.
5. **Engine convergence (Batch 4)** — `AgentConverter` onto `RoutingAgentProvider`. Highest risk, sequenced last.

## Artifacts

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md` |
| Decisions | `specs/issue-205-llm-fleet-manager/decisions.md` |
| Implementation plan | `plans/2026-09-21-llm-fleet-manager.md` |
| .plan (queue) | `.plan` |
| .slot | `/Users/mdproctor/claude/casehub/slots/202/.slot` |

## Context for Next Session

- Platform agent stack: `AgentProvider` → `RoutingAgentProvider` → `BackendInstanceRegistry` → `AgentBackend` (by key)
- `BackendInstanceCoordinator` auto-registers all CDI `AgentBackend` beans on startup
- `ClaudonyAgentBackend` key is "claudony", instanceId is "default"
- `AgentPool` API: `preWarm()`, `acquire()→String sessionId`, `release(sessionId)`, `shutdown()`, `status()→AgentPoolStatus`
- Protocols: PP-20260605-4b6c4e (createWorkerSession), PP-20260616-d32bc3 (reactive Panache)
- `NoOpModelRegistry @DefaultBean` provides empty model registry — sufficient until manifest config is wired
- `RoutingAgentConfig` defaults `casehub.platform.agent.default-backend=claude` — override to "claudony" for CLI-first routing
