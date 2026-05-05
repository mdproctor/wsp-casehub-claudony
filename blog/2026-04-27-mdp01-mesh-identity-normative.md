---
layout: post
title: "Phase 13: Two Identity Systems and a Normative Mesh"
date: 2026-04-27
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [qhorus, normative, casehub, mesh, architecture, channel-panel]
excerpt: "CaseHub uses worker roles while Claudony tracks session UUIDs — WorkerSessionMapping bridges the two identity systems, with a documented MVP limitation when concurrent same-role workers share a case."
---

## The name problem hiding an identity problem

`WorkResultSubmitter.complete(caseId, workerId, results)` looks up workers
by name in the case definition. Our provisioner was generating a UUID as
the worker name. So the lookup always failed with `IllegalArgumentException`.

The fix sounds simple: use `context.taskType()` — the role name from the case
definition — as the `Worker` name. But that reveals a deeper mismatch. CaseHub
thinks about workers by *role* ("code-reviewer"). Claudony tracks them by
*instance UUID* (the tmux session ID). These are different identity systems
running in parallel, and nothing was bridging them.

`WorkerSessionMapping` is that bridge: a small `@ApplicationScoped` bean with
two maps. One keyed by `caseId:roleName` for precise lookup, one keyed by
`roleName` alone for callers who don't have the caseId — like
`onWorkerCompleted`, which the engine calls without passing the case. The fallback
is why concurrent same-role workers across different cases would confuse it. That's
a documented MVP limitation (#93) that requires the engine to start passing caseId
in `WorkResult` — a one-line upstream change that can't happen without coordination.

What's interesting here isn't the fix — it's that the bug exposed an assumption
baked into the design from day one. CaseHub models identity around roles.
Claudony models it around instances. Any system that bridges them needs to
carry both, and needs to know when each is available.

## Channel badges that mean something

I wanted the Qhorus channel panel on the session page to reflect the normative
system, not just show chat bubbles. So the type badges carry meaning: blue for
QUERY (creates an obligation to inform), teal for RESPONSE (discharges it),
orange for COMMAND, green for DONE. EVENT messages are visually dimmed — they
have no deontic footprint, nothing is owed as a result of them.

```css
.msg-badge.msg-query   { background: rgba(0,122,204,.2); color: #4fc3f7; }
.msg-badge.msg-event   { background: rgba(80,80,80,.3);  color: #666; }
```

Human-posted messages get a warm amber sender colour. The interjection dock
defaults to EVENT for casual observations — because posting EVENT creates no
obligation, and most human comments in an ongoing agent conversation are
observations, not directives.

The visual choices aren't aesthetic decisions. They're a rendering of the
normative layer: obligation-creating acts are vivid, telemetry is quiet,
terminal acts (DONE, FAILURE, DECLINE, HANDOFF) are slightly faded because
they close something rather than open it.

## A channel layout that knows what layer it's serving

The biggest design work was the mesh framework spec:
`docs/superpowers/specs/2026-04-27-claudony-agent-mesh-framework.md`.

The core idea: channel purposes should reflect normative *roles*, not topics.
The default layout opens three channels per case:

- `work` — obligation-carrying acts between agents (QUERY, COMMAND, RESPONSE, HANDOFF, DONE)
- `observe` — EVENT only; pure telemetry with no deontic footprint
- `oversight` — human governance; where a human COMMAND can defeat a worker's current obligations under Layer 4 defeasibility

Mixing telemetry with coordination acts in a single channel seems fine until
you try to query the ledger. "Show me all open obligations" and "show me all
tool calls" return the same stream. The `observe` channel exists to keep
those concerns separate at the channel level, not just at query time.

Layer 3 (temporal) is expressed through the Qhorus `ChannelSemantic` — APPEND
for ordered coordination, BARRIER for convergence gates, EPHEMERAL for
deadline-critical exchanges — orthogonal to purpose, not a fourth channel.

## The layered examples pattern

The examples in the spec are designed to be importable. Layer 1 shows pure
Qhorus — two agents coordinating via channels, typed messages, shared artefacts.
Layer 2 adds the ledger — same scenario, but now every speech act is recorded
in the normative audit trail with Merkle chaining. Layer 3 adds Claudony —
managed sessions, the dashboard panel, the system prompt template that injects
channel names and startup sequence into each Claude agent. Layer 4 adds CaseHub
and Work — orchestration, worker assignment, lineage-informed context.

Each layer is self-contained. Qhorus can publish Layer 1 as its own example.
Claudony imports it and augments. CaseHub imports both and adds the orchestration.
No one project owns the full story, but the full story is coherent.

I want this to eventually become a wizard: answer three questions about your
case type and get a pre-configured channel layout. That belongs at the CaseHub
level where it can aggregate the full ecosystem — issues #84 and #85 exist to
make sure the idea doesn't dissolve between sessions.
