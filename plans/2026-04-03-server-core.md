# RemoteCC Server Core — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the RemoteCC Server — a Quarkus native binary that manages tmux sessions, exposes a REST API for session CRUD, and streams terminal I/O over WebSocket.

**Architecture:** tmux is the session source of truth. Quarkus wraps tmux via ProcessBuilder (native-image safe). Each WebSocket connection spawns `tmux attach-session` and pipes its stdin/stdout to the browser. Session metadata is kept in an in-memory registry, bootstrapped from `tmux list-sessions` on startup.

**Tech Stack:** Java 21, Quarkus 3.x (native), quarkus-rest, quarkus-rest-jackson, quarkus-websockets-next, JUnit 5, RestAssured

**Design doc:** `docs/superpowers/specs/2026-04-03-remotecc-design.md`

---

### Task 1: Project Bootstrap

**Files:**
- Create: `pom.xml`
- Create: `src/main/resources/application.properties`
- Create: `src/main/java/dev/remotecc/config/RemoteCCConfig.java`
- Create: `src/test/java/dev/remotecc/SmokeTest.java`

- [ ] **Step 1: Create pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>dev.remotecc</groupId>
  <artifactId>remotecc</artifactId>
  <version>1.0.0-SNAPSHOT</version>

  <properties>
    <compiler-plugin.version>3.13.0</compiler-plugin.version>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <quarkus.platform.version>3.9.0</quarkus.platform.version>
    <surefire-plugin.version>3.2.5</surefire-plugin.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-bom</artifactId>
        <version>${quarkus.platform.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-websockets-next</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.platform.version}</version>
        <extensions>true</extensions>
        <executions>
          <execution>
            <goals>
              <goal>build</goal>
              <goal>generate-code</goal>
              <goal>generate-code-tests</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>${compiler-plugin.version}</version>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${surefire-plugin.version}</version>
        <configuration>
          <systemPropertyVariables>
            <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
          </systemPropertyVariables>
        </configuration>
      </plugin>
    </plugins>
  </build>

  <profiles>
    <profile>
      <id>native</id>
      <activation>
        <property>
          <name>native</name>
        </property>
      </activation>
      <properties>
        <skipITs>false</skipITs>
        <quarkus.native.enabled>true</quarkus.native.enabled>
      </properties>
    </profile>
  </profiles>
</project>
```

- [ ] **Step 2: Create application.properties**

```properties
# Mode: server | agent
remotecc.mode=server
remotecc.port=7777
remotecc.bind=localhost
remotecc.server.url=http://localhost:7777
remotecc.claude-command=claude
remotecc.tmux-prefix=remotecc-
remotecc.terminal=auto

quarkus.http.port=${remotecc.port}
quarkus.http.host=${remotecc.bind}

# Logging
quarkus.log.level=INFO
quarkus.log.category."dev.remotecc".level=DEBUG
```

- [ ] **Step 3: Create RemoteCCConfig.java**

```java
package dev.remotecc.config;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;

@ConfigMapping(prefix = "remotecc")
public interface RemoteCCConfig {

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

    @WithDefault("remotecc-")
    String tmuxPrefix();

    @WithDefault("auto")
    String terminal();

    default boolean isServerMode() { return "server".equals(mode()); }
    default boolean isAgentMode()  { return "agent".equals(mode()); }
}
```

- [ ] **Step 4: Write smoke test**

```java
package dev.remotecc;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.is;

@QuarkusTest
class SmokeTest {

    @Test
    void healthEndpointResponds() {
        given()
            .when().get("/q/health")
            .then()
            .statusCode(200);
    }
}
```

- [ ] **Step 5: Run smoke test — expect FAIL (nothing compiles yet)**

```bash
./mvnw test -pl . -Dtest=SmokeTest
```

Expected: BUILD FAILURE — no source to compile yet. Confirms test harness is wired.

- [ ] **Step 6: Run in dev mode to verify project structure**

```bash
./mvnw quarkus:dev
```

Expected: Quarkus starts, `/q/health` returns 200. Ctrl+C to stop.

- [ ] **Step 7: Run smoke test — expect PASS**

```bash
./mvnw test -Dtest=SmokeTest
```

Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git init
git add pom.xml src/
git commit -m "feat: bootstrap Quarkus project with config and smoke test"
```

---

### Task 2: Session Model

