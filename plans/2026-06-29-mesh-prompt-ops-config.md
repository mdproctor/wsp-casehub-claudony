# Mesh System Prompt Delivery + OpsProviderConfigSource Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver mesh system prompts to Claude CLI during worker provisioning (#163), and add ProvisionerConfigRegistry SPI infrastructure for ops-driven agent config (#164).

**Architecture:** The provisioner extracts the mesh prompt from `ProvisionContext.workerContext().properties("systemPrompt")` and passes it to `WorkerCommandBuilder` as a separate dynamic parameter. The builder emits both `--system-prompt` and `--append-system-prompt` when present (removes mutual exclusion). For #164, a `ProvisionerConfigRegistry` SPI in engine-api provides a shared config lookup, and `CompositeProviderConfigSource` in Claudony delegates to it with config-mapping fallback.

**Tech Stack:** Java 21, Quarkus 3.32.2, AssertJ, Mockito, JUnit 5

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Use `mvn` not `./mvnw`
- Engine repo: `/Users/mdproctor/claude/casehub/engine`
- Claudony repo: `/Users/mdproctor/claude/casehub/claudony`
- Workspace repo: `/Users/mdproctor/claude/public/casehub/claudony`
- Claudony casehub module: `casehub/` (package `io.casehub.claudony.casehub`)
- All commits reference an issue: `Refs #163` or `Refs #164`
- Spec: `~/claude/public/casehub/claudony/specs/2026-06-29-mesh-prompt-delivery-ops-config-design.md`

---

### Task 1: ProvisionerConfigRegistry SPI in engine-api

**Files:**
- Create: `/Users/mdproctor/claude/casehub/engine/api/src/main/java/io/casehub/api/spi/ProvisionerConfigRegistry.java`
- Create: `/Users/mdproctor/claude/casehub/engine/runtime/src/main/java/io/casehub/engine/internal/worker/NoOpProvisionerConfigRegistry.java`

**Interfaces:**
- Produces: `ProvisionerConfigRegistry.configFor(String providerName, String agentId) → Map<String, Object>` and `declaredAgentIds(String providerName) → Set<String>` — consumed by Claudony Task 5 and future ops/openclaw implementations.

- [ ] **Step 1: Create SPI interface in engine-api**

```java
// api/src/main/java/io/casehub/api/spi/ProvisionerConfigRegistry.java
package io.casehub.api.spi;

import java.util.Map;
import java.util.Set;

public interface ProvisionerConfigRegistry {
    Map<String, Object> configFor(String providerName, String agentId);
    Set<String> declaredAgentIds(String providerName);
}
```

- [ ] **Step 2: Create NoOp @DefaultBean in engine-runtime**

Follow the established pattern from `NoOpReactiveWorkerProvisioner` in the same package:

```java
// runtime/src/main/java/io/casehub/engine/internal/worker/NoOpProvisionerConfigRegistry.java
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

- [ ] **Step 3: Build and install locally**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml`

Expected: BUILD SUCCESS. The SNAPSHOT now includes `ProvisionerConfigRegistry`.

- [ ] **Step 4: Commit to engine repo**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  api/src/main/java/io/casehub/api/spi/ProvisionerConfigRegistry.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpProvisionerConfigRegistry.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#584): ProvisionerConfigRegistry SPI + NoOp @DefaultBean

Shared provisioner config lookup — configFor(providerName, agentId).
NoOp returns empty; displaced by @Alternative when ops is co-deployed.

Refs engine#584"
```

---

### Task 2: WorkerCommandBuilder — remove mutual exclusion, add dynamic prompt

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/WorkerCommandBuilder.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/WorkerCommandBuilderTest.java`

**Interfaces:**
- Consumes: `ClaudonyProviderConfig` (unchanged)
- Produces: `WorkerCommandBuilder.build(String baseCommand, ClaudonyProviderConfig config, Optional<String> dynamicAppendPrompt) → String` — consumed by Task 4 provisioner changes.

- [ ] **Step 1: Write failing tests for the new 3-arg build() signature**

Add to `WorkerCommandBuilderTest.java`:

```java
@Test
void dynamicPromptOnly_emitsAppendSystemPrompt() {
    String cmd = WorkerCommandBuilder.build("claude", ClaudonyProviderConfig.EMPTY,
            Optional.of("You are on case XYZ"));
    assertThat(cmd).isEqualTo("claude --append-system-prompt 'You are on case XYZ'");
}

@Test
void dynamicAndStaticAppend_mergedStaticFirst() {
    var config = configWith(b -> b.appendSystemPrompt = Optional.of("Always write tests"));
    String cmd = WorkerCommandBuilder.build("claude", config,
            Optional.of("Mesh channels: work, observe"));
    assertThat(cmd).contains("--append-system-prompt 'Always write tests\n\nMesh channels: work, observe'");
}

@Test
void systemPromptAndDynamic_bothEmitted() {
    var config = configWith(b -> b.systemPrompt = Optional.of("Custom persona"));
    String cmd = WorkerCommandBuilder.build("claude", config,
            Optional.of("Mesh context"));
    assertThat(cmd).contains("--system-prompt 'Custom persona'");
    assertThat(cmd).contains("--append-system-prompt 'Mesh context'");
}

@Test
void allThreePromptSources_allEmitted() {
    var config = new ClaudonyProviderConfig(
            Optional.empty(), Optional.empty(),
            Optional.of("Operator append"), Optional.of("Custom persona"),
            Optional.empty(), Optional.empty(), Optional.empty(), Optional.empty(),
            Optional.empty(), Optional.empty(), Optional.empty());
    String cmd = WorkerCommandBuilder.build("claude", config,
            Optional.of("Mesh context"));
    assertThat(cmd).contains("--system-prompt 'Custom persona'");
    assertThat(cmd).contains("--append-system-prompt 'Operator append\n\nMesh context'");
}

@Test
void emptyDynamicPrompt_noExtraFlag() {
    String cmd = WorkerCommandBuilder.build("claude", ClaudonyProviderConfig.EMPTY, Optional.empty());
    assertThat(cmd).isEqualTo("claude");
}

@Test
void multiLineMeshPrompt_shellQuotingSurvives() {
    String mesh = "ROLE: agent\n\nMESH CHANNELS:\n  work: case-123/work\n\n"
            + "STARTUP:\n  1. register(\"worker-abc\", \"Starting\", [\"agent\"])\n";
    String cmd = WorkerCommandBuilder.build("claude", ClaudonyProviderConfig.EMPTY,
            Optional.of(mesh));
    assertThat(cmd).contains("--append-system-prompt");
    assertThat(cmd).doesNotContain("\"");
    // Shell quoting wraps the entire value in single quotes
    assertThat(cmd).contains("'ROLE: agent");
}
```

- [ ] **Step 2: Update existing mutual-exclusion test**

Replace `systemPromptWinsOverAppend_builderPolicy` with a test that asserts coexistence:

```java
@Test
void systemPromptAndAppendSystemPrompt_bothEmitted() {
    var config = new ClaudonyProviderConfig(
            Optional.empty(), Optional.empty(),
            Optional.of("Append this"), Optional.of("Replace entirely"),
            Optional.empty(), Optional.empty(), Optional.empty(), Optional.empty(),
            Optional.empty(), Optional.empty(), Optional.empty());
    String cmd = WorkerCommandBuilder.build("claude", config, Optional.empty());
    assertThat(cmd).contains("--system-prompt 'Replace entirely'");
    assertThat(cmd).contains("--append-system-prompt 'Append this'");
}
```

- [ ] **Step 3: Update all existing test calls to 3-arg signature**

Every existing call to `WorkerCommandBuilder.build("claude", config)` must become `WorkerCommandBuilder.build("claude", config, Optional.empty())`. Update: `emptyConfig_returnsBaseCommandUnchanged`, `singleFlag_model`, `systemPrompt_withSpacesAndQuotes`, `tools_commaJoinedAndQuoted`, `toolPattern_shellMetacharactersSurvive`, `addDirs_repeatedPerEntry`, `allFlagsPopulated`, `baseCommandWithExistingFlags_preserved`.

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=WorkerCommandBuilderTest`

Expected: Compilation failure — old 2-arg `build()` still exists, new 3-arg does not.

- [ ] **Step 5: Implement the new WorkerCommandBuilder**

Replace `WorkerCommandBuilder.java` with:

```java
package io.casehub.claudony.casehub;

import java.util.List;
import java.util.Optional;

public final class WorkerCommandBuilder {

    private WorkerCommandBuilder() {}

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

    static Optional<String> mergeAppendPrompts(Optional<String> staticAppend,
                                                Optional<String> dynamicAppend) {
        if (staticAppend.isEmpty() && dynamicAppend.isEmpty()) return Optional.empty();
        if (staticAppend.isEmpty()) return dynamicAppend;
        if (dynamicAppend.isEmpty()) return staticAppend;
        return Optional.of(staticAppend.get() + "\n\n" + dynamicAppend.get());
    }

