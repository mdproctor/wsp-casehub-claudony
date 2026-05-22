# Ecosystem Persistence Isolation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate Qhorus to a named persistence unit (`qhorus`), close all JPA bypass paths in the SPI by adding `PendingReplyStore` and two aggregate `MessageStore` methods, then update Claudony's config and tests to use the new datasource names and `quarkus-qhorus-testing`.

**Architecture:** Two-phase — Phase 1 works in the Qhorus project (`~/claude/quarkus-qhorus`), builds and installs the updated artifact, then Phase 2 wires Claudony to it. All persistence goes through store interfaces; no MCP tool or service calls `EntityManager` or Panache statics directly. The `InMemory*Store` alternatives in `quarkus-qhorus-testing` make all Qhorus data DB-free in unit tests.

**Tech Stack:** Java 21, Quarkus 3.x, Hibernate ORM (named persistence units), Panache, Flyway, JUnit 5, `quarkus-qhorus-testing` (InMemory alternatives), H2 (dev/single-node), PostgreSQL (prod/multi-node).

**Test commands:**
- Qhorus: `cd /Users/mdproctor/claude/quarkus-qhorus && mvn test`
- Claudony: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` (from `~/claude/claudony`)

---

## File Map

### Qhorus project (`~/claude/quarkus-qhorus`)

| Action | File |
|--------|------|
| Modify | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/MessageStore.java` |
| Modify | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaMessageStore.java` |
| Modify | `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryMessageStore.java` |
| **Create** | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/PendingReplyStore.java` |
| **Create** | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaPendingReplyStore.java` |
| **Create** | `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryPendingReplyStore.java` |
| Modify | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageService.java` |
| Modify | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/PendingReplyCleanupJob.java` |
| Modify | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java` |
| Modify | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` |
| Modify | `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntryRepository.java` |
| Modify | `runtime/src/test/resources/application.properties` |
| **Create** | `adr/0004-ledger-jpa-inheritance-constraint.md` |
| Modify | `adr/INDEX.md` |

### Claudony project (`~/claude/claudony`)

| Action | File |
|--------|------|
| Modify | `pom.xml` |
| Modify | `src/main/resources/application.properties` |
| Modify | `src/test/java/dev/claudony/.../MeshResourceInterjectionTest.java` |
| Modify | `docs/DESIGN.md` |
| Modify | `CLAUDE.md` |

---

## Phase 1 — Qhorus

---

### Task 1: Create GitHub epic and issue in Qhorus

**Files:** none

- [ ] **Step 1: Create the epic**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
gh issue create \
  --title "Epic: Named persistence unit isolation" \
  --body "Migrate Qhorus from the default datasource to named persistence unit 'qhorus'. Close all JPA bypass paths in the SPI. Prerequisite for Claudony multi-node fleet operation." \
  --label "epic"
```

Note the epic issue number — call it `EPIC_N`.

- [ ] **Step 2: Create the implementation issue**

```bash
gh issue create \
  --title "Named PU migration: PendingReplyStore SPI, MessageStore aggregates, bypass closure" \
  --body "Refs #EPIC_N

- Add PendingReplyStore as sixth store interface + JPA + InMemory implementations
- Add countByChannels and distinctNonEventSendersInChannel to MessageStore
- Close Message.getEntityManager() bypasses in QhorusMcpTools and ReactiveQhorusMcpTools
- Wire PendingReplyStore into MessageService and PendingReplyCleanupJob
- Add @PersistenceUnit(\"qhorus\") to AgentMessageLedgerEntryRepository
- Update test application.properties for named PU" \
  --label "enhancement"
```

Note the issue number — call it `ISSUE_N`.

---

### Task 2: MessageStore aggregate methods (TDD)

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/MessageStore.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaMessageStore.java`
- Modify: `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryMessageStore.java`
- Test: `runtime/src/test/java/io/quarkiverse/qhorus/runtime/store/InMemoryMessageStoreAggregateTest.java`

- [ ] **Step 1: Write the failing tests**

Create `runtime/src/test/java/io/quarkiverse/qhorus/runtime/store/InMemoryMessageStoreAggregateTest.java`:

```java
package io.quarkiverse.qhorus.runtime.store;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.testing.InMemoryMessageStore;

class InMemoryMessageStoreAggregateTest {

