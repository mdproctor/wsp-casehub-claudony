# Broadcaster Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #168 — chore: migrate from FleetMessageRelayObserver to casehub-qhorus-postgres-broadcaster
**Issue group:** #168

**Goal:** Replace Claudony's custom HTTP fleet relay with Qhorus's native PostgreSQL LISTEN/NOTIFY broadcaster, and migrate the qhorus datasource from H2 to PostgreSQL across all profiles.

**Architecture:** The broadcaster (`casehub-qhorus-postgres-broadcaster`) implements `ChannelActivityBroadcaster` via PostgreSQL LISTEN/NOTIFY. It activates by classpath presence (`@Alternative @Priority(1)`), displacing `NoOpChannelActivityBroadcaster`. On message commit, it fires `pg_notify`; on receive, it calls `ChannelGateway.deliverRemote()` which lazy-initializes channel backends and delivers to `ClaudonyChannelBackend.post()` → `channelEventBus.emit()` → SSE. This replaces the custom `FleetMessageRelayObserver` HTTP fan-out and `ChannelFleetBroadcaster` channel sync.

**Tech Stack:** Java 21, Quarkus 3.32.2, PostgreSQL, Quarkus Dev Services, casehub-qhorus 0.2-SNAPSHOT

## Global Constraints

- Dependency addition and PostgreSQL migration must land atomically — `PostgresChannelActivityBroadcaster` injects `@ReactiveDataSource("qhorus") PgPool`; H2 datasource with broadcaster on classpath crashes at startup.
- Docker must be running for `mvn test` and `quarkus:dev` after this migration.
- `LedgerPersistenceUnit` has NOT moved from `io.casehub.ledger.runtime.persistence` — do not change those imports.
- Hibernate ORM packages config must update `io.casehub.ledger.runtime.model` → `io.casehub.ledger.api.model` (ledger API migration).

---

## File Map

### Delete (production)
- `app/src/main/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserver.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelFleetBroadcaster.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncResource.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelNotifyRequest.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncRequest.java`
- `core/src/main/java/io/casehub/claudony/server/CaseChannelCreatedEvent.java`

### Delete (test)
- `app/src/test/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserverTest.java` (3 tests)
- `app/src/test/java/io/casehub/claudony/server/fleet/ChannelFleetBroadcasterTest.java` (3 tests)
- `app/src/test/java/io/casehub/claudony/server/fleet/ChannelSyncResourceTest.java` (4 tests)
- `app/src/test/resources/import-qhorus.sql`

### Modify (production)
- `app/pom.xml` — add broadcaster dep, remove H2 deps
- `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java` — remove `Event<CaseChannelCreatedEvent>` field, constructor param, and `fire()` call
- `app/src/main/java/io/casehub/claudony/server/fleet/PeerClient.java` — remove `syncChannel()` and `notifyChannel()` methods (lines 46-54)
- `app/src/main/resources/application.properties` — H2→PostgreSQL datasource config + ledger package fix

### Modify (test)
- `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java` — remove `channelCreatedEvent` mock field and `openChannel_firesCaseChannelCreatedEvent()` test
- `app/src/test/resources/application.properties` — remove H2 test datasource config, remove `import-qhorus.sql` reference

---

### Task 1: PostgreSQL migration + dependency swap (atomic)

This task must be a single commit — the broadcaster JAR + PostgreSQL datasource must arrive together.

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/main/resources/application.properties` (lines 97-112)
- Modify: `app/src/test/resources/application.properties` (lines 117-126)
- Delete: `app/src/test/resources/import-qhorus.sql`

**Interfaces:**
- Consumes: nothing
- Produces: PostgreSQL datasource for all profiles; broadcaster on classpath

- [ ] **Step 1: Verify Docker is running and test baseline**

```bash
docker info >/dev/null 2>&1 && echo "Docker OK" || echo "Docker NOT running"
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SmokeTest
```
Expected: PASS — confirms current baseline works.

- [ ] **Step 2: Modify `app/pom.xml` — add broadcaster, remove H2 deps**

Add after the `casehub-qhorus-testing` dependency:
```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-postgres-broadcaster</artifactId>
      <version>${casehub-qhorus.version}</version>
    </dependency>
```

Remove these two dependencies:
```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
    </dependency>
```
```xml
    <dependency>
      <groupId>io.quarkiverse.quarkus-reactive-h2-client</groupId>
      <artifactId>quarkus-reactive-h2-client</artifactId>
    </dependency>
