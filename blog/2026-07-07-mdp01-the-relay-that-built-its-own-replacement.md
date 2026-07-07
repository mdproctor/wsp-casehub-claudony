---
layout: post
title: "The Relay That Built Its Own Replacement"
date: 2026-07-07
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [fleet, qhorus, postgresql, migration]
series: issue-168-qhorus-broadcaster-migration
---

Claudony had two custom classes for cross-node channel delivery in fleet deployments: `FleetMessageRelayObserver` (HTTP tick fan-out on every message dispatch) and `ChannelFleetBroadcaster` (HTTP channel sync on creation). Both operated through Claudony's own fleet infrastructure — `PeerRegistry`, `PeerClient`, `FleetKeyClientFilter`, and a pair of internal REST endpoints. It worked. It was also a simulation of something the database was already doing.

When Qhorus shipped `casehub-qhorus-postgres-broadcaster` (qhorus#162), the simulation became redundant. The broadcaster implements `ChannelActivityBroadcaster` via PostgreSQL LISTEN/NOTIFY — on message commit, `pg_notify`; on receive, `ChannelGateway.deliverRemote()` lazy-initializes channel backends and delivers. No HTTP fan-out, no peer client calls, no internal endpoints. The database handles the coordination.

The interesting decision was scope. I could have added the broadcaster dependency and left the relay as a fallback for H2 deployments. But H2 is inherently single-node — fleet relay only matters when PostgreSQL is the datasource, which is exactly when the broadcaster activates. There's no configuration where both are needed. So rather than maintaining a hybrid H2/PostgreSQL stack, I chose to migrate the qhorus datasource to PostgreSQL across all profiles. Dev mode and tests now use Quarkus Dev Services — Docker starts a PostgreSQL container automatically.

The net change was 535 deletions against 70 insertions. Six production files deleted, three test files, plus cleanup in `ClaudonyReactiveCaseChannelProvider` (the `CaseChannelCreatedEvent` it fired became dead code once its only observer was removed) and `PeerClient` (two relay methods gone, session federation and resize proxy untouched).

One thing caught us mid-verification. After the PostgreSQL migration, three tests failed with `NoSuchMethodError` on `OutboundMessage`'s constructor — the error showed the JVM looking for a `UUID` fifth parameter where the record now had `String correlationId`. The source code was correct. The JAR was correct. Claude investigated it as an API incompatibility for longer than it should have. The actual cause: stale `.class` files. `mvn test` without `clean` left bytecode compiled against an older SNAPSHOT where the compiler had inferred `null` as `UUID` for that position. `mvn clean test` — all green.

The proactive-to-lazy channel sync shift is worth noting. `ChannelFleetBroadcaster` eagerly pushed new channels to peers on creation. The broadcaster doesn't do that — it relies on `deliverRemote()` to lazy-initialize channels on first message delivery. In practice the gap is near-zero: `deliverRemote()` initializes and delivers in the same call, and messages are persisted in the shared database regardless of in-memory channel state. But the mental model is different — channels are no longer globally consistent at creation time, only at first delivery.