    private static void appendString(StringBuilder sb, String flag, Optional<String> value) {
        value.ifPresent(v -> sb.append(' ').append(flag).append(' ').append(shellQuote(v)));
    }

    private static void appendList(StringBuilder sb, String flag, Optional<List<String>> value) {
        value.ifPresent(list -> {
            if (!list.isEmpty()) {
                sb.append(' ').append(flag).append(' ').append(shellQuote(String.join(",", list)));
            }
        });
    }

    static String shellQuote(String value) {
        return "'" + value.replace("'", "'\\''") + "'";
    }
}
```

- [ ] **Step 6: Fix remaining compilation errors**

The provisioner (`ClaudonyReactiveWorkerProvisioner.setupSession()` line 193) still calls the old 2-arg signature. Temporarily update to pass `Optional.empty()` as the third argument so the module compiles:

```java
String enrichedCommand = WorkerCommandBuilder.build(baseCommand, config, Optional.empty());
```

This is a transitional fix — Task 4 replaces it with the real mesh prompt extraction.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=WorkerCommandBuilderTest`

Expected: All tests PASS.

- [ ] **Step 8: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/WorkerCommandBuilder.java \
       casehub/src/test/java/io/casehub/claudony/casehub/WorkerCommandBuilderTest.java
git commit -m "feat(#163): WorkerCommandBuilder — coexist system-prompt + append, add dynamic prompt

Remove mutual exclusion between --system-prompt and --append-system-prompt.
Add dynamicAppendPrompt parameter for per-provisioning mesh context.
Static (operator) append first, dynamic (mesh) second in merged output.

Refs #163"
```

---

### Task 3: MeshSystemPromptTemplate — null workerId guard

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/MeshSystemPromptTemplate.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/MeshSystemPromptTemplateTest.java`

**Interfaces:**
- Consumes: nothing new
- Produces: `MeshSystemPromptTemplate.generate()` handles null workerId without producing `"null"` literal — consumed by Task 4 provisioner.

- [ ] **Step 1: Write failing test for null workerId**

Add to `MeshSystemPromptTemplateTest.java`:

```java
@Test
void active_nullWorkerId_producesPlaceholder() {
    String prompt = MeshSystemPromptTemplate.generate(
            null, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
            List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

    assertThat(prompt)
            .contains("register(<your-session-id>")
            .doesNotContain("register(\"null\"");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=MeshSystemPromptTemplateTest#active_nullWorkerId_producesPlaceholder`

Expected: FAIL — current code produces `register("null", ...)`.

- [ ] **Step 3: Implement the null guard**

In `MeshSystemPromptTemplate.java`, extract the registration instruction from `buildActive()` into a helper:

Replace the inline registration line in `buildActive()`:
```java
                + "  1. register(\"" + workerId + "\", \"Starting " + capability + "\", [\"" + capability + "\"])\n"
```

With a call to a new helper:
```java
                + registrationInstruction(workerId, capability)
```

Add the helper method:
```java
private static String registrationInstruction(String workerId, String capability) {
    String id = workerId != null ? "\"" + workerId + "\"" : "<your-session-id>";
    return "  1. register(" + id + ", \"Starting " + capability + "\", [\"" + capability + "\"])\n";
}
```

- [ ] **Step 4: Run tests to verify all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=MeshSystemPromptTemplateTest`

Expected: All 19 tests PASS (18 existing + 1 new).

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/MeshSystemPromptTemplate.java \
       casehub/src/test/java/io/casehub/claudony/casehub/MeshSystemPromptTemplateTest.java
git commit -m "fix(#163): MeshSystemPromptTemplate guards null workerId

Engine passes workerId=null to buildContext() — the ID isn't generated
until provisioning. Now emits <your-session-id> placeholder instead of
literal \"null\" in the register() instruction.

Refs #163"
```

---

