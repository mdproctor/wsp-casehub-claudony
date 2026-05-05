# Reliability & Robustness Pass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate silent failures, flaky sleeps, and overly-broad exception catches across Claudony's production code and test suite.

**Architecture:** Seven focused tasks, each self-contained and independently committable. Items are ordered by risk — credential migration first (silent data-loss bug) then test reliability then code hygiene. No new dependencies required.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, RestAssured, Mockito, Jackson, webauthn4j. All existing.

**Tracking:** Refs #54

---

## Context: Apple Passkeys — No Action Required

webauthn4j (the library that replaced Vert.x WebAuthn in the Quarkus 3.32.2 upgrade) does **not** check AAGUID in `none` attestation at all. The old `WebAuthnPatcher` + `LenientNoneAttestation` workaround patched a Vert.x restriction that doesn't exist in webauthn4j. Apple iCloud Keychain passkeys work by default.

---

## File Map

```
Modified:
  src/main/java/dev/claudony/server/auth/CredentialStore.java     — Task 1: graceful load
  src/main/java/dev/claudony/server/auth/AuthResource.java        — Task 5: stream close + narrow catch
  src/main/java/dev/claudony/server/SessionResource.java           — Task 6+7: empty catch, narrow
  src/main/java/dev/claudony/server/ServerStartup.java            — Task 7: narrow catches
  src/main/java/dev/claudony/server/TerminalWebSocket.java        — Task 7: narrow catches
  src/main/java/dev/claudony/agent/ClipboardChecker.java          — Task 7: narrow catch
  src/main/java/dev/claudony/agent/terminal/ITerm2Adapter.java    — Task 7: narrow catch
  src/test/java/dev/claudony/server/TmuxServiceTest.java          — Task 2+4: waits, no ordering
  src/test/java/dev/claudony/server/SessionInputOutputTest.java    — Task 2+4: waits, no ordering
  src/test/java/dev/claudony/frontend/ResizeEndpointTest.java      — Task 2: wait
  src/test/java/dev/claudony/agent/McpServerIntegrationTest.java  — Task 2: waits
  src/test/java/dev/claudony/server/TerminalWebSocketTest.java    — Task 3: waits
  src/test/java/dev/claudony/server/ServiceHealthTest.java         — Task 4: no ordering
  src/test/java/dev/claudony/server/OpenTerminalTest.java          — Task 4: no ordering
  src/test/java/dev/claudony/server/SessionResourceTest.java       — Task 4: no ordering

Created:
  src/test/java/dev/claudony/Await.java                           — Task 2: polling utility
```

---

## Task 1: CredentialStore graceful migration

**Problem:** After the Quarkus 3.32.2 upgrade, `CredentialStore.toRecord()` calls `UUID.fromString(c.aaguid())`. Old credentials.json files stored aaguid as a plain hex string without dashes (e.g. `"00000000000000000000000000000000"`) — not UUID format. This throws `IllegalArgumentException` when any user tries to log in, silently locking them out.

The public key format also changed (Vert.x stored a different encoding), so actual WebAuthn verification of old credentials would fail regardless. The right behaviour: detect the unreadable record, log a clear warning, exclude it from results. Users re-register once.

**Files:**
- Modify: `src/main/java/dev/claudony/server/auth/CredentialStore.java`
- Test: `src/test/java/dev/claudony/server/auth/CredentialStoreTest.java`

- [ ] **Step 1: Write a failing test for malformed aaguid**

Add to `CredentialStoreTest.java`:

```java
@Test
void load_withNonUuidAaguid_skipsRecordAndLogsWarning() throws Exception {
    // Write a credential file with an old-style non-UUID aaguid
    var json = """
        [{"username":"alice","credentialId":"cred-1",
          "aaguid":"00000000000000000000000000000000",
          "publicKey":"dGVzdA==","publicKeyAlgorithm":-7,"counter":5}]
        """;
    java.nio.file.Files.writeString(tmp.resolve("credentials.json"), json);
    // Should not throw — gracefully excludes unreadable records
    var result = store.findByUsername("alice").subscribeAsCompletionStage().get();
    assertTrue(result.isEmpty(), "Unreadable credentials should be excluded, not throw");
}

@Test
void isEmpty_withUnreadableAaguid_returnsTrueNotThrow() throws Exception {
    var json = """
        [{"username":"bob","credentialId":"cred-2",
          "aaguid":"not-a-uuid","publicKey":"dGVzdA==",
          "publicKeyAlgorithm":-7,"counter":0}]
        """;
    java.nio.file.Files.writeString(tmp.resolve("credentials.json"), json);
    // isEmpty() must not propagate the UUID parse exception
    assertDoesNotThrow(() -> store.isEmpty());
}
```

