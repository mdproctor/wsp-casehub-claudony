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
**Trade-offs:** Requires time-series storage for historical charts; more complex UI
**Sources:** AgentPoolStatus, PoolSnapshot, DemandMetrics, ManagedSession, ScalingDecision
**Exploration:** quick
**Status:** captured

## D8: Metrics instrumentation

**Choice:** Native pool metrics via EventBroadcaster for #208. Micrometer instrumentation deferred to the platform TSDB issue.
**Alternatives:**
- Micrometer + Prometheus registry now — adds quarkus-micrometer-registry-prometheus dependency, /q/metrics endpoint, potential auth configuration. Correct choice when a TSDB is in place, but premature for #208 where EventStore provides history.
- OpenTelemetry metrics — less mature in Quarkus for app metrics, more oriented to tracing
**Rationale:** Pool metrics (PoolSnapshot, DemandMetrics) are captured natively by ScalingScheduler and broadcast as JSON payloads via EventBroadcaster → EventStore. No intermediate instrumentation layer needed. When a platform TSDB is adopted (separate issue), Micrometer becomes the instrumentation layer and pool metrics migrate to gauges/counters.
**Trade-offs:** No /q/metrics endpoint for external scraping. Acceptable for #208 — pool metrics are consumed by the dashboard, not external monitoring.
**Sources:** PoolSnapshot record, DemandMetrics record, ScalingScheduler.evaluatePool(), EventBroadcaster.broadcast()
**Exploration:** quick
**Status:** revised (R1-12, R1-04: Micrometer deferred — no TSDB in #208 scope, EventStore provides history)

## D9: Time-series storage

**Choice:** Pages EventStore for pool metrics history in #208. IoTDB remains the platform TSDB choice but is separated into its own issue (see D11 revised).
**Alternatives:**
- Apache IoTDB in #208 — original choice; Java-native, one TSDB for the platform. Valid at platform level but disproportionate for pool metrics and couples infrastructure PoC to feature delivery.
- VictoriaMetrics — Prometheus-compatible, works with existing pages PrometheusDataProvider. Valid for platform TSDB; overkill for #208.
- In-memory ring buffer — redundant; EventStore IS a bounded ring buffer with persistence.
- TimescaleDB — PostgreSQL extension, reuses existing Qhorus PostgreSQL, but not embeddable.
**Rationale:** EventBroadcaster.broadcast() automatically persists events via EventStore.append(topic, payloadJson). EventStore.replay(topic, sinceSeq, limit) provides historical data. JDBC and Redis implementations already exist (casehub-pages-push-store-jdbc, casehub-pages-push-store-redis). InMemoryEventStore provides a bounded default. Zero additional infrastructure for #208.
**Trade-offs:** EventStore is not a true TSDB — no aggregation queries, downsampling, or retention policies beyond ring buffer bounds. Sufficient for pool metrics dashboard charts; insufficient for long-term platform metrics. Platform TSDB (IoTDB or VictoriaMetrics) addresses the long-term need separately.
**Depends on:** D10 (EventBroadcaster provides the EventStore integration)
**Sources:** pages EventStore SPI, InMemoryEventStore, JdbcEventStore, RedisEventStore, EventBroadcaster.broadcast()
**Exploration:** deep-analysis (original IoTDB analysis retained for platform decision; EventStore alternative surfaced in review R1-03)
**Status:** revised (R1-03: EventStore replaces IoTDB for #208 pool metrics; IoTDB separated to platform issue)

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

## D11: #208 scope — focused on pool management

**Choice:** #208 delivers: REST API (`/api/pools`), dashboard Pools tab, EventBroadcaster push for real-time updates, EventStore-backed historical charts. Self-contained Claudony feature. IoTDB proof-of-concept is a separate platform issue.
**Alternatives:**
- Original: #208 proves out IoTDB end-to-end (flat label adapter + pages IoTDB DataProvider + pool instrumentation + REST + dashboard). Rejected: couples platform infrastructure PoC to feature delivery; IoTDB blockers delay pool UI.
- VictoriaMetrics now, IoTDB later — valid for the TSDB issue, not needed for #208 (EventStore suffices).
**Rationale:** Pool management UI and IoTDB platform integration are independent concerns with different lifecycles and risk profiles. #208 can ship with EventStore history. The IoTDB issue gets its own consumer validation (casehub-iot is the natural first consumer, not pool metrics).
**Trade-offs:** Historical charts use EventStore replay (bounded, no aggregation) rather than TSDB queries (unbounded, with downsampling). Acceptable for pool metrics — the data volume is low (hundreds of snapshots per day per pool).
**Sources:** EventStore SPI, casehub-iot, pages DataProvider SPI
**Exploration:** quick
**Status:** revised (R1-04: decoupled IoTDB from #208; pool metrics use EventStore; #241/#242 absorption confirmed)

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
**Rationale:** Pool mutations affect infrastructure availability. An authenticated user can view pool status, but only an admin can modify pool capacity or destroy sessions. Claudony's auth module (L2) already supports roles via WebAuthn credential metadata.
**Trade-offs:** Requires admin role assignment during credential provisioning. Minor setup cost.
**Sources:** AgentPoolResource (@Authenticated), auth module (L2), WebAuthn CredentialStore
**Exploration:** quick (surfaced by R1-13)
**Status:** captured

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
