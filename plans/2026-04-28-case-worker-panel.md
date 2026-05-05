# Case Worker Panel Implementation Plan

> **Note (2026-05-05):** The 3-second polling described in this plan was replaced by SSE push in #104.
> See `docs/superpowers/specs/2026-05-05-sse-case-worker-panel.md` and
> `docs/superpowers/plans/2026-05-05-sse-case-worker-panel.md`.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a left panel to `session.html` that shows CaseHub worker teammates and lets you click to switch terminals, with a placeholder for standalone sessions.

**Architecture:** Add `caseId` + `roleName` to the `Session` record and stamp them during CaseHub provisioning. `SessionRegistry.findByCaseId()` drives a new `?caseId=` filter on `GET /api/sessions`. `terminal.js` fetches the current session on load, auto-expands the case panel when `caseId` is present, and polls every 3 s for live status.

**Tech Stack:** Java 21 records, Quarkus 3.32.2, vanilla JS, xterm.js, Playwright (E2E). Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`.

---

## File Map

| File | Change |
|---|---|
| `claudony-core/src/main/java/dev/claudony/server/model/Session.java` | Add `caseId`, `roleName` fields; update `withStatus`, `withLastActive` |
| `claudony-core/src/main/java/dev/claudony/server/SessionRegistry.java` | Add `findByCaseId(String)` |
| `claudony-app/src/main/java/dev/claudony/server/model/SessionResponse.java` | Add nullable `caseId`, `roleName` |
| `claudony-app/src/main/java/dev/claudony/server/SessionResource.java` | Add `?caseId=` filter to `list()` |
| `claudony-app/src/main/java/dev/claudony/server/ServerStartup.java` | Update `new Session(...)` call |
| `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerProvisioner.java` | Stamp `caseId` + `roleName` on Session |
| `claudony-app/src/main/resources/META-INF/resources/app/session.html` | Add Workers button + `<aside id="case-panel">` |
| `claudony-app/src/main/resources/META-INF/resources/app/style.css` | Add case panel styles |
| `claudony-app/src/main/resources/META-INF/resources/app/terminal.js` | Add case panel JS logic |
| `claudony-app/src/test/java/dev/claudony/server/model/SessionTest.java` | Add caseId/roleName propagation tests |
| `claudony-app/src/test/java/dev/claudony/server/SessionRegistryTest.java` | Add findByCaseId tests; fix all `new Session(...)` calls |
| `claudony-app/src/test/java/dev/claudony/server/expiry/StatusAwareExpiryPolicyTest.java` | Fix `new Session(...)` call |
| `claudony-app/src/test/java/dev/claudony/server/expiry/UserInteractionExpiryPolicyTest.java` | Fix `new Session(...)` call |
| `claudony-app/src/test/java/dev/claudony/server/expiry/TerminalOutputExpiryPolicyTest.java` | Fix `new Session(...)` call |
| `claudony-app/src/test/java/dev/claudony/server/expiry/SessionIdleSchedulerTest.java` | Fix `new Session(...)` calls (4×) |
| `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java` | Fix `new Session(...)` call |
| `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerProvisionerTest.java` | Add caseId/roleName assertions |
| `claudony-app/src/test/java/dev/claudony/e2e/CaseWorkerPanelE2ETest.java` | New: Playwright E2E tests |
| `CLAUDE.md` | Update test count, Session model description |
| `docs/DESIGN.md` | Update Session model section, SessionRegistry section, REST endpoint list |

---

## Task 1: Update Session record — add caseId + roleName

**Files:**
- Modify: `claudony-core/src/main/java/dev/claudony/server/model/Session.java`
- Modify: `claudony-app/src/test/java/dev/claudony/server/model/SessionTest.java`

- [ ] **Step 1: Write failing tests for caseId/roleName fields and propagation**

  In `SessionTest.java`, add three new tests after the existing ones:

  ```java
  @Test
  void caseIdAndRoleNameDefaultToEmpty() {
      var now = Instant.now();
      var session = new Session("id-1", "myproject", "/home/user/proj",
              "claude", SessionStatus.IDLE, now, now, Optional.empty(),
              Optional.empty(), Optional.empty());
      assertTrue(session.caseId().isEmpty());
      assertTrue(session.roleName().isEmpty());
  }

  @Test
  void withStatusPreservesCaseIdAndRoleName() {
      var now = Instant.now();
      var session = new Session("id-1", "myproject", "/home/user/proj",
              "claude", SessionStatus.IDLE, now, now, Optional.empty(),
              Optional.of("case-abc"), Optional.of("researcher"));
      var updated = session.withStatus(SessionStatus.ACTIVE);
      assertEquals(Optional.of("case-abc"), updated.caseId());
      assertEquals(Optional.of("researcher"), updated.roleName());
  }

  @Test
  void withLastActivePreservesCaseIdAndRoleName() {
      var now = Instant.now();
      var session = new Session("id-1", "myproject", "/home/user/proj",
              "claude", SessionStatus.IDLE, now, now, Optional.empty(),
              Optional.of("case-abc"), Optional.of("coder"));
      var updated = session.withLastActive();
      assertEquals(Optional.of("case-abc"), updated.caseId());
      assertEquals(Optional.of("coder"), updated.roleName());
  }
  ```

- [ ] **Step 2: Run the tests — expect compile failure**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionTest 2>&1 | tail -20
  ```
  Expected: compile error — `Session` constructor does not have 10 args.

