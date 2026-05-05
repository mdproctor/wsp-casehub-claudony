# claudony-casehub Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert Claudony to a 3-module Maven project and implement the four CaseHub Worker Provisioner SPIs in a new optional `claudony-casehub` module backed by TmuxService (provisioner), QhorusMcpTools (channels), CaseLedgerEntryRepository (context), and SessionRegistry (status).

**Architecture:** Root pom becomes an aggregator (`packaging=pom`); core services (TmuxService, SessionRegistry, Session, expiry) move to `claudony-core`; the Quarkus application stays in `claudony-app`; new `claudony-casehub` depends on `claudony-core` + `casehub-engine:api` + `casehub-ledger`. CDI discovery activates the SPI beans when `claudony-casehub` is on the classpath.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven multi-module, JUnit 5, Mockito (`quarkus-junit5-mockito`), `casehub-engine:api:0.2-SNAPSHOT`, `casehub-ledger:0.2-SNAPSHOT`, `quarkus-qhorus:1.0.0-SNAPSHOT`.

**Test commands:**
- All modules: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`

---

## File Map

### Files moving from `src/` → `claudony-core/src/`

| File | Package |
|---|---|
| `config/ClaudonyConfig.java` | `dev.claudony.config` |
| `server/TmuxService.java` | `dev.claudony.server` |
| `server/SessionRegistry.java` | `dev.claudony.server` |
| `server/model/Session.java` | `dev.claudony.server.model` |
| `server/model/SessionStatus.java` | `dev.claudony.server.model` |
| `server/model/SessionExpiredEvent.java` | `dev.claudony.server.model` |
| `server/expiry/ExpiryPolicy.java` | `dev.claudony.server.expiry` |
| `server/expiry/ExpiryPolicyRegistry.java` | `dev.claudony.server.expiry` |
| `server/expiry/SessionIdleScheduler.java` | `dev.claudony.server.expiry` |
| `server/expiry/StatusAwareExpiryPolicy.java` | `dev.claudony.server.expiry` |
| `server/expiry/TerminalOutputExpiryPolicy.java` | `dev.claudony.server.expiry` |
| `server/expiry/UserInteractionExpiryPolicy.java` | `dev.claudony.server.expiry` |

### Files staying in `claudony-app/src/` (unchanged)

Everything not listed above: REST resources, WebSocket, auth, fleet, agent, server/model request/response records.

### New files — `claudony-casehub/`

| File | Responsibility |
|---|---|
| `claudony-casehub/pom.xml` | Module POM |
| `src/main/java/dev/claudony/casehub/CaseHubConfig.java` | Config mapping for casehub properties |
| `src/main/java/dev/claudony/casehub/WorkerCommandResolver.java` | Resolves capability → command from config |
| `src/main/java/dev/claudony/casehub/ClaudonyWorkerProvisioner.java` | WorkerProvisioner SPI implementation |
| `src/main/java/dev/claudony/casehub/ClaudonyCaseChannelProvider.java` | CaseChannelProvider SPI implementation |
| `src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java` | WorkerContextProvider SPI implementation |
| `src/main/java/dev/claudony/casehub/ClaudonyWorkerStatusListener.java` | WorkerStatusListener SPI implementation |
| `src/test/java/dev/claudony/casehub/WorkerCommandResolverTest.java` | Unit tests |
| `src/test/java/dev/claudony/casehub/ClaudonyWorkerProvisionerTest.java` | Unit tests |
| `src/test/java/dev/claudony/casehub/ClaudonyCaseChannelProviderTest.java` | Unit tests |
| `src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java` | Unit tests |
| `src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java` | Unit tests |

### Modified files

| File | Change |
|---|---|
| `pom.xml` | Convert to parent aggregator (`packaging=pom`, add modules) |
| `claudony-core/pom.xml` | New: core library POM |
| `claudony-app/pom.xml` | New: Quarkus app POM (was root `pom.xml`) |
| `src/main/resources/application.properties` → `claudony-app/src/main/resources/application.properties` | Move + add casehub config |
| `CLAUDE.md` | Update build commands, module structure, test count |
| `docs/DESIGN.md` | Add claudony-casehub section |

---

## Task 0: GitHub epic and issue

**Files:** none

- [ ] **Step 1: Create epic**

```bash
cd /Users/mdproctor/claude/claudony
gh issue create \
  --title "Epic: claudony-casehub — CaseHub SPI implementations" \
  --body "Convert Claudony to a 3-module Maven project and implement the four CaseHub Worker Provisioner SPIs (WorkerProvisioner, CaseChannelProvider, WorkerContextProvider, WorkerStatusListener) in a new optional claudony-casehub module." \
  --label "epic"
```

Note the epic number — call it `EPIC_N`.

- [ ] **Step 2: Create implementation issue**

```bash
gh issue create \
  --title "feat: claudony-casehub module — Maven restructuring and CaseHub SPI implementations" \
  --body "Refs #EPIC_N

- Convert single-module Claudony to 3-module Maven project (claudony-core, claudony-app, claudony-casehub)
- Extract TmuxService, SessionRegistry, Session, expiry to claudony-core
- Implement ClaudonyWorkerProvisioner (config-driven capability→command mapping, tmux sessions)
- Implement ClaudonyCaseChannelProvider (Qhorus-backed channels)
- Implement ClaudonyWorkerContextProvider (CaseLedgerEntry lineage + clean-start flag)
- Implement ClaudonyWorkerStatusListener (SessionRegistry lifecycle updates)
- Full TDD: unit, integration, E2E tests" \
  --label "enhancement"
```

Note the issue number — call it `ISSUE_N`.

---

## Task 1: Maven restructuring — 3-module conversion

**Files:**
- Modify: `pom.xml` (root becomes parent aggregator)
- Create: `claudony-core/pom.xml`
- Create: `claudony-app/pom.xml`
- Move: 12 core files via `git mv`
- Move: all `src/` content to `claudony-app/src/`

This task has no test — the baseline test suite passing is the verification.

- [ ] **Step 1: Verify baseline tests pass before restructuring**

```bash
cd /Users/mdproctor/claude/claudony
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `BUILD SUCCESS`. If not, stop and fix before proceeding.

- [ ] **Step 2: Create directory structure**

```bash
cd /Users/mdproctor/claude/claudony
mkdir -p claudony-core/src/main/java/dev/claudony/{config,server/model,server/expiry}
mkdir -p claudony-core/src/test/java/dev/claudony
mkdir -p claudony-app/src/main/java/dev/claudony
mkdir -p claudony-app/src/main/resources
mkdir -p claudony-app/src/test/java/dev/claudony
mkdir -p claudony-app/src/test/resources
```

- [ ] **Step 3: Move core files to claudony-core**

