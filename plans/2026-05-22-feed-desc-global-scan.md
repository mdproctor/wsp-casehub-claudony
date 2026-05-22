# Feed DESC Global Scan Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the per-channel bucketed `getFeed()` with a single global `ORDER BY id DESC LIMIT N` scan so new messages always appear in the feed.

**Architecture:** Five Qhorus changes (SQL LIMIT fix, MessageQuery descending flag + factory method, getFeed rewrite, V11 index migration) plus two Claudony test updates. Two repos: `~/claude/casehub/qhorus` (Qhorus) and `~/claude/casehub/claudony` (Claudony). Qhorus changes must be built and installed (`mvn install -DskipTests`) before Claudony tests can verify them.

**Tech Stack:** Java 21/26, Quarkus 3.32.2, Hibernate ORM Panache, H2 (test), Mutiny Uni, JUnit 5, Mockito, RestAssured

---

### Task 1: MessageQuery — add `descending` flag and `recent()` factory

**Repo:** `~/claude/casehub/qhorus`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/query/MessageQuery.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/store/query/MessageQueryTest.java`

- [ ] **Step 1: Write the failing test — `recent` factory creates a descending query with no channel**

Add to `MessageQueryTest.java`:

```java
@Test
void recent_createsDescendingQueryWithNoChannel() {
    MessageQuery q = MessageQuery.recent(50);
    assertNull(q.channelId());
    assertEquals(50, q.limit());
    assertTrue(q.descending());
    assertNull(q.afterId());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageQueryTest#recent_createsDescendingQueryWithNoChannel -pl runtime -f ~/claude/casehub/qhorus/pom.xml`

Expected: FAIL — `recent` method does not exist, `descending()` accessor does not exist.

- [ ] **Step 3: Add `descending` field, accessor, builder method, and `recent()` factory to MessageQuery**

In `MessageQuery.java`:

1. Add field: `private final boolean descending;`
2. In constructor: `this.descending = b.descending;`
3. Add accessor: `public boolean descending() { return descending; }`
4. In `Builder`: add `private boolean descending;` and `public Builder descending(boolean v) { this.descending = v; return this; }`
5. In `toBuilder()`: add `.descending(descending)` to the chain.
6. Add factory:

```java
public static MessageQuery recent(int limit) {
    return new Builder().limit(limit).descending(true).build();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageQueryTest -pl runtime -f ~/claude/casehub/qhorus/pom.xml`

Expected: ALL MessageQueryTest tests PASS (existing + new).

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/store/query/MessageQuery.java runtime/src/test/java/io/casehub/qhorus/store/query/MessageQueryTest.java
git -C ~/claude/casehub/qhorus commit -m "feat(store): add descending flag and recent() factory to MessageQuery"
```

---

### Task 2: InMemoryMessageStore — support descending scan

**Repo:** `~/claude/casehub/qhorus`

**Files:**
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java`
- Modify: `testing/src/test/java/io/casehub/qhorus/testing/InMemoryMessageStoreTest.java`

- [ ] **Step 1: Write the failing test — descending scan returns newest first**

Add to `InMemoryMessageStoreTest.java`:

```java
@Test
void scan_descending_returnsNewestFirst() {
    UUID channelId = UUID.randomUUID();
    Message m1 = store.put(msg(channelId, "a", MessageType.COMMAND));
    Message m2 = store.put(msg(channelId, "b", MessageType.COMMAND));
    Message m3 = store.put(msg(channelId, "c", MessageType.COMMAND));

    List<Message> results = store.scan(
            MessageQuery.builder().channelId(channelId).descending(true).limit(2).build());
    assertEquals(2, results.size());
    assertEquals(m3.id, results.get(0).id);
    assertEquals(m2.id, results.get(1).id);
}
```

- [ ] **Step 2: Write the failing test — descending global scan (no channelId) returns newest across all channels**

Add to `InMemoryMessageStoreTest.java`:

```java
@Test
void scan_recent_globalDescending_returnsNewestAcrossChannels() {
    UUID ch1 = UUID.randomUUID();
    UUID ch2 = UUID.randomUUID();
    store.put(msg(ch1, "a", MessageType.COMMAND));  // id 1
    store.put(msg(ch2, "b", MessageType.COMMAND));  // id 2
    store.put(msg(ch1, "c", MessageType.COMMAND));  // id 3

    List<Message> results = store.scan(MessageQuery.recent(2));
    assertEquals(2, results.size());
    // newest first: id 3, id 2
    assertTrue(results.get(0).id > results.get(1).id);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryMessageStoreTest -pl testing -f ~/claude/casehub/qhorus/pom.xml`

Expected: The two new tests FAIL — descending flag is ignored in `InMemoryMessageStore.scan()`.

- [ ] **Step 4: Implement descending support in InMemoryMessageStore.scan()**

In `InMemoryMessageStore.java`, replace the `scan` method body:

```java
@Override
public List<Message> scan(MessageQuery query) {
    Stream<Message> stream = store.values().stream()
            .filter(query::matches);

    if (query.descending()) {
        List<Message> all = stream.collect(Collectors.toCollection(ArrayList::new));
        java.util.Collections.reverse(all);
        stream = all.stream();
    }

    List<Message> results = stream.toList();

    if (query.limit() != null && results.size() > query.limit()) {
        return results.subList(0, query.limit());
    }
    return results;
}
```

Add imports: `java.util.ArrayList`, `java.util.stream.Collectors`.

- [ ] **Step 5: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryMessageStoreTest -pl testing -f ~/claude/casehub/qhorus/pom.xml`

Expected: ALL InMemoryMessageStoreTest tests PASS (existing + 2 new).

Also run the contract tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -f ~/claude/casehub/qhorus/pom.xml`

Expected: ALL testing module tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/casehub/qhorus add testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java testing/src/test/java/io/casehub/qhorus/testing/InMemoryMessageStoreTest.java
git -C ~/claude/casehub/qhorus commit -m "feat(testing): support descending scan in InMemoryMessageStore"
```

---

### Task 3: JpaMessageStore — apply SQL-level LIMIT and support descending

**Repo:** `~/claude/casehub/qhorus`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/store/JpaMessageStoreTest.java`

- [ ] **Step 1: Read `JpaMessageStoreTest.java` to understand existing test patterns**

```bash
cat ~/claude/casehub/qhorus/runtime/src/test/java/io/casehub/qhorus/store/JpaMessageStoreTest.java
```

- [ ] **Step 2: Write the failing test — descending scan returns newest first (JPA)**

Add to `JpaMessageStoreTest.java` (follow its existing style — it's a `@QuarkusTest` with real H2):

```java
@Test
void scan_descending_returnsNewestFirst() {
    // Insert 3 messages, verify descending scan returns newest first
    // (This test will initially fail because JpaMessageStore ignores the descending flag)
}
```

Note: read the existing test first (Step 1) to match its exact fixture and assertion style before writing this test. The test should insert 3 messages to the same channel, scan with `descending(true).limit(2)`, and assert the result has 2 messages in newest-first order.

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JpaMessageStoreTest#scan_descending_returnsNewestFirst -pl runtime -f ~/claude/casehub/qhorus/pom.xml`

Expected: FAIL — JPA store always uses `ORDER BY id ASC`.

- [ ] **Step 4: Implement SQL LIMIT and descending support in JpaMessageStore.scan()**

In `JpaMessageStore.java`, replace the `scan` method:

```java
@Override
public List<Message> scan(MessageQuery q) {
    StringBuilder jpql = new StringBuilder("FROM Message WHERE 1=1");
    List<Object> params = new ArrayList<>();
    int idx = 1;

    if (q.channelId() != null) {
        jpql.append(" AND channelId = ?").append(idx++);
        params.add(q.channelId());
    }
    if (q.afterId() != null) {
        jpql.append(" AND id > ?").append(idx++);
        params.add(q.afterId());
    }
    if (q.sender() != null) {
        jpql.append(" AND sender = ?").append(idx++);
        params.add(q.sender());
    }
    if (q.target() != null) {
        jpql.append(" AND target = ?").append(idx++);
        params.add(q.target());
    }
    if (q.inReplyTo() != null) {
        jpql.append(" AND inReplyTo = ?").append(idx++);
        params.add(q.inReplyTo());
    }
    if (q.excludeTypes() != null && !q.excludeTypes().isEmpty()) {
        jpql.append(" AND messageType NOT IN ?").append(idx++);
        params.add(q.excludeTypes());
    }
    if (q.contentPattern() != null) {
        jpql.append(" AND LOWER(content) LIKE ?").append(idx++);
        params.add("%" + q.contentPattern().toLowerCase() + "%");
    }

    jpql.append(q.descending() ? " ORDER BY id DESC" : " ORDER BY id ASC");

    if (q.limit() != null) {
        return Message.find(jpql.toString(), params.toArray())
                .page(0, q.limit()).list();
    }
    return Message.list(jpql.toString(), params.toArray());
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JpaMessageStoreTest -pl runtime -f ~/claude/casehub/qhorus/pom.xml`

Expected: ALL JpaMessageStoreTest tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java runtime/src/test/java/io/casehub/qhorus/store/JpaMessageStoreTest.java
git -C ~/claude/casehub/qhorus commit -m "fix(store): apply SQL LIMIT via Panache page() and support descending order in JpaMessageStore"
```

---

### Task 4: QhorusDashboardService.getFeed() — rewrite to global DESC scan

**Repo:** `~/claude/casehub/qhorus`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java`

- [ ] **Step 1: Update the existing `getFeed_withChannels_tagsEachEntryWithChannelName` test mock**

The mock currently uses `when(messageStore.scan(any(MessageQuery.class)))`. After the rewrite, the query passed will be `MessageQuery.recent(...)` (global, no channelId). The `any()` matcher still matches, so the mock will work. However, update the test to also assert that the entry's `channel` field matches the channel name correctly.

No code change needed if the mock already uses `any()`. Verify by reading the test.

- [ ] **Step 2: Write a new test — `getFeed` returns newest messages first across channels**

Add to `QhorusDashboardServiceTest.java`:

```java
@Test
void getFeed_returnsNewestFirstAcrossChannels() {
    Channel ch1 = channel("ch-alpha", ChannelSemantic.APPEND);
    Channel ch2 = channel("ch-beta", ChannelSemantic.APPEND);

    Message older = message(ch1.id, "agent:a", MessageType.STATUS, "old");
    older.id = 1L;
    Message newer = message(ch2.id, "agent:b", MessageType.STATUS, "new");
    newer.id = 2L;

    when(channelService.listAll()).thenReturn(Uni.createFrom().item(List.of(ch1, ch2)));
    when(messageStore.scan(any(MessageQuery.class)))
            .thenReturn(Uni.createFrom().item(List.of(newer, older)));

    List<Map<String, Object>> result = service.getFeed(100)
            .await().atMost(Duration.ofSeconds(1));

    assertEquals(2, result.size());
    assertEquals("ch-beta", result.get(0).get("channel"));
    assertEquals("ch-alpha", result.get(1).get("channel"));
}
```

- [ ] **Step 3: Run test to verify it fails (or passes if mock is order-independent)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusDashboardServiceTest -pl runtime -f ~/claude/casehub/qhorus/pom.xml`

If the test passes even with the old implementation (because the mock returns the list in DESC order already), that's fine — it validates the contract. If it fails, proceed to implement.

- [ ] **Step 4: Rewrite `getFeed()` to use global DESC scan**

Replace the `getFeed` method in `QhorusDashboardService.java`:

```java
public Uni<List<Map<String, Object>>> getFeed(int limit) {
    int effectiveLimit = Math.min(Math.max(limit, 1), 200);
    return channelService.listAll().flatMap(channels -> {
        if (channels.isEmpty()) return Uni.createFrom().item(List.of());
        Map<UUID, String> nameMap = channels.stream()
                .collect(Collectors.toMap(ch -> ch.id, ch -> ch.name));
        return messageStore.scan(MessageQuery.recent(effectiveLimit))
                .map(msgs -> msgs.stream()
                        .map(m -> {
                            Map<String, Object> entry =
                                    new HashMap<>(entityMapper.toTimelineEntry(m));
                            entry.put("channel", nameMap.getOrDefault(
                                    m.channelId, m.channelId.toString()));
                            return entry;
                        })
                        .toList());
    });
}
```

Add import: `java.util.HashMap` (if not already present).

Remove imports that are no longer needed: `java.util.ArrayList`, `java.util.Comparator` (check if used elsewhere in the file first).

- [ ] **Step 5: Run all QhorusDashboardServiceTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusDashboardServiceTest -pl runtime -f ~/claude/casehub/qhorus/pom.xml`

Expected: ALL tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java
git -C ~/claude/casehub/qhorus commit -m "refactor(dashboard): rewrite getFeed to global DESC scan — Refs claudony#124"
```

---

### Task 5: V11 migration — index on `(channel_id, id)`

**Repo:** `~/claude/casehub/qhorus`

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V11__add_message_channel_id_index.sql`

- [ ] **Step 1: Create the migration file**

```sql
-- Composite index for per-channel timeline scans:
-- WHERE channel_id = ? AND id > ? ORDER BY id ASC
CREATE INDEX idx_message_channel_id ON message (channel_id, id);
```

- [ ] **Step 2: Run the full Qhorus test suite to verify the migration applies cleanly**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/qhorus/pom.xml`

Expected: ALL Qhorus tests PASS.

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/casehub/qhorus add runtime/src/main/resources/db/qhorus/migration/V11__add_message_channel_id_index.sql
git -C ~/claude/casehub/qhorus commit -m "perf(schema): add composite index on message(channel_id, id) for timeline scans"
```

---

### Task 6: Install Qhorus jar and verify Claudony tests

**Repos:** `~/claude/casehub/qhorus` (build), `~/claude/casehub/claudony` (test)

- [ ] **Step 1: Build and install Qhorus to local Maven repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -f ~/claude/casehub/qhorus/pom.xml`

Expected: BUILD SUCCESS.

- [ ] **Step 2: Run Claudony full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/claudony/pom.xml`

Expected: BUILD SUCCESS, 507+ tests pass, 0 failures.

If tests fail, diagnose — the `getFeed()` rewrite may have changed the order of messages in existing tests. Proceed to Task 7 to fix test assertions.

---

### Task 7: Claudony test updates

**Repo:** `~/claude/casehub/claudony`

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java`

- [ ] **Step 1: Update stale comment in `meshFeed_limitTruncates`**

Replace the comment at line 154-155:

```java
// Global DESC scan: returns the 3 most recent messages (newest first), limited at store level.
```

- [ ] **Step 2: Write new test — newest messages appear in feed, oldest fall off**

Add to `MeshResourceTest.java`:

```java
@Test
void meshFeed_returnsNewestMessages_oldestFallOff() {
    Channel ch = new Channel();
    ch.name = "feed-newest-" + System.nanoTime();
    ch.semantic = ChannelSemantic.APPEND;
    channelStore.put(ch);

    for (int i = 0; i < 10; i++) {
        Message msg = new Message();
        msg.channelId = ch.id;
        msg.sender = "agent:test";
        msg.messageType = MessageType.STATUS;
        msg.content = "msg-" + i;
        messageStore.put(msg);
    }

    // With limit=3, only the 3 newest messages should appear (msg-7, msg-8, msg-9)
    var body = given().when().get("/api/mesh/feed?limit=3")
        .then()
        .statusCode(200)
        .body("$", hasSize(3))
        .extract().body().asString();

    // Newest first: msg-9 should be first
    org.assertj.core.api.Assertions.assertThat(body).contains("msg-9");
    org.assertj.core.api.Assertions.assertThat(body).doesNotContain("msg-0");
}
```

- [ ] **Step 3: Write new test — feed includes entries from all active channels (no starvation)**

Add to `MeshResourceTest.java`:

```java
@Test
void meshFeed_noChannelStarvation_allChannelsRepresented() {
    Channel busy = new Channel();
    busy.name = "feed-busy-" + System.nanoTime();
    busy.semantic = ChannelSemantic.APPEND;
    channelStore.put(busy);

    Channel quiet = new Channel();
    quiet.name = "feed-quiet-" + System.nanoTime();
    quiet.semantic = ChannelSemantic.APPEND;
    channelStore.put(quiet);

    // Insert 1 message in quiet channel first (lower ID)
    Message quietMsg = new Message();
    quietMsg.channelId = quiet.id;
    quietMsg.sender = "agent:q";
    quietMsg.messageType = MessageType.STATUS;
    quietMsg.content = "quiet-msg";
    messageStore.put(quietMsg);

    // Insert 5 messages in busy channel (higher IDs)
    for (int i = 0; i < 5; i++) {
        Message msg = new Message();
        msg.channelId = busy.id;
        msg.sender = "agent:b";
        msg.messageType = MessageType.STATUS;
        msg.content = "busy-" + i;
        messageStore.put(msg);
    }

    // With limit=100, both channels should appear
    given().when().get("/api/mesh/feed?limit=100")
        .then()
        .statusCode(200)
        .body("channel", hasItems(busy.name, quiet.name));
}
```

- [ ] **Step 4: Run the full Claudony test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f ~/claude/casehub/claudony/pom.xml`

Expected: BUILD SUCCESS, 509+ tests pass (507 baseline + 2 new), 0 failures.

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/claudony add app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java
git -C ~/claude/casehub/claudony commit -m "test(mesh): verify feed DESC ordering and no-channel-starvation — Refs #124"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] JpaMessageStore SQL LIMIT fix → Task 3
- [x] MessageQuery descending flag → Task 1
- [x] MessageQuery `recent()` factory → Task 1
- [x] QhorusDashboardService.getFeed rewrite → Task 4
- [x] V11 index migration → Task 5
- [x] MeshResource no production changes → verified in Task 6 (Claudony tests pass with no code changes)
- [x] MeshResourceTest stale comment + new coverage → Task 7

**Placeholder scan:** No TBD/TODO. Task 3 Step 1 says "read the file first" — this is deliberate because JpaMessageStoreTest patterns may have changed since brainstorming.

**Type consistency:** `MessageQuery.recent(int)`, `MessageQuery.descending()`, `q.descending()` — consistent across all tasks. `messageStore.scan(MessageQuery.recent(effectiveLimit))` matches the factory signature in Task 1.
