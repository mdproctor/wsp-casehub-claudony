# Per-Agent Provider Config for Worker Provisioning

**Issue:** claudony#156
**Date:** 2026-06-25
**Status:** Approved (revised after review, round 2)

## Problem

When the CaseHub engine provisions a worker, Claudony launches a bare `claude` command for every agent — no per-agent model selection, system prompt, tool restrictions, or effort level. The deployment YAML (casehub-ops) declares per-agent provider-specific config, but Claudony can't read it.

Additionally, worker configuration is split across two mechanisms with different keying:
- `WorkerCommandResolver` resolves the base command from the full capability set (fragile — Set iteration order is non-deterministic)
- No per-agent CLI flag configuration exists at all

The engine passes ALL provisioner capabilities to `provision()` but sets `context.taskType()` to the specific capability being provisioned. Command resolution should use `taskType`, not the full set. ARC42STORIES.MD (line 279) already shows this intent: `WorkerCommandResolver.resolve(context.capability())` — singular — but the implementation takes `Set<String>`.

## Solution

Introduce a `ProviderConfigSource` SPI in claudony-casehub with an application.properties-backed `@DefaultBean` implementation. All per-agent configuration — including the base command — lives in `ClaudonyProviderConfig`, keyed by agentId (= `context.taskType()`). `WorkerCommandResolver` is removed; its concerns are subsumed by the unified config path.

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
                      ├── command (base CLI command)
                      ├── model, effort, permissionMode
                      ├── systemPrompt, appendSystemPrompt
                      ├── tools, allowedTools, disallowedTools
                      ├── addDirs, workingDir
                             │
                   WorkerCommandBuilder
                             │
                   enriched "claude --model opus --effort high ..."
                             │
                   TmuxService.createWorkerSession()
