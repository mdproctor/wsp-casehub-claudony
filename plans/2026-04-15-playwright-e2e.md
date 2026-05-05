# Playwright E2E Browser Testing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Playwright for Java as browser-based E2E test infrastructure with 12 tests covering architecture verification, dashboard golden paths, and terminal page structure.

**Architecture:** `PlaywrightBase` abstract class manages Playwright/Browser/Page lifecycle and injects the test API key via `setExtraHTTPHeaders`. Concrete test classes extend it and use `@QuarkusTest` so the server starts on port 8081 before the browser connects. All tests run under the existing `-Pe2e` Maven profile — no new profile needed.

**Tech Stack:** Java 21, Playwright for Java 1.52.0, JUnit 5, `@QuarkusTest`, Gson (transitive via Playwright), Maven `-Pe2e` profile

**Issue:** Refs mdproctor/claudony#49

---

## File Map

| File | Action |
|---|---|
| `pom.xml` | **Modify** — add `playwright` test dependency |
| `src/test/java/dev/claudony/e2e/PlaywrightBase.java` | **Create** — abstract base with Playwright lifecycle |
| `src/test/java/dev/claudony/e2e/PlaywrightSetupE2ETest.java` | **Create** — 4 architecture verification tests |
| `src/test/java/dev/claudony/e2e/DashboardE2ETest.java` | **Create** — 7 dashboard tests |
| `src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java` | **Create** — 1 terminal structure test |
| `CLAUDE.md` | **Modify** — Playwright install command, e2e test count |

---

## Task 1: Playwright Dependency + PlaywrightBase + Chromium Install

**Files:**
- Modify: `pom.xml`
- Create: `src/test/java/dev/claudony/e2e/PlaywrightBase.java`

### What this task does

Adds the Playwright for Java Maven dependency and creates the `PlaywrightBase` abstract class that all E2E test classes extend. Also installs Chromium (required before any browser test can run).

- [ ] **Step 1: Add Playwright dependency to pom.xml**

In `pom.xml`, add inside the `<dependencies>` block (alongside the other test dependencies):

```xml
<dependency>
    <groupId>com.microsoft.playwright</groupId>
    <artifactId>playwright</artifactId>
    <version>1.52.0</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Create PlaywrightBase**

Create `src/test/java/dev/claudony/e2e/PlaywrightBase.java`:

```java
package dev.claudony.e2e;

import com.microsoft.playwright.*;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;

import java.util.Map;

/**
 * Abstract base for all Playwright browser E2E tests.
 *
 * <p>Manages Playwright, Browser, BrowserContext, and Page lifecycle:
 * - Playwright + Browser created once per test class (@BeforeAll/@AfterAll)
 * - BrowserContext + Page created fresh per test (@BeforeEach/@AfterEach) — no state bleed
 *
 * <p>Auth: every request the page makes (page loads, fetch() calls) gets the test API key
 * via page.setExtraHTTPHeaders(). WebSocket connections do NOT inherit extra headers —
 * terminal WebSocket tests require a different auth strategy (deferred).
 *
 * <p>Headless by default. Run with -Dplaywright.headless=false for a visible browser.
 */
public abstract class PlaywrightBase {

    protected static Playwright playwright;
    protected static Browser browser;
    protected BrowserContext context;
    protected Page page;

    /** Base URL for the Quarkus test server. */
    protected static final String BASE_URL = "http://localhost:8081";

    /** Test API key — matches %test.claudony.agent.api-key in application.properties. */
    protected static final String API_KEY = "test-api-key-do-not-use-in-prod";

    @BeforeAll
    static void launchBrowser() {
        playwright = Playwright.create();
        var headless = Boolean.parseBoolean(System.getProperty("playwright.headless", "true"));
        browser = playwright.chromium().launch(
                new BrowserType.LaunchOptions().setHeadless(headless));
    }

