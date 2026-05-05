# Case Context Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enhance the channel panel on the session view to display CaseHub task context (role, status, elapsed, prior worker lineage) and auto-select the case's Qhorus channel when viewing a CaseHub worker session.

**Architecture:** One new `GET /api/sessions/{id}/lineage` endpoint in `SessionResource` injects `CaseLineageQuery` (always-available SPI). The channel panel in `terminal.js` gains case-aware rendering: when the session has a `caseId`, a compact case-context section is injected between the dropdown row and the message feed; `loadChannels()` auto-selects the `case-{caseId}/work` channel.

**Tech Stack:** Java 21, Quarkus 3.32.2, RestAssured, JUnit 5 (backend); Playwright + QuarkusTest (E2E); vanilla JS, CSS custom properties (frontend).

**Issue:** casehubio/claudony#77 — all commits must include `Refs #77` or `Closes #77`.

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `claudony-app/src/main/java/dev/claudony/server/SessionResource.java` | Modify | Add `GET /{id}/lineage`; inject `CaseLineageQuery` |
| `claudony-app/src/test/java/dev/claudony/server/SessionLineageResourceTest.java` | Create | QuarkusTest — happy path, no caseId, 404, non-UUID caseId robustness |
| `claudony-app/src/main/resources/META-INF/resources/app/style.css` | Modify | `.ch-case-header`, `.ch-lineage`, `.ch-lineage-row` — dark-theme CSS |
| `claudony-app/src/main/resources/META-INF/resources/app/terminal.js` | Modify | Case header rendering, lineage fetch, channel auto-select, cleanup |
| `claudony-app/src/test/java/dev/claudony/e2e/CaseContextPanelE2ETest.java` | Create | E2E — case header visible, lineage toggle, channel auto-select, no-caseId guard |
| `CLAUDE.md` | Modify | Update test count (425 → 433), add `SessionLineageResourceTest` and `CaseContextPanelE2ETest` to test inventory, add `GET /{id}/lineage` to API description |
| `docs/DESIGN.md` | Modify | Update dashboard section to describe case context panel |

---

## Task 1: Backend — lineage endpoint (TDD)

**Files:**
- Modify: `claudony-app/src/main/java/dev/claudony/server/SessionResource.java`
- Create: `claudony-app/src/test/java/dev/claudony/server/SessionLineageResourceTest.java`

- [ ] **Step 1.1 — Write the failing tests**

Create `claudony-app/src/test/java/dev/claudony/server/SessionLineageResourceTest.java`:

```java
package dev.claudony.server;

import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class SessionLineageResourceTest {

    @Inject SessionRegistry registry;

    @AfterEach
    void cleanup() {
        registry.all().stream().map(Session::id).toList().forEach(registry::remove);
    }

    // ── Happy path: session with caseId → 200 empty list (EmptyCaseLineageQuery is active)

    @Test
    void sessionWithCaseId_returns200WithList() {
        var now = Instant.now();
        registry.register(new Session("lin-s1", "claudony-lin-s1", "/tmp", "claude",
                SessionStatus.IDLE, now, now, Optional.empty(),
                Optional.of("550e8400-e29b-41d4-a716-446655440000"), Optional.of("researcher")));

        given().when().get("/api/sessions/lin-s1/lineage")
                .then()
                .statusCode(200)
                .body("$", hasSize(0));          // EmptyCaseLineageQuery returns []
    }

    // ── No caseId: session without caseId → 200 empty list

    @Test
    void sessionWithoutCaseId_returns200EmptyList() {
        var now = Instant.now();
        registry.register(new Session("lin-s2", "claudony-lin-s2", "/tmp", "claude",
                SessionStatus.IDLE, now, now, Optional.empty(),
                Optional.empty(), Optional.empty()));

        given().when().get("/api/sessions/lin-s2/lineage")
                .then()
                .statusCode(200)
                .body("$", hasSize(0));
    }

    // ── 404: unknown session id

    @Test
    void unknownSession_returns404() {
        given().when().get("/api/sessions/does-not-exist/lineage")
                .then()
                .statusCode(404);
    }

    // ── Robustness: non-UUID caseId stored in session → 200 empty list, not 500

    @Test
    void nonUuidCaseId_returns200EmptyListNotError() {
        var now = Instant.now();
        registry.register(new Session("lin-s3", "claudony-lin-s3", "/tmp", "claude",
                SessionStatus.IDLE, now, now, Optional.empty(),
                Optional.of("not-a-uuid"), Optional.of("analyst")));

        given().when().get("/api/sessions/lin-s3/lineage")
                .then()
                .statusCode(200)
                .body("$", hasSize(0));
    }
}
```

