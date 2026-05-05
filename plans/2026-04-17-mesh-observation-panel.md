# Mesh Observation Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a collapsible right panel to the Claudony dashboard that observes the local Qhorus agent communication mesh — channels, instances, and messages — via three switchable views.

**Architecture:** New `MeshResource` REST facade delegates to `QhorusMcpTools` (already embedded, N+1-safe). Frontend adds `MeshPanel` class + two strategies (poll/SSE) + three pure view renderers to `dashboard.js`. Layout is additive — no existing code is modified except appending to three frontend files.

**Tech Stack:** Java 21, Quarkus 3.32.2, `@Authenticated`, `QhorusMcpTools` (io.quarkiverse.qhorus), Vanilla JS, Playwright E2E.

**Design spec:** `docs/superpowers/specs/2026-04-17-mesh-observation-panel-design.md`

---

## Task 1: Create GitHub epic and child issues

**Files:** None — GitHub only.

- [ ] **Step 1: Create the epic**

```bash
gh issue create --repo mdproctor/claudony \
  --title "epic: Mesh observation panel — Qhorus channel/instance/message visibility in dashboard" \
  --label "epic" \
  --body "Three-task epic for the Mesh observation panel. See docs/superpowers/specs/2026-04-17-mesh-observation-panel-design.md"
```

