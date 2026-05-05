# Fleet Phase 2 — Dashboard + PROXY WebSocket Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the peer fleet visible and usable in the dashboard — instance badges on session cards, stale indicators, a fleet sidebar panel with peer management, and a PROXY WebSocket bridge for peers behind NAT.

**Architecture:** Java backend adds `ProxyWebSocket` at `/ws/proxy/{peerId}/{sessionId}/{cols}/{rows}` that pipes frames bidirectionally between browser and upstream peer using Vert.x async HTTP client. Frontend restructures `index.html` to a two-column layout (fleet sidebar + session grid), adds fleet panel rendering to `dashboard.js`, adds instance badges and stale indicators to session cards, and updates `terminal.js` to use the proxy URL when `proxyPeer` query param is present.

**Tech Stack:** Java 21, Quarkus 3.9.5, Quarkus WebSockets Next (`@WebSocket`), Vert.x Mutiny HTTP client, vanilla JS, CSS custom properties. No build tool for frontend — files served directly by Quarkus.

**Tracking:** Refs mdproctor/claudony#48

---

## File Map

| File | Action |
|---|---|
| `src/main/java/dev/claudony/server/fleet/ProxyWebSocket.java` | **Create** — WebSocket proxy bridge |
| `src/test/java/dev/claudony/server/fleet/ProxyWebSocketTest.java` | **Create** — QuarkusTest: unknown peer closes, endpoint registered |
| `src/main/resources/META-INF/resources/app/index.html` | **Modify** — two-column layout, Add Peer dialog |
| `src/main/resources/META-INF/resources/app/style.css` | **Modify** — fleet sidebar, instance badges, stale styles, peer health dots |
| `src/main/resources/META-INF/resources/app/dashboard.js` | **Modify** — fleet panel, peer rendering, session card fleet fields, peer map for terminal mode |
| `src/main/resources/META-INF/resources/app/terminal.js` | **Modify** — proxy mode: check `proxyPeer` param, build proxy WS URL |
| `CLAUDE.md` | **Modify** — update test count |

**Test count:** 207 current → ~212 after (5 new tests in `ProxyWebSocketTest`).

---

## Task 1: ProxyWebSocket Java Backend

**Files:**
- Create: `src/main/java/dev/claudony/server/fleet/ProxyWebSocket.java`
- Create: `src/test/java/dev/claudony/server/fleet/ProxyWebSocketTest.java`

### Design notes

Path: `/ws/proxy/{peerId}/{sessionId}/{cols}/{rows}` — 6 segments. Existing `TerminalWebSocket` is `/ws/{id}/{cols}/{rows}` — 4 segments. No routing conflict.

On open: look up peer by ID → if not found, close immediately. If found, open upstream WebSocket to `ws://{peerUrl}/ws/{sessionId}/{cols}/{rows}` with `X-Api-Key: {fleetKey}` header. Pipe text and binary frames bidirectionally. Close both sides on either disconnect.

- [ ] **Step 1: Write failing tests**

Create `src/test/java/dev/claudony/server/fleet/ProxyWebSocketTest.java`:

