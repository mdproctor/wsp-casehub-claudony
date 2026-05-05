# E2E Testing and Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix a silent bug in `TmuxService.sendKeys`, fill integration test gaps across all existing test classes, and add real Claude E2E smoke tests behind a `-Pe2e` Maven profile.

**Architecture:** TDD throughout — write failing test first, fix/implement, verify green, commit. Tasks are ordered by dependency: bug fix first (Tasks 1-2), then REST gaps (3), then MCP unit/integration gaps (4-5), then WebSocket/startup gaps (6-7), then CI plumbing (8), then Claude E2E (9). Each task is self-contained and leaves the suite green.

**Tech Stack:** Java 21, Quarkus 3.9.5, JUnit 5, RestAssured, Mockito, Quarkus WebSockets Next, tmux, claude CLI

---

## Files Changed

| File | Action |
|---|---|
| `src/main/java/dev/remotecc/server/TmuxService.java` | Fix `-l` flag in `sendKeys()` |
| `src/main/java/dev/remotecc/server/ServerStartup.java` | Make `bootstrapRegistry()` package-private |
| `src/test/java/dev/remotecc/server/TmuxServiceTest.java` | Add sendKeys literal test |
| `src/test/java/dev/remotecc/server/SessionInputOutputTest.java` | Add tmux key name REST round-trip test |
| `src/test/java/dev/remotecc/server/SessionResourceTest.java` | Add rename + resize tests |
| `src/test/java/dev/remotecc/server/ServerStartupTest.java` | Add explicit bootstrap test |
| `src/test/java/dev/remotecc/server/TerminalWebSocketTest.java` | Add history replay + concurrent connections tests |
| `src/test/java/dev/remotecc/agent/McpServerTest.java` | Add 5 unit tests |
| `src/test/java/dev/remotecc/agent/McpServerIntegrationTest.java` | Add 5 integration/protocol tests |
| `src/test/java/dev/remotecc/e2e/ClaudeE2ETest.java` | Create — 2 E2E tests |
| `pom.xml` | Exclude `*E2ETest` from default surefire; add `-Pe2e` profile |

---

## Task 1: Fix TmuxService.sendKeys — the `-l` flag bug

**Files:**
- Test first: `src/test/java/dev/remotecc/server/TmuxServiceTest.java`
- Fix: `src/main/java/dev/remotecc/server/TmuxService.java`

**Background:** `TmuxService.sendKeys()` currently passes text to `tmux send-keys` without the `-l` (literal) flag. Without `-l`, tmux interprets words like `"Escape"`, `"Enter"`, `"Space"` as key names and fires the corresponding key instead of sending the literal text. `TerminalWebSocket` already uses `-l` correctly — this inconsistency means REST input and WebSocket input behave differently.

- [ ] **Step 1: Add failing test to `TmuxServiceTest`**

Add this test as `@Order(6)` at the end of the class. It sends text containing the tmux key name `"Escape"` and asserts the literal word appears in pane output.

```java
@Test
@Order(6)
void sendKeysLiteralModeDoesNotInterpretTmuxKeyNames() throws Exception {
    tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "bash");
    Thread.sleep(300);
    // "Escape" is a tmux key name. Without -l, tmux fires the Escape key
    // instead of sending the literal text, so "Escape" never appears in output.
    tmux.sendKeys(TEST_SESSION, "echo Escape marker\n");
    Thread.sleep(300);
    var output = tmux.capturePane(TEST_SESSION, 20);
    assertTrue(output.contains("Escape marker"),
        "Expected literal 'Escape marker' in output — tmux may have fired " +
        "the Escape key (missing -l flag). Got: " + output);
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TmuxServiceTest#sendKeysLiteralModeDoesNotInterpretTmuxKeyNames
```

Expected: FAIL — "Escape marker" not found (Escape key was fired instead).

- [ ] **Step 3: Fix `TmuxService.sendKeys()`**

Replace the `sendKeys` method body. Change from `text, ""` (no `-l`, spurious empty arg) to `-l, text`:

```java
public void sendKeys(String sessionName, String text)
        throws IOException, InterruptedException {
    var p = new ProcessBuilder("tmux", "send-keys", "-t", sessionName, "-l", text)
            .redirectErrorStream(true).start();
    p.getInputStream().transferTo(OutputStream.nullOutputStream());
    p.waitFor();
}
```

- [ ] **Step 4: Run test to confirm it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TmuxServiceTest
```

Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/remotecc/server/TmuxService.java \
        src/test/java/dev/remotecc/server/TmuxServiceTest.java
git commit -m "fix: use -l flag in TmuxService.sendKeys to prevent tmux key name interpretation"
```

---

## Task 2: Prove the fix works through the REST API

**Files:**
- `src/test/java/dev/remotecc/server/SessionInputOutputTest.java`

This tests the full chain: HTTP request → `SessionResource` → `TmuxService.sendKeys()` → tmux → output. Without this, a future refactor could break the wiring and `TmuxServiceTest` would still pass.

- [ ] **Step 1: Add failing test to `SessionInputOutputTest`**

Add as `@Order(5)` at the end of the class:

```java
@Test
@Order(5)
void sendInputWithTmuxKeyNameViaRestPreservesLiteralText() throws Exception {
    var sessionId = given().contentType("application/json")
        .body("{\"name\":\"test-io-keyname\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}")
        .when().post("/api/sessions")
        .then().statusCode(201).extract().path("id");
    Thread.sleep(300);

    given().contentType("application/json")
        .body("{\"text\":\"echo Escape marker\\n\"}")
        .when().post("/api/sessions/" + sessionId + "/input")
        .then().statusCode(204);

    Thread.sleep(300);
    given().when().get("/api/sessions/" + sessionId + "/output?lines=20")
        .then()
        .statusCode(200)
        .body(containsString("Escape marker"));
}
```

- [ ] **Step 2: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionInputOutputTest
```

Expected: All 5 tests PASS. If `@Order(5)` fails, the bug fix in Task 1 wasn't wired through — check `SessionResource.sendInput()` calls `tmux.sendKeys()`.

- [ ] **Step 3: Commit**

```bash
git add src/test/java/dev/remotecc/server/SessionInputOutputTest.java
git commit -m "test: prove sendKeys -l fix works end-to-end through REST input endpoint"
```

---

## Task 3: Fill SessionResourceTest gaps — rename and resize

**Files:**
- `src/test/java/dev/remotecc/server/SessionResourceTest.java`

The `PATCH /rename` and `POST /resize` endpoints exist in `SessionResource.java` but have zero test coverage.

- [ ] **Step 1: Add rename and resize tests**

Add both as `@Order(6)` and `@Order(7)` at the end of `SessionResourceTest`:

```java
@Test
@Order(6)
void renameSessionReturns200WithNewName() throws Exception {
    var id = given().contentType("application/json")
        .body("{\"name\":\"test-rename\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}")
        .when().post("/api/sessions")
        .then().statusCode(201).extract().path("id");

    given().when().patch("/api/sessions/" + id + "/rename?name=renamed")
        .then()
        .statusCode(200)
        .body("name", equalTo("remotecc-renamed"));

    // Verify registry reflects the new name
    given().when().get("/api/sessions/" + id)
        .then().statusCode(200).body("name", equalTo("remotecc-renamed"));
}

@Test
@Order(7)
void resizeSessionReturns204() throws Exception {
    var id = given().contentType("application/json")
        .body("{\"name\":\"test-resize\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}")
        .when().post("/api/sessions")
        .then().statusCode(201).extract().path("id");

    given().when().post("/api/sessions/" + id + "/resize?cols=120&rows=40")
        .then().statusCode(204);
}
```

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionResourceTest
```