**Files:**
- Create: `src/main/java/dev/remotecc/server/model/Session.java`
- Create: `src/main/java/dev/remotecc/server/model/SessionStatus.java`
- Create: `src/test/java/dev/remotecc/server/model/SessionTest.java`

- [ ] **Step 1: Write failing test**

```java
package dev.remotecc.server.model;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class SessionTest {

    @Test
    void sessionHasRequiredFields() {
        var now = Instant.now();
        var session = new Session("id-1", "myproject", "/home/user/proj",
                "claude", SessionStatus.IDLE, now, now);

        assertEquals("id-1", session.id());
        assertEquals("myproject", session.name());
        assertEquals("/home/user/proj", session.workingDir());
        assertEquals("claude", session.command());
        assertEquals(SessionStatus.IDLE, session.status());
    }

    @Test
    void withStatusReturnsCopyWithUpdatedStatus() {
        var now = Instant.now();
        var session = new Session("id-1", "myproject", "/home/user/proj",
                "claude", SessionStatus.IDLE, now, now);

        var updated = session.withStatus(SessionStatus.ACTIVE);

        assertEquals(SessionStatus.ACTIVE, updated.status());
        assertEquals("id-1", updated.id()); // unchanged
    }

    @Test
    void sessionStatusValues() {
        assertNotNull(SessionStatus.valueOf("ACTIVE"));
        assertNotNull(SessionStatus.valueOf("WAITING"));
        assertNotNull(SessionStatus.valueOf("IDLE"));
    }
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./mvnw test -Dtest=SessionTest
```

Expected: FAIL — `Session` and `SessionStatus` not found.

- [ ] **Step 3: Create SessionStatus.java**

```java
package dev.remotecc.server.model;

public enum SessionStatus {
    ACTIVE,   // Claude is actively responding
    WAITING,  // Claude has shown a prompt, waiting for user input
    IDLE      // Shell prompt visible, no Claude running
}
```

- [ ] **Step 4: Create Session.java**

```java
package dev.remotecc.server.model;

import java.time.Instant;

public record Session(
        String id,
        String name,
        String workingDir,
        String command,
        SessionStatus status,
        Instant createdAt,
        Instant lastActive) {

    public Session withStatus(SessionStatus newStatus) {
        return new Session(id, name, workingDir, command, newStatus, createdAt, Instant.now());
    }

    public Session withLastActive() {
        return new Session(id, name, workingDir, command, status, createdAt, Instant.now());
    }
}
```

- [ ] **Step 5: Run — expect PASS**

```bash
./mvnw test -Dtest=SessionTest
```

Expected: BUILD SUCCESS, 3 tests passed.

- [ ] **Step 6: Commit**

```bash
git add src/
git commit -m "feat: add Session record and SessionStatus enum"
```

---

### Task 3: TmuxService

**Files:**
- Create: `src/main/java/dev/remotecc/server/TmuxService.java`
- Create: `src/test/java/dev/remotecc/server/TmuxServiceTest.java`

Note: TmuxService tests require tmux to be installed. They spawn real tmux sessions using the `test-` prefix to avoid clashing with real sessions. Tests clean up after themselves.

- [ ] **Step 1: Write failing tests**

```java
package dev.remotecc.server;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.*;
import jakarta.inject.Inject;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class TmuxServiceTest {

    @Inject
    TmuxService tmux;

    private static final String TEST_SESSION = "test-remotecc-unit";

    @AfterEach
    void cleanup() throws Exception {
        // Always try to kill the test session
        if (tmux.sessionExists(TEST_SESSION)) {
            tmux.killSession(TEST_SESSION);
        }
    }

    @Test
    @Order(1)
    void tmuxVersionReturnsNonEmpty() throws Exception {
        var version = tmux.tmuxVersion();
        assertFalse(version.isBlank());
        assertTrue(version.startsWith("tmux"), "Expected 'tmux X.Y', got: " + version);
    }

    @Test
    @Order(2)
    void sessionDoesNotExistBeforeCreation() throws Exception {
        assertFalse(tmux.sessionExists(TEST_SESSION));
    }

    @Test
    @Order(3)
    void createAndKillSession() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo hello");
        assertTrue(tmux.sessionExists(TEST_SESSION));
        tmux.killSession(TEST_SESSION);
        assertFalse(tmux.sessionExists(TEST_SESSION));
    }

    @Test
    @Order(4)
    void listSessionNamesIncludesCreatedSession() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo hello");
        var names = tmux.listSessionNames();
        assertTrue(names.contains(TEST_SESSION),
                "Expected session list to contain: " + TEST_SESSION + ", got: " + names);
    }

    @Test
    @Order(5)
    void capturePaneReturnsOutput() throws Exception {
        tmux.createSession(TEST_SESSION, System.getProperty("user.home"), "echo remotecc-marker");
        Thread.sleep(300); // let echo run
        var output = tmux.capturePane(TEST_SESSION, 20);
        assertTrue(output.contains("remotecc-marker"),
                "Expected pane output to contain 'remotecc-marker', got: " + output);
    }
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./mvnw test -Dtest=TmuxServiceTest
```

