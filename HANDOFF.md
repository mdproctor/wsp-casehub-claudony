# HANDOFF — casehub-claudony

## Last Session

Resumed claudony#288. Verified chat view with seeded data, fixed viewport layout, emoji picker positioning (upstream mosaic#308), and message ID mapping. Chat view is functional — channels render, messages display, emoji picker works. Remaining: real-time push integration.

## Branch State

`issue-288-chat-view` on claudony + workspace. **Active** — push integration needed for interactive features.

## What's Done

### Claudony (issue-288-chat-view)
1. **Viewport layout fix** — `flex-direction: column` on chat panel so workbench fills height (workbench uses `height: 100%` on `:host` which doesn't resolve without explicit parent height)
2. **REST-polling connectPush shim** — fetches channels + messages from Qhorus REST API on mount, feeds as dataset snapshot ops. Without this, the workbench shows zero data (it only accepts data via push, not REST).
3. **emoji-picker-element dependency** — added to package.json (required by mosaic's channel-emoji-picker component)
4. **Message ID mapping fix** — Qhorus API returns `id` not `messageId`, `topic` not `topicId`. Wrong field names caused reactions to POST to `/messages/undefined/reactions`.
5. **Debug artifact cleanup** — removed 82 tracked .playwright-mcp/ files, root PNGs, node_modules cache; added to .gitignore
6. **Removed dead connectPush** — the initial `/ws/push` WebSocket attempt that showed "Connection lost"

### casehub-mosaic (main, 2 commits — not pushed)
1. **mosaic#308** — replaced `transform: translateY(-50%)` with `top: -15px` on `.hover-toolbar-anchor` in channel-feed. The transform created a CSS containing block that made `position: fixed` on the emoji picker popover resolve relative to the anchor (not viewport), displacing it ~957px off-screen.
2. **mosaic#308** — added 200ms delay to `_onToolbarLeave` and composed mouseenter/mouseleave events from picker popover, preventing the toolbar from dismissing while the user moves to the picker.

## What Remains

### Push integration (the right fix)
The `agency-conversation-workbench` receives ALL data via push dataset ops — channels, messages, reactions, members, presence. The current REST-polling shim loads data once on mount but doesn't poll for updates. This means:
- **Reactions** — POST succeeds (200 OK) but no visual feedback (reaction badge doesn't appear)
- **Sent messages** — POST works but the message doesn't appear in the feed until reload
- **Member presence** — always shows "No members"
- **Real-time updates** — no live message delivery

**The fix:** Add `casehub-qhorus-push` dependency to claudony. This provides a WebSocket endpoint that delivers dataset ops (channels, messages, reactions, members) to connected clients. The workbench's `connectPush` config would connect to this endpoint instead of the REST-polling shim.

**Steps:**
1. Add `casehub-qhorus-push` to `app/pom.xml`
2. Configure the push WebSocket endpoint path in `application.properties`
3. Update `claudony-chat-panel.ts` `connectPush` to connect to the real WebSocket endpoint using `createEventConnection` from pages-data
4. Verify: reactions appear after clicking emoji, sent messages appear in feed, member count updates

**Reference:** The chat-app (`~/claude/casehub/chat-app`) has a working `connectPush` using `createEventConnection` — copy that pattern.

### Cross-repo pushes needed
- casehub-mosaic: 2 commits on main (mosaic#308 emoji picker fix), not pushed

### Branch hygiene
- 30+ commits with multiple WIP/pause markers — needs squash at work-end
- `.plan` state is `scaffolded`, all tasks marked done

## Resume

1. Push mosaic commits to origin
2. Add casehub-qhorus-push dependency and wire real push connection
3. Verify reactions, sent messages, and presence work
4. Then work-end on #288