    private InMemoryMessageStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryMessageStore();
        store.clear();
    }

    // --- countByChannels ---

    @Test
    void countByChannels_returnsCorrectCountsForEachChannel() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        putMessage(ch1, "alice", MessageType.REQUEST);
        putMessage(ch1, "bob", MessageType.RESPONSE);
        putMessage(ch2, "carol", MessageType.REQUEST);

        Map<UUID, Long> counts = store.countByChannels(List.of(ch1, ch2));

        assertEquals(2L, counts.get(ch1));
        assertEquals(1L, counts.get(ch2));
    }

    @Test
    void countByChannels_emptyInput_returnsEmptyMap() {
        Map<UUID, Long> counts = store.countByChannels(List.of());
        assertTrue(counts.isEmpty());
    }

    @Test
    void countByChannels_channelWithNoMessages_absentFromResult() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        putMessage(ch1, "alice", MessageType.REQUEST);

        Map<UUID, Long> counts = store.countByChannels(List.of(ch1, ch2));

        assertEquals(1L, counts.get(ch1));
        assertNull(counts.get(ch2));
        assertEquals(0L, counts.getOrDefault(ch2, 0L));
    }

    // --- distinctNonEventSendersInChannel ---

    @Test
    void distinctNonEventSendersInChannel_excludesEventMessages() {
        UUID channelId = UUID.randomUUID();
        putMessage(channelId, "alice", MessageType.REQUEST);
        putMessage(channelId, "bob", MessageType.EVENT);
        putMessage(channelId, "carol", MessageType.RESPONSE);

        List<String> senders = store.distinctNonEventSendersInChannel(channelId);

        assertTrue(senders.contains("alice"));
        assertTrue(senders.contains("carol"));
        assertFalse(senders.contains("bob"));
    }

    @Test
    void distinctNonEventSendersInChannel_deduplicatesSameSender() {
        UUID channelId = UUID.randomUUID();
        putMessage(channelId, "alice", MessageType.REQUEST);
        putMessage(channelId, "alice", MessageType.RESPONSE);

        List<String> senders = store.distinctNonEventSendersInChannel(channelId);

        assertEquals(1, senders.size());
        assertEquals("alice", senders.get(0));
    }

    @Test
    void distinctNonEventSendersInChannel_emptyChannel_returnsEmpty() {
        List<String> senders = store.distinctNonEventSendersInChannel(UUID.randomUUID());
        assertTrue(senders.isEmpty());
    }

    @Test
    void distinctNonEventSendersInChannel_onlyEventMessages_returnsEmpty() {
        UUID channelId = UUID.randomUUID();
        putMessage(channelId, "alice", MessageType.EVENT);
        putMessage(channelId, "bob", MessageType.EVENT);

        List<String> senders = store.distinctNonEventSendersInChannel(channelId);
        assertTrue(senders.isEmpty());
    }

    private Message putMessage(UUID channelId, String sender, MessageType type) {
        Message m = new Message();
        m.channelId = channelId;
        m.sender = sender;
        m.messageType = type;
        m.content = "test";
        return store.put(m);
    }
}
```

- [ ] **Step 2: Run tests — expect compilation failure (methods don't exist)**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test -pl runtime \
  -Dtest=InMemoryMessageStoreAggregateTest 2>&1 | tail -20
```

Expected: compilation error — `countByChannels` and `distinctNonEventSendersInChannel` not found.

- [ ] **Step 3: Add methods to MessageStore interface**

In `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/MessageStore.java`, add after `countByChannel`:

```java
import java.util.Collection;
import java.util.Map;

// existing methods...

/**
 * Returns message counts grouped by channel for the given channel IDs.
 * Channels with no messages are absent from the result — use getOrDefault(id, 0L).
 */
Map<UUID, Long> countByChannels(Collection<UUID> channelIds);

/**
 * Returns distinct sender values for all non-EVENT messages in the given channel.
 * Used by barrier channels to determine which contributors have written.
 */
List<String> distinctNonEventSendersInChannel(UUID channelId);
```

- [ ] **Step 4: Implement in JpaMessageStore**

In `JpaMessageStore.java`, add after `countByChannel`:

```java
@Override
public Map<UUID, Long> countByChannels(Collection<UUID> channelIds) {
    if (channelIds.isEmpty()) {
        return Map.of();
    }
    @SuppressWarnings("unchecked")
    List<Object[]> rows = Message.getEntityManager()
            .createQuery("SELECT m.channelId, COUNT(m) FROM Message m WHERE m.channelId IN :ids GROUP BY m.channelId")
            .setParameter("ids", channelIds)
            .getResultList();
    return rows.stream()
            .collect(java.util.stream.Collectors.toMap(r -> (UUID) r[0], r -> (Long) r[1]));
}

@Override
public List<String> distinctNonEventSendersInChannel(UUID channelId) {
    @SuppressWarnings("unchecked")
    List<String> result = Message.getEntityManager()
            .createQuery("SELECT DISTINCT m.sender FROM Message m WHERE m.channelId = ?1 AND m.messageType != ?2")
            .setParameter(1, channelId)
            .setParameter(2, io.quarkiverse.qhorus.runtime.message.MessageType.EVENT)
            .getResultList();
    return result;
}
```

- [ ] **Step 5: Implement in InMemoryMessageStore**

In `InMemoryMessageStore.java`, add after `countByChannel`:

```java
@Override
public Map<UUID, Long> countByChannels(Collection<UUID> channelIds) {
    if (channelIds.isEmpty()) {
        return Map.of();
    }
    return channelIds.stream()
            .filter(id -> store.values().stream().anyMatch(m -> id.equals(m.channelId)))
            .collect(java.util.stream.Collectors.toMap(
                    id -> id,
                    id -> store.values().stream().filter(m -> id.equals(m.channelId)).count()));
}

@Override
public List<String> distinctNonEventSendersInChannel(UUID channelId) {
    return store.values().stream()
            .filter(m -> channelId.equals(m.channelId) && m.messageType != MessageType.EVENT)
            .map(m -> m.sender)
            .distinct()
            .toList();
}
```

- [ ] **Step 6: Run tests — expect PASS**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test -pl runtime \
  -Dtest=InMemoryMessageStoreAggregateTest 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7: Run full Qhorus test suite — all tests still pass**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 8: Commit**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
git add -A
git commit -m "$(cat <<'EOF'
feat: add countByChannels and distinctNonEventSendersInChannel to MessageStore

