# Ops View — Fleet-Wide Visibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #271 — feat: ops view — fleet-wide visibility for UI shells (Stage 3c)
**Issue group:** #271

**Goal:** Add a fleet-wide ops view API that aggregates registry, pool, peer, app, and mesh state into a single typed snapshot, with SSE for live updates.

**Architecture:** New `OpsService` composes from 5 existing services to build an `OpsSnapshot` record tree. `ClaudonyOpsApi` `@McpDomain` exposes REST endpoints. `OpsEventBus` observes `RegistryEvent` CDI events and pushes snapshot updates to SSE subscribers via `OpsEventsResource`.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, Smallrye Mutiny (SSE), CDI async observers, RestAssured (tests)

## Global Constraints

- Java target: `release=21`, compile on Java 26 (`JAVA_HOME=$(/usr/libexec/java_home -v 26)`)
- All new classes in `claudony-app` module
- Records package: `io.casehub.claudony.server.ops`
- API class package: `io.casehub.claudony.server.api`
- Tests run with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=<TestClass>`
- `@QuarkusTest` integration tests use `@TestSecurity(user = "test", roles = "user")`
- Docker must be running (PostgreSQL Dev Services)
- `quarkus.http.test-port=0` (random port — use RestAssured defaults, not hardcoded URLs)

---

## Batch 1: Data model + OpsService + unit tests

### Task 1: Data model records + OpsService + OpsServiceTest

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/ops/OpsSnapshot.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/FleetHealth.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/PoolOpsView.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/PeerOpsView.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/AppOpsView.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/MeshOpsView.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/ChannelOpsView.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/TopologyEdge.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/OpsService.java`
- Test: `app/src/test/java/io/casehub/claudony/server/ops/OpsServiceTest.java`

**Interfaces:**
- Consumes: `RegistryService.discover(RegistryQuery)`, `RegistryService.relationships(String)`, `PoolService.listPools()`, `PeerRegistry.getAllPeers()`, `ApplicationRegistrationService.list()`, `QhorusDashboardService.listChannels()`, `QhorusDashboardService.listInstances()`
- Produces: `OpsService.snapshot() → OpsSnapshot`, `OpsService.health() → FleetHealth`, `OpsService.buildSnapshotJson() → String`

- [ ] **Step 1: Write OpsServiceTest — snapshot assembly**