    @AfterAll
    static void closeBrowser() {
        if (browser != null) browser.close();
        if (playwright != null) playwright.close();
    }

    @BeforeEach
    void createContextAndPage() {
        context = browser.newContext();
        page = context.newPage();
        // Inject API key into every HTTP request the page makes
        page.setExtraHTTPHeaders(Map.of("X-Api-Key", API_KEY));
    }

    @AfterEach
    void closeContext() {
        if (context != null) context.close();
    }
}
```

- [ ] **Step 3: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS` (Playwright classes resolve from the new dependency).

- [ ] **Step 4: Install Chromium (one-time per machine)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn exec:java \
  -e -D exec.mainClass=com.microsoft.playwright.CLI \
  -D exec.args="install chromium" 2>&1 | tail -5
```

Expected: Chromium downloads to `~/.cache/ms-playwright/`. On subsequent runs this is a no-op.

- [ ] **Step 5: Run existing test suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: 210 tests passing (Playwright dependency doesn't affect existing tests).

- [ ] **Step 6: Commit**

```bash
git add pom.xml src/test/java/dev/claudony/e2e/PlaywrightBase.java
git commit -m "feat: add Playwright for Java + PlaywrightBase E2E infrastructure (Refs #49)"
```

---

## Task 2: PlaywrightSetupE2ETest — Architecture Verification

**Files:**
- Create: `src/test/java/dev/claudony/e2e/PlaywrightSetupE2ETest.java`

These 4 tests verify the test infrastructure itself before any functional tests run. A failure here means "fix the setup before running other tests" — not a product bug.

- [ ] **Step 1: Write the 4 architecture tests**

Create `src/test/java/dev/claudony/e2e/PlaywrightSetupE2ETest.java`:

```java
package dev.claudony.e2e;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Architecture verification tests for the Playwright E2E test infrastructure.
 *
 * <p>These tests verify setup, not product features. Run these first on a new machine
 * or CI environment. A failure here means the infrastructure is broken — fix it before
 * running DashboardE2ETest or TerminalPageE2ETest.
 *
 * <p>Tests are intentionally ordered from simplest (Chromium launches) to most complex
 * (auth header injection blocks unauthenticated access).
 */
@QuarkusTest
class PlaywrightSetupE2ETest extends PlaywrightBase {

    /** Verifies Chromium is installed and can be launched. */
    @Test
    void playwright_chromiumLaunches() {
        assertThat(browser.isConnected()).isTrue();
        assertThat(page).isNotNull();
    }

    /** Verifies the Quarkus test server is reachable on port 8081. */
    @Test
    void playwright_canNavigateToServer() {
        var response = page.navigate(BASE_URL + "/app/");
        assertThat(response.status()).isEqualTo(200);
    }

    /**
     * Verifies page.setExtraHTTPHeaders() injects the API key and the server accepts it.
     * If this test fails, authenticated dashboard requests will silently fail too.
     */
    @Test
    void authHeader_allowsProtectedEndpoint() {
        var response = page.navigate(BASE_URL + "/api/sessions");
        assertThat(response.status()).isEqualTo(200);
    }

    /**
     * Verifies the test context is what provides access — not a misconfigured open server.
     * Creates a fresh context without any extra headers and confirms the server rejects it.
     */
    @Test
    void unauthenticated_context_isBlocked() {
        try (var unauthContext = browser.newContext();
             var unauthPage = unauthContext.newPage()) {
            var response = unauthPage.navigate(BASE_URL + "/api/sessions");
            // Server must reject unauthenticated requests with 401 (API) or 302 (page redirect)
            assertThat(response.status()).isIn(401, 302);
        }
    }
}
```

- [ ] **Step 2: Run setup tests — all 4 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=PlaywrightSetupE2ETest 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0, Skipped: 0`

If `playwright_chromiumLaunches` fails with "Executable doesn't exist", run the Chromium install from Task 1 Step 4.

