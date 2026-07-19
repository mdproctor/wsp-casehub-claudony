# Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #175 — rich conversation integration: absorb chat-app capabilities into Claudony workspace
**Issue group:** #175

**Goal:** Build a `<claudony-workbench>` LitElement that composes terminal, channel navigation, message feed, and contextual panels (task/correlation/artifact) for case-bound sessions, backed by a fully enriched MeshResource API that surfaces the complete Qhorus conversation model.

**Architecture:** Cross-repo changes to Qhorus (findByChannel SPI, timeline enrichment, sendHumanMessage widening) → Claudony backend enrichment (MeshResource validation, commitment endpoint) → frontend adapter enrichment → terminal extraction → absorbed panels → workbench composition → entry point routing.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, Lit 3, `@casehubio/blocks-ui-channel-activity` ^0.2.2, `@casehubio/pages-ui-tokens`, vitest, Playwright

## Global Constraints

- Java 21 API surface (compiled on Java 26): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Maven: `mvn` not `./mvnw`
- Qhorus: `0.2-SNAPSHOT` — changes installed locally via `mvn install -DskipTests`
- IntelliJ MCP mandatory for all `.java`/`.ts` edits — never bash grep/Edit on source files
- Fleet mode (standalone sessions) must remain untouched
- SSE + poll transport — no WebSocket changes
- Absorbed panels: zero Claudony-specific imports — blocks-ui types only
- Test baseline: 591 Java + 15 vitest — must not decrease
- Docker required for dev/test (PostgreSQL Dev Services)

---

### Task 1: Qhorus — `findByChannel` SPI + implementations

**Files:**
- Modify: `~/claude/casehub/qhorus/api/src/main/java/io/casehub/qhorus/api/store/CommitmentStore.java`
- Modify: `~/claude/casehub/qhorus/api/src/main/java/io/casehub/qhorus/api/store/ReactiveCommitmentStore.java`
- Modify: `~/claude/casehub/qhorus/persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCommitmentStore.java`
- Modify: `~/claude/casehub/qhorus/persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveCommitmentStore.java`
- Modify: `~/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaCommitmentStore.java`
- Test: `~/claude/casehub/qhorus/persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/InMemoryCommitmentStoreTest.java`
- Test: `~/claude/casehub/qhorus/persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveCommitmentStoreTest.java`

**Interfaces:**
- Produces: `CommitmentStore.findByChannel(UUID channelId): List<Commitment>` — all commitments for a channel regardless of state
- Produces: `ReactiveCommitmentStore.findByChannel(UUID channelId): Uni<List<Commitment>>` — reactive equivalent

- [ ] **Step 1: Write failing test for `InMemoryCommitmentStore.findByChannel`**

Add to `InMemoryCommitmentStoreTest.java`:

```java
@Test
void findByChannel_returnsAllStates() {
    UUID channelId = UUID.randomUUID();
    UUID otherChannel = UUID.randomUUID();
    Commitment open = store.save(Commitment.builder()
            .correlationId("c1").channelId(channelId)
            .messageType(MessageType.COMMAND).requester("a").obligor("b")
            .state(CommitmentState.OPEN).build());
    Commitment fulfilled = store.save(Commitment.builder()
            .correlationId("c2").channelId(channelId)
            .messageType(MessageType.COMMAND).requester("a").obligor("b")
            .state(CommitmentState.FULFILLED).build());
    store.save(Commitment.builder()
            .correlationId("c3").channelId(otherChannel)
            .messageType(MessageType.COMMAND).requester("a").obligor("b")
            .state(CommitmentState.OPEN).build());

    List<Commitment> result = store.findByChannel(channelId);
    assertThat(result).hasSize(2);
    assertThat(result).extracting(Commitment::correlationId)
            .containsExactlyInAnyOrder("c1", "c2");
}

@Test
void findByChannel_emptyChannel_returnsEmpty() {
    assertThat(store.findByChannel(UUID.randomUUID())).isEmpty();
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/qhorus/pom.xml -pl persistence-memory -Dtest=InMemoryCommitmentStoreTest#findByChannel_returnsAllStates
```

Expected: compilation error — `findByChannel` does not exist.

- [ ] **Step 3: Add `findByChannel` to `CommitmentStore` interface**

Use `ide_insert_member` on `CommitmentStore.java`:

```java
List<Commitment> findByChannel(UUID channelId);
```

- [ ] **Step 4: Add `findByChannel` to `ReactiveCommitmentStore` interface**

Use `ide_insert_member` on `ReactiveCommitmentStore.java`:

```java
Uni<List<Commitment>> findByChannel(UUID channelId);
```

- [ ] **Step 5: Implement in `InMemoryCommitmentStore`**

Use `ide_insert_member` on `InMemoryCommitmentStore.java`:

```java
@Override
public List<Commitment> findByChannel(UUID channelId) {
    return byId.values().stream()
            .filter(c -> channelId.equals(c.channelId()))
            .toList();
}
```

- [ ] **Step 6: Implement in `InMemoryReactiveCommitmentStore`**

Use `ide_insert_member` on `InMemoryReactiveCommitmentStore.java`:

```java
@Override
public Uni<List<Commitment>> findByChannel(UUID channelId) {
    return Uni.createFrom().item(delegate.findByChannel(channelId));
}
```

- [ ] **Step 7: Implement in `ReactiveJpaCommitmentStore`**

Use `ide_insert_member` on `ReactiveJpaCommitmentStore.java`:

```java
@Override
public Uni<List<Commitment>> findByChannel(UUID channelId) {
    return repo.<CommitmentEntity>list("channelId = ?1 ORDER BY createdAt ASC", channelId)
            .map(list -> list.stream().map(CommitmentEntity::toDomain).toList());
}
```

