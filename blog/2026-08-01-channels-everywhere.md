---
title: "Channels Everywhere — From Case-Scoped to Universal"
date: 2026-08-01
tags: [claudony, qhorus, channels, conversation, architecture]
---

Claudony started with a simple channel model: the CaseHub engine spins up a case, the case opens channels, agents talk on them. Three channels per case — work, observe, oversight. Clean, purposeful, and completely closed. If you weren't a case, you didn't get channels.

That worked until it didn't.

The moment we started thinking about team communication, issue-scoped discussions, and collaboration rooms, the architecture had a question to answer: is a channel a case concept, or a universal primitive? The answer shapes everything downstream — where channels are created, who can see them, how the UI discovers and renders them.

## The Three Chokepoints

The case-scoping lived in exactly three places:

1. **`ClaudonyCaseChannelProvider`** — the only channel creation path, driven by the engine SPI
2. **`ClaudonyChannelBackend.onChannelInitialised()`** — a `startsWith("case-")` filter that excluded non-case channels from the SSE delivery infrastructure
3. **`channel-panel.ts`** — auto-selecting `case-{caseId}/work` on load

Everything else — Qhorus's `ChannelService`, `MessageService`, the `MeshResource` REST endpoints, the blocks-ui `<channel-feed>` component — was already channel-agnostic. The case-scoping was a convention in three files, not an architectural constraint.

## Removing the Filter

The `ClaudonyChannelBackend` fix was two lines deleted. The guard clause `if (!event.channelName().startsWith("case-")) return;` became nothing. Every channel now registers with the gateway, every channel participates in `ChannelEventBus`, every channel can push events to browser subscribers.

The interesting part wasn't the deletion — it was discovering that the SSE delivery system was already polling-based via `dashboard.getTimeline()`, which reads the message store directly. The backend filter wasn't blocking SSE delivery at all. It was blocking `ChannelEventBus` participation — which matters for any future migration from polling to push. The design review caught this factual error in the spec and forced a correction. Getting the problem statement right matters more than getting the solution right.

## Qhorus Delivers the REST API

The original plan had us building a `POST /api/mesh/channels` proxy endpoint in `MeshResource` — inject `ChannelService`, construct a `ChannelCreateRequest`, return a `ChannelDetail`. Standard pass-through code.

Then Qhorus shipped its own `ChannelResource` (#387) with `POST /api/channels`, and we discovered something: Quarkus auto-mounts JAX-RS resources from indexed dependency JARs. The endpoint was already live in Claudony without a single line of proxy code. Six integration tests confirmed it worked — creation, idempotency, validation, `allowedTypes` parsing, default semantic, and mesh panel visibility.

This eliminated the proxy pattern entirely. The platform provided the capability; the application just used it. That's the right layer separation.

## The Full Stack: Reactions, Members, Presence

With the channel infrastructure open, we wired the conversation features that were half-built. The workbench already composed `<channel-feed>`, `<channel-input>`, and `<blocks-channel-member-panel>` from blocks-ui — but the member panel showed an empty list (no join endpoint), reactions from other users never appeared (no SSE push), and presence was absent.

The fixes were surprisingly small:

- **Member join/leave** — two REST endpoints delegating to Qhorus's `ChannelMembershipService`
- **Auto-join on post** — one line after `sendHumanMessage()`: `membershipService.join(channelId, sender)`
- **Reaction SSE push** — one line in each reaction endpoint: `channelEventBus.emit(name)`
- **Presence** — expose `ChannelEventBus.subscriberCount()` as a REST endpoint

The mesh panel upgrade was the largest piece — replacing raw text rendering with `<channel-feed>`, adding reaction batch-fetching, and wiring the member panel. But it was composition, not invention. The components existed; they just needed data.

## Namespace Conventions

With channels no longer tied to cases, naming matters. We documented five namespace prefixes — `case-{uuid}/`, `life/`, `team/`, `issue/`, `collab/` — each owned by a specific context. The `case-` prefix is enforced at the REST API level; the others are conventions that future contexts will adopt as they arrive.

The key insight: positive conventions are more useful than negative guards. Telling people "don't use `case-`" prevents collisions but doesn't help them name their channels. Telling them "team channels use `team/engineering`" does.

## What This Enables

General-purpose channels are the foundation for everything in Epic B's remaining scope. Debate channel integration (#158) needs channels that aren't case-scoped. Issue-scoped discussions need `issue/` channels. The case browser (#176) needs to surface channels from multiple contexts in a single view.

The architecture is ready. The next phase is the contexts that create and use these channels — each bringing its own lifecycle, its own layout decisions, and its own conventions. The channel primitive is universal now; what makes each context distinct is how it uses channels, not whether it can have them.
