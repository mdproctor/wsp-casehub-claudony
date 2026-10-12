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

**Choice:** AgentDescriptor.extensionData() with well-known key `claudony.pool`, plus sensible defaults when absent.
**Alternatives:**
- Separate claudony-side config mapping (agentId → pool sizing in application.properties) — works but scatters configuration across two systems.
- Eidos ProviderConfig — semantic fit but string-typed Map<String,String>, awkward for structured pool config (nested scaling, budget).
**Rationale:** extensionData is designed for provider-specific configuration. No Eidos API changes needed. Defaults (min=0, max=5) keep things working when extensionData is absent.
**Trade-offs:** Configuration lives in the Eidos descriptor rather than in Claudony's own config. Operators must understand where to configure pool sizing.
**Sources:** AgentDescriptor.extensionData(), AgentPoolDefinition.PoolConfig
**Exploration:** quick
**Status:** captured

## D5: Bridge lifecycle

**Choice:** CDI events for immediate reaction to AgentDescriptor changes, plus 30s polling reconciliation loop for drift safety.
**Alternatives:**
- Startup only — read once, require restart for changes. Too rigid for a reactive system.
- Polling only — eventually consistent but up to 30s lag on every change. Wasteful without events.
- CDI events only — immediate but can miss events (CDI async delivery, transient failures).
**Rationale:** Belt-and-suspenders: events handle the happy path instantly, reconciliation catches anything events miss. 30s matches the existing case-worker-heartbeat-ms=30000 pattern.
**Trade-offs:** Slightly more complex than either approach alone. Reconciliation must be idempotent.
**Sources:** AgentPoolDefinitionRegistry, existing @Scheduled patterns in Claudony
**Exploration:** quick
**Status:** captured

## D6: Eidos dependency model

**Choice:** Optional — guard with @DefaultBean / Instance.isResolvable() pattern. Claudony works without Eidos; bridge activates only when casehub-eidos-api is on the classpath.
**Alternatives:**
- Required dependency — simpler code but couples all Claudony deployments to Eidos.
**Rationale:** Claudony is deployed in contexts where Eidos may not be present (standalone terminal management, simple fleet setups). The bridge is an enhancement, not a prerequisite.
**Trade-offs:** Conditional activation code adds some complexity. Must be tested in both with-Eidos and without-Eidos configurations.
**Sources:** Existing @DefaultBean patterns (NoOpAgentProvider, EmptyCaseLineageQuery), fleet-script decisions.md (D2: "Eidos isn't on Claudony's classpath")
**Exploration:** quick
**Status:** captured

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