```java
package dev.claudony.server.fleet;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.WebSocket;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ProxyWebSocketTest {

    // Test profile runs on port 8081 (Quarkus default for tests)
    private static final String WS_BASE = "ws://localhost:8081";
    private static final String API_KEY = "test-api-key-do-not-use-in-prod";

    @Inject PeerRegistry registry;

    @AfterEach
    void cleanup() {
        registry.getAllPeers().stream()
                .filter(p -> p.source() == DiscoverySource.MANUAL)
                .map(PeerRecord::id)
                .toList()
                .forEach(registry::removePeer);
    }

    @Test
    void proxyWebSocket_unknownPeer_closesConnection() throws Exception {
        var closed = new AtomicBoolean(false);
        var latch = new CountDownLatch(1);

        HttpClient.newHttpClient()
                .newWebSocketBuilder()
                .header("X-Api-Key", API_KEY)
                .buildAsync(
                        URI.create(WS_BASE + "/ws/proxy/unknown-peer-id/any-session/80/24"),
                        new WebSocket.Listener() {
                            @Override
                            public CompletionStage<?> onClose(WebSocket ws, int code, String reason) {
                                closed.set(true);
                                latch.countDown();
                                return null;
                            }
                            @Override
                            public void onError(WebSocket ws, Throwable err) {
                                closed.set(true);
                                latch.countDown();
                            }
                        })
                .join();

        // Server closes the connection immediately for unknown peer
        assertThat(latch.await(3, TimeUnit.SECONDS)).isTrue();
        assertThat(closed.get()).isTrue();
    }

    @Test
    void proxyWebSocket_unauthenticated_rejected() throws Exception {
        var rejected = new AtomicBoolean(false);
        var latch = new CountDownLatch(1);

        try {
            HttpClient.newHttpClient()
                    .newWebSocketBuilder()
                    // No X-Api-Key header
                    .buildAsync(
                            URI.create(WS_BASE + "/ws/proxy/any-peer/any-session/80/24"),
                            new WebSocket.Listener() {
                                @Override
                                public CompletionStage<?> onClose(WebSocket ws, int code, String reason) {
                                    latch.countDown();
                                    return null;
                                }
                                @Override
                                public void onError(WebSocket ws, Throwable err) {
                                    rejected.set(true);
                                    latch.countDown();
                                }
                            })
                    .join();
        } catch (Exception e) {
            rejected.set(true);
            latch.countDown();
        }

        assertThat(latch.await(3, TimeUnit.SECONDS)).isTrue();
        // Connection rejected (401) or immediately closed without auth
        assertThat(rejected.get()).isTrue();
    }

    @Test
    void proxyWebSocket_unreachablePeer_closesGracefully() throws Exception {
        // Add a peer that exists in registry but can't be reached
        registry.addPeer("proxy-test-peer", "http://localhost:19999",
                "Unreachable", DiscoverySource.MANUAL, TerminalMode.PROXY);

        var closed = new AtomicBoolean(false);
        var latch = new CountDownLatch(1);

        HttpClient.newHttpClient()
                .newWebSocketBuilder()
                .header("X-Api-Key", API_KEY)
                .buildAsync(
                        URI.create(WS_BASE + "/ws/proxy/proxy-test-peer/any-session/80/24"),
                        new WebSocket.Listener() {
                            @Override
                            public CompletionStage<?> onClose(WebSocket ws, int code, String reason) {
                                closed.set(true);
                                latch.countDown();
                                return null;
                            }
                            @Override
                            public void onError(WebSocket ws, Throwable err) {
                                closed.set(true);
                                latch.countDown();
                            }
                        })
                .join();

        // Server opens the connection, fails upstream connect, then closes gracefully
        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(closed.get()).isTrue();
    }
}
```

- [ ] **Step 2: Run to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProxyWebSocketTest 2>&1 | tail -5
```

Expected: compilation error — `cannot find symbol: class ProxyWebSocket`

- [ ] **Step 3: Implement ProxyWebSocket**

Create `src/main/java/dev/claudony/server/fleet/ProxyWebSocket.java`:

```java
package dev.claudony.server.fleet;

import dev.claudony.server.auth.FleetKeyService;
import io.quarkus.websockets.next.*;
import io.vertx.core.buffer.Buffer;
import io.vertx.core.http.WebSocketConnectOptions;
import io.vertx.mutiny.core.Vertx;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.net.URI;
import java.util.concurrent.ConcurrentHashMap;

/**
 * WebSocket proxy for PROXY-mode peers.
 *
 * <p>Browser connects to ws://local:7777/ws/proxy/{peerId}/{sessionId}/{cols}/{rows}.
 * This endpoint opens an upstream WebSocket to the peer's /ws/{sessionId}/{cols}/{rows}
 * and pipes frames bidirectionally. The browser never needs direct access to the peer.</p>
 *
 * <p>Path is at /ws/proxy/... (6 segments) — no routing conflict with TerminalWebSocket
 * at /ws/{id}/{cols}/{rows} (4 segments).</p>
 */
@WebSocket(path = "/ws/proxy/{peerId}/{sessionId}/{cols}/{rows}")
public class ProxyWebSocket {

    private static final Logger LOG = Logger.getLogger(ProxyWebSocket.class);

    @Inject PeerRegistry peerRegistry;
    @Inject FleetKeyService fleetKeyService;
    @Inject Vertx vertx;

    /** connectionId → upstream WebSocket to the peer */
    private final ConcurrentHashMap<String, io.vertx.mutiny.core.http.WebSocket> upstreams
            = new ConcurrentHashMap<>();

