# Model Fallback Chains for Agent Routing

**Issue:** casehubio/claudony#212
**Date:** 2026-10-03
**Status:** Draft

---

## Problem

When an agent pool requests a model that isn't available — not registered in the catalog, budget exceeded, pool at capacity, or API rate-limited — the request fails hard. `RoutingAgentProvider.resolve()` throws `IllegalArgumentException`; `resolveQuery()` throws when no candidates match. There is no degradation path.

Operators need ordered model preferences with automatic fallback: "use opus if available, fall back to sonnet, then haiku." This applies to both programmatic agent invocation (`RoutingAgentProvider.invoke()`) and CLI-based worker provisioning (`WorkerCommandBuilder` → tmux session).

## Scope

- **In scope:** Fallback chain type system, resolution logic in `RoutingAgentProvider`, availability SPI, pool YAML configuration, CLI path integration, runtime circuit-breaker, degraded provisioning notification, cross-backend chains
- **Out of scope:** Mid-conversation model switching (CLI sessions can't swap models after startup), fleet-wide fallback policy (per-pool only for now), automatic model catalog discovery

## Architecture

### Three trigger levels

| Level | Signal | Detection | Path |
|-------|--------|-----------|------|
| Resolution | Model not in `ModelRegistry`, backend not in `BackendInstanceRegistry` | `resolve()`/`resolveQuery()` throws | Both |
| Operational | Pool at capacity, budget exceeded | `ModelAvailabilityFilter.isAvailable()` returns false | Both |
| Runtime | API 429, auth error, early session exit | Programmatic: exception from `AgentBackend.invoke()`. CLI: tmux session exits within grace period with non-zero exit code | Both (different mechanics) |

### Component layout

```
casehub-platform-agent-api          casehub-platform-agent-router-core
┌──────────────────────────┐        ┌─────────────────────────────────┐
│ ModelChain                │        │ RoutingAgentProvider            │
│ ModelChainEntry (sealed)  │───────>│   resolveChain()               │
│   Named(String)           │        │   resolveWithAvailability()     │
│   Queried(ModelQuery)     │        │                                 │
│                           │        │ invoke() / openSession()        │
│ ModelAvailabilityFilter   │───────>│   checks modelChain() first    │
│   isAvailable(descriptor) │        └─────────────────────────────────┘
│                           │
│ AgentSessionInit          │        claudony-casehub
│   + modelChain()          │        ┌─────────────────────────────────┐
│ AgentSessionConfig        │        │ AgentPoolYamlParser             │
│   + modelChain()          │        │   parses model-chain → ModelChain│
│                           │        │                                 │
│ ChainResolutionResult     │        │ AgentPoolDefinition.AgentConfig │
│   resolvedModel           │        │   + modelChain field            │
│   originalRequest         │        │                                 │
│   wasFallback             │        │ ClaudonyModelAvailabilityFilter │
│   attemptedEntries        │        │   checks pool + budget          │
└──────────────────────────┘        │                                 │
                                     │ CLI chain resolution            │
                                     │   lightweight local iteration   │
                                     │                                 │
                                     │ Runtime circuit breaker         │
                                     │   early-exit detection + retry  │
                                     └─────────────────────────────────┘
```

---

## Platform changes

### `casehub-platform-agent-api`

#### ModelChain and ModelChainEntry

```java
package io.casehub.platform.api.model;

public record ModelChain(List<ModelChainEntry> entries) {

    public static ModelChain of(String... modelRefs) {
        return new ModelChain(
            Arrays.stream(modelRefs)
                  .map(ModelChainEntry.Named::new)
                  .map(e -> (ModelChainEntry) e)
                  .toList());
    }

    public static ModelChain of(List<ModelChainEntry> entries) {
        return new ModelChain(List.copyOf(entries));
    }

    public boolean isEmpty() { return entries.isEmpty(); }

    public sealed interface ModelChainEntry {
        record Named(String modelRef) implements ModelChainEntry {}
        record Queried(ModelQuery query) implements ModelChainEntry {}
    }
}
```

#### ModelAvailabilityFilter

```java
package io.casehub.platform.api.model;

@FunctionalInterface
public interface ModelAvailabilityFilter {
    boolean isAvailable(ModelDescriptor descriptor);

    ModelAvailabilityFilter ALWAYS_AVAILABLE = descriptor -> true;
}
```

#### ChainResolutionResult

```java
package io.casehub.platform.api.model;

public record ChainResolutionResult(
        ModelDescriptor resolvedModel,
        ModelChainEntry originalEntry,
        boolean wasFallback,
        List<ModelChainEntry> attemptedEntries) {

    public static ChainResolutionResult direct(ModelDescriptor model, ModelChainEntry entry) {
        return new ChainResolutionResult(model, entry, false, List.of(entry));
    }
}
```

#### AgentSessionInit / AgentSessionConfig additions

Both records gain a `modelChain` field (nullable). Backward-compatible: existing constructors default to `null`. New `withModelChain()` builder method mirrors existing `withModel()`:

```java
// In AgentSessionInit
private final ModelChain modelChain;  // nullable

public AgentSessionInit withModelChain(ModelChain chain) {
    return new AgentSessionInit(systemPrompt, mcpServers, timeout,
                                correlationId, model, modelQuery, chain);
}
```

### `casehub-platform-agent-router-core`

#### RoutingAgentProvider — new chain resolution

```java
// New method — iterates chain, returns first resolvable + available entry
private ResolvedRoute resolveChain(ModelChain chain, ModelAvailabilityFilter filter) {
    List<ModelChainEntry> attempted = new ArrayList<>();
    for (var entry : chain.entries()) {
        attempted.add(entry);
        try {
            ResolvedRoute route = switch (entry) {
                case ModelChainEntry.Named n -> resolve(n.modelRef());
                case ModelChainEntry.Queried q -> resolveQuery(q.query());
            };

            // Resolution succeeded — check operational availability
            if (route.apiModelId() == null) return route;  // backend-key resolution, no filter
            var descriptor = modelRegistry.resolveById(route.apiModelId());
            if (descriptor.isPresent() && !filter.isAvailable(descriptor.get())) {
                LOG.debugf("Chain entry %s resolved but operationally unavailable, trying next", entry);
                continue;
            }

            LOG.infof("Chain resolved to %s after %d attempt(s) (fallback=%s)",
                       route.apiModelId(), attempted.size(), attempted.size() > 1);
            return route;
        } catch (IllegalArgumentException | IllegalStateException e) {
            LOG.debugf("Chain entry %s failed resolution: %s", entry, e.getMessage());
        }
    }
    throw new ModelChainExhaustedException(chain, attempted);
}
```

#### invoke() / openSession() integration

```java
@Override
public Multi<AgentEvent> invoke(AgentSessionConfig config) {
    var route = resolveFromConfig(config);
    var rewritten = new AgentSessionConfig(
            config.systemPrompt(), config.userPrompt(), config.mcpServers(),
            config.timeout(), config.correlationId(), route.apiModelId());
    return route.backend().invoke(rewritten);
}

private ResolvedRoute resolveFromConfig(AgentSessionConfig config) {
    if (config.modelChain() != null && !config.modelChain().isEmpty()) {
        return resolveChain(config.modelChain(), availabilityFilter);
    }
    return config.modelQuery() != null
           ? resolveQuery(config.modelQuery())
           : resolve(config.model());
}
```

The `availabilityFilter` is injected at construction time (defaults to `ALWAYS_AVAILABLE`):

```java
public RoutingAgentProvider(BackendInstanceRegistry registry,
                            String defaultBackendKey,
                            ModelRegistry modelRegistry,
                            Map<String, ModelQuery> aliases,
                            ModelAvailabilityFilter availabilityFilter) {
    // ...
    this.availabilityFilter = availabilityFilter != null
                              ? availabilityFilter : ModelAvailabilityFilter.ALWAYS_AVAILABLE;
}
```

#### ModelChainExhaustedException

```java
package io.casehub.platform.agent.router;

public class ModelChainExhaustedException extends IllegalStateException {
    private final ModelChain chain;
    private final List<ModelChainEntry> attemptedEntries;

    public ModelChainExhaustedException(ModelChain chain, List<ModelChainEntry> attempted) {
        super("All %d chain entries exhausted".formatted(attempted.size()));
        this.chain = chain;
        this.attemptedEntries = List.copyOf(attempted);
    }

    public ModelChain chain() { return chain; }
    public List<ModelChainEntry> attemptedEntries() { return attemptedEntries; }
}
```

---

## Claudony changes

### Pool YAML configuration

Chain entries in pool YAML support three forms:

```yaml
agent-pools:
  code-reviewer:
    model-chain:
      - opus                         # Named — simple model reference
      - model: sonnet                # Named — explicit key
      - tier: FLAGSHIP               # Queried — ModelQuery by tier
        vendor: google               #   with vendor filter
      - model: llama3                # Named — cross-backend
        command: ollama run           #   with CLI command override
    command: claude                   # default CLI command
    working-dir: /workspace/reviews
    pool:
      min-active: 0
      max-active: 5
```

Parsing rules:
- String entry → `Named(modelRef)`
- Map with `model` key → `Named(model)`, extract optional `command` override
- Map with `tier` key (and optional `vendor`, `capabilities`, `max-cost-tier`) → `Queried(ModelQuery)`

### AgentPoolDefinition changes

`AgentConfig` gains a `modelChain` field and per-entry command overrides:

```java
public record AgentConfig(String name, String workingDir, WorkingDirPolicy policy,
                           String command, ModelChain modelChain,
                           Map<String, String> entryCommands) {
    // entryCommands: modelRef → command override (for cross-backend CLI)
}
```

### AgentPoolYamlParser changes

New `parseModelChain()` method handles the three YAML forms:

```java
@SuppressWarnings("unchecked")
private ModelChainParseResult parseModelChain(List<Object> chainList) {
    var entries = new ArrayList<ModelChainEntry>();
    var commands = new LinkedHashMap<String, String>();

    for (Object item : chainList) {
        if (item instanceof String s) {
            entries.add(new ModelChainEntry.Named(s));
        } else if (item instanceof Map<?, ?> map) {
            var m = (Map<String, Object>) map;
            if (m.containsKey("model")) {
                var name = (String) m.get("model");
                entries.add(new ModelChainEntry.Named(name));
                if (m.containsKey("command")) {
                    commands.put(name, (String) m.get("command"));
                }
            } else if (m.containsKey("tier")) {
                var builder = ModelQuery.builder()
                        .tier(ModelTier.valueOf(((String) m.get("tier")).toUpperCase()));
                if (m.containsKey("vendor")) builder.vendor((String) m.get("vendor"));
                // ... other ModelQuery fields
                entries.add(new ModelChainEntry.Queried(builder.build()));
            }
        }
    }
    return new ModelChainParseResult(ModelChain.of(entries), Map.copyOf(commands));
}
```

### ClaudonyModelAvailabilityFilter

Implements `ModelAvailabilityFilter` — checks pool capacity and budget:

```java
public class ClaudonyModelAvailabilityFilter implements ModelAvailabilityFilter {

    private final AgentSessionManager sessionManager;
    private final BudgetTracker budgetTracker;
    private final String poolName;

    public ClaudonyModelAvailabilityFilter(AgentSessionManager sessionManager,
                                           BudgetTracker budgetTracker,
                                           String poolName) {
        this.sessionManager = sessionManager;
        this.budgetTracker = budgetTracker;
        this.poolName = poolName;
    }

    @Override
    public boolean isAvailable(ModelDescriptor descriptor) {
        // Pool capacity check
        if (sessionManager.activeCount() >= sessionManager.status().max()) {
            return false;
        }

        // Budget check
        if (budgetTracker.isBudgetExceeded(poolName)) {
            return false;
        }

        return true;
    }
}

// Factory — creates pool-scoped filters
@ApplicationScoped
public class ClaudonyModelAvailabilityFilterFactory {

    private final AgentPoolManagerRegistry poolRegistry;
    private final BudgetTracker budgetTracker;

    public ModelAvailabilityFilter forPool(String poolName) {
        var mgr = poolRegistry.get(poolName).orElse(null);
        if (mgr == null) return ModelAvailabilityFilter.ALWAYS_AVAILABLE;
        return new ClaudonyModelAvailabilityFilter(mgr, budgetTracker, poolName);
    }
}
```

### CLI path — chain resolution before command building

`ClaudonyWorkerProvisioner` (or `ClaudonyAgentBackend.openWorkerSession()`) resolves the chain before building the CLI command:

```java
private CliResolvedModel resolveForCli(AgentPoolDefinition def) {
    var chain = def.agent().modelChain();
    if (chain == null || chain.isEmpty()) {
        return new CliResolvedModel(null, def.agent().command());
    }

    for (var entry : chain.entries()) {
        String modelRef = switch (entry) {
            case ModelChainEntry.Named n -> n.modelRef();
            case ModelChainEntry.Queried q -> {
                // Resolve query to a specific model via ModelRegistry
                var candidates = modelRegistry.query(q.query());
                yield candidates.isEmpty() ? null : candidates.get(0).apiModelId();
            }
        };

        if (modelRef == null) continue;

        // Operational availability check
        var descriptor = modelRegistry.resolveById(modelRef).orElse(null);
        if (descriptor != null && !availabilityFilter.isAvailable(descriptor)) {
            continue;
        }

        // Resolve command (entry override → pool default)
        var command = def.agent().entryCommands().getOrDefault(modelRef, def.agent().command());
        return new CliResolvedModel(modelRef, command);
    }

    throw new ModelChainExhaustedException(chain, chain.entries());
}

private record CliResolvedModel(String model, String command) {}
```

The resolved model is passed to `WorkerCommandBuilder.build()` as the model parameter. The resolved command replaces the pool's default command if overridden.

### Runtime fallback — circuit breaker

For CLI sessions, runtime fallback detects early session failure and retries:

```java
public TmuxAgentSession openWorkerSessionWithFallback(
        String identity, String workingDir, AgentPoolDefinition def) {

    var chain = def.agent().modelChain();
    if (chain == null || chain.isEmpty()) {
        return openWorkerSession(identity, workingDir, def.agent().command());
    }

    for (var entry : chain.entries()) {
        var resolved = resolveForCli(entry, def);
        if (resolved == null) continue;

        try {
            var session = openWorkerSession(identity, workingDir,
                    buildCommand(resolved.command(), resolved.model()));

            // Watch for early exit (grace period + non-zero exit code)
            // Zero exit code within grace period = fast completion (not a failure)
            // Non-zero exit code within grace period = likely model/API failure (retry)
            // Session survives past grace period = running normally (no retry)
            if (waitForEarlyExit(session, GRACE_PERIOD) && exitCodeNonZero(session)) {
                LOG.warnf("Session exited within grace period (%s) with non-zero exit and model %s, trying next",
                          GRACE_PERIOD, resolved.model());
                sessionManager.destroySession(session.instanceId());
                continue;
            }

            return session;
        } catch (Exception e) {
            LOG.warnf("Failed to create session with model %s: %s", resolved.model(), e.getMessage());
        }
    }

    throw new ModelChainExhaustedException(chain, chain.entries());
}

private static final Duration GRACE_PERIOD = Duration.ofSeconds(30);
```

For programmatic sessions, runtime fallback wraps `AgentBackend.invoke()`:

```java
// In RoutingAgentProvider — runtime retry on invoke()
@Override
public Multi<AgentEvent> invoke(AgentSessionConfig config) {
    if (config.modelChain() == null || config.modelChain().isEmpty()) {
        var route = resolveFromConfig(config);
        return route.backend().invoke(rewrite(config, route));
    }

    // Chain with runtime retry
    return resolveAndInvokeWithRetry(config);
}

private Multi<AgentEvent> resolveAndInvokeWithRetry(AgentSessionConfig config) {
    var entries = new ArrayList<>(config.modelChain().entries());

    return Multi.createFrom().emitter(em -> {
        tryNextEntry(entries, 0, config, em);
    });
}
```

### Degraded provisioning notification

When fallback occurs, the resolved model is recorded in `ProvisionResult` metadata:

```java
// In ClaudonyWorkerProvisioner
var result = ProvisionResult.success(sessionId);
if (resolvedModel.wasFallback()) {
    result = result.withMetadata(Map.of(
        "requestedModel", chain.entries().get(0).toString(),
        "resolvedModel", resolvedModel.model(),
        "fallbackDepth", String.valueOf(resolvedModel.attemptedEntries())
    ));
}
```

The engine can inspect this metadata to adjust worker expectations.

---

## YAML schema extension

`AgentPoolSchema` gains:

```java
inputs.put("model-chain", new StepParameter(
        StepParameterType.LIST, false, null, null, null,
        "Ordered model fallback chain (string or structured entries)"));
```

Validation: `model-chain` and `command`'s `--model` flag are mutually exclusive intent — if `model-chain` is present, the model is resolved from the chain, not from the command string. The pool's `command` field provides the default CLI binary; per-entry `command` overrides it for cross-backend entries.

---

## Testing strategy

### Platform tests

- `ModelChainTest` — construction, `of()` factory, empty chain, immutability
- `RoutingAgentProviderChainTest` — resolution fallback (first entry not in registry → second resolves), operational filter rejects first → second accepted, all entries exhausted → `ModelChainExhaustedException`, mixed Named/Queried entries, single-entry chain (no fallback), `modelChain` takes precedence over `model`/`modelQuery`
- `ModelAvailabilityFilterTest` — `ALWAYS_AVAILABLE` default, filter rejects → skip, filter accepts → use

### Claudony tests

- `AgentPoolYamlParserChainTest` — string entries, structured entries with model key, ModelQuery entries with tier, cross-backend with command override, mixed list, empty chain, invalid entries
- `ClaudonyModelAvailabilityFilterTest` — pool at capacity → unavailable, budget exceeded → unavailable, pool available → available, no pools → unavailable
- `CliChainResolutionTest` — same-backend chain, cross-backend with command override, Queried entry resolved via ModelRegistry, all entries exhausted, operational filter applied
- `RuntimeFallbackTest` — early exit within grace period triggers retry, successful session not retried, all entries fail → exception, grace period timeout (session survives) → no retry
- `DegradedProvisioningTest` — fallback records metadata, no fallback → no metadata

### Integration tests

- `FleetPoolChainIntegrationTest` — real tmux, full chain resolution → session creation → model in command verified
- E2E: pool YAML with model-chain → acquire session → verify resolved model in session command

---

## References

- casehubio/claudony#212 — feat: model fallback chains for agent routing
- casehubio/claudony#205 — agent pool management spec (deferred model fallback to #212)
- `RoutingAgentProvider.java` — existing resolve()/resolveQuery()/resolveTier() chain
- `ModelRegistry` interface — resolveById(), query(), all()
- `ModelQuery` / `ModelDescriptor` / `ModelTier` — platform model types
- `AgentPoolYamlParser.java` — existing YAML parsing patterns
- `AgentSessionManager.java` — pool capacity management
- `BudgetTracker` — pool budget enforcement
- `ClaudonyWorkerExecutionManager` — session exit detection
- Decision review `issue-212-model-fallback-chains-decision-20261003-050900` — R1-01/R1-02/R1-11 circularity finding drove D3/D6 revision
