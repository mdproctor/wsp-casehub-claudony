# Eidos Pool Bridge Hardening — Design Spec

**Epic:** casehubio/claudony#275
**Children:** #276, #277, #278, #279, #280, #281
**Date:** 2026-10-14

## Overview

The Eidos pool bridge (#209) established the connection between agent identity (`AgentDescriptor`) and pool management (`AgentPoolDefinition`). This epic hardens that integration across six dimensions: making identity visible to sessions, reacting to descriptor changes, injecting identity into system prompts, resolving models through a registry, isolating pools by tenant, and receiving CDI events instead of polling.

The design follows one central principle: **identity flows downward** — from `AgentDescriptor` through the pool definition, into the tmux session metadata, and finally into the worker's system prompt. Each child issue extends a different segment of that flow.

## Identity Flow

```
AgentDescriptor (eidos-api)
    │
    ├─ #279: ModelRegistry resolves ModelQuery → ModelChain
    │
    ▼
EidosPoolConverter.toPoolDefinition()
    │
    ├─ #280: composite key (agentName, tenancyId)
    │
    ▼
AgentPoolDefinitionRegistry
    │
    ├─ #277: reconcile detects changes, updates capacity + config
    ├─ #281: CDI events replace polling (blocked on eidos#187)
    │
    ▼
TmuxSessionOperations.createWorkerSession()
    │
    ├─ #276: stamps @claudony_eidos_agent_id on tmux session
    │
    ▼
ClaudonyWorkerContextProvider.buildContext()
    │
    ├─ #278: resolves AgentDescriptor from eidosAgentId,
    │        injects briefing/goals/constraints as identity prompt
    │
    ▼
WorkerCommandBuilder.build()
    └─ --system-prompt: Eidos identity
       --append-system-prompt: mesh context (channels, prior workers)
```

---

## #276 — Stamp tmux sessions with eidosAgentId

**Scale:** S | **Complexity:** Low | **Blocks:** #278

### What changes

When a pool session is created and the `AgentConfig` carries a non-null `eidosAgentId`, write it as a tmux user option on the session:

```
@claudony_eidos_agent_id = <value>
```

### Where

`TmuxSessionOperations.createWorkerSession()` — after session creation, if `agentConfig.eidosAgentId()` is non-null, call `tmux set-option -t <session> @claudony_eidos_agent_id <value>`. This is the same pattern used for `@claudony_state`, `@claudony_conversation_id`, and `@claudony_identity`.

### Read path

Add `TmuxService.getEidosAgentId(String sessionName)` — reads `@claudony_eidos_agent_id` via `tmux show-options -v`. Returns `Optional<String>`. Used by #278 to resolve the descriptor for system prompt injection.

### Session model

Add `eidosAgentId` field to `Session` record. Populated at creation time from `AgentConfig`. Visible in REST API response (`GET /api/sessions`).

### Tests

- Unit: `TmuxSessionOperations` stamps option when eidosAgentId present, skips when null
- Unit: `Session` model carries eidosAgentId
- Integration: `FleetPoolIntegrationTest` — verify tmux option readable after create

---

## #277 — Reconciliation for changed Eidos descriptors

**Scale:** S | **Complexity:** Med

### What changes

`EidosPoolBridge.reconcile()` currently only adds new pools. Extend it to detect when an existing pool's configuration diverges from the current `AgentDescriptor` and update in place.

### Change detection

Compare the current `AgentDescriptor` against what was used to create the pool.

Store the converted `AgentPoolDefinition` at registration time in the bridge. On reconcile, re-convert via `EidosPoolConverter.toPoolDefinition()` and compare the resulting `AgentPoolDefinition` against the registered one. If different, update.

Fields that trigger updates:
- `PoolConfig` changes (minActive, maxActive) → `updateCapacity()`
- `ScalingConfig` changes → `updateScaling()`
- `AgentConfig` changes (command, model, workingDir) → new `updateAgentConfig()` method on registry

### New registry method

`AgentPoolDefinitionRegistry.updateAgentConfig(String agentName, AgentConfig newConfig)` — replaces the `AgentConfig` portion of the registered definition. Running sessions are unaffected; only new sessions pick up the updated config.

### Deletion handling

When a descriptor disappears from `AgentRegistry` (present in bridge's known set but absent from query results), do NOT auto-destroy the pool. Log a warning. Explicit pool destruction requires operator action. This prevents accidental pool teardown from transient registry issues.

### Bridge state

`EidosPoolBridge` maintains a `Map<String, AgentPoolDefinition> lastKnownDefinitions` — populated at startup, updated on reconcile. Comparison is against this map, not the registry directly (the registry may have been modified by other sources).

### Tests

- Unit: `EidosPoolBridgeTest` — reconcile detects capacity change, config change, no-change no-op, missing descriptor logs warning
- Unit: `AgentPoolDefinitionRegistry` — updateAgentConfig replaces config, preserves pool config

---

## #278 — Eidos-driven system prompt injection

**Scale:** M | **Complexity:** Med | **Depends on:** #276

### What changes

`ClaudonyWorkerContextProvider.buildContext()` resolves the `AgentDescriptor` from the session's `eidosAgentId` and constructs an identity prompt from `briefing`, `goals`, and `constraints`. This prompt is set as the primary system prompt (`--system-prompt`), while the existing mesh prompt becomes the append prompt (`--append-system-prompt`).

### Prompt structure

```
# Identity

<briefing text>

## Goals

- <public goal 1 — name: description>
- <public goal 2 — name: description>

## Constraints

- <public constraint 1 — name: description>
- <public constraint 2 — name: description>
```

Only public goals and constraints are included (filtered via `descriptor.publicGoals()` and `descriptor.publicConstraints()`).

### Resolution path

1. `ClaudonyWorkerContextProvider.buildContext()` reads `eidosAgentId` from the `Session` (via `SessionRegistry`)
2. If present, calls `AgentRegistry.findById(eidosAgentId, tenancyId)` to get the `AgentDescriptor`
3. Renders identity prompt via new `EidosIdentityPromptTemplate.generate(AgentDescriptor)` (package-private, static, mirrors `MeshSystemPromptTemplate` pattern)
4. Sets identity prompt as `systemPrompt` property, mesh prompt as `appendSystemPrompt` property

### Fallback

When `eidosAgentId` is null or `AgentDescriptor` not found: no identity prompt injected. Mesh prompt remains the primary system prompt (current behavior). No error — graceful degradation.

### Injection point

`ClaudonyWorkerContextProvider` is `@ApplicationScoped` regardless of the Eidos flag. Inject `Instance<AgentRegistry>` (CDI programmatic lookup). When unsatisfied (Eidos disabled), skip identity prompt. When satisfied, resolve descriptor and render identity prompt.

### WorkerCommandBuilder integration

`WorkerCommandBuilder.build()` already handles `--system-prompt` from `config.systemPrompt()` and `--append-system-prompt` from merged prompts. The context provider sets:
- `props.put("systemPrompt", eidosIdentityPrompt)` — becomes `--system-prompt`
- `props.put("appendSystemPrompt", meshPrompt)` — becomes `--append-system-prompt`

When no Eidos identity: `systemPrompt` stays null, mesh prompt goes to `appendSystemPrompt` (current behavior preserved).

### Tests

- Unit: `EidosIdentityPromptTemplateTest` — full prompt with briefing+goals+constraints, briefing only, empty goals/constraints, null descriptor returns empty
- Unit: `ClaudonyWorkerContextProviderTest` — with eidosAgentId resolves identity prompt, without skips, AgentRegistry unsatisfied skips gracefully
- Integration: verify prompt layering in full provisioning chain

---

## #279 — ModelRegistry resolution for capability ModelQuery

**Scale:** S | **Complexity:** Med

### What changes

`EidosPoolConverter.toPoolDefinition()` currently maps `ModelQuery.family()` directly to `ModelChain.of()`. Replace with `ModelRegistry.query()` when a registry is available, falling back to direct mapping when not.

### Design

`EidosPoolConverter` becomes a CDI bean (currently a static utility class) to accept optional injection:

```java
@ApplicationScoped
public class EidosPoolConverter {
    @Inject
    Instance<ModelRegistry> modelRegistry;

    public AgentPoolDefinition toPoolDefinition(AgentDescriptor descriptor,
                                                 EidosPoolBridgeConfig defaults) {
        // ... existing logic ...
        ModelChain chain = resolveModel(capability);
        // ...
    }

    ModelChain resolveModel(AgentCapability capability) {
        if (capability.model() == null) return null;
        if (modelRegistry.isResolvable()) {
            return modelRegistry.get()
                .query(capability.model())
                .map(ModelChain::of)
                .orElseGet(() -> directMapping(capability.model()));
        }
        return directMapping(capability.model());
    }
}
```

### Fallback chain

1. `ModelRegistry.query(modelQuery)` → ranked results → first match → `ModelChain`
2. If registry returns empty or is unavailable → `ModelQuery.family()` → `ModelChain.of()` (current behavior)

### Tests

- Unit: `EidosPoolConverterTest` — with registry resolves via query, without registry falls back to direct mapping, registry returns empty falls back, null model returns null

---

## #280 — Multi-tenant pool isolation

**Scale:** M | **Complexity:** Med

### What changes

`AgentPoolDefinitionRegistry` currently keys pools by `agentName` (flat `ConcurrentHashMap<String, AgentPoolDefinition>`). Change to composite key `(agentName, tenancyId)` so the same agent identity in different tenancies gets separate pools.

### Composite key

```java
record PoolKey(String agentName, String tenancyId) {
    PoolKey {
        Objects.requireNonNull(agentName);
        Objects.requireNonNull(tenancyId);
    }
}
```

Internal map becomes `ConcurrentHashMap<PoolKey, AgentPoolDefinition>`.

### Pool naming

Pool names (used for tmux session prefixes, logging, registry service registration) become `agentName@tenancyId`. For the default tenant (`tenancyId="default"`), the pool name remains just `agentName` — backwards compatible.

```java
static String poolName(String agentName, String tenancyId) {
    return "default".equals(tenancyId) ? agentName : agentName + "@" + tenancyId;
}
```

### API changes

All registry methods gain a `tenancyId` parameter:
- `register(AgentPoolDefinition, String namespace, String tenancyId)`
- `get(String agentName, String tenancyId)`
- `updateCapacity(String agentName, String tenancyId, int min, int max)`
- `updateScaling(String agentName, String tenancyId, ScalingConfig)`
- `updateAgentConfig(String agentName, String tenancyId, AgentConfig)` (from #277)
- `names(String tenancyId)` — filtered view
- `all(String tenancyId)` — filtered view

Existing single-argument methods become convenience overloads that pass `"default"` as tenancyId.

### EidosPoolBridge

The bridge already has a `tenancyId` config property. It passes this through to all registry operations. Multiple bridges (one per tenant) can coexist, each with its own `tenancyId`.

### Tests

- Unit: `AgentPoolDefinitionRegistryTest` — same agent name different tenants get separate pools, default tenant backwards compat, filtered views
- Unit: `EidosPoolBridgeTest` — tenancyId passed through to registry
- Integration: verify pool isolation with two tenants

---

## #281 — CDI events for AgentDescriptor changes

**Scale:** S | **Complexity:** Low | **Blocked by:** casehubio/eidos#187

### What changes

When eidos-api publishes CDI events for descriptor mutations (eidos#187), `EidosPoolBridge` observes them for immediate reaction instead of waiting for the 30s polling cycle.

### Event handling

```java
void onRegistered(@ObservesAsync AgentDescriptorRegistered event) {
    if (findClaudonyProviderConfig(event.descriptor()).isPresent()) {
        registerPool(event.descriptor());
    }
}

void onUpdated(@ObservesAsync AgentDescriptorUpdated event) {
    if (findClaudonyProviderConfig(event.current()).isPresent()) {
        reconcilePool(event.previous(), event.current());
    }
}

void onRemoved(@ObservesAsync AgentDescriptorRemoved event) {
    // Log warning, do NOT auto-destroy (same policy as #277)
}
```

### Polling retention

The 30s `reconcile()` polling remains as a drift safety net. Events handle the fast path; polling catches anything events miss (network partitions, missed events, manual registry edits).

### Tests

- Unit: `EidosPoolBridgeTest` — registered event triggers pool creation, updated event triggers reconcile, removed event logs warning
- Unit: verify polling still runs alongside events

---

## Cross-cutting concerns

### CDI bean wiring (protocol: engine-cdi-exclude-types-sync)

New beans introduced:
- `EidosIdentityPromptTemplate` — package-private static utility, no CDI (like `MeshSystemPromptTemplate`)
- `EidosPoolConverter` changes from static utility to `@ApplicationScoped` (for #279 ModelRegistry injection)

Both are guarded by `@IfBuildProperty("claudony.eidos.pool-bridge.enabled")` or use `Instance<>` programmatic lookup. No new exclude-types entries needed in test profiles unless `EidosPoolConverter` causes ambiguous resolution — verify during implementation.

### Test conventions

All new tests follow existing patterns:
- Unit tests: plain JUnit, no Quarkus context
- `@QuarkusTest` with `@TestSecurity(user = "test", roles = "user")` only for HTTP endpoint tests
- CDI-only tests: no `@TestSecurity`
- Qhorus data cleanup: `@AfterEach` clear on InMemoryChannelStore/MessageStore

### Implementation order

1. **#276** (session stamping) — no dependencies, enables #278
2. **#277** (reconciliation) — independent, can parallel with #276
3. **#278** (system prompt injection) — after #276
4. **#279** (ModelRegistry) — independent, can parallel
5. **#280** (multi-tenant) — independent, largest scope
6. **#281** (CDI events) — after eidos#187 lands

## References

- `WorkerCommandBuilder.java` — mergeAppendPrompts, --system-prompt + --append-system-prompt coexistence
- `MeshSystemPromptTemplate.java` — existing prompt template pattern
- `ClaudonyWorkerContextProvider.java` — buildContext/assemble, systemPrompt property
- `EidosPoolBridge.java` — startup, reconcile loop, tenancyId config
- `EidosPoolConverter.java` — toPoolDefinition, findClaudonyProviderConfig
- `AgentDescriptor.java` (eidos-api) — briefing, goals, constraints, capabilities, providerConfigs
- `AgentPoolDefinitionRegistry.java` — flat ConcurrentHashMap keyed by agent name
- `TmuxSessionOperations.java` — createWorkerSession, tmux user options pattern
- Protocol: engine-cdi-exclude-types-sync — exclude-types sync across test profiles
- Protocol: worker-session-creation — createWorkerSession for pool/worker sessions
- casehubio/eidos#187 — upstream CDI events for AgentDescriptor mutations
- casehubio/claudony#209 — original Eidos pool bridge implementation
