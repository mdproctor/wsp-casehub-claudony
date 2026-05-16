---
layout: post
title: "Five problems before the first assertion"
date: 2026-05-16
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [quarkus, cdi, testing, casehub-engine, jax-rs]
excerpt: "Closing the agent mesh epic required working through five separate infrastructure problems before the real integration test could be written. What they were and why they were non-obvious."
---

Getting a real engine integration test into Claudony meant working through five infrastructure problems before writing a single assertion. Here's what they were.

The goal was to replace a CDI-event stub with a test that actually exercises the engine — `CaseContextChangedEventHandler` evaluating a binding, calling `ClaudonyWorkerProvisioner.provision()`, completing via `WorkflowExecutionCompleted`, and verifying the ledger captures it. The full provisioning-to-lineage path, minus an actual tmux session.

**Problem one** was already resolved upstream: casehub-engine had been using `@ApplicationScoped` on its no-op SPI beans, which collided with Claudony's real implementations when `casehub-testing` was indexed. Switching to `@DefaultBean` made the no-ops yield automatically. That fix was a prerequisite for the entire approach.

**Problem two** hit immediately after: both `A2AResource` and `ReactiveA2AResource` from casehub-qhorus registered the same REST routes, causing Quarkus to reject the duplicate at startup. My first instinct was `quarkus.arc.exclude-types` — exclude the CDI beans, stop the endpoint registration. It doesn't work that way. The JAX-RS scanner runs independently of Arc and picks up `@Path` classes directly from the Jandex index. CDI exclusion has no effect on it. The real fix was `@IfBuildProperty`/`@UnlessBuildProperty` on the resource classes themselves, gated on `quarkus.datasource.qhorus.reactive`. That landed in qhorus and unblocked the rest.

**Problem three**: casehub-engine ships `application.properties` in its jar root. Quarkus treats any jar with `application.properties` as an application archive and scans all its classes — so a couple of dozen internal beans (handlers, reactors, orchestrators) became CDI candidates in the test context. Each needed explicit exclusion in `%test.quarkus.arc.exclude-types`.

**Problem four**: `casehub-testing` transitively pulls `casehub-persistence-memory`, which has abstract parent classes annotated with `@ApplicationScoped`. Jandex indexes them. Arc tries to create proxies for abstract classes and fails. More exclusions.

**Problem five** was the subtlest. The test's `CasehubEnabledProfile` needed to re-include `TestResearcherCase`, which the default test profile excludes because its `CaseHub` superclass injects `CaseHubRuntime` — unavailable in the default context. Claude and I discovered that returning `quarkus.arc.exclude-types` (unprefixed) from `getConfigOverrides()` completely replaces the `%test.quarkus.arc.exclude-types` from `application.properties` when the profile is active. It's not additive — it's a full replacement. The profile carries the entire exclusion list minus the beans it needs back.

After all that, the test itself is clean. We mock `TmuxService` to prevent real tmux sessions, fire `CONTEXT_CHANGED` into the engine, wait for `createSession()` to be called (which proves `tryProvision()` ran), publish `WorkflowExecutionCompleted`, and wait for `findCompletedWorkers()` to return a result. The first Awaitility wait verifies the provisioner was reached; the second verifies the ledger captured the completion and the lineage query can read it.

The one honest asterisk: `CaseHub.startCase()` isn't called. We fire `CONTEXT_CHANGED` directly because `CaseStartedEventHandler` calls `SchedulerService`, which tries to interact with Quartz from a Vert.x IO thread — and fails. Getting `startCase()` into the test path needs either a blocking-safe handler or a test-scoped `WorkerExecutionManager` that skips Quartz. That's filed and tracked.

The provisioner path is exercised, the ledger capture is verified, the lineage query is proven. That's what was needed to close the mesh epic.
