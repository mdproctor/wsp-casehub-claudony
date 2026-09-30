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
- Role infrastructure: extend `CredentialStore` with a `roles` field in `~/.claudony/credentials.json`. Build a `SecurityIdentityAugmentor` that reads roles from the credential store and augments the `SecurityIdentity`. The first registered credential gets the `admin` role automatically. This is the simplest path for pre-release (single operator). `casehub-platform-oidc` adoption is a future concern.

### Scope

- Per-node only. No fleet fan-out in #208.
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
2. Read and reset counter deltas
3. Insert into IoTDB Table Model:

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

**Translation rules:**
- `FilterOp.EQUALS_TO` → `WHERE tag = 'value'`
- `FilterOp.TIME_FRAME` → `WHERE time >= start AND time < end`
- `GroupOp` with aggregation → `SELECT AVG(fill_ratio) ... GROUP BY ([interval])`
- `SortOp` → `ORDER BY time ASC/DESC`
- Returns `DataSetResult` (columns + rows) matching the pages tabular format

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

## Workstream 5: EventBroadcaster Integration

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
| `pages EventBroadcaster` | claudony-app | Already on classpath (casehub-qhorus-push) |
| `pages-viz` (`PagesTimeseries`, `PagesMetric`) | claudony-app webui | Already available (casehub-pages-npm) |

## References

- `AgentSessionManager.java` — pool lifecycle operations
- `AgentPoolManagerRegistry.java` — multi-pool registry
- `ScalingScheduler.java` — periodic scaling evaluation
- `ScalingPolicy.java` — scaling SPI (TargetTracking, Step, Custom)
- `EvictionPolicy.java` — eviction SPI (DefaultEvictionPolicy)
- `PoolSnapshot.java` / `DemandMetrics` — metrics data model
- `AgentPoolResource.java` — existing placeholder endpoint
- `EventBroadcaster.java` (pages) — push infrastructure
- `EventStore.java` (pages) — event persistence SPI
- `DataProvider.java` (pages) — data query SPI
- `PrometheusDataProvider.java` (pages) — existing DataProvider pattern to follow
- `app.ts` — tab registration pattern
- `claudony-fleet-panel.ts` — sidebar panel pattern
- `claudony-action-inbox.ts` — table rendering pattern
- IoTDB Table Model: https://iotdb.apache.org
- casehub-iot — future IoTDB consumer
- scaffold#52 — ops perspective (Claudony REST API consumable by scaffold later)
