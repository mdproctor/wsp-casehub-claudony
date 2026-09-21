# Handoff — #205 LLM Fleet Manager

**Branch:** `issue-205-llm-fleet-manager`
**Slot:** 202
**Date:** 2026-09-21
**Repos:** claudony (primary), platform, engine

## What Was Done

### Session 1 — Design and planning (prior session)

7 brainstorming decisions captured. Spec and plan written.

### Session 2 — Spec reconciliation, design evolution, implementation

**Reconciled plan with spec.** Dropped platform `PoolManager` SPI (pool lifecycle internal to claudony). Then design evolved further during session: connection-pool model replaced with suspend/resume lifecycle based on discussion about agent identity, conversation continuity, and memory-weighted eviction.

**Decisions D8–D11 captured:**
- D8: Suspend/resume lifecycle (not destroy/create pool)
- D9: Memory-weighted additive eviction scoring
- D10: Identity-correlated instances with conversation continuity
- D11: Shared file coordination (open design question)

**Implementation (TDD, 234 total casehub tests green):**

| Commit | What |
|--------|------|
| `079d9d5` | AgentPool — initial pool (later superseded) |
| `ce601e2` | ClaudonyAgentBackend — key "claudony" |
| `c26e378` | Deploy agent-router, agent-gate, agent-langchain4j + @HandWrittenEndpoint fixes |
| `53d6ba5` | AgentPoolResource — GET /api/agent-pools |
| `da2c642` | AgentSessionManager — suspend/resume with memory-weighted eviction (13 tests) |
| `14065f5` | Refactor: wire backend to session manager, remove old AgentPool |
| `27e2e94` | TmuxSessionOperations — real tmux bridge (10 tests) |
| `a677e67` | Wire ClaudonyAgentBackend to real TmuxSessionOperations |

**Architecture:**

```
ClaudonyAgentBackend (AgentBackend key="claudony")
  → AgentSessionManager (suspend/resume lifecycle, eviction queue)
    → TmuxSessionOperations (implements SessionOperations SPI)
      → TmuxService (tmux create/kill/displayMessage)
```

**Session states:** ACTIVE (tmux live) ↔ SUSPENDED (tmux killed, conversation-id on disk)
**Eviction:** `idleSeconds + (memoryMB / 10)` — additive, memory sampled post-interaction
**Capacity:** `maxActive` ceiling, `minActive` eviction-immune floor

### Blocker: Pre-existing CDI deployment failures in app module

App module `@QuarkusTest` tests fail with 83 CDI errors from stale engine SNAPSHOTs. Pre-existing, not caused by agent deps. See previous HANDOFF for details. Blocks integration tests only; 234 unit tests green.

## What's Next

1. **`ClaudonyAgentBackend.openSession()`** — acquire from session manager, return an `AgentSession` that records interaction on close
2. **Wire `ClaudonyWorkerProvisioner` to `AgentProvider`** — replace direct `TmuxService.createWorkerSession()` with `agentProvider.openSession()`
3. **Conversation-id capture** — after `claude` starts, read its conversation-id from the session and store it for resume
4. **D11: Shared file coordination** — design for multi-agent shared workspaces (debates, parallel review)
5. **Fix CDI deployment issue** — refresh engine SNAPSHOTs, update exclude-types
6. **Engine convergence (Batch 4)** — `AgentConverter` onto `RoutingAgentProvider`

## Artifacts

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md` |
| Decisions (D1–D11) | `specs/issue-205-llm-fleet-manager/decisions.md` |
| Implementation plan | `plans/2026-09-21-llm-fleet-manager.md` |
| .plan | `.plan` |
| .slot | `/Users/mdproctor/claude/casehub/slots/202/.slot` |

## Context for Next Session

- Platform agent stack deployed: `RoutingAgentProvider` displaces `NoOpAgentProvider` via CDI
- `BackendInstanceCoordinator` auto-registers `ClaudonyAgentBackend` at startup
- `AgentSessionManager` tracks `ManagedSession` instances (ACTIVE/SUSPENDED)
- `TmuxSessionOperations` bridges to `TmuxService` for real tmux lifecycle
- Resume uses `claude -c <conversationId>` — conversation-id must be captured after session creation
- Eviction score: `idleSeconds + (memoryMB / 10.0)`, memory via `ps -o rss= -p <pane_pid>`
- Config defaults: `minActive=0`, `maxActive=10` — TODO: move to `@ConfigMapping`
- Protocols: PP-20260605-4b6c4e (createWorkerSession), PP-20260616-d32bc3 (reactive Panache)
