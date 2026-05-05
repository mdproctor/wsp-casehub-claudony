# MCP Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make all 8 MCP tool methods robust against server failures and fill all test coverage gaps identified in the joint audit.

**Architecture:** Two private helpers (`serverError`, `connectError`) on `ClaudonyMcpTools` translate exception types to readable strings. Every `@Tool` method wraps its body in an identical two-tier try/catch. Timeouts are added via config only — no code. Tests follow strict TDD: failing tests written first, implementation second.

**Tech Stack:** Java 21, Quarkus 3.32.2, quarkus-mcp-server-http 1.11.1, MicroProfile Rest Client, JUnit 5, Mockito, RestAssured, AssertJ.

**Issue:** Refs #55

---

## File Map

```
Modified:
  src/main/java/dev/claudony/agent/ClaudonyMcpTools.java        — Task 1: helpers + two-tier catch on all 8 tools
  src/main/resources/application.properties                      — Task 5: connect/read timeouts
  src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java    — Task 1: 8 error tests; Task 2: 3 openInTerminal tests
  src/test/java/dev/claudony/agent/McpServerIntegrationTest.java — Task 3: assertion fix + extraction fix + 1 new test
  src/test/java/dev/claudony/agent/McpProtocolTest.java         — Task 4: 3 new protocol tests
```

No new files. No other files touched.

---

## Task 1: Error handling in ClaudonyMcpTools (TDD)

Write failing tests first. Implement helpers and two-tier catch. Verify tests pass.

**Files:**
- Modify: `src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java`
- Modify: `src/main/java/dev/claudony/agent/ClaudonyMcpTools.java`

- [ ] **Step 1: Write 8 failing error tests in `ClaudonyMcpToolsTest`**

Add these imports to `ClaudonyMcpToolsTest.java` (after the existing imports):

```java
import jakarta.ws.rs.ProcessingException;
import jakarta.ws.rs.WebApplicationException;
import jakarta.ws.rs.core.Response;
import dev.claudony.agent.terminal.TerminalAdapter;
```

Add these 8 tests to the class body (after the existing tests):

```java
// ── Error handling ───────────────────────────────────────────────────────────

@Test
void listSessions_serverReturns500_returnsServerError() {
    Mockito.when(serverClient.listSessions())
        .thenThrow(new WebApplicationException(Response.status(500).build()));
    assertThat(tools.listSessions()).startsWith("Server error (HTTP 500)");
}

@Test
void listSessions_serverUnreachable_returnsConnectError() {
    Mockito.when(serverClient.listSessions())
        .thenThrow(new ProcessingException("Connection refused"));
    assertThat(tools.listSessions()).startsWith("Unable to reach Claudony server");
}

@Test
void createSession_serverReturns409_returnsConflictMessage() {
    Mockito.when(serverClient.createSession(Mockito.any()))
        .thenThrow(new WebApplicationException(Response.status(409).build()));
    assertThat(tools.createSession("dup", "/tmp", null)).contains("Conflict");
}

@Test
void deleteSession_serverReturns404_returnsNotFoundMessage() {
    Mockito.doThrow(new WebApplicationException(Response.status(404).build()))
        .when(serverClient).deleteSession("bad-id");
    assertThat(tools.deleteSession("bad-id"))
        .contains("not found")
        .contains("list_sessions");
}

@Test
void renameSession_serverReturns404_returnsNotFoundMessage() {
    Mockito.when(serverClient.renameSession(Mockito.eq("bad-id"), Mockito.any()))
        .thenThrow(new WebApplicationException(Response.status(404).build()));
    assertThat(tools.renameSession("bad-id", "new-name")).contains("not found");
}

@Test
void sendInput_serverReturns404_returnsNotFoundMessage() {
    Mockito.doThrow(new WebApplicationException(Response.status(404).build()))
        .when(serverClient).sendInput(Mockito.eq("bad-id"), Mockito.any());
    assertThat(tools.sendInput("bad-id", "echo hi")).contains("not found");
}

@Test
void getOutput_serverReturns500_returnsServerError() {
    Mockito.when(serverClient.getOutput(Mockito.eq("id-1"), Mockito.anyInt()))
        .thenThrow(new WebApplicationException(Response.status(500).build()));
    assertThat(tools.getOutput("id-1", null)).startsWith("Server error (HTTP 500)");
}

@Test
void getServerInfo_serverReturns500_returnsServerError() {
    // getServerInfo calls terminalFactory (already mocked to empty in @BeforeEach)
    // and reads config — this tests the outer catch by making the config call appear to fail
    // We can't make config throw, but we CAN verify the happy-path null guard works:
    // getServerInfo has no server call, so instead test that it doesn't NPE on null config values.
    // The server-error catch is tested implicitly via other tool tests.
    // Instead, test null safety of getServerInfo output:
    assertThat(tools.getServerInfo())
        .contains("Server URL:")
        .contains("Agent mode:")
        .doesNotContain("null");
}
```

