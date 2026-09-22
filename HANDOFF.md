# HANDOFF — casehub-claudony

## Last Session

Completed #218 engine AgentConverter convergence onto RoutingAgentProvider. Cross-repo work across engine and blocks.

**#218 — AgentConverter convergence (completed).** Replaced the inline 5-way LLM provider switch in `AgentConverter` with a `ChatModelProviderResolver` SPI. Blocks provides a routing implementation that delegates to `RoutingAgentProvider` via `AgentProviderChatModel`.

### Engine changes (2 commits on `issue-205-llm-fleet-manager`):

1. **SPI + default implementation:** `ChatModelProviderResolver` interface and `InlineChatModelProviderResolver` (extracts the existing 5-way switch). 9 unit tests.
2. **Refactor + thread resolver:** `AgentConverter.toApiAgent()` now takes a resolver parameter (no backward-compat overload — pre-release). Threaded through `CaseDefinitionYamlMapper.load()` → `YamlCaseDefinitionConverter.convert()` → `buildAgentFunction()`. All callers updated: `YamlCaseHub` (production), 4 test files (expression-override, json-node, variable, main mapper test — 11 call sites total). 16 `AgentConverterTest` tests pass.

### Blocks changes (2 commits on `issue-218-agentconverter-convergence`):

1. **`RoutingChatModelProviderResolver`** in `engine-adapter-core` — routes through `AgentProviderChatModel` with `ModelScopedAgentProvider` injecting the model reference via `config.withModel()`. Model resolution: `modelName` first (registry ID), falls back to `providerType` (backend key). 6 unit tests.
2. **CDI wiring** in `engine-adapter/EngineAdapterBeans` — produces `ChatModelProviderResolver`. When `AgentProvider` is resolvable, routes through `RoutingChatModelProviderResolver`. Falls back to `InlineChatModelProviderResolver` when absent.

### Dependencies added:
- `engine-adapter-core/pom.xml`: `casehub-platform-agent-router-core` + `casehub-platform-agent-langchain4j-core` (both `0.2-SNAPSHOT`)
- `engine-adapter-core/pom.xml`: `junit-jupiter` + `assertj-core` (test scope — first tests in this module)
- Slot-local `.mvn/maven.config` created for blocks (gitignored) pointing to slot `.m2`

## Immediate Next Step

Advance to #219 (ConfigMapping agent pool) — last issue in the queue. Run `work next` to advance.

## Pre-Existing Failures

These are NOT caused by #218 and exist in the baseline:

- Engine: 18 checkstyle errors (pre-existing), 106 build errors (all neocortex CBR classes — slot-local `.m2` issue, not related)
- Engine: `CaseContextChangedEnsembleTest.java` modified but not by this session (pre-existing dirty state)
- Claudony: same baseline failures as previous session (see prior handoff)

## Queue

Position 6/8. #205–#217 done. #218 done. Next: #219 (ConfigMapping agent pool).

## Key Discoveries

- **Package names differ from source:** The fork research reported `io.casehub.platform.agent.api.*` but the installed JARs use `io.casehub.platform.agent.*` (no `.api` suffix). Always verify against the installed JAR, not source-tree layout.
- **`AgentProviderChatModel` requires non-null `AgentLangchain4jProperties`:** Passing `null` causes NPE at `doChat()`. A `DEFAULT_PROPERTIES` anonymous impl with sensible defaults (30s timeout, 10 memory window, 100 max sessions) is needed.
- **`engine-adapter-core` had zero test dependencies:** First module in blocks to have tests. Needed explicit `junit-jupiter` + `assertj-core` in the module pom.
- **Slot-local Maven repo:** Blocks repo had no `.mvn/maven.config`. Created one (gitignored) pointing to the slot `.m2` so `mvn install` from engine-api would be visible.

## References

- `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md`
- `specs/issue-205-llm-fleet-manager/decisions.md` (D1–D11)
- `plans/2026-09-22-engine-agentconverter-convergence.md`
- `plans/2026-09-21-llm-fleet-manager.md`
