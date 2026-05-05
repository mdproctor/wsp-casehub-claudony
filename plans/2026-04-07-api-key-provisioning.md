# API Key Provisioning — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement first-run API key generation on the server and auto-discovery on the agent, so neither side requires manual key configuration when co-located.

**Architecture:** New `ApiKeyService` bean centralises all key logic (resolve from config → file → generate). `ApiKeyAuthMechanism` and `ApiKeyClientFilter` inject it instead of reading config directly. `ServerStartup` and `AgentStartup` call `initServer()` / `initAgent()` at startup.

**Tech Stack:** Java 21, Quarkus 3.9.5, JUnit 5, Mockito, `java.nio.file` (PosixFilePermissions)

**Spec:** `docs/superpowers/specs/2026-04-07-api-key-provisioning-design.md`

---

## File Map

| Action | Path | Purpose |
|--------|------|---------|
| **Create** | `src/main/java/dev/remotecc/server/auth/ApiKeyService.java` | All key resolution, generation, persistence, and startup banners |
| **Create** | `src/test/java/dev/remotecc/server/auth/ApiKeyServiceTest.java` | Unit tests for `ApiKeyService` (no Quarkus, uses `@TempDir`) |
| **Modify** | `src/main/java/dev/remotecc/server/auth/ApiKeyAuthMechanism.java` | Inject `ApiKeyService` instead of `RemoteCCConfig` |
| **Modify** | `src/main/java/dev/remotecc/agent/ApiKeyClientFilter.java` | Inject `ApiKeyService` instead of `RemoteCCConfig` |
| **Modify** | `src/main/java/dev/remotecc/server/ServerStartup.java` | Inject `ApiKeyService`, call `apiKeyService.initServer()` after `ensureDirectories()` |
| **Modify** | `src/main/java/dev/remotecc/agent/AgentStartup.java` | Inject `ApiKeyService`, call `apiKeyService.initAgent()` before `checkServerConnectivity()` |

---

## Task 1: Implement `ApiKeyService` with unit tests (TDD)

**Files:**
- Create: `src/test/java/dev/remotecc/server/auth/ApiKeyServiceTest.java`
- Create: `src/main/java/dev/remotecc/server/auth/ApiKeyService.java`

- [ ] **Step 1: Write the failing test file**

  Create `src/test/java/dev/remotecc/server/auth/ApiKeyServiceTest.java`:

  ```java
  package dev.remotecc.server.auth;

  import dev.remotecc.config.RemoteCCConfig;
  import org.junit.jupiter.api.BeforeEach;
  import org.junit.jupiter.api.Test;
  import org.junit.jupiter.api.io.TempDir;
  import org.mockito.Mockito;
  import java.nio.file.Files;
  import java.nio.file.Path;
  import java.nio.file.attribute.PosixFilePermissions;
  import java.util.Optional;
  import static org.junit.jupiter.api.Assertions.*;

  class ApiKeyServiceTest {

      @TempDir
      Path tmp;

      private ApiKeyService service;

      @BeforeEach
      void setUp() {
          var config = Mockito.mock(RemoteCCConfig.class);
          Mockito.when(config.credentialsFile())
                 .thenReturn(tmp.resolve("credentials.json").toString());
          Mockito.when(config.agentApiKey()).thenReturn(Optional.empty());
          service = new ApiKeyService(config);
      }

      @Test
      void serverGeneratesKeyWhenNoneConfigured() {
          service.initServer();
          assertTrue(service.getKey().isPresent());
          assertTrue(service.getKey().get().matches("remotecc-[a-f0-9]{32}"));
      }

      @Test
      void serverPersistsGeneratedKeyToFile() throws Exception {
          service.initServer();
          var keyFile = tmp.resolve("api-key");
          assertTrue(Files.exists(keyFile));
          assertEquals(service.getKey().get(), Files.readString(keyFile).strip());
      }

      @Test
      void serverSetsFilePermissionsTo600() throws Exception {
          service.initServer();
          var keyFile = tmp.resolve("api-key");
          var perms = Files.getPosixFilePermissions(keyFile);
          assertEquals(PosixFilePermissions.fromString("rw-------"), perms);
      }

      @Test
      void serverLoadsExistingKeyFromFileWithoutGenerating() throws Exception {
          var keyFile = tmp.resolve("api-key");
          Files.writeString(keyFile, "remotecc-existingkey");

          service.initServer();

          assertEquals("remotecc-existingkey", service.getKey().get());
          assertEquals("remotecc-existingkey", Files.readString(keyFile).strip());
      }

      @Test
      void configKeyWinsOverFile() throws Exception {
          var configWithKey = Mockito.mock(RemoteCCConfig.class);
          Mockito.when(configWithKey.credentialsFile())
                 .thenReturn(tmp.resolve("credentials.json").toString());
          Mockito.when(configWithKey.agentApiKey()).thenReturn(Optional.of("config-wins"));
          var svc = new ApiKeyService(configWithKey);

          Files.writeString(tmp.resolve("api-key"), "file-loses");

          svc.initServer();

          assertEquals("config-wins", svc.getKey().get());
      }

      @Test
      void agentReturnsEmptyWhenNoKeyFound() {
          service.initAgent();
          assertTrue(service.getKey().isEmpty());
      }

      @Test
      void agentLoadsKeyFromFile() throws Exception {
          Files.writeString(tmp.resolve("api-key"), "remotecc-fromfile");

          service.initAgent();

          assertEquals("remotecc-fromfile", service.getKey().get());
      }
  }
  ```

