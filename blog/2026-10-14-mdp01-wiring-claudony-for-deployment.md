---
layout: post
title: "Wiring Claudony for the Deployment Loop"
date: 2026-10-14
entry_type: note
subtype: diary
projects: [casehubio/claudony]
tags: [claudony, ops, desiredstate, registry, lifecycle, deployment]
---

CaseHub's deployment story converges three repos: desiredstate handles the reconciliation runtime — graphs, planning, fault policy. ops defines what to deploy — app types, containers, infrastructure. Claudony manages where things run — the fleet registry, health, topology.

The gap was on claudony's side. An ops `NodeProvisioner` targeting claudony needed four things that didn't exist: a namespace to scope deployments to tenants, a way to update an app's spec without tearing it down, a non-destructive pause, and somewhere to put the deployment artifact reference. Four issues, all straightforward individually, all pointless individually.

**Namespace was alignment, not invention.** `RegistryEntry` already had a `namespace` field. `ApplicationRegistrationService.register()` was passing `spec.name()` into it — using the app name as the namespace, which is wrong. The fix was a new `namespace` field on `ApplicationSpec` that defaults to `"default"` and passes through correctly. The `listApps` endpoint gained an optional namespace query parameter. The interesting bit was that the existing tests were all asserting `namespace == "scaffold"` because that was the app name being used as namespace. Every assertion needed updating.

**updateApp exposed a missing primitive.** `RegistryService` has no `update()` method — only register, deregister, heartbeat, and resolve. The question was whether to add one to platform-api or work within the existing API. We went with deregister + re-register: capture the original timestamps and health state, deregister, re-register with the preserved values. The pools needed diffing — new pools get registered, removed pools get deregistered (which required adding `AgentPoolDefinitionRegistry.deregister()`, since that method didn't exist either), and changed pools get their capacity updated. The brief window where the entry doesn't exist between deregister and register is acceptable for an in-process atomic operation.

**Suspend/resume needed a cross-repo change.** `HealthStatus` in `casehub-platform-api` only had `HEALTHY`, `DEGRADED`, and `DOWN`. Desiredstate's `TargetStatus.SUSPENDED` needs a corresponding health value — `DOWN` won't do because suspended is intentional, down is a failure. I filed platform#615 to add `SUSPENDED` to the enum, and it landed immediately. With the new value available, suspend and resume are simple: deregister the app, re-register with `SUSPENDED` or `HEALTHY`. No pool capacity manipulation — `PoolConfig` enforces `maxActive >= 1`, and more importantly the app's health status is what desiredstate's `ActualStateAdapter` reads. The pool capacity is the provisioner's concern.

`PeerRegistry.mapHealth()` had a switch expression on `HealthStatus` that wasn't exhaustive after the new value — the compiler caught it.

**The deployment artifact was the lightest touch.** ops's `CaseHubApplicationSpec` has `applicationId`, `source`, `image`, `RuntimeConfig`, `config`, `dependencies`, `replicas`. Claudony doesn't build containers or manage replicas — it manages agent pools. So `ApplicationSpec` absorbed only `image` (the artifact reference) and `config` (runtime key-value pairs). Both are stored in `RegistryEntry` metadata with a `config.` prefix on the config keys to avoid collisions. The provisioner passes the image reference; claudony stores and serves it.

The four changes compose naturally. Namespace scopes deployments. updateApp enables drift reconciliation without destroy cycles. Suspend/resume enables the desiredstate `SUSPENDED` path. The deployment artifact gives the provisioner somewhere to put the payload. An ops `NodeProvisioner` can now target claudony's API for the full lifecycle — register, update, suspend, resume, deregister — with proper namespace isolation and artifact tracking.
