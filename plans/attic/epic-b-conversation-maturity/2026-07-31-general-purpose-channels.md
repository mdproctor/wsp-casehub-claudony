# General-Purpose Channels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #177 — feat: general-purpose chat rooms (not case-scoped)
**Issue group:** #177

**Goal:** Enable channel creation outside the CaseHub engine lifecycle
and ensure all channels participate in the backend infrastructure equally.

**Architecture:** Two changes to `claudony-app`. (1) Remove the `"case-"`
prefix filter in `ClaudonyChannelBackend.onChannelInitialised()` so all
channels get backend registration. (2) Add `POST /api/mesh/channels` to
`MeshResource` as a thin pass-through to `ChannelService.findOrCreate()`.

**Tech Stack:** Java 21/26, Quarkus 3.32, Qhorus 0.2-SNAPSHOT
(`ChannelService`, `ChannelCreateRequest`, `FindOrCreateResult`,
`QhorusEntityMapper`), RestAssured, Mockito, AssertJ

## Global Constraints

- IntelliJ MCP for all `.java` edits — no bash Edit/Write on source files
- `ChannelCreateRequest.builder(name)` for constructing requests (GE-20260613-7b7ae1)
- `ChannelService.findOrCreate()` for race-safe idempotency (GE-20260529-88b7b6)
- `ChannelInitialisedEvent` fires automatically from `ChannelCreateHelper` (GE-20260609-9ee2ad)
- `@TestSecurity(user = "test", roles = "user")` on all QuarkusTest classes
- Qhorus test cleanup: `channelStore.clear()` + `messageStore.clear()` in `@AfterEach`

---

### Task 1: Broaden ClaudonyChannelBackend — remove case- prefix filter

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/ClaudonyChannelBackend.java:49-51`
- Modify: `app/src/test/java/io/casehub/claudony/server/ClaudonyChannelBackendTest.java:95-103`

**Interfaces:**
- Consumes: `ChannelInitialisedEvent(UUID channelId, String channelName, boolean recovered)`,
  `ChannelGateway.deregisterBackend(UUID, String)`, `ChannelGateway.registerBackend(UUID, ChannelBackend, String)`
- Produces: backend registers for all channels (no prefix filter)

- [ ] **Step 1: Update the test — change assertion for non-case channels**

The existing test `onChannelInitialised_nonCaseChannel_noRegistration` asserts
`verifyNoInteractions(gateway)`. Change it to verify the backend registers for
non-case channels too — same as case channels.

```java
@Test
void onChannelInitialised_nonCaseChannel_registersBackend() {
    UUID channelId = UUID.randomUUID();

    backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "team/engineering", false));

    verify(gateway).deregisterBackend(channelId, ClaudonyChannelBackend.BACKEND_ID);
    verify(gateway).registerBackend(channelId, backend, "human_observer");
}
```

Use `ide_edit_member` to replace the test method. Rename it from
`onChannelInitialised_nonCaseChannel_noRegistration` to
`onChannelInitialised_nonCaseChannel_registersBackend`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClaudonyChannelBackendTest#onChannelInitialised_nonCaseChannel_registersBackend`
Expected: FAIL — `verifyNoInteractions` was removed but the production code
still returns early for non-case channels.

- [ ] **Step 3: Remove the prefix filter in production code**

In `ClaudonyChannelBackend.onChannelInitialised()`, remove line 50:
`if (!event.channelName().startsWith("case-")) return;`

Also update the class javadoc (line 17) — change "Self-registers for
{@code case-*} channels" to "Self-registers for all channels".

Use `ide_edit_member` on `onChannelInitialised` to replace the method body:

```java
void onChannelInitialised(@Observes ChannelInitialisedEvent event) {
    gateway.deregisterBackend(event.channelId(), BACKEND_ID);
    gateway.registerBackend(event.channelId(), this, "human_observer");
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClaudonyChannelBackendTest`
Expected: all 8 tests PASS

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/claudony/server/ClaudonyChannelBackend.java \
       app/src/test/java/io/casehub/claudony/server/ClaudonyChannelBackendTest.java
git commit -m "feat(#177): broaden ClaudonyChannelBackend to register for all channels

Remove the case- prefix filter in onChannelInitialised() so all channels
participate in ChannelEventBus — required for general-purpose chat rooms.

Refs #177"
```

---

### Task 2: Add POST /api/mesh/channels endpoint

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/MeshResource.java`
  (add `CreateChannelRequest` record, `createChannel` method, inject `QhorusEntityMapper`)
- Modify: `app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java`
  (add channel creation tests)

**Interfaces:**
- Consumes: `ChannelService.findOrCreate(ChannelCreateRequest)` → `FindOrCreateResult(Channel, boolean)`,
  `QhorusEntityMapper.toChannelDetail(Channel, long, Optional<ChannelConnectorBinding>)` → `ChannelDetail`,
  `ChannelCreateRequest.builder(String name)` → builder chain
- Produces: `POST /api/mesh/channels` → 201 + ChannelDetail | 409 + ChannelDetail | 400 + error message

- [ ] **Step 1: Write failing tests for the endpoint**

Add these tests to `MeshResourceTest.java` using `ide_insert_member`:

```java
@Test
void createChannel_validRequest_returns201() {
    given()
        .contentType("application/json")
        .body("""
            {"name": "team/engineering", "description": "Engineering chat"}
            """)
    .when()
        .post("/api/mesh/channels")
    .then()
        .statusCode(201)
        .body("name", equalTo("team/engineering"))
        .body("description", equalTo("Engineering chat"))
        .body("messageCount", equalTo(0));
}

@Test
void createChannel_duplicateName_returns409() {
    var body = """
        {"name": "team/dupe", "description": "First"}
        """;
    given().contentType("application/json").body(body)
        .when().post("/api/mesh/channels")
        .then().statusCode(201);

    given().contentType("application/json").body(body)
        .when().post("/api/mesh/channels")
        .then().statusCode(409);
}

@Test
void createChannel_missingName_returns400() {
    given()
        .contentType("application/json")
        .body("""
            {"description": "No name"}
            """)
    .when()
        .post("/api/mesh/channels")
    .then()
        .statusCode(400);
}

@Test
void createChannel_casePrefix_returns400() {
    given()
        .contentType("application/json")
        .body("""
            {"name": "case-fake/work"}
            """)
    .when()
        .post("/api/mesh/channels")
    .then()
        .statusCode(400)
        .body(containsString("case-"));
}

@Test
void createChannel_withAllowedTypes_parsesCorrectly() {
    given()
        .contentType("application/json")
        .body("""
            {"name": "team/typed", "allowedTypes": ["QUERY", "COMMAND"]}
            """)
    .when()
        .post("/api/mesh/channels")
    .then()
        .statusCode(201)
        .body("name", equalTo("team/typed"))
        .body("allowedTypes", containsString("QUERY"));
}

@Test
void createChannel_defaultSemantic_isAppend() {
    given()
        .contentType("application/json")
        .body("""
            {"name": "team/defaultsem"}
            """)
    .when()
        .post("/api/mesh/channels")
    .then()
        .statusCode(201)
        .body("semantic", equalTo("APPEND"));
}

@Test
void createChannel_invalidSemantic_returns400() {
    given()
        .contentType("application/json")
        .body("""
            {"name": "team/badsem", "semantic": "INVALID"}
            """)
    .when()
        .post("/api/mesh/channels")
    .then()
        .statusCode(400);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MeshResourceTest#createChannel_validRequest_returns201`
Expected: FAIL — 404 (no endpoint)

- [ ] **Step 3: Add the request record to MeshResource**

Use `ide_insert_member` to add a request record inside `MeshResource`,
after the existing `PostMessageRequest` record:

```java
record CreateChannelRequest(
        String name,
        String description,
        String semantic,
        java.util.Set<String> allowedTypes,
        java.util.Set<String> deniedTypes) {}
```

- [ ] **Step 4: Inject QhorusEntityMapper**

Use `ide_insert_member` to add the field after the existing
`channelService` field (line 60):

```java
@Inject QhorusEntityMapper entityMapper;
```

- [ ] **Step 5: Implement the endpoint method**

Use `ide_insert_member` to add the method after the `channels()` method
(line 104), position `after`, anchor `channels`:

```java
@POST
@Path("/channels")
@Consumes(MediaType.APPLICATION_JSON)
public Response createChannel(CreateChannelRequest req) {
    if (req == null || req.name() == null || req.name().isBlank()) {
        return Response.status(400).entity("name is required").build();
    }
    if (req.name().startsWith("case-")) {
        return Response.status(400)
                .entity("'case-' prefix is reserved for case-scoped channels").build();
    }

    ChannelSemantic semantic;
    try {
        semantic = req.semantic() != null
                ? ChannelSemantic.valueOf(req.semantic().toUpperCase())
                : ChannelSemantic.APPEND;
    } catch (IllegalArgumentException e) {
        return Response.status(400).entity("invalid semantic: " + req.semantic()).build();
    }

    Set<MessageType> allowedTypes;
    try {
        allowedTypes = req.allowedTypes() != null
                ? req.allowedTypes().stream()
                      .map(s -> MessageType.valueOf(s.toUpperCase()))
                      .collect(java.util.stream.Collectors.toSet())
                : null;
    } catch (IllegalArgumentException e) {
        return Response.status(400).entity("invalid allowedTypes value: " + e.getMessage()).build();
    }

    Set<MessageType> deniedTypes;
    try {
        deniedTypes = req.deniedTypes() != null
                ? req.deniedTypes().stream()
                      .map(s -> MessageType.valueOf(s.toUpperCase()))
                      .collect(java.util.stream.Collectors.toSet())
                : null;
    } catch (IllegalArgumentException e) {
        return Response.status(400).entity("invalid deniedTypes value: " + e.getMessage()).build();
    }

    try {
        var createReq = ChannelCreateRequest.builder(req.name())
                .description(req.description())
                .semantic(semantic)
                .allowedTypes(allowedTypes)
                .deniedTypes(deniedTypes)
                .build();

        var result = channelService.findOrCreate(createReq);
        var detail = entityMapper.toChannelDetail(result.channel(), 0, java.util.Optional.empty());
        int status = result.wasCreated() ? 201 : 409;
        return Response.status(status).entity(detail).build();
    } catch (IllegalArgumentException e) {
        return Response.status(400).entity(e.getMessage()).build();
    }
}
```

- [ ] **Step 6: Add required imports**

Use `ide_reformat_code` on MeshResource.java to auto-optimize imports.
The method uses: `ChannelSemantic`, `ChannelCreateRequest`, `FindOrCreateResult`,
`QhorusEntityMapper`, `Set`, `MessageType`, `Collectors`, `Optional`.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MeshResourceTest`
Expected: all tests PASS (existing + 7 new)

- [ ] **Step 8: Run full test suite to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: all 596+ tests PASS across all 3 modules

- [ ] **Step 9: Commit**

```
git add app/src/main/java/io/casehub/claudony/server/MeshResource.java \
       app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java
git commit -m "feat(#177): add POST /api/mesh/channels for general-purpose channel creation

Thin pass-through to ChannelService.findOrCreate(). Returns 201/ChannelDetail
on creation, 409 on duplicate. Rejects case- prefix (reserved for engine SPI).
Validates semantic, allowedTypes, deniedTypes with 400 on bad input.

Refs #177"
```
