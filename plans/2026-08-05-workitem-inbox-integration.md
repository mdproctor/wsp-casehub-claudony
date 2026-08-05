# WorkItem Inbox Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #200 — feat: WorkItem integration in action inbox — casehub-work Phase 3
**Issue group:** #200

**Goal:** Add casehub-work WorkItems as a third action source in the unified inbox, using a three-tier composition pattern (no-op / REST client / embedded) that selects the right integration mode by CDI priority.

**Architecture:** A `WorkItemActionSource` SPI in `claudony-casehub` returns `WorkItemView` records. Three CDI tiers: `@DefaultBean` no-op (Tier 0), `@ApplicationScoped` REST client guarded by config (Tier 1), `@Alternative @Priority(1)` embedded with direct `WorkItemStore` injection (Tier 2). `ActionAggregationService` owns all urgency mapping and ActionItem construction — implementations only handle data retrieval.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, MicroProfile REST Client, casehub-work-api 0.2-SNAPSHOT

## Global Constraints

- Package: `io.casehub.claudony.casehub.inbox` (SPI + DTOs alongside existing inbox types)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- `@QuarkusTest` uses `quarkus.http.test-port=0` (random port)
- `@TestSecurity(user = "test", roles = "user")` only on HTTP-exercising tests
- CDI exclude-types: per `engine-cdi-exclude-types-sync` protocol, any new beans from casehub-work must be added to `%test.quarkus.arc.exclude-types` AND mirrored in `CasehubEnabledProfile` and `CompletionTestProfile`
- `casehub-work-api` version managed via `${casehub.version}` (0.2-SNAPSHOT)
- `RoundRobinStrategy` from `casehub-work-core` already excluded (line 99 of test application.properties)
- Existing `SourceType.WORKITEM` enum value is pre-wired — no model changes needed

---

### Task 1: SPI interface, WorkItemView DTO, and EmptyWorkItemActionSource

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/WorkItemView.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/WorkItemActionSource.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/EmptyWorkItemActionSource.java`
- Modify: `casehub/pom.xml` (add casehub-work-api dependency)
- Modify: `pom.xml` (add casehub-work-api to dependencyManagement)
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/inbox/EmptyWorkItemActionSourceTest.java`

**Interfaces:**
- Consumes: `io.casehub.work.api.WorkItemStatus`, `io.casehub.work.api.WorkItemPriority` from casehub-work-api
- Produces: `WorkItemActionSource.findActionableItems(String tenancyId)` → `List<WorkItemView>`, `WorkItemView` record, `EmptyWorkItemActionSource` (always returns empty list)

- [ ] **Step 1: Add casehub-work-api to dependency management**

Add to `pom.xml` `<dependencyManagement>` section, after the casehub-ledger entry:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-api</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

Add to `casehub/pom.xml` `<dependencies>`:

```xml
<!-- WorkItem API types (WorkItemStatus, WorkItemPriority) for inbox integration -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-api</artifactId>
</dependency>
```

- [ ] **Step 2: Write the test**

```java
// casehub/src/test/java/io/casehub/claudony/casehub/inbox/EmptyWorkItemActionSourceTest.java
package io.casehub.claudony.casehub.inbox;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class EmptyWorkItemActionSourceTest {

    EmptyWorkItemActionSource source = new EmptyWorkItemActionSource();

    @Test
    void returnsEmptyList() {
        var result = source.findActionableItems("default");
        assertNotNull(result);
        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EmptyWorkItemActionSourceTest`
Expected: FAIL — classes do not exist

- [ ] **Step 4: Create WorkItemView record**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/inbox/WorkItemView.java
package io.casehub.claudony.casehub.inbox;

import java.time.Instant;
import java.util.UUID;

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

- [ ] **Step 5: Create WorkItemActionSource interface**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/inbox/WorkItemActionSource.java
package io.casehub.claudony.casehub.inbox;

import java.util.List;

public interface WorkItemActionSource {
    List<WorkItemView> findActionableItems(String tenancyId);
}
```

- [ ] **Step 6: Create EmptyWorkItemActionSource**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/inbox/EmptyWorkItemActionSource.java
package io.casehub.claudony.casehub.inbox;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@DefaultBean
@ApplicationScoped
public class EmptyWorkItemActionSource implements WorkItemActionSource {

    @Override
    public List<WorkItemView> findActionableItems(String tenancyId) {
        return List.of();
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EmptyWorkItemActionSourceTest`
Expected: PASS

