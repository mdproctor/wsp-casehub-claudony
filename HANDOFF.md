# Session Handover — 2026-10-03

## What Happened

Designed and implemented **model fallback chains** (#212) — ordered model preferences with automatic degradation across three trigger levels (resolution, operational, runtime). Cross-repo: platform types + router resolution in `casehub-platform`, pool YAML parsing + CLI resolver + circuit breaker in `claudony`.

Branch landed on main (3 commits after squash). Issue closed. 7 follow-up issues filed (#251–#257).

## Key Decisions

- **Platform-owned resolution** — `RoutingAgentProvider.resolveChain()` owns the routing logic, not Claudony. Revised after decision review caught circular reasoning.
- **Cross-backend chains supported** — first-principles analysis showed no architectural blockers. Chain entries carry optional command overrides.
- **CDI event for degraded provisioning** — `ModelFallbackEvent` instead of `ProvisionResult` metadata (ProvisionResult has no metadata support).
- **Mutiny deferred() required** — `recoverWithMulti` eagerly evaluates the primary Multi during chain construction. Garden entry GE-20261003-189086.

## Blockers

- Platform commits (4) are in slot 202 local clone only — not pushed to casehubio/platform remote (#257). Spring module needs fixing first (8-arg AgentSessionConfig constructor).

## Next Action

**#251** — Wire CliChainResolver into ClaudonyWorkerProvisioner.setupSession(). All building blocks exist; the provisioner integration is the last mile (~30 min).

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-212-model-fallback-chains/2026-10-03-model-fallback-chains-design.md` |
| Decisions | `specs/issue-212-model-fallback-chains/decisions.md` |
| Plan | `plans/2026-10-03-model-fallback-chains.md` |
| Garden entry | GE-20261003-189086 (Mutiny recoverWithMulti gotcha) |
