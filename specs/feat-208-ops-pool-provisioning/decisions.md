# Decisions — #208 Ops Pool Provisioning

## D1: UI location

**Choice:** Claudony dashboard — add a "Pools" tab to the existing 5-tab layout
**Alternatives:**
- Scaffold ops perspective — cross-repo coupling, scaffold can consume the same REST API later
- Both (Claudony primary) — deferred scaffold integration adds unnecessary scope
- REST API only — half a feature for ops provisioning
**Rationale:** Claudony owns the pool backend; co-locating the UI avoids cross-repo coupling and delivers a self-contained feature
**Trade-offs:** Scaffold's ops perspective doesn't get pool management natively (but can consume the REST API later)
**Sources:** scaffold#52, claudony app.ts tab system, scaffold perspective.yaml
**Exploration:** quick
**Status:** captured

## D2: API and feature scope

**Choice:** Full CRUD — comprehensive read endpoints + mutation endpoints + dashboard tab. Absorbs #241 and #242 scope.
**Alternatives:**
- Read + display only — mutations deferred to #241
- Minimal — basic expansion of existing endpoint
**Rationale:** The backend already supports all operations (adjustMaxActive, suspend, resume, destroy). REST exposure is mechanical, not new logic. A read-only dashboard for a pre-release project is half a feature.
**Trade-offs:** Larger scope; #241 and #242 become redundant (close them)
**Sources:** AgentSessionManager API, existing GET /api/agent-pools endpoint
**Exploration:** quick
**Status:** captured

## D3: Data refresh strategy

**Choice:** Hybrid — REST polling for status + SSE push for scaling decisions and state changes
**Alternatives:**
- REST polling only — simpler but less responsive for scaling events
- SSE only — heaviest to implement, overkill for most state
**Rationale:** Pool status changes infrequently (polling is fine), but scaling decisions are event-driven and benefit from push notification
**Trade-offs:** More backend complexity than pure polling (needs SSE broadcaster)
**Sources:** CaseEventBroadcaster pattern, ChannelEventBus pattern, session-panel polling, worker-panel SSE
**Exploration:** quick
**Status:** captured

## D4: UI layout

**Choice:** List-detail pattern — pool list sidebar with health dots, selected pool detail in main area
**Alternatives:**
- Single expandable table — simpler but crowded with many pools
- Card grid — visual but less data-dense
**Rationale:** Scales well as pool count grows, matches ops management UX conventions, allows dense session/metrics display in the detail area
**Trade-offs:** More complex component structure (two coordinated views)
**Sources:** claudony-fleet-panel.ts (sidebar pattern), claudony-mesh-panel.ts (overview/detail views)
**Exploration:** quick
**Status:** captured

## D5: REST API structure

