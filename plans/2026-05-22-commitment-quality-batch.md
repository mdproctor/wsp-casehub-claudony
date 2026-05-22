# Commitment Wire-up and Quality Batch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire Qhorus Commitment tracking for COMMAND/QUERY messages, fix SSE double-frame bug, apply Java and JS quality fixes, and add missing test coverage across six issues (#122, #123, #128, #130, #132, #133).

**Architecture:** All changes are Claudony-only (no Qhorus or engine changes). #132 is the most impactful — it fixes the global mesh SSE stream that has silently broken since #101. #122 wires correlationId from engine COMMAND content into ReactiveMessageService.send(), enabling the Qhorus obligation state machine. The JS fixes (#128, #133-M5) are self-contained edits to terminal.js.

**Tech Stack:** Java 21 / Quarkus 3.32.2, Mutiny (Uni/Multi), Jackson, JUnit 5 + RestAssured + AssertJ + Mockito (unit), QuarkusTest (integration), Playwright (E2E), vanilla JS.

**Spec:** `docs/superpowers/specs/2026-05-22-commitment-quality-batch-design.md`

**Build commands:**
```bash
# Run a specific test class (fastest feedback loop)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ChannelEventBusTest
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyReactiveCaseChannelProviderTest

# E2E tests (requires Chromium installed)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=ChannelPanelE2ETest

# Full suite (run before each commit to confirm no regressions)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

---

## Task 1: Fix SSE double-frame bug (#132)

**Files:**
- Modify: `claudony-app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java`
- Modify: `claudony-app/src/main/java/io/casehub/claudony/server/MeshResource.java`

`events()` emits `"data: " + json + "\n\n"` as Multi<String> items. RESTEasy Reactive wraps each String as `data: <item>\n\n`, producing `data: data: {...}\n\n\n\n`. dashboard.js's `JSON.parse(e.data)` throws silently and the global mesh SSE never updates.

- [ ] **Step 1.1: Write the failing test in MeshResourceTest**

Add inside `class MeshResourceTest` (requires `@TestSecurity` already on the class):

```java
@Test
void meshEvents_sseFrameIsValidJson() throws Exception {
    int port = io.restassured.RestAssured.port;
    java.net.HttpURLConnection conn = (java.net.HttpURLConnection)
        new java.net.URL("http://localhost:" + port + "/api/mesh/events").openConnection();
    conn.setConnectTimeout(5000);
    conn.setReadTimeout(5000); // SSE tick interval is 3000ms in test config; wait up to 5s
    try {
        int status = conn.getResponseCode();
        org.assertj.core.api.Assertions.assertThat(status).isEqualTo(200);
        java.io.BufferedReader reader = new java.io.BufferedReader(
            new java.io.InputStreamReader(conn.getInputStream()));
        String line;
        // Skip blank lines; block until first "data:" line arrives (~3000ms)
        while ((line = reader.readLine()) != null && !line.startsWith("data:")) {}
        org.assertj.core.api.Assertions.assertThat(line).isNotNull().startsWith("data: ");
        String json = line.substring("data: ".length());
        com.fasterxml.jackson.databind.ObjectMapper om =
            new com.fasterxml.jackson.databind.ObjectMapper();
        com.fasterxml.jackson.databind.JsonNode node = om.readTree(json);
        org.assertj.core.api.Assertions.assertThat(node.has("channels")).isTrue();
        org.assertj.core.api.Assertions.assertThat(node.has("instances")).isTrue();
        org.assertj.core.api.Assertions.assertThat(node.has("feed")).isTrue();
    } finally {
        conn.disconnect();
    }
}
```

- [ ] **Step 1.2: Run the test to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest#meshEvents_sseFrameIsValidJson
```

Expected: FAIL — `om.readTree(json)` throws `JsonParseException` because `json` starts with `"data: "` (double-wrapped).

- [ ] **Step 1.3: Fix the three strings in `events()` in MeshResource.java**