```

Resolution: `providerConfigSource.forAgent(context.taskType())` → get command + all flags → build enriched command → fallback to `defaultCommand` when no per-agent command configured.

## Key Design Decisions

### Unified keying by taskType/agentId

The engine sets `ProvisionContext.taskType()` = `capability.name()` — the specific capability being provisioned. This is the correct key for all per-agent config lookup. The full capability set passed to `provision()` is vestigial for Claudony (the engine passes ALL capabilities from `getCapabilities()`, not the specific one being provisioned).

### WorkerCommandResolver removal

`WorkerCommandResolver` is removed. Its two concerns are subsumed:
- **Command resolution:** `ClaudonyProviderConfig.command()` with `defaultCommand` fallback
- **Capability declaration:** `ProviderConfigSource.declaredAgentIds()` feeds `getCapabilities()`

The `commands` map in `CaseHubConfig.Workers` is replaced by `defaultCommand` (global fallback) + per-agent `command` in `AgentProviderConfig`.

### Capability declaration is explicit

Every agent that the provisioner should advertise as a capability must have at least one `provider-config.<agentId>.*` entry in application.properties. This is a Quarkus @ConfigMapping constraint — a `Map<String, Interface>` key only exists when at least one property for that key is present. This is deliberate: it forces explicit declaration of what the provisioner can handle, rather than inferring capabilities from an opaque command map.

### systemPrompt/appendSystemPrompt scope

These fields control the **Claude Code CLI system prompt** via `--system-prompt` / `--append-system-prompt` flags. They are independent of the CaseHub mesh system prompt (built by `ClaudonyReactiveWorkerContextProvider` and stored in `WorkerContext.properties("systemPrompt")`). The mesh prompt is a separate delivery mechanism — its current non-delivery to the CLI is a separate gap, not in scope for this issue.

## New Types

### ClaudonyProviderConfig (record, claudony-casehub)

Per-agent configuration. All fields optional — missing means "use default."

```java
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
    Optional<String> workingDir
)
```

Construction paths:
- `ClaudonyProviderConfig.EMPTY` — all fields empty
- `ClaudonyProviderConfig.fromMap(Map<String, Object>)` — converts opaque map (for future ops bridge). Expects `List<String>` values for list fields (YAML-native format). Ignores unknown keys.
- `ClaudonyProviderConfig.fromConfigMapping(CaseHubConfig.AgentProviderConfig)` — converts Quarkus config interface

The record faithfully stores both `systemPrompt` and `appendSystemPrompt` if both are configured. Precedence is enforced by `WorkerCommandBuilder`: when both are present, only `--system-prompt` is emitted (appending on top of a replacement is nonsensical). The record is a data carrier; the builder is the policy enforcement point.

### ProviderConfigSource (SPI interface, claudony-casehub)

```java
public interface ProviderConfigSource {
    ClaudonyProviderConfig forAgent(String agentId);
    Set<String> declaredAgentIds();
}
```

`forAgent(agentId)` returns config for the given agent. `agentId` is `context.taskType()` at provision time. Returns `EMPTY` when no config exists.

`declaredAgentIds()` returns the set of explicitly configured agent IDs. Used by `getCapabilities()`.

### ConfigMappingProviderConfigSource (@DefaultBean, claudony-casehub)

Reads from `CaseHubConfig.workers().providerConfig()`. Returns `EMPTY` when no config exists for the requested agentId. `declaredAgentIds()` returns the keys of the providerConfig map.

Yields to any `@Alternative @Priority(1)` bean — the future ops bridge slot.

### WorkerCommandBuilder (utility, claudony-casehub)

```java
public static String build(String baseCommand, ClaudonyProviderConfig config)
```

Takes the base command (e.g., `"claude"`) and appends CLI flags. The `command` and `workingDir` fields are resolved by the caller before invoking `build()` — the builder only appends CLI flags and never reads these fields. The command flows through `sh -c` (TmuxService uses ProcessBuilder with `"sh", "-c", command`), so all flag values are single-quoted (internal single quotes escaped as `'\''`).

Flag mapping:

| Config field | CLI flag | Format |
|-------------|----------|--------|
| model | `--model` | single-quoted string |
| appendSystemPrompt | `--append-system-prompt` | single-quoted string |
| systemPrompt | `--system-prompt` | single-quoted string |
| effort | `--effort` | single-quoted string |
| permissionMode | `--permission-mode` | single-quoted string |
| tools | `--tools` | comma-joined, single-quoted |
| allowedTools | `--allowedTools` | comma-joined, single-quoted |
| disallowedTools | `--disallowedTools` | comma-joined, single-quoted |
| addDirs | `--add-dir` | repeated per entry, each single-quoted |
| workingDir | (tmux -c) | not a CLI flag — overrides defaultWorkingDir |

All values are single-quoted unconditionally. Tool patterns like `Bash(git *)` contain shell metacharacters (parentheses, glob) that must survive `sh -c` expansion.

## Config Mapping Changes

### CaseHubConfig

```java
interface Workers {
    @WithDefault("claude")
    String defaultCommand();                            // NEW: replaces commands map
    String defaultWorkingDir();                         // existing
    Map<String, AgentProviderConfig> providerConfig();  // NEW
}