    @OnOpen
    public void onOpen(WebSocketConnection connection) {
        var peerId = connection.pathParam("peerId");
        var sessionId = connection.pathParam("sessionId");
        var cols = connection.pathParam("cols");
        var rows = connection.pathParam("rows");

        var peer = peerRegistry.findById(peerId);
        if (peer.isEmpty()) {
            LOG.warnf("Proxy WS: peer not found id=%s — closing", peerId);
            connection.closeAndAwait();
            return;
        }

        var peerUrl = peer.get().url();
        var uri = URI.create(peerUrl);
        var port = uri.getPort() == -1 ? (peerUrl.startsWith("https") ? 443 : 80) : uri.getPort();
        var wsPath = "/ws/" + sessionId + "/" + cols + "/" + rows;

        var options = new WebSocketConnectOptions()
                .setHost(uri.getHost())
                .setPort(port)
                .setURI(wsPath);
        fleetKeyService.getKey().ifPresent(k -> options.addHeader("X-Api-Key", k));

        vertx.createHttpClient().webSocket(options).subscribe().with(
            upstream -> {
                upstreams.put(connection.id(), upstream);

                // upstream → browser
                upstream.textMessageHandler(text -> connection.sendTextAndForget(text));
                upstream.binaryMessageHandler(buf ->
                        connection.sendBinaryAndForget(buf.getBytes()));
                upstream.closeHandler(v -> {
                    upstreams.remove(connection.id());
                    connection.closeAndForget();
                });
                upstream.exceptionHandler(e -> {
                    LOG.debugf("Proxy upstream error for %s: %s", peerUrl, e.getMessage());
                    upstreams.remove(connection.id());
                    connection.closeAndForget();
                });

                LOG.debugf("Proxy WS open: peer=%s session=%s at %sx%s", peerUrl, sessionId, cols, rows);
            },
            err -> {
                LOG.warnf("Proxy WS upstream connect failed to %s: %s", peerUrl, err.getMessage());
                connection.closeAndAwait();
            }
        );
    }

    @OnTextMessage
    public void onText(WebSocketConnection connection, String text) {
        var upstream = upstreams.get(connection.id());
        if (upstream != null) upstream.writeTextMessageAndForget(text);
    }

    @OnBinaryMessage
    public void onBinary(WebSocketConnection connection, byte[] data) {
        var upstream = upstreams.get(connection.id());
        if (upstream != null) upstream.writeBinaryMessageAndForget(Buffer.buffer(data));
    }

    @OnClose
    public void onClose(WebSocketConnection connection) {
        var upstream = upstreams.remove(connection.id());
        if (upstream != null) upstream.closeAndForget();
    }
}
```

- [ ] **Step 4: Run ProxyWebSocketTest — all 3 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProxyWebSocketTest 2>&1 | tail -10
```

Expected: `Tests run: 3, Failures: 0, Errors: 0, Skipped: 0`

If any `@OnBinaryMessage` or `sendBinaryAndForget` signature fails to compile, check the Quarkus WebSockets Next 3.9.5 API. If `sendBinaryAndForget(byte[])` doesn't exist, use `sendBinary(buf).subscribe().with(ok -> {}, err -> {})`. If `@OnBinaryMessage` doesn't accept `byte[]`, use `io.vertx.core.buffer.Buffer` as the parameter type.

- [ ] **Step 5: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: 207 + 3 new = 210 tests passing.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/ProxyWebSocket.java \
        src/test/java/dev/claudony/server/fleet/ProxyWebSocketTest.java
git commit -m "feat: ProxyWebSocket — bidirectional WS bridge for PROXY-mode peers (Refs #48)"
```

---

## Task 2: index.html Layout Restructure + Add Peer Dialog

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/index.html`

Replace `src/main/resources/META-INF/resources/app/index.html` entirely:

- [ ] **Step 1: Replace index.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="theme-color" content="#007acc">
    <title>Claudony</title>
    <link rel="manifest" href="/manifest.json">
    <link rel="stylesheet" href="/app/style.css">