- [ ] **Step 3: Update Session record**

  Replace the entire `Session.java`:

  ```java
  package dev.claudony.server.model;

  import java.time.Instant;
  import java.util.Optional;

  public record Session(
          String id,
          String name,
          String workingDir,
          String command,
          SessionStatus status,
          Instant createdAt,
          Instant lastActive,
          Optional<String> expiryPolicy,
          Optional<String> caseId,
          Optional<String> roleName) {

      public Session withStatus(SessionStatus newStatus) {
          return new Session(id, name, workingDir, command, newStatus, createdAt, Instant.now(),
                  expiryPolicy, caseId, roleName);
      }

      public Session withLastActive() {
          return new Session(id, name, workingDir, command, status, createdAt, Instant.now(),
                  expiryPolicy, caseId, roleName);
      }
  }
  ```

- [ ] **Step 4: Fix the three existing SessionTest constructors** (they now compile to 8 args — add `, Optional.empty(), Optional.empty()` to each):

  Line 13–14: `new Session("id-1", "myproject", "/home/user/proj", "claude", SessionStatus.IDLE, now, now, Optional.empty(), Optional.empty(), Optional.empty())`

  Line 27–28: same pattern with `Optional.of("terminal-output")` for expiryPolicy.

  Line 40–41: same pattern with `Optional.of("status-aware")` for expiryPolicy.

  Do not change the new tests added in Step 1 — they already use 10 args.

- [ ] **Step 5: Run tests — expect PASS**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionTest 2>&1 | tail -10
  ```
  Expected: `Tests run: 7, Failures: 0, Errors: 0`.

---

## Task 2: Fix all remaining Session construction sites

**Files:**
- Modify: `claudony-app/src/main/java/dev/claudony/server/SessionResource.java` (lines 150–151, 203–205)
- Modify: `claudony-app/src/main/java/dev/claudony/server/ServerStartup.java` (line 71–75)
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerProvisioner.java` (line 86–87) — use `Optional.empty()` for now; Task 6 stamps real values
- Modify: `claudony-app/src/test/java/dev/claudony/server/SessionRegistryTest.java` — 7 constructor calls
- Modify: `claudony-app/src/test/java/dev/claudony/server/expiry/StatusAwareExpiryPolicyTest.java` — 1 call
- Modify: `claudony-app/src/test/java/dev/claudony/server/expiry/UserInteractionExpiryPolicyTest.java` — 1 call
- Modify: `claudony-app/src/test/java/dev/claudony/server/expiry/TerminalOutputExpiryPolicyTest.java` — 1 call
- Modify: `claudony-app/src/test/java/dev/claudony/server/expiry/SessionIdleSchedulerTest.java` — 4 calls
- Modify: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java` — 1 call

- [ ] **Step 1: Fix SessionResource.java**

  Line ~150 (`create()`):
  ```java
  var session = new Session(id, name, workingDir, command, SessionStatus.IDLE, now, now,
          Optional.ofNullable(req.expiryPolicy()), Optional.empty(), Optional.empty());
  ```

  Line ~203 (`rename()`):
  ```java
  var renamed = new Session(id, newTmuxName, session.workingDir(),
          session.command(), session.status(), session.createdAt(), Instant.now(),
          session.expiryPolicy(), session.caseId(), session.roleName());
  ```
  Note: `rename()` propagates `caseId` and `roleName` — a renamed worker is still the same worker.

- [ ] **Step 2: Fix ServerStartup.java**

  Lines ~71–75 (`bootstrapRegistry()`):
  ```java
  registry.register(new Session(
          UUID.randomUUID().toString(), name,
          "unknown", config.claudeCommand(),
          SessionStatus.IDLE, now, now, Optional.empty(),
          Optional.empty(), Optional.empty()));
  ```

- [ ] **Step 3: Fix ClaudonyWorkerProvisioner.java** (temporary — Task 6 will change this)

  Line ~86:
  ```java
  var session = new Session(sessionId, sessionName, defaultWorkingDir, command,
          SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty(),
          Optional.empty(), Optional.empty());
  ```

- [ ] **Step 4: Fix all test files** — add `, Optional.empty(), Optional.empty()` to every `new Session(...)` call that currently has 8 args.

  Files to update (all instances):
  - `SessionRegistryTest.java` — 7 calls (lines 31, 41, 49, 50, 57, 67–68)
  - `StatusAwareExpiryPolicyTest.java` — 1 call (line 70)
  - `UserInteractionExpiryPolicyTest.java` — 1 call (line 47)
  - `TerminalOutputExpiryPolicyTest.java` — 1 call (line 47)
  - `SessionIdleSchedulerTest.java` — 4 calls (lines 48, 62, 76, 92)
  - `ClaudonyWorkerStatusListenerTest.java` — 1 call (line 113)

- [ ] **Step 5: Run full test suite — expect PASS**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -20
  ```
  Expected: all tests pass (same count as before — 409).

