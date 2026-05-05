---
layout: post
title: "Phase 10: Sessions Don't Last Forever"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [expiry, cdi, tmux, session-management]
excerpt: "Three CDI-pluggable expiry policies distinguish an idle shell from an autonomously working agent, collected at startup via @Any Instance so new policies are just new beans."
---

## Idle is not one thing

Sessions accumulate. The cleanup logic wants to be simple — expire after 7 days —
but "idle" means different things depending on what's running. A Claude Code session
autonomously working through a long refactor shouldn't be killed because no human
typed anything. A shell prompt sitting untouched for 8 days probably should be.

I wanted three expiry policies, switchable per session, with a global default.
The CDI design: an interface with a `name()` method, three `@ApplicationScoped`
implementations, and a registry that collects them all at startup via
`@Any Instance<ExpiryPolicy>`. New policies are just new beans — no registration code.

The registry builds its map in the constructor:

```java
this.policies = StreamSupport.stream(all.spliterator(), false)
    .collect(Collectors.toMap(ExpiryPolicy::name, p -> p));
```

Most developers would reach for `@Named` + `NamedLiteral` for runtime dispatch.
The `name()` approach is cleaner: the strategy describes itself, the registry
discovers it, and startup validates the configured default immediately rather
than at first call.

**user-interaction** checks `session.lastActive()` — bumped whenever a user opens
a WebSocket, sends input, or calls `POST /{id}/input`. **terminal-output** checks
tmux's own activity timestamp. **status-aware** checks what process is in the
foreground — if it's not a shell, the session never expires regardless of how
old `lastActive` is.

## What tmux didn't tell us

I brought Claude in to implement the terminal-output policy, and it came back
with a problem: `tmux display-message -p "#{pane_activity}"` always returns blank
for detached sessions. No attached client, no pane activity tracking. We found
`#{window_activity}` instead — server-side, reflects actual output, and conspicuously
absent from the tmux man page.

Also: argument order in tmux 3.6a is strict. `-t target -p format` works. The
reverse fails with "too many arguments" — no indication of which argument it
objects to.

## The remove() that wasn't atomic

After the scheduler was working, Claude caught something in review. `registry.remove()`
sat outside the try/catch:

```java
try {
    expiryEvents.fire(event);
    tmux.killSession(session.name());
} catch (Exception e) { ... }
registry.remove(session.id()); // ← wrong
```

CDI synchronous events propagate observer exceptions. If the fire throws, the
catch runs, but `registry.remove()` still executes — session gone from the registry,
tmux session alive. Moving `remove()` inside the try block makes the whole
operation atomic from the error perspective.

The scheduler fires the event before killing the tmux session, deliberately.
The WebSocket observer sends `{"type":"session-expired"}` to connected clients on
a virtual thread — a structured message instead of a raw disconnect. Kill happens
after.

273 tests. Per-session policy override is wired through `CreateSessionRequest`,
`Session`, and `SessionResponse` — the CaseHub WorkerProvisioner will use it when
it provisions sessions.
