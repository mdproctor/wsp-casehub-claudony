# MeshResource Reactive Refactor — QhorusDashboardService Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `MeshResource`'s dependency on `ReactiveQhorusMcpTools` (MCP dispatch layer) with a new `QhorusDashboardService` in Qhorus that provides the correct abstraction level for dashboard consumers.

**Architecture:** Add `QhorusDashboardService` to Qhorus runtime (`io.casehub.qhorus.runtime.dashboard`), extend the in-memory test stores to implement both blocking and reactive interfaces (delegation pattern), then simplify `MeshResource` to inject only `QhorusDashboardService`. Two-repo change: Qhorus first, then Claudony.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny `Uni<T>`, Mockito (pure unit tests), `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn`

**Spec:** `specs/2026-05-20-mesh-resource-reactive-refactor-design.md`  
**Issues:** claudony#119, qhorus#175 (follow-on), qhorus#176 (follow-on)

---

## File Map

**Qhorus repo** (`/Users/mdproctor/claude/casehub/qhorus`):

| Action | File |
|--------|------|
| Create | `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java` |
| Create | `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java` |
| Create | `testing/src/test/java/io/casehub/qhorus/testing/InMemoryStoresDualInterfaceTest.java` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java` |
| Create | `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java` |

**Claudony repo** (`/Users/mdproctor/claude/casehub/claudony`):

| Action | File |
|--------|------|
| Modify | `app/src/main/java/io/casehub/claudony/server/MeshResource.java` |

**Parent repo** (`/Users/mdproctor/claude/casehub/parent`):

| Action | File |
|--------|------|
| Modify | `docs/PLATFORM.md` |
| Create | `docs/protocols/casehub/qhorus-consumer-integration-pattern.md` |

---

## Task 1: In-memory reactive stores (Qhorus testing module)

`InMemoryChannelStore` implements only `ChannelStore` (blocking). `ReactiveChannelService` injects `ReactiveChannelStore` — a different interface with `Uni<T>` return types. Direct multi-implementation is impossible (same method name, incompatible return types). Solution: `InMemoryReactiveChannelStore` and `InMemoryReactiveMessageStore` each inject the corresponding blocking store and delegate to it, wrapping with `Uni.createFrom().item()`. Both share the same backing map via CDI injection.

**Files:**
- Create: `testing/src/test/java/io/casehub/qhorus/testing/InMemoryStoresDualInterfaceTest.java`
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java`
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java`

- [ ] **Step 1.1: Write the failing test**

```java
// testing/src/test/java/io/casehub/qhorus/testing/InMemoryStoresDualInterfaceTest.java
package io.casehub.qhorus.testing;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Duration;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.query.ChannelQuery;
import io.casehub.qhorus.runtime.store.query.MessageQuery;

class InMemoryStoresDualInterfaceTest {

    InMemoryChannelStore channelStore = new InMemoryChannelStore();
    InMemoryMessageStore messageStore = new InMemoryMessageStore();
    InMemoryReactiveChannelStore reactiveChannelStore;
    InMemoryReactiveMessageStore reactiveMessageStore;

    @BeforeEach
    void setUp() {
        reactiveChannelStore = new InMemoryReactiveChannelStore();
        reactiveChannelStore.blocking = channelStore;
        reactiveMessageStore = new InMemoryReactiveMessageStore();
        reactiveMessageStore.blocking = messageStore;
    }

    @AfterEach
    void tearDown() {
        channelStore.clear();
        messageStore.clear();
    }

    @Test
    void blockingChannelPut_visibleToReactiveFindByName() {
        Channel ch = new Channel();
        ch.name = "work";
        ch.semantic = ChannelSemantic.APPEND;
        channelStore.put(ch);

        Optional<Channel> found = reactiveChannelStore.findByName("work")
                .await().atMost(Duration.ofSeconds(1));

        assertThat(found).isPresent();
        assertThat(found.get().name).isEqualTo("work");
    }

    @Test
    void blockingChannelPut_visibleToReactiveScan() {
        Channel ch = new Channel();
        ch.name = "observe";
        ch.semantic = ChannelSemantic.APPEND;
        channelStore.put(ch);

        List<Channel> found = reactiveChannelStore.scan(ChannelQuery.all())
                .await().atMost(Duration.ofSeconds(1));

        assertThat(found).hasSize(1);
    }

    @Test
    void blockingChannelClear_clearsReactiveView() {
        Channel ch = new Channel();
        ch.name = "work";
        ch.semantic = ChannelSemantic.APPEND;
        channelStore.put(ch);
        channelStore.clear();

        List<Channel> found = reactiveChannelStore.scan(ChannelQuery.all())
                .await().atMost(Duration.ofSeconds(1));

        assertThat(found).isEmpty();
    }

    @Test
    void blockingMessagePut_visibleToReactiveScan() {
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = "work";
        ch.semantic = ChannelSemantic.APPEND;

        Message msg = new Message();
        msg.channelId = ch.id;
        msg.sender = "agent:analyst@v1";
        messageStore.put(msg);

        List<Message> found = reactiveMessageStore.scan(MessageQuery.forChannel(ch.id))
                .await().atMost(Duration.ofSeconds(1));

        assertThat(found).hasSize(1);
        assertThat(found.get(0).sender).isEqualTo("agent:analyst@v1");
    }

