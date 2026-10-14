# ops/desiredstate Integration — App Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #282 — epic: ops/desiredstate integration — app lifecycle for fleet deployment
**Issue group:** #283, #284, #285, #286

**Goal:** Make ClaudonyAppApi a viable target for ops NodeProvisioner and ActualStateAdapter by adding namespace, in-place updates, suspend/resume, and deployment artifact reference.

**Architecture:** Four incremental additions to `ApplicationSpec` (casehub module) and `ClaudonyAppApi` (app module). Each sub-issue adds fields to the record, logic to `ApplicationRegistrationService`, and an endpoint to the API. All use `RegistryService` for storage — no new persistence layer. Suspend/resume stores pre-suspend pool capacity in app metadata for restoration.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, AssertJ, RestAssured, InMemoryRegistryService

## Global Constraints

- `HealthStatus.SUSPENDED` available from platform-api (casehubio/platform#615)
- `RegistryService` has no `update()` — use deregister + re-register with preserved timestamps
- Pool identity convention: `appName + "/" + agentId`
- Unit tests: plain JUnit + `InMemoryRegistryService` + AssertJ
- Integration tests: `@QuarkusTest` + `@TestSecurity` + RestAssured
- IntelliJ MCP required for all code operations

---

## Batch 1: Namespace (#283)

### Task 1: Add namespace to ApplicationSpec and ApplicationInfo

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationSpec.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationInfo.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationService.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationServiceTest.java`

**Interfaces:**
- Produces: `ApplicationSpec(name, namespace, displayName, description, pools)` — namespace defaults to `"default"` when null/blank
- Produces: `ApplicationInfo(name, namespace, displayName, description, health, registeredAt, pools)` — carries namespace through from spec
- Produces: `ApplicationRegistrationService.list(String namespace)` — filters by namespace

- [ ] **Step 1: Write failing tests for namespace on ApplicationSpec**

Add to `ApplicationRegistrationServiceTest.java`:

```java
@Test
void register_withExplicitNamespace_usesItInRegistryEntry() {
    var spec = new ApplicationSpec("myapp", "production", "My App", "desc", List.of());

    service.register(spec);

    var entries = registryService.discover(new RegistryQuery("default", "application", "production"));
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).namespace()).isEqualTo("production");
}

@Test
void register_withNullNamespace_defaultsToDefault() {
    var spec = new ApplicationSpec("myapp", null, "My App", "desc", List.of());

    service.register(spec);

    var entries = registryService.discover(new RegistryQuery("default", "application", "default"));
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).namespace()).isEqualTo("default");
}

@Test
void register_namespaceCarriedToApplicationInfo() {
    var spec = new ApplicationSpec("myapp", "staging", "My App", "desc", List.of());

    service.register(spec);
    var info = service.get("myapp");

    assertThat(info).isPresent();
    assertThat(info.get().namespace()).isEqualTo("staging");
}

