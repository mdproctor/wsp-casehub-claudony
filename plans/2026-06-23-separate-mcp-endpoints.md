# Separate Claudony and Qhorus MCP Endpoints — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split the unified `/mcp` endpoint into two named MCP servers — Claudony session tools at `/mcp`, Qhorus agent mesh tools at `/qhorus`.

**Architecture:** `@McpServer("qhorus")` annotation on Qhorus tool classes binds them to a named server. Default config shipped via `META-INF/microprofile-config.properties` in the Qhorus JAR ensures the endpoint exists without consumer configuration. Claudony's `ClaudonyMcpTools` stays on the default server (unannotated by design).

**Tech Stack:** Quarkus 3.32.2, quarkus-mcp-server 1.11.1 (`@McpServer` named server support), SmallRye Config (`microprofile-config.properties` ordinal 100)

## Global Constraints

- Java 21 API surface (release=21), compiled on Java 26: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Use `mvn` not `./mvnw`
- Qhorus project at `/Users/mdproctor/claude/casehub/qhorus`
- Claudony project at `/Users/mdproctor/claude/casehub/claudony`
- Every commit references an issue: `Refs casehubio/qhorus#NNN` or `Refs casehubio/claudony#105`
- Tests: 587 total in Claudony (all passing), must remain green after changes

---

### Task 1: Add `@McpServer("qhorus")` and default config to Qhorus

This task is in the **Qhorus** project (`/Users/mdproctor/claude/casehub/qhorus`). It must be completed and installed locally before Task 2.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add `@McpServer("qhorus")` and import
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — add `@McpServer("qhorus")` and import
- Create: `runtime/src/main/resources/META-INF/microprofile-config.properties` — default endpoint config
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/ToolErrorHandlingTest.java` — update 3 `.post("/mcp")` calls to `.post("/qhorus")`

**Interfaces:**
- Consumes: `io.quarkiverse.mcp.server.McpServer` annotation (already on classpath via quarkus-mcp-server dependency)
- Produces: Qhorus tools bound to named server `"qhorus"` at default path `/qhorus`; consumers see tools at `/qhorus` instead of `/mcp`

- [ ] **Step 1: Add `@McpServer("qhorus")` to `QhorusMcpTools`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`, add the import and annotation:

```java
// Add import (alongside existing io.quarkiverse.mcp.server.Tool import)
import io.quarkiverse.mcp.server.McpServer;
```

Add `@McpServer("qhorus")` to the class annotations, before `@UnlessBuildProperty`:

```java
@McpServer("qhorus")
@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)
@WrapBusinessError({ IllegalArgumentException.class, IllegalStateException.class })
@ApplicationScoped
public class QhorusMcpTools extends QhorusMcpToolsBase {
```

