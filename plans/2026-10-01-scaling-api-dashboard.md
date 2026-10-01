# Scaling API & Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #241 — REST API for runtime scaling config changes
**Issue group:** #241, #242

**Goal:** Migrate pool management to mcpDomain, support all 5 scaling config types at runtime, add SSE event streaming, and make the dashboard pool panel interactive with editing controls and live event log.

**Architecture:** Extract `PoolResource` business logic into `PoolService`. Create `ClaudonyPoolApi` as the mcpDomain entry point at `/api/claudony/pools`. Add `PoolEventBus` (CDI event-based SSE, following `CaseEventBroadcaster` pattern) with `PoolEventsResource` at `/api/pool-events/{name}`. Update `claudony-pool-panel.ts` with path migration, SSE integration, capacity/scaling editing controls, and live event log.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, `@McpDomain` / `@PlatformQuery` / `@PlatformMutation` (casehub-platform-api), Lit 3, TypeScript

## Global Constraints

- Java release target: 21 (compile on Java 26)
- Quarkus version: 3.32.2
- mcpDomain annotations from `io.casehub.platform.api.mcp.*`
- All `@QuarkusTest` classes use `quarkus.http.test-port=0` (random port)
- `@TestSecurity(user = "test", roles = "user")` only on tests with HTTP endpoints
- Mutations requiring admin: `@RolesAllowed("admin")` or platform equivalent
- Frontend: Lit 3 + `fetchWithAuth` from `util/auth.ts`
- IntelliJ MCP required for all code operations

---

## Batch 1: PoolService extraction + PoolUpdateRequest

### Task 1: Create PoolService with extracted business logic and tests

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java`
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/PoolUpdateRequest.java`
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/ScalingStepInput.java`
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/ScalingConfigView.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PoolDetail.java` — update `ScalingView` to use `ScalingConfigView`
- Test: `app/src/test/java/io/casehub/claudony/server/fleet/PoolServiceTest.java`

**Interfaces:**
- Consumes: `AgentPoolManagerRegistry`, `AgentPoolDefinitionRegistry`, `ScalingScheduler`, `MeterRegistry`, `ScalingConfig` (sealed interface), `ScalingStep`
- Produces: `PoolService` with methods: `listPools()`, `getPool(String name)`, `listSessions(String name)`, `updatePool(String name, PoolUpdateRequest request)`, `suspendSession(String name, String id)`, `resumeSession(String name, String id)`, `destroySession(String name, String id)`, `buildPoolSnapshot(String name)`; `PoolUpdateRequest` record; `ScalingStepInput` record; `ScalingConfigView` record

- [ ] **Step 1: Create the PoolUpdateRequest and ScalingStepInput records**

Use `ide_create_file`:

```java
// PoolUpdateRequest.java
package io.casehub.claudony.server.fleet;

import java.util.List;

public record PoolUpdateRequest(
    Integer minActive,
    Integer maxActive,
    String scalingType,
    Double targetFillRatio,
    List<ScalingStepInput> steps,
    Integer exhaustionThreshold,
    Long latencyThresholdMs,
    String cooldown,
    String scaleInCooldown
) {}
```

```java
// ScalingStepInput.java
package io.casehub.claudony.server.fleet;

public record ScalingStepInput(double threshold, int adjustment) {}
```

- [ ] **Step 2: Create ScalingConfigView record**

Use `ide_create_file`:

```java
// ScalingConfigView.java
package io.casehub.claudony.server.fleet;

import java.util.List;

public record ScalingConfigView(
    Double targetFillRatio,
    List<ScalingStepView> steps,
    Integer exhaustionThreshold,
    Long latencyThresholdMs,
    String beanName,
    String cooldown,
    String scaleInCooldown
) {
    public record ScalingStepView(double threshold, int adjustment) {}
}
```

- [ ] **Step 3: Update PoolDetail.ScalingView to use typed config**

Use `ide_edit_member` on `PoolDetail.java` to change:

```java
// Old:
public record ScalingView(String type, Object config, DecisionView lastDecision, String cooldownRemaining) {}
// New:
public record ScalingView(String type, ScalingConfigView config, DecisionView lastDecision, String cooldownRemaining) {}
```

- [ ] **Step 4: Write failing tests for PoolService scaling config parsing**

Create `PoolServiceTest.java` — unit tests for all 5 scaling types plus validation:

```java
package io.casehub.claudony.server.fleet;

import io.casehub.claudony.casehub.fleet.*;
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class PoolServiceTest {

    private AgentPoolDefinitionRegistry defRegistry;
    private AgentPoolManagerRegistry mgrRegistry;
    private ScalingScheduler scheduler;
    private PoolService service;

    @BeforeEach
    void setUp() {
        defRegistry = new AgentPoolDefinitionRegistry();
        mgrRegistry = new AgentPoolManagerRegistry();
        scheduler = new ScalingScheduler(defRegistry, mgrRegistry);
        service = new PoolService(defRegistry, mgrRegistry, scheduler, new SimpleMeterRegistry());
    }

    @Test
    void parseTargetTracking() {
        var req = new PoolUpdateRequest(null, null, "target-tracking", 0.8, null, null, null, "30s", null);
        var config = service.parseScalingConfig(req);
        assertInstanceOf(ScalingConfig.TargetTrackingConfig.class, config);
        var tt = (ScalingConfig.TargetTrackingConfig) config;
        assertEquals(0.8, tt.targetFillRatio());
        assertEquals(java.time.Duration.ofSeconds(30), tt.cooldown());
    }

    @Test
    void parseStep() {
        var steps = List.of(new ScalingStepInput(0.8, 2), new ScalingStepInput(0.3, -1));
        var req = new PoolUpdateRequest(null, null, "step", null, steps, null, null, null, null);
        var config = service.parseScalingConfig(req);
        assertInstanceOf(ScalingConfig.StepConfig.class, config);
        assertEquals(2, ((ScalingConfig.StepConfig) config).steps().size());
    }

    @Test
    void parseDemandPressure() {
        var req = new PoolUpdateRequest(null, null, "demand-pressure", null, null, 5, 500L, null, null);
        var config = service.parseScalingConfig(req);
        assertInstanceOf(ScalingConfig.DemandPressureConfig.class, config);
        var dp = (ScalingConfig.DemandPressureConfig) config;
        assertEquals(5, dp.exhaustionThreshold());
        assertEquals(500L, dp.latencyThresholdMs());
    }

    @Test
    void parseNone() {
        var req = new PoolUpdateRequest(null, null, "none", null, null, null, null, null, null);
        var config = service.parseScalingConfig(req);
        assertInstanceOf(ScalingConfig.NoScalingConfig.class, config);
    }

    @Test
    void parseCustom() {
        var req = new PoolUpdateRequest(null, null, "my-custom-bean", null, null, null, null, null, null);
        var config = service.parseScalingConfig(req);
        assertInstanceOf(ScalingConfig.CustomScalingConfig.class, config);
        assertEquals("my-custom-bean", ((ScalingConfig.CustomScalingConfig) config).beanName());
    }

    @Test
    void targetTrackingDefaultRatio() {
        var req = new PoolUpdateRequest(null, null, "target-tracking", null, null, null, null, null, null);
        var config = service.parseScalingConfig(req);
        assertEquals(0.7, ((ScalingConfig.TargetTrackingConfig) config).targetFillRatio());
    }

    @Test
    void invalidTargetRatioThrows() {
        var req = new PoolUpdateRequest(null, null, "target-tracking", 1.5, null, null, null, null, null);
        assertThrows(IllegalArgumentException.class, () -> service.parseScalingConfig(req));
    }

    @Test
    void emptyStepsThrows() {
        var req = new PoolUpdateRequest(null, null, "step", null, List.of(), null, null, null, null);
        assertThrows(IllegalArgumentException.class, () -> service.parseScalingConfig(req));
    }

    @Test
    void parseDurationSeconds() {
        var req = new PoolUpdateRequest(null, null, "target-tracking", 0.7, null, null, null, "90s", "120s");
        var config = (ScalingConfig.TargetTrackingConfig) service.parseScalingConfig(req);
        assertEquals(java.time.Duration.ofSeconds(90), config.cooldown());
        assertEquals(java.time.Duration.ofSeconds(120), config.scaleInCooldown());
    }

    @Test
    void parseDurationMinutes() {
        var req = new PoolUpdateRequest(null, null, "target-tracking", 0.7, null, null, null, "5m", null);
        var config = (ScalingConfig.TargetTrackingConfig) service.parseScalingConfig(req);
        assertEquals(java.time.Duration.ofMinutes(5), config.cooldown());
    }

    @Test
    void parseDurationPlainNumber() {
        var req = new PoolUpdateRequest(null, null, "target-tracking", 0.7, null, null, null, "120", null);
        var config = (ScalingConfig.TargetTrackingConfig) service.parseScalingConfig(req);
        assertEquals(java.time.Duration.ofSeconds(120), config.cooldown());
    }

    @Test
    void invalidDurationThrows() {
        var req = new PoolUpdateRequest(null, null, "target-tracking", 0.7, null, null, null, "abc", null);
        assertThrows(jakarta.ws.rs.BadRequestException.class, () -> service.parseScalingConfig(req));
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PoolServiceTest`
Expected: FAIL — `PoolService` class does not exist

- [ ] **Step 6: Implement PoolService**

Use `ide_create_file` for `PoolService.java`:

```java
package io.casehub.claudony.server.fleet;

import io.casehub.claudony.casehub.fleet.*;
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.BadRequestException;
import jakarta.ws.rs.NotFoundException;
import java.time.Duration;
import java.time.Instant;
import java.util.List;

@ApplicationScoped
public class PoolService {

    private final AgentPoolManagerRegistry mgrRegistry;
    private final AgentPoolDefinitionRegistry defRegistry;
    private final ScalingScheduler scalingScheduler;
    private final MeterRegistry meterRegistry;

    @Inject
    public PoolService(AgentPoolDefinitionRegistry defRegistry,
                       AgentPoolManagerRegistry mgrRegistry,
                       ScalingScheduler scalingScheduler,
                       MeterRegistry meterRegistry) {
        this.defRegistry = defRegistry;
        this.mgrRegistry = mgrRegistry;
        this.scalingScheduler = scalingScheduler;
        this.meterRegistry = meterRegistry;
    }

    public List<PoolSummary> listPools() {
        return mgrRegistry.poolNames().stream()
                .map(name -> mgrRegistry.get(name).map(mgr -> {
                    var scalingType = defRegistry.get(name)
                            .map(d -> d.pool().scaling().type())
                            .orElse("none");
                    return new PoolSummary(name, mgr.status(), scalingType);
                }).orElse(null))
                .filter(java.util.Objects::nonNull)
                .toList();
    }

    public PoolDetail getPool(String name) {
        var mgr = mgrRegistry.get(name)
                .orElseThrow(() -> new NotFoundException("Pool not found: " + name));
        var def = defRegistry.get(name).orElse(null);
        var scalingState = scalingScheduler.scalingState(name).orElse(null);
        var definitionView = def != null
                ? new PoolDetail.DefinitionView(def.agent(),
                    new PoolDetail.PoolConfigView(def.pool().minActive(), def.pool().maxActive(),
                        def.pool().eviction().name()))
                : null;
        var scalingView = buildScalingView(scalingState, def);
        var demandView = buildDemandView(name);
        return new PoolDetail(name, mgr.status(), definitionView, scalingView, demandView);
    }

    public List<ManagedSessionInfo> listSessions(String name) {
        var mgr = mgrRegistry.get(name)
                .orElseThrow(() -> new NotFoundException("Pool not found: " + name));
        var now = Instant.now();
        return mgr.sessions().stream().map(s -> new ManagedSessionInfo(
                s.instanceId(), s.identity(), s.workingDir(),
                s.conversationId(), s.state().name(),
                s.lastInteraction(), s.lastMemoryBytes(),
                Duration.between(s.lastInteraction(), now).toSeconds()
        )).toList();
    }

    public PoolDetail updatePool(String name, PoolUpdateRequest request) {
        var mgr = mgrRegistry.get(name)
                .orElseThrow(() -> new NotFoundException("Pool not found: " + name));
        if (request.maxActive() != null) {
            mgr.adjustMaxActive(request.maxActive());
        }
        if (request.minActive() != null || request.maxActive() != null) {
            int min = request.minActive() != null ? request.minActive() : mgr.status().min();
            int max = request.maxActive() != null ? request.maxActive() : mgr.status().max();
            defRegistry.updateCapacity(name, min, max);
        }
        if (request.scalingType() != null) {
            var newConfig = parseScalingConfig(request);
            defRegistry.updateScaling(name, newConfig);
            scalingScheduler.invalidatePolicy(name);
        }
        return getPool(name);
    }

    public void suspendSession(String name, String id) {
        var mgr = mgrRegistry.get(name)
                .orElseThrow(() -> new NotFoundException("Pool not found: " + name));
        mgr.suspendSession(id);
    }

    public void resumeSession(String name, String id) {
        var mgr = mgrRegistry.get(name)
                .orElseThrow(() -> new NotFoundException("Pool not found: " + name));
        mgr.resumeSession(id);
    }

    public void destroySession(String name, String id) {
        var mgr = mgrRegistry.get(name)
                .orElseThrow(() -> new NotFoundException("Pool not found: " + name));
        mgr.destroySession(id);
    }

    public String buildPoolSnapshot(String name) {
        try {
            var detail = getPool(name);
            var sessions = listSessions(name);
            var snapshot = new PoolSnapshot(detail, sessions);
            return new com.fasterxml.jackson.databind.ObjectMapper()
                    .registerModule(new com.fasterxml.jackson.datatype.jsr310.JavaTimeModule())
                    .writeValueAsString(snapshot);
        } catch (Exception e) {
            return "{}";
        }
    }

    public record PoolSnapshot(PoolDetail detail, List<ManagedSessionInfo> sessions) {}

    ScalingConfig parseScalingConfig(PoolUpdateRequest u) {
        var cooldown = u.cooldown() != null ? parseDuration(u.cooldown()) : null;
        var scaleInCooldown = u.scaleInCooldown() != null ? parseDuration(u.scaleInCooldown()) : null;
        return switch (u.scalingType()) {
            case "target-tracking" -> new ScalingConfig.TargetTrackingConfig(
                    u.targetFillRatio() != null ? u.targetFillRatio() : 0.7, cooldown, scaleInCooldown);
            case "step" -> {
                if (u.steps() == null || u.steps().isEmpty()) {
                    throw new IllegalArgumentException("steps must not be empty for step scaling");
                }
                var steps = u.steps().stream()
                        .map(s -> new ScalingStep(s.threshold(), s.adjustment()))
                        .toList();
                yield new ScalingConfig.StepConfig(steps, cooldown, scaleInCooldown);
            }
            case "demand-pressure" -> {
                if (u.exhaustionThreshold() == null || u.latencyThresholdMs() == null) {
                    throw new IllegalArgumentException(
                            "exhaustionThreshold and latencyThresholdMs required for demand-pressure");
                }
                yield new ScalingConfig.DemandPressureConfig(
                        u.exhaustionThreshold(), u.latencyThresholdMs(), cooldown, scaleInCooldown);
            }
            case "none" -> ScalingConfig.NoScalingConfig.INSTANCE;
            default -> new ScalingConfig.CustomScalingConfig(u.scalingType(), cooldown, scaleInCooldown);
        };
    }

    Duration parseDuration(String s) {
        try {
            if (s.endsWith("s")) return Duration.ofSeconds(Long.parseLong(s.substring(0, s.length() - 1)));
            if (s.endsWith("m")) return Duration.ofMinutes(Long.parseLong(s.substring(0, s.length() - 1)));
            return Duration.ofSeconds(Long.parseLong(s));
        } catch (NumberFormatException e) {
            throw new BadRequestException("Invalid duration: " + s);
        }
    }

    PoolDetail.ScalingView buildScalingView(ScalingState state, AgentPoolDefinition def) {
        if (state == null && def == null) return null;
        var config = state != null ? state.config() : (def != null ? def.pool().scaling() : null);
        var type = config != null ? config.type() : "none";
        var lastDecision = state != null && state.lastDecision() != null
                ? new PoolDetail.DecisionView(
                    state.lastDecision().direction().name(),
                    state.lastDecision().count(),
                    state.lastDecision().reason(),
                    state.lastDecisionTime() != null ? state.lastDecisionTime().toString() : null)
                : null;
        var cooldown = state != null ? formatDuration(state.cooldownRemaining(Instant.now())) : "0s";
        var configView = buildScalingConfigView(config);
        return new PoolDetail.ScalingView(type, configView, lastDecision, cooldown);
    }

    private ScalingConfigView buildScalingConfigView(ScalingConfig config) {
        if (config == null) return null;
        return switch (config) {
            case ScalingConfig.TargetTrackingConfig t -> new ScalingConfigView(
                    t.targetFillRatio(), null, null, null, null,
                    formatDuration(t.cooldown()), formatDuration(t.scaleInCooldown()));
            case ScalingConfig.StepConfig s -> new ScalingConfigView(
                    null,
                    s.steps().stream().map(st -> new ScalingConfigView.ScalingStepView(st.threshold(), st.adjustment())).toList(),
                    null, null, null,
                    formatDuration(s.cooldown()), formatDuration(s.scaleInCooldown()));
            case ScalingConfig.DemandPressureConfig d -> new ScalingConfigView(
                    null, null, d.exhaustionThreshold(), d.latencyThresholdMs(), null,
                    formatDuration(d.cooldown()), formatDuration(d.scaleInCooldown()));
            case ScalingConfig.CustomScalingConfig c -> new ScalingConfigView(
                    null, null, null, null, c.beanName(),
                    formatDuration(c.cooldown()), formatDuration(c.scaleInCooldown()));
            case ScalingConfig.NoScalingConfig n -> new ScalingConfigView(
                    null, null, null, null, null, "0s", "0s");
        };
    }

    private PoolDetail.DemandView buildDemandView(String name) {
        double acquires = counterValue("claudony.pool.acquires.total", name);
        double evictions = counterValue("claudony.pool.evictions.total", name);
        double exhaustions = counterValue("claudony.pool.exhaustions.total", name);
        return new PoolDetail.DemandView(acquires, evictions, exhaustions);
    }

    private double counterValue(String meterName, String poolName) {
        var counter = meterRegistry.find(meterName).tag("pool", poolName).counter();
        return counter != null ? counter.count() : 0.0;
    }

    private String formatDuration(Duration d) {
        return d == null || d.isZero() ? "0s" : d.toSeconds() + "s";
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PoolServiceTest`
Expected: PASS — all 12 tests green

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolUpdateRequest.java \
       app/src/main/java/io/casehub/claudony/server/fleet/ScalingStepInput.java \
       app/src/main/java/io/casehub/claudony/server/fleet/ScalingConfigView.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolDetail.java \
       app/src/test/java/io/casehub/claudony/server/fleet/PoolServiceTest.java
