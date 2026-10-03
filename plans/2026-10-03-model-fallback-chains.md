# Model Fallback Chains Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #212 — feat: model fallback chains for agent routing
**Issue group:** #212

**Goal:** Add ordered model fallback chains to the platform routing layer and Claudony pool management, enabling automatic degradation when a preferred model is unavailable.

**Architecture:** New `ModelChain` / `ModelChainEntry` types in `casehub-platform-api`. `RoutingAgentProvider` gains `resolveChain()` for iterative resolution with `ModelAvailabilityFilter` SPI. Claudony parses chains from pool YAML, bridges to platform types, and adds CLI-specific resolution with runtime circuit-breaker. Cross-backend chains supported via per-entry command overrides.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny (reactive retry), JUnit 5 + AssertJ, YAML (Jackson)

## Global Constraints

- Java source level: `release=21` (compile on Java 26)
- All new types in `casehub-platform-api` go in package `io.casehub.platform.api.model` (no split packages)
- All new types in `casehub-platform-agent-router-core` go in package `io.casehub.platform.agent.router`
- All new types in `claudony-casehub` go in package `io.casehub.claudony.casehub.fleet`
- Platform repo: `/Users/mdproctor/claude/casehub/slots/202/platform`
- Claudony repo: `/Users/mdproctor/claude/casehub/slots/202/claudony`
- Build platform: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
- Build router: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-router-core -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
- Build claudony: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
- Install platform SNAPSHOT after changes: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`

---

## Batch 1: Platform chain types (`casehub-platform-api`)

### Task 1: ModelChain, ModelChainEntry, ModelAvailabilityFilter, ChainResolutionResult

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/model/ModelChain.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/model/ModelAvailabilityFilter.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/model/ChainResolutionResult.java`
- Test: `platform-api/src/test/java/io/casehub/platform/api/model/ModelChainTest.java`

**Interfaces:**
- Consumes: `ModelQuery`, `ModelDescriptor` (existing platform-api types)
- Produces: `ModelChain`, `ModelChain.ModelChainEntry`, `ModelChain.ModelChainEntry.Named`, `ModelChain.ModelChainEntry.Queried`, `ModelAvailabilityFilter`, `ChainResolutionResult` — used by Task 2 (router), Task 3 (AgentSessionInit/Config), Tasks 4-7 (Claudony)

- [ ] **Step 1: Write ModelChainTest**

```java
package io.casehub.platform.api.model;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ModelChainTest {

    @Test
    void of_varargs_createsNamedEntries() {
        var chain = ModelChain.of("opus", "sonnet", "haiku");
        assertThat(chain.entries()).hasSize(3);
        assertThat(chain.entries().get(0)).isInstanceOf(ModelChain.ModelChainEntry.Named.class);
        assertThat(((ModelChain.ModelChainEntry.Named) chain.entries().get(0)).modelRef()).isEqualTo("opus");
    }

    @Test
    void of_list_createsImmutableCopy() {
        var entries = new java.util.ArrayList<ModelChain.ModelChainEntry>();
        entries.add(new ModelChain.ModelChainEntry.Named("opus"));
        var chain = ModelChain.of(entries);
        entries.clear();
        assertThat(chain.entries()).hasSize(1);
    }

    @Test
    void isEmpty_emptyChain() {
        assertThat(ModelChain.of().isEmpty()).isTrue();
    }

    @Test
    void isEmpty_nonEmptyChain() {
        assertThat(ModelChain.of("opus").isEmpty()).isFalse();
    }

    @Test
    void namedEntry_holdsModelRef() {
        var entry = new ModelChain.ModelChainEntry.Named("claude-opus-4-6");
        assertThat(entry.modelRef()).isEqualTo("claude-opus-4-6");
    }

    @Test
    void queriedEntry_holdsModelQuery() {
        var query = ModelQuery.builder().tier(ModelTier.FLAGSHIP).build();
        var entry = new ModelChain.ModelChainEntry.Queried(query);
        assertThat(entry.query().tier()).isEqualTo(ModelTier.FLAGSHIP);
    }

    @Test
    void sealedInterface_switchCoversAllCases() {
        ModelChain.ModelChainEntry entry = new ModelChain.ModelChainEntry.Named("opus");
        String result = switch (entry) {
            case ModelChain.ModelChainEntry.Named n -> n.modelRef();
            case ModelChain.ModelChainEntry.Queried q -> q.query().toString();
        };
        assertThat(result).isEqualTo("opus");
    }

    @Test
    void chainResolutionResult_wasFallback_singleEntry() {
        var entry = new ModelChain.ModelChainEntry.Named("opus");
        var result = ChainResolutionResult.direct(null, entry);
        assertThat(result.wasFallback()).isFalse();
    }

    @Test
    void chainResolutionResult_wasFallback_multipleEntries() {
        var e1 = new ModelChain.ModelChainEntry.Named("opus");
        var e2 = new ModelChain.ModelChainEntry.Named("sonnet");
        var result = new ChainResolutionResult(null, e2, java.util.List.of(e1, e2));
        assertThat(result.wasFallback()).isTrue();
    }

    @Test
    void availabilityFilter_alwaysAvailable() {
        assertThat(ModelAvailabilityFilter.ALWAYS_AVAILABLE.isAvailable(null)).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -Dtest=ModelChainTest -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
Expected: FAIL — `ModelChain` class not found

- [ ] **Step 3: Create ModelChain.java**

```java
package io.casehub.platform.api.model;

