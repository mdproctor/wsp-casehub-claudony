# Eidos Pool Bridge Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #275 — epic: Eidos pool bridge hardening
**Issue group:** #276, #277, #278, #279, #280, #281

**Goal:** Harden the Eidos pool bridge with session-visible identity, live reconciliation, system prompt injection, model resolution, multi-tenant isolation, and CDI event observation.

**Architecture:** Identity flows downward from `AgentDescriptor` through pool definitions, into tmux session metadata, and finally into the worker's system prompt. Each child issue extends a different segment of that flow. Changes are additive — existing behavior is preserved via fallback paths.

**Tech Stack:** Java 21, Quarkus 3.32.2, eidos-api, CDI, tmux, Mockito/AssertJ

## Global Constraints

- Java 21 API surface (compiled on Java 26 JVM)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Use `mvn` not `./mvnw`
- Docker required for dev/test (PostgreSQL Dev Services)
- Protocol: `engine-cdi-exclude-types-sync` — new `@ApplicationScoped` beans must be synced across production and test profile exclude-types
- Protocol: `worker-session-creation` — pool sessions use `createWorkerSession()`, not `createSession()`
- All commits reference an issue: `Refs #N` or `Closes #N`
- Test conventions: plain JUnit for unit tests, `@QuarkusTest` + `@TestSecurity` only for HTTP endpoint tests

---

## Batch 1: Session Stamping + Reconciliation (#276 + #277)

### Task 1: Stamp tmux sessions with eidosAgentId (#276)

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/TmuxSessionOperations.java:31-46`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java:46-67` (AgentConfig record)
- Modify: `core/src/main/java/io/casehub/claudony/server/model/Session.java:6-17`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/TmuxSessionOperationsTest.java`
- Modify: `core/src/test/java/io/casehub/claudony/server/model/SessionTest.java` (if exists)

**Interfaces:**
- Consumes: `AgentConfig.eidosAgentId()` (already exists, line 49)
- Produces: `@claudony_eidos_agent_id` tmux user option readable via `TmuxService.getSessionOption()`; `Session.eidosAgentId()` field on the core model

- [ ] **Step 1: Write failing test — TmuxSessionOperations stamps eidosAgentId**

In `TmuxSessionOperationsTest.java`, add a test that creates a session with an `AgentConfig` carrying `eidosAgentId` and verifies the tmux option is set:

```java
@Test
void createStampsEidosAgentId() throws Exception {
    var ops = new TmuxSessionOperations(tmux, "test-", "claude");

    // The create method currently takes (identity, workingDir, command)
    // but doesn't receive AgentConfig. We need to add eidosAgentId parameter.
    String sessionId = ops.create("test-identity", "/tmp", "claude", "my-eidos-agent");

    verify(tmux).setSessionOption(eq(sessionId), eq("@claudony_eidos_agent_id"), eq("my-eidos-agent"));
}