    @Test
    void blockingMessageCountByChannel_matchesReactiveCount() {
        UUID channelId = UUID.randomUUID();
        for (int i = 0; i < 3; i++) {
            Message msg = new Message();
            msg.channelId = channelId;
            messageStore.put(msg);
        }

        int reactiveCount = reactiveMessageStore.countByChannel(channelId)
                .await().atMost(Duration.ofSeconds(1));

        assertThat(reactiveCount).isEqualTo(3);
    }
}
```

- [ ] **Step 1.2: Run to confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing --also-make \
  -Dtest=InMemoryStoresDualInterfaceTest \
  -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: **COMPILE ERROR** — `InMemoryReactiveChannelStore` and `InMemoryReactiveMessageStore` do not exist yet.

- [ ] **Step 1.3: Implement InMemoryReactiveChannelStore**

```java
// testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java
package io.casehub.qhorus.testing;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.store.ReactiveChannelStore;
import io.casehub.qhorus.runtime.store.query.ChannelQuery;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveChannelStore implements ReactiveChannelStore {

    @Inject
    InMemoryChannelStore blocking;

    @Override
    public Uni<Channel> put(Channel channel) {
        return Uni.createFrom().item(() -> blocking.put(channel));
    }

    @Override
    public Uni<Optional<Channel>> find(UUID id) {
        return Uni.createFrom().item(() -> blocking.find(id));
    }

    @Override
    public Uni<Optional<Channel>> findByName(String name) {
        return Uni.createFrom().item(() -> blocking.findByName(name));
    }

    @Override
    public Uni<List<Channel>> scan(ChannelQuery query) {
        return Uni.createFrom().item(() -> blocking.scan(query));
    }

    @Override
    public Uni<Void> delete(UUID id) {
        return Uni.createFrom().voidItem().invoke(ignored -> blocking.delete(id));
    }
}
```

- [ ] **Step 1.4: Implement InMemoryReactiveMessageStore**

```java
// testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java
package io.casehub.qhorus.testing;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.ReactiveMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveMessageStore implements ReactiveMessageStore {

    @Inject
    InMemoryMessageStore blocking;

    @Override
    public Uni<Message> put(Message message) {
        return Uni.createFrom().item(() -> blocking.put(message));
    }

    @Override
    public Uni<Optional<Message>> find(Long id) {
        return Uni.createFrom().item(() -> blocking.find(id));
    }

    @Override
    public Uni<List<Message>> scan(MessageQuery query) {
        return Uni.createFrom().item(() -> blocking.scan(query));
    }

    @Override
    public Uni<Void> deleteAll(UUID channelId) {
        return Uni.createFrom().voidItem().invoke(ignored -> blocking.deleteAll(channelId));
    }

    @Override
    public Uni<Void> delete(Long id) {
        return Uni.createFrom().voidItem().invoke(ignored -> blocking.delete(id));
    }

    @Override
    public Uni<Integer> countByChannel(UUID channelId) {
        return Uni.createFrom().item(() -> blocking.countByChannel(channelId));
    }

    @Override
    public Uni<Map<UUID, Long>> countAllByChannel() {
        return Uni.createFrom().item(blocking::countAllByChannel);
    }

    @Override
    public Uni<List<String>> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
        return Uni.createFrom().item(() -> blocking.distinctSendersByChannel(channelId, excludedType));
    }
}
```

- [ ] **Step 1.5: Run tests — expect all 5 to pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing --also-make \
  -Dtest=InMemoryStoresDualInterfaceTest \
  -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: `Tests run: 5, Failures: 0, Errors: 0`

- [ ] **Step 1.6: Commit to Qhorus**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java \
  testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java \
  testing/src/test/java/io/casehub/qhorus/testing/InMemoryStoresDualInterfaceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m \
  "feat(testing): add reactive in-memory stores delegating to blocking stores

InMemoryReactiveChannelStore and InMemoryReactiveMessageStore implement the
reactive store interfaces by delegating to the corresponding blocking in-memory
stores. Enables @BeforeEach channelStore.put() setup to be visible to reactive
service code paths (ReactiveChannelService, QhorusDashboardService) in
@QuarkusTest contexts.

Refs claudony#119"
```

---