```java
package io.casehub.claudony.server.ops;

import io.casehub.claudony.casehub.fleet.AgentPoolHealth;
import io.casehub.claudony.casehub.fleet.AgentPoolStatus;
import io.casehub.claudony.casehub.fleet.ApplicationInfo;
import io.casehub.claudony.casehub.fleet.ApplicationRegistrationService;
import io.casehub.claudony.server.fleet.CircuitState;
import io.casehub.claudony.server.fleet.PeerHealth;
import io.casehub.claudony.server.fleet.PeerRecord;
import io.casehub.claudony.server.fleet.PeerRegistry;
import io.casehub.claudony.server.fleet.PoolService;
import io.casehub.claudony.server.fleet.PoolSummary;
import io.casehub.claudony.server.fleet.DiscoverySource;
import io.casehub.claudony.server.fleet.TerminalMode;
import io.casehub.platform.api.registry.HealthStatus;
import io.casehub.platform.api.registry.RegistryEntry;
import io.casehub.platform.api.registry.RegistryQuery;
import io.casehub.platform.api.registry.RegistryService;
import io.casehub.platform.api.registry.Relationship;
import io.casehub.qhorus.api.channel.ChannelDetail;
import io.casehub.qhorus.api.instance.InstanceInfo;
import io.casehub.qhorus.runtime.dashboard.QhorusDashboardService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class OpsServiceTest {

    private OpsService service;
    private RegistryService registryService;
    private PoolService poolService;
    private PeerRegistry peerRegistry;
    private ApplicationRegistrationService appService;
    private QhorusDashboardService dashboard;

    @BeforeEach
    void setUp() {
        registryService = mock(RegistryService.class);
        poolService = mock(PoolService.class);
        peerRegistry = mock(PeerRegistry.class);
        appService = mock(ApplicationRegistrationService.class);
        dashboard = mock(QhorusDashboardService.class);
        service = new OpsService(registryService, poolService, peerRegistry, appService, dashboard);
    }

    @Test
    void snapshotAssemblesAllSources() {
        when(registryService.discover(any())).thenReturn(List.of(
                new RegistryEntry("pool-default", "pool", "claudony", "default",
                        Map.of(), Instant.now(), Instant.now(), Duration.ofMinutes(5), HealthStatus.HEALTHY, null)
        ));
        when(registryService.relationships("pool-default")).thenReturn(List.of(
                new Relationship("app-scaffold", "pool-default", "consumes")
        ));
        when(poolService.listPools()).thenReturn(List.of(
                new PoolSummary("default", new AgentPoolStatus(0, 5, 2, 3, 5, AgentPoolHealth.HEALTHY), "target-tracking")
        ));
        when(peerRegistry.getAllPeers()).thenReturn(List.of(
                new PeerRecord("peer-1", "http://mac-mini:7777", "Mac Mini",
                        DiscoverySource.CONFIG, TerminalMode.PROXY, PeerHealth.UP, CircuitState.CLOSED,
                        Instant.now(), false, 3)
        ));
        when(appService.list()).thenReturn(List.of(
                new ApplicationInfo("scaffold", "Scaffold", "Build tool", HealthStatus.HEALTHY, Instant.now(), List.of())
        ));
        when(dashboard.listChannels()).thenReturn(List.of(
                new ChannelDetail(UUID.randomUUID(), "team/general", "General", null,
                        null, 42, null, false, null, null, null, null, null, null, null, null, null, null)
        ));
        when(dashboard.listInstances()).thenReturn(List.of(
                new InstanceInfo("inst-1", "worker", "ONLINE", List.of("code"), Instant.now().toString(), false)
        ));

        var snapshot = service.snapshot();

        assertNotNull(snapshot);
        assertNotNull(snapshot.timestamp());
        assertEquals(1, snapshot.pools().size());
        assertEquals("default", snapshot.pools().get(0).name());
        assertEquals(0.4, snapshot.pools().get(0).fillRatio(), 0.01);
        assertEquals(1, snapshot.peers().size());
        assertEquals("Mac Mini", snapshot.peers().get(0).name());
        assertEquals(1, snapshot.apps().size());
        assertEquals("scaffold", snapshot.apps().get(0).name());
        assertEquals(1, snapshot.mesh().totalChannels());
        assertEquals(1, snapshot.mesh().totalInstances());
        assertEquals(42, snapshot.mesh().channels().get(0).messageCount());
        assertEquals(1, snapshot.topology().size());
        assertEquals("consumes", snapshot.topology().get(0).type());
    }

    @Test
    void healthCountsFromRegistryEntries() {
        when(registryService.discover(any())).thenReturn(List.of(
                new RegistryEntry("a", "pool", "ns", "t", Map.of(), Instant.now(), Instant.now(), Duration.ofMinutes(5), HealthStatus.HEALTHY, null),
                new RegistryEntry("b", "peer", "ns", "t", Map.of(), Instant.now(), Instant.now(), Duration.ofMinutes(5), HealthStatus.DEGRADED, null),
                new RegistryEntry("c", "app", "ns", "t", Map.of(), Instant.now(), Instant.now(), Duration.ofMinutes(5), HealthStatus.DOWN, null),
                new RegistryEntry("d", "pool", "ns", "t", Map.of(), Instant.now(), Instant.now(), Duration.ofMinutes(5), HealthStatus.HEALTHY, Instant.now())
        ));

        var health = service.health();

        assertEquals(4, health.totalEntities());
        assertEquals(2, health.healthy());
        assertEquals(1, health.degraded());
        assertEquals(1, health.down());
        assertEquals(1, health.orphaned());
    }

    @Test
    void fillRatioZeroWhenMaxIsZero() {
        when(registryService.discover(any())).thenReturn(List.of());
        when(poolService.listPools()).thenReturn(List.of(
                new PoolSummary("empty", new AgentPoolStatus(0, 0, 0, 0, 0, AgentPoolHealth.HEALTHY), "none")
        ));
        when(peerRegistry.getAllPeers()).thenReturn(List.of());
        when(appService.list()).thenReturn(List.of());
        when(dashboard.listChannels()).thenReturn(List.of());
        when(dashboard.listInstances()).thenReturn(List.of());

        var snapshot = service.snapshot();
        assertEquals(0.0, snapshot.pools().get(0).fillRatio());
    }

    @Test
    void buildSnapshotJsonReturnsValidJson() {
        when(registryService.discover(any())).thenReturn(List.of());
        when(poolService.listPools()).thenReturn(List.of());
        when(peerRegistry.getAllPeers()).thenReturn(List.of());
        when(appService.list()).thenReturn(List.of());
        when(dashboard.listChannels()).thenReturn(List.of());
        when(dashboard.listInstances()).thenReturn(List.of());

        var json = service.buildSnapshotJson();
        assertNotNull(json);
        assertTrue(json.startsWith("{"));
        assertTrue(json.contains("\"health\""));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=OpsServiceTest`
