# Decisions — #205 LLM Fleet Manager

## D1: Fleet manager scope — unified abstraction

**Choice:** A single fleet abstraction that manages both API connection pools AND CLI agent sessions. Workers declare what they need via manifests; the fleet manager provisions and routes to the right backing.
**Alternatives:**
- API connections only — simpler, but leaves CLI sessions unmanaged and creates two disjoint provisioning systems
- CLI sessions only — misses the engine's ChatModelProvider/LangChain4j path entirely
**Rationale:** The manifest system in engine already declares LLM configuration (provider, model, capabilities). The fleet manager should be the runtime counterpart — managing any type of configured LLM instance. Splitting by backing type would defeat the purpose of having a declarative manifest.
**Trade-offs:** More complex initial implementation; need a clean abstraction that spans CLI sessions and API pools without leaking either's internals.
**Sources:** ClaudonyWorkerProvisioner.java, ChatModelProvider (engine), AgentDescriptor (eidos), CaseDefinition YAML schema
**Exploration:** quick
**Status:** captured

## D2: Fleet manager lives in Claudony

**Choice:** Claudony owns the fleet manager. Engine defines what workers need (manifests, SPIs); Claudony manages the infrastructure — pools, scaling, routing, health.
**Alternatives:**
- Engine — would couple the coordination engine to infrastructure concerns it shouldn't know about
- New standalone module — adds ecosystem complexity for something that naturally belongs in the integration layer
**Rationale:** Claudony is already the integration layer that implements engine SPIs (WorkerProvisioner, CaseChannelProvider, etc.). Fleet management is how Claudony satisfies those SPIs more intelligently — pooling and routing instead of naive 1:1 provisioning. The engine just asks for workers; Claudony decides how to provide them.
**Trade-offs:** Fleet abstractions are Claudony-specific — a non-Claudony deployer would need their own fleet implementation. This is fine: the SPI boundary is the contract, not the fleet internals.
**Sources:** ClaudonyWorkerProvisioner.java, WorkerProvisioner SPI, ecosystem design spec
**Exploration:** quick
**Status:** captured

## D3: Pool semantics on existing BackendInstanceRegistry — not a new SPI

**Choice:** Extend the platform's existing model resolution stack (ModelRegistry, BackendInstanceRegistry, RoutingAgentProvider) with pool lifecycle semantics. Do NOT create a new ModelResolver SPI — model resolution already exists. Register Claudony CLI sessions as an AgentBackend so they're routable through the same RoutingAgentProvider as API backends.
**Alternatives:**
- New ModelResolver SPI in engine-api — REJECTED: duplicates ModelRegistry.query() + RoutingAgentProvider. The resolution infrastructure already exists.
- CDI override of ChatModelProvider — REJECTED: fragile, wraps engine internals
**Rationale:** The platform already has ModelDescriptor + ModelQuery + ModelRegistry + AgentBackend + BackendInstanceRegistry + RoutingAgentProvider. The gap isn't resolution — it's pool management. BackendInstanceRegistry is a flat register/resolve; fleet management adds capacity tracking, acquire/release, scaling, health, cost. Claudony's CLI sessions need to register as an AgentBackend so they participate in the same routing. Engine's Agent/ChatModelProvider path should converge onto RoutingAgentProvider (separate engine issue).
**Trade-offs:** Requires understanding and extending the platform routing stack rather than creating something simpler and self-contained. More correct but higher integration burden.
**Depends on:** D1 (unified scope), D2 (Claudony owns fleet)
**Sources:** ModelDescriptor (platform-api), ModelQuery (platform-api), ModelRegistry (platform-api), AgentBackend (agent-api), BackendInstanceRegistry (agent-api), RoutingAgentProvider (agent-router-core), Manifest (agent-config-core), AgentConverter (engine)
**Exploration:** deep-analysis (revised after discovering existing platform stack)
**Status:** captured

## D4: Scope — all four pieces, slot across platform + engine + claudony

