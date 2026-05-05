# Per-Deployment Session Encryption Key Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the hardcoded shared prod session encryption key with a per-deployment auto-generated key that is persisted to `~/.claudony/encryption-key` via a MicroProfile `ConfigSource`.

**Architecture:** `EncryptionKeyConfigSource` implements `org.eclipse.microprofile.config.spi.ConfigSource` and is registered via `META-INF/services/`. It sits at ordinal 200, below `application.properties` profile keys (250) so dev/test fixed keys still win, but above nothing in prod — where the hardcoded `%prod` key is removed. On first run, generates a 256-bit `SecureRandom` key, persists it to `~/.claudony/encryption-key` with `rw-------` permissions, and caches it in memory. On subsequent runs, reads from disk.

**Tech Stack:** Java 21, MicroProfile Config 3.x (SmallRye Config), JUnit 5, QuarkusTest, AssertJ, `java.security.SecureRandom`, `java.nio.file`

---

## File Map

| File | Action |
|---|---|
| `src/main/java/dev/claudony/config/EncryptionKeyConfigSource.java` | **Create** — ConfigSource implementation |
| `src/main/resources/META-INF/services/org.eclipse.microprofile.config.spi.ConfigSource` | **Create** — ServiceLoader registration |
| `src/main/resources/application.properties` | **Modify** — remove `%prod.quarkus.http.auth.session.encryption-key` line |
| `src/test/java/dev/claudony/config/EncryptionKeyConfigSourceTest.java` | **Create** — unit tests (no Quarkus, 13 tests) |
| `src/test/java/dev/claudony/config/EncryptionKeyConfigSourceIntegrationTest.java` | **Create** — QuarkusTest integration tests (5 tests) |
| `CLAUDE.md` | **Modify** — update test count from 139 → 157 |

---

## Task 1: Write Failing Unit Tests

**Files:**
- Create: `src/test/java/dev/claudony/config/EncryptionKeyConfigSourceTest.java`

These tests are plain JUnit 5 — no Quarkus container. This is intentional: ConfigSources run before CDI, so testing without the container is more faithful. All tests inject a `@TempDir` to avoid touching `~/.claudony/` on the real machine.

- [ ] **Step 1: Create the test file**

