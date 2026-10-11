# Application Registration + Namespace Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #269 — feat: application registration + namespace model — Stage 3a
**Issue group:** #269

**Goal:** Add application as a first-class registry entity with namespace-based isolation, declarative pool specs, and a @McpDomain API surface.

**Architecture:** `ApplicationRegistrationService` in claudony-casehub orchestrates registration across `RegistryService` and `AgentPoolDefinitionRegistry`. It creates an application entry (type="application"), creates pools with the app's namespace, and links them via "owns" relationships. `ClaudonyAppApi` in claudony-app exposes the service as REST + MCP + GraphQL via `@McpDomain`.

**Tech Stack:** Java 21, Quarkus 3.32.2, `casehub-platform-api` RegistryService SPI, `InMemoryRegistryService` for tests, @McpDomain annotation, AssertJ, RestAssured

## Global Constraints

- Java 21 API surface (compiled on Java 26 JVM)
- All tests run with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Use `InMemoryRegistryService` for all unit tests — no database
- Use `@McpDomain` for API classes — gives REST + MCP + GraphQL from one class
- `namespace` on `RegistryEntry` = owning application identity (D1)
- Relationship types: `"owns"` (app→pool), `"contains"` (pool→session, existing)
- Pool IDs: `{appName}/{agentId}` format
- Channel prefix: `app:{namespace}/{purpose}`
- All commits reference #269

---

## Batch 1: Model + Service (core domain)

### Task 1: Add namespace parameter to AgentPoolDefinitionRegistry

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistryTest.java`

**Interfaces:**
- Consumes: `RegistryService` (platform-api), `AgentPoolDefinition`
- Produces: `register(AgentPoolDefinition, String namespace)` — new overload; existing `register(AgentPoolDefinition)` defaults namespace to `"fleet"`

- [ ] **Step 1: Write failing test for namespace-aware registration**

```java
@Test
void register_withNamespace_usesProvidedNamespace() {
    var def = AgentPoolDefinition.builder()
            .agent("code-reviewer")
            .workingDir("/reviews")
            .build();

    registry.register(def, "scaffold");

    var entries = registryService.discover(new RegistryQuery("default", "pool", "scaffold"));
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).id()).isEqualTo("code-reviewer");
    assertThat(entries.get(0).namespace()).isEqualTo("scaffold");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolDefinitionRegistryTest#register_withNamespace_usesProvidedNamespace`
Expected: Compilation error — no `register(def, namespace)` overload

- [ ] **Step 3: Add namespace-aware register overload**

Add to `AgentPoolDefinitionRegistry`:

```java
public void register(AgentPoolDefinition definition, String namespace) {
    var prev = definitions.putIfAbsent(definition.agent().name(), definition);
    if (prev != null) {
        throw new IllegalStateException(
                "Pool definition already registered for agent: " + definition.agent().name());
    }
    registryService.register(toRegistryEntry(definition, namespace));
}
```

Update existing `register(AgentPoolDefinition)` to delegate:

```java
public void register(AgentPoolDefinition definition) {
    register(definition, "fleet");
}
```

Update `toRegistryEntry` to accept namespace:

```java
private RegistryEntry toRegistryEntry(AgentPoolDefinition def, String namespace) {
    var now = java.time.Instant.now();
    return new RegistryEntry(
            def.agent().name(), "pool", namespace, "default",
            java.util.Map.of(
                    "agentName", def.agent().name(),
                    "minActive", String.valueOf(def.pool().minActive()),
                    "maxActive", String.valueOf(def.pool().maxActive())
            ),
            now, now, java.time.Duration.ofHours(24), HealthStatus.HEALTHY);
}
```

Update all internal callers of `toRegistryEntry(def)` to `toRegistryEntry(def, "fleet")`.

- [ ] **Step 4: Write backward-compatibility test**

```java
@Test
void register_withoutNamespace_defaultsToFleet() {
    var def = AgentPoolDefinition.builder()
            .agent("reviewer")
            .workingDir("/work")
            .build();

    registry.register(def);

    var entries = registryService.discover(new RegistryQuery("default", "pool", "fleet"));
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).namespace()).isEqualTo("fleet");
}
```

