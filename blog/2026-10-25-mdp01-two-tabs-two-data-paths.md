---
layout: post
title: "Two Tabs, Two Data Paths"
date: 2026-10-25
entry_type: note
subtype: diary
projects: [casehubio/claudony]
tags: [chat, mesh, websocket, debugging]
series: issue-288-chat-view
---

# Two Tabs, Two Data Paths

The Chat tab and the Mesh tab in Claudony both display Qhorus channel data. Same channels, same messages, same backend. But they use completely different data paths — and both are broken for different reasons.

## What we found

The Mesh panel polls three REST endpoints every three seconds: `/api/mesh/instances`, `/api/mesh/config`, and `/api/channels/feed`. Two of those URLs are wrong. The panel calls `/api/mesh/instances` but the actual endpoint lives at `/api/claudony/mesh/instances` — the Claudony MCP domain prefix was dropped when the panel was written. Same for `/api/mesh/config`. Both return 404. The channel feed at `/api/channels/feed` works, but the panel shows "No active channels" because the instance list comes back empty (404 swallowed by the catch block).

The Chat panel has a deeper problem. It relies entirely on WebSocket push at `/ws/push` for channel and message data. No REST fallback. When you click the Chat tab, the `<agency-conversation-workbench>` component opens a WebSocket, subscribes to dataset topics (`chat:channels`, `chat:messages`, etc.), and waits for the server to push snapshots. If the WebSocket connection fails — or if the channel doesn't have a registered backend — no data arrives. The channel tree stays empty. "No messages yet" is permanent.

The interesting part: posting messages works fine. The Chat panel's send button calls `POST /api/channels/{id}/messages` via REST. The message lands in the database. The qhorus#495 AUTO_REGISTER policy fires and registers the sender as a mesh instance. But the message never appears in the feed because the WebSocket push channel for general-purpose channels was never wired.

## Why general channels are invisible

`ClaudonyChannelBackend` registers itself as a WebSocket broadcaster backend — but only for channels whose name starts with `case-`. This was correct when Claudony only had case-scoped channels. A general channel like `team/testing-mesh` doesn't match the prefix filter, so it never gets a backend registration, and the WebSocket broadcaster never pushes its messages.

The data is there. The REST endpoints serve it. The WebSocket pipeline doesn't know about it.

## The fix

Two separate changes:

1. **Mesh panel URLs** — change `/api/mesh/instances` to `/api/claudony/mesh/instances` and `/api/mesh/config` to `/api/claudony/mesh/config`. Straightforward.

2. **Chat panel WebSocket delivery** — either broaden `ClaudonyChannelBackend` to register for all channels (not just `case-` prefixed), or add a REST polling fallback when the WebSocket connection fails. Broadening the backend is the cleaner fix — it means general channels get the same push delivery that case channels already have.

Both need tests before the fix. The Mesh panel URL change is verifiable with a vitest mock. The WebSocket delivery fix needs an E2E test: create a channel, post a message, verify the Chat tab renders it.
