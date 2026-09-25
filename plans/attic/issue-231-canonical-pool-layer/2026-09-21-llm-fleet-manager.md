# LLM Fleet Manager Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #205 — feat: LLM Fleet Manager — provision, scale, and route pools of LLM instances
**Issue group:** #205

**Goal:** Add pool lifecycle management to the platform's existing model resolution stack, register Claudony CLI sessions as a routable backend, and converge the engine's inline ChatModel construction onto the platform routing stack.

**Architecture:** Extends `BackendInstanceRegistry` (platform) with a `PoolManager` SPI for capacity-bounded acquire/release semantics. Claudony provides `FleetPoolManager` (full lifecycle) and `ClaudonyAgentBackend` (CLI sessions as a platform backend). Engine's `AgentConverter` converges from inline LangChain4j construction to `RoutingAgentProvider`. Pool configuration extends the manifest YAML with a `pools:` section.

**Tech Stack:** Java 21, Quarkus 3.32.2, platform-agent-api, agent-config-core, casehub-engine-api, tmux

## Global Constraints

- Java 21 API surface (compiled on Java 26)
- Quarkus 3.32.2 across all repos
- All new SPIs use `@DefaultBean` for no-op defaults
- Pool config uses existing manifest YAML parsing (Jackson)
- CLI sessions use `TmuxService.createWorkerSession()` (protocol PP-20260605-4b6c4e)
- Reactive Panache calls in provision() must be pre-constructed on the event loop (protocol PP-20260616-d32bc3)
- Multi-repo: platform (`casehub-platform`), engine (`casehub-engine`), claudony repos in slot

---

## Batch 1: Platform — PoolManager SPI + PoolConfig

The foundation. After this batch, the platform has pool abstractions with a no-op default that preserves current behaviour. No existing tests break.

### Task 1: PoolManager SPI, PoolConfig, PoolStatus, and DefaultPoolManager

**Files:**
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/PoolManager.java`
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/PoolConfig.java`
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/PoolStatus.java`
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/PoolHealth.java`
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/PooledBackendInstance.java`
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/PoolExhaustedException.java`
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/InstanceMode.java`
- Create: `platform/agent-api/src/main/java/io/casehub/platform/agent/pool/DefaultPoolManager.java`
- Test: `platform/agent-api/src/test/java/io/casehub/platform/agent/pool/DefaultPoolManagerTest.java`

**Interfaces:**
- Produces: `PoolManager.acquire(String backendKey, String instanceId) → PooledBackendInstance`
- Produces: `PoolManager.release(PooledBackendInstance instance)`
- Produces: `PoolManager.status(String backendKey) → PoolStatus`
- Produces: `PoolManager.allStatus() → Map<String, PoolStatus>`
- Produces: `PoolConfig(backendKey, instanceId, min, max, healthCheckInterval, idleTimeout, acquireTimeout)`
- Produces: `PooledBackendInstance(delegate, poolKey, acquiredAt, mode)` — implements `AutoCloseable`
- Produces: `PoolExhaustedException` — thrown on acquire timeout

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.platform.agent.pool;

import io.casehub.platform.agent.AgentBackend;
import io.casehub.platform.agent.BackendInstanceRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;

class DefaultPoolManagerTest {

    private BackendInstanceRegistry registry;
    private DefaultPoolManager poolManager;

    @BeforeEach
    void setUp() {
        // Stub registry with a test backend
        var testBackend = new StubAgentBackend("test-backend", "default");
        registry = new StubBackendInstanceRegistry(testBackend);
        poolManager = new DefaultPoolManager(registry);
    }

    @Test
    void acquire_delegatesToRegistry() {
        var pooled = poolManager.acquire("test-backend", "default");
        assertThat(pooled).isNotNull();
        assertThat(pooled.delegate().key()).isEqualTo("test-backend");
        assertThat(pooled.poolKey()).isEqualTo("test-backend");
        assertThat(pooled.acquiredAt()).isNotNull();
    }

    @Test
    void release_isNoOp() {
        var pooled = poolManager.acquire("test-backend", "default");
        assertThatCode(() -> poolManager.release(pooled)).doesNotThrowAnyException();
    }

    @Test
    void acquire_unknownBackend_throwsIllegalArgument() {
        assertThatThrownBy(() -> poolManager.acquire("nonexistent", "default"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void status_returnsUnpooledStatus() {
        var status = poolManager.status("test-backend");
        assertThat(status.backendKey()).isEqualTo("test-backend");
        assertThat(status.min()).isZero();
        assertThat(status.max()).isZero();
        assertThat(status.health()).isEqualTo(PoolHealth.HEALTHY);
    }

    @Test
    void allStatus_returnsEmpty() {
        assertThat(poolManager.allStatus()).isEmpty();
    }

    @Test
    void pooledInstance_autoCloseable_releasesOnClose() {
        var pooled = poolManager.acquire("test-backend", "default");
        assertThatCode(pooled::close).doesNotThrowAnyException();
    }
}
```

