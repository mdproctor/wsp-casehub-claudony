# Eidos Pool Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #209 — feat: Eidos agent identity integration for agent pools

**Goal:** Bridge Eidos agent identity into Claudony's pool layer so that agents defined in the Eidos AgentRegistry automatically get pool backing.

**Architecture:** A single `EidosPoolBridge` bean in `claudony-casehub` reads `AgentDescriptor`s from `AgentRegistry`, converts them to `AgentPoolDefinition`s via a pure `EidosPoolConverter`, and registers them. Config-driven activation (`claudony.eidos.pool-bridge.enabled`). Startup read + 30s polling reconciliation.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-eidos-api 0.2-SNAPSHOT, JUnit 5, AssertJ

## Global Constraints

- Java 21 API surface (compiles on Java 26 JVM)
- `casehub-eidos-api` is an optional dependency — the bridge must not break builds or tests when eidos is absent
- No Eidos API changes — consume existing `AgentRegistry`, `AgentDescriptor`, `ProviderConfig` as-is
- `@IfBuildProperty` guard on bridge bean — default off
- Existing `AgentConfig` constructors must remain source-compatible (new field has default)
- Test convention: `@TestSecurity` only on HTTP-exercising tests; unit tests mock directly

---

## Batch 1: Foundation — AgentConfig field + EidosPoolConverter

### Task 1: Add `eidosAgentId` field to `AgentConfig` and update Builder

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java:46-60` (AgentConfig record)
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java:77-105` (Builder class)
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionTest.java`

**Interfaces:**
- Produces: `AgentConfig.eidosAgentId()` — `String`, nullable. Returns the Eidos organizational agent identity. `null` for non-Eidos pools.
- Produces: `AgentBuilder.eidosAgentId(String)` — builder setter, returns `AgentBuilder`.

- [ ] **Step 1: Write failing test for eidosAgentId on AgentConfig**

Add to `AgentPoolDefinitionTest`:

```java
@Test
void eidosAgentIdOnAgentConfig() {
    var def = AgentPoolDefinition.builder()
            .agent("code-reviewer")
                .eidosAgentId("eidos-code-reviewer-001")
            .build();

    assertThat(def.agent().eidosAgentId()).isEqualTo("eidos-code-reviewer-001");
}

