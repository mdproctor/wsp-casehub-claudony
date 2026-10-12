# Eidos Agent Identity Integration for Agent Pools — Design Spec

**Issue:** casehubio/claudony#209
**Date:** 2026-10-12
**Status:** Draft

---

## Problem

Claudony's agent pools define agent identity independently — a name, a command, a model chain. There is no connection to the platform's agent identity system (Eidos), which defines rich agent descriptors with capabilities, model requirements, goals, constraints, and organizational relationships.

The result: pool definitions must be manually configured to match Eidos agent definitions. When Eidos says "agent code-reviewer exists with these capabilities," Claudony has no way to automatically provision pool backing for that agent.

## Goal

Bridge Eidos agent identity into Claudony's pool layer so that:

1. Agents defined in the Eidos `AgentRegistry` automatically get pool backing in Claudony
2. Pool configuration (sizing, scaling) flows through the established `ProvisionerConfigRegistry` path
3. Sessions created by Eidos-backed pools carry the agent's organizational identity
4. The bridge is optional — Claudony works without Eidos

---

## Architecture

```
AgentRegistry (Eidos)                     ProvisionerConfigRegistry (Engine)
  │                                         │
  │ AgentDescriptor                         │ configFor("claudony", agentId)
  │   agentId, name                         │   → {minActive, maxActive, ...}
  │   capabilities[].model (ModelQuery)     │
  │   providerConfigs[] (ProviderConfig)    │
  │                                         │
  └──────────────┬──────────────────────────┘
                 │
                 ▼
        EidosPoolBridge (claudony-casehub)
          @IfBuildProperty("claudony.eidos.pool-bridge.enabled")
          startup read + 30s reconciliation
                 │
                 ▼
    AgentPoolDefinitionRegistry (claudony-casehub)
      AgentPoolDefinition per matched agent
        AgentConfig: name, command, modelChain, eidosAgentId
        PoolConfig: minActive, maxActive, eviction, scaling
                 │
                 ▼
    AgentSessionManager → tmux session
      @claudony_eidos_agent_id = agentId
```

### Data Flow

1. **Identity source:** `AgentRegistry.find(AgentQuery.all(tenancyId))` returns all agent descriptors for the configured tenant.
2. **Filter:** Only agents with a `ProviderConfig` where `providerName = "claudony"` are pool candidates. Agents without this config are not Claudony-backed.
3. **Pool config source:** `CompositeProviderConfigSource.configFor("claudony", agentId)` — the existing path. Eidos `ProviderConfig` entries feed into `ProvisionerConfigRegistry` via the standard deployment pipeline. The bridge reads pool sizing from this path with sensible defaults when absent.
4. **Model resolution:** The first `AgentCapability` with a non-null `model` (ModelQuery) or `modelRef` determines the pool's model chain. This is a coarse-grained default — per-invocation model selection happens at the routing layer (`RoutingAgentProvider`).
5. **Registration:** `AgentPoolDefinitionRegistry.register(definition)` — one pool per matched agent.
6. **Session stamping:** Pool sessions carry `eidosAgentId` as a tmux user option (`@claudony_eidos_agent_id`), enabling downstream consumers (lineage, dashboard, context providers) to resolve the full `AgentDescriptor`.

### Repos and Responsibilities

| Repo | What changes |
|------|-------------|
| **claudony** (claudony-casehub) | New `EidosPoolBridge`, `EidosPoolConverter`. New `eidosAgentId` field on `AgentConfig`. Config properties. Tests. |
| **claudony** (claudony-app) | Config property wiring in `application.properties`. |
| No other repos | No Eidos API changes. No engine changes. Uses existing `ProvisionerConfigRegistry` path. |

---

## Component Design

### EidosPoolBridge

`@ApplicationScoped`, guarded by `@IfBuildProperty(name = "claudony.eidos.pool-bridge.enabled", stringValue = "true")`.

**Package:** `io.casehub.claudony.casehub.fleet`

**Injections:**
- `AgentRegistry` — Eidos agent registry (the real impl displaces `NoOpAgentRegistry` when eidos is active)
- `AgentPoolDefinitionRegistry` — existing pool definition store
- `CompositeProviderConfigSource` — existing per-agent config lookup

**Startup (`@Observes StartupEvent`):**
1. Query `AgentRegistry.find(AgentQuery.all(tenancyId))` for all descriptors
2. Filter to agents with `ProviderConfig` where `providerName = "claudony"`
3. For each matched agent, call `EidosPoolConverter.toPoolDefinition(descriptor, providerConfig)`
4. Register with `AgentPoolDefinitionRegistry`
5. Log: "Eidos pool bridge: registered N pool(s) for M agent(s)"

**Reconciliation (`@Scheduled(every = "${claudony.eidos.pool-bridge.reconcile-interval:30s}")`):**
1. Re-query `AgentRegistry`
2. Compare current registry state to `AgentPoolDefinitionRegistry.names()`
3. **New agents:** register new pool definitions
4. **Changed config:** update pool config via `AgentPoolDefinitionRegistry.updateCapacity()` / `updateScaling()`
5. **Removed agents:** log a warning but do NOT remove pools with active sessions. Pools without active sessions can be removed.
6. Reconciliation is idempotent — no-op when nothing changed

**Error handling:**
- Malformed descriptors: log warning, skip the agent, continue with remaining
- AgentRegistry unavailable: log error, retry on next reconciliation tick
- Never fail startup — partial pool provisioning is better than no pools

### EidosPoolConverter

**Package:** `io.casehub.claudony.casehub.fleet`

Pure conversion function — no CDI, no lifecycle, fully unit-testable.