- [ ] **Step 6: Commit**

  ```bash
  git add -p
  git commit -m "$(cat <<'EOF'
  feat: add caseId + roleName fields to Session record

  Propagated through withStatus/withLastActive. All construction
  sites updated with Optional.empty() for now; provisioner stamped in
  follow-on commit.

  Refs #76
  EOF
  )"
  ```

---

## Task 3: SessionResponse — add caseId + roleName

**Files:**
- Modify: `claudony-app/src/main/java/dev/claudony/server/model/SessionResponse.java`

- [ ] **Step 1: Write a failing test in SessionResourceTest**

  Find the existing `SessionResourceTest.java` (in `claudony-app/src/test/java/dev/claudony/server/`). Add:

  ```java
  @Test
  @TestSecurity(user = "test", roles = "user")
  void sessionResponseIncludesCaseIdAndRoleName() throws Exception {
      // Register a session with caseId and roleName
      var now = Instant.now();
      var session = new Session("case-session-id", "claudony-test-case", "/tmp", "claude",
              SessionStatus.IDLE, now, now, Optional.empty(),
              Optional.of("test-case-123"), Optional.of("researcher"));
      registry.register(session);

      var response = given().get("/api/sessions/case-session-id").then()
              .statusCode(200).extract().asString();

      assertTrue(response.contains("\"caseId\":\"test-case-123\""), "caseId missing: " + response);
      assertTrue(response.contains("\"roleName\":\"researcher\""), "roleName missing: " + response);

      registry.remove("case-session-id");
  }
  ```

- [ ] **Step 2: Run the test — expect FAIL**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionResourceTest#sessionResponseIncludesCaseIdAndRoleName 2>&1 | tail -15
  ```
  Expected: assertion failure — `caseId` not in response.

- [ ] **Step 3: Update SessionResponse**

  Replace the entire `SessionResponse.java`:

  ```java
  package dev.claudony.server.model;

  import com.fasterxml.jackson.annotation.JsonInclude;
  import java.time.Instant;

  @JsonInclude(JsonInclude.Include.NON_NULL)
  public record SessionResponse(
          String id,
          String name,
          String workingDir,
          String command,
          SessionStatus status,
          Instant createdAt,
          Instant lastActive,
          String wsUrl,
          String browserUrl,
          String instanceUrl,
          String instanceName,
          Boolean stale,
          String expiryPolicy,
          String caseId,
          String roleName) {

      public static SessionResponse from(Session session, int port, String effectivePolicy) {
          return new SessionResponse(
                  session.id(), session.name(), session.workingDir(), session.command(),
                  session.status(), session.createdAt(), session.lastActive(),
                  "ws://localhost:" + port + "/ws/" + session.id(),
                  "http://localhost:" + port + "/app/session/" + session.id(),
                  null, null, null, effectivePolicy,
                  session.caseId().orElse(null),
                  session.roleName().orElse(null));
      }

      public SessionResponse withInstance(String peerUrl, String peerName, boolean isStale) {
          return new SessionResponse(
                  id, name, workingDir, command, status, createdAt, lastActive,
                  wsUrl, browserUrl, peerUrl, peerName, isStale, expiryPolicy,
                  caseId, roleName);
      }
  }
  ```

- [ ] **Step 4: Run the test — expect PASS**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionResourceTest 2>&1 | tail -10
  ```
  Expected: all SessionResourceTest tests pass.

- [ ] **Step 5: Commit**

  ```bash
  git commit -am "$(cat <<'EOF'
  feat: add caseId + roleName to SessionResponse

  NON_NULL serialisation — fields absent for standalone sessions.

  Refs #76
  EOF
  )"
  ```

---

## Task 4: SessionRegistry.findByCaseId()

**Files:**
- Modify: `claudony-core/src/main/java/dev/claudony/server/SessionRegistry.java`
- Modify: `claudony-app/src/test/java/dev/claudony/server/SessionRegistryTest.java`

- [ ] **Step 1: Write three failing tests**

  Add to `SessionRegistryTest.java`:

  ```java
  @Test
  void findByCaseId_returnsSessionsWithMatchingCaseId() {
      var now = Instant.now();
      var s1 = new Session("s1", "w1", "/tmp", "claude", SessionStatus.IDLE,
              now.minusSeconds(10), now.minusSeconds(10), Optional.empty(),
              Optional.of("case-x"), Optional.of("researcher"));
      var s2 = new Session("s2", "w2", "/tmp", "claude", SessionStatus.ACTIVE,
              now, now, Optional.empty(),
              Optional.of("case-x"), Optional.of("coder"));
      var s3 = new Session("s3", "w3", "/tmp", "claude", SessionStatus.IDLE,
              now, now, Optional.empty(),
              Optional.of("case-y"), Optional.of("reviewer"));
      registry.register(s1);
      registry.register(s2);
      registry.register(s3);

      var result = registry.findByCaseId("case-x");
      assertThat(result).hasSize(2)
              .extracting(Session::id)
              .containsExactly("s1", "s2"); // ordered by createdAt
  }

  @Test
  void findByCaseId_returnsEmptyForUnknownCaseId() {
      assertTrue(registry.findByCaseId("nonexistent-case").isEmpty());
  }

  @Test
  void findByCaseId_excludesSessionsWithNoCaseId() {
      var now = Instant.now();
      var standalone = new Session("s-alone", "w1", "/tmp", "claude", SessionStatus.IDLE,
              now, now, Optional.empty(), Optional.empty(), Optional.empty());
      registry.register(standalone);
      assertTrue(registry.findByCaseId("case-z").isEmpty());
  }
  ```

  Add the import at the top of the file:
  ```java
  import java.util.List;
  import static org.assertj.core.api.Assertions.assertThat;
  ```

