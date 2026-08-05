---
title: "The Module That Wouldn't Let Go"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-claudony]
tags: [quarkus, cdi, spi, composition, architecture]
---

# Claudony — The Module That Wouldn't Let Go

**Date:** 2026-08-05
**Type:** phase-update

---

## What I was trying to achieve: surface work items in the action inbox

Claudony's action inbox already aggregates two types of things needing attention — Qhorus commitments (speech act obligations) and stalled workers. The next step was adding casehub-work WorkItems as a third source. Simple enough on paper: query the work service, map the results, sort by urgency.

The interesting question wasn't *what* to aggregate, but *how* — because where the work data lives depends on how you deploy. The work service might be a separate JVM behind a REST API, or it might be embedded in Claudony's own process. Or it might not be there at all.

## What we believed going in: one SPI, three CDI tiers, clean separation

I wanted a pattern that handled all three deployment topologies without if/else routing code. CDI's priority system is built for exactly this:

- **Tier 0:** `@DefaultBean` no-op — returns empty. Always active. The system works without the work service.
- **Tier 1:** `@ApplicationScoped` REST client — calls an external work service. Active when a URL is configured.
- **Tier 2:** `@Alternative @Priority(1)` embedded — injects the store directly. Active when the module is on the classpath.

CDI picks the highest-priority active tier. The consumer doesn't know or care which one won.

The other design decision: the SPI returns a Claudony-owned DTO (`WorkItemView`), not the external module's types. Urgency mapping — overdue, approaching deadline, priority escalation — stays in the aggregation service. One place to test, one place to change. The SPI implementations handle retrieval only. This matters because urgency rules are product decisions; retrieval topology is a deployment decision. They vary independently, so they shouldn't be coupled.

## Three tiers, two gotchas, and a module that wouldn't separate

Tiers 0 and 1 went in clean. Four commits, twenty-five tests, the REST client gracefully degrades when the work service is down.

Tier 2 is where it got educational.

I added `casehub-work` as a dependency and hit eighty-three CDI deployment errors. The `WorkItemStore` SPI is a clean interface with multiple implementations — JPA, MongoDB, in-memory — following the standard CDI priority ladder. But the interface lives in the runtime module alongside the JPA entity, the Quartz timer jobs, the REST resources, and sixty other beans that inject `EntityManager`. Claudony doesn't have a work persistence unit. It doesn't need one for Tier 2 — it just needs the query interface.

I tried `quarkus.arc.exclude-types` with regex patterns. Nothing happened. Eighty-three errors, unchanged. Then Claude found the deployment processor — `casehub-work` is a Quarkus extension, and its `@BuildStep` methods register beans via `AdditionalBeanBuildItem.setUnremovable()`. These beans bypass classpath scanning entirely. The exclude-types filter never sees them.

There was a third gotcha hiding in the REST client. I'd guarded `RestWorkItemActionSource` with `@UnlessBuildProperty` and injected the service URL as a `@ConfigProperty`. Quarkus validated the config property at build time and threw `DeploymentException` — even though the bean was guarded and would never be instantiated. Config validation runs independently of bean activation. The fix was `Optional<String>` instead of bare `String`.

## What changed and why: Tier 2 deferred, upstream issue filed

The right fix for Tier 2 isn't in Claudony — it's in casehub-work. `WorkItemStore`, `WorkItem` (as a POJO, not a JPA entity), and `WorkItemQuery` should live in `casehub-work-api`, not the runtime module. Then any consumer can depend on just the API types. I filed casehubio/work#337 for the extraction.

This is the kind of thing you only discover by trying to consume a module from the outside. The SPI design is correct — clean interface, multiple implementations, CDI priority ladder. But the packaging puts the interface in the same box as the heaviest implementation. The lesson is portable: if your SPI interface forces consumers to inherit your persistence stack, it isn't really an SPI yet.

## What's next

Tiers 0 and 1 ship now. Once work#337 lands, Tier 2 becomes a straightforward dependency swap — `casehub-work-api` instead of `casehub-work`, no JPA, no exclude-types list. The three-tier pattern itself is recorded as ADR-0001 for Claudony, so the next external module integration follows the same shape.
