# Feed Redesign: DESC Global Scan

**Issue:** casehubio/claudony#124
**Date:** 2026-05-22
**Status:** Proposed

---

## Problem

`QhorusDashboardService.getFeed(int limit)` has three architectural bugs:

1. **New messages invisible.** Per-channel slot bucketing (`perChannel = max(5, limit / numChannels)`) with `ORDER BY id ASC` returns the *oldest* N messages from each channel. Once any channel exceeds its budget, new messages never appear in the feed.

2. **N+1 queries.** One `channelService.listAll()` plus one `messageStore.scan()` per channel. The store already supports cross-channel global scans (`MessageQuery` with `channelId = null`).

3. **In-memory sort on a weaker key.** After fetching messages ordered by ID from the store, the merge re-sorts by `created_at` string comparison. Since ID order = insertion order = `created_at` order (for normal writes), this is redundant work.

Additionally, `JpaMessageStore.scan()` applies LIMIT in Java (`results.subList(0, limit)`), not in SQL. Every scan loads all matching rows into heap before truncating. This is masked today by the per-channel filter bounding result size. A global scan without SQL-level LIMIT is an OOM risk at scale.

## Design

### Approach: DESC global scan — sliding window of newest N messages

Replace per-channel bucketing with a single global query: `ORDER BY id DESC LIMIT N`. The 100 most recent messages globally, regardless of channel. New messages always appear (highest IDs). Old messages fall off the bottom. The client does replace-semantics (no cursor state needed).

Two queries total:
1. `channelService.listAll()` → build `Map<UUID, String>` channel name lookup
2. `messageStore.scan(MessageQuery.recent(limit))` → global DESC scan with SQL LIMIT

Each result entry is enriched with `"channel": channelName` from the lookup map.

### Why not cursor-based pagination?

Cursor (`?after=<id>`) would require the client to switch from replace-semantics to accumulate-semantics — tracking cursor state, appending new messages, managing a display window. The dashboard client (`dashboard.js`) currently does `panel.update(data)` which replaces the data model wholesale and re-renders. Cursor would require a non-trivial client rewrite for the primary SSE delivery path.

The sliding window (DESC LIMIT N) fixes the visibility bug with zero client changes. If a future consumer needs global incremental catch-up, `MessageQuery` already supports `afterId` — adding it to `getFeed` later is a one-line change.

### Client rendering compatibility

- **FeedView:** `feed.slice(0, 50)` — renders feed in received order. DESC gives newest-first, correct for a "recent activity" feed.
- **ChannelView:** `feed.filter(m => m.channel === selected)` — filters then renders. DESC gives newest-first within a channel. Correct.
- **OverviewView:** `feed.slice(0, 5)` — takes the first 5. With DESC, these are the 5 newest messages. Correct for a pulse check.

No client changes needed. No ordering assumptions violated.

### `events()` SSE impact

`events()` calls `getFeed(100)` as part of its full-state push on each tick. After the redesign, this returns the 100 most recent messages (DESC) instead of the oldest N per channel (ASC bucketed). Strictly better: new messages appear, old messages fall off. Same method signature, same client consumption pattern.

---

## Changes

### Qhorus (cross-repo)

**1. `JpaMessageStore.scan()` — apply SQL LIMIT**

When `query.limit() != null`, use `Message.find(jpql, params).page(0, query.limit()).list()` instead of `Message.list(jpql, params)` with Java subList. Consistent with the pattern already used by `QhorusMcpTools`. When `limit == null`, keep `Message.list(...)` for unbounded scans.

**2. `MessageQuery` — add `descending` flag**

Add `boolean descending` field to `MessageQuery.Builder`. Default: `false` (backwards compatible).

`JpaMessageStore.scan()`: use `ORDER BY id DESC` when `descending == true`.

`InMemoryMessageStore.scan()`: reverse the stream before applying limit when `descending == true`.

**3. `MessageQuery` — add `recent(int limit)` factory method**

```java
public static MessageQuery recent(int limit) {
    return new Builder().limit(limit).descending(true).build();
}
```

Clear intent: "most recent N messages globally, no channel scoping."

**4. `QhorusDashboardService.getFeed(int limit)` — rewrite**

```java
public Uni<List<Map<String, Object>>> getFeed(int limit) {
    int effectiveLimit = Math.min(Math.max(limit, 1), 200);
    return channelService.listAll().flatMap(channels -> {
        Map<UUID, String> nameMap = channels.stream()
                .collect(Collectors.toMap(ch -> ch.id, ch -> ch.name));
        return messageStore.scan(MessageQuery.recent(effectiveLimit))
                .map(msgs -> msgs.stream()
                        .map(m -> {
                            Map<String, Object> entry =
                                    new HashMap<>(entityMapper.toTimelineEntry(m));
                            entry.put("channel", nameMap.getOrDefault(
                                    m.channelId, m.channelId.toString()));
                            return entry;
                        })
                        .toList());
    });
}
```

**5. V11 migration: index on `(channel_id, id)`**

```sql
-- V11__add_message_channel_id_index.sql
CREATE INDEX idx_message_channel_id ON message (channel_id, id);
```

Covers `getTimeline` per-channel scans: `WHERE channel_id = ? AND id > ? ORDER BY id ASC`. Currently runs as a sequential scan in PostgreSQL (no index on `channel_id`). Pre-existing issue, right time to fix.

### Claudony

**6. `MeshResource` — no production code changes**

`feed()` calls `dashboard.getFeed(limit)` — same signature. `events()` calls `dashboard.getFeed(100)` — same call. Both get better results.

**7. `MeshResourceTest` — update stale comment and add coverage**

- Update `meshFeed_limitTruncates` comment (remove reference to per-channel bucketing formula).
- Add test: insert messages across 2 channels, verify the newest messages are in the feed and the oldest are not when limit < total.
- Add test: insert messages, verify feed includes entries from all channels (no channel is starved).

---

## Scaling analysis

| Operation | Before | After |
|-----------|--------|-------|
| `getFeed(100)`, 10 channels | 11 queries (1 channel list + 10 per-channel scans), each a full-table scan | 2 queries (1 channel list + 1 PK backward range scan with SQL LIMIT) |
| `getTimeline(ch, after, 50)` | 1 query, sequential scan filtered by channel_id | 1 query, composite index range scan |
| 1M messages in table | 10 × sequential scan of 1M rows + Java truncation | 1 backward PK scan, stops after 100 rows |
| Heap pressure | loads all per-channel results into heap before truncating | loads only `limit` rows from DB |
| Cost complexity | O(N × M) where N = channels, M = table size | O(log M + K) where K = limit |

The PK B-tree backward scan for `ORDER BY id DESC LIMIT 100` positions at the end of the tree and reads 100 entries. Cost does not grow with table size.

---

## Deferred items (out of scope)

These are tracked as separate GitHub issues:

1. **`events()` SSE is timer-driven full-state push** — should become event-driven (push on message arrival, not on 30-second tick). Already tracked as casehubio/claudony#131.

2. **Global feed cursor (`?after=<id>`)** — not needed for the dashboard snapshot use case. If a future consumer needs incremental catch-up on the global feed, `MessageQuery` already supports `afterId` — adding it is a one-line change. Defer until a concrete use case exists.

---

## Sequence gap note (fleet mode)

`message_seq` with `allocationSize=50` means IDs can have gaps between allocation batches. In a multi-JVM deployment with a shared database, two sessions could allocate overlapping ranges. Claudony is currently single-node; even in fleet mode, each node has its own H2 database. When/if fleet moves to shared PostgreSQL, the sequence is globally serialised by the database. Non-issue for current and planned deployment topologies.