```java
package dev.claudony.config;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.FileSystems;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.attribute.PosixFilePermission;
import java.nio.file.attribute.PosixFilePermissions;
import java.util.EnumSet;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assumptions.assumeTrue;

class EncryptionKeyConfigSourceTest {

    private static final String ENC_KEY_PROP = "quarkus.http.auth.session.encryption-key";

    @TempDir
    Path tempDir;

    // ─── First-run happy path ───────────────────────────────────────────────

    @Test
    void firstRun_generatesNonBlankKey() {
        var source = new EncryptionKeyConfigSource(tempDir);
        assertThat(source.getValue(ENC_KEY_PROP)).isNotBlank();
    }

    @Test
    void firstRun_persistsKeyToFile() throws IOException {
        var source = new EncryptionKeyConfigSource(tempDir);
        var key = source.getValue(ENC_KEY_PROP);

        var keyFile = tempDir.resolve("encryption-key");
        assertThat(keyFile).exists();
        assertThat(Files.readString(keyFile).strip()).isEqualTo(key);
    }

    @Test
    void firstRun_keyIs43CharBase64Url() {
        var source = new EncryptionKeyConfigSource(tempDir);
        var key = source.getValue(ENC_KEY_PROP);

        // 32 random bytes → base64url without padding → exactly 43 chars
        assertThat(key).matches("[A-Za-z0-9_\\-]{43}");
    }

    @Test
    void firstRun_filePermissionsAreOwnerOnly() throws IOException {
        assumeTrue(isPosix(), "POSIX permissions only testable on POSIX filesystems");

        var source = new EncryptionKeyConfigSource(tempDir);
        source.getValue(ENC_KEY_PROP);

        var perms = Files.getPosixFilePermissions(tempDir.resolve("encryption-key"));
        assertThat(perms).isEqualTo(EnumSet.of(
                PosixFilePermission.OWNER_READ,
                PosixFilePermission.OWNER_WRITE
        ));
    }

    // ─── Idempotency ────────────────────────────────────────────────────────

    @Test
    void idempotent_sameKeyReturnedOnRepeatCalls() {
        var source = new EncryptionKeyConfigSource(tempDir);
        var key1 = source.getValue(ENC_KEY_PROP);
        var key2 = source.getValue(ENC_KEY_PROP);

        assertThat(key1).isEqualTo(key2);
    }

    @Test
    void idempotent_fileWrittenExactlyOnce() throws IOException {
        var source = new EncryptionKeyConfigSource(tempDir);
        source.getValue(ENC_KEY_PROP);
        source.getValue(ENC_KEY_PROP);

        // Only the encryption-key file should exist in the temp dir
        assertThat(Files.list(tempDir).toList()).hasSize(1);
    }

    // ─── Subsequent run (existing file) ─────────────────────────────────────

    @Test
    void subsequentRun_loadsExistingKeyFromFile() throws IOException {
        var knownKey = "existing-test-key-abcdefghijklmnopqrstuvwxyz012";
        Files.writeString(tempDir.resolve("encryption-key"), knownKey);

        var source = new EncryptionKeyConfigSource(tempDir);
        assertThat(source.getValue(ENC_KEY_PROP)).isEqualTo(knownKey);
    }

    @Test
    void subsequentRun_doesNotOverwriteExistingFile() throws IOException {
        var knownKey = "existing-test-key-abcdefghijklmnopqrstuvwxyz012";
        Files.writeString(tempDir.resolve("encryption-key"), knownKey);

        var source = new EncryptionKeyConfigSource(tempDir);
        source.getValue(ENC_KEY_PROP);

        // File unchanged
        assertThat(Files.readString(tempDir.resolve("encryption-key")).strip()).isEqualTo(knownKey);
    }

    // ─── Property scoping ───────────────────────────────────────────────────

    @Test
    void unknownProperty_returnsNull() {
        var source = new EncryptionKeyConfigSource(tempDir);
        assertThat(source.getValue("some.random.property")).isNull();
        assertThat(source.getValue("claudony.mode")).isNull();
        assertThat(source.getValue("quarkus.http.host")).isNull();
    }

    // ─── Security: key not exposed in property map ──────────────────────────

    @Test
    void getProperties_returnsEmptyMapForSecurity() {
        var source = new EncryptionKeyConfigSource(tempDir);
        // Deliberately empty — prevents the key appearing in config dumps/actuator
        assertThat(source.getProperties()).isEmpty();
    }

    @Test
    void getPropertyNames_includesEncryptionKeyProperty() {
        var source = new EncryptionKeyConfigSource(tempDir);
        assertThat(source.getPropertyNames()).contains(ENC_KEY_PROP);
    }

    // ─── ConfigSource contract ───────────────────────────────────────────────

    @Test
    void getName_returnsExpectedName() {
        var source = new EncryptionKeyConfigSource(tempDir);
        assertThat(source.getName()).isEqualTo("claudony-file-encryption-key");
    }

    @Test
    void getOrdinal_returns200() {
        var source = new EncryptionKeyConfigSource(tempDir);
        assertThat(source.getOrdinal()).isEqualTo(200);
    }

    // ─── Resilience: unwritable directory ────────────────────────────────────

    @Test
    void unwritableDir_returnsKeyAndDoesNotCrash() throws IOException {
        assumeTrue(isPosix(), "Permission manipulation requires POSIX filesystem");

        var readOnlyDir = tempDir.resolve("readonly");
        Files.createDirectory(readOnlyDir);
        Files.setPosixFilePermissions(readOnlyDir, PosixFilePermissions.fromString("r-xr-xr-x"));

        try {
            var source = new EncryptionKeyConfigSource(readOnlyDir);
            var key = source.getValue(ENC_KEY_PROP);

            // Still returns a usable key even though file could not be persisted
            assertThat(key).isNotBlank();
            // File was NOT created (write failed gracefully)
            assertThat(readOnlyDir.resolve("encryption-key")).doesNotExist();
        } finally {
            // Restore permissions so @TempDir cleanup can delete the directory
            Files.setPosixFilePermissions(readOnlyDir, PosixFilePermissions.fromString("rwxr-xr-x"));
        }
    }

    // ─── Helpers ────────────────────────────────────────────────────────────

    private static boolean isPosix() {
        return FileSystems.getDefault().supportedFileAttributeViews().contains("posix");
    }
}
```

