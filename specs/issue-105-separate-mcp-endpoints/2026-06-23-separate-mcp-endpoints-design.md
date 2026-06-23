# Separate Claudony and Qhorus MCP Endpoints

**Issue:** claudony#105
**Date:** 2026-06-23
**Status:** Design

## Problem

Claudony serves 8 session-management tools and 54 Qhorus agent-mesh tools on a single `/mcp` endpoint. This causes:

1. **Silent tool loss.** `quarkus-mcp-server` paginates `tools/list` at 50 by default per server. With 62 tools, anything alphabetically past position 50 was dropped — including `send_input`. Current workaround: `quarkus.mcp.server.tools.page-size=0`.
2. **Fragile integration tests.** Claudony's `McpServerIntegrationTest` asserts exact tool counts that break every time Qhorus adds a tool (GE-20260604-8b199c).
3. **Audience mismatch.** Session tools are operator-facing; mesh tools are agent-facing (PLATFORM.md line 400). Mixing them forces every client to receive both.
4. **Phase A blockage.** Non-Claudony Claudes can't connect to Qhorus without also receiving session management tools they don't need.

## Phase A Lineage

This issue is Phase A groundwork from the ecosystem design spec (`docs/specs/2026-04-13-quarkus-ai-ecosystem-design.md`). The spec describes Phase A as "casehub-mcp extracted to the CaseHub project, Qhorus runs standalone, Claudony-managed Claudes get the enhanced experience." This issue establishes the platform convention that every library MCP surface gets its own named server — a prerequisite for Phase A's eventual process extraction of both Qhorus and casehub-mcp into standalone deployments.

## Design

### Mechanism: Named MCP Servers

`quarkus-mcp-server` 1.11.1 supports named server instances via `@McpServer("name")` annotation + per-server `http.root-path` configuration. Each named server gets its own HTTP endpoint, tool list, session management, and server-info metadata.

### Principle: Libraries Scope, Applications Default

The default MCP server belongs to the application. Libraries that expose MCP tools must declare a named server via `@McpServer`. This prevents tool-list pollution and pagination collisions when multiple libraries are embedded.

This mirrors existing platform conventions: named datasources (`quarkus.datasource.qhorus.*`), named Flyway (`quarkus.flyway.qhorus.*`), named MCP servers (`quarkus.mcp.server.qhorus.*`).

`ClaudonyMcpTools` remains **unannotated** — no `@McpServer`. It uses the default server by design. The application owns the default server; libraries scope to named servers. The absence of the annotation is the design decision.

### After State

| Server | Path | Tools | Audience |
|--------|------|-------|----------|
| `<default>` | `/mcp` | 8 Claudony session tools | Operator / controller Claude |
| `qhorus` | `/qhorus` | 54 Qhorus agent mesh tools | AI agent workers |

Both Server mode (port 7777) and Agent mode (port 7778) serve both endpoints — same Quarkus binary, same HTTP handler registration:

| Mode | Claudony tools | Qhorus tools |
|------|---------------|-------------|
| Server (7777) | `http://localhost:7777/mcp` | `http://localhost:7777/qhorus` |
| Agent (7778) | `http://localhost:7778/mcp` | `http://localhost:7778/qhorus` |

A controller Claude connecting to the Agent needs both URLs:
- `http://localhost:7778/mcp` — session management
- `http://localhost:7778/qhorus` — agent mesh

A non-Claudony Claude connects to Qhorus directly at `/qhorus` (or wherever the standalone deployment configures it).

### Why the Annotation Goes in Qhorus, Not Claudony

- Qhorus owns its MCP surface. The tools are defined in `QhorusMcpTools` / `ReactiveQhorusMcpTools`.
- The `@McpServer` annotation is compile-time (Jandex). It must be on the declaring class.
- A Claudony-side wrapper or annotation transformer would create fragile coupling to Qhorus internals.
- The pattern scales: when `casehub-connectors-mcp` or `casehub-mcp` embed, they use `@McpServer("connectors")` / `@McpServer("casehub")` respectively.

### Default Config via Library JAR

Without explicit config, `invalidServerNameStrategy` (default: `FAIL`) causes a startup failure when `@McpServer("qhorus")` names a server with no corresponding config entry. Qhorus must ship defaults.

