# PROXY Resize Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix PROXY-mode terminal resize so that resize commands reach the remote peer instead of silently failing on the local server.

**Architecture:** Add `resize()` to `PeerClient` and a new `POST /api/peers/{peerId}/sessions/{sessionId}/resize` endpoint to `PeerResource` that proxies the resize call to the peer. Update `terminal.js` `onResize` handler to detect the `proxyPeer` URL param and call the proxy endpoint instead of the local one. Add a Playwright E2E test that verifies the correct URL is called.

**Tech Stack:** Java 21, Quarkus 3.9.5, MicroProfile Rest Client, JUnit 5, RestAssured, Playwright for Java, vanilla JS

**Issue:** Refs mdproctor/claudony#50

---

## File Map

| File | Action |
|---|---|
| `src/main/java/dev/claudony/server/fleet/PeerClient.java` | **Modify** — add `resize()` method |
| `src/main/java/dev/claudony/server/fleet/PeerResource.java` | **Modify** — add `proxyResize()` endpoint |
| `src/test/java/dev/claudony/server/fleet/PeerResourceTest.java` | **Modify** — 2 new tests for proxy resize |
| `src/main/resources/META-INF/resources/app/terminal.js` | **Modify** — `onResize` uses proxy URL when `proxyPeer` param present |
| `src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java` | **Modify** — 1 new Playwright test verifying proxy resize URL |
| `CLAUDE.md` | **Modify** — update test count |

**Test count:** 210 current + 2 Java + 1 Playwright = 213 passing after.

---

## Task 1: PeerClient.resize() + PeerResource.proxyResize() + 2 Java Tests

**Files:**
- Modify: `src/main/java/dev/claudony/server/fleet/PeerClient.java`
- Modify: `src/main/java/dev/claudony/server/fleet/PeerResource.java`
- Modify: `src/test/java/dev/claudony/server/fleet/PeerResourceTest.java`

### Background

`PeerClient` currently only has `getSessions()`. Adding `resize()` lets `PeerResource` call the peer's own `/api/sessions/{sessionId}/resize` endpoint.

`PeerResource` class-level annotation is `@Consumes(MediaType.APPLICATION_JSON)`. The resize endpoint has no request body, so it needs `@Consumes(MediaType.WILDCARD)` to override — same pattern as `generateFleetKey()`.

- [ ] **Step 1: Write failing tests in PeerResourceTest**

Read `src/test/java/dev/claudony/server/fleet/PeerResourceTest.java` first. Add these two tests inside the class (they can go at the end, before the closing brace):

```java
@Test
void proxyResize_unknownPeer_returns404() {
    given()
        .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
        .when().post("/api/peers/does-not-exist/sessions/some-session/resize?cols=80&rows=24")
        .then().statusCode(404);
}

@Test
void proxyResize_knownPeer_peerUnreachable_returns502() {
    registry.addPeer("resize-test-peer", "http://localhost:19999",
            "Unreachable", DiscoverySource.MANUAL, TerminalMode.PROXY);

    given()
        .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
        .when().post("/api/peers/resize-test-peer/sessions/any-session/resize?cols=80&rows=24")
        .then().statusCode(502);
}
```

The `@AfterEach cleanup()` already removes MANUAL peers so `resize-test-peer` is cleaned up automatically.

- [ ] **Step 2: Run to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerResourceTest 2>&1 | tail -5
```

Expected: compilation succeeds (tests use only existing types) but tests FAIL with 405 (Method Not Allowed — endpoint doesn't exist yet).

- [ ] **Step 3: Add resize() to PeerClient**

In `src/main/java/dev/claudony/server/fleet/PeerClient.java`, add the import and method. The full updated file:

```java
package dev.claudony.server.fleet;

import dev.claudony.server.model.SessionResponse;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.rest.client.annotation.RegisterProvider;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;

import java.util.List;

/**
 * Typed REST client for calling another Claudony instance's session API.
 * The fleet key is injected via FleetKeyClientFilter.
 *
 * Note: the base URL is set dynamically per-peer using RestClientBuilder.newBuilder().
 * The quarkus.rest-client.peer-client.url property is a placeholder required by Quarkus
 * for compile-time validation — it is not used at runtime for peer calls.
 */
@RegisterRestClient(configKey = "peer-client")
@RegisterProvider(FleetKeyClientFilter.class)
@Path("/api")
public interface PeerClient {