</head>
<body>
    <header>
        <h1>Claudony</h1>
        <button id="new-session-btn">+ New Session</button>
    </header>

    <div class="app-body">
        <aside id="fleet-panel">
            <div class="fleet-panel-header">
                <span class="fleet-panel-title">Fleet</span>
                <button id="add-peer-btn" class="small">+ Add Peer</button>
            </div>
            <div id="peer-list">
                <div class="peer-empty">No peers configured</div>
            </div>
        </aside>

        <main id="session-grid"></main>
    </div>

    <!-- New Session dialog (unchanged) -->
    <dialog id="new-session-dialog">
        <form id="new-session-form">
            <h2>New Session</h2>
            <label>Name
                <input name="name" required placeholder="myproject" autocomplete="off">
                <span id="name-error" style="display:none;color:#ff6b6b;font-size:0.8rem;margin-top:0.25rem;display:none"></span>
            </label>
            <label>Working Directory
                <input name="workingDir" placeholder="~/claudony-workspace (default)" autocomplete="off">
            </label>
            <div class="dialog-actions">
                <button type="button" id="cancel-btn" class="secondary">Cancel</button>
                <button type="button" id="overwrite-btn" class="danger" style="display:none">Overwrite</button>
                <button type="submit" id="create-btn">Create</button>
            </div>
        </form>
    </dialog>

    <!-- Add Peer dialog -->
    <dialog id="add-peer-dialog">
        <form id="add-peer-form">
            <h2>Add Peer</h2>
            <label>URL
                <input name="url" required placeholder="http://mac-mini:7777" autocomplete="off">
            </label>
            <label>Name
                <input name="name" placeholder="Mac Mini (optional)" autocomplete="off">
            </label>
            <label>Terminal Mode
                <select name="terminalMode">
                    <option value="DIRECT">Direct — browser connects to peer directly</option>
                    <option value="PROXY">Proxy — traffic routed through this server</option>
                </select>
            </label>
            <div class="dialog-actions">
                <button type="button" id="cancel-peer-btn" class="secondary">Cancel</button>
                <button type="submit">Add Peer</button>
            </div>
        </form>
    </dialog>

    <script src="/app/dashboard.js"></script>
    <script>
    if ('serviceWorker' in navigator) {
        navigator.serviceWorker.register('/sw.js');
    }
    </script>
</body>
</html>
```

- [ ] **Step 2: Verify it loads in browser (dev server)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server 2>&1 &
# Open http://localhost:7777/app/ and verify page loads (may look broken without CSS — that's OK for now)
# Kill the dev server after checking
```

- [ ] **Step 3: Run StaticFilesTest to confirm index.html still serves**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StaticFilesTest 2>&1 | tail -5
```

Expected: passes (StaticFilesTest checks the file exists and returns 200, not DOM content).

- [ ] **Step 4: Commit**

```bash
git add src/main/resources/META-INF/resources/app/index.html
git commit -m "feat: fleet panel sidebar layout + Add Peer dialog in index.html (Refs #48)"
```

---

## Task 3: style.css Fleet Styles

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/style.css`

- [ ] **Step 1: Add fleet design tokens and layout styles**

Add these CSS rules to the END of `src/main/resources/META-INF/resources/app/style.css`:

```css
/* ─── Fleet: layout ──────────────────────────────────────────────────────── */

.app-body {
    display: flex;
    height: calc(100vh - 57px); /* subtract header height */
    overflow: hidden;
}

#fleet-panel {
    width: 240px;
    min-width: 200px;
    background: var(--surface);
    border-right: 1px solid var(--border);
    display: flex;
    flex-direction: column;
    overflow-y: auto;
    flex-shrink: 0;
}

.fleet-panel-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 14px;
    border-bottom: 1px solid var(--border);
    position: sticky;
    top: 0;
    background: var(--surface);
    z-index: 1;
}

.fleet-panel-title {
    font-size: 12px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: .5px;
    color: var(--text-dim);
}

button.small {
    font-size: 11px;
    padding: 4px 8px;
}

.peer-empty {
    padding: 16px 14px;
    font-size: 12px;
    color: var(--text-dim);
    font-style: italic;
}

/* Peer list card */
.peer-card {
    padding: 10px 14px;
    border-bottom: 1px solid var(--border);
    font-size: 12px;
}

.peer-header {
    display: flex;
    align-items: center;
    gap: 6px;
    margin-bottom: 4px;
}

.peer-name {
    font-weight: 600;
    font-size: 12px;
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.peer-source {
    font-size: 10px;
    color: var(--text-dim);
    background: rgba(255,255,255,.05);
    border-radius: 4px;
    padding: 1px 5px;
    flex-shrink: 0;
}

.peer-url {
    color: var(--text-dim);
    font-family: var(--mono);
    font-size: 10px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    margin-bottom: 4px;
}

.peer-meta {
    display: flex;
    align-items: center;
    gap: 6px;
    margin-bottom: 6px;
    color: var(--text-dim);
    font-size: 11px;
}

.peer-actions {
    display: flex;
    gap: 4px;
    flex-wrap: wrap;
}

.peer-actions button {
    font-size: 10px;
    padding: 3px 7px;
}

/* ─── Fleet: health dots ─────────────────────────────────────────────────── */

.health-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
}

.health-dot.up     { background: #3fb950; }
.health-dot.unknown { background: #d29922; }
.health-dot.down   { background: #f85149; }

/* Circuit state label */
.circuit-label {
    font-size: 10px;
    padding: 1px 5px;
    border-radius: 4px;
    flex-shrink: 0;
}

.circuit-label.closed    { background: rgba(63,185,80,.15); color: #3fb950; }
.circuit-label.open      { background: rgba(248,81,73,.15); color: #f85149; }
.circuit-label.half-open { background: rgba(210,153,34,.15); color: #d29922; }

/* ─── Fleet: session card instance badge ────────────────────────────────── */

.instance-badge {
    font-size: 10px;
    padding: 2px 7px;
    border-radius: 10px;
    font-weight: 600;
    flex-shrink: 0;
    max-width: 90px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    background: rgba(0,122,204,.2);
    color: #4c9aff;
    border: 1px solid rgba(76,154,255,.3);
}

/* ─── Fleet: stale session indicator ────────────────────────────────────── */

.session-card.stale {
    border-color: rgba(210,153,34,.4);
    opacity: .85;
}

.stale-badge {
    font-size: 10px;
    color: #d29922;
    display: flex;
    align-items: center;
    gap: 3px;
}

/* ─── Fleet: dialog select element ──────────────────────────────────────── */

dialog select {
    display: block;
    width: 100%;
    margin-top: 4px;
    background: var(--bg);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 8px 12px;
    color: var(--text);
    font-size: 14px;
    font-family: var(--font);
}

dialog select:focus { outline: none; border-color: var(--accent); }

/* ─── Fleet: responsive — hide fleet panel below 600px ──────────────────── */

@media (max-width: 600px) {
    #fleet-panel { display: none; }
    .app-body { display: block; }
    #session-grid { height: auto; overflow: auto; }
}
```