- [ ] **Step 1.2 — Run to confirm tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionLineageResourceTest -q 2>&1 | tail -20
```

Expected: compilation error — `GET /api/sessions/{id}/lineage` does not exist yet.

- [ ] **Step 1.3 — Implement the endpoint**

In `claudony-app/src/main/java/dev/claudony/server/SessionResource.java`:

Add the import:
```java
import dev.claudony.casehub.CaseLineageQuery;
```

Add the field (alongside existing `@Inject` fields):
```java
@Inject CaseLineageQuery lineageQuery;
```

Add the endpoint method (after the `get(@PathParam("id") String id)` method):
```java
@GET
@Path("/{id}/lineage")
public Response getLineage(@PathParam("id") String id) {
    return registry.find(id)
            .map(session -> {
                if (session.caseId().isEmpty()) {
                    return Response.ok(List.of()).build();
                }
                UUID caseUuid;
                try {
                    caseUuid = UUID.fromString(session.caseId().get());
                } catch (IllegalArgumentException e) {
                    return Response.ok(List.of()).build();
                }
                return Response.ok(lineageQuery.findCompletedWorkers(caseUuid)).build();
            })
            .orElse(Response.status(404).build());
}
```

Add `UUID` to the imports (already present via existing code — verify; if not: `import java.util.UUID;`).

- [ ] **Step 1.4 — Run to confirm all four tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionLineageResourceTest -q 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 1.5 — Run full module tests to check for regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -q 2>&1 | tail -10
```

Expected: all tests pass.

- [ ] **Step 1.6 — Commit**

```bash
git add claudony-app/src/main/java/dev/claudony/server/SessionResource.java \
        claudony-app/src/test/java/dev/claudony/server/SessionLineageResourceTest.java
git commit -m "feat(sessions): add GET /api/sessions/{id}/lineage endpoint

Exposes CaseLineageQuery via REST for the case context panel.
Returns [] for sessions without caseId or with non-UUID caseId.

Refs #77"
```

---

## Task 2: CSS — case context panel styles

**Files:**
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/style.css`

- [ ] **Step 2.1 — Add CSS after the `.ch-empty` block**

Find the line `/* Channel panel empty state */` in `style.css` (around line 317). Add the following CSS block after the `.ch-empty` rule:

```css
/* ── Case context header (injected when session has caseId) ────────────── */