```bash
cd /Users/mdproctor/claude/claudony
git mv src/main/java/dev/claudony/config/ClaudonyConfig.java \
        claudony-core/src/main/java/dev/claudony/config/
git mv src/main/java/dev/claudony/server/TmuxService.java \
        claudony-core/src/main/java/dev/claudony/server/
git mv src/main/java/dev/claudony/server/SessionRegistry.java \
        claudony-core/src/main/java/dev/claudony/server/
git mv src/main/java/dev/claudony/server/model/Session.java \
        claudony-core/src/main/java/dev/claudony/server/model/
git mv src/main/java/dev/claudony/server/model/SessionStatus.java \
        claudony-core/src/main/java/dev/claudony/server/model/
git mv src/main/java/dev/claudony/server/model/SessionExpiredEvent.java \
        claudony-core/src/main/java/dev/claudony/server/model/
git mv src/main/java/dev/claudony/server/expiry/ExpiryPolicy.java \
        claudony-core/src/main/java/dev/claudony/server/expiry/
git mv src/main/java/dev/claudony/server/expiry/ExpiryPolicyRegistry.java \
        claudony-core/src/main/java/dev/claudony/server/expiry/
git mv src/main/java/dev/claudony/server/expiry/SessionIdleScheduler.java \
        claudony-core/src/main/java/dev/claudony/server/expiry/
git mv src/main/java/dev/claudony/server/expiry/StatusAwareExpiryPolicy.java \
        claudony-core/src/main/java/dev/claudony/server/expiry/
git mv src/main/java/dev/claudony/server/expiry/TerminalOutputExpiryPolicy.java \
        claudony-core/src/main/java/dev/claudony/server/expiry/
git mv src/main/java/dev/claudony/server/expiry/UserInteractionExpiryPolicy.java \
        claudony-core/src/main/java/dev/claudony/server/expiry/
```

- [ ] **Step 4: Move remaining app files to claudony-app**

```bash
cd /Users/mdproctor/claude/claudony
# Move all remaining src/ content to claudony-app/
cp -r src/main/java/dev/claudony/. claudony-app/src/main/java/dev/claudony/
cp -r src/main/resources/. claudony-app/src/main/resources/
cp -r src/test/. claudony-app/src/test/
git rm -r src/
git add claudony-app/src/
```

Note: this moves all test files to `claudony-app`. The existing test suite runs from `claudony-app`.

- [ ] **Step 5: Also move test resources if they exist**

```bash
ls /Users/mdproctor/claude/claudony/src/test/resources/ 2>/dev/null && \
  cp -r src/test/resources/. claudony-app/src/test/resources/ || true
```

- [ ] **Step 6: Convert root pom.xml to parent aggregator**

Edit `pom.xml` — make these changes:
1. Add `<packaging>pom</packaging>` after `<version>`
2. Add `<modules>` section after `<packaging>`:

```xml
<modules>
    <module>claudony-core</module>
    <module>claudony-casehub</module>
    <module>claudony-app</module>
</modules>
```

3. Remove all `<dependencies>` and `<build>` sections (these move to child poms)
4. Keep `<dependencyManagement>` with the Quarkus BOM and versions
5. Keep `<properties>` section

The result should be a minimal parent pom managing versions only.

- [ ] **Step 7: Create claudony-core/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.claudony</groupId>
        <artifactId>claudony-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>claudony-core</artifactId>
    <name>Claudony :: Core</name>
    <description>Core services: TmuxService, SessionRegistry, expiry policies</description>

    <dependencies>
        <!-- CDI + Config -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-scheduler</artifactId>
        </dependency>
        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Note: update the parent `<artifactId>` to match whatever you set in the root pom (likely `claudony-parent` or `claudony`).

- [ ] **Step 8: Create claudony-app/pom.xml**

The `claudony-app/pom.xml` is based on the original root `pom.xml` but as a child module. Copy the original `pom.xml` to `claudony-app/pom.xml` then:
1. Add `<parent>` pointing to root parent
2. Change `<artifactId>` to `claudony-app`  
3. Remove all version properties already in parent
4. Add `claudony-core` as a dependency

```xml
<!-- Add inside <dependencies>: -->
<dependency>
    <groupId>dev.claudony</groupId>
    <artifactId>claudony-core</artifactId>
    <version>${project.version}</version>
</dependency>
```

Keep all existing Quarkus dependencies (quarkus-rest, quarkus-websockets-next, etc.). Keep the Quarkus build plugin configuration.

- [ ] **Step 9: Verify the multi-module build compiles**

```bash
cd /Users/mdproctor/claude/claudony
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile 2>&1 | grep -E "BUILD|ERROR" | tail -10
```

Fix any compilation errors (likely missing imports due to the split). Common issue: `TerminalWebSocket.java` observes `SessionExpiredEvent` — both are now in the right places but check imports are still valid.

- [ ] **Step 10: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `BUILD SUCCESS`, same test count as before restructuring. Fix any failures before proceeding.

- [ ] **Step 11: Commit**

```bash
git add pom.xml claudony-core/ claudony-app/
git commit -m "refactor: convert to 3-module Maven project (claudony-core, claudony-app)

Extracts TmuxService, SessionRegistry, Session, expiry policies to
claudony-core. Quarkus application stays in claudony-app. Root pom
becomes parent aggregator. All existing tests pass. Refs #ISSUE_N"
```

---

## Task 2: claudony-casehub skeleton + config

**Files:**
- Create: `claudony-casehub/pom.xml`
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java`
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/WorkerCommandResolver.java`
- Test: `claudony-casehub/src/test/java/dev/claudony/casehub/WorkerCommandResolverTest.java`
- Modify: `claudony-app/src/main/resources/application.properties` (add casehub config)

- [ ] **Step 1: Create claudony-casehub/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.claudony</groupId>
        <artifactId>claudony-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>claudony-casehub</artifactId>
    <name>Claudony :: CaseHub Integration</name>
    <description>Optional CaseHub SPI implementations for Claudony</description>

    <dependencies>
        <!-- Claudony core services -->
        <dependency>
            <groupId>dev.claudony</groupId>
            <artifactId>claudony-core</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- CaseHub SPI interfaces -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>api</artifactId>
            <version>0.2-SNAPSHOT</version>
        </dependency>

        <!-- CaseHub ledger for WorkerContextProvider lineage queries -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger</artifactId>
            <version>0.2-SNAPSHOT</version>
        </dependency>

        <!-- Qhorus tools for CaseChannelProvider -->
        <dependency>
            <groupId>io.quarkiverse.qhorus</groupId>
            <artifactId>quarkus-qhorus</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>

        <!-- CDI -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5-mockito</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.qhorus</groupId>
            <artifactId>quarkus-qhorus-testing</artifactId>
            <version>1.0.0-SNAPSHOT</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Also add claudony-casehub as dependency in claudony-app/pom.xml**

Add inside `claudony-app/pom.xml` `<dependencies>`:

```xml
<!-- CaseHub integration (optional — enabled via claudony.casehub.enabled=true) -->
<dependency>
    <groupId>dev.claudony</groupId>
    <artifactId>claudony-casehub</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 3: Write failing test for WorkerCommandResolver**

Create `claudony-casehub/src/test/java/dev/claudony/casehub/WorkerCommandResolverTest.java`:

```java
package dev.claudony.casehub;