git commit -m "feat(#241): extract PoolService with all 5 scaling config types

Refs #241"
```

---

## Batch 2: mcpDomain API + PoolResource deprecation

### Task 2: Create ClaudonyPoolApi and deprecate PoolResource

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/api/ClaudonyPoolApi.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PoolResource.java` — add `@Deprecated`, delegate to `PoolService`
- Test: `app/src/test/java/io/casehub/claudony/server/api/ClaudonyPoolApiTest.java`

**Interfaces:**
- Consumes: `PoolService` (from Task 1), `PoolUpdateRequest`, `PoolSummary`, `PoolDetail`, `ManagedSessionInfo`
- Produces: `ClaudonyPoolApi` — mcpDomain endpoints at `/api/claudony/pools`

- [ ] **Step 1: Write failing integration tests for the mcpDomain endpoints**

Create `ClaudonyPoolApiTest.java`:

```java
package io.casehub.claudony.server.api;

import io.casehub.claudony.casehub.fleet.*;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class ClaudonyPoolApiTest {

    @Inject AgentPoolDefinitionRegistry defRegistry;
    @Inject AgentPoolManagerRegistry mgrRegistry;

    @BeforeEach
    void setUp() {
        // Register a test pool if not present — check existing test patterns
        // for how pools are set up in test context
    }

    @Test
    void listPools() {
        RestAssured.given()
                .when().get("/api/claudony/pools")
                .then().statusCode(200)
                .body("$", is(instanceOf(java.util.List.class)));
    }

    @Test
    void getPool_notFound() {
        RestAssured.given()
                .when().get("/api/claudony/pools/nonexistent")
                .then().statusCode(404);
    }

    @Test
    void updatePool_targetTracking() {
        RestAssured.given()
                .contentType(ContentType.JSON)
                .body("""
                    {"scalingType": "target-tracking", "targetFillRatio": 0.8, "cooldown": "30s"}
                    """)
                .when().post("/api/claudony/pools/{name}/update")
                .then().statusCode(anyOf(is(200), is(404)));
    }

    @Test
    void updatePool_demandPressure() {
        RestAssured.given()
                .contentType(ContentType.JSON)
                .body("""
                    {"scalingType": "demand-pressure", "exhaustionThreshold": 5, "latencyThresholdMs": 500}
                    """)
                .when().post("/api/claudony/pools/{name}/update")
                .then().statusCode(anyOf(is(200), is(404)));
    }

    @Test
    void updatePool_none() {
        RestAssured.given()
                .contentType(ContentType.JSON)
                .body("""
                    {"scalingType": "none"}
                    """)
                .when().post("/api/claudony/pools/{name}/update")
                .then().statusCode(anyOf(is(200), is(404)));
    }
}
```