Note the epic issue number (e.g. #58). All child issues reference it.

- [ ] **Step 2: Create backend child issue**

```bash
gh issue create --repo mdproctor/claudony \
  --title "feat: MeshResource — /api/mesh/* REST endpoints for Qhorus observation" \
  --body "Part of epic #58. Implements GET /api/mesh/config, /channels, /instances, /channels/{name}/timeline, /feed, /events."
```

Note the issue number (e.g. #59).

- [ ] **Step 3: Create frontend child issue**

```bash
gh issue create --repo mdproctor/claudony \
  --title "feat: Mesh panel frontend — collapsible right panel with three views" \
  --body "Part of epic #58. Adds mesh panel HTML, CSS, and dashboard.js classes (MeshPanel, strategies, view renderers)."
```

Note the issue number (e.g. #60).

- [ ] **Step 4: Create E2E test child issue**

```bash
gh issue create --repo mdproctor/claudony \
  --title "test: MeshPanelE2ETest — Playwright tests for panel structure and behaviour" \
  --body "Part of epic #58. Four Playwright tests: visible, collapse/expand, view switching, empty state."
```

Note the issue number (e.g. #61).

---

## Task 2: Config additions + MeshConfig endpoint (TDD)

**Files:**
- Modify: `src/main/java/dev/claudony/config/ClaudonyConfig.java`
- Modify: `src/main/resources/application.properties`
- Create: `src/main/java/dev/claudony/server/MeshResource.java`
- Create: `src/test/java/dev/claudony/server/MeshResourceTest.java`

Use the backend issue number from Task 1 in all commits (e.g. `Refs #59`).

- [ ] **Step 1: Write the failing test**

Create `src/test/java/dev/claudony/server/MeshResourceTest.java`:

```java
package dev.claudony.server;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class MeshResourceTest {

    @Test
    void meshConfig_returnsStrategyAndInterval() {
        given().when().get("/api/mesh/config")
            .then()
            .statusCode(200)
            .contentType(containsString("application/json"))
            .body("strategy", equalTo("poll"))
            .body("interval", equalTo(3000));
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshResourceTest -Dno-format -q 2>&1 | tail -5
```

Expected: FAIL — 404 (no endpoint yet).

- [ ] **Step 3: Add config properties to ClaudonyConfig**

In `src/main/java/dev/claudony/config/ClaudonyConfig.java`, add before the closing brace:

```java
    @WithName("mesh.refresh-strategy")
    @WithDefault("poll")
    String meshRefreshStrategy();

    @WithName("mesh.refresh-interval")
    @WithDefault("3000")
    int meshRefreshInterval();
```

- [ ] **Step 4: Add config to application.properties**

In `src/main/resources/application.properties`, add after the `# Fleet configuration` block:

```properties
# Mesh observation panel
claudony.mesh.refresh-strategy=poll
claudony.mesh.refresh-interval=3000
%test.claudony.mesh.refresh-strategy=poll
%test.claudony.mesh.refresh-interval=3000
```

- [ ] **Step 5: Create MeshResource with config endpoint**

Create `src/main/java/dev/claudony/server/MeshResource.java`:

```java
package dev.claudony.server;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.claudony.config.ClaudonyConfig;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.security.Authenticated;
import io.smallrye.mutiny.Multi;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Path("/api/mesh")
@Produces(MediaType.APPLICATION_JSON)
@Authenticated
public class MeshResource {

    private static final Logger LOG = Logger.getLogger(MeshResource.class);

    @Inject ClaudonyConfig config;
    @Inject QhorusMcpTools qhorusMcpTools;
    @Inject ObjectMapper mapper;

    record MeshConfig(String strategy, int interval) {}

    @GET
    @Path("/config")
    public MeshConfig config() {
        return new MeshConfig(config.meshRefreshStrategy(), config.meshRefreshInterval());
    }
}
```

- [ ] **Step 6: Run test — should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshResourceTest -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`

- [ ] **Step 7: Commit**

```bash
git add src/main/java/dev/claudony/config/ClaudonyConfig.java \
        src/main/resources/application.properties \
        src/main/java/dev/claudony/server/MeshResource.java \
        src/test/java/dev/claudony/server/MeshResourceTest.java
git commit -m "feat: add mesh config endpoint and ClaudonyConfig mesh properties (Refs #59)"
```

---

## Task 3: Channels and instances endpoints (TDD)

**Files:**
- Modify: `src/main/java/dev/claudony/server/MeshResource.java`
- Modify: `src/test/java/dev/claudony/server/MeshResourceTest.java`

- [ ] **Step 1: Write the failing tests**

Add to `MeshResourceTest.java` (inside the class body, after the existing test):

```java
    @Test
    void meshChannels_returnsEmptyList() {
        given().when().get("/api/mesh/channels")
            .then()
            .statusCode(200)
            .contentType(containsString("application/json"))
            .body("$", hasSize(0));
    }

    @Test
    void meshInstances_returnsEmptyList() {
        given().when().get("/api/mesh/instances")
            .then()
            .statusCode(200)
            .contentType(containsString("application/json"))
            .body("$", hasSize(0));
    }
```

Add `import static org.hamcrest.Matchers.*;` if not already present (it is via the wildcard).

- [ ] **Step 2: Run to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshResourceTest -Dno-format -q 2>&1 | tail -5
```

Expected: 2 new failures — 404 (endpoints don't exist yet).

- [ ] **Step 3: Add channels and instances endpoints to MeshResource**

Add to `MeshResource.java` after the `config()` method:

```java
    @GET
    @Path("/channels")
    public List<QhorusMcpTools.ChannelDetail> channels() {
        return qhorusMcpTools.listChannels();
    }

    @GET
    @Path("/instances")
    public List<QhorusMcpTools.InstanceInfo> instances() {
        return qhorusMcpTools.listInstances(null);
    }
```

- [ ] **Step 4: Run — all 3 tests should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshResourceTest -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/MeshResource.java \
        src/test/java/dev/claudony/server/MeshResourceTest.java
git commit -m "feat: add /api/mesh/channels and /api/mesh/instances endpoints (Refs #59)"
```

---

## Task 4: Timeline and feed endpoints (TDD)

**Files:**
- Modify: `src/main/java/dev/claudony/server/MeshResource.java`
- Modify: `src/test/java/dev/claudony/server/MeshResourceTest.java`

- [ ] **Step 1: Write the failing tests**

Add to `MeshResourceTest.java`:

```java
    @Test
    void meshTimeline_unknownChannel_returnsEmptyList() {
        given().when().get("/api/mesh/channels/does-not-exist/timeline")
            .then()
            .statusCode(200)
            .body("$", hasSize(0));
    }

    @Test
    void meshFeed_returnsEmptyList() {
        given().when().get("/api/mesh/feed")
            .then()
            .statusCode(200)
            .body("$", hasSize(0));
    }
```

- [ ] **Step 2: Run to confirm failures**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshResourceTest -Dno-format -q 2>&1 | tail -5
```

Expected: 2 new failures.

- [ ] **Step 3: Add timeline and feed endpoints to MeshResource**

Add to `MeshResource.java` after the `instances()` method:

```java
    @GET
    @Path("/channels/{name}/timeline")
    public List<Map<String, Object>> timeline(
            @PathParam("name") String name,
            @QueryParam("limit") @DefaultValue("50") int limit) {
        try {
            return qhorusMcpTools.getChannelTimeline(name, null, limit);
        } catch (IllegalArgumentException e) {
            // Unknown channel — return empty rather than 404
            return List.of();
        }
    }

    @GET
    @Path("/feed")
    public List<Map<String, Object>> feed(
            @QueryParam("limit") @DefaultValue("100") int limit) {
        List<QhorusMcpTools.ChannelDetail> channels = qhorusMcpTools.listChannels();
        if (channels.isEmpty()) return List.of();

        int perChannel = Math.max(5, limit / channels.size());
        List<Map<String, Object>> combined = new ArrayList<>();

        for (QhorusMcpTools.ChannelDetail ch : channels) {
            try {
                List<Map<String, Object>> msgs = qhorusMcpTools.getChannelTimeline(
                        ch.name(), null, perChannel);
                for (Map<String, Object> m : msgs) {
                    Map<String, Object> tagged = new HashMap<>(m);
                    tagged.put("channel", ch.name());
                    combined.add(tagged);
                }
            } catch (Exception e) {
                LOG.debugf("Skipping channel %s in feed: %s", ch.name(), e.getMessage());
            }
        }

        // Sort newest-last; ISO-8601 strings sort lexicographically
        combined.sort((a, b) -> {
            String ta = String.valueOf(a.getOrDefault("created_at", ""));
            String tb = String.valueOf(b.getOrDefault("created_at", ""));
            return ta.compareTo(tb);
        });

        return combined.size() > limit ? combined.subList(0, limit) : combined;
    }
```

- [ ] **Step 4: Run — all 5 tests should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshResourceTest -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 5, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/MeshResource.java \
        src/test/java/dev/claudony/server/MeshResourceTest.java
git commit -m "feat: add /api/mesh/channels/{name}/timeline and /api/mesh/feed endpoints (Refs #59)"
```

---

## Task 5: Auth protection and SSE endpoint (TDD)

**Files:**
- Modify: `src/main/java/dev/claudony/server/MeshResource.java`
- Modify: `src/test/java/dev/claudony/server/MeshResourceTest.java`

- [ ] **Step 1: Write the failing tests**

Add to `MeshResourceTest.java`:

```java
    @Test
    void meshEvents_returnsEventStreamContentType() {
        given().when().get("/api/mesh/events")
            .then()
            .statusCode(200)
            .contentType(containsString("text/event-stream"));
    }
```

Add a new inner class (separate from `MeshResourceTest`) for auth tests — no `@TestSecurity`:

```java
// In the same file, after the closing brace of MeshResourceTest:

@QuarkusTest
class MeshResourceAuthTest {

    @Test
    void meshChannels_withoutAuth_returns401() {
        given().when().get("/api/mesh/channels")
            .then().statusCode(401);
    }

    @Test
    void meshConfig_withoutAuth_returns401() {
        given().when().get("/api/mesh/config")
            .then().statusCode(401);
    }
}
```

- [ ] **Step 2: Run to confirm SSE test fails (auth tests may already pass)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="MeshResourceTest,MeshResourceAuthTest" -Dno-format -q 2>&1 | tail -5
```

Expected: auth tests pass (already protected by `/api/*` policy), SSE test fails (endpoint doesn't exist).

- [ ] **Step 3: Add SSE endpoint to MeshResource**

Add imports at the top of `MeshResource.java`:

```java
import io.smallrye.mutiny.Multi;
import java.time.Duration;
```

Add the SSE endpoint method:

```java
    @GET
    @Path("/events")
    @Produces("text/event-stream")
    public Multi<String> events() {
        // Pushes a full mesh snapshot every refresh-interval seconds.
        // Default strategy is poll; SSE is for deployments that prefer push.
        long intervalMs = config.meshRefreshInterval();
        return Multi.createFrom().ticks().every(Duration.ofMillis(intervalMs))
                .map(tick -> {
                    try {
                        var snapshot = Map.of(
                                "channels", qhorusMcpTools.listChannels(),
                                "instances", qhorusMcpTools.listInstances(null),
                                "feed", feed(100));
                        return "data: " + mapper.writeValueAsString(snapshot) + "\n\n";
                    } catch (Exception e) {
                        LOG.debugf("SSE snapshot error: %s", e.getMessage());
                        return "data: {}\n\n";
                    }
                });
    }
```

- [ ] **Step 4: Run all MeshResource tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="MeshResourceTest,MeshResourceAuthTest" -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 8, Failures: 0, Errors: 0`

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: 239+ tests, 0 failures.

- [ ] **Step 6: Commit and close backend issue**

```bash
git add src/main/java/dev/claudony/server/MeshResource.java \
        src/test/java/dev/claudony/server/MeshResourceTest.java
git commit -m "feat: add /api/mesh/events SSE endpoint and auth protection tests (Refs #59)"
gh issue close 59 --repo mdproctor/claudony --comment "All REST endpoints implemented and tested."
```

---

## Task 6: StaticFilesTest assertion + HTML panel (TDD)

**Files:**
- Modify: `src/test/java/dev/claudony/frontend/StaticFilesTest.java`
- Modify: `src/main/resources/META-INF/resources/app/index.html`

Use the frontend issue number from Task 1 (e.g. `Refs #60`).

- [ ] **Step 1: Write the failing StaticFilesTest assertion**

Add to `StaticFilesTest.java` (inside the class body):

```java
    @Test
    void dashboardContainsMeshPanel() {
        given().when().get("/app/index.html")
            .then().statusCode(200)
            .body(containsString("id=\"mesh-panel\""));
    }
```

- [ ] **Step 2: Run to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StaticFilesTest -Dno-format -q 2>&1 | tail -5
```

Expected: 1 failure — `index.html` doesn't contain `id="mesh-panel"` yet.

- [ ] **Step 3: Add mesh panel HTML to index.html**

In `src/main/resources/META-INF/resources/app/index.html`, replace:

```html
        <main id="session-grid"></main>
    </div>
```

With:

```html
        <main id="session-grid"></main>
        <aside id="mesh-panel">
            <div class="mesh-header">
                <span class="mesh-title">MESH</span>
                <div class="mesh-view-switcher">
                    <button class="mesh-view-btn active" data-view="overview" title="Overview">&#9678;</button>
                    <button class="mesh-view-btn" data-view="channel" title="Channel">#</button>
                    <button class="mesh-view-btn" data-view="feed" title="Feed">&#8801;</button>
                </div>
                <button class="mesh-collapse-btn" id="mesh-collapse-btn" title="Collapse">&larr;</button>
            </div>
            <div class="mesh-body" id="mesh-body"></div>
        </aside>
    </div>
    <button id="mesh-expand-btn" class="mesh-expand" title="Open Mesh panel" style="display:none">MESH</button>
```

- [ ] **Step 4: Run StaticFilesTest — should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StaticFilesTest -Dno-format -q 2>&1 | tail -5
```

Expected: all StaticFilesTest tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/test/java/dev/claudony/frontend/StaticFilesTest.java \
        src/main/resources/META-INF/resources/app/index.html
git commit -m "feat: add mesh panel HTML structure to dashboard (Refs #60)"
```

---

## Task 7: CSS — mesh panel styles

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/style.css`

- [ ] **Step 1: Add mesh panel CSS**

At the end of `src/main/resources/META-INF/resources/app/style.css`, append:

```css
/* ── Mesh Panel ──────────────────────────────────────────────────────────── */

#mesh-panel {
    width: 300px;
    min-width: 300px;
    background: var(--surface);
    border-left: 1px solid var(--border);
    display: flex;
    flex-direction: column;
    overflow: hidden;
    transition: width 0.2s ease, min-width 0.2s ease;
    flex-shrink: 0;
}
#mesh-panel.collapsed {
    width: 0;
    min-width: 0;
}

.mesh-header {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 10px 12px;
    border-bottom: 1px solid var(--border);
    flex-shrink: 0;
}
.mesh-title {
    font-size: 11px;
    font-weight: 600;
    color: var(--active);
    flex: 1;
    letter-spacing: 0.05em;
}
.mesh-view-switcher { display: flex; gap: 2px; }
.mesh-view-btn {
    background: transparent;
    border: 1px solid var(--border);
    color: var(--text-dim);
    padding: 3px 7px;
    font-size: 12px;
    border-radius: var(--radius);
}
.mesh-view-btn.active {
    background: var(--active);
    color: #000;
    border-color: var(--active);
}
.mesh-collapse-btn {
    background: transparent;
    border: none;
    color: var(--text-dim);
    padding: 3px 6px;
    font-size: 13px;
    cursor: pointer;
}
.mesh-collapse-btn:hover { color: var(--text); }

.mesh-body {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
}
.mesh-expand {
    position: fixed;
    right: 0;
    top: 50%;
    transform: translateY(-50%);
    background: var(--surface);
    border: 1px solid var(--border);
    border-right: none;
    color: var(--active);
    padding: 10px 5px;
    writing-mode: vertical-rl;
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.05em;
    border-radius: var(--radius) 0 0 var(--radius);
    cursor: pointer;
    z-index: 5;
}

/* Mesh view content */
.mesh-empty {
    color: var(--text-dim);
    font-size: 13px;
    text-align: center;
    padding: 24px 0;
}
.mesh-section { margin-bottom: 12px; }
.mesh-label {
    font-size: 10px;
    font-weight: 600;
    color: var(--text-dim);
    letter-spacing: 0.05em;
    margin-bottom: 4px;
}
.mesh-presence { display: flex; flex-wrap: wrap; gap: 4px; margin-bottom: 4px; }
.mesh-instance {
    background: var(--active);
    color: #000;
    border-radius: 10px;
    padding: 1px 8px;
    font-size: 11px;
}
.mesh-channel-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 5px 6px;
    border-radius: var(--radius);
    cursor: pointer;
    margin-bottom: 2px;
}
.mesh-channel-item:hover { background: var(--bg); }
.mesh-channel-name { font-size: 12px; color: var(--active); }
.mesh-channel-count {
    font-size: 10px;
    background: var(--bg);
    border-radius: 8px;
    padding: 0 5px;
    color: var(--text-dim);
}
.mesh-msg { font-size: 11px; margin-bottom: 3px; line-height: 1.4; }
.mesh-sender { color: var(--active); font-weight: 600; margin-right: 4px; }
.mesh-content { color: var(--text-dim); }
.mesh-dim { color: var(--text-dim); font-size: 12px; }

.mesh-channel-select {
    width: 100%;
    background: var(--surface);
    color: var(--text);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 4px 6px;
    font-size: 12px;
    margin-bottom: 8px;
}
.mesh-timeline { overflow-y: auto; }
.mesh-feed { overflow-y: auto; }
.mesh-feed-item {
    font-size: 11px;
    margin-bottom: 4px;
    display: flex;
    gap: 5px;
    align-items: baseline;
    line-height: 1.4;
}
.mesh-channel-tag {
    color: var(--active);
    font-size: 10px;
    flex-shrink: 0;
}
.mesh-presence-footer {
    margin-top: 8px;
    padding-top: 8px;
    border-top: 1px solid var(--border);
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
}
```

- [ ] **Step 2: Run full suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: 239+ tests, 0 failures.

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/app/style.css
git commit -m "feat: add mesh panel CSS — layout, view switcher, view content styles (Refs #60)"
```

---

## Task 8: JS — MeshPanel class + strategies

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/dashboard.js`

- [ ] **Step 1: Append MeshPanel and strategy classes to dashboard.js**

At the very end of `src/main/resources/META-INF/resources/app/dashboard.js`, append:

```javascript
// ── Mesh Panel ───────────────────────────────────────────────────────────────

class MeshPanel {
    constructor() {
        this.panel = document.getElementById('mesh-panel');
        this.body = document.getElementById('mesh-body');
        this.expandBtn = document.getElementById('mesh-expand-btn');
        this.activeView = localStorage.getItem('mesh-view') || 'overview';
        this.collapsed = localStorage.getItem('mesh-collapsed') === 'true';
        this.data = { channels: [], instances: [], feed: [] };
        this.strategy = null;
    }

    async init() {
        this._wireButtons();
        if (this.collapsed) this._applyCollapsed();
        this._renderActiveView(); // show empty state immediately
        try {
            const cfg = await fetch('/api/mesh/config').then(r => r.json());
            this.strategy = cfg.strategy === 'sse'
                ? new SseMeshStrategy('/api/mesh/events', this)
                : new PollingMeshStrategy('/api/mesh', cfg.interval, this);
            this.strategy.start();
        } catch (e) {
            // Server not available — panel shows empty state, no crash
        }
    }

    _wireButtons() {
        document.getElementById('mesh-collapse-btn')
            .addEventListener('click', () => this.collapse());
        document.getElementById('mesh-expand-btn')
            .addEventListener('click', () => this.expand());
        document.querySelectorAll('.mesh-view-btn').forEach(btn => {
            btn.addEventListener('click', () => this.switchView(btn.dataset.view));
        });
        this._updateViewBtns();
    }

    _updateViewBtns() {
        document.querySelectorAll('.mesh-view-btn').forEach(b => {
            b.classList.toggle('active', b.dataset.view === this.activeView);
        });
    }

    update(data) {
        this.data = data;
        this._renderActiveView();
    }

    switchView(name) {
        this.activeView = name;
        localStorage.setItem('mesh-view', name);
        this._updateViewBtns();
        this._renderActiveView();
    }

    _renderActiveView() {
        const views = { overview: OverviewView, channel: ChannelView, feed: FeedView };
        const view = views[this.activeView] || OverviewView;
        view.render(this.body, this.data, this);
    }

    collapse() {
        this.collapsed = true;
        localStorage.setItem('mesh-collapsed', 'true');
        this._applyCollapsed();
    }

    expand() {
        this.collapsed = false;
        localStorage.setItem('mesh-collapsed', 'false');
        this.panel.classList.remove('collapsed');
        this.expandBtn.style.display = 'none';
    }

    _applyCollapsed() {
        this.panel.classList.add('collapsed');
        this.expandBtn.style.display = '';
    }
}

class PollingMeshStrategy {
    constructor(baseUrl, interval, panel) {
        this.baseUrl = baseUrl;
        this.interval = interval;
        this.panel = panel;
        this.timer = null;
    }

    start() {
        this._poll();
        this.timer = setInterval(() => this._poll(), this.interval);
    }

    stop() { clearInterval(this.timer); }

    async _poll() {
        try {
            const [channels, instances, feed] = await Promise.all([
                fetch(this.baseUrl + '/channels').then(r => r.json()),
                fetch(this.baseUrl + '/instances').then(r => r.json()),
                fetch(this.baseUrl + '/feed?limit=100').then(r => r.json()),
            ]);
            this.panel.update({ channels, instances, feed });
        } catch (e) {
            // Silently degrade — panel keeps showing last data
        }
    }
}

class SseMeshStrategy {
    constructor(url, panel) {
        this.url = url;
        this.panel = panel;
        this.source = null;
    }

    start() {
        this.source = new EventSource(this.url);
        this.source.addEventListener('mesh-update', e => {
            try { this.panel.update(JSON.parse(e.data)); } catch (_) {}
        });
    }

    stop() { this.source?.close(); }
}
```

- [ ] **Step 2: Run full suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: 239+ tests, 0 failures (JS changes don't affect Java tests).

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/app/dashboard.js
git commit -m "feat: add MeshPanel, PollingMeshStrategy, SseMeshStrategy to dashboard.js (Refs #60)"
```

---

## Task 9: JS — Three view renderers + initialisation

**Files:**
- Modify: `src/main/resources/META-INF/resources/app/dashboard.js`

- [ ] **Step 1: Append view renderers and init to dashboard.js**

At the very end of `dashboard.js` (after the strategy classes), append:

```javascript
// ── View Renderers ───────────────────────────────────────────────────────────

const OverviewView = {
    render(container, data, panel) {
        const { channels, instances, feed } = data;
        if (!channels.length && !instances.length) {
            container.innerHTML = '<div class="mesh-empty">No active channels</div>';
            return;
        }
        const presence = instances.length
            ? instances.map(i => `<span class="mesh-instance">${i.instanceId}</span>`).join('')
            : '<span class="mesh-dim">No agents online</span>';

        const channelItems = channels.length
            ? channels.map(ch => `
                <div class="mesh-channel-item" onclick="meshPanel.switchView('channel')">
                    <span class="mesh-channel-name">#${ch.name}</span>
                    <span class="mesh-channel-count">${ch.messageCount}</span>
                </div>`).join('')
            : '<div class="mesh-dim">No channels</div>';

        const recentMsgs = (feed || []).slice(0, 5).map(m =>
            `<div class="mesh-msg">
                <span class="mesh-sender">${m.sender || m.agent_id || '?'}</span>
                <span class="mesh-content">${String(m.content || '').substring(0, 60)}</span>
            </div>`
        ).join('');

        container.innerHTML = `
            <div class="mesh-section">
                <div class="mesh-label">ONLINE</div>
                <div class="mesh-presence">${presence}</div>
            </div>
            <div class="mesh-section">
                <div class="mesh-label">CHANNELS</div>
                ${channelItems}
            </div>
            ${recentMsgs ? `<div class="mesh-section"><div class="mesh-label">RECENT</div>${recentMsgs}</div>` : ''}
        `;
    }
};

const ChannelView = {
    _selected: null,
    render(container, data, panel) {
        const { channels, feed } = data;
        if (!channels.length) {
            container.innerHTML = '<div class="mesh-empty">No active channels</div>';
            return;
        }
        if (!this._selected || !channels.find(c => c.name === this._selected)) {
            this._selected = channels[0].name;
        }
        const opts = channels.map(ch =>
            `<option value="${ch.name}" ${ch.name === this._selected ? 'selected' : ''}>#${ch.name}</option>`
        ).join('');
        const msgs = (feed || [])
            .filter(m => m.channel === this._selected)
            .map(m => `<div class="mesh-msg">
                <span class="mesh-sender">${m.sender || m.agent_id || '?'}</span>
                <span class="mesh-content">${String(m.content || '').substring(0, 80)}</span>
            </div>`).join('') || '<div class="mesh-dim">No messages</div>';
        container.innerHTML = `
            <select class="mesh-channel-select"
                onchange="ChannelView._selected=this.value; meshPanel._renderActiveView()">
                ${opts}
            </select>
            <div class="mesh-timeline">${msgs}</div>
        `;
    }
};

const FeedView = {
    render(container, data, panel) {
        const { feed, instances } = data;
        if (!feed || !feed.length) {
            container.innerHTML = '<div class="mesh-empty">No recent activity</div>';
            return;
        }
        const items = feed.slice(0, 50).map(m => `
            <div class="mesh-feed-item">
                <span class="mesh-dim">${String(m.created_at || '').substring(11, 19)}</span>
                <span class="mesh-channel-tag">#${m.channel || '?'}</span>
                <span class="mesh-sender">${m.sender || m.agent_id || '?'}</span>
                <span class="mesh-content">${String(m.content || '').substring(0, 55)}</span>
            </div>`
        ).join('');
        const presenceFooter = instances?.length
            ? `<div class="mesh-presence-footer">${instances.map(i =>
                `<span class="mesh-instance">&#9679; ${i.instanceId}</span>`).join('')}</div>`
            : '';
        container.innerHTML = `<div class="mesh-feed">${items}</div>${presenceFooter}`;
    }
};

// ── Initialise ───────────────────────────────────────────────────────────────
const meshPanel = new MeshPanel();
meshPanel.init();
```

- [ ] **Step 2: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format -q 2>&1 | tail -3
```

Expected: 239+ tests, 0 failures.

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/resources/app/dashboard.js
git commit -m "feat: add OverviewView, ChannelView, FeedView renderers and mesh panel init (Refs #60)"
gh issue close 60 --repo mdproctor/claudony --comment "Mesh panel HTML, CSS, and JS complete."
```

---

## Task 10: Playwright E2E tests (TDD)

**Files:**
- Create: `src/test/java/dev/claudony/e2e/MeshPanelE2ETest.java`

Use the E2E issue number from Task 1 (e.g. `Refs #61`).

- [ ] **Step 1: Write all 4 Playwright tests**

Create `src/test/java/dev/claudony/e2e/MeshPanelE2ETest.java`:

```java
package dev.claudony.e2e;

import com.microsoft.playwright.Locator;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

/**
 * Playwright E2E tests for the Mesh observation panel.
 *
 * <p>Tests panel structure, collapse/expand behaviour, view switching, and empty
 * state rendering. Does NOT test live message content (requires real Qhorus agent
 * activity — deferred to future sessions).
 *
 * <p>Run with: mvn test -Pe2e -Dtest=MeshPanelE2ETest
 */
@QuarkusTest
class MeshPanelE2ETest extends PlaywrightBase {

    @Test
    void meshPanel_visibleOnDashboard() {
        page.navigate(BASE_URL + "/app/");

        // Panel header is visible
        var panel = page.locator("#mesh-panel");
        assertThat(panel.isVisible()).isTrue();

        // "MESH" title is present
        var title = page.locator(".mesh-title");
        assertThat(title.isVisible()).isTrue();
        assertThat(title.textContent()).isEqualTo("MESH");

        // All three view buttons are present
        assertThat(page.locator(".mesh-view-btn").count()).isEqualTo(3);
        assertThat(page.locator(".mesh-view-btn[data-view='overview']").isVisible()).isTrue();
        assertThat(page.locator(".mesh-view-btn[data-view='channel']").isVisible()).isTrue();
        assertThat(page.locator(".mesh-view-btn[data-view='feed']").isVisible()).isTrue();
    }

    @Test
    void meshPanel_collapseAndExpand() {
        page.navigate(BASE_URL + "/app/");

        // Ensure panel starts expanded (clear localStorage)
        page.evaluate("localStorage.removeItem('mesh-collapsed')");
        page.reload();

        var panel = page.locator("#mesh-panel");
        var expandBtn = page.locator("#mesh-expand-btn");
        var collapseBtn = page.locator("#mesh-collapse-btn");

        // Panel is visible initially
        assertThat(panel.isVisible()).isTrue();

        // Click collapse
        collapseBtn.click();

        // Panel width collapses (has 'collapsed' class)
        panel.waitFor(new Locator.WaitForOptions().setTimeout(2000));
        assertThat(panel.evaluate("el => el.classList.contains('collapsed')")).isEqualTo(true);

        // Expand button appears
        assertThat(expandBtn.isVisible()).isTrue();

        // Click expand
        expandBtn.click();

        // Panel is restored
        assertThat(panel.evaluate("el => el.classList.contains('collapsed')")).isEqualTo(false);
        assertThat(expandBtn.isVisible()).isFalse();

        // State survives reload
        page.reload();
        assertThat(panel.evaluate("el => el.classList.contains('collapsed')")).isEqualTo(false);
    }

    @Test
    void meshPanel_viewSwitching_updatesActiveButton() {
        page.navigate(BASE_URL + "/app/");

        // Clear view preference
        page.evaluate("localStorage.removeItem('mesh-view')");
        page.reload();

        var overviewBtn = page.locator(".mesh-view-btn[data-view='overview']");
        var channelBtn  = page.locator(".mesh-view-btn[data-view='channel']");
        var feedBtn     = page.locator(".mesh-view-btn[data-view='feed']");

        // Overview is active by default
        assertThat(overviewBtn.evaluate("el => el.classList.contains('active')")).isEqualTo(true);

        // Click Channel
        channelBtn.click();
        assertThat(channelBtn.evaluate("el => el.classList.contains('active')")).isEqualTo(true);
        assertThat(overviewBtn.evaluate("el => el.classList.contains('active')")).isEqualTo(false);

        // Click Feed
        feedBtn.click();
        assertThat(feedBtn.evaluate("el => el.classList.contains('active')")).isEqualTo(true);
        assertThat(channelBtn.evaluate("el => el.classList.contains('active')")).isEqualTo(false);

        // View persists across reload
        page.reload();
        assertThat(feedBtn.evaluate("el => el.classList.contains('active')")).isEqualTo(true);
    }

    @Test
    void meshPanel_emptyState_showsMessageNotError() {
        page.navigate(BASE_URL + "/app/");

        // Wait for the panel's init() to complete (fetch /api/mesh/config)
        page.waitForFunction("() => document.getElementById('mesh-body').textContent.trim().length > 0",
                new com.microsoft.playwright.Page.WaitForFunctionOptions().setTimeout(5000));

        var body = page.locator("#mesh-body");

        // Overview: empty state text visible, no JS error
        page.locator(".mesh-view-btn[data-view='overview']").click();
        var overviewText = body.textContent().trim();
        assertThat(overviewText).isNotEmpty();
        // Must not show an uncaught error message
        assertThat(overviewText).doesNotContainIgnoringCase("uncaught");
        assertThat(overviewText).doesNotContainIgnoringCase("typeerror");

        // Channel view
        page.locator(".mesh-view-btn[data-view='channel']").click();
        var channelText = body.textContent().trim();
        assertThat(channelText).isNotEmpty();

        // Feed view
        page.locator(".mesh-view-btn[data-view='feed']").click();
        var feedText = body.textContent().trim();
        assertThat(feedText).isNotEmpty();

        // No JS console errors
        // (Playwright captures console errors via page.onConsoleMessage — checked implicitly
        //  by verifying the page didn't break. Full console error assertion is opt-in.)
    }
}
```

- [ ] **Step 2: Run E2E tests to verify all 4 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e \
  -Dtest=MeshPanelE2ETest -Dno-format -q 2>&1 | tail -5
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

If any test fails, diagnose and fix before proceeding.

- [ ] **Step 3: Commit and close E2E issue**

```bash
git add src/test/java/dev/claudony/e2e/MeshPanelE2ETest.java
git commit -m "test: add MeshPanelE2ETest — 4 Playwright tests for panel structure and behaviour (Refs #61)"
gh issue close 61 --repo mdproctor/claudony --comment "4 Playwright tests passing."
```

---

## Task 11: Final verification + docs update + close epic

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Run the full default test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dno-format
```

Expected: 239+ tests, 0 failures, 0 errors. Note the exact count.

- [ ] **Step 2: Update CLAUDE.md test count**

Update the `**NNN tests passing**` line to reflect the actual count from Step 1.

- [ ] **Step 3: Commit and close epic**

```bash
git add CLAUDE.md
git commit -m "docs: update test count after Mesh observation panel"
```

Close the epic (replace #58 with actual epic number):
```bash
gh issue close 58 --repo mdproctor/claudony --comment "All three child issues complete. Mesh panel live — channels, instances, and messages visible in the dashboard right panel."
git push origin main
```

---

## Self-Review

**Spec coverage:**
- ✅ Layout Option A (right panel) → Task 6 HTML, Task 7 CSS
- ✅ View system (Overview/Channel/Feed + icon switcher) → Task 8/9 JS
- ✅ Collapse/expand with localStorage → Task 8 JS
- ✅ Configurable refresh strategy (poll/sse) → Task 2 config, Task 8 strategies
- ✅ MeshResource — all 6 endpoints → Tasks 2–5
- ✅ QhorusMcpTools facade (no new queries) → Tasks 3–4
- ✅ Unknown channel returns [] not 404 → Task 4
- ✅ Feed tags messages with channel name → Task 4
- ✅ MeshResourceTest — 6 endpoint tests + 2 auth tests → Tasks 2–5
- ✅ StaticFilesTest assertion → Task 6
- ✅ MeshPanelE2ETest — 4 Playwright tests → Task 10
- ✅ GitHub epic + child issues → Task 1
- ✅ All commits reference issues → every commit step

**Placeholder scan:** No TBDs. All code is complete. View renderers have real content including empty states.

**Type consistency:** `QhorusMcpTools.ChannelDetail` and `QhorusMcpTools.InstanceInfo` used consistently in Task 3. `MeshConfig` record defined in Task 2 and used only there. `meshPanel` global variable defined in Task 9 and referenced in Task 9's view renderers. `ChannelView._selected` set and read only within `ChannelView`.
