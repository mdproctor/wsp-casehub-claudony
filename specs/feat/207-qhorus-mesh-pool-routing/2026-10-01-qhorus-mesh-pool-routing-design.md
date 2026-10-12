# Qhorus Mesh Routing Integration for Agent Pools

**Issue:** casehubio/claudony#207
**Date:** 2026-10-01
**Status:** Approved

## Problem

Agent pools (`AgentSessionManager`) manage fleet LLM instances with full lifecycle
(acquire, suspend, resume, destroy), but these sessions are invisible to the Qhorus
agent mesh. They cannot be discovered via `listInstances`, routed to via capability
matching, or observed through the mesh dashboard. This creates a gap: the fleet
management layer and the communication layer don't know about each other.

## Solution

Bridge pool session lifecycle events into Qhorus instance registration via a CDI
event observer pattern. When a pool acquires a session, it becomes a Qhorus instance;
when suspended, it goes offline; when resumed, back online; when destroyed, deregistered.

## Design

### Component 1: `SessionLifecycleListener` (interface, `claudony-casehub`)

Callback interface for pool session lifecycle events. Follows the established SPI
callback pattern (`WorkerStatusListener`, `SessionOperations`).

```java
public interface SessionLifecycleListener {
    void onAcquired(ManagedSession session, String poolName);
    void onSuspended(ManagedSession session, String poolName);
    void onResumed(ManagedSession session, String poolName);
    void onDestroyed(String sessionId, String poolName);
}
```

A no-op default implementation (`SessionLifecycleListener.NOOP`) is provided for
pools that don't need mesh integration.

### Component 2: `AgentSessionManager` listener integration

`AgentSessionManager` accepts an optional `SessionLifecycleListener` in the constructor.
It calls the listener at each lifecycle transition point:

| Method | Listener call |
|--------|--------------|
| `acquireSession()` | `listener.onAcquired(session, poolName)` — after successful create/resume |
| `suspendSession()` | `listener.onSuspended(session, poolName)` — after state change |
| `resumeSession()` | `listener.onResumed(session, poolName)` — after successful resume |
| `destroySession()` | `listener.onDestroyed(sessionId, poolName)` — after successful destroy |
| `shutdown()` | `listener.onDestroyed(sessionId, poolName)` — for each active session |

The pool name is passed on each call so the listener knows which pool the session
belongs to. `AgentSessionManager` needs to store its pool name — currently it doesn't.
Add a `poolName` field set via a new constructor parameter or builder.

### Component 3: `PoolMeshRegistrar` (`@ApplicationScoped`, `claudony-app`)

Implements `SessionLifecycleListener`. Bridges pool lifecycle into Qhorus:

| Event | Qhorus action |
|-------|--------------|
| `onAcquired` | `instanceService.register(sessionId, "pool:{poolName}/{identity}", ["pool:{poolName}"], sessionId)` |
| `onSuspended` | `instanceService.markOffline(sessionId)` |
| `onResumed` | `instanceService.register(...)` — re-register with status online |
| `onDestroyed` | `instanceService.deregister(sessionId)` |

The `claudonySessionId` field on `Instance` is set to the managed session's
`instanceId`, enabling correlation between pool sessions and mesh instances.

### Capability mapping

Pool sessions register with capability tag `pool:{poolName}`:
- Pool "code-reviewer" → capability `["pool:code-reviewer"]`
- Pool "test-runner" → capability `["pool:test-runner"]`

The `pool:` prefix distinguishes pool-managed instances from self-registered ones.
Routing queries use `findByCapability("pool:code-reviewer")` to locate pool instances.

Instance description follows format `pool:{poolName}/{identity}` for human readability
in dashboard listings.

### Wiring

In `claudony-app`, wherever `AgentPoolManagerRegistry` constructs `AgentSessionManager`
instances, inject `PoolMeshRegistrar` as the lifecycle listener. The pool definition
registry and manager registry already exist — the change is passing the listener
through during pool construction.

## Integration tests

Extend the `FleetPoolIntegrationTest` pattern: real tmux sessions + `InMemoryInstanceStore`
+ `InstanceService`. Tests verify the full chain from pool lifecycle to Qhorus visibility.

### Test cases

1. **acquire_registersAsQhorusInstance** — acquire a pool session, verify
   `instanceService.findByInstanceId()` returns an online instance with correct
   capability tag and description.

2. **suspend_marksInstanceOffline** — acquire then suspend, verify instance status
   is "offline".

3. **resume_marksInstanceOnline** — acquire, suspend, resume, verify instance back
   to "online".

4. **destroy_deregistersInstance** — acquire then destroy, verify
   `findByInstanceId()` returns empty.

5. **fullLifecycle_cleanState** — acquire → suspend → resume → destroy, verify no
   orphaned instances remain.

6. **capabilityRouting_findsPoolInstances** — acquire two sessions in different pools,
   verify `findByCapability("pool:X")` returns only the correct pool's instances.

7. **multipleSessionsInPool_allRegistered** — acquire multiple sessions from same pool,
   verify all appear as instances with same capability tag.

### Test structure

Tests live in `claudony-casehub/src/test/java` alongside `FleetPoolIntegrationTest`.
They use real tmux (via `TmuxSessionOperations`) and `InMemoryInstanceStore` (from
`casehub-qhorus-testing`). No `@QuarkusTest` needed — plain JUnit with manual wiring,
same as the existing pool integration tests.

## What doesn't change

- Qhorus `InstanceService` API — no changes needed, it already supports everything
- Self-registered instances (via MCP `register` tool) — unchanged, no conflict
- The mesh dashboard UI — pool instances appear automatically via `listInstances()`
- `PoolService` and `ClaudonyPoolApi` — pool management API unchanged
- `ClaudonyMeshApi` — mesh API unchanged, already delegates to `QhorusDashboardService`

## Module changes summary

| Module | Change |
|--------|--------|
| `claudony-casehub` | `SessionLifecycleListener` interface |
| `claudony-casehub` | `AgentSessionManager` — add `poolName` field, accept listener, call at lifecycle points |
| `claudony-casehub` | `AgentPoolManagerRegistry` — accept listener for pool construction |
| `claudony-app` | `PoolMeshRegistrar` — `@ApplicationScoped` listener impl |
| `claudony-app` | Pool construction wiring — pass registrar as listener |
| `claudony-casehub/test` | Integration tests (7 test cases) |

## References

- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentSessionManager.java` — pool session manager
- `casehub/src/main/java/io/casehub/claudony/casehub/fleet/SessionOperations.java` — SPI pattern reference
- `qhorus/runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java` — Qhorus registration
- `qhorus/api/src/main/java/io/casehub/qhorus/api/instance/Instance.java` — instance model
- `casehub/src/test/java/io/casehub/claudony/casehub/fleet/FleetPoolIntegrationTest.java` — existing test pattern
- `app/src/main/java/io/casehub/claudony/server/api/ClaudonyMeshApi.java` — mesh API (unchanged)