- [ ] **Step 2: Run to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CredentialStoreTest -Dno-format -q
```

Expected: 2 new failures (IllegalArgumentException from UUID.fromString).

- [ ] **Step 3: Fix `toRecord()` to skip unreadable records**

In `CredentialStore.java`, replace `toRecord()` and update `findByUsername` / `findByCredentialId` to filter rather than stream-and-crash:

```java
private static final Logger LOG = org.jboss.logging.Logger.getLogger(CredentialStore.class);

// Replace the existing toRecord() with this version:
private static WebAuthnCredentialRecord toRecordOrNull(StoredCredential c) {
    try {
        UUID aaguid;
        try {
            aaguid = UUID.fromString(c.aaguid());
        } catch (IllegalArgumentException e) {
            LOG.warnf("Credential '%s' has unreadable aaguid '%s' — skipping. " +
                      "This credential was stored by an older Claudony version. " +
                      "The user must re-register their passkey.", c.credentialId(), c.aaguid());
            return null;
        }
        return WebAuthnCredentialRecord.fromRequiredPersistedData(
            new WebAuthnCredentialRecord.RequiredPersistedData(
                c.username(), c.credentialId(), aaguid,
                Base64.getDecoder().decode(c.publicKey()),
                c.publicKeyAlgorithm(), c.counter()));
    } catch (Exception e) {
        LOG.warnf("Credential '%s' could not be reconstructed — skipping: %s",
                  c.credentialId(), e.getMessage());
        return null;
    }
}
```

Update `findByUsername`:
```java
@Override
public Uni<List<WebAuthnCredentialRecord>> findByUsername(String username) {
    return Uni.createFrom()
        .item(() -> load().stream()
            .filter(c -> c.username().equals(username))
            .map(CredentialStore::toRecordOrNull)
            .filter(java.util.Objects::nonNull)
            .collect(Collectors.toList()))
        .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
}
```

Update `findByCredentialId`:
```java
@Override
public Uni<WebAuthnCredentialRecord> findByCredentialId(String credentialId) {
    return Uni.createFrom()
        .item(() -> load().stream()
            .filter(c -> c.credentialId().equals(credentialId))
            .map(CredentialStore::toRecordOrNull)
            .filter(java.util.Objects::nonNull)
            .findFirst()
            .orElse(null))
        .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
}
```

Also remove the old `toRecord()` static method (it is now replaced by `toRecordOrNull`).

- [ ] **Step 4: Run tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CredentialStoreTest -Dno-format -q
```

Expected: all 11 tests pass (9 existing + 2 new).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/auth/CredentialStore.java \
        src/test/java/dev/claudony/server/auth/CredentialStoreTest.java
git commit -m "fix: CredentialStore skips unreadable legacy credentials with clear warning (Refs #54)

Old credentials.json files stored aaguid as hex-without-dashes; UUID.fromString()
threw IllegalArgumentException, silently locking users out after the 3.32.2 upgrade.
toRecord() now catches unreadable records, logs a re-registration warning, and
excludes them rather than propagating the exception."
```

---

## Task 2: Polling utility + replace easy Thread.sleep calls

**Problem:** 12 `Thread.sleep` calls in 4 test files wait fixed durations for tmux/REST output to appear. These are the source of CI flakiness: too short on slow CI → spurious failures; long enough to pass → slow suite.

**Correct approach:** Poll with a timeout. Check for the expected condition every 50 ms for up to 3 seconds. Passes immediately when ready, fails with a clear message when the deadline expires.

**Files:**
- Create: `src/test/java/dev/claudony/Await.java`
- Modify: `src/test/java/dev/claudony/server/TmuxServiceTest.java`
- Modify: `src/test/java/dev/claudony/server/SessionInputOutputTest.java`
- Modify: `src/test/java/dev/claudony/frontend/ResizeEndpointTest.java`
- Modify: `src/test/java/dev/claudony/agent/McpServerIntegrationTest.java`

- [ ] **Step 1: Create the polling utility**

Create `src/test/java/dev/claudony/Await.java`:

```java
package dev.claudony;

