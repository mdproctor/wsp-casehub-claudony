# Scaling API & Dashboard — Design Spec

**Issues:** #241 (runtime scaling config API), #242 (dashboard scaling display)
**Branch:** `feat/241-242-scaling-api-dashboard`
**Date:** 2026-10-01

---

## Problem

Pool scaling operations exist but are incomplete and misaligned with platform conventions:

1. **API surface gap:** `PoolResource` uses `@HandWrittenEndpoint` (raw JAX-RS) instead of `@McpDomain`. Controller Claude agents — the primary consumer of pool management — can't discover or invoke scaling operations via MCP tools. The REST-only surface misses both MCP and GraphQL.

2. **Incomplete scaling type support:** `parseScalingConfig()` handles only `target-tracking` and `none`, despite the sealed `ScalingConfig` interface defining 5 variants (`target-tracking`, `step`, `demand-pressure`, `custom`, `none`). Three types are configurable via YAML at startup but not adjustable at runtime.

3. **Dashboard is read-only:** The pool panel displays scaling state as static text. No controls to adjust capacity or change scaling policy. The event log and metrics charts are placeholders.

4. **No real-time updates:** The pool panel polls every 10 seconds. `PoolEventEmitter` already broadcasts scaling/session/health events via `EventBroadcaster`, but no SSE endpoint exists for the frontend to consume them.

---

## Design

### 1. mcpDomain Pool API

#### 1.1 New class: `ClaudonyPoolApi`

**File:** `app/src/main/java/io/casehub/claudony/server/api/ClaudonyPoolApi.java`

```java
@McpDomain(value = "claudony/pools", app = "claudony",
           basePath = "/api/claudony/pools",
           summary = "Agent pool management — capacity, scaling, sessions")
@ApplicationScoped
public class ClaudonyPoolApi {
    @Inject PoolService poolService;
    // operations listed below
}
```

Follows the established pattern: `ClaudonySessionApi`, `ClaudonyPeerApi`, `ClaudonyCaseApi`, `ClaudonyMeshApi`.

#### 1.2 Operations

| Annotation | Path | Description |
|---|---|---|
| `@PlatformQuery` | `/` | List all pools → `List<PoolSummary>` |
| `@PlatformQuery` | `/{name}` | Pool detail → `PoolDetailResponse` |
| `@PlatformQuery` | `/{name}/sessions` | Pool sessions → `List<ManagedSessionInfo>` |
| `@PlatformMutation` | `/{name}/update` | Update pool config → `PoolDetailResponse` |
| `@PlatformMutation` | `/{name}/sessions/{id}/suspend` | Suspend session |
| `@PlatformMutation` | `/{name}/sessions/{id}/resume` | Resume session |
| `@PlatformMutation` | `/{name}/sessions/{id}/destroy` | Destroy session |

All operations delegate to `PoolService`.

#### 1.3 PoolUpdateRequest — coarse mutation

A single `updatePool` mutation replaces both `updateCapacity` and `updateScaling`. All fields are optional — only supplied fields are applied.

```java
public record PoolUpdateRequest(
    Integer minActive,
    Integer maxActive,
    String scalingType,                // "target-tracking" | "step" | "demand-pressure" | "none" | <custom-bean-name>
    Double targetFillRatio,            // target-tracking only
    List<ScalingStepInput> steps,      // step only
    Integer exhaustionThreshold,       // demand-pressure only
    Long latencyThresholdMs,           // demand-pressure only
    String cooldown,                   // duration string: "60s", "5m", or plain seconds
    String scaleInCooldown             // duration string
) {}

public record ScalingStepInput(double threshold, int adjustment) {}
```

**Parsing logic** (in `PoolService`):

| `scalingType` value | ScalingConfig variant | Required fields |
|---|---|---|
| `"target-tracking"` | `TargetTrackingConfig` | `targetFillRatio` (defaults to 0.7) |
| `"step"` | `StepConfig` | `steps` (non-empty list) |
| `"demand-pressure"` | `DemandPressureConfig` | `exhaustionThreshold`, `latencyThresholdMs` |
| `"none"` | `NoScalingConfig.INSTANCE` | (none) |
| any other string | `CustomScalingConfig(beanName)` | (none — bean lookup happens at policy creation time) |

`cooldown` and `scaleInCooldown` default to 60s when omitted. Duration parsing supports `"60s"`, `"5m"`, or plain number (interpreted as seconds).