- [ ] **Step 2: Run — expect compile failure**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionRegistryTest 2>&1 | tail -15
  ```
  Expected: `cannot find symbol — method findByCaseId`.

- [ ] **Step 3: Implement findByCaseId in SessionRegistry**

  Add after the `touch()` method:

  ```java
  public List<Session> findByCaseId(String caseId) {
      return sessions.values().stream()
              .filter(s -> s.caseId().map(caseId::equals).orElse(false))
              .sorted(java.util.Comparator.comparing(Session::createdAt))
              .toList();
  }
  ```

  Add the import at the top:
  ```java
  import java.util.List;
  ```

- [ ] **Step 4: Run — expect PASS**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionRegistryTest 2>&1 | tail -10
  ```
  Expected: `Tests run: 10, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

  ```bash
  git commit -am "$(cat <<'EOF'
  feat: SessionRegistry.findByCaseId() — ordered by createdAt

  Refs #76
  EOF
  )"
  ```

---

## Task 5: SessionResource — ?caseId= filter

**Files:**
- Modify: `claudony-app/src/main/java/dev/claudony/server/SessionResource.java`
- Modify: `claudony-app/src/test/java/dev/claudony/server/SessionResourceTest.java`

- [ ] **Step 1: Write two failing tests**

  In `SessionResourceTest.java`, add:

  ```java
  @Test
  @TestSecurity(user = "test", roles = "user")
  void listByCaseId_returnsOnlyMatchingSessions() {
      var now = Instant.now();
      var s1 = new Session("cq-s1", "claudony-worker-1", "/tmp", "claude",
              SessionStatus.ACTIVE, now.minusSeconds(5), now.minusSeconds(5), Optional.empty(),
              Optional.of("case-q"), Optional.of("researcher"));
      var s2 = new Session("cq-s2", "claudony-worker-2", "/tmp", "claude",
              SessionStatus.IDLE, now, now, Optional.empty(),
              Optional.of("case-q"), Optional.of("coder"));
      var s3 = new Session("cq-s3", "claudony-worker-3", "/tmp", "claude",
              SessionStatus.IDLE, now, now, Optional.empty(),
              Optional.of("case-other"), Optional.of("reviewer"));
      registry.register(s1);
      registry.register(s2);
      registry.register(s3);

      var result = given().queryParam("caseId", "case-q")
              .get("/api/sessions").then()
              .statusCode(200)
              .extract().jsonPath().getList("$");

      assertThat(result).hasSize(2);

      registry.remove("cq-s1");
      registry.remove("cq-s2");
      registry.remove("cq-s3");
  }

  @Test
  @TestSecurity(user = "test", roles = "user")
  void listByCaseId_returnsEmptyForUnknownCase() {
      var result = given().queryParam("caseId", "no-such-case")
              .get("/api/sessions").then()
              .statusCode(200)
              .extract().jsonPath().getList("$");

      assertThat(result).isEmpty();
  }
  ```

  Ensure imports include:
  ```java
  import static io.restassured.RestAssured.given;
  import static org.assertj.core.api.Assertions.assertThat;
  ```

- [ ] **Step 2: Run — expect FAIL**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionResourceTest#listByCaseId_returnsOnlyMatchingSessions 2>&1 | tail -15
  ```
  Expected: test returns all sessions (filter not implemented).

- [ ] **Step 3: Update SessionResource.list()**

  Change the `list()` method signature and add early return:

  ```java
  @GET
  public List<SessionResponse> list(
          @QueryParam("local") @DefaultValue("false") boolean localOnly,
          @QueryParam("caseId") String caseId) {
      if (caseId != null) {
          return registry.findByCaseId(caseId).stream()
                  .map(s -> SessionResponse.from(s, config.port(), resolvedPolicy(s)))
                  .toList();
      }

      // Local sessions — always returned
      var result = new ArrayList<>(registry.all().stream()
              // ... rest of existing method unchanged
  ```

  Keep everything after the `if (caseId != null)` block exactly as it was.

