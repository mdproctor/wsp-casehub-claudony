# Handoff — 2026-06-25

**Head commit (project):** `8322aa9` — feat(#156): add ProviderConfigSource SPI, migrate provisioner, remove WorkerCommandResolver

## What landed this session

### claudony#156 — Per-agent provider config

Workers now get per-agent CLI configuration (model, system prompt, tools, effort, permissions, working dir) at provision time. `WorkerCommandResolver` removed — replaced by `ProviderConfigSource` SPI keyed by `context.taskType()`, fixing non-deterministic Set iteration when multiple capabilities were configured.

New types: `ClaudonyProviderConfig` (record), `ProviderConfigSource` (SPI), `ConfigMappingProviderConfigSource` (@DefaultBean), `WorkerCommandBuilder`.

Config migration: `workers.commands.*` → `workers.provider-config.*.command` + `workers.default-command` (@WithDefault("claude")).

Also fixed pre-existing `FleetMessageRelayObserverTest` breakage from Qhorus SNAPSHOT API change (Instant timestamp parameter added to MessageReceivedEvent).

Garden entry: GE-20260625-a6bc3b — engine provision() passes ALL capabilities, not the specific one.

## State

- main: `8322aa9` (2 squashed commits from 6)
- 576 tests pass
- #156 closed

## Upstream fix pending

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
| #161 | Adopt casehub-pages for UI via Quinoa | L | High | Frontend architecture shift |
| — | Mesh system prompt delivery to CLI | S | Med | Gap discovered: WorkerContext.properties("systemPrompt") built but not consumed by provisioner |
| — | casehub-ops co-deployment (OpsProviderConfigSource) | M | Med | Phase 2 of #156 — bridge to DeploymentProviderConfigStore |
