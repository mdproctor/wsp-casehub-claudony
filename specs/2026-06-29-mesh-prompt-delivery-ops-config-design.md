# Mesh System Prompt Delivery + OpsProviderConfigSource

**Issues:** claudony#163 (fully addressed), claudony#164 (infrastructure only — ops integration is a follow-up)  
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
    return Optional.of(staticAppend.get() + "\n\n" + dynamicAppend.get());
}
```

Static (operator) append first — standing behavioral instructions anchor the agent's persona. Dynamic (mesh) prompt second — per-case operational context builds on top. This matches Claude's prompt processing: earlier content has stronger influence on baseline behavior.

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

**Edge cases:** `ClaudonyReactiveWorkerContextProvider.buildContext()` has two early-return paths — `clean-start=true` and `caseId == null` — that produce a `WorkerContext` with no `systemPrompt` property. The Optional chain above handles both correctly (returns `Optional.empty()`), so no `--append-system-prompt` flag is emitted. These paths are tested explicitly.

No changes to `ClaudonyProviderConfig` or `ProviderConfigSource`.

#### 1c. MeshSystemPromptTemplate null-workerId guard

`CaseContextChangedEventHandler.tryProvision()` passes `workerId=null` to `buildContext()` — the workerId isn't generated until provisioning (`setupSession()` creates `sessionId = UUID.randomUUID()`). Java string concatenation converts null to the literal string `"null"`, producing `register("null", ...)` in the mesh prompt. This has been harmless because the prompt was never delivered, but this spec changes that.

Fix: `MeshSystemPromptTemplate.buildActive()` handles null workerId by emitting a self-identification placeholder:

```java
private static String registrationInstruction(String workerId, String capability) {
    String id = workerId != null ? "\"" + workerId + "\"" : "<your-session-id>";
    return "  1. register(" + id + ", \"Starting " + capability + "\", [\"" + capability + "\"])\n";
}
```

When workerId is null, the agent is instructed to use its own session identifier for registration. The mesh server accepts any unique string.

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

#### 2b. NoOp @DefaultBean in casehub-engine-runtime

```java
package io.casehub.engine.internal.worker;

import io.casehub.api.spi.ProvisionerConfigRegistry;
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

Interface in engine-api: engine-api owns all provisioning-lifecycle SPIs (`ReactiveWorkerProvisioner`, `WorkerContextProvider`, `CaseChannelProvider`, `WorkerStatusListener`). ops, Claudony, and OpenClaw all depend on engine-api already — zero new dependency paths.

NoOp in engine-runtime: `@DefaultBean` is `io.quarkus.arc.DefaultBean` — a Quarkus Arc annotation not available in engine-api (Tier 1 pure-Java SPI module). All existing NoOp `@DefaultBean` implementations follow this split: interface in engine-api, NoOp in engine-runtime (`NoOpReactiveWorkerProvisioner`, `NoOpCaseChannelProvider`, `NoOpWorkerStatusListener`, etc.).

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
            if (workers.providerConfig().containsKey(agentId)) {
                LOG.warnf("Agent '%s': registry config displaces application.properties config entirely"
                        + " (no per-field merge — registry is authoritative when present)", agentId);
            }
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

- `forAgent()`: Registry wins. If registry has config for this agent, return it (full replacement — no per-field merge). Otherwise fall back to `application.properties`. **Cliff behavior:** if ops YAML declares an agent with only `model: opus`, the agent loses ALL `application.properties` defaults (`appendSystemPrompt`, `tools`, `effort`, etc.). The ops author must declare the complete config. A warning is logged when registry config displaces existing config-mapping config for the same agentId.
- `declaredAgentIds()`: Union of both sources. Registry agents + config-declared agents both contribute to capability discovery.

**Known limitation:** When ops is active, registry-declared agents appear in `getCapabilities()` even if their config references resources claudony cannot provide (e.g., an unavailable model). These agents will fail at provision time. The ops operator is responsible for declaring valid config; the config source does not validate provisioning feasibility — that is the provisioner's concern.

#### 3c. What gets deleted

- `ConfigMappingProviderConfigSource.java` — replaced by composite
- `ConfigMappingProviderConfigSourceTest.java` — replaced by `CompositeProviderConfigSourceTest`

## Test Strategy

### #163 tests

- `WorkerCommandBuilderTest`: system-prompt only, append-system-prompt only, both together (existing `systemPromptWinsOverAppend_builderPolicy` updated to assert both flags coexist), dynamic mesh prompt only, dynamic + static append merged (static first), dynamic + system-prompt + static append all three, empty dynamic prompt (backward compat), multi-line mesh prompt with embedded quotes and special characters survives shell quoting
- `MeshSystemPromptTemplateTest`: null workerId produces placeholder instruction (not literal `"null"`)
- `ClaudonyReactiveWorkerProvisionerTest`: new test verifying mesh prompt from WorkerContext is passed through to the enriched command; null WorkerContext handled gracefully; SILENT participation (empty mesh prompt) produces no append flag; `clean-start=true` context produces no append flag; `caseId == null` context produces no append flag

### #164 tests

- `CompositeProviderConfigSourceTest`: registry-has-config returns it; registry-empty falls back to config mapping; declaredAgentIds union; unknown agent returns EMPTY; registry-only agent not in config mapping still discovered; warning logged when registry displaces config-mapping config for same agentId
- Existing `ClaudonyReactiveWorkerProvisionerTest` passes unchanged (NoOp registry = same behavior)

## Execution Order

| Step | Repo | What |
|------|------|------|
| 1 | casehub-engine | `ProvisionerConfigRegistry` SPI in engine-api + `NoOpProvisionerConfigRegistry @DefaultBean` in engine-runtime |
| 2 | claudony | #163 — `WorkerCommandBuilder` + provisioner changes (TDD) |
| 3 | claudony | #164 — `CompositeProviderConfigSource` replaces `ConfigMappingProviderConfigSource` (TDD) |
| 4 | claudony | Full test suite verification |

## Follow-up Issues

| Repo | Description |
|------|-------------|
| casehubio/engine | Add `ProvisionerConfigRegistry` SPI to engine-api + `NoOpProvisionerConfigRegistry @DefaultBean` to engine-runtime |
| casehubio/casehub-ops | `DeploymentProvisionerConfigRegistry @Alternative @Priority(1)` wrapping `DeploymentProviderConfigStore` |
| casehubio/openclaw | Migrate `AgentProviderConfigSource` to delegate to `ProvisionerConfigRegistry` with `providerName="openclaw"`. Note: impedance mismatch — OpenClaw's `AgentConfig` is a typed record (`sessionKey()`, `capabilities()`), while `ProvisionerConfigRegistry` returns opaque `Map<String, Object>`. The migration must decide between adapting AgentConfig to/from the opaque map or keeping the typed SPI alongside the registry lookup. |
| casehubio/parent | Update PLATFORM.md capability ownership + cross-repo dependency map |

## Out of Scope

- Per-field merge of registry + config mapping configs (registry-wins is the semantic)
- Reactive variant of `ProvisionerConfigRegistry` (blocking correct for in-memory; add via build-gating later)
- Promoting `ProviderConfigSource` to engine-api (returns `ClaudonyProviderConfig` — correctly Claudony-specific)