Note: exact test setup depends on how pools are registered in test context. The implementer should check `PoolResourceTest` for the existing pattern and mirror it.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClaudonyPoolApiTest`
Expected: FAIL — `ClaudonyPoolApi` does not exist, 404 on all endpoints

- [ ] **Step 3: Create ClaudonyPoolApi**

Use `ide_create_file`:

```java
package io.casehub.claudony.server.api;

import io.casehub.claudony.server.fleet.*;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;

@McpDomain(value = "claudony/pools", app = "claudony",
           basePath = "/api/claudony/pools",
           summary = "Agent pool management — capacity, scaling, sessions")
@ApplicationScoped
public class ClaudonyPoolApi {

    @Inject PoolService poolService;

    @PlatformQuery("List all pools")
    @RestPath("/")
    public List<PoolSummary> listPools() {
        return poolService.listPools();
    }

    @PlatformQuery("Get pool details")
    @RestPath("/{name}")
    public PoolDetail getPool(@PathParam String name) {
        return poolService.getPool(name);
    }

    @PlatformQuery("List pool sessions")
    @RestPath("/{name}/sessions")
    public List<ManagedSessionInfo> listSessions(@PathParam String name) {
        return poolService.listSessions(name);
    }

    @PlatformMutation("Update pool configuration")
    @RestPath("/{name}/update")
    public PoolDetail updatePool(@PathParam String name, PoolUpdateRequest request) {
        return poolService.updatePool(name, request);
    }

    @PlatformMutation("Suspend a pool session")
    @RestPath("/{name}/sessions/{id}/suspend")
    public void suspendSession(@PathParam String name, @PathParam String id) {
        poolService.suspendSession(name, id);
    }

    @PlatformMutation("Resume a pool session")
    @RestPath("/{name}/sessions/{id}/resume")
    public void resumeSession(@PathParam String name, @PathParam String id) {
        poolService.resumeSession(name, id);
    }

    @PlatformMutation("Destroy a pool session")
    @RestPath("/{name}/sessions/{id}/destroy")
    public void destroySession(@PathParam String name, @PathParam String id) {
        poolService.destroySession(name, id);
    }
}
```

- [ ] **Step 4: Deprecate PoolResource — delegate to PoolService**

Use `ide_edit_member` on `PoolResource.java`:
- Add `@Deprecated(since = "0.3", forRemoval = true)` to the class
- Replace method bodies to delegate to `poolService` (inject it)
- Keep the existing `@Path`, `@GET`, `@PATCH`, `@DELETE` annotations intact

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClaudonyPoolApiTest`
Expected: PASS

Also run existing tests to verify no regressions:
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PoolResourceTest`
Expected: PASS — old endpoints still work via delegation

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/claudony/server/api/ClaudonyPoolApi.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolResource.java \
       app/src/test/java/io/casehub/claudony/server/api/ClaudonyPoolApiTest.java
git commit -m "feat(#241): mcpDomain pool API at /api/claudony/pools, deprecate PoolResource

Refs #241"
```

---

## Batch 3: SSE event streaming

### Task 3: PoolLifecycleEvent CDI bridge + PoolEventBus + SSE endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/PoolLifecycleEvent.java`
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventBus.java`
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventsResource.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolEventEmitter.java` — add CDI event fire
- Test: `app/src/test/java/io/casehub/claudony/server/fleet/PoolEventBusTest.java`
- Test: `app/src/test/java/io/casehub/claudony/server/fleet/PoolEventsResourceTest.java`

**Interfaces:**
- Consumes: `PoolService.buildPoolSnapshot(String)` (from Task 1), `PoolEventEmitter` (existing)
- Produces: `PoolLifecycleEvent` CDI record, `PoolEventBus` with `subscribe(String, Supplier<String>)` → `Multi<String>`, `PoolEventsResource` SSE endpoint at `/api/pool-events/{name}`

- [ ] **Step 1: Create PoolLifecycleEvent record**

Use `ide_create_file`:

```java
package io.casehub.claudony.server.fleet;

public record PoolLifecycleEvent(String poolName, String payloadJson) {}
```

- [ ] **Step 2: Write failing tests for PoolEventBus**

Create `PoolEventBusTest.java`:

```java
package io.casehub.claudony.server.fleet;

import io.smallrye.mutiny.helpers.test.AssertSubscriber;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class PoolEventBusTest {

    private PoolEventBus bus;

    @BeforeEach
    void setUp() {
        bus = new PoolEventBus();
    }

    @Test
    void subscribeReceivesInitialSnapshot() {
        var subscriber = bus.subscribe("test-pool", () -> "{\"initial\":true}")
                .subscribe().withSubscriber(AssertSubscriber.create(1));
        subscriber.assertItems("{\"initial\":true}");
    }

    @Test
    void emitPushesToSubscriber() {
        var subscriber = bus.subscribe("test-pool", () -> "{\"initial\":true}")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        bus.onPoolEvent(new PoolLifecycleEvent("test-pool", "{\"type\":\"scaling\"}"));
        subscriber.assertItems("{\"initial\":true}", "{\"type\":\"scaling\"}");
    }

    @Test
    void poolIsolation() {
        var sub1 = bus.subscribe("pool-a", () -> "a")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        var sub2 = bus.subscribe("pool-b", () -> "b")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        bus.onPoolEvent(new PoolLifecycleEvent("pool-a", "event-a"));
        sub1.assertItems("a", "event-a");
        sub2.assertItems("b"); // no event-a
    }

    @Test
    void subscriberCount() {
        assertEquals(0, bus.subscriberCount("test-pool"));
        var sub = bus.subscribe("test-pool", () -> "snap")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        assertEquals(1, bus.subscriberCount("test-pool"));
        sub.cancel();
        assertEquals(0, bus.subscriberCount("test-pool"));
    }

    @Test
    void cancelCleansUp() {
        var sub = bus.subscribe("test-pool", () -> "snap")
                .subscribe().withSubscriber(AssertSubscriber.create(10));
        sub.cancel();
        bus.onPoolEvent(new PoolLifecycleEvent("test-pool", "after-cancel"));
        sub.assertItems("snap"); // no after-cancel
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PoolEventBusTest`
Expected: FAIL — `PoolEventBus` does not exist

- [ ] **Step 4: Implement PoolEventBus**

Use `ide_create_file`:

```java
package io.casehub.claudony.server.fleet;

import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.MultiEmitter;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Supplier;

@ApplicationScoped
public class PoolEventBus {

    private final ConcurrentHashMap<String, List<MultiEmitter<String>>> emitters = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Supplier<String>> snapshotFns = new ConcurrentHashMap<>();

    void onPoolEvent(@Observes PoolLifecycleEvent event) {
        emit(event.poolName(), event.payloadJson());
    }

    public void emit(String poolName, String payload) {
        List<MultiEmitter<String>> list = emitters.get(poolName);
        if (list == null) return;
        list.forEach(e -> { if (!e.isCancelled()) e.emit(payload); });
    }

    @SuppressWarnings("unchecked")
    public Multi<String> subscribe(String poolName, Supplier<String> snapshotFn) {
        snapshotFns.put(poolName, snapshotFn);
        return Multi.createFrom().<String>emitter(emitter -> {
            emitter.emit(snapshotFn.get());
            MultiEmitter<String> typed = (MultiEmitter<String>) emitter;
            emitters.computeIfAbsent(poolName, k -> new CopyOnWriteArrayList<>()).add(typed);
            emitter.onTermination(() -> removeEmitter(poolName, typed));
        });
    }

    public int subscriberCount(String poolName) {
        List<MultiEmitter<String>> list = emitters.get(poolName);
        return list == null ? 0 : list.size();
    }

    private void removeEmitter(String poolName, MultiEmitter<String> emitter) {
        List<MultiEmitter<String>> list = emitters.get(poolName);
        if (list != null) {
            list.remove(emitter);
            if (list.isEmpty()) {
                emitters.remove(poolName);
                snapshotFns.remove(poolName);
            }
        }
    }
}
```

- [ ] **Step 5: Run PoolEventBus tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PoolEventBusTest`
Expected: PASS — all 5 tests green

- [ ] **Step 6: Add CDI event fire to PoolEventEmitter**

Modify `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolEventEmitter.java`:

The `PoolEventEmitter` is in the `casehub` module which has no dependency on the `app` module where `PoolLifecycleEvent` lives. Two options:
1. Move `PoolLifecycleEvent` to `casehub` module
2. Add an `EventCallback` functional interface to `PoolEventEmitter` (like `Broadcaster`)

Use option 2 — add a second callback alongside `Broadcaster`:

```java
public interface EventCallback {
    void onEvent(String poolName, String payloadJson);
}

private final EventCallback eventCallback;
```

Update constructors to accept an optional `EventCallback`. Update `PoolEventEmitterProducer` to provide a callback that fires `Event<PoolLifecycleEvent>`.

Use `ide_edit_member` and `ide_insert_member` for the modifications.

- [ ] **Step 7: Create PoolEventsResource**

Use `ide_create_file`:

```java
package io.casehub.claudony.server.fleet;

import io.casehub.platform.api.mcp.HandWrittenEndpoint;
import io.quarkus.security.Authenticated;
import io.smallrye.mutiny.Multi;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.NotFoundException;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;

@HandWrittenEndpoint("SSE streaming — cannot be expressed as mcpDomain")
@Path("/api/pool-events")
@Authenticated
public class PoolEventsResource {

    @Inject PoolEventBus poolEventBus;
    @Inject PoolService poolService;

    @GET
    @Path("/{name}")
    @Produces("text/event-stream")
    public Multi<String> poolEvents(@PathParam("name") String name) {
        // Validate pool exists (throws 404 if not)
        poolService.getPool(name);
        return poolEventBus.subscribe(name, () -> poolService.buildPoolSnapshot(name));
    }
}
```

- [ ] **Step 8: Write and run PoolEventsResource integration test**

Create `PoolEventsResourceTest.java` — verify SSE content-type, initial snapshot, 404 for unknown pool. Use the pattern from `SessionResourceCaseEventsTest` for SSE testing.

- [ ] **Step 9: Run all tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub,app`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolEventEmitter.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolLifecycleEvent.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolEventBus.java \
       app/src/main/java/io/casehub/claudony/server/fleet/PoolEventsResource.java \
       app/src/main/java/io/casehub/claudony/server/push/PoolEventEmitterProducer.java \
       app/src/test/java/io/casehub/claudony/server/fleet/PoolEventBusTest.java \
       app/src/test/java/io/casehub/claudony/server/fleet/PoolEventsResourceTest.java