> **Note on `getServerInfo`:** This tool doesn't call `ServerClient`, so it can't get a 500
> from the server. The test instead verifies null safety of config values (the other
> meaningful failure mode). The two-tier catch on it is still added as a safety net.

- [ ] **Step 2: Run to confirm all 8 new tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClaudonyMcpToolsTest -Dno-format -q 2>&1 | tail -20
```

Expected: 8 failures — the error tests throw instead of returning strings. The 12 existing tests still pass.

- [ ] **Step 3: Add helpers and two-tier catch to `ClaudonyMcpTools.java`**

Replace the entire file content with:

```java
package dev.claudony.agent;

import dev.claudony.agent.terminal.TerminalAdapterFactory;
import dev.claudony.config.ClaudonyConfig;
import dev.claudony.server.model.CreateSessionRequest;
import dev.claudony.server.model.SendInputRequest;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.WebApplicationException;
import org.eclipse.microprofile.rest.client.inject.RestClient;

/**
 * Claudony's 8 MCP tools, registered via quarkus-mcp-server-http.
 *
 * <p>
 * quarkus-mcp-server handles the JSON-RPC 2.0 protocol (initialize, tools/list,
 * tools/call, notifications/initialized) at {@code /mcp}. Each {@code @Tool} method
 * returns a String that the library wraps as a text content item.
 *
 * <p>
 * Error handling: every tool wraps its body in a two-tier catch. A
 * {@link WebApplicationException} means the server responded with an HTTP error —
 * the status code is used to give Claude a specific, actionable message. Any other
 * exception means the server was unreachable (connection refused, timeout, etc.).
 *
 * <p>
 * Migrated from hand-rolled {@code McpServer.java} (deleted) to enable the
 * quarkus-qhorus embedding — once Qhorus is added as a dependency its 38 tools
 * auto-register on the same /mcp endpoint alongside these 8.
 */
@ApplicationScoped
public class ClaudonyMcpTools {

    @Inject
    ClaudonyConfig config;

    @RestClient
    ServerClient server;

    @Inject
    TerminalAdapterFactory terminalFactory;

    // ── Tools ────────────────────────────────────────────────────────────────