Also update the existing `#session-grid` rule to fill available height:

Find this in `style.css`:
```css
#session-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 16px;
    padding: 24px;
}
```

Replace with:
```css
#session-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 16px;
    padding: 24px;
    align-content: start;
    overflow-y: auto;
    flex: 1;
}
```

- [ ] **Step 2: Run full test suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: all previous tests still passing.

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/app/style.css
git commit -m "feat: fleet panel CSS — sidebar layout, peer health dots, instance badges, stale styles (Refs #48)"
```

---

## Task 4: dashboard.js — Fleet Panel

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/dashboard.js`

This task adds fleet panel rendering (peer list + Add Peer action) to `dashboard.js`. The existing session code is unchanged.

- [ ] **Step 1: Add fleet panel code to dashboard.js**

Add this block BEFORE the final `})();` closing line of `dashboard.js`. The code adds peer loading, rendering, and Add Peer form handling:

```javascript
    // ─── Fleet panel ────────────────────────────────────────────────────────

    var peerList = document.getElementById('peer-list');
    var addPeerDialog = document.getElementById('add-peer-dialog');
    var addPeerForm = document.getElementById('add-peer-form');

    // Map of peer URL → {id, terminalMode} for session card lookup
    var peerTerminalModes = {};

    function healthClass(health) {
        if (health === 'UP') return 'up';
        if (health === 'DOWN') return 'down';
        return 'unknown';
    }

    function circuitClass(state) {
        if (state === 'CLOSED') return 'closed';
        if (state === 'OPEN') return 'open';
        return 'half-open';
    }

    function circuitLabel(state) {
        if (state === 'HALF_OPEN') return 'half-open';
        return state.toLowerCase();
    }

    function renderPeer(p) {
        var card = document.createElement('div');
        card.className = 'peer-card';
        card.dataset.id = p.id;

        var lastSeen = p.lastSeen ? timeAgo(p.lastSeen) : 'never';
        var staleNote = p.health === 'DOWN' && p.sessionCount > 0
            ? '<span class="stale-badge">⏰ ' + lastSeen + '</span>'
            : '<span>' + lastSeen + '</span>';

        card.innerHTML =
            '<div class="peer-header">' +
                '<div class="health-dot ' + healthClass(p.health) + '"></div>' +
                '<span class="peer-name">' + (p.name || p.url) + '</span>' +
                '<span class="peer-source">' + p.source.toLowerCase() + '</span>' +
            '</div>' +
            '<div class="peer-url">' + p.url + '</div>' +
            '<div class="peer-meta">' +
                '<span class="circuit-label ' + circuitClass(p.circuitState) + '">' +
                    circuitLabel(p.circuitState) +
                '</span>' +
                staleNote +
            '</div>' +
            '<div class="peer-actions">' +
                '<button class="peer-ping-btn secondary">Ping</button>' +
                '<button class="peer-mode-btn secondary" title="Click to toggle">' +
                    p.terminalMode +
                '</button>' +
                (p.source !== 'CONFIG'
                    ? '<button class="peer-remove-btn danger">Remove</button>'
                    : '') +
            '</div>';

        card.querySelector('.peer-ping-btn').addEventListener('click', function (e) {
            e.stopPropagation();
            var btn = e.target;
            btn.disabled = true;
            btn.textContent = '…';
            fetch('/api/peers/' + p.id + '/ping', { method: 'POST' })
                .then(function (r) { requireAuth(r); })
                .finally(function () {
                    btn.disabled = false;
                    btn.textContent = 'Ping';
                    setTimeout(loadPeers, 2000); // refresh after ping settles
                });
        });

        card.querySelector('.peer-mode-btn').addEventListener('click', function (e) {
            e.stopPropagation();
            var newMode = p.terminalMode === 'DIRECT' ? 'PROXY' : 'DIRECT';
            fetch('/api/peers/' + p.id, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ terminalMode: newMode })
            }).then(function (r) {
                requireAuth(r);
                loadPeers();
            });
        });

        if (p.source !== 'CONFIG') {
            card.querySelector('.peer-remove-btn').addEventListener('click', function (e) {
                e.stopPropagation();
                if (!confirm('Remove peer "' + (p.name || p.url) + '"?')) return;
                fetch('/api/peers/' + p.id, { method: 'DELETE' })
                    .then(function (r) { requireAuth(r); loadPeers(); });
            });
        }

        return card;
    }

    function loadPeers() {
        fetch('/api/peers').then(function (r) {
            if (!requireAuth(r)) return null;
            return r.json();
        }).then(function (peers) {
            if (!peers) return;
            // Rebuild terminal mode map for session card rendering
            peerTerminalModes = {};
            peers.forEach(function (p) {
                peerTerminalModes[p.url] = { id: p.id, terminalMode: p.terminalMode };
            });

            peerList.innerHTML = '';
            if (peers.length === 0) {
                peerList.innerHTML = '<div class="peer-empty">No peers configured</div>';
            } else {
                peers.forEach(function (p) { peerList.appendChild(renderPeer(p)); });
            }
        });
    }

    loadPeers();
    setInterval(loadPeers, 10000); // peer health changes more slowly than sessions

    document.getElementById('add-peer-btn').addEventListener('click', function () {
        addPeerForm.reset();
        addPeerDialog.showModal();
    });

    document.getElementById('cancel-peer-btn').addEventListener('click', function () {
        addPeerDialog.close();
    });

    addPeerForm.addEventListener('submit', function (e) {
        e.preventDefault();
        var data = new FormData(addPeerForm);
        fetch('/api/peers', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                url: data.get('url'),
                name: data.get('name') || null,
                terminalMode: data.get('terminalMode')
            })
        }).then(function (r) {
            if (!requireAuth(r)) return;
            addPeerDialog.close();
            addPeerForm.reset();
            loadPeers();
        });
    });
```

