# Handoff — 2026-06-22

**Head commit (project):** `7be4f61` — refactor(#159): migrate CaseChannelLayout + MeshParticipationStrategy to casehub-engine-api

## What landed this session

### claudony#159 — Migrate CaseChannelLayout + MeshParticipationStrategy to engine-api (parent#93)

Deleted 10 files (7 production types + 3 test classes) that moved to `io.casehub.api.spi.mesh` in engine-api.

Key changes:
- `ClaudonyReactiveCaseChannelProvider` — import update only
- `ClaudonyReactiveWorkerContextProvider` — delete `selectStrategy()`; use `MeshParticipationStrategy.named()`; fix `strategyFor(workerId, null)` → `strategyFor(workerId, caseId)` (resolves design lie documented in DESIGN.md line 540)
- `MeshSystemPromptTemplate` — import updates
- Tests: import updates for `ActiveParticipationStrategy`, `NormativeChannelLayout` etc.
- `docs/DESIGN.md` line 540: removed null-context Outstanding clause; shared-data keys / epic#86 clause preserved

143 tests green.

## State

- main: `7be4f61` (merged from `issue-159-mesh-spi-migration`)
- Branch `issue-159-mesh-spi-migration` stamped closed
- Issue claudony#159 closed

## Next candidates

- claudony#160 — update ARC42STORIES.MD for mesh SPI migration
- claudony#105 — separate Claudony/Qhorus MCP endpoints (M-scale, Phase A architecture)
