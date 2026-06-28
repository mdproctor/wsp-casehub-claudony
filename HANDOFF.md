# Handoff — 2026-06-28

**Head commit (project):** `ab05594` — feat(#121): multi-tenancy foundation — tenancyId enforcement throughout

## What landed this session

### claudony#121 — Multi-tenancy foundation

Unconditional tenancyId enforcement across Claudony's in-memory stores, CDI events, and cache keys per PP-20260520-439daf (unconditional filtering) and PP-20260520-e6a5f0 (filtering inside data access only).

New types: `TenantContext` (interface), `DefaultTenantContext` (`@ApplicationScoped`, delegates to `CurrentPrincipal` via `Arc.container().requestContext().isActive()`). `casehub-platform-api` dependency added to `claudony-core`.

`Session` record gained `tenancyId` as last field (non-optional `String`). `SessionRegistry` filters `all()`/`find()`/`findByCaseId()` unconditionally; system operations use `allUnscoped()`/`findUnscoped()`/`existsByName()`. CDI events `WorkerCaseLifecycleEvent` and `CaseChannelCreatedEvent` carry `tenancyId`. Channel provider cache keyed by `(tenancyId, caseId)`. Causal context keyed by `(tenancyId, caseId)`.

Design spec: `specs/2026-06-26-multitenancy-foundation-design.md` (v3, 3 review rounds).
Garden entry: GE-20260627-f3476f — Arc.container().requestContext().isActive() scope-safe delegation.
Parent doc sync: casehubio/parent#316 filed for claudony.md updates.

## State

- main: `ab05594` (1 squashed commit from 11)
- 587 tests pass (16 core + 163 casehub + 408 app)
- #121 closed

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
| #161 | Adopt casehub-pages for UI via Quinoa | S | Low | Frontend architecture shift |
| #141 | ActionRiskClassifier oversight gate | M | High | Blocked on engine#402 |
| — | Mesh system prompt delivery to CLI | S | Med | Gap: WorkerContext.properties("systemPrompt") not consumed by provisioner |
| — | casehub-ops co-deployment (OpsProviderConfigSource) | M | Med | Phase 2 of #156 |
