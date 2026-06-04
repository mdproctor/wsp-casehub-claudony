---
layout: post
title: "What the SSE Stream Was Hiding"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [qhorus, sse, concurrency, resteasy]
---

The global mesh dashboard SSE stream had been silently delivering nothing.

`GET /api/mesh/events` — the endpoint that feeds live channel and instance data to the dashboard — returned 200, correct `text/event-stream` content-type, and fired `onmessage` events. The browser received them. `JSON.parse(e.data)` threw on every single one, silently, because `e.data` wasn't JSON.

It was `"data: {...}"` — a string that starts with the literal prefix `data:`.

The cause: `MeshResource.events()` manually constructed SSE frames as `"data: " + json + "\n\n"`. RESTEasy Reactive wraps every item emitted from a `Multi<String>` SSE endpoint automatically as `data: <item>\n\n`. Add them together and you get `data: data: {...}\n\n\n\n` on the wire. The EventSource spec strips the first prefix and hands the remainder to the application. The remainder was `"data: {...}"`. Not JSON.

What made it hard to catch: nothing indicated failure. The SSE connection stayed alive. Events arrived on schedule. The only signal was that the dashboard never updated — and that could have been anything. We found it by writing a test that actually reads the first frame off the wire and calls `JSON.parse` on it. The test failed immediately.

The fix is one line: remove the manual framing and let RESTEasy own it.

```java
// Before — double-frames on the wire
return "data: " + mapper.writeValueAsString(payload) + "\n\n";

// After — RESTEasy adds data: <item>\n\n automatically
return mapper.writeValueAsString(payload);
```

The same rule applies to error fallback strings — `"data: {}\n\n"` becomes `"{}"`.

The other interesting bug was in `ChannelEventBus.removeSubscriber()`. When the last subscriber for a channel cancels, we prune the map entry:

```java
list.remove(emitter);
if (list.isEmpty()) subscribers.remove(channelName, list);
```

`ConcurrentHashMap.remove(key, value)` uses `equals()` — not reference equality. For a `CopyOnWriteArrayList`, `list.equals(list)` is always true regardless of contents. Between `isEmpty()` returning true and `remove()` executing, a concurrent subscribe can add to the same list object — and the removal still succeeds, orphaning the new subscriber. Currently harmless; will cause doubled tick delivery when we replace the 500ms poll with direct event-bus push.

The fix:

```java
subscribers.computeIfPresent(channelName, (key, list) -> {
    list.remove(emitter);
    return list.isEmpty() ? null : list;
});
```

`computeIfPresent` holds the per-bucket lock for the duration. Returning `null` atomically removes the entry. The window disappears.

We also wired Commitment tracking for engine COMMAND messages. `postToChannel()` had been passing `null` for `correlationId` since the beginning, which meant the Qhorus obligation state machine never fired for any engine-dispatched command. The engine embeds `correlationId` in the content JSON; we now extract it with a guarded Jackson parse for COMMAND and QUERY types and pass it through to `ReactiveMessageService.send()`. The content-coupling is deliberate but marked — the right fix is adding `correlationId` as a first-class parameter to the `postToChannel()` SPI, which is tracked separately.
