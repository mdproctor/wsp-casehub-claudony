# Decisions — #230 Agent Pool Test Utilities

## D1: Module structure

**Choice:** New `claudony-testing` Maven module at `testing/`
**Alternatives:**
- In-module test utility in `casehub/src/test/java` — simpler but doesn't solve downstream consumption
**Rationale:** The purpose is external consumption. Following the `casehub-engine-testing` pattern (separate JAR, `@DefaultBean` alternatives) makes this immediately usable by downstream projects as a test dependency.
**Trade-offs:** Extra module to maintain; no consumer to validate against yet
**Sources:** casehub-engine-testing (TestCaseInstanceRepository pattern), SessionOperations.java, AgentSessionManagerTest.java
**Exploration:** quick
**Status:** captured

## D2: API surface

**Choice:** InMemorySessionOperations + TestPoolBuilder returning TestPool(manager, ops) record
**Alternatives:**
- InMemorySessionOperations only — YAGNI argument, but forces boilerplate on every consumer
**Rationale:** The testing artifact exists to eliminate boilerplate. A builder that wires ops + config + manager and returns both for inspection is the natural API. Every consumer would write the same wiring code otherwise.
**Trade-offs:** Slightly more API surface to maintain; builder pattern may not match all downstream patterns
**Sources:** AgentSessionManagerTest inline mock pattern, AgentSessionManagerConfig constructor
**Exploration:** quick
**Depends on:** D1 (module structure)
**Status:** captured
