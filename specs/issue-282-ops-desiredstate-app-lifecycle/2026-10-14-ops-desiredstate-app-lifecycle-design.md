# Design: ops/desiredstate integration — app lifecycle for fleet deployment

**Epic:** casehubio/claudony#282
**Sub-issues:** #283 (namespace), #284 (updateApp), #285 (suspend/resume), #286 (deployment artifact)
**Date:** 2026-10-14

## Overview

Claudony's `ClaudonyAppApi` currently supports register, deregister, heartbeat, list, and get. An ops `NodeProvisioner` and `ActualStateAdapter` targeting claudony need four capabilities that don't exist yet: namespace-scoped apps, in-place spec updates, non-destructive suspend/resume, and a deployment artifact reference.

This epic adds all four to `ApplicationSpec`, `ApplicationInfo`, `ApplicationRegistrationService`, and `ClaudonyAppApi` — keeping claudony's scope as fleet/session manager (not build pipeline or container orchestrator).

## #283: Namespace field on ApplicationSpec

### What changes

**ApplicationSpec** gains an optional `namespace` field defaulting to `"default"`:

```java
public record ApplicationSpec(
        String name,
        String namespace,
        String displayName,
        String description,
        List<PoolSpec> pools
) {
    public ApplicationSpec {
        Objects.requireNonNull(name, "name");
        if (name.isBlank()) throw new IllegalArgumentException("name must not be blank");
        Objects.requireNonNull(displayName, "displayName");
        if (namespace == null || namespace.isBlank()) namespace = "default";
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }
}
```

**ApplicationInfo** gains matching `namespace` field.

**ApplicationRegistrationService.register()** passes `spec.namespace()` as the `RegistryEntry` namespace parameter (currently incorrectly passes `spec.name()`).

**ApplicationRegistrationService.list()** adds a `list(String namespace)` overload that filters `RegistryQuery` by namespace. The existing no-arg `list()` returns all apps for the current tenant (unchanged).

**ClaudonyAppApi** adds an optional `@QueryParam namespace` to the list endpoint for namespace filtering.

### Test plan

- Registering with no namespace → defaults to `"default"`, RegistryEntry.namespace == "default"
- Registering with explicit namespace → carried through to RegistryEntry and ApplicationInfo
- list() with namespace filter → returns only apps in that namespace
- list() without filter → returns all apps for tenant

## #284: updateApp — in-place spec updates

### What changes

**ClaudonyAppApi** gains:

```java
@PlatformMutation("Update an application")
@RestPath("/{name}")
public ApplicationInfo updateApp(@PathParam String name, ApplicationSpec spec) {
    return appService.update(name, spec);
}
```

**ApplicationRegistrationService.update()** implements deregister + re-register with preserved timestamps:

1. `resolve(name)` — throw 404 if not found
2. Validate `spec.name()` matches path `name` — throw 400 if mismatched (no rename via update)
3. Capture `registeredAt`, `lastHeartbeat` from existing entry
4. `deregister(name)` — removes entry and cascades relationships
4. Build new `RegistryEntry` with updated spec fields but original `registeredAt` and `lastHeartbeat`
5. `register(newEntry)`
6. Diff pools between old and new spec:
   - Pools in new but not old → register in `AgentPoolDefinitionRegistry`, link relationship
   - Pools in old but not new → deregister from pool registry
   - Pools in both with changed min/max → update pool definition
7. Re-link all pool relationships
8. Return updated `ApplicationInfo`

### Design notes

- Pool identity is `appName + "/" + agentId` (existing convention)
- Pool diff uses `agentId` as the identity key
- `registeredAt` and health state are preserved across updates
- `lastHeartbeat` is preserved (not reset) — the update is not a heartbeat

### Test plan

- Update displayName → preserved registeredAt, updated displayName
- Update pools (add new) → new pool registered, linked
- Update pools (remove old) → old pool deregistered, unlinked
- Update pools (change scaling) → pool definition updated
- Update non-existent app → 404
- Health and heartbeat timestamps preserved across update

## #285: Suspend/resume lifecycle

### What changes

**ClaudonyAppApi** gains two endpoints:

```java
@PlatformMutation("Suspend an application")
@RestPath("/{name}/suspend")
public void suspendApp(@PathParam String name) {
    appService.suspend(name);
}

@PlatformMutation("Resume an application")
@RestPath("/{name}/resume")
public void resumeApp(@PathParam String name) {
    appService.resume(name);
}
```

**ApplicationRegistrationService.suspend():**

1. `resolve(name)` — throw 404 if not found
2. If already `SUSPENDED` → no-op return
3. Deregister + re-register with `HealthStatus.SUSPENDED` (same deregister/register pattern as update, preserving all timestamps)
4. Iterate owned pools via `relationships(name)` → suspend each pool in `AgentPoolDefinitionRegistry`