- [ ] **Step 2: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: all tests still passing.

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/app/dashboard.js
git commit -m "feat: fleet panel in dashboard — peer list, health dots, ping/remove/mode-toggle, Add Peer modal (Refs #48)"
```

---

## Task 5: dashboard.js — Session Card Fleet Fields

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/dashboard.js`

Modifies `renderCard()` to show instance badge on remote sessions, stale indicator, and correct "Open Terminal" URL for remote sessions.

- [ ] **Step 1: Modify renderCard() in dashboard.js**

Find the existing `renderCard(s)` function. Replace it entirely:

```javascript
    function renderCard(s) {
        var card = document.createElement('div');
        card.className = 'session-card' + (s.stale ? ' stale' : '');

        var name = displayName(s.name);
        var status = s.status.toLowerCase();

        // Determine open URL: local sessions use /app/session.html directly;
        // remote sessions use the peer's URL (DIRECT) or the proxy URL (PROXY).
        var openUrl;
        if (!s.instanceUrl) {
            // Local session
            openUrl = '/app/session.html?id=' + s.id + '&name=' + encodeURIComponent(name);
        } else {
            var peerInfo = peerTerminalModes[s.instanceUrl];
            if (peerInfo && peerInfo.terminalMode === 'PROXY') {
                // PROXY: browser connects via local proxy WebSocket.
                // Note: resize calls will silently fail (404) for remote sessions — acceptable in Phase 2.
                openUrl = '/app/session.html?id=' + s.id
                    + '&name=' + encodeURIComponent(name)
                    + '&proxyPeer=' + encodeURIComponent(peerInfo.id);
            } else {
                // DIRECT: navigate browser to the remote Claudony instance
                openUrl = s.instanceUrl + '/app/session.html?id=' + s.id
                    + '&name=' + encodeURIComponent(name);
            }
        }

        // Instance badge: shown for remote sessions
        var instanceBadge = s.instanceUrl
            ? '<span class="instance-badge">' + (s.instanceName || s.instanceUrl) + '</span>'
            : '';

        // Stale badge: shown when peer is down but we have cached data
        var staleBadge = s.stale
            ? '<div class="stale-badge">⏰ last seen ' + timeAgo(s.lastActive) + '</div>'
            : '';

        var itermBtn = (isLocalhost && !s.instanceUrl)
            ? '<button class="iterm-btn">Open in iTerm2</button>'
            : '';
        var hasWorkingDir = s.workingDir && s.workingDir !== 'unknown';
        var prBtn = (hasWorkingDir && !s.instanceUrl) ? '<button class="pr-btn">Check PR</button>' : '';

        card.innerHTML =
            '<div class="card-header">' +
                '<span class="card-name">' + name + '</span>' +
                '<span class="badge ' + status + '">' + status + '</span>' +
                instanceBadge +
            '</div>' +
            staleBadge +
            '<div class="card-dir">' + s.workingDir + '</div>' +
            '<div class="card-meta">Active ' + timeAgo(s.lastActive) + '</div>' +
            (hasWorkingDir && !s.instanceUrl ? '<div class="card-git"></div>' : '') +
            (s.instanceUrl ? '' : '<div class="card-services"></div>') +
            '<div class="card-actions">' +
                '<button class="open-btn">Open Terminal</button>' +
                prBtn +
                (s.instanceUrl ? '' : '<button class="svc-btn">Check Services</button>') +
                itermBtn +
                (s.instanceUrl ? '' : '<button class="danger delete-btn">Delete</button>') +
            '</div>';

        card.querySelector('.open-btn').addEventListener('click', function (e) {
            e.stopPropagation();
            window.location.href = openUrl;
        });

        if (hasWorkingDir && !s.instanceUrl) {
            card.querySelector('.pr-btn').addEventListener('click', function (e) {
                e.stopPropagation();
                var btn = e.target;
                var gitDiv = card.querySelector('.card-git');
                btn.disabled = true;
                btn.textContent = '…';
                fetch('/api/sessions/' + s.id + '/git-status')
                    .then(function (r) { requireAuth(r); return r.json(); })
                    .then(function (data) {
                        gitDiv.innerHTML = prStatusHtml(data);
                        btn.disabled = false;
                        btn.textContent = 'Check PR';
                    })
                    .catch(function () {
                        gitDiv.innerHTML = '<span class="git-error">⚠ fetch failed</span>';
                        btn.disabled = false;
                        btn.textContent = 'Check PR';
                    });
            });
        }

        if (!s.instanceUrl) {
            card.querySelector('.svc-btn').addEventListener('click', function (e) {
                e.stopPropagation();
                var btn = e.target;
                var svcDiv = card.querySelector('.card-services');
                btn.disabled = true;
                btn.textContent = '…';
                fetch('/api/sessions/' + s.id + '/service-health')
                    .then(function (r) { requireAuth(r); return r.json(); })
                    .then(function (ports) {
                        if (ports.length === 0) {
                            svcDiv.innerHTML = '<span class="svc-label">Services:</span><span class="svc-none">none detected</span>';
                        } else {
                            svcDiv.innerHTML = '<span class="svc-label">Services:</span>' + ports.map(function (p) {
                                return '<a class="svc-badge" href="http://localhost:' + p.port +
                                       '" target="_blank" onclick="event.stopPropagation()" title="port ' +
                                       p.port + ' responded in ' + p.responseMs + 'ms">● :' + p.port + '</a>';
                            }).join('');
                        }
                        btn.disabled = false;
                        btn.textContent = 'Check Services';
                    })
                    .catch(function () {
                        svcDiv.innerHTML = '<span class="svc-none">check failed</span>';
                        btn.disabled = false;
                        btn.textContent = 'Check Services';
                    });
            });
        }

        if (isLocalhost && !s.instanceUrl) {
            card.querySelector('.iterm-btn').addEventListener('click', function (e) {
                e.stopPropagation();
                fetch('/api/sessions/' + s.id + '/open-terminal', { method: 'POST' })
                    .then(function (r) {
                        requireAuth(r);
                        if (r.status === 503) alert('No terminal adapter available on this machine.');
                    });
            });
        }

        if (!s.instanceUrl) {
            card.querySelector('.delete-btn').addEventListener('click', function (e) {
                e.stopPropagation();
                if (!confirm('Delete session "' + name + '"?')) return;
                fetch('/api/sessions/' + s.id, { method: 'DELETE' })
                    .then(function (r) { requireAuth(r); loadSessions(); });
            });
        }

        card.addEventListener('click', function () { window.location.href = openUrl; });
        return card;
    }
```

