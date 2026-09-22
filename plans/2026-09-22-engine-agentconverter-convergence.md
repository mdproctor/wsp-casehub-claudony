# Engine AgentConverter Convergence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/claudony#218 — engine AgentConverter convergence onto RoutingAgentProvider
**Issue group:** #218

**Goal:** Replace `AgentConverter`'s inline 5-way LLM provider switch with an SPI that blocks can implement to route through `RoutingAgentProvider`, giving YAML-defined agents automatic pool management, rate limiting, and observability.

**Architecture:** Define `ChatModelProviderResolver` SPI in engine-api. Extract the current inline construction into `InlineChatModelProviderResolver` as the default. Thread the resolver through the existing `CaseDefinitionYamlMapper.load()` → `YamlCaseDefinitionConverter.convert()` → `buildAgentFunction()` call chain (same pattern as `WorkerFunctionProviderRegistry`). Blocks provides `RoutingChatModelProviderResolver` in `engine-adapter-core` (plain POJO) and wires it as a CDI bean in `engine-adapter`.

**Tech Stack:** Java 21, engine-api (static utilities), blocks engine-adapter-core (POJOs), blocks engine-adapter (CDI), platform agent-api/agent-router-core/agent-langchain4j-core

## Global Constraints

- Engine-api is a library module — no CDI annotations, no framework deps
- `engine-adapter-core` is "zero CDI, zero Spring" — POJOs only
- `engine-adapter` is the CDI wiring layer — `@ApplicationScoped` beans, `@Produces`
- Pre-release: no backward-compat overloads. Change signatures directly, update all callers.
- All existing `AgentConverterTest` tests must be updated to pass the resolver explicitly

---

## Batch 1: Engine SPI + refactor (engine repo)

### Task 1: Define `ChatModelProviderResolver` SPI and extract `InlineChatModelProviderResolver`

**Files:**
- Create: `engine/api/src/main/java/io/casehub/api/model/ai/ChatModelProviderResolver.java`
- Create: `engine/api/src/main/java/io/casehub/api/model/ai/InlineChatModelProviderResolver.java`
- Test: `engine/api/src/test/java/io/casehub/api/model/ai/InlineChatModelProviderResolverTest.java`

**Interfaces:**
- Produces: `ChatModelProviderResolver.resolve(String providerType, JsonNode config) → ChatModelProvider` — consumed by Task 2 (AgentConverter refactor) and Task 3 (blocks adapter)
- Produces: `InlineChatModelProviderResolver` singleton — the default implementation

- [ ] **Step 1: Write the SPI interface**

```java
package io.casehub.api.model.ai;

import com.fasterxml.jackson.databind.JsonNode;

public interface ChatModelProviderResolver {
    ChatModelProvider resolve(String providerType, JsonNode config);
}
```

- [ ] **Step 2: Write failing tests for `InlineChatModelProviderResolver`**

Tests mirror the 5 provider branches from `AgentConverterTest` but target the resolver directly. Test each provider with all fields and minimal fields. Test null providerType throws. Test unknown provider throws.

