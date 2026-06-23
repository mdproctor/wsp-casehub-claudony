# Handoff — 2026-06-24

**Head commit (project):** `1d43b14` — feat(#105): separate Claudony and Qhorus MCP endpoints

## What landed this session

### claudony#105 — Separate MCP endpoints (Phase A groundwork)

Split unified `/mcp` endpoint into two named MCP servers: Claudony session tools (8) at `/mcp`, Qhorus agent mesh tools at `/qhorus`. Removed `page-size=0` workaround. Qhorus upstream change (`@McpServer("qhorus")` + default config via `microprofile-config.properties`) committed locally as `6407ee2`.

Three-round spec review (11 issues resolved): pagination per-server, session scoping, ToolErrorHandlingTest callout, isolation assertions, ARC42STORIES enumeration, CLAUDE.md rewrite, dual-port table.

Platform protocol `PP-20260623-105mcp` captured in garden. 6 cross-repo issues filed.

CI: 587 tests green, pushed to casehubio/claudony main.

## State

- main: `1d43b14`
- All 587 tests pass locally
- #105 closed, branch stamped
- Qhorus commit `6407ee2` on local main — needs push to casehubio/qhorus

## Cross-repo issues filed

| Repo | Issue | Status |
|------|-------|--------|
| casehubio/qhorus | #306 | Code committed locally, needs push |
| casehubio/parent | #308 | PLATFORM.md named-server convention |
| casehubio/devtown | #93 | Add second mcpServers entry |
| casehubio/openclaw | #47 | Add second mcpServers entry |
| casehubio/drafthouse | #77 | Verify MCP client config |
| casehubio/life | #42 | Verify MCP client config |

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#306 | Push @McpServer commit to casehubio/qhorus | XS | Low | Already committed locally |
| parent#308 | Update PLATFORM.md Capability Ownership | XS | Low | Add named-server convention |
| #157 | Migrate Worker imports to casehub-worker-api | S | Low | Refactor only |
