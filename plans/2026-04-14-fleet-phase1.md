# Fleet Manager Phase 1 — Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the fleet infrastructure — peer registry, three discovery mechanisms, fleet key auth, session federation, and Docker deployment — so multiple Claudony instances can see each other's sessions via REST API (dashboard UI changes are Phase 2).

**Architecture:** A `PeerRegistry` (@ApplicationScoped) holds all known peers from three pluggable discovery sources (static config, manual registration, mDNS). A `PeerClient` Quarkus REST client fans out `GET /api/sessions?local=true` to all healthy peers in parallel. Circuit breakers per peer handle network failures gracefully. A shared fleet key (same `X-Api-Key` mechanism) authenticates all peer-to-peer calls.

**Tech Stack:** Java 21, Quarkus 3.9.5, Quarkus REST Client (`@RegisterRestClient`), `@Scheduled`, virtual threads, JUnit 5, AssertJ, QuarkusTest, Mockito, WireMock

---

## File Map

| File | Action |
|---|---|
| `src/main/java/dev/claudony/server/fleet/DiscoverySource.java` | **Create** — enum |
| `src/main/java/dev/claudony/server/fleet/TerminalMode.java` | **Create** — enum |
| `src/main/java/dev/claudony/server/fleet/PeerHealth.java` | **Create** — enum |
| `src/main/java/dev/claudony/server/fleet/CircuitState.java` | **Create** — enum |
| `src/main/java/dev/claudony/server/fleet/PeerRecord.java` | **Create** — immutable API record |
| `src/main/java/dev/claudony/server/fleet/PeerEntry.java` | **Create** — mutable internal state (package-private) |
| `src/main/java/dev/claudony/server/fleet/PeerRegistry.java` | **Create** — @ApplicationScoped authoritative peer list |
| `src/main/java/dev/claudony/server/fleet/PeerDiscoverySource.java` | **Create** — interface |
| `src/main/java/dev/claudony/server/fleet/StaticConfigDiscovery.java` | **Create** — reads `claudony.peers` |
| `src/main/java/dev/claudony/server/fleet/ManualRegistrationDiscovery.java` | **Create** — REST-triggered |
| `src/main/java/dev/claudony/server/fleet/MdnsDiscovery.java` | **Create** — mDNS advertise/discover |
| `src/main/java/dev/claudony/server/fleet/PeerClient.java` | **Create** — @RegisterRestClient |
| `src/main/java/dev/claudony/server/fleet/FleetKeyClientFilter.java` | **Create** — ClientRequestFilter |
| `src/main/java/dev/claudony/server/fleet/PeerResource.java` | **Create** — REST /api/peers |
| `src/main/java/dev/claudony/server/auth/FleetKeyService.java` | **Create** — fleet key load/generate |
| `src/main/java/dev/claudony/server/auth/ApiKeyAuthMechanism.java` | **Modify** — accept fleet key |
| `src/main/java/dev/claudony/config/ClaudonyConfig.java` | **Modify** — add fleet-key, peers, mdns-discovery |
| `src/main/java/dev/claudony/server/model/SessionResponse.java` | **Modify** — add instanceUrl, instanceName, stale, withInstance() |
| `src/main/java/dev/claudony/server/SessionResource.java` | **Modify** — federated list() + local=true param |
| `src/main/resources/application.properties` | **Modify** — new fleet properties |
| `Dockerfile` | **Create** |
| `docker-compose.yml` | **Create** |
| `CLAUDE.md` | **Modify** — test count, fleet package |

Test files:
| File | Tests |
|---|---|
| `src/test/java/dev/claudony/server/fleet/PeerRegistryTest.java` | Unit — 12+ tests |
| `src/test/java/dev/claudony/server/fleet/StaticConfigDiscoveryTest.java` | Unit — 4 tests |
| `src/test/java/dev/claudony/server/fleet/MdnsDiscoveryTest.java` | Unit — 2 tests |
| `src/test/java/dev/claudony/server/auth/FleetKeyServiceTest.java` | Unit — 6 tests |
| `src/test/java/dev/claudony/server/fleet/PeerResourceTest.java` | @QuarkusTest — 10 tests |
| `src/test/java/dev/claudony/server/fleet/SessionFederationTest.java` | @QuarkusTest — 5 tests |
| `src/test/java/dev/claudony/server/auth/FleetKeyAuthTest.java` | @QuarkusTest — 3 tests |

**Total new tests: ~42. New CLAUDE.md count: 162 → ~204.**

---

## Task 1: Data Model — Enums, PeerRecord, PeerEntry

**Files:**
- Create: `src/main/java/dev/claudony/server/fleet/DiscoverySource.java`
- Create: `src/main/java/dev/claudony/server/fleet/TerminalMode.java`
- Create: `src/main/java/dev/claudony/server/fleet/PeerHealth.java`
- Create: `src/main/java/dev/claudony/server/fleet/CircuitState.java`
- Create: `src/main/java/dev/claudony/server/fleet/PeerRecord.java`
- Create: `src/main/java/dev/claudony/server/fleet/PeerEntry.java`

- [ ] **Step 1: Create the four enums**

```java
// DiscoverySource.java
package dev.claudony.server.fleet;
public enum DiscoverySource { CONFIG, MANUAL, MDNS }

// TerminalMode.java
package dev.claudony.server.fleet;
public enum TerminalMode { DIRECT, PROXY }

// PeerHealth.java
package dev.claudony.server.fleet;
public enum PeerHealth { UP, DOWN, UNKNOWN }

// CircuitState.java
package dev.claudony.server.fleet;
public enum CircuitState { CLOSED, OPEN, HALF_OPEN }
```

- [ ] **Step 2: Create PeerRecord (the external-facing API view)**

```java
package dev.claudony.server.fleet;

import java.time.Instant;

public record PeerRecord(
        String id,
        String url,
        String name,
        DiscoverySource source,
        TerminalMode terminalMode,
        PeerHealth health,
        CircuitState circuitState,
        Instant lastSeen,
        boolean stale,
        int sessionCount) {}
```

- [ ] **Step 3: Create PeerEntry (mutable internal state, package-private)**

```java
package dev.claudony.server.fleet;

import dev.claudony.server.model.SessionResponse;
import java.time.Instant;
import java.util.List;

final class PeerEntry {

    static final long INITIAL_BACKOFF_MS = 30_000L;
    static final long MAX_BACKOFF_MS = 300_000L; // 5 minutes
    static final int FAILURE_THRESHOLD = 3;

    final String id;
    final String url;
    volatile String name;
    final DiscoverySource source;
    volatile TerminalMode terminalMode;
    volatile PeerHealth health = PeerHealth.UNKNOWN;
    volatile CircuitState circuitState = CircuitState.CLOSED;
    volatile int consecutiveFailures = 0;
    volatile Instant lastSeen = null;
    volatile Instant circuitOpenedAt = null;
    volatile long currentBackoffMs = INITIAL_BACKOFF_MS;
    volatile List<SessionResponse> cachedSessions = List.of();

    PeerEntry(String id, String url, String name, DiscoverySource source, TerminalMode terminalMode) {
        this.id = id;
        this.url = url;
        this.name = name;
        this.source = source;
        this.terminalMode = terminalMode;
    }

    PeerRecord toRecord() {
        return new PeerRecord(id, url, name, source, terminalMode, health, circuitState,
                lastSeen, health == PeerHealth.DOWN && !cachedSessions.isEmpty(), cachedSessions.size());
    }

    /** Record a successful peer call — resets circuit to CLOSED. */
    void recordSuccess() {
        consecutiveFailures = 0;
        currentBackoffMs = INITIAL_BACKOFF_MS;
        circuitState = CircuitState.CLOSED;
        health = PeerHealth.UP;
        lastSeen = Instant.now();
    }

    /** Record a failed peer call — may open circuit after threshold. */
    void recordFailure() {
        consecutiveFailures++;
        health = PeerHealth.DOWN;
        if (consecutiveFailures >= FAILURE_THRESHOLD && circuitState == CircuitState.CLOSED) {
            circuitState = CircuitState.OPEN;
            circuitOpenedAt = Instant.now();
        }
    }

    /**
     * Returns true if a health check attempt is warranted.
     * CLOSED: always try. OPEN: try only after backoff. HALF_OPEN: try once.
     */
    boolean shouldAttemptHealthCheck() {
        return switch (circuitState) {
            case CLOSED -> true;
            case HALF_OPEN -> true;
            case OPEN -> {
                var elapsed = Instant.now().toEpochMilli() - circuitOpenedAt.toEpochMilli();
                if (elapsed >= currentBackoffMs) {
                    circuitState = CircuitState.HALF_OPEN;
                    currentBackoffMs = Math.min(currentBackoffMs * 2, MAX_BACKOFF_MS);
                    yield true;
                }
                yield false;
            }
        };
    }
}
```