import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class WorkerCommandResolverTest {

    @Test
    void resolve_exactCapabilityMatch_returnsConfiguredCommand() {
        var config = Map.of(
                "code-reviewer", "claude --mcp http://localhost:7778/mcp",
                "researcher", "ollama run llama3",
                "default", "claude");
        var resolver = new WorkerCommandResolver(config);

        assertThat(resolver.resolve(Set.of("code-reviewer")))
                .isEqualTo("claude --mcp http://localhost:7778/mcp");
    }

    @Test
    void resolve_noMatch_returnsDefault() {
        var config = Map.of("default", "claude");
        var resolver = new WorkerCommandResolver(config);

        assertThat(resolver.resolve(Set.of("unknown-capability"))).isEqualTo("claude");
    }

    @Test
    void resolve_multipleCapabilities_firstMatchWins() {
        var config = Map.of(
                "code-reviewer", "claude",
                "researcher", "ollama run llama3",
                "default", "claude");
        var resolver = new WorkerCommandResolver(config);

        String result = resolver.resolve(Set.of("researcher", "code-reviewer"));
        assertThat(result).isIn("claude", "ollama run llama3");
    }

    @Test
    void resolve_emptyCapabilities_returnsDefault() {
        var config = Map.of("default", "claude");
        var resolver = new WorkerCommandResolver(config);

        assertThat(resolver.resolve(Set.of())).isEqualTo("claude");
    }

    @Test
    void getAvailableCapabilities_returnsAllExceptDefault() {
        var config = Map.of(
                "code-reviewer", "claude",
                "researcher", "ollama run llama3",
                "default", "claude");
        var resolver = new WorkerCommandResolver(config);

        Set<String> caps = resolver.getAvailableCapabilities();
        assertThat(caps).containsExactlyInAnyOrder("code-reviewer", "researcher");
        assertThat(caps).doesNotContain("default");
    }

    @Test
    void resolve_noDefaultConfigured_throwsWhenNoMatch() {
        var config = Map.of("code-reviewer", "claude");
        var resolver = new WorkerCommandResolver(config);

        assertThatThrownBy(() -> resolver.resolve(Set.of("unknown")))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("no command configured");
    }
}
```

- [ ] **Step 4: Run — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=WorkerCommandResolverTest 2>&1 | tail -5
```

Expected: compilation error — `WorkerCommandResolver` not found.

- [ ] **Step 5: Create WorkerCommandResolver**

Create `claudony-casehub/src/main/java/dev/claudony/casehub/WorkerCommandResolver.java`:

```java
package dev.claudony.casehub;

import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

/**
 * Resolves a set of required capabilities to a launch command.
 * Checks each requested capability against the config map; falls back to "default".
 */
public class WorkerCommandResolver {

    private final Map<String, String> capabilityToCommand;

    public WorkerCommandResolver(Map<String, String> capabilityToCommand) {
        this.capabilityToCommand = Map.copyOf(capabilityToCommand);
    }

    public String resolve(Set<String> requiredCapabilities) {
        for (String cap : requiredCapabilities) {
            String cmd = capabilityToCommand.get(cap);
            if (cmd != null) return cmd;
        }
        String defaultCmd = capabilityToCommand.get("default");
        if (defaultCmd == null) {
            throw new IllegalStateException(
                    "No command configured for capabilities " + requiredCapabilities +
                    " and no default — add claudony.casehub.workers.default.command");
        }
        return defaultCmd;
    }

    public Set<String> getAvailableCapabilities() {
        return capabilityToCommand.keySet().stream()
                .filter(k -> !k.equals("default"))
                .collect(Collectors.toUnmodifiableSet());
    }
}
```

- [ ] **Step 6: Create CaseHubConfig**

Create `claudony-casehub/src/main/java/dev/claudony/casehub/CaseHubConfig.java`:

```java
package dev.claudony.casehub;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;
import java.util.Map;
import java.util.Optional;

@ConfigMapping(prefix = "claudony.casehub")
public interface CaseHubConfig {

    @WithDefault("false")
    boolean enabled();

    @WithName("workers")
    Workers workers();

    interface Workers {
        /** Map of capability name → launch command. Key "default" is the fallback. */
        Map<String, String> commands();

        @WithName("default-working-dir")
        @WithDefault("${user.home}/claudony-workspace")
        String defaultWorkingDir();
    }
}
```

- [ ] **Step 7: Add casehub config to application.properties**

In `claudony-app/src/main/resources/application.properties`, add:

```properties
# ── CaseHub integration (disabled by default) ────────────────────────────────
claudony.casehub.enabled=false
quarkus.datasource.casehub.db-kind=h2
quarkus.datasource.casehub.jdbc.url=jdbc:h2:file:~/.claudony/casehub;DB_CLOSE_ON_EXIT=FALSE;AUTO_SERVER=TRUE

# Capability-to-command mapping (add entries as needed)
claudony.casehub.workers.commands.default=claude
# claudony.casehub.workers.commands."code-reviewer"=claude --mcp http://localhost:7778/mcp
claudony.casehub.workers.default-working-dir=${claudony.default-working-dir}
```

- [ ] **Step 8: Run WorkerCommandResolverTest — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=WorkerCommandResolverTest 2>&1 | grep -E "Tests run:|BUILD" | tail -3
```

Expected: `BUILD SUCCESS`, 6 tests pass.

- [ ] **Step 9: Run full build to confirm everything still works**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 10: Commit**

```bash
git add claudony-casehub/ claudony-app/pom.xml claudony-app/src/main/resources/application.properties
git commit -m "feat: claudony-casehub module skeleton + WorkerCommandResolver

pom.xml with casehub-engine:api + casehub-ledger + quarkus-qhorus deps.
CaseHubConfig mapping for claudony.casehub.* properties.
WorkerCommandResolver: capability→command lookup with default fallback.
Refs #ISSUE_N"
```

---

## Task 3: ClaudonyWorkerProvisioner (TDD)

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerProvisioner.java`
- Test: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerProvisionerTest.java`

- [ ] **Step 1: Write failing tests**

Create `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerProvisionerTest.java`:

```java
package dev.claudony.casehub;

