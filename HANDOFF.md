# HANDOFF — casehub-claudony

## Last Session

Design evolved from connection-pool to suspend/resume session lifecycle for CLI agents (D8–D11). Implemented AgentSessionManager with memory-weighted eviction, TmuxSessionOperations bridge, and ClaudonyAgentBackend as platform AgentBackend key "claudony". Platform routing stack (agent-router, agent-gate, agent-langchain4j) deployed. 7 follow-on issues filed (#213–#219), all in .plan.

## Immediate Next Step

#213 — implement `ClaudonyAgentBackend.openSession()` with pool acquire and conversation-id capture.

## Cross-Module

#217 — CDI deployment failures block app module integration tests. Engine SNAPSHOTs in .m2 are stale vs the exclude-types list. Not blocking casehub module unit tests (234 green).

## References

- `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md`
- `specs/issue-205-llm-fleet-manager/decisions.md` (D1–D11)
- `plans/2026-09-21-llm-fleet-manager.md`
- `blog/2026-09-21-mdp01-agents-arent-connections.md`