- [ ] **Step 5: Run all AgentPoolDefinitionRegistryTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolDefinitionRegistryTest`
Expected: ALL PASS (existing tests use `register(def)` which now defaults to "fleet")

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistryTest.java
git commit -m "feat(#269): add namespace parameter to AgentPoolDefinitionRegistry

Existing register(def) defaults to namespace='fleet' for backward compatibility.
New register(def, namespace) overload allows app-scoped pool registration.

Refs #269"
```

### Task 2: Create ApplicationSpec and ApplicationInfo records

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationSpec.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationInfo.java`

**Interfaces:**
- Consumes: `HealthStatus` (platform-api), `PoolSummary` (claudony-app — but we create a local pool summary to avoid cross-module dependency)
- Produces: `ApplicationSpec(String name, String displayName, String description, List<PoolSpec> pools)` with nested `PoolSpec(String agentId, int minActive, int maxActive)`. `ApplicationInfo(String name, String displayName, String description, HealthStatus health, Instant registeredAt, List<AppPoolInfo> pools)` with nested `AppPoolInfo(String agentId, int minActive, int maxActive)`.

- [ ] **Step 1: Create ApplicationSpec record**

```java
package io.casehub.claudony.casehub.fleet;

import java.util.List;
import java.util.Objects;

public record ApplicationSpec(
        String name,
        String displayName,
        String description,
        List<PoolSpec> pools
) {
    public ApplicationSpec {
        Objects.requireNonNull(name, "name");
        if (name.isBlank()) throw new IllegalArgumentException("name must not be blank");
        Objects.requireNonNull(displayName, "displayName");
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }

    public record PoolSpec(
            String agentId,
            int minActive,
            int maxActive
    ) {
        public PoolSpec {
            Objects.requireNonNull(agentId, "agentId");
            if (agentId.isBlank()) throw new IllegalArgumentException("agentId must not be blank");
            if (minActive < 0) throw new IllegalArgumentException("minActive must be >= 0");
            if (maxActive < 1) throw new IllegalArgumentException("maxActive must be >= 1");
            if (maxActive < minActive) throw new IllegalArgumentException("maxActive must be >= minActive");
        }
    }
}
```

- [ ] **Step 2: Create ApplicationInfo record**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.platform.api.registry.HealthStatus;

import java.time.Instant;
import java.util.List;
import java.util.Objects;

public record ApplicationInfo(
        String name,
        String displayName,
        String description,
        HealthStatus health,
        Instant registeredAt,
        List<AppPoolInfo> pools
) {
    public ApplicationInfo {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(health, "health");
        Objects.requireNonNull(registeredAt, "registeredAt");
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }

    public record AppPoolInfo(
            String agentId,
            int minActive,
            int maxActive
    ) {}
}
```

- [ ] **Step 3: Compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl claudony-casehub -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationSpec.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationInfo.java
git commit -m "feat(#269): add ApplicationSpec and ApplicationInfo records

ApplicationSpec is the registration request (name, displayName, description, pools).
ApplicationInfo is the query response with health and pool summaries.

Refs #269"
```

### Task 3: Implement ApplicationRegistrationService with tests

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationService.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationServiceTest.java`

**Interfaces:**
- Consumes: `RegistryService`, `AgentPoolDefinitionRegistry`, `TenantContext`, `ApplicationSpec`, `ApplicationInfo`
- Produces: `register(ApplicationSpec)`, `deregister(String)`, `get(String)`, `list()`, `heartbeat(String)`

