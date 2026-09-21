# HANDOFF — casehub-claudony

## Last Session

Implemented #213 and #214. `ClaudonyAgentBackend.openSession()` now returns a `TmuxAgentSession` wrapping a `ManagedSession` from `AgentSessionManager`. `close()` records interaction metrics (memory sampling) without destroying the session — the eviction queue manages lifecycle. `ClaudonyWorkerProvisioner` wired to create sessions through `ClaudonyAgentBackend.openWorkerSession()` → `AgentSessionManager` instead of direct `TmuxService` calls. All worker sessions now tracked by the pool. Terminate destroys via session manager when pool-provisioned, falls back to direct tmux kill for legacy sessions. Extended `SessionOperations` with command override for CLI-specific enriched commands. Added `sendRawKeys` to `TmuxService` for non-literal key sequences (C-c interrupt). 250 casehub tests + 16 core tests green.

## Immediate Next Step

#215 — conversation-id capture. `TmuxSessionOperations.conversationId()` currently returns from an empty map — need to extract the conversation ID from running Claude CLI sessions after launch and store it in `ManagedSession` for suspend/resume via `claude -c <id>`.

## Cross-Module

#217 — CDI deployment failures block app module integration tests. Engine SNAPSHOTs in .m2 are stale vs the exclude-types list. Not blocking casehub module unit tests (250 green).

## References

- `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md`
- `specs/issue-205-llm-fleet-manager/decisions.md` (D1–D11)
- `plans/2026-09-21-llm-fleet-manager.md`
- `blog/2026-09-21-mdp01-agents-arent-connections.md`
