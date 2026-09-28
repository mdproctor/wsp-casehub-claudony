# Agent Pool Test Utilities — Design Spec

**Issue:** #230 — agent pool test utilities for downstream projects
**Branch:** issue-230-agent-pool-test-utilities

## Goal

Provide a `claudony-testing` Maven module that downstream projects can depend on (scope: test) to get a working in-memory agent pool without tmux. Follows the `casehub-engine-testing` pattern.

## Module

New Maven module at `testing/`, artifact `io.casehub:casehub-claudony-testing:0.2-SNAPSHOT`. Depends on `casehub-claudony-casehub` (the fleet package). No Quarkus dependency — plain Java + JUnit 5 + AssertJ.

Parent POM gains `<module>testing</module>`. The `casehub` module gains `<dependency>` on `claudony-testing` with `<scope>test</scope>` for validation.

## Classes

### `InMemorySessionOperations`

Implements `SessionOperations`. All operations are in-memory — no tmux, no processes. Thread-safe via `ConcurrentHashMap` and `AtomicInteger` counters.

**Session lifecycle mirrors the tmux-as-persistence model (#239):**
- `create()` → generates session ID (`"mem-" + counter`), stores in active map, generates and stores conversation ID. Returns session ID.
- `suspend()` → moves session from active to suspended map (does NOT remove — session survives suspension).
- `resume()` → moves session from suspended back to active map.
- `destroy()` → removes from whichever map the session is in. Clears conversation ID.
- `conversationId()` → lookup from conversation ID map.
- `memoryBytes()` → returns configurable default (100MB). Settable via `setDefaultMemoryBytes()`.

**Inspection API:**
- `createCount()`, `suspendCount()`, `resumeCount()`, `destroyCount()` — call counters
- `sessions()` → unmodifiable view of all sessions (active + suspended)
- `activeSessions()` → active session IDs
- `suspendedSessions()` → suspended session IDs
- `isActive(String sessionId)` → boolean
- `isSuspended(String sessionId)` → boolean
- `reset()` → clears all state and counters (for `@BeforeEach` use)

**Internal state:**
```java
ConcurrentHashMap<String, SessionRecord> active;    // sessionId → record
ConcurrentHashMap<String, SessionRecord> suspended;  // sessionId → record  
ConcurrentHashMap<String, String> conversationIds;   // sessionId → convId
AtomicInteger createCounter, suspendCounter, resumeCounter, destroyCounter;
```

`SessionRecord` is an internal record: `(String identity, String workingDir)`.

### `TestPool`

```java
public record TestPool(AgentSessionManager manager, InMemorySessionOperations ops) {}
```

Downstream gets the manager to drive and the ops to inspect. No methods beyond accessors.

### `TestPoolBuilder`

Fluent builder with sensible defaults:

```java
TestPoolBuilder.create()            // minActive=0, maxActive=10
    .minActive(2)
    .maxActive(5)
    .build();                       // → TestPool
```

Internally constructs `InMemorySessionOperations` + `AgentSessionManagerConfig` + `AgentSessionManager` and returns them as a `TestPool`.

## Validation

Migrate `AgentSessionManagerTest` in the `casehub` module to use `InMemorySessionOperations` from `claudony-testing` instead of its inline anonymous `SessionOperations` implementation. All 22 existing tests must pass unchanged. The counters and inspection methods replace the `AtomicInteger` fields and `ConcurrentHashMap` currently in the test class.

Some tests that directly construct managers with specific configs will use `TestPoolBuilder`; others that need custom session operations behaviour will use `InMemorySessionOperations` directly.

## Package

`io.casehub.claudony.testing.fleet` — mirrors the production `io.casehub.claudony.casehub.fleet` package but under a `testing` namespace.

## What This Does NOT Include

- No `@DefaultBean` / CDI annotations — downstream wires manually in `@BeforeEach`. CDI integration can be added when a consumer needs it.
- No pool state persistence — that's #239 (tmux-as-persistence model).
- No `TmuxSessionOperations` changes — #239 handles the production suspend/resume fix.

## References

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/SessionOperations.java` — the SPI interface
- `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentSessionManagerTest.java` — inline mock to extract from
- `engine/testing/src/main/java/io/casehub/testing/` — `casehub-engine-testing` pattern (TestCaseInstanceRepository, TestWorkerProvisioner)
- GitHub #239 — pool suspend model fix (tmux-as-persistence)
- GitHub #230 — this issue
