# Handoff — 2026-06-23

*Updated: parent#291 closed — removed from backlog.*

**Head commit (project):** `c72fb24` — docs(#145): add ARC42STORIES.MD — integration-tier record

## What landed this session

### claudony#145 — ARC42STORIES.MD complete

Full Arc42Stories architecture record written from scratch: 3 Journeys, 9 Chapters across the full delivery arc (Core Session Engine → Auth → Fleet → Agent/MCP → Qhorus → CaseHub SPIs → Agent Mesh → Case Worker Panel → Production Orchestration), 9 Layer entries each with Key files, Gotchas (Symptom→Cause→Fix), and Pattern-to-replicate sections.

Quality gate caught: stale class names in DESIGN.md (`McpServer` → `ClaudonyMcpTools`, `ClaudonyWorkerProvisioner` → `ClaudonyReactiveWorkerProvisioner`, `WebAuthnPatcher`/`LenientNoneAttestation` removed in Quarkus upgrade); issue #93 body described a pending fix already implemented in db61484. All corrected before commit. Tier label corrected to Integration. File placed in project repo per PP-20260603-33c84c.

DESIGN.md redirect header added. CLAUDE.md updated to declare ARC42STORIES.MD as primary architecture record.

CI: 587 tests green, pushed to casehubio/claudony main.

## State

- main: `c72fb24`
- All 587 tests pass locally and on CI
- #145 closed, branch stamped

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #105 | Separate Claudony/Qhorus MCP endpoints (Phase A architecture) | M | Med | No blockers; next discrete work |
| #157 | Migrate Worker imports to casehub-worker-api | S | Low | Refactor only |
