---
layout: post
title: "Phase 11: Everything in Its Own Datasource"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [architecture, persistence, qhorus, casehub, jpa, named-datasource]
excerpt: "Migrating Qhorus to a named datasource hardens into an ecosystem-wide rule — every library owns a named datasource matching its artifact ID — and exposes two direct EntityManager calls hiding behind a clean SPI surface."
---

## A rule, not a preference

It started as "Qhorus DB independence" — a vague sense that H2 with
`AUTO_SERVER=TRUE` wasn't going to survive multi-node. It ended with a
binding architectural rule for the whole ecosystem: every library owns a
named datasource matching its artifact ID. `quarkus.datasource.qhorus.*`.
`quarkus.datasource.casehub.*`. The default datasource belongs to the
embedding application, if it needs one at all.

The rule is simple. Getting there took longer.

## What Qhorus told us

I'd assumed Qhorus's persistence was already cleanly abstracted. It mostly
was — five store interfaces, JPA implementations, in-memory alternatives for
testing. But `PendingReply`, the correlation-ID tracking entity that powers
`wait_for_reply`, was accessing the database directly. No interface. MCP tools
calling `Message.getEntityManager()` in two places. A Redis backend would
have broken immediately.

So we sealed it. A sixth interface, `PendingReplyStore`, with a JPA
implementation and an in-memory alternative. The MCP bypasses got closed.
The store surface is now complete — anything calling persistence goes through
a swappable interface, and `quarkus-qhorus-testing` activates the in-memory
alternatives automatically in tests. No database required to run a
`@QuarkusTest`.

## The ledger trap

The named persistence unit migration hit one constraint I hadn't anticipated.
`AgentMessageLedgerEntry` extends `LedgerEntry` from the `quarkus-ledger`
library using `InheritanceType.JOINED`. JPA requires all entities in an
inheritance hierarchy to share one persistence unit. There's no workaround
short of flattening the inheritance. So the `qhorus` persistence unit now
includes both `io.quarkiverse.qhorus.runtime` and
`io.quarkiverse.ledger.runtime.model` — binding quarkus-ledger's entities
to the Qhorus datasource as a side effect.

We captured this as ADR 0001 in the CaseHub repo. Revisit trigger: when
quarkus-ledger defines its own named persistence unit.

## Where Claudony fits in CaseHub

As we were writing the ADRs, a real design question surfaced: when does
Claudony need to know about CaseHub, and when does CaseHub need to know
about Claudony?

It resolved cleanly. CaseHub defines the SPIs — `WorkerProvisioner`,
`CaseChannelProvider`, and the rest. Claudony implements them. CaseHub never
imports Claudony types. Any Claudony-side CaseHub dependency lives in a
separate optional `claudony-casehub` module. Both projects have ADRs now,
cross-referencing each other.

The one thing we didn't decide: UI unification. When the three-panel dashboard
matures enough that the seams between CaseHub, Qhorus, and Claudony become
visible, there'll be a real question about where the unified UI lives. That's
parked for now.

## One more sandbox

I came across [Nono](https://nono.sh/) — kernel-enforced isolation for AI
agent execution. Deny-by-default, filesystem rollback via content-addressed
snapshots, cryptographic audit trail. It's a natural complement to the
DockerProvisioner in the ecosystem design: Docker gives coarse container
isolation, Nono gives fine-grained process-level isolation and works on macOS.
Added it to the provisioner extensibility diagram and the roadmap.

The ecosystem is starting to look like it has real shape.