- [ ] **Step 2: Add `@McpServer("qhorus")` to `ReactiveQhorusMcpTools`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`, add the import and annotation:

```java
// Add import (alongside existing io.quarkiverse.mcp.server.Tool import)
import io.quarkiverse.mcp.server.McpServer;
```

Add `@McpServer("qhorus")` to the class annotations, before `@IfBuildProperty`:

```java
@McpServer("qhorus")
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@WrapBusinessError({ IllegalArgumentException.class, IllegalStateException.class })
@ApplicationScoped
public class ReactiveQhorusMcpTools extends QhorusMcpToolsBase {
```

- [ ] **Step 3: Create default config**

Create `runtime/src/main/resources/META-INF/microprofile-config.properties`:

```properties
# Default MCP server config for Qhorus named server.
# Consumers override via their own application.properties (ordinal 250 > 100).
quarkus.mcp.server.qhorus.http.root-path=/qhorus
quarkus.mcp.server.qhorus.server-info.name=qhorus
quarkus.mcp.server.qhorus.tools.page-size=0
```

- [ ] **Step 4: Update `ToolErrorHandlingTest` — change `/mcp` to `/qhorus`**

In `runtime/src/test/java/io/casehub/qhorus/mcp/ToolErrorHandlingTest.java`, update the `initMcpSession()` helper method (line 80):

```java
    private String initMcpSession() {
        return given()
                .contentType(ContentType.JSON)
                .accept("application/json, text/event-stream")
                .body("""
                        {"jsonrpc":"2.0","id":0,"method":"initialize",
                         "params":{"protocolVersion":"2024-11-05","capabilities":{},
                                   "clientInfo":{"name":"test","version":"1"}}}
                        """)
                .when().post("/qhorus")
                .then().statusCode(200)
                .extract().header("Mcp-Session-Id");
    }
```

Update `pauseChannel_nonExistentChannel_returnsIsErrorTrue` (line 103):

```java
                .when().post("/qhorus")
```

Update `sendMessage_nonExistentChannel_returnsIsErrorTrue` (line 126):

```java
                .when().post("/qhorus")
```

- [ ] **Step 5: Run Qhorus tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`

Expected: All tests pass. `ToolErrorHandlingTest` hits `/qhorus` and gets Qhorus tools.

- [ ] **Step 6: Install Qhorus SNAPSHOT locally**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`

This makes the updated Qhorus SNAPSHOT available in the local Maven repo for Claudony to consume.

- [ ] **Step 7: Commit in Qhorus**

```
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java \
  runtime/src/main/resources/META-INF/microprofile-config.properties \
  runtime/src/test/java/io/casehub/qhorus/mcp/ToolErrorHandlingTest.java
```

Commit message: `feat: scope MCP tools to named server "qhorus" via @McpServer`

---

### Task 2: Update Claudony `application.properties` and `McpServerIntegrationTest`

This task is in the **Claudony** project (`/Users/mdproctor/claude/casehub/claudony`). Depends on Task 1 (Qhorus SNAPSHOT installed locally).

**Files:**
- Modify: `app/src/main/resources/application.properties` — remove `page-size=0`
- Modify: `app/src/test/java/io/casehub/claudony/agent/McpServerIntegrationTest.java` — update all assertions

**Interfaces:**
- Consumes: Qhorus SNAPSHOT with `@McpServer("qhorus")` (from Task 1)
- Produces: Claudony tests verify endpoint separation — 8 tools at `/mcp`, Qhorus tools at `/qhorus`, no cross-leakage

- [ ] **Step 1: Write the updated `fullHandshakeSequence` test (before changing config)**

In `app/src/test/java/io/casehub/claudony/agent/McpServerIntegrationTest.java`, update `fullHandshakeSequence_asClaudeWouldSendIt` (the `tools/list` assertion near line 238):

Change:
```java
            .body("result.tools.size()", greaterThanOrEqualTo(8))
            .body("result.tools.name", hasItems(
                "list_sessions", "create_session", "delete_session",
                "rename_session", "send_input", "get_output",
                "open_in_terminal", "get_server_info"));
```

To:
```java
            // Exactly 8 Claudony tools — no Qhorus tools on the default server
            .body("result.tools.size()", equalTo(8))
            .body("result.tools.name", hasItems(
                "list_sessions", "create_session", "delete_session",
                "rename_session", "send_input", "get_output",
                "open_in_terminal", "get_server_info"))
            .body("result.tools.name", not(hasItems("register", "send_message", "check_messages")));
```

- [ ] **Step 2: Rewrite `toolsList_includesQhorusTools` as `qhorusToolsAvailableAtSeparateEndpoint`**

Replace the entire `toolsList_includesQhorusTools` method (lines 284-316) and the stale `// Phase 8` comment block (lines 319-322):

```java
    @Test
    void qhorusToolsAvailableAtSeparateEndpoint() {
        // Qhorus tools are on a separate named server — needs its own initialize handshake
        var initResponse = given()
            .contentType(ContentType.JSON)
            .accept("application/json, text/event-stream")
            .body("""
                {"jsonrpc":"2.0","id":1,"method":"initialize",
                 "params":{"protocolVersion":"2024-11-05","capabilities":{},
                           "clientInfo":{"name":"test","version":"1"}}}
                """)
            .when().post("/qhorus")
            .then().statusCode(200)
            .body("result.serverInfo.name", equalTo("qhorus"))
            .extract().response();

        var sid = initResponse.header("Mcp-Session-Id");

        given()
            .contentType(ContentType.JSON)
            .accept("application/json, text/event-stream")
            .header("Mcp-Session-Id", sid)
            .body("""
                {"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
                """)
            .when().post("/qhorus")
            .then()
            .statusCode(200)
            // Key Qhorus tools present — flexible count, no hardcoded total
            .body("result.tools.name", hasItems(
                "register", "send_message", "check_messages",
                "create_channel", "list_channels"))
            .body("result.tools.size()", greaterThanOrEqualTo(40))
            // No Claudony tools on the Qhorus server
            .body("result.tools.name", not(hasItems("list_sessions", "create_session")));
    }
```

- [ ] **Step 3: Remove `page-size=0` from `application.properties`**

In `app/src/main/resources/application.properties`, remove lines 116-118:

```properties
# Disable tools/list pagination — default cap is 50, which silently drops tools beyond that.
# Remove when Claudony and Qhorus tools move to separate MCP endpoints (see #105).
quarkus.mcp.server.tools.page-size=0
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

Expected: All 587 tests pass. `fullHandshakeSequence_asClaudeWouldSendIt` asserts exactly 8 tools at `/mcp`. `qhorusToolsAvailableAtSeparateEndpoint` verifies Qhorus tools at `/qhorus` with flexible count.

- [ ] **Step 5: Commit**

Stage: `McpServerIntegrationTest.java`, `application.properties`

Commit message: `feat(#105): separate Claudony and Qhorus MCP endpoints`

---

### Task 3: Update CLAUDE.md and ARC42STORIES.MD

This task is in the **Claudony** project. No code changes — documentation only.

**Files:**
- Modify: `CLAUDE.md` — Key URLs, Qhorus tool-count paragraph, configuration
- Modify: `ARC42STORIES.MD` — 4 sections referencing unified `/mcp`

- [ ] **Step 1: Update CLAUDE.md Key URLs**

Add two new entries to the Key URLs section:

```
- Qhorus MCP endpoint (Server): `http://localhost:7777/qhorus`
- Qhorus MCP endpoint (Agent): `http://localhost:7778/qhorus`
```

- [ ] **Step 2: Rewrite the Qhorus tool-count paragraph in CLAUDE.md**

Replace the paragraph starting with `**Qhorus tool count:**` with:

```
**MCP endpoint separation (#105):** Claudony session tools (8) are served at `/mcp` (default server). Qhorus agent mesh tools are served at `/qhorus` (named server `"qhorus"`, configured via Qhorus's `META-INF/microprofile-config.properties`). `McpServerIntegrationTest.qhorusToolsAvailableAtSeparateEndpoint` verifies key Qhorus tools are present at `/qhorus` with a flexible count (`greaterThanOrEqualTo(40)`) — no hardcoded total. `fullHandshakeSequence_asClaudeWouldSendIt` asserts exactly 8 tools at `/mcp`. Override the Qhorus endpoint path via `quarkus.mcp.server.qhorus.http.root-path` in `application.properties`.
```

- [ ] **Step 3: Update ARC42STORIES.MD — C2 context diagram (line 97) and §6 Scenario 2 (line 256)**

Line 97 — change:
```
  Rel(controller, claudony, "POST /mcp — JSON-RPC session management + Qhorus mesh tools")
```
To:
```
  Rel(controller, claudony, "POST /mcp (session tools) + POST /qhorus (agent mesh tools)")
```

Line 256 — change:
```
Agent (Claude session) → POST /mcp → ClaudonyMcpTools → QhorusMcpTools.sendMessage()
```
To:
```
Agent (Claude session) → POST /qhorus → ReactiveQhorusMcpTools.sendMessage()
```

- [ ] **Step 4: Update ARC42STORIES.MD — C5 Layer Impact (line 578)**

Change:
```
| L4 Agent Mode + MCP | Low — Qhorus tools join existing `/mcp` endpoint |
```

To:
```
| L4 Agent Mode + MCP | Low — Qhorus tools served at separate `/qhorus` endpoint (#105) |
```

- [ ] **Step 5: Update ARC42STORIES.MD — C4 "What this delivers" (line 550)**

Change:
```
`claudony.mode=agent` activates `POST /mcp` with 8 session management tools (JSON-RPC over HTTP — GraalVM-native compatible, no stdio subprocess). iTerm2 integration for co-located session launching. A controller Claude can now create, read, resize, and command any session via MCP.
```

To:
```
`claudony.mode=agent` activates `POST /mcp` with 8 session management tools (JSON-RPC over HTTP — GraalVM-native compatible, no stdio subprocess). Qhorus agent mesh tools are served at a separate `POST /qhorus` endpoint (#105). iTerm2 integration for co-located session launching. A controller Claude can now create, read, resize, and command any session via MCP, and reach agent mesh tools at `/qhorus`.
```

- [ ] **Step 6: Update ARC42STORIES.MD — §4 Solution Strategy MCP line (line 72)**

The line reads:
```
- HTTP JSON-RPC for MCP (`POST /mcp`) is non-negotiable — native image requires no stdio subprocess
```

Update to:
```
- HTTP JSON-RPC for MCP (`POST /mcp` session tools, `POST /qhorus` agent mesh) is non-negotiable — native image requires no stdio subprocess
```

- [ ] **Step 7: Update remaining ARC42STORIES.MD "Qhorus tools join" references**

There are 5 occurrences of "Qhorus tools join" in ARC42STORIES.MD. Steps 3-5 cover lines 256, 578, and 550. The remaining references are:

Line 160 (§9.2 Chapter ordering):
```
- C5 after C4: Qhorus tools join the same MCP endpoint alongside C4's session tools
```
Change to:
```
- C5 after C4: Qhorus tools served at separate `/qhorus` endpoint alongside C4's session tools at `/mcp`
```

Line 474 (C5 ordering rationale):
```
- C5 after C4: Qhorus tools join the same MCP endpoint as C4's session tools; `qhorus` datasource co-configured
```
Change to:
```
- C5 after C4: Qhorus tools served at separate `/qhorus` endpoint (#105); `qhorus` datasource co-configured
```

Line 569 (C5 "What this delivers"):
```
Qhorus tools join the MCP endpoint.
```
Change to:
```
Qhorus tools served at `/qhorus` (separate named MCP server, #105).
```

Line 807 (L4 layer "Participates in chapters"):
```
C5 (Qhorus tools join)
```
Change to:
```
C5 (Qhorus tools at /qhorus)
```

- [ ] **Step 8: Run tests to verify no breakage from doc changes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

Expected: All tests pass (doc-only changes, but verify nothing was accidentally broken).

- [ ] **Step 9: Commit**

Stage: `CLAUDE.md`, `ARC42STORIES.MD`

Commit message: `docs(#105): update CLAUDE.md and ARC42STORIES.MD for endpoint separation`

---

### Task 4: File cross-repo issues and platform protocol

No code changes — issue and protocol creation only.

**Files:** None

- [ ] **Step 1: Create Qhorus issue**

```
gh issue create --repo casehubio/qhorus \
  --title "feat: scope MCP tools to named server via @McpServer(\"qhorus\")" \
  --body "..."
```

Body should reference claudony#105 and describe the 4 file changes from Task 1.

- [ ] **Step 2: Create propagation issues**

One issue per consumer repo (devtown, openclaw, drafthouse, life) describing the MCP client config change — add a second `mcpServers` entry for `/qhorus` if the consumer has external Claude clients.

- [ ] **Step 3: Create parent issue for PLATFORM.md update**

Issue in `casehubio/parent` to add the named-server convention to the Capability Ownership table.

- [ ] **Step 4: Capture platform protocol**

File the "Library MCP tools must use `@McpServer` with a named server" protocol in `casehubio/garden` using the `protocol` skill.

- [ ] **Step 5: Commit issue references to workspace**

Update the workspace `HANDOFF.md` or issue tracking as needed to reference all created issues.