@Test
void eidosAgentIdDefaultsToNull() {
    var def = AgentPoolDefinition.builder()
            .agent("researcher")
            .build();

    assertThat(def.agent().eidosAgentId()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolDefinitionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — `eidosAgentId` method and builder method don't exist.

- [ ] **Step 3: Add eidosAgentId to AgentConfig record**

Use `ide_edit_member` to update the `AgentConfig` record declaration in `AgentPoolDefinition.java`. The record gains an 8th field `eidosAgentId` (String, nullable). Update the compact constructor to allow null. Update the 4-arg convenience constructor to pass `null`. Update the 7-arg constructor to pass `null`.

```java
public record AgentConfig(String name, String workingDir, WorkingDirPolicy policy, String command,
                           ModelChain modelChain, Map<String, String> entryCommands,
                           Map<String, Duration> gracePeriods,
                           String eidosAgentId) {
    public AgentConfig {
        Objects.requireNonNull(name, "agent name is required");
        if (name.isBlank()) throw new IllegalArgumentException("agent name must not be blank");
        if (policy == null) policy = WorkingDirPolicy.EXCLUSIVE;
        if (entryCommands == null) entryCommands = Map.of();
        if (gracePeriods == null) gracePeriods = Map.of();
    }

    public AgentConfig(String name, String workingDir, WorkingDirPolicy policy, String command) {
        this(name, workingDir, policy, command, null, Map.of(), Map.of(), null);
    }

    public AgentConfig(String name, String workingDir, WorkingDirPolicy policy, String command,
                       ModelChain modelChain, Map<String, String> entryCommands,
                       Map<String, Duration> gracePeriods) {
        this(name, workingDir, policy, command, modelChain, entryCommands, gracePeriods, null);
    }
}
```

The existing 7-arg constructor becomes a delegating constructor to the new 8-arg canonical constructor. This preserves source compatibility for all existing callers.

- [ ] **Step 4: Add eidosAgentId to Builder**

Add `private String eidosAgentId;` field to `Builder` (line ~90). Add `eidosAgentId(String)` method to `AgentBuilder`. Update `Builder.build()` to pass `eidosAgentId` to the `AgentConfig` constructor.

```java
// In Builder class, add field:
private String eidosAgentId;

// In Builder.build():
var agentConfig = new AgentConfig(agentName, workingDir, policy, command, modelChain, entryCommands, gracePeriods, eidosAgentId);
```

```java
// In AgentBuilder class, add method:
public AgentBuilder eidosAgentId(String eidosAgentId) {
    parent.eidosAgentId = eidosAgentId;
    return this;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolDefinitionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS (including 2 new ones).

- [ ] **Step 6: Run full casehub module tests to check for breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: All ~598 tests PASS. The new 8-arg canonical constructor is source-compatible because the old 7-arg constructor still exists as a delegating constructor.

- [ ] **Step 7: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionTest.java
git commit -m "feat(#209): add eidosAgentId field to AgentConfig

Refs #209"
```

---

### Task 2: Create EidosPoolConverter — pure conversion function

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolConverter.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolConverterTest.java`

**Interfaces:**
- Consumes: `AgentPoolDefinition.builder()...eidosAgentId(String)...build()` from Task 1
- Consumes: `AgentDescriptor` (eidos-api) — `agentId()`, `name()`, `capabilities()`, `providerConfigs()`
- Consumes: `AgentCapability` (eidos-api) — `model()` returns `ModelQuery`, `modelRef()` returns `String`
- Consumes: `ProviderConfig` (eidos-api) — `providerName()`, `config()` returns `Map<String,String>`
- Produces: `EidosPoolConverter.toPoolDefinition(AgentDescriptor, EidosPoolBridgeConfig)` → `AgentPoolDefinition`
- Produces: `EidosPoolConverter.findClaudonyProviderConfig(AgentDescriptor)` → `Optional<ProviderConfig>`

- [ ] **Step 1: Add casehub-eidos-api dependency to claudony-casehub pom.xml**

Add to `casehub/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-api</artifactId>
    <version>${casehub-eidos.version}</version>
    <optional>true</optional>
</dependency>
```

Add `<casehub-eidos.version>0.2-SNAPSHOT</casehub-eidos.version>` to the parent `pom.xml` `<properties>` if not already present.

- [ ] **Step 2: Create EidosPoolBridgeConfig record**

This is a simple config record that holds bridge defaults. Not a Quarkus `@ConfigMapping` — that lives in the app module. This is a plain record used by the converter.

```java
package io.casehub.claudony.casehub.fleet;

public record EidosPoolBridgeConfig(
        int defaultMinActive,
        int defaultMaxActive,
        String defaultCommand
) {
    public static final EidosPoolBridgeConfig DEFAULTS =
            new EidosPoolBridgeConfig(0, 5, "claude");
}
```

- [ ] **Step 3: Write failing tests for EidosPoolConverter**

Create `EidosPoolConverterTest.java`:

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.eidos.api.AgentCapability;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.ProviderConfig;
import io.casehub.platform.api.model.ModelQuery;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class EidosPoolConverterTest {

    private static final EidosPoolBridgeConfig DEFAULTS = EidosPoolBridgeConfig.DEFAULTS;

    @Test
    void fullDescriptorWithCapabilitiesAndPoolConfig() {
        var descriptor = AgentDescriptor.builder()
                .agentId("code-reviewer")
                .name("Code Reviewer")
                .slot("reviewer")
                .tenancyId("default")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("review")
                                .model(ModelQuery.builder().modelFamily("claude").build())
                                .build()))
                .providerConfigs(List.of(
                        new ProviderConfig("claudony", Map.of(
                                "minActive", "2",
                                "maxActive", "10",
                                "command", "claude --model opus"))))
                .build();

        var result = EidosPoolConverter.toPoolDefinition(descriptor, DEFAULTS);

        assertThat(result.agent().name()).isEqualTo("code-reviewer");
        assertThat(result.agent().eidosAgentId()).isEqualTo("code-reviewer");
        assertThat(result.agent().command()).isEqualTo("claude --model opus");
        assertThat(result.pool().minActive()).isEqualTo(2);
        assertThat(result.pool().maxActive()).isEqualTo(10);
    }

    @Test
    void missingPoolConfigUsesDefaults() {
        var descriptor = AgentDescriptor.builder()
                .agentId("researcher")
                .name("Researcher")
                .slot("researcher")
                .tenancyId("default")
                .providerConfigs(List.of(
                        new ProviderConfig("claudony", Map.of())))
                .build();

        var result = EidosPoolConverter.toPoolDefinition(descriptor, DEFAULTS);

        assertThat(result.pool().minActive()).isEqualTo(0);
        assertThat(result.pool().maxActive()).isEqualTo(5);
        assertThat(result.agent().command()).isEqualTo("claude");
    }

    @Test
    void noModelOnAnyCapability() {
        var descriptor = AgentDescriptor.builder()
                .agentId("worker")
                .name("Worker")
                .slot("worker")
                .tenancyId("default")
                .providerConfigs(List.of(
                        new ProviderConfig("claudony", Map.of())))
                .build();

        var result = EidosPoolConverter.toPoolDefinition(descriptor, DEFAULTS);

        assertThat(result.agent().modelChain()).isNull();
    }

    @Test
    void multipleCapabilitiesFirstWithModelWins() {
        var descriptor = AgentDescriptor.builder()
                .agentId("multi")
                .name("Multi")
                .slot("multi")
                .tenancyId("default")
                .capabilities(List.of(
                        AgentCapability.builder().name("no-model").build(),
                        AgentCapability.builder()
                                .name("has-model")
                                .model(ModelQuery.builder().modelFamily("claude").modelVersion("opus").build())
                                .build()))
                .providerConfigs(List.of(
                        new ProviderConfig("claudony", Map.of())))
                .build();

        var result = EidosPoolConverter.toPoolDefinition(descriptor, DEFAULTS);

        assertThat(result.agent().modelChain()).isNotNull();
    }

    @Test
    void findClaudonyProviderConfigPresent() {
        var descriptor = AgentDescriptor.builder()
                .agentId("test")
                .name("Test")
                .slot("test")
                .tenancyId("default")
                .providerConfigs(List.of(
                        new ProviderConfig("other", Map.of()),
                        new ProviderConfig("claudony", Map.of("command", "claude"))))
                .build();

        var result = EidosPoolConverter.findClaudonyProviderConfig(descriptor);

        assertThat(result).isPresent();
        assertThat(result.get().providerName()).isEqualTo("claudony");
    }

    @Test
    void findClaudonyProviderConfigAbsent() {
        var descriptor = AgentDescriptor.builder()
                .agentId("test")
                .name("Test")
                .slot("test")
                .tenancyId("default")
                .providerConfigs(List.of(
                        new ProviderConfig("openai", Map.of())))
                .build();

        assertThat(EidosPoolConverter.findClaudonyProviderConfig(descriptor)).isEmpty();
    }

    @Test
    void malformedMinActiveUsesDefault() {
        var descriptor = AgentDescriptor.builder()
                .agentId("bad")
                .name("Bad")
                .slot("bad")
                .tenancyId("default")
                .providerConfigs(List.of(
                        new ProviderConfig("claudony", Map.of("minActive", "not-a-number"))))
                .build();

        var result = EidosPoolConverter.toPoolDefinition(descriptor, DEFAULTS);

        assertThat(result.pool().minActive()).isEqualTo(0);
    }

    @Test
    void workingDirFromProviderConfig() {
        var descriptor = AgentDescriptor.builder()
                .agentId("test")
                .name("Test")
                .slot("test")
                .tenancyId("default")
                .providerConfigs(List.of(
                        new ProviderConfig("claudony", Map.of("workingDir", "/workspace/reviews"))))
                .build();

        var result = EidosPoolConverter.toPoolDefinition(descriptor, DEFAULTS);

        assertThat(result.agent().workingDir()).isEqualTo("/workspace/reviews");
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=EidosPoolConverterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — `EidosPoolConverter` doesn't exist.

- [ ] **Step 5: Implement EidosPoolConverter**

Create `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolConverter.java`:

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.eidos.api.AgentCapability;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.ProviderConfig;
import io.casehub.platform.api.model.ModelChain;
import io.casehub.platform.api.model.ModelQuery;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.Optional;

public final class EidosPoolConverter {

    private static final Logger LOG = Logger.getLogger(EidosPoolConverter.class);
    private static final String PROVIDER_NAME = "claudony";

    private EidosPoolConverter() {}

    public static AgentPoolDefinition toPoolDefinition(AgentDescriptor descriptor,
                                                        EidosPoolBridgeConfig defaults) {
        var providerConfig = findClaudonyProviderConfig(descriptor)
                .map(ProviderConfig::config)
                .orElse(Map.of());

        int minActive = parseIntOrDefault(providerConfig, "minActive", defaults.defaultMinActive());
        int maxActive = parseIntOrDefault(providerConfig, "maxActive", defaults.defaultMaxActive());
        String command = providerConfig.getOrDefault("command", defaults.defaultCommand());
        String workingDir = providerConfig.get("workingDir");

        ModelChain modelChain = resolveModelChain(descriptor);

        return AgentPoolDefinition.builder()
                .agent(descriptor.agentId())
                    .eidosAgentId(descriptor.agentId())
                    .command(command)
                    .workingDir(workingDir)
                    .modelChain(modelChain)
                .pool()
                    .minActive(minActive)
                    .maxActive(maxActive)
                .build();
    }

    public static Optional<ProviderConfig> findClaudonyProviderConfig(AgentDescriptor descriptor) {
        return descriptor.providerConfigs().stream()
                .filter(pc -> PROVIDER_NAME.equals(pc.providerName()))
                .findFirst();
    }

    private static ModelChain resolveModelChain(AgentDescriptor descriptor) {
        return descriptor.capabilities().stream()
                .filter(cap -> cap.model() != null || cap.modelRef() != null)
                .findFirst()
                .map(EidosPoolConverter::toModelChain)
                .orElse(null);
    }

    private static ModelChain toModelChain(AgentCapability capability) {
        if (capability.modelRef() != null) {
            return ModelChain.of(capability.modelRef());
        }
        ModelQuery query = capability.model();
        String modelId = query.modelFamily();
        if (query.modelVersion() != null) {
            modelId = modelId + "-" + query.modelVersion();
        }
        return ModelChain.of(modelId);
    }

    private static int parseIntOrDefault(Map<String, String> config, String key, int defaultValue) {
        String value = config.get(key);
        if (value == null) return defaultValue;
        try {
            return Integer.parseInt(value);
        } catch (NumberFormatException e) {
            LOG.warnf("Eidos pool bridge: invalid integer for '%s': '%s', using default %d", key, value, defaultValue);
            return defaultValue;
        }
    }
}
```

Note: `ModelChain.of(String)` — verify this factory method exists. If `ModelChain` uses a different construction pattern, adapt accordingly. Use `ide_file_structure` on `ModelChain` to check.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=EidosPoolConverterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All 8 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add casehub/pom.xml pom.xml casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolConverter.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridgeConfig.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolConverterTest.java
git commit -m "feat(#209): add EidosPoolConverter — pure descriptor-to-pool conversion

Refs #209"
```

---

## Batch 2: Bridge — EidosPoolBridge bean + config + integration test

### Task 3: Create EidosPoolBridge — startup + reconciliation bean

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolBridgeTest.java`

**Interfaces:**
- Consumes: `EidosPoolConverter.toPoolDefinition(AgentDescriptor, EidosPoolBridgeConfig)` from Task 2
- Consumes: `EidosPoolConverter.findClaudonyProviderConfig(AgentDescriptor)` from Task 2
- Consumes: `AgentRegistry.find(AgentQuery)` — eidos-api
- Consumes: `AgentPoolDefinitionRegistry.register(AgentPoolDefinition)`, `.names()`, `.get(String)`, `.updateCapacity(String, int, int)` — existing
- Produces: `EidosPoolBridge` — CDI bean with `onStartup()` and `reconcile()` methods

- [ ] **Step 1: Write failing tests for EidosPoolBridge**

Create `EidosPoolBridgeTest.java` — unit test with mocked `AgentRegistry` and `AgentPoolDefinitionRegistry`:

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.ProviderConfig;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class EidosPoolBridgeTest {

    private AgentRegistry agentRegistry;
    private AgentPoolDefinitionRegistry poolRegistry;
    private EidosPoolBridge bridge;
    private static final EidosPoolBridgeConfig CONFIG =
            new EidosPoolBridgeConfig(0, 5, "claude");
    private static final String TENANCY_ID = "default";

    @BeforeEach
    void setUp() {
        agentRegistry = mock(AgentRegistry.class);
        poolRegistry = mock(AgentPoolDefinitionRegistry.class);
        when(poolRegistry.names()).thenReturn(Set.of());
        bridge = new EidosPoolBridge(agentRegistry, poolRegistry, CONFIG, TENANCY_ID);
    }

    @Test
    void startupRegistersMatchedAgents() {
        var descriptor = descriptorWithClaudonyConfig("code-reviewer");
        when(agentRegistry.find(any(AgentQuery.class)))
                .thenReturn(List.of(new AgentMatch(descriptor, 1.0)));

        bridge.onStartup();

        verify(poolRegistry).register(argThat(def ->
                def.agent().name().equals("code-reviewer")
                && def.agent().eidosAgentId().equals("code-reviewer")));
    }

    @Test
    void agentsWithoutClaudonyConfigSkipped() {
        var descriptor = AgentDescriptor.builder()
                .agentId("openai-agent")
                .name("OpenAI Agent")
                .slot("worker")
                .tenancyId("default")
                .providerConfigs(List.of(new ProviderConfig("openai", Map.of())))
                .build();
        when(agentRegistry.find(any(AgentQuery.class)))
                .thenReturn(List.of(new AgentMatch(descriptor, 1.0)));

        bridge.onStartup();

        verify(poolRegistry, never()).register(any());
    }

    @Test
    void reconcileAddsNewAgents() {
        when(poolRegistry.names()).thenReturn(Set.of());
        var descriptor = descriptorWithClaudonyConfig("new-agent");
        when(agentRegistry.find(any(AgentQuery.class)))
                .thenReturn(List.of(new AgentMatch(descriptor, 1.0)));

        bridge.reconcile();

        verify(poolRegistry).register(argThat(def ->
                def.agent().name().equals("new-agent")));
    }

    @Test
    void reconcileSkipsAlreadyRegistered() {
        when(poolRegistry.names()).thenReturn(Set.of("existing-agent"));
        when(poolRegistry.get("existing-agent")).thenReturn(
                Optional.of(AgentPoolDefinition.builder().agent("existing-agent").build()));
        var descriptor = descriptorWithClaudonyConfig("existing-agent");
        when(agentRegistry.find(any(AgentQuery.class)))
                .thenReturn(List.of(new AgentMatch(descriptor, 1.0)));

        bridge.reconcile();

        verify(poolRegistry, never()).register(any());
    }

    @Test
    void emptyRegistryNoError() {
        when(agentRegistry.find(any(AgentQuery.class)))
                .thenReturn(List.of());

        bridge.onStartup();

        verify(poolRegistry, never()).register(any());
    }

    @Test
    void registryExceptionDoesNotPropagate() {
        when(agentRegistry.find(any(AgentQuery.class)))
                .thenThrow(new RuntimeException("registry unavailable"));

        bridge.onStartup();

        verify(poolRegistry, never()).register(any());
    }

    private AgentDescriptor descriptorWithClaudonyConfig(String agentId) {
        return AgentDescriptor.builder()
                .agentId(agentId)
                .name(agentId)
                .slot("worker")
                .tenancyId("default")
                .providerConfigs(List.of(
                        new ProviderConfig("claudony", Map.of())))
                .build();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=EidosPoolBridgeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — `EidosPoolBridge` doesn't exist.

- [ ] **Step 3: Implement EidosPoolBridge**

Create `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java`:

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.runtime.StartupEvent;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Set;

@ApplicationScoped
@IfBuildProperty(name = "claudony.eidos.pool-bridge.enabled", stringValue = "true")
public class EidosPoolBridge {

    private static final Logger LOG = Logger.getLogger(EidosPoolBridge.class);

    private final AgentRegistry agentRegistry;
    private final AgentPoolDefinitionRegistry poolRegistry;
    private final EidosPoolBridgeConfig config;
    private final String tenancyId;

    @Inject
    public EidosPoolBridge(AgentRegistry agentRegistry,
                           AgentPoolDefinitionRegistry poolRegistry,
                           EidosPoolBridgeConfig config,
                           @io.smallrye.config.ConfigMapping
                           String tenancyId) {
        this.agentRegistry = agentRegistry;
        this.poolRegistry = poolRegistry;
        this.config = config;
        this.tenancyId = tenancyId;
    }

    // Package-private constructor for unit tests
    EidosPoolBridge(AgentRegistry agentRegistry,
                    AgentPoolDefinitionRegistry poolRegistry,
                    EidosPoolBridgeConfig config,
                    String tenancyId) {
        this.agentRegistry = agentRegistry;
        this.poolRegistry = poolRegistry;
        this.config = config;
        this.tenancyId = tenancyId;
    }

    void onStartup(@Observes StartupEvent event) {
        onStartup();
    }

    void onStartup() {
        LOG.info("Eidos pool bridge: starting");
        try {
            List<AgentMatch> matches = agentRegistry.find(AgentQuery.all(tenancyId));
            int registered = 0;
            for (AgentMatch match : matches) {
                if (registerIfClaudony(match.descriptor())) {
                    registered++;
                }
            }
            LOG.infof("Eidos pool bridge: registered %d pool(s) from %d agent(s)", registered, matches.size());
        } catch (Exception e) {
            LOG.error("Eidos pool bridge: startup failed, will retry on next reconciliation", e);
        }
    }

    @Scheduled(every = "${claudony.eidos.pool-bridge.reconcile-interval:30s}")
    void reconcile() {
        try {
            List<AgentMatch> matches = agentRegistry.find(AgentQuery.all(tenancyId));
            Set<String> existingPools = poolRegistry.names();
            int added = 0;
            for (AgentMatch match : matches) {
                AgentDescriptor descriptor = match.descriptor();
                if (EidosPoolConverter.findClaudonyProviderConfig(descriptor).isEmpty()) {
                    continue;
                }
                if (!existingPools.contains(descriptor.agentId())) {
                    if (registerIfClaudony(descriptor)) {
                        added++;
                    }
                }
            }
            if (added > 0) {
                LOG.infof("Eidos pool bridge: reconciliation added %d new pool(s)", added);
            }
        } catch (Exception e) {
            LOG.warn("Eidos pool bridge: reconciliation failed", e);
        }
    }

    private boolean registerIfClaudony(AgentDescriptor descriptor) {
        if (EidosPoolConverter.findClaudonyProviderConfig(descriptor).isEmpty()) {
            return false;
        }
        try {
            var definition = EidosPoolConverter.toPoolDefinition(descriptor, config);
            poolRegistry.register(definition, "eidos");
            return true;
        } catch (Exception e) {
            LOG.warnf("Eidos pool bridge: failed to register pool for agent '%s': %s",
                    descriptor.agentId(), e.getMessage());
            return false;
        }
    }
}
```

Note: The CDI injection for `EidosPoolBridgeConfig` and `tenancyId` needs to be wired through a Quarkus `@ConfigMapping` or producer. This will be handled in Task 4 (app module config wiring). For unit testing, the package-private constructor is used directly.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=EidosPoolBridgeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolBridgeTest.java
git commit -m "feat(#209): add EidosPoolBridge — startup + reconciliation bean

Refs #209"
```

---

### Task 4: Config wiring and integration test

**Files:**
- Modify: `app/src/main/resources/application.properties` — add eidos bridge config defaults
- Create: `app/src/test/java/io/casehub/claudony/casehub/EidosPoolBridgeIntegrationTest.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java` — adjust CDI injection for config

**Interfaces:**
- Consumes: `EidosPoolBridge` from Task 3
- Consumes: `EidosPoolBridgeConfig` from Task 2
- Produces: Integration test proving bridge activates/deactivates based on config

- [ ] **Step 1: Add config properties to application.properties**

Add to `app/src/main/resources/application.properties`:

```properties
# Eidos pool bridge (default: off)
claudony.eidos.pool-bridge.enabled=false
claudony.eidos.pool-bridge.tenancy-id=default
claudony.eidos.pool-bridge.reconcile-interval=30s
claudony.eidos.pool-bridge.default-min-active=0
claudony.eidos.pool-bridge.default-max-active=5
claudony.eidos.pool-bridge.default-command=claude
```

- [ ] **Step 2: Create a CDI producer for EidosPoolBridgeConfig**

Add a `@Produces` method in `EidosPoolBridge` or a separate producer class that reads from `@ConfigProperty` and creates `EidosPoolBridgeConfig`. This keeps the config record plain (no Quarkus annotations in the casehub module).

```java
// In EidosPoolBridge, replace the @Inject constructor with:
@Inject
public EidosPoolBridge(
        AgentRegistry agentRegistry,
        AgentPoolDefinitionRegistry poolRegistry,
        @ConfigProperty(name = "claudony.eidos.pool-bridge.default-min-active", defaultValue = "0") int defaultMinActive,
        @ConfigProperty(name = "claudony.eidos.pool-bridge.default-max-active", defaultValue = "5") int defaultMaxActive,
        @ConfigProperty(name = "claudony.eidos.pool-bridge.default-command", defaultValue = "claude") String defaultCommand,
        @ConfigProperty(name = "claudony.eidos.pool-bridge.tenancy-id", defaultValue = "default") String tenancyId) {
    this.agentRegistry = agentRegistry;
    this.poolRegistry = poolRegistry;
    this.config = new EidosPoolBridgeConfig(defaultMinActive, defaultMaxActive, defaultCommand);
    this.tenancyId = tenancyId;
}
```

- [ ] **Step 3: Write integration test — bridge does NOT activate by default**

```java
package io.casehub.claudony.casehub;

import io.casehub.claudony.casehub.fleet.EidosPoolBridge;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class EidosPoolBridgeIntegrationTest {

    @Inject
    Instance<EidosPoolBridge> bridgeInstance;

    @Test
    void bridgeNotActiveByDefault() {
        assertThat(bridgeInstance.isResolvable()).isFalse();
    }
}
```

- [ ] **Step 4: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=EidosPoolBridgeIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — bridge bean is not instantiated because `claudony.eidos.pool-bridge.enabled` defaults to `false`.

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All ~1152 tests PASS. The new eidos-api dependency is optional and the bridge bean is guarded — no impact on existing tests.

- [ ] **Step 6: Update CLAUDE.md test count and test inventory**

Add entries for `EidosPoolConverterTest` (8 tests), `EidosPoolBridgeTest` (6 tests), and `EidosPoolBridgeIntegrationTest` (1 test) to the test inventory section.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/resources/application.properties app/src/test/java/io/casehub/claudony/casehub/EidosPoolBridgeIntegrationTest.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java CLAUDE.md
git commit -m "feat(#209): wire EidosPoolBridge config + integration test

Bridge activates only when claudony.eidos.pool-bridge.enabled=true.
Default off — no impact on existing deployments.

Refs #209"
```

---

## References

- [2026-10-12-eidos-pool-bridge-design.md] — design spec this plan implements
- [AgentPoolDefinition.java:46-60] — existing AgentConfig record (8th field added)
- [AgentPoolDefinition.java:77-105] — existing Builder (eidosAgentId field added)
- [AgentPoolDefinitionRegistry.java] — pool registration target
- [CompositeProviderConfigSource.java] — existing per-agent config path
- [ClaudonyProviderConfig.java] — existing provider config record
- [AgentSessionManager.java] — pool session lifecycle
- [AgentDescriptor (eidos-api)] — Eidos agent identity model
- [AgentRegistry (eidos-api)] — Eidos agent registry interface
- [ProviderConfig (eidos-api)] — per-provider config record
- [AgentQuery (eidos-api)] — registry query model
- [GitHub #209] — focal issue
- [GitHub #205] — fleet manager spec that deferred this