.ch-case-header {
    border-bottom: 1px solid var(--border);
    padding: 6px 10px 4px;
    flex-shrink: 0;
}
.ch-case-info {
    display: flex;
    align-items: center;
    gap: 6px;
    margin-bottom: 4px;
}
.ch-case-role {
    font-size: 12px;
    font-weight: 600;
    color: var(--text);
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
.ch-case-elapsed {
    font-size: 11px;
    color: var(--text-dim);
    flex-shrink: 0;
}
.ch-lineage-toggle {
    display: flex;
    align-items: center;
    gap: 5px;
    cursor: pointer;
    padding: 2px 0;
    font-size: 11px;
    color: var(--text-dim);
    user-select: none;
}
.ch-lineage-toggle:hover { color: var(--text); }
.ch-lineage-chevron {
    font-size: 9px;
    transition: transform 0.15s ease;
    display: inline-block;
}

/* ── Lineage section (collapsible) ─────────────────────────────────────── */

.ch-lineage {
    border-bottom: 1px solid var(--border);
    padding: 4px 10px;
    flex-shrink: 0;
    overflow: hidden;
}
.ch-lineage.ch-lineage-hidden { display: none; }
.ch-lineage-empty {
    font-size: 11px;
    color: var(--text-dim);
    font-style: italic;
    padding: 2px 0;
}
.ch-lineage-row {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 3px 0;
    font-size: 11px;
    border-bottom: 1px solid var(--border);
}
.ch-lineage-row:last-child { border-bottom: none; }
.ch-lineage-name {
    color: var(--active);
    font-weight: 600;
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
.ch-lineage-time {
    color: var(--text-dim);
    flex-shrink: 0;
    font-family: var(--mono);
}
```

- [ ] **Step 2.2 — Commit**

```bash
git add claudony-app/src/main/resources/META-INF/resources/app/style.css
git commit -m "feat(dashboard): add CSS for case context and lineage sections

Adds .ch-case-header, .ch-lineage, and .ch-lineage-row styles
using existing dark-theme CSS variables.

Refs #77"
```

---

## Task 3: JS — case context header and lineage (write failing E2E, then implement)

**Files:**
- Create: `claudony-app/src/test/java/dev/claudony/e2e/CaseContextPanelE2ETest.java`
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/terminal.js`

### Step 3a — Write failing E2E tests

- [ ] **Step 3.1 — Create the E2E test file**

Create `claudony-app/src/test/java/dev/claudony/e2e/CaseContextPanelE2ETest.java`:

```java
package dev.claudony.e2e;

import com.microsoft.playwright.Locator;
import dev.claudony.server.SessionRegistry;
import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.casehub.qhorus.testing.InMemoryChannelStore;
import io.casehub.qhorus.testing.InMemoryMessageStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * E2E tests for the case context section of the channel panel.
 *
 * <p>Covers: case header visible when caseId is present, absent when not,
 * lineage toggle expands/collapses, channel auto-selects to case-{caseId}/work.
 *
 * <p>Lineage always shows "0 prior workers" — EmptyCaseLineageQuery is the active bean.
 */
@QuarkusTest
class CaseContextPanelE2ETest extends PlaywrightBase {

    @Inject SessionRegistry registry;
    @Inject QhorusMcpTools tools;
    @Inject InMemoryChannelStore channelStore;
    @Inject InMemoryMessageStore messageStore;

    private static final String CASE_ID = "550e8400-e29b-41d4-a716-446655440001";

    @BeforeEach
    void setup() {
        var now = Instant.now();
        // CaseHub session — has caseId and roleName
        registry.register(new Session("ctx-case-session", "claudony-ctx-case", "/tmp", "claude",
                SessionStatus.ACTIVE, now.minusSeconds(600), now, Optional.empty(),
                Optional.of(CASE_ID), Optional.of("researcher")));

        // Standalone session — no caseId
        registry.register(new Session("ctx-standalone", "claudony-ctx-standalone", "/tmp", "claude",
                SessionStatus.IDLE, now, now, Optional.empty(),
                Optional.empty(), Optional.empty()));
    }

    @AfterEach
    void cleanup() {
        registry.all().stream().map(Session::id).toList().forEach(registry::remove);
        messageStore.clear();
        channelStore.clear();
    }

    private void openChannelPanel() {
        page.locator("#ch-toggle-btn").click();
        page.locator("#channel-panel:not(.collapsed)").waitFor(
                new Locator.WaitForOptions().setTimeout(5000));
    }

    // ── AC 1: case header visible when session has caseId ────────────────────

    @Test
    void caseSession_showsCaseHeaderWithRoleAndStatus() {
        page.navigate(BASE_URL + "/app/session.html?id=ctx-case-session&name=researcher");
        openChannelPanel();

        // Case header must be present
        page.locator(".ch-case-header").waitFor(new Locator.WaitForOptions().setTimeout(4000));
        assertThat(page.locator(".ch-case-header").count()).isEqualTo(1);

        // Role name must appear
        assertThat(page.locator(".ch-case-role").textContent()).isEqualTo("researcher");

        // Status dot must have the 'active' class (session status = ACTIVE)
        assertThat(page.locator(".ch-case-header .worker-status-dot").getAttribute("class"))
                .contains("active");

        // Elapsed display must be present (non-empty — session is 10 minutes old)
        var elapsed = page.locator(".ch-case-elapsed").textContent();
        assertThat(elapsed).isNotBlank();
    }

    // ── AC 2: no case header for standalone session ───────────────────────────

    @Test
    void standaloneSession_noCaseHeader() {
        page.navigate(BASE_URL + "/app/session.html?id=ctx-standalone&name=standalone");
        openChannelPanel();

        // Give JS time to settle
        page.waitForTimeout(1000);

        assertThat(page.locator(".ch-case-header").count())
                .as("No case header for standalone session")
                .isEqualTo(0);
    }

    // ── AC 3: lineage toggle expands and collapses ────────────────────────────

    @Test
    void lineageToggle_expandsAndCollapses() {
        page.navigate(BASE_URL + "/app/session.html?id=ctx-case-session&name=researcher");
        openChannelPanel();

        page.locator(".ch-lineage-toggle").waitFor(new Locator.WaitForOptions().setTimeout(4000));

        // Lineage section starts hidden
        assertThat(page.locator(".ch-lineage").getAttribute("class"))
                .contains("ch-lineage-hidden");

        // Click toggle → expands
        page.locator(".ch-lineage-toggle").click();
        assertThat(page.locator(".ch-lineage").getAttribute("class"))
                .doesNotContain("ch-lineage-hidden");

        // Toggle row shows "0 prior workers" (EmptyCaseLineageQuery)
        assertThat(page.locator(".ch-lineage-count").textContent())
                .isEqualTo("0 prior workers");

        // Click again → collapses
        page.locator(".ch-lineage-toggle").click();
        assertThat(page.locator(".ch-lineage").getAttribute("class"))
                .contains("ch-lineage-hidden");
    }

    // ── AC 4: channel auto-selects to case-{caseId}/work ─────────────────────

    @Test
    void caseSession_autoSelectsCaseChannel() {
        var caseChannelName = "case-" + CASE_ID + "/work";
        tools.createChannel(caseChannelName, "Case work channel", "APPEND", null);

        page.navigate(BASE_URL + "/app/session.html?id=ctx-case-session&name=researcher");
        openChannelPanel();

        // Wait for the case channel to be auto-selected in the dropdown
        page.locator("#ch-select option[value='" + caseChannelName + "']").waitFor(
                new Locator.WaitForOptions().setTimeout(5000));

        var selectedValue = page.evaluate("document.getElementById('ch-select').value");
        assertThat(selectedValue.toString()).isEqualTo(caseChannelName);
    }
}
```

- [ ] **Step 3.2 — Verify tests compile but fail (functionality not yet implemented)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Pe2e \
  -Dtest=CaseContextPanelE2ETest -q 2>&1 | tail -20
```

Expected: 4 failures — `.ch-case-header` not found, auto-select not working.

### Step 3b — Implement case header and lineage in terminal.js

- [ ] **Step 3.3 — Add case context state variables**

In `terminal.js`, find the block starting with `// ── Channel panel ─────`:

```javascript
// ── Channel panel ─────────────────────────────────────────────────────────

    var chPanel      = document.getElementById('channel-panel');
    var chSelect     = document.getElementById('ch-select');
    // ... existing vars ...
    var POLL_MS        = 3000;
```

After the existing channel panel variable declarations (after `var POLL_MS = 3000;`), add:

```javascript
    // Case context state (populated from session fetch)
    var sessionCaseId    = null;
    var sessionRoleName  = null;
    var sessionStatus    = null;
    var sessionCreatedAt = null;

    // Case context DOM references (created dynamically)
    var chCaseHeaderEl  = null;
    var chLineageEl     = null;
    var lineageExpanded = false;
    var lineagePollTimer = null;
    var elapsedTicker    = null;
```

- [ ] **Step 3.4 — Add helper functions**

Add these functions after the `formatTime` function (around line 185):

```javascript
    function caseElapsed(fromMs) {
        var diffM = Math.floor((Date.now() - fromMs) / 60000);
        if (diffM < 1) return '<1m';
        if (diffM < 60) return diffM + 'm';
        return Math.floor(diffM / 60) + 'h ' + (diffM % 60) + 'm';
    }

    function renderCaseHeader() {
        if (chCaseHeaderEl) chCaseHeaderEl.remove();
        chCaseHeaderEl = document.createElement('div');
        chCaseHeaderEl.className = 'ch-case-header';

        var role = sessionRoleName ? sessionRoleName.replace(/^claudony-worker-/, '') : '—';
        var status = (sessionStatus || 'idle').toLowerCase();
        var elapsed = sessionCreatedAt ? caseElapsed(sessionCreatedAt.getTime()) : '—';

        chCaseHeaderEl.innerHTML =
            '<div class="ch-case-info">' +
                '<span class="ch-case-role">' + escHtml(role) + '</span>' +
                '<span class="worker-status-dot ' + escHtml(status) + '"></span>' +
                '<span class="ch-case-elapsed">' + escHtml(elapsed) + '</span>' +
            '</div>' +
            '<div class="ch-lineage-toggle">' +
                '<span class="ch-lineage-chevron">▶</span>' +
                '<span class="ch-lineage-count">Loading…</span>' +
            '</div>';

        chPanel.insertBefore(chCaseHeaderEl, chFeed);

        chCaseHeaderEl.querySelector('.ch-lineage-toggle').addEventListener('click', toggleLineage);

        if (elapsedTicker) clearInterval(elapsedTicker);
        elapsedTicker = setInterval(function () {
            var el = chCaseHeaderEl && chCaseHeaderEl.querySelector('.ch-case-elapsed');
            if (el && sessionCreatedAt) el.textContent = caseElapsed(sessionCreatedAt.getTime());
        }, 30000);
    }

    function toggleLineage() {
        lineageExpanded = !lineageExpanded;
        if (chLineageEl) chLineageEl.classList.toggle('ch-lineage-hidden', !lineageExpanded);
        var chevron = chCaseHeaderEl && chCaseHeaderEl.querySelector('.ch-lineage-chevron');
        if (chevron) chevron.style.transform = lineageExpanded ? 'rotate(90deg)' : '';
    }

    function renderLineage(workers) {
        var countEl = chCaseHeaderEl && chCaseHeaderEl.querySelector('.ch-lineage-count');
        var n = workers.length;
        if (countEl) countEl.textContent = n + ' prior worker' + (n === 1 ? '' : 's');

        if (chLineageEl) chLineageEl.remove();
        chLineageEl = document.createElement('div');
        chLineageEl.className = 'ch-lineage ch-lineage-hidden';

        if (n === 0) {
            chLineageEl.innerHTML = '<div class="ch-lineage-empty">No prior workers</div>';
        } else {
            workers.forEach(function (w) {
                var row = document.createElement('div');
                row.className = 'ch-lineage-row';
                var name = (w.workerName || w.workerId || '?').replace(/^claudony-worker-/, '');
                var start = w.startedAt
                    ? new Date(w.startedAt).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
                    : '?';
                var end = w.completedAt
                    ? new Date(w.completedAt).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
                    : '?';
                var durMs = (w.startedAt && w.completedAt)
                    ? new Date(w.completedAt) - new Date(w.startedAt) : 0;
                var dur = durMs > 0 ? Math.ceil(durMs / 60000) + 'm' : '?';
                row.innerHTML =
                    '<span class="ch-lineage-name">' + escHtml(name) + '</span>' +
                    '<span class="ch-lineage-time">' + escHtml(start) + '→' + escHtml(end) +
                        ' (' + escHtml(dur) + ')</span>';
                chLineageEl.appendChild(row);
            });
        }
        chPanel.insertBefore(chLineageEl, chFeed);
        if (lineageExpanded) chLineageEl.classList.remove('ch-lineage-hidden');
    }

    function loadLineage() {
        fetch('/api/sessions/' + sessionId + '/lineage')
            .then(function (r) { return r.ok ? r.json() : []; })
            .then(renderLineage)
            .catch(function () { renderLineage([]); });
        if (lineagePollTimer) clearTimeout(lineagePollTimer);
        lineagePollTimer = setTimeout(loadLineage, 60000);
    }
```

- [ ] **Step 3.5 — Modify `openPanel()` to render case context**

Find the existing `openPanel()` function:

```javascript
    function openPanel() {
        chPanel.classList.remove('collapsed');
        loadChannels();
    }
```

Replace it with:

```javascript
    function openPanel() {
        chPanel.classList.remove('collapsed');
        if (sessionCaseId && !chCaseHeaderEl) {
            renderCaseHeader();
            loadLineage();
        }
        loadChannels();
    }
```

- [ ] **Step 3.6 — Modify `closePanel()` to clean up timers**

Find the existing `closePanel()` function:

```javascript
    function closePanel() {
        chPanel.classList.add('collapsed');
        clearTimeout(chPollTimer);
    }
```

Replace it with:

```javascript
    function closePanel() {
        chPanel.classList.add('collapsed');
        clearTimeout(chPollTimer);
        clearTimeout(lineagePollTimer);
        clearInterval(elapsedTicker);
    }
```

- [ ] **Step 3.7 — Populate case context from session fetch**

Find the existing session fetch at the bottom of `terminal.js` (the call to `/api/sessions/' + sessionId`):

```javascript
    // Fetch current session to get caseId, then start panel
    fetch('/api/sessions/' + sessionId)
        .then(function (r) { return r.ok ? r.json() : null; })
        .then(function (session) {
            if (session && session.caseId) {
                activeCaseId = session.caseId;
                openCasePanel();
                pollWorkers();
                casePoller = setInterval(pollWorkers, 3000);
            } else {
                showCasePlaceholder('No case assigned.');
            }
        })
        .catch(function () {
            showCasePlaceholder('No case assigned.');
        });
```

Replace it with (adds three lines to populate the channel panel context):

```javascript
    // Fetch current session to get caseId, then start panel
    fetch('/api/sessions/' + sessionId)
        .then(function (r) { return r.ok ? r.json() : null; })
        .then(function (session) {
            if (session && session.caseId) {
                activeCaseId    = session.caseId;
                sessionCaseId   = session.caseId;
                sessionRoleName = session.roleName || null;
                sessionStatus   = session.status  || null;
                sessionCreatedAt = session.createdAt ? new Date(session.createdAt) : null;
                openCasePanel();
                pollWorkers();
                casePoller = setInterval(pollWorkers, 3000);
            } else {
                showCasePlaceholder('No case assigned.');
            }
        })
        .catch(function () {
            showCasePlaceholder('No case assigned.');
        });
```

- [ ] **Step 3.8 — Add cleanup to beforeunload**

Find the existing `window.addEventListener('beforeunload', ...)` block:

```javascript
    window.addEventListener('beforeunload', function () {
        if (casePoller) clearInterval(casePoller);
    });
```

Replace it with:

```javascript
    window.addEventListener('beforeunload', function () {
        if (casePoller) clearInterval(casePoller);
        clearTimeout(lineagePollTimer);
        clearInterval(elapsedTicker);
    });
```

---

## Task 4: JS — channel auto-select (implement, then verify E2E)

**Files:**
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/terminal.js`

- [ ] **Step 4.1 — Add `selectCaseChannel` function**

Add the following function immediately after the `loadLineage` function added in Task 3:

```javascript
    function selectCaseChannel(caseId) {
        var prefix = 'case-' + caseId + '/';
        var opts = Array.from(chSelect.options);
        var workOpt = opts.find(function (o) { return o.value === prefix + 'work'; });
        var anyOpt  = opts.find(function (o) { return o.value.indexOf(prefix) === 0; });
        var target  = workOpt || anyOpt;
        if (target) {
            chSelect.value = target.value;
            selectChannel(target.value);
        }
    }
```

- [ ] **Step 4.2 — Modify `loadChannels()` to call `selectCaseChannel` after populating**

Find the existing `loadChannels()` function:

```javascript
    function loadChannels() {
        fetch('/api/mesh/channels').then(function (r) { return r.json(); }).then(function (channels) {
            // Prefer channels matching the session name pattern
            channels.sort(function (a, b) { return a.name.localeCompare(b.name); });
            channels.forEach(function (ch) {
                var opt = document.createElement('option');
                opt.value = ch.name;
                opt.textContent = ch.name + (ch.message_count ? ' (' + ch.message_count + ')' : '');
                chSelect.appendChild(opt);
            });
            // Auto-select if channel name passed in URL
            var preselect = params.get('channel');
            if (preselect) {
                chSelect.value = preselect;
                selectChannel(preselect);
                openPanel();
            }
        }).catch(function () {});
    }
```

Replace it with:

```javascript
    function loadChannels() {
        fetch('/api/mesh/channels').then(function (r) { return r.json(); }).then(function (channels) {
            channels.sort(function (a, b) { return a.name.localeCompare(b.name); });
            chSelect.innerHTML = '<option value="">— select channel —</option>';
            channels.forEach(function (ch) {
                var opt = document.createElement('option');
                opt.value = ch.name;
                opt.textContent = ch.name + (ch.message_count ? ' (' + ch.message_count + ')' : '');
                chSelect.appendChild(opt);
            });
            // Priority 1: URL ?channel= preselect (opens panel if not already open)
            var preselect = params.get('channel');
            if (preselect) {
                chSelect.value = preselect;
                selectChannel(preselect);
                openPanel();
                return;
            }
            // Priority 2: Case channel auto-select (no panel open required — already open)
            if (sessionCaseId) {
                selectCaseChannel(sessionCaseId);
            }
        }).catch(function () {});
    }
```

Note: `chSelect.innerHTML = '<option value="">…</option>'` is added to reset the dropdown before repopulating — the existing code used `chSelect.appendChild` without clearing first, which would duplicate options on repeated `loadChannels()` calls. This fix is included here.

- [ ] **Step 4.3 — Run the E2E tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Pe2e \
  -Dtest=CaseContextPanelE2ETest -q 2>&1 | tail -20
```

Expected: `Tests run: 4, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 4.4 — Run existing channel panel E2E tests to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Pe2e \
  -Dtest=ChannelPanelE2ETest,CaseWorkerPanelE2ETest -q 2>&1 | tail -10
```

Expected: all 11 existing tests pass.

- [ ] **Step 4.5 — Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -10
```

Expected: all tests pass (claudony-casehub + claudony-app).

- [ ] **Step 4.6 — Commit frontend changes**

```bash
git add claudony-app/src/main/resources/META-INF/resources/app/terminal.js \
        claudony-app/src/test/java/dev/claudony/e2e/CaseContextPanelE2ETest.java
git commit -m "feat(dashboard): case context panel — header, lineage, channel auto-select

When the session has a caseId:
- Injects case header (role, status dot, elapsed) above message feed
- Loads prior worker lineage (collapsible, polls every 60s)
- Auto-selects case-{caseId}/work channel on panel open

Closes #77"
```

---

## Task 5: Documentation updates

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 5.1 — Update CLAUDE.md test count and inventory**

In `CLAUDE.md`, find:

```
**425 tests passing** (as of 2026-04-29, all modules): 119 in `claudony-casehub` + 306 in `claudony-app`. Zero failures, zero errors.
```

Replace with:

```
**433 tests passing** (as of 2026-05-01, all modules): 119 in `claudony-casehub` + 314 in `claudony-app`. Zero failures, zero errors.
```

(+4 `SessionLineageResourceTest` + +4 `CaseContextPanelE2ETest` = +8 tests. Verify exact count from `mvn test` output before committing.)

- [ ] **Step 5.2 — Update CLAUDE.md test inventory**

Find the line:
```
- `server/` — TmuxService (real tmux; includes `displayMessage` tests), SessionRegistry (+ findByCaseId, 3 tests), SessionResource (+ ?caseId= filter, 2 tests; caseId/roleName in response, 1 test), TerminalWebSocket, ServerStartup, SessionInputOutput, MeshResourceInterjectionTest, `model/SessionTest` (session model + touch())
```

Replace with:
```
- `server/` — TmuxService (real tmux; includes `displayMessage` tests), SessionRegistry (+ findByCaseId, 3 tests), SessionResource (+ ?caseId= filter, 2 tests; caseId/roleName in response, 1 test), SessionLineageResourceTest (4 tests: happy path, no caseId, 404, non-UUID caseId robustness), TerminalWebSocket, ServerStartup, SessionInputOutput, MeshResourceInterjectionTest, `model/SessionTest` (session model + touch())
```

Find the line:
```
- `e2e/` — ClaudeE2ETest (real `claude` CLI), PlaywrightSetupE2ETest (4 browser infra), DashboardE2ETest (7 dashboard UI), TerminalPageE2ETest (2: structure + proxy resize URL), ChannelPanelE2ETest (8: toggle, dropdown, timeline, badges, human sender, post message, cursor polling, Ctrl+K), CaseWorkerPanelE2ETest (3: standalone placeholder, CaseHub auto-expand + worker list, click-to-switch) — all via `mvn test -Pe2e -Dtest=...`, skipped in default run
```

Replace with:
```
- `e2e/` — ClaudeE2ETest (real `claude` CLI), PlaywrightSetupE2ETest (4 browser infra), DashboardE2ETest (7 dashboard UI), TerminalPageE2ETest (2: structure + proxy resize URL), ChannelPanelE2ETest (8: toggle, dropdown, timeline, badges, human sender, post message, cursor polling, Ctrl+K), CaseWorkerPanelE2ETest (3: standalone placeholder, CaseHub auto-expand + worker list, click-to-switch), CaseContextPanelE2ETest (4: case header with role/status, no header for standalone, lineage toggle expand/collapse, channel auto-select) — all via `mvn test -Pe2e -Dtest=...`, skipped in default run
```

- [ ] **Step 5.3 — Update CLAUDE.md file structure section**

Find:
```
└── app/
    ├── index.html + dashboard.js      — session management dashboard
    ├── session.html + terminal.js     — xterm.js terminal view + iPad key bar
    └── style.css                      — shared dark theme
```

Replace with:
```
└── app/
    ├── index.html + dashboard.js      — session management dashboard
    ├── session.html + terminal.js     — xterm.js terminal view + iPad key bar;
    │                                    when session.caseId set: case context panel
    │                                    (role, status, elapsed, lineage, channel auto-select)
    └── style.css                      — shared dark theme
```

- [ ] **Step 5.4 — Update CLAUDE.md Sessions API section**

Find the line in SessionResource description:
```
├── SessionResource.java            — REST API /api/sessions
```

Replace with:
```
├── SessionResource.java            — REST API /api/sessions (+ GET /{id}/lineage → CaseLineageQuery)
```

- [ ] **Step 5.5 — Update docs/DESIGN.md**

In `docs/DESIGN.md`, find the section describing the session/terminal view dashboard panels. Add a paragraph or bullet describing the case context panel. The exact location depends on the current DESIGN.md structure — search for "channel panel" or "side panel" and add adjacent to the description of the right panel:

```
The right channel panel is case-aware: when the session carries a `caseId`, a case-context 
section is injected above the message feed, showing the worker role, session status, and elapsed 
time. A collapsible lineage row lists prior completed workers (polled from 
`GET /api/sessions/{id}/lineage` every 60 s). The panel auto-selects the 
`case-{caseId}/work` Qhorus channel on open.
```

- [ ] **Step 5.6 — Commit documentation**

```bash
git add CLAUDE.md docs/DESIGN.md
git commit -m "docs: update test count and dashboard docs for case context panel (#77)

- Test count 425 → 433
- Add SessionLineageResourceTest and CaseContextPanelE2ETest to inventory
- Document GET /api/sessions/{id}/lineage in SessionResource entry
- Update DESIGN.md with case context panel description

Refs #77"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `GET /api/sessions/{id}/lineage` endpoint | Task 1 |
| Returns 404 for unknown session | Task 1 (test case 3) |
| Returns `[]` for session without caseId | Task 1 (test case 2) |
| Non-UUID caseId → graceful `[]`, not 500 | Task 1 (test case 4) |
| CSS for case header + lineage | Task 2 |
| Case header injected when caseId present | Task 3 |
| Role, status dot, elapsed shown in header | Task 3 (E2E AC1) |
| No case header when no caseId | Task 3 (E2E AC2) |
| Lineage toggle collapsed by default | Task 3 (E2E AC3) |
| Lineage toggle expands and collapses | Task 3 (E2E AC3) |
| Channel auto-selects `case-{caseId}/work` | Task 4 (E2E AC4) |
| Elapsed ticker updates every 30s | Task 3 (step 3.4, `elapsedTicker`) |
| Lineage re-polls every 60s | Task 3 (step 3.4, `loadLineage`) |
| Timers cleaned up on panel close / page unload | Task 3 (steps 3.6, 3.8) |
| Dropdown reset before repopulation (bug fix) | Task 4 (step 4.2) |
| Documentation updated | Task 5 |

**No gaps found.**