- [ ] **Step 8: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/qhorus/pom.xml -pl persistence-memory -Dtest=InMemoryCommitmentStoreTest
```

- [ ] **Step 9: Write failing test for reactive store**

Add to `InMemoryReactiveCommitmentStoreTest.java`:

```java
@Test
void findByChannel_returnsAllStates() {
    UUID channelId = UUID.randomUUID();
    store.save(Commitment.builder()
            .correlationId("c1").channelId(channelId)
            .messageType(MessageType.COMMAND).requester("a").obligor("b")
            .state(CommitmentState.OPEN).build()).await().indefinitely();
    store.save(Commitment.builder()
            .correlationId("c2").channelId(channelId)
            .messageType(MessageType.COMMAND).requester("a").obligor("b")
            .state(CommitmentState.FULFILLED).build()).await().indefinitely();

    List<Commitment> result = store.findByChannel(channelId).await().indefinitely();
    assertThat(result).hasSize(2);
}
```

- [ ] **Step 10: Run reactive tests — verify pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/qhorus/pom.xml -pl persistence-memory -Dtest=InMemoryReactiveCommitmentStoreTest
```

- [ ] **Step 11: Verify diagnostics**

Run `ide_diagnostics` on all modified files to confirm no compilation errors.

- [ ] **Step 12: Install Qhorus SNAPSHOT locally**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -f ~/claude/casehub/qhorus/pom.xml
```

- [ ] **Step 13: Commit**

```bash
git -C ~/claude/casehub/qhorus add -A && git -C ~/claude/casehub/qhorus commit -m "feat: add findByChannel to CommitmentStore SPI

Returns all commitments for a channel regardless of state.
Implemented in InMemory, InMemoryReactive, and ReactiveJpa stores.

Refs casehubio/claudony#175"
```

---

### Task 2: Qhorus — Enrich `toTimelineEntry()` + widen `sendHumanMessage()`

**Files:**
- Modify: `~/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java`
- Modify: `~/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`
- Test: `~/claude/casehub/qhorus/runtime/src/test/java/io/casehub/qhorus/runtime/QhorusEntityMapperTest.java` (create if absent)

**Interfaces:**
- Consumes: `Message.inReplyTo()`, `Message.artefactRefs()`, `Message.target()`, `Message.replyCount()`, `Message.deadline()`, `Message.topic()`
- Produces: `toTimelineEntry()` output map now includes keys: `in_reply_to`, `artefact_refs`, `target`, `reply_count`, `deadline`, `topic`
- Produces: `sendHumanMessage(channelName, sender, type, content, inReplyTo, correlationId, artefactRefs, target, deadline, topic)` — 10-arg signature

- [ ] **Step 1: Write failing test for enriched `toTimelineEntry()`**

Create or add to `QhorusEntityMapperTest.java`:

```java
@Test
void toTimelineEntry_message_includesEnrichedFields() {
    ObjectMapper mapper = new ObjectMapper();
    QhorusEntityMapper entityMapper = new QhorusEntityMapper(mapper);

    Message m = Message.builder()
            .id(1L).channelId(UUID.randomUUID()).sender("agent-1")
            .messageType(MessageType.COMMAND).actorType(ActorType.AGENT)
            .content("do something").correlationId("corr-1")
            .inReplyTo(42L).target("agent-2").replyCount(3)
            .deadline(Instant.parse("2026-08-01T00:00:00Z"))
            .topic("work-items")
            .artefactRefs(List.of(ArtefactRef.builder()
                    .uri("doc://spec.md").type(ArtefactType.DOCUMENT).label("Spec").build()))
            .build();

    Map<String, Object> entry = entityMapper.toTimelineEntry(m);

    assertThat(entry.get("in_reply_to")).isEqualTo(42L);
    assertThat(entry.get("target")).isEqualTo("agent-2");
    assertThat(entry.get("reply_count")).isEqualTo(3);
    assertThat(entry.get("topic")).isEqualTo("work-items");
    assertThat(entry.get("deadline")).isNotNull();
    assertThat(entry.get("artefact_refs")).isNotNull();
    assertThat((List<?>) entry.get("artefact_refs")).hasSize(1);
}