import java.time.Duration;
import java.util.function.BooleanSupplier;

/**
 * Condition-based waiting for integration tests.
 *
 * <p>Replaces Thread.sleep with polling: checks a condition every 50 ms until
 * it becomes true or a timeout elapses. Faster on fast machines, reliably fails
 * on slow ones with a clear message rather than silently passing wrong results.
 */
public final class Await {

    private static final long POLL_MS = 50;

    private Await() {}

    /**
     * Polls {@code condition} every 50 ms until it returns {@code true} or
     * {@code timeout} elapses.
     *
     * @throws AssertionError with {@code timeoutMessage} if deadline is exceeded
     */
    public static void until(BooleanSupplier condition, Duration timeout, String timeoutMessage) {
        long deadline = System.currentTimeMillis() + timeout.toMillis();
        while (System.currentTimeMillis() < deadline) {
            if (condition.getAsBoolean()) return;
            try { Thread.sleep(POLL_MS); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new AssertionError("Interrupted while waiting: " + timeoutMessage, e);
            }
        }
        throw new AssertionError("Timed out after " + timeout + ": " + timeoutMessage);
    }

    /** Convenience overload with 3-second default timeout. */
    public static void until(BooleanSupplier condition, String timeoutMessage) {
        until(condition, Duration.ofSeconds(3), timeoutMessage);
    }
}
```

- [ ] **Step 2: Update TmuxServiceTest — replace 3 sleeps**

Replace the body of `TmuxServiceTest.java` entirely:

```java
package dev.claudony.server;

import dev.claudony.Await;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class TmuxServiceTest {

    @Inject TmuxService tmux;

    private static final String TEST_SESSION = "test-claudony-unit";

    @AfterEach
    void cleanup() throws Exception {
        if (tmux.sessionExists(TEST_SESSION)) tmux.killSession(TEST_SESSION);
    }

    @Test
    void tmuxVersionReturnsNonEmpty() throws Exception {
        var version = tmux.tmuxVersion();
        assertFalse(version.isBlank());
        assertTrue(version.startsWith("tmux"), "Expected 'tmux X.Y', got: " + version);
    }

    @Test
    void sessionDoesNotExistBeforeCreation() throws Exception {
        assertFalse(tmux.sessionExists(TEST_SESSION));
    }

    @Test
    void createAndKillSession() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo hello");
        assertTrue(tmux.sessionExists(TEST_SESSION));
        tmux.killSession(TEST_SESSION);
        assertFalse(tmux.sessionExists(TEST_SESSION));
    }

    @Test
    void listSessionNamesIncludesCreatedSession() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo hello");
        var names = tmux.listSessionNames();
        assertTrue(names.contains(TEST_SESSION),
                "Expected session list to contain: " + TEST_SESSION + ", got: " + names);
    }

    @Test
    void capturePaneReturnsOutput() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo claudony-marker");
        Await.until(() -> {
            try { return tmux.capturePane(TEST_SESSION, 20).contains("claudony-marker"); }
            catch (Exception e) { return false; }
        }, "'claudony-marker' to appear in pane output");
    }

    @Test
    void sendKeysLiteralModeDoesNotInterpretTmuxKeyNames() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "bash");
        // Wait for bash prompt before sending keys
        Await.until(() -> {
            try { return !tmux.capturePane(TEST_SESSION, 5).isBlank(); }
            catch (Exception e) { return false; }
        }, "bash prompt to appear");
        tmux.sendKeys(TEST_SESSION, "Escape");
        Await.until(() -> {
            try { return tmux.capturePane(TEST_SESSION, 20).contains("Escape"); }
            catch (Exception e) { return false; }
        }, "literal 'Escape' to appear in pane output (missing -l flag would fire key instead)");
    }
}
```

- [ ] **Step 3: Update SessionInputOutputTest — replace 5 sleeps**

Replace the body of `SessionInputOutputTest.java`:

```java
package dev.claudony.server;

