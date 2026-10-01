## D1: Move PoolResource from @HandWrittenEndpoint to @McpDomain

**Choice:** Move pool operations to mcpDomain pattern
**Alternatives:**
- Keep hand-written, add MCP separately — doubles the maintenance surface
- Split reads/writes (mcpDomain reads, hand-written writes) — inconsistent, confusing for consumers
**Rationale:** Pool scaling operations are a primary use case for controller Claude agents. The existing `@HandWrittenEndpoint("Pool management — infrastructure CRUD, not domain entities")` rationale doesn't hold when agents manage pools as a core workflow. mcpDomain gives REST + GraphQL + MCP from one definition.
**Trade-offs:** Requires migrating the dashboard from `/api/pools` to `/api/claudony/pools`. Old endpoints can be deprecated.
**Sources:** Memory `feedback_mcp-domain-rest.md`, existing mcpDomain APIs in `app/src/main/java/io/casehub/claudony/server/api/`
**Exploration:** quick
**Status:** captured

## D2: Base path — /api/claudony/pools

**Choice:** `/api/claudony/pools` following the existing claudony namespace convention
**Alternatives:**
- Keep `/api/pools` as basePath — breaks the `/api/claudony/*` convention used by sessions, peers, cases, mesh
**Rationale:** Consistency with all other mcpDomain APIs. All Claudony mcpDomain endpoints use `/api/claudony/<domain>`.
**Trade-offs:** Dashboard frontend needs path updates (6 references in `claudony-pool-panel.ts`).
**Depends on:** D1 (mcpDomain adoption)
**Sources:** `ClaudonySessionApi.java`, `ClaudonyPeerApi.java`, `ClaudonyCaseApi.java`, `ClaudonyMeshApi.java`
**Exploration:** quick
**Status:** captured

## D3: Coarse mutation granularity — single updatePool

**Choice:** One `updatePool` mutation that accepts both capacity and scaling config changes
**Alternatives:**
- Fine: separate updateCapacity + updateScaling — more precise but more tool sprawl in the MCP surface
- Fine + control: capacity + scaling + enable/disable — most expressive, most tools
**Rationale:** A single mutation is simpler for agents ("adjust this pool") and matches the dashboard UX where editing happens in one form. Internally, the handler updates both registry and invalidates the policy cache.
**Trade-offs:** Less granular MCP tool surface. An agent that only wants to change capacity still sends the full update object (with scaling fields null/omitted).
**Depends on:** D1 (mcpDomain adoption)
**Sources:** `PoolResource.updateCapacity()`, `PoolResource.updateScaling()`, `AgentPoolDefinitionRegistry.updateScaling()`, `AgentPoolDefinitionRegistry.updateCapacity()`
**Exploration:** quick
**Status:** captured

## D4: Dashboard — editable controls + live SSE event stream

**Choice:** Interactive editing controls for capacity and scaling config, plus SSE integration replacing 10s polling
**Alternatives:**
- Read-only enrichment — better display but no editing UI; editing stays CLI/MCP-only
- Editable controls without SSE — editing works but no real-time feedback
**Rationale:** Having the API without a UI to use it defeats the purpose of #242. SSE makes scaling decisions visible in real-time (the `PoolEventEmitter` already emits events, just needs an endpoint and frontend consumer).
**Trade-offs:** More frontend work. SSE adds lifecycle complexity (reconnect on pool switch, error handling).
**Depends on:** D1 (mcpDomain provides the mutation endpoint), D3 (single updatePool mutation)
**Sources:** `claudony-pool-panel.ts` (existing panel), `PoolEventEmitter.java` (existing event emitter), `PoolEventEmitterProducer.java` (broadcaster wiring)
**Exploration:** quick
**Status:** captured

## D5: Pool-scoped SSE endpoint

**Choice:** `GET /{name}/events` returns SSE for a single pool
**Alternatives:**
- Global stream (`GET /events` for all pools, client-side filtering) — one connection but noisier, simpler reconnect
**Rationale:** Matches the existing Claudony SSE pattern (channel events are channel-scoped). Clean separation — frontend opens EventSource for the selected pool, closes on pool switch. Less noise per connection.
**Trade-offs:** Frontend must manage EventSource lifecycle on pool selection change. Multiple pools can't be monitored from one connection (not a current UI requirement).
**Depends on:** D4 (SSE integration)
**Sources:** `ChannelEventBus.java` (existing channel SSE pattern), `CaseEventBroadcaster.java` (SSE fan-out pattern)
**Exploration:** quick
**Status:** captured