- [ ] **Step 4: Run — expect PASS**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SessionResourceTest 2>&1 | tail -10
  ```
  Expected: all SessionResourceTest tests pass.

- [ ] **Step 5: Commit**

  ```bash
  git commit -am "$(cat <<'EOF'
  feat: GET /api/sessions?caseId= filter — local-only, bypasses federation

  Refs #76
  EOF
  )"
  ```

---

## Task 6: ClaudonyWorkerProvisioner — stamp caseId + roleName

**Files:**
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerProvisioner.java`
- Modify: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerProvisionerTest.java`

- [ ] **Step 1: Write a failing test**

  In `ClaudonyWorkerProvisionerTest.java`, add:

  ```java
  @Test
  void provision_stampsSessionWithCaseIdAndRoleName() throws Exception {
      var caseId = UUID.randomUUID();
      when(contextProvider.buildContext(anyString(), any())).thenReturn(
              new WorkerContext("researcher", caseId, null, List.of(),
                      PropagationContext.createRoot(), Map.of()));

      provisioner.provision(Set.of("researcher"), provisionContext(caseId));

      var captor = ArgumentCaptor.forClass(Session.class);
      verify(registry).register(captor.capture());
      var session = captor.getValue();
      assertThat(session.caseId()).contains(caseId.toString());
      assertThat(session.roleName()).contains("researcher");
  }
  ```

  Add import: `import org.mockito.ArgumentCaptor;`

- [ ] **Step 2: Run — expect FAIL**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyWorkerProvisionerTest#provision_stampsSessionWithCaseIdAndRoleName 2>&1 | tail -15
  ```
  Expected: assertion failure — `caseId` is empty.

- [ ] **Step 3: Stamp caseId + roleName in provision()**

  In `ClaudonyWorkerProvisioner.java`, replace the session creation line (~86):

  ```java
  var session = new Session(sessionId, sessionName, defaultWorkingDir, command,
          SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty(),
          Optional.ofNullable(context.caseId()).map(UUID::toString),
          Optional.of(roleName));
  ```

- [ ] **Step 4: Run — expect PASS**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub 2>&1 | tail -10
  ```
  Expected: all claudony-casehub tests pass.

- [ ] **Step 5: Run full suite to confirm no regressions**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -15
  ```
  Expected: all tests pass.

- [ ] **Step 6: Commit**

  ```bash
  git commit -am "$(cat <<'EOF'
  feat: stamp caseId + roleName on sessions provisioned by CaseHub

  Refs #76
  EOF
  )"
  ```

---

## Task 7: Frontend — session.html markup + style.css

**Files:**
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/session.html`
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/style.css`

- [ ] **Step 1: Add Workers button and case panel to session.html**

  In the `<header>`, add `id="workers-toggle-btn"` button **before** the existing `ch-toggle-btn`:

  ```html
  <button id="workers-toggle-btn" class="compose-btn" title="Toggle workers panel">Workers</button>
  <button id="ch-toggle-btn" class="compose-btn" title="Toggle channel panel (Ctrl+K)">Channels</button>
  ```

  In `#session-layout`, add the case panel as the **first** child (before `#terminal-container`):

  ```html
  <div id="session-layout">
      <aside id="case-panel" class="case-panel collapsed" aria-label="Case workers">
          <div class="case-panel-header">
              <span class="case-panel-title">Workers</span>
              <button id="case-close-btn" class="ch-close-btn" title="Close">&#10005;</button>
          </div>
          <div id="case-worker-list" class="case-worker-list"></div>
      </aside>
      <div id="terminal-container"></div>
      <aside id="channel-panel" ...  (existing, unchanged)
  ```

- [ ] **Step 2: Add case panel styles to style.css**

  Append after the `/* ── Channel panel ── */` block (after line ~234 in the current file):

  ```css
  /* ── Case worker panel ───────────────────────────────────────────────────── */

  .case-panel {
      width: 220px;
      min-width: 220px;
      display: flex;
      flex-direction: column;
      background: var(--surface);
      border-right: 1px solid var(--border);
      overflow: hidden;
      transition: width 0.2s ease, min-width 0.2s ease;
      flex-shrink: 0;
  }
  .case-panel.collapsed { width: 0; min-width: 0; }

  .case-panel-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 8px 10px;
      border-bottom: 1px solid var(--border);
      flex-shrink: 0;
  }

  .case-panel-title {
      font-size: 11px;
      font-weight: 600;
      color: var(--text-dim);
      text-transform: uppercase;
      letter-spacing: 0.05em;
  }

  .case-worker-list {
      flex: 1;
      overflow-y: auto;
      padding: 8px;
      display: flex;
      flex-direction: column;
      gap: 4px;
  }

  .case-worker-row {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 7px 8px;
      border-radius: var(--radius);
      cursor: pointer;
      font-size: 12px;
      border: 1px solid transparent;
      transition: background 0.1s;
  }
  .case-worker-row:hover { background: var(--bg); }
  .case-worker-row.active-worker { background: var(--bg); border-color: var(--accent); }

  .worker-status-dot {
      width: 8px;
      height: 8px;
      border-radius: 50%;
      flex-shrink: 0;
  }
  .worker-status-dot.active  { background: var(--active); }
  .worker-status-dot.idle    { background: var(--idle); }
  .worker-status-dot.faulted { background: var(--danger); }
  .worker-status-dot.waiting { background: var(--waiting); }

  .case-worker-name {
      flex: 1;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
  }

  .case-worker-time {
      font-size: 10px;
      color: var(--text-dim);
      flex-shrink: 0;
  }

  .case-panel-placeholder {
      padding: 12px;
      font-size: 12px;
      color: var(--text-dim);
      font-style: italic;
  }
  ```

- [ ] **Step 3: Run existing tests to confirm no regressions**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=StaticFilesTest 2>&1 | tail -10
  ```
  Expected: PASS.

- [ ] **Step 4: Commit**

  ```bash
  git commit -am "$(cat <<'EOF'
  feat: case worker panel markup and styles in session.html

  Left aside with Workers toggle; CSS mirrors channel panel pattern.

  Refs #76
  EOF
  )"
  ```

---

## Task 8: Frontend — terminal.js case panel logic

**Files:**
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/terminal.js`

