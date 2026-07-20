# Handoff — 2026-07-20

**Head commit (project):** `f186aa8` — feat(#175): rich conversation integration — workbench + enriched backend

## What landed this session

- #175 closed — workbench view for case-bound sessions composing terminal, channel-nav, feed, input, task/correlation/artifact panels; enriched MeshResource with full Qhorus dispatch model (replies, commitments, correlation, artifacts); terminal-controller extraction; conditional fleet/workbench routing with code-splitting
- Cross-repo: 4 Qhorus commits (findByChannel SPI, timeline enrichment, sendHumanMessage widening, @Inject delegate fix, Flyway V37→V38 renumber)
- Deferred issues filed: #177 (general chat rooms), #178 (reactions/presence), #179 (responsive layouts), #180 (promote panels to blocks-ui), #181 closed (Flyway V37 conflict)
- Garden entry: GE-20260720-da57c7 (InMemoryReactiveCommitmentStore delegate @Inject gotcha)

## State

- main: `f186aa8`, pushed, 597 Java tests + 22 vitest pass
- Docker required for dev/test (PostgreSQL Dev Services)
- Qhorus commits on main (4 commits: findByChannel, timeline enrichment, sendHumanMessage, Flyway fix) — committed to active branches, now on Qhorus main

## Hygiene (carried forward)

- 4 unrecovered artifacts on closed branches (#161 blog+spec, #168 spec, #172 spec) — cherry-pick to workspace main
- 3 unstamped closed branches (#105, #145, #163) — stamp with landing SHA
- 2 stale branches (#151, #156) — 19 days without commits

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #176 | Case browser + task inbox | XL | High | Builds on #175 workbench + commitments |
| #177 | General-purpose chat rooms | M | Med | User-created channels not tied to cases |
| #178 | Reactions + member/presence panels | M | Med | Wire blocks-ui components already available |
| #179 | Responsive layouts for tablet/phone | M | Med | Workbench desktop/tablet/phone modes |
| #180 | Promote panels to blocks-ui | S | Low | After proven in Claudony |
| — | Consume `<blocks-timeline>` for case events | M | Med | Follow-on from #170 |
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
