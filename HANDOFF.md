# HANDOFF — casehub-claudony

## Last Session

Implemented #247 (standalone fleet script runner) end-to-end: brainstorm → spec → plan → implementation → work-end. Filed follow-up epic #248 with children #249 and #250.

### What was built (#247, landed on main as 8362cf5)

Fleet script runner — parses #246 desiredstate YAML format, topologically sorts nodes by `dependsOn`, provisions pools and channels via per-type SPI handlers. 14 new Java files, 32 tests, REST endpoint at `POST /api/claudony/fleet/execute`.

Core framework in `casehub/fleet/script/`: `FleetScript`, `FleetNode`, `FleetNodeHandler` SPI, `FleetScriptParser` (YAML + variable substitution), `FleetScriptRunner` (Kahn's topo-sort + handler dispatch), `PoolNodeHandler`, `ChannelNodeHandler`. App wiring: `FleetScriptService`, `FleetResource`.

### Follow-up filed

- **#248** (epic) — fleet script lifecycle: startup loading, mcpDomain tool, desiredstate
- **#249** — startup fleet script loading from `META-INF/fleet-scripts/` (XS)
- **#250** — mcpDomain fleet execution tool via `@McpDomain` (XS)
- **#246** — desiredstate integration in ops repo (L, cross-repo)

Recommended next: #249 + #250 together — both XS, ~30 min combined.

### Pre-existing issues

Maven compilation in casehub module fails due to stale SNAPSHOT deps (`StepValidator` from `casehub-yaml-core`, `LedgerPersistenceUnit` from `casehub-ledger`, `panePid`/`respawnPane` from `TmuxService`). New fleet script code compiles cleanly (verified via IntelliJ diagnostics). Tests can't run via Maven until SNAPSHOTs are rebuilt from source.

## References

| Artifact | Path |
|----------|------|
| Design spec | `docs/specs/feat/247-fleet-script-runner/2026-10-03-fleet-script-runner-design.md` |
| Decisions | `docs/specs/feat/247-fleet-script-runner/decisions.md` |
| Plan | `plans/2026-10-03-fleet-script-runner.md` (workspace) |