Expected: FAIL — `OpsService` class not found

- [ ] **Step 3: Create all record types**

Create `OpsSnapshot.java`:
```java
package io.casehub.claudony.server.ops;

import java.time.Instant;
import java.util.List;

public record OpsSnapshot(
        FleetHealth health,
        List<PoolOpsView> pools,
        List<PeerOpsView> peers,
        List<AppOpsView> apps,
        MeshOpsView mesh,
        List<TopologyEdge> topology,
        Instant timestamp
) {}
```

Create `FleetHealth.java`:
```java
package io.casehub.claudony.server.ops;

public record FleetHealth(int totalEntities, int healthy, int degraded, int down, int orphaned) {}
```

Create `PoolOpsView.java`:
```java
package io.casehub.claudony.server.ops;

public record PoolOpsView(String name, String scalingType, int active, int idle, int max, double fillRatio, String healthStatus) {}
```

Create `PeerOpsView.java`:
```java
package io.casehub.claudony.server.ops;

public record PeerOpsView(String id, String name, String url, String health, String circuitState, int sessionCount, boolean stale) {}
```

Create `AppOpsView.java`:
```java
package io.casehub.claudony.server.ops;

public record AppOpsView(String name, String displayName, String health, int poolCount) {}
```

Create `MeshOpsView.java`:
```java
package io.casehub.claudony.server.ops;

import java.util.List;

public record MeshOpsView(int totalChannels, int totalInstances, List<ChannelOpsView> channels) {}
```

Create `ChannelOpsView.java`:
```java
package io.casehub.claudony.server.ops;

public record ChannelOpsView(String name, String purpose, long messageCount) {}
```

Create `TopologyEdge.java`:
```java
package io.casehub.claudony.server.ops;

public record TopologyEdge(String sourceId, String targetId, String type) {}
```

- [ ] **Step 4: Implement OpsService**

```java
package io.casehub.claudony.server.ops;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.claudony.casehub.fleet.ApplicationRegistrationService;
import io.casehub.claudony.server.fleet.PeerRegistry;
import io.casehub.claudony.server.fleet.PoolService;
import io.casehub.platform.api.registry.HealthStatus;
import io.casehub.platform.api.registry.RegistryQuery;
import io.casehub.platform.api.registry.RegistryService;
import io.casehub.platform.api.registry.Relationship;
import io.casehub.qhorus.runtime.dashboard.QhorusDashboardService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.List;

@ApplicationScoped
public class OpsService {

    private final RegistryService registryService;
    private final PoolService poolService;
    private final PeerRegistry peerRegistry;
    private final ApplicationRegistrationService appService;
    private final QhorusDashboardService dashboard;

    private static final ObjectMapper MAPPER = new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

    @Inject
    public OpsService(RegistryService registryService, PoolService poolService,
                      PeerRegistry peerRegistry, ApplicationRegistrationService appService,
                      QhorusDashboardService dashboard) {
        this.registryService = registryService;
        this.poolService = poolService;
        this.peerRegistry = peerRegistry;
        this.appService = appService;
        this.dashboard = dashboard;
    }

    public OpsSnapshot snapshot() {
        var fleetHealth = health();
        var pools = poolService.listPools().stream()
                .map(p -> new PoolOpsView(
                        p.name(), p.scalingType(),
                        p.status().active(), p.status().idle(), p.status().max(),
                        p.status().max() > 0 ? (double) p.status().active() / p.status().max() : 0.0,
                        p.status().health().name()))
                .toList();
        var peers = peerRegistry.getAllPeers().stream()
                .map(p -> new PeerOpsView(
                        p.id(), p.name(), p.url(),
                        p.health().name(), p.circuitState().name(),
                        p.sessionCount(), p.stale()))
                .toList();
        var apps = appService.list().stream()
                .map(a -> new AppOpsView(
                        a.name(), a.displayName(),
                        a.health().name(), a.pools().size()))
                .toList();
        var channels = dashboard.listChannels().stream()
                .map(ch -> new ChannelOpsView(ch.name(), ch.description(), ch.messageCount()))
                .toList();
        var instances = dashboard.listInstances();
        var mesh = new MeshOpsView(channels.size(), instances.size(), channels);
        var entries = registryService.discover(new RegistryQuery("default", null, null));
        var topology = entries.stream()
                .flatMap(e -> registryService.relationships(e.id()).stream())
                .distinct()
                .map(r -> new TopologyEdge(r.sourceId(), r.targetId(), r.type()))
                .toList();
        return new OpsSnapshot(fleetHealth, pools, peers, apps, mesh, topology, Instant.now());
    }

    public FleetHealth health() {
        var entries = registryService.discover(new RegistryQuery("default", null, null));
        int healthy = 0, degraded = 0, down = 0, orphaned = 0;
        for (var e : entries) {
            if (e.isOrphaned()) orphaned++;
            switch (e.health()) {
                case HEALTHY -> healthy++;
                case DEGRADED -> degraded++;
                case DOWN -> down++;
            }
        }
        return new FleetHealth(entries.size(), healthy, degraded, down, orphaned);
    }

    String buildSnapshotJson() {
        try {
            return MAPPER.writeValueAsString(snapshot());
        } catch (Exception e) {
            return "{}";
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=OpsServiceTest`
Expected: 4 tests PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/ops/ app/src/test/java/io/casehub/claudony/server/ops/OpsServiceTest.java
git commit -m "feat(#271): add OpsService with typed snapshot records