import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.Worker;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.spi.ProvisioningException;
import dev.claudony.server.SessionRegistry;
import dev.claudony.server.TmuxService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ClaudonyWorkerProvisionerTest {

    private TmuxService tmux;
    private SessionRegistry registry;
    private ClaudonyWorkerContextProvider contextProvider;
    private WorkerCommandResolver resolver;
    private ClaudonyWorkerProvisioner provisioner;

    @BeforeEach
    void setUp() {
        tmux = mock(TmuxService.class);
        registry = mock(SessionRegistry.class);
        contextProvider = mock(ClaudonyWorkerContextProvider.class);
        resolver = new WorkerCommandResolver(Map.of("code-reviewer", "claude", "default", "claude"));
        provisioner = new ClaudonyWorkerProvisioner(true, tmux, registry, resolver, contextProvider, "/tmp/workers");
    }

    // --- Happy path ---

    @Test
    void provision_createsSessionAndRegistersWorker() throws Exception {
        var caseId = UUID.randomUUID();
        var capabilities = Set.of("code-reviewer");
        var ctx = provisionContext(caseId, capabilities);
        when(contextProvider.buildContext(anyString(), any())).thenReturn(
                new WorkerContext("refactor auth", caseId, null, List.of(),
                        PropagationContext.createRoot(), Map.of()));

        Worker worker = provisioner.provision(capabilities, ctx);

        assertThat(worker.getName()).isNotBlank();
        verify(tmux).createSession(contains("claudony-worker-"), eq("/tmp/workers"), eq("claude"));
        verify(registry).register(any());
    }

    @Test
    void provision_returnsWorkerWithRequestedCapabilities() throws Exception {
        var caseId = UUID.randomUUID();
        var capabilities = Set.of("code-reviewer");
        var ctx = provisionContext(caseId, capabilities);
        when(contextProvider.buildContext(anyString(), any())).thenReturn(
                new WorkerContext("task", caseId, null, List.of(), PropagationContext.createRoot(), Map.of()));

        Worker worker = provisioner.provision(capabilities, ctx);

        assertThat(worker.getCapabilities()).extracting(c -> c.name())
                .containsExactlyInAnyOrder("code-reviewer");
    }

    // --- Robustness ---

    @Test
    void provision_disabled_throwsProvisioningException() {
        var disabledProvisioner = new ClaudonyWorkerProvisioner(
                false, tmux, registry, resolver, contextProvider, "/tmp");
        var ctx = provisionContext(UUID.randomUUID(), Set.of("code-reviewer"));

        assertThatThrownBy(() -> disabledProvisioner.provision(Set.of("code-reviewer"), ctx))
                .isInstanceOf(ProvisioningException.class)
                .hasMessageContaining("disabled");
    }

    @Test
    void provision_tmuxFails_throwsProvisioningException() throws Exception {
        var ctx = provisionContext(UUID.randomUUID(), Set.of("code-reviewer"));
        when(contextProvider.buildContext(anyString(), any())).thenReturn(
                new WorkerContext("task", null, null, List.of(), PropagationContext.createRoot(), Map.of()));
        doThrow(new java.io.IOException("tmux not found")).when(tmux)
                .createSession(anyString(), anyString(), anyString());

        assertThatThrownBy(() -> provisioner.provision(Set.of("code-reviewer"), ctx))
                .isInstanceOf(ProvisioningException.class)
                .hasMessageContaining("Failed to create tmux session");
    }

    // --- Correctness ---

    @Test
    void terminate_killsSessionAndRemovesFromRegistry() throws Exception {
        provisioner.terminate("worker-abc");

        verify(tmux).killSession("claudony-worker-worker-abc");
        verify(registry).remove("worker-abc");
    }

    @Test
    void terminate_unknownWorker_isNoOp() throws Exception {
        doThrow(new java.io.IOException("session not found")).when(tmux).killSession(anyString());

        assertThatNoException().isThrownBy(() -> provisioner.terminate("ghost-worker"));
    }

    @Test
    void getCapabilities_returnsConfiguredCapabilities() {
        assertThat(provisioner.getCapabilities()).contains("code-reviewer");
        assertThat(provisioner.getCapabilities()).doesNotContain("default");
    }

    private ProvisionContext provisionContext(UUID caseId, Set<String> capabilities) {
        var wc = new WorkerContext("task", caseId, null, List.of(),
                PropagationContext.createRoot(), Map.of());
        return new ProvisionContext(caseId, "code-reviewer", wc, PropagationContext.createRoot());
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerProvisionerTest 2>&1 | tail -5
```

- [ ] **Step 3: Create ClaudonyWorkerProvisioner**

Create `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerProvisioner.java`:

```java
package dev.claudony.casehub;

import dev.claudony.server.SessionRegistry;
import dev.claudony.server.TmuxService;
import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import io.casehub.api.model.Capability;
import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.Worker;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.spi.ProvisioningException;
import io.casehub.api.spi.WorkerProvisioner;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.io.IOException;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class ClaudonyWorkerProvisioner implements WorkerProvisioner {

    static final String SESSION_PREFIX = "claudony-worker-";

    private final boolean enabled;
    private final TmuxService tmux;
    private final SessionRegistry registry;
    private final WorkerCommandResolver resolver;
    private final ClaudonyWorkerContextProvider contextProvider;
    private final String defaultWorkingDir;

    @Inject
    public ClaudonyWorkerProvisioner(
            CaseHubConfig config,
            TmuxService tmux,
            SessionRegistry registry,
            WorkerCommandResolver resolver,
            ClaudonyWorkerContextProvider contextProvider) {
        this(config.enabled(), tmux, registry, resolver, contextProvider,
                config.workers().defaultWorkingDir());
    }

    // Package-private constructor for unit tests
    ClaudonyWorkerProvisioner(boolean enabled, TmuxService tmux, SessionRegistry registry,
                               WorkerCommandResolver resolver,
                               ClaudonyWorkerContextProvider contextProvider,
                               String defaultWorkingDir) {
        this.enabled = enabled;
        this.tmux = tmux;
        this.registry = registry;
        this.resolver = resolver;
        this.contextProvider = contextProvider;
        this.defaultWorkingDir = defaultWorkingDir;
    }

    @Override
    public Worker provision(Set<String> capabilities, ProvisionContext context) {
        if (!enabled) {
            throw new ProvisioningException(
                    "CaseHub integration is disabled — set claudony.casehub.enabled=true");
        }
        String workerId = UUID.randomUUID().toString();
        String command = resolver.resolve(capabilities);

        // Build worker context for startup prompt
        WorkRequest task = WorkRequest.of(context.taskType(),
                Map.of("caseId", context.caseId() != null ? context.caseId().toString() : ""));
        var workerContext = contextProvider.buildContext(workerId, task);

        // Build startup command with prompt args
        String fullCommand = buildCommandWithPrompt(command, workerContext);

        // Create tmux session
        String sessionName = SESSION_PREFIX + workerId;
        try {
            tmux.createSession(sessionName, defaultWorkingDir, fullCommand);
        } catch (IOException | InterruptedException e) {
            throw new ProvisioningException("Failed to create tmux session for worker " + workerId, e);
        }

        // Register in session registry
        var session = new Session(workerId, sessionName, defaultWorkingDir, fullCommand,
                SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty());
        registry.register(session);

        // Build and return Worker — external worker with placeholder function
        List<Capability> capList = capabilities.stream()
                .map(Capability::new)
                .toList();
        return new Worker(workerId, capList, ctx -> Map.of());
    }

    @Override
    public void terminate(String workerId) {
        try {
            tmux.killSession(SESSION_PREFIX + workerId);
        } catch (IOException | InterruptedException e) {
            // Session may already be gone — no-op
        }
        try {
            registry.remove(workerId);
        } catch (Exception e) {
            // Registry may not have this worker — no-op
        }
    }

    @Override
    public Set<String> getCapabilities() {
        return resolver.getAvailableCapabilities();
    }

    private String buildCommandWithPrompt(String command, io.casehub.api.model.WorkerContext ctx) {
        if (ctx.priorWorkers().isEmpty() && ctx.taskDescription() == null) {
            return command;
        }
        // Build a context prompt that gets injected via environment or startup file
        // For now: append task description as a comment (Claude reads this on startup)
        StringBuilder sb = new StringBuilder(command);
        if (ctx.taskDescription() != null) {
            sb.append(" # Task: ").append(ctx.taskDescription().replace("\"", "\\\""));
        }
        if (ctx.channel() != null) {
            sb.append(" # Channel: ").append(ctx.channel().name());
        }
        return sb.toString();
    }
}
```

Note: Check if `Capability` has a `name()` method or is a record. Adjust the `capList` stream accordingly. Check if `new Worker(workerId, capList, ctx -> Map.of())` compiles — if Worker requires a `WorkerFunctionHolder`, wrap the function.

- [ ] **Step 4: Create WorkerCommandResolver as a CDI bean**

Update `WorkerCommandResolver.java` to be a CDI `@ApplicationScoped` bean with config injection:

```java
package dev.claudony.casehub;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

@ApplicationScoped
public class WorkerCommandResolver {

    private final Map<String, String> capabilityToCommand;

    @Inject
    public WorkerCommandResolver(CaseHubConfig config) {
        this(config.workers().commands());
    }

    // Package-private constructor for unit tests
    WorkerCommandResolver(Map<String, String> capabilityToCommand) {
        this.capabilityToCommand = Map.copyOf(capabilityToCommand);
    }

    public String resolve(Set<String> requiredCapabilities) {
        for (String cap : requiredCapabilities) {
            String cmd = capabilityToCommand.get(cap);
            if (cmd != null) return cmd;
        }
        String defaultCmd = capabilityToCommand.get("default");
        if (defaultCmd == null) {
            throw new IllegalStateException(
                    "No command configured for capabilities " + requiredCapabilities +
                    " and no default — add claudony.casehub.workers.commands.default");
        }
        return defaultCmd;
    }

    public Set<String> getAvailableCapabilities() {
        return capabilityToCommand.keySet().stream()
                .filter(k -> !k.equals("default"))
                .collect(Collectors.toUnmodifiableSet());
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerProvisionerTest 2>&1 | grep -E "Tests run:|BUILD" | tail -3
```

Expected: `BUILD SUCCESS`, 7 tests pass.

- [ ] **Step 6: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerProvisioner.java \
        claudony-casehub/src/main/java/dev/claudony/casehub/WorkerCommandResolver.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerProvisionerTest.java
git commit -m "feat: ClaudonyWorkerProvisioner — tmux-backed CaseHub worker provisioning

Config-driven capability→command mapping via WorkerCommandResolver.
Creates tmux sessions prefixed 'claudony-worker-', registers in
SessionRegistry. Disabled guard, tmux failure wrapping, idempotent
terminate(). Refs #ISSUE_N"
```

---

## Task 4: ClaudonyCaseChannelProvider (TDD)

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyCaseChannelProvider.java`
- Test: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyCaseChannelProviderTest.java`

- [ ] **Step 1: Write failing tests**

Create `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyCaseChannelProviderTest.java`:

```java
package dev.claudony.casehub;

import io.casehub.api.model.CaseChannel;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ClaudonyCaseChannelProviderTest {

    private QhorusMcpTools qhorusMcpTools;
    private ClaudonyCaseChannelProvider provider;

    @BeforeEach
    void setUp() {
        qhorusMcpTools = mock(QhorusMcpTools.class);
        provider = new ClaudonyCaseChannelProvider(qhorusMcpTools);
    }

    // --- Happy path ---

    @Test
    void openChannel_createsQhorusChannelAndReturnsCaseChannel() {
        UUID caseId = UUID.randomUUID();
        // QhorusMcpTools.createChannel returns a ChannelDetail — mock it
        var channelDetail = mock(io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.ChannelDetail.class);
        when(channelDetail.id()).thenReturn(UUID.randomUUID());
        when(channelDetail.name()).thenReturn("case-" + caseId + "/coordination");
        when(qhorusMcpTools.createChannel(anyString(), anyString(), any())).thenReturn(channelDetail);

        CaseChannel ch = provider.openChannel(caseId, "coordination");

        assertThat(ch).isNotNull();
        assertThat(ch.backendType()).isEqualTo("qhorus");
        assertThat(ch.purpose()).isEqualTo("coordination");
        assertThat(ch.properties()).containsKey("qhorus-name");
    }

    @Test
    void openChannel_channelNameContainsCaseId() {
        UUID caseId = UUID.randomUUID();
        var channelDetail = mock(io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.ChannelDetail.class);
        when(channelDetail.id()).thenReturn(UUID.randomUUID());
        when(channelDetail.name()).thenReturn("case-" + caseId + "/coordination");
        when(qhorusMcpTools.createChannel(anyString(), anyString(), any())).thenReturn(channelDetail);

        CaseChannel ch = provider.openChannel(caseId, "coordination");

        assertThat(ch.name()).contains(caseId.toString());
    }

    @Test
    void postToChannel_callsQhorussendMessage() {
        UUID caseId = UUID.randomUUID();
        String channelName = "case-" + caseId + "/coordination";
        CaseChannel ch = new CaseChannel("ch-id", channelName, "coordination", "qhorus",
                Map.of("qhorus-name", channelName));

        provider.postToChannel(ch, "alice", "hello");

        verify(qhorusMcpTools).sendMessage(eq(channelName), eq("alice"), anyString(),
                eq("hello"), isNull(), isNull());
    }

    @Test
    void closeChannel_isNoOp() {
        CaseChannel ch = new CaseChannel("ch-id", "channel", "purpose", "qhorus", Map.of("qhorus-name", "ch"));
        assertThatNoException().isThrownBy(() -> provider.closeChannel(ch));
        verifyNoInteractions(qhorusMcpTools);
    }

    @Test
    void listChannels_returnsChannelsFilteredByCaseId() {
        UUID caseId = UUID.randomUUID();
        var matching = mock(io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.ChannelDetail.class);
        when(matching.name()).thenReturn("case-" + caseId + "/coord");
        when(matching.id()).thenReturn(UUID.randomUUID());
        var other = mock(io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.ChannelDetail.class);
        when(other.name()).thenReturn("case-" + UUID.randomUUID() + "/coord");
        when(other.id()).thenReturn(UUID.randomUUID());

        when(qhorusMcpTools.listChannels()).thenReturn(List.of(matching, other));

        List<CaseChannel> result = provider.listChannels(caseId);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).name()).contains(caseId.toString());
    }

    // --- Robustness ---

    @Test
    void listChannels_noMatchingChannels_returnsEmpty() {
        when(qhorusMcpTools.listChannels()).thenReturn(List.of());
        assertThat(provider.listChannels(UUID.randomUUID())).isEmpty();
    }

    @Test
    void postToChannel_missingQhorusName_usesChannelId() {
        CaseChannel ch = new CaseChannel("ch-id", "channel", "purpose", "qhorus", Map.of());
        assertThatNoException().isThrownBy(() -> provider.postToChannel(ch, "alice", "hello"));
    }
}
```

- [ ] **Step 2: Run — expect compilation failure, then create implementation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyCaseChannelProviderTest 2>&1 | tail -5
```

Create `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyCaseChannelProvider.java`:

```java
package dev.claudony.casehub;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.spi.CaseChannelProvider;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class ClaudonyCaseChannelProvider implements CaseChannelProvider {

    private static final String CHANNEL_PREFIX = "case-";
    private static final String QHORUS_NAME_KEY = "qhorus-name";

    private final QhorusMcpTools qhorusMcpTools;

    @Inject
    public ClaudonyCaseChannelProvider(QhorusMcpTools qhorusMcpTools) {
        this.qhorusMcpTools = qhorusMcpTools;
    }

    @Override
    public CaseChannel openChannel(UUID caseId, String purpose) {
        String channelName = CHANNEL_PREFIX + caseId + "/" + purpose;
        QhorusMcpToolsBase.ChannelDetail detail =
                qhorusMcpTools.createChannel(channelName, purpose, null);
        return new CaseChannel(
                detail.id().toString(),
                detail.name(),
                purpose,
                "qhorus",
                Map.of(QHORUS_NAME_KEY, detail.name()));
    }

    @Override
    public void postToChannel(CaseChannel channel, String from, String content) {
        String qhorusName = (String) channel.properties().getOrDefault(QHORUS_NAME_KEY, channel.id());
        qhorusMcpTools.sendMessage(qhorusName, from, "status", content, null, null);
    }

    @Override
    public void closeChannel(CaseChannel channel) {
        // Qhorus channels are persistent — no close operation
    }

    @Override
    public List<CaseChannel> listChannels(UUID caseId) {
        String prefix = CHANNEL_PREFIX + caseId;
        return qhorusMcpTools.listChannels().stream()
                .filter(ch -> ch.name().startsWith(prefix))
                .map(ch -> new CaseChannel(
                        ch.id().toString(),
                        ch.name(),
                        extractPurpose(ch.name(), caseId),
                        "qhorus",
                        Map.of(QHORUS_NAME_KEY, ch.name())))
                .toList();
    }

    private String extractPurpose(String channelName, UUID caseId) {
        String prefix = CHANNEL_PREFIX + caseId + "/";
        return channelName.startsWith(prefix) ? channelName.substring(prefix.length()) : channelName;
    }
}
```

Note: Verify the exact signature of `qhorusMcpTools.createChannel()` — it may differ from what's shown. Check `QhorusMcpTools.java` for the actual method name and parameters. If `createChannel` doesn't exist, use `createChannel` equivalent or `send_message` to open a new channel.

- [ ] **Step 3: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyCaseChannelProviderTest 2>&1 | grep -E "Tests run:|BUILD" | tail -3
```

- [ ] **Step 4: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyCaseChannelProvider.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyCaseChannelProviderTest.java
git commit -m "feat: ClaudonyCaseChannelProvider — Qhorus-backed CaseHub channels

openChannel creates a Qhorus channel named 'case-{caseId}/{purpose}'.
postToChannel sends via QhorusMcpTools.sendMessage. listChannels filters
by caseId prefix. closeChannel is a no-op (Qhorus channels are persistent).
Refs #ISSUE_N"
```

---

## Task 5: ClaudonyWorkerContextProvider (TDD)

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java`
- Test: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java`

- [ ] **Step 1: Write failing tests**

Create `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java`:

```java
package dev.claudony.casehub;

import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerSummary;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class ClaudonyWorkerContextProviderTest {

    private CaseLedgerEntryRepository ledgerRepo;
    private CaseChannelProvider channelProvider;
    private ClaudonyWorkerContextProvider provider;

    @BeforeEach
    void setUp() {
        ledgerRepo = mock(CaseLedgerEntryRepository.class);
        channelProvider = mock(CaseChannelProvider.class);
        provider = new ClaudonyWorkerContextProvider(ledgerRepo, channelProvider);
    }

    // --- Happy path ---

    @Test
    void buildContext_noPriorWorkers_returnsEmptyPriorWorkers() {
        UUID caseId = UUID.randomUUID();
        when(ledgerRepo.findByCaseId(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = provider.buildContext("worker-1",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        assertThat(ctx.priorWorkers()).isEmpty();
        assertThat(ctx.taskDescription()).isEqualTo("researcher");
    }

    @Test
    void buildContext_withCompletedPriorWorkers_buildsWorkerSummaries() {
        UUID caseId = UUID.randomUUID();
        var entry = completedEntry(caseId, "alice", "AliceRole");
        when(ledgerRepo.findByCaseId(caseId)).thenReturn(List.of(entry));
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = provider.buildContext("worker-2",
                WorkRequest.of("coder", Map.of("caseId", caseId.toString())));

        assertThat(ctx.priorWorkers()).hasSize(1);
        WorkerSummary summary = ctx.priorWorkers().get(0);
        assertThat(summary.workerId()).isEqualTo("alice");
        assertThat(summary.workerName()).isEqualTo("AliceRole");
        assertThat(summary.ledgerEntryId()).isEqualTo(entry.id);
    }

    @Test
    void buildContext_withChannel_includesChannelInContext() {
        UUID caseId = UUID.randomUUID();
        var channel = new io.casehub.api.model.CaseChannel("ch-id", "case-coord", "coordination",
                "qhorus", Map.of());
        when(ledgerRepo.findByCaseId(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of(channel));

        WorkerContext ctx = provider.buildContext("worker-1",
                WorkRequest.of("task", Map.of("caseId", caseId.toString())));

        assertThat(ctx.channel()).isNotNull();
        assertThat(ctx.channel().name()).isEqualTo("case-coord");
    }

    // --- Clean start ---

    @Test
    void buildContext_cleanStart_returnsEmptyPriorWorkers() {
        UUID caseId = UUID.randomUUID();
        var entry = completedEntry(caseId, "alice", "AliceRole");
        when(ledgerRepo.findByCaseId(caseId)).thenReturn(List.of(entry));

        WorkerContext ctx = provider.buildContext("worker-new",
                WorkRequest.of("task", Map.of("caseId", caseId.toString(), "clean-start", true)));

        assertThat(ctx.priorWorkers()).isEmpty();
        verifyNoInteractions(ledgerRepo); // should not even query
    }

    // --- Robustness ---

    @Test
    void buildContext_missingCaseId_returnsEmptyContext() {
        WorkerContext ctx = provider.buildContext("worker-1",
                WorkRequest.of("task", Map.of())); // no caseId

        assertThat(ctx.priorWorkers()).isEmpty();
        assertThat(ctx.caseId()).isNull();
    }

    @Test
    void buildContext_onlyReturnsCompletedWorkerEntries() {
        UUID caseId = UUID.randomUUID();
        var completed = completedEntry(caseId, "alice", "AliceRole");
        var started = startedEntry(caseId, "bob", "BobRole");
        when(ledgerRepo.findByCaseId(caseId)).thenReturn(List.of(completed, started));
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = provider.buildContext("worker-3",
                WorkRequest.of("task", Map.of("caseId", caseId.toString())));

        assertThat(ctx.priorWorkers()).hasSize(1);
        assertThat(ctx.priorWorkers().get(0).workerId()).isEqualTo("alice");
    }

    @Test
    void buildContext_propagationContextIsAlwaysSet() {
        WorkerContext ctx = provider.buildContext("worker-1",
                WorkRequest.of("task", Map.of()));

        assertThat(ctx.propagationContext()).isNotNull();
    }

    private CaseLedgerEntry completedEntry(UUID caseId, String actorId, String actorRole) {
        var e = new CaseLedgerEntry();
        e.id = UUID.randomUUID();
        e.caseId = caseId;
        e.actorId = actorId;
        e.actorRole = actorRole;
        e.eventType = "WorkerExecutionCompleted";
        e.occurredAt = Instant.now().minusSeconds(60);
        return e;
    }

    private CaseLedgerEntry startedEntry(UUID caseId, String actorId, String actorRole) {
        var e = new CaseLedgerEntry();
        e.id = UUID.randomUUID();
        e.caseId = caseId;
        e.actorId = actorId;
        e.actorRole = actorRole;
        e.eventType = "WorkerExecutionStarted";
        e.occurredAt = Instant.now().minusSeconds(120);
        return e;
    }
}
```

Note: `CaseLedgerEntry` fields are public (no getters) — access directly: `entry.id`, `entry.caseId`, `entry.actorId`, `entry.actorRole`, `entry.eventType`, `entry.occurredAt`. The `occurredAt` and `actorId`/`actorRole` fields are inherited from `LedgerEntry` — verify they are accessible on `CaseLedgerEntry`.

- [ ] **Step 2: Run — expect compilation failure, then implement**

Create `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java`:

```java
package dev.claudony.casehub;

import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.CaseChannel;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerSummary;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.api.spi.WorkerContextProvider;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class ClaudonyWorkerContextProvider implements WorkerContextProvider {

    private static final String WORKER_EXECUTION_COMPLETED = "WorkerExecutionCompleted";

    private final CaseLedgerEntryRepository ledgerRepo;
    private final CaseChannelProvider channelProvider;

    @Inject
    public ClaudonyWorkerContextProvider(CaseLedgerEntryRepository ledgerRepo,
                                          CaseChannelProvider channelProvider) {
        this.ledgerRepo = ledgerRepo;
        this.channelProvider = channelProvider;
    }

    @Override
    public WorkerContext buildContext(String workerId, WorkRequest task) {
        // Clean-start: skip all lineage
        if (Boolean.TRUE.equals(task.input().get("clean-start"))) {
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), Map.of("clean-start", true));
        }

        // Extract caseId — graceful degradation if absent
        String caseIdStr = (String) task.input().get("caseId");
        if (caseIdStr == null || caseIdStr.isBlank()) {
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), Map.of());
        }

        UUID caseId;
        try {
            caseId = UUID.fromString(caseIdStr);
        } catch (IllegalArgumentException e) {
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), Map.of());
        }

        // Query ledger for prior COMPLETED worker entries
        List<WorkerSummary> priorWorkers = ledgerRepo.findByCaseId(caseId).stream()
                .filter(e -> WORKER_EXECUTION_COMPLETED.equals(e.eventType))
                .map(this::toWorkerSummary)
                .toList();

        // Find first available channel for this case
        CaseChannel channel = channelProvider.listChannels(caseId).stream()
                .findFirst()
                .orElse(null);

        return new WorkerContext(task.capability(), caseId, channel, priorWorkers,
                PropagationContext.createRoot(), Map.of());
    }

    private WorkerSummary toWorkerSummary(CaseLedgerEntry entry) {
        return new WorkerSummary(
                entry.actorId,
                entry.actorRole,
                null,                // startedAt: not in COMPLETED entry; STARTED entry needed
                entry.occurredAt,    // completedAt
                entry.caseStatus,    // outputSummary: use caseStatus as a proxy for now
                entry.id);           // ledgerEntryId for causal chain
    }
}
```

- [ ] **Step 3: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerContextProviderTest 2>&1 | grep -E "Tests run:|BUILD" | tail -3
```

- [ ] **Step 4: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java
git commit -m "feat: ClaudonyWorkerContextProvider — CaseLedgerEntry lineage

Queries CaseLedgerEntryRepository for COMPLETED worker entries per case.
Builds WorkerSummary list with ledgerEntryId for causal chain linking.
clean-start flag bypasses all lineage. Graceful degradation when caseId
is absent. Refs #ISSUE_N"
```

---

## Task 6: ClaudonyWorkerStatusListener (TDD)

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerStatusListener.java`
- Test: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java`

- [ ] **Step 1: Write failing tests**

Create `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java`:

```java
package dev.claudony.casehub;

import dev.claudony.server.SessionRegistry;
import dev.claudony.server.TmuxService;
import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import io.casehub.api.model.WorkResult;
import io.casehub.api.model.WorkStatus;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ClaudonyWorkerStatusListenerTest {

    private SessionRegistry registry;
    private TmuxService tmux;
    private ClaudonyWorkerStatusListener listener;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        registry = mock(SessionRegistry.class);
        tmux = mock(TmuxService.class);
        Event<Object> events = mock(Event.class);
        listener = new ClaudonyWorkerStatusListener(registry, tmux, events);
    }

    // --- Happy path ---

    @Test
    void onWorkerStarted_updatesSessionToActive() {
        String workerId = "worker-123";
        var session = session(workerId, SessionStatus.IDLE);
        when(registry.find(workerId)).thenReturn(Optional.of(session));

        listener.onWorkerStarted(workerId, Map.of("session", "tmux-123"));

        verify(registry).updateStatus(workerId, SessionStatus.ACTIVE);
    }

    @Test
    void onWorkerCompleted_updatesSessionToIdle() {
        String workerId = "worker-456";
        var session = session(workerId, SessionStatus.ACTIVE);
        when(registry.find(workerId)).thenReturn(Optional.of(session));
        var result = WorkResult.completed("corr-1", Map.of(), workerId);

        listener.onWorkerCompleted(workerId, result);

        verify(registry).updateStatus(workerId, SessionStatus.IDLE);
    }

    // --- Correctness ---

    @Test
    void onWorkerCompleted_faultedResult_terminatesSession() throws Exception {
        String workerId = "worker-faulted";
        var session = session(workerId, SessionStatus.ACTIVE);
        when(registry.find(workerId)).thenReturn(Optional.of(session));
        var result = WorkResult.faulted("corr-1", workerId);

        listener.onWorkerCompleted(workerId, result);

        verify(tmux).killSession("claudony-worker-" + workerId);
        verify(registry).remove(workerId);
    }

    @Test
    void onWorkerStalled_doesNotTerminateSession() throws Exception {
        String workerId = "worker-stalled";
        when(registry.find(workerId)).thenReturn(Optional.empty());

        listener.onWorkerStalled(workerId);

        verifyNoInteractions(tmux);
    }

    // --- Robustness ---

    @Test
    void onWorkerStarted_sessionNotFound_isNoOp() {
        when(registry.find("unknown")).thenReturn(Optional.empty());
        assertThatNoException().isThrownBy(() ->
                listener.onWorkerStarted("unknown", Map.of()));
        verify(registry, never()).updateStatus(any(), any());
    }

    @Test
    void onWorkerCompleted_terminateFails_stillRemovesFromRegistry() throws Exception {
        String workerId = "worker-x";
        when(registry.find(workerId)).thenReturn(Optional.of(session(workerId, SessionStatus.ACTIVE)));
        doThrow(new java.io.IOException("tmux gone")).when(tmux).killSession(anyString());
        var result = WorkResult.faulted("corr-1", workerId);

        assertThatNoException().isThrownBy(() -> listener.onWorkerCompleted(workerId, result));
        verify(registry).remove(workerId);
    }

    private Session session(String id, SessionStatus status) {
        return new Session(id, "claudony-worker-" + id, "/tmp", "claude",
                status, Instant.now(), Instant.now(), Optional.empty());
    }
}
```

- [ ] **Step 2: Create ClaudonyWorkerStatusListener**

Create `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerStatusListener.java`:

```java
package dev.claudony.casehub;

import dev.claudony.server.SessionRegistry;
import dev.claudony.server.TmuxService;
import dev.claudony.server.model.SessionStatus;
import io.casehub.api.model.WorkResult;
import io.casehub.api.model.WorkStatus;
import io.casehub.api.spi.WorkerStatusListener;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import java.io.IOException;
import java.util.Map;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ClaudonyWorkerStatusListener implements WorkerStatusListener {

    private static final Logger LOG = Logger.getLogger(ClaudonyWorkerStatusListener.class);
    static final String SESSION_PREFIX = "claudony-worker-";

    private final SessionRegistry registry;
    private final TmuxService tmux;
    private final Event<Object> events;

    @Inject
    public ClaudonyWorkerStatusListener(SessionRegistry registry, TmuxService tmux,
                                         Event<Object> events) {
        this.registry = registry;
        this.tmux = tmux;
        this.events = events;
    }

    @Override
    public void onWorkerStarted(String workerId, Map<String, String> sessionMeta) {
        registry.find(workerId).ifPresent(session ->
                registry.updateStatus(workerId, SessionStatus.ACTIVE));
        LOG.infof("Worker started: %s meta=%s", workerId, sessionMeta);
    }

    @Override
    public void onWorkerCompleted(String workerId, WorkResult result) {
        if (result.status() == WorkStatus.FAULTED) {
            // Terminate faulted sessions and remove from registry
            try {
                tmux.killSession(SESSION_PREFIX + workerId);
            } catch (IOException | InterruptedException e) {
                LOG.warnf("Could not kill faulted session for worker %s: %s", workerId, e.getMessage());
            }
            registry.remove(workerId);
        } else {
            registry.find(workerId).ifPresent(session ->
                    registry.updateStatus(workerId, SessionStatus.IDLE));
        }
        LOG.infof("Worker completed: %s status=%s", workerId, result.status());
    }

    @Override
    public void onWorkerStalled(String workerId) {
        LOG.warnf("Worker stalled: %s — firing stall event for dashboard notification", workerId);
        events.fire(new WorkerStalledEvent(workerId));
    }

    /** CDI event fired when a worker stalls — observed by dashboard/alert infrastructure. */
    public record WorkerStalledEvent(String workerId) {}
}
```

- [ ] **Step 3: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerStatusListenerTest 2>&1 | grep -E "Tests run:|BUILD" | tail -3
```

- [ ] **Step 4: Run all claudony-casehub tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  2>&1 | grep -E "Tests run:|BUILD" | tail -3
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 5: Commit and close issue**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerStatusListener.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java
git commit -m "feat: ClaudonyWorkerStatusListener — lifecycle callbacks to SessionRegistry

onWorkerStarted: ACTIVE. onWorkerCompleted: IDLE (or terminate+remove if faulted).
onWorkerStalled: fires WorkerStalledEvent CDI event for dashboard notification.
Robust: session-not-found is no-op, terminate-fail still removes from registry.
Closes #ISSUE_N"
```

---

## Task 7: Full build verification + documentation

**Files:**
- Modify: `CLAUDE.md` (update build commands, module structure, test count)
- Modify: `docs/DESIGN.md` (add claudony-casehub section)

- [ ] **Step 1: Run full test suite**

```bash
cd /Users/mdproctor/claude/claudony
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `BUILD SUCCESS`. Note the total test count for CLAUDE.md.

- [ ] **Step 2: Update CLAUDE.md**

Find the "Build and Test" section and update:
- Test command now runs all 3 modules: `JAVA_HOME=... mvn test` (still works from root)
- Add: `JAVA_HOME=... mvn test -pl claudony-casehub` — run only CaseHub integration tests
- Native build now targets the app module: `JAVA_HOME=... mvn package -Pnative -DskipTests -pl claudony-app`
- Update project structure to show 3 modules

Find "Test Count and Status" and update the test count.

Find or add a section documenting:
- `claudony-casehub` is activated by `claudony.casehub.enabled=true`
- Capability-to-command mapping: `claudony.casehub.workers.commands.<capability>=<command>`
- `MeshResourceInterjectionTest` cleanup pattern (still `InMemoryChannelStore.clear()` etc. — unchanged)

- [ ] **Step 3: Update docs/DESIGN.md**

Add a new section "CaseHub Integration (`claudony-casehub`)" covering:
- Module purpose and activation
- Four SPI implementations and what each does
- `quarkus.datasource.casehub.*` named datasource
- Worker provisioning flow: CaseEngine → ClaudonyWorkerProvisioner → tmux → SessionRegistry
- Channel flow: ClaudonyCaseChannelProvider → QhorusMcpTools → Qhorus mesh
- Lineage: ClaudonyWorkerContextProvider → CaseLedgerEntryRepository → WorkerSummary

- [ ] **Step 4: Push all changes**

```bash
git add CLAUDE.md docs/DESIGN.md
git commit -m "docs: update CLAUDE.md and DESIGN.md for claudony-casehub

3-module Maven structure, claudony-casehub activation guide,
capability-to-command config, CaseHub integration architecture.
Refs #EPIC_N"
git push
```

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| 3-module Maven restructuring | Task 1 |
| claudony-casehub pom + config | Task 2 |
| WorkerCommandResolver | Task 2 |
| CaseHubConfig mapping | Task 2 |
| ClaudonyWorkerProvisioner (tmux, registry, disabled guard) | Task 3 |
| ClaudonyCaseChannelProvider (Qhorus-backed) | Task 4 |
| ClaudonyWorkerContextProvider (ledger lineage + clean-start) | Task 5 |
| ClaudonyWorkerStatusListener (registry updates, stall event) | Task 6 |
| casehub config in application.properties | Task 2 |
| CLAUDE.md updated | Task 7 |
| DESIGN.md updated | Task 7 |
| All commits linked to issues/epics | All tasks ✓ |

**Type consistency:**
- `ClaudonyWorkerProvisioner.SESSION_PREFIX = "claudony-worker-"` used in Task 3 and referenced in Task 6 (`ClaudonyWorkerStatusListener.SESSION_PREFIX`) — same constant, consistent. ✓
- `WorkerContext`, `WorkerSummary`, `CaseChannel` from `casehub-engine:api` — used consistently across Tasks 3, 4, 5. ✓
- `WorkRequest.of(capability, input)` — consistent across Tasks 3, 5. ✓
- `CaseLedgerEntry.eventType == "WorkerExecutionCompleted"` — must verify against actual casehub-ledger string value during implementation.
