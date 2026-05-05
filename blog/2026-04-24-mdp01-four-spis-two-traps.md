---
layout: post
title: "Phase 12: Four SPIs, Two Traps"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [architecture, casehub, quarkus, cdi, jpa, config-mapping, dependency-management]
---

## The config mapping trap

The SPI implementations came together quickly. Four classes: `ClaudonyWorkerProvisioner` (tmux sessions), `ClaudonyCaseChannelProvider` (Qhorus channels), `ClaudonyWorkerContextProvider` (lineage), `ClaudonyWorkerStatusListener` (lifecycle updates to SessionRegistry).

Then we added `claudony.casehub.*` properties to `application.properties` and Quarkus died:

```
ConfigValidationException: SRCFG00050: claudony.casehub.enabled does not map to any root
```

The `@ConfigMapping` interface lives in the `claudony-casehub` JAR. The JAR has Jandex indexing. The main app has `quarkus.index-dependency` configured. Every other CDI mechanism — beans, producers, observers — works cleanly this way. Config mapping doesn't.

The issue is timing. SmallRye Config's strict validation fires during config source loading, before the library JAR's mapping registers as a config root. Properties with no registered root fail immediately — even though the mapping will register moments later.

The fix: remove the properties entirely. The `@ConfigMapping` injection works fine with empty/default values when no properties exist to trigger validation. Users who want to configure CaseHub add the properties themselves. It's counterintuitive, but it works.

## Repositories that aren't beans

The trickier discovery was `CaseLedgerEntryRepository`. The plan had `ClaudonyWorkerContextProvider` injecting it directly for lineage queries. It extends `JpaLedgerEntryRepository`, uses `@Inject EntityManager` — looks exactly like a CDI bean. It isn't. No `@ApplicationScoped`. At augmentation:

```
UnsatisfiedResolutionException: Unsatisfied dependency for type CaseLedgerEntryRepository
```

The fix: introduce `CaseLineageQuery` — a single-method interface. The default implementation, `EmptyCaseLineageQuery`, carries `@DefaultBean` and returns empty. The CDI graph stays satisfied. When a casehub datasource is configured later, a JPA-backed implementation replaces the default via `@Alternative`. The context provider's tests mock `CaseLineageQuery` directly, which is cleaner anyway.

## javap before writing

One technique worth naming. The `claudony-casehub` module depends on API JARs it doesn't own — `casehub:api` and `quarkus-qhorus`. The plan had wrong method signatures: `ChannelDetail.id()` when the actual accessor is `channelId()`, `createChannel` with three arguments when the real method takes four.

We caught these before writing a line of production code by running `javap` against the JARs:

```bash
javap -p -classpath ~/.m2/.../quarkus-qhorus-1.0.0-SNAPSHOT.jar \
  io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase\$ChannelDetail
```

The compiled output shows actual accessor names, constructor arities, and field types. It takes thirty seconds. Without it, you're writing test code against guesses.
