# HANDOFF — casehub-claudony

## Last Session

Implemented #288 (conversation-workbench — shared chat UI). Extracted `<conversation-workbench>` from chat-app into mosaic channel-activity as a reusable LitElement with callback-based config. Wired it in claudony as a Chat tab in the fleet home. Verified working in browser — channels render, messages display, channel switching works.

## Branch State

`issue-288-chat-view` on claudony + workspace. **Not closed** — blocked on qhorus#495.

## What's Done

- `conversation-workbench.ts` (450 lines) + `swipe-controller.ts` + `responsive.ts` in mosaic — landed on mosaic `main`
- `claudony-chat-panel.ts` — thin wrapper configuring workbench with claudony auth/APIs
- Chat tab in fleet home (`app.ts`)
- Startup fixes: THEME_CSS Lit tag, NoOpProvisionerConfigRegistry, ConnectorsBeans exclusion, 65 portal resolutions, index.html path, tree-shake guard

## Blocked On

- **casehubio/qhorus#495** — `sendMessage` currently posts directly to `/api/channels/{id}/messages`, bypassing mesh. After #495 lands (mesh enforcement on message posting), update `claudony-chat-panel.ts` sendMessage callback to use the mesh endpoint.

## Filed Issues

| Issue | Repo | Status |
|-------|------|--------|
| #292 | claudony | CI startup smoke test |
| #659 | platform | agent-gate config property — fixed |
| #165 | connectors | Twilio/WhatsApp config crash — fixed |
| #495 | qhorus | Mesh enforcement on message posting — open, blocks this branch |

## Resume

Wait for qhorus#495 → update sendMessage path → work-end.