@Test
void toTimelineEntry_message_nullOptionalFields() {
    ObjectMapper mapper = new ObjectMapper();
    QhorusEntityMapper entityMapper = new QhorusEntityMapper(mapper);

    Message m = Message.builder()
            .id(2L).channelId(UUID.randomUUID()).sender("agent-1")
            .messageType(MessageType.STATUS).actorType(ActorType.AGENT)
            .content("status update")
            .build();

    Map<String, Object> entry = entityMapper.toTimelineEntry(m);

    assertThat(entry.get("in_reply_to")).isNull();
    assertThat(entry.get("target")).isNull();
    assertThat(entry.get("reply_count")).isEqualTo(0);
    assertThat(entry.get("topic")).isNull();
    assertThat(entry.get("deadline")).isNull();
    assertThat(entry.get("artefact_refs")).isNull();
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/qhorus/pom.xml -pl runtime -Dtest=QhorusEntityMapperTest#toTimelineEntry_message_includesEnrichedFields
```

- [ ] **Step 3: Enrich `toTimelineEntry()` MESSAGE branch**

Use `ide_replace_member` on `QhorusEntityMapper.java`, method `toTimelineEntry` (2-arg overload). In the `else` (MESSAGE) branch, after the existing `correlation_id` line, add:

```java
entry.put("in_reply_to", m.inReplyTo());
entry.put("target", m.target());
entry.put("reply_count", m.replyCount());
entry.put("topic", m.topic());
entry.put("deadline", m.deadline() != null ? m.deadline().toString() : null);
entry.put("artefact_refs", m.artefactRefs() != null && !m.artefactRefs().isEmpty()
        ? m.artefactRefs() : null);
```

- [ ] **Step 4: Run tests — verify pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/qhorus/pom.xml -pl runtime -Dtest=QhorusEntityMapperTest
```

- [ ] **Step 5: Widen `sendHumanMessage()` signature**

Use `ide_edit_member` on `QhorusDashboardService.java`, method `sendHumanMessage`. New signature and body:

```java
public Uni<HumanMessageResult> sendHumanMessage(
        String channelName, String sender, MessageType type, String content,
        Long inReplyTo, String correlationId,
        java.util.List<io.casehub.qhorus.api.message.ArtefactRef> artefactRefs,
        String target, java.time.Instant deadline, String topic) {
    return channelService.findByName(channelName)
            .map(opt -> opt.orElseThrow(
                    () -> new IllegalArgumentException("Channel not found: " + channelName)))
            .flatMap(ch -> messageService.dispatch(
                    MessageDispatch.builder()
                            .channelId(ch.id())
                            .sender(sender)
                            .type(type)
                            .content(content)
                            .actorType(io.casehub.platform.api.identity.ActorType.HUMAN)
                            .inReplyTo(inReplyTo)
                            .correlationId(correlationId)
                            .artefactRefs(artefactRefs)
                            .target(target)
                            .deadline(deadline)
                            .topic(topic)
                            .build()))
            .map(result -> new HumanMessageResult(
                    result.messageId(), channelName, result.sender(),
                    result.type() != null ? result.type().name() : null,
                    result.correlationId(), result.inReplyTo(),
                    result.parentReplyCount(),
                    result.artefactRefs(),
                    result.target()));
}
```

- [ ] **Step 6: Find and update callers of `sendHumanMessage()`**

Run `ide_find_references` on `sendHumanMessage` to find all callers across qhorus. Update any existing callers to pass `null` for the new parameters. The main caller will be `MeshResource` in Claudony (updated in Task 3).

- [ ] **Step 7: Verify diagnostics across qhorus**

Run `ide_diagnostics` on `QhorusDashboardService.java` and `QhorusEntityMapper.java`.

- [ ] **Step 8: Run full qhorus tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/qhorus/pom.xml
```

- [ ] **Step 9: Install Qhorus SNAPSHOT**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -f ~/claude/casehub/qhorus/pom.xml
```

- [ ] **Step 10: Commit**

```bash
git -C ~/claude/casehub/qhorus add -A && git -C ~/claude/casehub/qhorus commit -m "feat: enrich toTimelineEntry + widen sendHumanMessage

toTimelineEntry MESSAGE entries now include: in_reply_to, artefact_refs,
target, reply_count, deadline, topic.

sendHumanMessage accepts full dispatch fields: inReplyTo, correlationId,
artefactRefs, target, deadline, topic.

Refs casehubio/claudony#175"
```

---

### Task 3: Claudony backend — Widen `PostMessageRequest` + commitment endpoint

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/MeshResource.java`
- Modify: `app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java`

**Interfaces:**
- Consumes: `QhorusDashboardService.sendHumanMessage(name, sender, type, content, inReplyTo, correlationId, artefactRefs, target, deadline, topic)` (from Task 2)
- Consumes: `ReactiveCommitmentStore.findByChannel(UUID)` (from Task 1)
- Produces: `POST /api/mesh/channels/{name}/messages` accepts `inReplyTo`, `correlationId`, `artefactRefs`, `target`, `deadline`, `topic`
- Produces: `GET /api/mesh/channels/{name}/commitments` returns commitment list
- Produces: Validation: RESPONSE/DONE/DECLINE require `inReplyTo`+`correlationId`; HANDOFF requires `target`; EVENT rejects non-null `content`

- [ ] **Step 1: Write failing tests for widened request + validation**

Add to `MeshResourceTest.java`:

```java
@Test
void postMessage_response_withoutInReplyTo_returns400() {
    given().contentType("application/json")
            .body("""
                    {"content":"reply text","type":"RESPONSE","correlationId":"c1"}
                    """)
            .post("/api/mesh/channels/test-channel/messages")
            .then().statusCode(400);
}

@Test
void postMessage_handoff_withoutTarget_returns400() {
    given().contentType("application/json")
            .body("""
                    {"content":"handoff","type":"HANDOFF","inReplyTo":1,"correlationId":"c1"}
                    """)
            .post("/api/mesh/channels/test-channel/messages")
            .then().statusCode(400);
}

@Test
void postMessage_command_withTarget_succeeds() {
    // Setup: create a channel via channelStore first
    // Then post a COMMAND with target
    given().contentType("application/json")
            .body("""
                    {"content":"do this","type":"COMMAND","target":"agent-2","topic":"work"}
                    """)
            .post("/api/mesh/channels/" + testChannelName + "/messages")
            .then().statusCode(200);
}
```

```java
@Test
void commitments_endpoint_returns200() {
    given().get("/api/mesh/channels/" + testChannelName + "/commitments")
            .then().statusCode(200)
            .body("$", instanceOf(List.class));
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest#postMessage_response_withoutInReplyTo_returns400
```

- [ ] **Step 3: Widen `PostMessageRequest` record**

Use `ide_edit_member` on `MeshResource.java`, member `PostMessageRequest`:

```java
record PostMessageRequest(String content, String type,
                          Long inReplyTo, String correlationId,
                          java.util.List<io.casehub.qhorus.api.message.ArtefactRef> artefactRefs,
                          String target, String deadline, String topic) {}
```

- [ ] **Step 4: Add validation to `postMessage()`**

Use `ide_replace_member` on `MeshResource.java`, method `postMessage`. Add speech-act validation before calling `sendHumanMessage`:

```java
switch (type) {
    case RESPONSE, DONE, DECLINE -> {
        if (req.inReplyTo() == null)
            return Uni.createFrom().item(Response.status(400)
                    .entity(type.name() + " requires inReplyTo").build());
        if (req.correlationId() == null || req.correlationId().isBlank())
            return Uni.createFrom().item(Response.status(400)
                    .entity(type.name() + " requires correlationId").build());
    }
    case HANDOFF -> {
        if (req.inReplyTo() == null)
            return Uni.createFrom().item(Response.status(400)
                    .entity("HANDOFF requires inReplyTo").build());
        if (req.correlationId() == null || req.correlationId().isBlank())
            return Uni.createFrom().item(Response.status(400)
                    .entity("HANDOFF requires correlationId").build());
        if (req.target() == null || req.target().isBlank())
            return Uni.createFrom().item(Response.status(400)
                    .entity("HANDOFF requires target").build());
    }
    default -> {}
}
```

Pass all new fields through to `sendHumanMessage`:

```java
Instant deadline = req.deadline() != null ? Instant.parse(req.deadline()) : null;
return dashboard.sendHumanMessage(name, sender, type, content,
        req.inReplyTo(), req.correlationId(), req.artefactRefs(),
        req.target(), deadline, req.topic())
```

- [ ] **Step 5: Add commitment endpoint**

Use `ide_insert_member` on `MeshResource.java`:

```java
@GET
@Path("/channels/{name}/commitments")
public Uni<List<Map<String, Object>>> commitments(@PathParam("name") String name) {
    return channelService.findByName(name)
            .flatMap(opt -> {
                if (opt.isEmpty()) return Uni.createFrom().item(List.<Map<String, Object>>of());
                return commitmentStore.findByChannel(opt.get().id())
                        .map(commitments -> commitments.stream()
                                .map(this::toCommitmentMap).toList());
            });
}

private Map<String, Object> toCommitmentMap(io.casehub.qhorus.api.message.Commitment c) {
    Map<String, Object> map = new java.util.LinkedHashMap<>();
    map.put("id", c.id());
    map.put("correlationId", c.correlationId());
    map.put("state", c.state().name());
    map.put("requester", c.requester());
    map.put("obligor", c.obligor());
    map.put("expiresAt", c.expiresAt() != null ? c.expiresAt().toString() : null);
    map.put("acknowledgedAt", c.acknowledgedAt() != null ? c.acknowledgedAt().toString() : null);
    map.put("resolvedAt", c.resolvedAt() != null ? c.resolvedAt().toString() : null);
    map.put("delegatedTo", c.delegatedTo());
    map.put("createdAt", c.createdAt() != null ? c.createdAt().toString() : null);
    return map;
}
```

Inject `ReactiveCommitmentStore`:

```java
@Inject ReactiveCommitmentStore commitmentStore;
```

- [ ] **Step 6: Run all MeshResource tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest
```

- [ ] **Step 7: Run full Claudony test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

- [ ] **Step 8: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/main/java/io/casehub/claudony/server/MeshResource.java app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java
git -C ~/claude/casehub/claudony commit -m "feat(#175): widen PostMessageRequest + add commitment endpoint

PostMessageRequest now accepts inReplyTo, correlationId, artefactRefs,
target, deadline, topic. Speech-act validation enforced (RESPONSE/DONE/
DECLINE require inReplyTo+correlationId, HANDOFF requires target).

New GET /api/mesh/channels/{name}/commitments endpoint surfaces all
commitments for a channel via ReactiveCommitmentStore.findByChannel().

Refs #175"
```

---

### Task 4: Frontend adapter enrichment

**Files:**
- Modify: `app/src/main/webui/src/util/channel-adapter.ts`
- Modify: `app/src/main/webui/src/util/channel-adapter.test.ts`

**Interfaces:**
- Consumes: Enriched timeline entries from MeshResource (Task 3) — `in_reply_to`, `artefact_refs`, `target`, `reply_count`, `deadline`, `topic`
- Consumes: Commitment endpoint response from MeshResource (Task 3)
- Produces: `TimelineEntry` interface with all enriched fields
- Produces: `toQhorusMessage()` maps all fields to `QhorusMessage`
- Produces: `CommitmentRecord` type for panel consumption
- Produces: `toCommitmentMap(commitments): Map<string, CommitmentRecord>` mapping function

- [ ] **Step 1: Write failing tests for enriched adapter**

Add to `channel-adapter.test.ts`:

```typescript
describe('toQhorusMessage enriched fields', () => {
  it('maps inReplyTo from timeline entry', () => {
    const entry: Partial<TimelineEntry> = {
      id: 1, sender: 'agent-1', content: 'reply',
      message_type: 'RESPONSE', in_reply_to: 42,
      correlation_id: 'corr-1', created_at: '2026-01-01T00:00:00Z',
    };
    const msg = toQhorusMessage(entry);
    expect(msg.inReplyTo).toBe('42');
    expect(msg.correlationId).toBe('corr-1');
  });

  it('maps artefactRefs from timeline entry', () => {
    const entry: Partial<TimelineEntry> = {
      id: 2, sender: 'agent-1', content: 'cmd',
      message_type: 'COMMAND', created_at: '2026-01-01T00:00:00Z',
      artefact_refs: [{ uri: 'doc://spec.md', type: 'DOCUMENT', label: 'Spec' }],
    };
    const msg = toQhorusMessage(entry);
    expect(msg.artefactRefs).toHaveLength(1);
    expect(msg.artefactRefs[0].uri).toBe('doc://spec.md');
  });

  it('maps target, replyCount, topic, deadline', () => {
    const entry: Partial<TimelineEntry> = {
      id: 3, sender: 'agent-1', content: 'do this',
      message_type: 'COMMAND', created_at: '2026-01-01T00:00:00Z',
      target: 'agent-2', reply_count: 5, topic: 'work',
      deadline: '2026-08-01T00:00:00Z',
    };
    const msg = toQhorusMessage(entry);
    expect(msg.target).toBe('agent-2');
    expect(msg.replyCount).toBe(5);
    expect(msg.topic).toBe('work');
    expect(msg.deadline).toBe('2026-08-01T00:00:00Z');
  });

  it('defaults enriched fields when absent', () => {
    const entry: Partial<TimelineEntry> = {
      id: 4, sender: 'agent-1', content: 'hello',
      message_type: 'STATUS', created_at: '2026-01-01T00:00:00Z',
    };
    const msg = toQhorusMessage(entry);
    expect(msg.inReplyTo).toBeUndefined();
    expect(msg.correlationId).toBeUndefined();
    expect(msg.artefactRefs).toEqual([]);
    expect(msg.target).toBeUndefined();
    expect(msg.replyCount).toBe(0);
    expect(msg.topic).toBe('');
    expect(msg.deadline).toBeUndefined();
  });
});
```

```typescript
describe('CommitmentRecord mapping', () => {
  it('maps commitment response to CommitmentRecord', () => {
    const raw = {
      id: 'uuid-1', correlationId: 'corr-1', state: 'OPEN',
      requester: 'human:alice', obligor: 'agent-1',
      expiresAt: '2026-08-01T00:00:00Z', acknowledgedAt: null,
      resolvedAt: null, createdAt: '2026-07-19T10:00:00Z',
    };
    const record = toCommitmentRecord(raw);
    expect(record.state).toBe('OPEN');
    expect(record.deadline).toBe('2026-08-01T00:00:00Z');
    expect(record.createdAt).toBe('2026-07-19T10:00:00Z');
    expect(record.updatedAt).toBe('2026-07-19T10:00:00Z');
  });

  it('updatedAt is max of resolvedAt, acknowledgedAt, createdAt', () => {
    const raw = {
      id: 'uuid-2', correlationId: 'corr-2', state: 'FULFILLED',
      requester: 'human:alice', obligor: 'agent-1',
      acknowledgedAt: '2026-07-19T11:00:00Z',
      resolvedAt: '2026-07-19T12:00:00Z',
      createdAt: '2026-07-19T10:00:00Z',
    };
    const record = toCommitmentRecord(raw);
    expect(record.updatedAt).toBe('2026-07-19T12:00:00Z');
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
GITHUB_TOKEN=$(gh auth token) npm --prefix app/src/main/webui test
```

- [ ] **Step 3: Widen `TimelineEntry` interface**

Use Edit on `channel-adapter.ts`. Add fields to the `TimelineEntry` interface:

```typescript
export interface TimelineEntry {
  id?: number;
  type?: string;
  message_type?: string;
  sender?: string;
  content?: string | null;
  created_at?: string;
  agent_id?: string;
  tool_name?: string;
  duration_ms?: number | null;
  token_count?: number | null;
  in_reply_to?: number | null;
  correlation_id?: string;
  artefact_refs?: ArtefactRef[] | null;
  target?: string | null;
  reply_count?: number;
  deadline?: string | null;
  topic?: string | null;
}
```

Add the `ArtefactRef` import from blocks-ui:

```typescript
import type { QhorusMessage, QhorusChannel, MessageType, ArtefactRef } from '@casehubio/blocks-ui-channel-activity';
```

- [ ] **Step 4: Enrich `toQhorusMessage()` mapping**

Update the return object to map all new fields:

```typescript
export function toQhorusMessage(entry: Partial<TimelineEntry>): QhorusMessage {
  const isEvent = entry.type === 'EVENT';
  const sender = isEvent ? (entry.agent_id || 'system') : (entry.sender || 'unknown');
  const rawType = isEvent ? 'EVENT' : (entry.message_type || 'status');

  return {
    id: String(entry.id ?? 0),
    channelId: '',
    sender,
    messageType: rawType.toUpperCase() as MessageType,
    actorType: resolveActorType(sender),
    content: entry.content ?? '',
    topic: entry.topic ?? '',
    correlationId: entry.correlation_id,
    inReplyTo: entry.in_reply_to ? String(entry.in_reply_to) : undefined,
    artefactRefs: entry.artefact_refs ?? [],
    target: entry.target ?? undefined,
    replyCount: entry.reply_count ?? 0,
    deadline: entry.deadline ?? undefined,
    createdAt: entry.created_at || new Date().toISOString(),
  };
}
```

- [ ] **Step 5: Add `CommitmentRecord` type and mapping function**

Add to `channel-adapter.ts`:

```typescript
import type { CommitmentState } from '@casehubio/blocks-ui-channel-activity';

export interface CommitmentRecord {
  readonly state: CommitmentState;
  readonly deadline?: string;
  readonly acknowledgedAt?: string;
  readonly createdAt: string;
  readonly updatedAt: string;
}

interface RawCommitment {
  id: string;
  correlationId: string;
  state: string;
  requester?: string;
  obligor?: string;
  expiresAt?: string | null;
  acknowledgedAt?: string | null;
  resolvedAt?: string | null;
  createdAt?: string | null;
}

export function toCommitmentRecord(raw: RawCommitment): CommitmentRecord {
  const timestamps = [raw.resolvedAt, raw.acknowledgedAt, raw.createdAt]
    .filter((t): t is string => t != null);
  const updatedAt = timestamps.length > 0
    ? timestamps.reduce((a, b) => a > b ? a : b)
    : raw.createdAt ?? new Date().toISOString();

  return {
    state: raw.state as CommitmentState,
    deadline: raw.expiresAt ?? undefined,
    acknowledgedAt: raw.acknowledgedAt ?? undefined,
    createdAt: raw.createdAt ?? new Date().toISOString(),
    updatedAt,
  };
}

export function toCommitmentMap(
  commitments: RawCommitment[]
): Map<string, CommitmentRecord> {
  const map = new Map<string, CommitmentRecord>();
  for (const c of commitments) {
    map.set(c.correlationId, toCommitmentRecord(c));
  }
  return map;
}
```

- [ ] **Step 6: Run tests — verify pass**

```bash
GITHUB_TOKEN=$(gh auth token) npm --prefix app/src/main/webui test
```

- [ ] **Step 7: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/main/webui/src/util/channel-adapter.ts app/src/main/webui/src/util/channel-adapter.test.ts
git -C ~/claude/casehub/claudony commit -m "feat(#175): enrich adapter — full QhorusMessage mapping + CommitmentRecord

TimelineEntry now carries in_reply_to, artefact_refs, target, reply_count,
deadline, topic. toQhorusMessage maps all fields. CommitmentRecord type and
toCommitmentMap function for commitment endpoint consumption.

Refs #175"
```

---

### Task 5: Terminal extraction — `terminal-controller.ts`

**Files:**
- Create: `app/src/main/webui/src/util/terminal-controller.ts`
- Modify: `app/src/main/webui/src/components/terminal-workspace.ts` — refactor to use controller
- Modify: `app/src/main/webui/src/terminal.ts` — update imports if needed

**Interfaces:**
- Produces: `attachTerminal(container, sessionId, opts?): TerminalHandle`
- Produces: `TerminalHandle { dispose(), resize(cols, rows), sendInput(text), switchSession(sessionId, opts?) }`

- [ ] **Step 1: Create `terminal-controller.ts`**

Use `ide_create_file`:

```typescript
import "@casehubio/pages-component-terminal";
import type { PagesTerminal } from "@casehubio/pages-component-terminal";
import { fetchWithAuth } from "./auth";

export interface TerminalHandle {
  dispose(): void;
  resize(cols: number, rows: number): void;
  sendInput(text: string): void;
  switchSession(sessionId: string, opts?: { proxyPeer?: string }): void;
  getTerminal(): PagesTerminal | null;
}

function buildWsUrl(sessionId: string, proxyPeer?: string): string {
  const proto = location.protocol === "https:" ? "wss:" : "ws:";
  return proxyPeer
    ? `${proto}//${location.host}/ws/proxy/${proxyPeer}/${sessionId}/{cols}/{rows}`
    : `${proto}//${location.host}/ws/${sessionId}/{cols}/{rows}`;
}

export function attachTerminal(
  container: HTMLElement,
  sessionId: string,
  opts?: { proxyPeer?: string },
): TerminalHandle {
  const el = document.createElement("pages-component-terminal") as PagesTerminal;
  container.appendChild(el);

  let currentSessionId = sessionId;
  let currentProxyPeer = opts?.proxyPeer;

  el.configure({ wsUrl: buildWsUrl(sessionId, opts?.proxyPeer) });

  return {
    dispose() {
      el.remove();
    },

    resize(cols: number, rows: number) {
      const resizeUrl = currentProxyPeer
        ? `/api/peers/${currentProxyPeer}/sessions/${currentSessionId}/resize?cols=${cols}&rows=${rows}`
        : `/api/sessions/${currentSessionId}/resize?cols=${cols}&rows=${rows}`;
      fetchWithAuth(resizeUrl, { method: "POST" }).catch(() => {});
    },

    sendInput(text: string) {
      el.sendInput(text);
    },

    switchSession(newSessionId: string, switchOpts?: { proxyPeer?: string }) {
      currentSessionId = newSessionId;
      currentProxyPeer = switchOpts?.proxyPeer ?? currentProxyPeer;
      el.configure({ wsUrl: buildWsUrl(newSessionId, currentProxyPeer) });
    },

    getTerminal() {
      return el;
    },
  };
}
```

- [ ] **Step 2: Refactor `terminal-workspace.ts` to use controller**

Use `ide_edit_member` on `ClaudonyTerminalWorkspace`. Replace the direct `PagesTerminal` management with `attachTerminal()`:

- Remove `_terminal` field, replace with `_handle: TerminalHandle | null`
- In `render()`, create a `<div id="terminal-container">` instead of `<pages-component-terminal>`
- In `configure()`, call `attachTerminal()` on the container div
- In `handleResize()`, call `_handle.resize()`
- In `wireEvents()` key-pressed handler, call `_handle.sendInput()`
- In `handleWorkerSwitch()`, call `_handle.switchSession()`
- `getTerminal()` delegates to `_handle.getTerminal()`

- [ ] **Step 3: Run existing E2E tests to verify no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=TerminalPageE2ETest
```

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/main/webui/src/util/terminal-controller.ts app/src/main/webui/src/components/terminal-workspace.ts
git -C ~/claude/casehub/claudony commit -m "refactor(#175): extract terminal-controller.ts from terminal-workspace

attachTerminal() encapsulates PagesTerminal creation, WebSocket URL
construction, proxy peer routing, and tmux resize calls. terminal-workspace
now delegates to TerminalHandle. Shared module for workbench reuse.

Refs #175"
```

---

### Task 6: Absorbed panels — task, correlation, artifact

**Files:**
- Create: `app/src/main/webui/src/components/claudony-task-panel.ts`
- Create: `app/src/main/webui/src/components/claudony-correlation-panel.ts`
- Create: `app/src/main/webui/src/components/claudony-artifact-panel.ts`

**Interfaces:**
- Consumes: `QhorusMessage[]`, `Map<string, CommitmentRecord>`, `ArtefactRef` — all from blocks-ui types
- Produces: `pages-event` with `channel:message-selected` topic
- Produces: Custom elements `<claudony-task-panel>`, `<claudony-correlation-panel>`, `<claudony-artifact-panel>`

- [ ] **Step 1: Create `claudony-task-panel.ts`**

Absorb from chat-app's `qhorus-task-panel.ts`. Use `ide_create_file`. The panel is a direct port — same logic, same rendering, same `pages-event` output. Key adaptations:
- Element name: `claudony-task-panel` (not `qhorus-task-panel`)
- Import `CommitmentRecord` from `../util/channel-adapter.js` (local type until promoted to blocks-ui)
- Use `@casehubio/pages-ui-tokens` CSS variables (already in the chat-app version)

Port the full 200 lines from `~/claude/casehub/chat-app/src/main/webui/src/panels/qhorus-task-panel.ts`, changing only the element name registration and the `CommitmentRecord` import source.

- [ ] **Step 2: Create `claudony-correlation-panel.ts`**

Same approach — port from `~/claude/casehub/chat-app/src/main/webui/src/panels/qhorus-correlation-panel.ts`. Change element name to `claudony-correlation-panel`. No Claudony-specific imports needed — it uses only `QhorusMessage` and `CommitmentRecord`.

- [ ] **Step 3: Create `claudony-artifact-panel.ts`**

Port from `~/claude/casehub/chat-app/src/main/webui/src/panels/qhorus-artifact-panel.ts`. Change element name to `claudony-artifact-panel`. For v1, the `resolveArtifact` callback receives a stub that returns `{content: ref.label, language: undefined}` — metadata display only, no content fetching.

- [ ] **Step 4: Verify diagnostics — no compilation errors**

Run TypeScript check:

```bash
GITHUB_TOKEN=$(gh auth token) npm --prefix app/src/main/webui run build
```

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/main/webui/src/components/claudony-task-panel.ts app/src/main/webui/src/components/claudony-correlation-panel.ts app/src/main/webui/src/components/claudony-artifact-panel.ts
git -C ~/claude/casehub/claudony commit -m "feat(#175): absorb task, correlation, artifact panels from chat-app

Three standalone LitElements ported from chat-app. Zero Claudony-specific
imports — use only blocks-ui types (QhorusMessage, CommitmentState,
ArtefactRef) and local CommitmentRecord. Artifact panel uses stub resolver
(v1: metadata only). All candidates for blocks-ui promotion.

Refs #175"
```

---

### Task 7: `<claudony-workbench>` LitElement

**Files:**
- Create: `app/src/main/webui/src/components/claudony-workbench.ts`

**Interfaces:**
- Consumes: `attachTerminal()` from `terminal-controller.ts` (Task 5)
- Consumes: `toQhorusMessage()`, `toCommitmentMap()`, `CommitmentRecord` from adapter (Task 4)
- Consumes: `<channel-nav>`, `<channel-feed>`, `<channel-input>` from blocks-ui
- Consumes: `<claudony-task-panel>`, `<claudony-correlation-panel>`, `<claudony-artifact-panel>` (Task 6)
- Consumes: MeshResource REST/SSE endpoints (Tasks 2-3)
- Produces: `<claudony-workbench>` custom element — composition root for case-bound sessions

This is the largest task. The workbench owns:
1. SSE connection lifecycle (per-channel, close-and-reopen)
2. Reactive state: channels, messages, commitments, selectedChannelId, selectedMessageId, replyTo, selectedArtefactRef
3. Event routing: blocks-ui events → MeshResource POST calls
4. Terminal integration via `attachTerminal()`
5. Case context: header, lineage, worker list (absorb from existing channel-panel + worker-panel)
6. Cursor persistence, stale cursor detection
7. Channel-switch resets (clear selectedMessageId, replyTo, artefactRef)
8. Worker-switch lifecycle

- [ ] **Step 1: Create the workbench file**

Use `ide_create_file` for `app/src/main/webui/src/components/claudony-workbench.ts`. The workbench follows the same structure as the chat-app's `qhorus-workbench.ts` but with Claudony's data sources. Key sections:

**Imports and element registration:**
```typescript
import { LitElement, html, css, nothing, type PropertyValues } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import type { QhorusMessage, QhorusChannel, MessageType, ArtefactRef } from '@casehubio/blocks-ui-channel-activity';
import { ChannelEventTopics } from '@casehubio/blocks-ui-channel-activity';
import '@casehubio/blocks-ui-channel-activity';
import { attachTerminal, type TerminalHandle } from '../util/terminal-controller.js';
import { toQhorusMessage, toQhorusChannel, toCommitmentMap, type TimelineEntry, type ChannelInfo, type CommitmentRecord } from '../util/channel-adapter.js';
import { fetchWithAuth } from '../util/auth.js';
import './claudony-task-panel.js';
import './claudony-correlation-panel.js';
import './claudony-artifact-panel.js';
```

**State properties** (mirror the spec's §Data Flow and §channel-panel.ts Logic Redistribution):
```typescript
@state() private _channels: QhorusChannel[] = [];
@state() private _messages: QhorusMessage[] = [];
@state() private _commitments: Map<string, CommitmentRecord> = new Map();
@state() private _selectedChannelId = '';
@state() private _selectedMessageId?: string;
@state() private _replyTo?: { messageId: string; senderName: string };
@state() private _selectedArtefactRef?: ArtefactRef;
@state() private _dockState: Record<string, boolean> = { nav: true, tasks: false, correlation: false, artifacts: false };
// ... case context, stale cursor, worker state from channel-panel.ts redistribution
```

**Lifecycle:** `configure(opts)`, `connectedCallback()`, `firstUpdated()` (attaches terminal), `disconnectedCallback()` (cleanup).

**Data flow methods:** `_loadChannels()`, `_selectChannel()`, `_openEventSource()`, `_closeEventSource()`, `_pollChannel()`, `_fetchCommitments()`, `_appendMessages()`, `_sendMessage()` — all absorbed from channel-panel.ts logic redistribution table, with the enriched adapter.

**Event routing:** `_onPagesEvent(e)` handler for all blocks-ui events — `send-message`, `select-channel`, `message-selected`, `artefact-selected`, `worker-selected`, `terminal-resize`, `key-pressed`.

**Render:** Desktop layout with dock strip, panels rendered via `_renderPanel(panelId)` switch.

This file will be ~500-600 lines. Build it incrementally — start with the data flow (SSE + state), add terminal, add panels, add event routing.

- [ ] **Step 2: Verify it builds**

```bash
GITHUB_TOKEN=$(gh auth token) npm --prefix app/src/main/webui run build
```

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/main/webui/src/components/claudony-workbench.ts
git -C ~/claude/casehub/claudony commit -m "feat(#175): claudony-workbench LitElement — composition root

Composes terminal, channel-nav, channel-feed, channel-input, task panel,
correlation panel, artifact panel. Owns SSE lifecycle, reactive state,
event routing, case context, cursor persistence. Absorbs all channel-panel
and terminal-workspace responsibilities for case-bound sessions.

Refs #175"
```

---

### Task 8: Entry point routing + dynamic import

**Files:**
- Modify: `app/src/main/webui/src/terminal.ts`

**Interfaces:**
- Consumes: `<claudony-workbench>` from Task 7 (dynamic import)
- Consumes: Session API response `{ caseId?: string }` to determine mode

- [ ] **Step 1: Add conditional rendering to `terminal.ts`**

After the session fetch (`fetchWithAuth("/api/sessions/" + sessionId)`), check `session.caseId`:

```typescript
if (session?.caseId) {
  const { ClaudonyWorkbench } = await import('./components/claudony-workbench.js');
  // Replace terminal-workspace with workbench
  const workspaceSlot = document.querySelector('claudony-terminal-workspace');
  if (workspaceSlot) {
    const workbench = document.createElement('claudony-workbench') as InstanceType<typeof ClaudonyWorkbench>;
    workspaceSlot.replaceWith(workbench);
    workbench.configure({
      sessionId,
      sessionName,
      proxyPeer,
      caseId: session.caseId,
      roleName: session.roleName,
      status: session.status,
      createdAt: session.createdAt,
      channel,
    });
  }
} else {
  // Existing fleet-mode path — unchanged
  workspace.configure({ sessionId, sessionName, proxyPeer, channel });
}
```

- [ ] **Step 2: Verify esbuild code splitting works**

Check that the workbench chunk is separate from the terminal chunk:

```bash
GITHUB_TOKEN=$(gh auth token) npm --prefix app/src/main/webui run build
ls -la app/src/main/webui/dist/
```

Verify a separate chunk file appears for the workbench code.

- [ ] **Step 3: Run E2E tests — fleet mode unchanged**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=TerminalPageE2ETest
```

- [ ] **Step 4: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/main/webui/src/terminal.ts
git -C ~/claude/casehub/claudony commit -m "feat(#175): conditional workbench/fleet routing based on case context

Case-bound sessions dynamically import and render <claudony-workbench>.
Standalone sessions keep existing terminal-workspace (fleet mode).
Dynamic import keeps workbench code out of the fleet-mode bundle.

Refs #175"
```

---

### Task 9: E2E tests for workbench

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/e2e/WorkbenchE2ETest.java`

**Interfaces:**
- Consumes: Workbench rendering for case-bound sessions
- Consumes: MeshResource endpoints with enriched data

- [ ] **Step 1: Create workbench E2E test class**

Extend `PlaywrightBase`. Set up a case-bound session in `@BeforeEach` with a `caseId`. Key test scenarios:

```java
@Test
void workbench_rendersForCaseBoundSession() {
    // Navigate to session page with a caseId
    // Assert workbench element is present, terminal-workspace is not
}

@Test
void fleetMode_rendersForStandaloneSession() {
    // Navigate to session page without caseId
    // Assert terminal-workspace is present, workbench is not
}

@Test
void channelNav_displaysChannels() {
    // Assert channel-nav element present with channel list
}

@Test
void taskPanel_showsCommitments() {
    // Post a COMMAND message, verify task panel shows it
}
```

- [ ] **Step 2: Run E2E tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=WorkbenchE2ETest
```

- [ ] **Step 3: Run full E2E suite to verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e
```

- [ ] **Step 4: Run complete test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/test/java/io/casehub/claudony/e2e/WorkbenchE2ETest.java
git -C ~/claude/casehub/claudony commit -m "test(#175): E2E tests for workbench rendering and fleet-mode routing

Verifies workbench renders for case-bound sessions, fleet mode for
standalone. Tests channel-nav, task panel with commitments.

Refs #175"
```

---

### Task 10: File deferred issues + CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — update test count and project structure

- [ ] **Step 1: File deferred GitHub issues**

Per the spec's scope boundaries, file issues for deferred items:

```bash
gh issue create --repo casehubio/claudony --title "feat: general-purpose chat rooms (not case-scoped)" --body "Follow-on from #175. Allow user-created channels not tied to any case."
gh issue create --repo casehubio/claudony --title "feat: reactions and member/presence panels" --body "Follow-on from #175. Wire <channel-member-panel> and reaction support from blocks-ui."
gh issue create --repo casehubio/claudony --title "feat: responsive layouts for tablet/phone" --body "Follow-on from #175. Workbench layouts for tablet (tabs) and phone (drawers)."
gh issue create --repo casehubio/claudony --title "chore: promote task/correlation/artifact panels to blocks-ui" --body "Follow-on from #175. After proven in Claudony, promote to @casehubio/blocks-ui-channel-activity."
```

- [ ] **Step 2: Update CLAUDE.md**

Update test count, add workbench to project structure section, add `terminal-controller.ts` and `claudony-workbench.ts` to the file tree.

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/casehub/claudony add CLAUDE.md
git -C ~/claude/casehub/claudony commit -m "docs(#175): update CLAUDE.md — workbench structure and test count

Refs #175"
```
