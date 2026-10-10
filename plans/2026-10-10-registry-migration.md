# Registry Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #268 — feat: migrate PeerRegistry, AgentPoolDefinitionRegistry, PoolMeshRegistrar to unified RegistryService
**Issue group:** #268

**Goal:** Migrate three Claudony registries to the platform's unified RegistryService SPI for unified topology discovery.

**Architecture:** Internal delegation — each existing class is modified in place to use RegistryService as its backing store. PeerRegistry and AgentPoolDefinitionRegistry keep their domain logic and public API unchanged. PoolMeshRegistrar switches from InstanceService to RegistryService relationships.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api (RegistryService SPI), casehub-platform-registry-inmem-core (InMemoryRegistryService for tests)

## Global Constraints

- `casehub-platform-api` is already on the classpath (transitive via `casehub-platform-agent-api` in casehub module, direct in app module)
- `InMemoryRegistryService` constructor: `new InMemoryRegistryService(event -> {})` for unit tests (no-op CDI event sink)
- RegistryEntry is immutable — mutations use `withHealth()` / `withHeartbeat()` or construct a new record
- RegistryQuery requires non-null `tenancyId` — always pass `"default"`
- All three migrations use `namespace="fleet"` and `tenancyId="default"`
- Use `ide_edit_member`/`ide_replace_member`/`ide_insert_member` for all Java edits — never Edit/Write on existing .java files

---

## Batch 1: Dependencies and AgentPoolDefinitionRegistry migration

### Task 1: Add registry-inmem dependencies

**Files:**
- Modify: `pom.xml` (parent dependencyManagement)
- Modify: `casehub/pom.xml` (add registry-inmem-core test dep)
- Modify: `app/pom.xml` (add registry-inmem runtime dep)

**Interfaces:**
- Produces: `InMemoryRegistryService` available on test classpath of both modules

- [ ] **Step 1: Add to parent dependencyManagement**

In `pom.xml`, add inside `<dependencyManagement><dependencies>`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-registry-inmem-core</artifactId>
  <version>${casehub-platform.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-registry-inmem</artifactId>
  <version>${casehub-platform.version}</version>
</dependency>
```

- [ ] **Step 2: Add test dep to casehub/pom.xml**

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-registry-inmem-core</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 3: Add runtime dep to app/pom.xml**

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-registry-inmem</artifactId>
</dependency>
```

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
feat(#268): add casehub-platform-registry-inmem dependencies

Refs casehubio/claudony#268
```

---

### Task 2: Migrate AgentPoolDefinitionRegistry to dual-store

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistryTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistryUpdateTest.java`

**Interfaces:**
- Consumes: `RegistryService` (from platform-api, already on classpath)
- Produces: Same public API — `register()`, `get()`, `all()`, `names()`, `isEmpty()`, `size()`, `updateScaling()`, `updateCapacity()`, `updateBudget()`

- [ ] **Step 1: Write failing test — registry entries are created on register**

In `AgentPoolDefinitionRegistryTest`, add a new field and update setUp:

```java
import io.casehub.platform.api.registry.RegistryService;
import io.casehub.platform.api.registry.RegistryQuery;
import io.casehub.platform.registry.memory.InMemoryRegistryService;

// Add field:
private RegistryService registryService;

// Update setUp:
void setUp() {
    registryService = new InMemoryRegistryService(event -> {});
    registry = new AgentPoolDefinitionRegistry(registryService);
}
```

Add test:

```java
@Test
void register_createsRegistryEntry() {
    var def = AgentPoolDefinition.builder()
            .agent("code-reviewer")
                .workingDir("/reviews")
            .build();

    registry.register(def);

    var entries = registryService.discover(new RegistryQuery("default", "pool", "fleet"));
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).id()).isEqualTo("code-reviewer");
    assertThat(entries.get(0).type()).isEqualTo("pool");
    assertThat(entries.get(0).metadata()).containsEntry("agentName", "code-reviewer");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolDefinitionRegistryTest#register_createsRegistryEntry`
Expected: FAIL — no constructor `AgentPoolDefinitionRegistry(RegistryService)`

- [ ] **Step 3: Implement — add RegistryService field and constructor**

In `AgentPoolDefinitionRegistry.java`:

1. Add field: `private final RegistryService registryService;`
2. Add constructor: `@Inject AgentPoolDefinitionRegistry(RegistryService registryService)`
3. Keep the no-arg constructor for backward compat during migration (delegates with `NoOpRegistryService`)
4. Add private method `toRegistryEntry(AgentPoolDefinition def)`:

```java
private RegistryEntry toRegistryEntry(AgentPoolDefinition def) {
    var now = Instant.now();
    return new RegistryEntry(
            def.agent().name(), "pool", "fleet", "default",
            Map.of(
                "agentName", def.agent().name(),
                "minActive", String.valueOf(def.pool().minActive()),
                "maxActive", String.valueOf(def.pool().maxActive())
            ),
            now, now, Duration.ofHours(24), HealthStatus.HEALTHY);
}
```

5. In `register()`, after `putIfAbsent`, add: `registryService.register(toRegistryEntry(definition));`
6. In `updateScaling()`, after `compute`, add: `registryService.register(toRegistryEntry(definitions.get(agentName)));`
7. Same for `updateCapacity()` and `updateBudget()`

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolDefinitionRegistryTest`
Expected: ALL PASS

- [ ] **Step 5: Write test — deregister removes registry entry (not yet needed but verifiable)**

Add test:

```java
@Test
void registryEntry_updatedOnScalingChange() {
    var def = AgentPoolDefinition.builder().agent("test").pool()
            .minActive(0).maxActive(10).build();
    registry.register(def);

    registry.updateCapacity("test", 2, 20);

    var entries = registryService.discover(new RegistryQuery("default", "pool", "fleet"));
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).metadata()).containsEntry("minActive", "2");
    assertThat(entries.get(0).metadata()).containsEntry("maxActive", "20");
}
```

- [ ] **Step 6: Run all tests in casehub module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: ALL PASS

- [ ] **Step 7: Update AgentPoolDefinitionRegistryUpdateTest**

Add `RegistryService` to test setup:

```java
private RegistryService registryService;

