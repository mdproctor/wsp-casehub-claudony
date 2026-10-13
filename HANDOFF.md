# HANDOFF — casehub-claudony

## Last Session

Implemented #209 (Eidos agent identity integration for agent pools). Built `EidosPoolBridge` — an `@ApplicationScoped` bean guarded by `@IfBuildProperty` that reads `AgentDescriptor`s from the Eidos `AgentRegistry` at startup + 30s reconciliation and creates `AgentPoolDefinition`s for agents with a "claudony" `ProviderConfig`. Pure conversion in `EidosPoolConverter`. Added `eidosAgentId` nullable field to `AgentConfig` (source-compatible — existing constructors preserved as delegating constructors). `casehub-eidos-api` added as optional dependency. 17 new tests. Landed as `7b984a1` on main. Also fixed pre-existing yaml-core API rename (`StepParameter` → `Parameter`) as `ff66aac`.

Design review (light, on decisions) caught 3 important issues that revised the design: D4 moved pool config from `extensionData` to the established `ProvisionerConfigRegistry` path, D5 dropped CDI events (they don't exist in the API) in favor of polling-only, D6 switched from classpath detection to config-driven activation (NoOpAgentRegistry always present).

Filed epic #275 with 6 children (#276–#281) for follow-up hardening.

## Next Work

**Epic #275 — Eidos pool bridge hardening.** Recommended start:

| # | Title | Scale | Cplx | Blocked by |
|---|-------|-------|------|------------|
| #276 | stamp tmux sessions with eidosAgentId | S | Low | — |
| #277 | reconciliation for changed descriptors | S | Med | — |
| #278 | Eidos-driven system prompt injection | M | Med | #276 |
| #279 | ModelRegistry resolution for ModelQuery | S | Med | — |
| #280 | multi-tenant pool isolation | M | Med | — |
| #281 | CDI events for reactive bridge | S | Low | eidos-api |

Start #276 and #277 in parallel (both small, no dependencies), then #278 which depends on #276.

## Pre-existing Issues

- `CaseBrowserServiceTest` and `AgentPoolYamlParserTest` have 1 error each — upstream SNAPSHOT changes, not caused by this session.
- `CaseLineageQueryIntegrationTest` and `ClaudonyLedgerEventCaptureTest` fail to compile — missing `casehub-ledger-core` SNAPSHOT. Pre-existing.
- App module `@QuarkusTest` classes fail with CDI deployment error (unsatisfied `CbrCaseMemoryStore`). Pre-existing.

## Key Files

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridge.java` — startup + reconciliation bean
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolConverter.java` — pure conversion
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/EidosPoolBridgeConfig.java` — defaults record
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java` — `AgentConfig.eidosAgentId` field added
- Config: `claudony.eidos.pool-bridge.enabled=false` (default off)
