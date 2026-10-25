# HANDOFF — casehub-claudony

## Last Session

qhorus#495 implemented and installed — mesh participation enforcement with OPEN/AUTO_REGISTER/ENFORCE modes. Claudony configured with `AUTO_REGISTER` as global default. Verified working: posting a message auto-registers the sender as a mesh instance. Blog written and published.

Resumed claudony#288 to wire up the feature. Discovered two frontend bugs preventing the Chat and Mesh tabs from working.

## Branch State

`issue-288-chat-view` on claudony + workspace. **Not closed** — two frontend bugs remain.

## What's Done

- qhorus#495 landed on `issue-495-enforce-mesh-participation` branch (4 commits, not yet merged to main)
- Claudony: `casehub.qhorus.mesh.participation-policy=AUTO_REGISTER` in application.properties
- Auto-registration verified via API: sender `human-mark` and `human` both auto-registered on first post
- Blog entry: `docs/blog/2026-10-25-mdp01-who-posted-that.md` in qhorus repo, published to personal + casehub sites

## Two Bugs to Fix (TDD)

### Bug 1: Mesh panel wrong API paths

**Symptom:** Mesh tab shows "No active channels" despite registered instances and channels existing.

**Root cause:** `claudony-mesh-panel.ts` calls `/api/mesh/instances` and `/api/mesh/config` — both return 404. The actual endpoints are `/api/claudony/mesh/instances` and `/api/claudony/mesh/config`.

**File:** `app/src/main/webui/src/components/claudony-mesh-panel.ts` line ~217

**Fix:** Change `/api/mesh/instances` → `/api/claudony/mesh/instances` and `/api/mesh/config` → `/api/claudony/mesh/config` in the `_poll()` method.

**Test:** Add vitest test that mocks fetch and verifies correct URLs. E2E: `MeshPanelE2ETest` — create a channel, post a message (triggers auto-register), verify Mesh tab shows the channel and instance.

### Bug 2: Chat panel relies on WebSocket push — no REST fallback

**Symptom:** Chat tab shows "Connection lost" banner, empty channel tree, "No messages yet" — even though `GET /api/channels/feed` returns 3 messages via REST.

**Root cause:** The Chat tab (`claudony-chat-panel.ts` → `<agency-conversation-workbench>`) uses **only** WebSocket push at `/ws/push` for channel/message data. No REST fallback. Two sub-issues:

1. **WebSocket connection itself may not be establishing** — the `ws/push` endpoint requires the Qhorus WebSocket broadcaster module (`casehub-pages-push`). Check if it's on the classpath and configured.

2. **`ClaudonyChannelBackend` only registers for `case-` prefixed channels** (line ~48). General channels like `team/testing-mesh` never get WebSocket push delivery — the broadcaster only pushes messages for channels with a registered backend.

**Files:**
- `app/src/main/java/.../server/ClaudonyChannelBackend.java` — remove or broaden the `case-` prefix filter
- `app/src/main/webui/src/components/claudony-chat-panel.ts` — the `connectPush` callback
- Check if `casehub-pages-push` dependency is in `app/pom.xml`

**Fix options:**
- Option A (recommended): Remove the `case-` prefix filter from `ClaudonyChannelBackend` so ALL channels get backend registration and WebSocket push delivery.
- Option B: Add REST polling fallback in `claudony-chat-panel.ts` when WebSocket fails (like the Mesh panel does).

**Test:** E2E: create channel via REST, post message, verify Chat tab shows the channel in the tree and the message in the feed.

## Cross-Repo State

| Repo | Branch | State |
|------|--------|-------|
| qhorus | `issue-495-enforce-mesh-participation` | Complete, not merged to main. `mvn install` done. |
| claudony | `issue-288-chat-view` | Active, 2 bugs remaining |
| qhorus workspace | `issue-495-enforce-mesh-participation` | Spec, plan, blog committed |
| claudony workspace | `issue-288-chat-view` | This handover |

## Resume

Fix Bug 1 (mesh panel URLs — 15 min), then Bug 2 (WebSocket push for general channels — investigate `ClaudonyChannelBackend` filter and `ws/push` endpoint). TDD both. Then work-end on claudony#288.

After claudony#288 closes: work-end on qhorus#495 (merge to main, push).
