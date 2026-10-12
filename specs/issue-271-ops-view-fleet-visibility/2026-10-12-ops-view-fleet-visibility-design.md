# Ops View — Fleet-Wide Visibility for UI Shells

**Issue:** casehubio/claudony#271
**Epic:** casehubio/claudony#267 (modular fleet architecture — Stage 3c)
**Depends on:** #270 (registry REST endpoints — done)
**Date:** 2026-10-12

## Problem

Claudony's fleet/pool/mesh state is only visible through 6 separate `@McpDomain` APIs. Each returns raw per-domain data. To get a fleet-wide operational picture — topology, health, utilisation, channel activity — a consumer must call all 6 APIs and correlate the results client-side.

External UI shells (scaffold ops perspective, trellis) need a single, composable ops view they can embed without deep knowledge of Claudony's internal API structure.

## Solution

A new `ClaudonyOpsApi` `@McpDomain` that aggregates fleet-wide operational state into a typed snapshot. Backed by an `OpsService` that composes from existing services. SSE endpoint for live topology updates.

Existing per-domain APIs are untouched — they remain the fine-grained control plane. The ops view is the read-only observation plane.

## Architecture

### Component Overview

```
OpsService (@ApplicationScoped)
├── composes from:
│   ├── RegistryService          → topology entries + relationships
│   ├── PoolService              → pool summaries + utilisation
│   ├── PeerRegistry             → peer health + circuit state
│   ├── ApplicationRegistrationService → app health + pool associations
│   └── QhorusDashboardService   → channel list + instance list
└── produces: OpsSnapshot (typed record tree)

ClaudonyOpsApi (@McpDomain "claudony/ops")
├── GET /api/claudony/ops/snapshot    → OpsSnapshot
├── GET /api/claudony/ops/health      → FleetHealth
└── (SSE handled by OpsEventsResource, see below)

OpsEventBus (@ApplicationScoped)
├── observes: RegistryEvent (CDI async), pool state changes
├── produces: Multi<String> (JSON snapshot per event)
└── pattern: same as PoolEventBus

OpsEventsResource (@HandWrittenEndpoint)
└── GET /api/ops/events (text/event-stream) → Multi<String>
```

### Data Model

```java
// Top-level snapshot — one call, full picture
record OpsSnapshot(
    FleetHealth health,
    List<PoolOpsView> pools,
    List<PeerOpsView> peers,
    List<AppOpsView> apps,
    MeshOpsView mesh,
    List<TopologyEdge> topology,
    Instant timestamp
)

// Aggregate health counts across all entity types
record FleetHealth(
    int totalEntities,
    int healthy,
    int degraded,
    int down,
    int orphaned
)

// Per-pool operational summary
record PoolOpsView(
    String name,
    String scalingType,
    int active,
    int idle,
    int max,
    double fillRatio,       // active / max
    String healthStatus     // from AgentPoolHealth
)

// Per-peer operational summary
record PeerOpsView(
    String id,
    String name,
    String url,
    String health,          // PeerHealth enum name
    String circuitState,    // CircuitState enum name
    int sessionCount,
    boolean stale
)

// Per-app operational summary
record AppOpsView(
    String name,
    String displayName,
    String health,          // HealthStatus enum name
    int poolCount
)

// Mesh aggregate + per-channel summary
record MeshOpsView(
    int totalChannels,
    int totalInstances,
    List<ChannelOpsView> channels
)

record ChannelOpsView(
    String name,
    String purpose,
    long messageCount,
    int subscriberCount     // from ChannelEventBus or QhorusDashboardService
)

// Registry relationship edge
record TopologyEdge(
    String sourceId,
    String targetId,
    String type             // owns, runs-on, serves, consumes
)
```

### OpsService

Single `@ApplicationScoped` service. All methods are blocking (consistent with existing services — `PoolService`, `PeerRegistry`, `ApplicationRegistrationService` are all blocking).

```java
@ApplicationScoped
public class OpsService {

    @Inject RegistryService registryService;
    @Inject PoolService poolService;
    @Inject PeerRegistry peerRegistry;
    @Inject ApplicationRegistrationService appService;
    @Inject QhorusDashboardService dashboard;

    public OpsSnapshot snapshot() { ... }
    public FleetHealth health() { ... }

    String buildSnapshotJson() { ... }  // for SSE
}
```

`snapshot()` calls each backing service, maps results to ops view records, and assembles the `OpsSnapshot`. The `FleetHealth` is computed by iterating registry entries and counting by `HealthStatus`. Pool fill ratios are `active / max` from `AgentPoolStatus`.

`health()` is a lightweight subset — just the aggregate counts, no per-entity detail. Useful for status badges and health indicators without the full snapshot overhead.

### OpsEventBus

Same pattern as `PoolEventBus`: `ConcurrentHashMap<String, List<MultiEmitter<String>>>` keyed by a topic string. For the ops view, a single topic `"ops"` is sufficient.

