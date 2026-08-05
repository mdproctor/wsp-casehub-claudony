# ADR-0001: Three-Tier CDI Composition for External CaseHub Modules

**Status:** Accepted
**Date:** 2026-08-05
**Issue:** #200

## Context

Claudony aggregates data from external CaseHub modules (casehub-work, and potentially others in future). These modules may be:
- **Absent** — not configured (graceful degradation)
- **Remote** — accessible via REST API (different JVM)
- **Co-located** — on the classpath with direct store access (same JVM)

Hardcoding one integration mode forces a deployment topology decision at compile time.

## Decision

Use a three-tier CDI priority pattern for each external module integration:

| Tier | Annotation | When active |
|------|-----------|-------------|
| 0 | `@DefaultBean` | Always — returns empty (graceful degradation) |
| 1 | `@ApplicationScoped` | Config URL present (REST client to external service) |
| 2 | `@Alternative @Priority(1)` | Module runtime on classpath (direct store injection) |

The SPI returns a **Claudony-owned DTO** (not the external module's domain types). All interpretation logic (urgency mapping, action construction, filtering) stays in the consumer service — never in the SPI implementations. Implementations handle only data retrieval and DTO mapping.

## Consequences

- CDI selects the right tier automatically — no if/else topology detection
- Urgency and mapping rules are tested once in the consumer, not per-tier
- Adding a new external module follows the same pattern: interface + 3 implementations
- Tier 2 (embedded) requires the external module to separate its SPI from JPA implementations (casehubio/work#337)

## Alternatives Considered

- **Pre-mapped ActionItems from SPI** — rejected: duplicates urgency logic across tiers
- **Single REST-only integration** — rejected: can't leverage co-location when available
- **Embedded-only** — rejected: forces persistence setup even when a remote service exists
