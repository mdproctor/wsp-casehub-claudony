# Decisions — #271 Ops View

## D1: Primary consumer model

**Choice:** Composable — standalone for external shells (scaffold, trellis), also consumed by Claudony's own dashboard
**Alternatives:**
- External shells only — would defer internal migration, creating two data paths
- Claudony dashboard only — would couple the API to Claudony's UI assumptions
**Rationale:** The ops view API should be self-contained so any UI shell can compose it. Claudony's dashboard becomes just another consumer.
**Trade-offs:** Slightly more design effort to keep the API shell-agnostic
**Sources:** Issue #271 body, epic #267 staged delivery
**Exploration:** quick
**Status:** captured

## D2: API shape — new aggregation layer

**Choice:** New `ClaudonyOpsApi` `@McpDomain` that composes from existing services. Existing per-domain APIs (pools, peers, apps, registry, mesh) stay untouched.
**Alternatives:**
- Extend existing APIs — adds topology/health to each domain API; fragments the ops picture across 5 endpoints
- Replace with unified API — consolidates all reads into one API; too disruptive, breaks existing consumers
**Rationale:** Aggregation layer follows the same pattern as existing `@McpDomain` classes. Per-domain APIs retain fine-grained control for direct consumers.
**Trade-offs:** Some data appears in both per-domain and ops APIs (intentional — ops views are leaner summaries)
**Sources:** ClaudonyRegistryApi.java, ClaudonyPoolApi.java, ClaudonyAppApi.java, ClaudonyMeshApi.java, ClaudonyPeerApi.java
**Exploration:** quick
**Status:** captured

## D3: Live updates — SSE from day one

**Choice:** SSE endpoint for live topology updates, leveraging existing `RegistryEvent` CDI events and `PoolEventBus` pattern
**Alternatives:**
- Polling first, SSE later — simpler but delays real-time visibility, the core value proposition
- Both REST + SSE with deltas — full snapshot on connect, deltas after; more complex protocol
**Rationale:** SSE infrastructure already exists (PoolEventBus, PoolEventsResource). RegistryEvent CDI events provide the change signal. Pattern is proven.
**Trade-offs:** SSE adds connection management complexity; consumers must handle reconnection
**Sources:** PoolEventBus.java, PoolEventsResource.java, RegistryEvent.java
**Exploration:** quick
**Status:** captured

## D4: Topology depth — typed summaries

**Choice:** Group registry entries by type with per-type summary records (PoolOpsView, PeerOpsView, AppOpsView, MeshOpsView) including health + key metrics. Relationships as edges.
**Alternatives:**
- Raw registry entries — simpler server-side but pushes interpretation to every consumer
- Full graph structure — nodes carry full detail records (PoolDetail, ApplicationInfo); heaviest to build and serialize
**Rationale:** Typed summaries give consumers domain-aware data without parsing metadata maps, while staying leaner than full detail records. Consumers needing full detail can fall back to per-domain APIs.
**Trade-offs:** Summary types must be maintained alongside full detail types; but they're intentionally simpler
**Sources:** RegistryEntry.java, PoolSummary.java, PeerRecord.java, ApplicationInfo.java
**Exploration:** quick
**Status:** captured

## D5: Mesh scope — included

**Choice:** Include mesh channel activity (per-channel subscriber counts, message counts) in the ops view for a complete operational picture
**Alternatives:**
- Fleet infrastructure only — excludes mesh; consumers must separately call mesh APIs for channel state
- Aggregate mesh stats only — total counts without per-channel detail; loses the ability to spot specific channel issues
**Rationale:** QhorusDashboardService already exposes channel list with message counts. Including it gives a single-call operational picture.
**Trade-offs:** Ops snapshot grows; mesh data may change more frequently than fleet topology
**Sources:** QhorusDashboardService.java (listChannels, listInstances)
**Exploration:** quick
**Status:** captured

## D6: Architecture — single OpsService with snapshot model

**Choice:** Single `@ApplicationScoped OpsService` that composes from existing services to build a typed `OpsSnapshot`. `OpsEventBus` (same pattern as `PoolEventBus`) observes CDI events for live updates. Full snapshot on SSE connect.
**Alternatives:**
- Federated composition — separate endpoints per concern (fleet health, topology, pool utilisation, mesh activity); more granular but fragments the ops picture and requires client-side composition
- GraphQL aggregation — maximum query flexibility but introduces a new API paradigm; overkill for current consumers
**Rationale:** Matches existing patterns. Single snapshot is serialisable, cacheable, and gives consumers a complete picture in one call. Typed summary records map directly to blocks-ui components.
**Trade-offs:** Full snapshot on every SSE event is heavier than deltas; acceptable at current scale
**Sources:** PoolService.java, PoolEventBus.java, RegistryService.java, QhorusDashboardService.java, PeerRegistry.java, ApplicationRegistrationService.java
**Exploration:** quick
**Status:** captured
