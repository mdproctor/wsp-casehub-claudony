# Handoff — #205 LLM Fleet Manager

**Branch:** `issue-205-llm-fleet-manager`
**Slot:** 202
**Date:** 2026-09-21
**Repos:** claudony (primary), platform, engine

## What Was Done

Design and planning session. No implementation code written yet.

### Brainstorming (7 decisions captured)

- **D1:** Unified fleet abstraction — manages both API connection pools and CLI agent sessions
- **D2:** Fleet manager lives in Claudony — integration layer, not engine
- **D3:** Extend existing platform stack (ModelRegistry, BackendInstanceRegistry, RoutingAgentProvider) with pool semantics — do NOT create a new ModelResolver SPI
- **D4:** Full scope across platform + engine + claudony (slot 202)
- **D5:** Capacity-bounded on-demand pool model (min/max instances, pre-warming, idle eviction)
- **D6:** Pool config extends the manifest YAML (`pools:` section)
- **D7:** CLI AgentBackend supports both invoke (one-shot) and openSession (interactive)

**Key discovery:** The platform already has a comprehensive model resolution and routing stack (`ModelDescriptor`, `ModelQuery`, `ModelRegistry`, `AgentBackend`, `BackendInstanceRegistry`, `RoutingAgentProvider`). The original D3 proposal (new ModelResolver SPI in engine-api) was revised after discovering this — fleet management adds pool lifecycle to the existing stack, not a parallel resolution system.

**Spec revision:** Another session revised the spec while this session was running. The revised spec (now titled "Agent Pool Management") simplifies the architecture: pool lifecycle is internal to `ClaudonyAgentBackend` rather than a platform SPI. The plan may need reconciliation with the revised spec.

### Implementation Plan (4 batches, 7 tasks)

| Batch | Tasks | Repo | What's working after |
|-------|-------|------|---------------------|
| 1 | PoolManager SPI + Manifest parsing | platform | Pool abstractions with no-op default |
| 2 | ClaudonyAgentBackend + FleetPoolManager | claudony | CLI sessions as platform backend, full pool lifecycle |
| 3 | REST endpoints + WorkerProvisioner wiring | claudony | Observable fleet, provisioner acquires from pool |
| 4 | AgentConverter convergence | engine | All YAML agents fleet-managed via routing stack |

## What's Next

1. **Reconcile plan with revised spec.** The spec was revised by another session — the plan references a `PoolManager` SPI in platform, but the revised spec makes pool lifecycle internal to `ClaudonyAgentBackend`. Read both and align.
2. **Begin Batch 1** — or Batch 2 if the revised spec's approach is adopted (no platform SPI needed).
3. **Engine convergence (Batch 4)** is highest risk — sequenced last.

## Artifacts

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md` |
| Decisions | `specs/issue-205-llm-fleet-manager/decisions.md` |
| Implementation plan | `plans/2026-09-21-llm-fleet-manager.md` |
| .plan (queue) | `.plan` |
| .slot | `/Users/mdproctor/claude/casehub/slots/202/.slot` |

## Context for Next Session

- Project and Eidos now have manifests for configuring LLMs and relationship orgs for agents — the fleet manager integrates with these, not duplicates them
- `AgentBackend` interface: `key()`, `instanceId()`, `invoke(AgentSessionConfig) → Multi<AgentEvent>`, `openSession(AgentSessionInit) → AgentSession`
- `BackendInstanceRegistry`: `register(AgentBackend)`, `resolve(key, instanceId)`, `resolveByKey(key)`
- `ClaudonyWorkerProvisioner` is the existing worker provisioning path — it will delegate to the pool
- Protocols to follow: PP-20260605-4b6c4e (createWorkerSession not createSession), PP-20260616-d32bc3 (reactive Panache event loop)