## Task 2: QhorusDashboardService (Qhorus runtime module)

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`

- [ ] **Step 2.1: Write failing tests**

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java
package io.casehub.qhorus.runtime.dashboard;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.reset;
import static org.mockito.Mockito.when;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ReactiveChannelService;
import io.casehub.qhorus.runtime.instance.Instance;
import io.casehub.qhorus.runtime.instance.ReactiveInstanceService;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.ReactiveMessageService;
import io.casehub.qhorus.runtime.store.ReactiveMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import io.smallrye.mutiny.Uni;

class QhorusDashboardServiceTest {

    ReactiveChannelService channelService = mock(ReactiveChannelService.class);
    ReactiveInstanceService instanceService = mock(ReactiveInstanceService.class);
    ReactiveMessageService messageService = mock(ReactiveMessageService.class);
    ReactiveMessageStore messageStore = mock(ReactiveMessageStore.class);
    QhorusDashboardService service;

    @BeforeEach
    void setUp() {
        service = new QhorusDashboardService();
        service.channelService = channelService;
        service.instanceService = instanceService;
        service.messageService = messageService;
        service.messageStore = messageStore;
        service.mapper = new ObjectMapper();
        reset(channelService, instanceService, messageService, messageStore);
    }

    // ── listChannels ──────────────────────────────────────────────────────────

    @Test
    void listChannels_emptyStore_returnsEmptyList() {
        when(channelService.listAll()).thenReturn(Uni.createFrom().item(List.of()));

        List<QhorusDashboardService.ChannelView> result = service.listChannels()
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).isEmpty();
    }

    @Test
    void listChannels_withChannel_returnsChannelViewWithMessageCount() {
        Channel ch = channel("work", ChannelSemantic.APPEND);
        when(channelService.listAll()).thenReturn(Uni.createFrom().item(List.of(ch)));
        when(messageStore.countByChannel(ch.id)).thenReturn(Uni.createFrom().item(7));

        List<QhorusDashboardService.ChannelView> result = service.listChannels()
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).hasSize(1);
        assertThat(result.get(0).name()).isEqualTo("work");
        assertThat(result.get(0).messageCount()).isEqualTo(7);
        assertThat(result.get(0).paused()).isFalse();
        assertThat(result.get(0).semantic()).isEqualTo("APPEND");
    }

    // ── listInstances ─────────────────────────────────────────────────────────

    @Test
    void listInstances_emptyStore_returnsEmptyList() {
        when(instanceService.listAll()).thenReturn(Uni.createFrom().item(List.of()));

        List<QhorusDashboardService.InstanceView> result = service.listInstances()
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).isEmpty();
    }

    @Test
    void listInstances_withInstance_returnsInstanceViewWithCapabilities() {
        Instance inst = instance("claude:analyst@v1", "analyst");
        when(instanceService.listAll()).thenReturn(Uni.createFrom().item(List.of(inst)));
        when(instanceService.findCapabilityTagsForInstance("claude:analyst@v1"))
                .thenReturn(Uni.createFrom().item(List.of("code-review", "security")));

        List<QhorusDashboardService.InstanceView> result = service.listInstances()
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).hasSize(1);
        assertThat(result.get(0).instanceId()).isEqualTo("claude:analyst@v1");
        assertThat(result.get(0).capabilities()).containsExactly("code-review", "security");
        assertThat(result.get(0).status()).isEqualTo("online");
    }

    // ── getTimeline ───────────────────────────────────────────────────────────

    @Test
    void getTimeline_unknownChannel_returnsEmptyList() {
        when(channelService.findByName("no-such")).thenReturn(Uni.createFrom().item(Optional.empty()));

        List<Map<String, Object>> result = service.getTimeline("no-such", null, 50)
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).isEmpty();
    }

    @Test
    void getTimeline_knownChannel_returnsTimelineEntries() {
        Channel ch = channel("work", ChannelSemantic.APPEND);
        Message msg = message(ch.id, "agent:analyst@v1", MessageType.STATUS, "working on it");
        when(channelService.findByName("work")).thenReturn(Uni.createFrom().item(Optional.of(ch)));
        when(messageStore.scan(any(MessageQuery.class))).thenReturn(Uni.createFrom().item(List.of(msg)));

        List<Map<String, Object>> result = service.getTimeline("work", null, 50)
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).hasSize(1);
        assertThat(result.get(0).get("type")).isEqualTo("MESSAGE");
        assertThat(result.get(0).get("sender")).isEqualTo("agent:analyst@v1");
        assertThat(result.get(0).get("message_type")).isEqualTo("status");
        assertThat(result.get(0).get("content")).isEqualTo("working on it");
    }

    @Test
    void getTimeline_limitCappedAt200() {
        Channel ch = channel("work", ChannelSemantic.APPEND);
        when(channelService.findByName("work")).thenReturn(Uni.createFrom().item(Optional.of(ch)));
        when(messageStore.scan(any(MessageQuery.class))).thenReturn(Uni.createFrom().item(List.of()));

        // Should not throw; limit > 200 is capped internally
        service.getTimeline("work", null, 999).await().atMost(Duration.ofSeconds(1));
    }

    // ── getFeed ───────────────────────────────────────────────────────────────

    @Test
    void getFeed_emptyChannels_returnsEmptyList() {
        when(channelService.listAll()).thenReturn(Uni.createFrom().item(List.of()));

        List<Map<String, Object>> result = service.getFeed(100)
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).isEmpty();
    }

    @Test
    void getFeed_withChannels_tagsEachEntryWithChannelName() {
        Channel ch = channel("work", ChannelSemantic.APPEND);
        Message msg = message(ch.id, "agent:analyst@v1", MessageType.STATUS, "progress");
        msg.createdAt = Instant.now();
        when(channelService.listAll()).thenReturn(Uni.createFrom().item(List.of(ch)));
        when(messageStore.scan(any(MessageQuery.class))).thenReturn(Uni.createFrom().item(List.of(msg)));

        List<Map<String, Object>> result = service.getFeed(100)
                .await().atMost(Duration.ofSeconds(1));

        assertThat(result).hasSize(1);
        assertThat(result.get(0).get("channel")).isEqualTo("work");
    }

    // ── sendHumanMessage ──────────────────────────────────────────────────────

    @Test
    void sendHumanMessage_unknownChannel_throwsIllegalArgumentException() {
        when(channelService.findByName("ghost")).thenReturn(Uni.createFrom().item(Optional.empty()));

        assertThatThrownBy(() -> service.sendHumanMessage("ghost", "human:alice", MessageType.STATUS, "hello")
                .await().atMost(Duration.ofSeconds(1)))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("ghost");
    }

    @Test
    void sendHumanMessage_pausedChannel_throwsIllegalStateException() {
        Channel ch = channel("oversight", ChannelSemantic.APPEND);
        ch.paused = true;
        when(channelService.findByName("oversight")).thenReturn(Uni.createFrom().item(Optional.of(ch)));

        assertThatThrownBy(() -> service.sendHumanMessage("oversight", "human:alice", MessageType.STATUS, "hello")
                .await().atMost(Duration.ofSeconds(1)))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("paused");
    }

    @Test
    void sendHumanMessage_success_returnsHumanMessageResultWithCorrectFields() {
        Channel ch = channel("work", ChannelSemantic.APPEND);
        Message saved = message(ch.id, "human:alice", MessageType.STATUS, "please prioritise security");
        saved.id = 42L;
        saved.messageType = MessageType.STATUS;
        when(channelService.findByName("work")).thenReturn(Uni.createFrom().item(Optional.of(ch)));
        when(messageService.send(eq(ch.id), eq("human:alice"), eq(MessageType.STATUS),
                eq("please prioritise security"), eq(null), eq(null), eq(null), eq(null), eq(ActorType.HUMAN)))
                .thenReturn(Uni.createFrom().item(saved));

        QhorusDashboardService.HumanMessageResult result =
                service.sendHumanMessage("work", "human:alice", MessageType.STATUS, "please prioritise security")
                        .await().atMost(Duration.ofSeconds(1));

        assertThat(result.messageId()).isEqualTo(42L);
        assertThat(result.channelName()).isEqualTo("work");
        assertThat(result.sender()).isEqualTo("human:alice");
        assertThat(result.messageType()).isEqualTo("STATUS");
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private Channel channel(String name, ChannelSemantic semantic) {
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = name;
        ch.semantic = semantic;
        ch.lastActivityAt = Instant.now();
        ch.paused = false;
        return ch;
    }

    private Instance instance(String instanceId, String description) {
        Instance inst = new Instance();
        inst.instanceId = instanceId;
        inst.description = description;
        inst.status = "online";
        inst.lastSeen = Instant.now();
        inst.readOnly = false;
        return inst;
    }

    private Message message(UUID channelId, String sender, MessageType type, String content) {
        Message msg = new Message();
        msg.channelId = channelId;
        msg.sender = sender;
        msg.messageType = type;
        msg.content = content;
        msg.createdAt = Instant.now();
        return msg;
    }
}
```

