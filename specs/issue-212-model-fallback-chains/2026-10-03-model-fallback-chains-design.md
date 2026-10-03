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
casehub-platform-api                casehub-platform-agent-api
┌──────────────────────────┐        ┌─────────────────────────────────┐
│ ModelChain                │        │ AgentSessionInit                │
│ ModelChainEntry (sealed)  │        │   + modelChain()                │
│   Named(String)           │        │ AgentSessionConfig              │
│   Queried(ModelQuery)     │        │   + modelChain()                │
│                           │        └─────────────────────────────────┘
│ ModelAvailabilityFilter   │
│   isAvailable(descriptor) │        casehub-platform-agent-router-core
│                           │        ┌─────────────────────────────────┐
│ ChainResolutionResult     │        │ RoutingAgentProvider            │
│   resolvedModel           │───────>│   resolveChain()               │
│   originalEntry           │        │   resolveFromConfig()           │
│   attemptedEntries        │        │   resolveFromInit()             │
│                           │        │                                 │
│ (alongside ModelQuery,    │        │ invoke() / openSession()        │
│  ModelDescriptor,         │        │   checks modelChain() first    │
│  ModelRegistry, ModelTier)│        │                                 │
└──────────────────────────┘        │ ModelChainExhaustedException    │
                                     └─────────────────────────────────┘

                                     claudony-casehub
                                     ┌─────────────────────────────────┐
                                     │ AgentPoolYamlParser             │
                                     │   parses model-chain → ModelChain│
                                     │   validates: no dupes, non-empty│
                                     │                                 │
                                     │ AgentPoolDefinition.AgentConfig │
                                     │   + modelChain field            │
                                     │                                 │
                                     │ CLI chain resolution            │
                                     │   resolveForCli() + per-entry   │
                                     │                                 │
                                     │ Runtime circuit breaker         │
                                     │   early-exit detection + retry  │
                                     │                                 │
                                     │ ModelFallbackEvent (CDI)        │
                                     │   fired on degraded provisioning│
                                     └─────────────────────────────────┘
```

---

## Platform changes

### `casehub-platform-api`

All chain types live in `casehub-platform-api` alongside `ModelQuery`, `ModelDescriptor`, `ModelRegistry`, and `ModelTier` — same Maven module, same `io.casehub.platform.api.model` package. No split packages.

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
        List<ModelChainEntry> attemptedEntries) {

    public boolean wasFallback() {
        return attemptedEntries.size() > 1;
    }

    public static ChainResolutionResult direct(ModelDescriptor model, ModelChainEntry entry) {
        return new ChainResolutionResult(model, entry, List.of(entry));
    }
}
```

#### AgentSessionInit / AgentSessionConfig additions

Both records gain a `modelChain` field (nullable). Existing constructors default to `null`. Three-way mutual exclusion: each `with*` method nulls the other two selection fields.

```java
// In AgentSessionInit
public record AgentSessionInit(
        String systemPrompt,
        List<AgentMcpServer> mcpServers,
        Duration timeout,
        String correlationId,
        String model,
        ModelQuery modelQuery,
        ModelChain modelChain) {           // new field

    // Existing constructors default modelChain to null
    public AgentSessionInit(String systemPrompt, List<AgentMcpServer> mcpServers,
                            Duration timeout, String correlationId, String model) {
        this(systemPrompt, mcpServers, timeout, correlationId, model, null, null);
    }

    public AgentSessionInit withModel(String model) {
        return new AgentSessionInit(systemPrompt, mcpServers, timeout,
                                    correlationId, model, null, null);
    }

    public AgentSessionInit withModel(ModelQuery modelQuery) {
        return new AgentSessionInit(systemPrompt, mcpServers, timeout,
                                    correlationId, null, modelQuery, null);
    }

    public AgentSessionInit withModelChain(ModelChain chain) {
        return new AgentSessionInit(systemPrompt, mcpServers, timeout,
                                    correlationId, null, null, chain);
    }
}
```

Same pattern applied to `AgentSessionConfig`. Each `with*` method zeroes the other two selection fields, enforcing mutual exclusion at every call site.