If `unauthenticated_context_isBlocked` fails (status is 200 instead of 401/302), the server auth configuration is not working — do not proceed until this passes.

- [ ] **Step 3: Commit**

```bash
git add src/test/java/dev/claudony/e2e/PlaywrightSetupE2ETest.java
git commit -m "test: PlaywrightSetupE2ETest — 4 architecture verification tests (Refs #49)"
```

---

## Task 3: DashboardE2ETest — 7 Dashboard Tests

**Files:**
- Create: `src/test/java/dev/claudony/e2e/DashboardE2ETest.java`

Note: The `sessionCard_appearsAfterApiCreate` test creates a real tmux session via the REST API — tmux must be running (standard for any e2e test run). The test cleans up the session in `@AfterEach`.

- [ ] **Step 1: Write the 7 dashboard tests**

Create `src/test/java/dev/claudony/e2e/DashboardE2ETest.java`:

```java
package dev.claudony.e2e;

import com.microsoft.playwright.Locator;
import com.microsoft.playwright.options.RequestOptions;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * E2E tests for the Claudony dashboard (/app/).
 *
 * <p>Each test navigates to the dashboard fresh — BrowserContext is reset between tests
 * by PlaywrightBase so there is no state bleed.
 *
 * <p>Auth: all page requests include the test API key via PlaywrightBase.setExtraHTTPHeaders,
 * including fetch() calls made by dashboard.js. Sessions are created/deleted via
 * page.request() which inherits the same headers.
 */
@QuarkusTest
class DashboardE2ETest extends PlaywrightBase {

    private String createdSessionId;

    @AfterEach
    void cleanupSession() {
        // page.request() inherits the API key header — no separate auth needed
        if (createdSessionId != null) {
            page.request().delete(BASE_URL + "/api/sessions/" + createdSessionId);
            createdSessionId = null;
        }
    }

    @Test
    void pageTitle_isClaudony() {
        page.navigate(BASE_URL + "/app/");
        assertThat(page.title()).isEqualTo("Claudony");
    }

    @Test
    void fleetPanel_visible_withNoPeersMessage() {
        page.navigate(BASE_URL + "/app/");
        var fleetPanel = page.locator("#fleet-panel");
        assertThat(fleetPanel.isVisible()).isTrue();
        // Default state when no peers are configured
        assertThat(page.locator(".peer-empty").textContent()).contains("No peers configured");
    }

    @Test
    void sessionGrid_showsEmptyState_whenNoSessions() {
        page.navigate(BASE_URL + "/app/");
        // Wait for the dashboard to finish its first poll (up to 5s)
        page.locator(".empty-state").waitFor(
                new Locator.WaitForOptions().setTimeout(6000));
        assertThat(page.locator(".empty-state").textContent()).contains("No active sessions");
    }

    @Test
    void newSessionDialog_opensAndCloses() {
        page.navigate(BASE_URL + "/app/");
        page.locator("#new-session-btn").click();
        var dialog = page.locator("#new-session-dialog");
        assertThat(dialog.isVisible()).isTrue();
        assertThat(page.locator("#new-session-form input[name='name']").isVisible()).isTrue();
        page.locator("#cancel-btn").click();
        assertThat(dialog.isVisible()).isFalse();
    }

    @Test
    void addPeerDialog_opensAndCloses() {
        page.navigate(BASE_URL + "/app/");
        page.locator("#add-peer-btn").click();
        var dialog = page.locator("#add-peer-dialog");
        assertThat(dialog.isVisible()).isTrue();
        // Verify all three form fields are present
        assertThat(page.locator("#add-peer-form input[name='url']").isVisible()).isTrue();
        assertThat(page.locator("#add-peer-form input[name='name']").isVisible()).isTrue();
        assertThat(page.locator("#add-peer-form select[name='terminalMode']").isVisible()).isTrue();
        page.locator("#cancel-peer-btn").click();
        assertThat(dialog.isVisible()).isFalse();
    }

    @Test
    void sessionCard_appearsAfterApiCreate() {
        // Create a session via REST API using the page's request context (inherits API key)
        var response = page.request().post(BASE_URL + "/api/sessions",
                RequestOptions.create()
                        .setHeader("Content-Type", "application/json")
                        .setData("{\"name\":\"playwright-test-session\"}"));
        assertThat(response.status()).isEqualTo(201);
        createdSessionId = response.json()
                .getAsJsonObject().get("id").getAsString();

        // Navigate to dashboard and wait for card (dashboard polls every 5s — allow 10s)
        page.navigate(BASE_URL + "/app/");
        page.locator(".session-card").waitFor(
                new Locator.WaitForOptions().setTimeout(10000));

        // Card renders with the correct session name (prefix stripped by displayName())
        assertThat(page.locator(".card-name").first().textContent())
                .isEqualTo("playwright-test-session");
        // Status badge is present and non-blank
        assertThat(page.locator(".badge").first().textContent().trim()).isNotBlank();
    }

    @Test
    void unauthenticated_redirectsToLogin() {
        // Create a context with no API key header
        try (var unauthContext = browser.newContext();
             var unauthPage = unauthContext.newPage()) {
            unauthPage.navigate(BASE_URL + "/app/");
            // Auth overlay appears (dashboard.js showAuthDialog) or redirect to login
            var redirectedToLogin = unauthPage.url().contains("/auth/login");
            var authOverlayShown = unauthPage.locator("#auth-overlay").count() > 0;
            assertThat(redirectedToLogin || authOverlayShown)
                    .withFailMessage("Expected auth redirect or overlay, but URL was: " + unauthPage.url())
                    .isTrue();
        }
    }
}
```

