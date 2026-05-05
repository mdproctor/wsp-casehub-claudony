---
layout: post
title: "Phase 9: Speaking Into the Mesh"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [qhorus, mesh, dashboard, human-in-the-loop]
excerpt: "A fixed send dock in all three mesh views lets humans post into any active channel without switching panels, turning the read-only mesh observer into an active participant."
---

## The observer becomes a participant

The Mesh panel has been read-only since Phase 8. You could watch channels,
see who was online, follow the message feed. But you couldn't say anything.

That changes now. A fixed dock sits below the scrollable panel content —
visible in all three views, always in reach. Channel select, message type
select, textarea. Enter to send. Immediate poll on success so the message
appears in the timeline without waiting for the next cycle.

I wanted this in all three views, not just Channel view. The use case is
watching the Feed and wanting to drop a note into whatever channel is
active — you shouldn't have to switch views for that. Clicking a channel
item in Overview or Feed now selects it as the send target. The dock follows.

## The endpoint

The backend is one new method in `MeshResource`:
`POST /api/mesh/channels/{name}/messages`. Body: `{"content": "...", "type": "status"}`.

```java
try {
    MessageResult result =
        qhorusMcpTools.sendMessage(name, "human", type, req.content(), null, null);
    return Response.ok(result).build();
} catch (ToolCallException e) {
    Throwable cause = e.getCause();
    if (cause instanceof IllegalArgumentException) {
        return Response.status(404).entity(cause.getMessage()).build();
    }
    return Response.status(409).entity(
        cause != null ? cause.getMessage() : e.getMessage()).build();
}
```

The `ToolCallException` handling is worth noting. `QhorusMcpTools` has
`@WrapBusinessError` on the class — every `IllegalArgumentException` and
`IllegalStateException` thrown inside a `@Tool` method gets rewrapped into
`ToolCallException(Throwable cause)` before it exits the CDI proxy. A bare
`catch (IllegalArgumentException)` misses channel-not-found entirely. The
distinction between 404 and 409 comes from inspecting `e.getCause()`.

I'd already noted this in the Phase 8 entry as a gotcha for `MeshResource.timeline()`.
The difference here is that we needed the distinction, not just a fallback.

## Two bugs that CSS and JS let through

The dock textarea and channel select were invisible at first run. The CSS used
`var(--bg-secondary)` for their backgrounds — a variable that doesn't exist in
this project's `:root`. When a CSS custom property can't be resolved, the browser
substitutes the initial value for the property. For `background`, that's
`transparent`. Dark panel, transparent inputs — nothing visible.

The correct variable is `var(--bg)`, same as the dialog inputs elsewhere. Claude
caught it during review. One-line fix in two places.

The JS had a subtler gap. Channel names from Qhorus agents were being embedded
directly in `onclick` attribute strings:

```javascript
onclick="meshPanel.selectChannel('${escapeHtml(ch.name)}')"
```

`escapeHtml()` covers HTML entities — `&`, `<`, `>`, `"`, `'`. It does not touch
JS metacharacters: `;`, `)`, backslash. A channel name containing `');alert(1);//`
breaks out of the string cleanly. HTML escaping is not JS string escaping.

The fix is the pattern you'd reach for in any modern codebase but probably wouldn't
if you were just trying to get something working: `data-channel` attribute plus
`querySelectorAll` binding after `innerHTML` is set. The DOM decodes the HTML
entities back to characters before your event listener reads `el.dataset.channel`.
The JS engine never sees the value inside a string literal.

## The test infrastructure

Running `@QuarkusTest` with both Hibernate ORM and `quarkus-qhorus`'s
`hibernate-reactive-panache` on the classpath hit a wall: H2 has no reactive
driver. The test context threw a HibernateReactive boot failure — all tests in
the run, not just Qhorus-related ones.

The fix is a single property:

```properties
%test.quarkus.datasource.reactive=false
```

This tells Quarkus to skip reactive datasource init in the test profile.
Hibernate Reactive falls back to JDBC mode. The pattern comes from Qhorus's
own test properties — not documented anywhere in the Quarkus guides.

246 tests passing. The human is now a participant in the mesh, not just a reader.
