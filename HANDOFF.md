# Handoff — 2026-06-27

**Head commit (project):** `08b5652` — test(#121): assert tenancyId on fired events, add fallback test
**Branch:** `issue-121-multitenancy-foundation` (10 commits ahead of main)

## What landed this session

### claudony#121 — Multi-tenancy foundation

Unconditional tenancyId enforcement across Claudony's in-memory stores, CDI events, and cache keys. Follows protocols PP-20260520-439daf (unconditional filtering) and PP-20260520-e6a5f0 (filtering inside data access only).

**Core changes:**
- `TenantContext` interface + `DefaultTenantContext` (`@ApplicationScoped`, delegates to `CurrentPrincipal` via `Arc.container().requestContext().isActive()` scope check)
- `casehub-platform-api` dependency added to `claudony-core`
- `Session` record gains `tenancyId` as last field (non-optional `String`)
- `SessionRegistry` — constructor injection of `TenantContext`, `all()`/`find()`/`findByCaseId()` filter by tenant unconditionally, `allUnscoped()`/`findUnscoped()`/`existsByName()` for system operations
- `WorkerCaseLifecycleEvent` and `CaseChannelCreatedEvent` gain `tenancyId`
- `SessionIdleScheduler` uses `allUnscoped()` (expiry across all tenants)
- Provisioner stamps `tenancyId` on Session and `@casehub_tenant_id` tmux option
- Causal context keyed by `(tenancyId, caseId)` with updated `drainCausalContext` signature
- `ClaudonyWorkerExecutionManager.watcherRunnable()` uses `findUnscoped()`
- `ClaudonyWorkerStatusListener` uses `findUnscoped()` in all 3 methods, resolves tenancyId from session with `DEFAULT_TENANT_ID` fallback
- Channel provider layout cache keyed by `(tenancyId, caseId)`, event carries `tenantContext.currentTenantId()`
- `ServerStartup.bootstrapRegistry()` reads `@casehub_tenant_id` from tmux
- `CasehubStartupService.bootstrapWatchers()` uses `allUnscoped()`
- `SessionResource` — `existsByName()` for tmux namespace checks, `tenantContext.currentTenantId()` for new sessions

**Test impact:** 587 tests (was 576). 10 new: `DefaultTenantContextTest` (1), `SessionRegistryTenantFilterTest` (9), `ClaudonyWorkerStatusListenerTest` gained 1 fallback test. 46 files changed, 448 insertions, 166 deletions.

**Design spec:** `specs/2026-06-26-multitenancy-foundation-design.md` (v3, 3 review rounds)

**Garden entry:** GE-20260627-f3476f — Arc.container().requestContext().isActive() scope-safe delegation technique

## State

- Branch: `issue-121-multitenancy-foundation` — 10 commits, NOT merged to main yet
- 587 tests pass (16 core + 163 casehub + 408 app)
- Final whole-branch code review passed (Opus) — 0 Critical, 0 Important after fixes
- #121 still open

## work-end pending

**Branch is implementation-complete and review-clean.** Resume work-end in a new session to complete the close:

Remaining work-end steps:
1. Pre-close sweep items still to run: protocol sweep, update-claude-md, implementation-doc-sync, ADR check, write-content (diary)
2. Code review (already done — Opus whole-branch review passed)
3. Artifact promotion (specs → project, plans → attic)
4. Journal merge → ARC42STORIES.MD (journal is currently empty — will need retrospective or skip)
5. Issue close (#121)
6. Squash (10 commits → semantic squash before push)
7. Rebase onto main, push
8. Mark branch closed

To resume: `work-end` on branch `issue-121-multitenancy-foundation`.

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
| #161 | Adopt casehub-pages for UI via Quinoa | S | Low | Frontend architecture shift |
| #141 | ActionRiskClassifier oversight gate | M | High | Blocked on engine#402 |
| — | Mesh system prompt delivery to CLI | S | Med | Gap: WorkerContext.properties("systemPrompt") not consumed by provisioner |
| — | casehub-ops co-deployment (OpsProviderConfigSource) | M | Med | Phase 2 of #156 |
