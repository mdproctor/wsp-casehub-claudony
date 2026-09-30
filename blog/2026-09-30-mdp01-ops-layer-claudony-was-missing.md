---
layout: post
title: "The Ops Layer Claudony Was Missing"
date: 2026-09-30
entry_type: note
subtype: diary
projects: [casehubio/claudony]
tags: [pool-management, rest-api, dashboard, iotdb, event-broadcasting]
series: feat/208-ops-pool-provisioning
---

Pool provisioning has been accumulating infrastructure for weeks — scaling policies, eviction, fleet integration tests, real tmux sessions. But it was all engine-room work. No API. No dashboard. No way for an operator to see what the pool is doing, let alone change it.

This session built the operator surface: a REST API at `/api/pools` with full CRUD (list, detail, sessions, suspend, resume, destroy, patch capacity, patch scaling), a Pools dashboard tab with live status, and the event pipeline to push scaling decisions to connected clients in real time.

The interesting discovery was about push infrastructure. The plan called for adding `casehub-pages-push-runtime` as a dependency — it provides `EventBroadcaster`, which is the pages platform's topic-based delivery mechanism. Thirty seconds into the Quarkus augmentation: `AmbiguousResolutionException`. Both `PushProducers` (from pages-push-runtime) and `QhorusPushInfrastructure` (from qhorus-push) produce the same CDI beans — `EventBroadcaster`, `TopicRegistry`, `EventStore`. Qhorus already embeds the entire push stack. The new dependency wasn't needed at all; Qhorus had already solved the problem.

The fix was to drop `casehub-pages-push-runtime` entirely and inject `EventBroadcaster` from Qhorus's existing infrastructure. `PoolEventEmitterProducer` wraps it via `EventBroadcaster::broadcast` — one CDI producer, no ambiguity. The `PoolEventEmitter` itself lives in the casehub module with a `Broadcaster` functional interface, so it has no dependency on pages or Qhorus at all. `ScalingScheduler` receives it via `Instance<PoolEventEmitter>` — present when the app module wires it, absent in pure unit tests.

The IoTDB integration follows the same decoupling pattern. `IoTDBFlatLabelAdapter` reads Micrometer meters and produces SQL insert statements through a `SqlWriter` functional interface. No IoTDB client JAR needed — the adapter is testable with a lambda that collects strings. The real IoTDB client comes later, when we actually deploy IoTDB. The pages `data-iotdb` module (new module in the pages repo) provides the `DataProvider` SPI implementation and an `IoTDBQueryTranslator` that converts `DataSetLookup` filter expressions to IoTDB SQL.

The code review caught a genuine bug: `CredentialRoleAugmentor` was calling `loadForTest()` — a package-private test helper that reads the entire credentials file — in production code. The fix added a proper `findRolesByUsername()` method. The robustness audit caught a subtler issue: that method was blocking on the event loop. `SecurityIdentityAugmentor.augment()` can run on a Vert.x event loop thread, but `findRolesByUsername()` reads from disk. Wrapping it in `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` — the same pattern the existing `findByUsername()` already uses — fixed it.

The dashboard panel is a list-detail layout. Sidebar shows pools with health dots and fill ratios. Detail view has KPI cards (active, idle, min, max), a capacity bar, a sessions table with suspend/resume/destroy actions, scaling status with last decision and cooldown, and placeholder sections for charts and event log. Charts need IoTDB end-to-end; event log needs WebSocket subscription to the EventBroadcaster topics. Both are wired — the plumbing is there, the endpoints just need a live IoTDB and a connected browser.

What this opens up: the pool panel is the first piece of Claudony's ops perspective — seeing and controlling the infrastructure that runs agents, not just the agents themselves. IoTDB as the platform TSDB (pool metrics are the first consumer) sets up the pattern for casehub-iot and any other component that wants time-series storage. The EventBroadcaster wiring proves that Qhorus push infrastructure serves Claudony's own operational needs, not just agent mesh communication.
