# Mesh System Prompt Delivery + OpsProviderConfigSource

**Issues:** claudony#163, claudony#164  
**Branch:** issue-163-mesh-prompt-ops-config  
**Date:** 2026-06-29

---

## Problem

Two gaps in Claudony's worker provisioning pipeline:

1. **#163 — Mesh system prompt not delivered to CLI.**  
   `MeshSystemPromptTemplate` generates a per-case system prompt (channels, prior workers, participation mode) and stores it in `WorkerContext.properties("systemPrompt")`. The engine passes this to the provisioner via `ProvisionContext.workerContext()`. But `ClaudonyReactiveWorkerProvisioner.setupSession()` ignores the `WorkerContext` entirely — it only reads static per-agent config from `ProviderConfigSource`. Workers start with no mesh awareness.

2. **#164 — No path for ops-driven agent config.**  
   `ProviderConfigSource` SPI (introduced in #156) has one implementation: `ConfigMappingProviderConfigSource` reading from `application.properties`. When casehub-ops is co-deployed, agent config should come from the deployment topology (`casehub-deployment.yaml` → `DeploymentProviderConfigStore`). But the store is in ops's deployment module (not API), and peer integration modules shouldn't have compile deps on each other's internals. No shared SPI exists for provisioner config lookup.

## Architecture Context

Three system prompt layers serve orthogonal purposes:

| Layer | Source | Purpose | Lifecycle |
|-------|--------|---------|-----------|
| Identity | Claude Code default | Tool usage, file editing, git | Static per CLI version |
| Persona | Operator config (`ClaudonyProviderConfig`) | Agent specialization | Static per deployment |
| Context | Mesh prompt (`MeshSystemPromptTemplate`) | Case channels, prior workers, participation | Dynamic per provisioning |

CLI flag mapping:
- `--system-prompt` replaces Identity (operator's Persona layer choice)
- `--append-system-prompt` adds to whatever the effective base is (Context layer, always additive)
- Both flags can coexist — `--system-prompt` sets the base, `--append-system-prompt` appends to it

## Design

### Part 1 — Mesh system prompt delivery (#163)

#### 1a. WorkerCommandBuilder changes

Remove the mutual exclusion between `--system-prompt` and `--append-system-prompt`. Add a `dynamicAppendPrompt` parameter for per-provisioning context (the mesh prompt):

```java
public static String build(String baseCommand, ClaudonyProviderConfig config,
                           Optional<String> dynamicAppendPrompt) {
    var sb = new StringBuilder(baseCommand);
    appendString(sb, "--model", config.model());
    appendString(sb, "--system-prompt", config.systemPrompt());
    Optional<String> effectiveAppend = mergeAppendPrompts(
            config.appendSystemPrompt(), dynamicAppendPrompt);
    appendString(sb, "--append-system-prompt", effectiveAppend);
    appendString(sb, "--effort", config.effort());
    appendString(sb, "--permission-mode", config.permissionMode());
    appendList(sb, "--tools", config.tools());
    appendList(sb, "--allowedTools", config.allowedTools());
    appendList(sb, "--disallowedTools", config.disallowedTools());
    config.addDirs().ifPresent(dirs ->
        dirs.forEach(dir -> appendString(sb, "--add-dir", Optional.of(dir))));
    return sb.toString();
}

private static Optional<String> mergeAppendPrompts(Optional<String> staticAppend,
                                                     Optional<String> dynamicAppend) {
    if (staticAppend.isEmpty() && dynamicAppend.isEmpty()) return Optional.empty();
    if (staticAppend.isEmpty()) return dynamicAppend;
    if (dynamicAppend.isEmpty()) return staticAppend;
    return Optional.of(dynamicAppend.get() + "\n\n" + staticAppend.get());
}
```

Dynamic (mesh) prompt first in the merged output — it provides foundational context. Operator's static append refines on top.

The old 2-arg `build(String, ClaudonyProviderConfig)` method is removed, not preserved as a backward-compatible overload. All call sites updated.

#### 1b. Provisioner changes

In `ClaudonyReactiveWorkerProvisioner.setupSession()`, extract the mesh prompt from the WorkerContext and pass it to the builder:

```java
ClaudonyProviderConfig config = providerConfigSource.forAgent(roleName);
String baseCommand = config.command().orElse(defaultCommand);

Optional<String> meshPrompt = Optional.ofNullable(context.workerContext())
    .map(wc -> wc.properties().get("systemPrompt"))
    .filter(String.class::isInstance)
    .map(String.class::cast);

String enrichedCommand = WorkerCommandBuilder.build(baseCommand, config, meshPrompt);
```

No changes to `ClaudonyProviderConfig`, `ProviderConfigSource`, or `MeshSystemPromptTemplate`.

### Part 2 — ProvisionerConfigRegistry SPI (#164, engine-api)

#### 2a. New SPI in casehub-engine-api

```java
package io.casehub.api.spi;

import java.util.Map;
import java.util.Set;

public interface ProvisionerConfigRegistry {
    Map<String, Object> configFor(String providerName, String agentId);
    Set<String> declaredAgentIds(String providerName);
}
```

#### 2b. NoOp @DefaultBean in casehub-engine-api

```java
package io.casehub.api.spi;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;
import java.util.Set;

@DefaultBean
@ApplicationScoped
public class NoOpProvisionerConfigRegistry implements ProvisionerConfigRegistry {
    @Override
    public Map<String, Object> configFor(String providerName, String agentId) {
        return Map.of();
    }
    @Override
    public Set<String> declaredAgentIds(String providerName) {
        return Set.of();
    }
}
```

#### 2c. Placement rationale

engine-api already owns all provisioning-lifecycle SPIs (`ReactiveWorkerProvisioner`, `WorkerContextProvider`, `CaseChannelProvider`, `WorkerStatusListener`). ops, Claudony, and OpenClaw all depend on engine-api already — zero new dependency paths.

### Part 3 — CompositeProviderConfigSource (#164, Claudony)

#### 3a. Replace ConfigMappingProviderConfigSource

```java
@ApplicationScoped
public class CompositeProviderConfigSource implements ProviderConfigSource {

    private static final String PROVIDER_NAME = "claudony";

    private final ProvisionerConfigRegistry registry;
    private final CaseHubConfig.Workers workers;

    @Inject
    public CompositeProviderConfigSource(ProvisionerConfigRegistry registry,
                                          CaseHubConfig config) {
        this.registry = registry;
        this.workers = config.workers();
    }

    @Override
    public ClaudonyProviderConfig forAgent(String agentId) {
        Map<String, Object> registryConfig = registry.configFor(PROVIDER_NAME, agentId);
        if (!registryConfig.isEmpty()) {
            return ClaudonyProviderConfig.fromMap(registryConfig);
        }
        var agentConfig = workers.providerConfig().get(agentId);
        return agentConfig == null ? ClaudonyProviderConfig.EMPTY
                                   : ClaudonyProviderConfig.fromConfigMapping(agentConfig);
    }

    @Override
    public Set<String> declaredAgentIds() {
        var all = new HashSet<>(registry.declaredAgentIds(PROVIDER_NAME));
        all.addAll(workers.providerConfig().keySet());
        return Set.copyOf(all);
    }
}
```

#### 3b. Resolution semantics

- `forAgent()`: Registry wins. If registry has config for this agent, return it. Otherwise fall back to `application.properties`. No per-field merge — registry is authoritative when present.
- `declaredAgentIds()`: Union of both sources. Registry agents + config-declared agents both contribute to capability discovery.

#### 3c. What gets deleted

- `ConfigMappingProviderConfigSource.java` — replaced by composite
- `ConfigMappingProviderConfigSourceTest.java` — replaced by `CompositeProviderConfigSourceTest`

## Test Strategy

### #163 tests

- `WorkerCommandBuilderTest`: system-prompt only, append-system-prompt only, both together, dynamic mesh prompt only, dynamic + static append merged, dynamic + system-prompt + static append all three, empty dynamic prompt (backward compat)
- `ClaudonyReactiveWorkerProvisionerTest`: new test verifying mesh prompt from WorkerContext is passed through to the enriched command; null WorkerContext handled gracefully; SILENT participation (empty mesh prompt) produces no append flag

### #164 tests

- `CompositeProviderConfigSourceTest`: registry-has-config returns it; registry-empty falls back to config mapping; declaredAgentIds union; unknown agent returns EMPTY; registry-only agent not in config mapping still discovered
- Existing `ClaudonyReactiveWorkerProvisionerTest` passes unchanged (NoOp registry = same behavior)

## Execution Order

| Step | Repo | What |
|------|------|------|
| 1 | casehub-engine | `ProvisionerConfigRegistry` SPI + `NoOpProvisionerConfigRegistry @DefaultBean` in engine-api |
| 2 | claudony | #163 — `WorkerCommandBuilder` + provisioner changes (TDD) |
| 3 | claudony | #164 — `CompositeProviderConfigSource` replaces `ConfigMappingProviderConfigSource` (TDD) |
| 4 | claudony | Full test suite verification |

## Follow-up Issues

| Repo | Description |
|------|-------------|
| casehubio/engine | Add `ProvisionerConfigRegistry` SPI to engine-api |
| casehubio/casehub-ops | `DeploymentProvisionerConfigRegistry @Alternative @Priority(1)` wrapping `DeploymentProviderConfigStore` |
| casehubio/openclaw | Migrate `AgentProviderConfigSource` to delegate to `ProvisionerConfigRegistry` with `providerName="openclaw"` |
| casehubio/parent | Update PLATFORM.md capability ownership + cross-repo dependency map |

## Out of Scope

- Per-field merge of registry + config mapping configs (registry-wins is the semantic)
- Reactive variant of `ProvisionerConfigRegistry` (blocking correct for in-memory; add via build-gating later)
- Promoting `ProviderConfigSource` to engine-api (returns `ClaudonyProviderConfig` — correctly Claudony-specific)