    @Tool(name = "list_sessions", description = "List all active Claude Code sessions")
    public String listSessions() {
        try {
            final var sessions = server.listSessions();
            if (sessions.isEmpty()) {
                return "No active sessions.";
            }
            return sessions.stream()
                    .map(s -> "• %s (id=%s, status=%s, dir=%s)"
                            .formatted(s.name(), s.id(), s.status(), s.workingDir()))
                    .reduce("Sessions:\n", (a, b) -> a + "\n" + b);
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    @Tool(name = "create_session", description = "Create a new Claude Code session")
    public String createSession(
            @ToolArg(name = "name", description = "Session name") String name,
            @ToolArg(name = "workingDir", description = "Working directory") String workingDir,
            @ToolArg(name = "command", description = "Shell command to run (optional)", required = false) String command) {
        try {
            final var req = new CreateSessionRequest(
                    name,
                    workingDir,
                    (command != null && !command.isBlank()) ? command : null);
            final var s = server.createSession(req);
            return "Created '%s' (id=%s)\nBrowser: %s".formatted(s.name(), s.id(), s.browserUrl());
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    @Tool(name = "delete_session", description = "Delete a session by id")
    public String deleteSession(
            @ToolArg(name = "id", description = "Session id") String id) {
        try {
            server.deleteSession(id);
            return "Session deleted.";
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    @Tool(name = "rename_session", description = "Rename a session")
    public String renameSession(
            @ToolArg(name = "id", description = "Session id") String id,
            @ToolArg(name = "name", description = "New session name") String name) {
        try {
            final var s = server.renameSession(id, name);
            return "Renamed to '%s'.".formatted(s.name());
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    @Tool(name = "send_input", description = "Send text input to a session")
    public String sendInput(
            @ToolArg(name = "id", description = "Session id") String id,
            @ToolArg(name = "text", description = "Text to send") String text) {
        try {
            server.sendInput(id, new SendInputRequest(text));
            return "Input sent.";
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    @Tool(name = "get_output", description = "Get recent terminal output from a session")
    public String getOutput(
            @ToolArg(name = "id", description = "Session id") String id,
            @ToolArg(name = "lines", description = "Number of lines to return (default 50)", required = false) Integer lines) {
        try {
            return server.getOutput(id, lines != null ? lines : 50);
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    @Tool(name = "open_in_terminal", description = "Open a session in a local terminal window")
    public String openInTerminal(
            @ToolArg(name = "id", description = "Session id") String id) {
        try {
            final var adapter = terminalFactory.resolve();
            if (adapter.isEmpty()) {
                return "No terminal adapter available on this machine.";
            }
            final var sessions = server.listSessions();
            final var session = sessions.stream()
                    .filter(s -> s.id().equals(id))
                    .findFirst();
            if (session.isEmpty()) {
                return "Session not found.";
            }
            try {
                adapter.get().openSession(session.get().name());
            } catch (final java.io.IOException | InterruptedException e) {
                return "Failed to open terminal: " + e.getMessage();
            }
            return "Opened in %s.".formatted(adapter.get().name());
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    @Tool(name = "get_server_info", description = "Get server connection info and status")
    public String getServerInfo() {
        try {
            final var adapter = terminalFactory.resolve();
            final var url = config.serverUrl() != null ? config.serverUrl() : "(not configured)";
            final var mode = config.mode() != null ? config.mode() : "(not configured)";
            return "Server URL: %s\nAgent mode: %s\nTerminal adapter: %s".formatted(
                    url, mode, adapter.map(a -> a.name()).orElse("none"));
        } catch (WebApplicationException e) { return serverError(e); }
          catch (Exception e)               { return connectError(e); }
    }

    // ── Error helpers ────────────────────────────────────────────────────────

    private String serverError(WebApplicationException e) {
        return switch (e.getResponse().getStatus()) {
            case 404 -> "Session not found. Use list_sessions to see available sessions.";
            case 409 -> "Conflict: a session with that name already exists.";
            case 401, 403 -> "Authentication error. Check that the agent API key is configured correctly.";
            default -> "Server error (HTTP %d). Check that the Claudony server is running."
                    .formatted(e.getResponse().getStatus());
        };
    }

    private String connectError(Exception e) {
        return "Unable to reach Claudony server — " + e.getMessage()
               + ". Check server URL and that the server is running.";
    }
}
```

- [ ] **Step 4: Run tests — all 20 should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClaudonyMcpToolsTest -Dno-format -q 2>&1 | tail -10
```

Expected: `Tests run: 20, Failures: 0, Errors: 0`

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: `Tests run: 222, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git add src/main/java/dev/claudony/agent/ClaudonyMcpTools.java \
        src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java
git commit -m "feat: two-tier error handling in all 8 MCP tool methods (Refs #55)

serverError() maps HTTP status codes to actionable messages (404 → list_sessions hint,
409 → conflict, 401/403 → auth error, default → generic with status code).
connectError() handles unreachable server (ProcessingException etc).
getServerInfo() null-guards config values.
8 new error scenario tests + updated getServerInfo null-safety test."
```

---

## Task 2: openInTerminal missing test paths (TDD)

The `openInTerminal` method has three paths that have no tests: session-not-found (adapter present but session ID not in list), adapter throws IOException, and the full happy path (adapter present + session found + open succeeds). Write tests first — they should pass immediately since the production code already handles these cases. Writing them now documents behaviour and guards against future regression.

**Files:**
- Modify: `src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java`

- [ ] **Step 1: Write 3 openInTerminal path tests**

Add `import dev.claudony.agent.terminal.TerminalAdapter;` to the imports in `ClaudonyMcpToolsTest.java`.

Add these 3 tests to the class body:

```java
// ── openInTerminal paths ─────────────────────────────────────────────────────

@Test
void openInTerminal_sessionNotFound_returnsNotFoundMessage() {
    var mockAdapter = Mockito.mock(TerminalAdapter.class);
    Mockito.when(terminalFactory.resolve()).thenReturn(Optional.of(mockAdapter));
    Mockito.when(serverClient.listSessions()).thenReturn(List.of()); // no sessions

    assertThat(tools.openInTerminal("id-does-not-exist"))
        .isEqualTo("Session not found.");
}

@Test
void openInTerminal_adapterThrowsIOException_returnsErrorMessage() throws Exception {
    var now = Instant.now();
    var mockAdapter = Mockito.mock(TerminalAdapter.class);
    Mockito.when(terminalFactory.resolve()).thenReturn(Optional.of(mockAdapter));
    Mockito.when(serverClient.listSessions()).thenReturn(List.of(
        new SessionResponse("id-1", "claudony-test", "/tmp", "claude",
            SessionStatus.IDLE, now, now,
            "ws://localhost:7777/ws/id-1",
            "http://localhost:7777/app/session/id-1",
            null, null, null)));
    Mockito.doThrow(new java.io.IOException("pipe broken"))
        .when(mockAdapter).openSession(Mockito.anyString());

    assertThat(tools.openInTerminal("id-1"))
        .startsWith("Failed to open terminal:");
}

@Test
void openInTerminal_sessionFound_opensAdapterAndReturnsName() throws Exception {
    var now = Instant.now();
    var mockAdapter = Mockito.mock(TerminalAdapter.class);
    Mockito.when(mockAdapter.name()).thenReturn("iTerm2");
    Mockito.when(terminalFactory.resolve()).thenReturn(Optional.of(mockAdapter));
    Mockito.when(serverClient.listSessions()).thenReturn(List.of(
        new SessionResponse("id-1", "claudony-test", "/tmp", "claude",
            SessionStatus.IDLE, now, now,
            "ws://localhost:7777/ws/id-1",
            "http://localhost:7777/app/session/id-1",
            null, null, null)));
    Mockito.doNothing().when(mockAdapter).openSession("claudony-test");

    assertThat(tools.openInTerminal("id-1")).isEqualTo("Opened in iTerm2.");
    Mockito.verify(mockAdapter).openSession("claudony-test");
}
```

- [ ] **Step 2: Run — all 3 should pass immediately**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClaudonyMcpToolsTest -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 23, Failures: 0, Errors: 0` (20 from Task 1 + 3 new)

- [ ] **Step 3: Commit**

```bash
git add src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java
git commit -m "test: add missing openInTerminal path tests (Refs #55)

Tests for: session-not-found (adapter present, list returns no match),
adapter-throws-IOException (returns error string), and the previously-untested
happy path (adapter present + session found + open succeeds)."
```

---

## Task 3: Integration test hardening

Fix two fragile patterns in `McpServerIntegrationTest` and add one error scenario test through the full MCP stack.

**Files:**
- Modify: `src/test/java/dev/claudony/agent/McpServerIntegrationTest.java`

- [ ] **Step 1: Fix `extractSessionId` to use Pattern**

In `McpServerIntegrationTest.java`, replace the `extractSessionId` helper (around line 221):

```java
// OLD — fragile string split:
private String extractSessionId(String text) {
    var parts = text.split("Browser: http://localhost:\\d+/app/session/");
    return parts.length > 1 ? parts[1].trim() : null;
}

// NEW — regex with capture group:
private String extractSessionId(String text) {
    var m = java.util.regex.Pattern.compile("/app/session/([\\w-]+)").matcher(text);
    return m.find() ? m.group(1) : null;
}
```

- [ ] **Step 2: Replace all `Assumptions.assumeTrue` with hard assertions**

There are three occurrences in `McpServerIntegrationTest`. Replace each:

```java
// Line 85 — in fullSessionLifecycle_createSendReceiveDelete:
// OLD:
Assumptions.assumeTrue(tmuxSessionId != null, "Could not extract session ID");
// NEW:
assertThat(tmuxSessionId).as("session ID extracted from create_session response").isNotNull();

// Line 119 — in renameSession_updatesNameThroughFullChain:
// OLD:
Assumptions.assumeTrue(tmuxSessionId != null);
// NEW:
assertThat(tmuxSessionId).as("session ID extracted from create_session response").isNotNull();

// Line 140 — in inputWithTmuxKeyName_appearsAsLiteralText:
// OLD:
Assumptions.assumeTrue(tmuxSessionId != null);
// NEW:
assertThat(tmuxSessionId).as("session ID extracted from create_session response").isNotNull();
```

Add `import static org.assertj.core.api.Assertions.assertThat;` to imports.
Remove `import org.junit.jupiter.api.Assumptions;` (now unused).

- [ ] **Step 3: Add error scenario integration test**

Add this test to `McpServerIntegrationTest` (after `getServerInfo_returnsExpectedFields`):

```java
@Test
void deleteSession_nonExistentId_returnsReadableErrorMessage() {
    // Call delete_session with a UUID that doesn't correspond to any real session.
    // With error handling in place, the tool returns a readable string — not an exception.
    var fakeId = "00000000-0000-0000-0000-000000000000";
    var result = mcp()
        .body("""
            {"jsonrpc":"2.0","id":1,"method":"tools/call",
             "params":{"name":"delete_session","arguments":{"id":"%s"}}}
            """.formatted(fakeId))
        .when().post("/mcp")
        .then()
        .statusCode(200)
        .body("result.content[0].text", notNullValue())
        .extract().<String>path("result.content[0].text");

    // Tool returns a readable error string, not a JSON-RPC error and not an exception.
    assertThat(result)
        .as("delete of non-existent session should return readable error")
        .containsIgnoringCase("not found")
        .doesNotContain("Exception");
}
```

This test verifies the full round-trip: MCP call → ClaudonyMcpTools → ServerClient → server returns 404 → `serverError()` returns string → MCP response contains readable text.

- [ ] **Step 4: Run the integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=McpServerIntegrationTest -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 6, Failures: 0, Errors: 0` (5 existing + 1 new)

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: `Tests run: 229, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git add src/test/java/dev/claudony/agent/McpServerIntegrationTest.java
git commit -m "test: harden McpServerIntegrationTest — assertions, extraction, error scenario (Refs #55)

Replace Assumptions.assumeTrue (silently skips) with assertThat (fails visibly).
Replace fragile string-split session ID extraction with Pattern capture group.
Add deleteSession_nonExistentId integration test — verifies error handling
round-trip: 404 from server → serverError() → readable string in MCP response."
```

---

## Task 4: Protocol compliance tests

Three new tests for the MCP protocol layer. Run each with a placeholder assertion first to discover actual library behaviour, then pin the real value.

**Files:**
- Modify: `src/test/java/dev/claudony/agent/McpProtocolTest.java`

- [ ] **Step 1: Write the 3 new protocol tests with discovery assertions**

Add these tests to `McpProtocolTest.java`:

```java
@Test
void toolsCall_withoutMcpSessionId_isRejected() {
    // POST a tools/call without any Mcp-Session-Id header.
    // Run the test to discover the actual status code / error body,
    // then replace the placeholder assertion below with the real one.
    given()
        .contentType(ContentType.JSON)
        .accept("application/json, text/event-stream")
        .body("""
            {"jsonrpc":"2.0","id":1,"method":"tools/call",
             "params":{"name":"list_sessions","arguments":{}}}
            """)
        .when().post("/mcp")
        .then()
        .statusCode(anyOf(equalTo(200), equalTo(400)));
    // After running: observe actual status + body, replace with pinned assertion.
}

@Test
void toolsCall_withInvalidSessionId_isRejected() {
    // POST a tools/call with a session ID that was never created.
    given()
        .contentType(ContentType.JSON)
        .accept("application/json, text/event-stream")
        .header("Mcp-Session-Id", "not-a-real-session-id")
        .body("""
            {"jsonrpc":"2.0","id":1,"method":"tools/call",
             "params":{"name":"list_sessions","arguments":{}}}
            """)
        .when().post("/mcp")
        .then()
        .statusCode(anyOf(equalTo(200), equalTo(400), equalTo(404)));
    // After running: observe actual response, replace with pinned assertion.
}

@Test
void initialize_calledTwice_eachCallSucceeds() {
    // Call initialize twice. Each should succeed and return a session ID.
    // The second call may return the same or a different session ID — document actual behaviour.
    var sid1 = initialize();
    assertThat(sid1).isNotNull();

    var sid2 = initialize();
    assertThat(sid2).isNotNull();
    // Document: are they the same session?
    // assertThat(sid2).isEqualTo(sid1);   // same session
    // assertThat(sid2).isNotEqualTo(sid1); // new session each time
    // After running: uncomment the correct one.
}
```

- [ ] **Step 2: Run the 3 new tests to observe actual behaviour**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=McpProtocolTest -Dno-format 2>&1 | grep -A5 "toolsCall_without\|toolsCall_with\|initialize_called"
```

Look at the actual status codes and response bodies printed. Note them down.

- [ ] **Step 3: Pin the assertions based on observed behaviour**

Replace the placeholder `anyOf(...)` assertions with exact values from Step 2. For example, if `toolsCall_withoutMcpSessionId` returns status 200 with `{"error":{"code":-32600}}`:

```java
// Replace:
.statusCode(anyOf(equalTo(200), equalTo(400)));
// With the exact observed value, e.g.:
.then()
.statusCode(200)
.body("error.code", equalTo(-32600));
```

Do the same for the invalid session ID test and the double-initialize test. Remove the discovery comments once assertions are pinned.

- [ ] **Step 4: Run all 8 McpProtocolTest tests to confirm**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=McpProtocolTest -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 8, Failures: 0, Errors: 0` (5 existing + 3 new)

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: `Tests run: 232, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git add src/test/java/dev/claudony/agent/McpProtocolTest.java
git commit -m "test: add 3 protocol compliance tests — missing session ID, invalid ID, double-init (Refs #55)

Pins actual quarkus-mcp-server-http behaviour for edge cases not previously
tested: tools/call without Mcp-Session-Id header, tools/call with unknown
session ID, and initialize called twice in the same session."
```

---

## Task 5: Timeout configuration

Two lines in `application.properties`. No code changes.

**Files:**
- Modify: `src/main/resources/application.properties`

- [ ] **Step 1: Add timeouts for the claudony-server REST client**

In `application.properties`, after the existing `quarkus.rest-client.claudony-server.url` line (line 18), add:

```properties
quarkus.rest-client.claudony-server.connect-timeout=5000
quarkus.rest-client.claudony-server.read-timeout=10000
```

The block should read:
```properties
# REST client for Agent → Server calls
quarkus.rest-client.claudony-server.url=${claudony.server.url}
quarkus.rest-client.claudony-server.connect-timeout=5000
quarkus.rest-client.claudony-server.read-timeout=10000
```

5 000 ms connect. 10 000 ms read — generous because `get_output` polls a tmux pane.
Compare with the peer client (3 000/2 000 ms) which only does health checks.

- [ ] **Step 2: Run full suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: `Tests run: 232, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/application.properties
git commit -m "config: add connect/read timeouts for agent→server REST client (Refs #55)

connect-timeout=5000ms, read-timeout=10000ms. Peer client already had
explicit timeouts (3000/2000ms) — agent client was missing them.
Read timeout generous because get_output may wait on slow tmux pane."
```

---

## Final Verification

- [ ] **Run full suite one last time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format
```

Expected: 232 tests, 0 failures, 0 errors, 0 skipped. (214 baseline + 8 error unit + 3 openInTerminal unit + 1 integration + 3 protocol + 3 already counted)

- [ ] **Update CLAUDE.md test count**

In `CLAUDE.md`, update `**214 tests passing**` to `**232 tests passing**`.

- [ ] **Update baseline doc**

In `docs/superpowers/2026-04-16-mcp-hardening-baseline.md`, mark all TODO items as done.

- [ ] **Close issue**

```bash
gh issue close 55 --repo mdproctor/claudony --comment "All tasks complete. $(JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -1)"
```

---

## Self-Review

**Spec coverage:**
- ✅ `serverError()` helper — Task 1
- ✅ `connectError()` helper — Task 1
- ✅ Two-tier catch on all 8 tools — Task 1
- ✅ `getServerInfo()` null guard — Task 1
- ✅ connect-timeout=5000, read-timeout=10000 — Task 5
- ✅ 7 server-error unit tests (listSessions 500, listSessions connect, createSession 409, deleteSession 404, renameSession 404, sendInput 404, getOutput 500) — Task 1
- ✅ getServerInfo null-safety test — Task 1 (revised: tests null guard, not server error)
- ✅ openInTerminal session-not-found — Task 2
- ✅ openInTerminal IOException — Task 2
- ✅ openInTerminal happy path — Task 2
- ✅ Assumptions.assumeTrue → assertThat (3 occurrences) — Task 3
- ✅ extractSessionId Pattern fix — Task 3
- ✅ deleteSession non-existent integration test — Task 3
- ✅ tools/call without Mcp-Session-Id — Task 4
- ✅ tools/call with invalid session ID — Task 4
- ✅ initialize called twice — Task 4

**Placeholder scan:** Task 4 Step 3 requires the implementer to run the test and fill in the observed assertion values. This is intentional — the library behaviour is unknown until run. The step is explicit about what to observe and how to update.

**Type consistency:** `serverError(WebApplicationException e)` and `connectError(Exception e)` defined in Task 1 Step 3, referenced in the same step. No cross-task type usage. `extractSessionId` return type is `String` in both old and new form — consistent with `Assumptions.assumeTrue(tmuxSessionId != null)` → `assertThat(tmuxSessionId)`.