- [ ] **Step 1: Append case panel logic at the end of terminal.js**

  Add the following block after the channel panel section (at the very end of the IIFE, before the closing `})();`):

  ```javascript
  // ── Case worker panel ─────────────────────────────────────────────────────

  var casePanel        = document.getElementById('case-panel');
  var caseWorkerList   = document.getElementById('case-worker-list');
  var caseCloseBtn     = document.getElementById('case-close-btn');
  var workersToggleBtn = document.getElementById('workers-toggle-btn');
  var activeCaseId     = null;
  var casePoller;

  function openCasePanel() { casePanel.classList.remove('collapsed'); }
  function closeCasePanel() { casePanel.classList.add('collapsed'); }

  workersToggleBtn.addEventListener('click', function () {
      if (casePanel.classList.contains('collapsed')) openCasePanel();
      else closeCasePanel();
  });
  caseCloseBtn.addEventListener('click', closeCasePanel);

  function workerTimeAgo(iso) {
      var diff = Date.now() - new Date(iso).getTime();
      var m = Math.floor(diff / 60000);
      if (m < 1) return 'now';
      if (m < 60) return m + 'm';
      return Math.floor(m / 60) + 'h';
  }

  function workerDisplayName(w) {
      return w.roleName || w.name.replace(/^claudony-worker-/, '').replace(/^claudony-/, '');
  }

  function renderWorkers(workers) {
      caseWorkerList.innerHTML = '';
      if (!workers || workers.length === 0) {
          var ph = document.createElement('div');
          ph.className = 'case-panel-placeholder';
          ph.textContent = 'No workers found.';
          caseWorkerList.appendChild(ph);
          return;
      }
      workers.forEach(function (w) {
          var row = document.createElement('div');
          var status = (w.status || 'idle').toLowerCase();
          row.className = 'case-worker-row' + (w.id === sessionId ? ' active-worker' : '');
          row.innerHTML =
              '<span class="worker-status-dot ' + status + '"></span>' +
              '<span class="case-worker-name">' + workerDisplayName(w) + '</span>' +
              '<span class="case-worker-time">' + workerTimeAgo(w.createdAt) + '</span>';
          row.addEventListener('click', function () {
              if (w.id === sessionId) return;
              switchToWorker(w.id, workerDisplayName(w));
          });
          caseWorkerList.appendChild(row);
      });
  }

  function pollWorkers() {
      if (!activeCaseId) return;
      fetch('/api/sessions?caseId=' + encodeURIComponent(activeCaseId))
          .then(function (r) { return r.ok ? r.json() : []; })
          .then(renderWorkers)
          .catch(function () {});
  }

  function switchToWorker(newSessionId, newName) {
      sessionId = newSessionId;
      sessionName = newName;
      document.getElementById('session-name').textContent = newName;
      history.replaceState(null, '',
          '?id=' + newSessionId + '&name=' + encodeURIComponent(newName));
      clearTimeout(reconnectTimer);
      if (ws) { ws.close(); } else { connect(); }
  }

  function showCasePlaceholder(text) {
      var ph = document.createElement('div');
      ph.className = 'case-panel-placeholder';
      ph.textContent = text;
      caseWorkerList.appendChild(ph);
  }

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

- [ ] **Step 2: Run static file test and full suite**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=StaticFilesTest,SessionResourceTest 2>&1 | tail -10
  ```
  Expected: PASS.

- [ ] **Step 3: Commit**

  ```bash
  git commit -am "$(cat <<'EOF'
  feat: terminal.js case worker panel — fetch, poll, render, switch

  Auto-expands when session has caseId. Clicking a worker closes WS
  and reconnects to new session. Placeholder for standalone sessions.

  Refs #76
  EOF
  )"
  ```

---

## Task 9: E2E tests — CaseWorkerPanelE2ETest

**Files:**
- Create: `claudony-app/src/test/java/dev/claudony/e2e/CaseWorkerPanelE2ETest.java`