`tools.page-size` is per-server runtime config (`McpServerRuntimeConfig.Tools.pageSize()`, `@WithDefault("50")`). Qhorus has 54 tools today and will grow — the library must disable pagination on its own server. This is Qhorus's responsibility; consumers should not need to know how many tools Qhorus has.

Qhorus ships defaults in `META-INF/microprofile-config.properties`:

```properties
quarkus.mcp.server.qhorus.http.root-path=/qhorus
quarkus.mcp.server.qhorus.server-info.name=qhorus
quarkus.mcp.server.qhorus.tools.page-size=0
```

Quarkus reads `microprofile-config.properties` from classpath JARs at build time (ordinal 100, below application.properties at 250). The `@WithDefaults` annotation on the server config map populates entries from all config sources. Consumers override by setting the property in their own `application.properties`.

### Interaction with Reactive Build Property

Qhorus gates its tool classes via `@UnlessBuildProperty`:
- `QhorusMcpTools` — active when `casehub.qhorus.reactive.enabled` is absent or false
- `ReactiveQhorusMcpTools` — active when `casehub.qhorus.reactive.enabled=true`

`@McpServer("qhorus")` goes on BOTH classes. Only the active class is in the CDI bean stream; the extension discovers tools from CDI beans only, so the inactive class's annotation is harmless.

Claudony sets `casehub.qhorus.reactive.enabled=true` → `ReactiveQhorusMcpTools` is the active class.

## Changes

### Qhorus (upstream — new issue)

1. Add `@McpServer("qhorus")` to `QhorusMcpTools` (class-level)
2. Add `@McpServer("qhorus")` to `ReactiveQhorusMcpTools` (class-level)
3. Create `runtime/src/main/resources/META-INF/microprofile-config.properties`:
   ```properties
   quarkus.mcp.server.qhorus.http.root-path=/qhorus
   quarkus.mcp.server.qhorus.server-info.name=qhorus
   quarkus.mcp.server.qhorus.tools.page-size=0
   ```
4. Update `ToolErrorHandlingTest.java` — 3 `.post("/mcp")` calls (lines 80, 103, 126) → `.post("/qhorus")`
5. Add `import io.quarkiverse.mcp.server.McpServer` (same package as existing `@Tool` import)
6. Publish SNAPSHOT for consumers to pick up

### Claudony (this issue, #105)

1. Remove `quarkus.mcp.server.tools.page-size=0` from `application.properties`
2. Update `McpServerIntegrationTest`:
   - `fullHandshakeSequence_asClaudeWouldSendIt` — change `.body("result.tools.size()", greaterThanOrEqualTo(8))` to `.body("result.tools.size()", equalTo(8))` and assert all 8 by name. Without this tightening, the test doesn't verify the split actually happened.
   - `toolsList_includesQhorusTools` — rename to `qhorusToolsAvailableAtSeparateEndpoint`; target `/qhorus`; assert key tools present (`register`, `send_message`, `check_messages`, `create_channel`); use `greaterThanOrEqualTo(40)` for count — no hardcoded exact number
   - Remove the stale `// Phase 8` comment block at end of file
3. Update CLAUDE.md:
   - Key URLs: add both `http://localhost:7777/qhorus` (Server) and `http://localhost:7778/qhorus` (Agent)
   - Test count: update tool-count assertion documentation
   - Configuration: document `quarkus.mcp.server.qhorus.*` override capability
4. Update ARC42STORIES.MD — the following sections reference the unified `/mcp` endpoint:
   - §6 Scenario 2 (line 256): `Agent → POST /mcp → ClaudonyMcpTools → QhorusMcpTools.sendMessage()` — after split, Qhorus tools are at `/qhorus`
   - L4 Layer entry (line 578): `"Qhorus tools join existing /mcp endpoint"` — no longer true; Qhorus has its own `/qhorus` endpoint
   - C5 Chapter: `"Qhorus tools join the same MCP endpoint as C4's session tools"` — update to reflect separate endpoints
   - §4 Solution Strategy (line 550): `"8 session management tools"` at `/mcp` — correct after split, but should note Qhorus at `/qhorus`

