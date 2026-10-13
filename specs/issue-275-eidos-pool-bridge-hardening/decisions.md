## D1: System prompt layering (#278)

**Choice:** Eidos identity first, mesh context after
**Alternatives:**
- Merge into MeshSystemPromptTemplate — tighter coupling, single source, but mixes identity with operational context
- Separate --system-prompt flags — cleanest separation but relies on CLI merging behavior
**Rationale:** Identity (who you are, what you do) is logically prior to operational context (channels, prior workers). WorkerCommandBuilder already supports --system-prompt and --append-system-prompt coexisting, so this layers naturally without new CLI assumptions.
**Trade-offs:** Two prompt-building paths to maintain (Eidos template + Mesh template) rather than one unified template.
**Sources:** WorkerCommandBuilder.java (mergeAppendPrompts), MeshSystemPromptTemplate.java, claudony #278
**Exploration:** quick
**Status:** captured

## D2: Reconciliation scope (#277)

**Choice:** Capacity + config updates in place
**Alternatives:**
- Capacity only — safer but requires pool destroy/recreate for config changes
- Full replace — cleanest semantics but kills running sessions
**Rationale:** Running sessions are unaffected (they keep their original config). Only new sessions pick up changes. This covers both scaling changes (minActive/maxActive) and identity changes (command, model, workingDir) without disruption.
**Trade-offs:** Pool state is mutable — a change that affects running sessions (e.g. a security-critical command change) won't take effect until sessions cycle.
**Sources:** EidosPoolBridge.java (reconcile loop), AgentPoolDefinition.java, claudony #277
**Exploration:** quick
**Status:** captured

## D3: Multi-tenant pool isolation (#280)

**Choice:** Composite key (agentName, tenancyId)
**Alternatives:**
- Tenant-scoped registries — stronger isolation but more complex lifecycle management
- Pool-level tenant tag — simpler than scoped registries but weaker isolation, cross-tenant bugs possible
**Rationale:** Single registry with composite key is backwards-compatible (tenancyId="default" for existing single-tenant pools). Pool names become "agentName@tenancyId". Lookups are tenant-aware. No new registry lifecycle to manage.
**Trade-offs:** Isolation is logical (key-based) not physical (separate registries). A registry-level bug could theoretically affect all tenants. Acceptable for current scale.
**Sources:** AgentPoolDefinitionRegistry (current flat map), EidosPoolBridge.java, claudony #280
**Exploration:** quick
**Status:** captured

## D4: Model resolution strategy (#279)

**Choice:** Optional ModelRegistry injection with fallback
**Alternatives:**
- Required injection — simpler but breaks standalone pool bridge without registry
- SPI with @DefaultBean — fully pluggable but adds indirection for a single consumer
**Rationale:** EidosPoolConverter takes an optional ModelRegistry. When present, resolves ModelQuery → ModelChain via registry (ranked results with fallback ordering). When absent, falls back to direct family mapping (current behavior). No hard dependency on registry infrastructure.
**Trade-offs:** Two code paths (with/without registry) to test. Direct mapping path may drift from registry behavior over time.
**Sources:** EidosPoolConverter.java (toPoolDefinition), eidos-api ModelQuery/ModelChain, claudony #279
**Exploration:** quick
**Status:** captured