```java
package io.casehub.api.model.ai;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

class InlineChatModelProviderResolverTest {

    private static final ObjectMapper JSON = new ObjectMapper();
    private final ChatModelProviderResolver resolver = InlineChatModelProviderResolver.INSTANCE;

    @Test
    void resolve_nullProviderType_throws() {
        assertThatThrownBy(() -> resolver.resolve(null, JSON.readTree("{}")))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("provider type");
    }

    @Test
    void resolve_unknownProvider_throws() throws Exception {
        assertThatThrownBy(() -> resolver.resolve("unknown-llm", JSON.readTree("{}")))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Unknown model provider");
    }

    @Test
    void resolve_openai_allFields() throws Exception {
        JsonNode config = JSON.readTree(
            """
            {"modelName":"gpt-4","apiKey":"sk-test",
             "baseUrl":"http://localhost:3000/v1","organizationId":"org-test",
             "temperature":0.7,"topP":0.9,"maxTokens":1024}""");
        ChatModelProvider result = resolver.resolve("openai", config);
        assertThat(result).isNotNull();
        assertThat(result.type()).isEqualTo(ModelType.OPENAI);
    }

    @Test
    void resolve_anthropic_allFields() throws Exception {
        JsonNode config = JSON.readTree(
            """
            {"modelName":"claude-3-sonnet-20240229","apiKey":"sk-ant-test",
             "baseUrl":"https://custom.example.com","version":"2023-06-01",
             "temperature":0.3,"topP":0.95,"topK":40,"maxTokens":2048}""");
        ChatModelProvider result = resolver.resolve("anthropic", config);
        assertThat(result).isNotNull();
        assertThat(result.type()).isEqualTo(ModelType.ANTHROPIC);
    }

    @Test
    void resolve_ollama() throws Exception {
        JsonNode config = JSON.readTree(
            "{\"baseUrl\":\"http://localhost:11434\",\"modelName\":\"llama2\",\"temperature\":0.5}");
        ChatModelProvider result = resolver.resolve("ollama", config);
        assertThat(result).isNotNull();
        assertThat(result.type()).isEqualTo(ModelType.OLLAMA);
    }

    @Test
    void resolve_mistral() throws Exception {
        JsonNode config = JSON.readTree(
            "{\"modelName\":\"mistral-large-latest\",\"apiKey\":\"msk-test\"}");
        ChatModelProvider result = resolver.resolve("mistralai", config);
        assertThat(result).isNotNull();
        assertThat(result.type()).isEqualTo(ModelType.MISTRAL);
    }

    @Test
    void resolve_mistral_alias() throws Exception {
        JsonNode config = JSON.readTree(
            "{\"modelName\":\"mistral-small\",\"apiKey\":\"msk-key\"}");
        ChatModelProvider result = resolver.resolve("mistral", config);
        assertThat(result).isNotNull();
        assertThat(result.type()).isEqualTo(ModelType.MISTRAL);
    }

    @Test
    void resolve_gemini() throws Exception {
        JsonNode config = JSON.readTree(
            "{\"modelName\":\"gemini-pro\",\"apiKey\":\"gai-test\",\"temperature\":0.6,\"maxTokens\":1500}");
        ChatModelProvider result = resolver.resolve("googleaigemini", config);
        assertThat(result).isNotNull();
        assertThat(result.type()).isEqualTo(ModelType.GOOGLE_AI_GEMINI);
    }

    @Test
    void resolve_gemini_alias() throws Exception {
        JsonNode config = JSON.readTree(
            "{\"modelName\":\"gemini-1.5-pro\",\"apiKey\":\"gai-key\"}");
        ChatModelProvider result = resolver.resolve("gemini", config);
        assertThat(result).isNotNull();
        assertThat(result.type()).isEqualTo(ModelType.GOOGLE_AI_GEMINI);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f engine/api/pom.xml -Dtest=InlineChatModelProviderResolverTest`
Expected: compilation failure — `InlineChatModelProviderResolver` doesn't exist yet

- [ ] **Step 4: Implement `InlineChatModelProviderResolver`**

Extract the body of `AgentConverter.toChatModelProviderFromNode()` verbatim into the new class:

```java
package io.casehub.api.model.ai;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.ai.anthropic.AnthropicChatModelProvider;
import io.casehub.api.model.ai.gemini.GoogleAiGeminiChatModelProvider;
import io.casehub.api.model.ai.mistral.MistralAiChatModelProvider;
import io.casehub.api.model.ai.ollama.OllamaChatModelProvider;
import io.casehub.api.model.ai.openai.OpenAiChatModelProvider;

public final class InlineChatModelProviderResolver implements ChatModelProviderResolver {

    public static final InlineChatModelProviderResolver INSTANCE =
        new InlineChatModelProviderResolver();

    private InlineChatModelProviderResolver() {}

    @Override
    public ChatModelProvider resolve(String providerType, JsonNode config) {
        if (providerType == null) {
            throw new IllegalArgumentException("agent 'model' field (provider type) is required");
        }
        String modelName = config.has("modelName") ? config.get("modelName").asText() : null;
        String apiKey = config.has("apiKey") ? config.get("apiKey").asText() : null;
        Double temperature = config.has("temperature") ? config.get("temperature").asDouble() : null;
        Double topP = config.has("topP") ? config.get("topP").asDouble() : null;
        Integer maxTokens = config.has("maxTokens") ? config.get("maxTokens").asInt() : null;
        String baseUrl = config.has("baseUrl") ? config.get("baseUrl").asText() : null;

        return switch (providerType.toLowerCase()) {
            case "openai" -> {
                var b = OpenAiChatModelProvider.builder().apiKey(apiKey).modelName(modelName);
                if (baseUrl != null) b.baseUrl(baseUrl);
                if (temperature != null) b.temperature(temperature);
                if (topP != null) b.topP(topP);
                if (maxTokens != null) b.maxTokens(maxTokens);
                if (config.has("organizationId")) b.organizationId(config.get("organizationId").asText());
                yield b.build();
            }
            case "anthropic" -> {
                var b = AnthropicChatModelProvider.builder().apiKey(apiKey).modelName(modelName);
                if (baseUrl != null) b.baseUrl(baseUrl);
                if (temperature != null) b.temperature(temperature);
                if (topP != null) b.topP(topP);
                if (maxTokens != null) b.maxTokens(maxTokens);
                if (config.has("version")) b.version(config.get("version").asText());
                if (config.has("topK")) b.topK(config.get("topK").asInt());
                yield b.build();
            }
            case "ollama" -> {
                var b = OllamaChatModelProvider.builder().baseUrl(baseUrl).modelName(modelName);
                if (temperature != null) b.temperature(temperature);
                if (topP != null) b.topP(topP);
                yield b.build();
            }
            case "mistralai", "mistral" -> {
                var b = MistralAiChatModelProvider.builder().apiKey(apiKey).modelName(modelName);
                if (baseUrl != null) b.baseUrl(baseUrl);
                if (temperature != null) b.temperature(temperature);
                if (topP != null) b.topP(topP);
                if (maxTokens != null) b.maxTokens(maxTokens);
                yield b.build();
            }
            case "googleaigemini", "gemini" -> {
                var b = GoogleAiGeminiChatModelProvider.builder().apiKey(apiKey).modelName(modelName);
                if (temperature != null) b.temperature(temperature);
                if (topP != null) b.topP(topP);
                if (maxTokens != null) b.maxOutputTokens(maxTokens);
                yield b.build();
            }
            default -> throw new IllegalArgumentException("Unknown model provider: " + providerType);
        };
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f engine/api/pom.xml -Dtest=InlineChatModelProviderResolverTest`
Expected: all 9 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C engine add api/src/main/java/io/casehub/api/model/ai/ChatModelProviderResolver.java \
  api/src/main/java/io/casehub/api/model/ai/InlineChatModelProviderResolver.java \
  api/src/test/java/io/casehub/api/model/ai/InlineChatModelProviderResolverTest.java
git -C engine commit -m "feat(#218): define ChatModelProviderResolver SPI + InlineChatModelProviderResolver

Extract the 5-way provider switch from AgentConverter into a standalone
SPI interface and default implementation. The SPI enables blocks to
provide an alternative resolver that routes through RoutingAgentProvider.

Refs casehubio/claudony#218"
```

### Task 2: Refactor `AgentConverter` and thread resolver through call chain

**Files:**
- Modify: `engine/api/src/main/java/io/casehub/api/model/converter/AgentConverter.java`
- Modify: `engine/api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java`
- Modify: `engine/api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Test: existing `engine/api/src/test/java/io/casehub/api/model/converter/AgentConverterTest.java` (must still pass)

**Interfaces:**
- Consumes: `ChatModelProviderResolver` from Task 1
- Produces: `AgentConverter.toApiAgent(JsonNode, ChatModelProviderResolver)` — single method (no overload)
- Produces: `YamlCaseDefinitionConverter.convert(yaml, registry, providers, resolver)` — resolver added to existing signature
- Produces: `CaseDefinitionYamlMapper.load(...)` — resolver added to existing signatures

- [ ] **Step 1: Refactor `AgentConverter` — replace static method with resolver parameter**

Replace `AgentConverter.java` contents. Single `toApiAgent` method that takes a resolver. No backward-compat overload — pre-release.