- [ ] **Step 1: Create the E2E test class**

  ```java
  package dev.claudony.e2e;

  import com.microsoft.playwright.*;
  import dev.claudony.server.SessionRegistry;
  import dev.claudony.server.model.Session;
  import dev.claudony.server.model.SessionStatus;
  import io.quarkus.test.junit.QuarkusTest;
  import jakarta.inject.Inject;
  import org.junit.jupiter.api.*;

  import java.time.Instant;
  import java.util.Optional;

  import static org.junit.jupiter.api.Assertions.*;

  @QuarkusTest
  @TestMethodOrder(MethodOrderer.OrderAnnotation.class)
  class CaseWorkerPanelE2ETest {

      @Inject SessionRegistry registry;

      private static Playwright playwright;
      private static Browser browser;
      private Browser.NewContextOptions contextOptions;

      @BeforeAll
      static void launchBrowser() {
          playwright = Playwright.create();
          browser = playwright.chromium().launch(new BrowserType.LaunchOptions()
                  .setHeadless(!"false".equals(System.getProperty("playwright.headless", "true"))));
      }

      @AfterAll
      static void closeBrowser() {
          if (browser != null) browser.close();
          if (playwright != null) playwright.close();
      }

      @BeforeEach
      void setup() {
          contextOptions = new Browser.NewContextOptions()
                  .setBaseURL("http://localhost:" + System.getProperty("quarkus.http.port", "8081"));
      }

      @AfterEach
      void cleanup() {
          registry.all().forEach(s -> registry.remove(s.id()));
      }

      private String baseUrl() {
          return "http://localhost:" + System.getProperty("quarkus.http.port", "8081");
      }

      private void devLogin(Page page) {
          page.navigate(baseUrl() + "/auth/dev-login");
          // dev-login is POST — navigate triggers GET which may 405; use fetch instead
          page.evaluate("fetch('/auth/dev-login', {method:'POST'})");
          page.waitForTimeout(300);
      }

      @Test
      @Order(1)
      void standalonSession_panelIsCollapsedWithPlaceholder() {
          var now = Instant.now();
          var session = new Session("e2e-standalone", "claudony-e2e-standalone", "/tmp", "claude",
                  SessionStatus.IDLE, now, now, Optional.empty(), Optional.empty(), Optional.empty());
          registry.register(session);

          try (var context = browser.newContext(contextOptions)) {
              var page = context.newPage();
              devLogin(page);
              page.navigate(baseUrl() + "/app/session.html?id=e2e-standalone&name=e2e-standalone");
              page.waitForTimeout(1500); // let JS init + fetch run

              var casePanel = page.locator("#case-panel");
              assertTrue(casePanel.getAttribute("class").contains("collapsed"),
                      "Panel should be collapsed for standalone session");

              // Toggle to open and verify placeholder visible
              page.locator("#workers-toggle-btn").click();
              page.waitForTimeout(300);
              assertFalse(casePanel.getAttribute("class").contains("collapsed"),
                      "Panel should open on toggle");
              var placeholder = page.locator(".case-panel-placeholder");
              assertTrue(placeholder.isVisible(), "Placeholder should be visible");
              assertTrue(placeholder.textContent().contains("No case assigned"));
          }
      }

      @Test
      @Order(2)
      void caseHubSession_panelAutoExpandsWithWorkers() {
          var now = Instant.now();
          var caseId = "e2e-case-001";
          var s1 = new Session("e2e-w1", "claudony-worker-w1", "/tmp", "claude",
                  SessionStatus.ACTIVE, now.minusSeconds(30), now.minusSeconds(30), Optional.empty(),
                  Optional.of(caseId), Optional.of("researcher"));
          var s2 = new Session("e2e-w2", "claudony-worker-w2", "/tmp", "claude",
                  SessionStatus.IDLE, now, now, Optional.empty(),
                  Optional.of(caseId), Optional.of("coder"));
          registry.register(s1);
          registry.register(s2);

          try (var context = browser.newContext(contextOptions)) {
              var page = context.newPage();
              devLogin(page);
              page.navigate(baseUrl() + "/app/session.html?id=e2e-w1&name=researcher");
              page.waitForTimeout(1500); // let fetch + render run

              var casePanel = page.locator("#case-panel");
              assertFalse(casePanel.getAttribute("class").contains("collapsed"),
                      "Panel should auto-expand for CaseHub session");

              var rows = page.locator(".case-worker-row");
              assertEquals(2, rows.count(), "Both workers should be listed");

              // First row (researcher) is active
              var firstRow = rows.nth(0);
              assertTrue(firstRow.getAttribute("class").contains("active-worker"),
                      "Current worker should be highlighted");
              assertTrue(firstRow.textContent().contains("researcher"));

              // Second row (coder) is not active
              var secondRow = rows.nth(1);
              assertFalse(secondRow.getAttribute("class").contains("active-worker"));
              assertTrue(secondRow.textContent().contains("coder"));
          }
      }

      @Test
      @Order(3)
      void clickingWorker_updatesUrlAndHighlight() {
          var now = Instant.now();
          var caseId = "e2e-case-002";
          var s1 = new Session("e2e-c1", "claudony-worker-c1", "/tmp", "claude",
                  SessionStatus.ACTIVE, now.minusSeconds(10), now.minusSeconds(10), Optional.empty(),
                  Optional.of(caseId), Optional.of("planner"));
          var s2 = new Session("e2e-c2", "claudony-worker-c2", "/tmp", "claude",
                  SessionStatus.IDLE, now, now, Optional.empty(),
                  Optional.of(caseId), Optional.of("executor"));
          registry.register(s1);
          registry.register(s2);

          try (var context = browser.newContext(contextOptions)) {
              var page = context.newPage();
              devLogin(page);
              page.navigate(baseUrl() + "/app/session.html?id=e2e-c1&name=planner");
              page.waitForTimeout(1500);

              // Click second worker (executor)
              var rows = page.locator(".case-worker-row");
              rows.nth(1).click();
              page.waitForTimeout(500);

              // URL should reflect new session
              assertTrue(page.url().contains("e2e-c2"),
                      "URL should update to executor session id");

              // Highlight should shift
              page.waitForTimeout(1500); // wait for next poll cycle
              var updatedRows = page.locator(".case-worker-row");
              assertFalse(updatedRows.nth(0).getAttribute("class").contains("active-worker"),
                      "Planner should no longer be highlighted");
              assertTrue(updatedRows.nth(1).getAttribute("class").contains("active-worker"),
                      "Executor should now be highlighted");
          }
      }
  }
  ```

