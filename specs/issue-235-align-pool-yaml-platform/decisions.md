# Decisions — #235 Align Agent Pool Canonical Layer with Platform YAML Language

## D1: Design scope — structural alignment, not literal @StepPlugin

**Choice:** Follow the platform's structural pattern (record + APT + schema generation + @Required/@Optional) without forcing pool definitions into the executable step model. No @Execute method, no StepResult, no StepAction.
**Alternatives:**
- Literal @StepPlugin adoption — would force an execution semantic onto purely declarative config, creating an artificial @Execute method that "registers" the pool definition
- Minimal (YAML parser only) — would leave the annotation scanner divergent from platform patterns
**Rationale:** Pool definitions are configuration declarations, not executable steps. The value is in typed validation, JSON Schema generation, and classpath discovery — not in the execution model.
**Trade-offs:** Can't reuse AptPluginSource directly; need a pool-specific classpath scanner (but it's small and follows the same pattern).
**Sources:** StepPlugin.java, AgentPoolDefinition.java, issue #235 body
**Exploration:** quick
**Status:** captured

## D2: YAML parser uses full StepDefinition + StepValidator

**Choice:** Define a StepDefinition with typed StepParameters describing the pool schema. Parse YAML to Map, validate with StepValidator, then extract typed values. Removes all manual stringValue/intValue/parsePolicy/parseEviction helpers.
**Alternatives:**
- Types only, custom extraction — would use StepParameterType for coercion but keep pool-specific extraction logic; misses the validation benefits
- Keep Jackson, add validation layer — keeps dual parsing paths (Jackson + StepDefinition), more code for no gain
**Rationale:** Full adoption gives typed validation for free, consistent error messages, and makes the pool parser a thin wrapper rather than a manual extraction engine.
**Trade-offs:** Adds casehub-platform-yaml-core as a dependency to claudony-casehub. This is a lightweight, zero-dependency library so the cost is minimal.
**Sources:** StepDefinition.java, StepValidator.java, StepParameterType.java, AgentPoolYamlParser.java
**Exploration:** quick
**Status:** captured

## D3: Flat StepDefinition with dotted keys for nested pool structure

**Choice:** Flatten the pool: sub-map to dotted keys (pool.min-active, pool.max-active, pool.eviction) so StepValidator's flat input model works directly. Agent-level fields (working-dir, policy, command) are top-level inputs with appropriate types and allowedValues constraints.
**Alternatives:**
- Two separate StepDefinitions — more precise typing but requires custom composition logic not supported by StepValidator
- OBJECT type for pool — simple but inner fields aren't type-validated, defeating the purpose
**Rationale:** Matches StepValidator's flat model, parallels AgentPoolConfigBridge's existing dotted-key convention (pool.minActive), and keeps the YAML surface unchanged.
**Trade-offs:** Requires a flatten step before validation — the parser must merge pool: sub-map entries with a "pool." prefix before passing to StepValidator.
**Sources:** AgentPoolConfigBridge.java (dotted key convention), StepValidator.java (flat model)
**Exploration:** quick
**Depends on:** D2 (StepDefinition adoption)
**Status:** captured

## D4: Separate pool APT processor in Claudony

**Choice:** Create a new PoolDefinitionProcessor in Claudony (the claudony-casehub module) that follows the same APT pattern as StepPluginProcessor but emits pool-specific schema and manifest. Lives in Claudony, not in the platform.
**Alternatives:**
- Extend StepPluginProcessor — would require the platform to know about pool semantics, violating boundary rules (platform shouldn't depend on Claudony)
- Shared processor framework in platform — premature abstraction; only two consumers (steps and pools), and their emission needs differ
**Rationale:** Clean separation — pool processing doesn't pollute the step plugin pipeline. The platform owns step concerns; Claudony owns pool concerns. The APT pattern is simple enough to replicate without a shared base.
**Trade-offs:** Some code duplication between StepPluginProcessor and PoolDefinitionProcessor (schema emission logic). Acceptable given the boundary constraint.
**Sources:** StepPluginProcessor.java, SchemaEmitter.java, BinderEmitter.java, boundary-rules.md
**Exploration:** quick
**Depends on:** D1 (structural alignment)
**Status:** captured

## D5: Single record replaces @PooledAgent + @AgentPool

**Choice:** Merge into a single @PoolDefinition record with all fields: @Required name, @Optional workingDir, policy, command, minActive, maxActive, eviction. The APT processor generates schema from this single record.
**Alternatives:**
- Keep two annotations — preserves agent/pool semantic split but adds complexity to APT processing (must handle cross-annotation composition)
- Record + nested record — most type-safe but APT processing of nested records adds significant complexity for no practical gain
**Rationale:** Matches the @StepPlugin pattern (one record = one definition). Simpler APT processing. The agent/pool semantic split was useful for annotations (you can have @PooledAgent without @AgentPool) but in a record model, optional fields with defaults serve the same purpose.
**Trade-offs:** Loses the ability to annotate a class with just @PooledAgent — but this was never practically useful since a pool definition without pool config just gets defaults.
**Sources:** PooledAgent.java, AgentPool.java, StepPlugin pattern (ValidPlugin.java)
**Exploration:** quick
**Depends on:** D4 (separate APT processor)
**Status:** captured
