---
layout: post
title: "A Denial Is Not a Restriction"
date: 2026-06-04
entry_type: note
subtype: diary
projects: [claudony, casehub-qhorus]
tags: [casehub, qhorus, channel-design, speech-act-theory]
---

The oversight channel in Claudony's three-channel layout was supposed to be the governance channel — the place where agents pause and ask humans for approval before doing something consequential. The problem: it was configured with `allowedTypes = {QUERY, COMMAND}`, and that restriction blocked most of what a governance conversation actually requires.

RESPONSE, DECLINE, STATUS, DONE, FAILURE, HANDOFF — all blocked. So was EVENT, which meant Watchdog deadline alerts couldn't get through either. The channel existed. The commitment lifecycle didn't work on it.

Worse: the mesh framework spec I'd written was internally inconsistent. One section said oversight takes `QUERY,COMMAND`. An example in the same document showed a human sending `RESPONSE` on the oversight channel. The documentation and the implementation were both wrong, just in different directions.

---

I initially proposed null. If the channel needs all obligation-carrying types, just open it — same as the work channel. The distinction between work and oversight is purpose and convention, not type enforcement.

I was pushed back on this. *Are you sure you mean null?*

That question was right. Null conflates oversight with work at the type layer. There's no machine-readable difference between them if both are unrestricted. And there's a real invariant available: every type in the oversight channel should be part of the commitment lifecycle — it should create or discharge an obligation. EVENT doesn't. EVENT is observer-only telemetry, not delivered to agent context, excluded from default `pollAfter` results. An EVENT on the oversight channel is invisible to every participant in the governance conversation.

So the right answer wasn't null. It was "everything except EVENT" — but expressed as a denial, not a restriction.

---

That's where `deniedTypes` came from. The observe channel has `allowedTypes = {EVENT}`: only telemetry, no obligations. The oversight channel needed the inverse: `deniedTypes = {EVENT}`. No telemetry on the governance channel.

The symmetry matters. Both channels express genuine architectural invariants — not labels, not descriptions, but constraints that must never be violated. The work channel has no such invariant, so it stays null on both dimensions.

Qhorus didn't have `deniedTypes` at all. We specced the change — adding the column, updating the policy, threading it through the service API and MCP tools — and the qhorus team shipped it. Then we wired the Claudony side: the `ChannelSpec` record gets a `deniedTypes` field, `NormativeChannelLayout` sets oversight to `allowedTypes=null, deniedTypes={EVENT}`, and `ClaudonyReactiveCaseChannelProvider` passes it through to the 10-arg `ReactiveChannelService.create()`.

One thing surfaced during the qhorus spec review that was genuinely non-obvious. The blocking `ChannelService` correctly routes through `ChannelCreateRequest`, so overlap validation fires at construction time. The reactive `ReactiveChannelService` did inline entity construction and never touched `ChannelCreateRequest`. The D1 argument — "compact constructor makes invalid state unrepresentable" — was true for the blocking path and false for the reactive one. Claudony uses the reactive path. The fix was making `ReactiveChannelService` also construct `ChannelCreateRequest` before entering the Panache transaction. Construction is pure; the transaction context doesn't matter.

---

The alternative we seriously considered was channel roles: GOVERNANCE, TELEMETRY, COORDINATION — a Qhorus-level concept where the infrastructure encodes domain semantics. Semantically cleaner at the API surface, but it would mean Qhorus knows what "governance" means. A different consumer with a different idea of what belongs in a governance conversation would be stuck with Qhorus's definition. The right answer is that Qhorus is a substrate — it enforces what callers tell it to enforce, without needing to understand why.

The oversight channel now works. Agents can ask humans for approval. Humans can respond, decline, ask clarifying questions, delegate, extend deadlines. The commitment lifecycle closes correctly. EVENT doesn't leak through.

The architectural statement — `allowedTypes` and `deniedTypes` are hard constraints, not labels — is now a protocol in the casehub parent repo. It needs to be said explicitly because the temptation to set `allowedTypes` as a description of what a channel is "for" is strong, and that temptation is exactly what created the original bug.