Two aggregate methods needed to close Message.getEntityManager() bypass
paths in the MCP tools. Both blocking SPI interface and JPA + InMemory
implementations. Refs #ISSUE_N
EOF
)"
```

---

### Task 3: Close MCP tool EntityManager bypasses (TDD)

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Test: verify via existing tool tests + integration tests

- [ ] **Step 1: Confirm MessageStore is already injected in QhorusMcpTools**

Read `QhorusMcpTools.java` lines 1–80 and locate the `@Inject MessageStore messageStore` field. If absent, it will need to be added.

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
grep -n "MessageStore\|messageStore" runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java | head -10
```

If `MessageStore messageStore` is not present, add `@Inject MessageStore messageStore;` alongside the other `@Inject` fields. Check ReactiveQhorusMcpTools similarly for `ReactiveMessageStore reactiveMessageStore` — but since the bypass uses blocking `Message.getEntityManager()`, inject `MessageStore messageStore` in the reactive tool too.

- [ ] **Step 2: Replace bypass 1 in QhorusMcpTools.listChannels() (lines 286–294)**

Replace:

```java
// Batch all message counts in one GROUP BY query — avoids N+1
@SuppressWarnings("unchecked")
List<Object[]> countRows = Message.getEntityManager()
        .createQuery("SELECT m.channelId, COUNT(m) FROM Message m GROUP BY m.channelId")
        .getResultList();
Map<UUID, Long> countByChannel = countRows.stream()
        .collect(Collectors.toMap(r -> (UUID) r[0], r -> (Long) r[1]));
return channels.stream()
        .map(ch -> toChannelDetail(ch, countByChannel.getOrDefault(ch.id, 0L)))
        .toList();
```

With:

```java
Map<UUID, Long> countByChannel = messageStore.countByChannels(
        channels.stream().map(ch -> ch.id).toList());
return channels.stream()
        .map(ch -> toChannelDetail(ch, countByChannel.getOrDefault(ch.id, 0L)))
        .toList();
```

- [ ] **Step 3: Replace bypass 2 in QhorusMcpTools.checkMessagesBarrier() (lines 642–648)**

Replace:

```java
@SuppressWarnings("unchecked")
List<String> written = Message.getEntityManager()
        .createQuery("SELECT DISTINCT m.sender FROM Message m "
                + "WHERE m.channelId = ?1 AND m.messageType != ?2")
        .setParameter(1, ch.id)
        .setParameter(2, MessageType.EVENT)
        .getResultList();
```

With:

```java
List<String> written = messageStore.distinctNonEventSendersInChannel(ch.id);
```

- [ ] **Step 4: Replace bypass in ReactiveQhorusMcpTools.checkMessagesBarrier() (line 752)**

Replace:

```java
@SuppressWarnings("unchecked")
List<String> written = Message.getEntityManager()
        .createQuery("SELECT DISTINCT m.sender FROM Message m "
                + "WHERE m.channelId = ?1 AND m.messageType != ?2")
        .setParameter(1, ch.id)
        .setParameter(2, MessageType.EVENT)
        .getResultList();
```

With:

```java
List<String> written = messageStore.distinctNonEventSendersInChannel(ch.id);
```

(The reactive tool can use the blocking `MessageStore` for this synchronous query — it is not a hot path.)

- [ ] **Step 5: Verify no remaining Message.getEntityManager() bypasses outside store layer**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
grep -rn "getEntityManager" runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/ \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/instance/
```

Expected: zero results outside `store/jpa/` and `ledger/` directories.

- [ ] **Step 6: Run full test suite**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
git add -A
git commit -m "$(cat <<'EOF'
refactor: close Message.getEntityManager() bypasses in MCP tools

listChannels() and checkMessagesBarrier() (blocking + reactive) now route
through MessageStore.countByChannels() and distinctNonEventSendersInChannel().
No EntityManager access outside the store/jpa layer. Refs #ISSUE_N
EOF
)"
```

---

### Task 4: PendingReplyStore SPI — interface + JPA + InMemory (TDD)

**Files:**
- **Create:** `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/PendingReplyStore.java`
- **Create:** `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaPendingReplyStore.java`
- **Create:** `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryPendingReplyStore.java`
- **Create:** `runtime/src/test/java/io/quarkiverse/qhorus/runtime/store/InMemoryPendingReplyStoreTest.java`

- [ ] **Step 1: Write the failing tests**

Create `runtime/src/test/java/io/quarkiverse/qhorus/runtime/store/InMemoryPendingReplyStoreTest.java`:

```java
package io.quarkiverse.qhorus.runtime.store;

import static org.junit.jupiter.api.Assertions.*;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.message.PendingReply;
import io.quarkiverse.qhorus.testing.InMemoryPendingReplyStore;

class InMemoryPendingReplyStoreTest {

    private InMemoryPendingReplyStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryPendingReplyStore();
        store.clear();
    }

    // --- save + findByCorrelationId ---

    @Test
    void saveAndFind_happyPath() {
        PendingReply pr = pendingReply("corr-1", Instant.now().plusSeconds(60));
        store.save(pr);

        Optional<PendingReply> found = store.findByCorrelationId("corr-1");
        assertTrue(found.isPresent());
        assertEquals("corr-1", found.get().correlationId);
    }

    @Test
    void save_assignsIdIfAbsent() {
        PendingReply pr = pendingReply("corr-2", Instant.now().plusSeconds(60));
        assertNull(pr.id);
        store.save(pr);
        assertNotNull(pr.id);
    }

    @Test
    void findByCorrelationId_notFound_returnsEmpty() {
        Optional<PendingReply> result = store.findByCorrelationId("nonexistent");
        assertTrue(result.isEmpty());
    }

    @Test
    void save_updatesExistingEntry() {
        PendingReply pr = pendingReply("corr-3", Instant.now().plusSeconds(60));
        store.save(pr);

        Instant newExpiry = Instant.now().plusSeconds(120);
        pr.expiresAt = newExpiry;
        store.save(pr);

        Optional<PendingReply> found = store.findByCorrelationId("corr-3");
        assertTrue(found.isPresent());
        assertEquals(newExpiry, found.get().expiresAt);
    }

    // --- deleteByCorrelationId ---

    @Test
    void deleteByCorrelationId_removesEntry() {
        store.save(pendingReply("corr-4", Instant.now().plusSeconds(60)));
        store.deleteByCorrelationId("corr-4");
        assertTrue(store.findByCorrelationId("corr-4").isEmpty());
    }

    @Test
    void deleteByCorrelationId_nonexistent_noError() {
        assertDoesNotThrow(() -> store.deleteByCorrelationId("ghost"));
    }

    // --- existsByCorrelationId ---

    @Test
    void existsByCorrelationId_trueWhenPresent() {
        store.save(pendingReply("corr-5", Instant.now().plusSeconds(60)));
        assertTrue(store.existsByCorrelationId("corr-5"));
    }

    @Test
    void existsByCorrelationId_falseWhenAbsent() {
        assertFalse(store.existsByCorrelationId("missing"));
    }

    // --- findExpiredBefore ---

    @Test
    void findExpiredBefore_returnsOnlyExpired() {
        Instant now = Instant.now();
        store.save(pendingReply("expired-1", now.minusSeconds(1)));
        store.save(pendingReply("expired-2", now.minusSeconds(10)));
        store.save(pendingReply("active-1", now.plusSeconds(60)));

        List<PendingReply> expired = store.findExpiredBefore(now);
        assertEquals(2, expired.size());
        assertTrue(expired.stream().allMatch(pr -> pr.expiresAt.isBefore(now)));
    }

    @Test
    void findExpiredBefore_noneExpired_returnsEmpty() {
        store.save(pendingReply("active", Instant.now().plusSeconds(60)));
        List<PendingReply> expired = store.findExpiredBefore(Instant.now());
        assertTrue(expired.isEmpty());
    }

    // --- deleteExpiredBefore ---

    @Test
    void deleteExpiredBefore_removesExpiredLeavesActive() {
        Instant now = Instant.now();
        store.save(pendingReply("expired", now.minusSeconds(5)));
        store.save(pendingReply("active", now.plusSeconds(60)));

        store.deleteExpiredBefore(now);

        assertFalse(store.existsByCorrelationId("expired"));
        assertTrue(store.existsByCorrelationId("active"));
    }

    private PendingReply pendingReply(String correlationId, Instant expiresAt) {
        PendingReply pr = new PendingReply();
        pr.correlationId = correlationId;
        pr.channelId = UUID.randomUUID();
        pr.instanceId = UUID.randomUUID();
        pr.expiresAt = expiresAt;
        return pr;
    }
}
```

- [ ] **Step 2: Run tests — expect compilation failure**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test -pl runtime \
  -Dtest=InMemoryPendingReplyStoreTest 2>&1 | tail -10
```

Expected: compilation error — `InMemoryPendingReplyStore` not found.

- [ ] **Step 3: Create PendingReplyStore interface**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/PendingReplyStore.java`:

```java
package io.quarkiverse.qhorus.runtime.store;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import io.quarkiverse.qhorus.runtime.message.PendingReply;

public interface PendingReplyStore {

    /** Persist a new PendingReply or update an existing one (matched by correlationId). */
    PendingReply save(PendingReply pr);

    Optional<PendingReply> findByCorrelationId(String correlationId);

    void deleteByCorrelationId(String correlationId);

    boolean existsByCorrelationId(String correlationId);

    /** All entries whose expiresAt is before the given cutoff. */
    List<PendingReply> findExpiredBefore(Instant cutoff);

    /** Delete all entries whose expiresAt is before the given cutoff. */
    void deleteExpiredBefore(Instant cutoff);
}
```

- [ ] **Step 4: Create InMemoryPendingReplyStore**

Create `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryPendingReplyStore.java`:

```java
package io.quarkiverse.qhorus.testing;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkiverse.qhorus.runtime.message.PendingReply;
import io.quarkiverse.qhorus.runtime.store.PendingReplyStore;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryPendingReplyStore implements PendingReplyStore {

    private final Map<String, PendingReply> store = new ConcurrentHashMap<>();

    @Override
    public PendingReply save(PendingReply pr) {
        if (pr.id == null) {
            pr.id = UUID.randomUUID();
        }
        store.put(pr.correlationId, pr);
        return pr;
    }

    @Override
    public Optional<PendingReply> findByCorrelationId(String correlationId) {
        return Optional.ofNullable(store.get(correlationId));
    }

    @Override
    public void deleteByCorrelationId(String correlationId) {
        store.remove(correlationId);
    }

    @Override
    public boolean existsByCorrelationId(String correlationId) {
        return store.containsKey(correlationId);
    }

    @Override
    public List<PendingReply> findExpiredBefore(Instant cutoff) {
        return store.values().stream()
                .filter(pr -> pr.expiresAt != null && pr.expiresAt.isBefore(cutoff))
                .toList();
    }

    @Override
    public void deleteExpiredBefore(Instant cutoff) {
        store.values().removeIf(pr -> pr.expiresAt != null && pr.expiresAt.isBefore(cutoff));
    }

    /** Call in @BeforeEach / @AfterEach for test isolation. */
    public void clear() {
        store.clear();
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test -pl runtime,testing \
  -Dtest=InMemoryPendingReplyStoreTest 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Create JpaPendingReplyStore**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaPendingReplyStore.java`:

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.quarkiverse.qhorus.runtime.message.PendingReply;
import io.quarkiverse.qhorus.runtime.store.PendingReplyStore;