```java
public final class EidosPoolConverter {

    public static AgentPoolDefinition toPoolDefinition(
            AgentDescriptor descriptor,
            ClaudonyProviderConfig providerConfig,
            EidosPoolBridgeConfig defaults) { ... }
}
```

**Conversion rules:**

| AgentDescriptor field | → AgentPoolDefinition field | Notes |
|---|---|---|
| `agentId` | `AgentConfig.name` | Pool keyed by agentId |
| `agentId` | `AgentConfig.eidosAgentId` | New field — organizational identity |
| First capability with `model` | `AgentConfig.modelChain` | ModelQuery → ModelChain via model family/version |
| ProviderConfig("claudony").config | `PoolConfig.minActive`, `maxActive` | String keys parsed to int, defaults from bridge config |
| ProviderConfig("claudony").config | `AgentConfig.command`, `workingDir` | Optional overrides |

**ModelQuery → ModelChain mapping:**
- `ModelQuery.modelFamily()` + `ModelQuery.modelVersion()` → single-entry `ModelChain`
- If `modelRef` is used instead, it's treated as a direct model name
- If no capability has model requirements, the pool uses the default command (no model override)

### AgentConfig changes

`AgentPoolDefinition.AgentConfig` gains one new field:

```java
public record AgentConfig(
    String name,
    String workingDir,
    WorkingDirPolicy policy,
    String command,
    ModelChain modelChain,
    Map<String, String> entryCommands,
    Map<String, Duration> gracePeriods,
    String eidosAgentId          // NEW — nullable, null for non-Eidos pools
) { ... }
```

The existing 4-arg and 7-arg constructors remain unchanged (eidosAgentId defaults to null). A new 8-arg canonical constructor adds the field.

### Session identity

When `AgentSessionManager` creates a tmux session for an Eidos-backed pool:
- Store `@claudony_eidos_agent_id` as a tmux user option (same pattern as `@claudony_state`)
- The `Session` model's existing fields are sufficient — no new model field needed. The tmux option is the authoritative source; downstream code reads it via `TmuxService`.

**Relationship to `roleName`:** `roleName` is a case-scoped role assignment ("researcher" on case X). `eidosAgentId` is the organizational identity ("senior-code-analyst" in the Eidos registry). They are complementary dimensions — a session can have both.

---

## Configuration

```properties
# Eidos pool bridge (default: off)
claudony.eidos.pool-bridge.enabled=false
claudony.eidos.pool-bridge.tenancy-id=default
claudony.eidos.pool-bridge.reconcile-interval=30s
claudony.eidos.pool-bridge.default-min-active=0
claudony.eidos.pool-bridge.default-max-active=5
```

---

## Multi-tenancy

Single-tenant for now. The bridge queries with a configurable `tenancyId` (defaults to `"default"`). `AgentPoolDefinitionRegistry` is keyed by agent name only — no tenant partitioning. Multi-tenant pool isolation is a separate concern for a future issue.

---

## Testing

### Unit tests (claudony-casehub)

- **EidosPoolConverterTest** — pure conversion: descriptor + config → pool definition. Cases:
  - Full descriptor with capabilities and model → correct AgentConfig + PoolConfig
  - No model on any capability → pool with null modelChain
  - Missing ProviderConfig fields → defaults applied
  - Multiple capabilities → first with model wins
  - eidosAgentId stamped correctly
  - Malformed config values (non-integer minActive) → graceful handling

- **EidosPoolBridgeTest** — startup + reconciliation:
  - Startup registers pools for matched agents
  - Agents without "claudony" ProviderConfig are skipped
  - Reconciliation adds new agents
  - Reconciliation updates changed config
  - Reconciliation skips removal of pools with active sessions
  - Malformed descriptors logged and skipped
  - Empty registry → no pools, no error

### Integration tests (claudony-app)

- **EidosPoolBridgeIntegrationTest** — `@QuarkusTest` with profile enabling the bridge:
  - Bridge activates when config property set
  - Bridge does NOT activate by default (property absent)
  - End-to-end: register descriptor in AgentRegistry → pool appears in AgentPoolDefinitionRegistry

---

## What This Does NOT Cover

- **Org structure integration** — relationships (SUPERVISES, DELEGATES_TO) don't affect pool provisioning. Separate concern for case orchestration.
- **Dynamic model resolution via ModelRegistry** — ModelQuery → ModelChain uses direct field mapping, not a ModelRegistry lookup. Full resolution is a routing-layer concern.
- **Multi-tenant pool isolation** — pools are cross-tenant by agent name.
- **Eidos-driven system prompt injection** — the bridge provides identity; `WorkerContextProvider` can resolve the descriptor for prompt rendering, but that's a separate feature.

---

## References

- `AgentDescriptor`, `AgentRegistry`, `AgentQuery`, `AgentCapability`, `ProviderConfig` (`casehub-eidos-api`) — Eidos identity model
- `AgentPoolDefinition`, `AgentPoolDefinitionRegistry`, `AgentPoolConfig` (`claudony-casehub`) — Claudony pool layer
- `CompositeProviderConfigSource`, `ClaudonyProviderConfig` (`claudony-casehub`) — existing per-agent config path
- `ProvisionerConfigRegistry` (`casehub-engine-api`) — platform provisioner config SPI
- `OrgStructure`, `OrganizationalUnit`, `Membership` (`casehub-eidos org-api`) — org layer (out of scope)
- `NoOpAgentRegistry` (`casehub-engine-runtime-core`) — always-present no-op (reason for config-driven activation)
- Issue #205 design spec — deferred this integration point
- Fleet script decisions.md D2 — "Eidos isn't on Claudony's classpath"