- [ ] **Step 4: Verify everything compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/
git commit -m "feat: fleet data model — enums, PeerRecord, PeerEntry with circuit breaker state"
```

---

## Task 2: Config Additions + SessionResponse Fleet Fields

**Files:**
- Modify: `src/main/java/dev/claudony/config/ClaudonyConfig.java`
- Modify: `src/main/java/dev/claudony/server/model/SessionResponse.java`
- Modify: `src/main/resources/application.properties`

- [ ] **Step 1: Add fleet properties to ClaudonyConfig**

Add to `src/main/java/dev/claudony/config/ClaudonyConfig.java` (add import `java.time.Duration` is already there from session-timeout):

```java
// Add this import if not present:
import java.util.Optional;

// Add these methods to the interface (before the default methods):

@WithName("fleet-key")
Optional<String> fleetKey();

@WithName("peers")
@WithDefault("")
String peers(); // comma-separated peer URLs; empty string means no static peers

@WithName("mdns-discovery")
@WithDefault("false")
boolean mdnsDiscovery();

@WithName("name")
@WithDefault("Claudony")
String name(); // human-readable instance name shown in fleet dashboard
```

- [ ] **Step 2: Add fleet comment to application.properties**

Add after the existing `claudony.*` properties block:

```properties
# Fleet configuration — optional, for multi-instance deployments
# claudony.peers=http://mac-mini:7777,http://vps.example.com:7777
# claudony.fleet-key=   (generate with POST /api/peers/generate-fleet-key)
# claudony.mdns-discovery=false
# claudony.name=Claudony
```

- [ ] **Step 3: Add fleet fields to SessionResponse**

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
        String instanceUrl,   // null for local sessions; non-null for remote peers
        String instanceName,  // null for local sessions; peer name for remote sessions
        Boolean stale) {      // null for local; true if peer's cached data, false if live

    /** Local session — no fleet fields. */
    public static SessionResponse from(Session session, int port) {
        return new SessionResponse(
                session.id(), session.name(), session.workingDir(), session.command(),
                session.status(), session.createdAt(), session.lastActive(),
                "ws://localhost:" + port + "/ws/" + session.id(),
                "http://localhost:" + port + "/app/session/" + session.id(),
                null, null, null);
    }

    /** Remote session from a peer — preserves the original wsUrl/browserUrl pointing to the peer. */
    public SessionResponse withInstance(String peerUrl, String peerName, boolean isStale) {
        return new SessionResponse(
                id, name, workingDir, command, status, createdAt, lastActive,
                wsUrl, browserUrl, peerUrl, peerName, isStale);
    }
}
```

- [ ] **Step 4: Run existing tests — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: `Tests run: 162, Failures: 0, Errors: 0, Skipped: 0`

`@JsonInclude(NON_NULL)` ensures null fleet fields are absent from JSON output, preserving backward compatibility with existing tests that check JSON structure.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/config/ClaudonyConfig.java \
        src/main/java/dev/claudony/server/model/SessionResponse.java \
        src/main/resources/application.properties
git commit -m "feat: fleet config properties + SessionResponse fleet fields (instanceUrl, instanceName, stale)"
```

---

## Task 3: FleetKeyService

**Files:**
- Create: `src/main/java/dev/claudony/server/auth/FleetKeyService.java`
- Create: `src/test/java/dev/claudony/server/auth/FleetKeyServiceTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `src/test/java/dev/claudony/server/auth/FleetKeyServiceTest.java`:

```java
package dev.claudony.server.auth;

import dev.claudony.config.ClaudonyConfig;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class FleetKeyServiceTest {

    @TempDir
    Path tempDir;

    private FleetKeyService service(String configKey) {
        var config = mock(ClaudonyConfig.class);
        when(config.fleetKey()).thenReturn(configKey == null ? Optional.empty() : Optional.of(configKey));
        when(config.credentialsFile()).thenReturn(tempDir.resolve("credentials.json").toString());
        return new FleetKeyService(config, tempDir.resolve("fleet-key"));
    }

    @Test
    void configKeyTakesPrecedence() {
        var svc = service("config-fleet-key");
        svc.init();
        assertThat(svc.getKey()).contains("config-fleet-key");
    }

    @Test
    void loadsKeyFromFile() throws IOException {
        Files.writeString(tempDir.resolve("fleet-key"), "file-fleet-key");
        var svc = service(null);
        svc.init();
        assertThat(svc.getKey()).contains("file-fleet-key");
    }

    @Test
    void emptyWhenNeitherConfigNorFile() {
        var svc = service(null);
        svc.init();
        assertThat(svc.getKey()).isEmpty();
    }

    @Test
    void generateAndSave_createsFileAndUpdatesKey() throws IOException {
        var svc = service(null);
        svc.init();

        var generated = svc.generateAndSave();

        assertThat(generated).isNotBlank().matches("[A-Za-z0-9_\\-]{43}");
        assertThat(svc.getKey()).contains(generated);
        assertThat(Files.readString(tempDir.resolve("fleet-key")).strip()).isEqualTo(generated);
    }

    @Test
    void blankFileIsIgnored() throws IOException {
        Files.writeString(tempDir.resolve("fleet-key"), "   ");
        var svc = service(null);
        svc.init();
        assertThat(svc.getKey()).isEmpty();
    }

    @Test
    void configKeyWinsOverFile() throws IOException {
        Files.writeString(tempDir.resolve("fleet-key"), "file-key");
        var svc = service("config-key");
        svc.init();
        assertThat(svc.getKey()).contains("config-key");
    }
}
```

- [ ] **Step 2: Run to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FleetKeyServiceTest 2>&1 | tail -10
```

Expected: compilation error — `cannot find symbol: class FleetKeyService`

- [ ] **Step 3: Implement FleetKeyService**

Create `src/main/java/dev/claudony/server/auth/FleetKeyService.java`:

```java
package dev.claudony.server.auth;

import dev.claudony.config.ClaudonyConfig;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.attribute.PosixFilePermissions;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.Optional;

@ApplicationScoped
public class FleetKeyService {

    private static final Logger LOG = Logger.getLogger(FleetKeyService.class);

    @Inject
    ClaudonyConfig config;

    private final Path keyFile;
    private volatile Optional<String> key = Optional.empty();

    /** CDI no-arg constructor. */
    FleetKeyService() {
        this.keyFile = Path.of(System.getProperty("user.home"), ".claudony", "fleet-key");
    }

    /** Package-private constructor for unit tests — injects temp key file path. */
    FleetKeyService(ClaudonyConfig config, Path keyFile) {
        this.config = config;
        this.keyFile = keyFile;
    }

    @PostConstruct
    void init() {
        key = config.fleetKey().filter(k -> !k.isBlank());
        if (key.isPresent()) return;

        key = loadFromFile();
        if (key.isEmpty()) {
            LOG.warn("claudony.fleet-key not configured — peer-to-peer calls will be" +
                    " rejected by authenticated peers. Generate with POST /api/peers/generate-fleet-key");
        }
    }

    public Optional<String> getKey() {
        return key;
    }

    /**
     * Generates a new random fleet key, persists it to disk, updates in-memory key.
     * Called by POST /api/peers/generate-fleet-key.
     */
    public String generateAndSave() throws IOException {
        var bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        var newKey = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);

        Files.createDirectories(keyFile.getParent());
        Files.writeString(keyFile, newKey);
        try {
            Files.setPosixFilePermissions(keyFile, PosixFilePermissions.fromString("rw-------"));
        } catch (UnsupportedOperationException ignored) {
            // Non-POSIX filesystem — skip chmod
        }

        key = Optional.of(newKey);
        LOG.infof("Generated new fleet key — saved to %s. Copy to all fleet members via CLAUDONY_FLEET_KEY.", keyFile);
        return newKey;
    }

    private Optional<String> loadFromFile() {
        if (!Files.exists(keyFile)) return Optional.empty();
        try {
            var k = Files.readString(keyFile).strip();
            return k.isBlank() ? Optional.empty() : Optional.of(k);
        } catch (IOException e) {
            LOG.warnf("Could not read fleet key from %s: %s", keyFile, e.getMessage());
            return Optional.empty();
        }
    }
}
```

- [ ] **Step 4: Run tests — all 6 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FleetKeyServiceTest 2>&1 | tail -5
```