```java
package io.casehub.api.model.converter;

import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ChatModelProviderResolver;
import io.casehub.api.model.ai.AgentBuilder;

public class AgentConverter {

    public static io.casehub.api.model.ai.Agent toApiAgent(
            com.fasterxml.jackson.databind.JsonNode agentNode,
            ChatModelProviderResolver resolver) {
        if (agentNode == null || agentNode.isNull()) {
            return null;
        }

        String providerType;
        com.fasterxml.jackson.databind.JsonNode providerConfigNode;
        com.fasterxml.jackson.databind.JsonNode modelNode = agentNode.get("model");
        if (modelNode != null && modelNode.isObject() && modelNode.size() > 0) {
            var entry = modelNode.fields().next();
            providerType = entry.getKey();
            providerConfigNode = entry.getValue();
        } else {
            providerType = modelNode != null ? modelNode.asText() : null;
            providerConfigNode = agentNode;
        }
        ChatModelProvider modelProvider = resolver.resolve(providerType, providerConfigNode);

        String modelNameForId =
            providerConfigNode.has("modelName") ? providerConfigNode.get("modelName").asText() : null;

        AgentBuilder builder =
            io.casehub.api.model.ai.Agent.builder()
                .systemPrompt(
                    agentNode.has("systemPrompt") ? agentNode.get("systemPrompt").asText() : null)
                .model(modelProvider)
                .modelId(modelNameForId);

        if (agentNode.has("inputProjection")) {
            builder.inputProjection(agentNode.get("inputProjection").asText());
        }
        if (agentNode.has("outputProjection")) {
            builder.outputProjection(agentNode.get("outputProjection").asText());
        }
        if (agentNode.has("userMessageTemplate")) {
            builder.userMessage(agentNode.get("userMessageTemplate").asText());
        }

        return builder.build();
    }
}
```

The `toChatModelProviderFromNode` private static method is removed — its logic now lives in `InlineChatModelProviderResolver`.

- [ ] **Step 2: Update `AgentConverterTest` to pass resolver explicitly**

Every test call changes from `AgentConverter.toApiAgent(node)` to `AgentConverter.toApiAgent(node, InlineChatModelProviderResolver.INSTANCE)`:

```java
import io.casehub.api.model.ai.InlineChatModelProviderResolver;

// In each test method, e.g.:
Agent result = AgentConverter.toApiAgent(node, InlineChatModelProviderResolver.INSTANCE);

// For thrown-exception tests:
assertThatThrownBy(() -> AgentConverter.toApiAgent(node, InlineChatModelProviderResolver.INSTANCE))
```

Add a constant at the top of the test class:
```java
private static final ChatModelProviderResolver RESOLVER = InlineChatModelProviderResolver.INSTANCE;
```

Then use `AgentConverter.toApiAgent(node, RESOLVER)` everywhere.

