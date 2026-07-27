# Handoff — 2026-07-27

**Head commit (project):** `f29ddd6` — feat(#183) + refactor(#184): session name subdirectory + retire reactive SPIs

## What landed this session

- #183 closed — session name as subdirectory of default working dir
- #184 closed — full reactive→blocking SPI migration: 3 renamed SPI implementations (CaseChannelProvider, WorkerProvisioner, WorkerContextProvider), CaseHubRuntimeCompat deleted, CaseLineageQuery/QhorusCausalLinkResolver/MeshResource/CasehubResource all migrated to blocking. Net -300 lines of reactive machinery
- CI fix: actions/checkout@v4 path restriction (checkout to .ci-deps/ + symlinks)
- Gitignore: untracked .idea/ and .claude/settings.local.json
- Filed pages-ui-components issues: pages#233 (closed), blocks-ui#93, claudony#185
- Garden: GE-20260726-53f142 (ConcurrentHashMap.computeIfAbsent recursive update gotcha)

## State

- main: `f29ddd6`, pushed to origin + upstream, 175 casehub tests pass
- App module has pre-existing test compilation issues from upstream SNAPSHOT drift (not caused by this migration)
- Docker required for dev/test (PostgreSQL Dev Services)

## Hygiene (carried forward)

- 3 unstamped closed branches (#105, #145, #163) — stamp with landing SHA
- Unrecovered artifacts: #161 blog+spec, #168 spec, #172 spec, #175 spec, #180 spec — cherry-pick to workspace main
- 2 stale branches: #151, #156 (25 days)

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #185 | Migrate to pages-ui-components + blocks-ui adoption | L | Med | Blocked on pages#233 (closed) — ready to start |
| #176 | Case browser + task inbox | XL | High | Builds on #175 workbench + commitments |
| #177 | General-purpose chat rooms | M | Med | User-created channels not tied to cases |
| #178 | Reactions + member/presence panels | M | Med | Wire blocks-ui components already available |
| #179 | Responsive layouts for tablet/phone | M | Med | Workbench desktop/tablet/phone modes |
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