Replace the entire `events()` method body's three emission strings:

```java
// Inside the combinedWith lambda — BEFORE:
return "data: " + mapper.writeValueAsString(Map.of(
        "channels", ch,
        "instances", inst,
        "feed", f)) + "\n\n";
// AFTER:
return mapper.writeValueAsString(Map.of(
        "channels", ch,
        "instances", inst,
        "feed", f));
```

```java
// Inner catch — BEFORE:
return "data: {}\n\n";
// AFTER:
return "{}";
```

```java
// Outer .recoverWithItem — BEFORE:
.recoverWithItem("data: {}\n\n");
// AFTER:
.recoverWithItem("{}");
```

The full corrected `events()` method after all three changes:

```java
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
                return Uni.combine().all().unis(channels, instances, feed)
                        .combinedWith((ch, inst, f) -> {
                            try {
                                return mapper.writeValueAsString(Map.of(
                                        "channels", ch,
                                        "instances", inst,
                                        "feed", f));
                            } catch (Exception e) {
                                return "{}";
                            }
                        })
                        .onFailure().recoverWithItem("{}");
            });
}
```

- [ ] **Step 1.4: Run the test to confirm it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest#meshEvents_sseFrameIsValidJson
```

Expected: PASS.

- [ ] **Step 1.5: Run the full MeshResourceTest to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest
```

Expected: all existing tests PASS.

- [ ] **Step 1.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  claudony-app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java \
  claudony-app/src/main/java/io/casehub/claudony/server/MeshResource.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
fix(mesh): remove manual SSE frame wrapping in events() — RESTEasy already wraps