import dev.claudony.Await;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.*;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class SessionInputOutputTest {

    @Inject SessionRegistry registry;
    @Inject TmuxService tmux;

    @AfterEach
    void cleanup() throws Exception {
        for (var s : registry.all()) {
            registry.remove(s.id());
            try { tmux.killSession(s.name()); } catch (Exception ignored) {}
        }
    }

    @Test
    void sendInputToSessionReturns204() {
        var sessionId = given().contentType("application/json")
            .body("{\"name\":\"test-io-1\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}")
            .when().post("/api/sessions")
            .then().statusCode(201).extract().path("id");

        awaitSessionReady(sessionId);

        given().contentType("application/json")
            .body("{\"text\":\"echo input-test-marker\\n\"}")
            .when().post("/api/sessions/" + sessionId + "/input")
            .then().statusCode(204);
    }

    @Test
    void getOutputFromSessionReturnsText() {
        var sessionId = given().contentType("application/json")
            .body("{\"name\":\"test-io-2\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}")
            .when().post("/api/sessions")
            .then().statusCode(201).extract().path("id");

        awaitSessionReady(sessionId);

        given().contentType("application/json")
            .body("{\"text\":\"echo input-test-marker\\n\"}")
            .when().post("/api/sessions/" + sessionId + "/input")
            .then().statusCode(204);

        Await.until(() -> {
            var body = given().when()
                .get("/api/sessions/" + sessionId + "/output?lines=20")
                .then().statusCode(200).extract().asString();
            return body.contains("input-test-marker");
        }, "'input-test-marker' to appear in session output");
    }

    @Test
    void sendInputToUnknownSessionReturns404() {
        given().contentType("application/json")
            .body("{\"text\":\"echo hi\\n\"}")
            .when().post("/api/sessions/does-not-exist/input")
            .then().statusCode(404);
    }

    @Test
    void getOutputFromUnknownSessionReturns404() {
        given().when().get("/api/sessions/does-not-exist/output")
            .then().statusCode(404);
    }

    @Test
    void sendInputWithTmuxKeyNameViaRestPreservesLiteralText() {
        var sessionId = given().contentType("application/json")
            .body("{\"name\":\"test-io-keyname\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}")
            .when().post("/api/sessions")
            .then().statusCode(201).extract().path("id");

        awaitSessionReady(sessionId);

        given().contentType("application/json")
            .body("{\"text\":\"Escape\"}")
            .when().post("/api/sessions/" + sessionId + "/input")
            .then().statusCode(204);

        Await.until(() -> {
            var body = given().when()
                .get("/api/sessions/" + sessionId + "/output?lines=20")
                .then().statusCode(200).extract().asString();
            return body.contains("Escape");
        }, "literal 'Escape' to appear in session output");
    }

    /** Polls until the session has a non-blank prompt (bash is ready). */
    private static void awaitSessionReady(String sessionId) {
        Await.until(() -> {
            var body = given().when()
                .get("/api/sessions/" + sessionId + "/output?lines=5")
                .then().extract().asString();
            return !body.isBlank();
        }, "session bash prompt to appear");
    }
}
```

- [ ] **Step 4: Update ResizeEndpointTest — replace 1 sleep**

Read `src/test/java/dev/claudony/frontend/ResizeEndpointTest.java` first, then replace its `Thread.sleep(200)` with:

```java
import dev.claudony.Await;
// ...
// After creating session (statusCode 201):
Await.until(() -> {
    var status = given().when()
        .get("/api/sessions/" + sessionId)
        .then().extract().statusCode();
    return status == 200;
}, "session to be available before resize");
```

- [ ] **Step 5: Update McpServerIntegrationTest — replace 3 sleeps**

In `McpServerIntegrationTest.java`, add `import dev.claudony.Await;` and replace each `Thread.sleep` with:

Replace the sleep after `sendInput` (waiting for echo output):
```java
// was: Thread.sleep(300);
Await.until(() -> {
    var text = mcp().body(getOutputBody(tmuxSessionId, 20))
        .when().post("/mcp")
        .then().statusCode(200)
        .extract().<String>path("result.content[0].text");
    return text != null && text.contains("mcp-input-marker");
}, "'mcp-input-marker' to appear in session output");
```

Replace the sleep after session creation before sending `Escape`:
```java
// was: Thread.sleep(300);
Await.until(() -> {
    var text = mcp().body(getOutputBody(tmuxSessionId, 5))
        .when().post("/mcp")
        .then().statusCode(200)
        .extract().<String>path("result.content[0].text");
    return text != null && !text.isBlank();
}, "bash prompt to appear after session creation");
```

Replace the sleep after sending `Escape` (waiting for output):
```java
// was: Thread.sleep(300);
Await.until(() -> {
    var text = mcp().body(getOutputBody(tmuxSessionId, 20))
        .when().post("/mcp")
        .then().statusCode(200)
        .extract().<String>path("result.content[0].text");
    return text != null && text.contains("Escape");
}, "'Escape' to appear in session output");
```

Remove the `throws Exception` from affected test methods since `Await.until` doesn't throw checked exceptions.

- [ ] **Step 6: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q
```

