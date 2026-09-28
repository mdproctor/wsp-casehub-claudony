# Design: Fleet Integration Test with Mock Java Agent

**Issue:** #238
**Date:** 2026-09-28

## Goal

Prove the full fleet chain works end-to-end: pool definition → registry → backend → tmux session → observable output → lifecycle management. Uses a mock Java agent instead of Claude CLI.

## Mock Agent

`MockAgent.java` — a tiny `main()` in test sources:

1. Prints `MOCK_AGENT_READY` to stdout
2. Reads stdin in a loop, echoing each line prefixed with `ECHO:`
3. Exits cleanly on EOF or when stdin closes

Accepts and ignores any command-line arguments (the fleet appends `--session-id <uuid>` — the mock agent doesn't need to parse it).

## Test Class

`FleetPoolIntegrationTest` — plain JUnit (not `@QuarkusTest`). Uses real `TmuxService` and real tmux, following the `TmuxAgentSessionTest` pattern.

### Setup

- Construct `TmuxService` directly
- Build the mock agent command: `java -cp <test-classpath> io.casehub.claudony.casehub.fleet.MockAgent`
- Create an `AgentPoolDefinition` via the builder with `command` set to the mock agent command
- Create `ClaudonyAgentBackend` via `fromDefinition()`

### Test Cases

1. **Full chain — YAML → registry → backend → session → output**
   - Parse YAML containing a pool definition with the mock agent command
   - Register in `AgentPoolDefinitionRegistry`
   - Create backend via `fromDefinition()`
   - `acquireSession()` → verify tmux session exists (`tmux has-session`)
   - Capture pane output (`tmux capture-pane`) → verify contains `MOCK_AGENT_READY`
   - `destroySession()` → verify tmux session is gone

2. **Pool capacity and eviction**
   - Create pool with `maxActive=2`
   - Acquire 2 sessions → both active
   - Acquire 3rd → evicts oldest (oldest suspended/destroyed)
   - Verify `poolStatus()` reflects correct counts

3. **Pool status reflects reality**
   - Create pool, acquire session
   - `poolStatus()` shows `active=1`
   - Destroy session
   - `poolStatus()` shows `active=0`

### Teardown

`@AfterEach` destroys all tmux sessions with the test prefix to prevent leaking sessions across test runs.

## What This Proves

- `AgentPoolDefinition` correctly configures the backend command, working directory, and pool capacity
- `AgentSessionManager` creates real tmux sessions that run real processes
- Pool eviction works when capacity is exceeded
- The fleet status API reflects actual tmux state
- The mock agent pattern is available for future tests that need a running agent without Claude CLI

## What This Does NOT Prove (future work)

- That a real LLM processes prompts (requires an actual model)
- That MCP tool calls work through the fleet (requires MCP server wiring)
- CDI injection wiring in a running Quarkus instance (separate concern, trivial)

## References

- `ClaudonyAgentBackend.java:45` — `fromDefinition()` factory method
- `AgentSessionManager.java` — capacity-bounded session pool
- `TmuxSessionOperations.java:31` — `create()` appends `--session-id` to command
- `TmuxAgentSessionTest` — existing pattern for real-tmux tests
- `AgentPoolDefinition.java` — pool definition model