Closes #132 Refs #123

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Fix ChannelEventBus TOCTOU (#133 M2)

**Files:**
- Modify: `claudony-app/src/test/java/io/casehub/claudony/server/ChannelEventBusTest.java`
- Modify: `claudony-app/src/main/java/io/casehub/claudony/server/ChannelEventBus.java`

`removeSubscriber()` checks `list.isEmpty()` then calls `subscribers.remove(channelName, list)`. Since `ConcurrentHashMap.remove(key, value)` uses `equals()` and `list.equals(list)` is always true (reflexive), a concurrent subscribe between those two steps would be silently evicted from the map. The fix uses `computeIfPresent` returning `null` to atomically remove-and-prune.

- [ ] **Step 2.1: Add test to ChannelEventBusTest**

```java
@Test
void subscribe_cancelBoth_mapEntryFullyRemoved() {
    var sub1 = bus.subscribe("ch").subscribe().with(t -> {});
    var sub2 = bus.subscribe("ch").subscribe().with(t -> {});
    assertThat(bus.subscriberCount("ch")).isEqualTo(2);

    sub1.cancel();
    assertThat(bus.subscriberCount("ch")).isEqualTo(1);

    sub2.cancel();
    assertThat(bus.subscriberCount("ch")).isZero();
}
```

Note: this test verifies correct sequential behavior and passes in both old and new code. The `computeIfPresent` fix makes removal correct under concurrent cancellation — the sequential test documents the expected behavior and guards regressions.

- [ ] **Step 2.2: Run to confirm test passes before the fix (expected)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ChannelEventBusTest#subscribe_cancelBoth_mapEntryFullyRemoved
```

Expected: PASS (the sequential case is correct in both old and new code — this is a regression guard).

- [ ] **Step 2.3: Apply the `computeIfPresent` fix in ChannelEventBus.java**

Replace the entire `removeSubscriber` method:

```java
private void removeSubscriber(String channelName, MultiEmitter<Integer> emitter) {
    subscribers.computeIfPresent(channelName, (key, list) -> {
        list.remove(emitter);
        return list.isEmpty() ? null : list;
    });
}
```

`computeIfPresent` executes atomically under the map segment lock. Returning `null` removes the map entry in the same atomic step, eliminating the TOCTOU between the empty-check and the conditional remove.

- [ ] **Step 2.4: Run all ChannelEventBusTest to confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ChannelEventBusTest
```

Expected: all 6 tests PASS.

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  claudony-app/src/test/java/io/casehub/claudony/server/ChannelEventBusTest.java \
  claudony-app/src/main/java/io/casehub/claudony/server/ChannelEventBus.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
fix(mesh): eliminate TOCTOU in ChannelEventBus.removeSubscriber via computeIfPresent

Closes #133 (partial — M2)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Registration atomicity + serializeEntries logging (#133 M1 + M4)

**Files:**
- Modify: `claudony-app/src/main/java/io/casehub/claudony/server/MeshResource.java`

M1: deregister+registerBackend in `channelEvents()` is not atomic — two concurrent SSE opens to the same channel can duplicate the `human_observer` registration, causing doubled tick delivery when #131 (ChannelEventBus-driven push) ships.

M4: `serializeEntries()` returns null on `JsonProcessingException` with no log, silently dropping SSE frames.

No new test for M1 (concurrency race is non-deterministic). Existing `MeshResourceTest.channelEvents_*` tests verify no regression. M4 verified by existing SSE tests.

- [ ] **Step 3.1: Add per-channel registration lock field to MeshResource**

Add this field alongside the other `@Inject` fields:

```java
private final ConcurrentHashMap<UUID, Object> channelRegistrationLocks = new ConcurrentHashMap<>();
```

`ConcurrentHashMap` is already imported (it's used in `ChannelEventBus`). Add import to `MeshResource.java` if missing:
```java
import java.util.concurrent.ConcurrentHashMap;
```

- [ ] **Step 3.2: Wrap the registration block in `channelEvents()` with a per-channel lock**

Find this block in `channelEvents()`:

```java
// Idempotent backend registration
ChannelRef ref = new ChannelRef(channelId, channelName);
gateway.deregisterBackend(channelId, ClaudonyChannelBackend.BACKEND_ID);
channelBackend.open(ref, Map.of());
gateway.registerBackend(channelId, channelBackend, "human_observer");
```

Replace with:

```java
// Per-channel lock prevents concurrent SSE opens from duplicating human_observer
// registration. Remove when ChannelGateway guards human_observer duplicates (#131).
ChannelRef ref = new ChannelRef(channelId, channelName);
synchronized (channelRegistrationLocks.computeIfAbsent(channelId, k -> new Object())) {
    gateway.deregisterBackend(channelId, ClaudonyChannelBackend.BACKEND_ID);
    channelBackend.open(ref, Map.of());
    gateway.registerBackend(channelId, channelBackend, "human_observer");
}
```

- [ ] **Step 3.3: Add LOG.errorf in serializeEntries()**

Find:

```java
private String serializeEntries(List<Map<String, Object>> entries) {
    try {
        return mapper.writeValueAsString(entries);
    } catch (com.fasterxml.jackson.core.JsonProcessingException e) {
        return null;
    }
}
```

Replace with:

```java
private String serializeEntries(List<Map<String, Object>> entries) {
    try {
        return mapper.writeValueAsString(entries);
    } catch (com.fasterxml.jackson.core.JsonProcessingException e) {
        LOG.errorf("Failed to serialize channel timeline entries: %s", e.getMessage());
        return null;
    }
}
```

- [ ] **Step 3.4: Run MeshResourceTest to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest
```

Expected: all tests PASS.

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  claudony-app/src/main/java/io/casehub/claudony/server/MeshResource.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
fix(mesh): per-channel lock for backend registration atomicity; log serializeEntries failure

Closes #133 (partial — M1, M4)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Commitment wire-up in postToChannel() (#122)

**Files:**
- Modify: `claudony-casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java`
- Modify: `claudony-casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`

`postToChannel()` passes `null` for `correlationId` to `messageService.send()`. When null, `ReactiveMessageService` never opens a Qhorus Commitment. Fix: parse `correlationId` from COMMAND/QUERY content JSON and pass it through.

**GE warning:** GE-20260521-e39ad1 — `CommitmentStore.findOpenByObligor(sender)` finds nothing for COMMAND messages because sender is stored as requester, not obligor. Tests verify at the `send()` call boundary only — never through CommitmentStore.

- [ ] **Step 4.1: Add four failing tests to ClaudonyReactiveCaseChannelProviderTest**

Add after the existing `postToChannel_nullType_sendsWithNullType` test:

```java
@Test
void postToChannel_commandWithCorrelationId_passesCorrelationIdToSend() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    String content = "{\"type\":\"COMMAND\",\"capability\":\"research\","
            + "\"correlationId\":\"42\",\"input\":{}}";
    when(messageService.send(any(), any(), any(), any(), any(), any(), any(), any(), any()))
            .thenReturn(Uni.createFrom().nullItem());

    provider.postToChannel(ch, "engine", content, MessageType.COMMAND).await().indefinitely();

    verify(messageService).send(
            eq(channelId), eq("engine"), eq(MessageType.COMMAND), eq(content),
            eq("42"), isNull(), isNull(), isNull(), isNull());
}

@Test
void postToChannel_queryWithCorrelationId_passesCorrelationIdToSend() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    String content = "{\"type\":\"QUERY\",\"correlationId\":\"q-99\",\"input\":{}}";
    when(messageService.send(any(), any(), any(), any(), any(), any(), any(), any(), any()))
            .thenReturn(Uni.createFrom().nullItem());

    provider.postToChannel(ch, "engine", content, MessageType.QUERY).await().indefinitely();

    verify(messageService).send(
            eq(channelId), eq("engine"), eq(MessageType.QUERY), eq(content),
            eq("q-99"), isNull(), isNull(), isNull(), isNull());
}