- [ ] **Step 2: Run to confirm compilation fails (class not yet created)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EncryptionKeyConfigSourceTest 2>&1 | tail -20
```

Expected: compilation error — `cannot find symbol: class EncryptionKeyConfigSource`

---

## Task 2: Implement EncryptionKeyConfigSource

**Files:**
- Create: `src/main/java/dev/claudony/config/EncryptionKeyConfigSource.java`

- [ ] **Step 1: Create the implementation**

```java
package dev.claudony.config;

import io.quarkus.runtime.annotations.RegisterForReflection;
import org.eclipse.microprofile.config.spi.ConfigSource;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.attribute.PosixFilePermissions;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.Map;
import java.util.Set;

/**
 * MicroProfile ConfigSource that provides a per-deployment session encryption key.
 *
 * <p>Ordinal 200 — sits below application.properties profile keys (250) so dev/test
 * fixed keys still win, but provides the only value in prod where the hardcoded
 * shared key has been removed.</p>
 *
 * <p>On first run, generates a 256-bit SecureRandom key, persists it to
 * ~/.claudony/encryption-key with rw------- permissions, and caches it in memory.
 * On subsequent runs, reads from disk. The env var
 * QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY (ordinal 300) always wins.</p>
 *
 * <p>getProperties() intentionally returns an empty map so the key value never
 * appears in config dumps, actuator endpoints, or log lines that enumerate
 * all config properties.</p>
 */
@RegisterForReflection
public class EncryptionKeyConfigSource implements ConfigSource {

    private static final Logger LOG = Logger.getLogger(EncryptionKeyConfigSource.class);
    private static final String KEY_PROPERTY = "quarkus.http.auth.session.encryption-key";
    private static final int ORDINAL = 200;

    private final Path configDir;
    private volatile String cachedKey;

    /** Called by ServiceLoader at Quarkus bootstrap. */
    public EncryptionKeyConfigSource() {
        this.configDir = Path.of(System.getProperty("user.home"), ".claudony");
    }

    /** Package-private — for unit tests only. Injects a temp directory. */
    EncryptionKeyConfigSource(Path configDir) {
        this.configDir = configDir;
    }

    @Override
    public String getValue(String propertyName) {
        if (!KEY_PROPERTY.equals(propertyName)) {
            return null;
        }
        return getOrGenerateKey();
    }

    @Override
    public String getName() {
        return "claudony-file-encryption-key";
    }

    @Override
    public int getOrdinal() {
        return ORDINAL;
    }

    /**
     * Intentionally empty — prevents the encryption key from appearing in config
     * dumps, /q/config actuator responses, or any log that iterates all properties.
     * SmallRye Config resolves values via getValue(), not getProperties(), so
     * returning an empty map here does not affect key resolution.
     */
    @Override
    public Map<String, String> getProperties() {
        return Map.of();
    }

    @Override
    public Set<String> getPropertyNames() {
        return Set.of(KEY_PROPERTY);
    }

    // ─── Private implementation ──────────────────────────────────────────────

    private String getOrGenerateKey() {
        if (cachedKey != null) {
            return cachedKey;
        }
        synchronized (this) {
            if (cachedKey != null) {
                return cachedKey;
            }
            cachedKey = loadOrGenerate();
            return cachedKey;
        }
    }