- [ ] **Step 2.2: Run to confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime --also-make \
  -Dtest=QhorusDashboardServiceTest \
  -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: **COMPILE ERROR** — `QhorusDashboardService` does not exist yet.

- [ ] **Step 2.3: Implement QhorusDashboardService**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java
package io.casehub.qhorus.runtime.dashboard;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ReactiveChannelService;
import io.casehub.qhorus.runtime.instance.Instance;
import io.casehub.qhorus.runtime.instance.ReactiveInstanceService;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.ReactiveMessageService;
import io.casehub.qhorus.runtime.store.ReactiveMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import io.smallrye.mutiny.Uni;

@ApplicationScoped
public class QhorusDashboardService {

    @Inject ReactiveChannelService channelService;
    @Inject ReactiveInstanceService instanceService;
    @Inject ReactiveMessageService messageService;
    @Inject ReactiveMessageStore messageStore;
    @Inject ObjectMapper mapper;

    // ── Response types ────────────────────────────────────────────────────────
    // Field names match QhorusMcpToolsBase DTO shapes for dashboard JS compat.
    // Migration to casehub-qhorus-api tracked in qhorus#175.

    public record ChannelView(
            UUID channelId, String name, String description, String semantic,
            String barrierContributors, long messageCount, String lastActivityAt,
            boolean paused, String allowedWriters, String adminInstances,
            Integer rateLimitPerChannel, Integer rateLimitPerInstance, String allowedTypes) {}

    public record InstanceView(
            String instanceId, String description, String status,
            List<String> capabilities, String lastSeen, boolean readOnly) {}

    public record HumanMessageResult(
            Long messageId, String channelName, String sender, String messageType,
            String correlationId, Long inReplyTo, int parentReplyCount,
            List<String> artefactRefs, String target) {}

    // ── Public API ────────────────────────────────────────────────────────────