@Test
void postToChannel_commandMalformedJson_sendsWithNullCorrelationId() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    when(messageService.send(any(), any(), any(), any(), any(), any(), any(), any(), any()))
            .thenReturn(Uni.createFrom().nullItem());

    // Must not throw; must still deliver the message
    provider.postToChannel(ch, "engine", "not-valid-json", MessageType.COMMAND)
            .await().indefinitely();

    verify(messageService).send(
            eq(channelId), eq("engine"), eq(MessageType.COMMAND), eq("not-valid-json"),
            isNull(), isNull(), isNull(), isNull(), isNull());
}

@Test
void postToChannel_nonCommandType_doesNotParseCorrelationId() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    String content = "{\"correlationId\":\"should-be-ignored\"}";
    when(messageService.send(any(), any(), any(), any(), any(), any(), any(), any(), any()))
            .thenReturn(Uni.createFrom().nullItem());

    provider.postToChannel(ch, "engine", content, MessageType.STATUS).await().indefinitely();

    verify(messageService).send(
            eq(channelId), eq("engine"), eq(MessageType.STATUS), eq(content),
            isNull(), isNull(), isNull(), isNull(), isNull());
}
```

- [ ] **Step 4.2: Run to confirm the two correlation tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyReactiveCaseChannelProviderTest#postToChannel_commandWithCorrelationId_passesCorrelationIdToSend+postToChannel_queryWithCorrelationId_passesCorrelationIdToSend
```

Expected: FAIL — `send()` is called with `null` for correlationId (5th arg) instead of `"42"` / `"q-99"`.

- [ ] **Step 4.3: Add imports to ClaudonyReactiveCaseChannelProvider.java**

Add after existing imports:

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
```

- [ ] **Step 4.4: Add MAPPER field and extractCorrelationId method; update postToChannel()**

Add the static field after the class-level constants (`CHANNEL_PREFIX`, `QHORUS_NAME_KEY`):

```java
private static final ObjectMapper MAPPER = new ObjectMapper();
```

Replace `postToChannel()`:

```java
@Override
public Uni<Void> postToChannel(CaseChannel channel, String from, String content, MessageType type) {
    UUID channelId = UUID.fromString(channel.id());
    String correlationId = (type == MessageType.COMMAND || type == MessageType.QUERY)
            ? extractCorrelationId(content) : null;
    return messageService.send(channelId, from, type, content, correlationId, null, null, null, null)
            .replaceWithVoid();
}

