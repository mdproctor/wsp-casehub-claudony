# Handoff — 2026-06-30

**Head commit (project):** `a8ed052` — fix(#461): add @WorkerBackend qualifier to 4 test injection sites

## What landed this session

- Fixed engine's cross-repo `@WorkerBackend` migration (`ffa8e1b`) — 4 test `@Inject`/`@InjectMock` sites needed the qualifier (`a8ed052`)
- Closed #165 — duplicate of #164 (CompositeProviderConfigSource already delegates to ProvisionerConfigRegistry)
- Coherence audit: ProvisionerConfigRegistry SPI aligned across engine/claudony/openclaw/ops. openclaw#56 also confirmed done.
- Garden entry GE-20260630-95eb64 — CDI qualifier cross-repo propagation gotcha

## State

- main: `a8ed052` (2 commits since last session: engine's @WorkerBackend + test fix)
- 601 tests pass (16 core + 177 casehub + 408 app)
- **2 unpushed commits** on project main — push before starting new branch work
- #163 closed, #164 closed, #165 closed

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #161 | Adopt casehub-pages for UI via Quinoa | S | Low | Only unblocked S issue |
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
| #141 | ActionRiskClassifier oversight gate | M | High | Blocked on engine#402 |
