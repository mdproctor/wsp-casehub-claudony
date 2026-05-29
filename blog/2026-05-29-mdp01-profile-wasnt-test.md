---
layout: post
title: "The profile that wasn't %test"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [quarkus, testing, postgresql, reactive]
---

The core change was two lines. `ClaudonyReactiveCaseChannelProvider.listChannels()` was calling `listAll()` — every channel in the database, then filter client-side for the ones belonging to the case. `findByNamePrefix()` had shipped in qhorus recently. We swapped it in, removed the filter. The unit test stubs needed updating to match; done in fifteen minutes.

Then we started on the PostgreSQL integration test, and it took the rest of the day.

I assumed Quarkus Dev Services would handle it. Set `db-kind=postgresql`, `devservices.enabled=true` in a `%reactive-pg` profile, annotate the test class with `@TestProfile`. The eidos and qhorus repos both use this pattern — I had the examples in front of me.

The part I had wrong: `QuarkusTestProfile.getConfigProfile()` does not *add to* the `%test` profile. It *replaces it*. When the profile is `"reactive-pg"`, the active config is the production `application.properties` plus any `%reactive-pg.*` entries — nothing from `%test.*`. The production config has `quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:~/.claudony/qhorus`. Dev Services sees that URL, decides the datasource is already configured, and never starts a container.

Attempt one: override the URL with an empty string in the profile block. Dev Services did start the container — the logs confirmed it. But Quarkus had already deactivated the `AgroalDataSource` bean at augmentation time, before Dev Services could inject anything:

```
Datasource 'qhorus' was deactivated automatically because its URL is not set.
```

Attempt two: leave the URL out entirely and rely on `devservices.enabled=true` to force it. Quarkus did not override the production H2 URL. Flyway connected and failed immediately:

```
Driver does not support the provided URL: jdbc:h2:file:~/.claudony/qhorus
```

Attempt three: disable JDBC, use Hibernate schema generation. The `EntityManager` injection for `ClaudonyLedgerEventCapture` failed — no JDBC pool, no persistence unit, no beans.

`QuarkusTestResourceLifecycleManager.start()` was the solution. Its return value is applied at the highest priority in Quarkus config — above system properties, above profile config, above production config. Start the container, return the URLs:

```java
@Override
public Map<String, String> start() {
    postgres = new PostgreSQLContainer<>("postgres:17-alpine")
            .withDatabaseName("qhorus")...;
    postgres.start();
    return Map.of(
        "quarkus.datasource.qhorus.jdbc.url", postgres.getJdbcUrl(),
        "quarkus.datasource.qhorus.reactive.url", reactiveUrl, ...);
}
```

Augmentation sees a real PostgreSQL URL before deciding anything about bean activation. Flyway ran against the actual schema. The reactive path worked.

One more hurdle: `ReactiveChannelService.create()` calls `Panache.withTransaction()`, which requires an active Vert.x context. The JUnit thread does not have one:

```
java.lang.IllegalStateException: No current Vertx context found
```

`@RunOnVertxContext` on the test class runs each method on the event loop. `UniAsserter` parameter injection replaces the `.await()` call — tests return a `Uni<Void>` pipeline rather than blocking. Four tests, all green.

While running the full suite, the engine package refactoring surfaced. Some time after the branch started, the engine moved its internals from `engine.internal.*` to `engine.common.*`. Several test files had stale imports. `WorkloadProvider` had been removed entirely; there was a new `WorkerExecutionManager` SPI to satisfy instead. Fixed as a separate commit.

The original motivation for this branch — linking `CaseLedgerEntry.causedByEntryId` to the Qhorus message that triggered provisioning — did not ship. The right design needs the engine to fire a `WorkerStarted` lifecycle event carrying the message ledger entry ID. Doing it as a state bridge inside claudony would have been wrong. I filed the engine issue and will pick it up when the engine is ready.