OpsSnapshot, FleetHealth, PoolOpsView, PeerOpsView, AppOpsView,
MeshOpsView, ChannelOpsView, TopologyEdge records.
OpsService composes from RegistryService, PoolService, PeerRegistry,
ApplicationRegistrationService, QhorusDashboardService.
4 unit tests: snapshot assembly, health counts, zero fill ratio, JSON.

Refs #271"
```

---

## Batch 2: OpsEventBus + unit tests

### Task 2: OpsEventBus + OpsEventBusTest

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/ops/OpsEventBus.java`
- Test: `app/src/test/java/io/casehub/claudony/server/ops/OpsEventBusTest.java`

**Interfaces:**
- Consumes: `OpsService.buildSnapshotJson()` (passed as `Supplier<String>`)
- Produces: `OpsEventBus.subscribe(Supplier<String>) → Multi<String>`, `OpsEventBus.emit()`, `OpsEventBus.subscriberCount() → int`

- [ ] **Step 1: Write OpsEventBusTest**

```java
package io.casehub.claudony.server.ops;

import io.smallrye.mutiny.helpers.test.AssertSubscriber;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class OpsEventBusTest {

    private OpsEventBus bus;

    @BeforeEach
    void setUp() {
        bus = new OpsEventBus();
    }

    @Test
    void subscribeReceivesInitialSnapshot() {
        var subscriber = bus.subscribe(() -> "{\"initial\":true}")
                .subscribe().withSubscriber(AssertSubscriber.create(1));
        subscriber.assertItems("{\"initial\":true}");
    }

    @Test
    void emitPushesToAllSubscribers() {
        var sub1 = bus.subscribe(() -> "snap")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        var sub2 = bus.subscribe(() -> "snap")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        bus.emit("{\"type\":\"update\"}");
        sub1.assertItems("snap", "{\"type\":\"update\"}");
        sub2.assertItems("snap", "{\"type\":\"update\"}");
    }

    @Test
    void emitNoOpWhenNoSubscribers() {
        bus.emit("{\"type\":\"update\"}");
        assertEquals(0, bus.subscriberCount());
    }

    @Test
    void subscriberCount() {
        assertEquals(0, bus.subscriberCount());
        var sub = bus.subscribe(() -> "snap")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        assertEquals(1, bus.subscriberCount());
        sub.cancel();
        assertEquals(0, bus.subscriberCount());
    }

    @Test
    void cancelCleansUp() {
        var sub = bus.subscribe(() -> "snap")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        sub.cancel();
        bus.emit("after-cancel");
        sub.assertItems("snap");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=OpsEventBusTest`
Expected: FAIL — `OpsEventBus` class not found

- [ ] **Step 3: Implement OpsEventBus**