    private String loadOrGenerate() {
        var keyFile = configDir.resolve("encryption-key");

        // Happy path: existing file from a previous run
        if (Files.exists(keyFile)) {
            try {
                var key = Files.readString(keyFile).strip();
                if (!key.isBlank()) {
                    LOG.debugf("Loaded session encryption key from %s", keyFile);
                    return key;
                }
            } catch (IOException e) {
                LOG.warnf("Could not read session encryption key from %s: %s — regenerating",
                        keyFile, e.getMessage());
            }
        }

        // First run — generate a new key
        var key = generateKey();

        // Ensure ~/.claudony/ exists
        if (!Files.exists(configDir)) {
            try {
                Files.createDirectories(configDir);
            } catch (IOException e) {
                LOG.warnf("Could not create config directory %s: %s" +
                        " — sessions will not survive restart", configDir, e.getMessage());
                return key;
            }
        }

        // Persist with owner-only permissions
        try {
            Files.writeString(keyFile, key);
            try {
                Files.setPosixFilePermissions(keyFile, PosixFilePermissions.fromString("rw-------"));
            } catch (UnsupportedOperationException ignored) {
                // Non-POSIX filesystem (Windows) — skip chmod, log a note
                LOG.info("Non-POSIX filesystem: session encryption key file permissions not restricted");
            }
            logGenerationBanner(keyFile);
        } catch (IOException e) {
            LOG.warnf("Could not persist session encryption key to %s: %s" +
                    " — sessions will not survive restart. Check directory permissions.", keyFile, e.getMessage());
        }

        return key;
    }

