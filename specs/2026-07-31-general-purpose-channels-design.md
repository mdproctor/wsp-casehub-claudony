# General-Purpose Channels — Design Spec

**Issue:** casehubio/claudony#177
**Date:** 2026-07-31
**Branch:** epic-b-conversation-maturity

## Problem

All Qhorus channels in Claudony are created through the CaseHub engine lifecycle
(`ClaudonyCaseChannelProvider.openChannel()`). There is no way to create channels
outside this path.

Additionally, `ClaudonyChannelBackend` filters on the `"case-"` prefix in
`onChannelInitialised()`, so only case-scoped channels get backend registration
and `post()` callbacks (which drive `ChannelEventBus`). SSE delivery itself is
unaffected — it uses 500ms polling via `dashboard.getTimeline()` which reads the
message store directly — but the backend filter prevents non-case channels from
participating in `ChannelEventBus` and any future migration from polling to push.

This blocks general-purpose chat rooms, issue-scoped discussions, collaboration
channels, and any future context that needs channels without a case lifecycle.

## Design Direction

Channels are the universal communication primitive. What changes is how they're
scoped and surfaced depending on context — a case gets its channels, an issue
gets its channels, a collaboration gets its channels. The chat capabilities
(feed, input, reactions, threading, commitments, correlation) are the same
everywhere. The REST endpoint enables both system-initiated channel creation
(e.g. a work context provisioning its channels on setup) and user-initiated
creation (e.g. team communication rooms per #177). Who calls the endpoint is
orthogonal to the endpoint's design.

Claudony is evolving toward being both an application and a collection of
reusable components. Channel management belongs in Qhorus (the generic mesh),
not in Claudony. Claudony adds case-specific augmentation on top.

## Architectural Layering

```
┌─────────────────────────────────────┐
│  Claudony App (claudony-app)        │
│  MeshResource: thin REST proxy      │
│  ClaudonyChannelBackend: SSE        │
│  delivery for all channels          │
└──────────────┬──────────────────────┘
               │ CDI inject
┌──────────────┴──────────────────────┐
│  Claudony CaseHub (claudony-casehub)│
│  ClaudonyCaseChannelProvider:       │
│  case-scoped channel creation       │
│  via engine SPI (unchanged)         │
└──────────────┬──────────────────────┘
               │ CDI inject
┌──────────────┴──────────────────────┐
│  Qhorus (casehub-qhorus)           │
│  ChannelService: channel CRUD       │
│  ChannelGateway: backend fan-out    │
│  MessageService: messaging          │
│  Future: native REST API (#387)     │
└─────────────────────────────────────┘
```

No new Claudony abstraction layer. `ChannelService` (Qhorus) is the channel
management layer. `MeshResource` proxies it via REST until Qhorus ships its
own REST API (casehubio/qhorus#387).

## Changes

### 1. Channel creation endpoint on MeshResource

Add `POST /api/mesh/channels` as a thin pass-through to `ChannelService.create()`.

**Request body:**

```json
{
  "name": "team/engineering",
  "description": "Engineering team discussion",
  "semantic": "APPEND",
  "allowedTypes": ["QUERY", "COMMAND", "DONE", "DECLINE", "EVENT"],
  "deniedTypes": null
}
```

- `name` — required, Qhorus slug-path validated
- `description` — optional
- `semantic` — optional, defaults to `APPEND`
- `allowedTypes` — optional `Set<MessageType>`, defaults to null (all types permitted)
- `deniedTypes` — optional `Set<MessageType>`, defaults to null (no types denied)

Only these 5 fields are exposed. Advanced configuration (rate limits, barrier
contributors, allowed writers, admin/reviewer instances, protocols, connector
bindings, delivery tracking, space assignment) is not needed at creation time and
can be set via post-creation mutation methods on `ChannelService`.

**Namespace guard:** reject names starting with `case-` — those are managed by
`ClaudonyCaseChannelProvider` and the engine SPI. Return 400 with a message
explaining the reserved prefix.

**Response:** 201 Created with a `ChannelDetail` body (consistent with
`GET /api/mesh/channels` which returns `List<ChannelDetail>`), or 409 Conflict
if the name already exists. Convert the `Channel` to `ChannelDetail` via
`entityMapper.toChannelDetail(channel, 0, Optional.empty())` — message count is
0 for a newly created channel.

**Implementation:** inject `ChannelService` and `QhorusEntityMapper`, construct a
`ChannelCreateRequest` via the builder, call `channelService.findOrCreate()`.
The `ChannelInitialisedEvent` fires automatically from `ChannelCreateHelper`
(resolved in qhorus#254), so backend registration happens without explicit
`gateway.initChannel()`. Use `FindOrCreateResult.wasCreated()` to determine 201
vs 409.

**Idempotency:** `ChannelService.create()` is not idempotent (GE-20260529-88b7b6).
Use `channelService.findOrCreate()` which handles the TOCTOU race internally —
it catches `PersistenceException` on unique-constraint violation and retries
`findByName()`. Return 201 when `wasCreated()` is true, 409 when false.

**Error handling:**
- Invalid channel name (format, length, UUID-shaped) → 400 with validator message
- Name starts with `case-` → 400 ("'case-' prefix is reserved for case-scoped channels")
- Invalid `semantic` value → 400
- Invalid `allowedTypes` or `deniedTypes` values → 400
- `allowedTypes`/`deniedTypes` overlap → 400

Catch `IllegalArgumentException` from `ChannelSlugValidator.validateSlugPath()`,
`ChannelSemantic.valueOf()`, and `MessageType.valueOf()` and map to 400. Follow
the same try-catch pattern used by `postMessage()` for `MessageType` parsing.

**Garden gotchas to respect:**
- Use `ChannelCreateRequest` not the removed 9-arg overload (GE-20260613-7b7ae1)
- Include response types (DONE, DECLINE) in `allowedTypes` if human interaction
  is expected (GE-20260519-28967d)

### 2. Broaden ClaudonyChannelBackend SSE delivery

Remove the `startsWith("case-")` guard in `onChannelInitialised()`. The backend
should register for all channels so `post()` callbacks fire and `ChannelEventBus`
is notified regardless of channel naming convention. This is required for any
future migration from polling-based SSE to push-based SSE, and ensures all
channels participate equally in the backend infrastructure.

**Current code (line 50):**
```java
if (!event.channelName().startsWith("case-")) return;
```

**New code:**
```java
// Register for all channels — SSE delivery is not case-specific
```

Remove the filter entirely. The `ChannelInitialisedEvent` pattern
(GE-20260529-d1397c) handles both startup recovery and runtime registration
for all channels automatically. No performance concern — `registerBackend`
is a map put per channel, and the SSE fan-out only fires when messages are
posted to channels that have active browser subscribers.

### 3. Follow-up: Qhorus native REST API

casehubio/qhorus#387 tracks adding `POST/GET/DELETE /api/channels` directly
in Qhorus. Once that lands, Claudony's `MeshResource` proxy endpoint becomes
redundant and can be removed or redirected.

## What Does NOT Change

- `ClaudonyCaseChannelProvider` — case-scoped channel creation via engine SPI
  stays as-is. Note: `createQhorusChannel()` has a pre-existing redundant
  `gateway.initChannel()` call (already fired internally by `channelService.create()`
  via `ChannelCreateHelper`). Cleanup tracked in #193.
- `NormativeChannelLayout` / `SimpleLayout` — case channel specs unchanged
- Frontend components — `<channel-feed>`, `<channel-input>`, `<channel-nav>`
  from blocks-ui are already composable and context-agnostic
- `claudony-mesh-panel.ts` — already lists all channels from
  `/api/mesh/channels` with no prefix filtering
- All existing MeshResource endpoints (timeline, feed, messages, reactions,
  commitments, topics, members) — already work with any channel by name

## Testing

### Unit tests

- `MeshResourceChannelCreationTest` — valid creation, missing name validation,
  duplicate name 409, default semantic, allowedTypes parsing
- `ClaudonyChannelBackendTest` — update existing test
  `onChannelInitialised_nonCaseChannel_noRegistration`: change assertion from
  `verifyNoInteractions(gateway)` to `verify(gateway).deregisterBackend(...)` and
  `verify(gateway).registerBackend(...)`, reflecting that the backend now registers
  for all channels. The original test encoded a deliberate design choice (case-only
  registration); this spec intentionally reverses that choice.

### Integration tests

- `MeshResourceIntegrationTest` — end-to-end: create channel via REST, post
  message, verify SSE delivery works for the new channel
- Verify existing case-scoped channel tests still pass (no regression)

### E2E tests

- Verify the mesh panel displays channels created via the new endpoint
- Verify messages posted to non-case channels appear in the mesh panel feed

## Out of Scope

- UI for channel creation (#190)
- Channel deletion/archival endpoint (follows with qhorus#387)
- Channel member management (#191)
- Namespace conventions for non-case channels — each future context defines
  its own, like `life/` does for life-domain channels (#192)