- [ ] **Step 2: Run tests to verify they fail**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ApiKeyServiceTest
  ```

  Expected: compilation failure — `ApiKeyService` does not exist yet.

- [ ] **Step 3: Create `ApiKeyService`**

  Create `src/main/java/dev/remotecc/server/auth/ApiKeyService.java`:

  ```java
  package dev.remotecc.server.auth;

  import dev.remotecc.config.RemoteCCConfig;
  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.inject.Inject;
  import org.jboss.logging.Logger;
  import java.io.IOException;
  import java.nio.file.Files;
  import java.nio.file.Path;
  import java.nio.file.attribute.PosixFilePermissions;
  import java.util.Optional;
  import java.util.UUID;

  @ApplicationScoped
  public class ApiKeyService {

      private static final Logger LOG = Logger.getLogger(ApiKeyService.class);

      @Inject
      RemoteCCConfig config;

      private Optional<String> resolvedKey = Optional.empty();

      // For direct construction in tests — matches CredentialStore pattern
      ApiKeyService(RemoteCCConfig config) {
          this.config = config;
      }

      // CDI no-arg constructor
      ApiKeyService() {}

      // TODO: Alternative provisioning — env var in launchd plist (REMOTECC_AGENT_API_KEY):
      //       no file needed, suitable for production macOS service deployment.
      // TODO: Alternative provisioning — interactive stdin prompt on first run:
      //       ask user to enter a key or press Enter to generate; useful for headless setup scripts.

      /** Called by ServerStartup after ensureDirectories(). Generates key if absent. */
      public void initServer() {
          resolvedKey = loadFromConfig();
          if (resolvedKey.isPresent()) return;

          resolvedKey = loadFromFile();
          if (resolvedKey.isPresent()) return;

          var key = "remotecc-" + UUID.randomUUID().toString().replace("-", "");
          persistKey(key);
          resolvedKey = Optional.of(key);
          logGenerationBanner(key);
      }

      /** Called by AgentStartup before checkServerConnectivity(). Warns if key absent. */
      public void initAgent() {
          resolvedKey = loadFromConfig();
          if (resolvedKey.isPresent()) return;

          resolvedKey = loadFromFile();
          if (resolvedKey.isPresent()) return;

          logMissingKeyWarning();
      }

      /** Returns the resolved key after init. Empty if agent started without a key. */
      public Optional<String> getKey() {
          return resolvedKey;
      }

      private Optional<String> loadFromConfig() {
          return config.agentApiKey().filter(k -> !k.isBlank());
      }

      private Optional<String> loadFromFile() {
          var keyFile = keyFilePath();
          if (!Files.exists(keyFile)) return Optional.empty();
          try {
              var key = Files.readString(keyFile).strip();
              return key.isBlank() ? Optional.empty() : Optional.of(key);
          } catch (IOException e) {
              LOG.warnf("Could not read API key file %s: %s", keyFile, e.getMessage());
              return Optional.empty();
          }
      }

      private void persistKey(String key) {
          var keyFile = keyFilePath();
          try {
              Files.writeString(keyFile, key);
              Files.setPosixFilePermissions(keyFile, PosixFilePermissions.fromString("rw-------"));
          } catch (IOException e) {
              LOG.errorf("Failed to write API key file %s: %s", keyFile, e.getMessage());
          }
      }

      private Path keyFilePath() {
          return Path.of(config.credentialsFile()).getParent().resolve("api-key");
      }

      private void logGenerationBanner(String key) {
          var keyFile = keyFilePath().toAbsolutePath();
          LOG.warn("\n================================================================\n" +
                  "REMOTECC — API Key Generated (first run)\n" +
                  "  Key:      " + key + "\n" +
                  "  Saved to: " + keyFile + "\n\n" +
                  "  Same machine (agent + server co-located): no action needed.\n" +
                  "  Different machine: configure the agent with —\n" +
                  "    export REMOTECC_AGENT_API_KEY=" + key + "\n" +
                  "  or in agent config:\n" +
                  "    remotecc.agent.api-key=" + key + "\n" +
                  "================================================================");
      }

      private void logMissingKeyWarning() {
          LOG.warn("\n================================================================\n" +
                  "REMOTECC AGENT — No API Key Configured\n" +
                  "  MCP tools will return 401 until a key is set.\n" +
                  "  Copy the key from the server's ~/.remotecc/api-key and set:\n" +
                  "    export REMOTECC_AGENT_API_KEY=<key>\n" +
                  "  or in agent config:\n" +
                  "    remotecc.agent.api-key=<key>\n" +
                  "================================================================");
      }
  }
  ```

- [ ] **Step 4: Run tests to verify they pass**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ApiKeyServiceTest
  ```

  Expected: `BUILD SUCCESS`, 7 tests passing.