Expected: All 7 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add src/test/java/dev/remotecc/server/SessionResourceTest.java
git commit -m "test: add rename and resize endpoint coverage to SessionResourceTest"
```

---

## Task 4: Fill McpServerTest unit gaps — 5 new mocked tests

**Files:**
- `src/test/java/dev/remotecc/agent/McpServerTest.java`

All existing 5 tests use `@InjectMock` for `ServerClient` and `TerminalAdapterFactory`. The `@BeforeEach` resets mocks and sets `terminalFactory.resolve()` to return `Optional.empty()`. Maintain that pattern.

Add the following imports if not already present:
```java
import dev.remotecc.server.model.SendInputRequest;
```

- [ ] **Step 1: Add 5 new tests**

Add as `@Order(6)` through `@Order(10)` at the end of `McpServerTest`:

```java
@Test
@Order(6)
void serverErrorOnToolCallReturnsGracefulMcpError() {
    Mockito.when(serverClient.createSession(Mockito.any()))
        .thenThrow(new RuntimeException("tmux: bad session name"));

    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":6,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"create_session\"," +
              "\"arguments\":{\"name\":\"bad\",\"workingDir\":\"/tmp\"}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("error.code", equalTo(-32603))
        .body("error.message", containsString("Internal error"));
}

@Test
@Order(7)
void sendInputPassesTextLiterallyToServer() {
    Mockito.doNothing().when(serverClient)
        .sendInput(Mockito.anyString(), Mockito.any(SendInputRequest.class));

    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":7,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"send_input\"," +
              "\"arguments\":{\"id\":\"id-1\",\"text\":\"echo Escape marker\\n\"}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.content[0].text", equalTo("Input sent."));

    // Verify the text was passed through unchanged — not interpreted
    Mockito.verify(serverClient).sendInput(
        Mockito.eq("id-1"),
        Mockito.argThat(req -> "echo Escape marker\n".equals(req.text())));
}

@Test
@Order(8)
void renameSessionToolReturnsNewName() {
    var now = Instant.now();
    Mockito.when(serverClient.renameSession(
            Mockito.eq("id-1"), Mockito.eq("newname")))
        .thenReturn(new SessionResponse("id-1", "remotecc-newname", "/tmp", "claude",
            SessionStatus.IDLE, now, now,
            "ws://localhost:7777/ws/id-1",
            "http://localhost:7777/app/session/id-1"));

    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":8,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"rename_session\"," +
              "\"arguments\":{\"id\":\"id-1\",\"name\":\"newname\"}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.content[0].text", containsString("remotecc-newname"));
}

@Test
@Order(9)
void openInTerminalWithNoAdapterReturnsHelpfulMessage() {
    // terminalFactory.resolve() already returns Optional.empty() from @BeforeEach
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":9,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"open_in_terminal\"," +
              "\"arguments\":{\"id\":\"id-1\"}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.content[0].text",
              equalTo("No terminal adapter available on this machine."));
}

@Test
@Order(10)
void getServerInfoReturnsExpectedFields() {
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":10,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"get_server_info\",\"arguments\":{}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.content[0].text", containsString("Server URL:"))
        .body("result.content[0].text", containsString("Agent mode:"))
        .body("result.content[0].text", containsString("Terminal adapter: none"));
}
```

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=McpServerTest
```

Expected: All 10 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add src/test/java/dev/remotecc/agent/McpServerTest.java
git commit -m "test: add 5 unit tests to McpServerTest — error handling, send_input, rename, open_in_terminal, get_server_info"
```

---

## Task 5: Fill McpServerIntegrationTest gaps — 5 new protocol tests

**Files:**
- `src/test/java/dev/remotecc/agent/McpServerIntegrationTest.java`

These use real tmux and the real `ServerClient` looped back to the same Quarkus test instance. Each new test that creates a session must clean it up. Add after existing `@Order(7)`.

Add import if missing: `import static org.hamcrest.Matchers.hasItems;`

- [ ] **Step 1: Add 5 new tests**

```java
@Test
@Order(8)
void toolsListReturnsAll8ToolsWithCorrectNames() {
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":8,\"method\":\"tools/list\",\"params\":{}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.tools", hasSize(8))
        .body("result.tools.name", hasItems(
            "list_sessions", "create_session", "delete_session",
            "rename_session", "send_input", "get_output",
            "open_in_terminal", "get_server_info"));
}

@Test
@Order(9)
void unknownToolNameReturnsJsonRpcError() {
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":9,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"nonexistent_tool\",\"arguments\":{}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("error.code", equalTo(-32603))
        .body("error.message", containsString("Internal error"));
}