@Test
void createSkipsEidosAgentIdWhenNull() throws Exception {
    var ops = new TmuxSessionOperations(tmux, "test-", "claude");

    String sessionId = ops.create("test-identity", "/tmp", "claude", null);

    verify(tmux, never()).setSessionOption(anyString(), eq("@claudony_eidos_agent_id"), anyString());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=TmuxSessionOperationsTest#createStampsEidosAgentId`
Expected: FAIL — no 4-arg `create()` method exists

- [ ] **Step 3: Add eidosAgentId parameter to TmuxSessionOperations.create()**

Add a new overload of `create()` at `TmuxSessionOperations.java:31` that accepts `eidosAgentId`:

```java
@Override
public String create(String identity, String workingDir, String command, String eidosAgentId) {
    String sessionId        = sessionPrefix + UUID.randomUUID().toString().substring(0, 8);
    String conversationUuid = UUID.randomUUID().toString();
    String fullCommand      = command + " --session-id " + conversationUuid;
    try {
        tmux.createWorkerSession(sessionId, workingDir, fullCommand);
        tmux.setSessionOption(sessionId, "@claudony_identity", identity);
        tmux.setSessionOption(sessionId, "@claudony_conversation_id", conversationUuid);
        tmux.setSessionOption(sessionId, "@claudony_working_dir", workingDir);
        tmux.setSessionOption(sessionId, "@claudony_state", "active");
        if (eidosAgentId != null) {
            tmux.setSessionOption(sessionId, "@claudony_eidos_agent_id", eidosAgentId);
        }
        conversationIds.put(sessionId, conversationUuid);
    } catch (IOException | InterruptedException e) {
        throw new RuntimeException("Failed to create session for identity " + identity, e);
    }
    return sessionId;
}
```

Update the existing 3-arg `create(String, String, String)` to delegate:
```java
@Override
public String create(String identity, String workingDir, String command) {
    return create(identity, workingDir, command, null);
}
```

Also add `eidosAgentId` to `SessionOperations` interface:
```java
default String create(String identity, String workingDir, String command, String eidosAgentId) {
    return create(identity, workingDir, command);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=TmuxSessionOperationsTest`
Expected: PASS

- [ ] **Step 5: Add eidosAgentId to Session record and bootstrap**

Add `Optional<String> eidosAgentId` field to `Session` record in `core/src/main/java/io/casehub/claudony/server/model/Session.java`. Update all `Session` constructors, `withStatus()`, and `withLastActive()` to carry the new field. Update `bootstrapFromTmux()` in `TmuxSessionOperations` to read `@claudony_eidos_agent_id` during bootstrap.

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass (fix any Session constructor callers)

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#276): stamp tmux sessions with eidosAgentId

Adds @claudony_eidos_agent_id tmux user option when AgentConfig
carries a non-null eidosAgentId. Adds eidosAgentId field to Session
record. Bootstrap reads the option on restart.

Refs #276"
```

### Task 2: Reconciliation for changed Eidos descriptors (#277)

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java:75-97`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolBridgeTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistryUpdateTest.java`

**Interfaces:**
- Consumes: `EidosPoolConverter.toPoolDefinition()`, `AgentPoolDefinitionRegistry.get()`
- Produces: `AgentPoolDefinitionRegistry.updateAgentConfig(String agentName, AgentConfig newConfig)`, `EidosPoolBridge.lastKnownDefinitions` internal state

- [ ] **Step 1: Write failing test — updateAgentConfig on registry**

In `AgentPoolDefinitionRegistryUpdateTest.java`:

```java
@Test
void updateAgentConfigReplacesConfig() {
    var original = AgentPoolDefinition.builder()
            .agent("reviewer").command("claude").pool().minActive(1).maxActive(5).build();
    registry.register(original, "eidos");

    var newAgentConfig = new AgentPoolDefinition.AgentConfig(
            "reviewer", "/new/dir", WorkingDirPolicy.EXCLUSIVE, "claude --model opus",
            null, Map.of(), Map.of(), "eidos-reviewer");
    registry.updateAgentConfig("reviewer", newAgentConfig);

    var updated = registry.get("reviewer").orElseThrow();
    assertThat(updated.agent().command()).isEqualTo("claude --model opus");
    assertThat(updated.agent().workingDir()).isEqualTo("/new/dir");
    assertThat(updated.pool().minActive()).isEqualTo(1); // pool unchanged
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentPoolDefinitionRegistryUpdateTest#updateAgentConfigReplacesConfig`
Expected: FAIL — `updateAgentConfig` method doesn't exist

- [ ] **Step 3: Implement updateAgentConfig on AgentPoolDefinitionRegistry**

Add to `AgentPoolDefinitionRegistry.java` after `updateBudget()`:

```java
public void updateAgentConfig(String agentName, AgentPoolDefinition.AgentConfig newAgentConfig) {
    definitions.compute(agentName, (k, existing) -> {
        if (existing == null) {throw new IllegalArgumentException("Pool not found: " + k);}
        return new AgentPoolDefinition(newAgentConfig, existing.pool());
    });
    var updated = definitions.get(agentName);
    var ns = registryService.resolve(agentName).map(RegistryEntry::namespace).orElse("fleet");
    registryService.register(toRegistryEntry(updated, ns));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentPoolDefinitionRegistryUpdateTest`
Expected: PASS

- [ ] **Step 5: Write failing tests — reconciliation detects changes**

In `EidosPoolBridgeTest.java`:

```java
@Test
void reconcileDetectsCapacityChange() {
    var original = descriptorWithClaudonyConfig("agent-a", Map.of("minActive", "1", "maxActive", "5"));
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(original, null)));
    bridge.onStartup();
    verify(poolRegistry).register(any(), eq("eidos"));

    // Change descriptor capacity
    var updated = descriptorWithClaudonyConfig("agent-a", Map.of("minActive", "2", "maxActive", "10"));
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(updated, null)));
    when(poolRegistry.names()).thenReturn(Set.of("agent-a"));

    bridge.reconcile();

    verify(poolRegistry).updateCapacity("agent-a", 2, 10);
}

@Test
void reconcileDetectsConfigChange() {
    var original = descriptorWithClaudonyConfig("agent-b", Map.of("command", "claude"));
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(original, null)));
    bridge.onStartup();

    var updated = descriptorWithClaudonyConfig("agent-b", Map.of("command", "claude --model opus"));
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(updated, null)));
    when(poolRegistry.names()).thenReturn(Set.of("agent-b"));

    bridge.reconcile();

    verify(poolRegistry).updateAgentConfig(eq("agent-b"), argThat(c -> c.command().equals("claude --model opus")));
}

@Test
void reconcileNoOpWhenUnchanged() {
    var descriptor = descriptorWithClaudonyConfig("agent-c");
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(descriptor, null)));
    bridge.onStartup();

    when(poolRegistry.names()).thenReturn(Set.of("agent-c"));
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(descriptor, null)));

    bridge.reconcile();

    verify(poolRegistry, never()).updateCapacity(anyString(), anyInt(), anyInt());
    verify(poolRegistry, never()).updateAgentConfig(anyString(), any());
}

@Test
void reconcileLogsMissingDescriptor() {
    var descriptor = descriptorWithClaudonyConfig("agent-d");
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(descriptor, null)));
    bridge.onStartup();

    when(poolRegistry.names()).thenReturn(Set.of("agent-d"));
    when(agentRegistry.find(any(AgentQuery.class)))
            .thenReturn(List.of()); // descriptor gone

    bridge.reconcile(); // should not throw, just log warning
}
```

Add helper method:
```java
private AgentDescriptor descriptorWithClaudonyConfig(String agentId, Map<String, String> config) {
    return AgentDescriptor.builder()
            .agentId(agentId)
            .name(agentId)
            .slot("worker")
            .tenancyId("default")
            .providerConfigs(List.of(new ProviderConfig("claudony", config)))
            .build();
}
```

- [ ] **Step 6: Implement reconciliation changes in EidosPoolBridge**

Add `lastKnownDefinitions` field and update `onStartup()` to populate it. Rewrite `reconcile()` to compare current vs last-known definitions:

```java
private final Map<String, AgentPoolDefinition> lastKnownDefinitions = new ConcurrentHashMap<>();

// In onStartup(), after successful register:
lastKnownDefinitions.put(descriptor.agentId(), definition);

// Rewritten reconcile():
void reconcile() {
    try {
        List<AgentMatch> matches = agentRegistry.find(AgentQuery.all(tenancyId));
        Set<String> currentIds = new HashSet<>();
        int added = 0, updated = 0;

        for (AgentMatch match : matches) {
            AgentDescriptor descriptor = match.descriptor();
            if (EidosPoolConverter.findClaudonyProviderConfig(descriptor).isEmpty()) continue;

            currentIds.add(descriptor.agentId());
            var newDef = EidosPoolConverter.toPoolDefinition(descriptor, config);
            var existing = lastKnownDefinitions.get(descriptor.agentId());

            if (existing == null) {
                if (registerIfClaudony(descriptor)) {
                    lastKnownDefinitions.put(descriptor.agentId(), newDef);
                    added++;
                }
            } else if (!newDef.pool().equals(existing.pool()) || !newDef.agent().equals(existing.agent())) {
                if (newDef.pool().minActive() != existing.pool().minActive()
                    || newDef.pool().maxActive() != existing.pool().maxActive()) {
                    poolRegistry.updateCapacity(descriptor.agentId(), newDef.pool().minActive(), newDef.pool().maxActive());
                }
                if (!newDef.agent().equals(existing.agent())) {
                    poolRegistry.updateAgentConfig(descriptor.agentId(), newDef.agent());
                }
                lastKnownDefinitions.put(descriptor.agentId(), newDef);
                updated++;
            }
        }

        // Log missing descriptors (present in lastKnown but absent from current)
        for (String knownId : lastKnownDefinitions.keySet()) {
            if (!currentIds.contains(knownId)) {
                LOG.warnf("Eidos pool bridge: descriptor '%s' no longer in registry (pool retained)", knownId);
            }
        }

        if (added > 0 || updated > 0) {
            LOG.infof("Eidos pool bridge: reconciliation — added %d, updated %d pool(s)", added, updated);
        }
    } catch (Exception e) {
        LOG.warn("Eidos pool bridge: reconciliation failed", e);
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EidosPoolBridgeTest`
Expected: PASS

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#277): reconciliation for changed Eidos descriptors

EidosPoolBridge.reconcile() now detects when a registered pool's config
diverges from the current AgentDescriptor and updates capacity and agent
config in place. Running sessions unaffected. Missing descriptors logged
as warnings (pools retained). Adds updateAgentConfig() to registry.

Closes #277"
```

---

## Batch 2: System Prompt Injection (#278)

### Task 3: Eidos-driven system prompt injection (#278)

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/EidosIdentityPromptTemplate.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerContextProvider.java:24-106`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/EidosIdentityPromptTemplateTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerContextProviderTest.java`

**Interfaces:**
- Consumes: `AgentRegistry.findById(agentId, tenancyId)` from eidos-api, `Session.eidosAgentId()` from Task 1
- Produces: `EidosIdentityPromptTemplate.generate(AgentDescriptor)` returns `Optional<String>`, identity prompt injected via `props.put("systemPrompt", ...)` while mesh prompt moves to `props.put("appendSystemPrompt", ...)`

- [ ] **Step 1: Write failing tests — EidosIdentityPromptTemplate**

Create `EidosIdentityPromptTemplateTest.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.eidos.api.AgentConstraint;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.Visibility;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class EidosIdentityPromptTemplateTest {

    @Test
    void fullPromptWithBriefingGoalsConstraints() {
        var descriptor = AgentDescriptor.builder()
                .agentId("code-reviewer")
                .name("Code Reviewer")
                .slot("worker")
                .tenancyId("default")
                .briefing("You review code for quality and correctness.")
                .goals(List.of(
                        AgentGoal.builder().name("quality").description("Ensure code quality").visibility(Visibility.PUBLIC).build(),
                        AgentGoal.builder().name("internal").description("Track metrics").visibility(Visibility.PRIVATE).build()))
                .constraints(List.of(
                        AgentConstraint.builder().name("no-force-push").description("Never force push").visibility(Visibility.PUBLIC).build()))
                .build();

        var result = EidosIdentityPromptTemplate.generate(descriptor);

        assertThat(result).isPresent();
        var prompt = result.get();
        assertThat(prompt).contains("# Identity");
        assertThat(prompt).contains("You review code for quality and correctness.");
        assertThat(prompt).contains("## Goals");
        assertThat(prompt).contains("quality — Ensure code quality");
        assertThat(prompt).doesNotContain("internal"); // private goal excluded
        assertThat(prompt).contains("## Constraints");
        assertThat(prompt).contains("no-force-push — Never force push");
    }

    @Test
    void briefingOnlyNoGoalsNoConstraints() {
        var descriptor = AgentDescriptor.builder()
                .agentId("simple-agent")
                .name("Simple Agent")
                .slot("worker")
                .tenancyId("default")
                .briefing("A simple agent.")
                .build();

        var result = EidosIdentityPromptTemplate.generate(descriptor);

        assertThat(result).isPresent();
        var prompt = result.get();
        assertThat(prompt).contains("A simple agent.");
        assertThat(prompt).doesNotContain("## Goals");
        assertThat(prompt).doesNotContain("## Constraints");
    }

    @Test
    void nullDescriptorReturnsEmpty() {
        assertThat(EidosIdentityPromptTemplate.generate(null)).isEmpty();
    }

    @Test
    void noBriefingReturnsEmpty() {
        var descriptor = AgentDescriptor.builder()
                .agentId("no-briefing")
                .name("No Briefing")
                .slot("worker")
                .tenancyId("default")
                .build();

        assertThat(EidosIdentityPromptTemplate.generate(descriptor)).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EidosIdentityPromptTemplateTest`
Expected: FAIL — class doesn't exist

- [ ] **Step 3: Implement EidosIdentityPromptTemplate**

Create `casehub/src/main/java/io/casehub/claudony/casehub/EidosIdentityPromptTemplate.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.eidos.api.AgentDescriptor;
import java.util.Optional;

class EidosIdentityPromptTemplate {

    static Optional<String> generate(AgentDescriptor descriptor) {
        if (descriptor == null) return Optional.empty();
        if (descriptor.briefing() == null || descriptor.briefing().isBlank()) return Optional.empty();

        var sb = new StringBuilder();
        sb.append("# Identity\n\n");
        sb.append(descriptor.briefing()).append("\n");

        var publicGoals = descriptor.publicGoals();
        if (!publicGoals.isEmpty()) {
            sb.append("\n## Goals\n\n");
            for (var goal : publicGoals) {
                sb.append("- ").append(goal.name()).append(" — ").append(goal.description()).append("\n");
            }
        }

        var publicConstraints = descriptor.publicConstraints();
        if (!publicConstraints.isEmpty()) {
            sb.append("\n## Constraints\n\n");
            for (var constraint : publicConstraints) {
                sb.append("- ").append(constraint.name()).append(" — ").append(constraint.description()).append("\n");
            }
        }

        return Optional.of(sb.toString());
    }
}
```

- [ ] **Step 4: Run template tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EidosIdentityPromptTemplateTest`
Expected: PASS

- [ ] **Step 5: Write failing test — ClaudonyWorkerContextProvider with Eidos identity**

Add to `ClaudonyWorkerContextProviderTest.java`:

```java
@Test
void buildContextWithEidosAgentIdResolvesIdentityPrompt() {
    var descriptor = AgentDescriptor.builder()
            .agentId("reviewer")
            .name("Reviewer")
            .slot("worker")
            .tenancyId("default")
            .briefing("You are a code reviewer.")
            .build();
    when(agentRegistry.findById("reviewer", "default"))
            .thenReturn(Optional.of(descriptor));

    // Need to add eidosAgentId to the provider's context
    var context = provider.buildContext("worker-1", caseId, task, "reviewer");

    assertThat(context.properties().get("systemPrompt").toString())
            .contains("# Identity")
            .contains("You are a code reviewer.");
    assertThat(context.properties().get("appendSystemPrompt")).isNotNull();
}
```

- [ ] **Step 6: Implement Eidos identity in ClaudonyWorkerContextProvider**

Add `Instance<AgentRegistry>` injection to `ClaudonyWorkerContextProvider`:

```java
@Inject
Instance<AgentRegistry> agentRegistryInstance;

@ConfigProperty(name = "claudony.eidos.pool-bridge.tenancy-id", defaultValue = "default")
String tenancyId;
```

Update `buildContext()` to accept `eidosAgentId` parameter and `assemble()` to handle prompt layering:

In `assemble()`, when an Eidos identity prompt is available:
- Set `props.put("systemPrompt", identityPrompt)` for the Eidos identity
- Set `props.put("appendSystemPrompt", meshPrompt)` for the mesh context

When no Eidos identity:
- Keep current behavior: `props.put("systemPrompt", meshPrompt)`

- [ ] **Step 7: Run all context provider tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyWorkerContextProviderTest`
Expected: PASS

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#278): Eidos-driven system prompt injection

ClaudonyWorkerContextProvider resolves AgentDescriptor from the
session's eidosAgentId and injects briefing/goals/constraints as the
primary system prompt. Mesh prompt moves to --append-system-prompt.
Graceful fallback when Eidos is disabled or descriptor not found.

Closes #278"
```

---

## Batch 3: Model Resolution (#279)

### Task 4: ModelRegistry resolution for capability ModelQuery (#279)

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolConverter.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java` (inject converter)
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolConverterTest.java`

**Interfaces:**
- Consumes: `ModelRegistry.query(ModelQuery)` from platform-api (optional), `AgentCapability.model()` / `AgentCapability.modelRef()`
- Produces: `EidosPoolConverter` becomes `@ApplicationScoped` CDI bean with optional `Instance<ModelRegistry>` injection

- [ ] **Step 1: Write failing tests — ModelRegistry resolution**

Add to `EidosPoolConverterTest.java`:

```java
@Test
void withRegistryResolvesViaQuery() {
    var modelRegistry = mock(ModelRegistry.class);
    var converter = new EidosPoolConverter(modelRegistry);
    var descriptor = descriptorWithModel("reviewer", ModelQuery.family("opus"));
    when(modelRegistry.query(any(ModelQuery.class)))
            .thenReturn(Optional.of(ModelChain.of("claude-opus-4-6")));

    var definition = converter.toPoolDefinition(descriptor, EidosPoolBridgeConfig.DEFAULTS);

    assertThat(definition.agent().modelChain().models().get(0).toString())
            .contains("claude-opus-4-6");
}

@Test
void withoutRegistryFallsBackToDirectMapping() {
    var converter = new EidosPoolConverter(); // no registry
    var descriptor = descriptorWithModel("reviewer", ModelQuery.family("opus"));

    var definition = converter.toPoolDefinition(descriptor, EidosPoolBridgeConfig.DEFAULTS);

    assertThat(definition.agent().modelChain()).isNotNull();
}

@Test
void registryReturnsEmptyFallsBack() {
    var modelRegistry = mock(ModelRegistry.class);
    var converter = new EidosPoolConverter(modelRegistry);
    var descriptor = descriptorWithModel("reviewer", ModelQuery.family("opus"));
    when(modelRegistry.query(any(ModelQuery.class))).thenReturn(Optional.empty());

    var definition = converter.toPoolDefinition(descriptor, EidosPoolBridgeConfig.DEFAULTS);

    assertThat(definition.agent().modelChain()).isNotNull(); // falls back to direct
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EidosPoolConverterTest#withRegistryResolvesViaQuery`
Expected: FAIL — no constructor accepting ModelRegistry

- [ ] **Step 3: Convert EidosPoolConverter to CDI bean**

Change `EidosPoolConverter` from `final class` with private constructor to `@ApplicationScoped` bean with optional `Instance<ModelRegistry>`:

```java
@ApplicationScoped
@IfBuildProperty(name = "claudony.eidos.pool-bridge.enabled", stringValue = "true")
public class EidosPoolConverter {

    private static final Logger LOG = Logger.getLogger(EidosPoolConverter.class);
    private static final String PROVIDER_NAME = "claudony";

    private final Instance<ModelRegistry> modelRegistry;

    @Inject
    public EidosPoolConverter(Instance<ModelRegistry> modelRegistry) {
        this.modelRegistry = modelRegistry;
    }

    EidosPoolConverter() {
        this.modelRegistry = null;
    }

    EidosPoolConverter(ModelRegistry registry) {
        // Test constructor — wrap in a satisfied Instance
        this.modelRegistry = new io.quarkus.arc.InjectableInstance.Satisfied<>(registry);
    }

    public AgentPoolDefinition toPoolDefinition(AgentDescriptor descriptor,
                                                 EidosPoolBridgeConfig defaults) {
        // ... existing logic, but resolveModelChain uses modelRegistry ...
    }

    ModelChain resolveModelChain(AgentDescriptor descriptor) {
        return descriptor.capabilities().stream()
                .filter(cap -> cap.model() != null || cap.modelRef() != null)
                .findFirst()
                .map(this::toModelChain)
                .orElse(null);
    }

    private ModelChain toModelChain(AgentCapability capability) {
        if (capability.modelRef() != null) {
            return ModelChain.of(capability.modelRef());
        }
        if (modelRegistry != null && modelRegistry.isResolvable()) {
            var resolved = modelRegistry.get().query(capability.model());
            if (resolved.isPresent()) {
                return resolved.get();
            }
        }
        return ModelChain.of(List.of(new ModelChain.ModelChainEntry.Queried(capability.model())));
    }
}
```

Update `EidosPoolBridge` to inject `EidosPoolConverter` instead of calling static methods.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EidosPoolConverterTest`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#279): ModelRegistry resolution for capability ModelQuery

EidosPoolConverter becomes a CDI bean with optional ModelRegistry
injection. When present, resolves ModelQuery via registry with ranked
results. Falls back to direct family mapping when registry is absent
or returns empty.

Closes #279"
```

---

## Batch 4: Multi-Tenant Pool Isolation (#280)

### Task 5: Multi-tenant pool isolation (#280)

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolKey.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistryTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistryUpdateTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolBridgeTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor.tenancyId()` from eidos-api
- Produces: `PoolKey(agentName, tenancyId)` composite key, all registry methods gain `tenancyId` parameter, existing single-arg methods become convenience overloads passing `"default"`

- [ ] **Step 1: Write failing test — composite key isolation**

In `AgentPoolDefinitionRegistryTest.java`:

```java
@Test
void samAgentDifferentTenantsSeparatePools() {
    var defA = AgentPoolDefinition.builder()
            .agent("reviewer").command("claude").pool().minActive(1).maxActive(5).build();
    var defB = AgentPoolDefinition.builder()
            .agent("reviewer").command("claude --model opus").pool().minActive(2).maxActive(8).build();

    registry.register(defA, "eidos", "tenant-a");
    registry.register(defB, "eidos", "tenant-b");

    assertThat(registry.get("reviewer", "tenant-a")).isPresent();
    assertThat(registry.get("reviewer", "tenant-b")).isPresent();
    assertThat(registry.get("reviewer", "tenant-a").get().agent().command()).isEqualTo("claude");
    assertThat(registry.get("reviewer", "tenant-b").get().agent().command()).isEqualTo("claude --model opus");
}

@Test
void defaultTenantBackwardsCompat() {
    var def = AgentPoolDefinition.builder()
            .agent("reviewer").command("claude").pool().minActive(1).maxActive(5).build();
    registry.register(def, "eidos"); // no tenancyId → "default"

    assertThat(registry.get("reviewer")).isPresent(); // single-arg
    assertThat(registry.get("reviewer", "default")).isPresent(); // explicit default
}

@Test
void namesFilteredByTenant() {
    var defA = AgentPoolDefinition.builder()
            .agent("reviewer").command("claude").pool().minActive(1).maxActive(5).build();
    var defB = AgentPoolDefinition.builder()
            .agent("coder").command("claude").pool().minActive(0).maxActive(3).build();

    registry.register(defA, "eidos", "tenant-a");
    registry.register(defB, "eidos", "tenant-b");

    assertThat(registry.names("tenant-a")).containsExactly("reviewer");
    assertThat(registry.names("tenant-b")).containsExactly("coder");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentPoolDefinitionRegistryTest#samAgentDifferentTenantsSeparatePools`
Expected: FAIL — no multi-arg methods exist

- [ ] **Step 3: Create PoolKey record**

Create `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolKey.java`:

```java
package io.casehub.claudony.casehub.fleet;

import java.util.Objects;

record PoolKey(String agentName, String tenancyId) {
    PoolKey {
        Objects.requireNonNull(agentName, "agentName is required");
        Objects.requireNonNull(tenancyId, "tenancyId is required");
    }

    static String poolName(String agentName, String tenancyId) {
        return "default".equals(tenancyId) ? agentName : agentName + "@" + tenancyId;
    }
}
```

- [ ] **Step 4: Refactor AgentPoolDefinitionRegistry to use PoolKey**

Change internal map from `Map<String, AgentPoolDefinition>` to `Map<PoolKey, AgentPoolDefinition>`. Add tenant-aware overloads for all public methods. Existing single-arg methods delegate with `"default"`:

```java
private final Map<PoolKey, AgentPoolDefinition> definitions = new ConcurrentHashMap<>();

public void register(AgentPoolDefinition definition, String namespace) {
    register(definition, namespace, "default");
}

public void register(AgentPoolDefinition definition, String namespace, String tenancyId) {
    var key = new PoolKey(definition.agent().name(), tenancyId);
    var prev = definitions.putIfAbsent(key, definition);
    if (prev != null) {
        throw new IllegalStateException(
                "Pool definition already registered for agent: " + PoolKey.poolName(key.agentName(), key.tenancyId()));
    }
    registryService.register(toRegistryEntry(definition, namespace));
}

public Optional<AgentPoolDefinition> get(String agentName) {
    return get(agentName, "default");
}

public Optional<AgentPoolDefinition> get(String agentName, String tenancyId) {
    return Optional.ofNullable(definitions.get(new PoolKey(agentName, tenancyId)));
}

public Set<String> names() {
    return names("default");
}

public Set<String> names(String tenancyId) {
    return definitions.keySet().stream()
            .filter(k -> k.tenancyId().equals(tenancyId))
            .map(PoolKey::agentName)
            .collect(Collectors.toSet());
}

// Similar overloads for all(), updateCapacity(), updateScaling(), updateBudget(), updateAgentConfig()
```

- [ ] **Step 5: Update EidosPoolBridge to pass tenancyId**

The bridge already has `this.tenancyId`. Pass it through to all registry operations:
```java
poolRegistry.register(definition, "eidos", tenancyId);
poolRegistry.names(tenancyId);
poolRegistry.updateCapacity(descriptor.agentId(), tenancyId, ...);
```

- [ ] **Step 6: Run all registry and bridge tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=AgentPoolDefinitionRegistryTest,AgentPoolDefinitionRegistryUpdateTest,EidosPoolBridgeTest`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass (fix any callers of old single-arg methods)

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#280): multi-tenant pool isolation

AgentPoolDefinitionRegistry now uses composite key (agentName, tenancyId).
Same agent in different tenancies gets separate pools. Pool names become
'agentName@tenancyId' for non-default tenants. Single-arg convenience
methods pass 'default' for backwards compatibility.

Closes #280"
```

---

## Batch 5: CDI Events (#281)

### Task 6: CDI events for AgentDescriptor changes (#281)

**Blocked by:** casehubio/eidos#187 — implement only after eidos-api publishes event records.

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/EidosPoolBridgeTest.java`

**Interfaces:**
- Consumes: `AgentDescriptorRegistered`, `AgentDescriptorUpdated`, `AgentDescriptorRemoved` from eidos-api (eidos#187)
- Produces: Event observer methods on `EidosPoolBridge` — immediate pool registration/update on events; polling retained as safety net

- [ ] **Step 1: Write failing tests — event observers**

Add to `EidosPoolBridgeTest.java`:

```java
@Test
void registeredEventTriggersPoolCreation() {
    var descriptor = descriptorWithClaudonyConfig("event-agent");
    var event = new AgentDescriptorRegistered(descriptor);

    bridge.onRegistered(event);

    verify(poolRegistry).register(argThat(def ->
            def.agent().name().equals("event-agent")), eq("eidos"), eq(TENANCY_ID));
}

@Test
void updatedEventTriggersReconcile() {
    var previous = descriptorWithClaudonyConfig("event-agent", Map.of("command", "claude"));
    var current = descriptorWithClaudonyConfig("event-agent", Map.of("command", "claude --model opus"));
    bridge.onRegistered(new AgentDescriptorRegistered(previous));

    bridge.onUpdated(new AgentDescriptorUpdated(previous, current));

    verify(poolRegistry).updateAgentConfig(eq("event-agent"), eq(TENANCY_ID),
            argThat(c -> c.command().equals("claude --model opus")));
}

@Test
void removedEventLogsWarning() {
    var descriptor = descriptorWithClaudonyConfig("event-agent");
    bridge.onRegistered(new AgentDescriptorRegistered(descriptor));

    bridge.onRemoved(new AgentDescriptorRemoved("event-agent", TENANCY_ID));

    verify(poolRegistry, never()).remove(anyString()); // no auto-destroy
}
```

- [ ] **Step 2: Implement event observer methods**

Add to `EidosPoolBridge.java`:

```java
void onRegistered(@ObservesAsync AgentDescriptorRegistered event) {
    if (EidosPoolConverter.findClaudonyProviderConfig(event.descriptor()).isPresent()) {
        registerIfClaudony(event.descriptor());
    }
}

void onUpdated(@ObservesAsync AgentDescriptorUpdated event) {
    if (EidosPoolConverter.findClaudonyProviderConfig(event.current()).isPresent()) {
        var newDef = converter.toPoolDefinition(event.current(), config);
        var existing = lastKnownDefinitions.get(event.current().agentId());
        if (existing != null && !newDef.equals(existing)) {
            // Same reconcile logic as polling path
            reconcilePool(event.current().agentId(), existing, newDef);
        }
    }
}

void onRemoved(@ObservesAsync AgentDescriptorRemoved event) {
    LOG.warnf("Eidos pool bridge: descriptor '%s' removed (pool retained)", event.agentId());
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EidosPoolBridgeTest`
Expected: PASS

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#281): CDI events for AgentDescriptor changes

EidosPoolBridge observes AgentDescriptorRegistered/Updated/Removed
CDI events for immediate pool registration/update. Polling
reconciliation retained as drift safety net.

Closes #281"
```

---

## References

- [2026-10-14-eidos-pool-bridge-hardening-design.md] — design spec this plan implements
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java:21-113` — existing bridge
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolConverter.java:13-77` — existing converter
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java:14-107` — registry (flat key)
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/TmuxSessionOperations.java:10-141` — session creation
- `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerContextProvider.java:24-106` — context provider
- `casehub/src/main/java/io/casehub/claudony/casehub/MeshSystemPromptTemplate.java:10-95` — prompt template pattern
- `core/src/main/java/io/casehub/claudony/server/model/Session.java:6-28` — session model record
- Protocol: engine-cdi-exclude-types-sync — test profile sync
- Protocol: worker-session-creation — createWorkerSession for pool sessions
- casehubio/eidos#187 — upstream CDI events
- GitHub #275, #276, #277, #278, #279, #280, #281