Also create test stubs in the test source:

```java
// StubAgentBackend.java — minimal test impl
package io.casehub.platform.agent.pool;

import io.casehub.platform.agent.*;
import io.smallrye.mutiny.Multi;

record StubAgentBackend(String key, String instanceId) implements AgentBackend {
    @Override public Multi<AgentEvent> invoke(AgentSessionConfig config) { return Multi.createFrom().empty(); }
    @Override public AgentSession openSession(AgentSessionInit init) { return null; }
}

// StubBackendInstanceRegistry.java
package io.casehub.platform.agent.pool;

import io.casehub.platform.agent.*;
import java.util.*;

class StubBackendInstanceRegistry implements BackendInstanceRegistry {
    private final Map<String, AgentBackend> backends = new HashMap<>();

    StubBackendInstanceRegistry(AgentBackend... backends) {
        for (var b : backends) this.backends.put(b.key(), b);
    }

    @Override public void register(AgentBackend backend) { backends.put(backend.key(), backend); }
    @Override public Optional<AgentBackend> resolve(String key, String instanceId) {
        return Optional.ofNullable(backends.get(key));
    }
    @Override public List<AgentBackend> resolveByKey(String key) {
        var b = backends.get(key);
        return b != null ? List.of(b) : List.of();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-api -Dtest=DefaultPoolManagerTest -f platform/pom.xml`
Expected: Compilation failure — classes don't exist yet.

- [ ] **Step 3: Implement the pool types**