- [ ] **Step 2: Run dashboard tests — expect one failure initially**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=DashboardE2ETest 2>&1 | tail -15
```

Expected initial result: 6 pass, 1 fails — `sessionCard_appearsAfterApiCreate` fails because the card name shows `claudony-playwright-test-session` instead of `playwright-test-session`.

**Why:** `dashboard.js` `displayName()` function still strips the old `remotecc-` prefix, not the current `claudony-` prefix. This is a pre-existing bug the test just caught.

- [ ] **Step 3: Fix the displayName() bug in dashboard.js**

In `src/main/resources/META-INF/resources/app/dashboard.js`, find:

```javascript
    function displayName(name) {
        return name.replace(/^remotecc-/, '');
    }
```

Replace with:

```javascript
    function displayName(name) {
        return name.replace(/^claudony-/, '');
    }
```

- [ ] **Step 4: Run dashboard tests again — all 7 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=DashboardE2ETest 2>&1 | tail -10
```

Expected: `Tests run: 7, Failures: 0, Errors: 0, Skipped: 0`

Other common failure modes:
- `sessionCard_appearsAfterApiCreate` timeout — tmux may not be running; confirm with `tmux ls`
- `fleetPanel_visible_withNoPeersMessage` fails — check `#fleet-panel` exists in `index.html`
- `unauthenticated_redirectsToLogin` fails — check `dashboard.js` `showAuthDialog()` renders `#auth-overlay`

- [ ] **Step 5: Commit both the test and the displayName fix together**

```bash
git add src/test/java/dev/claudony/e2e/DashboardE2ETest.java \
        src/main/resources/META-INF/resources/app/dashboard.js
git commit -m "test: DashboardE2ETest — 7 dashboard E2E tests; fix displayName() remotecc→claudony prefix (Refs #49)"
```

---

## Task 4: TerminalPageE2ETest — Terminal Page Structure

**Files:**
- Create: `src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java`