Expected: `Tests run: 6, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/auth/FleetKeyService.java \
        src/test/java/dev/claudony/server/auth/FleetKeyServiceTest.java
git commit -m "feat: FleetKeyService — fleet key load from config/file, generate-and-save"
```

---

## Task 4: Fleet Key Auth (extend ApiKeyAuthMechanism)

**Files:**
- Modify: `src/main/java/dev/claudony/server/auth/ApiKeyAuthMechanism.java`
- Create: `src/test/java/dev/claudony/server/auth/FleetKeyAuthTest.java`

- [ ] **Step 1: Write failing integration test**

Create `src/test/java/dev/claudony/server/auth/FleetKeyAuthTest.java`:

```java
package dev.claudony.server.auth;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.not;
import static org.hamcrest.Matchers.equalTo;

@QuarkusTest
class FleetKeyAuthTest {

    // The test profile has claudony.agent.api-key=test-api-key-do-not-use-in-prod
    // We need a test fleet key — set via application.properties %test profile

    @Test
    void agentApiKeyStillAccepted() {
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().get("/api/sessions")
            .then().statusCode(200);
    }

    @Test
    void fleetKeyAccepted() {
        given()
            .header("X-Api-Key", "test-fleet-key-do-not-use-in-prod")
            .when().get("/api/sessions")
            .then().statusCode(200);
    }

    @Test
    void unknownKeyRejected() {
        given()
            .header("X-Api-Key", "not-a-valid-key")
            .when().get("/api/sessions")
            .then().statusCode(401);
    }
}
```

- [ ] **Step 2: Add test fleet key to application.properties**

In `src/main/resources/application.properties`, add:

```properties
# Test profile: fixed fleet key so FleetKeyAuthTest can test the mechanism
%test.claudony.fleet-key=test-fleet-key-do-not-use-in-prod
```

- [ ] **Step 3: Run to confirm fleetKeyAccepted fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FleetKeyAuthTest 2>&1 | grep -E "FAIL|ERROR|Tests run"
```

Expected: `fleetKeyAccepted` fails (fleet key not yet accepted by auth mechanism).

- [ ] **Step 4: Extend ApiKeyAuthMechanism to accept fleet key**

In `src/main/java/dev/claudony/server/auth/ApiKeyAuthMechanism.java`:

Add `@Inject FleetKeyService fleetKeyService;` field.

Replace the authentication check block (after the existing agent key check) to also accept fleet key. The full updated `authenticate()` method:

```java
@Override
public Uni<SecurityIdentity> authenticate(RoutingContext context, IdentityProviderManager identityProviderManager) {
    var apiKey = context.request().getHeader("X-Api-Key");
    if (apiKey == null && LaunchMode.current() == LaunchMode.DEVELOPMENT) {
        var devCookie = context.getCookie("claudony-dev-key");
        if (devCookie != null) apiKey = devCookie.getValue();
    }
    if (apiKey == null) {
        return Uni.createFrom().optional(Optional.empty());
    }

    // Check agent API key
    var agentKey = apiKeyService.getKey();
    if (agentKey.isPresent() && MessageDigest.isEqual(
            agentKey.get().getBytes(StandardCharsets.UTF_8),
            apiKey.getBytes(StandardCharsets.UTF_8))) {
        return Uni.createFrom().item(
            QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("agent"))
                .addRole("user")
                .build());
    }

    // Check fleet key (peer-to-peer calls)
    var fleetKey = fleetKeyService.getKey();
    if (fleetKey.isPresent() && MessageDigest.isEqual(
            fleetKey.get().getBytes(StandardCharsets.UTF_8),
            apiKey.getBytes(StandardCharsets.UTF_8))) {
        return Uni.createFrom().item(
            QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("peer"))
                .addRole("user")
                .build());
    }

    return Uni.createFrom().failure(
        new io.quarkus.security.AuthenticationFailedException("Invalid API key"));
}
```

Also add the import and field at the top of the class:

```java
@Inject
FleetKeyService fleetKeyService;
```

- [ ] **Step 5: Run all 3 fleet auth tests — all must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FleetKeyAuthTest 2>&1 | tail -5
```

Expected: `Tests run: 3, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 6: Run full test suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: all existing 162 tests + 9 new (6 FleetKeyService + 3 FleetKeyAuth) = 171 tests passing.

- [ ] **Step 7: Commit**

```bash
git add src/main/java/dev/claudony/server/auth/ApiKeyAuthMechanism.java \
        src/main/resources/application.properties \
        src/test/java/dev/claudony/server/auth/FleetKeyAuthTest.java
git commit -m "feat: fleet key auth — ApiKeyAuthMechanism accepts fleet key alongside agent key"
```

---

## Task 5: PeerRegistry Core

**Files:**
- Create: `src/main/java/dev/claudony/server/fleet/PeerRegistry.java`
- Create: `src/test/java/dev/claudony/server/fleet/PeerRegistryTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `src/test/java/dev/claudony/server/fleet/PeerRegistryTest.java`:

```java
package dev.claudony.server.fleet;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class PeerRegistryTest {

    @TempDir
    Path tempDir;

    PeerRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new PeerRegistry(tempDir);
    }

    // ─── Basic CRUD ──────────────────────────────────────────────────────────

    @Test
    void emptyOnStart() {
        assertThat(registry.getAllPeers()).isEmpty();
    }

    @Test
    void addPeerAppearsInList() {
        registry.addPeer("id1", "http://peer-a:7777", "Peer A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        assertThat(registry.getAllPeers()).hasSize(1);
        assertThat(registry.getAllPeers().get(0).url()).isEqualTo("http://peer-a:7777");
    }

    @Test
    void removePeer() {
        registry.addPeer("id1", "http://peer-a:7777", "Peer A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        registry.removePeer("id1");
        assertThat(registry.getAllPeers()).isEmpty();
    }

    @Test
    void findPeerById() {
        registry.addPeer("id1", "http://peer-a:7777", "Peer A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        assertThat(registry.findById("id1")).isPresent();
        assertThat(registry.findById("missing")).isEmpty();
    }

    // ─── Deduplication ───────────────────────────────────────────────────────

    @Test
    void deduplicatesByUrl_lowerSourceWins() {
        // CONFIG wins over MANUAL
        registry.addPeer("id1", "http://peer-a:7777", "Config Peer", DiscoverySource.CONFIG, TerminalMode.DIRECT);
        registry.addPeer("id2", "http://peer-a:7777", "Manual Peer", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        assertThat(registry.getAllPeers()).hasSize(1);
        assertThat(registry.getAllPeers().get(0).source()).isEqualTo(DiscoverySource.CONFIG);
    }

    @Test
    void deduplicatesByUrl_mdnsLosesToManual() {
        registry.addPeer("id1", "http://peer-a:7777", "mDNS Peer", DiscoverySource.MDNS, TerminalMode.DIRECT);
        registry.addPeer("id2", "http://peer-a:7777", "Manual Peer", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        assertThat(registry.getAllPeers()).hasSize(1);
        assertThat(registry.getAllPeers().get(0).source()).isEqualTo(DiscoverySource.MANUAL);
    }

    @Test
    void differentUrlsCoexist() {
        registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        registry.addPeer("id2", "http://peer-b:7777", "B", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        assertThat(registry.getAllPeers()).hasSize(2);
    }

    // ─── Config peers cannot be removed ─────────────────────────────────────

    @Test
    void configPeerCannotBeRemoved() {
        registry.addPeer("id1", "http://peer-a:7777", "Config", DiscoverySource.CONFIG, TerminalMode.DIRECT);
        var removed = registry.removePeer("id1");
        assertThat(removed).isFalse();
        assertThat(registry.getAllPeers()).hasSize(1);
    }

    @Test
    void manualPeerCanBeRemoved() {
        registry.addPeer("id1", "http://peer-a:7777", "Manual", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        var removed = registry.removePeer("id1");
        assertThat(removed).isTrue();
        assertThat(registry.getAllPeers()).isEmpty();
    }

    // ─── Health and circuit state ─────────────────────────────────────────

    @Test
    void newPeerHealthIsUnknown() {
        registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        assertThat(registry.getAllPeers().get(0).health()).isEqualTo(PeerHealth.UNKNOWN);
        assertThat(registry.getAllPeers().get(0).circuitState()).isEqualTo(CircuitState.CLOSED);
    }

    @Test
    void recordSuccessUpdatesHealth() {
        registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        registry.recordSuccess("id1");
        assertThat(registry.findById("id1").get().health()).isEqualTo(PeerHealth.UP);
    }

    @Test
    void threeFailuresOpenCircuit() {
        registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        registry.recordFailure("id1");
        registry.recordFailure("id1");
        assertThat(registry.findById("id1").get().circuitState()).isEqualTo(CircuitState.CLOSED);
        registry.recordFailure("id1");
        assertThat(registry.findById("id1").get().circuitState()).isEqualTo(CircuitState.OPEN);
    }

    @Test
    void successAfterOpenResetsToClose() {
        registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        registry.recordFailure("id1");
        registry.recordFailure("id1");
        registry.recordFailure("id1");
        assertThat(registry.findById("id1").get().circuitState()).isEqualTo(CircuitState.OPEN);
        registry.recordSuccess("id1");
        assertThat(registry.findById("id1").get().circuitState()).isEqualTo(CircuitState.CLOSED);
    }

    // ─── Healthy peers for federation ────────────────────────────────────────

    @Test
    void getHealthyPeers_excludesOpenCircuit() {
        registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        registry.addPeer("id2", "http://peer-b:7777", "B", DiscoverySource.MANUAL, TerminalMode.DIRECT);
        // Open circuit on peer-a
        registry.recordFailure("id1");
        registry.recordFailure("id1");
        registry.recordFailure("id1");
        assertThat(registry.getHealthyPeers()).hasSize(1);
        assertThat(registry.getHealthyPeers().get(0).id()).isEqualTo("id2");
    }
}
```