- [ ] **Step 2: Run the E2E tests**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -pl claudony-app -Dtest=CaseWorkerPanelE2ETest 2>&1 | tail -20
  ```
  Expected: `Tests run: 3, Failures: 0, Errors: 0`.

  If any fail due to timing, increase `waitForTimeout` values. Do not suppress failures — fix them.

- [ ] **Step 3: Run the full E2E suite to check for regressions**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -pl claudony-app -Dtest=PlaywrightSetupE2ETest,DashboardE2ETest,TerminalPageE2ETest,ChannelPanelE2ETest,CaseWorkerPanelE2ETest 2>&1 | tail -20
  ```
  Expected: all pass.

- [ ] **Step 4: Commit**

  ```bash
  git commit -am "$(cat <<'EOF'
  test(e2e): CaseWorkerPanelE2ETest — 3 Playwright tests

  Covers: standalone placeholder, CaseHub auto-expand + worker list,
  click-to-switch URL update and highlight shift.

  Refs #76
  EOF
  )"
  ```

---

## Task 10: Documentation — CLAUDE.md + DESIGN.md

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Update CLAUDE.md test count**

  The new test count is 409 + 3 new unit/integration tests (findByCaseId × 3, listByCaseId × 2, sessionResponse × 1, provision stamps × 1 = 7) + 3 E2E = **419 tests** in non-E2E run, **422** with E2E.

  Count exactly:
  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep "Tests run:" | tail -5
  ```
  Update the test count line in CLAUDE.md to match the actual number shown.

- [ ] **Step 2: Update Session model description in CLAUDE.md**

  Find the line:
  ```
  └── model/                          — Session, SessionStatus, SessionExpiredEvent
  ```
  Replace with:
  ```
  └── model/                          — Session (id, name, workingDir, command, status, createdAt,
                                          lastActive, expiryPolicy, caseId, roleName),
                                          SessionStatus, SessionExpiredEvent
  ```

- [ ] **Step 3: Update SessionRegistry description in CLAUDE.md**

  Find:
  ```
  └── SessionRegistry.java            — in-memory ConcurrentHashMap session store
  ```
  Replace with:
  ```
  └── SessionRegistry.java            — in-memory ConcurrentHashMap session store;
                                          findByCaseId(caseId) returns workers ordered by createdAt
  ```

- [ ] **Step 4: Update claudony-casehub test list in CLAUDE.md**

  Find the `ClaudonyWorkerProvisionerTest` entry and add the new assertion:
  ```
  - `ClaudonyWorkerProvisionerTest` — tmux session creation, disabled guard, terminate robustness, caseId/roleName stamped
  ```

- [ ] **Step 5: Update E2E test list in CLAUDE.md**

  Find the `ChannelPanelE2ETest` line and add:
  ```
  ChannelPanelE2ETest (8: ...), CaseWorkerPanelE2ETest (3: standalone placeholder, CaseHub auto-expand, click-to-switch)
  ```

- [ ] **Step 6: Update docs/DESIGN.md**

  In the `dev.claudony — claudony-core + claudony-app` component listing, find:
  ```
  │   ├── model/                  — Session, SessionStatus, request/response records
  ```
  Replace with:
  ```
  │   ├── model/                  — Session (+ caseId, roleName for CaseHub workers),
  │   │                             SessionStatus, request/response records
  ```

  Find the `SessionResource` line and update to note the `?caseId=` filter:
  ```
  │   ├── SessionResource         — REST /api/sessions (CRUD + resize + ?caseId= filter)
  ```

  Add a new entry for the case worker panel in the frontend section if one exists, or add a note under the session.html description.

- [ ] **Step 7: Run full suite to confirm all still green**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -10
  ```
  Expected: all tests pass.

- [ ] **Step 8: Final commit**

  ```bash
  git add CLAUDE.md docs/DESIGN.md
  git commit -m "$(cat <<'EOF'
  docs: update CLAUDE.md + DESIGN.md for case worker panel — issue #76

  Test count, Session model fields, SessionRegistry.findByCaseId,
  SessionResource ?caseId= filter, E2E test list.

  Closes #76
  EOF
  )"
  ```

---

## Verification Checklist

- [ ] All existing 409 tests still pass after Task 2
- [ ] 7 new unit/integration tests added (Tasks 1, 3, 4, 5, 6)
- [ ] 3 new E2E tests added (Task 9)
- [ ] `GET /api/sessions?caseId=xxx` returns workers sorted by `createdAt`
- [ ] `SessionResponse` includes `caseId` + `roleName` (NON_NULL — absent for standalone)
- [ ] CaseHub-provisioned sessions have `caseId` + `roleName` set
- [ ] session.html has case panel `<aside>` + Workers toggle button
- [ ] Panel auto-expands for CaseHub sessions; collapsed + placeholder for standalone
- [ ] Clicking a worker switches WebSocket and updates URL
- [ ] CLAUDE.md test count accurate
- [ ] Issue #76 closed in final commit message