```java
package io.casehub.claudony.server.ops;

import io.casehub.platform.api.registry.RegistryEvent;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.MultiEmitter;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Supplier;

@ApplicationScoped
public class OpsEventBus {

    private final List<MultiEmitter<String>> emitters = new CopyOnWriteArrayList<>();
    private volatile Supplier<String> snapshotFn;

    @SuppressWarnings("unchecked")
    public Multi<String> subscribe(Supplier<String> snapshotFn) {
        this.snapshotFn = snapshotFn;
        return Multi.createFrom().<String>emitter(emitter -> {
            emitter.emit(snapshotFn.get());
            MultiEmitter<String> typed = (MultiEmitter<String>) emitter;
            emitters.add(typed);
            emitter.onTermination(() -> emitters.remove(typed));
        });
    }

    public void emit(String payload) {
        emitters.forEach(e -> { if (!e.isCancelled()) e.emit(payload); });
    }

    void onRegistryEvent(@ObservesAsync RegistryEvent event) {
        if (snapshotFn != null && !emitters.isEmpty()) {
            emit(snapshotFn.get());
        }
    }

    public int subscriberCount() {
        return emitters.size();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=OpsEventBusTest`
Expected: 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/ops/OpsEventBus.java app/src/test/java/io/casehub/claudony/server/ops/OpsEventBusTest.java
git commit -m "feat(#271): add OpsEventBus with RegistryEvent CDI observer

SSE event bus for ops view. Observes RegistryEvent async to push
snapshot updates. Same pattern as PoolEventBus.
5 unit tests: initial snapshot, broadcast, no-op, count, cleanup.

Refs #271"
```

---

## Batch 3: REST + SSE endpoints + integration tests

### Task 3: ClaudonyOpsApi + OpsEventsResource + integration tests

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/api/ClaudonyOpsApi.java`
- Create: `app/src/main/java/io/casehub/claudony/server/ops/OpsEventsResource.java`
- Test: `app/src/test/java/io/casehub/claudony/server/api/ClaudonyOpsApiTest.java`
- Test: `app/src/test/java/io/casehub/claudony/server/ops/OpsEventsResourceTest.java`

**Interfaces:**
- Consumes: `OpsService.snapshot()`, `OpsService.health()`, `OpsService.buildSnapshotJson()`, `OpsEventBus.subscribe(Supplier<String>)`
- Produces: REST `GET /api/claudony/ops/snapshot`, `GET /api/claudony/ops/health`, SSE `GET /api/ops/events`

- [ ] **Step 1: Write ClaudonyOpsApiTest**

```java
package io.casehub.claudony.server.api;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class ClaudonyOpsApiTest {

    @Test
    void snapshotReturnsOpsSnapshot() {
        RestAssured.given()
                .when().get("/api/claudony/ops/snapshot")
                .then().statusCode(200)
                .body("health", notNullValue())
                .body("pools", instanceOf(java.util.List.class))
                .body("peers", instanceOf(java.util.List.class))
                .body("apps", instanceOf(java.util.List.class))
                .body("mesh", notNullValue())
                .body("topology", instanceOf(java.util.List.class))
                .body("timestamp", notNullValue());
    }

    @Test
    void healthReturnsFleetHealth() {
        RestAssured.given()
                .when().get("/api/claudony/ops/health")
                .then().statusCode(200)
                .body("totalEntities", greaterThanOrEqualTo(0))
                .body("healthy", greaterThanOrEqualTo(0))
                .body("degraded", greaterThanOrEqualTo(0))
                .body("down", greaterThanOrEqualTo(0))
                .body("orphaned", greaterThanOrEqualTo(0));
    }

    @Test
    void snapshotIncludesPoolData() {
        RestAssured.given()
                .when().get("/api/claudony/ops/snapshot")
                .then().statusCode(200)
                .body("pools.size()", greaterThanOrEqualTo(1))
                .body("pools[0].name", notNullValue())
                .body("pools[0].fillRatio", notNullValue());
    }
}
```

- [ ] **Step 2: Write OpsEventsResourceTest**

```java
package io.casehub.claudony.server.ops;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class OpsEventsResourceTest {

    @Test
    void sseEndpointReturnsEventStream() {
        RestAssured.given()
                .accept("text/event-stream")
                .when().get("/api/ops/events")
                .then().statusCode(200)
                .contentType(containsString("text/event-stream"));
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ClaudonyOpsApiTest,OpsEventsResourceTest`
Expected: FAIL — endpoints not found (404)

- [ ] **Step 4: Implement ClaudonyOpsApi**