    private String generateKey() {
        var bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    private void logGenerationBanner(Path keyFile) {
        LOG.info("\n================================================================\n" +
                "CLAUDONY — Session Encryption Key Generated (first run)\n" +
                "  Saved to: " + keyFile.toAbsolutePath() + "\n" +
                "  Permissions: rw------- (owner read/write only)\n\n" +
                "  Sessions will now survive server restarts.\n\n" +
                "  To use a custom key (e.g. from a secrets manager):\n" +
                "    export QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY=<your-key>\n" +
                "  The env var takes precedence over the persisted file.\n" +
                "================================================================");
    }
}
```

- [ ] **Step 2: Run unit tests — expect all 13 to pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EncryptionKeyConfigSourceTest 2>&1 | tail -30
```

Expected output:
```
Tests run: 13, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Some tests will be `Skipped` (not `Errors`) on non-POSIX filesystems — that's correct.

- [ ] **Step 3: Commit the RED→GREEN unit test cycle**

```bash
git add src/main/java/dev/claudony/config/EncryptionKeyConfigSource.java \
        src/test/java/dev/claudony/config/EncryptionKeyConfigSourceTest.java
git commit -m "feat: EncryptionKeyConfigSource — per-deployment session key with unit tests"
```

---

## Task 3: Wire Up ServiceLoader and Remove Hardcoded Prod Key

**Files:**
- Create: `src/main/resources/META-INF/services/org.eclipse.microprofile.config.spi.ConfigSource`
- Modify: `src/main/resources/application.properties`

- [ ] **Step 1: Create the ServiceLoader registration file**

Create the file `src/main/resources/META-INF/services/org.eclipse.microprofile.config.spi.ConfigSource` with this exact content (one line, no trailing whitespace):

```
dev.claudony.config.EncryptionKeyConfigSource
```

This registers the ConfigSource with both the JVM ServiceLoader and Quarkus's native image build processor.

- [ ] **Step 2: Remove the hardcoded shared prod key from application.properties**

In `src/main/resources/application.properties`, replace the prod key block:

```properties
# Cookie encryption key — used to encrypt persistent auth cookies (WebAuthn sessions)
# Production: set QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY env var to a secret (>16 chars)
# Dev: fixed key so sessions survive restarts during development
%dev.quarkus.http.auth.session.encryption-key=WDvVpqyk2J-CdtTj6FTpEIus7ofJ9Wh0eZUysCwEuZc
%test.quarkus.http.auth.session.encryption-key=test-encryption-key-not-for-prod-1234567
# Local testing without env var: stable key so auth cookie survives dev server restarts
# Override in production with QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY env var
%prod.quarkus.http.auth.session.encryption-key=XrK9mP2vLq8nT5wY3jH7bN4cF6dA1eZ0
```

With this updated block (the `%prod` line is gone, comments updated):

```properties
# Cookie encryption key — used to encrypt persistent auth cookies (WebAuthn sessions)
# Dev/test: fixed key in application.properties so sessions survive restarts locally.
# Prod: auto-generated unique key via EncryptionKeyConfigSource, persisted to
#       ~/.claudony/encryption-key (rw------- permissions) on first server run.
# Override any profile: set QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY env var (takes precedence).
%dev.quarkus.http.auth.session.encryption-key=WDvVpqyk2J-CdtTj6FTpEIus7ofJ9Wh0eZUysCwEuZc
%test.quarkus.http.auth.session.encryption-key=test-encryption-key-not-for-prod-1234567
```

- [ ] **Step 3: Run the full test suite — all 139 existing tests must still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected output:
```
Tests run: 152, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

(139 existing + 13 new unit tests = 152)

- [ ] **Step 4: Commit**

```bash
git add src/main/resources/META-INF/services/org.eclipse.microprofile.config.spi.ConfigSource \
        src/main/resources/application.properties
git commit -m "feat: register EncryptionKeyConfigSource via ServiceLoader, remove hardcoded prod key"
```

---

## Task 4: Integration Tests (QuarkusTest)

**Files:**
- Create: `src/test/java/dev/claudony/config/EncryptionKeyConfigSourceIntegrationTest.java`

These tests start the full Quarkus container and verify that the ConfigSource is properly wired into the SmallRye Config runtime. The test profile's fixed key (ordinal 250) still wins for the actual Quarkus config value — but the tests inspect the ConfigSource directly to verify it is registered and behaves correctly as a component.

**Important:** `configSourceProvidesValueForEncryptionKey` calls `ourSource().getValue()` directly. This will read from (or create) `~/.claudony/encryption-key` on the test runner's machine — which is correct prod-like behaviour. The key will be persisted after the first test run, and read on subsequent runs.

- [ ] **Step 1: Write the integration test**

```java
package dev.claudony.config;

import io.quarkus.test.junit.QuarkusTest;
import org.eclipse.microprofile.config.ConfigProvider;
import org.eclipse.microprofile.config.spi.ConfigSource;
import org.junit.jupiter.api.Test;

import java.util.stream.StreamSupport;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class EncryptionKeyConfigSourceIntegrationTest {

    private static final String ENC_KEY_PROP = "quarkus.http.auth.session.encryption-key";

    /** Locates our ConfigSource in the live SmallRye Config registry. */
    private ConfigSource ourSource() {
        return StreamSupport.stream(
                        ConfigProvider.getConfig().getConfigSources().spliterator(), false)
                .filter(cs -> "claudony-file-encryption-key".equals(cs.getName()))
                .findFirst()
                .orElseThrow(() -> new AssertionError(
                        "EncryptionKeyConfigSource not registered — " +
                        "check META-INF/services/org.eclipse.microprofile.config.spi.ConfigSource"));
    }

    @Test
    void configSourceIsRegisteredInQuarkusRuntime() {
        // Verifies ServiceLoader registration is picked up at Quarkus bootstrap
        assertThat(ourSource()).isNotNull();
    }

    @Test
    void configSourceHasCorrectOrdinal() {
        // Ordinal 200 — below application.properties (250), above nothing in prod
        assertThat(ourSource().getOrdinal()).isEqualTo(200);
    }

    @Test
    void configSourceDoesNotExposeKeyInPropertyMap() {
        // Security: key must never appear in getProperties() to avoid config dumps/actuator exposure
        assertThat(ourSource().getProperties()).isEmpty();
    }

    @Test
    void configSourceProvidesValueForEncryptionKey() {
        // Direct getValue() call — bypasses ordinal resolution so we test our source specifically.
        // In test profile, the test key (ordinal 250) wins at the Config level,
        // but our source still returns a value when called directly.
        // This verifies the full generate/persist/load cycle runs in a live Quarkus instance.
        var key = ourSource().getValue(ENC_KEY_PROP);
        assertThat(key)
                .isNotBlank()
                .matches("[A-Za-z0-9_\\-]{10,}"); // Base64url, at least 10 chars
    }

    @Test
    void configSourceReturnsNullForUnrelatedProperties() {
        var source = ourSource();
        assertThat(source.getValue("some.other.property")).isNull();
        assertThat(source.getValue("claudony.mode")).isNull();
        assertThat(source.getValue("quarkus.http.host")).isNull();
    }
}
```

- [ ] **Step 2: Run integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EncryptionKeyConfigSourceIntegrationTest 2>&1 | tail -30
```

Expected:
```
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

- [ ] **Step 3: Run full test suite — verify total count**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected:
```
Tests run: 157, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

(139 existing + 13 unit + 5 integration = 157)

- [ ] **Step 4: Commit**

```bash
git add src/test/java/dev/claudony/config/EncryptionKeyConfigSourceIntegrationTest.java
git commit -m "test: integration tests for EncryptionKeyConfigSource in live Quarkus runtime"
```

---

## Task 5: Native Image Configuration

**Files:**
- Modify: `src/main/resources/META-INF/native-image/reflect-config.json`

`@RegisterForReflection` handles most native image registration, but the ServiceLoader needs the constructor to be reachable at native image runtime. Adding an explicit entry to `reflect-config.json` is belt-and-braces insurance alongside the annotation.

- [ ] **Step 1: Add EncryptionKeyConfigSource to reflect-config.json**

In `src/main/resources/META-INF/native-image/reflect-config.json`, add the new entry to the existing array. The final file should be:

```json
[
  {
    "name": "dev.claudony.server.model.Session",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.claudony.server.model.SessionStatus",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.claudony.server.model.CreateSessionRequest",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.claudony.server.model.SessionResponse",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.claudony.config.EncryptionKeyConfigSource",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredMethods": true
  }
]
```

- [ ] **Step 2: Run full test suite to confirm nothing is broken**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -10
```

Expected: `Tests run: 157, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/META-INF/native-image/reflect-config.json
git commit -m "feat: add EncryptionKeyConfigSource to native image reflect-config"
```

---

## Task 6: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update test count and prod key comment in CLAUDE.md**

Find this line in `CLAUDE.md`:
```
**139 tests passing** across:
```

Replace with:
```
**157 tests passing** across:
```

Find the bullet that lists auth test classes:
```
- `server/auth/` — ApiKeyService, ApiKeyAuthMechanism, AuthResource, AuthRateLimiter (+ AuthRateLimiterHttpTest for HTTP-level), CredentialStore, InviteService
```

Replace with:
```
- `server/auth/` — ApiKeyService, ApiKeyAuthMechanism, AuthResource, AuthRateLimiter (+ AuthRateLimiterHttpTest for HTTP-level), CredentialStore, InviteService
- `config/` — EncryptionKeyConfigSource (unit + QuarkusTest integration)
```

Also find:
```
# The application.properties has a fallback dev key, but prod mode generates a random one.
```

Replace with:
```
# Prod: auto-generated unique key via EncryptionKeyConfigSource, persisted to ~/.claudony/encryption-key.
```

- [ ] **Step 2: Run full test suite one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -10
```

Expected: `Tests run: 157, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 3: Final commit**

```bash
git add CLAUDE.md
git commit -m "docs: update test count to 157, fix prod key comment after EncryptionKeyConfigSource"
```

---

## E2E Verification (Manual)

Automated E2E for session persistence requires server restart, which the test suite doesn't support. Verify manually after deployment:

1. Start server in prod mode: `JAVA_HOME=$(/usr/libexec/java_home -v 26) java -Dclaudony.mode=server -Dclaudony.bind=0.0.0.0 -jar target/quarkus-app/quarkus-run.jar`
2. Check logs — expect the generation banner: `CLAUDONY — Session Encryption Key Generated (first run)`
3. Verify file created: `ls -la ~/.claudony/encryption-key` — should show `-rw-------`
4. Log in via passkey at `http://localhost:7777/auth/login`
5. Stop the server (`Ctrl+C`)
6. Restart the server (same command)
7. Check logs — expect `Loaded session encryption key from ~/.claudony/encryption-key` (DEBUG, not INFO banner)
8. Navigate to `http://localhost:7777/app/` — session should still be valid, no login prompt

If step 8 redirects to login, the key is not being loaded from file. Check file permissions and path.

---

## Summary

| Task | Tests added | Commit |
|---|---|---|
| Task 1+2: Unit tests + implementation | 13 | `feat: EncryptionKeyConfigSource` |
| Task 3: ServiceLoader + properties | 0 | `feat: register via ServiceLoader, remove hardcoded key` |
| Task 4: Integration tests | 5 | `test: integration tests` |
| Task 5: Native image | 0 | `feat: reflect-config` |
| Task 6: Docs | 0 | `docs: test count + comments` |
| **Total** | **18** | |
