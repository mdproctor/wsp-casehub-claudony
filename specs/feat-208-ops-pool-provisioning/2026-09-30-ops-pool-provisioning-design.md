# Ops Pool Provisioning — Design Spec

**Issue:** casehubio/claudony#208
**Absorbs:** #241 (REST API for runtime scaling config), #242 (dashboard scaling status display)
**Cross-repo:** Claudony + casehub-pages
**Date:** 2026-09-30

## Overview

Add a "Pools" tab to Claudony's dashboard with full pool management — status, sessions, scaling, historical metrics — backed by a comprehensive REST API, Micrometer instrumentation, IoTDB time-series storage, and pages EventBroadcaster for real-time push.

This issue also proves out Apache IoTDB as the platform TSDB, with pool metrics as the first consumer. The flat label adapter and pages IoTDB DataProvider built here will be reused by casehub-iot and other components.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Dashboard (Pools tab)                                       │
│  ┌──────────┐ ┌────────────────────────────────────────────┐ │
│  │ Pool List │ │ Pool Detail                                │ │
│  │           │ │ ┌─────────────────────────────────────────┐│ │
│  │ ● default │ │ │ Status: HEALTHY  Active: 3/10  Idle: 2 ││ │
│  │ ○ review  │ │ ├─────────────────────────────────────────┤│ │
│  │           │ │ │ Sessions table (actions: suspend/resume)││ │
│  │           │ │ ├─────────────────────────────────────────┤│ │
│  │           │ │ │ Scaling: target-tracking @ 0.7          ││ │
│  │           │ │ │ Last decision: scale-out +2 (2m ago)    ││ │
│  │           │ │ ├─────────────────────────────────────────┤│ │
│  │           │ │ │ Charts: fill ratio, demand metrics      ││ │
│  │           │ │ │ (PagesTimeseries ← IoTDB DataProvider)  ││ │
│  │           │ │ ├─────────────────────────────────────────┤│ │
│  │           │ │ │ Scaling decision log / eviction history  ││ │
│  └──────────┘ │ └─────────────────────────────────────────┘│ │
│               └────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
    REST polling        WebSocket push       DataProvider query
    (status, sessions)  (EventBroadcaster)   (IoTDB time-series)
         │                    │                    │
