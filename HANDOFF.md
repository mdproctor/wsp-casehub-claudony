# HANDOFF — casehub-claudony

## Last Session

Resumed claudony#288 to fix two frontend bugs preventing Chat and Mesh tabs from working. Both bugs fixed and committed.

## Branch State

`issue-288-chat-view` on claudony + workspace. **Both bugs fixed** — ready for work-end.

## What's Done

### Bug 1: Stale API paths after MeshResource removal (commit 8d2c7e7)
- `claudony-mesh-panel.ts`: `/api/mesh/config` → `/api/claudony/mesh/config`, `/api/mesh/instances` → `/api/claudony/mesh/instances`
- `claudony-mesh-panel.ts`, `channel-panel.ts`, `claudony-workbench.ts`: `/api/mesh/channels/{name}/messages` → `/api/channels/{name}/messages`
- `MeshResourceTest.java`, `MeshResourceInterjectionTest.java`, `ChannelPanelE2ETest.java`, `MeshPanelE2ETest.java`: paths updated to match
- `CLAUDE.md`: updated stale endpoint documentation
- Added vitest test file for mesh panel (tests blocked by pre-existing casehub-packages PnP resolution issue)

### Bug 2: Channel backend prefix filter blocked WebSocket push (commit 7e9d3cc)
- `ClaudonyChannelBackend`: removed `case-` prefix filter from `onChannelInitialised()` — now registers for ALL channels
- Added `onChannelInitialised_registersNonCaseChannels` test — 8/8 pass
- Chat tab's WebSocket push (`/ws/push` via qhorus-push) now receives real-time messages for all channel types (team/, life/, issue/, etc.)

### Compilation fixes (commit 5de702b)
- `ClaudonyMcpToolsTest.java`: 8 `SessionResponse` constructors updated (eidosAgentId field)
- `SessionFederationTest.java`: 1 `SessionResponse` constructor updated
- `OpsServiceTest.java`: 1 `ApplicationInfo` constructor updated (namespace, image, config fields)

## Test State

- `ClaudonyChannelBackendTest`: 8/8 pass
- casehub module: 667/669 pass (2 pre-existing failures: `CaseBrowserServiceTest.getCaseDetail_notFound` NPE, `AgentPoolYamlParserTest.emptyYamlProducesEmptyList` parser)
- app module `@QuarkusTest` classes: CDI deployment error (pre-existing `CbrCaseMemoryStore` unsatisfied — casehub-neocortex SNAPSHOT)
- Frontend vitest: infrastructure broken (pre-existing `@casehubio/pages-tsconfig` PnP resolution)
- Frontend build: passes

## Resume

Ready for work-end on claudony#288. Both handover bugs resolved. The Chat and Mesh tabs should now be functional at runtime — Mesh tab polls correct API paths, Chat tab receives WebSocket push for all channels.

Pre-existing issues (not #288): 2 casehub test failures, CDI deployment error in QuarkusTest, vitest infrastructure.
