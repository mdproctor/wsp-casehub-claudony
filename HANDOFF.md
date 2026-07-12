# Handoff — 2026-07-13

**Head commit (project):** `977782a` — fix(ci): use GH_PAT org secret for npm package auth

## What landed this session

- #168 closed — broadcaster migration (FleetMessageRelayObserver → casehub-qhorus-postgres-broadcaster, H2 → PostgreSQL)
- CI fixed: `file:` local deps → published `@casehubio` npm packages, Node.js build step in ci.yml, maven-enforcer-plugin guard, `quarkus-opentelemetry` dep, GoalKind interface migration, GH_PAT for cross-repo package auth
- qhorus#323 filed and closed (allowedWriters NPE)
- blocks-ui#39 created (claudony component migration epic)
- CI is green

## State

- main: `977782a`, pushed, CI green
- 591 tests pass (16 core + 176 casehub + 399 app)
- Docker required for dev/test (PostgreSQL Dev Services)
- npm packages resolved from GitHub Packages (CI uses GH_PAT org secret)

## Next candidates

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| blocks-ui#39 | Pages + blocks-ui adoption — consume available components, promote channel-panel/worker-panel | L | Med | 10 implemented blocks-ui components available; pages published to GitHub Packages |
| #158 | Debate channel integration | M | Med | Blocked on drafthouse#71 |
