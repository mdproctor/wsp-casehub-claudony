---
layout: post
title: "Tests That Make Things Real"
date: 2026-04-15
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [fleet, playwright, e2e-testing, proxy]
excerpt: "Fleet Phase 2 adds the fleet sidebar and WebSocket proxy bridge, and the first Playwright test immediately finds a four-month-old bug hiding in displayName() that no one had noticed."
---

## Fleet Phase 2: the fleet you can see

Fleet Phase 1 gave us the peer mesh — `PeerRegistry`, session federation, the REST API. But you couldn't see any of it from the browser. The dashboard still showed only local sessions. That changed today.

We added the fleet sidebar panel: peer health dots (●green, ●amber, ●red), circuit breaker state, source tag (config/manual/mdns), last seen timestamp. Session cards picked up instance badges and stale indicators — a clock icon plus "last seen N min ago" when the peer is down but we have cached data. The "Add Peer" modal lets you register a new Claudony instance by URL.

The PROXY WebSocket bridge landed too. `ProxyWebSocket` at `/ws/proxy/{peerId}/{sessionId}/{cols}/{rows}` opens an upstream WebSocket to the peer and pipes frames bidirectionally — Vert.x `HttpClient` for the upstream, full `@OnOpen`/`@OnTextMessage`/`@OnBinaryMessage`/`@OnClose` wiring. Browser never touches the remote Claudony directly; everything goes through the local instance.

The code review on this caught something I almost missed: `vertx.createHttpClient()` in `@OnOpen` creates a new HTTP client on every WebSocket connection. That's a pool leak. We moved to a singleton `@PostConstruct`-initialised client.

## The argument for browser tests now

I've been putting off Playwright. The mental model was: add it when the UI stabilises, or when we get a regression.

The problem with that logic is visible in retrospect: the UI already had a bug I hadn't noticed, and I only found it because we added tests. The `displayName()` function in `dashboard.js` was still stripping `remotecc-` from session names — the old project name, replaced months ago. Every session in the dashboard was showing its full tmux name with the `claudony-` prefix. It just looked slightly off; I never registered it as a bug.

The first Playwright test that created a session and asserted the displayed name failed immediately. Not because of anything we built — because of a four-month-old bug sitting in plain sight.

I decided to add Playwright before fixing the resize limitation specifically because of this: I wanted the infrastructure in place before we had a reason to need it, not after the first regression.

## The architecture: Java over Node.js

The choice was straightforward. Claudony is a pure Java/Maven project. Adding Node.js as a second runtime just for tests means two tools to install, two commands to run, CI configuration for both. Playwright for Java runs under the existing `-Pe2e` Maven profile — same `mvn test -Pe2e` command as the Claude CLI E2E tests. No new runtime dependency.

The Java API is slightly more verbose than Node.js Playwright. That's acceptable.

## What headless breaks

Two things went wrong that I didn't anticipate.

First: `page.request()` in Playwright Java doesn't inherit headers set via `page.setExtraHTTPHeaders()`. I'd set the API key header on the page to authenticate all dashboard requests. But `page.request().post(url, ...)` goes through the `BrowserContext` API request context, not the page — no headers. Every REST call from test code returned 401. Fix: add the API key header explicitly to each `page.request()` call.

Second: `fitAddon.fit()` is a no-op in headless Chromium. The xterm.js container has no visible dimensions — the computed size is the same 80×24 as the terminal default, so `terminal.onResize` never fires. I'd planned to use `page.waitForRequest()` to capture the resize event. It timed out. The fix was `page.route("**/resize**", ...)` to intercept the request, plus `page.evaluate("window._xtermTerminal.resize(100, 30)")` to force `onResize`. That required exposing the terminal object on the global window, which I gated behind `window.__CLAUDONY_TEST_MODE__` set by `page.addInitScript()`.

The code review flagged that I'd exposed `window._xtermTerminal` unconditionally — it was in every production page load. A `terminal.paste()` call from a browser extension would have sent keystrokes to the tmux session. Fixed before commit.

## Four architecture verification tests before anything else

One design I'm glad we included: `PlaywrightSetupE2ETest` — four tests that verify the test infrastructure itself before any functional tests run.

1. Chromium launches
2. Server is reachable on port 8081
3. Auth header injection works (protected endpoint returns 200)
4. Unauthenticated context is actually blocked (401, not silently open)

That fourth test matters most. Without it, a misconfigured server that accepts all requests would let every other test pass for the wrong reason.

## The resize fix

PROXY mode had a known limitation: resize commands went to the local `/api/sessions/{sessionId}/resize`, which returned 404 silently (session lives on the peer, not locally). Tmux on the peer never received the resize.

The fix: new endpoint `POST /api/peers/{peerId}/sessions/{sessionId}/resize` on `PeerResource`. It looks up the peer, builds a `PeerClient` with fleet key auth, calls the peer's own resize endpoint, and propagates the status. `terminal.js` detects the `proxyPeer` URL param and routes the resize call there instead of the local endpoint.

The Playwright test for this verifies the URL, not just that a request was made:

```java
assertThat(resizeRequest.url())
    .contains("/api/peers/" + proxyPeerId + "/sessions/fake-session/resize")
    .contains("cols=100")
    .contains("rows=30");
```

The cols/rows assertion is the important one. A bug that strips the query parameters would produce a silent resize to 0×0 in tmux.

212 Java tests, 13 Playwright browser tests. The resize limitation is closed.