> **Follow-up:** A sealed `ModelSelection` type (`Named | Queried | Chained`) would collapse the three nullable fields into a single non-null discriminated union — better long-term design, but a broader refactoring. Filed as casehubio/platform#TBD.

### `casehub-platform-agent-router-core`

#### RoutingAgentProvider — new chain resolution

```java
// New method — iterates chain, returns first resolvable + available entry
// Only catches IllegalArgumentException (model not in catalog).
// IllegalStateException (backend infrastructure broken) propagates immediately —
// falling through to the next entry won't help if the backend itself is misconfigured.
private ResolvedRoute resolveChain(ModelChain chain, ModelAvailabilityFilter filter) {
    List<ModelChainEntry> attempted = new ArrayList<>();
    for (var entry : chain.entries()) {
        attempted.add(entry);
        try {
            ResolvedRoute route = switch (entry) {
                case ModelChainEntry.Named n -> resolve(n.modelRef());
                case ModelChainEntry.Queried q -> resolveQuery(q.query());
            };

            // Resolution succeeded — check model-specific availability
            if (route.apiModelId() == null) return route;  // backend-key resolution, no filter
            var descriptor = modelRegistry.resolveById(route.apiModelId());
            if (descriptor.isPresent() && !filter.isAvailable(descriptor.get())) {
                LOG.debugf("Chain entry %s resolved but unavailable, trying next", entry);
                continue;
            }

            LOG.infof("Chain resolved to %s after %d attempt(s) (fallback=%s)",
                       route.apiModelId(), attempted.size(), attempted.size() > 1);
            return route;
        } catch (IllegalArgumentException e) {
            LOG.debugf("Chain entry %s failed resolution: %s", entry, e.getMessage());
        }
    }
    throw new ModelChainExhaustedException(chain, attempted);
}
```

#### invoke() / openSession() integration

Both `invoke()` and `openSession()` check `modelChain` first via parallel resolution helpers:

```java
@Override
public Multi<AgentEvent> invoke(AgentSessionConfig config) {
    if (config.modelChain() != null && !config.modelChain().isEmpty()) {
        return resolveAndInvokeWithRetry(config);
    }
    var route = resolveFromConfig(config);
    var rewritten = new AgentSessionConfig(
            config.systemPrompt(), config.userPrompt(), config.mcpServers(),
            config.timeout(), config.correlationId(), route.apiModelId());
    return route.backend().invoke(rewritten);
}

@Override
public AgentSession openSession(AgentSessionInit init) {
    var route = resolveFromInit(init);
    var rewritten = new AgentSessionInit(
            init.systemPrompt(), init.mcpServers(),
            init.timeout(), init.correlationId(), route.apiModelId());
    return route.backend().openSession(rewritten);
}

private ResolvedRoute resolveFromConfig(AgentSessionConfig config) {
    if (config.modelChain() != null && !config.modelChain().isEmpty()) {
        return resolveChain(config.modelChain(), availabilityFilter);
    }
    return config.modelQuery() != null
           ? resolveQuery(config.modelQuery())
           : resolve(config.model());
}

private ResolvedRoute resolveFromInit(AgentSessionInit init) {
    if (init.modelChain() != null && !init.modelChain().isEmpty()) {
        return resolveChain(init.modelChain(), availabilityFilter);
    }
    return init.modelQuery() != null
           ? resolveQuery(init.modelQuery())
           : resolve(init.model());
}
```

#### Runtime retry for programmatic invoke()

`resolveAndInvokeWithRetry()` builds a Mutiny fallback chain from last entry to first. Each entry's `Multi<AgentEvent>` stream falls through to the next on retryable errors (resolution failures, API errors). Non-retryable failures (backend infrastructure broken) propagate immediately.