**Update flow:**
1. Validate pool exists in `AgentPoolManagerRegistry`
2. If capacity fields present: call `mgr.adjustMaxActive()` and `defRegistry.updateCapacity()`
3. If `scalingType` present: parse into `ScalingConfig`, call `defRegistry.updateScaling()`, call `scalingScheduler.invalidatePolicy()`
4. Return the updated `PoolDetailResponse` (not just echo the input)

**Validation rules:**
- `minActive >= 0`
- `maxActive >= 1` and `maxActive >= minActive`
- `targetFillRatio` in `(0.0, 1.0]`
- `exhaustionThreshold >= 0`
- `latencyThresholdMs > 0`
- `steps` must be non-empty, thresholds unique, scale-in thresholds strictly below all scale-out thresholds (delegated to `StepScalingPolicy.validateSteps()`)

Invalid input returns 400 with a descriptive error message.

#### 1.4 PoolService — extracted business logic

**File:** `app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java`

Extracts all business logic from `PoolResource` into an `@ApplicationScoped` service. The mcpDomain API delegates to this service. The deprecated `PoolResource` also delegates here (DRY — no duplicated logic during deprecation window).

Responsibilities:
- Pool listing, detail, session queries
- Capacity and scaling config updates (with validation)
- Scaling config parsing (all 5 types)
- Duration parsing
- Building response views (`ScalingView`, `DemandView`)

Injected dependencies: `AgentPoolManagerRegistry`, `AgentPoolDefinitionRegistry`, `ScalingScheduler`, `MeterRegistry`.

#### 1.5 PoolDetailResponse — typed scaling config

The current `PoolDetail.ScalingView` uses `Object config` for the scaling configuration. Replace with a typed structure so the dashboard can render type-specific controls:

```java
public record ScalingView(
    String type,
    ScalingConfigView config,      // typed, not Object
    DecisionView lastDecision,
    String cooldownRemaining
) {}

public record ScalingConfigView(
    Double targetFillRatio,             // non-null for target-tracking
    List<ScalingStepView> steps,        // non-null for step
    Integer exhaustionThreshold,        // non-null for demand-pressure
    Long latencyThresholdMs,            // non-null for demand-pressure
    String beanName,                    // non-null for custom
    String cooldown,                    // formatted duration
    String scaleInCooldown              // formatted duration
) {}
```

All fields nullable — only the fields relevant to the current scaling type are populated. This avoids a discriminated union in JSON while keeping the response self-describing.

### 2. SSE — Pool Event Streaming

#### 2.1 SSE endpoint

**File:** `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventsResource.java`

```java
@HandWrittenEndpoint("SSE streaming — cannot be expressed as mcpDomain")
@Path("/api/claudony/pools")
@Authenticated
public class PoolEventsResource {
    @Inject PoolEventBus poolEventBus;
    @Inject PoolService poolService;

    @GET
    @Path("/{name}/events")
    @Produces("text/event-stream")
    public Multi<String> poolEvents(@PathParam("name") String name) {
        // validate pool exists, then subscribe
        return poolEventBus.subscribe(name,
            () -> poolService.buildPoolSnapshot(name));
    }
}
```

Follows the same pattern as `SessionResource.caseEvents()`: returns `Multi<String>`, sends an initial snapshot on connect, then streams events.

The SSE endpoint lives at `/api/claudony/pools/{name}/events` — same base path as the mcpDomain API, but in a separate `@HandWrittenEndpoint` class since mcpDomain can't express `Multi<String>` SSE streams.

#### 2.2 PoolEventBus

**File:** `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventBus.java`

Thin `@ApplicationScoped` wrapper that:
1. Manages per-pool `Multi<String>` subscriptions
2. On connect: sends initial snapshot (full pool detail JSON)
3. Forwards events from `EventBroadcaster` topics (`pool:<name>:scaling`, `pool:<name>:session`, `pool:<name>:health`)
4. Handles subscriber lifecycle (cleanup on disconnect)

Modelled on `ChannelEventBus` — same subscribe/emit/cancel pattern.

The `PoolEventEmitter` already broadcasts to the correct topics. No changes to `PoolEventEmitter` or `PoolEventEmitterProducer`.

#### 2.3 Event format

Events arrive as JSON with a `type` discriminator:

```json
// scaling decision
{"type": "scaling", "direction": "OUT", "count": 1, "reason": "...", "previousMax": 5, "newMax": 6, "timestamp": "..."}

// session lifecycle
{"type": "session", "event": "CREATED", "instanceId": "...", "identity": "...", "reason": "...", "timestamp": "..."}

// health change
{"type": "health", "previous": "HEALTHY", "current": "DEGRADED", "reason": "...", "timestamp": "..."}
```