```java
// PoolHealth.java
package io.casehub.platform.agent.pool;
public enum PoolHealth { HEALTHY, DEGRADED, UNHEALTHY }

// InstanceMode.java
package io.casehub.platform.agent.pool;
public enum InstanceMode { INVOKE, SESSION }

// PoolConfig.java
package io.casehub.platform.agent.pool;
import java.time.Duration;
public record PoolConfig(
    String backendKey,
    String instanceId,
    int min,
    int max,
    Duration healthCheckInterval,
    Duration idleTimeout,
    Duration acquireTimeout
) {
    public PoolConfig {
        if (min < 0) throw new IllegalArgumentException("min must be >= 0");
        if (max > 0 && max < min) throw new IllegalArgumentException("max must be >= min");
    }

    public static final Duration DEFAULT_HEALTH_CHECK_INTERVAL = Duration.ofSeconds(30);
    public static final Duration DEFAULT_IDLE_TIMEOUT = Duration.ofMinutes(5);
    public static final Duration DEFAULT_ACQUIRE_TIMEOUT = Duration.ofSeconds(10);
}

// PoolStatus.java
package io.casehub.platform.agent.pool;
public record PoolStatus(
    String backendKey,
    int min,
    int max,
    int active,
    int idle,
    int total,
    PoolHealth health
) {}

// PoolExhaustedException.java
package io.casehub.platform.agent.pool;
public class PoolExhaustedException extends RuntimeException {
    private final PoolStatus poolStatus;
    public PoolExhaustedException(PoolStatus poolStatus) {
        super("Pool exhausted for backend '" + poolStatus.backendKey()
                + "': " + poolStatus.active() + "/" + poolStatus.max() + " active");
        this.poolStatus = poolStatus;
    }
    public PoolStatus poolStatus() { return poolStatus; }
}

// PooledBackendInstance.java
package io.casehub.platform.agent.pool;
import io.casehub.platform.agent.AgentBackend;
import java.time.Instant;
import java.util.Objects;

public final class PooledBackendInstance implements AutoCloseable {
    private final AgentBackend delegate;
    private final String poolKey;
    private final Instant acquiredAt;
    private final InstanceMode mode;
    private final PoolManager pool;

    public PooledBackendInstance(AgentBackend delegate, String poolKey,
                                 Instant acquiredAt, InstanceMode mode,
                                 PoolManager pool) {
        this.delegate = Objects.requireNonNull(delegate);
        this.poolKey = Objects.requireNonNull(poolKey);
        this.acquiredAt = Objects.requireNonNull(acquiredAt);
        this.mode = Objects.requireNonNull(mode);
        this.pool = pool;
    }

    public AgentBackend delegate() { return delegate; }
    public String poolKey() { return poolKey; }
    public Instant acquiredAt() { return acquiredAt; }
    public InstanceMode mode() { return mode; }

    @Override
    public void close() {
        if (pool != null) pool.release(this);
    }
}

// PoolManager.java
package io.casehub.platform.agent.pool;
import java.util.List;
import java.util.Map;

public interface PoolManager {
    PooledBackendInstance acquire(String backendKey, String instanceId);
    void release(PooledBackendInstance instance);
    PoolStatus status(String backendKey);
    Map<String, PoolStatus> allStatus();
    default void configure(List<PoolConfig> configs) {}
}

// DefaultPoolManager.java
package io.casehub.platform.agent.pool;
import io.casehub.platform.agent.BackendInstanceRegistry;
import jakarta.enterprise.inject.Default;
import jakarta.inject.Singleton;
import java.time.Instant;
import java.util.List;
import java.util.Map;

@Singleton
@Default
public class DefaultPoolManager implements PoolManager {
    private final BackendInstanceRegistry registry;

    public DefaultPoolManager(BackendInstanceRegistry registry) {
        this.registry = registry;
    }

    @Override
    public PooledBackendInstance acquire(String backendKey, String instanceId) {
        var backend = registry.resolve(backendKey, instanceId)
                .orElseThrow(() -> new IllegalArgumentException(
                        "No backend registered for key='" + backendKey + "', instanceId='" + instanceId + "'"));
        return new PooledBackendInstance(backend, backendKey, Instant.now(), InstanceMode.INVOKE, this);
    }

    @Override
    public void release(PooledBackendInstance instance) {
        // no-op — no pooling in default implementation
    }

    @Override
    public PoolStatus status(String backendKey) {
        return new PoolStatus(backendKey, 0, 0, 0, 0, 0, PoolHealth.HEALTHY);
    }

    @Override
    public Map<String, PoolStatus> allStatus() {
        return Map.of();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-api -Dtest=DefaultPoolManagerTest -f platform/pom.xml`
Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C platform add agent-api/src/main/java/io/casehub/platform/agent/pool/ agent-api/src/test/java/io/casehub/platform/agent/pool/
git -C platform commit -m "feat(#205): PoolManager SPI with DefaultPoolManager no-op impl

Adds PoolManager interface, PoolConfig, PoolStatus, PooledBackendInstance,
PoolExhaustedException, and DefaultPoolManager (pass-through to registry).

