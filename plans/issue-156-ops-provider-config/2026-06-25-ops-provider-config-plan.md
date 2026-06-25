# Per-Agent Provider Config Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable per-agent CLI configuration (model, system prompt, tools, effort, etc.) at worker provision time, replacing the fragile WorkerCommandResolver with a unified config path keyed by agentId/taskType.

**Architecture:** A `ProviderConfigSource` SPI with `@DefaultBean` implementation reads per-agent config from `application.properties`. The provisioner resolves config by `context.taskType()`, builds an enriched CLI command via `WorkerCommandBuilder`, and passes it to tmux. `WorkerCommandResolver` is removed.

**Tech Stack:** Java 21, Quarkus 3.32.2 (@ConfigMapping, @DefaultBean), SmallRye Config, JUnit 5, AssertJ, Mockito

**Spec:** `specs/issue-156-ops-provider-config/2026-06-25-ops-provider-config-design.md`

## Global Constraints

- Java 21 API surface (compiled on Java 26 JVM)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- All values single-quoted in CLI commands (shell metacharacter safety)
- `context.taskType()` is the agentId key — never use the capabilities set for config lookup
- `@WithDefault("claude")` on `defaultCommand` — always non-null
- Session record must store effective values (enriched command, effective working dir)
- Use `superpowers:test-driven-development` — write failing tests first

---

### Task 1: ClaudonyProviderConfig + WorkerCommandBuilder

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyProviderConfig.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/WorkerCommandBuilder.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyProviderConfigTest.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/WorkerCommandBuilderTest.java`

**Interfaces:**
- Consumes: nothing (leaf types)
- Produces:
  - `ClaudonyProviderConfig` record — used by Task 2 (`ConfigMappingProviderConfigSource`) and Task 3 (provisioner)
  - `ClaudonyProviderConfig.EMPTY` — sentinel for "no config"
  - `ClaudonyProviderConfig.fromMap(Map<String, Object>)` — factory for future ops bridge
  - `ClaudonyProviderConfig.fromConfigMapping(CaseHubConfig.AgentProviderConfig)` — factory for Task 2
  - `WorkerCommandBuilder.build(String baseCommand, ClaudonyProviderConfig config)` — used by Task 3 (provisioner)

- [ ] **Step 1: Write ClaudonyProviderConfigTest**

```java
package io.casehub.claudony.casehub;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;

class ClaudonyProviderConfigTest {

    @Test
    void empty_allFieldsAbsent() {
        var config = ClaudonyProviderConfig.EMPTY;
        assertThat(config.command()).isEmpty();
        assertThat(config.model()).isEmpty();
        assertThat(config.appendSystemPrompt()).isEmpty();
        assertThat(config.systemPrompt()).isEmpty();
        assertThat(config.effort()).isEmpty();
        assertThat(config.permissionMode()).isEmpty();
        assertThat(config.tools()).isEmpty();
        assertThat(config.allowedTools()).isEmpty();
        assertThat(config.disallowedTools()).isEmpty();
        assertThat(config.addDirs()).isEmpty();
        assertThat(config.workingDir()).isEmpty();
    }

    @Test
    void fromMap_allFieldsPopulated() {
        var map = Map.<String, Object>of(
                "command", "claude",
                "model", "opus",
                "appendSystemPrompt", "You are a reviewer",
                "systemPrompt", "Replace default",
                "effort", "high",
                "permissionMode", "auto",
                "tools", List.of("Read", "Bash"),
                "allowedTools", List.of("Bash(git *)"),
                "disallowedTools", List.of("Write"),
                "addDirs", List.of("/tmp/docs", "/tmp/data"));
        var workingDirMap = new java.util.HashMap<>(map);
        workingDirMap.put("workingDir", "/custom/dir");
        var config = ClaudonyProviderConfig.fromMap(workingDirMap);

        assertThat(config.command()).contains("claude");
        assertThat(config.model()).contains("opus");
        assertThat(config.appendSystemPrompt()).contains("You are a reviewer");
        assertThat(config.systemPrompt()).contains("Replace default");
        assertThat(config.effort()).contains("high");
        assertThat(config.permissionMode()).contains("auto");
        assertThat(config.tools()).contains(List.of("Read", "Bash"));
        assertThat(config.allowedTools()).contains(List.of("Bash(git *)"));
        assertThat(config.disallowedTools()).contains(List.of("Write"));
        assertThat(config.addDirs()).contains(List.of("/tmp/docs", "/tmp/data"));
        assertThat(config.workingDir()).contains("/custom/dir");
    }