// In each test, change:
// var registry = new AgentPoolDefinitionRegistry();
// to:
// var registryService = new InMemoryRegistryService(event -> {});
// var registry = new AgentPoolDefinitionRegistry(registryService);
```

- [ ] **Step 8: Run update tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolDefinitionRegistryUpdateTest`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat(#268): migrate AgentPoolDefinitionRegistry to dual-store with RegistryService

Pool definitions are registered in RegistryService (type="pool") for
unified topology discovery. ConcurrentHashMap kept for type-safe domain
data. Both stores updated on every mutation.

Refs casehubio/claudony#268
```

---

## Batch 2: PeerRegistry migration

### Task 3: Migrate PeerRegistry to RegistryService internal delegation

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PeerRegistry.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PeerEntry.java`
- Modify: `app/src/test/java/io/casehub/claudony/server/fleet/PeerRegistryTest.java`

**Interfaces:**
- Consumes: `RegistryService` (from platform-api)
- Produces: Same public API — `addPeer()`, `removePeer()`, `findById()`, `getAllPeers()`, `getHealthyPeers()`, `getAllEntries()`, `updatePeer()`, `recordSuccess()`, `recordFailure()`, `updateCachedSessions()`, `getCachedSessions()`

- [ ] **Step 1: Write failing test — addPeer creates a RegistryEntry**

In `PeerRegistryTest`, add field and update setUp:

```java
import io.casehub.platform.api.registry.RegistryService;
import io.casehub.platform.api.registry.RegistryQuery;
import io.casehub.platform.api.registry.HealthStatus;
import io.casehub.platform.registry.memory.InMemoryRegistryService;

// Add field:
private InMemoryRegistryService registryService;

// Update setUp:
void setUp() {
    registryService = new InMemoryRegistryService(event -> {});
    registry = new PeerRegistry(tempDir, registryService);
}
```

Add test:

```java
@Test
void addPeer_createsRegistryEntry() {
    registry.addPeer("id1", "http://peer-a:7777", "Peer A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
    var entry = registryService.resolve("id1");
    assertThat(entry).isPresent();
    assertThat(entry.get().type()).isEqualTo("node");
    assertThat(entry.get().namespace()).isEqualTo("fleet");
    assertThat(entry.get().metadata()).containsEntry("url", "http://peer-a:7777");
    assertThat(entry.get().metadata()).containsEntry("name", "Peer A");
    assertThat(entry.get().metadata()).containsEntry("source", "MANUAL");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PeerRegistryTest#addPeer_createsRegistryEntry`
Expected: FAIL — no constructor `PeerRegistry(Path, RegistryService)`

- [ ] **Step 3: Implement PeerEntry changes — remove health field**

In `PeerEntry.java`:
1. Remove `volatile PeerHealth health = PeerHealth.UNKNOWN;` field
2. `recordSuccess()` no longer sets `health = PeerHealth.UP` — it only resets circuit state. Health updates go to RegistryService (done by PeerRegistry).
3. `recordFailure()` no longer sets `health = PeerHealth.DOWN` — same reason.
4. `toRecord()` needs health from the registry. Change signature to accept a `HealthStatus` parameter or have PeerRegistry resolve it. Better approach: PeerRegistry passes health when constructing PeerRecord.

Change `toRecord()` to `toRecord(PeerHealth health)`:

```java
PeerRecord toRecord(PeerHealth health) {
    return new PeerRecord(id, url, name, source, terminalMode, health, circuitState,
            lastSeen, health == PeerHealth.DOWN && !cachedSessions.isEmpty(), cachedSessions.size());
}
```

- [ ] **Step 4: Implement PeerRegistry changes — add RegistryService delegation**

In `PeerRegistry.java`:
1. Add field: `private final RegistryService registryService;`
2. Add constructor: `PeerRegistry(Path configDir, RegistryService registryService)` (test constructor)
3. Update CDI constructor to `@Inject` RegistryService
4. Add private helper:

```java
private RegistryEntry toPeerEntry(String id, String url, String name,
                                   DiscoverySource source, TerminalMode terminalMode) {
    var now = Instant.now();
    return new RegistryEntry(id, "node", "fleet", "default",
            Map.of("url", url, "name", name,
                    "source", source.name(),
                    "terminalMode", terminalMode.name()),
            now, now, Duration.ofMinutes(5), HealthStatus.DEGRADED);
}
```

Note: new peers start with `DEGRADED` (maps to `PeerHealth.UNKNOWN`).

5. In `addPeer()`: after `peers.put(id, ...)`, add `registryService.register(toPeerEntry(...))`. When deduplication removes an old peer, call `registryService.deregister(existing.get().id)` first.

6. In `removePeer()`: after `peers.remove(id)`, add `registryService.deregister(id)`.

7. In `recordSuccess()`: after `entry.recordSuccess()`, add:
```java
registryService.resolve(id).ifPresent(e -> registryService.register(e.withHealth(HealthStatus.HEALTHY)));
```

8. In `recordFailure()`: after `entry.recordFailure()`, add:
```java
registryService.resolve(id).ifPresent(e -> registryService.register(e.withHealth(HealthStatus.DOWN)));
```

9. Add helper to map HealthStatus → PeerHealth:
```java
private static PeerHealth mapHealth(HealthStatus status) {
    return switch (status) {
        case HEALTHY -> PeerHealth.UP;
        case DOWN -> PeerHealth.DOWN;
        case DEGRADED -> PeerHealth.UNKNOWN;
    };
}
```

10. Update `findById()` — read health from registry:
```java
public Optional<PeerRecord> findById(String id) {
    var entry = peers.get(id);
    if (entry == null) return Optional.empty();
    var health = registryService.resolve(id)
            .map(e -> mapHealth(e.health()))
            .orElse(PeerHealth.UNKNOWN);
    return Optional.of(entry.toRecord(health));
}
```

11. Update `getAllPeers()` and `getHealthyPeers()` similarly.

12. Update `getHealthyPeers()` to filter by registry health:
```java
public List<PeerRecord> getHealthyPeers() {
    return peers.values().stream()
            .filter(e -> e.circuitState != CircuitState.OPEN)
            .map(e -> {
                var health = registryService.resolve(e.id)
                        .map(re -> mapHealth(re.health()))
                        .orElse(PeerHealth.UNKNOWN);
                return e.toRecord(health);
            })
            .toList();
}
```

- [ ] **Step 5: Run the new test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PeerRegistryTest#addPeer_createsRegistryEntry`
Expected: PASS

- [ ] **Step 6: Write test — removePeer deregisters from registry**

```java
@Test
void removePeer_deregistersFromRegistry() {
    registry.addPeer("id1", "http://peer-a:7777", "Peer A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
    registry.removePeer("id1");
    assertThat(registryService.resolve("id1")).isEmpty();
}
```

- [ ] **Step 7: Write test — recordSuccess updates registry health**

```java
@Test
void recordSuccess_updatesRegistryHealth() {
    registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
    registry.recordSuccess("id1");
    var entry = registryService.resolve("id1");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.HEALTHY);
}
```

- [ ] **Step 8: Write test — recordFailure updates registry health**

```java
@Test
void recordFailure_updatesRegistryHealthToDown() {
    registry.addPeer("id1", "http://peer-a:7777", "A", DiscoverySource.MANUAL, TerminalMode.DIRECT);
    registry.recordFailure("id1");
    var entry = registryService.resolve("id1");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.DOWN);
}
```

- [ ] **Step 9: Run all PeerRegistryTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PeerRegistryTest`
Expected: ALL PASS (existing tests should pass unchanged — same public API)

- [ ] **Step 10: Run full app module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app`
Expected: PASS (watch for CDI conflicts with InMemoryRegistryService in @QuarkusTest classes)

- [ ] **Step 11: Commit**

```
feat(#268): migrate PeerRegistry to RegistryService internal delegation

Peers stored as RegistryEntry(type="node", namespace="fleet"). PeerHealth
mapped to HealthStatus. Circuit breaker FSM stays on PeerEntry. File
persistence unchanged.

Refs casehubio/claudony#268
```

---

## Batch 3: PoolMeshRegistrar migration

### Task 4: Migrate PoolMeshRegistrar to RegistryService relationships

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PoolMeshRegistrar.java`
- Modify: `app/src/test/java/io/casehub/claudony/server/fleet/PoolMeshIntegrationTest.java`

**Interfaces:**
- Consumes: `RegistryService` (replaces `InstanceService`)
- Produces: Same SPI — `SessionLifecycleListener` implementation (onAcquired, onSuspended, onResumed, onDestroyed)

- [ ] **Step 1: Write failing test — onAcquired links pool to session**

Rewrite `PoolMeshIntegrationTest` setUp to use `InMemoryRegistryService`:

```java
import io.casehub.platform.api.registry.RegistryService;
import io.casehub.platform.api.registry.RegistryEntry;
import io.casehub.platform.api.registry.HealthStatus;
import io.casehub.platform.api.registry.Relationship;
import io.casehub.platform.api.registry.RegistryQuery;
import io.casehub.platform.registry.memory.InMemoryRegistryService;

// Replace fields:
private InMemoryRegistryService registryService;
// Remove: private InMemoryInstanceStore instanceStore;
// Remove: private InstanceService instanceService;

// Update setUp:
registryService = new InMemoryRegistryService(event -> {});
// Remove: instanceStore = new InMemoryInstanceStore();
// Remove: instanceService = new InstanceService(instanceStore);
```

Add test:

```java
@Test
void acquire_linksPoolToSession() {
    var registrar = new PoolMeshRegistrar(registryService);
    var session = new ManagedSession("session-1", "reviewer-1", "/tmp", null);

    registrar.onAcquired(session, "code-reviewer");

    var rels = registryService.relationships("code-reviewer");
    assertThat(rels).hasSize(1);
    assertThat(rels.get(0).sourceId()).isEqualTo("code-reviewer");
    assertThat(rels.get(0).targetId()).isEqualTo("session-1");
    assertThat(rels.get(0).type()).isEqualTo("contains");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PoolMeshIntegrationTest#acquire_linksPoolToSession`
Expected: FAIL — no constructor `PoolMeshRegistrar(RegistryService)`

- [ ] **Step 3: Implement PoolMeshRegistrar — replace InstanceService with RegistryService**

Replace the entire class body:

```java
@ApplicationScoped
public class PoolMeshRegistrar implements SessionLifecycleListener {

    private final RegistryService registryService;

    @Inject
    public PoolMeshRegistrar(RegistryService registryService) {
        this.registryService = registryService;
    }

    @Override
    public void onAcquired(ManagedSession session, String poolName) {
        registryService.link(new Relationship(poolName, session.instanceId(), "contains"));
    }

    @Override
    public void onSuspended(ManagedSession session, String poolName) {
        registryService.resolve(session.instanceId())
                .ifPresent(entry -> registryService.register(entry.withHealth(HealthStatus.DEGRADED)));
    }

    @Override
    public void onResumed(ManagedSession session, String poolName) {
        registryService.resolve(session.instanceId())
                .ifPresent(entry -> registryService.register(entry.withHealth(HealthStatus.HEALTHY)));
    }

    @Override
    public void onDestroyed(String sessionId, String poolName) {
        registryService.unlink(poolName, sessionId);
    }
}
```

- [ ] **Step 4: Run the new test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PoolMeshIntegrationTest#acquire_linksPoolToSession`
Expected: PASS

- [ ] **Step 5: Write test — onSuspended updates health to DEGRADED**

```java
@Test
void suspend_updatesHealthToDegraded() {
    var registrar = new PoolMeshRegistrar(registryService);
    var session = new ManagedSession("session-1", "reviewer-1", "/tmp", null);

    // Pre-register the session as an agent-instance (simulates RegistryBackedInstanceManager)
    var now = Instant.now();
    registryService.register(new RegistryEntry(
            "session-1", "agent-instance", "default", "default",
            Map.of("description", "pool:code-reviewer/reviewer-1"),
            now, now, Duration.ofMinutes(5), HealthStatus.HEALTHY));

    registrar.onSuspended(session, "code-reviewer");

    var entry = registryService.resolve("session-1");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.DEGRADED);
}
```

- [ ] **Step 6: Write test — onResumed updates health to HEALTHY**

```java
@Test
void resume_updatesHealthToHealthy() {
    var registrar = new PoolMeshRegistrar(registryService);
    var session = new ManagedSession("session-1", "reviewer-1", "/tmp", null);

    var now = Instant.now();
    registryService.register(new RegistryEntry(
            "session-1", "agent-instance", "default", "default",
            Map.of("description", "pool:code-reviewer/reviewer-1"),
            now, now, Duration.ofMinutes(5), HealthStatus.DEGRADED));

    registrar.onResumed(session, "code-reviewer");

    var entry = registryService.resolve("session-1");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.HEALTHY);
}
```

- [ ] **Step 7: Write test — onDestroyed unlinks**

```java
@Test
void destroy_unlinksPoolFromSession() {
    var registrar = new PoolMeshRegistrar(registryService);
    var session = new ManagedSession("session-1", "reviewer-1", "/tmp", null);

    registrar.onAcquired(session, "code-reviewer");
    assertThat(registryService.relationships("code-reviewer")).hasSize(1);

    registrar.onDestroyed("session-1", "code-reviewer");
    assertThat(registryService.relationships("code-reviewer")).isEmpty();
}
```

- [ ] **Step 8: Rewrite remaining integration tests**

The existing tests (`acquire_registersAsQhorusInstance`, `suspend_marksInstanceOffline`, etc.) test with real tmux and InstanceService. These need to be updated or replaced:

- Tests that verify InstanceService behavior (instance registration, capability routing) should be deleted — that responsibility moved to RegistryBackedInstanceManager (Qhorus).
- Tests that verify lifecycle callbacks (link/unlink/health) are covered by the new unit tests above.
- Keep the `fullLifecycle_cleanState` test but rewrite assertions against RegistryService.

- [ ] **Step 9: Run all PoolMeshIntegrationTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=PoolMeshIntegrationTest`
Expected: ALL PASS

- [ ] **Step 10: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: ALL PASS across all modules

- [ ] **Step 11: Check for CDI conflicts**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SmokeTest`
Expected: PASS — InMemoryRegistryService @Alternative @Priority(50) should displace NoOpRegistryService without conflict

If CDI issues arise with `CasehubEnabledProfile` or `CompletionTestProfile`, update their `quarkus.arc.exclude-types` per protocol `PP-20260612-d6e7ec`.

- [ ] **Step 12: Commit**

```
feat(#268): migrate PoolMeshRegistrar to RegistryService relationships

Replace InstanceService with RegistryService. Pool→session containment
expressed via Relationship links. Suspend/resume mapped to health status
updates. Instance registration delegated to RegistryBackedInstanceManager.

Closes casehubio/claudony#268
```

---

## References

- [2026-10-10-registry-migration-design.md] — design spec this plan implements
- `platform-api/RegistryService.java` — SPI interface (register, deregister, heartbeat, resolve, discover, link, unlink, watch)
- `platform-api/RegistryEntry.java:8` — entry record (id, type, namespace, tenancyId, metadata, health, ttl)
- `platform-api/Relationship.java:5` — relationship record (sourceId, targetId, type)
- `platform-registry-inmem-core/InMemoryRegistryService.java:18` — test implementation
- `qhorus/RegistryBackedInstanceManager.java:28` — reference migration pattern
- Protocol `PP-20260612-d6e7ec` — CDI exclude-types sync rule
- [GitHub #268](https://github.com/casehubio/claudony/issues/268) — focal issue
- [GitHub #267](https://github.com/casehubio/claudony/issues/267) — parent epic