@Test
@Order(10)
void renameSessionViaMcpUpdatesNameThroughFullChain() {
    // Create session
    var createText = given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":10,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"create_session\"," +
              "\"arguments\":{\"name\":\"mcp-rename-test\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}}}")
        .when().post("/mcp")
        .then().statusCode(200).extract().<String>path("result.content[0].text");

    var parts = createText.split("Browser: http://localhost:\\d+/app/session/");
    Assumptions.assumeTrue(parts.length > 1, "Could not extract session ID");
    var sid = parts[1].trim();

    // Rename
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":11,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"rename_session\"," +
              "\"arguments\":{\"id\":\"" + sid + "\",\"name\":\"mcp-renamed\"}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.content[0].text", containsString("remotecc-mcp-renamed"));

    // Cleanup
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":12,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"delete_session\"," +
              "\"arguments\":{\"id\":\"" + sid + "\"}}}")
        .when().post("/mcp");
}

@Test
@Order(11)
void fullLifecycleWithTmuxKeyNameInInputPreservesLiteralText() throws Exception {
    // Create
    var createText = given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":13,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"create_session\"," +
              "\"arguments\":{\"name\":\"mcp-keyname-test\",\"workingDir\":\"/tmp\",\"command\":\"bash\"}}}")
        .when().post("/mcp")
        .then().statusCode(200).extract().<String>path("result.content[0].text");

    var parts = createText.split("Browser: http://localhost:\\d+/app/session/");
    Assumptions.assumeTrue(parts.length > 1, "Could not extract session ID");
    var sid = parts[1].trim();
    Thread.sleep(300);

    // Send input containing "Escape" — without -l fix, this would fire Escape key
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":14,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"send_input\"," +
              "\"arguments\":{\"id\":\"" + sid + "\",\"text\":\"echo Escape literal-marker\\n\"}}}")
        .when().post("/mcp")
        .then().statusCode(200).body("result.content[0].text", equalTo("Input sent."));

    Thread.sleep(300);

    // Get output — must contain literal "Escape", not a fired Escape key
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":15,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"get_output\"," +
              "\"arguments\":{\"id\":\"" + sid + "\",\"lines\":20}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.content[0].text", containsString("Escape literal-marker"));

    // Cleanup
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":16,\"method\":\"tools/call\"," +
              "\"params\":{\"name\":\"delete_session\"," +
              "\"arguments\":{\"id\":\"" + sid + "\"}}}")
        .when().post("/mcp");
}

@Test
@Order(12)
void fullMcpHandshakeSequenceAsClaudeWouldSendIt() {
    // Step 1: initialize
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":17,\"method\":\"initialize\"," +
              "\"params\":{\"protocolVersion\":\"2024-11-05\"," +
              "\"capabilities\":{},\"clientInfo\":{\"name\":\"claude\",\"version\":\"1\"}}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.protocolVersion", equalTo("2024-11-05"))
        .body("result.serverInfo.name", equalTo("remotecc"));

    // Step 2: notifications/initialized (server returns 204, no body)
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"method\":\"notifications/initialized\",\"params\":{}}")
        .when().post("/mcp")
        .then()
        .statusCode(204);

    // Step 3: tools/list (what Claude does after handshake)
    given().contentType("application/json")
        .body("{\"jsonrpc\":\"2.0\",\"id\":18,\"method\":\"tools/list\",\"params\":{}}")
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.tools", hasSize(8));
}
```

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=McpServerIntegrationTest
```

