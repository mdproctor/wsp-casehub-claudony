---
layout: post
title: "When the Ledger Enforces Its Own Rules"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [casehub-ledger, snapshot, tenancy, ci]
---

# Claudony — When the Ledger Enforces Its Own Rules

## What I was trying to fix: a single null check

The branch started as one line — a null tenancyId guard in `ClaudonyLedgerEventCapture`. The previous code silently defaulted to `"default"` when `event.tenancyId()` was null:

```java
entry.tenancyId = event.tenancyId() != null ? event.tenancyId() : "default";
```

That's the wrong behaviour in a multi-tenant system. An event with no tenant identity should be rejected, not silently routed to the default tenant. Cross-tenant data corruption is quiet and takes a long time to find. The fix: early return with an error log when `tenancyId` is null.

There was a second issue hiding nearby. `CaseLedgerEntry` extends `LedgerEntry` via JPA JOINED inheritance. Both the child and the parent class have a `tenancyId` field — the child's shadows the parent's. Setting `entry.tenancyId = tenancyId` only reaches the child column. To write the parent column too, you need an explicit cast:

```java
entry.tenancyId = tenancyId;
((LedgerEntry) entry).tenancyId = tenancyId;
```

That's ugly. It's also correct until the entity hierarchy is refactored. The TODO comment names it as design debt.

## What CI revealed: five repos out of alignment

Running tests after the fix produced a cascade of failures that had nothing to do with the guard. The casehub SNAPSHOT ecosystem had drifted while multiple changes were in flight.

The first failure: `LedgerEntry must be persisted through LedgerEntryRepository, which handles sequence allocation, enrichment, hashing, and signing. Direct em.persist() bypasses the entire save pipeline.`

The ledger had added a `@PrePersist` interceptor (via a Quarkus build step in `casehub-ledger-deployment`) that blocks any direct `em.persist()` on `LedgerEntry` subclasses. `ClaudonyLedgerEventCapture` was persisting directly. The fix was to inject `LedgerEntryRepository` and call `save()` instead, which also removed the manual `nextSequenceNumber()` query — the repository handles sequence allocation internally.

The second failure: `NoSuchMethodError: ActorIdentityProvider.tokenise(java.lang.String)`.

The ledger had changed the `ActorIdentityProvider.tokenise()` signature from `tokenise(String)` to `tokenise(String, ActorType)`. Qhorus had two implementations (`QhorusLedgerEntryRepository` and `ReactiveLedgerEntryJpaRepository`) that still called the old one-argument form. Rebuilding Qhorus from source surfaced the compilation errors. The fix was two lines in each file — pass `entry.actorType` where available, `null` for the attestation case.

After that: `NullPointerException` in `CaseDefinitionYamlMapper` when `ResearcherCaseStartupTest` called `new ResearcherCase().getDefinition()`. The engine SNAPSHOT had added CDI injection of `ObjectMapper` into the mapper — a `@ConfigProperty`-annotated field. Without CDI context, the field is null. This test was a plain JUnit test, no `@QuarkusTest`. It's filed as #153 and left for the next session.

## The underlying pattern

Four repos had accumulated incompatible changes between publishes: ledger changed the `tokenise()` signature and added the `em.persist()` block, engine added the `ObjectMapper` injection, platform updated `MockCurrentPrincipal` with a `@ConfigProperty` on `tenancyId`, and Qhorus hadn't been rebuilt after any of it.

The fix is mechanical: identify the dependency order (ledger → platform → qhorus → engine-ledger) and rebuild from source in that order. Each rebuild either succeeds (change already in source) or surfaces the next compilation error. Once the binaries are consistent, the test failures resolve.

The `ledger_subject_sequence` table was one more thing. The `LedgerEntryRepository.save()` path allocates sequence numbers from a native SQL table that Flyway creates but Hibernate's `drop-and-create` doesn't — because it's not a JPA entity. The fix was a `import-qhorus.sql` file referenced via `quarkus.hibernate-orm.qhorus.sql-load-script`, which runs after Hibernate generates the schema. `import.sql` only works for the default persistence unit. The named-PU variant is in the docs but easy to miss.

Two known failures remain. `ResearcherCaseStartupTest` (#153) needs a CDI-aware approach to test YAML structure. `ResearcherCaseCompletionTest` (#154) is creating too many worker sessions when `signalStarted()` races against the engine's provisioning context patch in the mocked test environment — not a problem in production where the Vert.x event bus ordering is reliable.