@Test
void list_filtersByNamespace() {
    service.register(new ApplicationSpec("app1", "prod", "App 1", "", List.of()));
    service.register(new ApplicationSpec("app2", "staging", "App 2", "", List.of()));
    service.register(new ApplicationSpec("app3", "prod", "App 3", "", List.of()));

    var prodApps = service.list("prod");

    assertThat(prodApps).hasSize(2);
    assertThat(prodApps).extracting(ApplicationInfo::name).containsExactlyInAnyOrder("app1", "app3");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ApplicationRegistrationServiceTest -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: Compilation errors — `ApplicationSpec` constructor doesn't accept namespace yet.

- [ ] **Step 3: Add namespace to ApplicationSpec**

Update `ApplicationSpec` record to add `namespace` as 2nd parameter with default in compact constructor:

```java
public record ApplicationSpec(
        String name,
        String namespace,
        String displayName,
        String description,
        List<PoolSpec> pools
) {
    public ApplicationSpec {
        Objects.requireNonNull(name, "name");
        if (name.isBlank()) throw new IllegalArgumentException("name must not be blank");
        if (namespace == null || namespace.isBlank()) namespace = "default";
        Objects.requireNonNull(displayName, "displayName");
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }
}
```

- [ ] **Step 4: Add namespace to ApplicationInfo**

Update `ApplicationInfo` record to add `namespace` as 2nd parameter:

```java
public record ApplicationInfo(
        String name,
        String namespace,
        String displayName,
        String description,
        HealthStatus health,
        Instant registeredAt,
        List<AppPoolInfo> pools
) {
    public ApplicationInfo {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(namespace, "namespace");
        Objects.requireNonNull(health, "health");
        Objects.requireNonNull(registeredAt, "registeredAt");
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }
}
```

- [ ] **Step 5: Update ApplicationRegistrationService to use namespace**

In `register()`: change the `RegistryEntry` constructor call to pass `spec.namespace()` instead of `spec.name()` as the namespace (3rd arg). Also store namespace in metadata.

In `toApplicationInfo()`: read namespace from `entry.namespace()`.

Add `list(String namespace)` overload:

```java
public List<ApplicationInfo> list(String namespace) {
    var tenancyId = tenantContext.currentTenantId();
    return registryService.discover(new RegistryQuery(tenancyId, "application", namespace))
            .stream()
            .map(this::toApplicationInfo)
            .toList();
}
```

Update the `toApplicationInfo` method to include namespace:

```java
return new ApplicationInfo(
        entry.id(),
        entry.namespace(),
        entry.metadata().getOrDefault("displayName", entry.id()),
        entry.metadata().getOrDefault("description", ""),
        entry.health(),
        entry.registeredAt(),
        poolInfos);
```

Update `register()` return to include namespace:

```java
return new ApplicationInfo(spec.name(), spec.namespace(), spec.displayName(),
        spec.description(), HealthStatus.HEALTHY, now, poolInfos);
```

- [ ] **Step 6: Fix existing tests — update ApplicationSpec constructor calls**

All existing tests use the 4-arg constructor `ApplicationSpec(name, displayName, description, pools)`. Update them to 5-arg: `ApplicationSpec(name, "default", displayName, description, pools)`. Also update `ApplicationInfo` assertions to account for the new `namespace` field.

Fix the existing test `register_createsAppEntryWithCorrectTypeAndNamespace` to assert `namespace == "default"` instead of `namespace == "scaffold"`.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ApplicationRegistrationServiceTest -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: All tests PASS.

- [ ] **Step 8: Fix ClaudonyAppApiTest constructor calls**

Update the JSON bodies in `ClaudonyAppApiTest` — `registerApp_success` and `heartbeat_success` — to include `"namespace": "default"` (or omit it, since the server defaults to `"default"`). Update response assertions to check `namespace` field if present.

- [ ] **Step 9: Fix any other compilation errors across modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: All tests PASS. If there are other callers of the old 4-arg `ApplicationSpec` or 6-arg `ApplicationInfo` constructors, fix them (use `ide_find_references` to locate all call sites).

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#283): add namespace field to ApplicationSpec and ApplicationInfo

Refs #283"
```

---

## Batch 2: updateApp (#284)

### Task 2: Add update method to ApplicationRegistrationService

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationService.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationServiceTest.java`

**Interfaces:**
- Consumes: `ApplicationSpec(name, namespace, displayName, description, pools)` from Task 1
- Produces: `ApplicationRegistrationService.update(String name, ApplicationSpec spec) → ApplicationInfo`

- [ ] **Step 1: Write failing tests for update**

Add to `ApplicationRegistrationServiceTest.java`:

```java
@Test
void update_changesDisplayNamePreservesTimestamps() {
    var spec = new ApplicationSpec("myapp", "default", "Original", "desc", List.of());
    var original = service.register(spec);
    var originalRegisteredAt = original.registeredAt();

    var updated = service.update("myapp",
            new ApplicationSpec("myapp", "default", "Updated", "new desc", List.of()));

    assertThat(updated.displayName()).isEqualTo("Updated");
    assertThat(updated.description()).isEqualTo("new desc");
    assertThat(updated.registeredAt()).isEqualTo(originalRegisteredAt);
}

@Test
void update_addsNewPool() {
    service.register(new ApplicationSpec("myapp", "default", "App", "",
            List.of(new ApplicationSpec.PoolSpec("review", 0, 3))));

    service.update("myapp", new ApplicationSpec("myapp", "default", "App", "",
            List.of(
                    new ApplicationSpec.PoolSpec("review", 0, 3),
                    new ApplicationSpec.PoolSpec("code", 0, 5)
            )));

    assertThat(poolRegistry.get("myapp/code")).isPresent();
    assertThat(poolRegistry.get("myapp/review")).isPresent();
}

@Test
void update_removesOldPool() {
    service.register(new ApplicationSpec("myapp", "default", "App", "",
            List.of(
                    new ApplicationSpec.PoolSpec("review", 0, 3),
                    new ApplicationSpec.PoolSpec("code", 0, 5)
            )));

    service.update("myapp", new ApplicationSpec("myapp", "default", "App", "",
            List.of(new ApplicationSpec.PoolSpec("review", 0, 3))));

    assertThat(poolRegistry.get("myapp/code")).isEmpty();
    assertThat(poolRegistry.get("myapp/review")).isPresent();
}

@Test
void update_changesPoolScaling() {
    service.register(new ApplicationSpec("myapp", "default", "App", "",
            List.of(new ApplicationSpec.PoolSpec("review", 0, 3))));

    service.update("myapp", new ApplicationSpec("myapp", "default", "App", "",
            List.of(new ApplicationSpec.PoolSpec("review", 1, 10))));

    var def = poolRegistry.get("myapp/review");
    assertThat(def).isPresent();
    assertThat(def.get().pool().minActive()).isEqualTo(1);
    assertThat(def.get().pool().maxActive()).isEqualTo(10);
}

@Test
void update_nonExistentThrows() {
    assertThatThrownBy(() -> service.update("nonexistent",
            new ApplicationSpec("nonexistent", "default", "X", "", List.of())))
            .isInstanceOf(jakarta.ws.rs.WebApplicationException.class);
}

@Test
void update_nameMismatchThrows() {
    service.register(new ApplicationSpec("myapp", "default", "App", "", List.of()));

    assertThatThrownBy(() -> service.update("myapp",
            new ApplicationSpec("different", "default", "App", "", List.of())))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("name");
}

@Test
void update_preservesHealthState() {
    service.register(new ApplicationSpec("myapp", "default", "App", "", List.of()));

    var updated = service.update("myapp",
            new ApplicationSpec("myapp", "default", "Updated", "", List.of()));

    assertThat(updated.health()).isEqualTo(HealthStatus.HEALTHY);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ApplicationRegistrationServiceTest -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: Compilation error — `update()` doesn't exist.

- [ ] **Step 3: Implement update method**

Add to `ApplicationRegistrationService`:

```java
public ApplicationInfo update(String name, ApplicationSpec spec) {
    if (!name.equals(spec.name())) {
        throw new IllegalArgumentException("spec name must match path name");
    }

    var existing = registryService.resolve(name)
            .filter(e -> "application".equals(e.type()))
            .orElseThrow(() -> new WebApplicationException(Response.Status.NOT_FOUND));

    var originalRegisteredAt = existing.registeredAt();
    var originalLastHeartbeat = existing.lastHeartbeat();
    var originalHealth = existing.health();

    // Capture old pool IDs for diff
    var oldPoolIds = registryService.relationships(name).stream()
            .filter(r -> "owns".equals(r.type()) && r.sourceId().equals(name))
            .map(Relationship::targetId)
            .collect(java.util.stream.Collectors.toSet());

    // Deregister (cascades relationships)
    registryService.deregister(name);

    // Re-register with preserved timestamps
    var tenancyId = tenantContext.currentTenantId();
    var metadata = new java.util.HashMap<String, String>();
    metadata.put("displayName", spec.displayName());
    metadata.put("description", spec.description() != null ? spec.description() : "");

    var appEntry = new RegistryEntry(
            spec.name(), "application", spec.namespace(), tenancyId,
            Map.copyOf(metadata),
            originalRegisteredAt, originalLastHeartbeat,
            Duration.ofHours(24), originalHealth, null);
    registryService.register(appEntry);

    // Diff pools
    var newPoolIds = new java.util.HashSet<String>();
    var poolInfos = new java.util.ArrayList<ApplicationInfo.AppPoolInfo>();
    for (var poolSpec : spec.pools()) {
        var poolId = spec.name() + "/" + poolSpec.agentId();
        newPoolIds.add(poolId);

        if (oldPoolIds.contains(poolId)) {
            // Update existing pool scaling
            poolRegistry.updateCapacity(poolId, poolSpec.minActive(), poolSpec.maxActive());
        } else {
            // Register new pool
            var def = AgentPoolDefinition.builder()
                    .agent(poolId)
                    .pool()
                        .minActive(poolSpec.minActive())
                        .maxActive(poolSpec.maxActive())
                    .build();
            poolRegistry.register(def, spec.namespace());
        }
        registryService.link(new Relationship(spec.name(), poolId, "owns"));
        poolInfos.add(new ApplicationInfo.AppPoolInfo(poolSpec.agentId(),
                poolSpec.minActive(), poolSpec.maxActive()));
    }

    // Remove pools no longer in spec
    for (var oldId : oldPoolIds) {
        if (!newPoolIds.contains(oldId)) {
            registryService.deregister(oldId);
        }
    }

    return new ApplicationInfo(spec.name(), spec.namespace(), spec.displayName(),
            spec.description(), originalHealth, originalRegisteredAt, poolInfos);
}
```

Add import for `WebApplicationException`, `Response`, `Relationship`, `Duration` if not present.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ApplicationRegistrationServiceTest -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: All tests PASS.

- [ ] **Step 5: Add updateApp endpoint to ClaudonyAppApi**

Add to `ClaudonyAppApi.java`:

```java
@PlatformMutation("Update an application")
@RestPath("/{name}")
public ApplicationInfo updateApp(@PathParam String name, ApplicationSpec spec) {
    return appService.update(name, spec);
}
```

- [ ] **Step 6: Add integration test for updateApp**

Add to `ClaudonyAppApiTest.java`:

```java
@Test
void updateApp_success() {
    var name = "upd-test-" + java.util.UUID.randomUUID().toString().substring(0, 8);
    RestAssured.given()
            .contentType(ContentType.JSON)
            .body("""
                    {"name": "%s", "displayName": "Original", "description": "", "pools": []}
                    """.formatted(name))
            .when().post("/api/claudony/apps")
            .then().statusCode(200);

    RestAssured.given()
            .contentType(ContentType.JSON)
            .body("""
                    {"name": "%s", "displayName": "Updated", "description": "new desc", "pools": []}
                    """.formatted(name))
            .when().put("/api/claudony/apps/" + name)
            .then().statusCode(200)
            .body("displayName", is("Updated"))
            .body("description", is("new desc"));
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: All tests PASS.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#284): add updateApp to ClaudonyAppApi — in-place spec updates

Deregister + re-register with preserved timestamps. Pool diff
handles add/remove/update.

Refs #284"
```

---

## Batch 3: Suspend/Resume (#285)

### Task 3: Add suspend and resume to ApplicationRegistrationService

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationService.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationServiceTest.java`

**Interfaces:**
- Consumes: `HealthStatus.SUSPENDED` from platform-api (platform#615)
- Consumes: `AgentPoolDefinitionRegistry.updateCapacity(agentName, min, max)` from existing API
- Produces: `ApplicationRegistrationService.suspend(String name)` — sets health to SUSPENDED, stores pool capacity, zeros pools
- Produces: `ApplicationRegistrationService.resume(String name)` — restores health to HEALTHY, restores pool capacity

- [ ] **Step 1: Write failing tests for suspend/resume**

Add to `ApplicationRegistrationServiceTest.java`:

```java
@Test
void suspend_setsHealthToSuspended() {
    service.register(new ApplicationSpec("myapp", "default", "App", "", List.of()));

    service.suspend("myapp");

    var entry = registryService.resolve("myapp");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.SUSPENDED);
}

@Test
void suspend_alreadySuspendedIsNoOp() {
    service.register(new ApplicationSpec("myapp", "default", "App", "", List.of()));

    service.suspend("myapp");
    service.suspend("myapp"); // no-op

    var entry = registryService.resolve("myapp");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.SUSPENDED);
}

@Test
void suspend_nonExistentThrows() {
    assertThatThrownBy(() -> service.suspend("nonexistent"))
            .isInstanceOf(jakarta.ws.rs.WebApplicationException.class);
}

@Test
void resume_restoresHealthToHealthy() {
    service.register(new ApplicationSpec("myapp", "default", "App", "", List.of()));
    service.suspend("myapp");

    service.resume("myapp");

    var entry = registryService.resolve("myapp");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.HEALTHY);
}

@Test
void resume_nonSuspendedIsNoOp() {
    service.register(new ApplicationSpec("myapp", "default", "App", "", List.of()));

    service.resume("myapp"); // no-op — already healthy

    var entry = registryService.resolve("myapp");
    assertThat(entry).isPresent();
    assertThat(entry.get().health()).isEqualTo(HealthStatus.HEALTHY);
}

@Test
void resume_nonExistentThrows() {
    assertThatThrownBy(() -> service.resume("nonexistent"))
            .isInstanceOf(jakarta.ws.rs.WebApplicationException.class);
}

@Test
void suspend_zerosPoolCapacityAndStoresOriginal() {
    service.register(new ApplicationSpec("myapp", "default", "App", "",
            List.of(new ApplicationSpec.PoolSpec("review", 2, 5))));

    service.suspend("myapp");

    var def = poolRegistry.get("myapp/review");
    assertThat(def).isPresent();
    assertThat(def.get().pool().minActive()).isEqualTo(0);
    assertThat(def.get().pool().maxActive()).isEqualTo(0);
}

@Test
void resume_restoresPoolCapacity() {
    service.register(new ApplicationSpec("myapp", "default", "App", "",
            List.of(new ApplicationSpec.PoolSpec("review", 2, 5))));
    service.suspend("myapp");

    service.resume("myapp");

    var def = poolRegistry.get("myapp/review");
    assertThat(def).isPresent();
    assertThat(def.get().pool().minActive()).isEqualTo(2);
    assertThat(def.get().pool().maxActive()).isEqualTo(5);
}

@Test
void suspend_applicationInfoReflectsSuspendedHealth() {
    service.register(new ApplicationSpec("myapp", "default", "App", "", List.of()));
    service.suspend("myapp");

    var info = service.get("myapp");

    assertThat(info).isPresent();
    assertThat(info.get().health()).isEqualTo(HealthStatus.SUSPENDED);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ApplicationRegistrationServiceTest -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: Compilation error — `suspend()` and `resume()` don't exist.

- [ ] **Step 3: Implement suspend method**

Add to `ApplicationRegistrationService`:

```java
public void suspend(String name) {
    var existing = registryService.resolve(name)
            .filter(e -> "application".equals(e.type()))
            .orElseThrow(() -> new WebApplicationException(Response.Status.NOT_FOUND));

    if (existing.health() == HealthStatus.SUSPENDED) {
        return; // already suspended
    }

    // Save pool capacities in app metadata and zero pools
    var rels = registryService.relationships(name);
    var metadata = new java.util.HashMap<>(existing.metadata());
    for (var rel : rels) {
        if ("owns".equals(rel.type()) && rel.sourceId().equals(name)) {
            var poolDef = poolRegistry.get(rel.targetId());
            if (poolDef.isPresent()) {
                var pool = poolDef.get().pool();
                metadata.put("saved.minActive." + rel.targetId(), String.valueOf(pool.minActive()));
                metadata.put("saved.maxActive." + rel.targetId(), String.valueOf(pool.maxActive()));
                poolRegistry.updateCapacity(rel.targetId(), 0, 0);
            }
        }
    }

    // Deregister + re-register with SUSPENDED health
    registryService.deregister(name);
    var updated = new RegistryEntry(
            existing.id(), existing.type(), existing.namespace(), existing.tenancyId(),
            Map.copyOf(metadata),
            existing.registeredAt(), existing.lastHeartbeat(),
            existing.ttl(), HealthStatus.SUSPENDED, existing.orphanedAt());
    registryService.register(updated);

    // Re-link pool relationships
    for (var rel : rels) {
        if ("owns".equals(rel.type()) && rel.sourceId().equals(name)) {
            registryService.link(rel);
        }
    }
}
```

- [ ] **Step 4: Implement resume method**

Add to `ApplicationRegistrationService`:

```java
public void resume(String name) {
    var existing = registryService.resolve(name)
            .filter(e -> "application".equals(e.type()))
            .orElseThrow(() -> new WebApplicationException(Response.Status.NOT_FOUND));

    if (existing.health() != HealthStatus.SUSPENDED) {
        return; // not suspended
    }

    // Restore pool capacities from saved metadata
    var rels = registryService.relationships(name);
    var metadata = new java.util.HashMap<>(existing.metadata());
    for (var rel : rels) {
        if ("owns".equals(rel.type()) && rel.sourceId().equals(name)) {
            var savedMin = metadata.remove("saved.minActive." + rel.targetId());
            var savedMax = metadata.remove("saved.maxActive." + rel.targetId());
            if (savedMin != null && savedMax != null) {
                poolRegistry.updateCapacity(rel.targetId(),
                        Integer.parseInt(savedMin), Integer.parseInt(savedMax));
            }
        }
    }

    // Deregister + re-register with HEALTHY
    registryService.deregister(name);
    var updated = new RegistryEntry(
            existing.id(), existing.type(), existing.namespace(), existing.tenancyId(),
            Map.copyOf(metadata),
            existing.registeredAt(), existing.lastHeartbeat(),
            existing.ttl(), HealthStatus.HEALTHY, existing.orphanedAt());
    registryService.register(updated);

    // Re-link pool relationships
    for (var rel : rels) {
        if ("owns".equals(rel.type()) && rel.sourceId().equals(name)) {
            registryService.link(rel);
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ApplicationRegistrationServiceTest -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: All tests PASS.

- [ ] **Step 6: Add suspend/resume endpoints to ClaudonyAppApi**

Add to `ClaudonyAppApi.java`:

```java
@PlatformMutation("Suspend an application")
@RestPath("/{name}/suspend")
public void suspendApp(@PathParam String name) {
    appService.suspend(name);
}

@PlatformMutation("Resume an application")
@RestPath("/{name}/resume")
public void resumeApp(@PathParam String name) {
    appService.resume(name);
}
```

- [ ] **Step 7: Add integration tests for suspend/resume**

Add to `ClaudonyAppApiTest.java`:

```java
@Test
void suspendApp_success() {
    var name = "sus-test-" + java.util.UUID.randomUUID().toString().substring(0, 8);
    RestAssured.given()
            .contentType(ContentType.JSON)
            .body("""
                    {"name": "%s", "displayName": "Test", "description": "", "pools": []}
                    """.formatted(name))
            .when().post("/api/claudony/apps")
            .then().statusCode(200);

    RestAssured.given()
            .contentType(ContentType.JSON)
            .when().post("/api/claudony/apps/" + name + "/suspend")
            .then().statusCode(200);

    RestAssured.given()
            .when().get("/api/claudony/apps/" + name)
            .then().statusCode(200)
            .body("health", is("SUSPENDED"));
}

@Test
void resumeApp_success() {
    var name = "res-test-" + java.util.UUID.randomUUID().toString().substring(0, 8);
    RestAssured.given()
            .contentType(ContentType.JSON)
            .body("""
                    {"name": "%s", "displayName": "Test", "description": "", "pools": []}
                    """.formatted(name))
            .when().post("/api/claudony/apps")
            .then().statusCode(200);

    RestAssured.given()
            .contentType(ContentType.JSON)
            .when().post("/api/claudony/apps/" + name + "/suspend")
            .then().statusCode(200);

    RestAssured.given()
            .contentType(ContentType.JSON)
            .when().post("/api/claudony/apps/" + name + "/resume")
            .then().statusCode(200);

    RestAssured.given()
            .when().get("/api/claudony/apps/" + name)
            .then().statusCode(200)
            .body("health", is("HEALTHY"));
}
```

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: All tests PASS.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#285): add suspend/resume lifecycle to ClaudonyAppApi

App-level suspend sets health to SUSPENDED, zeros all pool capacity.
Resume restores original capacity from saved metadata.

Refs #285"
```

---

## Batch 4: Deployment artifact (#286)

### Task 4: Add image and config to ApplicationSpec

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationSpec.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationInfo.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationService.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationServiceTest.java`
- Modify: `app/src/test/java/io/casehub/claudony/server/api/ClaudonyAppApiTest.java`

**Interfaces:**
- Consumes: `ApplicationSpec(name, namespace, displayName, description, pools)` from Tasks 1-3
- Produces: `ApplicationSpec(name, namespace, displayName, description, image, config, pools)` — image optional (nullable), config defaults to empty map
- Produces: `ApplicationInfo(name, namespace, displayName, description, image, config, health, registeredAt, pools)` — carries through from spec

- [ ] **Step 1: Write failing tests for image and config**

Add to `ApplicationRegistrationServiceTest.java`:

```java
@Test
void register_withImageAndConfig_storedInMetadata() {
    var spec = new ApplicationSpec("myapp", "default", "App", "desc",
            "ghcr.io/casehubio/myapp:latest", Map.of("LOG_LEVEL", "DEBUG"), List.of());

    service.register(spec);
    var info = service.get("myapp");

    assertThat(info).isPresent();
    assertThat(info.get().image()).isEqualTo("ghcr.io/casehubio/myapp:latest");
    assertThat(info.get().config()).containsEntry("LOG_LEVEL", "DEBUG");
}

@Test
void register_withoutImageOrConfig_defaultsToNullAndEmpty() {
    var spec = new ApplicationSpec("myapp", "default", "App", "desc",
            null, null, List.of());

    service.register(spec);
    var info = service.get("myapp");

    assertThat(info).isPresent();
    assertThat(info.get().image()).isNull();
    assertThat(info.get().config()).isEmpty();
}

@Test
void update_changesImageAndConfig() {
    service.register(new ApplicationSpec("myapp", "default", "App", "",
            "old:v1", Map.of("A", "1"), List.of()));

    service.update("myapp", new ApplicationSpec("myapp", "default", "App", "",
            "new:v2", Map.of("B", "2"), List.of()));

    var info = service.get("myapp");
    assertThat(info).isPresent();
    assertThat(info.get().image()).isEqualTo("new:v2");
    assertThat(info.get().config()).containsEntry("B", "2");
    assertThat(info.get().config()).doesNotContainKey("A");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ApplicationRegistrationServiceTest -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: Compilation error — `ApplicationSpec` doesn't accept image/config.

- [ ] **Step 3: Add image and config to ApplicationSpec**

Update `ApplicationSpec` record:

```java
public record ApplicationSpec(
        String name,
        String namespace,
        String displayName,
        String description,
        String image,
        Map<String, String> config,
        List<PoolSpec> pools
) {
    public ApplicationSpec {
        Objects.requireNonNull(name, "name");
        if (name.isBlank()) throw new IllegalArgumentException("name must not be blank");
        if (namespace == null || namespace.isBlank()) namespace = "default";
        Objects.requireNonNull(displayName, "displayName");
        if (config == null) config = Map.of();
        config = Map.copyOf(config);
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }
}
```

- [ ] **Step 4: Add image and config to ApplicationInfo**

Update `ApplicationInfo` record:

```java
public record ApplicationInfo(
        String name,
        String namespace,
        String displayName,
        String description,
        String image,
        Map<String, String> config,
        HealthStatus health,
        Instant registeredAt,
        List<AppPoolInfo> pools
) {
    public ApplicationInfo {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(namespace, "namespace");
        Objects.requireNonNull(health, "health");
        Objects.requireNonNull(registeredAt, "registeredAt");
        if (config == null) config = Map.of();
        config = Map.copyOf(config);
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }
}
```

- [ ] **Step 5: Update ApplicationRegistrationService to store/retrieve image and config**

In `register()`: add image and config entries to the metadata map:

```java
if (spec.image() != null) metadata.put("image", spec.image());
for (var entry : spec.config().entrySet()) {
    metadata.put("config." + entry.getKey(), entry.getValue());
}
```

In `toApplicationInfo()`: extract image and config from metadata:

```java
var image = entry.metadata().get("image");
var config = entry.metadata().entrySet().stream()
        .filter(e -> e.getKey().startsWith("config."))
        .collect(java.util.stream.Collectors.toMap(
                e -> e.getKey().substring("config.".length()),
                Map.Entry::getValue));
```

Update all `new ApplicationInfo(...)` calls in register, update, and toApplicationInfo to include image and config.

- [ ] **Step 6: Fix all existing constructor call sites**

Use `ide_find_references` to find all `ApplicationSpec` and `ApplicationInfo` constructor calls. Update them to include `image` (null) and `config` (Map.of() or null) parameters. This includes:
- All tests in `ApplicationRegistrationServiceTest`
- `ClaudonyAppApiTest` JSON bodies (add `"image": null, "config": {}` or omit — Jackson ignores unknown)
- Any other module callers

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Expected: All tests PASS.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#286): add deployment artifact reference to ApplicationSpec

Adds image (String, optional) and config (Map<String,String>, optional)
to ApplicationSpec and ApplicationInfo. Stored in RegistryEntry metadata
with 'config.' prefix to avoid collisions.

Refs #286"
```

---

## Batch 5: CLAUDE.md and issue closure

### Task 5: Update CLAUDE.md and close sub-issues

**Files:**
- Modify: `CLAUDE.md` (project root) — update test count, API URLs, ApplicationSpec docs

- [ ] **Step 1: Run full test suite and count**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/claudony/pom.xml`
Count total tests and update CLAUDE.md test baseline.

- [ ] **Step 2: Update CLAUDE.md**

Update test count, add new API endpoints to Key URLs section, update ApplicationSpec description in Project Structure.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for #282 ops/desiredstate app lifecycle

Refs #282"
```

## References

- [2026-10-14-ops-desiredstate-app-lifecycle-design.md] — design spec this plan implements
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationSpec.java` — current 4-field record
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationInfo.java` — current 6-field record
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationService.java` — current registration logic
- `app/src/main/java/io/casehub/claudony/server/api/ClaudonyAppApi.java` — current 5-endpoint API
- `casehub/src/test/java/io/casehub/claudony/casehub/fleet/ApplicationRegistrationServiceTest.java` — unit test pattern
- `app/src/test/java/io/casehub/claudony/server/api/ClaudonyAppApiTest.java` — integration test pattern
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java` — pool scaling API
- casehubio/platform#615 — SUSPENDED added to HealthStatus
- casehubio/claudony#283, #284, #285, #286 — sub-issues