- [ ] **Step 2: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: all tests passing.

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/app/dashboard.js
git commit -m "feat: session card fleet fields — instance badge, stale indicator, remote/proxy terminal open (Refs #48)"
```

---

## Task 6: terminal.js — Proxy Mode Support

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/terminal.js`

When `proxyPeer` query param is present, `terminal.js` connects to `/ws/proxy/{proxyPeer}/{sessionId}/{cols}/{rows}` instead of `/ws/{sessionId}/{cols}/{rows}`. This is the only change; all other terminal behaviour is identical.

- [ ] **Step 1: Modify the connect() function in terminal.js**

Find the existing `connect()` function:

```javascript
    function connect() {
        // Include current terminal dimensions in the URL so the server can
        // resize the tmux pane BEFORE capturing history, causing TUI apps to
        // redraw first so capture-pane gets the fresh state (not stale).
        var wsUrl = proto + '//' + location.host + '/ws/' + sessionId
            + '/' + terminal.cols + '/' + terminal.rows;
```

Replace the first two lines of the function body (just the `wsUrl` construction) with:

```javascript
    function connect() {
        var proxyPeer = params.get('proxyPeer');
        var wsUrl = proxyPeer
            ? proto + '//' + location.host + '/ws/proxy/' + proxyPeer
              + '/' + sessionId + '/' + terminal.cols + '/' + terminal.rows
            : proto + '//' + location.host + '/ws/' + sessionId
              + '/' + terminal.cols + '/' + terminal.rows;
```