@ApplicationScoped
public class JpaPendingReplyStore implements PendingReplyStore {

    @Override
    @Transactional
    public PendingReply save(PendingReply pr) {
        if (pr.id == null) {
            pr.persist();
        } else {
            PendingReply.getEntityManager().merge(pr);
        }
        return pr;
    }

    @Override
    public Optional<PendingReply> findByCorrelationId(String correlationId) {
        return PendingReply.<PendingReply> find("correlationId", correlationId).firstResultOptional();
    }

    @Override
    @Transactional
    public void deleteByCorrelationId(String correlationId) {
        PendingReply.delete("correlationId", correlationId);
    }

    @Override
    public boolean existsByCorrelationId(String correlationId) {
        return PendingReply.count("correlationId", correlationId) > 0;
    }

    @Override
    public List<PendingReply> findExpiredBefore(Instant cutoff) {
        return PendingReply.<PendingReply> find("expiresAt < ?1", cutoff).list();
    }

    @Override
    @Transactional
    public void deleteExpiredBefore(Instant cutoff) {
        PendingReply.delete("expiresAt < ?1", cutoff);
    }
}
```

- [ ] **Step 7: Run full test suite**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 8: Commit**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
git add -A
git commit -m "$(cat <<'EOF'
feat: add PendingReplyStore as sixth store SPI interface

PendingReply previously accessed JPA directly — blocking Redis/MongoDB
drop-in support. New interface + JpaPendingReplyStore + InMemoryPendingReplyStore
(with clear() for test isolation). Refs #ISSUE_N
EOF
)"
```

---

### Task 5: Wire PendingReplyStore into MessageService and PendingReplyCleanupJob

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageService.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/PendingReplyCleanupJob.java`
- Test: `runtime/src/test/java/io/quarkiverse/qhorus/runtime/message/MessageServicePendingReplyTest.java`

- [ ] **Step 1: Write failing tests for MessageService PendingReply operations**

Create `runtime/src/test/java/io/quarkiverse/qhorus/runtime/message/MessageServicePendingReplyTest.java`:

```java
package io.quarkiverse.qhorus.runtime.message;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import io.quarkiverse.qhorus.runtime.store.PendingReplyStore;

class MessageServicePendingReplyTest {

    private PendingReplyStore pendingReplyStore;
    private MessageService service;

    @BeforeEach
    void setUp() {
        pendingReplyStore = mock(PendingReplyStore.class);
        service = new MessageService();
        // inject via field — requires package-private or setter, see Step 3
        service.pendingReplyStore = pendingReplyStore;
    }

    @Test
    void registerPendingReply_newEntry_savesViaStore() {
        String correlationId = "corr-new";
        UUID channelId = UUID.randomUUID();
        UUID instanceId = UUID.randomUUID();
        Instant expiry = Instant.now().plusSeconds(60);

        when(pendingReplyStore.findByCorrelationId(correlationId)).thenReturn(Optional.empty());

        service.registerPendingReply(correlationId, channelId, instanceId, expiry);

        ArgumentCaptor<PendingReply> captor = ArgumentCaptor.forClass(PendingReply.class);
        verify(pendingReplyStore).save(captor.capture());
        PendingReply saved = captor.getValue();
        assertEquals(correlationId, saved.correlationId);
        assertEquals(channelId, saved.channelId);
        assertEquals(instanceId, saved.instanceId);
        assertEquals(expiry, saved.expiresAt);
    }

    @Test
    void registerPendingReply_existingEntry_updatesExpiryViaStore() {
        String correlationId = "corr-existing";
        Instant newExpiry = Instant.now().plusSeconds(120);
        PendingReply existing = new PendingReply();
        existing.id = UUID.randomUUID();
        existing.correlationId = correlationId;
        existing.expiresAt = Instant.now().plusSeconds(30);

        when(pendingReplyStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(existing));

        service.registerPendingReply(correlationId, UUID.randomUUID(), UUID.randomUUID(), newExpiry);

        assertEquals(newExpiry, existing.expiresAt);
        verify(pendingReplyStore).save(existing);
    }

    @Test
    void deletePendingReply_delegatesToStore() {
        service.deletePendingReply("corr-del");
        verify(pendingReplyStore).deleteByCorrelationId("corr-del");
    }

    @Test
    void pendingReplyExists_delegatesToStore() {
        when(pendingReplyStore.existsByCorrelationId("corr-x")).thenReturn(true);
        assertTrue(service.pendingReplyExists("corr-x"));
        when(pendingReplyStore.existsByCorrelationId("corr-y")).thenReturn(false);
        assertFalse(service.pendingReplyExists("corr-y"));
    }
}
```

Note: this test uses direct field injection. If `pendingReplyStore` is currently private in `MessageService`, it needs to be package-private (remove the `private` modifier) for testing, consistent with the existing pattern in the codebase (e.g. `AuthRateLimiter.resetForTest()`).

- [ ] **Step 2: Run tests — expect compilation failure**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test -pl runtime \
  -Dtest=MessageServicePendingReplyTest 2>&1 | tail -10
```

Expected: compilation error — `service.pendingReplyStore` not accessible.

- [ ] **Step 3: Update MessageService to inject and use PendingReplyStore**

In `MessageService.java`:

1. Add `@Inject PendingReplyStore pendingReplyStore;` field (package-private, no `private`):

```java
@Inject
PendingReplyStore pendingReplyStore;
```

2. Replace `registerPendingReply` body:

```java
@Transactional
public void registerPendingReply(String correlationId, UUID channelId, UUID instanceId,
        Instant expiresAt) {
    pendingReplyStore.findByCorrelationId(correlationId).ifPresentOrElse(
            existing -> {
                existing.expiresAt = expiresAt;
                pendingReplyStore.save(existing);
            },
            () -> {
                PendingReply pr = new PendingReply();
                pr.correlationId = correlationId;
                pr.channelId = channelId;
                pr.instanceId = instanceId;
                pr.expiresAt = expiresAt;
                pendingReplyStore.save(pr);
            });
}
```

3. Replace `deletePendingReply` body:

```java
@Transactional
public void deletePendingReply(String correlationId) {
    pendingReplyStore.deleteByCorrelationId(correlationId);
}
```

4. Replace `pendingReplyExists` body:

```java
public boolean pendingReplyExists(String correlationId) {
    return pendingReplyStore.existsByCorrelationId(correlationId);
}
```

5. Remove the direct `PendingReply.<PendingReply>find(...)` import if it was specific to static access.

- [ ] **Step 4: Update PendingReplyCleanupJob to use PendingReplyStore**

Replace `PendingReplyCleanupJob.java` body:

```java
package io.quarkiverse.qhorus.runtime.message;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.qhorus.runtime.store.PendingReplyStore;
import io.quarkus.scheduler.Scheduled;

@ApplicationScoped
public class PendingReplyCleanupJob {

    @Inject
    PendingReplyStore pendingReplyStore;

    @Scheduled(every = "${quarkus.qhorus.cleanup.pending-reply-check-seconds}s")
    @Transactional
    public void cleanupExpired() {
        java.util.List<io.quarkiverse.qhorus.runtime.message.PendingReply> expired =
                pendingReplyStore.findExpiredBefore(java.time.Instant.now());
        if (!expired.isEmpty()) {
            pendingReplyStore.deleteExpiredBefore(java.time.Instant.now());
            io.quarkus.logging.Log.infof("PendingReply cleanup: deleted %d expired entries", expired.size());
        }
    }
}
```

- [ ] **Step 5: Run tests**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test -pl runtime \
  -Dtest=MessageServicePendingReplyTest 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Run full test suite**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
git add -A
git commit -m "$(cat <<'EOF'
refactor: wire PendingReplyStore into MessageService and PendingReplyCleanupJob

All PendingReply JPA access now goes through the store SPI. Direct Panache
static calls removed from service and scheduler. Refs #ISSUE_N
EOF
)"
```

---

### Task 6: Named persistence unit — config and @PersistenceUnit qualifier

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntryRepository.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Add @PersistenceUnit qualifier to AgentMessageLedgerEntryRepository**

In `AgentMessageLedgerEntryRepository.java`, change the `EntityManager` injection from:

```java
@Inject
EntityManager em;
```

To:

```java
@io.quarkus.hibernate.orm.PersistenceUnit("qhorus")
EntityManager em;
```

Remove the `@Inject` annotation — `@PersistenceUnit` is the injection point on its own in Quarkus.

- [ ] **Step 2: Update test application.properties for named persistence unit**

Replace the content of `runtime/src/test/resources/application.properties`:

```properties
quarkus.http.test-port=0

# Named datasource for Qhorus persistence unit
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:test;DB_CLOSE_DELAY=-1

# Named Hibernate persistence unit — includes ledger package for inheritance
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.quarkiverse.qhorus.runtime,io.quarkiverse.ledger.runtime.model
quarkus.hibernate-orm.qhorus.database.generation=drop-and-create

# Disable reactive for named datasource — tests use JDBC only
quarkus.datasource.qhorus.reactive.enabled=false

# quarkus-ledger configuration
quarkus.ledger.enabled=true
quarkus.ledger.hash-chain.enabled=true
quarkus.ledger.decision-context.enabled=false
quarkus.ledger.attestations.enabled=true
quarkus.ledger.trust-score.enabled=false
```

- [ ] **Step 3: Run full test suite**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`. If Hibernate fails to bind the persistence unit, the error will mention "no persistence unit" or "no entity found" — check that `packages` matches the actual entity locations.

- [ ] **Step 4: Commit**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
git add -A
git commit -m "$(cat <<'EOF'
feat: migrate to named persistence unit 'qhorus'

AgentMessageLedgerEntryRepository uses @PersistenceUnit("qhorus").
Test application.properties updated to quarkus.datasource.qhorus.* and
quarkus.hibernate-orm.qhorus.* config. Refs #ISSUE_N
EOF
)"
```

---

### Task 7: Write ADR for ledger inheritance constraint

**Files:**
- **Create:** `adr/0004-ledger-jpa-inheritance-constraint.md`
- Modify: `adr/INDEX.md`

- [ ] **Step 1: Write the ADR**

Create `adr/0004-ledger-jpa-inheritance-constraint.md`:

```markdown
# ADR 0004 — Ledger JPA Inheritance Constraint

**Status:** Accepted  
**Date:** 2026-04-22

## Context

`AgentMessageLedgerEntry` extends `LedgerEntry` using `InheritanceType.JOINED`.
JPA requires all entities in an inheritance hierarchy to share a single persistence unit.
`LedgerEntry` lives in `io.quarkiverse.ledger.runtime.model` (quarkus-ledger);
`AgentMessageLedgerEntry` is in `io.quarkiverse.qhorus.runtime.ledger` (this project).