### Cross-Repo Propagation (deferred issues)

Each Qhorus consumer that exposes MCP endpoints needs its MCP client configs updated to reference `/qhorus` for mesh tools. File issues in:

| Repo | Issue scope |
|------|------------|
| `casehubio/qhorus` | Add `@McpServer("qhorus")` + default config + `tools.page-size=0` |
| `casehubio/devtown` | Update MCP client config for `/qhorus` |
| `casehubio/openclaw` | Update MCP client config for `/qhorus` |
| `casehubio/casehub-drafthouse` | Update MCP client config for `/qhorus` |
| `casehubio/casehub-life` | Update MCP client config for `/qhorus` |
| `casehubio/parent` | PLATFORM.md: add named-server convention to Capability Ownership |

### Platform Protocol

Capture as a new protocol in `casehubio/garden`:

> **Library MCP tools must use `@McpServer` with a named server.** The default MCP server belongs to the application. Libraries that expose `@Tool` methods must annotate their tool classes with `@McpServer("<library-name>")` and ship default config via `META-INF/microprofile-config.properties` — including `http.root-path`, `server-info.name`, and `tools.page-size=0` (if tool count exceeds or may grow past 50). This prevents tool-list pollution, pagination collisions, and audience mismatch when multiple libraries are embedded in the same Quarkus application.

## Test Strategy

### Claudony-side assertions (exact — Claudony controls these)

```java
// At /mcp — exactly 8 Claudony tools, no Qhorus tools
.body("result.tools.size()", equalTo(8))
.body("result.tools.name", hasItems(
    "list_sessions", "create_session", "delete_session",
    "rename_session", "send_input", "get_output",
    "open_in_terminal", "get_server_info"))
.body("result.tools.name", not(hasItems("register", "send_message", "check_messages")))
```

### Qhorus integration assertions (flexible — Qhorus controls these)

```java
// At /qhorus — key tools present, no Claudony tools, no hardcoded count
.body("result.tools.name", hasItems(
    "register", "send_message", "check_messages",
    "create_channel", "list_channels"))
.body("result.tools.size()", greaterThanOrEqualTo(40))
.body("result.tools.name", not(hasItems("list_sessions", "create_session")))
```

This directly addresses GE-20260604-8b199c: the Claudony-side count is stable (Claudony owns it); the Qhorus-side count is flexible (Qhorus evolves independently). The negative assertions verify the split — tools must not leak across endpoints.

## Sequencing

1. Create Qhorus issue → implement `@McpServer("qhorus")` + default config → publish SNAPSHOT
2. Create propagation issues in consumer repos
3. Claudony pulls new Qhorus SNAPSHOT → implement #105 changes → verify tests
4. File protocol in `casehubio/garden`
5. Update PLATFORM.md Capability Ownership

## Not In Scope

- **Qhorus standalone deployment** — the default config enables it; verifying it is a Qhorus concern
- **`casehub-mcp` extraction** — Phase A CaseHub tool extraction is a separate future effort
- **Per-server auth** — separate servers enable independent auth config, but wiring it is out of scope
- **Process extraction** — running Qhorus as a separate process is Phase A end-state, not this issue

## References

- Ecosystem design spec: `docs/specs/2026-04-13-quarkus-ai-ecosystem-design.md` §MCP Surfaces, §Deployment Topology (Phase A)
- Garden: GE-20260604-8b199c (hardcoded tool-count assertion fragility)
- Garden: GE-20260430-b015f5 (quarkus-mcp-server @Tool overload drop)
- Protocol: PP-20260604-c0a86d (MCP tool exception catch-all)
- Protocol: PP-20260604-995096 (reactive @Tool resolveChannel @Blocking)
- PLATFORM.md line 386: "Qhorus MCP tool surface"
- PLATFORM.md line 400: "claudony's MCP surface is operator-facing; Qhorus MCP tools are agent-facing"
- `McpServerRuntimeConfig.Tools.pageSize()` — `@WithDefault("50")`, per-server runtime config
- `McpServersRuntimeConfig.invalidServerNameStrategy()` — `@WithDefault("fail")`, startup failure without config