The resize handler (`terminal.onResize`) is left unchanged. For PROXY-mode remote sessions the resize call goes to the local `/api/sessions/{id}/resize` — the session isn't in the local registry, so it returns 404, which `.catch(function() {})` silently absorbs. Resize won't work for proxied remote sessions in Phase 2; this is a known and accepted limitation.

- [ ] **Step 2: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: all tests passing (JavaScript changes don't affect Java tests).

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/app/terminal.js
git commit -m "feat: terminal.js proxy mode — use /ws/proxy/ URL when proxyPeer param is set (Refs #48)"
```

---

## Task 7: CLAUDE.md Update + Final Verification

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Run full test suite and get exact count**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep "Tests run:" | tail -1
```

Note the final count.

- [ ] **Step 2: Update CLAUDE.md**

Find `**207 tests passing**` and update to the new count.

Add to the `server/fleet/` test bullet:
```
- `server/fleet/` — PeerRegistryTest (unit), StaticConfigDiscoveryTest (unit), MdnsDiscoveryTest (unit), PeerResourceTest (QuarkusTest), SessionFederationTest (QuarkusTest), ProxyWebSocketTest (QuarkusTest)
```

- [ ] **Step 3: Final commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md test count after fleet Phase 2 (Refs #48)"
```

---

## E2E Verification (Manual)

After all tasks:

1. Build and run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests && JAVA_HOME=$(/usr/libexec/java_home -v 26) java -Dclaudony.mode=server -Dclaudony.bind=0.0.0.0 -jar target/quarkus-app/quarkus-run.jar`
2. Open `http://localhost:7777/app/` — fleet sidebar should be visible on the left
3. Click **+ Add Peer** — modal should open
4. Add a peer URL (another Claudony instance or `http://localhost:7778` if you have a second running)
5. Peer appears in fleet panel with health indicator
6. Sessions from the peer appear in the session grid with instance badge
7. Stale sessions (peer goes down) show clock icon and "last seen N ago"
8. DIRECT mode: "Open Terminal" on remote session navigates browser to peer URL
9. PROXY mode: toggle terminal mode to PROXY, "Open Terminal" on remote session uses `/ws/proxy/...` — terminal loads via local WebSocket proxy

---

## Summary

| Task | What | Tests added | Commit |
|---|---|---|---|
| 1 | ProxyWebSocket Java backend | 3 QuarkusTest | `feat: ProxyWebSocket` |
| 2 | index.html layout + Add Peer dialog | 0 | `feat: fleet panel HTML` |
| 3 | style.css fleet styles | 0 | `feat: fleet CSS` |
| 4 | dashboard.js fleet panel | 0 | `feat: fleet panel JS` |
| 5 | dashboard.js session card fleet | 0 | `feat: session card fleet fields` |
| 6 | terminal.js proxy mode | 0 | `feat: terminal.js proxy` |
| 7 | CLAUDE.md | 0 | `docs: test count` |
| **Total** | | **3** | |