```java
private Multi<AgentEvent> resolveAndInvokeWithRetry(AgentSessionConfig config) {
    var entries = config.modelChain().entries();

    // Seed: exhaustion error if all entries fail
    Multi<AgentEvent> chain = Multi.createFrom().failure(
        new ModelChainExhaustedException(config.modelChain(), entries));

    // Build from last to first — first entry is attempted first
    for (int i = entries.size() - 1; i >= 0; i--) {
        var entry = entries.get(i);
        final Multi<AgentEvent> fallback = chain;
        chain = attemptInvoke(entry, config)
            .onFailure(ModelChainRetryableException.class)
            .recoverWithMulti(fallback);
    }
    return chain;
}

private Multi<AgentEvent> attemptInvoke(ModelChainEntry entry, AgentSessionConfig config) {
    ResolvedRoute route;
    try {
        route = switch (entry) {
            case ModelChainEntry.Named n -> resolve(n.modelRef());
            case ModelChainEntry.Queried q -> resolveQuery(q.query());
        };
    } catch (IllegalArgumentException e) {
        return Multi.createFrom().failure(new ModelChainRetryableException(entry, e));
    }
    // IllegalStateException (backend broken) is NOT caught — propagates immediately

    var rewritten = new AgentSessionConfig(
            config.systemPrompt(), config.userPrompt(), config.mcpServers(),
            config.timeout(), config.correlationId(), route.apiModelId());
    return route.backend().invoke(rewritten)
        .onFailure(this::isRetryableApiError)
        .recoverWithMulti(err -> Multi.createFrom().failure(
            new ModelChainRetryableException(entry, err)));
}

private boolean isRetryableApiError(Throwable t) {
    // API 429, transient auth errors, connection refused — retryable
    // NPE, ClassCast, IllegalState — not retryable
    return t instanceof io.casehub.platform.agent.AgentApiException
        || t instanceof java.net.ConnectException;
}
```

`ModelChainRetryableException` wraps errors that should trigger fallback. Non-retryable errors (infrastructure broken, programming errors) propagate through the Mutiny stream without triggering `recoverWithMulti`.

> **Limitation:** If a backend starts streaming events and then errors mid-stream, the partial events are already downstream. Runtime retry only covers pre-output errors (connection refused, immediate 429, auth failures) cleanly. Mid-stream recovery is out of scope — it would require buffering and replay, which is a separate design concern.

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

#### ModelChainExhaustedException / ModelChainRetryableException

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

// Internal to router — wraps errors that should trigger fallback to next chain entry
class ModelChainRetryableException extends RuntimeException {
    private final ModelChainEntry failedEntry;

    ModelChainRetryableException(ModelChainEntry entry, Throwable cause) {
        super("Chain entry %s failed: %s".formatted(entry, cause.getMessage()), cause);
        this.failedEntry = entry;
    }

    ModelChainEntry failedEntry() { return failedEntry; }
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

`AgentConfig` gains `modelChain`, per-entry command overrides, and per-entry grace periods:

```java
public record AgentConfig(String name, String workingDir, WorkingDirPolicy policy,
                           String command, ModelChain modelChain,
                           Map<String, String> entryCommands,
                           Map<String, Duration> gracePeriods) {
    public AgentConfig {
        Objects.requireNonNull(name, "agent name is required");
        if (name.isBlank()) throw new IllegalArgumentException("agent name must not be blank");
        if (policy == null) policy = WorkingDirPolicy.EXCLUSIVE;
        if (entryCommands == null) entryCommands = Map.of();
        if (gracePeriods == null) gracePeriods = Map.of();
    }
    // entryCommands: modelRef → command override (for cross-backend CLI)
    // gracePeriods: modelRef → grace period override (for slow-starting models)
}
```

This breaks the canonical constructor (4 fields → 7 fields). The builder shields production callers; test call sites need updating — see §Breakage acknowledgment.

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
    // Structural validation — warn on duplicate Named entries, reject empty chains
    var namedRefs = entries.stream()
        .filter(ModelChainEntry.Named.class::isInstance)
        .map(e -> ((ModelChainEntry.Named) e).modelRef())
        .toList();
    var seen = new HashSet<String>();
    for (var ref : namedRefs) {
        if (!seen.add(ref)) {
            LOG.warnf("Duplicate model '%s' in chain for pool — same model tried twice is likely misconfiguration", ref);
        }
    }

