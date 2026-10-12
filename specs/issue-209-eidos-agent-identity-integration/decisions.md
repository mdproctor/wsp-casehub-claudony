# Decisions — Issue #209: Eidos Agent Identity Integration

## D1: Integration direction

**Choice:** Eidos drives pools — AgentDescriptors are the source of truth for which agents exist; Claudony pools provide the LLM backing.
**Alternatives:**
- Pools register into Eidos — pool config stays source of truth, Eidos gets a mirror. Simpler but wrong direction of authority.
- Bidirectional sync — both directions, more complex, potentially circular dependencies.
**Rationale:** Eidos defines agent identity at the org level. Claudony is an infrastructure provider, not an identity authority. The pool manager shouldn't decide which agents exist — it should provision backing for agents that Eidos says exist.
**Trade-offs:** Pool creation depends on Eidos data being correct and present. Misconfigured descriptors → missing or misconfigured pools.
**Sources:** AgentDescriptor (eidos-api), AgentPoolDefinition (claudony), issue #209 body
**Exploration:** quick
**Status:** captured

## D2: Scope of descriptor consumption

**Choice:** Core identity + capabilities — agentId, name, capabilities (with ModelQuery), providerConfigs. Pool needs to know WHAT to run and HOW.
**Alternatives:**
- Full descriptor — carry everything through. Over-coupled; disposition/voice/archetype are system-prompt concerns, not pool concerns.
- Just agent identity — only agentId + name. Too minimal; can't derive model requirements or provider config.
**Rationale:** Pool provisioning needs model requirements (from capabilities) and provider config. Personality/voice/disposition belong in the system prompt pipeline (WorkerContextProvider), not the pool layer.
**Trade-offs:** If future pool logic needs disposition or archetype data, the bridge must be extended. Unlikely — those are rendering concerns.
**Sources:** AgentDescriptor fields, AgentCapability.model()/modelRef(), ProviderConfig record
**Exploration:** quick
**Status:** captured

## D3: Org layer inclusion

**Choice:** Flat AgentRegistry only — no org structure relationships in this integration.
**Alternatives:**
- Include org structure — OrgStructure drives pool topology. Premature; pool provisioning doesn't need to know about supervision chains.
- Org-aware but deferred — design for it, don't build it. Unnecessary abstraction without a use case.
**Rationale:** Pool provisioning is per-agent: "this agent needs LLM backing with these model requirements." Org relationships (SUPERVISES, DELEGATES_TO) matter for case orchestration, not resource allocation.
**Trade-offs:** If org structure later needs to influence pool co-location or sizing, a separate concern must be introduced.
**Sources:** OrgStructure.java, OrganizationalUnit.java, Membership.java (eidos org-api)
**Exploration:** quick
**Status:** captured

## D4: Pool configuration source

**Choice:** Eidos `ProviderConfig` (providerName="claudony") feeds into the engine's `ProvisionerConfigRegistry`, which Claudony already reads via `CompositeProviderConfigSource`. Pool sizing (minActive, maxActive) and other pool config are carried as ProviderConfig entries. Sensible defaults (min=0, max=5) when no ProviderConfig exists for "claudony".
**Alternatives:**
- AgentDescriptor.extensionData() with well-known key — creates a parallel config channel that bypasses the existing ProvisionerConfigRegistry path. Untyped bag with no schema validation.
- Separate claudony-side config mapping (agentId → pool sizing in application.properties) — scatters configuration across two systems.
**Rationale:** Claudony already has `CompositeProviderConfigSource` calling `registry.configFor("claudony", agentId)`. The infrastructure path exists — this decision uses it rather than creating a parallel channel. Eidos `ProviderConfig` has `providerName` + `Map<String,String> config`, which maps cleanly to pool sizing keys.
**Trade-offs:** String-typed config values require parsing (e.g. "minActive" → int). The mapping is simple and bounded.
**Sources:** ProviderConfig (eidos-api), ProvisionerConfigRegistry (engine-api), CompositeProviderConfigSource (claudony-casehub), decision review R1-02/R1-12
**Exploration:** quick
**Status:** revised (review R1-02, R1-12)

