---
title: "Three registries, one service"
date: 2026-10-10
author: mdp
projects: [casehub-claudony]
entry_type: note
subtype: diary
tags: [registry, platform, migration, architecture]
refs:
  - casehubio/claudony#268
  - casehubio/platform#569
  - casehubio/qhorus#489
---

Claudony had three registries maintaining their own state independently: `PeerRegistry` for fleet peer nodes, `AgentPoolDefinitionRegistry` for pool definitions, and `PoolMeshRegistrar` for bridging pool lifecycle to Qhorus instance registration. The platform now has a unified `RegistryService` SPI, and Qhorus already migrated its `InstanceManager`. Time for Claudony to follow.

The first design question was whether to use the strangler-fig pattern — extract an interface, create a `RegistryBacked*` adapter, wire via CDI priority — the same approach Qhorus took. I decided against it. Qhorus is a library with multiple consumers; the strangler-fig protects callers who can't all migrate at once. Claudony is a single application. We control the wiring. Pre-release means breaking changes cost nothing. Internal delegation — modifying each class in place to use `RegistryService` as its backing store — is simpler and produces fewer files.

The second question was more interesting. `AgentPoolDefinitionRegistry` stores `AgentPoolDefinition` — a deeply nested record with `AgentConfig`, `PoolConfig`, `ScalingConfig`, `BudgetConfig`. But `RegistryEntry.metadata` is `Map<String, String>`. Flattening that structure to string pairs loses type safety and makes queries awkward. The answer was a dual-store: RegistryService for discovery and relationships (minimal metadata — agent name, min/max capacity), the existing `ConcurrentHashMap` for the rich domain data. Each store does what it's good at.

**The self-review catch.** The initial design spec mapped circuit breaker state to `HealthStatus`: CLOSED→HEALTHY, OPEN→DOWN, HALF_OPEN→DEGRADED. That's wrong. `CircuitState` and `PeerHealth` are separate concerns. A peer can be DOWN with circuit OPEN (unreachable, stopped trying). CircuitState controls whether health checks are attempted; PeerHealth records the result. The mapping should be PeerHealth→HealthStatus (UP→HEALTHY, DOWN→DOWN, UNKNOWN→DEGRADED), with CircuitState staying on `PeerEntry` as the circuit breaker's internal FSM. Catching this in the design phase rather than in code review saved a rework cycle.

**The code review catch.** Claude caught that `loadPersistedPeers()` — the `@PostConstruct` method that loads peers from `~/.claudony/peers.json` — was putting peers into the `ConcurrentHashMap` but never registering them in `RegistryService`. This meant persisted peers were invisible to `resolveHealth()`, and subsequent `recordSuccess()` calls silently dropped the health update because `registryService.resolve(id)` returned empty. A one-line fix in the loading loop, but the kind of bug that would have been painful to diagnose in production — the health updates just... wouldn't happen, with no error.

The `PoolMeshRegistrar` migration was the cleanest of the three. The old version called `InstanceService.register/markOffline/deregister` to manage Qhorus presence. The new version uses `RegistryService.link/unlink` for pool→session containment and health updates for suspend/resume. Instance registration is now `RegistryBackedInstanceManager`'s job — PoolMeshRegistrar just adds the topology relationship on top. The test rewrite dropped the real-tmux integration tests in favour of focused unit tests against `InMemoryRegistryService`, cutting the test from 244 lines to 115.

What this opens up: all three entity types (peers, pools, agent-instances) are now discoverable through one service. A fleet dashboard that queries "show me everything in the fleet namespace" gets peers, pools, and their relationships in a single call. The relationship model (`pool contains session`) means pool topology is queryable without going through PoolService. And when the JPA-backed `RegistryService` lands, all this state persists across restarts for free.