    return new ModelChainParseResult(ModelChain.of(entries), Map.copyOf(commands));
}
```

### Pool-level pre-conditions

Pool capacity and budget are pool-level constraints — they apply uniformly to all models in a chain. Checking them inside the chain loop is wasteful (every entry gets the same answer) and produces misleading errors ("chain exhausted" when the real problem is "pool full"). These checks happen before chain resolution:

```java
// In ClaudonyWorkerProvisioner.setupSession(), before resolveForCli()
if (sessionManager.activeCount() >= sessionManager.status().max()) {
    throw new PoolAtCapacityException(poolName);
}
if (budgetTracker.isBudgetExceeded(poolName)) {
    throw new BudgetExceededException(poolName);
}
```

### ModelAvailabilityFilter — model-specific availability

The platform SPI `ModelAvailabilityFilter` is reserved for model-specific availability checks (per-model rate limiting, per-model budget allocation). For V1, no model-specific availability tracking exists in Claudony — the injected filter defaults to `ALWAYS_AVAILABLE`. The SPI parameter `ModelDescriptor descriptor` is intentionally there for future implementations that differentiate by model.

```java
// V1: no model-specific availability checks in Claudony
// RoutingAgentProvider receives ModelAvailabilityFilter.ALWAYS_AVAILABLE
// Pool-level guards (capacity, budget) are pre-conditions — see above
```

> **Future:** When per-model rate limit tracking or per-model budget allocation is added, a `ClaudonyModelAvailabilityFilter` implementation would check `descriptor.apiModelId()` against per-model counters. The SPI is ready; the implementation is deferred until per-model tracking exists.

### CLI path — chain resolution before command building

Two methods with distinct responsibilities:
- `resolveForCli(AgentPoolDefinition)` — full-chain resolution, returns first available entry
- `resolveCliEntry(ModelChainEntry, AgentPoolDefinition)` — per-entry resolution, used by the runtime fallback loop

```java
// Full-chain resolution — used by setupSession() for initial provisioning
private CliResolvedModel resolveForCli(AgentPoolDefinition def) {
    var chain = def.agent().modelChain();
    if (chain == null || chain.isEmpty()) {
        return new CliResolvedModel(null, def.agent().command());
    }

    for (var entry : chain.entries()) {
        var resolved = resolveCliEntry(entry, def);
        if (resolved != null) return resolved;
    }

    throw new ModelChainExhaustedException(chain, chain.entries());
}

// Per-entry resolution — extracted for reuse by runtime fallback
private CliResolvedModel resolveCliEntry(ModelChainEntry entry, AgentPoolDefinition def) {
    String modelRef = switch (entry) {
        case ModelChainEntry.Named n -> n.modelRef();
        case ModelChainEntry.Queried q -> {
            var candidates = modelRegistry.query(q.query());
            yield candidates.isEmpty() ? null : candidates.get(0).apiModelId();
        }
    };

    if (modelRef == null) return null;

    // Resolve command (entry override → pool default)
    var command = def.agent().entryCommands().getOrDefault(modelRef, def.agent().command());
    return new CliResolvedModel(modelRef, command);
}

private record CliResolvedModel(String model, String command) {}
```

#### Integration with ClaudonyWorkerProvisioner.setupSession()

The chain resolution slots into the existing provisioning flow between config lookup and command building. `ClaudonyProviderConfig` gains a `withModel(String)` method to create a copy with the resolved model overriding the config's model. The resolved command overrides the base command.

```java
// In ClaudonyWorkerProvisioner.setupSession() — modified flow
ClaudonyProviderConfig config = providerConfigSource.forAgent(roleName);
AgentPoolDefinition poolDef = poolRegistry.getDefinition(roleName);

// Chain resolution (pool-level guards already passed — see §Pool-level pre-conditions)
CliResolvedModel resolved = resolveForCli(poolDef);

// Resolved command overrides config command; resolved model overrides config model
String baseCommand = resolved.command() != null
    ? resolved.command()
    : config.command().orElse(defaultCommand);
ClaudonyProviderConfig effectiveConfig = resolved.model() != null
    ? config.withModel(resolved.model())
    : config;

Optional<String> meshPrompt = Optional.ofNullable(context.workerContext())
                                      .map(wc -> wc.properties().get("systemPrompt"))
                                      .filter(String.class::isInstance)
                                      .map(String.class::cast);

