# Design: MeshResource Reactive Refactor — QhorusDashboardService

**Date:** 2026-05-20  
**Issue:** claudony#119  
**Branch:** epic-gateway-reliability  

---

## Problem

`MeshResource` (the dashboard REST API) calls `ReactiveQhorusMcpTools` directly. This is wrong for two reasons:

1. `ReactiveQhorusMcpTools` is the MCP protocol dispatch layer for Claude Code. It carries `@WrapBusinessError` interceptor semantics that wrap business exceptions in `ToolCallException`, forcing `MeshResource` to catch and unwrap them. Internal consumers must not be exposed to this.

2. The existing boundary rule prescribed "inject `ReactiveChannelService` / `ReactiveMessageService` directly" as the fix — but this is also wrong. `ReactiveChannelService` returns raw entities without message counts. Building `ChannelDetail` (with `messageCount`) requires injecting `ReactiveMessageStore` from the store layer directly into a REST resource — bypassing the service layer and worse than the original violation.

The real diagnosis: Qhorus has two tiers (entity services and MCP dispatch) and is missing a third — a dashboard/consumer integration service at the right abstraction level.

---

## Solution

### New: `QhorusDashboardService` (Qhorus runtime)

**Package:** `io.casehub.qhorus.runtime.dashboard`

A new `@ApplicationScoped` service that encapsulates all composition and mapping logic needed by dashboard consumers. It is the correct integration point for any UI or REST resource consuming Qhorus data.

**Injections:**
- `ReactiveChannelService` — channel CRUD
- `ReactiveInstanceService` — instance lookup + capability tags  
- `ReactiveMessageService` — message send (commitment tracking included)
- `ReactiveMessageStore` — message counts and timeline queries
- `ObjectMapper` — injected once (fixes the inline `new ObjectMapper()` in the existing `toTimelineEntry`)

**Operations:**

| Method | Reactive pipeline |
|--------|------------------|
| `listChannels()` → `Uni<List<ChannelView>>` | `channelService.listAll()` → parallel `messageStore.countByChannel()` per channel → `Uni.join()` |
| `listInstances()` → `Uni<List<InstanceView>>` | `instanceService.listAll()` → parallel `findCapabilityTagsForInstance()` per instance → `Uni.join()` |
| `getTimeline(name, afterId, limit)` → `Uni<List<Map<String,Object>>>` | channel lookup by name → `messageStore.scan(MessageQuery.poll(...))` → map via private `toTimelineEntry()`. Unknown channel → `List.of()` (no exception). |
| `getFeed(limit)` → `Uni<List<Map<String,Object>>>` | `channelService.listAll()` → parallel per-channel timeline queries by channel ID → flatten + sort. Parallel, not sequential. |
| `sendHumanMessage(channelName, sender, type, content)` → `Uni<HumanMessageResult>` | channel lookup → paused check → `messageService.send(ch.id, sender, type, content, null, null, null, null, HUMAN)` → map `Message` to `HumanMessageResult` |

**Exception semantics:**
- Channel not found → `IllegalArgumentException` — callers map to 404
- Channel paused → `IllegalStateException` — callers map to 409
- Unknown channel in `getTimeline()` → `Uni.createFrom().item(List.of())` — consistent with current behavior

**DTOs:** `QhorusDashboardService` defines its own response records (`ChannelView`, `InstanceView`, `HumanMessageResult`) with JSON field names identical to the existing `QhorusMcpToolsBase` types — the dashboard JS sees no change. These are dashboard-scoped types. Migrating them to `casehub-qhorus-api` is tracked in qhorus#175.

**Mapping methods:** `toChannelDetail()` and `toTimelineEntry()` are private methods of `QhorusDashboardService`. Deduplication with `QhorusMcpToolsBase` is tracked in qhorus#176.

**`feed()` improvement:** The current `MeshResource.feed()` queries channel timelines sequentially in a loop. `QhorusDashboardService.getFeed()` fans out queries in parallel via `Uni.join().all()` — correct and faster.

---

### Simplified `MeshResource` (Claudony)

**Single injected service:** `QhorusDashboardService`. Also: `ClaudonyConfig` (config endpoint + SSE interval), `ObjectMapper` (SSE snapshot serialisation), `SecurityIdentity` (postMessage sender).

`ReactiveQhorusMcpTools` injection removed entirely. `TIMEOUT` constant removed. `ToolCallException` import removed.

