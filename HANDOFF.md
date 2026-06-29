# Handoff — 2026-06-29

**Head commit (project):** `ed82477` — fix: adapt to engine SNAPSHOT — WorkerExecutionManager.supports()

## What landed this session

### claudony#163 — Mesh system prompt delivery to CLI

Workers now receive the mesh system prompt (`MeshSystemPromptTemplate` output) via `--append-system-prompt` on the Claude CLI command. The provisioner extracts `systemPrompt` from `ProvisionContext.workerContext().properties()`. `WorkerCommandBuilder` supports `--system-prompt` and `--append-system-prompt` coexisting (mutual exclusion removed). Static (operator) append first, dynamic (mesh) second.

`MeshSystemPromptTemplate` guards null `workerId` — engine passes null because the session ID is generated later in the provisioning pipeline.

### claudony#164 — ProvisionerConfigRegistry SPI infrastructure

`ProvisionerConfigRegistry` SPI added to `casehub-engine-api` (engine#584). `CompositeProviderConfigSource` replaces `ConfigMappingProviderConfigSource` — registry primary, config-mapping fallback. Registry-wins when present (full replacement, no per-field merge). `declaredAgentIds()` returns union of both sources.

Design spec: `specs/2026-06-29-mesh-prompt-delivery-ops-config-design.md` (adversarial review: 4 rounds, 12 issues, all resolved).
Garden entry: GE-20260629-b049bb — null string concatenation latent bug pattern.

### Engine SNAPSHOT adaptation

`WorkerExecutionManager.supports(providerName, capabilityName)` — new abstract method. `CompositeWorkerExecutionManager` excluded via `quarkus.arc.exclude-types`.

## State

- main: `ed82477` (3 commits: feat #163, feat #164, SNAPSHOT fix)
- 601 tests pass (16 core + 177 casehub + 408 app)
- #163 closed, #164 closed

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
| #161 | Adopt casehub-pages for UI via Quinoa | S | Low | Frontend architecture shift |
| #141 | ActionRiskClassifier oversight gate | M | High | Blocked on engine#402 |
| — | casehub-ops DeploymentProvisionerConfigRegistry | S | Low | ops#24 — producer side of #164's SPI |
| — | OpenClaw AgentProviderConfigSource migration | S | Low | openclaw#56 — consolidate parallel SPI |