- [ ] **Step 2: Run to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerRegistryTest 2>&1 | tail -5
```

Expected: compilation error — `cannot find symbol: class PeerRegistry`

- [ ] **Step 3: Implement PeerRegistry**

Create `src/main/java/dev/claudony/server/fleet/PeerRegistry.java`:

```java
package dev.claudony.server.fleet;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import dev.claudony.server.model.SessionResponse;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class PeerRegistry {

    private static final Logger LOG = Logger.getLogger(PeerRegistry.class);
    private static final int SOURCE_PRIORITY_CONFIG = 0;
    private static final int SOURCE_PRIORITY_MANUAL = 1;
    private static final int SOURCE_PRIORITY_MDNS = 2;

    private final ConcurrentHashMap<String, PeerEntry> peers = new ConcurrentHashMap<>();
    private final Path peersFile;
    private final ObjectMapper mapper;

    /** CDI constructor. */
    @Inject
    PeerRegistry() {
        this.peersFile = Path.of(System.getProperty("user.home"), ".claudony", "peers.json");
        this.mapper = new ObjectMapper().registerModule(new JavaTimeModule());
    }

    /** Package-private constructor for unit tests. */
    PeerRegistry(Path configDir) {
        this.peersFile = configDir.resolve("peers.json");
        this.mapper = new ObjectMapper().registerModule(new JavaTimeModule());
    }

    @PostConstruct
    void loadPersistedPeers() {
        if (!Files.exists(peersFile)) return;
        try {
            var json = Files.readString(peersFile);
            var records = mapper.readValue(json, PeerRecord[].class);
            for (var record : records) {
                if (record.source() == DiscoverySource.CONFIG) continue; // re-loaded from config
                var entry = new PeerEntry(record.id(), record.url(), record.name(), record.source(), record.terminalMode());
                peers.put(record.id(), entry);
            }
            LOG.infof("Loaded %d peers from %s", peers.size(), peersFile);
        } catch (Exception e) {
            LOG.warnf("Could not load peers from %s: %s — starting with empty peer list", peersFile, e.getMessage());
        }
    }

    // ─── Public API ───────────────────────────────────────────────────────────

    /**
     * Adds a peer. Deduplicates by URL — higher source priority wins.
     * Source priority: CONFIG (0) < MANUAL (1) < MDNS (2).
     */
    public synchronized void addPeer(String id, String url, String name, DiscoverySource source, TerminalMode terminalMode) {
        // Check for existing peer with same URL
        var existing = peers.values().stream()
                .filter(e -> e.url.equals(url))
                .findFirst();

        if (existing.isPresent()) {
            if (sourcePriority(source) < sourcePriority(existing.get().source)) {
                // New source has higher trust — replace
                peers.remove(existing.get().id);
            } else {
                // Existing has equal or higher trust — keep existing
                return;
            }
        }

        peers.put(id, new PeerEntry(id, url, name, source, terminalMode));
        if (source == DiscoverySource.MANUAL || source == DiscoverySource.MDNS) {
            persistAsync();
        }
    }

    /**
     * Removes a peer by ID. CONFIG peers cannot be removed — returns false.
     */
    public synchronized boolean removePeer(String id) {
        var entry = peers.get(id);
        if (entry == null) return false;
        if (entry.source == DiscoverySource.CONFIG) return false;
        peers.remove(id);
        persistAsync();
        return true;
    }

    public Optional<PeerRecord> findById(String id) {
        return Optional.ofNullable(peers.get(id)).map(PeerEntry::toRecord);
    }

    public List<PeerRecord> getAllPeers() {
        return peers.values().stream().map(PeerEntry::toRecord).toList();
    }

    /**
     * Returns PeerRecords for peers whose circuit is CLOSED or HALF_OPEN — eligible for federation calls.
     * Returns PeerRecord (public) not PeerEntry (package-private) so callers outside this package can use it.
     */
    public List<PeerRecord> getHealthyPeers() {
        return peers.values().stream()
                .filter(e -> e.circuitState != CircuitState.OPEN)
                .map(PeerEntry::toRecord)
                .toList();
    }

    /** Returns all entries (package-private — for health check loop inside this package). */
    List<PeerEntry> getAllEntries() {
        return List.copyOf(peers.values());
    }

    public boolean updatePeer(String id, String name, TerminalMode terminalMode) {
        var entry = peers.get(id);
        if (entry == null) return false;
        if (name != null) entry.name = name;
        if (terminalMode != null) entry.terminalMode = terminalMode;
        persistAsync();
        return true;
    }

    public void recordSuccess(String id) {
        var entry = peers.get(id);
        if (entry != null) entry.recordSuccess();
    }

    public void recordFailure(String id) {
        var entry = peers.get(id);
        if (entry != null) entry.recordFailure();
    }

    public void updateCachedSessions(String id, List<SessionResponse> sessions) {
        var entry = peers.get(id);
        if (entry != null) entry.cachedSessions = List.copyOf(sessions);
    }

    public List<SessionResponse> getCachedSessions(String id) {
        var entry = peers.get(id);
        return entry == null ? List.of() : entry.cachedSessions;
    }

    // ─── Persistence ──────────────────────────────────────────────────────────

    private void persistAsync() {
        Thread.ofVirtual().start(this::persist);
    }

    void persist() {
        try {
            var toSave = peers.values().stream()
                    .filter(e -> e.source != DiscoverySource.CONFIG)
                    .map(PeerEntry::toRecord)
                    .toList();
            var json = mapper.writeValueAsString(toSave);
            Files.createDirectories(peersFile.getParent());
            var tmp = peersFile.resolveSibling("peers.json.tmp");
            Files.writeString(tmp, json);
            Files.move(tmp, peersFile, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            LOG.warnf("Could not persist peers to %s: %s", peersFile, e.getMessage());
        }
    }

    // ─── Helpers ─────────────────────────────────────────────────────────────

    private static int sourcePriority(DiscoverySource source) {
        return switch (source) {
            case CONFIG -> SOURCE_PRIORITY_CONFIG;
            case MANUAL -> SOURCE_PRIORITY_MANUAL;
            case MDNS -> SOURCE_PRIORITY_MDNS;
        };
    }
}
```

- [ ] **Step 4: Run PeerRegistryTest — all 15 tests must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerRegistryTest 2>&1 | tail -5
```

Expected: `Tests run: 15, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/PeerRegistry.java \
        src/test/java/dev/claudony/server/fleet/PeerRegistryTest.java
git commit -m "feat: PeerRegistry — in-memory peer list with circuit breaker and atomic persistence"
```

---

## Task 6: StaticConfigDiscovery

**Files:**
- Create: `src/main/java/dev/claudony/server/fleet/PeerDiscoverySource.java`
- Create: `src/main/java/dev/claudony/server/fleet/StaticConfigDiscovery.java`
- Create: `src/test/java/dev/claudony/server/fleet/StaticConfigDiscoveryTest.java`

- [ ] **Step 1: Create PeerDiscoverySource interface**

```java
package dev.claudony.server.fleet;

public interface PeerDiscoverySource {
    String name();
    void discover(PeerRegistry registry);
}
```

- [ ] **Step 2: Write failing tests**

Create `src/test/java/dev/claudony/server/fleet/StaticConfigDiscoveryTest.java`:

```java
package dev.claudony.server.fleet;

import dev.claudony.config.ClaudonyConfig;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class StaticConfigDiscoveryTest {

    @TempDir Path tempDir;

    private PeerRegistry registry() { return new PeerRegistry(tempDir); }

    private ClaudonyConfig config(String peers) {
        var c = mock(ClaudonyConfig.class);
        when(c.peers()).thenReturn(peers == null ? "" : peers);
        when(c.fleetKey()).thenReturn(Optional.empty());
        return c;
    }

    @Test
    void emptyConfigAddsNoPeers() {
        var registry = registry();
        new StaticConfigDiscovery(config(""), registry).discover(registry);
        assertThat(registry.getAllPeers()).isEmpty();
    }

    @Test
    void singlePeerAdded() {
        var registry = registry();
        new StaticConfigDiscovery(config("http://mac-mini:7777"), registry).discover(registry);
        assertThat(registry.getAllPeers()).hasSize(1);
        assertThat(registry.getAllPeers().get(0).url()).isEqualTo("http://mac-mini:7777");
        assertThat(registry.getAllPeers().get(0).source()).isEqualTo(DiscoverySource.CONFIG);
    }

    @Test
    void multiplePeersCommaSeparated() {
        var registry = registry();
        new StaticConfigDiscovery(config("http://a:7777,http://b:7777,http://c:7777"), registry).discover(registry);
        assertThat(registry.getAllPeers()).hasSize(3);
    }

    @Test
    void blanksInCommaSeparatedListSkipped() {
        var registry = registry();
        new StaticConfigDiscovery(config("http://a:7777, , http://b:7777"), registry).discover(registry);
        assertThat(registry.getAllPeers()).hasSize(2);
    }
}
```

- [ ] **Step 3: Run to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StaticConfigDiscoveryTest 2>&1 | tail -5
```

Expected: compilation error — `cannot find symbol: class StaticConfigDiscovery`

- [ ] **Step 4: Implement StaticConfigDiscovery**

Create `src/main/java/dev/claudony/server/fleet/StaticConfigDiscovery.java`:

```java
package dev.claudony.server.fleet;

import dev.claudony.config.ClaudonyConfig;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.UUID;

@ApplicationScoped
public class StaticConfigDiscovery implements PeerDiscoverySource {

    private static final Logger LOG = Logger.getLogger(StaticConfigDiscovery.class);

    @Inject ClaudonyConfig config;
    @Inject PeerRegistry registry;

    /** Package-private constructor for unit tests. */
    StaticConfigDiscovery(ClaudonyConfig config, PeerRegistry registry) {
        this.config = config;
        this.registry = registry;
    }

    /** CDI no-arg constructor. */
    StaticConfigDiscovery() {}

    @PostConstruct
    void init() {
        discover(registry);
    }

    @Override
    public String name() { return "static-config"; }

    @Override
    public void discover(PeerRegistry registry) {
        var peers = config.peers();
        if (peers == null || peers.isBlank()) return;

        for (var raw : peers.split(",")) {
            var url = raw.strip();
            if (url.isBlank()) continue;
            var id = UUID.randomUUID().toString();
            registry.addPeer(id, url, url, DiscoverySource.CONFIG, TerminalMode.DIRECT);
            LOG.infof("Fleet: registered static peer %s", url);
        }
    }
}
```

- [ ] **Step 5: Run tests — all 4 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StaticConfigDiscoveryTest 2>&1 | tail -5
```

Expected: `Tests run: 4, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 6: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/ \
        src/test/java/dev/claudony/server/fleet/StaticConfigDiscoveryTest.java
git commit -m "feat: StaticConfigDiscovery — loads claudony.peers into PeerRegistry at startup"
```

---

## Task 7: FleetKeyClientFilter + PeerClient

**Files:**
- Create: `src/main/java/dev/claudony/server/fleet/FleetKeyClientFilter.java`
- Create: `src/main/java/dev/claudony/server/fleet/PeerClient.java`

- [ ] **Step 1: Create FleetKeyClientFilter**

```java
package dev.claudony.server.fleet;

import dev.claudony.server.auth.FleetKeyService;
import jakarta.inject.Inject;
import jakarta.ws.rs.client.ClientRequestContext;
import jakarta.ws.rs.client.ClientRequestFilter;

import java.io.IOException;

public class FleetKeyClientFilter implements ClientRequestFilter {

    @Inject
    FleetKeyService fleetKeyService;

    @Override
    public void filter(ClientRequestContext requestContext) throws IOException {
        fleetKeyService.getKey().ifPresent(key ->
                requestContext.getHeaders().putSingle("X-Api-Key", key));
    }
}
```

- [ ] **Step 2: Create PeerClient**

```java
package dev.claudony.server.fleet;

import dev.claudony.server.model.SessionResponse;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.QueryParam;
import org.eclipse.microprofile.rest.client.annotation.RegisterProvider;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;

import java.util.List;

@RegisterRestClient(configKey = "peer-client")
@RegisterProvider(FleetKeyClientFilter.class)
@Path("/api")
public interface PeerClient {

    /**
     * Fetches sessions from a peer.
     * Pass local=true to get only that peer's local sessions (prevents recursive federation).
     */
    @GET
    @Path("/sessions")
    List<SessionResponse> getSessions(@QueryParam("local") boolean localOnly);
}
```

- [ ] **Step 3: Add PeerClient config to application.properties**

In `src/main/resources/application.properties`, add:

```properties
# PeerClient — URL overridden programmatically per-peer; base URL required by Quarkus but not used
quarkus.rest-client.peer-client.url=http://localhost:7777
quarkus.rest-client.peer-client.connect-timeout=3000
quarkus.rest-client.peer-client.read-timeout=2000
```

- [ ] **Step 4: Verify everything compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`

Note: `PeerClient` is called programmatically with dynamic URLs via `RestClientBuilder.newBuilder()`, not via `@RestClient` injection. The config above satisfies Quarkus's validation only.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/FleetKeyClientFilter.java \
        src/main/java/dev/claudony/server/fleet/PeerClient.java \
        src/main/resources/application.properties
git commit -m "feat: FleetKeyClientFilter + PeerClient REST client for peer-to-peer session calls"
```

---

## Task 8: PeerResource (REST CRUD)

**Files:**
- Create: `src/main/java/dev/claudony/server/fleet/ManualRegistrationDiscovery.java`
- Create: `src/main/java/dev/claudony/server/fleet/PeerResource.java`
- Create: `src/test/java/dev/claudony/server/fleet/PeerResourceTest.java`

- [ ] **Step 1: Create ManualRegistrationDiscovery**

```java
package dev.claudony.server.fleet;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.UUID;

@ApplicationScoped
public class ManualRegistrationDiscovery implements PeerDiscoverySource {

    @Inject PeerRegistry registry;

    @Override
    public String name() { return "manual"; }

    @Override
    public void discover(PeerRegistry registry) {
        // Peers are loaded by PeerRegistry.loadPersistedPeers() at startup.
        // This source is triggered by REST API calls, not on startup.
    }

    public PeerRecord addPeer(String url, String name, TerminalMode terminalMode) {
        var id = UUID.randomUUID().toString();
        registry.addPeer(id, url, name == null ? url : name,
                DiscoverySource.MANUAL, terminalMode == null ? TerminalMode.DIRECT : terminalMode);
        return registry.findById(id).orElseThrow();
    }
}
```

- [ ] **Step 2: Write failing integration tests**

Create `src/test/java/dev/claudony/server/fleet/PeerResourceTest.java`:

```java
package dev.claudony.server.fleet;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class PeerResourceTest {

    @Inject PeerRegistry registry;

    @AfterEach
    void cleanup() {
        // Remove any manually-added peers between tests
        registry.getAllPeers().stream()
                .filter(p -> p.source() == DiscoverySource.MANUAL)
                .forEach(p -> registry.removePeer(p.id()));
    }

    @Test
    void listPeers_emptyInitially() {
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().get("/api/peers")
            .then().statusCode(200).body("$", hasSize(0));
    }

    @Test
    void addPeer_appearsInList() {
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .contentType(ContentType.JSON)
            .body("{\"url\":\"http://test-peer:7777\",\"name\":\"Test Peer\",\"terminalMode\":\"DIRECT\"}")
            .when().post("/api/peers")
            .then().statusCode(201)
            .body("url", equalTo("http://test-peer:7777"))
            .body("name", equalTo("Test Peer"))
            .body("source", equalTo("MANUAL"));

        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().get("/api/peers")
            .then().statusCode(200).body("$", hasSize(1));
    }

    @Test
    void deletePeer_removesFromList() {
        var id = given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .contentType(ContentType.JSON)
            .body("{\"url\":\"http://delete-peer:7777\",\"name\":\"Delete Me\"}")
            .when().post("/api/peers")
            .then().statusCode(201).extract().path("id");

        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().delete("/api/peers/" + id)
            .then().statusCode(204);

        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().get("/api/peers")
            .then().statusCode(200).body("$", hasSize(0));
    }

    @Test
    void patchPeer_updatesName() {
        var id = given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .contentType(ContentType.JSON)
            .body("{\"url\":\"http://patch-peer:7777\",\"name\":\"Old Name\"}")
            .when().post("/api/peers")
            .then().statusCode(201).extract().path("id");

        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .contentType(ContentType.JSON)
            .body("{\"name\":\"New Name\"}")
            .when().patch("/api/peers/" + id)
            .then().statusCode(200)
            .body("name", equalTo("New Name"));
    }

    @Test
    void deleteUnknownPeer_returns404() {
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().delete("/api/peers/does-not-exist")
            .then().statusCode(404);
    }

    @Test
    void unauthenticated_returns401() {
        given()
            .when().get("/api/peers")
            .then().statusCode(401);
    }

    @Test
    void generateFleetKey_returnsKey() {
        var key = given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().post("/api/peers/generate-fleet-key")
            .then().statusCode(200)
            .extract().asString();

        assertThat(key).isNotBlank();
    }

    // Inline assertThat for this test class
    private static void assertThat(String value) {
        if (value == null || value.isBlank()) throw new AssertionError("Expected non-blank value but got: " + value);
    }
}
```

- [ ] **Step 3: Create AddPeerRequest record**

```java
package dev.claudony.server.fleet;

public record AddPeerRequest(String url, String name, TerminalMode terminalMode) {}
```

Create `src/main/java/dev/claudony/server/fleet/AddPeerRequest.java` with the above content.

- [ ] **Step 4: Create UpdatePeerRequest record**

Create `src/main/java/dev/claudony/server/fleet/UpdatePeerRequest.java`:

```java
package dev.claudony.server.fleet;

public record UpdatePeerRequest(String name, TerminalMode terminalMode) {}
```

- [ ] **Step 5: Create PeerResource**

Create `src/main/java/dev/claudony/server/fleet/PeerResource.java`:

```java
package dev.claudony.server.fleet;

import dev.claudony.server.auth.FleetKeyService;
import io.quarkus.security.Authenticated;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.util.List;

@Path("/api/peers")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Authenticated
public class PeerResource {

    private static final Logger LOG = Logger.getLogger(PeerResource.class);

    @Inject PeerRegistry registry;
    @Inject ManualRegistrationDiscovery manual;
    @Inject FleetKeyService fleetKeyService;

    @GET
    public List<PeerRecord> list() {
        return registry.getAllPeers();
    }

    @POST
    public Response add(AddPeerRequest req) {
        if (req.url() == null || req.url().isBlank()) {
            return Response.status(400).entity("{\"error\":\"url is required\"}").build();
        }
        var created = manual.addPeer(req.url(), req.name(), req.terminalMode());
        return Response.status(201).entity(created).build();
    }

    @DELETE
    @Path("/{id}")
    public Response delete(@PathParam("id") String id) {
        if (registry.findById(id).isEmpty()) {
            return Response.status(404).build();
        }
        var removed = registry.removePeer(id);
        if (!removed) {
            return Response.status(405)
                    .entity("{\"error\":\"Cannot remove a peer registered via static config\"}")
                    .build();
        }
        return Response.noContent().build();
    }

    @PATCH
    @Path("/{id}")
    public Response update(@PathParam("id") String id, UpdatePeerRequest req) {
        if (registry.findById(id).isEmpty()) {
            return Response.status(404).build();
        }
        registry.updatePeer(id, req.name(), req.terminalMode());
        return Response.ok(registry.findById(id)).build();
    }

    @GET
    @Path("/{id}/sessions")
    public Response peerSessions(@PathParam("id") String id) {
        return registry.findById(id)
                .map(peer -> Response.ok(registry.getCachedSessions(id)).build())
                .orElse(Response.status(404).build());
    }

    @POST
    @Path("/{id}/ping")
    public Response ping(@PathParam("id") String id) {
        if (registry.findById(id).isEmpty()) {
            return Response.status(404).build();
        }
        // Health check runs asynchronously — returns immediately
        Thread.ofVirtual().start(() -> {
            var entry = registry.getAllEntries().stream()
                    .filter(e -> e.id.equals(id)).findFirst();
            entry.ifPresent(e -> {
                try {
                    var client = org.eclipse.microprofile.rest.client.RestClientBuilder.newBuilder()
                            .baseUri(java.net.URI.create(e.url))
                            .connectTimeout(5, java.util.concurrent.TimeUnit.SECONDS)
                            .readTimeout(5, java.util.concurrent.TimeUnit.SECONDS)
                            .build(dev.claudony.server.fleet.PeerClient.class);
                    var sessions = client.getSessions(true);
                    registry.recordSuccess(id);
                    registry.updateCachedSessions(id, sessions);
                } catch (Exception ex) {
                    registry.recordFailure(id);
                }
            });
        });
        return Response.accepted().entity("{\"status\":\"ping dispatched\"}").build();
    }

    @POST
    @Path("/generate-fleet-key")
    @Produces(MediaType.TEXT_PLAIN)
    public Response generateFleetKey() {
        try {
            var key = fleetKeyService.generateAndSave();
            LOG.info("Fleet key generated via API — distribute to all fleet members.");
            return Response.ok(key).build();
        } catch (IOException e) {
            return Response.serverError()
                    .entity("{\"error\":\"" + e.getMessage() + "\"}")
                    .build();
        }
    }
}
```

- [ ] **Step 6: Run PeerResourceTest — all 7 tests must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PeerResourceTest 2>&1 | tail -5
```

Expected: `Tests run: 7, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 7: Run full suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

- [ ] **Step 8: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/ \
        src/test/java/dev/claudony/server/fleet/PeerResourceTest.java
git commit -m "feat: PeerResource — REST CRUD for peer management + generate-fleet-key"
```

---

## Task 9: Health Check Loop + Session Federation

**Files:**
- Modify: `src/main/java/dev/claudony/server/fleet/PeerRegistry.java` — add `@Scheduled` health check
- Modify: `src/main/java/dev/claudony/server/SessionResource.java` — federated list()
- Create: `src/test/java/dev/claudony/server/fleet/SessionFederationTest.java`

- [ ] **Step 1: Add health check loop to PeerRegistry**

Add these imports and methods to `PeerRegistry.java`:

```java
// New imports to add:
import io.quarkus.scheduler.Scheduled;
import org.eclipse.microprofile.rest.client.RestClientBuilder;
import java.net.URI;
```

Add `@Inject FleetKeyService fleetKeyService;` field.

Add the health check method (inside the class):

```java
@Scheduled(every = "30s", delayed = "10s")
void healthCheckLoop() {
    for (var entry : getAllEntries()) {
        if (!entry.shouldAttemptHealthCheck()) continue;
        Thread.ofVirtual().start(() -> checkPeerHealth(entry));
    }
}

private void checkPeerHealth(PeerEntry entry) {
    try {
        var client = RestClientBuilder.newBuilder()
                .baseUri(URI.create(entry.url))
                .connectTimeout(5, java.util.concurrent.TimeUnit.SECONDS)
                .readTimeout(5, java.util.concurrent.TimeUnit.SECONDS)
                .build(PeerClient.class);
        var sessions = client.getSessions(true);
        recordSuccess(entry.id);
        updateCachedSessions(entry.id, sessions);
        LOG.debugf("Fleet health check OK: %s (%d sessions)", entry.url, sessions.size());
    } catch (Exception e) {
        recordFailure(entry.id);
        LOG.debugf("Fleet health check FAILED: %s — %s", entry.url, e.getMessage());
    }
}
```

- [ ] **Step 2: Write failing federation test**

Create `src/test/java/dev/claudony/server/fleet/SessionFederationTest.java`:

```java
package dev.claudony.server.fleet;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class SessionFederationTest {

    @Inject PeerRegistry registry;

    @AfterEach
    void cleanup() {
        registry.getAllPeers().stream()
                .filter(p -> p.source() == DiscoverySource.MANUAL)
                .forEach(p -> registry.removePeer(p.id()));
    }

    @Test
    void localOnly_returnsOnlyLocalSessions() {
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .queryParam("local", "true")
            .when().get("/api/sessions")
            .then().statusCode(200);
        // All sessions have null instanceUrl (local sessions)
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .queryParam("local", "true")
            .when().get("/api/sessions")
            .then().statusCode(200)
            .body("instanceUrl", everyItem(nullValue()));
    }

    @Test
    void federated_includesDownPeerCachedSessions() {
        // Add a peer that will be unreachable
        registry.addPeer("test-dead-peer", "http://nonexistent-peer:7777",
                "Dead Peer", DiscoverySource.MANUAL, TerminalMode.DIRECT);

        // Pre-populate its cache with a fake session to simulate stale data
        var fakeSessions = List.of(new dev.claudony.server.model.SessionResponse(
                "fake-id", "claudony-fake", "/tmp", "claude",
                dev.claudony.server.model.SessionStatus.IDLE,
                java.time.Instant.now(), java.time.Instant.now(),
                "ws://nonexistent-peer:7777/ws/fake-id",
                "http://nonexistent-peer:7777/app/session/fake-id",
                null, null, null));
        registry.updateCachedSessions("test-dead-peer", fakeSessions);

        // Open the circuit (simulate 3 failures)
        registry.recordFailure("test-dead-peer");
        registry.recordFailure("test-dead-peer");
        registry.recordFailure("test-dead-peer");

        // Federated call should include stale cached sessions
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .when().get("/api/sessions")
            .then().statusCode(200)
            .body("find { it.id == 'fake-id' }.stale", equalTo(true))
            .body("find { it.id == 'fake-id' }.instanceName", equalTo("Dead Peer"));
    }

    @Test
    void localParam_preventsRecursiveFederation() {
        // Calling /api/sessions?local=true must not fan out to peers
        // Verified by the endpoint returning only local sessions (no instanceUrl)
        given()
            .header("X-Api-Key", "test-api-key-do-not-use-in-prod")
            .queryParam("local", "true")
            .when().get("/api/sessions")
            .then().statusCode(200)
            .body("instanceUrl", everyItem(nullValue()));
    }
}
```

- [ ] **Step 3: Update SessionResource.list() to support federation**

In `src/main/java/dev/claudony/server/SessionResource.java`:

Add these fields:

```java
@Inject PeerRegistry peerRegistry;
```

Add imports:

```java
import dev.claudony.server.fleet.PeerRegistry;
import dev.claudony.server.fleet.PeerClient;
import org.eclipse.microprofile.rest.client.RestClientBuilder;
import java.net.URI;
import java.util.ArrayList;
import java.util.concurrent.TimeUnit;
```

Replace the existing `list()` method:

```java
@GET
public List<SessionResponse> list(@QueryParam("local") @DefaultValue("false") boolean localOnly) {
    // Always collect local sessions first — never blocked
    var result = new ArrayList<>(registry.all().stream()
            .map(s -> SessionResponse.from(s, config.port()))
            .toList());

    if (localOnly) return result;

    // Fan out to healthy peers in parallel (virtual threads, 2s timeout each)
    // getHealthyPeers() returns List<PeerRecord> (public) — accessible from this package
    var healthyPeers = peerRegistry.getHealthyPeers();
    if (healthyPeers.isEmpty()) return result;

    try (var executor = java.util.concurrent.Executors.newVirtualThreadPerTaskExecutor()) {
        var futures = healthyPeers.stream()
                .map(peer -> executor.submit(() -> fetchPeerSessions(peer.url())))
                .toList();

        for (int i = 0; i < futures.size(); i++) {
            var peer = healthyPeers.get(i);
            try {
                var sessions = futures.get(i).get(2, TimeUnit.SECONDS);
                peerRegistry.recordSuccess(peer.id());
                peerRegistry.updateCachedSessions(peer.id(), sessions);
                sessions.stream()
                        .map(s -> s.withInstance(peer.url(), peer.name(), false))
                        .forEach(result::add);
            } catch (Exception e) {
                peerRegistry.recordFailure(peer.id());
                // Return stale cached sessions for this peer
                peerRegistry.getCachedSessions(peer.id()).stream()
                        .map(s -> s.withInstance(peer.url(), peer.name(), true))
                        .forEach(result::add);
            }
        }
    }

    return result;
}

private List<SessionResponse> fetchPeerSessions(String peerUrl) {
    var client = RestClientBuilder.newBuilder()
            .baseUri(URI.create(peerUrl))
            .connectTimeout(3, TimeUnit.SECONDS)
            .readTimeout(2, TimeUnit.SECONDS)
            .build(PeerClient.class);
    return client.getSessions(true); // local=true prevents recursion
}
```

- [ ] **Step 4: Run federation tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SessionFederationTest 2>&1 | tail -10
```

Expected: `Tests run: 3, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 5: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: all tests passing.

- [ ] **Step 6: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/PeerRegistry.java \
        src/main/java/dev/claudony/server/SessionResource.java \
        src/test/java/dev/claudony/server/fleet/SessionFederationTest.java
git commit -m "feat: health check loop + session federation with stale cache fallback"
```

---

## Task 10: MdnsDiscovery

**Files:**
- Create: `src/main/java/dev/claudony/server/fleet/MdnsDiscovery.java`
- Create: `src/test/java/dev/claudony/server/fleet/MdnsDiscoveryTest.java`

- [ ] **Step 1: Write failing tests**

Create `src/test/java/dev/claudony/server/fleet/MdnsDiscoveryTest.java`:

```java
package dev.claudony.server.fleet;

import dev.claudony.config.ClaudonyConfig;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class MdnsDiscoveryTest {

    @TempDir Path tempDir;

    @Test
    void disabledByDefault_doesNothing() {
        var config = mock(ClaudonyConfig.class);
        when(config.mdnsDiscovery()).thenReturn(false);
        when(config.fleetKey()).thenReturn(Optional.empty());
        when(config.name()).thenReturn("Test");
        var registry = new PeerRegistry(tempDir);

        var discovery = new MdnsDiscovery(config, registry);
        discovery.init(); // should not throw even when mDNS unavailable

        assertThat(registry.getAllPeers()).isEmpty();
    }

    @Test
    void failsGracefully_whenMdnsUnavailable() {
        var config = mock(ClaudonyConfig.class);
        when(config.mdnsDiscovery()).thenReturn(true);
        when(config.fleetKey()).thenReturn(Optional.empty());
        when(config.name()).thenReturn("Test");
        var registry = new PeerRegistry(tempDir);

        // Should not throw even if mDNS registration fails
        var discovery = new MdnsDiscovery(config, registry);
        discovery.init();
        // No exception = pass; mDNS failure is non-fatal
    }
}
```

- [ ] **Step 2: Implement MdnsDiscovery**

Create `src/main/java/dev/claudony/server/fleet/MdnsDiscovery.java`:

```java
package dev.claudony.server.fleet;

import dev.claudony.config.ClaudonyConfig;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

/**
 * Advertises this Claudony instance via mDNS and discovers peers on the local network.
 * Disabled by default (claudony.mdns-discovery=false).
 * Fails non-fatally if mDNS is unavailable (VPN, Docker without multicast, etc.).
 *
 * Uses Vert.x service discovery for mDNS — available via quarkus-vertx dependency.
 * Full implementation deferred until Vert.x mDNS API is validated in this Quarkus version.
 * Currently: logs that it would advertise, no-ops if disabled or mDNS fails.
 */
@ApplicationScoped
public class MdnsDiscovery implements PeerDiscoverySource {

    private static final Logger LOG = Logger.getLogger(MdnsDiscovery.class);
    private static final String SERVICE_TYPE = "_claudony._tcp.local.";

    @Inject ClaudonyConfig config;
    @Inject PeerRegistry registry;

    /** Package-private constructor for unit tests. */
    MdnsDiscovery(ClaudonyConfig config, PeerRegistry registry) {
        this.config = config;
        this.registry = registry;
    }

    /** CDI no-arg constructor. */
    MdnsDiscovery() {}

    @PostConstruct
    void init() {
        if (!config.mdnsDiscovery()) {
            LOG.debug("mDNS discovery disabled (claudony.mdns-discovery=false)");
            return;
        }
        try {
            startAdvertising();
            startDiscovering();
        } catch (Exception e) {
            LOG.warnf("mDNS discovery unavailable — %s. " +
                    "This is normal on VPNs, Docker networks without multicast, or some cloud environments. " +
                    "Static config and manual peer registration still work.", e.getMessage());
        }
    }

    @Override
    public String name() { return "mdns"; }

    @Override
    public void discover(PeerRegistry registry) {
        // Discovery is continuous via mDNS listener; initial peers come from startDiscovering()
    }

    private void startAdvertising() {
        // TODO: Implement via Vert.x ServiceDiscovery when mDNS support confirmed for this version.
        // The service type is SERVICE_TYPE with TXT records: name, version.
        // Advertises on port from config.port().
        LOG.infof("mDNS: would advertise as %s on %s", config.name(), SERVICE_TYPE);
    }

    private void startDiscovering() {
        // TODO: Implement via Vert.x ServiceDiscovery mDNS lookup.
        // Each discovered instance: addPeer(uuid, url, name, DiscoverySource.MDNS, TerminalMode.DIRECT)
        LOG.infof("mDNS: would discover peers of type %s", SERVICE_TYPE);
    }
}
```

- [ ] **Step 3: Run tests — both must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MdnsDiscoveryTest 2>&1 | tail -5
```

Expected: `Tests run: 2, Failures: 0, Errors: 0, Skipped: 0`

Note: The mDNS implementation is intentionally stubbed — the service type, advertising, and discovery are logged but not active. The key requirement at this stage is that the class initialises without crashing when mDNS is unavailable, which is what these tests verify. Full mDNS implementation is a follow-on task once the Vert.x mDNS API integration is validated.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/dev/claudony/server/fleet/MdnsDiscovery.java \
        src/test/java/dev/claudony/server/fleet/MdnsDiscoveryTest.java
git commit -m "feat: MdnsDiscovery scaffold — non-fatal startup, disabled by default (full mDNS impl follow-on)"
```

---

## Task 11: Dockerfile + docker-compose.yml

**Files:**
- Create: `Dockerfile`
- Create: `docker-compose.yml`

- [ ] **Step 1: Create Dockerfile**

Create `Dockerfile` at the project root:

```dockerfile
# Claudony — JVM mode Docker image
# Requires: mvn package -DskipTests has been run first
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app
COPY target/quarkus-app/lib/ lib/
COPY target/quarkus-app/*.jar .
COPY target/quarkus-app/app/ app/
COPY target/quarkus-app/quarkus/ quarkus/

# tmux is required by Claudony server mode
RUN apk add --no-cache tmux

EXPOSE 7777

ENTRYPOINT ["java", \
  "-Dclaudony.mode=server", \
  "-Dclaudony.bind=0.0.0.0", \
  "-jar", "quarkus-run.jar"]
```

- [ ] **Step 2: Create docker-compose.yml**

Create `docker-compose.yml` at the project root:

```yaml
# Claudony — two-node fleet example
# Prerequisites:
#   1. Build: mvn package -DskipTests
#   2. Set fleet key: export CLAUDONY_FLEET_KEY=$(openssl rand -base64 32)
#   3. Start: docker compose up

services:
  claudony-a:
    build: .
    ports:
      - "7777:7777"
    volumes:
      - claudony-a-data:/root/.claudony
      - claudony-a-workspace:/root/claudony-workspace
    environment:
      CLAUDONY_PEERS: "http://claudony-b:7777"
      CLAUDONY_FLEET_KEY: "${CLAUDONY_FLEET_KEY}"
      CLAUDONY_SERVER_URL: "http://claudony-a:7777"
      CLAUDONY_NAME: "Node A"
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:7777/q/health/ready"]
      interval: 10s
      timeout: 5s
      retries: 5

  claudony-b:
    build: .
    ports:
      - "7778:7777"
    volumes:
      - claudony-b-data:/root/.claudony
      - claudony-b-workspace:/root/claudony-workspace
    environment:
      CLAUDONY_PEERS: "http://claudony-a:7777"
      CLAUDONY_FLEET_KEY: "${CLAUDONY_FLEET_KEY}"
      CLAUDONY_SERVER_URL: "http://claudony-b:7777"
      CLAUDONY_NAME: "Node B"
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:7777/q/health/ready"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  claudony-a-data:
  claudony-b-data:
  claudony-a-workspace:
  claudony-b-workspace:
```

- [ ] **Step 3: Build and verify Docker image**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests && \
docker build -t claudony:latest . 2>&1 | tail -5
```

Expected: `Successfully built ...` and `Successfully tagged claudony:latest`

- [ ] **Step 4: Commit**

```bash
git add Dockerfile docker-compose.yml
git commit -m "feat: Dockerfile (JVM mode) + docker-compose.yml two-node fleet example"
```

---

## Task 12: CLAUDE.md Update + Final Verification

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update test count in CLAUDE.md**

Find `**162 tests passing**` and update to the new count after running `mvn test`.

Find the `config/` bullet and ensure it's still accurate.

Add a `fleet/` bullet to the test list:

```
- `server/fleet/` — PeerRegistryTest (unit), StaticConfigDiscoveryTest (unit), MdnsDiscoveryTest (unit), PeerResourceTest (QuarkusTest), SessionFederationTest (QuarkusTest)
- `server/auth/` — ... FleetKeyServiceTest (unit), FleetKeyAuthTest (QuarkusTest)
```

Update the project structure in CLAUDE.md to note the new `fleet/` package:

Find the project structure section and add under `server/`:

```
│   ├── fleet/
│   │   ├── PeerRegistry.java           — authoritative peer list, circuit breaker, persistence
│   │   ├── PeerResource.java           — REST /api/peers (CRUD + generate-fleet-key)
│   │   ├── PeerClient.java             — REST client for peer /api/sessions calls
│   │   ├── StaticConfigDiscovery.java  — loads claudony.peers at startup
│   │   ├── ManualRegistrationDiscovery.java — REST-triggered peer management
│   │   └── MdnsDiscovery.java          — mDNS advertise/discover (stubbed, non-fatal)
│   └── auth/
│       ├── FleetKeyService.java        — fleet key load/generate, ~/.claudony/fleet-key
```

- [ ] **Step 2: Run full test suite — final verification**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -5
```

Expected: all tests passing, 0 failures.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md — fleet package, test count, project structure"
```

---

## E2E Verification (Manual — Docker)

After all tasks complete:

```bash
# Set fleet key
export CLAUDONY_FLEET_KEY=$(openssl rand -base64 32)

# Build and start two-node fleet
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests
docker compose up -d

# Wait for health checks to pass
docker compose ps

# Verify Node A sees Node B's sessions via REST
curl -H "X-Api-Key: $(cat ~/.claudony/api-key)" http://localhost:7777/api/peers
curl -H "X-Api-Key: $(cat ~/.claudony/api-key)" http://localhost:7777/api/sessions

# Create a session on Node B
curl -X POST -H "X-Api-Key: $(cat ~/.claudony/api-key)" \
     -H "Content-Type: application/json" \
     -d '{"name":"test-b"}' \
     http://localhost:7778/api/sessions

# Verify Node A's /api/sessions now includes Node B's session
curl -H "X-Api-Key: $(cat ~/.claudony/api-key)" http://localhost:7777/api/sessions \
  | jq '.[] | select(.instanceName == "Node B")'

# Kill Node B — verify Node A degrades gracefully (stale sessions)
docker compose stop claudony-b
curl -H "X-Api-Key: $(cat ~/.claudony/api-key)" http://localhost:7777/api/sessions \
  | jq '.[] | select(.stale == true)'
```
