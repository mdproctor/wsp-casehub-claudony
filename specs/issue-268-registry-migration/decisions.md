## D1: Migration pattern — internal delegation vs strangler-fig

**Choice:** Internal delegation — modify existing classes to use RegistryService as backing store
**Alternatives:**
- Strangler-fig (extract interface + adapter, @Alternative @Priority) — matches Qhorus pattern but adds complexity unnecessary for a single pre-release app
- Hybrid (strangler-fig for PeerRegistry, internal delegation for AgentPoolDefinitionRegistry) — inconsistent pattern across the codebase
**Rationale:** Claudony is a single application controlling its own wiring. Pre-release platform means breaking changes cost nothing. No need to protect callers with an interface layer.
**Trade-offs:** Cannot run old and new implementations side-by-side for comparison. Less reversible than strangler-fig.
**Sources:** RegistryBackedInstanceManager (qhorus), platform#569, qhorus#489
**Exploration:** quick
**Status:** captured

## D2: AgentPoolDefinitionRegistry data model — dual-store vs single-store

**Choice:** Registry for lifecycle + parallel ConcurrentHashMap for rich data
**Alternatives:**
- JSON in metadata — serialize AgentPoolDefinition to JSON in a single metadata key. Loses type safety, awkward queries.
- Skip pool definitions — don't migrate. Pool definitions are config, not topology.
**Rationale:** RegistryEntry.metadata is Map<String,String>, but AgentPoolDefinition has deeply nested records (AgentConfig, PoolConfig, ScalingConfig, BudgetConfig). Registry provides discovery and relationships; the map provides type-safe domain data. Each store does what it's good at.
**Trade-offs:** Two stores to keep in sync. Registration and mutation must update both.
**Sources:** AgentPoolDefinitionRegistry, RegistryEntry record, AgentPoolDefinition
**Exploration:** quick
**Status:** captured

## D3: PoolMeshRegistrar — direct RegistryService vs keep InstanceService

**Choice:** Direct RegistryService — use registry.link/unlink for pool→session containment + health updates
**Alternatives:**
- Keep InstanceService, add relationships alongside — safer but PoolMeshRegistrar has two dependencies (InstanceService + RegistryService)
- Full bypass — replace all InstanceService calls, taking over instance registration entirely. Most complete but takes InstanceService's responsibility.
**Rationale:** With Qhorus's RegistryBackedInstanceManager active, InstanceService already delegates to RegistryService (double-hop). Pool sessions are registered as agent-instances by RegistryBackedInstanceManager. PoolMeshRegistrar adds relationship links on top — clean separation of concerns.
**Trade-offs:** Depends on RegistryBackedInstanceManager being active for session registration. If it's not active (build property disabled), sessions won't be in the registry and links will point to non-existent entries.
**Depends on:** D1 (internal delegation means PoolMeshRegistrar changes in place)
**Sources:** PoolMeshRegistrar, InstanceService, RegistryBackedInstanceManager, Relationship record
**Exploration:** quick
**Status:** captured