- [ ] **Step 5: Commit**

  ```bash
  git add src/main/java/dev/remotecc/server/auth/ApiKeyService.java \
          src/test/java/dev/remotecc/server/auth/ApiKeyServiceTest.java
  git commit -m "feat(auth): add ApiKeyService — key resolution, generation, file persistence"
  ```

---

## Task 2: Wire `ApiKeyService` into auth mechanism and client filter

**Files:**
- Modify: `src/main/java/dev/remotecc/server/auth/ApiKeyAuthMechanism.java`
- Modify: `src/main/java/dev/remotecc/agent/ApiKeyClientFilter.java`

The existing `ApiKeyAuthTest` (a `@QuarkusTest`) serves as the integration test. It already sets `%test.remotecc.agent.api-key=test-api-key-do-not-use-in-prod` in `application.properties`, which `ApiKeyService.loadFromConfig()` will pick up — so no test changes needed.

- [ ] **Step 1: Update `ApiKeyAuthMechanism`**

  In `src/main/java/dev/remotecc/server/auth/ApiKeyAuthMechanism.java`, replace the `@Inject RemoteCCConfig config` field with `@Inject ApiKeyService apiKeyService`, and update the `authenticate()` method accordingly.

  Replace the field:
  ```java
  // Remove:
  @Inject
  RemoteCCConfig config;

  // Add:
  @Inject
  ApiKeyService apiKeyService;
  ```

  Replace the key lookup in `authenticate()` (around line 41):
  ```java
  // Remove:
  var expected = config.agentApiKey();

  // Add:
  var expected = apiKeyService.getKey();
  ```

  Also remove the now-unused `RemoteCCConfig` import:
  ```java
  // Remove:
  import dev.remotecc.config.RemoteCCConfig;
  ```

  The full updated `authenticate()` method:
  ```java
  @Override
  public Uni<SecurityIdentity> authenticate(RoutingContext context, IdentityProviderManager identityProviderManager) {
      var apiKey = context.request().getHeader("X-Api-Key");
      if (apiKey == null && LaunchMode.current() == LaunchMode.DEVELOPMENT) {
          var devCookie = context.getCookie("remotecc-dev-key");
          if (devCookie != null) apiKey = devCookie.getValue();
      }
      if (apiKey == null) {
          return Uni.createFrom().optional(Optional.empty());
      }
      var expected = apiKeyService.getKey();
      if (expected.isEmpty() || !MessageDigest.isEqual(
              expected.get().getBytes(StandardCharsets.UTF_8),
              apiKey.getBytes(StandardCharsets.UTF_8))) {
          return Uni.createFrom().failure(
              new io.quarkus.security.AuthenticationFailedException("Invalid API key"));
      }
      return Uni.createFrom().item(
          QuarkusSecurityIdentity.builder()
              .setPrincipal(new QuarkusPrincipal("agent"))
              .addRole("user")
              .build());
  }
  ```

