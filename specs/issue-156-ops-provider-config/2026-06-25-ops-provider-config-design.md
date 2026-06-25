# Per-Agent Provider Config for Worker Provisioning

**Issue:** claudony#156
**Date:** 2026-06-25
**Status:** Approved

## Problem

When the CaseHub engine provisions a worker, Claudony launches a bare `claude` command for every agent — no per-agent model selection, system prompt, tool restrictions, or effort level. The deployment YAML (casehub-ops) declares per-agent provider-specific config, but Claudony can't read it.

## Solution

Introduce a `ProviderConfigSource` SPI in claudony-casehub with an application.properties-backed `@DefaultBean` implementation. The provisioner resolves per-agent config at provision time and constructs an enriched CLI command with the appropriate flags.

When casehub-ops-deployment is co-deployed in the future, a higher-priority bridge bean plugs into the same SPI — no Claudony changes needed.

## Architecture

```
                          ProviderConfigSource (SPI)
                         /                          \
    ConfigMappingProviderConfigSource      [future: OpsProviderConfigSource]
         (@DefaultBean)                     (@Alternative @Priority(1))
              │                                      │
    CaseHubConfig.workers()              DeploymentProviderConfigStore
     .providerConfig()                     .forAgent(agentId)
    (application.properties)               (ops co-deployed)
              │                                      │
              └──────────────┬───────────────────────┘
                             │
                   ClaudonyProviderConfig (record)
                             │
                   WorkerCommandBuilder
                             │
                   enriched "claude --model opus --effort high ..."
                             │
                   TmuxService.createWorkerSession()
```

Resolution order: Ops store (if available, future) → application.properties → base command (existing).

## New Types

### ClaudonyProviderConfig (record, claudony-casehub)

Per-agent configuration. All fields optional — missing means "use default."

```java
public record ClaudonyProviderConfig(
    Optional<String> model,
    Optional<String> appendSystemPrompt,
    Optional<String> systemPrompt,
    Optional<String> effort,
    Optional<String> permissionMode,
    Optional<List<String>> tools,
    Optional<List<String>> allowedTools,
    Optional<List<String>> disallowedTools,
    Optional<List<String>> addDirs,
    Optional<String> workingDir
)
```

Construction paths:
- `ClaudonyProviderConfig.EMPTY` — all fields empty
- `ClaudonyProviderConfig.fromMap(Map<String, Object>)` — converts opaque map (for future ops bridge). Handles both `List<String>` values (YAML) and comma-separated strings (properties) for list fields — whitespace around commas is trimmed. Ignores unknown keys.
- `ClaudonyProviderConfig.fromConfigMapping(CaseHubConfig.AgentProviderConfig)` — converts Quarkus config interface

When both `systemPrompt` and `appendSystemPrompt` are set, `systemPrompt` wins (replaces the default entirely — appending on top of a replacement is nonsensical).

### ProviderConfigSource (SPI interface, claudony-casehub)

```java
public interface ProviderConfigSource {
    ClaudonyProviderConfig forAgent(String agentId);
}
```

The `agentId` parameter is `context.taskType()` at provision time (the role name — "code-reviewer", "researcher", etc.).

### ConfigMappingProviderConfigSource (@DefaultBean, claudony-casehub)

Reads from `CaseHubConfig.workers().providerConfig()`. Returns `EMPTY` when no config exists for the requested agentId. Yields to any `@Alternative @Priority(1)` bean.

### WorkerCommandBuilder (utility, claudony-casehub)

```java
public static String build(String baseCommand, ClaudonyProviderConfig config)
```

Takes the base command (e.g., `"claude"`) and appends CLI flags. The command flows through `sh -c` (TmuxService uses ProcessBuilder with `"sh", "-c", command`), so values with spaces/special characters are single-quoted (internal single quotes escaped as `'\''`).

Flag mapping:

| Config field | CLI flag | Format |
|-------------|----------|--------|
| model | `--model` | string |
| appendSystemPrompt | `--append-system-prompt` | single-quoted string |
| systemPrompt | `--system-prompt` | single-quoted string |
| effort | `--effort` | string |
| permissionMode | `--permission-mode` | string |
| tools | `--tools` | comma-joined |
| allowedTools | `--allowedTools` | comma-joined |
| disallowedTools | `--disallowedTools` | comma-joined |
| addDirs | `--add-dir` | repeated per entry |
| workingDir | (tmux -c) | not a CLI flag — overrides defaultWorkingDir |

## Config Mapping Extension

### CaseHubConfig changes

```java
interface Workers {
    Map<String, String> commands();          // existing
    String defaultWorkingDir();              // existing
    Map<String, AgentProviderConfig> providerConfig();  // NEW
}

interface AgentProviderConfig {
    Optional<String> model();
    Optional<String> appendSystemPrompt();
    Optional<String> systemPrompt();
    Optional<String> effort();
    Optional<String> permissionMode();
    Optional<List<String>> tools();
    Optional<List<String>> allowedTools();
    Optional<List<String>> disallowedTools();
    Optional<List<String>> addDirs();
    Optional<String> workingDir();
}
```

### Properties format

```properties
claudony.casehub.workers.provider-config.code-reviewer.model=opus
claudony.casehub.workers.provider-config.code-reviewer.append-system-prompt=You are a code reviewer. Focus on correctness, security, and clarity.
claudony.casehub.workers.provider-config.code-reviewer.effort=high
claudony.casehub.workers.provider-config.code-reviewer.tools=Read,Bash,Edit
claudony.casehub.workers.provider-config.code-reviewer.permission-mode=auto

claudony.casehub.workers.provider-config.researcher.model=sonnet
claudony.casehub.workers.provider-config.researcher.effort=medium
claudony.casehub.workers.provider-config.researcher.add-dirs=/Users/shared/docs,/Users/shared/data
```

The map key matches `context.taskType()` at provision time.

## Provisioner Modification

In `ClaudonyReactiveWorkerProvisioner.setupSession()`, after resolving the base command:

```java
String command = resolver.resolve(capabilities);

ClaudonyProviderConfig providerConfig = providerConfigSource.forAgent(roleName);
String enrichedCommand = WorkerCommandBuilder.build(command, providerConfig);
String effectiveWorkingDir = providerConfig.workingDir().orElse(defaultWorkingDir);

tmux.createWorkerSession(sessionName, effectiveWorkingDir, enrichedCommand);
```

Three lines added. `WorkerCommandResolver` untouched. `TmuxService` signature unchanged.

## Test Plan

### Unit tests (claudony-casehub)

**ClaudonyProviderConfigTest:**
- fromMap with all fields populated
- fromMap with empty map → EMPTY
- fromMap handles comma-separated strings for list fields
- fromMap handles List<String> values for list fields
- fromMap ignores unknown keys
- fromConfigMapping maps all fields correctly
- systemPrompt and appendSystemPrompt both set — systemPrompt wins

**WorkerCommandBuilderTest:**
- Empty config → base command returned unchanged
- Single flag (model) → `claude --model opus`
- All flags populated → full command string
- System prompt with spaces/quotes → properly single-quoted
- List fields: tools comma-joined, addDirs repeated
- systemPrompt takes precedence over appendSystemPrompt
- Base command with existing flags preserved

**ConfigMappingProviderConfigSourceTest:**
- Known agentId → returns populated config
- Unknown agentId → returns EMPTY
- Empty providerConfig map → returns EMPTY

### Integration tests (claudony-app)

Extend existing `ClaudonyReactiveWorkerProvisionerTest`:
- Provision with provider config → tmux receives enriched command
- Provision with workingDir override → tmux receives overridden directory
- Provision with no provider config → tmux receives base command

## What This Does Not Cover (Phase 2)

- casehub-ops-deployment dependency and `OpsProviderConfigSource` bridge bean
- Desired-state reconciliation loop (store population)
- MCP server config (`--mcp-config` requires JSON file generation)
- Settings config (`--settings` requires JSON file generation)
- Agents config (`--agents` requires JSON serialization)