    /**
     * Fetches sessions from a peer.
     *
     * @param localOnly when true, the peer returns only its own local sessions —
     *                  prevents recursive federation (peer A calls peer B which calls peer A)
     */
    @GET
    @Path("/sessions")
    List<SessionResponse> getSessions(@QueryParam("local") boolean localOnly);

    /**
     * Proxies a terminal resize command to the peer's session.
     * Returns the peer's response status (204 on success, 404 if session not found).
     */
    @POST
    @Path("/sessions/{sessionId}/resize")
    @Consumes(MediaType.WILDCARD)
    Response resize(@PathParam("sessionId") String sessionId,
                    @QueryParam("cols") int cols,
                    @QueryParam("rows") int rows);
}
```

- [ ] **Step 4: Add proxyResize() to PeerResource**

In `src/main/java/dev/claudony/server/fleet/PeerResource.java`, add the new method. Read the file first to find the right location (add after the existing `ping()` method, before `generateFleetKey()`):

```java
@POST
@Path("/{peerId}/sessions/{sessionId}/resize")
@Consumes(MediaType.WILDCARD)
public Response proxyResize(
        @PathParam("peerId") String peerId,
        @PathParam("sessionId") String sessionId,
        @QueryParam("cols") @DefaultValue("80") int cols,
        @QueryParam("rows") @DefaultValue("24") int rows) {

    var peer = registry.findById(peerId);
    if (peer.isEmpty()) {
        return Response.status(404).build();
    }

    var client = RestClientBuilder.newBuilder()
            .baseUri(URI.create(peer.get().url()))
            .connectTimeout(3, TimeUnit.SECONDS)
            .readTimeout(2, TimeUnit.SECONDS)
            .register(FleetKeyClientFilter.class)
            .build(PeerClient.class);

    try {
        var peerResponse = client.resize(sessionId, cols, rows);
        return Response.status(peerResponse.getStatus()).build();
    } catch (Exception e) {
        LOG.debugf("Proxy resize failed for peer %s session %s: %s", peerId, sessionId, e.getMessage());
        return Response.status(502).build();
    }
}
```

- [ ] **Step 5: Run the 2 new tests — both must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="PeerResourceTest#proxyResize_unknownPeer_returns404+proxyResize_knownPeer_peerUnreachable_returns502" 2>&1 | tail -8
```

Expected: `Tests run: 2, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 6: Run full test suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: 210 + 2 = 212 tests passing, 0 failures.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/PeerClient.java \
        src/main/java/dev/claudony/server/fleet/PeerResource.java \
        src/test/java/dev/claudony/server/fleet/PeerResourceTest.java
git commit -m "feat: PeerResource.proxyResize() — proxy resize to peer via /api/peers/{id}/sessions/{id}/resize (Refs #50)"
```

---

## Task 2: terminal.js Resize Handler + Playwright E2E Test

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/terminal.js`
- Modify: `src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java`

### Background

`terminal.js` `terminal.onResize` currently always calls `/api/sessions/{sessionId}/resize`. In PROXY mode (when `?proxyPeer=` URL param is set), it should call `/api/peers/{proxyPeer}/sessions/{sessionId}/resize` instead.

The Playwright test uses `page.waitForRequest()` to capture the first resize request that fires when xterm.js initializes (via `fitAddon.fit()` in `terminal.onResize`), then asserts the URL contains the proxy endpoint.

- [ ] **Step 1: Write the Playwright E2E test first (TDD)**

Read `src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java`. Add this test inside the class:

```java
@Test
void proxyMode_resizeCallsProxyEndpoint() {
    var proxyPeerId = "test-proxy-peer-id";

    // Wait for the resize request that fires when xterm.js initializes (fitAddon.fit())
    // page.waitForRequest() starts listening, then the lambda navigates — request is captured
    var resizeRequest = page.waitForRequest(
            request -> request.url().contains("/resize"),
            () -> page.navigate(BASE_URL + "/app/session.html?id=fake-session&name=test&proxyPeer=" + proxyPeerId));

    assertThat(resizeRequest.url())
            .as("In PROXY mode, resize must call /api/peers/{peerId}/sessions/ not /api/sessions/")
            .contains("/api/peers/" + proxyPeerId + "/sessions/fake-session/resize");
}
```

- [ ] **Step 2: Run to confirm the test fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=TerminalPageE2ETest#proxyMode_resizeCallsProxyEndpoint 2>&1 | tail -10
```

Expected: test FAILS — the resize request goes to `/api/sessions/fake-session/resize` (local, wrong URL) instead of the proxy endpoint.