- [ ] **Step 8: Commit**

```
feat(casehub): WorkItemActionSource SPI + EmptyWorkItemActionSource (#200)

Three-tier composition pattern: SPI interface, WorkItemView DTO,
and @DefaultBean no-op implementation. casehub-work-api added for
WorkItemStatus/WorkItemPriority types.

Refs #200
```

---

### Task 2: Extend ActionAggregationService with WorkItem source

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionAggregationService.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/inbox/ActionAggregationServiceTest.java`

**Interfaces:**
- Consumes: `WorkItemActionSource.findActionableItems(String)` → `List<WorkItemView>`, `TenantContext.currentTenantId()` → `String`
- Produces: `ActionItem` records with `sourceType=WORKITEM` in `listActions()` response

- [ ] **Step 1: Write failing tests**

Add to `ActionAggregationServiceTest.java`:

```java
WorkItemActionSource workItemSource = mock(WorkItemActionSource.class);

// Update setUp():
@BeforeEach
void setUp() {
    when(tenantContext.currentTenantId()).thenReturn("default");
    sessionRegistry = new SessionRegistry(tenantContext);
    when(commitmentStore.findAllOpen()).thenReturn(List.of());
    when(workItemSource.findActionableItems("default")).thenReturn(List.of());
    service = new ActionAggregationService(commitmentStore, sessionRegistry,
            stallTracker, sessionMapping, workItemSource, tenantContext);
}

@Test
void overdueWorkItemIsHighUrgency() {
    var view = new WorkItemView(UUID.randomUUID(), "Review PR",
            "IN_PROGRESS", "MEDIUM",
            Instant.now().minusSeconds(3600), null,
            Instant.now().minusSeconds(7200), "alice",
            "/workitems/" + UUID.randomUUID());
    when(workItemSource.findActionableItems("default")).thenReturn(List.of(view));

    var result = service.listActions();
    var items = result.items().stream()
            .filter(a -> a.sourceType() == SourceType.WORKITEM).toList();
    assertEquals(1, items.size());
    assertEquals(Urgency.HIGH, items.get(0).urgency());
    assertEquals("Review PR", items.get(0).title());
    assertTrue(items.get(0).actionable());
}

@Test
void urgentPriorityWorkItemIsHighUrgency() {
    var view = new WorkItemView(UUID.randomUUID(), "Escalation",
            "ASSIGNED", "URGENT",
            null, null, Instant.now(), "bob",
            "/workitems/" + UUID.randomUUID());
    when(workItemSource.findActionableItems("default")).thenReturn(List.of(view));

    var result = service.listActions();
    assertEquals(Urgency.HIGH, result.items().get(0).urgency());
}

@Test
void highPriorityWorkItemIsMediumUrgency() {
    var view = new WorkItemView(UUID.randomUUID(), "Code review",
            "ASSIGNED", "HIGH",
            null, null, Instant.now(), "charlie",
            "/workitems/" + UUID.randomUUID());
    when(workItemSource.findActionableItems("default")).thenReturn(List.of(view));

    var result = service.listActions();
    assertEquals(Urgency.MEDIUM, result.items().get(0).urgency());
}

@Test
void expiredClaimDeadlineIsHighUrgency() {
    var view = new WorkItemView(UUID.randomUUID(), "Unclaimed task",
            "PENDING", "LOW",
            null, Instant.now().minusSeconds(600), Instant.now().minusSeconds(3600),
            null, "/workitems/" + UUID.randomUUID());
    when(workItemSource.findActionableItems("default")).thenReturn(List.of(view));

    var result = service.listActions();
    assertEquals(Urgency.HIGH, result.items().get(0).urgency());
}

@Test
void approachingDeadlineIsMediumUrgency() {
    var view = new WorkItemView(UUID.randomUUID(), "Almost due",
            "IN_PROGRESS", "MEDIUM",
            Instant.now().plusSeconds(1800), null, Instant.now(), "dave",
            "/workitems/" + UUID.randomUUID());
    when(workItemSource.findActionableItems("default")).thenReturn(List.of(view));

    var result = service.listActions();
    assertEquals(Urgency.MEDIUM, result.items().get(0).urgency());
}