    public Uni<List<ChannelView>> listChannels() {
        return channelService.listAll().flatMap(channels -> {
            if (channels.isEmpty()) return Uni.createFrom().item(List.of());
            List<Uni<ChannelView>> unis = channels.stream()
                    .map(ch -> messageStore.countByChannel(ch.id)
                            .map(count -> toChannelView(ch, count)))
                    .toList();
            return Uni.join().all(unis).andFailFast();
        });
    }

    public Uni<List<InstanceView>> listInstances() {
        return instanceService.listAll().flatMap(instances -> {
            if (instances.isEmpty()) return Uni.createFrom().item(List.of());
            List<Uni<InstanceView>> unis = instances.stream()
                    .map(i -> instanceService.findCapabilityTagsForInstance(i.instanceId)
                            .map(tags -> new InstanceView(
                                    i.instanceId, i.description, i.status,
                                    tags, i.lastSeen.toString(), i.readOnly)))
                    .toList();
            return Uni.join().all(unis).andFailFast();
        });
    }

    /** Unknown channel → empty list (consistent with existing MeshResource behavior). */
    public Uni<List<Map<String, Object>>> getTimeline(String channelName, Long afterId, int limit) {
        return channelService.findByName(channelName).flatMap(opt -> {
            if (opt.isEmpty()) return Uni.createFrom().item(List.of());
            int effectiveLimit = Math.min(Math.max(limit, 1), 200);
            return messageStore.scan(MessageQuery.poll(opt.get().id, afterId, effectiveLimit))
                    .map(msgs -> msgs.stream().map(this::toTimelineEntry).toList());
        });
    }

    /** Fetches per-channel timelines in parallel (improvement over sequential loop). */
    public Uni<List<Map<String, Object>>> getFeed(int limit) {
        return channelService.listAll().flatMap(channels -> {
            if (channels.isEmpty()) return Uni.createFrom().item(List.of());
            int perChannel = Math.max(5, limit / channels.size());
            List<Uni<List<Map<String, Object>>>> unis = channels.stream()
                    .map(ch -> messageStore.scan(MessageQuery.poll(ch.id, null, perChannel))
                            .map(msgs -> msgs.stream()
                                    .map(m -> {
                                        Map<String, Object> tagged = new HashMap<>(toTimelineEntry(m));
                                        tagged.put("channel", ch.name);
                                        return tagged;
                                    })
                                    .toList())
                            .onFailure().recoverWithItem(List.of()))
                    .toList();
            return Uni.join().all(unis).andFailFast().map(lists -> {
                List<Map<String, Object>> combined = lists.stream()
                        .flatMap(List::stream)
                        .collect(Collectors.toCollection(ArrayList::new));
                combined.sort(Comparator.comparing(
                        m -> String.valueOf(m.getOrDefault("created_at", ""))));
                return combined.size() > limit ? combined.subList(0, limit) : combined;
            });
        });
    }

    /**
     * Post a message from an authenticated human operator.
     * Checks channel existence (→ IllegalArgumentException) and paused state (→ IllegalStateException).
     * Human operators bypass agent-to-agent ACL and rate limiting.
     */
    public Uni<HumanMessageResult> sendHumanMessage(
            String channelName, String sender, MessageType type, String content) {
        return channelService.findByName(channelName)
                .map(opt -> opt.orElseThrow(
                        () -> new IllegalArgumentException("Channel not found: " + channelName)))
                .invoke(ch -> {
                    if (ch.paused) throw new IllegalStateException(
                            "Channel '" + channelName + "' is paused");
                })
                .flatMap(ch -> messageService.send(
                        ch.id, sender, type, content, null, null, null, null, ActorType.HUMAN))
                .map(m -> new HumanMessageResult(
                        m.id, channelName, m.sender,
                        m.messageType != null ? m.messageType.name() : null,
                        m.correlationId, m.inReplyTo, 0, List.of(), m.target));
    }

    // ── Private mapping — replicated from QhorusMcpToolsBase ─────────────────
    // Deduplication into a shared QhorusEntityMapper tracked in qhorus#176.

    private ChannelView toChannelView(Channel ch, int count) {
        return new ChannelView(
                ch.id, ch.name, ch.description,
                ch.semantic != null ? ch.semantic.name() : null,
                ch.barrierContributors, count,
                ch.lastActivityAt != null ? ch.lastActivityAt.toString() : null,
                ch.paused, ch.allowedWriters, ch.adminInstances,
                ch.rateLimitPerChannel, ch.rateLimitPerInstance, ch.allowedTypes);
    }

