---
layout: post
title: "Shipped: the case panel. Found: a fifty-tool cliff."
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
---

The case context panel is built. When a Claudony-managed Claude session has a
case attached, the channel panel now shows the worker's role, elapsed time, a
collapsible list of prior workers from the lineage endpoint, and auto-selects
the case's work channel. Human interjection is the existing send dock, unchanged.
Everything I planned, with two unplanned detours en route.

## The upstream tax

Before the panel code, three separate upstream API changes needed working through.

`WorkerContext.channel` had become `channels` — plural, a `List<CaseChannel>`.
`WorkerContextProvider.buildContext` had grown a new `UUID caseId` parameter
between the existing `workerId` and `WorkRequest`. And Qhorus had made all its
short-form `sendMessage` and `createChannel` overloads package-private (to fix a
tool discovery bug), leaving only the 9-argument `@Tool` signatures public. We
updated every call site across four files.

Each migration was mechanical once found. Finding them was the work — the first
symptom was a runtime `NoSuchMethod` in a test, because the installed Qhorus JAR
had been updated between builds without recompiling Claudony against it. The error
named the old signature, not the new one. `javap -p` on the installed JAR showed
the actual method table; from there the delta was obvious.

## The fifty-tool cliff

Partway through, two tests failed with the same message: `send_input` wasn't in
the tools list. No error, no warning, no compile failure. Just gone. The response
had exactly fifty tools, alphabetically sorted, and `send_input` would have been
tool fifty-one.

The quarkus-mcp-server extension paginates `tools/list` at 50 by default. The
default lives in a `@WithDefault("50")` annotation on a config interface called
`Tools`. With forty-nine Qhorus tools and eight Claudony tools totalling
fifty-seven, anything beyond position fifty is silently dropped. There's no log
message. The count in the response matches the page size exactly, giving nothing
away.

Setting `quarkus.mcp.server.tools.page-size=0` disables pagination and returns all
tools in one response. I filed a separate issue to split the Claudony and Qhorus
endpoints — the correct long-term fix — and moved on.

## The panel

The panel itself was a few hours of JavaScript and CSS. The session fetch that
already ran to populate the workers panel now also stores `sessionCaseId`,
`sessionRoleName`, and `sessionCreatedAt`. When the channel panel opens on a
session with a caseId, it injects a case header above the feed, fetches
`/api/sessions/{id}/lineage` (a new endpoint backed by `CaseLineageQuery`), and
auto-selects `case-{caseId}/work` from the channel dropdown.

One Playwright gotcha: `<option>` elements inside a `<select>` register as
"hidden" in Playwright 1.52 regardless of whether the parent is visible, because
they have zero rendered dimensions. `waitFor()` with default state times out even
when the option is in the DOM. The fix is `setState(WaitForSelectorState.ATTACHED)`.

## The messaging conventions

Before wrapping, the correct defaults for the interjection dock needed fixing.
EVENT content is null by design — telemetry lives in `tool_name`, `duration_ms`,
and `token_count`. The feed renderer now shows those fields for EVENT entries.
The dock default changed from EVENT to COMMAND. And `NormativeChannelLayout`'s
`allowedTypes` declarations are now passed through to Qhorus when channels are
created — the oversight channel actively rejects non-QUERY/COMMAND messages at
the infrastructure level, not just by convention.