When Qhorus migrated to the named persistence unit `qhorus`, both packages had to be
included in `quarkus.hibernate-orm.qhorus.packages` — binding quarkus-ledger entities
to the `qhorus` datasource as a side-effect.

## Decision

Accept this coupling (Option A). Both packages included in the `qhorus` PU. The
quarkus-ledger schema is managed by Qhorus's Flyway migrations.

## Consequences

- quarkus-ledger entities are tied to the `qhorus` datasource; they cannot independently
  use a different datasource without a code change here.
- When quarkus-ledger defines its own named persistence unit, the `InheritanceType.JOINED`
  relationship between `LedgerEntry` and `AgentMessageLedgerEntry` must be reconsidered.
  Options at that point: (a) a shared persistence unit across both libraries, (b) flatten
  the inheritance to a plain FK column with no Hibernate mapping, (c) keep them on the same
  physical datasource with separate PU configs pointing at the same schema.

## Revisit trigger

quarkus-ledger defines its own named persistence unit `ledger`.
```

- [ ] **Step 2: Add to ADR index**

In `adr/INDEX.md`, add:

```markdown
| 0004 | Ledger JPA Inheritance Constraint | Accepted |
```

- [ ] **Step 3: Commit**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
git add adr/
git commit -m "$(cat <<'EOF'
docs: ADR 0004 — ledger JPA inheritance constraint

Records why quarkus-ledger entities are included in the qhorus persistence
unit and the conditions under which this coupling should be revisited.
Refs #ISSUE_N
EOF
)"
```

---

### Task 8: Build and install Qhorus artifact

- [ ] **Step 1: Run full test suite one final time**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn test 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`. Do not proceed if any test fails.

- [ ] **Step 2: Install artifact to local Maven repository**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus && mvn install -DskipTests 2>&1 | tail -10
```

Expected: `BUILD SUCCESS` with `quarkus-qhorus-1.0.0-SNAPSHOT.jar` installed.

- [ ] **Step 3: Close Qhorus issue**

```bash
cd /Users/mdproctor/claude/quarkus-qhorus
gh issue close ISSUE_N --comment "All changes landed: PendingReplyStore SPI, MessageStore aggregates, bypass closure, named PU, ADR 0004."
```

---

## Phase 2 — Claudony

---

### Task 9: Create GitHub epic and issue in Claudony

**Files:** none

- [ ] **Step 1: Create the epic**

```bash
cd /Users/mdproctor/claude/claudony
gh issue create \
  --title "Epic: Ecosystem persistence isolation — Claudony wiring" \
  --body "Wire Claudony to the updated Qhorus named persistence unit. Rename datasource config, add quarkus-qhorus-testing, clean up test DB dependencies. Prerequisite for multi-node fleet." \
  --label "epic"
```

Note the epic issue number — call it `CL_EPIC_N`.

- [ ] **Step 2: Create the implementation issue**

```bash
gh issue create \
  --title "Named datasource config: wire quarkus-qhorus-testing, rename quarkus.datasource.qhorus.*" \
  --body "Refs #CL_EPIC_N

- Add quarkus-qhorus-testing test dependency to pom.xml
- Rename quarkus.datasource.* → quarkus.datasource.qhorus.* in application.properties
- Remove database.generation=update, add Flyway config
- Remove old %test datasource config, verify reactive guard
- Update MeshResourceInterjectionTest cleanup (UserTransaction → InMemory store clear)" \
  --label "enhancement"
```

Note the issue number — call it `CL_ISSUE_N`.

---

### Task 10: Add quarkus-qhorus-testing dependency

**Files:**
- Modify: `pom.xml`

- [ ] **Step 1: Locate the existing Qhorus dependency in pom.xml**

```bash
grep -n "qhorus" /Users/mdproctor/claude/claudony/pom.xml
```

Note the line numbers and existing version.

- [ ] **Step 2: Add quarkus-qhorus-testing in test scope**

After the existing `quarkus-qhorus` dependency block, add:

```xml
<dependency>
    <groupId>io.quarkiverse.qhorus</groupId>
    <artifactId>quarkus-qhorus-testing</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 3: Verify pom.xml compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn validate 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add pom.xml
git commit -m "$(cat <<'EOF'
build: add quarkus-qhorus-testing test dependency

Activates InMemory*Store alternatives in @QuarkusTest runs — Qhorus data
no longer needs a real datasource in unit tests. Refs #CL_ISSUE_N
EOF
)"
```

---

### Task 11: Rename datasource config in application.properties

**Files:**
- Modify: `src/main/resources/application.properties`

- [ ] **Step 1: Read the current datasource config block**

```bash
grep -n "datasource\|hibernate-orm\|flyway" \
  /Users/mdproctor/claude/claudony/src/main/resources/application.properties
```

Note all lines that need updating.

- [ ] **Step 2: Replace the datasource config block**