**ApplicationRegistrationService.resume():**

1. `resolve(name)` — throw 404 if not found
2. If not `SUSPENDED` → no-op return
3. Deregister + re-register with `HealthStatus.HEALTHY`
4. Iterate owned pools → resume each pool

**ApplicationInfo** gains a convenience method or the existing `health` field naturally reports `SUSPENDED`.

### Dependency

Uses `HealthStatus.SUSPENDED` from casehubio/platform#615 (landed).

### Design notes

- Suspend is app-level: all owned pools are suspended together
- Pool suspend/resume uses existing `AgentPoolDefinitionRegistry` lifecycle
- No per-pool granular suspend — use pool scaling (minActive=0) for that
- Idempotent: suspend on suspended = no-op, resume on non-suspended = no-op

### Test plan

- Suspend healthy app → health becomes SUSPENDED, pools suspended
- Suspend already-suspended app → no-op
- Resume suspended app → health becomes HEALTHY, pools resumed
- Resume non-suspended app → no-op
- Suspend non-existent app → 404
- ApplicationInfo reflects SUSPENDED status

## #286: Deployment artifact reference

### What changes

**ApplicationSpec** gains two optional fields:

```java
public record ApplicationSpec(
        String name,
        String namespace,
        String displayName,
        String description,
        String image,
        Map<String, String> config,
        List<PoolSpec> pools
) {
    public ApplicationSpec {
        Objects.requireNonNull(name, "name");
        if (name.isBlank()) throw new IllegalArgumentException("name must not be blank");
        Objects.requireNonNull(displayName, "displayName");
        if (namespace == null || namespace.isBlank()) namespace = "default";
        if (config == null) config = Map.of();
        config = Map.copyOf(config);
        if (pools == null) pools = List.of();
        pools = List.copyOf(pools);
    }
}
```

**ApplicationInfo** gains matching `image` and `config` fields.

**ApplicationRegistrationService.register()** stores `image` and `config` entries in `RegistryEntry.metadata()`:

- `image` → `metadata.put("image", spec.image())` (if non-null)
- `config` entries → `metadata.put("config." + key, value)` (prefixed to avoid collisions)

**ApplicationRegistrationService.toApplicationInfo()** reads them back from metadata.

### Design notes

- `image` is the artifact reference an ops provisioner passes — container image, JAR URI, or any deployment target reference
- `config` is runtime configuration the provisioner needs to pass through — env vars, feature flags, etc.
- Both are optional — existing apps without deployment artifacts continue to work
- `updateApp()` (#284) handles updating these fields naturally since it rebuilds the full RegistryEntry
- No build pipeline or container orchestration logic — claudony stores and serves the reference, the provisioner acts on it

### Alignment with ops

Claudony's `image` + `config` maps to `CaseHubApplicationSpec`'s `image()` and `config()`. Fields not mirrored:
- `source` — build pipeline concern (ops only)
- `RuntimeConfig` — engine/modules metadata (ops only)
- `dependencies` — deployment graph concern (ops only)
- `replicas` — container scaling (claudony uses pool minActive/maxActive instead)

### Test plan

- Register with image and config → stored in metadata, returned in ApplicationInfo
- Register without image/config → defaults (null image, empty config)
- Update image via updateApp → metadata updated
- Config keys prefixed to avoid metadata collisions

## Implementation order

1. **#283 (namespace)** — XS, no dependencies. Foundation for everything else.
2. **#284 (updateApp)** — S, uses the namespace-aware registration from #283.
3. **#285 (suspend/resume)** — S, uses the deregister/register pattern from #284.
4. **#286 (deployment artifact)** — S, adds fields to the spec established in #283, updatable via #284.

Each sub-issue is independently shippable. The order minimises rework.

## References

- `ApplicationSpec.java` (claudony casehub module) — current 4-field record
- `ApplicationInfo.java` (claudony casehub module) — current response record
- `ApplicationRegistrationService.java` (claudony casehub module) — current registration logic
- `ClaudonyAppApi.java` (claudony app module) — current 5-endpoint API
- `RegistryEntry.java` (platform-api) — namespace is 3rd constructor arg
- `RegistryService.java` (platform-api) — no update() method
- `HealthStatus.java` (platform-api) — HEALTHY, DEGRADED, DOWN, SUSPENDED (platform#615)
- `CaseHubApplicationSpec.java` (ops api) — ops-side application node spec
- `CaseHubAppProvisionHandler.java` (ops deployment) — provision/deprovision flow
- casehubio/platform#615 — SUSPENDED added to HealthStatus