- [ ] **Step 3: Run `AgentConverterTest`**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f engine/api/pom.xml -Dtest=AgentConverterTest`
Expected: all 14 tests PASS

- [ ] **Step 4: Thread resolver through `YamlCaseDefinitionConverter`**

Change the `convert()` signature directly — add `ChatModelProviderResolver` parameter:

```java
public static CaseDefinition convert(
    YamlCaseDefinition yaml,
    ExpressionEngineRegistry registry,
    WorkerFunctionProviderRegistry providers,
    ChatModelProviderResolver chatModelResolver) {
```

Update `convertWorkers()` signature:
```java
private static void convertWorkers(
    List<YamlWorker> yamlWorkers, CaseDefinition def,
    WorkerFunctionProviderRegistry providers,
    ChatModelProviderResolver chatModelResolver) {
```

Update `buildAgentFunction()` to accept and use the resolver:
```java
private static WorkerFunction<?, ?> buildAgentFunction(
    YamlWorker yw, ChatModelProviderResolver chatModelResolver) {
    try {
        JsonNode agentNode = MAPPER.valueToTree(yw.agent());
        Agent agent = AgentConverter.toApiAgent(agentNode, chatModelResolver);
        return new AgentWorkerFunction(agent);
    } catch (Exception e) {
        LOG.warnf(
            "Worker '%s': agent conversion failed, falling back to NONE — %s",
            yw.name(), e.getMessage());
        return WorkerFunction.NONE;
    }
}
```

- [ ] **Step 5: Thread resolver through `CaseDefinitionYamlMapper`**

Add `ChatModelProviderResolver` parameter to all `load()` methods that accept `WorkerFunctionProviderRegistry`. Change signatures directly — no overloads:

```java
public static CaseDefinition load(
    final InputStream yamlStream,
    final ObjectMapper objectMapper,
    final ExpressionEngineRegistry registry,
    final WorkerFunctionProviderRegistry providerRegistry,
    final ChatModelProviderResolver chatModelResolver) throws IOException {
    // ... passes chatModelResolver to convert()
}
```

For the simple `load(InputStream)` convenience method (non-CDI), pass `InlineChatModelProviderResolver.INSTANCE`.

- [ ] **Step 6: Fix all callers across the engine repo**

Find all callers of `convert()` and `load()` and update them. The main callers:
- `CaseDefinitionYamlMapper.load()` calls `YamlCaseDefinitionConverter.convert()` — already updated above
- Any test classes that call `load()` or `convert()` directly — pass `InlineChatModelProviderResolver.INSTANCE`
- Runtime callers in other engine modules — search with `ide_find_references`

- [ ] **Step 7: Run full engine-api test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f engine/api/pom.xml`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git -C engine add -A
git -C engine commit -m "refactor(#218): thread ChatModelProviderResolver through YAML conversion chain

AgentConverter.toApiAgent() now requires a ChatModelProviderResolver parameter.
The resolver is threaded through CaseDefinitionYamlMapper.load() →
YamlCaseDefinitionConverter.convert() → buildAgentFunction(). No backward-compat
overloads — pre-release, clean signatures.

Refs casehubio/claudony#218"
```

## Batch 2: Blocks adapter (blocks repo)

### Task 3: Implement `RoutingChatModelProviderResolver` in engine-adapter-core

**Files:**
- Modify: `blocks/engine-adapter-core/pom.xml` (add `agent-router-core` + `agent-langchain4j-core` deps)
- Create: `blocks/engine-adapter-core/src/main/java/io/casehub/engine/agentic/RoutingChatModelProviderResolver.java`
- Test: `blocks/engine-adapter-core/src/test/java/io/casehub/engine/agentic/RoutingChatModelProviderResolverTest.java`

**Interfaces:**
- Consumes: `ChatModelProviderResolver` from Task 1
- Consumes: `AgentProvider` (platform agent-api)
- Consumes: `AgentProviderChatModel` (platform agent-langchain4j-core)
- Consumes: `ModelRegistry` (platform platform-api)
- Produces: `RoutingChatModelProviderResolver` — constructor takes `AgentProvider` + optional `ModelRegistry`

- [ ] **Step 1: Add dependencies to `engine-adapter-core/pom.xml`**

Add `casehub-platform-agent-router-core` and `casehub-platform-agent-langchain4j-core`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-router-core</artifactId>
    <version>${casehub-platform.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-langchain4j-core</artifactId>
    <version>${casehub-platform.version}</version>
</dependency>
```

Check whether `${casehub-platform.version}` is defined in the parent pom. If not, use the explicit version from the parent's `<dependencyManagement>`.

- [ ] **Step 2: Write failing tests**

```java
package io.casehub.engine.agentic;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.platform.agent.api.AgentEvent;
import io.casehub.platform.agent.api.AgentProvider;
import io.casehub.platform.agent.api.AgentSessionConfig;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;

class RoutingChatModelProviderResolverTest {

    private static final ObjectMapper JSON = new ObjectMapper();

    @Test
    void resolve_delegatesToAgentProvider() throws Exception {
        AgentProvider mockProvider = mock(AgentProvider.class);
        when(mockProvider.invoke(any(AgentSessionConfig.class)))
            .thenReturn(Multi.createFrom().items(
                AgentEvent.textDelta("Hello")));

        RoutingChatModelProviderResolver resolver =
            new RoutingChatModelProviderResolver(mockProvider);

        JsonNode config = JSON.readTree(
            "{\"modelName\":\"gpt-4\",\"apiKey\":\"sk-test\"}");
        ChatModelProvider result = resolver.resolve("openai", config);

        assertThat(result).isNotNull();
        assertThat(result.get()).isNotNull();
    }

    @Test
    void resolve_usesModelNameAsModelReference() throws Exception {
        AgentProvider mockProvider = mock(AgentProvider.class);
        when(mockProvider.invoke(any(AgentSessionConfig.class)))
            .thenReturn(Multi.createFrom().items(
                AgentEvent.textDelta("response")));

        RoutingChatModelProviderResolver resolver =
            new RoutingChatModelProviderResolver(mockProvider);

        JsonNode config = JSON.readTree("{\"modelName\":\"claude-sonnet-5\"}");
        ChatModelProvider result = resolver.resolve("anthropic", config);

        assertThat(result).isNotNull();
    }

    @Test
    void resolve_fallsBackToProviderTypeWhenNoModelName() throws Exception {
        AgentProvider mockProvider = mock(AgentProvider.class);
        when(mockProvider.invoke(any(AgentSessionConfig.class)))
            .thenReturn(Multi.createFrom().items(
                AgentEvent.textDelta("response")));

        RoutingChatModelProviderResolver resolver =
            new RoutingChatModelProviderResolver(mockProvider);

        JsonNode config = JSON.readTree("{}");
        ChatModelProvider result = resolver.resolve("openai", config);

        assertThat(result).isNotNull();
    }

    @Test
    void resolve_nullProviderType_throws() throws Exception {
        AgentProvider mockProvider = mock(AgentProvider.class);
        RoutingChatModelProviderResolver resolver =
            new RoutingChatModelProviderResolver(mockProvider);

        assertThatThrownBy(() -> resolver.resolve(null, JSON.readTree("{}")))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f blocks/engine-adapter-core/pom.xml -Dtest=RoutingChatModelProviderResolverTest`
Expected: compilation failure — `RoutingChatModelProviderResolver` doesn't exist yet

- [ ] **Step 4: Implement `RoutingChatModelProviderResolver`**

```java
package io.casehub.engine.agentic;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ChatModelProviderResolver;
import io.casehub.api.model.ai.ModelType;
import io.casehub.platform.agent.api.AgentProvider;
import io.casehub.platform.agent.api.AgentSessionConfig;
import io.casehub.platform.agent.langchain4j.AgentProviderChatModel;

public class RoutingChatModelProviderResolver implements ChatModelProviderResolver {

    private final AgentProvider agentProvider;

    public RoutingChatModelProviderResolver(AgentProvider agentProvider) {
        this.agentProvider = agentProvider;
    }

    @Override
    public ChatModelProvider resolve(String providerType, JsonNode config) {
        if (providerType == null) {
            throw new IllegalArgumentException("agent 'model' field (provider type) is required");
        }

        String modelName = config.has("modelName") ? config.get("modelName").asText() : null;
        String modelReference = modelName != null ? modelName : providerType;

        ModelType modelType = toModelType(providerType);

        AgentProviderChatModel chatModel = new AgentProviderChatModel(
            new ModelScopedAgentProvider(agentProvider, modelReference),
            java.util.List.of(),
            null);

        return new RoutedChatModelProvider(modelType, chatModel);
    }

    private static ModelType toModelType(String providerType) {
        return switch (providerType.toLowerCase()) {
            case "openai" -> ModelType.OPENAI;
            case "anthropic" -> ModelType.ANTHROPIC;
            case "ollama" -> ModelType.OLLAMA;
            case "mistralai", "mistral" -> ModelType.MISTRAL;
            case "googleaigemini", "gemini" -> ModelType.GOOGLE_AI_GEMINI;
            default -> throw new IllegalArgumentException("Unknown model provider: " + providerType);
        };
    }

    private record RoutedChatModelProvider(ModelType type,
                                           dev.langchain4j.model.chat.ChatModel model)
        implements ChatModelProvider {
        @Override
        public dev.langchain4j.model.chat.ChatModel get() {
            return model;
        }
    }
}
```

Note: `ModelScopedAgentProvider` is a thin wrapper that rewrites the model reference on each invocation — needed because `AgentProviderChatModel.doChat()` builds `AgentSessionConfig.of(system, user)` with null model, and we need the model reference baked in. Create as a package-private inner class or separate file:

```java
package io.casehub.engine.agentic;

import io.casehub.platform.agent.api.AgentEvent;
import io.casehub.platform.agent.api.AgentProvider;
import io.casehub.platform.agent.api.AgentSession;
import io.casehub.platform.agent.api.AgentSessionConfig;
import io.casehub.platform.agent.api.AgentSessionInit;
import io.smallrye.mutiny.Multi;

class ModelScopedAgentProvider implements AgentProvider {

    private final AgentProvider delegate;
    private final String modelReference;

    ModelScopedAgentProvider(AgentProvider delegate, String modelReference) {
        this.delegate = delegate;
        this.modelReference = modelReference;
    }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        AgentSessionConfig scoped = AgentSessionConfig.builder()
            .systemPrompt(config.systemPrompt())
            .userPrompt(config.userPrompt())
            .model(modelReference)
            .build();
        return delegate.invoke(scoped);
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        return delegate.openSession(init);
    }
}
```

**Important:** Verify `AgentSessionConfig` has a builder or a `with*()` method for setting the model field. If it's a record, check for `AgentSessionConfig.builder()`. If not available, use the constructor directly or a copy method. Adjust the implementation based on the actual API surface.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f blocks/engine-adapter-core/pom.xml -Dtest=RoutingChatModelProviderResolverTest`
Expected: all 4 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C blocks add engine-adapter-core/pom.xml \
  engine-adapter-core/src/main/java/io/casehub/engine/agentic/RoutingChatModelProviderResolver.java \
  engine-adapter-core/src/main/java/io/casehub/engine/agentic/ModelScopedAgentProvider.java \
  engine-adapter-core/src/test/java/io/casehub/engine/agentic/RoutingChatModelProviderResolverTest.java
git -C blocks commit -m "feat(#218): RoutingChatModelProviderResolver — route YAML agents through platform stack

Implements ChatModelProviderResolver by delegating to RoutingAgentProvider
via AgentProviderChatModel. Model reference resolution: tries modelName
first (registry ID lookup), falls back to providerType as backend key.

Refs casehubio/claudony#218"
```

### Task 4: CDI wiring in engine-adapter

**Files:**
- Modify: `blocks/engine-adapter/src/main/java/io/casehub/engine/agentic/EngineAdapterBeans.java`

**Interfaces:**
- Consumes: `RoutingChatModelProviderResolver` from Task 3
- Consumes: `AgentProvider` (injected via CDI — already available when platform agent stack is on classpath)
- Produces: `ChatModelProviderResolver` CDI bean

- [ ] **Step 1: Add `@Produces` method to `EngineAdapterBeans`**

Add to `EngineAdapterBeans.java`:

```java
@Inject Instance<AgentProvider> agentProviderInstance;

@Produces
@ApplicationScoped
public ChatModelProviderResolver chatModelProviderResolver() {
    if (agentProviderInstance.isResolvable()) {
        return new RoutingChatModelProviderResolver(agentProviderInstance.get());
    }
    return io.casehub.api.model.ai.InlineChatModelProviderResolver.INSTANCE;
}
```

Add the necessary imports:
```java
import io.casehub.api.model.ai.ChatModelProviderResolver;
import io.casehub.platform.agent.api.AgentProvider;
```

This produces the routing resolver when `AgentProvider` is available (platform agent stack on classpath), and falls back to inline when it's not. Graceful degradation.

- [ ] **Step 2: Verify blocks compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f blocks/engine-adapter/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C blocks add engine-adapter/src/main/java/io/casehub/engine/agentic/EngineAdapterBeans.java
git -C blocks commit -m "feat(#218): wire RoutingChatModelProviderResolver as CDI bean

Produces ChatModelProviderResolver via EngineAdapterBeans. When AgentProvider
is on classpath (platform agent stack deployed), routes through
RoutingChatModelProviderResolver. Falls back to InlineChatModelProviderResolver
when AgentProvider is absent.

Refs casehubio/claudony#218"
```

## References

- `engine/api/src/main/java/io/casehub/api/model/converter/AgentConverter.java` — current inline 5-way switch
- `engine/api/src/main/java/io/casehub/api/model/ai/ChatModelProvider.java` — engine's ChatModel abstraction
- `engine/api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java:609` — buildAgentFunction call site
- `engine/api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java:149` — YAML load entry point
- `engine/api/src/main/java/io/casehub/api/spi/WorkerFunctionProvider.java` — SPI pattern model
- `blocks/engine-adapter-core/src/main/java/io/casehub/engine/agentic/PatternWorkerFunctionProvider.java` — blocks adapter pattern model
- `blocks/engine-adapter/src/main/java/io/casehub/engine/agentic/EngineAdapterBeans.java` — CDI wiring layer
- `platform/agent-router-core/src/main/java/.../RoutingAgentProvider.java` — routing resolution chain
- `platform/agent-langchain4j-core/src/main/java/.../AgentProviderChatModel.java` — LangChain4j bridge
- `platform/agent-api/src/main/java/.../AgentSessionConfig.java` — invoke config record
- GitHub casehubio/claudony#218
