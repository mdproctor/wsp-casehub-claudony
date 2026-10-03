## D1: Execution path scope

**Choice:** Both CLI sessions (WorkerCommandBuilder) and programmatic routing (RoutingAgentProvider), with path-specific resolution
**Alternatives:**
- CLI only — immediate value but leaves programmatic path without fallback
- Programmatic only — correct long-term but Claudony doesn't use this path for worker provisioning today
- Shared lowest-common-denominator abstraction — can't exploit either path's native capabilities
**Rationale:** Both paths serve different consumers and need fallback. Shared chain TYPE (ModelChain) with path-specific resolution: programmatic path uses RoutingAgentProvider.resolveChain() with full ModelRegistry; CLI path uses lightweight local resolution with availability filter.
**Trade-offs:** Two resolution paths to maintain, but each is simple and uses the path's native strengths
**Sources:** RoutingAgentProvider.java (platform router), WorkerCommandBuilder.java (CLI command builder), AgentSessionManager.java (pool session management)
**Exploration:** deep-analysis
**Status:** revised (was: shared abstraction; revised per decision review R1-03)

## D2: Fallback trigger scope

**Choice:** All three levels — resolution (model not in registry), operational (pool full, budget exceeded), and runtime (API errors, early session exit)
**Alternatives:**
- Resolution only — simplest, deterministic, but misses the most valuable fallback scenarios
- Resolution + operational — handles pool/budget but not API failures
- Resolution + operational for CLI, all three for programmatic — reliable coverage where implementable but inconsistent behavior
**Rationale:** Full coverage handles the real-world failure modes operators care about
**Trade-offs:** Runtime fallback for CLI sessions requires circuit-breaker pattern (detect early exit within grace period, retry). Mid-conversation failures can't trigger fallback. Early exit classification (API error vs fast completion vs crash) needs heuristics — exit within grace period + non-zero exit code = fallback candidate.
**Sources:** ModelRegistry.resolveById() (resolution), ModelAvailabilityFilter SPI (operational), ClaudonyWorkerExecutionManager (runtime exit detection)
**Exploration:** quick
**Status:** captured

## D3: Configuration surface

**Choice:** Pool YAML as operator-facing config, translated to platform ModelChain types at parse time
**Alternatives:**
- Platform manifest only — correct layering but operators already configure agents in pool YAML
- Platform manifest + pool YAML — two config surfaces to maintain
- ModelQuery chain in API — most generic but operators think in model names not query predicates
**Rationale:** Pool YAML is where operators configure agents today. Claudony translates YAML entries into platform ModelChain objects, bridging the operator config surface to the platform routing layer. Routing logic stays in the platform; config stays where operators expect it.
**Trade-offs:** Claudony must translate YAML → ModelChain. Platform router doesn't have its own standalone chain config. Acceptable because pool YAML is the only config surface operators use in Claudony.
**Sources:** AgentPoolYamlParser.java, AgentPoolSchema.java, AgentPoolDefinition.java, ManifestResult (platform config)
**Exploration:** deep-analysis
**Status:** revised (was: pool YAML only with Claudony-owned resolver; revised per decision review R1-02 to bridge to platform types)

## D4: Cross-backend fallback

**Choice:** Supported — chain can span backends (Claude → Gemini → Ollama)
**Alternatives:**
- Same-backend only — simpler, just a list of model names, but artificially limits fallback options
**Rationale:** First-principles analysis shows no architectural blockers. AgentBackend abstracts protocol/auth differences. ModelRegistry maps models to backends. TmuxSessionOperations accepts per-session commands. Capability filtering via ModelDescriptor.capabilities() skips incompatible entries automatically.
**Trade-offs:** Chain entries need optional command override for non-Claude backends. Cross-backend testing requires multi-backend test environment. More testing surface.
**Sources:** AgentBackend interface (protocol abstraction), BackendInstanceRegistry.resolve() (backend lookup), TmuxSessionOperations (per-session command), ModelDescriptor.capabilities() (capability filter)
**Exploration:** deep-analysis
**Status:** captured

## D5: Chain entry format

**Choice:** Structured entries using platform sealed interface — Named(String modelRef) | Queried(ModelQuery), with optional command override for CLI cross-backend
**Alternatives:**
- Model names only — simpler but no per-entry command override and no ModelQuery-based entries
- Full ModelQuery objects only — verbose in YAML, operators think in model names
- String-or-map polymorphic (original D5) — doesn't leverage platform's ModelQuery type system
**Rationale:** Named entries cover the common case (operator writes "opus"). Queried entries leverage ModelRegistry.query() for adaptive resolution (tier: FLAGSHIP adapts to catalog changes). Both map cleanly to RoutingAgentProvider's existing resolve() and resolveQuery() methods.
**Trade-offs:** YAML parser must handle string→Named, map-with-model→Named, map-with-tier/query→Queried
**Depends on:** D4 (cross-backend needs command override)
**Sources:** ModelQuery (platform-api), RoutingAgentProvider.resolve()/resolveQuery() (existing resolution), AgentPoolYamlParser.java
**Exploration:** deep-analysis
**Status:** revised (was: string-or-object; revised per decision review R1-05 to use platform ModelQuery type)

## D6: Resolution logic home

**Choice:** Platform-owned resolution with Claudony availability bridge
**Alternatives:**
- Claudony-owned resolver (original D6) — all logic in Claudony, no platform changes. Circular reasoning: config in Claudony → resolver in Claudony → contradicts #212's stated direction.
- Hybrid — chain types in platform, resolution in Claudony. Still doesn't put routing in the routing layer.
**Rationale:** Issue #212 and fleet manager spec #205 both state fallback chains belong in RoutingAgentProvider. RoutingAgentProvider.resolveChain() wraps existing resolve()/resolveQuery() in try-catch iteration — ~15 lines of new logic. ModelAvailabilityFilter SPI lets Claudony inject operational checks (pool/budget) without the platform knowing about pools. CLI path uses same chain type with lightweight local resolution.
**Trade-offs:** Requires cross-repo changes (platform-api + platform-agent-router-core + claudony). Acceptable at pre-release stage.
**Depends on:** D1 (both paths), D2 (all three trigger levels), D3 (YAML → platform types bridge)
**Sources:** RoutingAgentProvider.java (existing resolve/resolveQuery), ModelRegistry interface (resolution), issue #212 and #205 (architectural direction)
**Exploration:** deep-analysis
**Status:** revised (was: Claudony-owned; revised per decision review R1-01/R1-11 circularity finding)

## D7: Degraded provisioning notification

**Choice:** Record resolved model in ProvisionResult metadata when fallback occurs
**Alternatives:**
- Silent fallback — simpler but engine doesn't know it got a less capable worker
**Rationale:** Falling back from opus to haiku is a capability change. The engine should know so it can adjust expectations (simpler prompts, additional review gates, different mesh participation).
**Trade-offs:** Minimal — ProvisionResult already supports metadata. Engine must choose whether to act on it.
**Sources:** ProvisionResult (engine API), ClaudonyWorkerProvisioner (provisioning), decision review R1-08
**Exploration:** quick
**Status:** captured
