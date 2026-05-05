# Session Expiry Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce idle session expiry via a pluggable CDI policy registry with three implementations (user-interaction, terminal-output, status-aware), a scheduled killer, and CDI event notification on expiry.

**Architecture:** `ExpiryPolicy` interface + `ExpiryPolicyRegistry` (CDI bean collecting all policies by name); `SessionIdleScheduler` (@Scheduled every 5m) iterates the registry, resolves per-session policy, kills expired tmux sessions, fires `SessionExpiredEvent`; `TerminalWebSocket` observes the event and sends `{"type":"session-expired"}` to connected clients. Per-session policy override stored on `Session.expiryPolicy` and set via `CreateSessionRequest`.

**Tech Stack:** Java 21, Quarkus 3.9.5, Quarkus Scheduler, CDI `Event<T>` / `@Observes`, JUnit 5, AssertJ, tmux CLI

---

## File Map

| File | Action | Purpose |
|---|---|---|
| `server/model/Session.java` | Modify | Add `Optional<String> expiryPolicy` field |
| `server/model/SessionResponse.java` | Modify | Add `String expiryPolicy` field; update `from()` signature |
| `server/model/SessionExpiredEvent.java` | Create | CDI event record |
| `server/model/CreateSessionRequest.java` | Modify | Add optional `String expiryPolicy` field |
| `server/SessionRegistry.java` | Modify | Add `touch(String id)` |
| `server/ServerStartup.java` | Modify | Update `new Session(...)` call site (add `Optional.empty()`) |
| `server/SessionResource.java` | Modify | Inject registry; resolve policy for `SessionResponse.from()`; `touch()` on input; update all `from()` call sites |
| `server/TerminalWebSocket.java` | Modify | `touch()` on open/message; store connections map; observe `SessionExpiredEvent` |
| `server/expiry/ExpiryPolicy.java` | Create | Interface: `name()`, `isExpired(Session, Duration)` |
| `server/expiry/ExpiryPolicyRegistry.java` | Create | CDI registry bean |
| `server/expiry/UserInteractionExpiryPolicy.java` | Create | Checks `session.lastActive()` |
| `server/expiry/TerminalOutputExpiryPolicy.java` | Create | Checks `tmux display-message #{pane_activity}` |
| `server/expiry/StatusAwareExpiryPolicy.java` | Create | Checks `#{pane_current_command}` |
| `server/expiry/SessionIdleScheduler.java` | Create | @Scheduled expiry checker |
| `server/TmuxService.java` | Modify | Add `displayMessage(name, format)` |
| `config/ClaudonyConfig.java` | Modify | Add `sessionExpiryPolicy()` |
| `resources/application.properties` | Modify | Add `claudony.session-expiry-policy=user-interaction` |
| `test/.../model/SessionTest.java` | Modify | Add `Optional.empty()` to `new Session(...)` calls |
| `test/.../SessionRegistryTest.java` | Modify | Add `Optional.empty()` to `new Session(...)` calls; add `touch()` tests |
| `test/.../ServerStartupTest.java` | Modify | Update `new Session(...)` if it constructs sessions directly |
| `test/.../fleet/SessionFederationTest.java` | Modify | Add `null` to `new SessionResponse(...)` calls |
| `test/.../agent/ClaudonyMcpToolsTest.java` | Modify | Add `null` to `new SessionResponse(...)` calls |
| `test/.../expiry/ExpiryPolicyRegistryTest.java` | Create | Registry resolves by name, defaults, unknown names |
| `test/.../expiry/UserInteractionExpiryPolicyTest.java` | Create | Pure unit test — no Quarkus |
| `test/.../expiry/TerminalOutputExpiryPolicyTest.java` | Create | @QuarkusTest — real tmux |
| `test/.../expiry/StatusAwareExpiryPolicyTest.java` | Create | @QuarkusTest — real tmux |
| `test/.../expiry/SessionIdleSchedulerTest.java` | Create | @QuarkusTest — real tmux |
| `docs/DESIGN.md` | Modify | Document expiry architecture |
| `CLAUDE.md` | Modify | Update test count |

---

## Task 1: Create GitHub issue and epic

**Files:** None (GitHub)

- [ ] **Step 1: Check for existing epic or create one**

```bash
gh issue list --repo mdproctor/claudony --state open --label epic
```

If no session-expiry epic exists:
```bash
gh issue create --repo mdproctor/claudony \
  --title "epic: session expiry enforcement — pluggable expiry policy" \
  --label epic \
  --body "Implement idle tmux session cleanup with pluggable CDI expiry policy registry.

Spec: docs/superpowers/specs/2026-04-20-session-expiry-design.md

Child issues:
- Config + Session model changes
- ExpiryPolicy interface + registry + three implementations
- TmuxService displayMessage
- SessionIdleScheduler + SessionExpiredEvent
- Activity tracking (WebSocket + REST)
- WebSocket expiry observer
- SessionResponse expiryPolicy field
- Doc updates"
```

