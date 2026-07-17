# Handoff — 2026-07-17

**Head commit (project):** `37c7f36` — fix(#171): enable legacy decorators for Lit E2E compat

## What landed this session

- #171 closed — root cause was TC39 decorator pass-through in esbuild, not GITHUB_TOKEN/npm install as originally documented. Fix: `experimentalDecorators: true` + `useDefineForClassFields: false` in `tsconfig.json`
- Garden entry GE-20260717-19540a submitted (esbuild/Lit/Chromium decorator gotcha, score 13/15)

## State

- main: `37c7f36`, pushed, 591 Java tests + 15 vitest pass
- E2E `terminalPage_loadsStructure` now passes
- E2E tests for channel-panel/case-context/proxy-resize still fail — DOM expectations predate #170's LitElement refactor (separate issue needed)
- Docker required for dev/test (PostgreSQL Dev Services)

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Update E2E tests for #170 LitElement DOM structure | M | Med | ChannelPanelE2ETest (19), CaseContextPanelE2ETest (4), proxyMode resize — Shadow DOM selectors needed |
| — | Consume `<blocks-timeline>` for case lifecycle events | M | Med | Follow-on from #170; eventChronologyStrategy from blocks-ui |
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