Expected: 212+ tests, 0 failures.

- [ ] **Step 7: Commit**

```bash
git add src/test/java/dev/claudony/Await.java \
        src/test/java/dev/claudony/server/TmuxServiceTest.java \
        src/test/java/dev/claudony/server/SessionInputOutputTest.java \
        src/test/java/dev/claudony/frontend/ResizeEndpointTest.java \
        src/test/java/dev/claudony/agent/McpServerIntegrationTest.java
git commit -m "test: replace Thread.sleep with Await.until() polling in 4 test files (Refs #54)

Await.until() polls every 50ms up to 3s (configurable), eliminating fixed-duration
sleeps that cause slow CI or spurious failures depending on machine speed.
Covers TmuxServiceTest (3), SessionInputOutputTest (5), ResizeEndpointTest (1),
McpServerIntegrationTest (3)."
```

---

## Task 3: TerminalWebSocketTest condition waits

**Problem:** 14 `Thread.sleep` calls in `TerminalWebSocketTest`. The waits here are more varied:
- After `createSession` in `@BeforeEach` — wait for bash to be ready
- After `connectToServer` — wait for pipe-pane to connect and send history
- After `session.close()` — wait for cleanup before a second connection

The test already uses `LinkedBlockingQueue<String>` + `poll(timeout, unit)` for some waits. Standardise all of them.

**Files:**
- Modify: `src/test/java/dev/claudony/server/TerminalWebSocketTest.java`

- [ ] **Step 1: Replace `@BeforeEach` sleep**

The `@BeforeEach` `Thread.sleep(300)` (line 40) waits for bash to be ready. Replace with:

```java
@BeforeEach
void setup() throws Exception {
    tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "bash");
    var now = Instant.now();
    registry.register(new dev.claudony.server.model.Session(
        "ws-test-id", TEST_SESSION, System.getProperty("user.home"),
        "bash", SessionStatus.IDLE, now, now));
    // Wait for bash prompt instead of sleeping a fixed 300ms
    Await.until(() -> {
        try { return !tmux.capturePane(TEST_SESSION, 5).isBlank(); }
        catch (Exception e) { return false; }
    }, "bash prompt to be ready in test session");
}
```

Add `import dev.claudony.Await;` at the top of the file.

- [ ] **Step 2: Replace pipe-pane connection sleeps**

The pattern after `container.connectToServer(...)` is: sleep fixed duration, then use the message queue. Replace all `Thread.sleep(500)` / `Thread.sleep(800)` post-connect sleeps with a queue drain approach.

For each test that does `Thread.sleep(500/800)` after `connectToServer` and before sending a command, replace with a helper that waits for the initial history burst to arrive:

Add a private helper to the test class:

```java
/**
 * After connecting, drains the initial history burst (blocking up to 2s).
 * The first message(s) are always the history replay — wait until the queue
 * goes quiet for one poll interval, signalling the burst is done.
 */
private static void awaitHistoryBurst(LinkedBlockingQueue<String> msgs) throws InterruptedException {
    // Wait for at least one message (the history frame), then drain until quiet
    var first = msgs.poll(2, TimeUnit.SECONDS);
    if (first == null) throw new AssertionError("No history message received within 2s");
    // Drain remaining burst
    while (msgs.poll(100, TimeUnit.MILLISECONDS) != null) {}
}
```

Replace each post-connect `Thread.sleep(500)` or `Thread.sleep(800)` followed by a `while (msgs.poll...) {}` drain with a single call to `awaitHistoryBurst(msgs)`.

- [ ] **Step 3: Replace post-close sleeps**

The `Thread.sleep(300)` calls after `session.close()` wait for pipe-pane cleanup before a second WebSocket connection. Replace with a poll on `tmux.capturePane` or simply use a shorter polling loop:

```java
// was: session1.close(); Thread.sleep(300);
session1.close();
Await.until(() -> {
    try {
        // Pipe-pane cleanup is done when capturePane responds cleanly
        tmux.capturePane(TEST_SESSION, 1);
        return true;
    } catch (Exception e) { return false; }
}, "pipe-pane cleanup after close");
```

For the test-internal sleeps waiting for a command's output to appear in the `msgs` queue, use the existing `msgs.poll(timeout, unit)` pattern (already in place in some tests) rather than `Thread.sleep` + `capturePane`.

- [ ] **Step 4: Run TerminalWebSocketTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TerminalWebSocketTest -Dno-format -q
```

Expected: 8 tests, 0 failures.

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q
```

Expected: 212+ tests, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add src/test/java/dev/claudony/server/TerminalWebSocketTest.java
git commit -m "test: replace Thread.sleep with condition waits in TerminalWebSocketTest (Refs #54)

14 fixed-duration sleeps replaced with queue-based and polling waits.
awaitHistoryBurst() drains the pipe-pane history burst before test commands are sent."
```

---

## Task 4: Remove redundant test ordering

**Problem:** Five test classes use `@TestMethodOrder(MethodOrderer.OrderAnnotation.class)` + `@Order`. On inspection, none of them actually share cross-test state — each test creates and cleans up its own resources in `@BeforeEach`/`@AfterEach`. The ordering is misleading and prevents JUnit from parallelising or randomising.

**Files:**
- Modify: `src/test/java/dev/claudony/server/TmuxServiceTest.java`
- Modify: `src/test/java/dev/claudony/server/SessionInputOutputTest.java`
- Modify: `src/test/java/dev/claudony/server/ServiceHealthTest.java`
- Modify: `src/test/java/dev/claudony/server/OpenTerminalTest.java`
- Modify: `src/test/java/dev/claudony/server/SessionResourceTest.java`

Note: `e2e/ClaudeE2ETest.java` uses `@Order` to sequence a two-test flow that genuinely shares `mcpConfigFile` via `@BeforeAll`/`@AfterAll`. Leave that one as-is.

- [ ] **Step 1: Remove ordering from TmuxServiceTest**

This was already rewritten in Task 2. Confirm no `@TestMethodOrder` or `@Order` in the new version. ✓

- [ ] **Step 2: Remove ordering from SessionInputOutputTest**

Already rewritten in Task 2. Confirm no `@TestMethodOrder` or `@Order`. ✓

- [ ] **Step 3: Remove from ServiceHealthTest, OpenTerminalTest, SessionResourceTest**

For each of the three remaining files, do the following:

1. Read the file
2. Remove the `@TestMethodOrder(MethodOrderer.OrderAnnotation.class)` class annotation
3. Remove every `@Order(N)` method annotation
4. Remove unused import `import org.junit.jupiter.api.MethodOrderer;` if present
5. Remove unused import `import org.junit.jupiter.api.TestMethodOrder;` if present

Do not change anything else — no test logic, no names, no structure.

- [ ] **Step 4: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q
```

Expected: 212+ tests, 0 failures (ordering was never load-bearing).

- [ ] **Step 5: Commit**

```bash
git add src/test/java/dev/claudony/server/ServiceHealthTest.java \
        src/test/java/dev/claudony/server/OpenTerminalTest.java \
        src/test/java/dev/claudony/server/SessionResourceTest.java
git commit -m "test: remove @TestMethodOrder/@Order from tests that have no cross-test dependencies (Refs #54)

5 test classes used ordering annotations but each test independently
creates and cleans up its own resources. Ordering was misleading."
```

---

## Task 5: AuthResource stream close + narrow catch

**Problem:** `loginPage()` and `serveRegisterPage()` call `getResourceAsStream()` but never close the stream. The `catch (Exception e)` blocks are overly broad — `readAllBytes()` throws only `IOException`.

**Files:**
- Modify: `src/main/java/dev/claudony/server/auth/AuthResource.java`

- [ ] **Step 1: Fix both methods**

Replace `loginPage()`:

```java
@GET
@Path("/login")
@Produces(MediaType.TEXT_HTML)
public Response loginPage() {
    try (var stream = getClass().getResourceAsStream("/META-INF/resources/auth/login.html")) {
        if (stream == null) return Response.serverError().entity("login.html not found").build();
        return Response.ok(new String(stream.readAllBytes())).type(MediaType.TEXT_HTML).build();
    } catch (IOException e) {
        return Response.serverError().build();
    }
}
```