Refs casehubio/claudony#205"
```

### Task 2: Manifest pools: section parsing

**Files:**
- Create: `platform/agent-config-core/src/main/java/io/casehub/platform/agent/config/PoolDeclaration.java`
- Modify: `platform/agent-config-core/src/main/java/io/casehub/platform/agent/config/Manifest.java`
- Modify: `platform/agent-config-core/src/main/java/io/casehub/platform/agent/config/ManifestProcessor.java`
- Test: `platform/agent-config-core/src/test/java/io/casehub/platform/agent/config/ManifestPoolParsingTest.java`

**Interfaces:**
- Consumes: `PoolConfig` from Task 1
- Consumes: `PoolManager.configure(List<PoolConfig>)` from Task 1
- Produces: `PoolDeclaration(backend, model, min, max, healthCheckInterval, idleTimeout, acquireTimeout)` — YAML-mapped record
- Produces: `Manifest.pools()` — `Map<String, PoolDeclaration>`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.platform.agent.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.platform.agent.pool.PoolConfig;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import static org.assertj.core.api.Assertions.*;

class ManifestPoolParsingTest {

    private final ObjectMapper yaml = new ObjectMapper(new YAMLFactory());

    @Test
    void parsesPoolsSection() throws Exception {
        var input = """
                pools:
                  api-pool:
                    backend: claude
                    model: opus
                    min: 1
                    max: 5
                    healthCheckInterval: 30s
                    idleTimeout: 5m
                  cli-pool:
                    backend: claudony
                    min: 2
                    max: 10
                    healthCheckInterval: 10s
                    idleTimeout: 15m
                    acquireTimeout: 20s
                """;
        var manifest = yaml.readValue(input, Manifest.class);

        assertThat(manifest.pools()).hasSize(2);

        var apiPool = manifest.pools().get("api-pool");
        assertThat(apiPool.backend()).isEqualTo("claude");
        assertThat(apiPool.model()).isEqualTo("opus");
        assertThat(apiPool.min()).isEqualTo(1);
        assertThat(apiPool.max()).isEqualTo(5);

        var cliPool = manifest.pools().get("cli-pool");
        assertThat(cliPool.backend()).isEqualTo("claudony");
        assertThat(cliPool.min()).isEqualTo(2);
        assertThat(cliPool.max()).isEqualTo(10);
    }

    @Test
    void poolDeclarationToPoolConfig() {
        var decl = new PoolDeclaration("claude", "opus", 1, 5, "30s", "5m", "10s");
        var config = decl.toPoolConfig("api-pool");
        assertThat(config.backendKey()).isEqualTo("claude");
        assertThat(config.min()).isEqualTo(1);
        assertThat(config.max()).isEqualTo(5);
        assertThat(config.healthCheckInterval()).isEqualTo(Duration.ofSeconds(30));
        assertThat(config.idleTimeout()).isEqualTo(Duration.ofMinutes(5));
        assertThat(config.acquireTimeout()).isEqualTo(Duration.ofSeconds(10));
    }

    @Test
    void emptyPoolsDefaultsToEmptyMap() throws Exception {
        var input = """
                models: []
                providers: []
                """;
        var manifest = yaml.readValue(input, Manifest.class);
        assertThat(manifest.pools()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-config-core -Dtest=ManifestPoolParsingTest -f platform/pom.xml`
Expected: Compilation failure — `PoolDeclaration` doesn't exist, `Manifest.pools()` doesn't exist.

- [ ] **Step 3: Implement PoolDeclaration and extend Manifest**

```java
// PoolDeclaration.java
package io.casehub.platform.agent.config;

import com.fasterxml.jackson.annotation.JsonProperty;
import io.casehub.platform.agent.pool.PoolConfig;
import java.time.Duration;

public record PoolDeclaration(
    String backend,
    String model,
    int min,
    int max,
    @JsonProperty("healthCheckInterval") String healthCheckIntervalStr,
    @JsonProperty("idleTimeout") String idleTimeoutStr,
    @JsonProperty("acquireTimeout") String acquireTimeoutStr
) {
    public PoolDeclaration {
        if (backend == null || backend.isBlank())
            throw new IllegalArgumentException("pool backend is required");
    }

    public PoolConfig toPoolConfig(String poolName) {
        return new PoolConfig(
            backend,
            poolName,
            min,
            max,
            healthCheckIntervalStr != null ? parseDuration(healthCheckIntervalStr) : PoolConfig.DEFAULT_HEALTH_CHECK_INTERVAL,
            idleTimeoutStr != null ? parseDuration(idleTimeoutStr) : PoolConfig.DEFAULT_IDLE_TIMEOUT,
            acquireTimeoutStr != null ? parseDuration(acquireTimeoutStr) : PoolConfig.DEFAULT_ACQUIRE_TIMEOUT
        );
    }

    private static Duration parseDuration(String s) {
        if (s.endsWith("s")) return Duration.ofSeconds(Long.parseLong(s.replace("s", "")));
        if (s.endsWith("m")) return Duration.ofMinutes(Long.parseLong(s.replace("m", "")));
        if (s.endsWith("h")) return Duration.ofHours(Long.parseLong(s.replace("h", "")));
        return Duration.parse("PT" + s);
    }
}
```

Add `pools` field to `Manifest.java`:
```java
// Add to Manifest record components:
Map<String, PoolDeclaration> pools

// Add to compact constructor:
pools = pools != null ? Map.copyOf(pools) : Map.of();
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-config-core -Dtest=ManifestPoolParsingTest -f platform/pom.xml`
Expected: All 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C platform add agent-config-core/src/main/java/io/casehub/platform/agent/config/PoolDeclaration.java agent-config-core/src/main/java/io/casehub/platform/agent/config/Manifest.java agent-config-core/src/test/java/io/casehub/platform/agent/config/ManifestPoolParsingTest.java
git -C platform commit -m "feat(#205): manifest pools: section parsing

