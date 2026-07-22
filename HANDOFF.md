# Handoff — 2026-07-22

**Head commit (project):** `6f938e7` — fix: quarkus-junit5 → quarkus-junit (Quarkus 3.31+ rename)

## What landed this session

- #180 closed — promoted claudony-task-panel, claudony-correlation-panel, claudony-artifact-panel to `@casehubio/blocks-ui-channel-activity` as channel-task-panel, channel-correlation-panel, channel-artifact-panel
- Cross-repo: 6 blocks-ui commits (commitment module, 3 panel components, showcase demos, version 0.3.1)
- CommitmentRecord + conversion functions moved from Claudony's channel-adapter to blocks-ui commitment.ts
- Claudony: 792 lines deleted, 21 added — pure extraction
- Design review: 12 issues raised, 11 verified, 1 accepted — drove commitment.ts separation, ResolvedArtifact type, CSS fallback stripping, internal import fix
- Also landed: quarkus-junit5 → quarkus-junit rename (Quarkus 3.31+ parent enforcer)

## State

- main: `6f938e7`, pushed to origin + upstream, 597 Java tests + 19 vitest pass
- blocks-ui: `8bccf88` on main, pushed, `@casehubio/blocks-ui-channel-activity@0.3.1` published
- Docker required for dev/test (PostgreSQL Dev Services)
- 1 pre-existing Java test failure: CaseEngineRoundTripTest (engine SNAPSHOT instability)

## Hygiene (carried forward)

- 3 unstamped closed branches (#105, #145, #163) — stamp with landing SHA
- Previous session's unrecovered artifacts (#161 blog+spec, #168 spec, #172 spec) — cherry-pick to workspace main

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #176 | Case browser + task inbox | XL | High | Builds on #175 workbench + commitments |
| #177 | General-purpose chat rooms | M | Med | User-created channels not tied to cases |
| #178 | Reactions + member/presence panels | M | Med | Wire blocks-ui components already available |
| #179 | Responsive layouts for tablet/phone | M | Med | Workbench desktop/tablet/phone modes |
| — | Consume `<blocks-timeline>` for case events | M | Med | Follow-on from #170 |
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