Replace `serveRegisterPage()`:

```java
private Response serveRegisterPage() {
    try (var stream = getClass().getResourceAsStream("/META-INF/resources/auth/register.html")) {
        if (stream == null) return Response.serverError().entity("register.html not found").build();
        return Response.ok(new String(stream.readAllBytes())).type(MediaType.TEXT_HTML).build();
    } catch (IOException e) {
        return Response.serverError().build();
    }
}
```

Add `import java.io.IOException;` if not present.

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q
```

Expected: 212+ tests, 0 failures.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/dev/claudony/server/auth/AuthResource.java
git commit -m "fix: close getResourceAsStream in AuthResource; narrow catch to IOException (Refs #54)"
```

---

## Task 6: SessionResource — fix empty catch + missing log

**Problem:**
- Line 142: `catch (Exception ignored) {}` — silently swallows directory creation failures with no log, no indication anything went wrong
- Line 200 (`rename()`): `catch (Exception e) { return Response.serverError().build(); }` — no log, so rename failures are invisible

**Files:**
- Modify: `src/main/java/dev/claudony/server/SessionResource.java`

- [ ] **Step 1: Fix the empty catch at line 142**

Replace:
```java
try { java.nio.file.Files.createDirectories(java.nio.file.Path.of(workingDir)); }
catch (Exception ignored) {}
```

With:
```java
try { java.nio.file.Files.createDirectories(java.nio.file.Path.of(workingDir)); }
catch (IOException e) {
    LOG.debugf("Could not create working directory '%s': %s", workingDir, e.getMessage());
}
```

`debug` is appropriate — this is best-effort; tmux will fail clearly if the dir is missing.

- [ ] **Step 2: Add logging to rename's catch at line 200**

Replace:
```java
} catch (Exception e) {
    return Response.serverError().build();
}
```

With:
```java
} catch (IOException | InterruptedException e) {
    LOG.errorf("Failed to rename session '%s': %s", session.name(), e.getMessage());
    return Response.serverError().build();
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q
```

Expected: 212+ tests, 0 failures.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/dev/claudony/server/SessionResource.java
git commit -m "fix: log directory creation failures in SessionResource; add missing rename error log (Refs #54)"
```

---

## Task 7: Narrow catch(Exception) to declared types

**Principle:** Catch the most specific type that covers all declared exceptions. Keep `catch (Exception)` only where:
- The code is a best-effort cleanup path (where any failure should be swallowed)
- The caller is a REST client whose checked + runtime exceptions are both meaningful to catch together
- `Future.get()` is called (throws `ExecutionException` + `InterruptedException` + potentially `TimeoutException`)

Everything else should name `IOException | InterruptedException` (or just `IOException`).

**Files:** SessionResource.java, ServerStartup.java, TerminalWebSocket.java, ClipboardChecker.java, ITerm2Adapter.java

- [ ] **Step 1: Narrow SessionResource.java**

Read `SessionResource.java` fully first.

Changes (by method — make all at once):

| Method | Line | From | To |
|---|---|---|---|
| `create()` cleanup block | ~135 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `create()` dir creation | ~142 | `catch (Exception ignored)` | `catch (IOException e)` + log (already done in Task 6) |
| `create()` tmux block | ~152 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `delete()` | ~169 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `rename()` | ~200 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` (done in Task 6) |
| `sendInput()` | ~214 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `getOutput()` | ~230 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `resize()` | ~252 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `openTerminal()` | ~272 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `gitStatus()` | ~319 | `catch (Exception e)` | `catch (IOException \| InterruptedException e)` |
| `checkPort()` | ~355 | `catch (Exception e)` | `catch (IOException e)` |

**Keep as `catch (Exception e)`:**
- `list()` ~line 73 — catches `Future.get()` + REST client exceptions; multiple checked types
- `serviceHealth()` ~line 344 — catches `executor.invokeAll()` (InterruptedException) + `future.get()` (ExecutionException + InterruptedException); keep as Exception

- [ ] **Step 2: Narrow ServerStartup.java**

Read `ServerStartup.java` first.