```

- [ ] **Step 3: Modify `app/src/main/resources/application.properties` — H2 → PostgreSQL**

Replace lines 97-112 with:
```properties
# ── Qhorus named persistence unit ────────────────────────────────────────────
# PostgreSQL for all profiles. Dev and test: Quarkus Dev Services auto-starts a container.
# Production: configure explicit URLs via %prod. prefix.
quarkus.datasource.qhorus.db-kind=postgresql
quarkus.datasource.qhorus.reactive=true
%prod.quarkus.datasource.qhorus.reactive.url=postgresql://localhost:5432/claudony-qhorus
%prod.quarkus.datasource.qhorus.jdbc.url=jdbc:postgresql://localhost:5432/claudony-qhorus
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.api.model,io.casehub.ledger.model
quarkus.flyway.qhorus.migrate-at-start=true
%dev.quarkus.hibernate-orm.qhorus.database.generation=update
```

Key changes:
- `db-kind=h2` → `db-kind=postgresql`
- H2 file/mem URLs removed; explicit URLs under `%prod.` only
- Dev `drop-and-create` replaced with `update` (Flyway handles schema)
- `io.casehub.ledger.runtime.model` → `io.casehub.ledger.api.model` (ledger API migration)
- Dev H2 in-memory URLs removed (Dev Services provides PostgreSQL)
- Dev `flyway.migrate-at-start=false` removed (inherits `true` from default)

- [ ] **Step 4: Modify `app/src/test/resources/application.properties` — remove H2 test config**

Remove lines 117-126 (the H2 test datasource block):
```properties
# Qhorus named datasource — in-memory H2 for tests (no file path, no AUTO_SERVER)
%test.quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:claudony-qhorus-test;DB_CLOSE_ON_EXIT=FALSE
# Reactive H2 client also needs an in-memory URL (no jdbc: prefix, no AUTO_SERVER)
%test.quarkus.datasource.qhorus.reactive.url=h2:mem:claudony-qhorus-test
# Flyway disabled in tests — schema managed by Hibernate database.generation instead
%test.quarkus.flyway.qhorus.migrate-at-start=false
%test.quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
# Run after drop-and-create to create native SQL tables not in JPA entity model
# (e.g. ledger_subject_sequence — required by LedgerEntryRepository.save()).
%test.quarkus.hibernate-orm.qhorus.sql-load-script=import-qhorus.sql
```

Replace with:
```properties
# Qhorus named datasource — Dev Services auto-starts PostgreSQL container.
# Flyway inherits migrate-at-start=true from default profile.
%test.quarkus.hibernate-orm.qhorus.database.generation=update
```

- [ ] **Step 5: Delete `app/src/test/resources/import-qhorus.sql`**

No longer needed — Flyway migration `V1000__ledger_base_schema.sql` creates `ledger_subject_sequence`.

- [ ] **Step 6: Run tests to verify PostgreSQL migration**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=SmokeTest
```
Expected: PASS — Dev Services starts PostgreSQL, Flyway runs migrations, app starts.

If SmokeTest passes, run the full test suite:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```
Expected: All existing tests pass (relay tests still present, exercising the old code path — they will be removed in Task 2).

- [ ] **Step 7: Commit**

```bash
git add app/pom.xml app/src/main/resources/application.properties app/src/test/resources/application.properties
git rm app/src/test/resources/import-qhorus.sql
git commit -m "feat(#168): migrate qhorus datasource to PostgreSQL and add broadcaster dependency

