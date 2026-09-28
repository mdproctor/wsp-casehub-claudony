# Decisions — #238 Fleet Integration Test

## D1: Mock agent is a Java main class

**Choice:** Tiny `MockAgent.java` in test sources, compiled by Maven, invoked via `java -cp`.
**Alternatives:**
- Shell script — simpler but less realistic; doesn't prove fleet can run a JVM process
**Rationale:** More realistic — closer to what a real agent would be. Proves the fleet can launch, observe, and terminate a Java process in tmux.
**Trade-offs:** Requires computing the classpath at test time to invoke the mock agent.
**Sources:** TmuxSessionOperations.java (command passed to `sh -c`), ClaudonyAgentBackend.fromDefinition()
**Exploration:** quick
**Status:** captured

## D2: Plain JUnit test with real TmuxService

**Choice:** Plain JUnit test (not @QuarkusTest) with real TmuxService and real tmux.
**Alternatives:**
- @QuarkusTest — tests full CDI wiring but adds container startup overhead for no gain here
**Rationale:** Faster, more focused. Tests the pool→backend→tmux chain directly. Follows existing pattern (TmuxAgentSessionTest). No container startup needed.
**Trade-offs:** Doesn't test CDI injection of AgentPoolConfig — but that's trivial wiring, not the thing that could break.
**Sources:** TmuxAgentSessionTest pattern, AgentSessionManagerTest
**Exploration:** quick
**Status:** captured