┌────────┴────────────────────┴────────────────────┴──────────┐
│  Claudony Backend                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ PoolResource  │  │ ScalingScheduler│ │ Flat Label Adapter│ │
│  │ /api/pools/*  │  │ (15s tick)   │  │ Micrometer→IoTDB  │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────┘ │
│         │                 │                    │             │
│  ┌──────┴─────────────────┴────────────────────┴───────────┐│
│  │ AgentPoolManagerRegistry → AgentSessionManager(s)       ││
│  │ MeterRegistry (Micrometer gauges/counters/timers)       ││
│  │ EventBroadcaster (pool:* topics → EventStore)           ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
         │                                        │
    /q/metrics                              IoTDB (Docker)
    (Prometheus scrape)                     (Table Model)
```

## Workstream 1: REST API

### New resource: `PoolResource`

Replaces the existing `AgentPoolResource` at `/api/agent-pools`.

```
GET    /api/pools                              → PoolSummary[]
GET    /api/pools/{name}                       → PoolDetail
GET    /api/pools/{name}/sessions              → ManagedSessionInfo[]
POST   /api/pools/{name}/sessions/{id}/suspend → void (204)
POST   /api/pools/{name}/sessions/{id}/resume  → void (204)
DELETE /api/pools/{name}/sessions/{id}         → void (204)
PATCH  /api/pools/{name}/scaling               → ScalingConfigUpdate (200)
PATCH  /api/pools/{name}/capacity              → CapacityUpdate (200)
```

### Response shapes

**PoolSummary** (list endpoint):
```json
{
  "name": "default",
  "status": { "min": 0, "max": 10, "active": 3, "idle": 2, "total": 5, "health": "HEALTHY" },
  "scalingType": "target-tracking"
}
```

`scalingType` is derived from the `ScalingConfig` sealed interface variant via pattern match:
- `TargetTrackingConfig` → `"target-tracking"`
- `StepConfig` → `"step"`
- `CustomScalingConfig` → `"custom"`
- `NoScalingConfig` → `"none"`

Add a `default String type()` method to `ScalingConfig` that returns this mapping.

**PoolDetail** (detail endpoint):
```json
{
  "name": "default",
  "status": { "min": 0, "max": 10, "active": 3, "idle": 2, "total": 5, "health": "HEALTHY" },
  "definition": {
    "agent": { "name": "default", "workingDir": "/workspace", "policy": "EXCLUSIVE", "command": "claude" },
    "pool": { "minActive": 0, "maxActive": 10, "eviction": "MEMORY_WEIGHTED" }
  },
  "scaling": {
    "type": "target-tracking",
    "config": { "targetFillRatio": 0.7, "cooldown": "60s", "scaleInCooldown": "300s" },
    "lastDecision": { "direction": "OUT", "count": 2, "reason": "fillRatio 0.8 > target 0.7", "timestamp": "..." },
    "cooldownRemaining": "45s"
  },
  "demand": { "evictions": 3, "exhaustions": 0, "acquires": 12 }
}
```

**Data sources for PoolDetail assembly:**

`PoolResource` injects both `AgentPoolManagerRegistry` (runtime state) and `AgentPoolDefinitionRegistry` (static configuration):

| Section | Source |
|---|---|
| `status` | `AgentPoolManagerRegistry.get(name)` → `AgentSessionManager.status()` |
| `definition` | `AgentPoolDefinitionRegistry.get(name)` → `AgentPoolDefinition.agent()` / `.pool()` |
| `scaling` | `ScalingScheduler.scalingState(name)` → `ScalingState` (see below) |
| `demand` | Micrometer `MeterRegistry` — cumulative totals from counters (see §Workstream 3) |

**Scaling state:** `demand` values are **cumulative totals** from Micrometer counters (`claudony.pool.acquires.total`, etc.), not recent deltas. The scheduler's internal `snapshotAndResetDemandMetrics()` mechanism is independent — it reads and resets private counters for delta-based scaling decisions. The REST endpoint never calls `snapshotAndResetDemandMetrics()`.

### ScalingState — new internal model

`ScalingScheduler` currently discards `ScalingDecision` after applying it and stores cooldown timestamps in separate private maps. To populate the `PoolDetail.scaling` section, introduce a `ScalingState` record:

```java
public record ScalingState(
    ScalingDecision lastDecision,
    Instant lastDecisionTime,
    Instant lastScaleOut,
    Instant lastScaleIn,
    ScalingConfig config
) {
    public Duration cooldownRemaining(Instant now) {
        // compute from lastScaleOut/lastScaleIn and config.cooldown()/scaleInCooldown()
    }
}
```

`ScalingScheduler` changes:
- Replace `Map<String, Instant> lastScaleOut` and `lastScaleIn` with `Map<String, ScalingState> scalingStates`
- After `evaluatePool()` computes a decision, store it in the `ScalingState`
- Add public `Optional<ScalingState> scalingState(String poolName)` method
- `PoolResource` calls `scheduler.scalingState(poolName)` to populate the scaling section

### PATCH request bodies and mutation paths

**`ScalingConfigUpdate`** (for `PATCH /api/pools/{name}/scaling`):
```json
{
  "type": "target-tracking",
  "targetFillRatio": 0.8,
  "cooldown": "120s",
  "scaleInCooldown": "600s"
}
```

Mutable fields: all fields of the active scaling config variant. Switching `type` replaces the entire `ScalingConfig` with a new variant. Validation: same rules as `ScalingConfig` record constructors (e.g., `targetFillRatio ∈ (0.0, 1.0]`, non-empty steps for `StepConfig`).

**Propagation path:**
1. `PoolResource` validates and constructs a new `ScalingConfig` instance
2. `AgentPoolDefinitionRegistry` needs a new `updateScaling(String poolName, ScalingConfig config)` method that replaces the `PoolConfig.scaling` field in the definition
3. `ScalingScheduler.policyCache` must be invalidated for the pool: add `invalidatePolicy(String poolName)` method
4. Next `tick()` call reconstructs the `ScalingPolicy` from the updated config

**`CapacityUpdate`** (for `PATCH /api/pools/{name}/capacity`):
```json
{
  "minActive": 2,
  "maxActive": 15
}
```

Mutable fields: `minActive`, `maxActive`. Validation: `minActive >= 0`, `maxActive >= 1`, `maxActive >= minActive`, `maxActive >= current active count`.

**Propagation path:**
1. `PoolResource` validates constraints
2. For `maxActive`: `AgentSessionManager.adjustMaxActive(newMax)` — already exists
3. For `minActive`: add `adjustMinActive(int newMin)` to `AgentSessionManager` — clamps to `[0, effectiveMaxActive]` and updates the config-level minimum used by the eviction guard
4. `AgentPoolDefinitionRegistry` updates the definition to reflect the new values

**ManagedSessionInfo** (sessions endpoint):
```json
{
  "instanceId": "claudony-pool-abc123",
  "identity": "code-reviewer",
  "workingDir": "/workspace/reviews",
  "conversationId": "conv-xyz",
  "state": "ACTIVE",
  "lastInteraction": "2026-09-30T10:15:00Z",
  "memoryBytes": 104857600,
  "idleSeconds": 45
}
```

### Authorization

- Read endpoints: `@Authenticated` (consistent with existing pattern)
- Mutation endpoints: `@RolesAllowed("admin")`
- Role infrastructure:

**`StoredCredential` schema change:** Add a `roles` field (`List<String>`, nullable) to the `StoredCredential` record in `CredentialStore`. Jackson record deserialization handles the migration transparently — existing `credentials.json` files without a `roles` field will deserialize with `roles = null`, which the compact constructor defaults to an empty list.

```java
record StoredCredential(
    String username, String credentialId, String aaguid,
    String publicKey, long publicKeyAlgorithm, long counter,
    List<String> roles  // nullable — old files omit this; default to empty
) {
    public StoredCredential {
        if (roles == null) roles = List.of();
    }
}
```

**Auto-admin assignment:** On `store()`, if `isEmpty()` returns true before adding the credential, the first credential receives `roles = List.of("admin")`. "First" means the store is empty at registration time — not first-ever across reinstalls.

**`SecurityIdentityAugmentor`:** A Quarkus `SecurityIdentityAugmentor` CDI bean that:
1. Looks up the authenticated credential ID from `SecurityIdentity`
2. Loads roles from `CredentialStore` for that credential
3. Augments the identity with `QuarkusSecurityIdentity.builder().addRole(role)` for each role

**Caching:** The augmentor reads `credentials.json` on every request (via the existing synchronized `load()` method). For a single-operator pre-release system this is acceptable — the file is small and reads are fast. If performance becomes a concern, add a file-watch cache invalidation.

**Role assignment for invited users:** The invite flow creates new credentials via `store()`. For #208, invited users get no roles (read-only). Role management UI is out of scope — admin assigns roles by editing `credentials.json` or via a future CLI command.

### Scope

- Per-node only. No fleet fan-out in #208.
- **Single pool only.** `ClaudonyAgentBackend` currently registers a single `"default"` pool via `managerRegistry.register("default", sessionManager)`. Multi-pool definition (YAML config, annotation-driven, or REST API) is out of scope. The REST API is designed for multi-pool (resource-per-pool URLs, list endpoint) to avoid rework — `ClaudonyAgentBackend.fromDefinition()` is the intended future path for additional pools.
- Deprecate existing `AgentPoolResource` at `/api/agent-pools`.

## Workstream 2: Dashboard

### Tab registration

In `app.ts`:
```typescript
import './components/claudony-pool-panel.js';
registerPanel("pool-panel", "claudony-pool-panel");
// Add to tabs:
tabs(
  ["Sessions", hostPanel("session-panel")],
  ["Cases",    hostPanel("case-browser")],
  ["Inbox",    hostPanel("action-inbox")],
  ["Pools",    hostPanel("pool-panel")],
  ["Fleet",    hostPanel("fleet-panel")],
  ["Mesh",     hostPanel("mesh-panel")],
)
```

### Component: `claudony-pool-panel.ts`

LitElement component following existing patterns (`@customElement`, `@state()`, `fetchWithAuth()`).

**Left sidebar (pool list):**
- Polls `GET /api/pools` every 10s
- Each pool: `pages-status-dot` for health, name, `active/max` count
- Click selects → loads detail

**Detail area (selected pool):**

Five sections, each a sub-component or inline render:

1. **Status header** — health badge, capacity bar (active/max visual), min/max/active/idle numbers. `PagesMetric` KPI cards for fill ratio and demand counts.

2. **Session table** — HTML table (consistent with action-inbox pattern). Columns: identity, workingDir, state, idle time, memory (MB), actions. Action buttons: Suspend (active sessions), Resume (suspended), Destroy (any). Mutations call the REST API, then refresh.

3. **Scaling state** — current policy type and config, last scaling decision with timestamp and reason, cooldown timer (countdown). Updated via EventBroadcaster push on `pool:{name}:scaling` topic.

4. **Historical charts** — `PagesTimeseries` components querying the pages IoTDB DataProvider for fill ratio over time and demand metrics (acquires, evictions, exhaustions) over time. Time range selector (1h, 6h, 24h).

5. **Event log** — scaling decision log and eviction history. Rendered from EventBroadcaster replay (`pool:{name}:scaling` and `pool:{name}:session` topics). Newest first, paginated.

### Data flow

| Data | Source | Refresh |
|---|---|---|
| Pool list + status | `GET /api/pools` | 10s polling |
| Pool detail | `GET /api/pools/{name}` | 10s polling |
| Session list | `GET /api/pools/{name}/sessions` | 10s polling |
| Scaling decisions | EventBroadcaster `pool:{name}:scaling` | WebSocket push |
| Session lifecycle | EventBroadcaster `pool:{name}:session` | WebSocket push |
| Health changes | EventBroadcaster `pool:{name}:health` | WebSocket push |
| Historical charts | Pages IoTDB DataProvider | On render + time range change |
| Event log | EventStore replay | On render + push updates |

## Workstream 3: Micrometer Instrumentation

### Dependency

Add to `app/pom.xml`:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
</dependency>
```

### Metrics registration

In `ScalingScheduler` (inject `MeterRegistry`):

**Gauges** (tagged by `pool` name):
- `claudony.pool.active` — current active session count
- `claudony.pool.idle` — current suspended session count
- `claudony.pool.max` — current effective maxActive
- `claudony.pool.fill_ratio` — activeCount / maxActive

**Counters** (tagged by `pool` name):
- `claudony.pool.acquires.total` — total acquire calls
- `claudony.pool.evictions.total` — total evictions
- `claudony.pool.exhaustions.total` — total exhaustion events (pool full, minActive prevents eviction)
- `claudony.pool.scaling.decisions.total` — tagged by `direction` (OUT/IN/NONE)

**Timers** (tagged by `pool` name):
- `claudony.pool.acquire.duration` — time to acquire a session (includes eviction time)

### Endpoint

Standard `/q/metrics` Prometheus scrape endpoint. Auth: the Micrometer extension exposes this under Quarkus management interface — configure access as needed.

## Workstream 4: IoTDB Integration

### IoTDB setup

**Dev:** Docker Compose alongside existing PostgreSQL:
```yaml
services:
  iotdb:
    image: apache/iotdb:latest
    ports:
      - "6667:6667"
```

**Connection config:**
```properties
claudony.iotdb.host=localhost
claudony.iotdb.port=6667
claudony.iotdb.user=root
claudony.iotdb.password=root
claudony.iotdb.enabled=false  # opt-in; dashboard works without (charts show "no data")
```

### Flat label adapter

A `@ApplicationScoped` bean that bridges Micrometer → IoTDB. Runs on a scheduled tick (aligned with ScalingScheduler's 15s interval or independently configurable).

**Responsibilities:**
1. Read current gauge values from `MeterRegistry` for all pool-tagged metrics
2. Read cumulative counter values from `MeterRegistry` (Micrometer counters are monotonically increasing — there is no reset operation)
3. Insert into IoTDB Table Model (cumulative values — rate/delta computation happens at query time via IoTDB's `DIFFERENCE()` function or client-side calculation):

```sql
CREATE TABLE IF NOT EXISTS pool_metrics (
  pool TAG,
  active INT32,
  idle INT32,
  max_active INT32,
  fill_ratio DOUBLE,
  acquires INT32,
  evictions INT32,
  exhaustions INT32,
  scaling_direction TEXT
)
```

4. Use `SessionPool` for thread-safe IoTDB access

**Graceful degradation:** If IoTDB is not configured (`claudony.iotdb.enabled=false`) or unavailable, the adapter is a no-op. Dashboard charts show "No metrics data available." Everything else (REST API, push events, session management) works without IoTDB.

### Pages IoTDB DataProvider

New module in casehub-pages: `backend/data-iotdb/`

Implements `DataProvider` SPI:
```java
@ApplicationScoped
public class IoTDBDataProvider implements DataProvider {
    String type() { return "iotdb"; }
    boolean canHandle(String dataSetId) { /* match iotdb-prefixed datasets */ }
    QueryResult query(DataSetLookup lookup) { /* translate to IoTDB SQL */ }
}
```

**Translation rules (IoTDB Table Model SQL dialect):**

IoTDB's Table Model SQL differs from standard SQL in key areas:
- **Implicit `time` column:** all tables have an implicit `time` column (TIMESTAMP type). Do not include `time` in CREATE TABLE — but use `time` in WHERE and GROUP BY.
- **TAG vs FIELD columns:** TAG columns are indexed metadata (e.g., `pool`); FIELD columns are measurement values. WHERE clauses on TAG and FIELD columns have the same syntax but different performance characteristics.
- **Time-based grouping:** IoTDB uses `GROUP BY ([startTime, endTime), interval)` syntax, not standard `GROUP BY column`.
- **Data types:** `INT32`, `INT64`, `DOUBLE`, `TEXT`, `BOOLEAN` — map IoTDB types to `DataSetResult` column types accordingly.

Translation from pages `DataSetLookup` operations:
- `FilterOp.EQUALS_TO` → `WHERE tag = 'value'`
- `FilterOp.TIME_FRAME` → `WHERE time >= start AND time < end`
- `GroupOp` with aggregation → `SELECT AVG(fill_ratio) FROM pool_metrics WHERE time >= start AND time < end GROUP BY ([start, end), interval)`
- `SortOp` → `ORDER BY time ASC/DESC`
- Counter-based metrics (acquires, evictions, exhaustions) are stored as cumulative values. For rate display, apply `DIFFERENCE()` in the IoTDB query or compute deltas client-side.
- Returns `DataSetResult` (columns + rows) matching the pages tabular format. Follow the `PrometheusDataProvider` pattern in `pages/backend/data-prometheus/`.

**Configuration:**
```properties
casehub.pages.data.iotdb.endpoint=localhost:6667
casehub.pages.data.iotdb.datasets.pool-fill-ratio.table=pool_metrics
casehub.pages.data.iotdb.datasets.pool-fill-ratio.columns=fill_ratio
casehub.pages.data.iotdb.datasets.pool-demand.table=pool_metrics
casehub.pages.data.iotdb.datasets.pool-demand.columns=acquires,evictions,exhaustions
```

### GraalVM compatibility

IoTDB Java Session API uses reflection for Thrift transport. For native image builds, reflection config may be needed. Validate during implementation — JVM mode works regardless.

### IoTDB evaluation criteria

This is the first platform use of IoTDB. Success criteria for the proof-of-concept:

1. **Functional:** flat label adapter writes metrics, IoTDB DataProvider reads them, dashboard charts render correctly
2. **Performance:** sub-100ms query latency for 24h time-range aggregations at 15s granularity
3. **Reliability:** IoTDB Docker container survives restart without data loss (WAL recovery)
4. **GraalVM:** if native image build fails due to Thrift reflection, document the blocker — JVM mode is acceptable for now
5. **Developer ergonomics:** `claudony.iotdb.enabled=false` (default) means zero IoTDB setup for contributors who don't need charts

### Fallback path

If IoTDB proves unsuitable (poor query performance, GraalVM incompatibility, operational complexity), the fallback is:
1. Keep the flat label adapter interface but swap the IoTDB backend for Prometheus remote write
2. Reuse the existing `PrometheusDataProvider` in `pages/backend/data-prometheus/` — zero new DataProvider code
3. Trade-off: requires a Prometheus/VictoriaMetrics binary (external Go process), which the D9 decision rejected for Java-native preference

The `claudony.iotdb.enabled=false` default ensures the dashboard works without IoTDB (charts show "No metrics data available"). This is already the escape hatch — disabling IoTDB does not break any other functionality.

## Workstream 5: EventBroadcaster Integration

### CDI wiring

`EventBroadcaster` (from `casehub-pages-push`) requires four dependencies: `EventStore`, `TopicRegistry`, `SessionSender`, and `JsonWriter`. The `casehub-pages-push-runtime` module provides CDI producers for three of these (`EventStore` as `InMemoryEventStore`, `TopicRegistry`, `JsonWriter` as Jackson `ObjectMapper::writeValueAsString`). The fourth — `SessionSender` — is application-specific and must be provided by claudony.

**Integration steps:**
1. Add `casehub-pages-push-runtime` dependency to `app/pom.xml`
2. Implement and `@Produces` a `SessionSender` bean that bridges to claudony's WebSocket infrastructure (Quarkus `io.quarkus.websockets.next` API — send to a connection by ID)
3. `EventBroadcaster` is then available for CDI injection (produced by `PushProducers`)
4. Frontend subscribes to pool topics via the pages push WebSocket endpoint

The Qhorus project already integrates with `EventBroadcaster` via `A2AEventBroadcasterBridgeProducer` (reflection-based, for optional coupling). Claudony can use direct CDI injection since `casehub-pages-push-runtime` will be an explicit dependency.

### Event emission

In `ScalingScheduler.tick()` and `AgentSessionManager` lifecycle methods, broadcast events:

**Topics and payloads:**

`pool:{name}:scaling`:
```json
{
  "direction": "OUT",
  "count": 2,
  "reason": "fillRatio 0.8 > target 0.7",
  "previousMax": 8,
  "newMax": 10,
  "timestamp": "2026-09-30T10:15:00Z"
}
```

`pool:{name}:session`:
```json
{
  "event": "suspended",
  "instanceId": "claudony-pool-abc123",
  "identity": "code-reviewer",
  "reason": "eviction (score: 183.0)",
  "timestamp": "2026-09-30T10:15:00Z"
}
```

`pool:{name}:health`:
```json
{
  "previous": "HEALTHY",
  "current": "DEGRADED",
  "reason": "fillRatio > 0.9",
  "timestamp": "2026-09-30T10:15:00Z"
}
```

### EventStore persistence

Events are automatically persisted by EventBroadcaster → EventStore. The dashboard uses `EventStore.replay(topic, sinceSeq, limit)` for the event log panel and for reconnection replay.

## Testing Strategy

### Backend (Java)

- `PoolResourceTest` — QuarkusTest for all REST endpoints (read + mutations), auth checks
- `PoolResourceAuthTest` — `@RolesAllowed` enforcement (admin vs non-admin)
- `IoTDBFlatLabelAdapterTest` — unit tests with mock MeterRegistry and mock IoTDB SessionPool
- `IoTDBDataProviderTest` — unit tests: DataSetLookup translation to IoTDB SQL, response mapping
- Existing pool tests (`AgentSessionManagerTest`, `ScalingSchedulerTest`, etc.) unchanged

### Frontend (vitest)

- `claudony-pool-panel.test.ts` — component rendering, pool selection, data refresh
- Mock REST responses and EventBroadcaster messages

### E2E (Playwright)

- `PoolPanelE2ETest` — Pools tab visible, pool list renders, detail view loads, session actions work
- Requires IoTDB Docker for chart tests (skip gracefully if unavailable)

## Dependencies

| Dependency | Where | New? |
|---|---|---|
| `quarkus-micrometer-registry-prometheus` | claudony-app | Yes |
| `iotdb-session` (Apache IoTDB Java client) | claudony-app | Yes |
| `iotdb-session` | pages data-iotdb | Yes (new pages module) |
| `casehub-pages-push-runtime` | claudony-app | Yes — CDI producers for EventBroadcaster, EventStore, TopicRegistry, JsonWriter |
| `pages EventBroadcaster` | claudony-app | API on classpath (casehub-qhorus-push → casehub-pages-push); runtime producers require casehub-pages-push-runtime + SessionSender implementation (see §Workstream 5) |
| `pages-viz` (`PagesTimeseries`, `PagesMetric`) | claudony-app webui | Already available (casehub-pages-npm) |

## References

- `AgentSessionManager.java` — pool lifecycle operations (acquire, suspend, resume, destroy, adjustMaxActive)
- `AgentPoolManagerRegistry.java` — multi-pool runtime registry (maps pool names to `AgentSessionManager` instances)
- `AgentPoolDefinitionRegistry.java` — multi-pool static config registry (maps pool names to `AgentPoolDefinition` — agent config, pool config, scaling config)
- `AgentPoolDefinition.java` — declarative pool definition: `AgentConfig` (name, workingDir, policy, command) + `PoolConfig` (minActive, maxActive, eviction, scaling)
- `ScalingScheduler.java` — periodic scaling evaluation (15s tick)
- `ScalingPolicy.java` — scaling SPI (TargetTracking, Step, Custom)
- `ScalingDecision.java` — scaling decision record (direction, count, reason)
- `EvictionPolicy.java` — eviction scoring SPI interface
- `DefaultEvictionPolicy.java` — default `@ApplicationScoped @DefaultBean` implementation (MEMORY_WEIGHTED: idle-time × memory-pressure)
- `EvictionStrategy.java` — eviction strategy enum (`MEMORY_WEIGHTED`, `LRU`) — stored in `PoolConfig`, distinct from `EvictionPolicy` SPI
- `PoolSnapshot.java` / `DemandMetrics` — metrics data model
- `AgentPoolResource.java` — existing placeholder endpoint (to be replaced by `PoolResource`)
- `CredentialStore.java` — WebAuthn credential persistence (`~/.claudony/credentials.json`)
- `EventBroadcaster.java` (pages) — push infrastructure
- `PushProducers.java` (pages push-runtime) — CDI producers for EventBroadcaster, EventStore, TopicRegistry, JsonWriter
- `A2AEventBroadcasterBridgeProducer.java` (qhorus) — reference pattern for EventBroadcaster integration
- `EventStore.java` (pages) — event persistence SPI
- `DataProvider.java` (pages) — data query SPI
- `PrometheusDataProvider.java` (pages data-prometheus) — existing DataProvider pattern to follow
- `app.ts` — tab registration pattern
- `claudony-fleet-panel.ts` — sidebar panel pattern
- `claudony-action-inbox.ts` — table rendering pattern
- IoTDB Table Model: https://iotdb.apache.org
- casehub-iot — future IoTDB consumer
- scaffold#52 — ops perspective (Claudony REST API consumable by scaffold later)