These are already the formats emitted by `PoolEventEmitter`. No changes needed.

### 3. Dashboard — Editable Controls + SSE

#### 3.1 Path migration

All `/api/pools` references in `claudony-pool-panel.ts` change to `/api/claudony/pools`.

Affected calls (6 total):
- `fetchWithAuth('/api/pools')` → list
- `fetchWithAuth('/api/pools/${name}')` → detail
- `fetchWithAuth('/api/pools/${name}/sessions')` → sessions
- `fetchWithAuth('/api/pools/${name}/sessions/${id}/suspend')` → suspend
- `fetchWithAuth('/api/pools/${name}/sessions/${id}/resume')` → resume
- `fetchWithAuth('/api/pools/${name}/sessions/${id}')` → destroy (path changes to `.../destroy` for mcpDomain mutation pattern)

The destroy call changes from `DELETE` to `POST` (mcpDomain mutations are all POST via action paths).

#### 3.2 SSE integration

Replace `setInterval` 10s polling:

```typescript
// on pool selection or connectedCallback
private _connectSSE() {
  if (this._eventSource) this._eventSource.close();
  this._eventSource = new EventSource(
    `/api/claudony/pools/${this._selectedPool}/events`
  );
  this._eventSource.onmessage = (e) => {
    const data = JSON.parse(e.data);
    this._handleEvent(data);
  };
  this._eventSource.onerror = () => {
    // fallback: reconnect after delay, or revert to polling
  };
}
```

**Event handling:**
- `type: "scaling"` → update `_detail.scaling.lastDecision`, append to event log
- `type: "session"` → re-fetch sessions list (or update inline for known events)
- `type: "health"` → update health dot in sidebar
- Initial snapshot message → update `_detail` and `_sessions` from full payload

**Fallback:** Keep a 60s backup poll for resilience if EventSource fails. Clear the interval when SSE is connected.

#### 3.3 Capacity editing

In the KPI row, make Min and Max values editable:

```html
<div class="kpi-card">
  <input type="number" .value=${d.status.min}
         @change=${(e) => this._updateCapacity({ minActive: +e.target.value })} />
  <div class="kpi-label">Min</div>
</div>
<div class="kpi-card">
  <input type="number" .value=${d.status.max}
         @change=${(e) => this._updateCapacity({ maxActive: +e.target.value })} />
  <div class="kpi-label">Max</div>
</div>
```

`_updateCapacity` calls `POST /api/claudony/pools/{name}/update` with the capacity fields.

#### 3.4 Scaling config editing

Expand the scaling section from read-only text to an interactive form:

**Type selector:** Dropdown with options `none`, `target-tracking`, `step`, `demand-pressure`. Selecting a type shows/hides the relevant fields.

**Type-specific fields:**

| Type | Fields |
|---|---|
| `target-tracking` | Target fill ratio: range slider 0.1–1.0, step 0.05 |
| `step` | Threshold/adjustment pairs: editable table rows with add/remove |
| `demand-pressure` | Exhaustion threshold: number input; Latency threshold (ms): number input |
| `none` | No fields — scaling disabled |

**Shared fields:** Cooldown and scale-in cooldown inputs (text, e.g. "60s").

**Save button:** Calls `POST /api/claudony/pools/{name}/update` with the scaling fields. On success, updates the local state. On error, shows the validation error message.

**UX flow:** Editing is inline — no modal or separate page. The scaling section shows current config values as defaults in the form fields. Changes are not applied until Save is clicked (not on every keystroke).

#### 3.5 Event log

The placeholder event log section becomes a live feed:

```typescript
@state() private _events: PoolEvent[] = [];

private _handleEvent(data: PoolEvent) {
  this._events = [data, ...this._events].slice(0, 50); // keep last 50
  // also update detail/sessions as appropriate
}
```

Each event renders as a row: timestamp, type badge (scaling/session/health), and a summary line. Most recent first. Capped at 50 entries (in-memory, not persisted).

### 4. Deprecation

| Endpoint class | Path | Action |
|---|---|---|
| `PoolResource` | `/api/pools` | Add `@Deprecated(since = "0.3", forRemoval = true)`. Delegate to `PoolService`. |
| `AgentPoolResource` | `/api/agent-pools` | Already deprecated — no changes. |

