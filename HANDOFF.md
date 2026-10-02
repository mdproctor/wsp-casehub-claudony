# HANDOFF — casehub-claudony

## Last Session

Design and planning session for declarative fleet deployment. No implementation code — filed issues and fixed a build break.

### Build fix (landed on main)

`@HandWrittenEndpoint` annotation on 7 hand-written `@Path` resources — required by the `@McpDomain` annotation processor from #204. Removed dead `QhorusMcpTools` import/injection from 2 E2E tests (class deleted in qhorus #452). Needed `platform-api` rebuilt from source to get the annotation class. Commit `00db804`.

### Fleet manager state assessment

Reviewed what #205 delivered vs what's missing. The fleet manager is substantially complete:
- `ClaudonyPoolApi` via `@McpDomain` — 7 operations (list, detail, sessions, update, suspend, resume, destroy)
- `AgentPoolDefinition` + `AgentPoolYamlParser` — declarative pool definitions from YAML
- `AgentSessionManager` — suspend/resume lifecycle with memory-weighted eviction
- Auto-scaling (5 policy types), observability (Micrometer + IoTDB), SSE event streaming, interactive dashboard

**Gap identified:** no declarative path for deploying a complete fleet (agents + pools + mesh) from a single YAML script. Pools are YAML-driven but agents and channels are provisioned separately.

### Issues filed

- **#246** — declarative LLM fleet deployment with desiredstate reconciliation + ops provisioning. Three execution modes: ad-hoc (exists), standalone script (#247), desiredstate nodes (ops). Includes full fleet YAML example with agents, pools, and mesh channels. Cross-repo: ops (PoolNodeSpec + provisioner), claudony (script runner), platform (manifest), eidos (binding).

- **#247** — standalone fleet script runner. Claudony-local, no ops dependency. `FleetScriptRunner` parses the same YAML as desiredstate, topo-sorts by `dependsOn`, provisions pools + channels in order. E2E testable in this repo. Ready to start now.

### Slot 202

Created for #205 work. All #205 implementation landed on main. Slot 202 workspace has the design artifacts (spec, decisions, plan). The slot branch is closed (`chore: branch closed`).

## Immediate Next Step

Start #247 (standalone fleet script runner). All building blocks exist:
- `AgentPoolYamlParser` — pool YAML parsing
- `AgentPoolDefinitionRegistry` — pool registration
- `ChannelService` — Qhorus channel CRUD (embedded)
- `ClaudonyPoolApi` — pool management via MCP

New pieces: `FleetScriptRunner`, `FleetNodeHandler` SPI (per-type handlers), variable substitution, e2e tests.

## CI Status

**Build and Publish** was red on main — the `@HandWrittenEndpoint` fix resolved the Java compilation error. A separate TypeScript error in `site.ts` (casehub-pages dependency, `LayoutState` type mismatch with `exactOptionalPropertyTypes`) persists — needs fixing in the pages repo, not claudony.

## References

- #246 — declarative fleet deployment (desiredstate + ops)
- #247 — standalone fleet script runner (claudony, ready to start)
- #205 — fleet manager (complete, landed on main)
- Build flags for app tests: `-Denforcer.skip=true -Dquinoa.build.skip=true`