import java.util.Arrays;
import java.util.List;

public record ModelChain(List<ModelChainEntry> entries) {

    public ModelChain {
        entries = List.copyOf(entries);
    }

    public static ModelChain of(String... modelRefs) {
        return new ModelChain(
            Arrays.stream(modelRefs)
                  .map(ModelChainEntry.Named::new)
                  .<ModelChainEntry>map(e -> e)
                  .toList());
    }

    public static ModelChain of(List<ModelChainEntry> entries) {
        return new ModelChain(entries);
    }

    public boolean isEmpty() { return entries.isEmpty(); }

    public sealed interface ModelChainEntry {
        record Named(String modelRef) implements ModelChainEntry {}
        record Queried(ModelQuery query) implements ModelChainEntry {}
    }
}
```

- [ ] **Step 4: Create ModelAvailabilityFilter.java**

```java
package io.casehub.platform.api.model;

@FunctionalInterface
public interface ModelAvailabilityFilter {
    boolean isAvailable(ModelDescriptor descriptor);

    ModelAvailabilityFilter ALWAYS_AVAILABLE = descriptor -> true;
}
```

- [ ] **Step 5: Create ChainResolutionResult.java**

```java
package io.casehub.platform.api.model;

import java.util.List;

public record ChainResolutionResult(
        ModelDescriptor resolvedModel,
        ModelChain.ModelChainEntry originalEntry,
        List<ModelChain.ModelChainEntry> attemptedEntries) {

    public boolean wasFallback() {
        return attemptedEntries.size() > 1;
    }

    public static ChainResolutionResult direct(ModelDescriptor model, ModelChain.ModelChainEntry entry) {
        return new ChainResolutionResult(model, entry, List.of(entry));
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api -Dtest=ModelChainTest -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
Expected: 10 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/202/platform add platform-api/src/main/java/io/casehub/platform/api/model/ModelChain.java platform-api/src/main/java/io/casehub/platform/api/model/ModelAvailabilityFilter.java platform-api/src/main/java/io/casehub/platform/api/model/ChainResolutionResult.java platform-api/src/test/java/io/casehub/platform/api/model/ModelChainTest.java
git -C /Users/mdproctor/claude/casehub/slots/202/platform commit -m "feat(model): add ModelChain, ModelAvailabilityFilter, ChainResolutionResult types Refs casehubio/claudony#212"
```

---

## Batch 2: Platform router chain resolution (`casehub-platform-agent-router-core` + `agent-api`)

### Task 2: AgentSessionInit and AgentSessionConfig gain modelChain field

**Files:**
- Modify: `agent-api/src/main/java/io/casehub/platform/agent/AgentSessionInit.java`
- Modify: `agent-api/src/main/java/io/casehub/platform/agent/AgentSessionConfig.java`
- Test: `agent-api/src/test/java/io/casehub/platform/agent/AgentSessionInitChainTest.java`

**Interfaces:**
- Consumes: `ModelChain` (from Task 1)
- Produces: `AgentSessionInit.modelChain()`, `AgentSessionConfig.modelChain()`, `withModelChain(ModelChain)` — used by Task 3 (router resolution)

- [ ] **Step 1: Write AgentSessionInitChainTest**

```java
package io.casehub.platform.agent;

import io.casehub.platform.api.model.ModelChain;
import io.casehub.platform.api.model.ModelQuery;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentSessionInitChainTest {

    @Test
    void existingConstructor_modelChainIsNull() {
        var init = AgentSessionInit.of("prompt");
        assertThat(init.modelChain()).isNull();
    }

    @Test
    void withModelChain_setsChainAndNullsModelAndQuery() {
        var init = AgentSessionInit.of("prompt", "opus");
        var chain = ModelChain.of("opus", "sonnet");
        var chained = init.withModelChain(chain);
        assertThat(chained.modelChain()).isEqualTo(chain);
        assertThat(chained.model()).isNull();
        assertThat(chained.modelQuery()).isNull();
    }

    @Test
    void withModel_string_nullsModelChain() {
        var chain = ModelChain.of("opus", "sonnet");
        var init = new AgentSessionInit("prompt", java.util.List.of(), null, null, null, null, chain);
        var rewritten = init.withModel("opus");
        assertThat(rewritten.model()).isEqualTo("opus");
        assertThat(rewritten.modelChain()).isNull();
        assertThat(rewritten.modelQuery()).isNull();
    }

    @Test
    void withModel_query_nullsModelChain() {
        var chain = ModelChain.of("opus", "sonnet");
        var init = new AgentSessionInit("prompt", java.util.List.of(), null, null, null, null, chain);
        var query = ModelQuery.builder().build();
        var rewritten = init.withModel(query);
        assertThat(rewritten.modelQuery()).isEqualTo(query);
        assertThat(rewritten.model()).isNull();
        assertThat(rewritten.modelChain()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-api -Dtest=AgentSessionInitChainTest -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
Expected: FAIL — no 7-arg constructor, no `modelChain()` accessor

- [ ] **Step 3: Modify AgentSessionInit.java — add modelChain field**

Add `modelChain` as the 7th record component. Update all constructors and `with*` methods to enforce three-way mutual exclusion. Keep backward compatibility:

```java
package io.casehub.platform.agent;

import io.casehub.platform.api.model.ModelChain;
import java.time.Duration;
import java.util.List;
import java.util.Objects;

public record AgentSessionInit(
        String systemPrompt,
        List<AgentMcpServer> mcpServers,
        Duration timeout,
        String correlationId,
        String model,
        io.casehub.platform.api.model.ModelQuery modelQuery,
        ModelChain modelChain
) {
    public AgentSessionInit {
        Objects.requireNonNull(systemPrompt, "systemPrompt");
        mcpServers = mcpServers != null ? List.copyOf(mcpServers) : List.of();
    }

    public AgentSessionInit(String systemPrompt, List<AgentMcpServer> mcpServers,
                            Duration timeout, String correlationId, String model,
                            io.casehub.platform.api.model.ModelQuery modelQuery) {
        this(systemPrompt, mcpServers, timeout, correlationId, model, modelQuery, null);
    }

    public AgentSessionInit(String systemPrompt, List<AgentMcpServer> mcpServers,
                            Duration timeout, String correlationId, String model) {
        this(systemPrompt, mcpServers, timeout, correlationId, model, null, null);
    }

    public static AgentSessionInit of(String systemPrompt) {
        return new AgentSessionInit(systemPrompt, List.of(), null, null, null, null, null);
    }

    public static AgentSessionInit of(String systemPrompt, String model) {
        return new AgentSessionInit(systemPrompt, List.of(), null, null, model, null, null);
    }

    public AgentSessionInit withModel(String model) {
        return new AgentSessionInit(systemPrompt, mcpServers, timeout, correlationId, model, null, null);
    }

    public AgentSessionInit withModel(io.casehub.platform.api.model.ModelQuery modelQuery) {
        return new AgentSessionInit(systemPrompt, mcpServers, timeout, correlationId, null, modelQuery, null);
    }

    public AgentSessionInit withModelChain(ModelChain chain) {
        return new AgentSessionInit(systemPrompt, mcpServers, timeout, correlationId, null, null, chain);
    }
}
```

- [ ] **Step 4: Modify AgentSessionConfig.java — same pattern**

Add `modelChain` as the 8th record component with identical mutual exclusion pattern. All constructors default `modelChain` to `null`. Add `withModelChain(ModelChain)`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-api -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
Expected: All existing tests + 4 new tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/202/platform add agent-api/
git -C /Users/mdproctor/claude/casehub/slots/202/platform commit -m "feat(agent-api): add modelChain field to AgentSessionInit and AgentSessionConfig Refs casehubio/claudony#212"
```

### Task 3: RoutingAgentProvider.resolveChain() and chain-aware invoke/openSession

**Files:**
- Modify: `agent-router-core/src/main/java/io/casehub/platform/agent/router/RoutingAgentProvider.java`
- Create: `agent-router-core/src/main/java/io/casehub/platform/agent/router/ModelChainExhaustedException.java`
- Create: `agent-router-core/src/main/java/io/casehub/platform/agent/router/ModelChainRetryableException.java`
- Test: `agent-router-core/src/test/java/io/casehub/platform/agent/router/RoutingAgentProviderChainTest.java`

**Interfaces:**
- Consumes: `ModelChain`, `ModelChainEntry`, `ModelAvailabilityFilter` (Task 1), `AgentSessionInit.modelChain()` / `AgentSessionConfig.modelChain()` (Task 2)
- Produces: `RoutingAgentProvider.resolveChain()`, `ModelChainExhaustedException`, `ModelChainRetryableException` — used by Tasks 4-7 (Claudony integration)

- [ ] **Step 1: Write RoutingAgentProviderChainTest**

```java
package io.casehub.platform.agent.router;

import io.casehub.platform.agent.*;
import io.casehub.platform.api.model.*;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Optional;
import java.util.Set;

import static org.assertj.core.api.Assertions.*;

class RoutingAgentProviderChainTest {

    private final StubBackend claudeBackend = new StubBackend("claude");
    private final StubBackend geminiBackend = new StubBackend("gemini");

    private ModelDescriptor descriptor(String id, String backendKey) {
        return new ModelDescriptor(id, id, backendKey, "default",
                "anthropic", "claude", id, ModelTier.FLAGSHIP,
                Set.of(), 200000, 16000, ModelLocality.CLOUD, CostTier.HIGH, null, java.util.Map.of());
    }

    private ModelRegistry registryWith(ModelDescriptor... descriptors) {
        return new ModelRegistry() {
            private final List<ModelDescriptor> all = List.of(descriptors);
            @Override public Optional<ModelDescriptor> resolveById(String id) {
                return all.stream().filter(d -> d.id().equals(id)).findFirst();
            }
            @Override public List<ModelDescriptor> query(ModelQuery query) {
                return all.stream().filter(d -> query.tier() == null || d.tier() == query.tier()).toList();
            }
            @Override public List<ModelDescriptor> all() { return all; }
        };
    }

    private BackendInstanceRegistry backendRegistry(AgentBackend... backends) {
        return new BackendInstanceRegistry() {
            @Override public void register(AgentBackend b) {}
            @Override public Optional<AgentBackend> resolve(String key, String instanceId) {
                for (var b : backends) {if (b.key().equals(key)) return Optional.of(b);}
                return Optional.empty();
            }
            @Override public List<AgentBackend> resolveByKey(String key) {
                return java.util.Arrays.stream(backends).filter(b -> b.key().equals(key)).toList();
            }
        };
    }

    @Test
    void resolveChain_firstEntryResolves() {
        var registry = registryWith(descriptor("opus", "claude"), descriptor("sonnet", "claude"));
        var provider = new RoutingAgentProvider(backendRegistry(claudeBackend), "claude", registry,
                java.util.Map.of(), ModelAvailabilityFilter.ALWAYS_AVAILABLE);
        var config = AgentSessionConfig.of("sys", "user").withModelChain(ModelChain.of("opus", "sonnet"));
        // Should resolve opus without fallback — invoke doesn't throw
        assertThatCode(() -> provider.invoke(config)).doesNotThrowAnyException();
    }

    @Test
    void resolveChain_firstEntryMissing_fallsToSecond() {
        var registry = registryWith(descriptor("sonnet", "claude"));  // no opus
        var provider = new RoutingAgentProvider(backendRegistry(claudeBackend), "claude", registry,
                java.util.Map.of(), ModelAvailabilityFilter.ALWAYS_AVAILABLE);
        var config = AgentSessionConfig.of("sys", "user").withModelChain(ModelChain.of("opus", "sonnet"));
        assertThatCode(() -> provider.invoke(config)).doesNotThrowAnyException();
    }

    @Test
    void resolveChain_allExhausted_throws() {
        var registry = registryWith();  // empty
        var provider = new RoutingAgentProvider(backendRegistry(claudeBackend), "claude", registry,
                java.util.Map.of(), ModelAvailabilityFilter.ALWAYS_AVAILABLE);
        var config = AgentSessionConfig.of("sys", "user").withModelChain(ModelChain.of("opus", "sonnet"));
        assertThatThrownBy(() -> provider.invoke(config))
                .isInstanceOf(ModelChainExhaustedException.class)
                .hasMessageContaining("2");
    }

    @Test
    void resolveChain_filterRejectsFirst_fallsToSecond() {
        var opus = descriptor("opus", "claude");
        var sonnet = descriptor("sonnet", "claude");
        var registry = registryWith(opus, sonnet);
        ModelAvailabilityFilter rejectOpus = d -> !d.id().equals("opus");
        var provider = new RoutingAgentProvider(backendRegistry(claudeBackend), "claude", registry,
                java.util.Map.of(), rejectOpus);
        var config = AgentSessionConfig.of("sys", "user").withModelChain(ModelChain.of("opus", "sonnet"));
        assertThatCode(() -> provider.invoke(config)).doesNotThrowAnyException();
    }

    @Test
    void resolveChain_mixedNamedAndQueried() {
        var opus = descriptor("opus", "claude");
        var registry = registryWith(opus);
        var provider = new RoutingAgentProvider(backendRegistry(claudeBackend), "claude", registry,
                java.util.Map.of(), ModelAvailabilityFilter.ALWAYS_AVAILABLE);
        var entries = List.<ModelChain.ModelChainEntry>of(
                new ModelChain.ModelChainEntry.Queried(ModelQuery.builder().tier(ModelTier.EMBEDDING).build()),
                new ModelChain.ModelChainEntry.Named("opus"));
        var config = AgentSessionConfig.of("sys", "user").withModelChain(ModelChain.of(entries));
        assertThatCode(() -> provider.invoke(config)).doesNotThrowAnyException();
    }

    @Test
    void resolveChain_modelChainTakesPrecedenceOverModel() {
        var sonnet = descriptor("sonnet", "claude");
        var registry = registryWith(sonnet);  // no opus
        var provider = new RoutingAgentProvider(backendRegistry(claudeBackend), "claude", registry,
                java.util.Map.of(), ModelAvailabilityFilter.ALWAYS_AVAILABLE);
        // model = "nonexistent" but chain = ["sonnet"] — chain wins
        var config = new AgentSessionConfig("sys", "user", List.of(), null, null, "nonexistent",
                null, ModelChain.of("sonnet"));
        assertThatCode(() -> provider.invoke(config)).doesNotThrowAnyException();
    }

    @Test
    void openSession_usesChainResolution() {
        var sonnet = descriptor("sonnet", "claude");
        var registry = registryWith(sonnet);
        var provider = new RoutingAgentProvider(backendRegistry(claudeBackend), "claude", registry,
                java.util.Map.of(), ModelAvailabilityFilter.ALWAYS_AVAILABLE);
        var init = AgentSessionInit.of("prompt").withModelChain(ModelChain.of("nonexistent", "sonnet"));
        // Would throw if chain wasn't resolved — nonexistent is not in registry
        // But sonnet is, so openSession should succeed
        assertThatCode(() -> provider.openSession(init)).doesNotThrowAnyException();
    }

    private static class StubBackend implements AgentBackend {
        private final String backendKey;
        StubBackend(String key) { this.backendKey = key; }
        @Override public String key() { return backendKey; }
        @Override public Multi<AgentEvent> invoke(AgentSessionConfig config) { return Multi.createFrom().empty(); }
        @Override public AgentSession openSession(AgentSessionInit init) {
            return new AgentSession() {
                @Override public void sendInput(String input) {}
                @Override public void close() {}
            };
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-router-core -Dtest=RoutingAgentProviderChainTest -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
Expected: FAIL — constructor missing, resolveChain not implemented

- [ ] **Step 3: Create ModelChainExhaustedException.java**

```java
package io.casehub.platform.agent.router;

import io.casehub.platform.api.model.ModelChain;
import java.util.List;

public class ModelChainExhaustedException extends IllegalStateException {
    private final ModelChain chain;
    private final List<ModelChain.ModelChainEntry> attemptedEntries;

    public ModelChainExhaustedException(ModelChain chain, List<ModelChain.ModelChainEntry> attempted) {
        super("All %d chain entries exhausted".formatted(attempted.size()));
        this.chain = chain;
        this.attemptedEntries = List.copyOf(attempted);
    }

    public ModelChain chain() { return chain; }
    public List<ModelChain.ModelChainEntry> attemptedEntries() { return attemptedEntries; }
}
```

- [ ] **Step 4: Create ModelChainRetryableException.java**

```java
package io.casehub.platform.agent.router;

import io.casehub.platform.api.model.ModelChain;

class ModelChainRetryableException extends RuntimeException {
    private final ModelChain.ModelChainEntry failedEntry;

    ModelChainRetryableException(ModelChain.ModelChainEntry entry, Throwable cause) {
        super("Chain entry %s failed: %s".formatted(entry, cause.getMessage()), cause);
        this.failedEntry = entry;
    }

    ModelChain.ModelChainEntry failedEntry() { return failedEntry; }
}
```

- [ ] **Step 5: Modify RoutingAgentProvider.java — add chain resolution**

Add new constructor with `ModelAvailabilityFilter`, `resolveChain()`, `resolveFromConfig()`, `resolveFromInit()`. Update `invoke()` and `openSession()` to use the new resolution helpers. Keep existing constructors backward-compatible (default filter = `ALWAYS_AVAILABLE`).

Key change: `invoke()` and `openSession()` call `resolveFromConfig()`/`resolveFromInit()` which check `modelChain` first, then `modelQuery`, then `model`. `resolveChain()` iterates entries, catches only `IllegalArgumentException` (resolution failure), checks availability filter, returns first match.

- [ ] **Step 6: Run all router tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-router-core -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`
Expected: All existing + 7 new tests PASS

- [ ] **Step 7: Install platform SNAPSHOT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -f /Users/mdproctor/claude/casehub/slots/202/platform/pom.xml`

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/202/platform add agent-api/ agent-router-core/
git -C /Users/mdproctor/claude/casehub/slots/202/platform commit -m "feat(router): add chain resolution to RoutingAgentProvider Refs casehubio/claudony#212"
```

---

## Batch 3: Claudony pool YAML parsing and definition

### Task 4: AgentPoolDefinition.AgentConfig gains modelChain + YAML parsing

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParserChainTest.java`

**Interfaces:**
- Consumes: `ModelChain`, `ModelChainEntry.Named`, `ModelChainEntry.Queried`, `ModelQuery`, `ModelTier` (from Task 1)
- Produces: `AgentConfig.modelChain()`, `AgentConfig.entryCommands()`, `AgentConfig.gracePeriods()` — used by Tasks 5-7

- [ ] **Step 1: Write AgentPoolYamlParserChainTest**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.platform.api.model.ModelChain;
import io.casehub.platform.api.model.ModelTier;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentPoolYamlParserChainTest {

    private final AgentPoolYamlParser parser = new AgentPoolYamlParser();

    @Test
    void stringEntries_parsedAsNamed() {
        var yaml = """
                agent-pools:
                  reviewer:
                    model-chain:
                      - opus
                      - sonnet
                    pool:
                      max-active: 3
                """;
        var defs = parser.parse(yaml);
        assertThat(defs).hasSize(1);
        var chain = defs.get(0).agent().modelChain();
        assertThat(chain.entries()).hasSize(2);
        assertThat(chain.entries().get(0)).isInstanceOf(ModelChain.ModelChainEntry.Named.class);
        assertThat(((ModelChain.ModelChainEntry.Named) chain.entries().get(0)).modelRef()).isEqualTo("opus");
    }

    @Test
    void structuredEntry_withCommandOverride() {
        var yaml = """
                agent-pools:
                  reviewer:
                    model-chain:
                      - model: llama3
                        command: ollama run
                    pool:
                      max-active: 3
                """;
        var defs = parser.parse(yaml);
        var agent = defs.get(0).agent();
        assertThat(agent.entryCommands()).containsEntry("llama3", "ollama run");
    }

    @Test
    void queriedEntry_withTier() {
        var yaml = """
                agent-pools:
                  reviewer:
                    model-chain:
                      - tier: FLAGSHIP
                        vendor: google
                    pool:
                      max-active: 3
                """;
        var defs = parser.parse(yaml);
        var entry = defs.get(0).agent().modelChain().entries().get(0);
        assertThat(entry).isInstanceOf(ModelChain.ModelChainEntry.Queried.class);
        var queried = (ModelChain.ModelChainEntry.Queried) entry;
        assertThat(queried.query().tier()).isEqualTo(ModelTier.FLAGSHIP);
        assertThat(queried.query().vendor()).isEqualTo("google");
    }

    @Test
    void gracePeriodOverride() {
        var yaml = """
                agent-pools:
                  reviewer:
                    model-chain:
                      - model: llama3
                        command: ollama run
                        grace-period: 60s
                    pool:
                      max-active: 3
                """;
        var defs = parser.parse(yaml);
        assertThat(defs.get(0).agent().gracePeriods())
                .containsEntry("llama3", java.time.Duration.ofSeconds(60));
    }

    @Test
    void noModelChain_nullChain() {
        var yaml = """
                agent-pools:
                  reviewer:
                    command: claude
                    pool:
                      max-active: 3
                """;
        var defs = parser.parse(yaml);
        assertThat(defs.get(0).agent().modelChain()).isNull();
    }

    @Test
    void mixedEntries() {
        var yaml = """
                agent-pools:
                  reviewer:
                    model-chain:
                      - opus
                      - model: sonnet
                      - tier: STANDARD
                    pool:
                      max-active: 3
                """;
        var defs = parser.parse(yaml);
        var chain = defs.get(0).agent().modelChain();
        assertThat(chain.entries()).hasSize(3);
        assertThat(chain.entries().get(0)).isInstanceOf(ModelChain.ModelChainEntry.Named.class);
        assertThat(chain.entries().get(1)).isInstanceOf(ModelChain.ModelChainEntry.Named.class);
        assertThat(chain.entries().get(2)).isInstanceOf(ModelChain.ModelChainEntry.Queried.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolYamlParserChainTest -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: FAIL

- [ ] **Step 3: Modify AgentPoolDefinition.AgentConfig — add modelChain, entryCommands, gracePeriods**

Add three new fields to the `AgentConfig` record. Update the `AgentBuilder` to set them. Update `Builder.build()` to pass them to the `AgentConfig` constructor.

- [ ] **Step 4: Modify AgentPoolYamlParser — add parseModelChain()**

Add `parseModelChain(List<Object>)` method returning `ModelChainParseResult(ModelChain, Map<String,String> commands, Map<String,Duration> gracePeriods)`. Wire it into `toDefinition()` when `model-chain` key is present in the config map.

- [ ] **Step 5: Modify AgentPoolSchema — add model-chain parameter**

Add `model-chain` as a `StepParameterType.LIST` parameter.

- [ ] **Step 6: Fix any broken existing tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Fix `AgentConfig` constructor call sites in existing tests (4 → 7 fields). Add `null, Map.of(), Map.of()` to direct constructions.

- [ ] **Step 7: Run all tests to verify they pass**

Expected: All existing + 6 new tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/202/claudony add casehub/
git -C /Users/mdproctor/claude/casehub/slots/202/claudony commit -m "feat(fleet): add model-chain parsing to pool YAML and AgentPoolDefinition Refs #212"
```

---

## Batch 4: CLI chain resolution and provisioner integration

### Task 5: CLI chain resolution in ClaudonyWorkerProvisioner + ClaudonyProviderConfig.withModel()

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyProviderConfig.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisioner.java` (or the equivalent provisioning path)
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/CliChainResolver.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/CliChainResolverTest.java`

**Interfaces:**
- Consumes: `ModelChain`, `ModelChainEntry`, `AgentPoolDefinition.AgentConfig` (Task 4), `ModelRegistry` (platform)
- Produces: `CliChainResolver.resolve(AgentPoolDefinition, ModelRegistry)` → `CliResolvedModel(model, command)`, `ClaudonyProviderConfig.withModel(String)` — used by Task 6 (runtime fallback)

- [ ] **Step 1: Write CliChainResolverTest**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.platform.agent.router.ModelChainExhaustedException;
import io.casehub.platform.api.model.*;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class CliChainResolverTest {

    private ModelRegistry registryWith(String... modelIds) {
        return new ModelRegistry() {
            @Override public Optional<ModelDescriptor> resolveById(String id) {
                return Arrays.stream(modelIds).filter(m -> m.equals(id)).findFirst()
                        .map(m -> new ModelDescriptor(m, m, "claude", "default",
                                "anthropic", "claude", m, ModelTier.FLAGSHIP,
                                Set.of(), 200000, 16000, ModelLocality.CLOUD, CostTier.HIGH, null, Map.of()));
            }
            @Override public List<ModelDescriptor> query(ModelQuery query) {
                return all().stream().filter(d -> query.tier() == null || d.tier() == query.tier()).toList();
            }
            @Override public List<ModelDescriptor> all() {
                return Arrays.stream(modelIds).map(m -> resolveById(m).orElseThrow()).toList();
            }
        };
    }

    @Test
    void sameBackendChain_resolvesFirstAvailable() {
        var def = AgentPoolDefinition.builder()
                .agent("test").command("claude")
                .pool().maxActive(5).build();
        // Override with chain via builder (once builder supports it)
        // For now, construct directly
        var result = CliChainResolver.resolve(
                ModelChain.of("opus", "sonnet"), "claude", Map.of(), registryWith("opus", "sonnet"));
        assertThat(result.model()).isEqualTo("opus");
        assertThat(result.command()).isEqualTo("claude");
    }

    @Test
    void firstMissing_fallsToSecond() {
        var result = CliChainResolver.resolve(
                ModelChain.of("opus", "sonnet"), "claude", Map.of(), registryWith("sonnet"));
        assertThat(result.model()).isEqualTo("sonnet");
    }

    @Test
    void crossBackend_usesCommandOverride() {
        var commands = Map.of("llama3", "ollama run");
        var result = CliChainResolver.resolve(
                ModelChain.of("llama3"), "claude", commands, registryWith("llama3"));
        assertThat(result.model()).isEqualTo("llama3");
        assertThat(result.command()).isEqualTo("ollama run");
    }

    @Test
    void allExhausted_throws() {
        assertThatThrownBy(() -> CliChainResolver.resolve(
                ModelChain.of("opus"), "claude", Map.of(), registryWith()))
                .isInstanceOf(ModelChainExhaustedException.class);
    }

    @Test
    void nullChain_returnsNullModel() {
        var result = CliChainResolver.resolve(null, "claude", Map.of(), registryWith());
        assertThat(result.model()).isNull();
        assertThat(result.command()).isEqualTo("claude");
    }

    @Test
    void queriedEntry_resolvesViaTier() {
        var entries = List.<ModelChain.ModelChainEntry>of(
                new ModelChain.ModelChainEntry.Queried(ModelQuery.builder().tier(ModelTier.FLAGSHIP).build()));
        var result = CliChainResolver.resolve(
                ModelChain.of(entries), "claude", Map.of(), registryWith("opus"));
        assertThat(result.model()).isEqualTo("opus");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Create CliChainResolver.java**

Static utility — iterates chain entries, resolves Named via `ModelRegistry.resolveById()`, resolves Queried via `ModelRegistry.query()`, applies command overrides from the entry map.

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.platform.agent.router.ModelChainExhaustedException;
import io.casehub.platform.api.model.*;
import java.util.Map;

public final class CliChainResolver {
    private CliChainResolver() {}

    public record CliResolvedModel(String model, String command) {}

    public static CliResolvedModel resolve(ModelChain chain, String defaultCommand,
                                           Map<String, String> entryCommands,
                                           ModelRegistry modelRegistry) {
        if (chain == null || chain.isEmpty()) {
            return new CliResolvedModel(null, defaultCommand);
        }

        for (var entry : chain.entries()) {
            String modelRef = switch (entry) {
                case ModelChain.ModelChainEntry.Named n -> {
                    var desc = modelRegistry.resolveById(n.modelRef());
                    yield desc.isPresent() ? n.modelRef() : null;
                }
                case ModelChain.ModelChainEntry.Queried q -> {
                    var candidates = modelRegistry.query(q.query());
                    yield candidates.isEmpty() ? null : candidates.get(0).apiModelId();
                }
            };

            if (modelRef == null) continue;

            var command = entryCommands.getOrDefault(modelRef, defaultCommand);
            return new CliResolvedModel(modelRef, command);
        }

        throw new ModelChainExhaustedException(chain, chain.entries());
    }
}
```

- [ ] **Step 4: Add ClaudonyProviderConfig.withModel()**

```java
public ClaudonyProviderConfig withModel(String model) {
    return new ClaudonyProviderConfig(command, Optional.ofNullable(model), appendSystemPrompt,
        systemPrompt, effort, permissionMode, tools, allowedTools, disallowedTools, addDirs, workingDir);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=CliChainResolverTest -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`
Expected: 6 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/202/claudony add casehub/
git -C /Users/mdproctor/claude/casehub/slots/202/claudony commit -m "feat(fleet): CLI chain resolver and ClaudonyProviderConfig.withModel() Refs #212"
```

---

## Batch 5: ModelFallbackEvent and integration wiring

### Task 6: ModelFallbackEvent CDI event

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ModelFallbackEvent.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ModelFallbackEventTest.java`

**Interfaces:**
- Consumes: none (pure data record)
- Produces: `ModelFallbackEvent(poolName, requestedModel, resolvedModel, fallbackDepth)` — CDI event fired by provisioner, observed by any component

- [ ] **Step 1: Write ModelFallbackEventTest**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ModelFallbackEventTest {

    @Test
    void recordFieldsAccessible() {
        var event = new ModelFallbackEvent("code-reviewer", "opus", "sonnet", 2);
        assertThat(event.poolName()).isEqualTo("code-reviewer");
        assertThat(event.requestedModel()).isEqualTo("opus");
        assertThat(event.resolvedModel()).isEqualTo("sonnet");
        assertThat(event.fallbackDepth()).isEqualTo(2);
    }

    @Test
    void equality() {
        var a = new ModelFallbackEvent("pool", "opus", "sonnet", 1);
        var b = new ModelFallbackEvent("pool", "opus", "sonnet", 1);
        assertThat(a).isEqualTo(b);
    }
}
```

- [ ] **Step 2: Create ModelFallbackEvent.java**

```java
package io.casehub.claudony.casehub.fleet;

public record ModelFallbackEvent(
        String poolName,
        String requestedModel,
        String resolvedModel,
        int fallbackDepth) {}
```

- [ ] **Step 3: Run tests to verify they pass**

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/202/claudony add casehub/
git -C /Users/mdproctor/claude/casehub/slots/202/claudony commit -m "feat(fleet): add ModelFallbackEvent CDI event for degraded provisioning Refs #212"
```

### Task 7: Integration test — pool YAML with model chain end-to-end

**Files:**
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/FleetPoolChainIntegrationTest.java`

**Interfaces:**
- Consumes: `AgentPoolYamlParser` (Task 4), `CliChainResolver` (Task 5), `ModelFallbackEvent` (Task 6), `ModelChain` (Task 1)

- [ ] **Step 1: Write FleetPoolChainIntegrationTest**

Test that parses a pool YAML with `model-chain`, resolves through `CliChainResolver`, and verifies the resolved model and command. Uses in-memory `ModelRegistry`. Verifies fallback behavior (first model missing → second used). Verifies cross-backend command override.

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=FleetPoolChainIntegrationTest -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/202/claudony add casehub/
git -C /Users/mdproctor/claude/casehub/slots/202/claudony commit -m "test(fleet): integration test for model chain YAML → CLI resolution Refs #212"
```

---

## References

- [2026-10-03-model-fallback-chains-design.md] — design spec this plan implements
- `RoutingAgentProvider.java` — existing resolve()/resolveQuery() resolution chain
- `AgentSessionInit.java`, `AgentSessionConfig.java` — current record structure (6/7 fields)
- `AgentPoolYamlParser.java` — existing YAML parsing patterns
- `AgentPoolDefinition.java` — current AgentConfig record (4 fields)
- `ClaudonyProviderConfig.java` — current config record (11 fields)
- `RoutingAgentProviderCreateTest.java` — existing test patterns (StubBackend, EmptyModelRegistry)
- `ModelChainTest.java` (platform-api) — existing test patterns for model types
- GitHub #212 — feat: model fallback chains for agent routing
- GitHub #205 — agent pool management spec (deferred fallback chains)