```java
package io.casehub.claudony.server.api;

import io.casehub.claudony.server.ops.FleetHealth;
import io.casehub.claudony.server.ops.OpsService;
import io.casehub.claudony.server.ops.OpsSnapshot;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@McpDomain(value = "claudony/ops", app = "claudony",
           basePath = "/api/claudony/ops",
           summary = "Fleet-wide operational visibility — topology, health, utilisation")
@ApplicationScoped
public class ClaudonyOpsApi {

    @Inject OpsService opsService;

    @PlatformQuery("Full operational snapshot")
    @RestPath("/snapshot")
    public OpsSnapshot snapshot() {
        return opsService.snapshot();
    }

    @PlatformQuery("Fleet health summary")
    @RestPath("/health")
    public FleetHealth health() {
        return opsService.health();
    }
}
```

- [ ] **Step 5: Implement OpsEventsResource**

```java
package io.casehub.claudony.server.ops;

import io.casehub.platform.api.mcp.HandWrittenEndpoint;
import io.quarkus.security.Authenticated;
import io.smallrye.mutiny.Multi;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;

@HandWrittenEndpoint("SSE streaming — cannot be expressed as mcpDomain")
@Path("/api/ops/events")
@Authenticated
public class OpsEventsResource {

    @Inject OpsEventBus opsEventBus;
    @Inject OpsService opsService;

    @GET
    @Produces("text/event-stream")
    public Multi<String> events() {
        return opsEventBus.subscribe(() -> opsService.buildSnapshotJson());
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ClaudonyOpsApiTest,OpsEventsResourceTest`
Expected: 4 tests PASS

- [ ] **Step 7: Run full test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app`
Expected: All existing tests still pass, new tests pass

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/api/ClaudonyOpsApi.java app/src/main/java/io/casehub/claudony/server/ops/OpsEventsResource.java app/src/test/java/io/casehub/claudony/server/api/ClaudonyOpsApiTest.java app/src/test/java/io/casehub/claudony/server/ops/OpsEventsResourceTest.java
git commit -m "feat(#271): add ClaudonyOpsApi @McpDomain + OpsEventsResource SSE

REST: GET /api/claudony/ops/snapshot, GET /api/claudony/ops/health
SSE: GET /api/ops/events (full snapshot on connect, updates on RegistryEvent)
3 integration tests for REST, 1 for SSE content-type.

Refs #271"
```

---

## Batch 4: Documentation + CLAUDE.md update

### Task 4: Update CLAUDE.md and issue

**Files:**
- Modify: `CLAUDE.md` — add ops API URLs, update test count, document ClaudonyOpsApi in project structure
- Modify: issue #267 on GitHub — mark Stage 3c as done

**Interfaces:**
- Consumes: none (documentation only)
- Produces: updated project docs

- [ ] **Step 1: Add ops API URLs to CLAUDE.md Key URLs section**

Add after the existing Pool SSE events entry:
```
- Ops snapshot: `http://localhost:7777/api/claudony/ops/snapshot`
- Ops health: `http://localhost:7777/api/claudony/ops/health`
- Ops SSE events: `http://localhost:7777/api/ops/events` (text/event-stream)
```

- [ ] **Step 2: Update project structure in CLAUDE.md**

Add `ClaudonyOpsApi.java` to the `server/api/` listing.
Add `server/ops/` section with OpsService, OpsEventBus, OpsEventsResource, and the record types.

- [ ] **Step 3: Update test count in CLAUDE.md**

Count new tests: 4 unit (OpsServiceTest) + 5 unit (OpsEventBusTest) + 3 integration (ClaudonyOpsApiTest) + 1 integration (OpsEventsResourceTest) = 13 new tests. Update the baseline total.

- [ ] **Step 4: Update epic #267 — mark Stage 3c as Done**

```bash
gh issue view 267 --repo casehubio/claudony --json body -q '.body' > /tmp/epic-body.md
# Edit to change Stage 3c status from "Open" to "Done"
# gh issue edit 267 --repo casehubio/claudony --body-file /tmp/epic-body.md
```

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#271): add ops view API to CLAUDE.md

New endpoints, project structure, test count update.

Refs #271"
```

## References

- [2026-10-12-ops-view-fleet-visibility-design.md] — design spec this plan implements
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java` — aggregation pattern reference
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventBus.java` — SSE event bus pattern
- `app/src/test/java/io/casehub/claudony/server/fleet/PoolEventBusTest.java` — event bus test pattern
- `app/src/test/java/io/casehub/claudony/server/api/ClaudonyPoolApiTest.java` — integration test pattern
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventsResource.java` — SSE resource pattern
- GitHub #271 — focal issue
- GitHub #267 — parent epic