**Choice:** New `/api/pools` path with standard REST verbs, SSE stream at `/api/pools/events`
**Alternatives:**
- /api/agent-pools — preserves existing naming but more verbose, single existing endpoint returns flat object (breaking change to make it a list)
- Action-based — POST /api/pools/{name}/actions/* — non-standard REST
**Rationale:** Existing /api/agent-pools has no consumers. Shorter path, clean REST conventions, no backward compat concerns. SSE follows CaseEventBroadcaster pattern.
**Trade-offs:** Existing AgentPoolResource at /api/agent-pools becomes dead code (deprecate or remove)
**Sources:** AgentPoolResource.java, CaseEventBroadcaster pattern
**Exploration:** quick
**Status:** captured

## D6: SSE event scope

**Choice:** Scaling decisions + session lifecycle — push scaling decisions (scale-out/in with reason), session state changes (created/suspended/resumed/destroyed), pool health changes. Poll demand metrics and session detail.
**Alternatives:**
- Scaling only — simpler SSE surface, poll everything else
- Everything — push all state changes including demand metrics updates
**Rationale:** Scaling decisions and session lifecycle are event-driven (discrete occurrences). Demand metrics are continuous (better suited to polling or time-series query).
**Trade-offs:** Dashboard needs both push subscription and polling timer
**Sources:** CaseEventBroadcaster pattern, EventBroadcaster (pages push)
**Exploration:** quick
**Status:** captured

## D7: Detail view content

**Choice:** Full ops dashboard — pool status header (health, capacity bar, min/max/active/idle) + session table (identity, workingDir, state, idle time, memory, actions) + scaling state (current policy, last decision, cooldown timer) + historical demand metrics graph + scaling decision log + eviction history
**Alternatives:**
- Status + sessions only — simpler but incomplete ops view
- Status + sessions + scaling — no historical data
**Rationale:** Ops perspective needs full observability to provision and manage pools effectively
**Trade-offs:** Requires time-series storage for historical charts; more complex UI
**Sources:** AgentPoolStatus, PoolSnapshot, DemandMetrics, ManagedSession, ScalingDecision
**Exploration:** quick
**Status:** captured

## D8: Metrics instrumentation

**Choice:** Micrometer + Prometheus registry — standard Quarkus metrics instrumentation via quarkus-micrometer-registry-prometheus. Pool gauges (active, idle, max, fill_ratio), counters (acquires, evictions, exhaustions, scaling decisions), timers (acquire duration). Exposed at /q/metrics.
**Alternatives:**
- OpenTelemetry metrics — less mature in Quarkus for app metrics, more oriented to tracing
- Custom ring buffer — ad-hoc, non-standard, reinvents what Micrometer provides
**Rationale:** Micrometer is the Quarkus-recommended metrics API, registry-agnostic (TSDB backend is independent of instrumentation)
**Trade-offs:** None significant — Micrometer is the standard
**Sources:** quarkus.io/guides/telemetry-micrometer, existing quarkus-opentelemetry dep (tracing only)
**Exploration:** quick
**Status:** captured

## D9: Time-series storage

**Choice:** Apache IoTDB with flat label set adapter — Java-native TSDB, Table Model for labeled metrics, potentially embeddable (32MB min runtime). Complements existing casehub-iot project. Single TSDB serves both application metrics and IoT telemetry across the platform.
**Alternatives:**
- VictoriaMetrics — Prometheus-compatible drop-in, works with pages' existing Prometheus DataProvider today, but Go binary (external process, not Java-native)
- TimescaleDB — PostgreSQL extension, reuses existing Qhorus PostgreSQL, but not embeddable
- Defer — instrument with Micrometer only, choose TSDB later
**Rationale:** casehub-iot already exists; IoTDB serves both application metrics and IoT. Java-native aligns with the all-Java Quarkus platform. Table Model handles Prometheus-style labeled metrics via flat label adapter. One TSDB for the entire platform.
**Trade-offs:** More upfront work than VictoriaMetrics (need flat label adapter + pages IoTDB DataProvider). IoTQL is non-standard. GraalVM native image compat needs validation.
**Depends on:** D8 (Micrometer instrumentation)
**Sources:** iotdb.apache.org, casehub-iot, pages DataProvider SPI, pages Prometheus DataProvider
**Exploration:** deep-analysis
**Status:** captured

## D10: Real-time push mechanism

**Choice:** Pages EventBroadcaster — emit pool events (scaling decisions, session lifecycle, health changes) via pages' existing push infrastructure. Topic-based (pool:default:scaling, pool:default:session, etc.). EventStore persists for replay on reconnect. WebSocket delivery to dashboard.
**Alternatives:**
- Custom SSE broadcaster (CaseEventBroadcaster pattern) — Claudony-specific, duplicates pages infrastructure
- REST polling only — simpler but no real-time updates
**Rationale:** Pages EventBroadcaster is built, tested, and supports topic wildcards, sequence-based replay, and durable storage (JDBC/Redis). No reason to build a parallel push infrastructure.
**Trade-offs:** Couples Claudony's pool UI to pages push infrastructure (acceptable — pages is a platform dependency)
**Sources:** pages EventBroadcaster, EventStore SPI, TopicRegistry wildcard matching
**Exploration:** quick
**Status:** captured

## D11: #208 scope — proves out IoTDB for the platform

**Choice:** #208 is the issue that proves out IoTDB integration end-to-end. Pool metrics are the first consumer. Includes: flat label adapter (Micrometer → IoTDB), pages IoTDB DataProvider, Claudony pool instrumentation, REST API, dashboard tab with historical charts. Cross-repo: Claudony + pages.
**Alternatives:**
- Split — #208 builds REST API + dashboard + Micrometer instrumentation; IoTDB adapter and pages DataProvider are separate issues. Historical charts deferred.
- VictoriaMetrics now, IoTDB later — fastest path for #208 but defers platform TSDB story
**Rationale:** Building the adapter with a real consumer (pool metrics) validates the approach before casehub-iot and other components adopt it. Cross-repo work is manageable since pages is now in this slot.
**Trade-offs:** Larger scope for #208. IoTDB adapter is platform infrastructure, not Claudony-specific — but proving it here is the right first step.
**Sources:** casehub-iot, pages DataProvider SPI, slot 202 layout
**Exploration:** quick
**Status:** captured