Expected: FAIL — `TmuxService` not found.

- [ ] **Step 3: Create TmuxService.java**

```java
package dev.remotecc.server;

import jakarta.enterprise.context.ApplicationScoped;
import java.io.*;
import java.util.List;
import java.util.stream.Collectors;

@ApplicationScoped
public class TmuxService {

    public List<String> listSessionNames() throws IOException, InterruptedException {
        var pb = new ProcessBuilder("tmux", "list-sessions", "-F", "#{session_name}");
        pb.redirectErrorStream(true);
        var process = pb.start();
        int exit = process.waitFor();
        if (exit != 0) return List.of(); // no sessions running
        try (var reader = new BufferedReader(new InputStreamReader(process.getInputStream()))) {
            return reader.lines()
                    .filter(l -> !l.isBlank())
                    .collect(Collectors.toList());
        }
    }

    public void createSession(String name, String workingDir, String command)
            throws IOException, InterruptedException {
        new ProcessBuilder("tmux", "new-session", "-d", "-s", name, "-c", workingDir)
                .start().waitFor();
        new ProcessBuilder("tmux", "send-keys", "-t", name, command, "Enter")
                .start().waitFor();
    }

    public void killSession(String name) throws IOException, InterruptedException {
        new ProcessBuilder("tmux", "kill-session", "-t", name)
                .start().waitFor();
    }

    public void sendKeys(String sessionName, String text)
            throws IOException, InterruptedException {
        new ProcessBuilder("tmux", "send-keys", "-t", sessionName, text, "")
                .start().waitFor();
    }

    public String capturePane(String sessionName, int lines)
            throws IOException, InterruptedException {
        var pb = new ProcessBuilder(
                "tmux", "capture-pane", "-t", sessionName, "-p", "-S", String.valueOf(-lines));
        var p = pb.start();
        p.waitFor();
        return new String(p.getInputStream().readAllBytes());
    }

    public Process attachSession(String sessionName) throws IOException {
        var pb = new ProcessBuilder("tmux", "attach-session", "-t", sessionName);
        pb.redirectErrorStream(true);
        return pb.start();
    }

    public boolean sessionExists(String name) throws IOException, InterruptedException {
        return new ProcessBuilder("tmux", "has-session", "-t", name)
                .start().waitFor() == 0;
    }

    public String tmuxVersion() throws IOException, InterruptedException {
        var pb = new ProcessBuilder("tmux", "-V");
        var p = pb.start();
        p.waitFor();
        return new String(p.getInputStream().readAllBytes()).trim();
    }
}
```

- [ ] **Step 4: Run — expect PASS**

```bash
./mvnw test -Dtest=TmuxServiceTest
```

Expected: BUILD SUCCESS, 5 tests passed. Requires tmux installed (`brew install tmux`).

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: add TmuxService wrapping tmux via ProcessBuilder"
```

---

### Task 4: SessionRegistry

**Files:**
- Create: `src/main/java/dev/remotecc/server/SessionRegistry.java`
- Create: `src/test/java/dev/remotecc/server/SessionRegistryTest.java`

- [ ] **Step 1: Write failing tests**

```java
package dev.remotecc.server;

