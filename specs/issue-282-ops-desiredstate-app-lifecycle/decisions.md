# Decisions — #282 ops/desiredstate integration

## D1: SUSPENDED HealthStatus — cross-repo in platform-api

**Choice:** Add `SUSPENDED` to `HealthStatus` enum in `casehub-platform-api`
**Alternatives:**
- Claudony-local appStatus in metadata — avoids cross-repo change but creates a parallel status concept that diverges from registry health semantics
- Defer #285 entirely — delays the feature unnecessarily when the platform change is trivial
**Rationale:** `SUSPENDED` is an intentional operational state, semantically distinct from `DOWN` (failure). It maps directly to desiredstate's `TargetStatus.SUSPENDED`. The enum addition has low blast radius — existing consumers ignore unknown values.
**Trade-offs:** Requires platform SNAPSHOT refresh in all consumers before they can compile against the new value. Mitigated: platform#615 already landed and published.
**Sources:** `HealthStatus.java` (platform-api), `CaseHubApplicationSpec.java` (ops), claudony#285 issue body
**Exploration:** quick
**Status:** captured

## D2: updateApp strategy — deregister + re-register with preserved timestamps

**Choice:** Deregister then re-register atomically, preserving `registeredAt` and `lastHeartbeat` from the original entry
**Alternatives:**
- Add `update()` to `RegistryService` — cleaner semantics but requires a platform-api change and InMemoryRegistryService implementation for what is essentially a metadata swap
- Call `register()` hoping it overwrites — depends on undocumented behavior; fragile
**Rationale:** `RegistryService` has no update method. Deregister+register is explicit, uses existing API, and we control the preserved fields by passing the original timestamps into the new `RegistryEntry`.
**Trade-offs:** Brief window where the entry doesn't exist (between deregister and register). Acceptable for in-process atomic operation. Pool relationships need explicit re-linking.
**Sources:** `RegistryService.java` (platform-api), `ApplicationRegistrationService.java` (claudony), claudony#284 issue body
**Exploration:** quick
**Status:** captured

## D3: Namespace — explicit field defaulting to "default"

**Choice:** Add `namespace` as an optional field on `ApplicationSpec`, defaulting to `"default"`
**Alternatives:**
- Derive from `tenancyId` — simpler API but conflates organizational grouping with security isolation; they're different concerns
- Hierarchical `tenantId/appName` — richer but no existing consumer uses hierarchical namespaces in RegistryEntry
**Rationale:** `RegistryEntry` already has a `namespace` field. `ApplicationRegistrationService` currently misuses `spec.name()` as namespace. An explicit field aligns with the existing data model and gives callers control over organizational grouping.
**Trade-offs:** Callers must pass namespace explicitly for multi-tenant deployments. Mitigated by the `"default"` default — single-tenant deployments don't change.
**Sources:** `RegistryEntry.java` (platform-api — namespace is 3rd constructor arg), `ApplicationRegistrationService.java` (currently passes name as namespace), claudony#283 issue body
**Exploration:** quick
**Status:** captured

## D4: Suspend scope — app-level only

**Choice:** `suspendApp()` suspends the entire app — all owned pools stop accepting work, health transitions to `SUSPENDED`
**Alternatives:**
- App + per-pool suspend — more granular but adds complexity; per-pool suspend is partially covered by setting `minActive=0` via scaling
- Per-pool only — doesn't map to desiredstate's node-level `SUSPENDED` which operates on apps as nodes
**Rationale:** desiredstate's `TargetStatus.SUSPENDED` is per-node, and an app is a node. App-level suspend is the natural mapping. Per-pool granularity can be added later.
**Trade-offs:** No per-pool suspend — if you want one pool running while others are paused, you'd use scaling (minActive=0) instead. Acceptable for now.
**Sources:** claudony#285 issue body, desiredstate `TargetStatus` concept
**Exploration:** quick
**Status:** captured

## D5: Deployment artifact shape — image + config only

**Choice:** Add `image` (String, optional) and `config` (Map<String,String>, optional) to `ApplicationSpec`
**Alternatives:**
- Full mirror of `CaseHubApplicationSpec` (image, source, runtime, config, dependencies, replicas) — adds fields claudony can't act on without a build pipeline
- Typed `DeploymentDescriptor` record wrapping image + config — adds a type for two fields, over-engineering for the current need
**Rationale:** Claudony manages agent pools and fleet sessions, not build pipelines or container orchestration. `image` is the artifact reference the provisioner passes; `config` is runtime configuration. Source, RuntimeConfig, dependencies, and replicas are ops-provisioner concerns.
**Trade-offs:** If claudony later needs to trigger builds or manage replicas, those fields would need adding. Unlikely given claudony's fleet management role.
**Depends on:** D3 (namespace — namespace scopes the deployment target)
**Sources:** `CaseHubApplicationSpec.java` (ops), `CaseHubAppProvisionHandler.java` (ops — uses image and config for deploy), claudony#286 issue body
**Exploration:** quick
**Status:** captured