Atomically switch qhorus datasource from H2 to PostgreSQL across all profiles
and add casehub-qhorus-postgres-broadcaster dependency. These must land together
because the broadcaster injects @ReactiveDataSource(\"qhorus\") PgPool — H2
datasource with broadcaster on classpath crashes at startup.

- Remove quarkus-jdbc-h2 and quarkus-reactive-h2-client dependencies
- Production URLs under %prod. prefix; dev/test use Quarkus Dev Services
- Flyway now runs in all profiles (creates ledger_subject_sequence)
- Delete import-qhorus.sql (superseded by Flyway V1000)
- Fix Hibernate packages: ledger.runtime.model → ledger.api.model

Refs #168"
```

---

### Task 2: Remove relay code and clean up references

**Files:**
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserver.java`
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/ChannelFleetBroadcaster.java`
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncResource.java`
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/ChannelNotifyRequest.java`
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncRequest.java`
- Delete: `core/src/main/java/io/casehub/claudony/server/CaseChannelCreatedEvent.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PeerClient.java` — remove lines 46-54 (syncChannel + notifyChannel methods and their imports)
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java` — remove `Event<CaseChannelCreatedEvent>` field (line 34), constructor params (lines 41, 60), and `fire()` call (line 193)
- Delete: `app/src/test/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserverTest.java`
- Delete: `app/src/test/java/io/casehub/claudony/server/fleet/ChannelFleetBroadcasterTest.java`
- Delete: `app/src/test/java/io/casehub/claudony/server/fleet/ChannelSyncResourceTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java` — remove `channelCreatedEvent` mock field (line 35) and `openChannel_firesCaseChannelCreatedEvent()` test method (line 259+)

**Interfaces:**
- Consumes: PostgreSQL datasource from Task 1
- Produces: Clean codebase with no relay infrastructure

- [ ] **Step 1: Delete production relay files**

Delete all 6 production files:
- `app/src/main/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserver.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelFleetBroadcaster.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncResource.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelNotifyRequest.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncRequest.java`
- `core/src/main/java/io/casehub/claudony/server/CaseChannelCreatedEvent.java`

- [ ] **Step 2: Clean up PeerClient.java**

Remove lines 46-54 from `app/src/main/java/io/casehub/claudony/server/fleet/PeerClient.java`:
```java
    @POST
    @Path("/internal/channels/sync")
    @Consumes(MediaType.APPLICATION_JSON)
    Response syncChannel(ChannelSyncRequest request);

    @POST
    @Path("/internal/channels/notify")
    @Consumes(MediaType.APPLICATION_JSON)
    Response notifyChannel(ChannelNotifyRequest request);
```

The interface retains `getSessions()` and `resize()` methods.

- [ ] **Step 3: Clean up ClaudonyReactiveCaseChannelProvider.java**

In `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`:

Remove the `channelCreatedEvent` field (line 34):
```java
    private final jakarta.enterprise.event.Event<io.casehub.claudony.server.CaseChannelCreatedEvent> channelCreatedEvent;
```

Remove the `channelCreatedEvent` parameter from both constructors (lines 41 and 60).

Remove the `channelCreatedEvent.fire(...)` call in `createQhorusChannel()` (line 193):
```java
                            new io.casehub.claudony.server.CaseChannelCreatedEvent(detail.id(), detail.name(), tenantContext.currentTenantId()));
```

- [ ] **Step 4: Delete test files**

Delete:
- `app/src/test/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserverTest.java`
- `app/src/test/java/io/casehub/claudony/server/fleet/ChannelFleetBroadcasterTest.java`
- `app/src/test/java/io/casehub/claudony/server/fleet/ChannelSyncResourceTest.java`

- [ ] **Step 5: Clean up ClaudonyReactiveCaseChannelProviderTest.java**

In `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java`:

Remove the `channelCreatedEvent` mock field (line 35):
```java
    private jakarta.enterprise.event.Event<io.casehub.claudony.server.CaseChannelCreatedEvent> channelCreatedEvent;
```

Remove the entire `openChannel_firesCaseChannelCreatedEvent()` test method starting at line 259 (including its `@Test` annotation and the assertion/captor logic through the end of the method).

Update any constructor call that passes `channelCreatedEvent` — remove that argument from the provider construction.

- [ ] **Step 6: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -DskipTests
```
Expected: BUILD SUCCESS — no dangling references to deleted classes.

- [ ] **Step 7: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```
Expected: All tests pass. Count should be previous total minus 11 (3 + 3 + 4 relay tests + 1 channel-created event test).

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#168): remove FleetMessageRelayObserver and fleet relay infrastructure

Delete custom HTTP relay code replaced by casehub-qhorus-postgres-broadcaster:
- FleetMessageRelayObserver (MessageObserver, HTTP tick fan-out)
- ChannelFleetBroadcaster (CDI observer, HTTP channel sync)
- ChannelSyncResource (/api/internal/channels/notify + /sync endpoints)
- CaseChannelCreatedEvent (dead code — only observer was ChannelFleetBroadcaster)
- ChannelNotifyRequest, ChannelSyncRequest DTOs
- PeerClient.syncChannel() and notifyChannel() methods

Clean up ClaudonyReactiveCaseChannelProvider: remove Event<CaseChannelCreatedEvent>
injection and fire() call — channel creation sync now handled by broadcaster's
lazy initChannel() via ChannelGateway.deliverRemote().

Removes 11 tests. Broadcaster handles cross-node delivery via PostgreSQL
LISTEN/NOTIFY — no application-level relay needed.

Refs #168"
```

