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

**Choice:** Hybrid — REST polling for status + real-time push for scaling decisions and state changes. Push mechanism is pages EventBroadcaster (WebSocket) via existing casehub-qhorus-push integration — see D10.
**Alternatives:**
- REST polling only — simpler but less responsive for scaling events
- Push only — heaviest to implement, overkill for most state
**Rationale:** Pool status changes infrequently (polling is fine), but scaling decisions are event-driven and benefit from push notification. EventBroadcaster also provides automatic history via EventStore — pool snapshots are persisted and replayable for historical charts (see D9 revised).
**Trade-offs:** More backend complexity than pure polling (needs EventBroadcaster integration — but already on classpath via casehub-qhorus-push)
**Depends on:** D10 (push mechanism)
**Sources:** QhorusWebSocketBroadcaster (already uses EventBroadcaster), pages EventStore SPI, session-panel polling
**Exploration:** quick
**Status:** revised (R1-02: removed CaseEventBroadcaster/ChannelEventBus references; clarified EventStore history)

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

**Choice:** New `/api/pools` path with standard REST verbs. No SSE endpoint — real-time push handled by D10 (EventBroadcaster over WebSocket).
**Alternatives:**
- /api/agent-pools — preserves existing naming but more verbose, single existing endpoint returns flat object (breaking change to make it a list)
- Action-based — POST /api/pools/{name}/actions/* — non-standard REST
**Rationale:** Existing /api/agent-pools has no consumers. Shorter path, clean REST conventions, no backward compat concerns. Push delivery is a separate concern handled by EventBroadcaster (D10), not an SSE endpoint on the REST resource.
**Trade-offs:** Existing AgentPoolResource at /api/agent-pools becomes dead code (deprecate or remove)
**Depends on:** D10 (push mechanism), D12 (multi-pool URL structure)
**Sources:** AgentPoolResource.java, QhorusWebSocketBroadcaster pattern
**Exploration:** quick
**Status:** revised (R1-02: removed SSE endpoint; push is D10's concern, not the REST API's)

## D6: Push event scope

**Choice:** Scaling decisions + session lifecycle — push scaling decisions (scale-out/in with reason), session state changes (created/suspended/resumed/destroyed), pool health changes via EventBroadcaster topics (e.g. `pool:{name}:scaling`, `pool:{name}:session`, `pool:{name}:health`). Poll demand metrics and session detail via REST.
**Alternatives:**
- Scaling only — simpler push surface, poll everything else
- Everything — push all state changes including demand metrics updates
**Rationale:** Scaling decisions and session lifecycle are event-driven (discrete occurrences). Demand metrics are continuous (better suited to polling or EventStore replay for historical charts). EventBroadcaster topic-based routing allows selective subscription.
**Trade-offs:** Dashboard needs both push subscription (WebSocket via EventBroadcaster) and polling timer (REST for demand metrics detail)
**Depends on:** D10 (EventBroadcaster push mechanism)
**Sources:** EventBroadcaster topic-based routing, TopicRegistry wildcard matching, ScalingDecision, AgentSessionManager lifecycle events
**Exploration:** quick
**Status:** revised (R1-02: changed from SSE to EventBroadcaster WebSocket push; aligned with D10)

## D7: Detail view content

**Choice:** Full ops dashboard — pool status header (health, capacity bar, min/max/active/idle) + session table (identity, workingDir, state, idle time, memory, actions) + scaling state (current policy, last decision, cooldown timer) + historical demand metrics graph + scaling decision log + eviction history
**Alternatives:**
- Status + sessions only — simpler but incomplete ops view
- Status + sessions + scaling — no historical data
**Rationale:** Ops perspective needs full observability to provision and manage pools effectively
**Trade-offs:** Requires IoTDB for historical charts (see D9); more complex UI
**Sources:** AgentPoolStatus, PoolSnapshot, DemandMetrics, ManagedSession, ScalingDecision
**Exploration:** quick
**Status:** revised (R2-01: trade-off text updated — EventStore replaces TSDB reference)

## D8: Metrics instrumentation

**Choice:** Micrometer + Prometheus registry — standard Quarkus metrics instrumentation via quarkus-micrometer-registry-prometheus. Pool gauges (active, idle, max, fill_ratio), counters (acquires, evictions, exhaustions, scaling decisions), timers (acquire duration). Exposed at /q/metrics. IoTDB flat label adapter reads from Micrometer registry and writes to IoTDB.
**Alternatives:**
- Native pool metrics via EventBroadcaster only — skips Micrometer, no /q/metrics, no external scraping. Avoids a dependency but loses standard instrumentation.
- OpenTelemetry metrics — less mature in Quarkus for app metrics, more oriented to tracing
**Rationale:** Micrometer is the Quarkus-recommended metrics API and the industry standard for JVM metrics. Registry-agnostic — the same gauges/counters feed both IoTDB (via flat label adapter) and /q/metrics (for external Prometheus/Grafana). Adding Micrometer now avoids re-instrumenting later.
**Trade-offs:** Adds quarkus-micrometer-registry-prometheus dependency
**Sources:** quarkus.io/guides/telemetry-micrometer, existing quarkus-opentelemetry dep (tracing only)
**Exploration:** quick
**Status:** restored (user override: IoTDB stays in #208, Micrometer is the instrumentation layer)

## D9: Time-series storage

**Choice:** Apache IoTDB with flat label set adapter — Java-native TSDB, Table Model for labeled metrics. Pool metrics are the first consumer, proving out IoTDB for the platform. Complements existing casehub-iot project. Single TSDB serves both application metrics and IoT telemetry.
**Alternatives:**
- VictoriaMetrics — Prometheus-compatible drop-in, works with pages' existing Prometheus DataProvider today, but Go binary (external process, not Java-native)
- Pages EventStore only — bounded ring buffer, no aggregation/downsampling. Works for real-time push but insufficient for time-range queries and historical charts.
- TimescaleDB — PostgreSQL extension, reuses existing Qhorus PostgreSQL, but not embeddable
- Defer — instrument with Micrometer only, choose TSDB later (creates rework)
**Rationale:** casehub-iot already exists; IoTDB serves both application metrics and IoT. Java-native aligns with the all-Java Quarkus platform. Table Model handles Prometheus-style labeled metrics naturally. Building the adapter with a real consumer (pool metrics) validates the approach before other components adopt it. Separating IoTDB to a later issue creates rework (instrument twice).
**Trade-offs:** More upfront work than EventStore-only (flat label adapter + pages IoTDB DataProvider). IoTDB documentation quality is poor. GraalVM native image compat needs validation. Docker dependency for dev.
**Depends on:** D8 (Micrometer instrumentation feeds the flat label adapter)
**Sources:** iotdb.apache.org, casehub-iot, pages DataProvider SPI, IoTDB Table Model SQL, IoTDB Java Session API
**Exploration:** deep-analysis
**Status:** restored (user override: IoTDB stays in #208 — avoiding rework from separating)

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

**Choice:** #208 delivers end-to-end: REST API (`/api/pools`), dashboard Pools tab, Micrometer instrumentation, IoTDB flat label adapter, pages IoTDB DataProvider, EventBroadcaster push for real-time updates, IoTDB-backed historical charts. Cross-repo: Claudony + pages. Absorbs #241 and #242.
**Alternatives:**
- Split: #208 builds REST API + dashboard + EventStore history; IoTDB adapter and pages DataProvider are separate issues. Rejected by user: creates rework (instrument with EventStore, then re-instrument with IoTDB).
- EventStore only — bounded ring buffer, no aggregation. Works for real-time but insufficient for proper time-series queries.
**Rationale:** Building the IoTDB integration with a real consumer (pool metrics) validates the approach before casehub-iot and other components adopt it. Separating creates rework. Cross-repo work is manageable since pages is in this slot. The integration work is straightforward Java library integration, not exotic.
**Trade-offs:** Larger scope. IoTDB Docker dependency for dev. Flat label adapter and pages IoTDB DataProvider are platform infrastructure built within a feature issue — but they're proven by a real consumer.
**Sources:** casehub-iot, pages DataProvider SPI, IoTDB Session API, slot 202 layout
**Exploration:** quick
**Status:** restored (user override: IoTDB stays in #208)

## D12: Multi-pool REST API URL structure

**Choice:** Resource-per-pool — `/api/pools/{name}` with sub-resources for sessions and scaling
**Alternatives:**
- Query parameter — `/api/pools?name=default` — non-RESTful for a named resource
- Flat endpoints — `/api/pools` returns all pools with embedded sessions — doesn't scale, violates HATEOAS
**Rationale:** `AgentPoolManagerRegistry` keys pools by name (`Map<String, AgentSessionManager>`). `AgentPoolDefinitionRegistry` keys by agent name. Pool names are stable identifiers from configuration, not UUIDs. Natural URL hierarchy: `/api/pools` (list), `/api/pools/{name}` (detail), `/api/pools/{name}/sessions` (pool's sessions), `/api/pools/{name}/scaling` (scaling config and state).
**Trade-offs:** None significant — standard REST resource naming
**Sources:** AgentPoolManagerRegistry.get(String), AgentPoolDefinitionRegistry.get(String)
**Exploration:** quick (surfaced by R1-08)
**Status:** captured

## D13: Pool management scope — per-node vs fleet-wide

**Choice:** Per-node pool management. Read-only fleet aggregation as follow-on.
**Alternatives:**
- Fleet-wide mutations — coordinated adjustMaxActive/suspend/resume across nodes. Requires either REST-proxied mutations or shared state mechanism. High complexity for unclear benefit.
- Per-node only, no fleet visibility — simplest, but operators must visit each node's dashboard.
**Rationale:** `AgentSessionManager` manages local tmux sessions via `TmuxService`. Sessions are bound to the host — Node A's pool cannot create sessions on Node B. Pool mutations (adjustMaxActive, suspend, resume, destroy) are inherently node-local. Fleet-wide read aggregation can reuse the existing session federation pattern (`GET /api/pools` with `?local=true` guard to prevent recursive fan-out), but this is a separate concern.
**Trade-offs:** Multi-node operators see one node's pools per dashboard. Acceptable for pre-release deployment (1–2 nodes). Fleet aggregation is additive when needed.
**Sources:** AgentSessionManager (ConcurrentHashMap), TmuxService (local tmux), session federation (?local=true pattern)
**Exploration:** quick (surfaced by R1-09)
**Status:** captured

## D14: Authorization for pool mutations

**Choice:** `@RolesAllowed("admin")` for mutation endpoints; `@Authenticated` for read endpoints
**Alternatives:**
- `@Authenticated` for all — current pattern on AgentPoolResource. Insufficient: adjustMaxActive, destroySession, suspend are destructive ops that should be restricted.
- Fine-grained per-operation roles — overcomplicated for the current user model (single operator + fleet key).
**Rationale:** Pool mutations affect infrastructure availability. An authenticated user can view pool status, but only an admin can modify pool capacity or destroy sessions. Role infrastructure does not exist yet — `@RolesAllowed` is unused in the codebase and no `SecurityIdentityAugmentor` is present. Implementation requires either adopting `casehub-platform-oidc` or building a custom `SecurityIdentityAugmentor` that maps WebAuthn credentials to roles.
**Trade-offs:** Requires new role infrastructure (SecurityIdentityAugmentor or platform-oidc adoption) and admin role assignment during credential provisioning. The role infrastructure is new work, not a minor configuration step.
**Sources:** AgentPoolResource (@Authenticated), auth module (L2), WebAuthn CredentialStore, casehub-platform-oidc (not yet adopted)
**Exploration:** quick (surfaced by R1-13)
**Status:** revised (R2-01: corrected rationale — role infrastructure is new work, not pre-existing)

## D15: Frontend technology for pool dashboard

**Choice:** Pages components — pages-viz for charts, pages-table for session table, pages-ui for layout
**Alternatives:**
- Vanilla HTML/JS — inconsistent with the existing dashboard which already uses pages components
- Custom Web Components — reinvents what pages components provide
**Rationale:** Claudony's dashboard is already built with `@casehubio/pages-runtime` and `@casehubio/pages-ui` (see app.ts: `import { loadSite, registerPanel } from "@casehubio/pages-runtime"`). The pool panel follows the same pattern as session-panel, fleet-panel, mesh-panel: a custom element registered via `registerPanel()`, using pages components for data display. EventStore-backed charts use pages-viz with the pages DataProvider SPI.
**Trade-offs:** None — consistent with existing dashboard architecture
**Sources:** app.ts (pages-runtime, pages-ui imports), casehub-pages-npm dependency (already in app/pom.xml line 269)
**Exploration:** quick (surfaced by R1-13)
**Status:** captured

## D16: MCP tools for pool operations

**Choice:** Deferred — not in #208 scope
**Alternatives:**
- Include pool MCP tools — expose adjustMaxActive, suspend, resume, listPools as MCP tools for controller Claude
- Never — pool management is always human-operated
**Rationale:** Pool management in #208 is an operator/human concern. Controller Claude manages sessions (create, command, resize) via existing MCP tools — pool infrastructure is a layer below session management. If autonomous pool scaling via LLM controller becomes a need, MCP tools can be added as a follow-on.
**Trade-offs:** Controller Claude cannot programmatically manage pool capacity. Acceptable — ScalingScheduler provides automatic scaling; human intervention is the exception, not the norm.
**Sources:** ClaudonyMcpTools (8 session tools), ScalingScheduler (automatic scaling)
**Exploration:** quick (surfaced by R1-13)
**Status:** captured