### Task 4: Provisioner — extract mesh prompt from WorkerContext

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java`

**Interfaces:**
- Consumes: `WorkerCommandBuilder.build(String, ClaudonyProviderConfig, Optional<String>)` from Task 2. `ProvisionContext.workerContext()` from engine-api.
- Produces: enriched CLI command now includes mesh prompt via `--append-system-prompt`.

- [ ] **Step 1: Write failing tests for mesh prompt extraction**

Add to `ClaudonyReactiveWorkerProvisionerTest.java`. First, add the required import at the top:

```java
import io.casehub.api.model.WorkerContext;
import io.casehub.api.context.PropagationContext;
import java.util.Map;
```

Then add the test helper:

```java
private ProvisionContext provisionContextWithWorkerContext(UUID caseId, Map<String, Object> properties) {
    var wc = new WorkerContext("code-reviewer", caseId, List.of(), List.of(),
            PropagationContext.createRoot(), properties);
    return new ProvisionContext(caseId,
            io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID,
            "code-reviewer", wc, null, null, null);
}
```

Then add the tests:

```java
@Test
void provision_withMeshPrompt_passesItToCommand() throws Exception {
    var caseId = UUID.randomUUID();
    var ctx = provisionContextWithWorkerContext(caseId,
            Map.of("systemPrompt", "You are on case " + caseId));

    provisioner.provision(Set.of("code-reviewer"), ctx)
            .await().indefinitely();

    var captor = ArgumentCaptor.forClass(String.class);
    verify(tmux).createWorkerSession(anyString(), anyString(), captor.capture());
    assertThat(captor.getValue()).contains("--append-system-prompt");
    assertThat(captor.getValue()).contains("You are on case " + caseId);
}

@Test
void provision_withNullWorkerContext_noAppendFlag() throws Exception {
    provisioner.provision(Set.of("code-reviewer"), provisionContext(UUID.randomUUID()))
            .await().indefinitely();

    var captor = ArgumentCaptor.forClass(String.class);
    verify(tmux).createWorkerSession(anyString(), anyString(), captor.capture());
    assertThat(captor.getValue()).doesNotContain("--append-system-prompt");
}

@Test
void provision_silentParticipation_noSystemPromptProperty_noAppendFlag() throws Exception {
    var caseId = UUID.randomUUID();
    var ctx = provisionContextWithWorkerContext(caseId,
            Map.of("meshParticipation", "SILENT"));

    provisioner.provision(Set.of("code-reviewer"), ctx)
            .await().indefinitely();

    var captor = ArgumentCaptor.forClass(String.class);
    verify(tmux).createWorkerSession(anyString(), anyString(), captor.capture());
    assertThat(captor.getValue()).doesNotContain("--append-system-prompt");
}