```java
@ApplicationScoped
public class OpsEventBus {

    // Observes registry changes
    void onRegistryEvent(@ObservesAsync RegistryEvent event) {
        emit();  // rebuild and push full snapshot
    }

    // Subscribe returns Multi<String> with initial snapshot
    Multi<String> subscribe(Supplier<String> snapshotFn) { ... }

    void emit() { ... }  // push snapshot to all subscribers
}
```

On any `RegistryEvent` (register, deregister, health change, link, unlink), the bus rebuilds the snapshot and pushes it to all SSE subscribers. This captures pool, peer, and app state changes because those entities are registered in the registry.

For mesh channel activity (which doesn't fire registry events), the bus can either:
- Piggyback on registry events (mesh data included in the next snapshot regardless of what triggered it)
- Add a periodic refresh (e.g., every 30s) to catch channel activity changes

The piggyback approach is simpler and sufficient — channel counts don't need sub-second precision in an ops view.

### ClaudonyOpsApi

```java
@McpDomain(value = "claudony/ops", app = "claudony",
           basePath = "/api/claudony/ops",
           summary = "Fleet-wide operational visibility — topology, health, utilisation")
@ApplicationScoped
public class ClaudonyOpsApi {

    @Inject OpsService opsService;

    @PlatformQuery("Full operational snapshot")
    @RestPath("/snapshot")
    public OpsSnapshot snapshot() {
        return opsService.snapshot();
    }

    @PlatformQuery("Fleet health summary")
    @RestPath("/health")
    public FleetHealth health() {
        return opsService.health();
    }
}
```

### OpsEventsResource (SSE)

```java
@HandWrittenEndpoint("SSE streaming — cannot be expressed as mcpDomain")
@Path("/api/ops/events")
@Authenticated
public class OpsEventsResource {

    @Inject OpsEventBus opsEventBus;
    @Inject OpsService opsService;

    @GET
    @Produces("text/event-stream")
    public Multi<String> events() {
        return opsEventBus.subscribe(() -> opsService.buildSnapshotJson());
    }
}
```

Full snapshot on connect. Subsequent events push updated snapshots when topology changes. Consumers render the latest snapshot — no delta merging required.

### Module Placement

All new classes go in `claudony-app`:

| Class | Package |
|-------|---------|
| `OpsSnapshot`, `FleetHealth`, `PoolOpsView`, `PeerOpsView`, `AppOpsView`, `MeshOpsView`, `ChannelOpsView`, `TopologyEdge` | `io.casehub.claudony.server.ops` |
| `OpsService` | `io.casehub.claudony.server.ops` |
| `OpsEventBus` | `io.casehub.claudony.server.ops` |
| `ClaudonyOpsApi` | `io.casehub.claudony.server.api` |
| `OpsEventsResource` | `io.casehub.claudony.server.ops` |

All in `claudony-app` because they compose from app-level services (`PoolService`, `PeerRegistry`) that are already in the app module. No new dependencies.

## Testing

### Unit Tests

- `OpsServiceTest` — mock all 5 backing services, verify snapshot assembly, health counts, fill ratio calculation, edge mapping
- `OpsEventBusTest` — subscribe/emit lifecycle, subscriber count, cleanup on cancel (follow `PoolEventBusTest` pattern)

### Integration Tests

- `ClaudonyOpsApiTest` (`@QuarkusTest`) — REST endpoint tests for `/snapshot` and `/health`
- `OpsEventsResourceTest` (`@QuarkusTest`) — SSE connection, initial snapshot delivery, event push on registry change

### What Not to Test

- Individual backing services — already tested in their own test classes
- Data correctness of backing services — ops view trusts them; test the mapping, not the source data

## Scope Boundaries

### In scope
- `OpsService` + typed snapshot records
- `ClaudonyOpsApi` `@McpDomain` (REST)
- `OpsEventBus` + `OpsEventsResource` (SSE)
- Unit + integration tests

### Out of scope
- UI components (blocks-ui panels) — that's #273
- Migrating Claudony's existing dashboard panels to consume the ops API — follow-up
- Per-pool SSE migration (existing `PoolEventsResource` stays as-is)
- Authentication changes — ops endpoints use existing `@Authenticated`
- Historical metrics or time-series data — ops view is current state only

## References

- `ClaudonyRegistryApi.java` — existing registry REST surface
- `ClaudonyPoolApi.java` — existing pool REST surface
- `ClaudonyAppApi.java` — existing app REST surface
- `ClaudonyMeshApi.java` — existing mesh REST surface
- `ClaudonyPeerApi.java` — existing peer REST surface
- `PoolService.java` — pool aggregation logic (pattern reference)
- `PoolEventBus.java` — SSE event bus pattern
- `PoolEventsResource.java` — SSE endpoint pattern
- `RegistryService.java` — unified registry SPI
- `RegistryEntry.java` — registry entry model
- `RegistryEvent.java` — CDI event model for registry changes
- `QhorusDashboardService.java` — mesh channel/instance queries
- Issue #267 — epic: modular fleet architecture
- Issue #270 — registry REST endpoints (dependency, done)
