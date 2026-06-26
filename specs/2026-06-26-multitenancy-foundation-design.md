# Multi-Tenancy Foundation — Design Spec

**Issue:** casehubio/claudony#121
**Branch:** issue-121-multitenancy-foundation
**Date:** 2026-06-26

---

## Summary

Add unconditional tenancyId enforcement to Claudony's in-memory stores, events, and cache keys. Every Session carries a tenancyId, every registry query filters by it, and every CDI event propagates it. Follows protocols PP-20260520-439daf (unconditional filtering) and PP-20260520-e6a5f0 (filtering inside data access only).

---

## Approach: Thin Compliance + Forward Contracts

Claudony's primary state is ephemeral — in-memory `ConcurrentHashMap` stores rebuilt on restart. The only durable tenant-aware state is `CaseLedgerEntry` (already implemented). The issue as filed assumes JPA entities and Repository pattern, but Claudony has no entity tables of its own.

The approach adds tenancyId to the data model, filters unconditionally in the registry, and creates a `TenantContext` seam for future OIDC integration — without overbuilding auth or fleet stories that don't exist yet.

**What this is not:** full multi-tenant auth. There is no `CurrentPrincipal.tenancyId()` integration (requires OIDC, platform#16). The `TenantContext` default bean returns `DEFAULT_TENANT_ID` — filtering runs but is a no-op in single-tenant deployments. When OIDC lands, swap the `@DefaultBean` for a `@RequestScoped` impl backed by `CurrentPrincipal` — zero changes to stores or endpoints.

---

## Decisions Made

### TenantContext lives in core, not casehub module
`SessionRegistry` is in `claudony-core`, which has no casehub-platform dependency. A `TenantContext` functional interface in core avoids the dependency while providing the filtering seam. The `DEFAULT_TENANT_ID` constant is duplicated from `TenancyConstants` — the value is a stable sentinel UUID.

### Session.tenancyId is non-optional, last field
`String tenancyId` (not `Optional`) — tenancy is never absent. Positioned last in the record to force compile errors at every construction site, ensuring each explicitly provides a tenancyId source.

### Write operations stay unscoped
`register()`, `remove()`, `updateStatus()`, `touch()` operate on known session IDs. Callers already have a valid session reference (from a filtered lookup or from system code). Filtering writes would prevent the expiry scheduler from cleaning up other tenants' sessions.

### allUnscoped() is the controlled escape hatch
System operations (expiry, bootstrap, fleet sync) need the full dataset. `allUnscoped()` is named explicitly so it stands out in code review. It is not a "pass a different tenant" method — it is a separate operation for background/admin tasks.

### Fleet stays unscoped
Fleet nodes are same-owner by design. Fleet-key auth (principal = "peer", role = "fleet") is an infrastructure concern, not a tenant concern. Future fleet tenant isolation is a separate issue.

### Events carry tenancyId for forward compatibility
`WorkerCaseLifecycleEvent` and `CaseChannelCreatedEvent` gain `tenancyId`. Observers don't filter by it yet — the event field establishes the contract for future tenant-scoped broadcasting.

### Bootstrap uses DEFAULT_TENANT_ID
Sessions recovered from `tmux list-sessions` on restart get `DEFAULT_TENANT_ID`. A new tmux session option `@casehub_tenant_id` is stored during provisioning so future bootstraps can recover the real tenancyId.

---

## New Types

### TenantContext — claudony-core

```java
package io.casehub.claudony.server;

public interface TenantContext {
    String DEFAULT_TENANT_ID = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
    String currentTenantId();
}
```

### DefaultTenantContext — claudony-core

```java
@DefaultBean
@ApplicationScoped
public class DefaultTenantContext implements TenantContext {
    @Override
    public String currentTenantId() {
        return DEFAULT_TENANT_ID;
    }
}
```

`@ApplicationScoped` — stateless, returns a constant. `@DefaultBean` — a future `@RequestScoped` impl backed by `CurrentPrincipal` overrides it automatically.

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
| `ServerStartup.bootstrapRegistry()` | `TenantContext.DEFAULT_TENANT_ID` (read from tmux `@casehub_tenant_id` option when available) |
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
| `WorkerCaseLifecycleEvent` | `ClaudonyWorkerStatusListener.onWorkerStarted()` | Session lookup via `sessionMapping` → `registry.findUnscoped()` → `session.tenancyId()` |
| `WorkerCaseLifecycleEvent` | `ClaudonyWorkerStatusListener.onWorkerCompleted()` | Same |
| `WorkerCaseLifecycleEvent` | `ClaudonyWorkerStatusListener.onWorkerStalled()` | Same via `registry.findUnscoped()` |
| `CaseChannelCreatedEvent` | `ClaudonyReactiveCaseChannelProvider.createQhorusChannel()` | `tenantContext.currentTenantId()` (injected) |

---

## SessionRegistry Changes — claudony-core

`SessionRegistry` injects `TenantContext`. Public query methods filter unconditionally.

**Filtered (tenant-scoped):**
- `all()` — returns only sessions matching `tenantContext.currentTenantId()`
- `find(id)` — returns `Optional.empty()` if session belongs to different tenant
- `findByCaseId(caseId)` — filters by tenant on top of caseId filter

**Unscoped (system operations):**
- `allUnscoped()` — returns all sessions regardless of tenant
- `findUnscoped(id)` — returns session by ID regardless of tenant
- `register()` — writes are unscoped; tenancyId is on the Session object
- `remove()`, `updateStatus()`, `touch()` — operate by known ID; callers have valid references

**Callers that use unscoped methods:**
- `SessionIdleScheduler.expiryCheck()` — `allUnscoped()`, must expire across all tenants
- Fleet session listing via `PeerResource` — `allUnscoped()`, fleet is same-owner infrastructure
- `ClaudonyWorkerStatusListener` — `findUnscoped()`, called by engine observers in system context (no request scope); must resolve sessions across tenants to fire lifecycle events and update status

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

---

## tmux Session Options

During provisioning, store tenancyId alongside existing case metadata:

```java
tmux.setSessionOption(sessionName, "@casehub_tenant_id", context.tenancyId());
```

Read during bootstrap:

```java
Optional<String> tenancyId = tmux.getSessionOption(name, "@casehub_tenant_id");
// Use tenancyId.orElse(TenantContext.DEFAULT_TENANT_ID)
```

---

## Unchanged Components

| Component | Why unchanged |
|---|---|
| REST endpoints (`SessionResource`, `MeshResource`) | Get filtered results from registry automatically — no tenancy logic at call sites |
| `TerminalWebSocket` | Calls `registry.find()` which is now tenant-filtered — wrong-tenant connections rejected |
| `CaseEventBroadcaster` | Observes `WorkerCaseLifecycleEvent` — SSE endpoint is already session-scoped |
| `ChannelFleetBroadcaster` | Fleet is same-owner; event carries tenancyId for future use |
| Auth (`ApiKeyAuthMechanism`, `CredentialStore`) | No tenant association until OIDC (platform#16) |
| `PeerRegistry` | Fleet topology is infrastructure, not tenant-scoped |
| `AuthRateLimiter`, `InviteService` | Rate limiting and invites are global, not per-tenant |

---

## Testing

### Test constants

```java
static final String TENANT_A = TenantContext.DEFAULT_TENANT_ID;
static final String TENANT_B = "00000000-0000-0000-0000-000000000002";
```

### MutableTenantContext (test support)

```java
@Alternative @Priority(1)
@ApplicationScoped
class MutableTenantContext implements TenantContext {
    private String tenantId = DEFAULT_TENANT_ID;
    void setTenantId(String id) { this.tenantId = id; }
    void reset() { this.tenantId = DEFAULT_TENANT_ID; }
    @Override public String currentTenantId() { return tenantId; }
}
```

### New tests

**SessionRegistryTenantFilterTest (core):**
- `all()` returns only sessions matching current tenant
- `find()` returns empty for other tenant's session
- `find()` returns session for matching tenant
- `findByCaseId()` filters by tenant
- `allUnscoped()` returns all sessions regardless of tenant
- `findUnscoped()` returns session regardless of tenant
- `remove()` works on any session regardless of tenant
- `updateStatus()` works on any session regardless of tenant

**Event tenancyId tests:**
- `WorkerCaseLifecycleEvent` carries tenancyId from session
- `CaseChannelCreatedEvent` carries tenancyId from TenantContext

### Existing test impact

All tests that construct `Session` add `TenantContext.DEFAULT_TENANT_ID` as the last argument. No assertion changes — filtered `all()` returns the same results when all sessions share the default tenant.

### Out of scope

- Auth-to-tenant binding (no OIDC)
- Fleet tenant isolation (same-owner)
- Cross-tenant admin patterns (no use case)

---

## Files Changed

| File | Module | Change |
|---|---|---|
| `TenantContext.java` | core | New interface |
| `DefaultTenantContext.java` | core | New `@DefaultBean` impl |
| `Session.java` | core | Add `tenancyId` field |
| `SessionRegistry.java` | core | Inject `TenantContext`, filter queries, add `allUnscoped()` |
| `WorkerCaseLifecycleEvent.java` | core | Add `tenancyId` field |
| `CaseChannelCreatedEvent.java` | core | Add `tenancyId` field |
| `ClaudonyReactiveWorkerProvisioner.java` | casehub | Pass `context.tenancyId()` to Session, store in tmux option, update causal context key |
| `ClaudonyWorkerStatusListener.java` | casehub | Use `registry.findUnscoped()`, resolve tenancyId from session for events |
| `ClaudonyReactiveCaseChannelProvider.java` | casehub | Inject TenantContext, update cache key, pass tenancyId to event |
| `ServerStartup.java` | app | Read `@casehub_tenant_id` from tmux, default to `DEFAULT_TENANT_ID` |
| `SessionResource.java` | app | Pass `tenantContext.currentTenantId()` to new Session |
| `SessionIdleScheduler.java` | core | Use `allUnscoped()` instead of `all()` |
| Test files (many) | all | Add `tenancyId` arg to Session construction |
| `SessionRegistryTenantFilterTest.java` | core | New — two-tenant isolation tests |
| `MutableTenantContext.java` | core/test | New — test support bean |
