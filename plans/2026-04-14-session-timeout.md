# Session Timeout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Quarkus WebAuthn's 30-minute default session timeout with a configurable 7-day inactivity timeout so sessions survive browser close.

**Architecture:** Add `claudony.session-timeout` (`Duration`, default `P7D`) to `ClaudonyConfig`, reference it from `application.properties` via `quarkus.webauthn.session-timeout=${claudony.session-timeout}`, and set `quarkus.webauthn.new-cookie-interval=1H`. No cookie rewriting or Quarkus patching — pure config.

**Tech Stack:** Java 21, Quarkus 3.9.5, SmallRye Config (`@ConfigMapping`), `quarkus-security-webauthn`, JUnit 5, QuarkusTest, AssertJ

---

## File Map

| File | Action |
|---|---|
| `src/main/java/dev/claudony/config/ClaudonyConfig.java` | **Modify** — add `sessionTimeout()` method |
| `src/main/resources/application.properties` | **Modify** — add `quarkus.webauthn.session-timeout` and `quarkus.webauthn.new-cookie-interval` |
| `src/test/java/dev/claudony/config/SessionTimeoutConfigTest.java` | **Create** — 3 QuarkusTest integration tests |
| `CLAUDE.md` | **Modify** — update test count 159 → 162 |

---

## Task 1: Write Failing Tests + Implement + Verify Green

This task follows the full TDD cycle: write tests that fail to compile, add the implementation to make them compile, add the config to make them pass.

**Files:**
- Create: `src/test/java/dev/claudony/config/SessionTimeoutConfigTest.java`
- Modify: `src/main/java/dev/claudony/config/ClaudonyConfig.java`
- Modify: `src/main/resources/application.properties`

### Step 1: Write the failing integration tests

Create `src/test/java/dev/claudony/config/SessionTimeoutConfigTest.java`:

```java
package dev.claudony.config;

import io.quarkus.security.webauthn.WebAuthnRunTimeConfig;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class SessionTimeoutConfigTest {

    @Inject
    ClaudonyConfig claudonyConfig;

    @Inject
    WebAuthnRunTimeConfig webAuthnConfig;

    @Test
    void sessionTimeoutDefaultsTo7Days() {
        // claudony.session-timeout defaults to P7D (7 days) when not overridden
        assertThat(claudonyConfig.sessionTimeout()).isEqualTo(Duration.ofDays(7));
    }

    @Test
    void quarkusWebAuthnReceivesSessionTimeout() {
        // Verifies the ${claudony.session-timeout} reference in application.properties
        // is resolved and reaches Quarkus's WebAuthn config — the critical wiring test
        assertThat(webAuthnConfig.sessionTimeout()).isEqualTo(Duration.ofDays(7));
    }

    @Test
    void newCookieIntervalIsOneHour() {
        // With a 7-day session window, refreshing the cookie every minute (Quarkus default)
        // is unnecessary; 1H balances responsiveness with server load
        assertThat(webAuthnConfig.newCookieInterval()).isEqualTo(Duration.ofHours(1));
    }
}
```

- [ ] **Step 2: Run to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionTimeoutConfigTest 2>&1 | tail -15
```

Expected: compilation error — `cannot find symbol: method sessionTimeout()` in `ClaudonyConfig`

- [ ] **Step 3: Add `sessionTimeout()` to ClaudonyConfig**

In `src/main/java/dev/claudony/config/ClaudonyConfig.java`, add the import and new method.

The full updated file:

```java
package dev.claudony.config;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;
import java.time.Duration;
import java.util.Optional;

@ConfigMapping(prefix = "claudony")
public interface ClaudonyConfig {

    @WithDefault("server")
    String mode();

    @WithDefault("7777")
    int port();

    @WithDefault("localhost")
    String bind();

    @WithName("server.url")
    @WithDefault("http://localhost:7777")
    String serverUrl();

    @WithDefault("claude")
    String claudeCommand();

    @WithDefault("claudony-")
    String tmuxPrefix();

    @WithDefault("auto")
    String terminal();

    @WithName("agent.api-key")
    Optional<String> agentApiKey();