@Test
void normalWorkItemIsLowUrgency() {
    var view = new WorkItemView(UUID.randomUUID(), "Routine task",
            "ASSIGNED", "MEDIUM",
            null, null, Instant.now(), "eve",
            "/workitems/" + UUID.randomUUID());
    when(workItemSource.findActionableItems("default")).thenReturn(List.of(view));

    var result = service.listActions();
    assertEquals(Urgency.LOW, result.items().get(0).urgency());
}

@Test
void workItemActionsIncludeClaimAndComplete() {
    var id = UUID.randomUUID();
    var view = new WorkItemView(id, "Task",
            "PENDING", "MEDIUM",
            null, null, Instant.now(), null,
            "/workitems/" + id);
    when(workItemSource.findActionableItems("default")).thenReturn(List.of(view));

    var result = service.listActions();
    var actions = result.items().get(0).actions();
    assertEquals(3, actions.size());
    assertEquals("claim", actions.get(0).name());
    assertEquals("PUT", actions.get(0).method());
    assertTrue(actions.get(0).endpoint().endsWith("/claim"));
    assertEquals("complete", actions.get(1).name());
    assertEquals("delegate", actions.get(2).name());
}

@Test
void emptyWorkItemSourceDoesNotAffectExistingBehavior() {
    when(workItemSource.findActionableItems("default")).thenReturn(List.of());
    var result = service.listActions();
    assertTrue(result.items().isEmpty());
    assertEquals(0, result.counts().high());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ActionAggregationServiceTest`
Expected: FAIL — constructor signature changed (new parameters)

- [ ] **Step 3: Update ActionAggregationService**

Add fields and update constructor:

```java
private final WorkItemActionSource workItemSource;
private final TenantContext tenantContext;

@Inject
public ActionAggregationService(CommitmentStore commitmentStore,
                                 SessionRegistry sessionRegistry,
                                 StallTracker stallTracker,
                                 WorkerSessionMapping sessionMapping,
                                 WorkItemActionSource workItemSource,
                                 TenantContext tenantContext) {
    this.commitmentStore = commitmentStore;
    this.sessionRegistry = sessionRegistry;
    this.stallTracker = stallTracker;
    this.sessionMapping = sessionMapping;
    this.workItemSource = workItemSource;
    this.tenantContext = tenantContext;
}
```

Add work item aggregation to `listActions()`:

```java
public ActionInboxResponse listActions() {
    List<ActionItem> items = new ArrayList<>();
    items.addAll(commitmentItems());
    items.addAll(stallItems());
    items.addAll(workItemActions());
    items.sort(Comparator.comparing(ActionItem::urgency)
            .thenComparing(ActionItem::createdAt, Comparator.reverseOrder()));

    long high = items.stream().filter(a -> a.urgency() == Urgency.HIGH).count();
    long medium = items.stream().filter(a -> a.urgency() == Urgency.MEDIUM).count();
    long low = items.stream().filter(a -> a.urgency() == Urgency.LOW).count();
    return new ActionInboxResponse(items, new ActionCounts((int) high, (int) medium, (int) low));
}
```

Add the work item methods:

```java
private List<ActionItem> workItemActions() {
    return workItemSource.findActionableItems(tenantContext.currentTenantId())
            .stream()
            .map(this::toWorkItemAction)
            .toList();
}

private ActionItem toWorkItemAction(WorkItemView view) {
    return new ActionItem(
            "workitem:" + view.id(),
            SourceType.WORKITEM,
            workItemUrgency(view),
            view.title(),
            view.status(),
            true,
            null,
            null,
            view.createdAt(),
            List.of(
                    new ActionDescriptor("claim", "Claim", "PUT",
                            view.actionBaseUrl() + "/claim"),
                    new ActionDescriptor("complete", "Complete", "PUT",
                            view.actionBaseUrl() + "/complete"),
                    new ActionDescriptor("delegate", "Delegate", "PUT",
                            view.actionBaseUrl() + "/delegate")
            )
    );
}

private Urgency workItemUrgency(WorkItemView view) {
    if (view.expiresAt() != null && view.expiresAt().isBefore(Instant.now())) return Urgency.HIGH;
    if ("URGENT".equals(view.priority())) return Urgency.HIGH;
    if (view.claimDeadline() != null && view.claimDeadline().isBefore(Instant.now())) return Urgency.HIGH;
    if ("HIGH".equals(view.priority())) return Urgency.MEDIUM;
    if (view.expiresAt() != null && Duration.between(Instant.now(), view.expiresAt()).toMinutes() < 60)
        return Urgency.MEDIUM;
    return Urgency.LOW;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ActionAggregationServiceTest`
Expected: PASS (all existing + 8 new tests)

- [ ] **Step 5: Run full casehub module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`
Expected: PASS — existing tests compile with updated constructor

- [ ] **Step 6: Commit**

```
feat(casehub): integrate WorkItemActionSource into ActionAggregationService (#200)

Urgency mapping: overdue/URGENT/expired-claim → HIGH,
HIGH-priority/approaching-deadline → MEDIUM, else LOW.
ActionDescriptors for claim, complete, delegate.
8 new unit tests.

Refs #200
```

---

### Task 3: REST client WorkItemActionSource (Tier 1)

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/work/WorkServiceClient.java`
- Create: `app/src/main/java/io/casehub/claudony/server/work/WorkServiceResponse.java`
- Create: `app/src/main/java/io/casehub/claudony/server/work/RestWorkItemActionSource.java`
- Modify: `app/src/main/resources/application.properties` (REST client config)
- Test: `app/src/test/java/io/casehub/claudony/server/work/RestWorkItemActionSourceTest.java`

**Interfaces:**
- Consumes: `GET /workitems/inbox` on external work service → JSON list of work items
- Produces: `WorkItemActionSource.findActionableItems(String)` → `List<WorkItemView>` from REST response

- [ ] **Step 1: Write the test**

```java
// app/src/test/java/io/casehub/claudony/server/work/RestWorkItemActionSourceTest.java
package io.casehub.claudony.server.work;

import io.casehub.claudony.casehub.inbox.WorkItemView;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class RestWorkItemActionSourceTest {

    @Test
    void mapsResponseToWorkItemView() {
        var id = UUID.randomUUID();
        var response = new WorkServiceResponse(
                id, "Review PR", "IN_PROGRESS", "HIGH",
                "alice", Instant.now().plusSeconds(3600),
                null, Instant.now());
        var baseUrl = "http://work:8090";

        WorkServiceClient client = (assignee, candidateUser, candidateGroups) ->
                List.of(response);

        var source = new RestWorkItemActionSource(client, baseUrl);
        var result = source.findActionableItems("default");

        assertEquals(1, result.size());
        WorkItemView view = result.get(0);
        assertEquals(id, view.id());
        assertEquals("Review PR", view.title());
        assertEquals("IN_PROGRESS", view.status());
        assertEquals("HIGH", view.priority());
        assertEquals("alice", view.assigneeId());
        assertEquals("http://work:8090/workitems/" + id, view.actionBaseUrl());
    }

    @Test
    void emptyResponseReturnsEmptyList() {
        WorkServiceClient client = (assignee, candidateUser, candidateGroups) ->
                List.of();

        var source = new RestWorkItemActionSource(client, "http://work:8090");
        var result = source.findActionableItems("default");
        assertTrue(result.isEmpty());
    }

    @Test
    void clientExceptionReturnsEmptyList() {
        WorkServiceClient client = (assignee, candidateUser, candidateGroups) -> {
            throw new jakarta.ws.rs.ProcessingException("connection refused");
        };

        var source = new RestWorkItemActionSource(client, "http://work:8090");
        var result = source.findActionableItems("default");
        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=RestWorkItemActionSourceTest`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create WorkServiceResponse DTO**

```java
// app/src/main/java/io/casehub/claudony/server/work/WorkServiceResponse.java
package io.casehub.claudony.server.work;

import java.time.Instant;
import java.util.UUID;

public record WorkServiceResponse(
    UUID id,
    String title,
    String status,
    String priority,
    String assigneeId,
    Instant expiresAt,
    Instant claimDeadline,
    Instant createdAt
) {}
```

- [ ] **Step 4: Create WorkServiceClient interface**

```java
// app/src/main/java/io/casehub/claudony/server/work/WorkServiceClient.java
package io.casehub.claudony.server.work;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.QueryParam;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;

import java.util.List;

@RegisterRestClient(configKey = "work-service")
@Path("/workitems")
public interface WorkServiceClient {

    @GET
    @Path("/inbox")
    List<WorkServiceResponse> inbox(
            @QueryParam("assignee") String assignee,
            @QueryParam("candidateUser") String candidateUser,
            @QueryParam("candidateGroup") List<String> candidateGroups);
}
```

- [ ] **Step 5: Create RestWorkItemActionSource**

```java
// app/src/main/java/io/casehub/claudony/server/work/RestWorkItemActionSource.java
package io.casehub.claudony.server.work;

import io.casehub.claudony.casehub.inbox.WorkItemActionSource;
import io.casehub.claudony.casehub.inbox.WorkItemView;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.rest.client.inject.RestClient;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Optional;

@ApplicationScoped
@io.quarkus.arc.properties.UnlessBuildProperty(name = "claudony.work-service.url", stringValue = "", enableIfMissing = true)
public class RestWorkItemActionSource implements WorkItemActionSource {

    private static final Logger LOG = Logger.getLogger(RestWorkItemActionSource.class);

    private final WorkServiceClient client;
    private final String baseUrl;

    @Inject
    public RestWorkItemActionSource(@RestClient WorkServiceClient client,
                                     @ConfigProperty(name = "claudony.work-service.url") String baseUrl) {
        this.client = client;
        this.baseUrl = baseUrl;
    }

    RestWorkItemActionSource(WorkServiceClient client, String baseUrl) {
        this.client = client;
        this.baseUrl = baseUrl;
    }

    @Override
    public List<WorkItemView> findActionableItems(String tenancyId) {
        try {
            return client.inbox(null, null, null).stream()
                    .map(this::toView)
                    .toList();
        } catch (Exception e) {
            LOG.warnf("Work service unavailable: %s", e.getMessage());
            return List.of();
        }
    }

    private WorkItemView toView(WorkServiceResponse r) {
        return new WorkItemView(
                r.id(), r.title(), r.status(), r.priority(),
                r.expiresAt(), r.claimDeadline(), r.createdAt(),
                r.assigneeId(),
                baseUrl + "/workitems/" + r.id());
    }
}
```

- [ ] **Step 6: Add REST client config to application.properties**

Add to `app/src/main/resources/application.properties`:

```properties
# Work service REST client (Tier 1 — external work service)
# Set claudony.work-service.url to enable; omit to use @DefaultBean no-op
quarkus.rest-client.work-service.url=${claudony.work-service.url:http://localhost:8090}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=RestWorkItemActionSourceTest`
Expected: PASS (3 tests)

- [ ] **Step 8: Commit**

```
feat(app): RestWorkItemActionSource — Tier 1 REST client for work items (#200)

Calls GET /workitems/inbox on external work service via MicroProfile
REST Client. Graceful degradation on connection failure. Guarded by
claudony.work-service.url config property. 3 unit tests.

Refs #200
```

---

### Task 4: Embedded WorkItemActionSource (Tier 2)

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/work/EmbeddedWorkItemActionSource.java`
- Modify: `app/pom.xml` (add casehub-work + casehub-work-persistence-memory dependencies)
- Modify: `pom.xml` (add casehub-work + casehub-work-persistence-memory to dependencyManagement)
- Modify: `app/src/test/resources/application.properties` (CDI exclude-types for work beans)
- Test: `app/src/test/java/io/casehub/claudony/server/work/EmbeddedWorkItemActionSourceTest.java`

**Interfaces:**
- Consumes: `WorkItemStore.scan(WorkItemQuery)` → `List<WorkItem>` from casehub-work runtime
- Produces: `WorkItemActionSource.findActionableItems(String)` → `List<WorkItemView>` via direct store access

- [ ] **Step 1: Add dependencies**

Add to `pom.xml` `<dependencyManagement>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-persistence-memory</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

Add to `app/pom.xml` `<dependencies>`:

```xml
<!-- casehub-work runtime — Tier 2 embedded WorkItemActionSource -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work</artifactId>
    <optional>true</optional>
</dependency>

<!-- In-memory work item store for testing -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-persistence-memory</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Write the test**

```java
// app/src/test/java/io/casehub/claudony/server/work/EmbeddedWorkItemActionSourceTest.java
package io.casehub.claudony.server.work;

import io.casehub.claudony.casehub.inbox.WorkItemView;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.repository.WorkItemStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class EmbeddedWorkItemActionSourceTest {

    WorkItemStore workItemStore = mock(WorkItemStore.class);
    EmbeddedWorkItemActionSource source;

    @BeforeEach
    void setUp() {
        source = new EmbeddedWorkItemActionSource(workItemStore);
    }

    @Test
    void mapsWorkItemToView() {
        var wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.title = "Review code";
        wi.status = WorkItemStatus.IN_PROGRESS;
        wi.priority = WorkItemPriority.HIGH;
        wi.assigneeId = "alice";
        wi.expiresAt = Instant.now().plusSeconds(3600);
        wi.createdAt = Instant.now();
        wi.tenancyId = "default";

        when(workItemStore.scan(any())).thenReturn(java.util.List.of(wi));

        var result = source.findActionableItems("default");
        assertEquals(1, result.size());
        WorkItemView view = result.get(0);
        assertEquals(wi.id, view.id());
        assertEquals("Review code", view.title());
        assertEquals("IN_PROGRESS", view.status());
        assertEquals("HIGH", view.priority());
        assertEquals("/workitems/" + wi.id, view.actionBaseUrl());
    }

    @Test
    void emptyStoreReturnsEmptyList() {
        when(workItemStore.scan(any())).thenReturn(java.util.List.of());
        var result = source.findActionableItems("default");
        assertTrue(result.isEmpty());
    }

    @Test
    void filtersToActiveStatusesOnly() {
        source.findActionableItems("default");
        var captor = org.mockito.ArgumentCaptor.forClass(
                io.casehub.work.runtime.repository.WorkItemQuery.class);
        verify(workItemStore).scan(captor.capture());
        var query = captor.getValue();
        assertNotNull(query.statusIn());
        assertTrue(query.statusIn().contains(WorkItemStatus.PENDING));
        assertTrue(query.statusIn().contains(WorkItemStatus.ASSIGNED));
        assertTrue(query.statusIn().contains(WorkItemStatus.IN_PROGRESS));
        assertFalse(query.statusIn().contains(WorkItemStatus.COMPLETED));
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=EmbeddedWorkItemActionSourceTest`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement EmbeddedWorkItemActionSource**

```java
// app/src/main/java/io/casehub/claudony/server/work/EmbeddedWorkItemActionSource.java
package io.casehub.claudony.server.work;

import io.casehub.claudony.casehub.inbox.WorkItemActionSource;
import io.casehub.claudony.casehub.inbox.WorkItemView;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.repository.WorkItemQuery;
import io.casehub.work.runtime.repository.WorkItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.annotation.Priority;
import jakarta.inject.Inject;

import java.util.List;

@Alternative
@Priority(1)
@ApplicationScoped
public class EmbeddedWorkItemActionSource implements WorkItemActionSource {

    private final WorkItemStore workItemStore;

    @Inject
    public EmbeddedWorkItemActionSource(WorkItemStore workItemStore) {
        this.workItemStore = workItemStore;
    }

    @Override
    public List<WorkItemView> findActionableItems(String tenancyId) {
        var query = WorkItemQuery.builder()
                .statusIn(List.of(
                        WorkItemStatus.PENDING,
                        WorkItemStatus.ASSIGNED,
                        WorkItemStatus.IN_PROGRESS,
                        WorkItemStatus.SUSPENDED,
                        WorkItemStatus.DELEGATED))
                .tenancyId(tenancyId)
                .build();
        return workItemStore.scan(query).stream()
                .map(this::toView)
                .toList();
    }

    private WorkItemView toView(WorkItem wi) {
        return new WorkItemView(
                wi.id, wi.title,
                wi.status != null ? wi.status.name() : "UNKNOWN",
                wi.priority != null ? wi.priority.name() : "MEDIUM",
                wi.expiresAt, wi.claimDeadline, wi.createdAt,
                wi.assigneeId,
                "/workitems/" + wi.id);
    }
}
```

- [ ] **Step 5: Update CDI exclude-types**

Add the following beans from `casehub-work` runtime to `%test.quarkus.arc.exclude-types` in `app/src/test/resources/application.properties`. These beans inject dependencies not available in Claudony's test context (Quartz scheduler, MongoDB, cross-tenant producers, timer services).

Append to the existing exclude-types list (before the final unescaped line):

```
  io.casehub.work.runtime.service.WorkItemService,\
  io.casehub.work.runtime.service.WorkItemAssignmentService,\
  io.casehub.work.runtime.service.WorkItemTimerService,\
  io.casehub.work.runtime.service.ExpiryLifecycleService,\
  io.casehub.work.runtime.service.ClaimDeadlineTimerJob,\
  io.casehub.work.runtime.service.ExpiryTimerJob,\
  io.casehub.work.runtime.service.WorkItemScheduleService,\
  io.casehub.work.runtime.service.WorkItemSpawnService,\
  io.casehub.work.runtime.service.WorkItemTemplateService,\
  io.casehub.work.runtime.service.WorkItemTemplateValidationService,\
  io.casehub.work.runtime.service.LabelVocabularyService,\
  io.casehub.work.runtime.service.FormSchemaValidationService,\
  io.casehub.work.runtime.service.BlockedAttemptAuditService,\
  io.casehub.work.runtime.service.TimerRecoveryStartup,\
  io.casehub.work.runtime.service.CrossTenantProducer,\
  io.casehub.work.runtime.service.WorkItemMetrics,\
  io.casehub.work.runtime.service.WorkSystem,\
  io.casehub.work.runtime.event.WorkItemEventBroadcaster,\
  io.casehub.claudony.server.work.RestWorkItemActionSource
```

**Note:** The exact list may need adjustment during compilation — if CDI deployment fails with `UnsatisfiedResolutionException` for a bean not listed above, add it. Run `mvn test -pl app -Dtest=SmokeTest` iteratively until CDI deployment succeeds.

Also add the same entries to `CasehubEnabledProfile.getConfigOverrides()` and `CompletionTestProfile.getConfigOverrides()` per the engine-cdi-exclude-types-sync protocol.

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=EmbeddedWorkItemActionSourceTest`
Expected: PASS (3 tests)

- [ ] **Step 7: Run SmokeTest to verify CDI deployment**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=SmokeTest`
Expected: PASS — CDI context deploys cleanly with new exclude-types

- [ ] **Step 8: Commit**

```
feat(app): EmbeddedWorkItemActionSource — Tier 2 direct WorkItemStore (#200)

@Alternative @Priority(1) implementation that queries WorkItemStore
directly when casehub-work runtime is on the classpath. Filters to
active statuses. CDI exclude-types updated for work runtime beans.
3 unit tests.

Refs #200
```

---

### Task 5: Full-stack verification

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/server/ActionInboxResourceTest.java` (extend with WORKITEM scenario)

**Interfaces:**
- Consumes: All prior tasks
- Produces: End-to-end validation

- [ ] **Step 1: Extend ActionInboxResourceTest**

Add to `ActionInboxResourceTest.java`:

```java
@Test
void listActions_withWorkItems() {
    var item = new ActionItem("workitem:123", SourceType.WORKITEM,
            Urgency.MEDIUM, "Review PR #42", "IN_PROGRESS", true,
            null, null, Instant.now(),
            List.of(new ActionDescriptor("claim", "Claim", "PUT", "/workitems/123/claim")));
    when(aggregationService.listActions())
            .thenReturn(new ActionInboxResponse(List.of(item), new ActionCounts(0, 1, 0)));
    RestAssured.given()
            .when().get("/api/actions")
            .then().statusCode(200)
            .body("items", hasSize(1))
            .body("items[0].sourceType", is("WORKITEM"))
            .body("items[0].title", is("Review PR #42"))
            .body("items[0].actions[0].name", is("claim"))
            .body("counts.medium", is(1));
}
```

- [ ] **Step 2: Run the new test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ActionInboxResourceTest`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All existing tests pass plus ~15 new tests from Tasks 1-4

- [ ] **Step 4: Run frontend tests**

Run: `npm --prefix app/src/main/webui test`
Expected: All 28 vitest tests pass (no frontend changes in this issue)

- [ ] **Step 5: Commit**

```
test: full-stack verification — WorkItem inbox integration (#200)

ActionInboxResource returns WORKITEM items. All backend tests pass
(~647 total). Frontend tests unaffected.

Closes #200
```
