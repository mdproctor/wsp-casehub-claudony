# CaseChannelLayout SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Define and wire the `CaseChannelLayout` SPI so `ClaudonyCaseChannelProvider` opens all layout-defined channels on first case touch, with `NormativeChannelLayout` (3 channels) and `SimpleLayout` (2 channels) as built-in implementations.

**Architecture:** `CaseChannelLayout` is a plain Java SPI. `ClaudonyCaseChannelProvider` selects the implementation from config and caches opened channels per case — first `openChannel()` call initialises all layout channels via Qhorus, subsequent calls hit the cache. `CaseDefinition` is passed as `null` for now (layouts don't use it yet).

**Tech Stack:** Java 21, Quarkus 3.9.5, `io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools`, `io.quarkiverse.qhorus.runtime.channel.ChannelSemantic`, `io.quarkiverse.qhorus.runtime.message.MessageType`, SmallRye Config `@ConfigMapping`

**GitHub issue:** #87 (part of epic #86)

---

## File Map

| Status | File | Role |
|--------|------|------|
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/CaseChannelLayout.java` | SPI interface + `ChannelSpec` record |
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/NormativeChannelLayout.java` | 3-channel default (work/observe/oversight) |
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/SimpleLayout.java` | 2-channel variant (work/observe) |
| Create | `claudony-casehub/src/test/java/dev/claudony/casehub/NormativeChannelLayoutTest.java` | Unit tests for normative layout |
| Create | `claudony-casehub/src/test/java/dev/claudony/casehub/SimpleLayoutTest.java` | Unit tests for simple layout |
| Modify | `claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java` | Add `channelLayout()` |
| Modify | `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyCaseChannelProvider.java` | Inject layout, init-on-first-touch, per-case channel cache |
| Modify | `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyCaseChannelProviderTest.java` | Update existing + add layout tests |

---

## Task 1: Define `CaseChannelLayout` SPI

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/CaseChannelLayout.java`

No test needed — this is a pure interface definition.

- [ ] **Step 1: Create the interface**

```java
package dev.claudony.casehub;

import io.casehub.api.model.CaseDefinition;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public interface CaseChannelLayout {

    List<ChannelSpec> channelsFor(UUID caseId, CaseDefinition definition);

    record ChannelSpec(
            String purpose,
            ChannelSemantic semantic,
            Set<MessageType> allowedTypes,
            String description
    ) {}
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl claudony-casehub -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/CaseChannelLayout.java
git commit -m "feat: add CaseChannelLayout SPI interface and ChannelSpec record Refs #87"
```

---

## Task 2: `NormativeChannelLayout` with Tests

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/NormativeChannelLayout.java`
- Create: `claudony-casehub/src/test/java/dev/claudony/casehub/NormativeChannelLayoutTest.java`

- [ ] **Step 1: Write the failing tests**

```java
package dev.claudony.casehub;

import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class NormativeChannelLayoutTest {

    private final NormativeChannelLayout layout = new NormativeChannelLayout();

    @Test
    void channelsFor_returnsThreeChannels() {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(UUID.randomUUID(), null);
        assertThat(specs).hasSize(3);
    }

    @Test
    void channelsFor_purposes_areWorkObserveOversight() {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(UUID.randomUUID(), null);
        assertThat(specs).extracting(CaseChannelLayout.ChannelSpec::purpose)
                .containsExactly("work", "observe", "oversight");
    }

    @Test
    void channelsFor_allUseAppendSemantic() {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(UUID.randomUUID(), null);
        assertThat(specs).extracting(CaseChannelLayout.ChannelSpec::semantic)
                .containsOnly(ChannelSemantic.APPEND);
    }

    @Test
    void channelsFor_observeChannel_allowsOnlyEventType() {
        CaseChannelLayout.ChannelSpec observe = layout.channelsFor(UUID.randomUUID(), null).stream()
                .filter(s -> s.purpose().equals("observe"))
                .findFirst().orElseThrow();
        assertThat(observe.allowedTypes()).containsExactly(MessageType.EVENT);
    }

    @Test
    void channelsFor_oversightChannel_allowsQueryAndCommand() {
        CaseChannelLayout.ChannelSpec oversight = layout.channelsFor(UUID.randomUUID(), null).stream()
                .filter(s -> s.purpose().equals("oversight"))
                .findFirst().orElseThrow();
        assertThat(oversight.allowedTypes()).containsExactlyInAnyOrder(MessageType.QUERY, MessageType.COMMAND);
    }

    @Test
    void channelsFor_workChannel_allowsAllTypes() {
        CaseChannelLayout.ChannelSpec work = layout.channelsFor(UUID.randomUUID(), null).stream()
                .filter(s -> s.purpose().equals("work"))
                .findFirst().orElseThrow();
        assertThat(work.allowedTypes()).isNull();
    }

    @Test
    void channelsFor_caseIdIsIgnored_returnsConsistentSpecs() {
        UUID id1 = UUID.randomUUID();
        UUID id2 = UUID.randomUUID();
        assertThat(layout.channelsFor(id1, null))
                .usingRecursiveComparison()
                .isEqualTo(layout.channelsFor(id2, null));
    }
}
```

- [ ] **Step 2: Run tests — expect failures**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=NormativeChannelLayoutTest -q 2>&1 | tail -5
```

Expected: COMPILE ERROR or test failure (class doesn't exist yet)

- [ ] **Step 3: Implement `NormativeChannelLayout`**

```java
package dev.claudony.casehub;

import io.casehub.api.model.CaseDefinition;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public class NormativeChannelLayout implements CaseChannelLayout {

    @Override
    public List<ChannelSpec> channelsFor(UUID caseId, CaseDefinition definition) {
        return List.of(
                new ChannelSpec("work", ChannelSemantic.APPEND, null,
                        "Primary coordination — all obligation-carrying message types"),
                new ChannelSpec("observe", ChannelSemantic.APPEND, Set.of(MessageType.EVENT),
                        "Telemetry — EVENT only, no obligations created"),
                new ChannelSpec("oversight", ChannelSemantic.APPEND,
                        Set.of(MessageType.QUERY, MessageType.COMMAND),
                        "Human governance — agent QUERY and human COMMAND")
        );
    }
}
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=NormativeChannelLayoutTest
```

Expected: Tests run: 7, Failures: 0, Errors: 0

- [ ] **Step 5: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/NormativeChannelLayout.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/NormativeChannelLayoutTest.java
git commit -m "feat: implement NormativeChannelLayout — work/observe/oversight channels Refs #87"
```

---

## Task 3: `SimpleLayout` with Tests

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/SimpleLayout.java`
- Create: `claudony-casehub/src/test/java/dev/claudony/casehub/SimpleLayoutTest.java`

- [ ] **Step 1: Write the failing tests**

```java
package dev.claudony.casehub;

import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class SimpleLayoutTest {

    private final SimpleLayout layout = new SimpleLayout();

    @Test
    void channelsFor_returnsTwoChannels() {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(UUID.randomUUID(), null);
        assertThat(specs).hasSize(2);
    }

    @Test
    void channelsFor_purposes_areWorkAndObserve() {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(UUID.randomUUID(), null);
        assertThat(specs).extracting(CaseChannelLayout.ChannelSpec::purpose)
                .containsExactly("work", "observe");
    }

    @Test
    void channelsFor_allUseAppendSemantic() {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(UUID.randomUUID(), null);
        assertThat(specs).extracting(CaseChannelLayout.ChannelSpec::semantic)
                .containsOnly(ChannelSemantic.APPEND);
    }

    @Test
    void channelsFor_observeChannel_allowsOnlyEventType() {
        CaseChannelLayout.ChannelSpec observe = layout.channelsFor(UUID.randomUUID(), null).stream()
                .filter(s -> s.purpose().equals("observe"))
                .findFirst().orElseThrow();
        assertThat(observe.allowedTypes()).containsExactly(MessageType.EVENT);
    }

    @Test
    void channelsFor_workChannel_allowsAllTypes() {
        CaseChannelLayout.ChannelSpec work = layout.channelsFor(UUID.randomUUID(), null).stream()
                .filter(s -> s.purpose().equals("work"))
                .findFirst().orElseThrow();
        assertThat(work.allowedTypes()).isNull();
    }

    @Test
    void channelsFor_hasNoOversightChannel() {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(UUID.randomUUID(), null);
        assertThat(specs).extracting(CaseChannelLayout.ChannelSpec::purpose)
                .doesNotContain("oversight");
    }
}
```

- [ ] **Step 2: Run tests — expect compile error**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=SimpleLayoutTest -q 2>&1 | tail -5
```

Expected: COMPILE ERROR (class doesn't exist yet)

- [ ] **Step 3: Implement `SimpleLayout`**

```java
package dev.claudony.casehub;

import io.casehub.api.model.CaseDefinition;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public class SimpleLayout implements CaseChannelLayout {

    @Override
    public List<ChannelSpec> channelsFor(UUID caseId, CaseDefinition definition) {
        return List.of(
                new ChannelSpec("work", ChannelSemantic.APPEND, null,
                        "Primary coordination — all obligation-carrying message types"),
                new ChannelSpec("observe", ChannelSemantic.APPEND, Set.of(MessageType.EVENT),
                        "Telemetry — EVENT only, no obligations created")
        );
    }
}
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=SimpleLayoutTest
```

Expected: Tests run: 6, Failures: 0, Errors: 0

- [ ] **Step 5: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/SimpleLayout.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/SimpleLayoutTest.java
git commit -m "feat: implement SimpleLayout — work/observe channels (no oversight) Refs #87"
```

---

## Task 4: Config Property + Provider Refactor

**Files:**
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java`
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyCaseChannelProvider.java`
- Modify: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyCaseChannelProviderTest.java`

### Step 4a: Add `channelLayout()` to config

- [ ] **Step 1: Update `CaseHubConfig`**

Replace the entire file content:

```java
package dev.claudony.casehub;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;
import java.util.Map;

@ConfigMapping(prefix = "claudony.casehub")
public interface CaseHubConfig {

    @WithDefault("false")
    boolean enabled();

    @WithName("channel-layout")
    @WithDefault("normative")
    String channelLayout();

    Workers workers();

    interface Workers {
        Map<String, String> commands();

        @WithName("default-working-dir")
        @WithDefault("${user.home}/claudony-workspace")
        String defaultWorkingDir();
    }
}
```

### Step 4b: Refactor `ClaudonyCaseChannelProvider`

- [ ] **Step 2: Rewrite `ClaudonyCaseChannelProvider`**

The provider now:
1. Selects `CaseChannelLayout` from config on construction
2. On first `openChannel()` for a caseId, calls the layout and creates all channels via Qhorus
3. Caches channels per case in a `ConcurrentHashMap`
4. Returns the requested purpose channel from cache (creates ad-hoc if not in layout)

```java
package dev.claudony.casehub;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.spi.CaseChannelProvider;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ClaudonyCaseChannelProvider implements CaseChannelProvider {

    private static final String CHANNEL_PREFIX = "case-";
    private static final String QHORUS_NAME_KEY = "qhorus-name";

    private final QhorusMcpTools qhorusMcpTools;
    private final CaseChannelLayout layout;
    private final ConcurrentHashMap<UUID, Map<String, CaseChannel>> caseChannels = new ConcurrentHashMap<>();

    @Inject
    public ClaudonyCaseChannelProvider(QhorusMcpTools qhorusMcpTools, CaseHubConfig config) {
        this.qhorusMcpTools = qhorusMcpTools;
        this.layout = selectLayout(config.channelLayout());
    }

    ClaudonyCaseChannelProvider(QhorusMcpTools qhorusMcpTools, CaseChannelLayout layout) {
        this.qhorusMcpTools = qhorusMcpTools;
        this.layout = layout;
    }

    @Override
    public CaseChannel openChannel(UUID caseId, String purpose) {
        Map<String, CaseChannel> channels = caseChannels.computeIfAbsent(caseId, this::initializeLayout);
        return channels.computeIfAbsent(purpose, p -> createQhorusChannel(caseId, p, null));
    }

    @Override
    public void postToChannel(CaseChannel channel, String from, String content) {
        String qhorusName = (String) channel.properties().getOrDefault(QHORUS_NAME_KEY, channel.id());
        qhorusMcpTools.sendMessage(qhorusName, from, "status", content, null, null);
    }

    @Override
    public void closeChannel(CaseChannel channel) {
        // Qhorus channels are persistent — no close operation
    }

    @Override
    public List<CaseChannel> listChannels(UUID caseId) {
        String prefix = CHANNEL_PREFIX + caseId;
        return qhorusMcpTools.listChannels().stream()
                .filter(ch -> ch.name().startsWith(prefix))
                .map(ch -> new CaseChannel(
                        ch.channelId().toString(),
                        ch.name(),
                        extractPurpose(ch.name(), caseId),
                        "qhorus",
                        Map.of(QHORUS_NAME_KEY, ch.name())))
                .toList();
    }

    private Map<String, CaseChannel> initializeLayout(UUID caseId) {
        Map<String, CaseChannel> channels = new HashMap<>();
        for (CaseChannelLayout.ChannelSpec spec : layout.channelsFor(caseId, null)) {
            CaseChannel ch = createQhorusChannel(caseId, spec.purpose(), spec.semantic().name());
            channels.put(spec.purpose(), ch);
        }
        return channels;
    }

    private CaseChannel createQhorusChannel(UUID caseId, String purpose, String semantic) {
        String channelName = CHANNEL_PREFIX + caseId + "/" + purpose;
        QhorusMcpToolsBase.ChannelDetail detail =
                qhorusMcpTools.createChannel(channelName, purpose, semantic, null);
        return new CaseChannel(
                detail.channelId().toString(),
                detail.name(),
                purpose,
                "qhorus",
                Map.of(QHORUS_NAME_KEY, detail.name()));
    }

    private String extractPurpose(String channelName, UUID caseId) {
        String prefix = CHANNEL_PREFIX + caseId + "/";
        return channelName.startsWith(prefix) ? channelName.substring(prefix.length()) : channelName;
    }

    private static CaseChannelLayout selectLayout(String name) {
        return switch (name) {
            case "normative" -> new NormativeChannelLayout();
            case "simple" -> new SimpleLayout();
            default -> throw new IllegalArgumentException("Unknown channel layout: " + name);
        };
    }
}
```

### Step 4c: Update `ClaudonyCaseChannelProviderTest`

- [ ] **Step 3: Rewrite `ClaudonyCaseChannelProviderTest`**

The constructor now takes `(QhorusMcpTools, CaseChannelLayout)` via the package-private constructor, so no config mock is needed in tests. Key behavioral change: `openChannel(caseId, "work")` now triggers 3 `createChannel` calls (layout initialisation); second call for same caseId hits the cache.

```java
package dev.claudony.casehub;

import io.casehub.api.model.CaseChannel;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ClaudonyCaseChannelProviderTest {

    private QhorusMcpTools qhorusMcpTools;
    private ClaudonyCaseChannelProvider provider;

    @BeforeEach
    void setUp() {
        qhorusMcpTools = mock(QhorusMcpTools.class);
        provider = new ClaudonyCaseChannelProvider(qhorusMcpTools, new NormativeChannelLayout());
    }

    private QhorusMcpToolsBase.ChannelDetail channelDetail(UUID channelId, String name) {
        return new QhorusMcpToolsBase.ChannelDetail(
                channelId, name, "description", null, null, 0L, null, false, null, null, null, null);
    }

    private void stubCreateChannel(UUID caseId) {
        when(qhorusMcpTools.createChannel(contains(caseId.toString()), anyString(), anyString(), isNull()))
                .thenAnswer(inv -> channelDetail(UUID.randomUUID(), inv.getArgument(0)));
    }

    @Test
    void openChannel_initializesAllLayoutChannels() {
        UUID caseId = UUID.randomUUID();
        stubCreateChannel(caseId);

        provider.openChannel(caseId, "work");

        // NormativeChannelLayout opens 3 channels on first touch
        verify(qhorusMcpTools, times(3)).createChannel(anyString(), anyString(), anyString(), isNull());
    }

    @Test
    void openChannel_returnsChannelMatchingRequestedPurpose() {
        UUID caseId = UUID.randomUUID();
        stubCreateChannel(caseId);

        CaseChannel ch = provider.openChannel(caseId, "work");

        assertThat(ch).isNotNull();
        assertThat(ch.purpose()).isEqualTo("work");
        assertThat(ch.backendType()).isEqualTo("qhorus");
        assertThat(ch.properties()).containsKey("qhorus-name");
    }

    @Test
    void openChannel_secondCallSameCaseId_hitsCache() {
        UUID caseId = UUID.randomUUID();
        stubCreateChannel(caseId);

        provider.openChannel(caseId, "work");
        provider.openChannel(caseId, "observe");

        // Still only 3 createChannel calls total (initialised on first touch)
        verify(qhorusMcpTools, times(3)).createChannel(anyString(), anyString(), anyString(), isNull());
    }

    @Test
    void openChannel_differentCaseIds_initializeSeparately() {
        UUID caseId1 = UUID.randomUUID();
        UUID caseId2 = UUID.randomUUID();
        stubCreateChannel(caseId1);
        stubCreateChannel(caseId2);

        provider.openChannel(caseId1, "work");
        provider.openChannel(caseId2, "work");

        verify(qhorusMcpTools, times(6)).createChannel(anyString(), anyString(), anyString(), isNull());
    }

    @Test
    void openChannel_purposeNotInLayout_createsAdHocChannel() {
        UUID caseId = UUID.randomUUID();
        stubCreateChannel(caseId);

        CaseChannel ch = provider.openChannel(caseId, "custom-purpose");

        assertThat(ch.purpose()).isEqualTo("custom-purpose");
        // 3 layout channels + 1 ad-hoc
        verify(qhorusMcpTools, times(4)).createChannel(anyString(), anyString(), any(), isNull());
    }

    @Test
    void openChannel_channelNameContainsCaseIdAndPurpose() {
        UUID caseId = UUID.randomUUID();
        stubCreateChannel(caseId);

        provider.openChannel(caseId, "work");

        verify(qhorusMcpTools).createChannel(eq("case-" + caseId + "/work"), anyString(), anyString(), isNull());
    }

    @Test
    void openChannel_passesSemanticToQhorus() {
        UUID caseId = UUID.randomUUID();
        stubCreateChannel(caseId);

        provider.openChannel(caseId, "work");

        verify(qhorusMcpTools).createChannel(contains("/work"), anyString(), eq("APPEND"), isNull());
    }

    @Test
    void postToChannel_sendsViaQhorus() {
        UUID caseId = UUID.randomUUID();
        String channelName = "case-" + caseId + "/work";
        CaseChannel ch = new CaseChannel("ch-id", channelName, "work", "qhorus",
                Map.of("qhorus-name", channelName));

        provider.postToChannel(ch, "alice", "hello");

        verify(qhorusMcpTools).sendMessage(eq(channelName), eq("alice"), anyString(),
                eq("hello"), isNull(), isNull());
    }

    @Test
    void closeChannel_isNoOp() {
        CaseChannel ch = new CaseChannel("ch-id", "channel", "purpose", "qhorus", Map.of("qhorus-name", "ch"));
        assertThatNoException().isThrownBy(() -> provider.closeChannel(ch));
        verifyNoInteractions(qhorusMcpTools);
    }

    @Test
    void listChannels_returnsChannelsFilteredByCaseId() {
        UUID caseId = UUID.randomUUID();
        var matching = channelDetail(UUID.randomUUID(), "case-" + caseId + "/coord");
        var other = channelDetail(UUID.randomUUID(), "case-" + UUID.randomUUID() + "/coord");
        when(qhorusMcpTools.listChannels()).thenReturn(List.of(matching, other));

        List<CaseChannel> result = provider.listChannels(caseId);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).name()).contains(caseId.toString());
    }

    @Test
    void listChannels_noMatchingChannels_returnsEmpty() {
        when(qhorusMcpTools.listChannels()).thenReturn(List.of());
        assertThat(provider.listChannels(UUID.randomUUID())).isEmpty();
    }

    @Test
    void postToChannel_missingQhorusName_fallsBackToChannelId() {
        CaseChannel ch = new CaseChannel("ch-id", "channel", "purpose", "qhorus", Map.of());
        assertThatNoException().isThrownBy(() -> provider.postToChannel(ch, "alice", "hello"));
        verify(qhorusMcpTools).sendMessage(eq("ch-id"), eq("alice"), anyString(),
                eq("hello"), isNull(), isNull());
    }
}
```

- [ ] **Step 4: Run all casehub tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub
```