    @WithName("default-working-dir")
    @WithDefault("${user.home}/claudony-workspace")
    String defaultWorkingDir();

    @WithName("credentials-file")
    @WithDefault("${user.home}/.claudony/credentials.json")
    String credentialsFile();

    @WithName("session-timeout")
    @WithDefault("P7D")
    Duration sessionTimeout();

    default boolean isServerMode() { return "server".equals(mode()); }
    default boolean isAgentMode()  { return "agent".equals(mode()); }
}
```

- [ ] **Step 4: Run to confirm compilation passes but `quarkusWebAuthnReceivesSessionTimeout` fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionTimeoutConfigTest 2>&1 | tail -20
```

Expected: compilation succeeds; `sessionTimeoutDefaultsTo7Days` passes; `quarkusWebAuthnReceivesSessionTimeout` fails (still 30 minutes because `application.properties` not yet updated); `newCookieIntervalIsOneHour` fails (still 1 minute default).

- [ ] **Step 5: Add the two properties to application.properties**

In `src/main/resources/application.properties`, add these lines after the existing WebAuthn block (after `quarkus.webauthn.login-page=/auth/login`):

```properties
# Session inactivity timeout — how long a login stays valid without activity.
# Default: 7 days (P7D). Override via CLAUDONY_SESSION_TIMEOUT env var.
quarkus.webauthn.session-timeout=${claudony.session-timeout}

# Cookie renewal interval — how often an active session's cookie Max-Age is refreshed.
# 1H: reduces unnecessary cookie regeneration vs. the Quarkus default of 1 minute.
quarkus.webauthn.new-cookie-interval=1H
```

- [ ] **Step 6: Run tests — all 3 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionTimeoutConfigTest 2>&1 | tail -10
```

Expected:
```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

- [ ] **Step 7: Run the full test suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -10
```

Expected: `Tests run: 162, Failures: 0, Errors: 0, Skipped: 0`

(159 previous + 3 new = 162)

- [ ] **Step 8: Commit**

```bash
git add src/main/java/dev/claudony/config/ClaudonyConfig.java \
        src/main/resources/application.properties \
        src/test/java/dev/claudony/config/SessionTimeoutConfigTest.java
git commit -m "feat: configurable session timeout — default P7D via claudony.session-timeout"
```

---

## Task 2: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update test count**

In `CLAUDE.md`, find:
```
**159 tests passing** across:
```
Replace with:
```
**162 tests passing** across:
```

Find the `config/` bullet:
```
- `config/` — EncryptionKeyConfigSource (15 unit tests + 5 QuarkusTest integration)
```
Replace with:
```
- `config/` — EncryptionKeyConfigSource (15 unit tests + 5 QuarkusTest integration), SessionTimeoutConfigTest (3 QuarkusTest integration)
```

- [ ] **Step 2: Run full suite one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: `Tests run: 162, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update test count to 162 after session timeout feature"
```

---

## E2E Verification (Manual)

After building and running the jar:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests
JAVA_HOME=$(/usr/libexec/java_home -v 26) java \
  -Dclaudony.mode=server -Dclaudony.bind=0.0.0.0 \
  -jar target/quarkus-app/quarkus-run.jar
```

1. Open `http://localhost:7777/auth/login` and log in via passkey
2. Open browser dev tools → Application → Cookies → `localhost`
3. Find the session cookie (name: `quarkus-credential`)
4. Confirm `Max-Age` ≈ `604800` (7 × 24 × 3600 seconds)
5. Close the browser, reopen, navigate to `http://localhost:7777/app/`
6. Confirm no login redirect — session persists across browser close

To test the configurable override:
```bash
CLAUDONY_SESSION_TIMEOUT=P1D \
JAVA_HOME=$(/usr/libexec/java_home -v 26) java \
  -Dclaudony.mode=server \
  -jar target/quarkus-app/quarkus-run.jar
```
Cookie `Max-Age` should be `86400` (1 day).

---

## Summary

| Task | Tests added | Commit |
|---|---|---|
| Task 1: Implementation + tests | 3 | `feat: configurable session timeout` |
| Task 2: CLAUDE.md | 0 | `docs: update test count to 162` |
| **Total** | **3** | |
