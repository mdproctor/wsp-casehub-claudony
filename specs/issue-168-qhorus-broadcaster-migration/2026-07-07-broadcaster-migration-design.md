# Migrate from FleetMessageRelayObserver to casehub-qhorus-postgres-broadcaster

**Issue:** casehubio/claudony#168
**Date:** 2026-07-07
**Branch:** issue-168-qhorus-broadcaster-migration

## Context

Claudony has two custom classes that handle cross-node channel delivery in fleet deployments:

- **`FleetMessageRelayObserver`** — a Qhorus `MessageObserver` that HTTP-relays a channel-name tick to every healthy fleet peer on message dispatch
- **`ChannelFleetBroadcaster`** — a CDI observer that HTTP-syncs new channels to peers on creation

Both use Claudony's fleet infrastructure (`PeerRegistry`, `PeerClient`, `FleetKeyClientFilter`, `ChannelSyncResource` endpoints) to simulate cross-node delivery via REST.

Qhorus now ships `casehub-qhorus-postgres-broadcaster` (casehubio/qhorus#162), which implements the `ChannelActivityBroadcaster` SPI via PostgreSQL LISTEN/NOTIFY. It activates by classpath presence (`@Alternative @Priority(1)`) and handles cross-node delivery natively — no application-level relay needed.

The broadcaster requires PostgreSQL. Claudony currently uses H2 for the qhorus datasource. Since fleet deployments require a shared database (PostgreSQL), and there's no point maintaining a hybrid H2/PostgreSQL stack, this migration also switches the qhorus datasource to PostgreSQL across all profiles.

## Approach — Clean removal + PostgreSQL migration

**Atomicity constraint:** §1 (dependency addition) and §3 (PostgreSQL migration) must land in the same commit. `PostgresChannelActivityBroadcaster` injects `@ReactiveDataSource("qhorus") PgPool pool` — if the broadcaster JAR is on the classpath while the datasource is still H2, the `PgPool` injection fails and Quarkus startup crashes. There is no configuration guard; activation is by classpath presence.

### 1. Dependency changes

**Add to `app/pom.xml`:**
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-qhorus-postgres-broadcaster</artifactId>
  <version>${casehub-qhorus.version}</version>
</dependency>
```

**Remove from `app/pom.xml`:**
- `quarkus-reactive-h2-client` (reactive H2 driver — no longer needed)
- `quarkus-jdbc-h2` (JDBC H2 driver — no longer needed)

**Keep:**
- `quarkus-jdbc-postgresql` (already present)
- `quarkus-reactive-pg-client` (already present or pulled transitively)

### 2. Relay removal

**Delete production code:**
- `FleetMessageRelayObserver.java`
- `ChannelFleetBroadcaster.java`
- `ChannelSyncResource.java`
- `ChannelNotifyRequest.java`
- `ChannelSyncRequest.java`
- `CaseChannelCreatedEvent.java` — the only observer (`ChannelFleetBroadcaster`) is deleted; the event becomes dead code

**Clean up `ClaudonyReactiveCaseChannelProvider.java`:**
- Remove `Event<CaseChannelCreatedEvent>` constructor parameter and field
- Remove `channelCreatedEvent.fire(...)` call in `createQhorusChannel()` (line 193)

**Clean up `PeerClient.java`:**
- Remove `notifyChannel()` and `syncChannel()` methods and their JAX-RS annotations
- Keep session federation, ping, resize proxy methods

**Delete test code:**
- `FleetMessageRelayObserverTest.java` (3 tests)
- `ChannelFleetBroadcasterTest.java` (3 tests)
- `ChannelSyncResourceTest.java` (4 tests)

**Clean up `ClaudonyReactiveCaseChannelProviderTest.java`:**
- Remove `openChannel_firesCaseChannelCreatedEvent()` test method and `channelCreatedEvent` mock field

**Unchanged:**
- `ClaudonyChannelBackend` — broadcaster delivers into it via `ChannelGateway.deliverRemote()` → `post()` → `channelEventBus.emit()`
- `ChannelEventBus` — still the SSE fan-out mechanism
- `PeerRegistry`, `PeerResource`, `PeerClient` (minus two methods) — fleet health, session federation, proxy resize
- All fleet auth (`FleetKeyService`, `FleetKeyAuth`, `FleetKeyClientFilter`)

**Behavioral change — proactive sync replaced by lazy initialization:**
Deleting `ChannelFleetBroadcaster` removes proactive channel-creation sync to fleet peers. Cross-node delivery now relies entirely on `ChannelGateway.deliverRemote()`, which lazy-initializes unknown channels on first remote delivery (`if (!registry.containsKey(channelId)) { initChannel(...) }`). This is correct because:
1. `deliverRemote()` lazy-initializes the channel AND delivers the message in the same call — no delivery gap
2. The broadcaster's NOTIFY/LISTEN mechanism triggers `deliverRemote()` on all non-sending nodes for every message
3. Messages are persisted in the shared database regardless of in-memory channel state

The practical impact is near-zero: users open channels via the UI, which triggers `initChannel()` locally. The only theoretical gap is a locally-sent message on a node that hasn't initialized the channel — the message persists but local SSE delivery waits for the next heartbeat. This is bounded, configurable, and acceptable for a system already designed around eventually-consistent SSE push.

### 3. PostgreSQL migration for qhorus datasource

**`application.properties` — default (production) profile:**

Remove:
```properties
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.reactive.url=h2:file:~/.claudony/qhorus
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:~/.claudony/qhorus;DB_CLOSE_ON_EXIT=FALSE;AUTO_SERVER=TRUE
```

Add:
```properties
quarkus.datasource.qhorus.db-kind=postgresql
%prod.quarkus.datasource.qhorus.reactive.url=postgresql://localhost:5432/claudony-qhorus
%prod.quarkus.datasource.qhorus.jdbc.url=jdbc:postgresql://localhost:5432/claudony-qhorus
```

**`application.properties` — dev profile:**

Remove:
```properties
%dev.quarkus.datasource.qhorus.reactive.url=h2:mem:claudony-qhorus-dev
%dev.quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:claudony-qhorus-dev;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%dev.quarkus.flyway.qhorus.migrate-at-start=false
%dev.quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
```

Add:
```properties
%dev.quarkus.hibernate-orm.qhorus.database.generation=update
```

Dev Services auto-starts PostgreSQL when no URL is configured. Flyway inherits `migrate-at-start=true` from the default profile. Hibernate `update` fills gaps for JPA entities without Flyway migrations (e.g. `case_ledger_entry`).

Unchanged: `quarkus.datasource.qhorus.reactive=true` (already present).

Prod URLs go under `%prod.` prefix only. Docker is now required for dev mode and tests.

### 4. Test infrastructure

**Test `application.properties` — remove:**
```properties
%test.quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:claudony-qhorus-test;DB_CLOSE_ON_EXIT=FALSE
%test.quarkus.datasource.qhorus.reactive.url=h2:mem:claudony-qhorus-test
%test.quarkus.flyway.qhorus.migrate-at-start=false
%test.quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
%test.quarkus.hibernate-orm.qhorus.sql-load-script=import-qhorus.sql
```

**Test `application.properties` — add:**
```properties
%test.quarkus.hibernate-orm.qhorus.database.generation=update
```

**Rationale:**
- H2 URLs removed — Dev Services provides PostgreSQL automatically
- Flyway inherits `migrate-at-start=true` from default profile — Flyway creates the base Qhorus + ledger schema including `ledger_subject_sequence`
- `import-qhorus.sql` no longer needed — Flyway migration `V1000__ledger_base_schema.sql` creates `ledger_subject_sequence`; delete the file
- Hibernate `update` fills gaps for JPA entities without Flyway migrations

**`%reactive-pg` test profile:**
Left as-is. `ClaudonyReactiveCaseChannelProviderPostgresIT` uses its own `PostgresTestResource` — cleanup is separate.

Existing test cleanup patterns unchanged:
- `InMemoryChannelStore` / `InMemoryMessageStore` + `clear()` in `@AfterEach`
- `channelStore.put(new Channel())` for test setup

### 5. Documentation updates

**ARC42STORIES.MD:**
- §3 Context diagram (line 103): remove `/api/internal/channels/sync` endpoint reference
- §5 Building Block View L3 (line 187): remove `ChannelSyncResource`, `ChannelFleetBroadcaster`, `FleetMessageRelayObserver`; add `casehub-qhorus-postgres-broadcaster` as external dependency
- §6 Runtime View Scenario 2 (lines 268-272): replace `FleetMessageRelayObserver` HTTP relay flow with LISTEN/NOTIFY broadcaster flow
- §7 Deployment View (line 328): change "H2 single-node / PostgreSQL multi-node" → "PostgreSQL all profiles"
- L3 Key files (line 786): remove `FleetMessageRelayObserver.java`, `ChannelFleetBroadcaster.java`
- §12 Risks (line 1083): mark "Shared H2 qhorus datasource is single-instance" as RESOLVED

**docs/DESIGN.md:**
- Line 123: remove `ChannelFleetBroadcaster` reference
- Line 125: remove `FleetMessageRelayObserver` reference
- Line 321: replace `FleetMessageRelayObserver` flow with broadcaster description

## Test impact

- **Removed:** 11 tests (relay observer 3, fleet broadcaster 3, sync resource 4, channel-created event assertion 1)
- **Expected baseline:** Verify at implementation time — ARC42STORIES §11 reports 587 as of 2026-06-20; additional tests may have been added since
- **New requirement:** Docker must be running for `mvn test`