Adds PoolDeclaration record and pools field to Manifest.
ManifestProcessor can read pool config from YAML.

Refs casehubio/claudony#205"
```

---

## Batch 2: Claudony — ClaudonyAgentBackend + FleetPoolManager

After this batch, Claudony registers CLI sessions as a platform backend and manages pools with full lifecycle. The WorkerProvisioner path acquires from the pool.

### Task 3: ClaudonyAgentBackend — CLI sessions as AgentBackend

**Files:**
- Create: `claudony/casehub/src/main/java/io/casehub/claudony/casehub/fleet/ClaudonyAgentBackend.java`
- Test: `claudony/casehub/src/test/java/io/casehub/claudony/casehub/fleet/ClaudonyAgentBackendTest.java`

**Interfaces:**
- Consumes: `AgentBackend` (platform agent-api) — `key()`, `instanceId()`, `invoke()`, `openSession()`
- Consumes: `TmuxService.createWorkerSession()` (claudony-core)
- Consumes: `SessionRegistry` (claudony-core)
- Produces: `ClaudonyAgentBackend` — registered as `AgentBackend` with key `"claudony"`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.platform.agent.AgentBackend;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ClaudonyAgentBackendTest {

    @Test
    void key_isClaudony() {
        var backend = new ClaudonyAgentBackend(null, null, null);
        assertThat(backend.key()).isEqualTo("claudony");
    }

    @Test
    void instanceId_isDefault() {
        var backend = new ClaudonyAgentBackend(null, null, null);
        assertThat(backend.instanceId()).isEqualTo("default");
    }

    @Test
    void implementsAgentBackend() {
        var backend = new ClaudonyAgentBackend(null, null, null);
        assertThat(backend).isInstanceOf(AgentBackend.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyAgentBackendTest`
Expected: Compilation failure — class doesn't exist.

