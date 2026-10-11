# HANDOFF — casehub-claudony

## Last Session

Implemented #269 (application registration + namespace model — Stage 3a) end-to-end: brainstorm → spec → plan → implementation → work-end.

### What was built (#269, landed on main as 3053a96)

Application as a first-class registry entity with namespace-based isolation. 6 new Java files, 17 new tests, @McpDomain API at `/api/claudony/apps`.

- `ApplicationRegistrationService` orchestrates registration across `RegistryService` and `AgentPoolDefinitionRegistry`
- `ApplicationSpec`/`ApplicationInfo` records for registration request/response
- `ClaudonyAppApi` @McpDomain — REST + MCP + GraphQL at `claudony/apps`
- Namespace parameter on `AgentPoolDefinitionRegistry` (backward-compatible, defaults to "fleet")
- `app:` channel prefix convention in `docs/CHANNELS.md`
- Design spec + 6 decisions at `docs/specs/issue-269-application-registration-namespace/`

### Key design decisions

- D1: `namespace` on `RegistryEntry` = owning app identity (not coarse category)
- D2: Namespace for discovery queries, relationships for lifecycle cascade
- D3: Apps declare pools at registration time (declarative, not negotiated)

## Next

**claudony#270** — generic registry @McpDomain (`ClaudonyRegistryApi`). Issue updated to use @McpDomain, not REST-only.

**platform#570** — registry-client. Dependency direction needs resolution: platform is upstream of claudony, so the contract definition may belong in platform-api.

## Pre-existing Issues

- casehub module has compilation errors from missing upstream SNAPSHOTs (`StepValidator` from yaml-core, `panePid`/`respawnPane` from TmuxService). Not caused by #269. IntelliJ diagnostics on changed files pass clean.