## D5: Bridge lifecycle

**Choice:** Startup read + 30s polling reconciliation. No CDI events — `AgentRegistry` has no event mechanism and adding one is out of scope.
**Alternatives:**
- Startup only — read once, require restart for changes. Too rigid for a reactive system.
- CDI events + polling — the ideal, but AgentRegistry doesn't fire events today. Would require eidos-api changes.
**Rationale:** AgentRegistry is a plain interface (register/findById/find) with no event infrastructure. Polling reconciliation at 30s provides eventual consistency without requiring Eidos API changes. The bridge reads the registry at startup, then reconciles every 30s. Reconciliation is idempotent — it compares current registry state to current pool state and applies the diff.
**Trade-offs:** Up to 30s delay for descriptor changes to propagate to pools. Acceptable — pool topology changes are infrequent.
**Sources:** AgentRegistry interface (eidos-api, no event methods), AgentPoolDefinitionRegistry, decision review R1-04
**Exploration:** quick
**Status:** revised (review R1-04)
**Depends on:** D4

## D6: Eidos dependency model

**Choice:** Config-driven activation via `claudony.eidos.pool-bridge.enabled=false` (default off). The bridge bean is guarded by `@IfBuildProperty`. `casehub-eidos-api` is an optional compile dependency — present on classpath but bridge only activates when explicitly enabled.
**Alternatives:**
- @DefaultBean / Instance.isResolvable() — won't work: `NoOpAgentRegistry` from engine-runtime-core is always resolvable, so the bridge would always activate even without real Eidos data.
- Required dependency — couples all Claudony deployments to Eidos.
**Rationale:** `NoOpAgentRegistry` is always on the classpath (engine-runtime-core provides it). Classpath detection and Instance.isResolvable() both give false positives. A config property is explicit, testable, and matches how other optional features are gated (e.g. `claudony.casehub.enabled`).
**Trade-offs:** Operators must set a config property. But this is consistent with `claudony.casehub.enabled=true` — the bridge is an integration feature, not a default.
**Sources:** NoOpAgentRegistry (engine-runtime-core), claudony.casehub.enabled pattern, decision review R1-06
**Exploration:** quick
**Status:** revised (review R1-06)

## D7: Session identity stamping

**Choice:** Stamp sessions with eidosAgentId so downstream consumers (lineage, dashboard, context providers) can resolve the full AgentDescriptor.
**Alternatives:**
- No stamping — sessions don't know about Eidos. Loses the ability to trace sessions back to their org-level identity.
**Rationale:** Lightweight (just a string in tmux user options or Session model). Enables the dashboard to show agent archetype/name, lineage to reference the Eidos identity, and context providers to inject Eidos-sourced briefings.
**Trade-offs:** Session model gains a nullable field. Downstream consumers must handle the null case (non-Eidos sessions).
**Sources:** Session model, TmuxService user options (@claudony_state pattern)
**Exploration:** quick
**Status:** captured

## D8: Bridge architecture

**Choice:** Single `EidosPoolBridge` @ApplicationScoped startup bean that reads AgentRegistry, converts descriptors to AgentPoolDefinitions, and registers them.
**Alternatives:**
- AgentDescriptorRegistrar SPI implementation — awkward round-trip (push descriptors to Eidos, read them back). The registrar SPI is for declaring agents, not consuming them.
- Quarkus extension / build-time processing — overkill, loses reactive lifecycle, significant build infrastructure for a simple bridge.
**Rationale:** Simplest path. Fits existing Claudony patterns (startup beans, registry). Single integration point with clear responsibility.
**Trade-offs:** All mapping logic in one class — could be refactored if conversion grows complex, but unlikely given the scoped field set (D2).
**Sources:** AgentRegistry interface, AgentPoolDefinitionRegistry, ServerStartup pattern
**Exploration:** quick
**Status:** captured
