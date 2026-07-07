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
- `quarkus-reactive-h2-client` (no longer needed)

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

**Clean up `PeerClient.java`:**
- Remove `notifyChannel()` and `syncChannel()` methods and their JAX-RS annotations
- Keep session federation, ping, resize proxy methods

**Delete test code:**
- `FleetMessageRelayObserverTest.java` (3 tests)
- `ChannelFleetBroadcasterTest.java` (3 tests)
- `ChannelSyncResourceTest.java` (4 tests)

**Unchanged:**
- `ClaudonyChannelBackend` — broadcaster delivers into it via `ChannelGateway.deliverRemote()` → `post()` → `channelEventBus.emit()`
- `ChannelEventBus` — still the SSE fan-out mechanism
- `PeerRegistry`, `PeerResource`, `PeerClient` (minus two methods) — fleet health, session federation, proxy resize
- All fleet auth (`FleetKeyService`, `FleetKeyAuth`, `FleetKeyClientFilter`)

### 3. PostgreSQL migration for qhorus datasource

**`application.properties` — replace H2 config with:**
```properties
quarkus.datasource.qhorus.db-kind=postgresql
quarkus.datasource.qhorus.reactive=true
# No default URL — Quarkus Dev Services auto-starts PostgreSQL for dev and test
%prod.quarkus.datasource.qhorus.reactive.url=postgresql://localhost:5432/claudony-qhorus
%prod.quarkus.datasource.qhorus.jdbc.url=jdbc:postgresql://localhost:5432/claudony-qhorus
```

Prod URLs go under `%prod.` prefix only. Dev and test profiles see `db-kind=postgresql` with no URL → Dev Services starts a PostgreSQL container automatically.

Flyway migrations (`quarkus.flyway.qhorus.migrate-at-start=true`) already support PostgreSQL — no migration script changes needed.

Docker is now required for dev mode and tests.

### 4. Test infrastructure

Dev Services replaces all H2-specific qhorus datasource config. No manual `PostgresTestResource` or Testcontainers needed for the default test run.

Existing test cleanup patterns unchanged:
- `InMemoryChannelStore` / `InMemoryMessageStore` + `clear()` in `@AfterEach`
- `channelStore.put(new Channel())` for test setup

`ClaudonyReactiveCaseChannelProviderPostgresIT` (manual `PostgresTestResource`) left as-is — cleanup is separate.

### 5. Ledger API import migration

When updating the qhorus/ledger SNAPSHOT, the latest casehub-ledger has a package migration (`runtime.model` → `api.model`, `runtime.repository` → `api.spi`).

**`application.properties`:**
- `quarkus.hibernate-orm.qhorus.packages` — change `io.casehub.ledger.runtime.model` → `io.casehub.ledger.api.model`

**Java imports (if moved):**
- `JpaCaseLineageQuery.java` — `io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit`
- `ClaudonyLedgerEventCaptureTest.java` — same

Already correct: `ClaudonyLedgerEventCapture.java`, `CaseLineageQueryIntegrationTest.java` — already use `api.model` and `api.spi`.

## Test impact

- **Removed:** 10 tests (relay observer 3, fleet broadcaster 3, sync resource 4)
- **Expected baseline:** 590 (600 − 10)
- **New requirement:** Docker must be running for `mvn test`