    private Map<String, Object> toTimelineEntry(Message m) {
        Map<String, Object> entry = new LinkedHashMap<>();
        entry.put("id", m.id);
        if (m.messageType == MessageType.EVENT) {
            entry.put("type", "EVENT");
            entry.put("created_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("occurred_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("agent_id", m.sender);
            entry.put("message_type", null);
            String toolName = null;
            Long durationMs = null;
            Long tokenCount = null;
            if (m.content != null) {
                try {
                    JsonNode node = mapper.readTree(m.content);
                    JsonNode tn = node.get("tool_name");
                    if (tn != null && tn.isTextual()) toolName = tn.asText();
                    JsonNode dm = node.get("duration_ms");
                    if (dm != null && dm.isNumber()) durationMs = dm.asLong();
                    JsonNode tc = node.get("token_count");
                    if (tc != null && tc.isNumber()) tokenCount = tc.asLong();
                } catch (Exception ignored) {
                }
            }
            entry.put("tool_name", toolName);
            entry.put("duration_ms", durationMs);
            entry.put("token_count", tokenCount);
        } else {
            entry.put("type", "MESSAGE");
            entry.put("created_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("sender", m.sender);
            entry.put("message_type", m.messageType != null ? m.messageType.name().toLowerCase() : null);
            entry.put("content", m.content);
            entry.put("correlation_id", m.correlationId);
            entry.put("tool_name", null);
        }
        return entry;
    }
}
```

- [ ] **Step 2.4: Run tests — expect all 12 to pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime --also-make \
  -Dtest=QhorusDashboardServiceTest \
  -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: `Tests run: 12, Failures: 0, Errors: 0`

- [ ] **Step 2.5: Run full Qhorus runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime --also-make \
  -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: All pre-existing tests pass. No regressions.

- [ ] **Step 2.6: Commit to Qhorus**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m \
  "feat(dashboard): add QhorusDashboardService for consumer integration

Provides the correct abstraction layer for dashboard/UI consumers — between the
low-level entity services and the MCP dispatch layer (ReactiveQhorusMcpTools).

Operations: listChannels (parallel message counts), listInstances (parallel
capability tags), getTimeline, getFeed (parallel per-channel fan-out),
sendHumanMessage. Defines ChannelView, InstanceView, HumanMessageResult DTOs.
Migration to casehub-qhorus-api tracked in qhorus#175.
Mapping deduplication tracked in qhorus#176.

Refs claudony#119"
```

---

## Task 3: Install Qhorus SNAPSHOT in Claudony

- [ ] **Step 3.1: Build and install Qhorus to local Maven repo**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests \
  -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: `BUILD SUCCESS` — `casehub-qhorus 0.2-SNAPSHOT` installed to `~/.m2`.

- [ ] **Step 3.2: Run Claudony tests with updated Qhorus SNAPSHOT (regression baseline)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml
```

Expected: All existing tests pass (475 tests, same baseline as before). The new `InMemoryReactiveChannelStore` and `InMemoryReactiveMessageStore` are now on the classpath via `casehub-qhorus-testing`, but `MeshResource` still uses `ReactiveQhorusMcpTools` — no behavior change yet.

If tests fail here, diagnose before proceeding.

---

## Task 4: Simplify MeshResource (Claudony app module)

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/MeshResource.java`

The existing `MeshResourceTest` and `MeshResourceInterjectionTest` serve as the test suite — they are black-box REST tests that verify the HTTP API contract. They pass without modification once `MeshResource` is correctly refactored.

- [ ] **Step 4.1: Implement simplified MeshResource**

Replace the entire file content:

```java
// app/src/main/java/io/casehub/claudony/server/MeshResource.java
package io.casehub.claudony.server;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.Set;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.claudony.config.ClaudonyConfig;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.dashboard.QhorusDashboardService;
import io.quarkus.security.Authenticated;
import io.quarkus.security.identity.SecurityIdentity;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;

@Path("/api/mesh")
@Produces(MediaType.APPLICATION_JSON)
@Authenticated
public class MeshResource {

    private static final Set<MessageType> VALID_HUMAN_TYPES = Set.of(
            MessageType.QUERY, MessageType.COMMAND, MessageType.RESPONSE,
            MessageType.STATUS, MessageType.DECLINE, MessageType.HANDOFF,
            MessageType.DONE, MessageType.EVENT);

    record MeshConfig(String strategy, int interval) {}
    record PostMessageRequest(String content, String type) {}

    @Inject ClaudonyConfig config;
    @Inject QhorusDashboardService dashboard;
    @Inject ObjectMapper mapper;
    @Inject SecurityIdentity securityIdentity;

    @GET
    @Path("/config")
    public MeshConfig config() {
        return new MeshConfig(config.meshRefreshStrategy(), config.meshRefreshInterval());
    }

    @GET
    @Path("/channels")
    public Uni<List<QhorusDashboardService.ChannelView>> channels() {
        return dashboard.listChannels();
    }

    @GET
    @Path("/instances")
    public Uni<List<QhorusDashboardService.InstanceView>> instances() {
        return dashboard.listInstances();
    }

    @GET
    @Path("/channels/{name}/timeline")
    public Uni<List<Map<String, Object>>> timeline(
            @PathParam("name") String name,
            @QueryParam("after") Long after,
            @QueryParam("limit") @DefaultValue("50") int limit) {
        return dashboard.getTimeline(name, after, limit);
    }

    @GET
    @Path("/feed")
    public Uni<List<Map<String, Object>>> feed(
            @QueryParam("limit") @DefaultValue("100") int limit) {
        return dashboard.getFeed(limit);
    }

    @GET
    @Path("/events")
    @Produces("text/event-stream")
    public Multi<String> events() {
        long intervalMs = config.meshRefreshInterval();
        return Multi.createFrom().ticks().every(Duration.ofMillis(intervalMs))
                .onItem().transformToUniAndConcatenate(tick -> {
                    Uni<List<QhorusDashboardService.ChannelView>> channels =
                            dashboard.listChannels().onFailure().recoverWithItem(List.of());
                    Uni<List<QhorusDashboardService.InstanceView>> instances =
                            dashboard.listInstances().onFailure().recoverWithItem(List.of());
                    Uni<List<Map<String, Object>>> feed =
                            dashboard.getFeed(100).onFailure().recoverWithItem(List.of());
                    return Uni.combine().all().unis(channels, instances, feed).asTuple()
                            .map(tuple -> {
                                try {
                                    return "data: " + mapper.writeValueAsString(Map.of(
                                            "channels", tuple.getItem1(),
                                            "instances", tuple.getItem2(),
                                            "feed", tuple.getItem3())) + "\n\n";
                                } catch (Exception e) {
                                    return "data: {}\n\n";
                                }
                            })
                            .onFailure().recoverWithItem("data: {}\n\n");
                });
    }

    @POST
    @Path("/channels/{name}/messages")
    @Consumes(MediaType.APPLICATION_JSON)
    public Uni<Response> postMessage(
            @PathParam("name") String name,
            PostMessageRequest req) {
        if (req == null || req.content() == null || req.content().isBlank()) {
            return Uni.createFrom().item(Response.status(400).entity("content must not be blank").build());
        }
        MessageType type;
        try {
            type = MessageType.valueOf((req.type() == null ? "status" : req.type()).toUpperCase());
        } catch (IllegalArgumentException e) {
            return Uni.createFrom().item(
                    Response.status(400).entity("invalid type: " + req.type()).build());
        }
        if (!VALID_HUMAN_TYPES.contains(type)) {
            return Uni.createFrom().item(
                    Response.status(400).entity("invalid type: " + req.type()).build());
        }
        String sender = "human:" + securityIdentity.getPrincipal().getName();
        return dashboard.sendHumanMessage(name, sender, type, req.content())
                .map(result -> Response.ok(result).build())
                .onFailure(IllegalArgumentException.class)
                    .recoverWithItem(e -> Response.status(404).entity(e.getMessage()).build())
                .onFailure(IllegalStateException.class)
                    .recoverWithItem(e -> Response.status(409).entity(e.getMessage()).build());
    }
}
```

- [ ] **Step 4.2: Run MeshResource-specific tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="MeshResourceTest,MeshResourceInterjectionTest" \
  -pl claudony-app --also-make \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml
```

Expected: All tests pass. If `MeshResourceInterjectionTest` fails on channel-not-found, verify `InMemoryReactiveChannelStore` and `InMemoryReactiveMessageStore` are on the test classpath and are `@Alternative @Priority(1)`.

- [ ] **Step 4.3: Run full Claudony test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -f /Users/mdproctor/claude/casehub/claudony/pom.xml
```

Expected: Same test count as baseline (475 passing). Zero failures.

- [ ] **Step 4.4: Commit to Claudony**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/main/java/io/casehub/claudony/server/MeshResource.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m \
  "refactor(mesh): replace ReactiveQhorusMcpTools with QhorusDashboardService (#119)

MeshResource now injects QhorusDashboardService — the correct integration tier
for dashboard consumers. Removes @Blocking from all REST endpoints (now return
Uni<T>). Eliminates ToolCallException unwrapping. The /events SSE endpoint
builds its snapshot in parallel (Uni.combine) instead of sequentially.

Closes #119"
```

---

## Task 5: Update PLATFORM.md and add protocol file (parent repo)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`
- Create: `/Users/mdproctor/claude/casehub/parent/docs/protocols/casehub/qhorus-consumer-integration-pattern.md`

- [ ] **Step 5.1: Update PLATFORM.md — Capability Ownership table**

In the Capability Ownership table, add a new row after the `casehub-qhorus` agent messaging row:

```
| Dashboard read/write API (composed views: channel with message count, instance with capability tags, timeline mapping, human message send) | `casehub-qhorus` | `QhorusDashboardService` in `io.casehub.qhorus.runtime.dashboard` — inject this for dashboard/UI consumers. Do NOT inject raw entity services for this use case. |
```

- [ ] **Step 5.2: Update PLATFORM.md — Boundary Rules section**

Find and replace the existing rule:

> "Do not call `QhorusMcpTools` or `ReactiveQhorusMcpTools` from consumer service code. Those classes are the MCP tool dispatch layer for external callers (Claude Code). Consumer service code should inject `ChannelService` / `ReactiveChannelService` or `MessageService` / `ReactiveMessageService` directly, or implement the `ChannelBackend` SPI."

Replace with:

> "Do not call `QhorusMcpTools` or `ReactiveQhorusMcpTools` from consumer service code. Those classes are the MCP tool dispatch layer for external callers (Claude Code); they carry `@WrapBusinessError` exception semantics that internal consumers must not be exposed to. Consumer service code has three correct integration points: (1) **Dashboard/UI consumers** needing composed views (channel with message count, instance with capability tags, timeline entries) — inject `QhorusDashboardService`. (2) **Service-layer integrations** needing raw entity access (e.g. SPI implementations, background workers) — inject `ReactiveChannelService` / `ReactiveMessageService` directly. (3) **Reactive event-driven integrations** — implement `ChannelBackend` or `MessageObserver` SPI. The "inject entity services directly" alternative is also wrong for dashboard consumers: `ReactiveChannelService.listAll()` returns entities without message counts, forcing consumers to inject `ReactiveMessageStore` (store layer) — bypassing the service layer and creating a worse coupling than the one it fixes. See `docs/protocols/casehub/qhorus-consumer-integration-pattern.md`."

- [ ] **Step 5.3: Add protocol file**

```markdown
<!-- /Users/mdproctor/claude/casehub/parent/docs/protocols/casehub/qhorus-consumer-integration-pattern.md -->
---
id: PP-20260520-mesh-dashboard
title: "Qhorus consumer integration: use QhorusDashboardService for dashboard consumers, not raw entity services"
type: rule
scope: platform
applies_to: "Any REST resource, scheduled task, or service in a consumer repo (claudony, devtown, aml, clinical) that reads or writes to Qhorus channels, instances, or messages"
severity: critical
refs:
  - claudony#119
  - qhorus#175
  - qhorus#176
created: 2026-05-20
---

## Rule

Do not call `QhorusMcpTools` or `ReactiveQhorusMcpTools` from consumer service code.
Do not inject `ReactiveChannelService` / `ReactiveMessageService` directly for dashboard-style operations.

Inject `QhorusDashboardService` instead.

## Why the rule exists

`QhorusMcpTools` is the MCP protocol dispatch layer for Claude Code. Calling it from internal service code:
- Exposes `@WrapBusinessError` exception semantics (`ToolCallException` wrapping), forcing consumers to catch and unwrap
- Creates coupling to an external protocol layer that may evolve independently

"Inject entity services directly" sounds like the correct fix but is also wrong for dashboard consumers:
- `ReactiveChannelService.listAll()` returns raw `Channel` entities without message counts
- To build `ChannelView` (with count), the consumer must inject `ReactiveMessageStore` (store layer) directly — bypassing the service layer and creating a worse cross-layer coupling

## The three correct integration points

| Consumer type | Integration point |
|---|---|
| Dashboard / UI (needs composed views: count, tags, timeline) | `QhorusDashboardService` |
| Service-layer integration (SPI impls, background workers needing raw entities) | `ReactiveChannelService` / `ReactiveMessageService` |
| Reactive event-driven (reacting to new messages on a channel) | `ChannelBackend` or `MessageObserver` SPI |

## Violation hint

A REST resource or service bean imports from `io.casehub.qhorus.runtime.mcp.*` and calls `.await()` or `@Blocking` on tool methods — or injects `ReactiveMessageStore` alongside `ReactiveChannelService` to assemble composed views.

## Follow-on

- qhorus#175: move `ChannelView`, `InstanceView`, `HumanMessageResult` DTO types to `casehub-qhorus-api`
- qhorus#176: extract `toTimelineEntry` / `toChannelDetail` mapping to a shared `QhorusEntityMapper`
```

- [ ] **Step 5.4: Commit to parent repo**

```bash
git -C /Users/mdproctor/claude/casehub/parent add \
  docs/PLATFORM.md \
  docs/protocols/casehub/qhorus-consumer-integration-pattern.md
git -C /Users/mdproctor/claude/casehub/parent commit -m \
  "docs: add QhorusDashboardService integration pattern + update boundary rule

Three-tier consumer integration model: QhorusDashboardService (dashboard/UI),
entity services (SPI/worker code), ChannelBackend/MessageObserver SPIs (events).
Corrects the previous rule which prescribed injecting entity services directly —
that approach forces store-layer coupling for composed views (message counts).

Refs claudony#119"
```

---

## Self-Review

**Spec coverage:**
- ✅ `QhorusDashboardService` with all 5 operations (listChannels, listInstances, getTimeline, getFeed, sendHumanMessage)
- ✅ `ChannelView`, `InstanceView`, `HumanMessageResult` DTOs with JSON-compatible field names
- ✅ `InMemoryReactiveChannelStore` + `InMemoryReactiveMessageStore` delegation pattern
- ✅ `MeshResource` simplified: `@Blocking` removed, `Uni<T>` returns, no `ToolCallException`
- ✅ `/events` SSE parallel snapshot via `Uni.combine().all()`
- ✅ PLATFORM.md boundary rule updated (three-tier model)
- ✅ New protocol file added
- ✅ Follow-on issues referenced (qhorus#175, qhorus#176)
- ✅ `VALID_HUMAN_TYPES` migrated from `Set<String>` to `Set<MessageType>`

**Type consistency:**
- `QhorusDashboardService.ChannelView` used consistently in `MeshResource.channels()` return type
- `QhorusDashboardService.InstanceView` used in `MeshResource.instances()`
- `QhorusDashboardService.HumanMessageResult` returned by `sendHumanMessage` and mapped to `Response.ok()` in `MeshResource`
- `InMemoryReactiveChannelStore.blocking` field name matches test setup
- `service.channelService`, `service.instanceService` etc. field names in tests match `QhorusDashboardService` field declarations

**Placeholder scan:** None found. All steps contain complete code.
