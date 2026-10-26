# HANDOFF — casehub-claudony

## Last Session

Resumed claudony#288. Fixed two original bugs (stale API paths, channel backend prefix filter) plus discovered and fixed several additional issues across three repos during Playwright-driven testing.

## Branch State

`issue-288-chat-view` on claudony + workspace. **Paused** — layout UX not yet verified.

## What's Done

### Claudony (issue-288-chat-view, 7 commits)
1. **Stale API paths** — `/api/mesh/*` corrected to `/api/claudony/mesh/*` and `/api/channels/*` across 5 frontend + 4 test files + CLAUDE.md
2. **Compilation fixes** — `SessionResponse` (eidosAgentId), `ApplicationInfo` (namespace, image, config) constructor mismatches
3. **Channel backend filter** — removed `case-` prefix from `ClaudonyChannelBackend.onChannelInitialised()` so all channels get WebSocket push
4. **Sender field** — added `sender` to message POST in mesh-panel (from config actorId), channel-panel and workbench (`'human'`)
5. **Type uppercase** — `.toUpperCase()` on MessageType in all message POST paths
6. **Viewport layout** — CSS flex chain from body through `#app > [data-component-type=tabs] > [data-slot] > [data-component-type=host-panel]` so chat fills viewport (NOT YET VISUALLY VERIFIED)

### Qhorus (main, 2 commits)
1. `@ServerExceptionMapper` methods made `public` for dev-mode classloader compatibility
2. `actorType` defaults to `HUMAN` when not provided in `ChannelCore.postMessage`

### casehub-mosaic (main, 3 commits)
1. `restBase` resolved lazily via getter in `ReactionController` and `MessagingController`
2. Emoji picker popover on react button (was directly firing REACT with no emoji)
3. Emoji picker uses `position: fixed` with JS viewport clamping

## What Remains

### Layout UX (not verified)
- CSS flex chain committed but not tested in Playwright (session was paused before verification)
- Emoji picker positioning committed but not verified at full viewport size
- Chat view should fill entire browser viewport and resize with it

### Cross-repo pushes needed
- Qhorus: 2 commits on main, not pushed
- casehub-mosaic: 3 commits on main, not pushed

### Pre-existing issues (not #288)
- 2 casehub module test failures (CaseBrowserServiceTest NPE, AgentPoolYamlParserTest parser)
- CDI deployment error in `@QuarkusTest` classes (CbrCaseMemoryStore unsatisfied)
- Vitest infrastructure broken (pages-tsconfig PnP resolution)

## Resume

1. Verify viewport layout fix in Playwright (resize to 1280x800, check chat fills)
2. Verify emoji picker positioning at full viewport
3. Push qhorus and mosaic commits
4. Then work-end on #288
