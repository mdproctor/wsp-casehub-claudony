# Reactions, Members, and Presence — Design Spec

**Issue:** casehubio/claudony#178
**Date:** 2026-07-31
**Branch:** epic-b-conversation-maturity

## Problem

The workbench has reaction and member UI wired to blocks-ui components, but
the backend gaps make them non-functional:

1. No member join/leave endpoint — the member list is always empty
2. No auto-join on message post — posting doesn't register membership
3. The mesh panel renders messages as raw text, not via `<channel-feed>` — no reactions, no rich rendering
4. No presence — no way to see who's currently watching a channel
5. Reaction changes aren't pushed via SSE — other users' reactions only appear on page refresh

## Changes

### 1. Member join/leave endpoints on MeshResource

Add `POST /api/mesh/channels/{name}/members` (join) and
`DELETE /api/mesh/channels/{name}/members` (leave).

**Join request:** no body needed — the current user (from `SecurityIdentity`)
is the member. Calls `membershipService.join(channelId, actorId)`.
Returns the `ChannelMembership` record. Idempotent — `join()` returns the
existing membership if already joined.

**Leave request:** `DELETE` with no body. Calls
`membershipService.leave(channelId, actorId)`. Returns 204.

**Error handling:** 404 if channel name not found.

### 2. Auto-join on message post

In `MeshResource.postMessage()`, after `dashboard.sendHumanMessage()`
succeeds, call `membershipService.join(channelId, sender)`. The `sender`
is already computed as `"human:" + securityIdentity.getPrincipal().getName()`.

The `join()` call is idempotent — repeated posts don't create duplicate
memberships. The channel lookup (`channelService.findByName()`) is already
done at the top of the method; use the resolved `Channel` object.

### 3. Mesh panel upgrade — use `<channel-feed>` with reactions

Replace the raw text rendering in `_renderChannel()` with the blocks-ui
`<channel-feed>` component, matching the workbench pattern:

- Fetch timeline via `/api/mesh/channels/{name}/timeline` (already used by workbench)
- Fetch reactions via `/api/mesh/channels/{name}/reactions/batch`
- Pass reactions to `<channel-feed .reactions=${...}>`
- Add a members count indicator to the channel view header

The overview and feed views stay as compact summaries — only the channel
detail view gets the full `<channel-feed>`.

### 4. Presence via SSE subscriber tracking

`ChannelEventBus` already tracks active SSE subscribers per channel via
its `ConcurrentHashMap<String, List<MultiEmitter>>`. It has a
package-private `subscriberCount(String channelName)` method.

Add a `GET /api/mesh/channels/{name}/presence` endpoint that returns
the active SSE subscriber count for the channel. This is a cheap proxy
for "who's watching" — no heartbeats, no new state, no persistence.

Expose `subscriberCount()` as public on `ChannelEventBus` (currently
package-private). The member panel can show an "active" indicator for
channels with subscribers > 0.

### 5. Reaction SSE push

When a reaction is added or removed via `MeshResource.addReaction()` /
`removeReaction()`, tick `ChannelEventBus.emit(channelName)` so SSE
subscribers get notified. They already re-fetch the full timeline on
SSE ticks — the reaction update comes for free.

The channel name is already available as the `@PathParam("name")` in
both endpoints. No CDI event observer or message lookup needed.

## What Does NOT Change

- Workbench reaction wiring — already complete, just needs the SSE push
  to make other users' reactions visible in real-time
- `<blocks-channel-member-panel>` composition in workbench — already
  wired, just needs members to actually be populated
- Qhorus `ReactionService` and `ChannelMembershipService` — used as-is

## Testing

### Unit tests (MeshResource)

- `postMessage_autoJoinsAsChannelMember` — post a message, verify
  `membershipService.join()` was called with the correct actorId
- `joinChannel_returns200_withMembership` — POST to members endpoint,
  verify join called and response contains membership data
- `leaveChannel_returns204` — DELETE to members endpoint
- `joinChannel_unknownChannel_returns404`
- `presence_returnsSubscriberCount` — verify presence endpoint returns
  the `ChannelEventBus` subscriber count
- `addReaction_ticksChannelEventBus` — verify SSE tick fires after
  reaction add
- `removeReaction_ticksChannelEventBus` — verify SSE tick fires after
  reaction remove

### Integration tests

- End-to-end: post message → verify member appears in members list
- End-to-end: add reaction → verify SSE subscriber receives tick

### Frontend (vitest)

- Mesh panel `_renderChannel()` uses `<channel-feed>` component
- Mesh panel fetches reactions for the selected channel

## Out of Scope

- Per-user presence tracking (who specifically is online) — would require
  correlating SSE connections to user identities
- Presence persistence across server restarts
- Member roles and permissions beyond basic join/leave
- Typing indicators