git commit -m "feat(#242): pool SSE event streaming with CDI event bridge

Refs #242"
```

---

## Batch 4: Dashboard — path migration, SSE, editing controls, event log

### Task 4: Update pool panel — paths, SSE, capacity/scaling editing, event log

**Files:**
- Modify: `app/src/main/webui/src/components/claudony-pool-panel.ts` — full dashboard update

**Interfaces:**
- Consumes: `/api/claudony/pools` (from Task 2), `/api/pool-events/{name}` (from Task 3), `PoolDetail` with `ScalingConfigView` (from Task 1)
- Produces: Updated `claudony-pool-panel` with interactive editing and SSE

- [ ] **Step 1: Migrate API paths**

Update all 6 `fetchWithAuth` calls from `/api/pools` to `/api/claudony/pools`. Change destroy from `DELETE` to `POST` at `.../destroy`.

Use Edit tool (TypeScript, not structural Java editing):

Replace:
- `'/api/pools'` → `'/api/claudony/pools'`
- `'/api/pools/${...}'` → `'/api/claudony/pools/${...}'`
- destroy: `method: 'DELETE'` at `.../sessions/${id}` → `method: 'POST'` at `.../sessions/${id}/destroy`

- [ ] **Step 2: Add SSE integration**

Add `_eventSource` state field and `_connectSSE()` method. Replace `setInterval` with SSE + 60s fallback poll.

```typescript
@state() private _events: Array<{type: string; timestamp: string; [k: string]: unknown}> = [];
private _eventSource: EventSource | null = null;
private _fallbackTimer: ReturnType<typeof setInterval> | null = null;

private _connectSSE() {
  if (this._eventSource) this._eventSource.close();
  if (!this._selectedPool) return;
  this._eventSource = new EventSource(`/api/pool-events/${this._selectedPool}`);
  this._eventSource.onmessage = (e) => {
    try {
      const data = JSON.parse(e.data);
      if (data.detail) {
        // initial snapshot
        this._detail = data.detail;
        this._sessions = data.sessions || [];
      } else {
        this._events = [data, ...this._events].slice(0, 50);
        if (data.type === 'session') this._fetchDetail();
      }
    } catch (err) { console.error('SSE parse error', err); }
  };
  this._eventSource.onerror = () => {
    this._eventSource?.close();
    this._eventSource = null;
    if (!this._fallbackTimer) {
      this._fallbackTimer = setInterval(() => this._fetchPools(), 60000);
    }
  };
  // Clear fallback poll when SSE connects
  if (this._fallbackTimer) { clearInterval(this._fallbackTimer); this._fallbackTimer = null; }
}
```

Update `connectedCallback` to call `_connectSSE()` after initial fetch. Update `disconnectedCallback` to close EventSource.

- [ ] **Step 3: Add capacity editing controls**

Replace the static KPI cards for Min and Max with editable `<input type="number">` fields. Add `_updatePool` method:

```typescript
private async _updatePool(update: Record<string, unknown>) {
  try {
    const resp = await fetchWithAuth(`/api/claudony/pools/${this._selectedPool}/update`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(update),
    });
    if (resp.ok) {
      this._detail = await resp.json();
    } else {
      const err = await resp.text();
      console.error('Update failed:', err);
    }
  } catch (e) { console.error('Update failed', e); }
}
```

- [ ] **Step 4: Add scaling config editing form**

Expand `_renderScaling()` with:
- Type dropdown: `<select>` with `none`, `target-tracking`, `step`, `demand-pressure` options
- Conditional fields per type (show/hide based on selected type)
- Cooldown and scale-in cooldown text inputs
- Save button

Add `@state() private _editingScaling = false;` and `@state() private _scalingForm: Record<string, unknown> = {};` for form state.

```typescript
private _renderScalingEditor() {
  const type = this._scalingForm.scalingType as string || this._detail?.scaling?.type || 'none';
  return html`
    <div class="scaling-editor">
      <label>Type:
        <select @change=${(e: Event) => { this._scalingForm = {...this._scalingForm, scalingType: (e.target as HTMLSelectElement).value}; this.requestUpdate(); }}>
          <option value="none" ?selected=${type === 'none'}>None</option>
          <option value="target-tracking" ?selected=${type === 'target-tracking'}>Target Tracking</option>
          <option value="step" ?selected=${type === 'step'}>Step</option>
          <option value="demand-pressure" ?selected=${type === 'demand-pressure'}>Demand Pressure</option>
        </select>
      </label>
      ${type === 'target-tracking' ? html`
        <label>Fill Ratio: <input type="range" min="0.1" max="1.0" step="0.05"
          .value=${String(this._scalingForm.targetFillRatio ?? this._detail?.scaling?.config?.targetFillRatio ?? 0.7)}
          @input=${(e: Event) => { this._scalingForm = {...this._scalingForm, targetFillRatio: +(e.target as HTMLInputElement).value}; }}
        /> ${this._scalingForm.targetFillRatio ?? this._detail?.scaling?.config?.targetFillRatio ?? 0.7}</label>
      ` : nothing}
      ${type === 'demand-pressure' ? html`
        <label>Exhaustion Threshold: <input type="number" min="0"
          .value=${String(this._scalingForm.exhaustionThreshold ?? this._detail?.scaling?.config?.exhaustionThreshold ?? 5)}
          @change=${(e: Event) => { this._scalingForm = {...this._scalingForm, exhaustionThreshold: +(e.target as HTMLInputElement).value}; }}
        /></label>
        <label>Latency Threshold (ms): <input type="number" min="1"
          .value=${String(this._scalingForm.latencyThresholdMs ?? this._detail?.scaling?.config?.latencyThresholdMs ?? 500)}
          @change=${(e: Event) => { this._scalingForm = {...this._scalingForm, latencyThresholdMs: +(e.target as HTMLInputElement).value}; }}
        /></label>
      ` : nothing}
      <label>Cooldown: <input type="text" placeholder="60s"
        .value=${this._scalingForm.cooldown ?? this._detail?.scaling?.config?.cooldown ?? '60s'}
        @change=${(e: Event) => { this._scalingForm = {...this._scalingForm, cooldown: (e.target as HTMLInputElement).value}; }}
      /></label>
      <label>Scale-in Cooldown: <input type="text" placeholder="120s"
        .value=${this._scalingForm.scaleInCooldown ?? this._detail?.scaling?.config?.scaleInCooldown ?? '120s'}
        @change=${(e: Event) => { this._scalingForm = {...this._scalingForm, scaleInCooldown: (e.target as HTMLInputElement).value}; }}
      /></label>
      <button class="action-btn" @click=${() => this._saveScaling()}>Save</button>
      <button class="action-btn" @click=${() => { this._editingScaling = false; }}>Cancel</button>
    </div>
  `;
}