- [ ] **Step 1: Write first failing test — register creates app entry**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.claudony.server.TenantContext;
import io.casehub.platform.api.registry.RegistryQuery;
import io.casehub.platform.api.registry.RegistryService;
import io.casehub.platform.registry.memory.InMemoryRegistryService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ApplicationRegistrationServiceTest {

    private RegistryService registryService;
    private AgentPoolDefinitionRegistry poolRegistry;
    private ApplicationRegistrationService service;
    private final TenantContext tenantContext = () -> "default";

    @BeforeEach
    void setUp() {
        registryService = new InMemoryRegistryService(event -> {});
        poolRegistry = new AgentPoolDefinitionRegistry(registryService);
        service = new ApplicationRegistrationService(registryService, poolRegistry, tenantContext);
    }

    @Test
    void register_createsAppEntryWithCorrectTypeAndNamespace() {
        var spec = new ApplicationSpec("scaffold", "Scaffold", "Dev platform", List.of());

        service.register(spec);

        var entries = registryService.discover(new RegistryQuery("default", "application", "scaffold"));
        assertThat(entries).hasSize(1);
        assertThat(entries.get(0).id()).isEqualTo("scaffold");
        assertThat(entries.get(0).type()).isEqualTo("application");
        assertThat(entries.get(0).namespace()).isEqualTo("scaffold");
        assertThat(entries.get(0).metadata()).containsEntry("displayName", "Scaffold");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ApplicationRegistrationServiceTest#register_createsAppEntryWithCorrectTypeAndNamespace`
Expected: Compilation error — `ApplicationRegistrationService` does not exist

- [ ] **Step 3: Implement ApplicationRegistrationService skeleton**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.claudony.server.TenantContext;
import io.casehub.platform.api.registry.HealthStatus;
import io.casehub.platform.api.registry.RegistryEntry;
import io.casehub.platform.api.registry.RegistryQuery;
import io.casehub.platform.api.registry.RegistryService;
import io.casehub.platform.api.registry.Relationship;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;

@ApplicationScoped
public class ApplicationRegistrationService {

    private final RegistryService registryService;
    private final AgentPoolDefinitionRegistry poolRegistry;
    private final TenantContext tenantContext;

    @Inject
    public ApplicationRegistrationService(RegistryService registryService,
                                          AgentPoolDefinitionRegistry poolRegistry,
                                          TenantContext tenantContext) {
        this.registryService = registryService;
        this.poolRegistry = poolRegistry;
        this.tenantContext = tenantContext;
    }

    public ApplicationInfo register(ApplicationSpec spec) {
        var tenancyId = tenantContext.currentTenantId();
        var existing = registryService.resolve(spec.name());
        if (existing.isPresent()) {
            throw new IllegalStateException("Application already registered: " + spec.name());
        }

        var now = Instant.now();
        var appEntry = new RegistryEntry(
                spec.name(), "application", spec.name(), tenancyId,
                Map.of("displayName", spec.displayName(),
                       "description", spec.description() != null ? spec.description() : ""),
                now, now, Duration.ofHours(24), HealthStatus.HEALTHY);
        registryService.register(appEntry);

        var poolInfos = new java.util.ArrayList<ApplicationInfo.AppPoolInfo>();
        for (var poolSpec : spec.pools()) {
            var poolId = spec.name() + "/" + poolSpec.agentId();
            var def = AgentPoolDefinition.builder()
                    .agent(poolId)
                        .workingDir(null)
                    .pool()
                        .minActive(poolSpec.minActive())
                        .maxActive(poolSpec.maxActive())
                    .build();
            poolRegistry.register(def, spec.name());
            registryService.link(new Relationship(spec.name(), poolId, "owns"));
            poolInfos.add(new ApplicationInfo.AppPoolInfo(poolSpec.agentId(),
                    poolSpec.minActive(), poolSpec.maxActive()));
        }

        return new ApplicationInfo(spec.name(), spec.displayName(),
                spec.description(), HealthStatus.HEALTHY, now, poolInfos);
    }

    public void deregister(String appName) {
        var rels = registryService.relationships(appName);
        for (var rel : rels) {
            if ("owns".equals(rel.type()) && rel.sourceId().equals(appName)) {
                registryService.deregister(rel.targetId());
            }
        }
        registryService.deregister(appName);
    }

    public Optional<ApplicationInfo> get(String appName) {
        return registryService.resolve(appName)
                .filter(e -> "application".equals(e.type()))
                .map(this::toApplicationInfo);
    }

    public List<ApplicationInfo> list() {
        var tenancyId = tenantContext.currentTenantId();
        return registryService.discover(new RegistryQuery(tenancyId, "application", null))
                .stream()
                .map(this::toApplicationInfo)
                .toList();
    }

    public void heartbeat(String appName) {
        registryService.heartbeat(appName);
    }

    private ApplicationInfo toApplicationInfo(RegistryEntry entry) {
        var rels = registryService.relationships(entry.id());
        var poolInfos = rels.stream()
                .filter(r -> "owns".equals(r.type()) && r.sourceId().equals(entry.id()))
                .map(r -> registryService.resolve(r.targetId()))
                .flatMap(Optional::stream)
                .map(poolEntry -> {
                    var meta = poolEntry.metadata();
                    return new ApplicationInfo.AppPoolInfo(
                            meta.getOrDefault("agentName", poolEntry.id()),
                            Integer.parseInt(meta.getOrDefault("minActive", "0")),
                            Integer.parseInt(meta.getOrDefault("maxActive", "10")));
                })
                .toList();

        return new ApplicationInfo(
                entry.id(),
                entry.metadata().getOrDefault("displayName", entry.id()),
                entry.metadata().getOrDefault("description", ""),
                entry.health(),
                entry.registeredAt(),
                poolInfos);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ApplicationRegistrationServiceTest#register_createsAppEntryWithCorrectTypeAndNamespace`
Expected: PASS

- [ ] **Step 5: Add remaining tests**

Add to `ApplicationRegistrationServiceTest`:

```java
@Test
void register_createsPoolsWithAppNamespace() {
    var spec = new ApplicationSpec("scaffold", "Scaffold", "Dev platform",
            List.of(new ApplicationSpec.PoolSpec("code-review", 1, 5)));

    service.register(spec);

    var pools = registryService.discover(new RegistryQuery("default", "pool", "scaffold"));
    assertThat(pools).hasSize(1);
    assertThat(pools.get(0).id()).isEqualTo("scaffold/code-review");
    assertThat(pools.get(0).namespace()).isEqualTo("scaffold");
}

@Test
void register_linksAppToPoolsWithOwnsRelationship() {
    var spec = new ApplicationSpec("scaffold", "Scaffold", "Dev",
            List.of(new ApplicationSpec.PoolSpec("review", 0, 3)));

    service.register(spec);

    var rels = registryService.relationships("scaffold");
    assertThat(rels).hasSize(1);
    assertThat(rels.get(0).sourceId()).isEqualTo("scaffold");
    assertThat(rels.get(0).targetId()).isEqualTo("scaffold/review");
    assertThat(rels.get(0).type()).isEqualTo("owns");
}

@Test
void register_poolIdIncludesAppPrefix() {
    var spec = new ApplicationSpec("trellis", "Trellis", "IoT",
            List.of(new ApplicationSpec.PoolSpec("monitor", 0, 2)));

    service.register(spec);

    assertThat(poolRegistry.get("trellis/monitor")).isPresent();
}

@Test
void register_duplicateNameThrows() {
    var spec = new ApplicationSpec("myapp", "My App", "test", List.of());
    service.register(spec);

    assertThatThrownBy(() -> service.register(spec))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("myapp");
}

@Test
void deregister_removesAppAndPools() {
    var spec = new ApplicationSpec("scaffold", "Scaffold", "Dev",
            List.of(new ApplicationSpec.PoolSpec("review", 0, 3)));
    service.register(spec);

    service.deregister("scaffold");

    assertThat(registryService.resolve("scaffold")).isEmpty();
    assertThat(registryService.resolve("scaffold/review")).isEmpty();
}

@Test
void deregister_unknownAppNoOp() {
    service.deregister("nonexistent");
    // no exception
}

@Test
void get_returnsAppWithPools() {
    var spec = new ApplicationSpec("scaffold", "Scaffold", "Dev",
            List.of(new ApplicationSpec.PoolSpec("review", 1, 5)));
    service.register(spec);

    var result = service.get("scaffold");

    assertThat(result).isPresent();
    assertThat(result.get().name()).isEqualTo("scaffold");
    assertThat(result.get().displayName()).isEqualTo("Scaffold");
    assertThat(result.get().pools()).hasSize(1);
    assertThat(result.get().pools().get(0).agentId()).isEqualTo("scaffold/review");
}

@Test
void list_returnsAllApps() {
    service.register(new ApplicationSpec("app1", "App 1", "first", List.of()));
    service.register(new ApplicationSpec("app2", "App 2", "second", List.of()));

    var apps = service.list();

    assertThat(apps).hasSize(2);
    assertThat(apps).extracting(ApplicationInfo::name).containsExactlyInAnyOrder("app1", "app2");
}

@Test
void heartbeat_updatesAppEntry() {
    service.register(new ApplicationSpec("myapp", "My App", "test", List.of()));

    service.heartbeat("myapp");

    var entry = registryService.resolve("myapp");
    assertThat(entry).isPresent();
}

@Test
void register_multipleAppsIsolatedByNamespace() {
    service.register(new ApplicationSpec("app-a", "App A", "",
            List.of(new ApplicationSpec.PoolSpec("worker", 0, 5))));
    service.register(new ApplicationSpec("app-b", "App B", "",
            List.of(new ApplicationSpec.PoolSpec("worker", 0, 3))));

    var appAPools = registryService.discover(new RegistryQuery("default", "pool", "app-a"));
    var appBPools = registryService.discover(new RegistryQuery("default", "pool", "app-b"));

    assertThat(appAPools).hasSize(1);
    assertThat(appBPools).hasSize(1);
    assertThat(appAPools.get(0).id()).isEqualTo("app-a/worker");
    assertThat(appBPools.get(0).id()).isEqualTo("app-b/worker");
}
```

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=ApplicationRegistrationServiceTest`
Expected: ALL 11 PASS

- [ ] **Step 7: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationService.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationServiceTest.java
git commit -m "feat(#269): implement ApplicationRegistrationService

Orchestrates app registration across RegistryService and AgentPoolDefinitionRegistry.
Creates app entry (type=application, namespace=appName), per-app pools with namespace
scoping, and owns relationships for lifecycle cascade.

Refs #269"
```

---

## Batch 2: @McpDomain API + full test pass

### Task 4: Create ClaudonyAppApi @McpDomain class with tests

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/api/ClaudonyAppApi.java`
- Create: `app/src/test/java/io/casehub/claudony/server/api/ClaudonyAppApiTest.java`

**Interfaces:**
- Consumes: `ApplicationRegistrationService`, `ApplicationSpec`, `ApplicationInfo`
- Produces: REST endpoints at `/api/claudony/apps` + MCP tools in `claudony/apps` domain + GraphQL

- [ ] **Step 1: Write failing test — list apps returns empty**

```java
package io.casehub.claudony.server.api;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class ClaudonyAppApiTest {

    @Test
    void listApps_empty() {
        RestAssured.given()
                .when().get("/api/claudony/apps")
                .then().statusCode(200)
                .body("size()", is(0));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ClaudonyAppApiTest#listApps_empty`
Expected: 404 — endpoint does not exist

- [ ] **Step 3: Implement ClaudonyAppApi**

```java
package io.casehub.claudony.server.api;

import io.casehub.claudony.casehub.fleet.ApplicationInfo;
import io.casehub.claudony.casehub.fleet.ApplicationRegistrationService;
import io.casehub.claudony.casehub.fleet.ApplicationSpec;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.WebApplicationException;
import jakarta.ws.rs.core.Response;

import java.util.List;

@McpDomain(value = "claudony/apps", app = "claudony",
           basePath = "/api/claudony/apps",
           summary = "Application registration — namespace, pools, lifecycle")
@ApplicationScoped
public class ClaudonyAppApi {

    @Inject ApplicationRegistrationService appService;

    @PlatformQuery("List registered applications")
    @RestPath("/")
    public List<ApplicationInfo> listApps() {
        return appService.list();
    }

    @PlatformQuery("Get application details")
    @RestPath("/{name}")
    public ApplicationInfo getApp(@PathParam String name) {
        return appService.get(name)
                .orElseThrow(() -> new WebApplicationException(Response.Status.NOT_FOUND));
    }

    @PlatformMutation("Register an application")
    @RestPath("/")
    public ApplicationInfo registerApp(ApplicationSpec spec) {
        return appService.register(spec);
    }

    @PlatformMutation("Deregister an application")
    @RestPath("/{name}")
    public void deregisterApp(@PathParam String name) {
        appService.deregister(name);
    }

    @PlatformMutation("Heartbeat an application")
    @RestPath("/{name}/heartbeat")
    public void heartbeat(@PathParam String name) {
        appService.heartbeat(name);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ClaudonyAppApiTest#listApps_empty`
Expected: PASS

- [ ] **Step 5: Add remaining API tests**

```java
@Test
void registerApp_success() {
    var body = """
            {
                "name": "test-app-%s",
                "displayName": "Test App",
                "description": "A test application",
                "pools": [
                    {"agentId": "reviewer", "minActive": 0, "maxActive": 3}
                ]
            }
            """.formatted(java.util.UUID.randomUUID().toString().substring(0, 8));

    RestAssured.given()
            .contentType(ContentType.JSON)
            .body(body)
            .when().post("/api/claudony/apps")
            .then().statusCode(200)
            .body("name", startsWith("test-app-"))
            .body("displayName", is("Test App"))
            .body("health", is("HEALTHY"))
            .body("pools.size()", is(1));
}

@Test
void getApp_notFound() {
    RestAssured.given()
            .when().get("/api/claudony/apps/nonexistent-app")
            .then().statusCode(404);
}

@Test
void heartbeat_success() {
    var name = "heartbeat-test-" + java.util.UUID.randomUUID().toString().substring(0, 8);
    RestAssured.given()
            .contentType(ContentType.JSON)
            .body("{\"name\": \"%s\", \"displayName\": \"HB Test\", \"description\": \"\", \"pools\": []}".formatted(name))
            .when().post("/api/claudony/apps")
            .then().statusCode(200);

    RestAssured.given()
            .contentType(ContentType.JSON)
            .when().post("/api/claudony/apps/" + name + "/heartbeat")
            .then().statusCode(200);
}
```

- [ ] **Step 6: Run all ClaudonyAppApiTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=ClaudonyAppApiTest`
Expected: ALL PASS

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: ALL modules pass — no regressions in existing tests

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/api/ClaudonyAppApi.java app/src/test/java/io/casehub/claudony/server/api/ClaudonyAppApiTest.java
git commit -m "feat(#269): add ClaudonyAppApi @McpDomain for application registration

REST + MCP + GraphQL surface for app registration, deregistration,
listing, detail, and heartbeat at /api/claudony/apps.

Refs #269"
```

---

## References

- [2026-10-11-application-registration-namespace-design.md] — design spec this plan implements
- `RegistryEntry.java` (platform-api) — registry entity model
- `RegistryService.java` (platform-api) — registry SPI
- `InMemoryRegistryService.java` (platform-registry-inmem-core) — test backend
- `AgentPoolDefinitionRegistry.java:25-95` (claudony-casehub) — existing pool registration
- `AgentPoolDefinitionRegistryTest.java` (claudony-casehub) — test pattern reference
- `ClaudonyPoolApi.java` (claudony-app) — @McpDomain pattern reference
- `ClaudonyPoolApiTest.java` (claudony-app) — @QuarkusTest API test pattern
- `TenantContext.java` (claudony-core) — tenant resolution interface
- casehubio/claudony#269 — focal issue
- casehubio/claudony#267 — parent epic