**Choice:** Full scope: (1) register Claudony CLI sessions as AgentBackend, (2) add pool semantics to BackendInstanceRegistry, (3) converge engine's ChatModelProvider onto RoutingAgentProvider, (4) fleet lifecycle management. Work-slot across platform, engine, and claudony repos.
**Alternatives:**
- Claudony only, file upstream issues — lower risk but slower, pieces are tightly coupled
- Platform + Claudony, defer engine convergence — partial but misses the API path
**Rationale:** The pieces are tightly coupled — pool semantics in platform inform how Claudony registers backends and how engine converges. Designing and implementing coherently across all three avoids integration mismatches. The slot model supports this.
**Trade-offs:** Larger blast radius. Three-repo coordination. But the alternative is sequential issues with integration gaps between them.
**Depends on:** D3 (extending existing stack)
**Sources:** Slot 194 repo layout, platform + engine + claudony module structure
**Exploration:** quick
**Status:** captured

## D5: Capacity-bounded on-demand pool model

**Choice:** Each backend key has a capacity config (min/max instances). Instances created on-demand up to max. Below min, instances are pre-warmed. `acquire()` creates or reuses; `release()` returns to pool or destroys if over max.
**Alternatives:**
- Pre-provisioned fixed pools — simpler but no scaling; wastes resources when idle, can't handle spikes
- Elastic auto-scaling — scaling policies (step, target-tracking) based on demand metrics; more operational complexity than needed initially
**Rationale:** Suits both API backends (cheap to create, mostly on-demand) and CLI sessions (expensive to create, worth pre-warming to min). The min/max model is the natural first step — elastic policies can be layered on top later if needed, by making the scaling policy pluggable.
**Trade-offs:** min/max is static config — doesn't adapt to variable load patterns automatically. But it's predictable, debuggable, and covers the 80% case. Auto-scaling can be added as a policy option without changing the core pool model.
**Depends on:** D3 (pool semantics on BackendInstanceRegistry)
**Sources:** Connection pool patterns (HikariCP, database pools), container orchestration min/max replicas
**Exploration:** quick
**Status:** captured

## D6: Pool config extends the manifest YAML

**Choice:** Add a `pools:` section to the existing manifest YAML. Each pool references a model/backend and declares min, max, health check interval. `ManifestProcessor` populates the fleet manager at startup alongside `ModelRegistry` and `BackendInstanceRegistry`.
**Alternatives:**
- Separate fleet.yaml — cleaner separation but two files to manage; pool config is tightly coupled to the models it manages
- Quarkus application.properties — simplest but not portable, doesn't travel with the manifest
**Rationale:** The manifest is already the declarative config surface for models, providers, and sources. Pool config is an operational extension of the same declarations — "I have model X, pool it with min 2 max 5." Keeping it in one file means one source of truth for what LLM resources exist and how they're managed.
**Trade-offs:** Manifest grows in scope — it was purely about model declarations, now also covers operational config. But pool config per model is a natural concern co-located with the model declaration.
**Depends on:** D5 (pool model)
**Sources:** Manifest (agent-config-core), ManifestProcessor, existing SourceDeclaration/ProviderDeclaration patterns
**Exploration:** quick
**Status:** captured

## D7: CLI AgentBackend supports both invoke and openSession

**Choice:** The Claudony CLI backend supports both modes: `invoke()` for one-shot commands (run claude with a prompt, wait for exit, return result) and `openSession()` for interactive persistent sessions (tmux session with terminal streaming). The pool manages both modes — one-shot instances are released on completion, sessions are released when the worker finishes.
**Alternatives:**
- openSession only — ignores the valid use case of running claude as a one-shot tool (fire-and-forget tasks, batch processing)
- New backend type (SessionBackend) — unnecessary; the existing AgentBackend interface already has both methods, some backends support one or both
**Rationale:** Claude Code CLI naturally supports both patterns: `claude -p "do X"` (one-shot) and `claude` (interactive REPL). Both are backed by tmux sessions in Claudony. The pool's acquire/release lifecycle handles both — the difference is when release happens (invoke: on exit; session: on worker completion).
**Trade-offs:** Pool management needs to track which instances are in invoke vs session mode for proper lifecycle. But this is a per-instance state flag, not structural complexity.
**Depends on:** D3 (CLI as AgentBackend), D5 (pool model)
**Sources:** AgentBackend interface (agent-api), TmuxService.createSession vs createWorkerSession
**Exploration:** quick
**Status:** captured