The old `PoolResource` endpoints remain functional. The dashboard switches to the new paths immediately. No backward-compatibility shim beyond keeping the old class.

### 5. Testing

#### 5.1 Java tests

| Test class | Type | Covers |
|---|---|---|
| `PoolServiceTest` | Unit | Scaling config parsing (all 5 types), validation errors (bad thresholds, invalid durations, empty steps), capacity update logic, duration parsing |
| `ClaudonyPoolApiTest` | `@QuarkusTest` | RestAssured against `/api/claudony/pools/*`: list, detail, sessions, updatePool (target-tracking, step, demand-pressure, none, custom), suspend/resume/destroy, 404 for unknown pool |
| `PoolEventsResourceTest` | `@QuarkusTest` | SSE connection, initial snapshot delivery, event delivery on scaling decision, content-type `text/event-stream` |
| `PoolEventBusTest` | Unit | Subscribe/emit, pool isolation, subscriber count, cancel cleanup |

#### 5.2 Frontend tests

| Test | Type | Covers |
|---|---|---|
| vitest: capacity editing | Unit | `_updateCapacity` calls correct endpoint with correct payload |
| vitest: scaling type switching | Unit | Type dropdown shows/hides correct fields |
| Playwright: pool panel | E2E | Scaling section renders, capacity editing round-trip, scaling type change round-trip, event log populates |

#### 5.3 Test count impact

Estimated ~25-30 new Java tests, ~5-8 new vitest, 3-4 new E2E assertions.

---

## Files Changed

### New files
- `app/src/main/java/io/casehub/claudony/server/api/ClaudonyPoolApi.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolService.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolUpdateRequest.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ScalingStepInput.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventsResource.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolEventBus.java`
- `app/src/main/java/io/casehub/claudony/server/fleet/ScalingConfigView.java`
- `app/src/test/java/io/casehub/claudony/server/fleet/PoolServiceTest.java`
- `app/src/test/java/io/casehub/claudony/server/api/ClaudonyPoolApiTest.java`
- `app/src/test/java/io/casehub/claudony/server/fleet/PoolEventsResourceTest.java`
- `app/src/test/java/io/casehub/claudony/server/fleet/PoolEventBusTest.java`

### Modified files
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolResource.java` — add `@Deprecated`, delegate to `PoolService`
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolDetail.java` — update `ScalingView` to use typed `ScalingConfigView`
- `app/src/main/webui/src/components/claudony-pool-panel.ts` — path migration, SSE, editing controls, event log
- `CLAUDE.md` — update test count, add new endpoints to key URLs

### Deleted files
- `app/src/main/java/io/casehub/claudony/server/fleet/ScalingConfigUpdate.java` — replaced by `PoolUpdateRequest`
- `app/src/main/java/io/casehub/claudony/server/fleet/CapacityUpdate.java` — replaced by `PoolUpdateRequest`

---

## References

- `app/src/main/java/io/casehub/claudony/server/api/ClaudonySessionApi.java` — mcpDomain pattern reference
- `app/src/main/java/io/casehub/claudony/server/api/ClaudonyPeerApi.java` — mcpDomain mutation pattern reference
- `app/src/main/java/io/casehub/claudony/server/fleet/PoolResource.java` — existing implementation being migrated
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingConfig.java` — sealed interface with 5 variants
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinitionRegistry.java` — runtime mutation support (`updateScaling`, `updateCapacity`)
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/ScalingScheduler.java` — `invalidatePolicy()` cache invalidation flow
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolEventEmitter.java` — event broadcasting (scaling, session, health)
- `app/src/main/java/io/casehub/claudony/server/push/PoolEventEmitterProducer.java` — `EventBroadcaster` wiring
- `app/src/main/java/io/casehub/claudony/server/CaseEventBroadcaster.java` — SSE subscriber pattern
- `app/src/main/java/io/casehub/claudony/server/SessionResource.java:31-40` — SSE endpoint pattern (`Multi<String>`, initial snapshot)
- `app/src/main/java/io/casehub/claudony/server/ChannelEventBus.java` — event bus pattern (subscribe/emit/cancel)
- `app/src/main/webui/src/components/claudony-pool-panel.ts` — existing dashboard panel
- `app/src/main/webui/src/components/worker-panel.ts:97` — EventSource usage pattern in frontend
- Memory `feedback_mcp-domain-rest.md` — mcpDomain mandatory for REST endpoints
- `io.casehub.platform.api.mcp.McpDomain` (platform-api jar) — annotation definition
- GitHub #241, #242
