# HANDOFF — casehub-claudony

## Last Session

Implemented #271 (ops view — Stage 3c of epic #267): fleet-wide operational visibility API. `OpsService` aggregates from 5 backing services (RegistryService, PoolService, PeerRegistry, ApplicationRegistrationService, QhorusDashboardService) into a typed `OpsSnapshot`. `ClaudonyOpsApi` @McpDomain exposes `/snapshot` and `/health` REST endpoints. `OpsEventBus` observes `RegistryEvent` CDI events and pushes live topology updates via SSE at `/api/ops/events`. 13 new tests (4 unit OpsServiceTest, 5 unit OpsEventBusTest, 3 integration ClaudonyOpsApiTest, 1 integration OpsEventsResourceTest). Landed as `364d0dc` on main. Epic #267 Stage 3c marked Done.

Key discovery: `domainFilter` in `app/pom.xml` compiler args must include any new @McpDomain domain — without it, the GraphQLResolverProcessor won't generate the REST resource class, causing 404s.

Pre-existing issue: `CaseLineageQueryIntegrationTest` and `ClaudonyLedgerEventCaptureTest` fail to compile — `io.casehub.ledger.model.CaseLedgerEntry` is missing from the installed `casehub-ledger-core` SNAPSHOT. Workaround: temporarily move the two files to `/tmp/` before running tests, restore after. Not caused by this session's work.

## Next Issue

**#272 — feat: module decomposition — fleet/pool/mesh as independent services (Stage 3d)**

Depends on #271 (done). Decomposes Claudony's fleet/pool/mesh capabilities into independently deployable services. This is architectural restructuring, not feature addition — different mode of work from #271.

Also available: **platform#575** (XS/Low) — add `registry-client` to `spring-parity-exceptions.txt`. Quick win to start with.

## How to Continue

Run `work start #272` to begin the next epic stage. The `.plan` will be scaffolded by work-start. Consider brainstorming first — module decomposition has significant design decisions around service boundaries and API contracts.

## Branch State

- **main**: clean, up to date with origin
- **issue-271-ops-view-fleet-visibility**: stamped as closed (backup at `backup/pre-squash-issue-271-ops-view-fleet-visibility-20261012`)
- Workspace branch `issue-271-ops-view-fleet-visibility` has unpushed spec/plan commits (workspace artifacts, not source)

## Epic #267 Progress

| Stage | Issue | Status |
|-------|-------|--------|
| 1. Registry core | platform#557 | Done |
| 2. Migration | #268 | Done |
| 3a. App registration | #269 | Done |
| 3b. Registry REST | #270 | Done |
| 3c. Ops view | #271 | Done |
| 3d. Module decomposition | #272 | Open — next |
| 3e. UI extraction | #273 | Open |
| 4. Durable cascade | #274 | Open |