String enrichedCommand = WorkerCommandBuilder.build(baseCommand, effectiveConfig, meshPrompt);
String effectiveWorkingDir = config.workingDir().orElse(defaultWorkingDir);
TmuxAgentSession agentSession = agentBackend.openWorkerSession(
    roleName, effectiveWorkingDir, enrichedCommand);
```

`ClaudonyProviderConfig.withModel()`:

```java
// In ClaudonyProviderConfig
public ClaudonyProviderConfig withModel(String model) {
    return new ClaudonyProviderConfig(command, Optional.ofNullable(model), appendSystemPrompt,
        systemPrompt, effort, permissionMode, tools, allowedTools, disallowedTools, addDirs, workingDir);
}
```

`WorkerCommandBuilder.build()` reads `config.model()` for `--model` — no changes to `WorkerCommandBuilder` itself. The resolved model flows through the config override.

### Runtime fallback — circuit breaker (CLI)

For CLI sessions, runtime fallback detects early session failure and retries with the next chain entry. Uses `resolveCliEntry()` for per-entry resolution.

Grace period is configurable per chain entry via YAML (`grace-period` key, default 30s). Local models (Ollama) may need longer cold-start times.

```java
public TmuxAgentSession openWorkerSessionWithFallback(
        String identity, String workingDir, AgentPoolDefinition def) {

    var chain = def.agent().modelChain();
    if (chain == null || chain.isEmpty()) {
        return openWorkerSession(identity, workingDir, def.agent().command());
    }

    for (var entry : chain.entries()) {
        var resolved = resolveCliEntry(entry, def);
        if (resolved == null) continue;

        try {
            var session = openWorkerSession(identity, workingDir,
                    buildCommand(resolved.command(), resolved.model()));

            Duration gracePeriod = def.agent().gracePeriods()
                .getOrDefault(resolved.model(), DEFAULT_GRACE_PERIOD);

            // Exit code semantics (Claude CLI):
            //   0  = normal completion (fast task, graceful shutdown — not a failure)
            //   1  = general error (auth failure, invalid config, API error)
            //   2  = usage error (invalid arguments)
            //   >128 = signal (e.g. 139 = segfault — not model-related, don't retry)
            //
            // Non-zero within grace period and exit < 128 = likely model/API failure → retry
            // Session survives past grace period = running normally → no retry
            if (waitForEarlyExit(session, gracePeriod)
                    && exitCodeNonZero(session)
                    && session.exitCode() < 128) {
                LOG.warnf("Session exited within grace period (%s) with exit code %d, model %s — trying next",
                          gracePeriod, session.exitCode(), resolved.model());
                sessionManager.destroySession(session.managedSession().instanceId());
                continue;
            }

            return session;
        } catch (Exception e) {
            LOG.warnf("Failed to create session with model %s: %s", resolved.model(), e.getMessage());
        }
    }

    throw new ModelChainExhaustedException(chain, chain.entries());
}

private static final Duration DEFAULT_GRACE_PERIOD = Duration.ofSeconds(30);
```

Programmatic session runtime retry is handled by `resolveAndInvokeWithRetry()` in `RoutingAgentProvider` — see §Runtime retry for programmatic invoke() above.

### Degraded provisioning notification

`ProvisionResult` is a minimal record in `casehub-engine-api` — `(UUID causedByEntryId, String resolvedWorkerId)`. It has no metadata support and modifying it is a cross-repo engine API change. Instead, fallback notification uses a CDI event:

```java
// CDI event — fired when chain resolution falls back to a non-primary model
public record ModelFallbackEvent(
        String poolName,
        String requestedModel,
        String resolvedModel,
        int fallbackDepth) {}

// In ClaudonyWorkerProvisioner — after chain resolution
@Inject Event<ModelFallbackEvent> fallbackEvent;

// In setupSession(), after resolveForCli()
if (resolved.model() != null && !resolved.model().equals(primaryModel(poolDef))) {
    fallbackEvent.fire(new ModelFallbackEvent(
        roleName,
        poolDef.agent().modelChain().entries().get(0).toString(),
        resolved.model(),
        chainDepth(resolved, poolDef)));
}
```

Any component that needs to act on fallback (adjust prompts, add review gates, log for observability) observes the CDI event. The engine integration can bridge this to `WorkerContext` properties if needed — no changes to `ProvisionResult` or `casehub-engine-api`.

---

## YAML schema extension

`AgentPoolSchema` gains:

```java
inputs.put("model-chain", new StepParameter(
        StepParameterType.LIST, false, null, null, null,
        "Ordered model fallback chain (string or structured entries)"));