Find and replace the current Qhorus-related config. The before block (exact lines may vary — verify first):

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:file:~/.claudony/qhorus;DB_CLOSE_DELAY=-1;AUTO_SERVER=TRUE
quarkus.hibernate-orm.database.generation=update
```

Replace with:

```properties
# ── Qhorus named persistence unit ────────────────────────────────────────────
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:~/.claudony/qhorus;DB_CLOSE_ON_EXIT=FALSE;AUTO_SERVER=TRUE
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.quarkiverse.qhorus.runtime,io.quarkiverse.ledger.runtime.model
quarkus.flyway.qhorus.migrate-at-start=true
```

- [ ] **Step 3: Update the test config block**

Find and replace the current `%test` datasource lines. The before block:

```properties
%test.quarkus.datasource.jdbc.url=jdbc:h2:mem:claudony-test;DB_CLOSE_ON_EXIT=FALSE
%test.quarkus.hibernate-orm.database.generation=drop-and-create
%test.quarkus.datasource.reactive=false
```

Replace with (TBD guard — verify in Step 5 whether the reactive line is still needed):

```properties
# Qhorus test datasource — only needed if Hibernate ORM extension still initialises
# despite InMemory*Store alternatives. Verify and remove if tests pass without it.
%test.quarkus.datasource.qhorus.reactive.enabled=false
```

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

- [ ] **Step 5: Resolve reactive guard**

If tests fail with a Hibernate Reactive / no-reactive-datasource error, add the minimal guard that silences it. The error message will indicate which exact property is needed. Common pattern:

```properties
%test.quarkus.datasource.qhorus.reactive.enabled=false
```

If tests pass without any `%test` datasource config, remove the guard entirely — InMemory*Store is fully bypassing Hibernate.

Run tests again after any adjustment until `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add src/main/resources/application.properties
git commit -m "$(cat <<'EOF'
feat: rename datasource config to quarkus.datasource.qhorus.*

Switches from default datasource to named persistence unit 'qhorus'.
Drops database.generation=update — Flyway manages schema. Refs #CL_ISSUE_N
EOF
)"
```

---

### Task 12: Update MeshResourceInterjectionTest cleanup

**Files:**
- Modify: `src/test/java/dev/claudony/.../MeshResourceInterjectionTest.java`

- [ ] **Step 1: Read the current test file**

```bash
find /Users/mdproctor/claude/claudony/src/test -name "MeshResourceInterjectionTest.java"
```

Read the file and locate all `UserTransaction ut` usage and `@AfterEach` cleanup blocks.

- [ ] **Step 2: Inject InMemory*Store beans for cleanup**

Remove the `@Inject UserTransaction ut` field and Panache delete calls in `@AfterEach`. Replace with injections of the InMemory stores used in this test, and call `clear()` on each in `@AfterEach`:

```java
@Inject
io.quarkiverse.qhorus.testing.InMemoryChannelStore channelStore;

@Inject
io.quarkiverse.qhorus.testing.InMemoryMessageStore messageStore;

// Add any other InMemory*Store beans whose state this test creates

@AfterEach
void cleanUp() {
    channelStore.clear();
    messageStore.clear();
    // clear any additional stores used in this test
}
```

Determine which stores need clearing by reading the test body — inject only the stores that the test creates data in.

- [ ] **Step 3: Run only MeshResourceInterjectionTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest=MeshResourceInterjectionTest 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -20
```

Expected: `BUILD SUCCESS` — all 273+ tests passing.

- [ ] **Step 5: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add src/test/
git commit -m "$(cat <<'EOF'
refactor: replace UserTransaction cleanup with InMemory store clear()

MeshResourceInterjectionTest now clears InMemory*Store beans in @AfterEach
instead of using UserTransaction + Panache deletes. Refs #CL_ISSUE_N
EOF
)"
```

---

### Task 13: Update DESIGN.md and CLAUDE.md

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update DESIGN.md persistence section**

Open `docs/DESIGN.md` and find the persistence / datasource section. Update to reflect:
- Named datasource `qhorus` (not default datasource)
- Flyway manages schema (not `database.generation`)
- `quarkus-qhorus-testing` in test scope — no DB needed for tests
- PendingReplyStore as the sixth store interface in the Qhorus SPI

- [ ] **Step 2: Update CLAUDE.md test count**

Find the "Test Count and Status" section in `CLAUDE.md`. Update the test count if it has changed from 273, and add a note about the `MeshResourceInterjectionTest` cleanup pattern change (InMemory store `clear()` instead of `UserTransaction`).

- [ ] **Step 3: Run full test suite one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Close issue and commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/DESIGN.md CLAUDE.md
git commit -m "$(cat <<'EOF'
docs: update DESIGN.md and CLAUDE.md for named persistence unit

Reflects quarkus.datasource.qhorus.* config, Flyway schema management,
PendingReplyStore as sixth SPI interface, and updated test cleanup pattern.
Closes #CL_ISSUE_N
EOF
)"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Named datasource `qhorus` in Qhorus | Task 6 |
| Named datasource `qhorus` in Claudony | Task 11 |
| PendingReplyStore as sixth SPI | Task 4 + 5 |
| MessageStore aggregate methods (countByChannels, distinctNonEventSendersInChannel) | Task 2 |
| Close MCP tool EntityManager bypasses (QhorusMcpTools + ReactiveQhorusMcpTools) | Task 3 |
| Wire PendingReplyStore into MessageService + PendingReplyCleanupJob | Task 5 |
| @PersistenceUnit("qhorus") on AgentMessageLedgerEntryRepository | Task 6 |
| ADR for ledger inheritance constraint | Task 7 |
| quarkus-qhorus-testing in Claudony | Task 10 |
| MeshResourceInterjectionTest cleanup | Task 12 |
| DESIGN.md + CLAUDE.md updated | Task 13 |
| CaseHub naming convention | Documented in spec; no code change needed now ✓ |
| Flyway version range table | Documented in spec; no code change needed now ✓ |
| GitHub issues and epics on all commits | Tasks 1 and 9 ✓ |

All spec requirements covered.