Expected: All tests pass. If any `ClaudonyCaseChannelProviderTest` test fails, re-check mock stub patterns — `stubCreateChannel` uses `contains(caseId.toString())`, ensure Mockito resolves the stub in the right order.

- [ ] **Step 5: Run full test suite to check nothing regressed**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java \
        claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyCaseChannelProvider.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyCaseChannelProviderTest.java
git commit -m "feat: wire CaseChannelLayout into ClaudonyCaseChannelProvider — init-on-first-touch Closes #87"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `CaseChannelLayout` interface with `channelsFor(UUID, CaseDefinition)` | Task 1 |
| `ChannelSpec` record with `purpose`, `semantic`, `allowedTypes`, `description` | Task 1 |
| `NormativeChannelLayout` opens work/observe/oversight | Task 2 |
| All three channels use APPEND semantic | Task 2 |
| `observe` allows EVENT only | Task 2 |
| `oversight` allows QUERY + COMMAND | Task 2 |
| `work` allows all types (null) | Task 2 |
| `SimpleLayout` opens work + observe only (no oversight) | Task 3 |
| Config property `claudony.casehub.channel-layout=normative` | Task 4a |
| `ClaudonyCaseChannelProvider` uses layout SPI | Task 4b |
| Channels opened on case start (first `openChannel()` call) | Task 4b |
| Tests verify all three channels created with correct semantics | Tasks 2, 3, 4c |

**Placeholder scan:** No TBD, TODO, or "implement later" items. All code blocks are complete.

**Type consistency:**
- `CaseChannelLayout.ChannelSpec` record defined in Task 1, used identically in Tasks 2, 3, 4b/4c
- `NormativeChannelLayout` constructor (no-arg) matches usage in Task 4b `selectLayout()`
- `SimpleLayout` constructor (no-arg) matches usage in Task 4b `selectLayout()`
- Package-private constructor `ClaudonyCaseChannelProvider(QhorusMcpTools, CaseChannelLayout)` defined in Task 4b, used in Task 4c tests
- `createChannel(name, description, semantic, barrierContributors)` — 4-arg overload confirmed from Qhorus source
