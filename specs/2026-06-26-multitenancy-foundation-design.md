# Multi-Tenancy Foundation — Design Spec

**Issue:** casehubio/claudony#121
**Branch:** issue-121-multitenancy-foundation
**Date:** 2026-06-26 (revised 2026-06-27, v3)

---

## Summary

Add unconditional tenancyId enforcement to Claudony's in-memory stores, events, and cache keys. Every Session carries a tenancyId, every registry query filters by it, and every CDI event propagates it. Follows protocols PP-20260520-439daf (unconditional filtering) and PP-20260520-e6a5f0 (filtering inside data access only).

---

## Approach: Thin Compliance + Forward Contracts

Claudony's primary state is ephemeral — in-memory `ConcurrentHashMap` stores rebuilt on restart. The only durable tenant-aware state is `CaseLedgerEntry` (already implemented). The issue as filed assumes JPA entities and Repository pattern, but Claudony has no entity tables of its own.

The approach adds tenancyId to the data model, filters unconditionally in the registry, and wires `TenantContext` with a `CurrentPrincipal` delegation pattern that makes the OIDC migration (platform#16) a zero-touch event for stores and endpoints.

**What this is not:** full multi-tenant auth. There is no OIDC `CurrentPrincipal` implementation yet (platform#16). `TenantContext` delegates to `CurrentPrincipal` when request scope is active, otherwise falls back to `DEFAULT_TENANT_ID` — filtering runs unconditionally but is a no-op in single-tenant deployments. When OIDC lands, `CurrentPrincipal.tenancyId()` starts returning real values and TenantContext picks them up automatically — zero changes to stores or endpoints.

---

## Decisions Made

### claudony-core depends on casehub-platform-api
`SessionRegistry` needs `TenantContext`, which needs `TenancyConstants.DEFAULT_TENANT_ID` and optionally `CurrentPrincipal` for the delegation pattern. `casehub-platform-api` is the bottom tier of the platform stack — a thin API module with no runtime coupling, no CDI beans, no Quarkus extensions. It contains identity primitives (`CurrentPrincipal`, `TenancyConstants`) and endpoint registry SPIs.

The ARC42STORIES L1 constraint ("clean of CaseHub/Qhorus concepts") refers to engine orchestration types and Qhorus messaging, not platform identity primitives. Multi-tenancy is a cross-cutting infrastructure concern. `casehub-platform-api` is already in `dependencyManagement` at the parent level and `claudony-app` depends on it directly. Adding it to `claudony-core` is consistent layering.

This eliminates the UUID constant duplication and enables `TenantContext` to reference `TenancyConstants.DEFAULT_TENANT_ID` and `CurrentPrincipal` directly from a single source of truth.

### TenantContext is @ApplicationScoped with CurrentPrincipal delegation
`TenantContext` is not a `@DefaultBean` to be swapped later. It is the single `@ApplicationScoped` implementation that internally delegates to `CurrentPrincipal` when request scope is active:

```java
@ApplicationScoped
public class DefaultTenantContext implements TenantContext {
    @Inject Instance<CurrentPrincipal> principal;

    @Override
    public String currentTenantId() {
        if (principal.isResolvable()
                && Arc.container().requestContext().isActive()) {
            return principal.get().tenancyId();
        }
        return TenancyConstants.DEFAULT_TENANT_ID;
    }
}
```

**Why not `@DefaultBean` + future `@RequestScoped` swap:** `SessionRegistry` is `@ApplicationScoped`. It injects `TenantContext` and calls `currentTenantId()` on every query. A `@RequestScoped` TenantContext would throw `ContextNotActiveException` when called from any of the six+ system contexts that lack request scope (virtual thread watchers, `@Scheduled` expiry, startup observers, engine CDI observers, WebSocket handlers, `@ObservesAsync` threads). The `@ApplicationScoped` delegation pattern avoids this entirely — consistent with how Qhorus handles it (`QhorusSystemCurrentPrincipal`).

**Why `Arc.container().requestContext().isActive()` instead of try/catch:** Exception-as-control-flow is slow and noisy in logs. The explicit scope check is a clean conditional.

**CDI resolution (verified):** `Instance<CurrentPrincipal>` resolves unambiguously to `QhorusInboundCurrentPrincipal` (`@ApplicationScoped`, unqualified — displaces `MockCurrentPrincipal` `@DefaultBean`). `QhorusSystemCurrentPrincipal` is `@QhorusSystem`-qualified and won't match. `QhorusInboundCurrentPrincipal.tenancyId()` has its own `ContextNotActiveException` → `DEFAULT_TENANT_ID` fallback, but `DefaultTenantContext`'s scope check is defensive — it does not rely on the delegate's internal behavior.

### Session.tenancyId is non-optional, last field
`String tenancyId` (not `Optional`) — tenancy is never absent. Positioned last in the record to force compile errors at every construction site, ensuring each explicitly provides a tenancyId source.

### Write operations stay unscoped
`register()`, `remove()`, `updateStatus()`, `touch()` operate on known session IDs. Callers already have a valid session reference (from a filtered lookup or from system code). Filtering writes would prevent the expiry scheduler from cleaning up other tenants' sessions.

### allUnscoped() and findUnscoped() are controlled escape hatches
System operations (expiry, bootstrap, fleet sync, engine observers, watcher threads) need unscoped access. Named explicitly so they stand out in code review. These are not "pass a different tenant" methods — they are separate operations for system code that must work across tenants.

### SessionRegistry uses constructor injection
`SessionRegistry` is tested in plain JUnit (`new SessionRegistry()`). Field injection would leave `tenantContext` null and NPE on every filtered query. Constructor injection forces explicit dependency provision:

```java
@ApplicationScoped
public class SessionRegistry {
    private final TenantContext tenantContext;

    @Inject
    public SessionRegistry(TenantContext tenantContext) {
        this.tenantContext = tenantContext;
    }
}
```

Existing `SessionRegistryTest` adapts to:

```java
@BeforeEach
void setUp() { registry = new SessionRegistry(() -> TenancyConstants.DEFAULT_TENANT_ID); }
```

`SessionRegistryTenantFilterTest` uses `MutableTenantContext` as a plain POJO — no CDI needed:

```java
var tenantCtx = new MutableTenantContext();
var registry = new SessionRegistry(tenantCtx);
tenantCtx.setTenantId(TENANT_B);
// assertions against filtered methods
```

### existsByName() enforces the tmux namespace invariant
tmux session names are a global OS-level namespace — no tenant concept. Duplicate name checks must be unscoped regardless of tenant filtering. `existsByName(String name)` is always unscoped and named for its purpose (resource-namespace uniqueness, not data access).

### Fleet stays unscoped
Fleet nodes are same-owner by design. Fleet-key auth (principal = "peer", role = "fleet") is an infrastructure concern, not a tenant concern. Future fleet tenant isolation is a separate issue.

### Events carry tenancyId for forward compatibility
`WorkerCaseLifecycleEvent` and `CaseChannelCreatedEvent` gain `tenancyId`. Observers don't filter by it yet — the event field establishes the contract for future tenant-scoped broadcasting.

### Bootstrap reads tenancyId from tmux, falls back to DEFAULT
Sessions recovered from `tmux list-sessions` on restart read `@casehub_tenant_id` from tmux session options (set during provisioning). If absent, fall back to `TenancyConstants.DEFAULT_TENANT_ID`. `CasehubStartupService.bootstrapWatchers()` uses `allUnscoped()` because it must recover watchers across all tenants.

---

## New Types

### TenantContext — claudony-core

```java
package io.casehub.claudony.server;

import io.casehub.platform.api.identity.TenancyConstants;

public interface TenantContext {
    String currentTenantId();
}
```

No `DEFAULT_TENANT_ID` constant on the interface — use `TenancyConstants.DEFAULT_TENANT_ID` directly (single source of truth from `casehub-platform-api`).

### DefaultTenantContext — claudony-core

```java
@ApplicationScoped
public class DefaultTenantContext implements TenantContext {
    @Inject Instance<CurrentPrincipal> principal;

    @Override
    public String currentTenantId() {
        if (principal.isResolvable()
                && Arc.container().requestContext().isActive()) {
            return principal.get().tenancyId();
        }
        return TenancyConstants.DEFAULT_TENANT_ID;
    }
}
```

`@ApplicationScoped` — safe in all contexts (request, background, startup, virtual thread). Delegates to `CurrentPrincipal` when request scope is active; falls back to `DEFAULT_TENANT_ID` otherwise. Not `@DefaultBean` — this is the permanent implementation.

---

## Data Model Changes

### Session record — claudony-core

Add `tenancyId` as last field:

```java
public record Session(
        String id, String name, String workingDir, String command,
        SessionStatus status, Instant createdAt, Instant lastActive,
        Optional<String> expiryPolicy, Optional<String> caseId,
        Optional<String> roleName, String tenancyId) {
```

`withStatus()` and `withLastActive()` carry `tenancyId` through the copy.

**tenancyId source by creation site:**

| Creation site | Source |
|---|---|
| `ServerStartup.bootstrapRegistry()` | Read from tmux `@casehub_tenant_id` option; fall back to `TenancyConstants.DEFAULT_TENANT_ID` |
| `ClaudonyReactiveWorkerProvisioner.setupSession()` | `context.tenancyId()` from `ProvisionContext` |
| `SessionResource.create()` | `tenantContext.currentTenantId()` (injected) |
| `SessionResource.rename()` | Carry from existing session |

### CDI Events — claudony-core

```java
public record WorkerCaseLifecycleEvent(String caseId, String tenancyId) {}
public record CaseChannelCreatedEvent(UUID channelId, String channelName, String tenancyId) {}
```

`SessionExpiredEvent` unchanged — `Session` already carries `tenancyId`.

**tenancyId source at fire sites:**

| Event | Fire site | Source |
|---|---|---|
| `WorkerCaseLifecycleEvent` | `ClaudonyWorkerStatusListener.onWorkerStarted()` | Session lookup via `sessionMapping` → `registry.findUnscoped()` → `session.tenancyId()`; fallback `TenancyConstants.DEFAULT_TENANT_ID` if session not found |
| `WorkerCaseLifecycleEvent` | `ClaudonyWorkerStatusListener.onWorkerCompleted()` | Same pattern — `registry.findUnscoped()` → `session.tenancyId()`; fallback `TenancyConstants.DEFAULT_TENANT_ID` |
| `WorkerCaseLifecycleEvent` | `ClaudonyWorkerStatusListener.onWorkerStalled()` | Same pattern via `registry.findUnscoped()` |
| `CaseChannelCreatedEvent` | `ClaudonyReactiveCaseChannelProvider.createQhorusChannel()` | `tenantContext.currentTenantId()` (injected) |

`ClaudonyWorkerStatusListener` injects `TenantContext` for the fallback case. All three methods (`onWorkerStarted`, `onWorkerCompleted`, `onWorkerStalled`) use `findUnscoped()` for session reads — they are called by engine CDI observers in system context (no request scope).

---

## SessionRegistry Changes — claudony-core

`SessionRegistry` takes `TenantContext` via constructor injection. Public query methods filter unconditionally.

**Filtered (tenant-scoped):**
- `all()` — returns only sessions matching `tenantContext.currentTenantId()`
- `find(id)` — returns `Optional.empty()` if session belongs to different tenant
- `findByCaseId(caseId)` — filters by tenant on top of caseId filter

**Unscoped (system operations):**
- `allUnscoped()` — returns all sessions regardless of tenant
- `findUnscoped(id)` — returns session by ID regardless of tenant
- `existsByName(name)` — checks tmux session name uniqueness across all tenants (tmux namespace invariant)
- `register()` — writes are unscoped; tenancyId is on the Session object
- `remove()`, `updateStatus()`, `touch()` — operate by known ID; callers have valid references

**Callers that use unscoped methods:**

| Caller | Method | Why unscoped |
|---|---|---|
| `SessionIdleScheduler.expiryCheck()` | `allUnscoped()` | `@Scheduled` — no request scope; must expire across all tenants |
| `CasehubStartupService.bootstrapWatchers()` | `allUnscoped()` | Startup observer — no request scope; must recover watchers for all tenants |
| `ClaudonyWorkerExecutionManager.watcherRunnable()` | `findUnscoped()` | Virtual thread — no request scope; watcher must see its own session regardless of tenant |
| `ClaudonyWorkerStatusListener` (all 3 methods) | `findUnscoped()` | Engine CDI observer — no request scope; must resolve sessions across tenants |
| `SessionResource.create()` duplicate check | `existsByName()` | tmux names are a global OS namespace; collision check must span all tenants |
| `SessionResource.rename()` duplicate check | `existsByName()` | Same tmux namespace invariant |

---

## Cache Key Changes

### Channel layout cache — claudony-casehub

```java
// Before
ConcurrentHashMap<UUID, Uni<Map<String, CaseChannel>>> layoutCache;

// After
private record CacheKey(String tenancyId, UUID caseId) {}
ConcurrentHashMap<CacheKey, Uni<Map<String, CaseChannel>>> layoutCache;
```

Key constructed from `tenantContext.currentTenantId()` + `caseId`.

### Causal context map — claudony-casehub

```java
// Before
ConcurrentHashMap<UUID, UUID> causalContext;  // caseId → causedByEntryId

// After
private record CausalKey(String tenancyId, UUID caseId) {}
ConcurrentHashMap<CausalKey, UUID> causalContext;
```

Key constructed from `context.tenancyId()` + `context.caseId()`.

`drainCausalContext()` signature changes from `drainCausalContext(UUID caseId)` to `drainCausalContext(String tenancyId, UUID caseId)`. Called by `ClaudonyLedgerEventCapture.onCaseLifecycleEvent()` which has both values available (`event.tenancyId()` at line 58, `event.caseId()` at line 54).

---

## tmux Session Options

During provisioning, store tenancyId alongside existing case metadata:

```java
tmux.setSessionOption(sessionName, "@casehub_tenant_id", context.tenancyId());
```

Read during bootstrap:

```java
Optional<String> tenancyId = tmux.getSessionOption(name, "@casehub_tenant_id");
// Use tenancyId.orElse(TenancyConstants.DEFAULT_TENANT_ID)
```

---

## Unchanged Components

| Component | Why unchanged |
|---|---|
| `MeshResource` | Gets filtered results from registry via injected services — no tenancy logic at call sites |
| `TerminalWebSocket` | Calls `registry.find()` which is now tenant-filtered — ready for per-tenant isolation when OIDC arrives |
| `CaseEventBroadcaster` | Observes `WorkerCaseLifecycleEvent` — SSE endpoint is already session-scoped |
| `ChannelFleetBroadcaster` | Fleet is same-owner; event carries tenancyId for future use |
| Auth (`ApiKeyAuthMechanism`, `CredentialStore`) | No tenant association until OIDC (platform#16) |
| `PeerRegistry` | Fleet topology is infrastructure, not tenant-scoped |
| `AuthRateLimiter`, `InviteService` | Rate limiting and invites are global, not per-tenant |

---

## Testing

### Test constants

```java
static final String TENANT_A = TenancyConstants.DEFAULT_TENANT_ID;
static final String TENANT_B = "00000000-0000-0000-0000-000000000002";
```

### MutableTenantContext (test support — core/test)

```java
@Alternative @Priority(1)
@ApplicationScoped
class MutableTenantContext implements TenantContext {
    private String tenantId = TenancyConstants.DEFAULT_TENANT_ID;
    void setTenantId(String id) { this.tenantId = id; }
    void resetForTest() { this.tenantId = TenancyConstants.DEFAULT_TENANT_ID; }
    @Override public String currentTenantId() { return tenantId; }
}
```

Follows the established `AuthRateLimiter` pattern: `@ApplicationScoped` with mutable state, `resetForTest()` package-private hook. **Tests that change the tenant must call `resetForTest()` in `@AfterEach`** to prevent state bleeding across `@QuarkusTest` classes (which share one app instance per test run).

### New tests

**SessionRegistryTenantFilterTest (core):**
- `all()` returns only sessions matching current tenant
- `find()` returns empty for other tenant's session
- `find()` returns session for matching tenant
- `findByCaseId()` filters by tenant
- `allUnscoped()` returns all sessions regardless of tenant
- `findUnscoped()` returns session regardless of tenant
- `existsByName()` finds names across all tenants
- `remove()` works on any session regardless of tenant
- `updateStatus()` works on any session regardless of tenant

**Event tenancyId tests:**
- `WorkerCaseLifecycleEvent` carries tenancyId from session
- `WorkerCaseLifecycleEvent` carries `DEFAULT_TENANT_ID` when session not found (fallback)
- `CaseChannelCreatedEvent` carries tenancyId from TenantContext

### Existing test impact

All tests that construct `Session` add `TenancyConstants.DEFAULT_TENANT_ID` as the last argument. No assertion changes — filtered `all()` returns the same results when all sessions share the default tenant.

### Out of scope

- Auth-to-tenant binding (no OIDC)
- Fleet tenant isolation (same-owner)
- Cross-tenant admin patterns (no use case)

---

## Known Forward Issues

**Fleet session federation is tenant-unaware:** Fleet session listing currently works because all sessions are `DEFAULT_TENANT_ID`. The path is: `PeerClient.getSessions(true)` → remote peer's `SessionResource.list()` → remote peer's tenant-filtered `registry.all()`. When OIDC adds real tenants, sessions provisioned for non-default tenants will be invisible to fleet federation. The fix is a dedicated fleet-internal endpoint (e.g. `GET /api/internal/sessions`) using `allUnscoped()`, routed via `PeerClient`. Belongs in a follow-up when fleet crosses tenant boundaries.

**SSE snapshot callbacks are tenant-filtered in system context:** `HybridStrategy.tickAllCases()` and `EventsOnlyStrategy.onLifecycleEvent()` call `snapshotFn.get()` → `SessionResource.buildCaseSnapshot()` → `registry.findByCaseId()` on Mutiny tick threads / CDI observer threads without request scope. After OIDC, heartbeat and lifecycle-driven snapshots for non-default tenants would return zero workers. Same root cause as the fleet issue. Fix: capture tenant at subscribe-time, thread it through the snapshot callback.

**WorkerSessionMapping role→sessionId collision:** The fallback `byRole` map will collide if two tenants provision the same role name. Not introduced by this spec — pre-existing. Noted for the road ahead; the fix is tenant-scoping the mapping keys, which belongs in a follow-up.

---

## Files Changed

| File | Module | Change |
|---|---|---|
| `pom.xml` | core | Add `casehub-platform-api` dependency |
| `TenantContext.java` | core | New interface |
| `DefaultTenantContext.java` | core | New `@ApplicationScoped` impl with `CurrentPrincipal` delegation |
| `Session.java` | core | Add `tenancyId` field |
| `SessionRegistry.java` | core | Constructor injection of `TenantContext`, filter queries, add `allUnscoped()`, `findUnscoped()`, `existsByName()` |
| `WorkerCaseLifecycleEvent.java` | core | Add `tenancyId` field |
| `CaseChannelCreatedEvent.java` | core | Add `tenancyId` field |
| `SessionIdleScheduler.java` | core | Use `allUnscoped()` instead of `all()` |
| `ClaudonyReactiveWorkerProvisioner.java` | casehub | Pass `context.tenancyId()` to Session, store in tmux option, update causal context key, update `drainCausalContext(String tenancyId, UUID caseId)` signature |
| `ClaudonyWorkerStatusListener.java` | casehub | Inject `TenantContext`, use `registry.findUnscoped()` in all 3 methods, resolve tenancyId from session (fallback to `DEFAULT_TENANT_ID`) |
| `ClaudonyReactiveCaseChannelProvider.java` | casehub | Inject TenantContext, update cache key, pass tenancyId to event |
| `ClaudonyLedgerEventCapture.java` | casehub | Update `drainCausalContext()` call to pass `tenancyId` |
| `ClaudonyWorkerExecutionManager.java` | casehub | Use `registry.findUnscoped()` in `watcherRunnable()` |
| `CasehubStartupService.java` | app | Use `registry.allUnscoped()` in `bootstrapWatchers()` |
| `ServerStartup.java` | app | Read `@casehub_tenant_id` from tmux, default to `TenancyConstants.DEFAULT_TENANT_ID` |
| `SessionResource.java` | app | Inject `TenantContext`, pass `tenantContext.currentTenantId()` to new Session; use `registry.existsByName()` for duplicate checks in `create()` and `rename()` |
| Test files (many) | all | Add `TenancyConstants.DEFAULT_TENANT_ID` arg to Session construction |
| `SessionRegistryTenantFilterTest.java` | core | New — two-tenant isolation tests |
| `MutableTenantContext.java` | core/test | New — test support bean with `resetForTest()` |