---

### Task 3: Update documentation (ARC42STORIES.MD, DESIGN.md, CLAUDE.md)

**Files:**
- Modify: `ARC42STORIES.MD` — update fleet/relay references
- Modify: `docs/DESIGN.md` — update fleet component references
- Modify: `CLAUDE.md` — update test count, datasource config, project structure notes

**Interfaces:**
- Consumes: completed relay removal from Task 2
- Produces: accurate documentation reflecting the new architecture

- [ ] **Step 1: Read and update ARC42STORIES.MD**

Read the file and update these sections (line numbers from the spec):
- §3 Context diagram (~line 103): remove `/api/internal/channels/sync` endpoint reference
- §5 Building Block View L3 (~line 187): remove `ChannelSyncResource`, `ChannelFleetBroadcaster`, `FleetMessageRelayObserver`; add `casehub-qhorus-postgres-broadcaster` as external dependency
- §6 Runtime View Scenario 2 (~lines 268-272): replace `FleetMessageRelayObserver` HTTP relay flow with LISTEN/NOTIFY broadcaster flow
- §7 Deployment View (~line 328): change "H2 single-node / PostgreSQL multi-node" → "PostgreSQL all profiles"
- L3 Key files (~line 786): remove `FleetMessageRelayObserver.java`, `ChannelFleetBroadcaster.java`
- §12 Risks (~line 1083): mark "Shared H2 qhorus datasource is single-instance" as RESOLVED

- [ ] **Step 2: Read and update docs/DESIGN.md**

Read the file and remove references to:
- `ChannelFleetBroadcaster` (~line 123)
- `FleetMessageRelayObserver` (~line 125)
- Replace relay flow description (~line 321) with broadcaster description

- [ ] **Step 3: Update CLAUDE.md**

Update these sections:
- **Test Count and Status**: update baseline count (previous minus 11 relay tests, verify exact number after Task 2)
- **Configuration Properties**: change qhorus datasource description from H2 to PostgreSQL, note Docker requirement
- **Project Structure**: remove `ChannelFleetBroadcaster`, `FleetMessageRelayObserver` from the tree listing; remove `CaseChannelCreatedEvent` from core listing; add note about `casehub-qhorus-postgres-broadcaster` dependency
- **Known Issues and Quirks**: add Docker requirement for dev/test

- [ ] **Step 4: Commit**

```bash
git add ARC42STORIES.MD docs/DESIGN.md CLAUDE.md
git commit -m "docs(#168): update architecture docs for broadcaster migration

- ARC42STORIES.MD: remove relay references, add broadcaster, update deployment view
- DESIGN.md: remove fleet relay component descriptions
- CLAUDE.md: update test count, datasource config, project structure

Refs #168"
```

---

### Task 4: Platform coherence — update PLATFORM.md

**Files:**
- Modify: `../parent/docs/PLATFORM.md` — update capability ownership entries

**Interfaces:**
- Consumes: completed migration
- Produces: accurate platform docs

- [ ] **Step 1: Update PLATFORM.md capability ownership**

Line 416 currently reads:
> `FleetMessageRelayObserver` in claudony (`Scope.CLUSTER`) — relays a channel-name tick to all healthy fleet peers on every Qhorus message dispatch, enabling real-time SSE delivery across fleet nodes (claudony#118)

Update to reflect that `FleetMessageRelayObserver` has been removed and cross-node delivery is now handled by `casehub-qhorus-postgres-broadcaster` via the `ChannelActivityBroadcaster` SPI (line 417 already documents this correctly).

Line 433 ("Fleet management + peer discovery") — remove the `FleetMessageRelayObserver` mention from the description.

- [ ] **Step 2: Commit to parent repo**

```bash
git -C ../parent add docs/PLATFORM.md
git -C ../parent commit -m "docs(#168): update claudony fleet capability after broadcaster migration

FleetMessageRelayObserver removed from claudony — cross-node channel delivery
now handled by casehub-qhorus-postgres-broadcaster via ChannelActivityBroadcaster
SPI (PostgreSQL LISTEN/NOTIFY).

Refs casehubio/claudony#168"
```

- [ ] **Step 3: Update workspace HANDOFF for cross-repo commit**

Record the parent commit in the workspace HANDOFF.md per the cross-repo convention.