**All non-config, non-SSE endpoints return `Uni<T>`.** `@Blocking` removed from all REST methods.

| Endpoint | Before | After |
|----------|--------|-------|
| `GET /config` | sync `MeshConfig` | unchanged — pure config read |
| `GET /channels` | `@Blocking List<ChannelDetail>` | `Uni<List<ChannelView>>` |
| `GET /instances` | `@Blocking List<InstanceInfo>` | `Uni<List<InstanceView>>` |
| `GET /channels/{name}/timeline` | `@Blocking` + `ToolCallException` catch | `Uni<List<Map<String,Object>>>` |
| `GET /feed` | `@Blocking` sequential loop | `Uni<List<Map<String,Object>>>` |
| `POST /channels/{name}/messages` | `@Blocking` + `ToolCallException` unwrap | `Uni<Response>` with `onFailure()` chains |
| `GET /events` (SSE) | sequential awaits in worker lambda | `Uni.combine().all()` parallel snapshot |

**`postMessage` type validation stays in `MeshResource`** — it is REST input validation, not a Qhorus concern. `MessageType.valueOf()` + `VALID_HUMAN_TYPES` check → 400 on invalid type, before calling the service.

**`/events` SSE:** `Uni.combine().all().unis(listChannels, listInstances, getFeed)` → parallel snapshot. No `runSubscriptionOn` needed — service methods are already reactive.

---

### Qhorus testing module

**`InMemoryChannelStore`** extended to implement both `ChannelStore` and `ReactiveChannelStore` from the same `LinkedHashMap`. Reactive methods wrap with `Uni.createFrom().item()`.

**`InMemoryMessageStore`** extended to implement both `MessageStore` and `ReactiveMessageStore` from the same backing structure.

This ensures `@BeforeEach channelStore.put()` is visible to both blocking and reactive code paths. `MeshResourceInterjectionTest` requires no changes.

---

### Platform and protocol updates

**PLATFORM.md — Capability Ownership:** New row added:

> Dashboard read/write API (rich views: channel with message count, instance with capability tags, timeline mapping, feed) | `casehub-qhorus` | `QhorusDashboardService`

**PLATFORM.md — Boundary Rules:** The existing rule "inject `ReactiveChannelService` / `ReactiveMessageService` directly" is replaced with a three-tier model:

1. **Dashboard/UI consumers** needing composed views → inject `QhorusDashboardService`
2. **Service-layer integrations** needing raw entity access → inject `ReactiveChannelService` / `ReactiveMessageService` directly
3. **Reactive event-driven integrations** → implement `ChannelBackend` or `MessageObserver` SPI

**New protocol file:** `docs/protocols/casehub/qhorus-consumer-integration-pattern.md` — captures the three-tier model and explains why raw entity services are the wrong choice for dashboard consumers.

---

## Testing

### Qhorus (new)
Unit tests for `QhorusDashboardService` using the dual-interface in-memory stores:
- `listChannels()` — empty → `[]`; populated → correct `messageCount`
- `listInstances()` — empty → `[]`; instance with tags → correct `InstanceView`
- `getTimeline()` — unknown channel → `[]`; known channel → ordered entries; `afterId` cursor works
- `getFeed()` — empty → `[]`; fan-out with correct `"channel"` tag; sort order correct
- `sendHumanMessage()` — unknown channel → `IllegalArgumentException`; paused → `IllegalStateException`; success → correct `HumanMessageResult`

### Claudony (unchanged)
`MeshResourceTest` and `MeshResourceInterjectionTest` are black-box RestAssured tests. They pass without modification — full behavioral equivalence.

---

## Follow-on issues

| Issue | Description |
|-------|-------------|
| qhorus#175 | Move `ChannelDetail`, `InstanceInfo`, `MessageResult` to `casehub-qhorus-api` |
| qhorus#176 | Extract `toTimelineEntry` + `toChannelDetail` to shared `QhorusEntityMapper` (also fixes inline `new ObjectMapper()`) |

---

## Sequence

1. Qhorus: extend `InMemoryChannelStore` + `InMemoryMessageStore` (dual interface)
2. Qhorus: add `QhorusDashboardService` + unit tests
3. Qhorus: install updated SNAPSHOT in Claudony (`mvn install`)
4. Claudony: simplify `MeshResource`
5. Claudony: run full test suite — verify all existing tests pass
6. Both repos: update PLATFORM.md boundary rule + add protocol file