This test navigates to the terminal page with a fake session ID. The WebSocket connection fails (session doesn't exist) — the test asserts the page degrades gracefully and shows "reconnecting", which is exactly what `terminal.js` does on close.

- [ ] **Step 1: Write the terminal page test**

Create `src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java`:

```java
package dev.claudony.e2e;

import com.microsoft.playwright.Locator;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * E2E test for the terminal page (/app/session.html).
 *
 * <p>Tests page structure only — WebSocket connection is not authenticated
 * via extra headers (browsers don't support custom headers on WebSocket upgrades).
 * The WebSocket fails gracefully (reconnecting state), which is what we assert.
 *
 * <p>Full terminal I/O testing (connecting to a live tmux session, verifying output,
 * testing PROXY resize) is deferred to the "Expand E2E coverage" epic.
 */
@QuarkusTest
class TerminalPageE2ETest extends PlaywrightBase {

    @Test
    void terminalPage_loadsStructure_andDegracefullyHandlesFailedWebSocket() {
        // Navigate with a fake session ID — the page loads, WebSocket fails, shows reconnecting
        page.navigate(BASE_URL + "/app/session.html?id=fake-id&name=test-terminal");

        // xterm.js container must be rendered in the DOM
        assertThat(page.locator("#terminal-container").isVisible()).isTrue();

        // Status badge shows "reconnecting" after WebSocket close (terminal.js ws.onclose handler)
        // Wait up to 3s for the reconnect cycle to begin
        page.locator("#status-badge").waitFor(
                new Locator.WaitForOptions().setTimeout(3000));
        assertThat(page.locator("#status-badge").textContent()).contains("reconnecting");

        // Session name is shown in the header
        assertThat(page.locator("#session-name").textContent()).isEqualTo("test-terminal");
    }
}
```

- [ ] **Step 2: Run terminal page test — must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=TerminalPageE2ETest 2>&1 | tail -10
```

Expected: `Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`

If `#status-badge` isn't found: check `session.html` has an element with `id="status-badge"`. From reading `terminal.js`: `var badge = document.getElementById('status-badge')` — this element must exist in `session.html`.

- [ ] **Step 3: Run full E2E suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e 2>&1 | tail -10
```

Expected: `Tests run: 12, Failures: 0, Errors: 0, Skipped: 0` (4 setup + 7 dashboard + 1 terminal)

- [ ] **Step 4: Commit**

```bash
git add src/test/java/dev/claudony/e2e/TerminalPageE2ETest.java
git commit -m "test: TerminalPageE2ETest — terminal page structure and graceful WS failure (Refs #49)"
```

---

## Task 5: CLAUDE.md Update

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add Playwright install command to Build and Test section**

In `CLAUDE.md`, find the "Build and Test" section. Add after the existing test commands:

```bash
# Install Chromium for browser E2E tests (one-time per machine)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn exec:java \
  -e -D exec.mainClass=com.microsoft.playwright.CLI \
  -D exec.args="install chromium"

# Run browser E2E tests (requires Chromium installed above)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e

# Run with visible browser (local debugging)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dplaywright.headless=false
```

- [ ] **Step 2: Update e2e test description**

Find the line describing the `e2e/` test category and update it:

Old:
```
- `e2e/` — ClaudeE2ETest (real `claude` CLI via `mvn test -Pe2e`, skipped in default run)
```

New:
```
- `e2e/` — ClaudeE2ETest (real `claude` CLI), PlaywrightSetupE2ETest (browser infra), DashboardE2ETest (dashboard UI), TerminalPageE2ETest (terminal page structure) — all via `mvn test -Pe2e`, skipped in default run
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add Playwright install command and e2e test descriptions to CLAUDE.md (Refs #49)"
```

---

## E2E Manual Verification

After all tasks:

```bash
# Run all e2e tests headless
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e

# Run with visible browser to watch the tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e \
  -Dplaywright.headless=false \
  -Dtest=DashboardE2ETest
```

Watching the visible browser confirms the tests are exercising real UI behaviour, not just passing by coincidence.
