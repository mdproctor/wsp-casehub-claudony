# WorkItem Integration in Action Inbox — Three-Tier Composition

**Issue:** #200 — feat: WorkItem integration in action inbox — casehub-work Phase 3
**Parent:** #176 — case browser and task inbox

## Context

The action inbox (Phase 2 of #176) aggregates two source types today: Qhorus commitments and stalled workers. This design adds casehub-work WorkItems as a third source type (`WORKITEM`), surfacing assigned work items with urgency-based ordering in the unified inbox.

casehub-work is a standalone project with its own REST API, persistence, and lifecycle. Rather than hardcoding one integration mode, this design introduces a three-tier composition pattern that works across deployment topologies — a pattern reusable by other CaseHub modules.

## SPI: WorkItemActionSource

A thin query interface in `claudony-casehub/inbox/`:

```java
public interface WorkItemActionSource {
    List<WorkItemView> findActionableItems(String tenancyId);
}
```

Returns `WorkItemView` — a Claudony-owned DTO carrying only the fields `ActionAggregationService` needs for urgency mapping and action construction:

```java
public record WorkItemView(
    UUID id,
    String title,
    String status,
    String priority,
    Instant expiresAt,
    Instant claimDeadline,
    Instant createdAt,
    String assigneeId,
    String actionBaseUrl
) {}
```

`actionBaseUrl` is set by the implementation: REST mode uses the external service URL (`http://work:8090/workitems/{id}`), embedded mode uses the local path (`/workitems/{id}`). `ActionAggregationService` appends `/claim`, `/complete`, `/delegate` to construct `ActionDescriptor` endpoints.

### Why raw views, not pre-mapped ActionItems

Urgency rules are product decisions (not topology decisions). Centralising them in `ActionAggregationService` — alongside the existing commitment and stall urgency logic — means one place to change, one place to test. Pushing mapping into each SPI implementation would duplicate urgency logic across tiers.

## Three-Tier CDI Priority

Same pattern as `WorkItemStore` persistence backends and `CaseLineageQuery` / `EmptyCaseLineageQuery` in Claudony.

| Tier | Class | Annotation | Activates when |
|------|-------|-----------|---------------|
| 0 | `EmptyWorkItemActionSource` | `@DefaultBean` | Always — returns empty list |
| 1 | `RestWorkItemActionSource` | `@ApplicationScoped` + `@IfBuildProperty` | `claudony.work-service.url` is configured |
| 2 | `EmbeddedWorkItemActionSource` | `@Alternative @Priority(1)` | `casehub-work` runtime on classpath |

### Tier 0: No-op default

```java
@DefaultBean
@ApplicationScoped
public class EmptyWorkItemActionSource implements WorkItemActionSource {
    public List<WorkItemView> findActionableItems(String tenancyId) {
        return List.of();
    }
}
```

Graceful degradation — inbox works without casehub-work. No configuration needed.

### Tier 1: REST client

Uses Quarkus `@RegisterRestClient` to call the casehub-work REST API at `GET /workitems/inbox`. Deserialises the JSON response into a local DTO (no dependency on `casehub-work-rest` module), maps to `WorkItemView`.

Configuration:

```properties
claudony.work-service.url=http://localhost:8090
```

The REST client is guarded by `@IfBuildProperty(name = "claudony.work-service.url", enableIfMissing = false)` — when the URL is not configured, the bean is not registered and the `@DefaultBean` (Tier 0) activates.

Action URLs point directly to the external work service: `http://work:8090/workitems/{id}/claim`. The frontend calls the work service directly — no proxy through Claudony.

### Tier 2: Embedded

Injects `WorkItemStore` directly and calls `scan(WorkItemQuery.inbox(...))`. Maps `WorkItem` JPA entities to `WorkItemView`.

This tier activates when `casehub-work` (runtime) is on the classpath. It requires a datasource for work item persistence (either shared with Qhorus or a dedicated `work` named datasource). Datasource configuration is outside this issue's scope — the embedded implementation assumes `WorkItemStore` is injectable.

Action URLs use the local path: `/workitems/{id}/claim` — the work REST endpoints mount inside Claudony when the runtime module is on the classpath.

## Urgency Mapping

Centralised in `ActionAggregationService.workItemUrgency(WorkItemView)`:

| Condition | Urgency |
|-----------|---------|
| `expiresAt` before now (overdue) | HIGH |
| priority = `URGENT` | HIGH |
| `claimDeadline` before now | HIGH |
| priority = `HIGH` | MEDIUM |
| `expiresAt` within 60 minutes | MEDIUM |
| Everything else | LOW |

Conditions are evaluated top-to-bottom; first match wins. This mirrors the existing `commitmentUrgency()` pattern: deadline-based escalation with a 60-minute approaching threshold.

## ActionAggregationService Changes

Add a fourth source to `listActions()`:

```java
@Inject
WorkItemActionSource workItemSource;

@Inject
TenantContext tenantContext;

public ActionInboxResponse listActions() {
    List<ActionItem> items = new ArrayList<>();
    items.addAll(commitmentItems());
    items.addAll(stallItems());
    items.addAll(workItemActions());
    // ... existing sort + count logic
}

private List<ActionItem> workItemActions() {
    return workItemSource.findActionableItems(tenantContext.currentTenantId())
            .stream()
            .map(this::toWorkItemAction)
            .toList();
}
```

Each `WorkItemView` maps to:

```java
new ActionItem(
    "workitem:" + view.id(),
    SourceType.WORKITEM,
    workItemUrgency(view),
    view.title(),
    view.status(),
    true,  // actionable
    null,  // caseId — not available from work item
    null,  // channelName
    view.createdAt(),
    List.of(
        new ActionDescriptor("claim", "Claim", "PUT", view.actionBaseUrl() + "/claim"),
        new ActionDescriptor("complete", "Complete", "PUT", view.actionBaseUrl() + "/complete"),
        new ActionDescriptor("delegate", "Delegate", "PUT", view.actionBaseUrl() + "/delegate")
    )
)
```

## Dependencies

### casehub-work-api (compile, claudony-casehub)

Provides `WorkItemStatus` and `WorkItemPriority` enums used in `WorkItemView` and urgency mapping. Lightweight — no CDI beans, no JPA.

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-api</artifactId>
</dependency>
```

### casehub-work (runtime, optional — claudony-app)

Only needed for Tier 2 (embedded). Provides `WorkItemStore`, `WorkItem`, `WorkItemQuery`. Pulls in JPA entities and persistence — heavyweight.

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work</artifactId>
    <optional>true</optional>
</dependency>
```

### casehub-work-persistence-memory (test scope)

For testing the embedded tier without a real database.

### CDI exclude-types

Per the `engine-cdi-exclude-types-sync` protocol: any new beans from `casehub-work` and `casehub-work-core` that inject unavailable dependencies must be added to `%test.quarkus.arc.exclude-types` in `application.properties`, and mirrored in `CasehubEnabledProfile` and `CompletionTestProfile` overrides.

## Configuration

```properties
# REST mode — external work service
claudony.work-service.url=http://localhost:8090

# Embedded mode — no extra config if casehub-work runtime is on classpath
# (datasource setup for work PU is outside this issue's scope)
```

## Test Strategy

### Unit tests (claudony-casehub)

- `EmptyWorkItemActionSourceTest` — returns empty list
- `ActionAggregationServiceTest` — extend with WorkItemView scenarios:
  - overdue expiresAt → HIGH
  - URGENT priority → HIGH
  - claimDeadline past → HIGH
  - HIGH priority → MEDIUM
  - expiresAt within 60 min → MEDIUM
  - no deadline, MEDIUM priority → LOW
  - empty WorkItemActionSource → existing tests pass unchanged
  - sorting: WORKITEM items interleave with commitments and stalls by urgency

### Integration tests (claudony-app)

- `RestWorkItemActionSourceTest` — WireMock or `@QuarkusTest` with a mock HTTP endpoint; verify JSON deserialisation → WorkItemView mapping
- `ActionInboxResourceTest` — extend: mock `ActionAggregationService` returns WORKITEM items, verify JSON response

### Embedded tier test

- `EmbeddedWorkItemActionSourceTest` — inject `InMemoryWorkItemStore` (from `casehub-work-persistence-memory`), populate with test WorkItems, verify `findActionableItems()` returns correct WorkItemViews

## Non-Goals

- Work item creation from the Claudony UI
- Proxying action calls through Claudony (actions go directly to the work service or local endpoints)
- Datasource setup for embedded mode (follow-on when the deployment topology is decided)
- Full work queue management (routing, assignment rules) — stays in casehub-work
