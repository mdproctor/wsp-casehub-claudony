---
layout: post
title: "What Polling Was Hiding"
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [push, sse, polling, architecture, identity]
excerpt: "Switching from polling to server-initiated push exposes three guarantees polling provided for free — restart resilience, catch-up, and identity — that each require deliberate design in a push model."
---

The Qhorus gateway will replace the channel panel's 3-second polling loop with server-initiated push. Lower latency, cleaner architecture, messages from any backend automatically flowing to the panel. The design is right.

I spent today stress-testing it from Claudony's side, and polling turns out to have been doing a lot of quiet work.

## The silent guarantees

Polling is resilient by construction. Miss a cycle and nothing is lost — the next one catches up. Restart the server and the next poll from any panel picks up where it left off. No registration, no re-establishment. It just works.

Push requires deliberate care in each of these cases. When Claudony's backend endpoint is temporarily unreachable, the gateway either retries or silently drops the message. When Claudony restarts, backend registrations created at channel-open time are gone — channels that were active before the restart are now dark, with no error surfaced to anyone watching the panel.

The restart case is straightforward: extend `ServerStartup.bootstrapRegistry()` to re-register backends for all active case sessions, the same pattern that already rebuilds the session registry from tmux on startup. Transient delivery failures need either client-side catch-up (`timeline?after=lastId`, already supported) or gateway-side retry. Both are tractable. Neither is automatic.

## The identity gap

The more interesting concern is what gets lost through Claudony's proxy.

When a human interjection is posted, it flows from the browser to Claudony's REST endpoint and from there to the channel. The gateway sees one sender: Claudony's instanceId. The channel history records "claudony." The actual user — whoever was authenticated when they pressed send — is invisible.

For a single-user deployment this doesn't matter. For anything with multiple people able to observe and interject, the audit trail becomes useless. Two engineers on the same case, both interjecting, are indistinguishable. Fixable — attach human identity as a supplement before the gateway persists the message — but it requires thinking about it before the gateway API hardens.

## The SPI couldn't say what it meant

One finding was structural. `CaseChannelProvider.postToChannel()` has no `messageType` parameter, so every implementation picks a fixed default. Claudony sends `"status"` for everything: a reply to a query, a completion signal, a directive — all identical in the channel history. The Qhorus speech-act taxonomy (QUERY, COMMAND, RESPONSE, STATUS, HANDOFF, DONE) is doing real work, but the SPI above it provides no way to use it.

This is a casehub-engine fix, not a Claudony one. A nullable `MessageType` parameter — null defaults to STATUS for backwards compatibility — is small and correct. The sort of thing that gets harder to retrofit as more implementations ship.

## None of it blocks #77

Every one of these concerns is a clean retrofit onto the end-to-end path, not a prerequisite. Build #77 with polling, switch to push when the gateway lands, layer the maturity work on after.

I spent the session documenting this — seven issues across three repos, each with a sequencing note making the ordering explicit. The next session picks up with the analysis already in place.