- [ ] **Step 2: Update `ApiKeyClientFilter`**

  In `src/main/java/dev/remotecc/agent/ApiKeyClientFilter.java`, replace the `@Inject RemoteCCConfig config` field with `@Inject ApiKeyService apiKeyService`, and update `filter()`.

  Replace the field:
  ```java
  // Remove:
  @Inject
  RemoteCCConfig config;

  // Add:
  @Inject
  ApiKeyService apiKeyService;
  ```

  Replace the key lookup in `filter()`:
  ```java
  // Remove:
  config.agentApiKey().ifPresent(key ->

  // Add:
  apiKeyService.getKey().ifPresent(key ->
  ```

  Remove the now-unused `RemoteCCConfig` import. Add the `ApiKeyService` import:
  ```java
  import dev.remotecc.server.auth.ApiKeyService;
  ```

  The full updated class:
  ```java
  package dev.remotecc.agent;

  import dev.remotecc.server.auth.ApiKeyService;
  import jakarta.inject.Inject;
  import jakarta.ws.rs.client.ClientRequestContext;
  import jakarta.ws.rs.client.ClientRequestFilter;
  import jakarta.ws.rs.ext.Provider;
  import java.io.IOException;

  @Provider
  public class ApiKeyClientFilter implements ClientRequestFilter {

      @Inject
      ApiKeyService apiKeyService;

      @Override
      public void filter(ClientRequestContext requestContext) throws IOException {
          apiKeyService.getKey().ifPresent(key ->
              requestContext.getHeaders().add("X-Api-Key", key));
      }
  }
  ```

- [ ] **Step 3: Run auth tests to verify nothing broken**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ApiKeyAuthTest,ApiKeyServiceTest
  ```

  Expected: `BUILD SUCCESS`, all tests passing.

- [ ] **Step 4: Commit**

  ```bash
  git add src/main/java/dev/remotecc/server/auth/ApiKeyAuthMechanism.java \
          src/main/java/dev/remotecc/agent/ApiKeyClientFilter.java
  git commit -m "feat(auth): wire ApiKeyService into auth mechanism and client filter"
  ```

---

## Task 3: Wire startup beans and run full suite

**Files:**
- Modify: `src/main/java/dev/remotecc/server/ServerStartup.java`
- Modify: `src/main/java/dev/remotecc/agent/AgentStartup.java`

- [ ] **Step 1: Update `ServerStartup`**

  In `src/main/java/dev/remotecc/server/ServerStartup.java`, add the `ApiKeyService` injection and call `apiKeyService.initServer()` between `ensureDirectories()` and `bootstrapRegistry()`.

  Add the field (after `@Inject SessionRegistry registry;`):
  ```java
  @Inject ApiKeyService apiKeyService;
  ```

  Update `onStart()`:
  ```java
  void onStart(@Observes StartupEvent event) {
      if (!config.isServerMode()) return;
      checkTmux();
      ensureDirectories();
      apiKeyService.initServer();
      bootstrapRegistry();
      LOG.infof("RemoteCC Server ready — http://%s:%d", config.bind(), config.port());
  }
  ```

  Add the import:
  ```java
  import dev.remotecc.server.auth.ApiKeyService;
  ```

- [ ] **Step 2: Update `AgentStartup`**

  In `src/main/java/dev/remotecc/agent/AgentStartup.java`, add the `ApiKeyService` injection and call `apiKeyService.initAgent()` as the first action in `onStart()`.

  Add the field (after `@Inject TerminalAdapterFactory terminalFactory;`):
  ```java
  @Inject ApiKeyService apiKeyService;
  ```

  Update `onStart()`:
  ```java
  void onStart(@Observes StartupEvent event) {
      if (!config.isAgentMode()) return;

      LOG.infof("RemoteCC Agent starting — proxying to %s", config.serverUrl());
      apiKeyService.initAgent();
      checkServerConnectivity();
      detectTerminalAdapter();
      reportClipboardStatus();
      LOG.infof("RemoteCC Agent ready — MCP endpoint: http://localhost:%d/mcp", config.port());
  }
  ```

  Add the import:
  ```java
  import dev.remotecc.server.auth.ApiKeyService;
  ```

- [ ] **Step 3: Run the full test suite**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
  ```

  Expected: `BUILD SUCCESS`, 116+ tests passing. The `%test.remotecc.agent.api-key` in `application.properties` ensures `ApiKeyService.loadFromConfig()` returns the test key in the `@QuarkusTest` context — no key file is written during tests.

- [ ] **Step 4: Commit**

  ```bash
  git add src/main/java/dev/remotecc/server/ServerStartup.java \
          src/main/java/dev/remotecc/agent/AgentStartup.java
  git commit -m "feat(auth): call ApiKeyService from ServerStartup and AgentStartup"
  ```