interface AgentProviderConfig {
    Optional<String> command();
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

The `commands` map (`Map<String, String>`) is removed. `defaultCommand` with `@WithDefault("claude")` replaces it as the global fallback — always non-null, no startup failure when unset.

### Properties format

```properties
# Global default command — @WithDefault("claude"), only override if needed
# claudony.casehub.workers.default-command=claude

# Per-agent provider config
claudony.casehub.workers.provider-config.code-reviewer.command=claude
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

### Constructor change

```java
@Inject
public ClaudonyReactiveWorkerProvisioner(
        CaseHubConfig config,
        TmuxService tmux,
        SessionRegistry registry,
        ProviderConfigSource providerConfigSource,    // replaces WorkerCommandResolver
        WorkerSessionMapping sessionMapping,
        Instance<CaseHubRuntime> caseHubRuntime,
        ClaudonyWorkerExecutionManager execManager,
        QhorusCausalLinkResolver causalLinkResolver) {
    this(config.enabled(), tmux, registry, providerConfigSource, sessionMapping,
            config.workers().defaultCommand(), config.workers().defaultWorkingDir(),
            caseHubRuntime, execManager, causalLinkResolver);
}

ClaudonyReactiveWorkerProvisioner(boolean enabled, TmuxService tmux, SessionRegistry registry,
                                   ProviderConfigSource providerConfigSource,
                                   WorkerSessionMapping sessionMapping,
                                   String defaultCommand, String defaultWorkingDir,
                                   Instance<CaseHubRuntime> caseHubRuntime,
                                   ClaudonyWorkerExecutionManager execManager,
                                   QhorusCausalLinkResolver causalLinkResolver) {
    this.enabled = enabled;
    this.tmux = tmux;
    this.registry = registry;
    this.providerConfigSource = providerConfigSource;
    this.sessionMapping = sessionMapping;
    this.defaultCommand = defaultCommand;
    this.defaultWorkingDir = defaultWorkingDir;
    this.caseHubRuntime = caseHubRuntime;
    this.execManager = execManager;
    this.causalLinkResolver = causalLinkResolver;
}
```

### setupSession() change

```java
private void setupSession(Set<String> capabilities, ProvisionContext context) {
    if (!enabled) {
        throw new ProvisioningException(
                "CaseHub integration is disabled — set claudony.casehub.enabled=true");
    }
    String sessionId = UUID.randomUUID().toString();
    String roleName = context.taskType() != null
            ? context.taskType()
            : capabilities.stream().findFirst().orElse("worker");

    // Unified config resolution by agentId (= roleName = taskType)
    ClaudonyProviderConfig providerConfig = providerConfigSource.forAgent(roleName);
    String baseCommand = providerConfig.command().orElse(defaultCommand);
    String enrichedCommand = WorkerCommandBuilder.build(baseCommand, providerConfig);
    String effectiveWorkingDir = providerConfig.workingDir().orElse(defaultWorkingDir);

    String sessionName = SESSION_PREFIX + sessionId;

    try {
        tmux.createWorkerSession(sessionName, effectiveWorkingDir, enrichedCommand);
        if (context.caseId() != null) {
            tmux.setSessionOption(sessionName, "@casehub_case_id", context.caseId().toString());
            tmux.setSessionOption(sessionName, "@casehub_role", roleName);
        }
    } catch (IOException | InterruptedException e) {
        throw new ProvisioningException("Failed to create tmux session for worker " + sessionId, e);
    }

    // Session records the effective values — what was actually launched
    var session = new Session(sessionId, sessionName, effectiveWorkingDir, enrichedCommand,
            SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty(),
            Optional.ofNullable(context.caseId()).map(UUID::toString),
            Optional.of(roleName));
    registry.register(session);
    sessionMapping.register(roleName, context.caseId(), sessionId);
}
```

### getCapabilities() change

```java
@Override
public Uni<Set<String>> getCapabilities() {
    return Uni.createFrom().item(providerConfigSource.declaredAgentIds());
}
```

## Removed Type

### WorkerCommandResolver — REMOVED

Subsumed by `ProviderConfigSource.forAgent(agentId).command()` + `defaultCommand` fallback. All callers migrate:
- `ClaudonyReactiveWorkerProvisioner` — injects `ProviderConfigSource` instead
- `WorkerCommandResolverTest` — deleted (tests move to `ConfigMappingProviderConfigSourceTest` and `WorkerCommandBuilderTest`)

## Migration

### Property mapping

| Old | New |
|-----|-----|
| `workers.commands.default=claude` | `workers.default-command=claude` (or omit — `@WithDefault("claude")`) |
| `workers.commands.agent=claude` | `workers.provider-config.agent.command=claude` |
| `workers.commands.code-reviewer=claude --mcp ...` | `workers.provider-config.code-reviewer.command=claude --mcp ...` |

### Affected files

**Production config:**
- `app/src/main/resources/application.properties:189` — `%dev.claudony.casehub.workers.commands.agent=claude` → `%dev.claudony.casehub.workers.provider-config.agent.command=claude`

**Test profiles (config overrides):**
- `CaseEngineRoundTripTest.CasehubEnabledProfile` (lines 55-56) — `workers.commands.agent` + `workers.commands.default` → `workers.provider-config.agent.command` + `workers.default-command`
- `AgentCaseCompletionTest.CompletionTestProfile` (line 51) — `workers.commands.default` → `workers.default-command` (or omit — `@WithDefault`)

**Test classes (direct construction):**
- `WorkerCommandResolverTest` — deleted
- `ClaudonyReactiveWorkerProvisionerTest:35` — replace `new WorkerCommandResolver(...)` with `ProviderConfigSource` mock/stub
- `WorkerLifecycleSequenceTest:55` — replace `new WorkerCommandResolver(...)` with `ProviderConfigSource` mock/stub

**Documentation:**
- `ARC42STORIES.MD` — 5 locations (see ARC42STORIES.MD Update section)
- `CLAUDE.md:253` — file listing: `WorkerCommandResolver.java` → remove, add new files
- `CLAUDE.md:329-330` — config examples: `workers.commands.*` → `workers.default-command` + `workers.provider-config.*`

## Test Plan

### Unit tests (claudony-casehub)

**ClaudonyProviderConfigTest:**
- fromMap with all fields populated
- fromMap with empty map → EMPTY
- fromMap handles List<String> values for list fields
- fromMap ignores unknown keys
- fromConfigMapping maps all fields correctly
- systemPrompt and appendSystemPrompt both set — record holds both (data carrier)

**WorkerCommandBuilderTest:**
- Empty config → base command returned unchanged
- Single flag (model) → `claude --model 'opus'`
- All flags populated → full command string
- System prompt with spaces/quotes → properly single-quoted
- List fields: tools comma-joined and single-quoted
- Tool pattern with shell metacharacters (`Bash(git *)`) → properly quoted
- systemPrompt and appendSystemPrompt both set → only --system-prompt emitted (builder policy)
- addDirs repeated per entry, each single-quoted

**ConfigMappingProviderConfigSourceTest:**
- Known agentId → returns populated config
- Unknown agentId → returns EMPTY
- Empty providerConfig map → returns EMPTY
- declaredAgentIds → returns providerConfig keys

### Integration tests (claudony-app)

Extend existing `ClaudonyReactiveWorkerProvisionerTest`:
- Provision with provider config → tmux receives enriched command
- Provision with per-agent command override → tmux receives overridden command
- Provision with workingDir override → tmux receives overridden directory
- Provision with no per-agent config → tmux receives defaultCommand
- getCapabilities() returns declaredAgentIds from config source
- Session records effectiveWorkingDir and enrichedCommand (not defaults)
- Session with no per-agent config records defaultCommand and defaultWorkingDir

### Removed tests

- `WorkerCommandResolverTest` — deleted (functionality moved to config source + builder)

## What This Does Not Cover (Phase 2)

- casehub-ops-deployment dependency and `OpsProviderConfigSource` bridge bean
- Desired-state reconciliation loop (store population)
- MCP server config (`--mcp-config` requires JSON file generation)
- Settings config (`--settings` requires JSON file generation)
- Agents config (`--agents` requires JSON serialization)
- Mesh system prompt delivery to CLI (separate gap — WorkerContext.properties("systemPrompt") is built but not passed to the Claude session)

## ARC42STORIES.MD Update

All 5 `WorkerCommandResolver` references must be updated:

- **Line 147** — L6 layer component list: remove `WorkerCommandResolver`, add `ProviderConfigSource`, `WorkerCommandBuilder`, `ClaudonyProviderConfig`
- **Line 201** — C4 Container description: same component list update
- **Line 279** — Scenario 3 provision flow:
  ```
  → ProviderConfigSource.forAgent(context.taskType()) → ClaudonyProviderConfig
  → WorkerCommandBuilder.build(config.command() ?? defaultCommand, config) → enriched command
  ```
- **Line 658** — Journey chapter change-impact: update to reference ProviderConfigSource instead of WorkerCommandResolver
- **Line 898** — File listing: remove `WorkerCommandResolver.java`, add `ProviderConfigSource.java`, `ConfigMappingProviderConfigSource.java`, `ClaudonyProviderConfig.java`, `WorkerCommandBuilder.java`