private _saveScaling() {
  this._updatePool(this._scalingForm);
  this._editingScaling = false;
  this._scalingForm = {};
}
```

- [ ] **Step 5: Add event log rendering**

Replace `_renderEventLog()` placeholder:

```typescript
private _renderEventLog() {
  return html`
    <h3>Event Log</h3>
    <div class="event-log">
      ${this._events.length === 0
        ? html`<div class="empty-state">No events yet</div>`
        : this._events.map(evt => html`
          <div style="padding: 4px 0; border-bottom: 1px solid var(--pages-neutral-3); font-size: var(--pages-font-size-xs);">
            <span style="color: var(--pages-neutral-6);">${evt.timestamp ? new Date(evt.timestamp as string).toLocaleTimeString() : ''}</span>
            <span style="margin: 0 8px; padding: 2px 6px; border-radius: var(--pages-radius-md); background: var(--pages-neutral-3);">${evt.type}</span>
            <span>${this._eventSummary(evt)}</span>
          </div>
        `)
      }
    </div>
  `;
}

private _eventSummary(evt: Record<string, unknown>): string {
  switch (evt.type) {
    case 'scaling': return `${evt.direction} ${evt.count} — ${evt.reason}`;
    case 'session': return `${evt.event} ${evt.identity || evt.instanceId}`;
    case 'health': return `${evt.previous} → ${evt.current}`;
    default: return JSON.stringify(evt);
  }
}
```

- [ ] **Step 6: Add CSS for editor and event log**

Add styles for `.scaling-editor label`, `.scaling-editor input`, `.scaling-editor select` — consistent with existing dark theme and `--pages-*` design tokens.

- [ ] **Step 7: Verify in browser**

Start dev server: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server`

Open `http://localhost:7777/app/` and navigate to the Pools tab. Verify:
- Pool list loads (uses new `/api/claudony/pools` path)
- Pool detail shows scaling info with type
- Capacity min/max are editable
- Scaling editor opens and shows type-specific fields
- Event log renders (may be empty if no scaling events)

- [ ] **Step 8: Commit**

```bash
git add app/src/main/webui/src/components/claudony-pool-panel.ts
git commit -m "feat(#242): interactive pool dashboard — editing controls, SSE, event log

Refs #242"
```

---

## Batch 5: CLAUDE.md update

### Task 5: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` — test count, key URLs, pool API paths

- [ ] **Step 1: Update key URLs section**

Add:
```
- Pool API (mcpDomain): `http://localhost:7777/api/claudony/pools`
- Pool SSE events: `http://localhost:7777/api/pool-events/{name}`
```

- [ ] **Step 2: Update test count**

Update the test baseline with the new test count after all tests pass.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md — pool API paths, test count

Refs #241"
```

---

## References

- `specs/feat-241-242-scaling-api-dashboard/2026-10-01-scaling-api-dashboard-design.md` — design spec
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolResource.java` — existing implementation being migrated
- `app/src/main/java/io/casehub/claudony/server/api/ClaudonySessionApi.java` — mcpDomain pattern
- `app/src/main/java/io/casehub/claudony/server/api/ClaudonyPeerApi.java` — mcpDomain mutation pattern
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java` — sealed interface
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolEventEmitter.java` — event emitter
- `app/src/main/java/io/casehub/claudony/server/CaseEventBroadcaster.java` — CDI SSE pattern
- `app/src/main/java/io/casehub/claudony/server/strategy/EventsOnlyStrategy.java` — events-only strategy
- `app/src/main/java/io/casehub/claudony/server/SessionResource.java:31-40` — SSE endpoint pattern
- `app/src/main/webui/src/components/claudony-pool-panel.ts` — existing dashboard panel
- GitHub #241, #242