import dev.remotecc.server.model.Session;
import dev.remotecc.server.model.SessionStatus;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.*;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class SessionRegistryTest {

    @Inject
    SessionRegistry registry;

    @BeforeEach
    void clearRegistry() {
        registry.all().forEach(s -> registry.remove(s.id()));
    }

    @Test
    void emptyRegistryHasNoSessions() {
        assertTrue(registry.all().isEmpty());
    }

    @Test
    void registerAndFindSession() {
        var now = Instant.now();
        var session = new Session("id-1", "proj", "/tmp", "claude", SessionStatus.IDLE, now, now);
        registry.register(session);

        var found = registry.find("id-1");
        assertTrue(found.isPresent());
        assertEquals("proj", found.get().name());
    }

    @Test
    void removeSession() {
        var now = Instant.now();
        registry.register(new Session("id-2", "proj2", "/tmp", "claude", SessionStatus.IDLE, now, now));
        registry.remove("id-2");
        assertTrue(registry.find("id-2").isEmpty());
    }

    @Test
    void allReturnsAllSessions() {
        var now = Instant.now();
        registry.register(new Session("id-3", "a", "/tmp", "claude", SessionStatus.IDLE, now, now));
        registry.register(new Session("id-4", "b", "/tmp", "claude", SessionStatus.IDLE, now, now));
        assertEquals(2, registry.all().size());
    }

    @Test
    void updateSessionStatus() {
        var now = Instant.now();
        registry.register(new Session("id-5", "proj", "/tmp", "claude", SessionStatus.IDLE, now, now));
        registry.updateStatus("id-5", SessionStatus.ACTIVE);

        var updated = registry.find("id-5");
        assertTrue(updated.isPresent());
        assertEquals(SessionStatus.ACTIVE, updated.get().status());
    }
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./mvnw test -Dtest=SessionRegistryTest
```

Expected: FAIL — `SessionRegistry` not found.

- [ ] **Step 3: Create SessionRegistry.java**

```java
package dev.remotecc.server;

import dev.remotecc.server.model.Session;
import dev.remotecc.server.model.SessionStatus;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class SessionRegistry {

    private final Map<String, Session> sessions = new ConcurrentHashMap<>();

    public void register(Session session) {
        sessions.put(session.id(), session);
    }

    public Optional<Session> find(String id) {
        return Optional.ofNullable(sessions.get(id));
    }

    public Collection<Session> all() {
        return Collections.unmodifiableCollection(sessions.values());
    }

    public void remove(String id) {
        sessions.remove(id);
    }

    public void updateStatus(String id, SessionStatus status) {
        sessions.computeIfPresent(id, (k, s) -> s.withStatus(status));
    }
}
```

- [ ] **Step 4: Run — expect PASS**

```bash
./mvnw test -Dtest=SessionRegistryTest
```

Expected: BUILD SUCCESS, 5 tests passed.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: add in-memory SessionRegistry with thread-safe ConcurrentHashMap"
```

---

### Task 5: Server Startup — Bootstrap Registry from tmux

**Files:**
- Create: `src/main/java/dev/remotecc/server/ServerStartup.java`
- Create: `src/test/java/dev/remotecc/server/ServerStartupTest.java`

- [ ] **Step 1: Write failing test**

```java
package dev.remotecc.server;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class ServerStartupTest {

    @Inject
    SessionRegistry registry;

    @Inject
    TmuxService tmux;

    @Test
    void registryIsBootstrappedFromTmuxOnStartup() throws Exception {
        // Sessions already running in tmux (even if zero) should be reflected
        var tmuxNames = tmux.listSessionNames();
        var registryIds = registry.all().stream()
                .map(s -> s.name())
                .toList();
        // Every tmux session with our prefix should be in the registry
        tmuxNames.stream()
                .filter(n -> n.startsWith("remotecc-"))
                .forEach(name ->
                    assertTrue(registryIds.contains(name),
                        "Expected registry to contain tmux session: " + name));
    }
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./mvnw test -Dtest=ServerStartupTest
```

Expected: FAIL — bootstrap not implemented, registry is empty.

- [ ] **Step 3: Create ServerStartup.java**

```java
package dev.remotecc.server;

import dev.remotecc.config.RemoteCCConfig;
import dev.remotecc.server.model.Session;
import dev.remotecc.server.model.SessionStatus;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.time.Instant;
import java.util.UUID;

@ApplicationScoped
public class ServerStartup {

    private static final Logger LOG = Logger.getLogger(ServerStartup.class);

    @Inject RemoteCCConfig config;
    @Inject TmuxService tmux;
    @Inject SessionRegistry registry;

    void onStart(@Observes StartupEvent event) {
        if (!config.isServerMode()) return;

        checkTmux();
        bootstrapRegistry();

        LOG.infof("RemoteCC Server ready — http://%s:%d", config.bind(), config.port());
        LOG.infof("MCP endpoint — configure Agent to point to http://%s:%d/api",
                config.bind(), config.port());
    }

    private void checkTmux() {
        try {
            var version = tmux.tmuxVersion();
            LOG.infof("tmux found: %s", version);
        } catch (Exception e) {
            throw new IllegalStateException(
                "tmux not found on PATH. Install with: brew install tmux", e);
        }
    }

    private void bootstrapRegistry() {
        try {
            var names = tmux.listSessionNames();
            var prefix = config.tmuxPrefix();
            int count = 0;
            for (var name : names) {
                if (!name.startsWith(prefix)) continue;
                var now = Instant.now();
                var session = new Session(
                        UUID.randomUUID().toString(),
                        name,
                        "unknown", // workingDir not recoverable from tmux alone
                        config.claudeCommand(),
                        SessionStatus.IDLE,
                        now, now);
                registry.register(session);
                count++;
            }
            LOG.infof("Bootstrapped %d existing session(s) from tmux", count);
        } catch (Exception e) {
            LOG.warn("Could not bootstrap from tmux list-sessions: " + e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run — expect PASS**

```bash
./mvnw test -Dtest=ServerStartupTest
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: bootstrap SessionRegistry from running tmux sessions on startup"
```

---

### Task 6: Session REST API

**Files:**
- Create: `src/main/java/dev/remotecc/server/SessionResource.java`
- Create: `src/main/java/dev/remotecc/server/model/CreateSessionRequest.java`
- Create: `src/main/java/dev/remotecc/server/model/SessionResponse.java`
- Create: `src/test/java/dev/remotecc/server/SessionResourceTest.java`

- [ ] **Step 1: Write failing tests**

```java
package dev.remotecc.server;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.*;
import jakarta.inject.Inject;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class SessionResourceTest {

    @Inject SessionRegistry registry;
    @Inject TmuxService tmux;

    @AfterEach
    void cleanup() throws Exception {
        registry.all().forEach(s -> {
            registry.remove(s.id());
            try { tmux.killSession(s.name()); } catch (Exception ignored) {}
        });
    }

    @Test
    @Order(1)
    void listSessionsReturnsEmptyArray() {
        given()
            .when().get("/api/sessions")
            .then()
            .statusCode(200)
            .body("$", hasSize(0));
    }

    @Test
    @Order(2)
    void createSessionReturns201WithSessionId() {
        given()
            .contentType("application/json")
            .body("""
                {
                  "name": "test-remotecc-api",
                  "workingDir": "/tmp",
                  "command": "echo hello"
                }
                """)
            .when().post("/api/sessions")
            .then()
            .statusCode(201)
            .body("id", notNullValue())
            .body("name", equalTo("test-remotecc-api"))
            .body("status", equalTo("IDLE"));
    }

    @Test
    @Order(3)
    void getSessionById() {
        var id = given()
            .contentType("application/json")
            .body("""
                {"name":"test-remotecc-get","workingDir":"/tmp","command":"echo hi"}
                """)
            .when().post("/api/sessions")
            .then().statusCode(201)
            .extract().path("id");

        given()
            .when().get("/api/sessions/" + id)
            .then()
            .statusCode(200)
            .body("id", equalTo(id));
    }

    @Test
    @Order(4)
    void deleteSessionReturns204() {
        var id = given()
            .contentType("application/json")
            .body("""
                {"name":"test-remotecc-del","workingDir":"/tmp","command":"echo bye"}
                """)
            .when().post("/api/sessions")
            .then().statusCode(201)
            .extract().path("id");

        given()
            .when().delete("/api/sessions/" + id)
            .then().statusCode(204);

        given()
            .when().get("/api/sessions/" + id)
            .then().statusCode(404);
    }

    @Test
    @Order(5)
    void getUnknownSessionReturns404() {
        given()
            .when().get("/api/sessions/does-not-exist")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./mvnw test -Dtest=SessionResourceTest
```

Expected: FAIL — `SessionResource` not found.

- [ ] **Step 3: Create CreateSessionRequest.java**

```java
package dev.remotecc.server.model;

public record CreateSessionRequest(
        String name,
        String workingDir,
        String command) {

    public String effectiveCommand(String defaultCommand) {
        return (command != null && !command.isBlank()) ? command : defaultCommand;
    }
}
```

- [ ] **Step 4: Create SessionResponse.java**

```java
package dev.remotecc.server.model;

import java.time.Instant;

public record SessionResponse(
        String id,
        String name,
        String workingDir,
        String command,
        SessionStatus status,
        Instant createdAt,
        Instant lastActive,
        String wsUrl,
        String browserUrl) {

    public static SessionResponse from(Session session, int port) {
        return new SessionResponse(
                session.id(),
                session.name(),
                session.workingDir(),
                session.command(),
                session.status(),
                session.createdAt(),
                session.lastActive(),
                "ws://localhost:" + port + "/ws/" + session.id(),
                "http://localhost:" + port + "/app/session/" + session.id());
    }
}
```

- [ ] **Step 5: Create SessionResource.java**

```java
package dev.remotecc.server;

import dev.remotecc.config.RemoteCCConfig;
import dev.remotecc.server.model.*;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import org.jboss.logging.Logger;
import java.time.Instant;
import java.util.UUID;

@Path("/api/sessions")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class SessionResource {

    private static final Logger LOG = Logger.getLogger(SessionResource.class);

    @Inject RemoteCCConfig config;
    @Inject SessionRegistry registry;
    @Inject TmuxService tmux;

    @GET
    public java.util.List<SessionResponse> list() {
        return registry.all().stream()
                .map(s -> SessionResponse.from(s, config.port()))
                .toList();
    }

    @GET
    @Path("/{id}")
    public Response get(@PathParam("id") String id) {
        return registry.find(id)
                .map(s -> Response.ok(SessionResponse.from(s, config.port())).build())
                .orElse(Response.status(404).build());
    }

    @POST
    public Response create(CreateSessionRequest req) {
        var id = UUID.randomUUID().toString();
        var name = config.tmuxPrefix() + req.name();
        var command = req.effectiveCommand(config.claudeCommand());
        var now = Instant.now();
        var session = new Session(id, name, req.workingDir(), command,
                SessionStatus.IDLE, now, now);
        try {
            tmux.createSession(name, req.workingDir(), command);
            registry.register(session);
            LOG.infof("Created session '%s' (id=%s)", name, id);
            return Response.status(201)
                    .entity(SessionResponse.from(session, config.port()))
                    .build();
        } catch (Exception e) {
            LOG.errorf("Failed to create session '%s': %s", name, e.getMessage());
            return Response.serverError()
                    .entity("{\"error\":\"" + e.getMessage() + "\"}")
                    .build();
        }
    }

    @DELETE
    @Path("/{id}")
    public Response delete(@PathParam("id") String id) {
        return registry.find(id).map(session -> {
            try {
                tmux.killSession(session.name());
                registry.remove(id);
                LOG.infof("Deleted session '%s' (id=%s)", session.name(), id);
                return Response.noContent().build();
            } catch (Exception e) {
                LOG.errorf("Failed to delete session '%s': %s", session.name(), e.getMessage());
                return Response.serverError().build();
            }
        }).orElse(Response.status(404).build());
    }

    @PATCH
    @Path("/{id}/rename")
    public Response rename(@PathParam("id") String id, @QueryParam("name") String newName) {
        return registry.find(id).map(session -> {
            try {
                var newTmuxName = config.tmuxPrefix() + newName;
                new ProcessBuilder("tmux", "rename-session", "-t", session.name(), newTmuxName)
                        .start().waitFor();
                var renamed = new Session(id, newTmuxName, session.workingDir(),
                        session.command(), session.status(), session.createdAt(), Instant.now());
                registry.register(renamed);
                return Response.ok(SessionResponse.from(renamed, config.port())).build();
            } catch (Exception e) {
                return Response.serverError().build();
            }
        }).orElse(Response.status(404).build());
    }
}
```

- [ ] **Step 6: Run — expect PASS**

```bash
./mvnw test -Dtest=SessionResourceTest
```

Expected: BUILD SUCCESS, 5 tests passed.

- [ ] **Step 7: Commit**

```bash
git add src/
git commit -m "feat: add Session REST API — list, get, create, delete, rename"
```

---

### Task 7: WebSocket Terminal Streaming

**Files:**
- Create: `src/main/java/dev/remotecc/server/TerminalWebSocket.java`
- Create: `src/test/java/dev/remotecc/server/TerminalWebSocketTest.java`

Note: WebSocket tests use Quarkus's built-in WebSocket test client. The terminal streaming uses Java 21 virtual threads to bridge the tmux process stdout → WebSocket without blocking.

- [ ] **Step 1: Write failing test**

```java
package dev.remotecc.server;

import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.websocket.*;
import org.junit.jupiter.api.*;
import java.net.URI;
import java.util.concurrent.*;
import dev.remotecc.server.model.*;
import java.time.Instant;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class TerminalWebSocketTest {

    @TestHTTPResource("/ws/")
    URI wsBaseUri;

    @Inject SessionRegistry registry;
    @Inject TmuxService tmux;

    private static final String TEST_SESSION_NAME = "test-remotecc-ws";

    @BeforeEach
    void setup() throws Exception {
        tmux.createSession(TEST_SESSION_NAME, System.getProperty("user.home"), "echo ws-test-marker");
        var now = Instant.now();
        var session = new Session(
            "ws-test-id", TEST_SESSION_NAME, System.getProperty("user.home"),
            "echo ws-test-marker", SessionStatus.IDLE, now, now);
        registry.register(session);
        Thread.sleep(200); // let echo run
    }

    @AfterEach
    void cleanup() throws Exception {
        registry.remove("ws-test-id");
        if (tmux.sessionExists(TEST_SESSION_NAME)) {
            tmux.killSession(TEST_SESSION_NAME);
        }
    }

    @Test
    void connectToSessionReceivesOutput() throws Exception {
        var received = new LinkedBlockingQueue<String>();

        var container = ContainerProvider.getWebSocketContainer();
        var session = container.connectToServer(new Endpoint() {
            @Override
            public void onOpen(Session s, EndpointConfig c) {
                s.addMessageHandler(String.class, received::offer);
            }
        }, URI.create(wsBaseUri + "ws-test-id"));

        var message = received.poll(3, TimeUnit.SECONDS);
        assertNotNull(message, "Expected to receive terminal output within 3s");
        session.close();
    }

    @Test
    void connectToUnknownSessionCloses() throws Exception {
        var closed = new CountDownLatch(1);
        var container = ContainerProvider.getWebSocketContainer();
        var session = container.connectToServer(new Endpoint() {
            @Override
            public void onOpen(Session s, EndpointConfig c) {}
            @Override
            public void onClose(Session s, CloseReason r) { closed.countDown(); }
        }, URI.create(wsBaseUri + "unknown-id"));

        assertTrue(closed.await(2, TimeUnit.SECONDS),
            "Expected WebSocket to close for unknown session");
    }
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./mvnw test -Dtest=TerminalWebSocketTest
```

Expected: FAIL — `TerminalWebSocket` not found.

- [ ] **Step 3: Create TerminalWebSocket.java**

```java
package dev.remotecc.server;

import io.quarkus.websockets.next.*;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.io.*;
import java.util.concurrent.*;

@WebSocket(path = "/ws/{id}")
public class TerminalWebSocket {

    private static final Logger LOG = Logger.getLogger(TerminalWebSocket.class);

    @Inject SessionRegistry registry;

    // Maps connection ID → attached tmux process
    private final ConcurrentHashMap<String, Process> processes = new ConcurrentHashMap<>();

    @OnOpen
    public void onOpen(WebSocketConnection connection) {
        var sessionId = connection.pathParam("id");
        var session = registry.find(sessionId);

        if (session.isEmpty()) {
            LOG.warnf("WebSocket open for unknown session id=%s — closing", sessionId);
            connection.closeAndAwait();
            return;
        }

        var tmuxName = session.get().name();
        LOG.debugf("WebSocket open for session '%s' (id=%s)", tmuxName, sessionId);

        try {
            var process = new ProcessBuilder("tmux", "attach-session", "-t", tmuxName)
                    .redirectErrorStream(true)
                    .start();
            processes.put(connection.id(), process);

            // Virtual thread: pipe process stdout → WebSocket
            Thread.ofVirtual().start(() -> {
                try (var reader = new BufferedInputStream(process.getInputStream())) {
                    var buf = new byte[4096];
                    int n;
                    while ((n = reader.read(buf)) != -1) {
                        connection.sendTextAndAwait(new String(buf, 0, n));
                    }
                } catch (IOException e) {
                    LOG.debugf("Stdout pipe closed for session %s: %s", sessionId, e.getMessage());
                }
            });

        } catch (IOException e) {
            LOG.errorf("Failed to attach to tmux session '%s': %s", tmuxName, e.getMessage());
            connection.closeAndAwait();
        }
    }

    @OnTextMessage
    public void onMessage(WebSocketConnection connection, String message) {
        var process = processes.get(connection.id());
        if (process == null || !process.isAlive()) return;
        try {
            process.getOutputStream().write(message.getBytes());
            process.getOutputStream().flush();
        } catch (IOException e) {
            LOG.debugf("Failed to write to process stdin: %s", e.getMessage());
        }
    }

    @OnClose
    public void onClose(WebSocketConnection connection) {
        var process = processes.remove(connection.id());
        if (process != null && process.isAlive()) {
            process.destroy(); // detach from tmux — session keeps running
        }
        LOG.debugf("WebSocket closed for connection %s", connection.id());
    }

    @OnError
    public void onError(WebSocketConnection connection, Throwable error) {
        LOG.warnf("WebSocket error for connection %s: %s", connection.id(), error.getMessage());
        onClose(connection);
    }
}
```

- [ ] **Step 4: Run — expect PASS**

```bash
./mvnw test -Dtest=TerminalWebSocketTest
```

Expected: BUILD SUCCESS, 2 tests passed.

- [ ] **Step 5: Run all tests**

```bash
./mvnw test
```

Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/
git commit -m "feat: add WebSocket terminal streaming via ProcessBuilder tmux attach"
```

---

### Task 8: Native Image Configuration

**Files:**
- Create: `src/main/resources/META-INF/native-image/reflect-config.json`
- Modify: `pom.xml` — verify native profile present (already added in Task 1)

- [ ] **Step 1: Verify JVM build is clean**

```bash
./mvnw package -DskipTests
ls target/quarkus-app/
```

Expected: `quarkus-run.jar` present. Run with: `java -jar target/quarkus-app/quarkus-run.jar`

- [ ] **Step 2: Create reflect-config.json**

ProcessBuilder and standard Java I/O work natively without reflection config. This file handles any Quarkus internals that need explicit registration:

```json
[
  {
    "name": "dev.remotecc.server.model.Session",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.remotecc.server.model.SessionStatus",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.remotecc.server.model.CreateSessionRequest",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.remotecc.server.model.SessionResponse",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  }
]
```

- [ ] **Step 3: Build native image**

Requires GraalVM with native-image installed (`sdk install java 21.0.x-graal` via SDKMAN, or `brew install --cask graalvm-jdk`).

```bash
./mvnw package -Pnative -DskipTests
```

Expected: `target/remotecc-1.0.0-SNAPSHOT-runner` binary produced. Takes 2-5 minutes.

- [ ] **Step 4: Smoke test the native binary**

```bash
./target/remotecc-1.0.0-SNAPSHOT-runner &
sleep 1
curl http://localhost:7777/api/sessions
curl http://localhost:7777/q/health
kill %1
```

Expected: `[]` for sessions, `{"status":"UP"}` for health. Startup should be under 100ms.

- [ ] **Step 5: Run tests against native binary** (optional CI step)

```bash
./mvnw verify -Pnative
```

Expected: All tests pass against the native binary.

- [ ] **Step 6: Commit**

```bash
git add src/main/resources/META-INF/
git commit -m "feat: add native image reflection config for model classes"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| tmux as session source of truth | Task 3, 4 |
| Session create/list/delete/rename REST API | Task 6 |
| WebSocket terminal streaming | Task 7 |
| Bootstrap registry from `tmux list-sessions` on startup | Task 5 |
| ProcessBuilder over pty4j (native-image safe) | Task 3, 7 |
| Quarkus native image | Task 8 |
| Server mode config (`remotecc.mode=server`) | Task 1 |
| tmux version check on startup | Task 5 |
| Session status enum (ACTIVE/WAITING/IDLE) | Task 2 |
| `wsUrl` and `browserUrl` in session response | Task 6 |
| Unknown session → 404 / WebSocket close | Task 6, 7 |

**Not in this plan (covered in later plans):**
- Agent MCP server → Plan 2
- Terminal adapters (iTerm2) → Plan 2
- Clipboard detection → Plan 2
- Web dashboard + xterm.js → Plan 3
- iPad key bar → Plan 3
- PWA manifest → Plan 3
- Authentication → Plan 4 (future)

**Placeholder scan:** None found. All steps have concrete code and commands.

**Type consistency:** `Session`, `SessionStatus`, `SessionResponse`, `CreateSessionRequest` — all defined in Task 2-6 and used consistently in Tasks 6-7. `SessionRegistry.find()` returns `Optional<Session>` and is used with `.map()` in Task 6 consistently.