```

Extended per-entry YAML for grace period override:

```yaml
model-chain:
  - opus
  - model: llama3
    command: ollama run
    grace-period: 60s     # longer cold-start for local models
```

`AgentConfig` gains `gracePeriods` alongside `entryCommands`:

```java
public record AgentConfig(String name, String workingDir, WorkingDirPolicy policy,
                           String command, ModelChain modelChain,
                           Map<String, String> entryCommands,
                           Map<String, Duration> gracePeriods) {
    public AgentConfig {
        Objects.requireNonNull(name, "agent name is required");
        if (name.isBlank()) throw new IllegalArgumentException("agent name must not be blank");
        if (policy == null) policy = WorkingDirPolicy.EXCLUSIVE;
        if (entryCommands == null) entryCommands = Map.of();
        if (gracePeriods == null) gracePeriods = Map.of();
    }
}
```

Validation: `model-chain` and `command`'s `--model` flag are mutually exclusive intent — if `model-chain` is present, the model is resolved from the chain, not from the command string. The pool's `command` field provides the default CLI binary; per-entry `command` overrides it for cross-backend entries.

---

## Testing strategy

### Platform tests (`casehub-platform-api`)

- `ModelChainTest` — construction, `of()` factory, empty chain, immutability, `wasFallback()` derived method
- `ChainResolutionResultTest` — `direct()` factory gives `wasFallback()=false`, multi-entry gives `wasFallback()=true`

### Platform tests (`casehub-platform-agent-router-core`)

- `RoutingAgentProviderChainTest` — resolution fallback (first entry not in registry → second resolves), model-specific filter rejects first → second accepted, all entries exhausted → `ModelChainExhaustedException`, mixed Named/Queried entries, single-entry chain (no fallback), `modelChain` takes precedence over `model`/`modelQuery`
- `RoutingAgentProviderOpenSessionChainTest` — `openSession()` uses `resolveFromInit()` with chain, mirrors invoke() chain behavior
- `RoutingAgentProviderRetryTest` — `resolveAndInvokeWithRetry()`: first entry fails with retryable error → falls through to second, non-retryable error propagates immediately, all entries retryable-fail → `ModelChainExhaustedException`
- `ModelAvailabilityFilterTest` — `ALWAYS_AVAILABLE` default, filter rejects → skip, filter accepts → use
- `ModelChainExhaustedExceptionTest` — carries chain and attempted entries

### Claudony tests

- `AgentPoolYamlParserChainTest` — string entries, structured entries with model key, ModelQuery entries with tier, cross-backend with command override, grace-period override, mixed list, empty chain, invalid entries, **duplicate Named entries → warning logged**
- `CliChainResolutionTest` — same-backend chain, cross-backend with command override, Queried entry resolved via ModelRegistry, all entries exhausted
- `ClaudonyWorkerProvisionerChainTest` — setupSession() with chain: resolved model appears in enriched command, resolved command overrides base command, pool-level guards reject before chain resolution
- `RuntimeFallbackTest` — early exit within grace period triggers retry, successful session not retried, all entries fail → exception, grace period timeout (session survives) → no retry, exit code ≥128 (signal) skips retry, configurable grace period per entry
- `ModelFallbackEventTest` — fallback fires CDI event with correct metadata, no fallback → no event

### Integration tests

- `FleetPoolChainIntegrationTest` — real tmux, full chain resolution → session creation → model in command verified
- E2E: pool YAML with model-chain → acquire session → verify resolved model in session command

### Breakage acknowledgment

Adding `modelChain`, `entryCommands`, and `gracePeriods` to `AgentConfig` breaks its canonical constructor. The builder shields `AgentPoolYamlParser` and other production callers. Tests constructing `AgentConfig` directly (`AgentPoolDefinitionTest` and related) need updating — this is expected mechanical breakage per the design philosophy.

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