// Content-coupling workaround: postToChannel() SPI doesn't carry correlationId as a
// first-class parameter. Track claudony#135 for the SPI fix that removes this method.
private static String extractCorrelationId(String content) {
    try {
        JsonNode node = MAPPER.readTree(content);
        JsonNode cid = node.get("correlationId");
        return (cid != null && !cid.isNull()) ? cid.asText() : null;
    } catch (Exception e) {
        log.warnf("Could not parse correlationId from COMMAND/QUERY content — Commitment will not be tracked");
        return null;
    }
}
```

- [ ] **Step 4.5: Run all ClaudonyReactiveCaseChannelProviderTest to confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyReactiveCaseChannelProviderTest
```

Expected: all tests PASS including the 4 new ones.

- [ ] **Step 4.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  claudony-casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java \
  claudony-casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
feat(casehub): extract correlationId from COMMAND/QUERY content to open Qhorus Commitment

Closes #122

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: JS quality fixes in terminal.js (#128 + #133-M5)

**Files:**
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/terminal.js`

Five targeted fixes. No Java unit tests. Behavior is covered by existing ChannelPanelE2ETest and the new E2E test in Task 7.

- [ ] **Step 5.1: m-1 — Add ts comment**

Find line 432:
```javascript
chCursors[chSelectedName] = {id: entry.id, ts: Date.now()};
```

Replace with:
```javascript
chCursors[chSelectedName] = {id: entry.id, ts: Date.now() /* time of last catch-up, not last message */};
```

- [ ] **Step 5.2: m-2 — Guard catchUp() against non-OK HTTP responses**

Find lines 484-491:
```javascript
function catchUp(name, fromId) {
    var url = '/api/mesh/channels/' + encodeURIComponent(name) +
              '/timeline?limit=50&after=' + fromId;
    fetch(url).then(function (r) { return r.json(); }).then(function (entries) {
        if (entries && entries.length) appendMessages(entries);
    }).catch(function () {});
    // EventSource already open from selectChannel(); no need to open again
}
```

Replace with:
```javascript
function catchUp(name, fromId) {
    var url = '/api/mesh/channels/' + encodeURIComponent(name) +
              '/timeline?limit=50&after=' + fromId;
    fetch(url).then(function (r) {
        if (!r.ok) return;
        return r.json();
    }).then(function (entries) {
        if (entries && entries.length) appendMessages(entries);
    }).catch(function () {
        chError.textContent = 'Catch-up failed — some messages may be missing.';
    });
    // EventSource already open from selectChannel(); no need to open again
}
```

- [ ] **Step 5.3: M5 — Guard fullLoad() the same way**

Find lines 493-506:
```javascript
function fullLoad(name) {
    var url = '/api/mesh/channels/' + encodeURIComponent(name) + '/timeline?limit=100';
    fetch(url).then(function (r) { return r.json(); }).then(function (entries) {
        if (entries && entries.length) {
            appendMessages(entries);
        } else {
            var empty = document.createElement('div');
            empty.className = 'ch-empty';
            empty.textContent = 'No messages yet.';
            chFeed.appendChild(empty);
        }
    }).catch(function () {});
    // EventSource already open from selectChannel(); no need to open again
}
```

Replace with:
```javascript
function fullLoad(name) {
    var url = '/api/mesh/channels/' + encodeURIComponent(name) + '/timeline?limit=100';
    fetch(url).then(function (r) {
        if (!r.ok) return;
        return r.json();
    }).then(function (entries) {
        if (entries && entries.length) {
            appendMessages(entries);
        } else {
            var empty = document.createElement('div');
            empty.className = 'ch-empty';
            empty.textContent = 'No messages yet.';
            chFeed.appendChild(empty);
        }
    }).catch(function () {
        chError.textContent = 'Failed to load messages.';
    });
    // EventSource already open from selectChannel(); no need to open again
}
```

- [ ] **Step 5.4: m-3 — Store stale prompt button refs as closure variables**

Find the variable declarations around line 165 (near `chStalePromptEl`):
```javascript
var chStalePromptEl   = null;
```

Replace with:
```javascript
var chStalePromptEl        = null;
var chStalePromptCatchupBtn = null;
var chStalePromptReloadBtn  = null;
```

Find inside `showStalePrompt()`, after button creation (lines ~520-531):
```javascript
chStalePromptEl.appendChild(msg);
chStalePromptEl.appendChild(catchupBtn);
chStalePromptEl.appendChild(reloadBtn);
chFeed.parentNode.insertBefore(chStalePromptEl, chFeed);
```

Add two assignment lines after creating the buttons, before `appendChild`:
```javascript
chStalePromptCatchupBtn = catchupBtn;
chStalePromptReloadBtn  = reloadBtn;
chStalePromptEl.appendChild(msg);
chStalePromptEl.appendChild(catchupBtn);
chStalePromptEl.appendChild(reloadBtn);
chFeed.parentNode.insertBefore(chStalePromptEl, chFeed);
```

Then replace both `document.getElementById(...)` calls in `showStalePrompt()`:

```javascript
// BEFORE:
document.getElementById('ch-stale-catchup-btn').onclick = function () {
// AFTER:
chStalePromptCatchupBtn.onclick = function () {
```

```javascript
// BEFORE:
document.getElementById('ch-stale-reload-btn').onclick = function () {
// AFTER:
chStalePromptReloadBtn.onclick = function () {
```

- [ ] **Step 5.5: m-4 — closePanel() calls hideStalePrompt()**

Find lines 624-630:
```javascript
function closePanel() {
    chPanel.classList.add('collapsed');
    clearTimeout(chPollTimer);
    if (chEventSource) { chEventSource.close(); chEventSource = null; }
    clearTimeout(lineagePollTimer);
    clearInterval(elapsedTicker);
}
```

Replace with:
```javascript
function closePanel() {
    hideStalePrompt();
    chPanel.classList.add('collapsed');
    clearTimeout(chPollTimer);
    if (chEventSource) { chEventSource.close(); chEventSource = null; }
    clearTimeout(lineagePollTimer);
    clearInterval(elapsedTicker);
}
```

- [ ] **Step 5.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  claudony-app/src/main/resources/META-INF/resources/app/terminal.js
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
fix(mesh): JS quality — catchUp/fullLoad guards, button refs, closePanel hygiene, ts comment

Closes #128
Closes #133 (partial — M5)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Feed test coverage (#123)

**Files:**
- Modify: `claudony-app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java`

Add three tests for `GET /api/mesh/feed` content behavior (the empty-list case is already covered). Tests insert messages directly via `InMemoryMessageStore.put()` — this is the correct approach for `@QuarkusTest` integration tests (bypasses Panache transaction context constraint).

Add the following import to `MeshResourceTest.java`:
```java
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.api.message.MessageType;
```

- [ ] **Step 6.1: Write three new feed tests**

Add inside `class MeshResourceTest`:

```java
@Test
void meshFeed_withMessages_returnsEntriesTaggedWithChannelName() {
    Channel ch = new Channel();
    ch.name = "feed-tagged-" + System.nanoTime();
    ch.semantic = io.casehub.qhorus.api.channel.ChannelSemantic.APPEND;
    channelStore.put(ch);

    Message msg = new Message();
    msg.channelId = ch.id;
    msg.sender = "agent:test";
    msg.messageType = MessageType.STATUS;
    msg.content = "hello from channel";
    messageStore.put(msg);

    given().when().get("/api/mesh/feed")
        .then()
        .statusCode(200)
        .body("$", hasSize(greaterThan(0)))
        .body("[0].channel", equalTo(ch.name));
}

@Test
void meshFeed_multiChannel_returnsMergedEntries() {
    Channel ch1 = new Channel();
    ch1.name = "feed-multi-a-" + System.nanoTime();
    ch1.semantic = io.casehub.qhorus.api.channel.ChannelSemantic.APPEND;
    channelStore.put(ch1);

    Channel ch2 = new Channel();
    ch2.name = "feed-multi-b-" + System.nanoTime();
    ch2.semantic = io.casehub.qhorus.api.channel.ChannelSemantic.APPEND;
    channelStore.put(ch2);

    Message m1 = new Message();
    m1.channelId = ch1.id;
    m1.sender = "a";
    m1.messageType = MessageType.STATUS;
    m1.content = "from ch1";
    messageStore.put(m1);

    Message m2 = new Message();
    m2.channelId = ch2.id;
    m2.sender = "b";
    m2.messageType = MessageType.STATUS;
    m2.content = "from ch2";
    messageStore.put(m2);

    given().when().get("/api/mesh/feed")
        .then()
        .statusCode(200)
        .body("$", hasSize(2));
}

@Test
void meshFeed_limitTruncates() {
    Channel ch = new Channel();
    ch.name = "feed-limit-" + System.nanoTime();
    ch.semantic = io.casehub.qhorus.api.channel.ChannelSemantic.APPEND;
    channelStore.put(ch);

    for (int i = 0; i < 10; i++) {
        Message msg = new Message();
        msg.channelId = ch.id;
        msg.sender = "agent:test";
        msg.messageType = MessageType.STATUS;
        msg.content = "msg-" + i;
        messageStore.put(msg);
    }

    // getFeed() uses Math.max(5, limit/channels.size()) per channel, then truncates to limit.
    // With 1 channel and limit=3: perChannel=5 (minimum), fetches 5, truncated to 3.
    given().when().get("/api/mesh/feed?limit=3")
        .then()
        .statusCode(200)
        .body("$", hasSize(equalTo(3)));
}
```

Note: `greaterThan` and `equalTo` are already available via `import static org.hamcrest.Matchers.*`.

- [ ] **Step 6.2: Run all feed tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=MeshResourceTest
```

Expected: all tests PASS (these are coverage for existing correct behavior).

- [ ] **Step 6.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  claudony-app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
test(mesh): add feed content tests — tagged entries, multi-channel merge, limit truncation

Closes #123

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: E2E EventSource error → poll fallback (#130)

**Files:**
- Modify: `claudony-app/src/test/java/io/casehub/claudony/e2e/ChannelPanelE2ETest.java`

The channel panel falls back to `pollChannel()` (3-second timer) on EventSource error/close. This path has no E2E test. Playwright's `page.route()` intercepts and aborts the SSE connection, triggering `onerror` → `pollChannel()` fallback.

- [ ] **Step 7.1: Add `channelId` field and set it in @BeforeEach**

The existing `@BeforeEach` has `private String channelName`. Add `channelId` alongside it so tests can construct messages with the channel's UUID.

Add the field:
```java
private String channelName;
private java.util.UUID channelId;   // ADD — set after channelStore.put() assigns the UUID
```

Update `createChannel()` `@BeforeEach` — add `channelId = ch.id;` after `channelStore.put(ch)`:

```java
@BeforeEach
void createChannel() {
    channelName = "ch-panel-e2e-" + System.nanoTime();
    Channel ch = new Channel();
    ch.name = channelName;
    ch.description = "E2E test channel";
    ch.semantic = ChannelSemantic.APPEND;
    channelStore.put(ch);
    channelId = ch.id;   // ADD
}
```

- [ ] **Step 7.2: Add the import for Message and MessageType**

Add to imports in `ChannelPanelE2ETest.java`:
```java
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.api.message.MessageType;
```

- [ ] **Step 7.3: Write the failing E2E test**

Add as a new `@Test` method:

```java
@Test
void channelPanel_eventSourceError_fallsBackToPoll() {
    // Abort SSE connections BEFORE navigation so the route is registered when EventSource opens.
    // The route matches /api/mesh/channels/{name}/events?after=... for any channel name.
    page.route("**/api/mesh/channels/*/events*", route -> route.abort());

    // Navigate with the channel pre-selected in the URL param — openPanel() will auto-select it.
    navigateToSessionPageWithChannel();
    openPanel();

    // Wait for fullLoad() to complete: channel is empty, so "No messages yet." appears.
    // fullLoad() uses /timeline (not /events), so it is NOT affected by the route abort.
    page.locator(".ch-empty").waitFor(
            new com.microsoft.playwright.Locator.WaitForOptions().setTimeout(5000));

    // Insert a message directly into the in-memory store.
    // The next pollChannel() tick (fires after POLL_MS=3000ms from EventSource onerror) will
    // fetch /timeline and deliver this message.
    Message msg = new Message();
    msg.channelId = channelId;
    msg.sender = "agent:poll-fallback-test";
    msg.messageType = MessageType.STATUS;
    msg.content = "poll-delivers-this";
    messageStore.put(msg);

    // Wait up to 6 seconds for the poll cycle to deliver the message (POLL_MS=3000 + 2s margin).
    page.locator("#ch-feed .ch-msg").first().waitFor(
            new com.microsoft.playwright.Locator.WaitForOptions().setTimeout(6000));

    assertThat(page.locator("#ch-feed").textContent()).contains("poll-delivers-this");
}
```

- [ ] **Step 7.4: Run the test (expect it passes — production code already handles onerror)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=ChannelPanelE2ETest#channelPanel_eventSourceError_fallsBackToPoll
```

Expected: PASS. The `onerror` handler is already wired to `pollChannel()`. This test provides regression coverage.

If FAIL: verify that `page.locator(".ch-empty")` is the correct selector (look at `fullLoad()` in terminal.js — it creates `div.ch-empty`). Adjust the timeout if the poll cycle is slower in CI.

- [ ] **Step 7.5: Run the full E2E test suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=ChannelPanelE2ETest
```

Expected: all 19 tests PASS (18 existing + 1 new).

- [ ] **Step 7.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  claudony-app/src/test/java/io/casehub/claudony/e2e/ChannelPanelE2ETest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
test(e2e): EventSource abort → poll fallback via Playwright route interception

Closes #130
Closes #133 (completes all M-items from this batch)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Full suite verification + push

- [ ] **Step 8.1: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: 502+ tests (498 baseline + 4 new unit tests in claudony-casehub, new tests in claudony-app), 0 failures.

- [ ] **Step 8.2: Push the branch**

```bash
git -C /Users/mdproctor/claude/casehub/claudony push -u origin issue-122-commitment-quality-batch
```

---

## Self-review

**Spec coverage:**
- #122 correlationId wire-up → Task 4 ✅
- #132 double-frame fix → Task 1 ✅
- #133 M1 registration atomicity → Task 3 ✅
- #133 M2 TOCTOU → Task 2 ✅
- #133 M4 logging → Task 3 ✅
- #133 M5 JS error feedback → Task 5 (m-2/M5) ✅
- #128 m-1 ts comment → Task 5 ✅
- #128 m-2 catchUp guard → Task 5 ✅
- #128 m-3 button refs → Task 5 ✅
- #128 m-4 closePanel hygiene → Task 5 ✅
- #123 feed with messages, multi-channel, limit → Task 6 ✅
- #123 events SSE frame valid JSON → Task 1 (Step 1.1) ✅
- #130 EventSource fallback → Task 7 ✅
- #124 deferred → not in plan ✅

**Placeholder scan:** No TBDs, no "similar to Task N" references. All test code is complete. ✅

**Type consistency:** `extractCorrelationId()` named consistently in Task 4 spec/impl. `MAPPER` static field referenced only within Task 4. `channelRegistrationLocks` field referenced only in Task 3. ✅
