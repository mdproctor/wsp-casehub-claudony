---
layout: post
title: "What the mock was hiding"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [quarkus, cdi, mutiny, reactive, casehub-engine, qhorus]
excerpt: "The @InjectMock in CaseEngineRoundTripTest was technically correct. It was also hiding a production bug. Removing it required an architecture session that went further than expected."
---

The `@InjectMock ClaudonyWorkerContextProvider` in `CaseEngineRoundTripTest` was placed there deliberately. The previous entry covered why the test was hard to write; this mock was the final workaround — calling the real context provider from the Vert.x IO thread would throw `BlockingOperationNotAllowedException`. The mock made the test green. The production bug remained.

Fixing it turned out to be a longer conversation than I expected.

The obvious fix — switch to reactive SPIs — was correct, but casehub-engine already had `ReactiveWorkerContextProvider`, `ReactiveWorkerProvisioner`, and `ReactiveCaseChannelProvider` defined. The question was what Claudony's implementations should look like, and how far to go.

The channel provider was where it got interesting. `ClaudonyCaseChannelProvider` calls Qhorus for channel management. Making it reactive meant either wrapping `QhorusMcpTools` on virtual threads (it returns blocking types) or injecting `ReactiveChannelService` directly from Qhorus's reactive stack. Qhorus has had a full reactive dual-stack since ADR-0003 — I'd just never looked at whether Claudony used it.

It didn't. It was calling `QhorusMcpTools` for channel operations, which is the MCP dispatch layer, not the service API. We were going through the external-facing tool class to do internal work.

Switching to `ReactiveChannelService` directly required activating the reactive Qhorus stack, which meant setting `quarkus.datasource.qhorus.reactive=true` in `application.properties` — a build-time property. Which meant H2 needed a reactive JDBC driver. H2 doesn't have one.

My first instinct was a two-variant design: one channel provider for H2 (virtual-thread offload, wrapping blocking calls), one for PostgreSQL (true reactive). I had the `@IfBuildProperty`/`@UnlessBuildProperty` scaffolding mentally outlined.

The user pushed back: "I don't want to design Claudony on H2's limitations."

That was the right call. H2 is a test database. Designing the production architecture around it would mean carrying the H2-safe variant indefinitely. I looked for whether a reactive H2 client existed — `quarkus-reactive-h2-client` does, it's a Vert.x JDBC-bridging adapter designed explicitly for testing, and it's compatible with Quarkus 3.32.2. One implementation, no conditional split.

We filed three Qhorus issues during the design session: `findByNamePrefix()` on `ReactiveChannelService` for efficient per-case channel listing (today it calls `listAll()` and filters in Java), the mechanics for consumers to activate `@Alternative` reactive beans without listing 15 class names in `quarkus.arc.selected-alternatives`, and a note about `quarkus-reactive-h2-client` potentially unlocking the `@Disabled` reactive service tests.

I also asked Claude to steelman the case against the qhorus direction — whether messaging transport abstraction even belongs there. The strongest counterargument wasn't the architecture, it was topology. Claudony embeds Qhorus: same JVM, LOCAL CDI scope, `ClaudonyChannelBackend` as a CDI bean works fine. But for a multi-node fleet, CDI delivery doesn't cross process boundaries. That's a separate problem, filed separately, and doesn't affect single-node deployments.

The implementation followed: `CaseLineageQuery` shifted to returning `Uni<List<WorkerSummary>>` with CDI self-injection for the `@Transactional` interceptor — calling `this.blockingMethod()` inside a Mutiny lambda bypasses the CDI proxy silently, which is easy to miss. `ClaudonyReactiveWorkerContextProvider` runs lineage and channel queries in parallel via `Uni.combine().all()`. `ClaudonyReactiveWorkerProvisioner` offloads the tmux `ProcessBuilder` call to a virtual-thread worker pool.

The engine's `tryProvision()` changed in casehub-engine — it now returns `Uni<Void>` and injects the reactive SPIs. The two call sites that previously fire-and-forgot now integrate into the reactive chain. Provisioning errors propagate instead of disappearing.

`CaseEngineRoundTripTest` now has no `@InjectMock` on the context provider. It runs the real `ClaudonyReactiveWorkerContextProvider` through the engine pipeline. The test passes.

The whole thing was also a reminder that what looks like a test fix often isn't. The mock was correct as a workaround. Removing it correctly required understanding the qhorus reactive stack, deciding where transport abstraction lives, picking the right H2 strategy, and updating the engine in a separate repo. That's the actual size of the problem — the mock just made it invisible.