Note the epic issue number (e.g. #66).

- [ ] **Step 2: Create implementation issue**

```bash
gh issue create --repo mdproctor/claudony \
  --title "feat: session expiry — pluggable ExpiryPolicy CDI registry, scheduler, activity tracking" \
  --body "Refs #<EPIC_NUMBER>

Implements the session expiry spec: docs/superpowers/specs/2026-04-20-session-expiry-design.md

- ExpiryPolicy interface + ExpiryPolicyRegistry
- Three policies: user-interaction, terminal-output, status-aware
- SessionIdleScheduler (@Scheduled every 5m)
- SessionExpiredEvent CDI event
- Activity tracking via SessionRegistry.touch()
- Per-session policy override on CreateSessionRequest"
```

Note the issue number (e.g. #67). Use `Refs #67` in all commits; `Closes #67` in the final commit.

---

## Task 2: Config — add `sessionExpiryPolicy()`

**Files:**
- Modify: `src/main/java/dev/claudony/config/ClaudonyConfig.java`
- Modify: `src/main/resources/application.properties`
- Modify: `src/test/java/dev/claudony/config/SessionTimeoutConfigTest.java`

- [ ] **Step 1: Write the failing test**

Add to `SessionTimeoutConfigTest.java` (inside the existing `@QuarkusTest` class):

```java
@Test
void sessionExpiryPolicyDefaultsToUserInteraction() {
    assertThat(claudonyConfig.sessionExpiryPolicy()).isEqualTo("user-interaction");
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionTimeoutConfigTest -pl . 2>&1 | tail -20
```

Expected: compilation error — `cannot find symbol: method sessionExpiryPolicy()`.

- [ ] **Step 3: Add method to ClaudonyConfig**

In `src/main/java/dev/claudony/config/ClaudonyConfig.java`, add after the `sessionTimeout()` method:

```java
@WithName("session-expiry-policy")
@WithDefault("user-interaction")
String sessionExpiryPolicy();
```

- [ ] **Step 4: Add property to application.properties**

In `src/main/resources/application.properties`, add after the session-timeout comment block:

```properties
# Expiry policy for idle tmux sessions. Available: user-interaction, terminal-output, status-aware
# user-interaction: expires based on last user input/WebSocket open
# terminal-output: expires based on last tmux pane activity (tmux's clock)
# status-aware: never expires if a non-shell process is running; falls back to user-interaction
claudony.session-expiry-policy=user-interaction
```

- [ ] **Step 5: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionTimeoutConfigTest -pl . 2>&1 | tail -20
```

Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/dev/claudony/config/ClaudonyConfig.java \
        src/main/resources/application.properties \
        src/test/java/dev/claudony/config/SessionTimeoutConfigTest.java
git commit -m "feat: add claudony.session-expiry-policy config (default: user-interaction) Refs #<ISSUE>"
```

---

## Task 3: Session model — add `expiryPolicy` field and `SessionRegistry.touch()`

**Files:**
- Modify: `src/main/java/dev/claudony/server/model/Session.java`
- Modify: `src/main/java/dev/claudony/server/model/CreateSessionRequest.java`
- Modify: `src/main/java/dev/claudony/server/SessionRegistry.java`
- Modify: `src/main/java/dev/claudony/server/ServerStartup.java`
- Modify: `src/main/java/dev/claudony/server/SessionResource.java` (two `new Session(...)` call sites)
- Modify: `src/test/java/dev/claudony/server/model/SessionTest.java`
- Modify: `src/test/java/dev/claudony/server/SessionRegistryTest.java`

- [ ] **Step 1: Write failing tests**

Replace `src/test/java/dev/claudony/server/model/SessionTest.java` entirely:

```java
package dev.claudony.server.model;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

class SessionTest {

    @Test
    void sessionHasRequiredFields() {
        var now = Instant.now();
        var session = new Session("id-1", "myproject", "/home/user/proj",
                "claude", SessionStatus.IDLE, now, now, Optional.empty());

        assertEquals("id-1", session.id());
        assertEquals("myproject", session.name());
        assertEquals("/home/user/proj", session.workingDir());
        assertEquals("claude", session.command());
        assertEquals(SessionStatus.IDLE, session.status());
        assertTrue(session.expiryPolicy().isEmpty());
    }

    @Test
    void withStatusReturnsCopyWithUpdatedStatusAndPreservesExpiryPolicy() {
        var now = Instant.now();
        var session = new Session("id-1", "myproject", "/home/user/proj",
                "claude", SessionStatus.IDLE, now, now, Optional.of("terminal-output"));

        var updated = session.withStatus(SessionStatus.ACTIVE);

        assertEquals(SessionStatus.ACTIVE, updated.status());
        assertEquals("id-1", updated.id());
        assertEquals(Optional.of("terminal-output"), updated.expiryPolicy());
    }

    @Test
    void withLastActivePreservesExpiryPolicy() {
        var now = Instant.now();
        var session = new Session("id-1", "myproject", "/home/user/proj",
                "claude", SessionStatus.IDLE, now, now, Optional.of("status-aware"));

        var updated = session.withLastActive();

        assertEquals(Optional.of("status-aware"), updated.expiryPolicy());
        assertTrue(updated.lastActive().isAfter(now) || updated.lastActive().equals(now));
    }

    @Test
    void sessionStatusValues() {
        assertNotNull(SessionStatus.valueOf("ACTIVE"));
        assertNotNull(SessionStatus.valueOf("WAITING"));
        assertNotNull(SessionStatus.valueOf("IDLE"));
    }
}
```

Add to `src/test/java/dev/claudony/server/SessionRegistryTest.java` (inside the existing `@QuarkusTest` class, update all existing `new Session(...)` calls and add the touch test):

```java
// Replace all existing new Session(...) calls by adding Optional.empty() as the 8th argument.
// Example — existing:
//   new Session("id-1", "proj", "/tmp", "claude", SessionStatus.IDLE, now, now)
// Updated:
//   new Session("id-1", "proj", "/tmp", "claude", SessionStatus.IDLE, now, now, Optional.empty())

// Add this new test at the end of the class:
@Test
void touchUpdatesLastActive() throws InterruptedException {
    var past = Instant.now().minusSeconds(60);
    registry.register(new Session("id-touch", "proj", "/tmp", "claude",
            SessionStatus.IDLE, past, past, Optional.empty()));

    Thread.sleep(10); // ensure clock advances
    registry.touch("id-touch");

    var updated = registry.find("id-touch");
    assertTrue(updated.isPresent());
    assertTrue(updated.get().lastActive().isAfter(past),
            "touch() should update lastActive beyond the original value");
}

@Test
void touchOnUnknownIdIsNoOp() {
    registry.touch("nonexistent-id"); // must not throw
}
```

- [ ] **Step 2: Run to verify tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionTest,SessionRegistryTest -pl . 2>&1 | tail -20
```

Expected: compilation errors — `Optional<String>` not in `Session`, `touch()` not in `SessionRegistry`.

- [ ] **Step 3: Update Session.java**

Replace `src/main/java/dev/claudony/server/model/Session.java` entirely:

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
        Optional<String> expiryPolicy) {

    public Session withStatus(SessionStatus newStatus) {
        return new Session(id, name, workingDir, command, newStatus, createdAt, Instant.now(), expiryPolicy);
    }

    public Session withLastActive() {
        return new Session(id, name, workingDir, command, status, createdAt, Instant.now(), expiryPolicy);
    }
}
```

- [ ] **Step 4: Update CreateSessionRequest.java**

Replace `src/main/java/dev/claudony/server/model/CreateSessionRequest.java` entirely:

```java
package dev.claudony.server.model;

public record CreateSessionRequest(
        String name,
        String workingDir,
        String command,
        String expiryPolicy) {

    public String effectiveCommand(String defaultCommand) {
        return (command != null && !command.isBlank()) ? command : defaultCommand;
    }
}
```

- [ ] **Step 5: Add touch() to SessionRegistry.java**

In `src/main/java/dev/claudony/server/SessionRegistry.java`, add after `updateStatus()`:

```java
public void touch(String id) {
    sessions.computeIfPresent(id, (k, s) -> s.withLastActive());
}
```

- [ ] **Step 6: Fix broken call sites — ServerStartup.java**

In `src/main/java/dev/claudony/server/ServerStartup.java`, update `bootstrapRegistry()`:

```java
// Old:
registry.register(new Session(
        UUID.randomUUID().toString(), name,
        "unknown", config.claudeCommand(),
        SessionStatus.IDLE, now, now));

// New:
registry.register(new Session(
        UUID.randomUUID().toString(), name,
        "unknown", config.claudeCommand(),
        SessionStatus.IDLE, now, now, java.util.Optional.empty()));
```

- [ ] **Step 7: Fix broken call sites — SessionResource.java**

In `src/main/java/dev/claudony/server/SessionResource.java`:

Add import at top: `import java.util.Optional;`

Update `create()` — find the line `var session = new Session(id, name, workingDir, command, SessionStatus.IDLE, now, now);` and replace:

```java
var session = new Session(id, name, workingDir, command, SessionStatus.IDLE, now, now,
        Optional.ofNullable(req.expiryPolicy()));
```

Update `rename()` — find the line constructing `renamed` and replace:

```java
var renamed = new Session(id, newTmuxName, session.workingDir(),
        session.command(), session.status(), session.createdAt(), Instant.now(),
        session.expiryPolicy());
```

- [ ] **Step 8: Fix broken call sites — SessionRegistryTest.java**

In `src/test/java/dev/claudony/server/SessionRegistryTest.java`, add import:

```java
import java.util.Optional;
```

Update all five `new Session(...)` calls to add `Optional.empty()` as the 8th argument:

```java
// Example — all occurrences follow this pattern:
new Session("id-1", "proj", "/tmp", "claude", SessionStatus.IDLE, now, now, Optional.empty())
```

- [ ] **Step 9: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionTest,SessionRegistryTest -pl . 2>&1 | tail -20
```

Expected: all tests PASS.

- [ ] **Step 10: Run full test suite to verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected: all previously-passing tests still PASS (some may fail if they construct `Session` or `SessionResponse` directly — fix them in this step).

If `SessionFederationTest` or `ClaudonyMcpToolsTest` fail due to `SessionResponse` constructor mismatch, that will be fixed in Task 12. For now, note the failures and proceed. The task 12 note: SessionResponse is not changed until Task 12.

Actually — `SessionResponse.from(session, port)` in `SessionResource` still compiles because `Session` has a new field but `SessionResponse.from()` doesn't use it yet. The `SessionResponse` record itself is unchanged until Task 12. So the only compilation breaks are the `new Session(...)` call sites — all fixed in steps 6–8.

- [ ] **Step 11: Commit**

```bash
git add src/main/java/dev/claudony/server/model/Session.java \
        src/main/java/dev/claudony/server/model/CreateSessionRequest.java \
        src/main/java/dev/claudony/server/SessionRegistry.java \
        src/main/java/dev/claudony/server/ServerStartup.java \
        src/main/java/dev/claudony/server/SessionResource.java \
        src/test/java/dev/claudony/server/model/SessionTest.java \
        src/test/java/dev/claudony/server/SessionRegistryTest.java
git commit -m "feat: Session.expiryPolicy field, CreateSessionRequest.expiryPolicy, SessionRegistry.touch() Refs #<ISSUE>"
```

---

## Task 4: ExpiryPolicy interface + ExpiryPolicyRegistry

**Files:**
- Create: `src/main/java/dev/claudony/server/expiry/ExpiryPolicy.java`
- Create: `src/main/java/dev/claudony/server/expiry/ExpiryPolicyRegistry.java`
- Create: `src/test/java/dev/claudony/server/expiry/ExpiryPolicyRegistryTest.java`

Note: the registry test requires all three policy beans to exist. Write the registry and its test now (the registry will compile but its `@ApplicationScoped` init will warn about missing policies). The policy tests come in Tasks 5–8. If running the registry test before policies exist, Quarkus will warn but the test itself will handle unknown-name fallback.

To avoid a chicken-and-egg: create minimal stub implementations first, then replace them properly in Tasks 5–8. OR write the tests in Task 4 and run them only after all policies are in place (Task 8). This plan takes the latter approach — **do not run `ExpiryPolicyRegistryTest` until Task 8 is complete**.

- [ ] **Step 1: Create ExpiryPolicy.java**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.model.Session;
import java.time.Duration;

public interface ExpiryPolicy {
    String name();
    boolean isExpired(Session session, Duration timeout);
}
```

- [ ] **Step 2: Create ExpiryPolicyRegistry.java**

```java
package dev.claudony.server.expiry;

import dev.claudony.config.ClaudonyConfig;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.util.Collections;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

@ApplicationScoped
public class ExpiryPolicyRegistry {

    private static final Logger LOG = Logger.getLogger(ExpiryPolicyRegistry.class);

    private final Map<String, ExpiryPolicy> policies;
    private final String defaultPolicyName;

    @Inject
    ExpiryPolicyRegistry(@Any Instance<ExpiryPolicy> all, ClaudonyConfig config) {
        this.defaultPolicyName = config.sessionExpiryPolicy();
        this.policies = StreamSupport.stream(all.spliterator(), false)
                .collect(Collectors.toMap(ExpiryPolicy::name, p -> p));
        if (!this.policies.containsKey(defaultPolicyName)) {
            LOG.warnf("Configured session-expiry-policy '%s' not found. Available: %s",
                    defaultPolicyName, policies.keySet());
        }
        LOG.infof("ExpiryPolicyRegistry initialised with %d policies: %s",
                policies.size(), policies.keySet());
    }

    public ExpiryPolicy resolve(String name) {
        if (name != null && policies.containsKey(name)) return policies.get(name);
        var fallback = policies.get(defaultPolicyName);
        return fallback != null ? fallback : policies.values().iterator().next();
    }

    public Set<String> availableNames() {
        return Collections.unmodifiableSet(policies.keySet());
    }
}
```

- [ ] **Step 3: Create ExpiryPolicyRegistryTest.java**

```java
package dev.claudony.server.expiry;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ExpiryPolicyRegistryTest {

    @Inject ExpiryPolicyRegistry registry;

    @Test
    void resolvesUserInteractionPolicy() {
        assertThat(registry.resolve("user-interaction").name()).isEqualTo("user-interaction");
    }

    @Test
    void resolvesTerminalOutputPolicy() {
        assertThat(registry.resolve("terminal-output").name()).isEqualTo("terminal-output");
    }

    @Test
    void resolvesStatusAwarePolicy() {
        assertThat(registry.resolve("status-aware").name()).isEqualTo("status-aware");
    }

    @Test
    void resolvesNullToDefault() {
        assertThat(registry.resolve(null).name()).isEqualTo("user-interaction");
    }

    @Test
    void resolvesUnknownNameToDefault() {
        assertThat(registry.resolve("no-such-policy").name()).isEqualTo("user-interaction");
    }

    @Test
    void availableNamesContainsAllThreePolicies() {
        assertThat(registry.availableNames())
                .containsExactlyInAnyOrder("user-interaction", "terminal-output", "status-aware");
    }
}
```

Do not run this test yet — wait until Task 8 when all three policies exist.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/dev/claudony/server/expiry/ExpiryPolicy.java \
        src/main/java/dev/claudony/server/expiry/ExpiryPolicyRegistry.java \
        src/test/java/dev/claudony/server/expiry/ExpiryPolicyRegistryTest.java
git commit -m "feat: ExpiryPolicy interface + ExpiryPolicyRegistry CDI bean Refs #<ISSUE>"
```

---

## Task 5: UserInteractionExpiryPolicy

**Files:**
- Create: `src/main/java/dev/claudony/server/expiry/UserInteractionExpiryPolicy.java`
- Create: `src/test/java/dev/claudony/server/expiry/UserInteractionExpiryPolicyTest.java`

- [ ] **Step 1: Write the failing test (pure unit — no @QuarkusTest)**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

class UserInteractionExpiryPolicyTest {

    final UserInteractionExpiryPolicy policy = new UserInteractionExpiryPolicy();

    @Test
    void notExpiredWhenLastActiveIsRecent() {
        var session = sessionWith(Instant.now().minus(Duration.ofHours(1)));
        assertFalse(policy.isExpired(session, Duration.ofDays(7)));
    }

    @Test
    void expiredWhenLastActiveIsBeyondTimeout() {
        var session = sessionWith(Instant.now().minus(Duration.ofDays(8)));
        assertTrue(policy.isExpired(session, Duration.ofDays(7)));
    }

    @Test
    void notExpiredAtExactBoundary() {
        // One second before the timeout boundary — should NOT be expired
        var session = sessionWith(Instant.now().minus(Duration.ofDays(7)).plusSeconds(1));
        assertFalse(policy.isExpired(session, Duration.ofDays(7)));
    }

    @Test
    void expiredWithShortTimeout() {
        var session = sessionWith(Instant.now().minus(Duration.ofMinutes(10)));
        assertTrue(policy.isExpired(session, Duration.ofMinutes(5)));
    }

    @Test
    void nameIsUserInteraction() {
        assertThat(policy.name()).isEqualTo("user-interaction");
    }

    private Session sessionWith(Instant lastActive) {
        var now = Instant.now();
        return new Session("id", "name", "/tmp", "claude",
                SessionStatus.IDLE, now, lastActive, Optional.empty());
    }
}
```

Note: add `import static org.assertj.core.api.Assertions.assertThat;` at the top.

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=UserInteractionExpiryPolicyTest -pl . 2>&1 | tail -20
```

Expected: compilation error — `UserInteractionExpiryPolicy` does not exist.

- [ ] **Step 3: Create UserInteractionExpiryPolicy.java**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.model.Session;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Duration;
import java.time.Instant;

@ApplicationScoped
public class UserInteractionExpiryPolicy implements ExpiryPolicy {

    @Override
    public String name() { return "user-interaction"; }

    @Override
    public boolean isExpired(Session session, Duration timeout) {
        return Duration.between(session.lastActive(), Instant.now()).compareTo(timeout) > 0;
    }
}
```

- [ ] **Step 4: Run to verify tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=UserInteractionExpiryPolicyTest -pl . 2>&1 | tail -20
```

Expected: all 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/expiry/UserInteractionExpiryPolicy.java \
        src/test/java/dev/claudony/server/expiry/UserInteractionExpiryPolicyTest.java
git commit -m "feat: UserInteractionExpiryPolicy — expires on lastActive threshold Refs #<ISSUE>"
```

---

## Task 6: TmuxService.displayMessage()

**Files:**
- Modify: `src/main/java/dev/claudony/server/TmuxService.java`
- Modify: `src/test/java/dev/claudony/server/TmuxServiceTest.java`

- [ ] **Step 1: Write the failing test**

Add to `TmuxServiceTest.java` (inside the existing `@QuarkusTest` class):

```java
@Test
void displayMessageReturnsPaneActivity() throws Exception {
    tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo hello");
    Thread.sleep(300);
    var activity = tmux.displayMessage(TEST_SESSION, "#{pane_activity}");
    assertFalse(activity.isBlank(), "pane_activity should return a non-empty timestamp");
    // pane_activity is a Unix epoch second — must parse as a long
    assertDoesNotThrow(() -> Long.parseLong(activity.trim()),
            "pane_activity should be a numeric Unix timestamp, got: " + activity);
}

@Test
void displayMessageReturnsPaneCurrentCommand() throws Exception {
    tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "sleep 10");
    Thread.sleep(300);
    var command = tmux.displayMessage(TEST_SESSION, "#{pane_current_command}");
    assertFalse(command.isBlank());
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TmuxServiceTest -pl . 2>&1 | tail -20
```

Expected: compilation error — `displayMessage()` does not exist.

- [ ] **Step 3: Add displayMessage() to TmuxService.java**

In `src/main/java/dev/claudony/server/TmuxService.java`, add after `capturePane()`:

```java
public String displayMessage(String sessionName, String format)
        throws IOException, InterruptedException {
    var p = new ProcessBuilder("tmux", "display-message", "-p", format, "-t", sessionName)
            .redirectErrorStream(false).start();
    var output = new String(p.getInputStream().readAllBytes()).trim();
    p.waitFor();
    return output;
}
```

- [ ] **Step 4: Run to verify tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TmuxServiceTest -pl . 2>&1 | tail -20
```

Expected: all tests PASS including the two new ones.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/TmuxService.java \
        src/test/java/dev/claudony/server/TmuxServiceTest.java
git commit -m "feat: TmuxService.displayMessage() for tmux format queries Refs #<ISSUE>"
```

---

## Task 7: TerminalOutputExpiryPolicy

**Files:**
- Create: `src/main/java/dev/claudony/server/expiry/TerminalOutputExpiryPolicy.java`
- Create: `src/test/java/dev/claudony/server/expiry/TerminalOutputExpiryPolicyTest.java`

- [ ] **Step 1: Write the failing test**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.TmuxService;
import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.*;
import java.time.Duration;
import java.time.Instant;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class TerminalOutputExpiryPolicyTest {

    @Inject TerminalOutputExpiryPolicy policy;
    @Inject TmuxService tmux;

    private static final String TEST_SESSION = "test-expiry-terminal-output";

    @AfterEach
    void cleanup() throws Exception {
        if (tmux.sessionExists(TEST_SESSION)) tmux.killSession(TEST_SESSION);
    }

    @Test
    void notExpiredForSessionWithRecentTmuxActivity() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo active");
        Thread.sleep(500);
        assertFalse(policy.isExpired(session(TEST_SESSION, Instant.now()), Duration.ofDays(7)));
    }

    @Test
    void expiredForNonExistentSession() {
        assertTrue(policy.isExpired(session("no-such-session-zzz", Instant.now()), Duration.ofDays(7)));
    }

    @Test
    void nameIsTerminalOutput() {
        assertThat(policy.name()).isEqualTo("terminal-output");
    }

    private Session session(String name, Instant lastActive) {
        var now = Instant.now();
        return new Session("id", name, "/tmp", "bash", SessionStatus.IDLE, now, lastActive, Optional.empty());
    }
}
```

Add import: `import static org.assertj.core.api.Assertions.assertThat;`

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TerminalOutputExpiryPolicyTest -pl . 2>&1 | tail -20
```

Expected: compilation error — `TerminalOutputExpiryPolicy` does not exist.

- [ ] **Step 3: Create TerminalOutputExpiryPolicy.java**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.TmuxService;
import dev.claudony.server.model.Session;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.time.Duration;
import java.time.Instant;

@ApplicationScoped
public class TerminalOutputExpiryPolicy implements ExpiryPolicy {

    private static final Logger LOG = Logger.getLogger(TerminalOutputExpiryPolicy.class);

    @Inject TmuxService tmux;

    @Override
    public String name() { return "terminal-output"; }

    @Override
    public boolean isExpired(Session session, Duration timeout) {
        try {
            var raw = tmux.displayMessage(session.name(), "#{pane_activity}");
            if (raw.isBlank()) return true;
            var lastActivity = Instant.ofEpochSecond(Long.parseLong(raw.trim()));
            return Duration.between(lastActivity, Instant.now()).compareTo(timeout) > 0;
        } catch (Exception e) {
            LOG.debugf("terminal-output policy: tmux call failed for '%s': %s",
                    session.name(), e.getMessage());
            return true;
        }
    }
}
```

- [ ] **Step 4: Run to verify tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TerminalOutputExpiryPolicyTest -pl . 2>&1 | tail -20
```

Expected: all 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/expiry/TerminalOutputExpiryPolicy.java \
        src/test/java/dev/claudony/server/expiry/TerminalOutputExpiryPolicyTest.java
git commit -m "feat: TerminalOutputExpiryPolicy — uses tmux pane_activity timestamp Refs #<ISSUE>"
```

---

## Task 8: StatusAwareExpiryPolicy (+ run ExpiryPolicyRegistryTest)

**Files:**
- Create: `src/main/java/dev/claudony/server/expiry/StatusAwareExpiryPolicy.java`
- Create: `src/test/java/dev/claudony/server/expiry/StatusAwareExpiryPolicyTest.java`

- [ ] **Step 1: Write the failing test**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.TmuxService;
import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.*;
import java.time.Duration;
import java.time.Instant;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class StatusAwareExpiryPolicyTest {

    @Inject StatusAwareExpiryPolicy policy;
    @Inject TmuxService tmux;

    private static final String TEST_SESSION = "test-expiry-status-aware";

    @AfterEach
    void cleanup() throws Exception {
        if (tmux.sessionExists(TEST_SESSION)) tmux.killSession(TEST_SESSION);
    }

    @Test
    void neverExpiresWhenNonShellCommandRunning() throws Exception {
        // sleep keeps a non-shell process in the foreground
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "sleep 60");
        Thread.sleep(500);
        var veryOldLastActive = Instant.now().minus(Duration.ofDays(30));
        assertFalse(policy.isExpired(session(TEST_SESSION, veryOldLastActive), Duration.ofMinutes(1)));
    }

    @Test
    void expiresAtShellPromptWhenLastActiveIsOld() throws Exception {
        // "true" exits immediately, leaving the shell at a prompt
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "true");
        Thread.sleep(500);
        var oldLastActive = Instant.now().minus(Duration.ofDays(8));
        assertTrue(policy.isExpired(session(TEST_SESSION, oldLastActive), Duration.ofDays(7)));
    }

    @Test
    void notExpiredAtShellPromptWhenLastActiveIsRecent() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "true");
        Thread.sleep(500);
        var recentLastActive = Instant.now().minus(Duration.ofHours(1));
        assertFalse(policy.isExpired(session(TEST_SESSION, recentLastActive), Duration.ofDays(7)));
    }

    @Test
    void expiredForNonExistentSession() {
        assertTrue(policy.isExpired(session("no-such-session-zzz", Instant.now()), Duration.ofDays(7)));
    }

    @Test
    void nameIsStatusAware() {
        assertThat(policy.name()).isEqualTo("status-aware");
    }

    private Session session(String name, Instant lastActive) {
        var now = Instant.now();
        return new Session("id", name, "/tmp", "bash", SessionStatus.IDLE, now, lastActive, Optional.empty());
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StatusAwareExpiryPolicyTest -pl . 2>&1 | tail -20
```

Expected: compilation error — `StatusAwareExpiryPolicy` does not exist.

- [ ] **Step 3: Create StatusAwareExpiryPolicy.java**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.TmuxService;
import dev.claudony.server.model.Session;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.time.Duration;
import java.time.Instant;
import java.util.Set;

@ApplicationScoped
public class StatusAwareExpiryPolicy implements ExpiryPolicy {

    private static final Logger LOG = Logger.getLogger(StatusAwareExpiryPolicy.class);
    private static final Set<String> SHELL_COMMANDS = Set.of("bash", "zsh", "sh", "dash", "fish");

    @Inject TmuxService tmux;

    @Override
    public String name() { return "status-aware"; }

    @Override
    public boolean isExpired(Session session, Duration timeout) {
        try {
            var currentCommand = tmux.displayMessage(session.name(), "#{pane_current_command}");
            if (currentCommand.isBlank()) return true;
            if (!SHELL_COMMANDS.contains(currentCommand.trim())) return false;
            return Duration.between(session.lastActive(), Instant.now()).compareTo(timeout) > 0;
        } catch (Exception e) {
            LOG.debugf("status-aware policy: tmux call failed for '%s': %s",
                    session.name(), e.getMessage());
            return true;
        }
    }
}
```

- [ ] **Step 4: Run StatusAwareExpiryPolicyTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StatusAwareExpiryPolicyTest -pl . 2>&1 | tail -20
```

Expected: all 5 tests PASS.

- [ ] **Step 5: Now run ExpiryPolicyRegistryTest (all three policies are registered)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ExpiryPolicyRegistryTest -pl . 2>&1 | tail -20
```

Expected: all 6 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/dev/claudony/server/expiry/StatusAwareExpiryPolicy.java \
        src/test/java/dev/claudony/server/expiry/StatusAwareExpiryPolicyTest.java
git commit -m "feat: StatusAwareExpiryPolicy — defers expiry when non-shell command running Refs #<ISSUE>"
```

---

## Task 9: SessionExpiredEvent + SessionIdleScheduler

**Files:**
- Create: `src/main/java/dev/claudony/server/model/SessionExpiredEvent.java`
- Create: `src/main/java/dev/claudony/server/expiry/SessionIdleScheduler.java`
- Create: `src/test/java/dev/claudony/server/expiry/SessionIdleSchedulerTest.java`

- [ ] **Step 1: Create SessionExpiredEvent.java**

```java
package dev.claudony.server.model;

public record SessionExpiredEvent(Session session) {}
```

- [ ] **Step 2: Write the failing scheduler test**

```java
package dev.claudony.server.expiry;

import dev.claudony.server.SessionRegistry;
import dev.claudony.server.TmuxService;
import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionExpiredEvent;
import dev.claudony.server.model.SessionStatus;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.inject.Singleton;
import org.junit.jupiter.api.*;
import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class SessionIdleSchedulerTest {

    @Inject SessionIdleScheduler scheduler;
    @Inject SessionRegistry registry;
    @Inject TmuxService tmux;
    @Inject EventCaptor eventCaptor;

    private static final String EXPIRED_TMUX = "test-sched-expired";
    private static final String ACTIVE_TMUX  = "test-sched-active";

    @BeforeEach
    void setUp() {
        registry.all().forEach(s -> registry.remove(s.id()));
        eventCaptor.events.clear();
    }

    @AfterEach
    void tearDown() throws Exception {
        registry.all().forEach(s -> registry.remove(s.id()));
        if (tmux.sessionExists(EXPIRED_TMUX)) tmux.killSession(EXPIRED_TMUX);
        if (tmux.sessionExists(ACTIVE_TMUX)) tmux.killSession(ACTIVE_TMUX);
    }

    @Test
    void expiredSessionIsKilledAndRemovedFromRegistry() throws Exception {
        tmux.createSession(EXPIRED_TMUX, System.getProperty("user.home"), "bash");
        var now = Instant.now();
        var session = new Session("exp-id", EXPIRED_TMUX, "/tmp", "bash",
                SessionStatus.IDLE, now, now.minus(Duration.ofDays(8)), Optional.empty());
        registry.register(session);

        scheduler.expiryCheck();

        assertFalse(tmux.sessionExists(EXPIRED_TMUX), "Expired tmux session should be killed");
        assertTrue(registry.find("exp-id").isEmpty(), "Expired registry entry should be removed");
    }

    @Test
    void activeSessionIsNotKilledOrRemoved() throws Exception {
        tmux.createSession(ACTIVE_TMUX, System.getProperty("user.home"), "bash");
        var now = Instant.now();
        var session = new Session("active-id", ACTIVE_TMUX, "/tmp", "bash",
                SessionStatus.IDLE, now, now, Optional.empty()); // lastActive = now
        registry.register(session);

        scheduler.expiryCheck();

        assertTrue(tmux.sessionExists(ACTIVE_TMUX), "Active tmux session should survive");
        assertTrue(registry.find("active-id").isPresent(), "Active registry entry should survive");
    }

    @Test
    void expiryEventFiredForExpiredSession() throws Exception {
        tmux.createSession(EXPIRED_TMUX, System.getProperty("user.home"), "bash");
        var now = Instant.now();
        var session = new Session("evt-id", EXPIRED_TMUX, "/tmp", "bash",
                SessionStatus.IDLE, now, now.minus(Duration.ofDays(8)), Optional.empty());
        registry.register(session);

        scheduler.expiryCheck();

        assertEquals(1, eventCaptor.events.size(), "SessionExpiredEvent should be fired once");
        assertEquals("evt-id", eventCaptor.events.get(0).session().id());
    }

    @Test
    void perSessionPolicyOverridesDefault() throws Exception {
        // terminal-output policy: pane_activity is recent → not expired despite old lastActive
        tmux.createSession(EXPIRED_TMUX, System.getProperty("user.home"), "bash");
        Thread.sleep(300);
        var now = Instant.now();
        var session = new Session("pol-id", EXPIRED_TMUX, "/tmp", "bash",
                SessionStatus.IDLE, now, now.minus(Duration.ofDays(8)),
                Optional.of("terminal-output")); // policy override
        registry.register(session);

        scheduler.expiryCheck();

        // terminal-output uses pane_activity which is recent — should NOT expire
        assertTrue(registry.find("pol-id").isPresent(),
                "Session with terminal-output policy and recent tmux activity should not be expired");
        assertTrue(tmux.sessionExists(EXPIRED_TMUX), "Tmux session should survive");
    }

    @Singleton
    static class EventCaptor {
        final List<SessionExpiredEvent> events = new ArrayList<>();
        void observe(@Observes SessionExpiredEvent e) { events.add(e); }
    }
}
```

- [ ] **Step 3: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionIdleSchedulerTest -pl . 2>&1 | tail -20
```

Expected: compilation error — `SessionIdleScheduler` does not exist.

- [ ] **Step 4: Create SessionIdleScheduler.java**

```java
package dev.claudony.server.expiry;

import dev.claudony.config.ClaudonyConfig;
import dev.claudony.server.SessionRegistry;
import dev.claudony.server.TmuxService;
import dev.claudony.server.model.SessionExpiredEvent;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class SessionIdleScheduler {

    private static final Logger LOG = Logger.getLogger(SessionIdleScheduler.class);

    @Inject ClaudonyConfig config;
    @Inject SessionRegistry registry;
    @Inject TmuxService tmux;
    @Inject ExpiryPolicyRegistry policyRegistry;
    @Inject Event<SessionExpiredEvent> expiryEvents;

    @Scheduled(every = "5m", delayed = "1m")
    void scheduledCheck() {
        if (!config.isServerMode()) return;
        expiryCheck();
    }

    void expiryCheck() {
        var timeout = config.sessionTimeout();
        for (var session : registry.all()) {
            var policy = policyRegistry.resolve(session.expiryPolicy().orElse(null));
            if (!policy.isExpired(session, timeout)) continue;
            LOG.infof("Expiring session '%s' (policy=%s, lastActive=%s)",
                    session.name(), policy.name(), session.lastActive());
            try {
                expiryEvents.fire(new SessionExpiredEvent(session));
                tmux.killSession(session.name());
            } catch (Exception e) {
                LOG.warnf("Could not kill session '%s': %s", session.name(), e.getMessage());
            }
            registry.remove(session.id());
        }
    }
}
```

- [ ] **Step 5: Run to verify tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionIdleSchedulerTest -pl . 2>&1 | tail -20
```

Expected: all 4 tests PASS.

- [ ] **Step 6: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected: all previously-passing tests still pass.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/dev/claudony/server/model/SessionExpiredEvent.java \
        src/main/java/dev/claudony/server/expiry/SessionIdleScheduler.java \
        src/test/java/dev/claudony/server/expiry/SessionIdleSchedulerTest.java
git commit -m "feat: SessionExpiredEvent + SessionIdleScheduler — @Scheduled expiry with CDI event Refs #<ISSUE>"
```

---

## Task 10: Activity tracking — touch() call sites

**Files:**
- Modify: `src/main/java/dev/claudony/server/TerminalWebSocket.java`
- Modify: `src/main/java/dev/claudony/server/SessionResource.java`
- Modify: `src/test/java/dev/claudony/server/SessionRegistryTest.java` (already has `touchUpdatesLastActive` from Task 3)

The goal: call `registry.touch(sessionId)` whenever a user interacts with a session so `lastActive` stays current.

- [ ] **Step 1: Add sessionIds map to TerminalWebSocket and touch on open**

In `src/main/java/dev/claudony/server/TerminalWebSocket.java`:

After the `fifoPaths` field declaration, add:

```java
// connection.id() → session id (needed to touch registry on message)
private final ConcurrentHashMap<String, String> sessionIds = new ConcurrentHashMap<>();
```

In `onOpen()`, after `var sessionId = connection.pathParam("id");` and `var session = registry.find(sessionId);` (immediately after the empty-check guard), add:

```java
sessionIds.put(connection.id(), sessionId);
registry.touch(sessionId);
```

In `cleanup()`, add:

```java
sessionIds.remove(connection.id());
```

- [ ] **Step 2: Touch on incoming message**

In `onMessage()`, after the null-check `if (tmuxName == null) return;`, add:

```java
var sid = sessionIds.get(connection.id());
if (sid != null) registry.touch(sid);
```

- [ ] **Step 3: Touch on REST input**

In `src/main/java/dev/claudony/server/SessionResource.java`, in `sendInput()`, inside the `.map(session -> { try {` block, after `tmux.sendKeys(...)` succeeds and before `return Response.noContent().build();`:

```java
registry.touch(id);
```

The method should look like:

```java
@POST
@Path("/{id}/input")
@Produces(MediaType.APPLICATION_JSON)
public Response sendInput(@PathParam("id") String id, SendInputRequest req) {
    return registry.find(id).map(session -> {
        try {
            tmux.sendKeys(session.name(), req.text());
            registry.touch(id);
            return Response.noContent().build();
        } catch (IOException | InterruptedException e) {
            LOG.errorf("Failed to send input to session '%s': %s", session.name(), e.getMessage());
            return Response.serverError().build();
        }
    }).orElse(Response.status(404).build());
}
```

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/TerminalWebSocket.java \
        src/main/java/dev/claudony/server/SessionResource.java
git commit -m "feat: activity tracking — touch() on WebSocket open/message and REST input Refs #<ISSUE>"
```

---

## Task 11: WebSocket expiry event observer

**Files:**
- Modify: `src/main/java/dev/claudony/server/TerminalWebSocket.java`

When a session expires, send `{"type":"session-expired"}` to any connected WebSocket clients before the connection drops.

- [ ] **Step 1: Add connections map and observer**

In `src/main/java/dev/claudony/server/TerminalWebSocket.java`:

Add import: `import dev.claudony.server.model.SessionExpiredEvent;` and `import jakarta.enterprise.event.Observes;`

After the `sessionIds` field, add:

```java
// connection.id() → WebSocketConnection (needed to send expiry message)
private final ConcurrentHashMap<String, WebSocketConnection> connections = new ConcurrentHashMap<>();
```

In `onOpen()`, after `sessionIds.put(connection.id(), sessionId);`, add:

```java
connections.put(connection.id(), connection);
```

In `cleanup()`, add:

```java
connections.remove(connection.id());
```

Add the observer method (inside the class, before `cleanup()`):

```java
void onSessionExpired(@Observes SessionExpiredEvent event) {
    var expiredName = event.session().name();
    sessionNames.entrySet().stream()
            .filter(e -> e.getValue().equals(expiredName))
            .map(e -> connections.get(e.getKey()))
            .filter(java.util.Objects::nonNull)
            .forEach(conn -> {
                try { conn.sendTextAndAwait("{\"type\":\"session-expired\"}"); }
                catch (Exception ignored) {}
            });
}
```

- [ ] **Step 2: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected: all tests PASS.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/dev/claudony/server/TerminalWebSocket.java
git commit -m "feat: WebSocket observer — send session-expired message to connected clients Refs #<ISSUE>"
```

---

## Task 12: SessionResponse expiryPolicy field

**Files:**
- Modify: `src/main/java/dev/claudony/server/model/SessionResponse.java`
- Modify: `src/main/java/dev/claudony/server/SessionResource.java`
- Modify: `src/test/java/dev/claudony/server/fleet/SessionFederationTest.java`
- Modify: `src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java`

- [ ] **Step 1: Update SessionResponse.java**

Replace `src/main/java/dev/claudony/server/model/SessionResponse.java` entirely:

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
        String expiryPolicy) {

    public static SessionResponse from(Session session, int port, String effectivePolicy) {
        return new SessionResponse(
                session.id(), session.name(), session.workingDir(), session.command(),
                session.status(), session.createdAt(), session.lastActive(),
                "ws://localhost:" + port + "/ws/" + session.id(),
                "http://localhost:" + port + "/app/session/" + session.id(),
                null, null, null, effectivePolicy);
    }

    public SessionResponse withInstance(String peerUrl, String peerName, boolean isStale) {
        return new SessionResponse(
                id, name, workingDir, command, status, createdAt, lastActive,
                wsUrl, browserUrl, peerUrl, peerName, isStale, expiryPolicy);
    }
}
```

- [ ] **Step 2: Inject ExpiryPolicyRegistry in SessionResource and update all from() call sites**

In `src/main/java/dev/claudony/server/SessionResource.java`:

Add inject after other injects:

```java
@Inject ExpiryPolicyRegistry policyRegistry;
```

Add import: `import dev.claudony.server.expiry.ExpiryPolicyRegistry;`

Create a helper method at the bottom of the class (before the private helpers):

```java
private String resolvedPolicy(Session session) {
    return policyRegistry.resolve(session.expiryPolicy().orElse(null)).name();
}
```

Update all `SessionResponse.from(s, config.port())` calls:

```java
// In list():
.map(s -> SessionResponse.from(s, config.port(), resolvedPolicy(s)))

// In get():
.map(s -> Response.ok(SessionResponse.from(s, config.port(), resolvedPolicy(s))).build())

// In create():
return Response.status(201)
        .entity(SessionResponse.from(session, config.port(), resolvedPolicy(session)))
        .build();

// In rename():
return Response.ok(SessionResponse.from(renamed, config.port(), resolvedPolicy(renamed))).build();
```

- [ ] **Step 3: Fix SessionFederationTest.java**

In `src/test/java/dev/claudony/server/fleet/SessionFederationTest.java`, find the `new SessionResponse(...)` call and add `null` as the 13th argument:

```java
var fakeSession = new SessionResponse(
        "fake-session-id", "claudony-fake", "/tmp", "claude",
        SessionStatus.IDLE,
        Instant.now(), Instant.now(),
        "ws://unreachable-peer:7777/ws/fake-session-id",
        "http://unreachable-peer:7777/app/session/fake-session-id",
        null, null, null, null); // 13th arg: expiryPolicy (null = NON_NULL omits it)
```

- [ ] **Step 4: Fix ClaudonyMcpToolsTest.java**

In `src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java`, for every `new SessionResponse(...)` call (there are ~7), add `null` as the 13th argument. All calls follow the same pattern:

```java
new SessionResponse("id-N", "claudony-name", "/path", "claude",
    SessionStatus.IDLE, now, now,
    "ws://localhost:7777/ws/id-N",
    "http://localhost:7777/app/session/id-N",
    null, null, null, null) // 13th arg added
```

- [ ] **Step 5: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/dev/claudony/server/model/SessionResponse.java \
        src/main/java/dev/claudony/server/SessionResource.java \
        src/test/java/dev/claudony/server/fleet/SessionFederationTest.java \
        src/test/java/dev/claudony/agent/ClaudonyMcpToolsTest.java
git commit -m "feat: SessionResponse.expiryPolicy — expose resolved policy name in API Refs #<ISSUE>"
```

---

## Task 13: Doc updates and final commit

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update DESIGN.md**

In `docs/DESIGN.md`, add a section on session expiry (find the "Architecture Notes" or equivalent section and add after it):

```markdown
## Session Expiry

Idle tmux sessions are cleaned up by `SessionIdleScheduler` (`@Scheduled every 5m`).
Expiry logic is pluggable via the `ExpiryPolicy` CDI interface:

| Policy | Name | Mechanism |
|---|---|---|
| `UserInteractionExpiryPolicy` | `user-interaction` | Checks `session.lastActive()` (default) |
| `TerminalOutputExpiryPolicy` | `terminal-output` | Checks `tmux display-message #{pane_activity}` |
| `StatusAwareExpiryPolicy` | `status-aware` | Never expires non-shell foreground processes |

Global default: `claudony.session-expiry-policy=user-interaction`. Per-session override via `CreateSessionRequest.expiryPolicy`. On expiry: `SessionExpiredEvent` fired (WebSocket observer sends `{"type":"session-expired"}`), tmux session killed, registry entry removed. `lastActive` is bumped by `SessionRegistry.touch()` on WebSocket open/input and REST `/input`.
```

- [ ] **Step 2: Update CLAUDE.md test count**

Run the test suite and count passing tests:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep "Tests run:" | tail -5
```

Update the test count line in `CLAUDE.md` (search for `**246 tests passing**` and update to the new count).

Update the test class list to include the new classes:
- `server/expiry/` — `ExpiryPolicyRegistryTest`, `UserInteractionExpiryPolicyTest`, `TerminalOutputExpiryPolicyTest`, `StatusAwareExpiryPolicyTest`, `SessionIdleSchedulerTest`

- [ ] **Step 3: Run full suite one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -10
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Final commit closing the issue**

```bash
git add docs/DESIGN.md CLAUDE.md
git commit -m "docs: update DESIGN.md and test count for session expiry enforcement Closes #<ISSUE> Refs #<EPIC>"
```
