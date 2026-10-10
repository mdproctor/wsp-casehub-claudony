# Registry Migration Design — #268

Migrate PeerRegistry, AgentPoolDefinitionRegistry, and PoolMeshRegistrar
to the unified RegistryService SPI (platform#569).

**Pattern:** Internal delegation — modify existing classes in place to use
RegistryService as backing store. No interface extraction, no strangler-fig.
See decisions.md D1.

**Dependencies:** platform#569 (done), qhorus#489 (done).

---

## 1. PeerRegistry — internal delegation to RegistryService

### Current state

`PeerRegistry` is `@ApplicationScoped` with a `ConcurrentHashMap<String, PeerEntry>`.
It handles:
- Peer CRUD with source priority (CONFIG > MANUAL > MDNS)
- Circuit breaker state per peer (CLOSED/OPEN/HALF_OPEN with backoff)
- File persistence for non-CONFIG peers (`~/.claudony/peers.json`)
- Health recording (recordSuccess/recordFailure)
- Cached session lists per peer

### Design

Inject `RegistryService` into `PeerRegistry`. Replace the ConcurrentHashMap
with registry calls for storage. Domain logic stays in PeerRegistry/PeerEntry.

**Registry entry mapping:**
- `id`: peer ID (unchanged)
- `type`: `"node"`
- `namespace`: `"fleet"`
- `tenancyId`: `"default"`
- `ttl`: `Duration.ofMinutes(5)` (peer health check interval is the natural TTL)
- `metadata`: `url`, `name`, `source`, `terminalMode`
- `health`: mapped from PeerHealth — UP→HEALTHY, DOWN→DOWN, UNKNOWN→DEGRADED

**Two separate concerns:**
- `PeerHealth` (UP/DOWN/UNKNOWN) — reachability, maps to `HealthStatus` on RegistryEntry
- `CircuitState` (CLOSED/OPEN/HALF_OPEN) — circuit breaker FSM, stays on PeerEntry

These are not the same thing. A peer can be DOWN with circuit OPEN (unreachable,
stopped trying). CircuitState controls whether health checks are attempted;
PeerHealth records the result.

**What moves to RegistryService:**
- Storage: register/deregister/resolve/discover replaces ConcurrentHashMap put/remove/get/values
- Peer health: PeerHealth maps to HealthStatus on the RegistryEntry
- Discovery: `registry.discover(new RegistryQuery("default", "node", "fleet"))` replaces `getAllPeers()`

**What stays in PeerRegistry/PeerEntry:**
- Circuit breaker FSM (circuitState, failure counting, backoff timing, shouldAttemptHealthCheck)
- Source priority (CONFIG > MANUAL > MDNS dedup) — domain rule
- File persistence (non-CONFIG peers → `peers.json`) — RegistryService InMemory is volatile
- Cached sessions per peer — transient runtime state, not topology
- `getHealthyPeers()` — queries registry by HealthStatus

**PeerEntry changes:**
PeerEntry loses the `health` field (moves to RegistryEntry.health as HealthStatus).
PeerEntry keeps `circuitState`, `consecutiveFailures`, `circuitOpenedAt`, `currentBackoffMs`
for the circuit breaker FSM. `recordSuccess()` updates circuit state AND calls
`registry.register(entry.withHealth(HEALTHY))`. `recordFailure()` updates circuit state
AND calls `registry.register(entry.withHealth(DOWN))`.

A local `ConcurrentHashMap<String, PeerEntry>` remains for circuit breaker state and
cached sessions — runtime-only state that RegistryService doesn't model. The RegistryEntry
is the source of truth for peer existence and health.

**Consumers unchanged:** All 16 consumers inject `PeerRegistry` — no API change.

### Test changes

`PeerRegistryTest` gains a `RegistryService` dependency (`InMemoryRegistryService`).
Existing assertions on add/remove/find/health still pass — the API is unchanged.
New assertions verify registry entries are created/updated/removed in lockstep.

---

## 2. AgentPoolDefinitionRegistry — dual-store delegation

### Current state

`@ApplicationScoped` with `ConcurrentHashMap<String, AgentPoolDefinition>`.
Simple CRUD: register, get, all, names, updateScaling, updateCapacity, updateBudget.

### Design

Inject `RegistryService`. On every mutation, update both the ConcurrentHashMap
(for type-safe domain data) and the registry (for discovery/relationships).

**Registry entry mapping:**
- `id`: agent name (the pool's unique key)
- `type`: `"pool"`
- `namespace`: `"fleet"`
- `tenancyId`: `"default"`
- `ttl`: `Duration.ofHours(24)` (effectively infinite — pools don't expire)
- `metadata`: `agentName`, `minActive`, `maxActive` (summary only — full data in map)
- `health`: always HEALTHY (pools are config, not runtime)

**Mutation flow:**
```
register(def):
  1. definitions.putIfAbsent(name, def)  // existing check
  2. registry.register(toEntry(def))     // registry for discovery

updateScaling(name, scaling):
  1. definitions.compute(name, ...)      // update rich data
  2. registry.register(toEntry(updated)) // sync metadata

// Same pattern for updateCapacity, updateBudget
```

**Query flow:**
- `get(name)`, `all()`, `names()` → read from ConcurrentHashMap (full data)
- Pool discovery by external systems → `registry.discover(query)` (topology view)

### Test changes

`AgentPoolDefinitionRegistryTest` and `AgentPoolDefinitionRegistryUpdateTest` gain
`InMemoryRegistryService`. Existing assertions unchanged. New assertions verify
registry entries track pool lifecycle.

---

## 3. PoolMeshRegistrar — RegistryService relationships

### Current state

`@ApplicationScoped` implementing `SessionLifecycleListener`. Injects `InstanceService`.
On pool session lifecycle events:
- `onAcquired`: register as Qhorus instance
- `onSuspended`: mark offline
- `onResumed`: re-register
- `onDestroyed`: deregister

### Design

Replace `InstanceService` injection with `RegistryService`. Use relationships
for pool→session containment and health updates for suspend/resume.

**Pool sessions are registered as agent-instances by RegistryBackedInstanceManager**
(Qhorus, qhorus#489). PoolMeshRegistrar no longer handles instance registration —
it manages the containment topology.

**Lifecycle mapping:**
```
onAcquired(session, poolName):
  registry.link(new Relationship(poolName, session.instanceId(), "contains"))

onSuspended(session, poolName):
  registry.resolve(session.instanceId())
    .ifPresent(entry -> registry.register(entry.withHealth(DEGRADED)))

onResumed(session, poolName):
  registry.resolve(session.instanceId())
    .ifPresent(entry -> registry.register(entry.withHealth(HEALTHY)))

onDestroyed(sessionId, poolName):
  registry.unlink(poolName, sessionId)
```

**Precondition:** RegistryBackedInstanceManager must be active (Qhorus build
property `casehub.qhorus.instance.registry-backed=true`). If disabled, session
entries won't exist in the registry and link/health calls are no-ops. This is
acceptable — without the registry-backed instance manager, there's no unified
topology to update.

### Test changes

`PoolMeshIntegrationTest` replaces `InstanceService` mock with `InMemoryRegistryService`.
Tests verify relationships and health transitions instead of instance service calls.

---

## 4. Dependency changes

### Parent pom.xml (`<dependencyManagement>`)

Add:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-registry-inmem</artifactId>
  <version>${casehub-platform.version}</version>
</dependency>
```

### claudony-app/pom.xml

Add:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-registry-inmem</artifactId>
</dependency>
```

This provides `InMemoryRegistryService` as `@Alternative @Priority(50)`,
automatically displacing `NoOpRegistryService @DefaultBean` from platform-api.

### claudony-casehub/pom.xml

Add `casehub-platform-api` if not already present (for `RegistryService` SPI types
used by AgentPoolDefinitionRegistry). Check first — it may already be transitive.

### CDI exclude-types

Per protocol `PP-20260612-d6e7ec`: if `InMemoryRegistryService` or `HeartbeatScheduler`
cause CDI conflicts in test profiles (CasehubEnabledProfile, CompletionTestProfile),
add them to the exclude-types list in both profiles.

---

## 5. File changes summary

| File | Change |
|------|--------|
| `app/.../fleet/PeerRegistry.java` | Inject RegistryService, delegate storage |
| `app/.../fleet/PeerEntry.java` | Remove health/circuitState fields (move to registry) |
| `app/.../fleet/PoolMeshRegistrar.java` | Replace InstanceService with RegistryService |
| `casehub/.../fleet/AgentPoolDefinitionRegistry.java` | Inject RegistryService, dual-store |
| `app/pom.xml` | Add registry-inmem dependency |
| `pom.xml` | Add registry-inmem to dependencyManagement |
| `app/.../fleet/PeerRegistryTest.java` | Add InMemoryRegistryService |
| `casehub/.../fleet/AgentPoolDefinitionRegistryTest.java` | Add InMemoryRegistryService |
| `casehub/.../fleet/AgentPoolDefinitionRegistryUpdateTest.java` | Add InMemoryRegistryService |
| `app/.../fleet/PoolMeshIntegrationTest.java` | Replace InstanceService with InMemoryRegistryService |

---

## References

- `platform-api/RegistryService.java` — SPI interface
- `platform-api/RegistryEntry.java` — entry record (id, type, namespace, tenancyId, metadata, health, ttl)
- `platform-api/Relationship.java` — relationship record (sourceId, targetId, type)
- `platform-api/RegistryQuery.java` — query record (tenancyId, type, namespace)
- `qhorus/RegistryBackedInstanceManager.java` — reference migration (qhorus#489)
- `claudony#267` — parent epic (modular fleet architecture)
- `platform#569` — RegistryService SPI implementation
- `qhorus#489` — Qhorus InstanceManager migration
- Protocol `PP-20260612-d6e7ec` — CDI exclude-types sync rule