    @Test
    void fromMap_emptyMap_returnsEmpty() {
        var config = ClaudonyProviderConfig.fromMap(Map.of());
        assertThat(config).isEqualTo(ClaudonyProviderConfig.EMPTY);
    }

    @Test
    void fromMap_listsAreListOfString() {
        var config = ClaudonyProviderConfig.fromMap(Map.of(
                "tools", List.of("Read", "Bash", "Edit")));
        assertThat(config.tools()).contains(List.of("Read", "Bash", "Edit"));
    }

    @Test
    void fromMap_unknownKeysIgnored() {
        var config = ClaudonyProviderConfig.fromMap(Map.of(
                "model", "opus",
                "unknownKey", "ignored",
                "anotherUnknown", 42));
        assertThat(config.model()).contains("opus");
        assertThat(config.command()).isEmpty();
    }

    @Test
    void bothSystemPrompts_recordHoldsBoth() {
        var config = ClaudonyProviderConfig.fromMap(Map.of(
                "systemPrompt", "Replace",
                "appendSystemPrompt", "Append"));
        assertThat(config.systemPrompt()).contains("Replace");
        assertThat(config.appendSystemPrompt()).contains("Append");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyProviderConfigTest`
Expected: Compilation failure — `ClaudonyProviderConfig` does not exist.

- [ ] **Step 3: Implement ClaudonyProviderConfig**

```java
package io.casehub.claudony.casehub;

import java.util.List;
import java.util.Map;
import java.util.Optional;

public record ClaudonyProviderConfig(
        Optional<String> command,
        Optional<String> model,
        Optional<String> appendSystemPrompt,
        Optional<String> systemPrompt,
        Optional<String> effort,
        Optional<String> permissionMode,
        Optional<List<String>> tools,
        Optional<List<String>> allowedTools,
        Optional<List<String>> disallowedTools,
        Optional<List<String>> addDirs,
        Optional<String> workingDir) {

    public static final ClaudonyProviderConfig EMPTY = new ClaudonyProviderConfig(
            Optional.empty(), Optional.empty(), Optional.empty(), Optional.empty(),
            Optional.empty(), Optional.empty(), Optional.empty(), Optional.empty(),
            Optional.empty(), Optional.empty(), Optional.empty());

    @SuppressWarnings("unchecked")
    public static ClaudonyProviderConfig fromMap(Map<String, Object> map) {
        if (map.isEmpty()) return EMPTY;
        return new ClaudonyProviderConfig(
                optString(map, "command"),
                optString(map, "model"),
                optString(map, "appendSystemPrompt"),
                optString(map, "systemPrompt"),
                optString(map, "effort"),
                optString(map, "permissionMode"),
                optList(map, "tools"),
                optList(map, "allowedTools"),
                optList(map, "disallowedTools"),
                optList(map, "addDirs"),
                optString(map, "workingDir"));
    }

    public static ClaudonyProviderConfig fromConfigMapping(CaseHubConfig.AgentProviderConfig cfg) {
        return new ClaudonyProviderConfig(
                cfg.command(), cfg.model(), cfg.appendSystemPrompt(), cfg.systemPrompt(),
                cfg.effort(), cfg.permissionMode(), cfg.tools(), cfg.allowedTools(),
                cfg.disallowedTools(), cfg.addDirs(), cfg.workingDir());
    }

    private static Optional<String> optString(Map<String, Object> map, String key) {
        Object v = map.get(key);
        return v instanceof String s ? Optional.of(s) : Optional.empty();
    }

    @SuppressWarnings("unchecked")
    private static Optional<List<String>> optList(Map<String, Object> map, String key) {
        Object v = map.get(key);
        return v instanceof List<?> list ? Optional.of((List<String>) list) : Optional.empty();
    }
}
```

- [ ] **Step 4: Run ClaudonyProviderConfigTest to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyProviderConfigTest`
Expected: All 6 tests PASS.

- [ ] **Step 5: Write WorkerCommandBuilderTest**

```java
package io.casehub.claudony.casehub;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;

class WorkerCommandBuilderTest {

    @Test
    void emptyConfig_returnsBaseCommandUnchanged() {
        assertThat(WorkerCommandBuilder.build("claude", ClaudonyProviderConfig.EMPTY))
                .isEqualTo("claude");
    }

    @Test
    void singleFlag_model() {
        var config = configWith(b -> b.model = Optional.of("opus"));
        assertThat(WorkerCommandBuilder.build("claude", config))
                .isEqualTo("claude --model 'opus'");
    }

    @Test
    void systemPrompt_withSpacesAndQuotes() {
        var config = configWith(b -> b.appendSystemPrompt = Optional.of("You're a reviewer"));
        assertThat(WorkerCommandBuilder.build("claude", config))
                .isEqualTo("claude --append-system-prompt 'You'\\''re a reviewer'");
    }

    @Test
    void tools_commaJoinedAndQuoted() {
        var config = configWith(b -> b.tools = Optional.of(List.of("Read", "Bash", "Edit")));
        assertThat(WorkerCommandBuilder.build("claude", config))
                .isEqualTo("claude --tools 'Read,Bash,Edit'");
    }

    @Test
    void toolPattern_shellMetacharactersSurvive() {
        var config = configWith(b -> b.allowedTools = Optional.of(List.of("Read", "Bash(git *)", "Edit")));
        assertThat(WorkerCommandBuilder.build("claude", config))
                .isEqualTo("claude --allowedTools 'Read,Bash(git *),Edit'");
    }

    @Test
    void addDirs_repeatedPerEntry() {
        var config = configWith(b -> b.addDirs = Optional.of(List.of("/tmp/docs", "/tmp/data")));
        assertThat(WorkerCommandBuilder.build("claude", config))
                .isEqualTo("claude --add-dir '/tmp/docs' --add-dir '/tmp/data'");
    }

    @Test
    void systemPromptWinsOverAppend_builderPolicy() {
        var config = new ClaudonyProviderConfig(
                Optional.empty(), Optional.empty(),
                Optional.of("Append this"), Optional.of("Replace entirely"),
                Optional.empty(), Optional.empty(), Optional.empty(), Optional.empty(),
                Optional.empty(), Optional.empty(), Optional.empty());
        String cmd = WorkerCommandBuilder.build("claude", config);
        assertThat(cmd).contains("--system-prompt");
        assertThat(cmd).doesNotContain("--append-system-prompt");
    }

    @Test
    void allFlagsPopulated() {
        var config = new ClaudonyProviderConfig(
                Optional.of("claude-code"), Optional.of("opus"),
                Optional.of("Append prompt"), Optional.empty(),
                Optional.of("high"), Optional.of("auto"),
                Optional.of(List.of("Read", "Bash")), Optional.of(List.of("Edit")),
                Optional.of(List.of("Write")),
                Optional.of(List.of("/dir1")), Optional.of("/workspace"));
        String cmd = WorkerCommandBuilder.build("claude", config);
        assertThat(cmd).startsWith("claude ");
        assertThat(cmd).contains("--model 'opus'");
        assertThat(cmd).contains("--append-system-prompt 'Append prompt'");
        assertThat(cmd).contains("--effort 'high'");
        assertThat(cmd).contains("--permission-mode 'auto'");
        assertThat(cmd).contains("--tools 'Read,Bash'");
        assertThat(cmd).contains("--allowedTools 'Edit'");
        assertThat(cmd).contains("--disallowedTools 'Write'");
        assertThat(cmd).contains("--add-dir '/dir1'");
        // workingDir is NOT a CLI flag — resolved by caller
        assertThat(cmd).doesNotContain("/workspace");
    }

    @Test
    void baseCommandWithExistingFlags_preserved() {
        var config = configWith(b -> b.model = Optional.of("opus"));
        assertThat(WorkerCommandBuilder.build("claude --mcp http://localhost:7778/mcp", config))
                .isEqualTo("claude --mcp http://localhost:7778/mcp --model 'opus'");
    }

    private static ClaudonyProviderConfig configWith(java.util.function.Consumer<ConfigBuilder> customizer) {
        var b = new ConfigBuilder();
        customizer.accept(b);
        return new ClaudonyProviderConfig(
                b.command, b.model, b.appendSystemPrompt, b.systemPrompt,
                b.effort, b.permissionMode, b.tools, b.allowedTools,
                b.disallowedTools, b.addDirs, b.workingDir);
    }

    private static class ConfigBuilder {
        Optional<String> command = Optional.empty();
        Optional<String> model = Optional.empty();
        Optional<String> appendSystemPrompt = Optional.empty();
        Optional<String> systemPrompt = Optional.empty();
        Optional<String> effort = Optional.empty();
        Optional<String> permissionMode = Optional.empty();
        Optional<List<String>> tools = Optional.empty();
        Optional<List<String>> allowedTools = Optional.empty();
        Optional<List<String>> disallowedTools = Optional.empty();
        Optional<List<String>> addDirs = Optional.empty();
        Optional<String> workingDir = Optional.empty();
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=WorkerCommandBuilderTest`
Expected: Compilation failure — `WorkerCommandBuilder` does not exist.

- [ ] **Step 7: Implement WorkerCommandBuilder**

```java
package io.casehub.claudony.casehub;

import java.util.List;
import java.util.Optional;

public final class WorkerCommandBuilder {

    private WorkerCommandBuilder() {}

    public static String build(String baseCommand, ClaudonyProviderConfig config) {
        var sb = new StringBuilder(baseCommand);

        appendString(sb, "--model", config.model());
        if (config.systemPrompt().isPresent()) {
            appendString(sb, "--system-prompt", config.systemPrompt());
        } else {
            appendString(sb, "--append-system-prompt", config.appendSystemPrompt());
        }
        appendString(sb, "--effort", config.effort());
        appendString(sb, "--permission-mode", config.permissionMode());
        appendList(sb, "--tools", config.tools());
        appendList(sb, "--allowedTools", config.allowedTools());
        appendList(sb, "--disallowedTools", config.disallowedTools());
        config.addDirs().ifPresent(dirs -> dirs.forEach(dir -> appendString(sb, "--add-dir", Optional.of(dir))));

        return sb.toString();
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

- [ ] **Step 8: Run WorkerCommandBuilderTest to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=WorkerCommandBuilderTest`
Expected: All 9 tests PASS.

- [ ] **Step 9: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyProviderConfig.java \
       casehub/src/main/java/io/casehub/claudony/casehub/WorkerCommandBuilder.java \
       casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyProviderConfigTest.java \
       casehub/src/test/java/io/casehub/claudony/casehub/WorkerCommandBuilderTest.java
git commit -m "feat(#156): add ClaudonyProviderConfig record and WorkerCommandBuilder

ClaudonyProviderConfig is the per-agent config record with fromMap()
and fromConfigMapping() factories. WorkerCommandBuilder constructs
enriched CLI commands with shell-safe quoting.

Refs #156"
```

---

### Task 2: CaseHubConfig Changes + ProviderConfigSource SPI + ConfigMappingProviderConfigSource

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/CaseHubConfig.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/ProviderConfigSource.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSource.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSourceTest.java`

**Interfaces:**
- Consumes: `ClaudonyProviderConfig` (Task 1), `CaseHubConfig.AgentProviderConfig` (this task)
- Produces:
  - `ProviderConfigSource.forAgent(String agentId)` → `ClaudonyProviderConfig` — used by Task 3 (provisioner)
  - `ProviderConfigSource.declaredAgentIds()` → `Set<String>` — used by Task 3 (getCapabilities)
  - `CaseHubConfig.Workers.defaultCommand()` → `String` — used by Task 3 (provisioner constructor)

- [ ] **Step 1: Write ConfigMappingProviderConfigSourceTest**

```java
package io.casehub.claudony.casehub;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class ConfigMappingProviderConfigSourceTest {

    @Test
    void forAgent_knownId_returnsPopulatedConfig() {
        var agentConfig = stubAgentConfig("opus", Optional.of("high"));
        var source = new ConfigMappingProviderConfigSource(
                stubWorkersConfig(Map.of("code-reviewer", agentConfig)));

        var config = source.forAgent("code-reviewer");
        assertThat(config.model()).contains("opus");
        assertThat(config.effort()).contains("high");
    }

    @Test
    void forAgent_unknownId_returnsEmpty() {
        var source = new ConfigMappingProviderConfigSource(
                stubWorkersConfig(Map.of()));

        assertThat(source.forAgent("unknown")).isEqualTo(ClaudonyProviderConfig.EMPTY);
    }

    @Test
    void forAgent_emptyMap_returnsEmpty() {
        var source = new ConfigMappingProviderConfigSource(
                stubWorkersConfig(Map.of()));

        assertThat(source.forAgent("anything")).isEqualTo(ClaudonyProviderConfig.EMPTY);
    }

    @Test
    void declaredAgentIds_returnsProviderConfigKeys() {
        var source = new ConfigMappingProviderConfigSource(
                stubWorkersConfig(Map.of(
                        "code-reviewer", stubAgentConfig("opus", Optional.empty()),
                        "researcher", stubAgentConfig("sonnet", Optional.empty()))));

        assertThat(source.declaredAgentIds()).containsExactlyInAnyOrder("code-reviewer", "researcher");
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

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ConfigMappingProviderConfigSourceTest`
Expected: Compilation failure — `ProviderConfigSource`, `ConfigMappingProviderConfigSource`, `AgentProviderConfig` do not exist.

- [ ] **Step 3: Modify CaseHubConfig — remove commands(), add defaultCommand() + AgentProviderConfig + providerConfig()**

Replace the full content of `casehub/src/main/java/io/casehub/claudony/casehub/CaseHubConfig.java`:

```java
package io.casehub.claudony.casehub;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;
import java.util.List;
import java.util.Map;
import java.util.Optional;

@ConfigMapping(prefix = "claudony.casehub")
public interface CaseHubConfig {

    @WithDefault("false")
    boolean enabled();

    @WithName("channel-layout")
    @WithDefault("normative")
    String channelLayout();

    @WithName("mesh-participation")
    @WithDefault("active")
    String meshParticipation();

    @WithName("worker-exit-poll-ms")
    @WithDefault("5000")
    long workerExitPollMs();

    @WithName("worker-exit-max-poll-failures")
    @WithDefault("3")
    int workerExitMaxPollFailures();

    Workers workers();

    interface Workers {
        @WithName("default-command")
        @WithDefault("claude")
        String defaultCommand();

        @WithName("default-working-dir")
        @WithDefault("${user.home}/claudony-workspace")
        String defaultWorkingDir();

        @WithName("provider-config")
        Map<String, AgentProviderConfig> providerConfig();
    }

    interface AgentProviderConfig {
        Optional<String> command();
        Optional<String> model();
        @WithName("append-system-prompt")
        Optional<String> appendSystemPrompt();
        @WithName("system-prompt")
        Optional<String> systemPrompt();
        Optional<String> effort();
        @WithName("permission-mode")
        Optional<String> permissionMode();
        Optional<List<String>> tools();
        @WithName("allowed-tools")
        Optional<List<String>> allowedTools();
        @WithName("disallowed-tools")
        Optional<List<String>> disallowedTools();
        @WithName("add-dirs")
        Optional<List<String>> addDirs();
        @WithName("working-dir")
        Optional<String> workingDir();
    }
}
```

**Note:** SmallRye Config uses kebab-case for property names by default, but `@WithName` overrides ensure multi-word fields map correctly (e.g., `append-system-prompt` in properties → `appendSystemPrompt()` in Java). Quarkus @ConfigMapping maps `Map<String, Interface>` keys from the property path segment — `provider-config.code-reviewer.model=opus` → `providerConfig().get("code-reviewer").model()`.

- [ ] **Step 4: Create ProviderConfigSource SPI**

```java
package io.casehub.claudony.casehub;

import java.util.Set;

public interface ProviderConfigSource {
    ClaudonyProviderConfig forAgent(String agentId);
    Set<String> declaredAgentIds();
}
```

- [ ] **Step 5: Create ConfigMappingProviderConfigSource**

```java
package io.casehub.claudony.casehub;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Set;

@DefaultBean
@ApplicationScoped
public class ConfigMappingProviderConfigSource implements ProviderConfigSource {

    private final CaseHubConfig.Workers workers;

    @Inject
    public ConfigMappingProviderConfigSource(CaseHubConfig config) {
        this(config.workers());
    }

    ConfigMappingProviderConfigSource(CaseHubConfig.Workers workers) {
        this.workers = workers;
    }

    @Override
    public ClaudonyProviderConfig forAgent(String agentId) {
        var agentConfig = workers.providerConfig().get(agentId);
        if (agentConfig == null) return ClaudonyProviderConfig.EMPTY;
        return ClaudonyProviderConfig.fromConfigMapping(agentConfig);
    }

    @Override
    public Set<String> declaredAgentIds() {
        return Set.copyOf(workers.providerConfig().keySet());
    }
}
```

- [ ] **Step 6: Run ConfigMappingProviderConfigSourceTest to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ConfigMappingProviderConfigSourceTest`
Expected: All 4 tests PASS.

- [ ] **Step 7: Verify Task 1 tests still pass (CaseHubConfig changed)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: All casehub module tests pass (ClaudonyProviderConfigTest, WorkerCommandBuilderTest, ConfigMappingProviderConfigSourceTest + existing tests). Some existing tests may fail due to CaseHubConfig.Workers.commands() removal — these are fixed in Task 3.

- [ ] **Step 8: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/CaseHubConfig.java \
       casehub/src/main/java/io/casehub/claudony/casehub/ProviderConfigSource.java \
       casehub/src/main/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSource.java \
       casehub/src/test/java/io/casehub/claudony/casehub/ConfigMappingProviderConfigSourceTest.java
git commit -m "feat(#156): add ProviderConfigSource SPI and config mapping

CaseHubConfig.Workers: remove commands() map, add defaultCommand()
with @WithDefault(\"claude\") and providerConfig() map.

ProviderConfigSource SPI with @DefaultBean ConfigMappingProviderConfigSource
reads per-agent config from application.properties.

Refs #156"
```

---

### Task 3: Provisioner Migration + Config Migration + Delete WorkerCommandResolver + Docs

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/WorkerCommandResolver.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java`
- Delete: `casehub/src/test/java/io/casehub/claudony/casehub/WorkerCommandResolverTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/WorkerLifecycleSequenceTest.java`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java`
- Modify: `app/src/test/java/io/casehub/claudony/AgentCaseCompletionTest.java`
- Modify: `ARC42STORIES.MD`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: `ClaudonyProviderConfig` (Task 1), `WorkerCommandBuilder.build()` (Task 1), `ProviderConfigSource` (Task 2), `CaseHubConfig.Workers.defaultCommand()` (Task 2)
- Produces: nothing — this is the integration task

- [ ] **Step 1: Modify ClaudonyReactiveWorkerProvisioner — replace WorkerCommandResolver with ProviderConfigSource**

Replace the full content of `ClaudonyReactiveWorkerProvisioner.java`. The key changes are:
1. `WorkerCommandResolver resolver` → `ProviderConfigSource providerConfigSource` + `String defaultCommand`
2. `setupSession()` uses `providerConfigSource.forAgent(roleName)` + `WorkerCommandBuilder.build()`
3. `getCapabilities()` uses `providerConfigSource.declaredAgentIds()`
4. Session constructor uses `effectiveWorkingDir` and `enrichedCommand`

The full file is the existing file with these specific changes applied to the constructor (both CDI and test-visible), `setupSession()`, and `getCapabilities()`. The `provision()`, `terminate()`, `signalStarted()`, `startWatcher()`, causal context methods are unchanged.

- [ ] **Step 2: Delete WorkerCommandResolver.java and WorkerCommandResolverTest.java**

```bash
git rm casehub/src/main/java/io/casehub/claudony/casehub/WorkerCommandResolver.java
git rm casehub/src/test/java/io/casehub/claudony/casehub/WorkerCommandResolverTest.java
```

- [ ] **Step 3: Update ClaudonyReactiveWorkerProvisionerTest — replace WorkerCommandResolver with ProviderConfigSource**

Key changes:
- Replace `private WorkerCommandResolver resolver;` with `private ProviderConfigSource configSource;`
- `setUp()`: create a `ProviderConfigSource` that returns config with `command=Optional.of("claude")` for "code-reviewer"
- Constructor call: `new ClaudonyReactiveWorkerProvisioner(true, tmux, registry, configSource, sessionMapping, "claude", "/tmp/workers", null, null, null)`
- Add new tests:
  - `provision_withProviderConfig_tmuxReceivesEnrichedCommand` — config with model=opus → verify `createWorkerSession` called with command containing `--model 'opus'`
  - `provision_withWorkingDirOverride_tmuxReceivesOverriddenDir` — config with workingDir="/custom" → verify `createWorkerSession` called with "/custom"
  - `provision_withNoPerAgentConfig_usesDefaultCommand` — unknown agentId → verify `createWorkerSession` called with "claude" (defaultCommand)
  - `provision_sessionRecordsEffectiveValues` — verify Session registered with enriched command and effective workingDir
  - `getCapabilities_returnsDeclaredAgentIds` — verify returns config source's declared IDs

- [ ] **Step 4: Update WorkerLifecycleSequenceTest — replace WorkerCommandResolver**

Change `setUp()`:
```java
// Before:
var resolver = new WorkerCommandResolver(Map.of("default", "claude"));
provisioner = new ClaudonyReactiveWorkerProvisioner(
        true, tmux, registry, resolver, sessionMapping, "/workspace", null, null, null);

// After:
ProviderConfigSource configSource = new ProviderConfigSource() {
    @Override public ClaudonyProviderConfig forAgent(String agentId) {
        return ClaudonyProviderConfig.EMPTY;
    }
    @Override public java.util.Set<String> declaredAgentIds() {
        return java.util.Set.of("default", "reviewer");
    }
};
provisioner = new ClaudonyReactiveWorkerProvisioner(
        true, tmux, registry, configSource, sessionMapping, "claude", "/workspace", null, null, null);
```

- [ ] **Step 5: Migrate application.properties**

Change line 189:
```properties
# Before:
%dev.claudony.casehub.workers.commands.agent=claude

# After:
%dev.claudony.casehub.workers.provider-config.agent.command=claude
```

- [ ] **Step 6: Migrate CaseEngineRoundTripTest.CasehubEnabledProfile**

Change config overrides (lines 55-56):
```java
// Before:
"claudony.casehub.workers.commands.agent", "claude",
"claudony.casehub.workers.commands.default", "claude",

// After:
"claudony.casehub.workers.provider-config.agent.command", "claude",
```

(No `default-command` override needed — `@WithDefault("claude")` handles it.)

- [ ] **Step 7: Migrate AgentCaseCompletionTest.CompletionTestProfile**

Change config overrides (line 51):
```java
// Before:
"claudony.casehub.workers.commands.default", "claude",

// After (remove the line entirely — @WithDefault("claude") handles it):
// Line removed.
```

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass (557 expected baseline, adjusted for test additions/deletions).

- [ ] **Step 9: Update ARC42STORIES.MD — all 5 WorkerCommandResolver references**

- Line 147 (L6 layer component list): remove `WorkerCommandResolver`, add `ProviderConfigSource, ConfigMappingProviderConfigSource, ClaudonyProviderConfig, WorkerCommandBuilder`
- Line 201 (C4 Container description): same update
- Line 279 (Scenario 3): replace `WorkerCommandResolver.resolve(context.capability()) → command string` with:
  ```
  → ProviderConfigSource.forAgent(context.taskType()) → ClaudonyProviderConfig
  → WorkerCommandBuilder.build(config.command() ?? defaultCommand, config) → enriched command
  ```
- Line 658 (Journey chapter change-impact): replace `WorkerCommandResolver used by provisioner` with `ProviderConfigSource used by provisioner`
- Line 898 (File listing): remove `WorkerCommandResolver.java — capability → command; default fallback`, add entries for new files

- [ ] **Step 10: Update CLAUDE.md**

- Line 253 (file listing): remove `WorkerCommandResolver.java` line, add:
  ```
  ├── ProviderConfigSource.java            — SPI: per-agent config lookup by agentId
  ├── ConfigMappingProviderConfigSource.java — @DefaultBean: reads from application.properties
  ├── ClaudonyProviderConfig.java           — per-agent config record (command, model, tools, etc.)
  ├── WorkerCommandBuilder.java             — builds enriched CLI command with shell-safe quoting
  ```
- Lines 329-330 (config examples): replace `workers.commands.*` with:
  ```properties
  # claudony.casehub.workers.default-command=claude  (default; override if needed)
  claudony.casehub.workers.provider-config.code-reviewer.command=claude
  claudony.casehub.workers.provider-config.code-reviewer.model=opus
  ```
- Update test count if changed.

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#156): migrate provisioner to ProviderConfigSource, remove WorkerCommandResolver

Provisioner now resolves per-agent config via ProviderConfigSource.forAgent(taskType)
and builds enriched CLI commands with WorkerCommandBuilder. WorkerCommandResolver
removed — its concerns subsumed by unified config path.

Config migration: workers.commands.* → workers.provider-config.*.command
+ workers.default-command (@WithDefault(\"claude\")).

Closes #156"
```

- [ ] **Step 12: Run full test suite one final time**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass.