- [ ] **Step 3: Implement ClaudonyAgentBackend**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.claudony.config.ClaudonyConfig;
import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TmuxService;
import io.casehub.platform.agent.*;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class ClaudonyAgentBackend implements AgentBackend {

    private final TmuxService tmuxService;
    private final SessionRegistry sessionRegistry;
    private final ClaudonyConfig config;

    @Inject
    public ClaudonyAgentBackend(TmuxService tmuxService,
                                 SessionRegistry sessionRegistry,
                                 ClaudonyConfig config) {
        this.tmuxService = tmuxService;
        this.sessionRegistry = sessionRegistry;
        this.config = config;
    }

    @Override
    public String key() { return "claudony"; }

    @Override
    public String instanceId() { return "default"; }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig sessionConfig) {
        // One-shot: create tmux worker session, run command, stream events
        throw new UnsupportedOperationException("CLI invoke not yet implemented");
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        // Interactive: create tmux session, return handle
        throw new UnsupportedOperationException("CLI openSession not yet implemented");
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyAgentBackendTest`
Expected: All 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ClaudonyAgentBackend.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/ClaudonyAgentBackendTest.java
git commit -m "feat(#205): ClaudonyAgentBackend — CLI sessions as platform backend

Registers Claudony CLI sessions as an AgentBackend with key 'claudony'.
invoke() and openSession() stubs — full impl in follow-on.

Refs #205"
```

### Task 4: FleetPoolManager — capacity-bounded pool lifecycle

**Files:**
- Create: `claudony/casehub/src/main/java/io/casehub/claudony/casehub/fleet/FleetPoolManager.java`
- Create: `claudony/casehub/src/main/java/io/casehub/claudony/casehub/fleet/BackendPool.java`
- Test: `claudony/casehub/src/test/java/io/casehub/claudony/casehub/fleet/FleetPoolManagerTest.java`

**Interfaces:**
- Consumes: `PoolManager` SPI from Task 1
- Consumes: `PoolConfig` from Task 1
- Consumes: `BackendInstanceRegistry` (platform)
- Produces: `FleetPoolManager` — `@Alternative @Priority(1)` impl of `PoolManager`
- Produces: `BackendPool` — per-backend pool with acquire/release/eviction/health

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.platform.agent.pool.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class FleetPoolManagerTest {

    private FleetPoolManager poolManager;
    private StubBackendFactory factory;

    @BeforeEach
    void setUp() {
        factory = new StubBackendFactory();
        poolManager = new FleetPoolManager(factory);
    }

    @Test
    void configure_preWarmsToMin() {
        poolManager.configure(List.of(new PoolConfig(
                "test", "default", 2, 5,
                Duration.ofSeconds(30), Duration.ofMinutes(5), Duration.ofSeconds(10))));
        var status = poolManager.status("test");
        assertThat(status.idle()).isEqualTo(2);
        assertThat(status.total()).isEqualTo(2);
        assertThat(status.active()).isZero();
    }

    @Test
    void acquire_returnsPreWarmedInstance() {
        poolManager.configure(List.of(new PoolConfig(
                "test", "default", 1, 5,
                Duration.ofSeconds(30), Duration.ofMinutes(5), Duration.ofSeconds(10))));
        var instance = poolManager.acquire("test", "default");
        assertThat(instance).isNotNull();
        assertThat(instance.poolKey()).isEqualTo("test");

        var status = poolManager.status("test");
        assertThat(status.active()).isEqualTo(1);
        assertThat(status.idle()).isZero();
    }

    @Test
    void acquire_createsOnDemandUpToMax() {
        poolManager.configure(List.of(new PoolConfig(
                "test", "default", 0, 2,
                Duration.ofSeconds(30), Duration.ofMinutes(5), Duration.ofSeconds(10))));
        var i1 = poolManager.acquire("test", "default");
        var i2 = poolManager.acquire("test", "default");
        assertThat(poolManager.status("test").active()).isEqualTo(2);

        assertThatThrownBy(() -> poolManager.acquire("test", "default"))
                .isInstanceOf(PoolExhaustedException.class);
    }

    @Test
    void release_returnsToIdle() {
        poolManager.configure(List.of(new PoolConfig(
                "test", "default", 0, 5,
                Duration.ofSeconds(30), Duration.ofMinutes(5), Duration.ofSeconds(10))));
        var instance = poolManager.acquire("test", "default");
        poolManager.release(instance);

        var status = poolManager.status("test");
        assertThat(status.active()).isZero();
        assertThat(status.idle()).isEqualTo(1);
    }

    @Test
    void allStatus_returnsConfiguredPools() {
        poolManager.configure(List.of(
                new PoolConfig("a", "default", 1, 3, Duration.ofSeconds(30), Duration.ofMinutes(5), Duration.ofSeconds(10)),
                new PoolConfig("b", "default", 0, 2, Duration.ofSeconds(30), Duration.ofMinutes(5), Duration.ofSeconds(10))
        ));
        assertThat(poolManager.allStatus()).hasSize(2);
        assertThat(poolManager.allStatus()).containsKeys("a", "b");
    }

    @Test
    void status_unknownPool_returnsEmpty() {
        var status = poolManager.status("unknown");
        assertThat(status.backendKey()).isEqualTo("unknown");
        assertThat(status.total()).isZero();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=FleetPoolManagerTest`
Expected: Compilation failure.

- [ ] **Step 3: Implement FleetPoolManager and BackendPool**

`BackendPool` manages a single pool: maintains idle and active sets, handles acquire/release, pre-warming, and capacity enforcement.

`FleetPoolManager` maps `backendKey → BackendPool` and delegates. It takes a `BackendFactory` functional interface to create new backend instances (for testability — in production, this wraps the real `BackendInstanceRegistry` + `TmuxService`/`ChatModelProvider`).

Key implementation points:
- Pre-warm `min` instances on `configure()`
- `acquire()` takes from idle set, or creates on demand up to `max`; throws `PoolExhaustedException` if at max and acquireTimeout expires
- `release()` returns to idle set (API backends) or destroys and replaces (CLI backends — session recycling per spec)
- Thread-safe via `ReentrantLock` per pool

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=FleetPoolManagerTest`
Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ casehub/src/test/java/io/casehub/claudony/casehub/fleet/
git commit -m "feat(#205): FleetPoolManager — capacity-bounded pool lifecycle

Capacity-bounded on-demand pools with pre-warming, acquire/release,
and PoolExhaustedException on timeout.

Refs #205"
```

---

## Batch 3: Claudony — Observability + WorkerProvisioner Integration

After this batch, the fleet is observable via REST and the WorkerProvisioner path acquires from the pool.

### Task 5: Fleet REST endpoints

**Files:**
- Create: `claudony/app/src/main/java/io/casehub/claudony/server/fleet/FleetResource.java`
- Test: `claudony/app/src/test/java/io/casehub/claudony/server/fleet/FleetResourceTest.java`

**Interfaces:**
- Consumes: `PoolManager.allStatus()`, `PoolManager.status(backendKey)` from Task 1
- Produces: `GET /api/fleet` → all pool statuses JSON
- Produces: `GET /api/fleet/{backendKey}` → single pool status JSON

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.claudony.server.fleet;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class FleetResourceTest {

    @Test
    void getFleetStatus_returnsAllPools() {
        RestAssured.given()
                .when().get("/api/fleet")
                .then()
                .statusCode(200)
                .body("$", instanceOf(Map.class));
    }

    @Test
    void getPoolStatus_unknownBackend_returnsEmptyStatus() {
        RestAssured.given()
                .when().get("/api/fleet/nonexistent")
                .then()
                .statusCode(200)
                .body("backendKey", equalTo("nonexistent"))
                .body("total", equalTo(0));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=FleetResourceTest`
Expected: 404 — endpoint doesn't exist.

- [ ] **Step 3: Implement FleetResource**

```java
package io.casehub.claudony.server.fleet;

import io.casehub.platform.agent.pool.PoolManager;
import io.casehub.platform.agent.pool.PoolStatus;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import java.util.Map;

@Path("/api/fleet")
@Produces(MediaType.APPLICATION_JSON)
public class FleetResource {

    @Inject
    PoolManager poolManager;

    @GET
    public Map<String, PoolStatus> allStatus() {
        return poolManager.allStatus();
    }

    @GET
    @Path("/{backendKey}")
    public PoolStatus status(@PathParam("backendKey") String backendKey) {
        return poolManager.status(backendKey);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=FleetResourceTest`
Expected: All 2 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/fleet/FleetResource.java app/src/test/java/io/casehub/claudony/server/fleet/FleetResourceTest.java
git commit -m "feat(#205): fleet REST endpoints — GET /api/fleet

Pool status observability via REST.

Refs #205"
```

### Task 6: Wire WorkerProvisioner to acquire from pool

**Files:**
- Modify: `claudony/casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisioner.java`
- Modify: `claudony/casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisionerTest.java`

**Interfaces:**
- Consumes: `PoolManager.acquire("claudony", ...)` from Task 1/4
- Consumes: `PoolManager.release()` from Task 1/4
- Modifies: `ClaudonyWorkerProvisioner.provision()` — acquires from pool instead of direct `TmuxService.createWorkerSession()`

- [ ] **Step 1: Write the failing test**

Add a new test to `ClaudonyWorkerProvisionerTest`:

```java
@Test
void provision_acquiresFromPoolManager() {
    // Arrange: configure pool manager mock
    when(poolManager.acquire(eq("claudony"), any()))
            .thenReturn(new PooledBackendInstance(
                    claudonyBackend, "claudony", Instant.now(),
                    InstanceMode.SESSION, poolManager));

    // Act: provision
    var result = provisioner.provision(capabilities, context);

    // Assert: pool was used
    verify(poolManager).acquire(eq("claudony"), any());
    assertThat(result).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyWorkerProvisionerTest#provision_acquiresFromPoolManager`
Expected: FAIL — provisioner doesn't use PoolManager yet.

- [ ] **Step 3: Inject PoolManager into ClaudonyWorkerProvisioner**

Add `PoolManager` as a constructor parameter. In `provision()`, call `poolManager.acquire("claudony", ...)` to get a session from the pool. On worker completion (via `ClaudonyWorkerStatusListener`), call `poolManager.release()`.

- [ ] **Step 4: Run full provisioner test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ClaudonyWorkerProvisionerTest`
Expected: All tests PASS (existing + new).

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisioner.java casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisionerTest.java
git commit -m "feat(#205): wire WorkerProvisioner to acquire from PoolManager

Provision path now uses pool acquire/release instead of direct
TmuxService calls.

Refs #205"
```

---

## Batch 4: Engine — AgentConverter convergence

After this batch, engine YAML-defined agents resolve models through `RoutingAgentProvider` instead of constructing LangChain4j models inline. This is the highest-risk batch — sequenced last so Batches 1-3 are stable.

### Task 7: Converge AgentConverter onto RoutingAgentProvider

**Files:**
- Modify: `engine/api/src/main/java/io/casehub/api/model/converter/AgentConverter.java`
- Test: `engine/api/src/test/java/io/casehub/api/model/converter/AgentConverterTest.java`

**Interfaces:**
- Consumes: `AgentProvider.invoke(AgentSessionConfig)` (platform)
- Consumes: `RoutingAgentProvider` (platform agent-router-core) — resolves model refs to backends
- Modifies: `AgentConverter.toApiAgent()` — replaces inline `ChatModelProvider` construction with `AgentProvider` delegation

- [ ] **Step 1: Write the failing test**

```java
@Test
void toApiAgent_usesAgentProviderInsteadOfInlineConstruction() {
    // Verify that after convergence, AgentConverter delegates to AgentProvider
    // rather than constructing ChatModelProvider instances directly
    var agentNode = objectMapper.readTree("""
            {
              "model": "opus",
              "systemPrompt": "You are a test agent"
            }
            """);

    var agent = AgentConverter.toApiAgent(agentNode, agentProvider);
    assertThat(agent).isNotNull();
    verify(agentProvider).invoke(argThat(config ->
            config.model().equals("opus")));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentConverterTest -f engine/pom.xml`
Expected: FAIL — `toApiAgent` doesn't take `AgentProvider` parameter.

- [ ] **Step 3: Refactor AgentConverter**

Add `AgentProvider` as a parameter to `toApiAgent()`. Replace the `toChatModelProviderFromNode()` switch with delegation to `AgentProvider`:

```java
public static Agent toApiAgent(JsonNode agentNode, AgentProvider agentProvider) {
    if (agentNode == null || agentNode.isNull()) return null;

    String modelRef;
    JsonNode modelNode = agentNode.get("model");
    if (modelNode != null && modelNode.isObject() && modelNode.size() > 0) {
        var entry = modelNode.fields().next();
        modelRef = entry.getKey();
    } else {
        modelRef = modelNode != null ? modelNode.asText() : null;
    }

    return Agent.builder()
            .systemPrompt(agentNode.has("systemPrompt") ? agentNode.get("systemPrompt").asText() : null)
            .modelRef(modelRef)
            .agentProvider(agentProvider)
            .build();
}
```

Preserve the old `toApiAgent(JsonNode)` as `@Deprecated` for backward compatibility during migration.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentConverterTest -f engine/pom.xml`
Expected: All tests PASS.

- [ ] **Step 5: Run full engine test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f engine/pom.xml`
Expected: All tests PASS. Watch for failures in tests that call `toApiAgent(node)` without the new parameter — update callers to pass `AgentProvider`.

- [ ] **Step 6: Commit**

```bash
git -C engine add api/src/main/java/io/casehub/api/model/converter/AgentConverter.java api/src/test/java/io/casehub/api/model/converter/AgentConverterTest.java
git -C engine commit -m "feat(#205): converge AgentConverter onto RoutingAgentProvider

YAML-defined agents now resolve models through the platform routing
stack instead of constructing LangChain4j ChatModel instances inline.
Old single-arg toApiAgent preserved as @Deprecated.

Refs casehubio/claudony#205"
```

---

## References

- [2026-09-21-llm-fleet-manager-design.md] — design spec this plan implements
- `io.casehub.platform.agent.AgentBackend` (platform agent-api) — backend SPI
- `io.casehub.platform.agent.BackendInstanceRegistry` (platform agent-api) — backend registry
- `io.casehub.platform.agent.config.Manifest` (platform agent-config-core) — manifest record
- `io.casehub.platform.agent.config.ManifestProcessor` (platform agent-config-core) — manifest processing
- `io.casehub.api.model.converter.AgentConverter` (engine api) — inline ChatModel construction
- `io.casehub.claudony.casehub.ClaudonyWorkerProvisioner` (claudony casehub) — worker provisioning
- `io.casehub.claudony.server.TmuxService` (claudony core) — tmux session management
- Protocol PP-20260605-4b6c4e — CaseHub workers must use createWorkerSession()
- Protocol PP-20260616-d32bc3 — reactive Panache calls in provision()
- GitHub casehubio/claudony#205 — focal issue