Expected: All 12 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add src/test/java/dev/remotecc/agent/McpServerIntegrationTest.java
git commit -m "test: add 5 integration tests to McpServerIntegrationTest — tools/list, error handling, rename, key name lifecycle, handshake sequence"
```

---

## Task 6: Fill TerminalWebSocketTest gaps — history replay and concurrent connections

**Files:**
- `src/test/java/dev/remotecc/server/TerminalWebSocketTest.java`

**Background on concurrent connections:** `tmux pipe-pane` only allows one active pipe per pane. When a second WebSocket connects, its `pipe-pane` call replaces the first. The first connection's FIFO reader gets EOF (pipe closed) and stops receiving new output. The second connection gets all subsequent output. Neither connection should crash.

Add these imports to `TerminalWebSocketTest` if missing:
```java
import java.util.concurrent.CountDownLatch;
```

- [ ] **Step 1: Add history replay test**

Add after the existing two tests:

```java
@Test
void historyIsReplayedOnReconnect() throws Exception {
    var received = new LinkedBlockingQueue<String>();
    var container = ContainerProvider.getWebSocketContainer();
    var cec = ClientEndpointConfig.Builder.create().build();

    // First connection: send a command to generate visible output
    var session1 = container.connectToServer(new Endpoint() {
        @Override public void onOpen(Session s, EndpointConfig c) {
            s.addMessageHandler(String.class, received::offer);
        }
    }, cec, URI.create(wsBaseUri + "ws-test-id"));
    Thread.sleep(500);
    session1.getBasicRemote().sendText("echo history-replay-marker\n");

    // Wait for marker to appear in live output
    var live = new StringBuilder();
    long deadline = System.currentTimeMillis() + 3000;
    while (System.currentTimeMillis() < deadline) {
        var chunk = received.poll(200, TimeUnit.MILLISECONDS);
        if (chunk != null) {
            live.append(chunk);
            if (live.toString().contains("history-replay-marker")) break;
        }
    }
    assertTrue(live.toString().contains("history-replay-marker"),
        "Command must appear in live output before testing history replay. Got: " + live);
    session1.close();
    Thread.sleep(300);

    // Second connection: collect everything sent on connect (the history replay)
    var history = new LinkedBlockingQueue<String>();
    var session2 = container.connectToServer(new Endpoint() {
        @Override public void onOpen(Session s, EndpointConfig c) {
            s.addMessageHandler(String.class, history::offer);
        }
    }, cec, URI.create(wsBaseUri + "ws-test-id"));

    var historyText = new StringBuilder();
    deadline = System.currentTimeMillis() + 2000;
    while (System.currentTimeMillis() < deadline) {
        var chunk = history.poll(100, TimeUnit.MILLISECONDS);
        if (chunk != null) historyText.append(chunk);
    }
    session2.close();

    assertTrue(historyText.toString().contains("history-replay-marker"),
        "Reconnect history must contain 'history-replay-marker'. Got: " + historyText);
}
```

- [ ] **Step 2: Add concurrent connections test**

```java
@Test
void concurrentConnectionsToSameSessionDoNotCrash() throws Exception {
    var container = ContainerProvider.getWebSocketContainer();
    var cec = ClientEndpointConfig.Builder.create().build();
    var opened1 = new CountDownLatch(1);
    var opened2 = new CountDownLatch(1);
    var received2 = new LinkedBlockingQueue<String>();

    // First connection
    var session1 = container.connectToServer(new Endpoint() {
        @Override public void onOpen(Session s, EndpointConfig c) {
            s.addMessageHandler(String.class, msg -> {});
            opened1.countDown();
        }
    }, cec, URI.create(wsBaseUri + "ws-test-id"));
    assertTrue(opened1.await(3, TimeUnit.SECONDS), "First connection should open");
    Thread.sleep(300);

    // Second connection: its pipe-pane replaces first connection's pipe
    var session2 = container.connectToServer(new Endpoint() {
        @Override public void onOpen(Session s, EndpointConfig c) {
            s.addMessageHandler(String.class, received2::offer);
            opened2.countDown();
        }
    }, cec, URI.create(wsBaseUri + "ws-test-id"));
    assertTrue(opened2.await(3, TimeUnit.SECONDS), "Second connection should open");
    Thread.sleep(500);

    // Second connection holds the active pipe — it should receive output
    session2.getBasicRemote().sendText("echo concurrent-test-marker\n");
    var sb = new StringBuilder();
    long deadline = System.currentTimeMillis() + 3000;
    while (System.currentTimeMillis() < deadline) {
        var chunk = received2.poll(200, TimeUnit.MILLISECONDS);
        if (chunk != null) {
            sb.append(chunk);
            if (sb.toString().contains("concurrent-test-marker")) break;
        }
    }
    assertTrue(sb.toString().contains("concurrent-test-marker"),
        "Second connection should receive output after both are connected. Got: " + sb);

    // Both connections close cleanly (no crash)
    session1.close();
    session2.close();
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TerminalWebSocketTest
```

Expected: All 4 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src/test/java/dev/remotecc/server/TerminalWebSocketTest.java
git commit -m "test: add history replay and concurrent connections tests to TerminalWebSocketTest"
```

---

## Task 7: Fill ServerStartupTest — explicit bootstrap test

**Files:**
- `src/main/java/dev/remotecc/server/ServerStartup.java` (make `bootstrapRegistry` package-private)
- `src/test/java/dev/remotecc/server/ServerStartupTest.java`

**Background:** `bootstrapRegistry()` is currently `private`. The startup event fires at Quarkus start — before any `@BeforeEach` can pre-create sessions. Making it package-private lets the test call it directly after pre-creating a tmux session, proving bootstrap discovers sessions created post-startup.

- [ ] **Step 1: Make `bootstrapRegistry()` package-private in `ServerStartup.java`**

Change `private void bootstrapRegistry()` to `void bootstrapRegistry()` (remove `private`):

```java
void bootstrapRegistry() {   // was: private void bootstrapRegistry()
    try {
        var names = tmux.listSessionNames();
        var prefix = config.tmuxPrefix();
        int count = 0;
        for (var name : names) {
            if (!name.startsWith(prefix)) continue;
            var now = Instant.now();
            registry.register(new Session(
                    UUID.randomUUID().toString(), name,
                    "unknown", config.claudeCommand(),
                    SessionStatus.IDLE, now, now));
            count++;
        }
        LOG.infof("Bootstrapped %d existing session(s) from tmux", count);
    } catch (Exception e) {
        LOG.warn("Could not bootstrap from tmux list-sessions: " + e.getMessage());
    }
}
```

- [ ] **Step 2: Add test to `ServerStartupTest`**

Add `@Inject ServerStartup serverStartup;` field and this test:

```java
@Inject
ServerStartup serverStartup;

@Test
void bootstrapRegistryPicksUpTmuxSessionsCreatedAfterStartup() throws Exception {
    var sessionName = "remotecc-bootstrap-test-" + System.currentTimeMillis();
    tmux.createSession(sessionName, System.getProperty("user.home"), "bash");

    try {
        // Verify NOT in registry yet (created after Quarkus startup)
        var beforeNames = registry.all().stream().map(s -> s.name()).toList();
        assertFalse(beforeNames.contains(sessionName),
            "Session created post-startup should not be in registry before bootstrap. Registry: " + beforeNames);

        // Run bootstrap — it's idempotent (register uses put, won't duplicate)
        serverStartup.bootstrapRegistry();

        // Now it should be in the registry
        var afterNames = registry.all().stream().map(s -> s.name()).toList();
        assertTrue(afterNames.contains(sessionName),
            "Session should be in registry after bootstrap. Registry: " + afterNames);
    } finally {
        // Clean up registry entry
        registry.all().stream()
            .filter(s -> s.name().equals(sessionName))
            .map(s -> s.id())
            .findFirst()
            .ifPresent(registry::remove);
        if (tmux.sessionExists(sessionName)) tmux.killSession(sessionName);
    }
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ServerStartupTest
```

Expected: Both tests PASS.

- [ ] **Step 4: Run the full suite to confirm nothing broken**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All ~82 tests PASS (64 existing + ~18 new). Note the count — confirm it in the output.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/remotecc/server/ServerStartup.java \
        src/test/java/dev/remotecc/server/ServerStartupTest.java
git commit -m "test: add explicit bootstrap test to ServerStartupTest; make bootstrapRegistry package-private"
```

---

## Task 8: Add Maven `-Pe2e` profile

**Files:**
- `pom.xml`

- [ ] **Step 1: Exclude `*E2ETest` from the default surefire configuration**

In `pom.xml`, find the existing `maven-surefire-plugin` configuration block and add the `<excludes>` element:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>${surefire-plugin.version}</version>
  <configuration>
    <systemPropertyVariables>
      <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
    </systemPropertyVariables>
    <excludes>
      <exclude>**/*E2ETest.java</exclude>
    </excludes>
  </configuration>
</plugin>
```

- [ ] **Step 2: Add the `-Pe2e` profile inside `<profiles>`**

Add after the existing `<profile id="native">`:

```xml
<profile>
  <id>e2e</id>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${surefire-plugin.version}</version>
        <configuration>
          <systemPropertyVariables>
            <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
          </systemPropertyVariables>
          <includes>
            <include>**/*E2ETest.java</include>
          </includes>
          <excludes combine.self="override"/>
        </configuration>
      </plugin>
    </plugins>
  </build>
</profile>
```

- [ ] **Step 3: Verify default `mvn test` still works and excludes E2E**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: Same test count as before. No `ClaudeE2ETest` in output (it doesn't exist yet — that's fine).

- [ ] **Step 4: Commit**

```bash
git add pom.xml
git commit -m "build: exclude *E2ETest from default surefire run; add -Pe2e profile"
```

---

## Task 9: Create ClaudeE2ETest — real claude CLI invocations

**Files:**
- Create: `src/test/java/dev/remotecc/e2e/ClaudeE2ETest.java`

**Before writing the test — verify claude CLI MCP config mechanism:**

- [ ] **Step 1: Check available claude CLI flags**

```bash
claude --help 2>&1 | grep -iE "config|mcp"
claude mcp --help 2>&1
```

Look for one of:
- `--mcp-config <file>` — pass a JSON file with MCP server config
- `--config-dir <dir>` — use alternate config directory
- `CLAUDE_CONFIG_DIR` env var

Also check what format the MCP server entry uses:
```bash
cat ~/.claude/settings.json 2>/dev/null | python3 -m json.tool | grep -A5 mcp
```

Use whichever mechanism is available. The test below assumes `CLAUDE_CONFIG_DIR` env var pointing to a temp dir containing `settings.json`. Adjust if the CLI uses a different mechanism.

- [ ] **Step 2: Create the test class**

Create `src/test/java/dev/remotecc/e2e/ClaudeE2ETest.java`:

```java
package dev.remotecc.e2e;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable;
import java.nio.file.*;
import java.util.concurrent.TimeUnit;
import static org.junit.jupiter.api.Assertions.*;

/**
 * End-to-end tests that invoke the real claude CLI against the running MCP server.
 *
 * These tests are EXCLUDED from the default mvn test run.
 * Run with: ANTHROPIC_API_KEY=... mvn test -Pe2e
 *
 * Assertions are on SIDE EFFECTS (tmux state) not on Claude's text output.
 * Claude's exact words are non-deterministic; what it did to the system is not.
 *
 * The Quarkus test instance runs on port 8081. The Agent's ServerClient
 * loops back to the same instance (configured in test application.properties).
 * The MCP endpoint at http://localhost:8081/mcp is fully functional.
 */
@QuarkusTest
@EnabledIfEnvironmentVariable(named = "ANTHROPIC_API_KEY", matches = ".+")
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class ClaudeE2ETest {

    private static Path configDir;

    @BeforeAll
    static void setupMcpConfig() throws Exception {
        // Create a temp config directory with MCP server pointing to test instance
        configDir = Files.createTempDirectory("remotecc-e2e-config-");
        var settings = """
            {
              "mcpServers": {
                "remotecc": {
                  "type": "http",
                  "url": "http://localhost:8081/mcp"
                }
              }
            }
            """;
        // Try settings.json (Claude Code default config file name)
        Files.writeString(configDir.resolve("settings.json"), settings);
    }

    @AfterAll
    static void cleanupConfig() throws Exception {
        if (configDir != null) {
            Files.walk(configDir)
                .sorted(java.util.Comparator.reverseOrder())
                .forEach(p -> p.toFile().delete());
        }
    }

    private int runClaude(String prompt) throws Exception {
        var pb = new ProcessBuilder(
            "claude", "--dangerously-skip-permissions", "-p", prompt)
            .redirectErrorStream(true);
        pb.environment().put("CLAUDE_CONFIG_DIR", configDir.toString());
        // Propagate API key from test environment
        var apiKey = System.getenv("ANTHROPIC_API_KEY");
        if (apiKey != null) pb.environment().put("ANTHROPIC_API_KEY", apiKey);
        var process = pb.start();
        // Drain stdout/stderr to avoid blocking
        process.getInputStream().transferTo(java.io.OutputStream.nullOutputStream());
        return process.waitFor(60, TimeUnit.SECONDS) ? process.exitValue() : -1;
    }

    private boolean tmuxSessionExists(String name) throws Exception {
        return new ProcessBuilder("tmux", "has-session", "-t", name)
            .redirectErrorStream(true).start().waitFor() == 0;
    }

    @Test
    @Order(1)
    void claudeCanDiscoverRemoteCCMcpTools() throws Exception {
        // Ask Claude to list its MCP tools — if it can connect and enumerate tools,
        // the MCP handshake (initialize → tools/list) is working end-to-end.
        // We assert the process exits 0, not Claude's exact words.
        var exitCode = runClaude(
            "What tools do you have available from the remotecc MCP server? " +
            "Just list their names briefly.");
        assertEquals(0, exitCode,
            "claude process should exit 0 when it can connect to MCP server and list tools");
    }

    @Test
    @Order(2)
    void claudeCanCreateAndDeleteSessionViaMcp() throws Exception {
        var sessionName = "remotecc-e2e-test";

        try {
            // Ask Claude to create a session — assert on tmux state, not Claude's words
            var exitCode = runClaude(
                "Using the remotecc MCP tools, create a new session called 'e2e-test' " +
                "in the /tmp directory running bash.");
            assertEquals(0, exitCode, "claude process should exit 0");

            // Side effect: tmux session must exist
            assertTrue(tmuxSessionExists(sessionName),
                "tmux session '" + sessionName + "' should exist after Claude created it. " +
                "If not, Claude did not call create_session or the tool failed.");

        } finally {
            // Clean up regardless of test outcome
            if (tmuxSessionExists(sessionName)) {
                new ProcessBuilder("tmux", "kill-session", "-t", sessionName)
                    .redirectErrorStream(true).start().waitFor();
            }
        }
    }
}
```

**Important:** If `CLAUDE_CONFIG_DIR` is not the correct env var or `settings.json` is not the right filename, adjust based on what `claude --help` showed in Step 1. The alternative is to use `claude mcp add` before the test and `claude mcp remove` after:
```java
// Alternative: temporarily add/remove MCP server from global config
new ProcessBuilder("claude", "mcp", "add", "--transport", "http",
    "remotecc-test", "http://localhost:8081/mcp").start().waitFor();
// ... run test ...
new ProcessBuilder("claude", "mcp", "remove", "remotecc-test").start().waitFor();
```

- [ ] **Step 3: Run E2E tests**

```bash
ANTHROPIC_API_KEY="your-key-here" JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e
```

Expected:
- Test 1 exits 0 — Claude connected to MCP and listed tools
- Test 2 exits 0 AND `tmux has-session -t remotecc-e2e-test` returns 0 during the test

If tests fail, check:
- `claude --help` output to find the correct config mechanism
- `~/.claude/settings.json` to confirm the MCP server format
- That port 8081 is actually the Quarkus test port (check test output for "Quarkus started on port")

- [ ] **Step 4: Commit**

```bash
git add src/test/java/dev/remotecc/e2e/ClaudeE2ETest.java
git commit -m "test: add ClaudeE2ETest — real claude CLI invocations against MCP server (-Pe2e)"
```

---

## Task 10: Final verification

- [ ] **Step 1: Run the full default test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected output: approximately 82-84 tests, all PASS. Confirm `ClaudeE2ETest` is NOT in the output.

- [ ] **Step 2: Confirm test count in output**

Look for the surefire summary line:
```
Tests run: 84, Failures: 0, Errors: 0, Skipped: 0
```

Record the actual count. If count is less than expected, grep for skipped tests and investigate.

- [ ] **Step 3: Run E2E suite separately**

```bash
ANTHROPIC_API_KEY="..." JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e
```

Expected: 2 tests run, both PASS.

- [ ] **Step 4: Final commit message**

All individual tasks were committed. No additional commit needed. The branch is ready.
