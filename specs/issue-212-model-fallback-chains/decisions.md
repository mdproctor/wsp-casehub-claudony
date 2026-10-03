## D1: Execution path scope

**Choice:** Both CLI sessions (WorkerCommandBuilder) and programmatic routing (RoutingAgentProvider)
**Alternatives:**
- CLI only — immediate value but leaves programmatic path without fallback
- Programmatic only — correct long-term but Claudony doesn't use this path for worker provisioning today
**Rationale:** Shared chain definition consumable by both paths future-proofs the design
**Trade-offs:** More implementation surface than either path alone
**Sources:** RoutingAgentProvider.java (platform router), WorkerCommandBuilder.java (CLI command builder), AgentSessionManager.java (pool session management)
**Exploration:** quick
**Status:** captured

## D2: Fallback trigger scope

**Choice:** All three levels — resolution (model not in registry), operational (pool full, budget exceeded), and runtime (API errors, early session exit)
**Alternatives:**
- Resolution only — simplest, deterministic, but misses the most valuable fallback scenarios
- Resolution + operational — handles pool/budget but not API failures
**Rationale:** Full coverage handles the real-world failure modes operators care about
**Trade-offs:** Runtime fallback for CLI sessions requires circuit-breaker pattern (detect early exit, retry). Mid-conversation failures can't trigger fallback.
**Sources:** ModelRegistry.resolveById() (resolution), AgentSessionManager.status()/BudgetTracker (operational), ClaudonyWorkerExecutionManager (runtime exit detection)
**Exploration:** quick
**Status:** captured

## D3: Configuration surface

**Choice:** Pool YAML only — chains defined per-pool in agent-pools YAML
**Alternatives:**
- Platform manifest + pool YAML — two config surfaces, maximum flexibility but more maintenance
- ModelQuery chain in API — most generic but operators think in model names not query predicates
**Rationale:** Pool YAML is where operators configure pools today. Single source of truth.
**Trade-offs:** Platform router doesn't get its own fallback config — Claudony must bridge
**Sources:** AgentPoolYamlParser.java, AgentPoolSchema.java, AgentPoolDefinition.java
**Exploration:** quick
**Status:** captured

## D4: Cross-backend fallback

**Choice:** Supported — chain can span backends (Claude → Gemini → Ollama)
**Alternatives:**
- Same-backend only — simpler, just a list of model names, but artificially limits fallback options
**Rationale:** First-principles analysis shows no architectural blockers. AgentBackend abstracts protocol/auth differences. ModelRegistry maps models to backends. TmuxSessionOperations accepts per-session commands. Capability filtering via ModelDescriptor.capabilities() skips incompatible entries automatically.
**Trade-offs:** Chain entries need optional command override for non-Claude backends. More testing surface.
**Sources:** AgentBackend interface (protocol abstraction), BackendInstanceRegistry.resolve() (backend lookup), TmuxSessionOperations (per-session command), ModelDescriptor.capabilities() (capability filter)
**Exploration:** deep-analysis
**Status:** captured

## D5: Chain entry format

**Choice:** Structured entries — plain string (model name) OR object with model + optional command/backend override
**Alternatives:**
- Model names only — simpler but no per-entry command override for cross-backend
- ModelQuery objects — most powerful but verbose in YAML
**Rationale:** Covers both simple same-backend chains (`[opus, sonnet]`) and cross-backend chains with command overrides
**Trade-offs:** YAML parser must handle both string and map entries
**Sources:** AgentPoolYamlParser.java (existing YAML parsing patterns)
**Exploration:** quick
**Status:** captured

## D6: Resolution logic home

**Choice:** Claudony-owned resolver (Approach B) — new FallbackChainResolver in claudony-casehub/fleet/
**Alternatives:**
- Platform SPI (Approach A) — FallbackChain + ModelAvailabilityCheck SPI in platform-api, resolution in RoutingAgentProvider. Clean SPI but requires cross-repo changes for a single consumer today.
- Hybrid (Approach C) — chain data types in platform-api, resolution in Claudony. Shareable data structure but platform API change for a type only Claudony uses.
**Rationale:** All config and state needed for resolution lives in Claudony (pool YAML, ModelRegistry access, pool/budget state, session lifecycle). No platform changes needed. Uses existing platform APIs (ModelRegistry.resolveById()) without new SPI. Extract to platform later if pattern proves useful.
**Trade-offs:** Other platform consumers can't reuse the resolver. Must be extracted if needed elsewhere.
**Depends on:** D1 (both paths), D2 (all three trigger levels), D3 (pool YAML config)
**Sources:** RoutingAgentProvider.java (existing resolution), ModelRegistry interface (resolution checks), AgentSessionManager (operational checks), ClaudonyWorkerExecutionManager (runtime exit detection)
**Exploration:** deep-analysis
**Status:** captured