- [ ] **Step 3: Fix the onResize handler in terminal.js**

In `src/main/resources/META-INF/resources/app/terminal.js`, find:

```javascript
    terminal.onResize(function (size) {
        fetch('/api/sessions/' + sessionId + '/resize?cols=' + size.cols + '&rows=' + size.rows, {
            method: 'POST'
        }).catch(function () {});
    });
```

Replace with:

```javascript
    terminal.onResize(function (size) {
        var proxyPeer = params.get('proxyPeer');
        var resizeUrl = proxyPeer
            ? '/api/peers/' + proxyPeer + '/sessions/' + sessionId
              + '/resize?cols=' + size.cols + '&rows=' + size.rows
            : '/api/sessions/' + sessionId + '/resize?cols=' + size.cols + '&rows=' + size.rows;
        fetch(resizeUrl, { method: 'POST' }).catch(function () {});
    });
```

Note: `params` is already defined at the top of terminal.js as `var params = new URLSearchParams(window.location.search);` — no new variable needed.

- [ ] **Step 4: Run the Playwright test — must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=TerminalPageE2ETest 2>&1 | tail -10
```

Expected: `Tests run: 2, Failures: 0, Errors: 0, Skipped: 0` (existing test + new proxy resize test)

- [ ] **Step 5: Run the original terminal page test to confirm no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest="TerminalPageE2ETest#terminalPage_loadsStructure_andDegracefullyHandlesFailedWebSocket" 2>&1 | tail -5
```

Expected: still passes (non-proxy mode unaffected).

- [ ] **Step 6: Run full Java test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: 212 tests passing (terminal.js change doesn't affect Java tests).

- [ ] **Step 7: Commit**

```bash
git add src/main/resources/META-INF/resources/app/terminal.js \
        src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java
git commit -m "fix: terminal.js PROXY resize — call /api/peers/{id}/sessions/{id}/resize when proxyPeer param set (Refs #50)"
```

---

## Task 3: CLAUDE.md Update + Final Verification

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update test count**

Find `**210 tests passing**` and replace with `**212 tests passing**`.

Update the `server/fleet/` bullet to include the two new proxy resize tests:

Old:
```
- `server/fleet/` — PeerRegistryTest (unit), StaticConfigDiscoveryTest (unit), MdnsDiscoveryTest (unit), PeerResourceTest (QuarkusTest), SessionFederationTest (QuarkusTest), ProxyWebSocketTest (QuarkusTest)
```

New:
```
- `server/fleet/` — PeerRegistryTest (unit), StaticConfigDiscoveryTest (unit), MdnsDiscoveryTest (unit), PeerResourceTest (QuarkusTest + proxy resize), SessionFederationTest (QuarkusTest), ProxyWebSocketTest (QuarkusTest)
```

Update the Playwright e2e bullet to include the new test:
```
- `e2e/` — ClaudeE2ETest (real `claude` CLI), PlaywrightSetupE2ETest (4 browser infra), DashboardE2ETest (7 dashboard UI), TerminalPageE2ETest (2: terminal page structure + proxy resize URL) — all via `mvn test -Pe2e -Dtest=...`, skipped in default run
```

- [ ] **Step 2: Run full Java suite one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: `Tests run: 212, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 3: Run Playwright tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e \
  -Dtest=PlaywrightSetupE2ETest,DashboardE2ETest,TerminalPageE2ETest 2>&1 | tail -5
```

Expected: `Tests run: 13, Failures: 0, Errors: 0, Skipped: 0` (4 + 7 + 2)

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update test count to 212 after PROXY resize fix (Refs #50)"
```

---

## E2E Manual Verification

After all tasks complete, to confirm resize actually works end-to-end in a two-node setup:

```bash
# Start two-node fleet
export CLAUDONY_FLEET_KEY=$(cat ~/.claudony/fleet-key)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests
docker compose up -d

# Add Node B as a PROXY peer to Node A
curl -X POST http://localhost:7777/api/peers \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: $(cat ~/.claudony/api-key)" \
  -d '{"url":"http://claudony-b:7777","name":"Node B","terminalMode":"PROXY"}'

# Create a session on Node B
curl -X POST http://localhost:7778/api/sessions \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: $(cat ~/.claudony/api-key)" \
  -d '{"name":"resize-test"}'

# Open the session from Node A's dashboard in PROXY mode
# Resize the browser window — the tmux pane on Node B should resize to match
```
