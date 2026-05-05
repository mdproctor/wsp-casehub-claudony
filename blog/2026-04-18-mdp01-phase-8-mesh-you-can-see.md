---
layout: post
title: "Phase 8: The Mesh You Can See"
date: 2026-04-18
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [qhorus, mcp, mesh, quarkus, dashboard]
---

## Fourteen weeks of foundation, three hours to embed

Qhorus is now embedded in Claudony. Forty-seven tools at one `/mcp` endpoint — Claudony's eight session management tools plus Qhorus's thirty-nine agent communication tools, all auto-registered by `quarkus-mcp-server-http` without a line of wiring code.

Getting there required a Quarkus upgrade first. We were on 3.9.5; Qhorus was built on 3.32.2. Not a small jump. The WebAuthn extension was the main casualty — the entire Vert.x WebAuthn layer was replaced with `webauthn4j`, which meant `WebAuthnUserProvider` had new method signatures, `Authenticator` was gone, and the reflection-based `WebAuthnPatcher` we'd written to allow Apple passkeys was patching internals that no longer existed. We deleted it and rewrote `CredentialStore` against the new API. `rest-client-reactive-jackson` was renamed to `rest-client-jackson`. `quarkus-junit5` to `quarkus-junit`. These things compound.

The upgrade shook out one more gotcha: old `credentials.json` files stored `aaguid` as a plain hex string without dashes — not a UUID. The new code called `UUID.fromString()` on load and would throw, silently locking users out. We added graceful skipping with a clear warning message rather than crashing.

## The error handling debate

Before the Mesh panel we hardened the MCP tool layer. Eight `@Tool` methods with zero error handling — if the server was down, Claude received an unhandled exception in unpredictable format. We added a two-tier catch to every method:

```java
} catch (WebApplicationException e) { return serverError(e); }
  catch (Exception e)               { return connectError(e); }
```

`serverError()` maps status codes to actionable messages (404 suggests `list_sessions`, 409 is a conflict). `connectError()` handles unreachable servers. Claude reads them like any other tool output.

One non-obvious thing: `QhorusMcpTools` has `@WrapBusinessError({IllegalArgumentException.class})` on the class. This means any `IllegalArgumentException` thrown inside a `@Tool` method gets wrapped into `ToolCallException` before it exits the CDI proxy. Catching `IllegalArgumentException` in a caller — like `MeshResource.timeline()` — silently misses it. Must catch both.

## The panel

The Mesh panel is a collapsible right panel alongside the session grid — the third column the dashboard always had room for. Three switchable views: Overview (channels + presence + recent messages), Channel (focused timeline with dropdown), Feed (chronological cross-channel stream).

The data layer is thin by design. `MeshResource` injects `QhorusMcpTools` and calls its existing methods — `listChannels()`, `listInstances(null)`, `getChannelTimeline()`. No new queries, no duplicate N+1 concerns. The MCP tools already solved that.

Claude caught an XSS issue in the view renderers: every user-controlled field — channel name, sender, message content, instance ID — was being inserted into `innerHTML` without escaping. We added `escapeHtml()` before the first Playwright test ran.

240 tests passing. The Mesh panel is live.