@Test
void provision_cleanStart_noSystemPromptProperty_noAppendFlag() throws Exception {
    var caseId = UUID.randomUUID();
    var ctx = provisionContextWithWorkerContext(caseId,
            Map.of("meshParticipation", "ACTIVE", "clean-start", true));

    provisioner.provision(Set.of("code-reviewer"), ctx)
            .await().indefinitely();

    var captor = ArgumentCaptor.forClass(String.class);
    verify(tmux).createWorkerSession(anyString(), anyString(), captor.capture());
    assertThat(captor.getValue()).doesNotContain("--append-system-prompt");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveWorkerProvisionerTest#provision_withMeshPrompt_passesItToCommand`

Expected: FAIL — provisioner currently passes `Optional.empty()` for the dynamic prompt.

- [ ] **Step 3: Implement mesh prompt extraction in setupSession()**

In `ClaudonyReactiveWorkerProvisioner.java`, update `setupSession()`. Replace:

```java
        ClaudonyProviderConfig config = providerConfigSource.forAgent(roleName);
        String baseCommand = config.command().orElse(defaultCommand);
        String enrichedCommand = WorkerCommandBuilder.build(baseCommand, config, Optional.empty());
```

With:

```java
        ClaudonyProviderConfig config = providerConfigSource.forAgent(roleName);
        String baseCommand = config.command().orElse(defaultCommand);

        Optional<String> meshPrompt = Optional.ofNullable(context.workerContext())
                .map(wc -> wc.properties().get("systemPrompt"))
                .filter(String.class::isInstance)
                .map(String.class::cast);

        String enrichedCommand = WorkerCommandBuilder.build(baseCommand, config, meshPrompt);
```

Add the import at the top of the file if not present:

```java
import io.casehub.api.model.WorkerContext;
```

(Note: `WorkerContext` is already imported transitively — check and add only if needed.)

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveWorkerProvisionerTest`

Expected: All tests PASS (existing + 4 new).

- [ ] **Step 5: Run full casehub module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`

Expected: All tests PASS. The `WorkerContext` import and `Optional.empty()` → mesh prompt change should not break any other test.

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java \
       casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java
git commit -m "feat(#163): provisioner delivers mesh system prompt to CLI

Extract systemPrompt from ProvisionContext.workerContext().properties()
and pass to WorkerCommandBuilder as dynamicAppendPrompt. Null-safe for
SILENT participation, clean-start, and null-caseId edge cases.

Closes #163"
```

---

### Task 5: CompositeProviderConfigSource replaces ConfigMappingProviderConfigSource

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/CompositeProviderConfigSource.java`
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSource.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/CompositeProviderConfigSourceTest.java`
- Delete: `casehub/src/test/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSourceTest.java`

**Interfaces:**
- Consumes: `ProvisionerConfigRegistry.configFor(String, String) → Map<String, Object>` from Task 1. `CaseHubConfig.Workers` (unchanged). `ClaudonyProviderConfig.fromMap(Map)` (existing).
- Produces: `ProviderConfigSource` implementation — registry primary, config-mapping fallback. Consumed by the provisioner (no API change — same SPI).

- [ ] **Step 1: Write failing tests for CompositeProviderConfigSource**

Create `casehub/src/test/java/io/casehub/claudony/casehub/CompositeProviderConfigSourceTest.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.spi.ProvisionerConfigRegistry;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class CompositeProviderConfigSourceTest {

    @Test
    void forAgent_registryHasConfig_returnsRegistryConfig() {
        var registry = stubRegistry(Map.of("agent-a", Map.of("model", "opus")));
        var source = composite(registry, Map.of());

        var config = source.forAgent("agent-a");
        assertThat(config.model()).contains("opus");
    }

    @Test
    void forAgent_registryEmpty_fallsBackToConfigMapping() {
        var registry = stubRegistry(Map.of());
        var source = composite(registry, Map.of(
                "agent-b", stubAgentConfig("sonnet", Optional.of("high"))));

        var config = source.forAgent("agent-b");
        assertThat(config.model()).contains("sonnet");
        assertThat(config.effort()).contains("high");
    }

    @Test
    void forAgent_unknownAgent_returnsEmpty() {
        var source = composite(stubRegistry(Map.of()), Map.of());

        assertThat(source.forAgent("unknown")).isEqualTo(ClaudonyProviderConfig.EMPTY);
    }

    @Test
    void forAgent_registryDisplacesConfigMapping() {
        var registry = stubRegistry(Map.of("agent-a", Map.of("model", "opus")));
        var source = composite(registry, Map.of(
                "agent-a", stubAgentConfig("sonnet", Optional.of("low"))));

        var config = source.forAgent("agent-a");
        assertThat(config.model()).contains("opus");
        assertThat(config.effort()).isEmpty();
    }

    @Test
    void declaredAgentIds_unionOfBothSources() {
        var registry = stubRegistry(Map.of("registry-agent", Map.of()));
        var source = composite(registry, Map.of(
                "config-agent", stubAgentConfig("sonnet", Optional.empty())));

        assertThat(source.declaredAgentIds())
                .containsExactlyInAnyOrder("registry-agent", "config-agent");
    }

    @Test
    void declaredAgentIds_registryOnlyAgent_stillDiscovered() {
        var registry = stubRegistry(Map.of("ops-only", Map.of("model", "opus")));
        var source = composite(registry, Map.of());

        assertThat(source.declaredAgentIds()).containsExactly("ops-only");
    }

    @Test
    void noOpRegistry_behavesLikeConfigMappingOnly() {
        var noOp = stubRegistry(Map.of());
        var source = composite(noOp, Map.of(
                "agent-a", stubAgentConfig("opus", Optional.empty())));

        assertThat(source.forAgent("agent-a").model()).contains("opus");
        assertThat(source.declaredAgentIds()).containsExactly("agent-a");
    }

    // --- helpers ---

    private CompositeProviderConfigSource composite(ProvisionerConfigRegistry registry,
                                                      Map<String, CaseHubConfig.AgentProviderConfig> providerConfig) {
        return new CompositeProviderConfigSource(registry, stubWorkersConfig(providerConfig));
    }

    private ProvisionerConfigRegistry stubRegistry(Map<String, Map<String, Object>> data) {
        return new ProvisionerConfigRegistry() {
            @Override
            public Map<String, Object> configFor(String providerName, String agentId) {
                return data.getOrDefault(agentId, Map.of());
            }
            @Override
            public Set<String> declaredAgentIds(String providerName) {
                return data.keySet();
            }
        };
    }

    private CaseHubConfig.Workers stubWorkersConfig(Map<String, CaseHubConfig.AgentProviderConfig> providerConfig) {
        return new CaseHubConfig.Workers() {
            @Override public String defaultCommand() { return "claude"; }
            @Override public String defaultWorkingDir() { return "/tmp"; }
            @Override public Map<String, CaseHubConfig.AgentProviderConfig> providerConfig() { return providerConfig; }
        };
    }

    private CaseHubConfig.AgentProviderConfig stubAgentConfig(String model, Optional<String> effort) {
        return new CaseHubConfig.AgentProviderConfig() {
            @Override public Optional<String> command() { return Optional.empty(); }
            @Override public Optional<String> model() { return Optional.of(model); }
            @Override public Optional<String> appendSystemPrompt() { return Optional.empty(); }
            @Override public Optional<String> systemPrompt() { return Optional.empty(); }
            @Override public Optional<String> effort() { return effort; }
            @Override public Optional<String> permissionMode() { return Optional.empty(); }
            @Override public Optional<List<String>> tools() { return Optional.empty(); }
            @Override public Optional<List<String>> allowedTools() { return Optional.empty(); }
            @Override public Optional<List<String>> disallowedTools() { return Optional.empty(); }
            @Override public Optional<List<String>> addDirs() { return Optional.empty(); }
            @Override public Optional<String> workingDir() { return Optional.empty(); }
        };
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=CompositeProviderConfigSourceTest`

Expected: Compilation failure — `CompositeProviderConfigSource` does not exist yet.

- [ ] **Step 3: Create CompositeProviderConfigSource**

Create `casehub/src/main/java/io/casehub/claudony/casehub/CompositeProviderConfigSource.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.spi.ProvisionerConfigRegistry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import org.jboss.logging.Logger;

@ApplicationScoped
public class CompositeProviderConfigSource implements ProviderConfigSource {

    private static final Logger LOG = Logger.getLogger(CompositeProviderConfigSource.class);
    private static final String PROVIDER_NAME = "claudony";

    private final ProvisionerConfigRegistry registry;
    private final CaseHubConfig.Workers workers;

    @Inject
    public CompositeProviderConfigSource(ProvisionerConfigRegistry registry,
                                          CaseHubConfig config) {
        this(registry, config.workers());
    }

    CompositeProviderConfigSource(ProvisionerConfigRegistry registry,
                                    CaseHubConfig.Workers workers) {
        this.registry = registry;
        this.workers = workers;
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

- [ ] **Step 4: Delete ConfigMappingProviderConfigSource and its test**

```bash
git rm casehub/src/main/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSource.java
git rm casehub/src/test/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSourceTest.java
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=CompositeProviderConfigSourceTest`

Expected: All 7 tests PASS.

- [ ] **Step 6: Run full casehub module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`

Expected: All tests PASS. The old `ConfigMappingProviderConfigSource` was injected in CDI tests via `@DefaultBean` — the new `CompositeProviderConfigSource` as `@ApplicationScoped` replaces it. Existing provisioner tests use inline `ProviderConfigSource` implementations, so they're unaffected.

- [ ] **Step 7: Run full test suite across all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

Expected: 586+ tests PASS. The `NoOpProvisionerConfigRegistry @DefaultBean` from engine-api ensures the composite source falls back to config mapping in all app-module `@QuarkusTest` tests.

- [ ] **Step 8: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/CompositeProviderConfigSource.java \
       casehub/src/test/java/io/casehub/claudony/casehub/CompositeProviderConfigSourceTest.java
git commit -m "feat(#164): CompositeProviderConfigSource — registry primary, config-mapping fallback

Replace ConfigMappingProviderConfigSource with CompositeProviderConfigSource.
Injects ProvisionerConfigRegistry (engine#584) as primary source; falls back
to application.properties when registry returns empty. declaredAgentIds()
returns union of both sources. Warns when registry displaces config-mapping.

Refs #164"
```

---

### Task 6: Final verification and CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — update test count and document new classes

- [ ] **Step 1: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

Expected: All tests PASS. Count the new total.

- [ ] **Step 2: Update CLAUDE.md test baseline**

Update the test count baseline section with the new total and add the new class documentation:
- `WorkerCommandBuilder.build()` now takes 3 args (added `dynamicAppendPrompt`)
- `CompositeProviderConfigSource` replaces `ConfigMappingProviderConfigSource`
- `MeshSystemPromptTemplate` null workerId guard

- [ ] **Step 3: Commit CLAUDE.md**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for #163/#164 — test baseline, new classes

Refs #163, #164"
```