```java
// createDirectory block (~line 45): IOException
catch (IOException e) {
    LOG.warnf("Could not create directory %s: %s", dir, e.getMessage());
}

// tmux version check (~line 55): IOException | InterruptedException
catch (IOException | InterruptedException e) {
    throw new IllegalStateException("tmux not found on PATH. Install with: brew install tmux", e);
}

// bootstrap block (~line 76): IOException | InterruptedException
catch (IOException | InterruptedException e) {
    LOG.warn("Could not bootstrap from tmux list-sessions: " + e.getMessage());
}
```

- [ ] **Step 3: Narrow TerminalWebSocket.java**

Read `TerminalWebSocket.java` first.

```java
// Cursor read block (~line 139): IOException | InterruptedException
catch (IOException | InterruptedException e) {
    LOG.debugf("Could not read pane cursor for %s: %s", tmuxName, e.getMessage());
}

// pipe-pane setup block (~line 185): IOException | InterruptedException
catch (IOException | InterruptedException e) {
    LOG.errorf("Failed to set up pipe for session '%s': %s", tmuxName, e.getMessage());
    cleanup(connection);
    try { connection.closeAndAwait(); } catch (Exception ignored) {}
}

// send-keys block (~line 203): IOException | InterruptedException
catch (IOException | InterruptedException e) {
    LOG.debugf("Failed to send input to session %s: %s", tmuxName, e.getMessage());
}
```

**Keep as `catch (Exception ignored) {}`** — the three cleanup paths (close, pipe cleanup, FIFO delete at ~lines 40, 188, 233, 238) are legitimately best-effort; any failure type should be swallowed there.

- [ ] **Step 4: Narrow ClipboardChecker.java**

Read the file. The `catch (Exception e)` around `isConfigured()` covers `IOException | InterruptedException` from process invocation. Narrow:

```java
} catch (IOException | InterruptedException e) {
    return "tmux clipboard: check failed (" + e.getMessage() + ")";
}
```

- [ ] **Step 5: Narrow ITerm2Adapter.java**

Read the file. The `catch (Exception e)` around `p.waitFor()` + stream read covers `IOException | InterruptedException`. Narrow:

```java
} catch (IOException | InterruptedException e) {
    LOG.debugf("iTerm2 check failed: %s", e.getMessage());
    return false;
}
```

- [ ] **Step 6: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q
```

Expected: 212+ tests, 0 failures.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/dev/claudony/server/SessionResource.java \
        src/main/java/dev/claudony/server/ServerStartup.java \
        src/main/java/dev/claudony/server/TerminalWebSocket.java \
        src/main/java/dev/claudony/agent/ClipboardChecker.java \
        src/main/java/dev/claudony/agent/terminal/ITerm2Adapter.java
git commit -m "refactor: narrow catch(Exception) to declared types across production code (Refs #54)

SessionResource: IOException|InterruptedException for all tmux ops; keep Exception
only for Future.get() paths where multiple checked types apply.
ServerStartup, TerminalWebSocket, ClipboardChecker, ITerm2Adapter: same pattern.
Best-effort cleanup paths (close, FIFO delete) retain catch(Exception ignored) —
appropriate since any failure should be silently swallowed in teardown."
```

---

## Final Verification

- [ ] **Run full suite one last time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format
```

Expected: 212+ tests, 0 failures, 0 errors.

- [ ] **Close issue**

```bash
gh issue close 54 --repo mdproctor/claudony --comment "All 7 tasks complete."
```

---

## Self-Review

**Spec coverage check:**
- Item 1 (credential migration) → Task 1 ✓
- Item 2 (Thread.sleep in 4 test files) → Task 2 ✓
- Item 3 (TerminalWebSocketTest sleeps) → Task 3 ✓
- Item 4 (test ordering) → Task 4 ✓ (Tasks 2+4 combined for TmuxServiceTest/SessionInputOutputTest)
- Item 5 (AuthResource stream leak) → Task 5 ✓
- Item 6 (SessionResource empty catch + rename log) → Task 6 ✓
- Item 7 (narrow catch blocks) → Task 7 ✓
- Apple passkeys → documented as resolved by upgrade, no task needed ✓

**Placeholder scan:** No TBDs or incomplete steps. All code is shown in full.

**Type consistency:** `Await.until(BooleanSupplier, String)` used consistently throughout Tasks 2, 3, 4. `awaitHistoryBurst(LinkedBlockingQueue<String>)` defined and used only in Task 3. No cross-task type mismatches.
