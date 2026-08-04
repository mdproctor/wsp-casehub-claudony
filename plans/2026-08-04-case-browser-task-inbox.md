# Case Browser and Task Inbox Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #176 — feat: case browser and task inbox — surface work across sessions
**Issue group:** #176

**Goal:** Add case browser and unified action inbox to Claudony's fleet home, composing existing platform components (blocks-case-explorer, QhorusDashboardService, CommitmentStore, CaseInstanceRepository) with new aggregation endpoints.

**Architecture:** Phase 1 adds a case browser (REST endpoint aggregating CaseInstanceRepository + SessionRegistry + Qhorus channels, frontend via blocks-case-explorer). Phase 2 adds a unified action inbox (ActionItem abstraction over Qhorus commitments + stalled workers + oversight messages, custom frontend using pages-table). Phase 3 (follow-on) adds casehub-work WorkItem integration. The fleet home switches from `columns()` to `tabs()` layout.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, Lit 3, TypeScript, pages-ui DSL, blocks-ui components, vitest, Playwright

## Global Constraints

- Package: `io.casehub.claudony` (not `dev.claudony`)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Frontend tests: `npm --prefix app/src/main/webui test`
- `@QuarkusTest` uses `quarkus.http.test-port=0` (random port)
- `@TestSecurity(user = "test", roles = "user")` only on HTTP-exercising tests
- Qhorus test cleanup: `@Inject InMemoryChannelStore channelStore` + `clear()` in `@AfterEach`
- CaseInstance has no `createdAt` field — omit from list, derive from ledger in detail
- SessionStatus: ACTIVE, WAITING, IDLE (no FAULTED — stall detection via CDI event observer)
- CommitmentStore extends CommitmentReader — inject CommitmentStore for query methods
- Use `tabs()` from `@casehubio/pages-ui` DSL for fleet home layout

---

### Task 1: Fleet home tab layout

**Files:**
- Modify: `app/src/main/webui/src/app.ts`
- Test: `app/src/main/webui/src/app.test.ts` (new)

**Interfaces:**
- Consumes: `tabs()`, `hostPanel()`, `registerPanel()` from `@casehubio/pages-ui` and `@casehubio/pages-runtime`
- Produces: Tab-based fleet home with 5 tabs (Sessions, Cases, Inbox, Fleet, Mesh). Cases and Inbox tabs render placeholder divs until Tasks 4 and 7 wire the real components.

- [ ] **Step 1: Write the test**

```typescript
// app/src/main/webui/src/app.test.ts
import { describe, it, expect, vi } from 'vitest';

vi.mock('@casehubio/pages-runtime', () => ({
  loadSite: vi.fn().mockResolvedValue(undefined),
  registerPanel: vi.fn(),
}));

vi.mock('@casehubio/pages-ui', () => ({
  tabs: vi.fn((...entries: any[]) => ({ type: 'tabs', entries })),
  hostPanel: vi.fn((name: string) => ({ type: 'host', name })),
  columns: vi.fn(),
}));

vi.mock('./theme', () => ({ initTheme: vi.fn() }));
vi.mock('./components/session-panel', () => ({}));
vi.mock('./components/claudony-fleet-panel', () => ({}));
vi.mock('./components/claudony-mesh-panel', () => ({}));

describe('app', () => {
  it('registers five panels and uses tabs layout', async () => {
    document.body.innerHTML = '<div id="app"></div>';
    const { registerPanel } = await import('@casehubio/pages-runtime');
    const { tabs } = await import('@casehubio/pages-ui');
    await import('./app');

    expect(registerPanel).toHaveBeenCalledWith('session-panel', 'claudony-session-panel');
    expect(registerPanel).toHaveBeenCalledWith('fleet-panel', 'claudony-fleet-panel');
    expect(registerPanel).toHaveBeenCalledWith('mesh-panel', 'claudony-mesh-panel');
    expect(tabs).toHaveBeenCalled();
    const tabsCall = vi.mocked(tabs).mock.calls[0];
    expect(tabsCall).toHaveLength(5);
    expect(tabsCall[0][0]).toBe('Sessions');
    expect(tabsCall[1][0]).toBe('Cases');
    expect(tabsCall[2][0]).toBe('Inbox');
    expect(tabsCall[3][0]).toBe('Fleet');
    expect(tabsCall[4][0]).toBe('Mesh');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm --prefix app/src/main/webui test -- --run app.test`
Expected: FAIL — app.ts still uses `columns()`, not `tabs()`

- [ ] **Step 3: Update app.ts to use tabs layout**

Replace `app/src/main/webui/src/app.ts` with:

```typescript
import { loadSite, registerPanel } from "@casehubio/pages-runtime";
import { hostPanel, tabs } from "@casehubio/pages-ui";
import { initTheme } from "./theme";
import "./components/session-panel";
import "./components/claudony-fleet-panel";
import "./components/claudony-mesh-panel";

initTheme();

registerPanel("session-panel", "claudony-session-panel");
registerPanel("fleet-panel", "claudony-fleet-panel");
registerPanel("mesh-panel", "claudony-mesh-panel");

const app = tabs(
  ["Sessions", hostPanel("session-panel")],
  ["Cases", hostPanel("case-browser")],
  ["Inbox", hostPanel("action-inbox")],
  ["Fleet", hostPanel("fleet-panel")],
  ["Mesh", hostPanel("mesh-panel")],
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, app).catch(console.error);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm --prefix app/src/main/webui test -- --run app.test`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(ui): switch fleet home from columns to tabs layout (#176)

Adds Cases and Inbox tabs alongside Sessions, Fleet, and Mesh.
New tabs render empty until case browser and action inbox components
are wired in subsequent commits.

Refs #176
```

---

### Task 2: CaseBrowserService and DTOs

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/browser/CaseSummary.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/browser/CaseDetail.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/browser/WorkerInfo.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/browser/CaseBrowserService.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/browser/CaseBrowserServiceTest.java`

**Interfaces:**
- Consumes: `CaseInstanceRepository.findAll(String tenancyId)` → `List<CaseInstance>`, `CaseInstanceRepository.findByUuid(UUID, String)` → `CaseInstance`, `SessionRegistry.findByCaseId(String)` → `List<Session>`, `SessionRegistry.all()` → `Collection<Session>`, `QhorusDashboardService.listChannels()` → `List<ChannelDetail>`, `QhorusDashboardService.getTimeline(String, Long, int)` → `List<Map<String, Object>>`, `CaseLineageQuery.findCompletedWorkers(UUID)` → `List<WorkerSummary>`, `TenantContext.currentTenantId()` → `String`
- Produces: `CaseBrowserService.listCases()` → `List<CaseSummary>`, `CaseBrowserService.getCaseDetail(UUID)` → `Optional<CaseDetail>`

- [ ] **Step 1: Create DTOs**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/browser/WorkerInfo.java
package io.casehub.claudony.casehub.browser;

import java.time.Instant;

public record WorkerInfo(
    String sessionId,
    String roleName,
    String status,
    Instant lastActive,
    boolean active
) {}
```

```java
// casehub/src/main/java/io/casehub/claudony/casehub/browser/CaseSummary.java
package io.casehub.claudony.casehub.browser;

import java.time.Instant;
import java.util.UUID;

public record CaseSummary(
    UUID id,
    String status,
    String definitionName,
    int activeWorkerCount,
    int channelCount,
    Instant lastActivity
) {}
```

```java
// casehub/src/main/java/io/casehub/claudony/casehub/browser/CaseDetail.java
package io.casehub.claudony.casehub.browser;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public record CaseDetail(
    UUID id,
    String status,
    String definitionName,
    List<WorkerInfo> workers,
    List<String> channels,
    List<Map<String, Object>> timeline,
    Instant lastActivity
) {}
```

- [ ] **Step 2: Write failing tests for CaseBrowserService**

```java
// casehub/src/test/java/io/casehub/claudony/casehub/browser/CaseBrowserServiceTest.java
package io.casehub.claudony.casehub.browser;

import io.casehub.api.model.CaseStatus;
import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TenantContext;
import io.casehub.claudony.server.model.Session;
import io.casehub.claudony.server.model.SessionStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.qhorus.runtime.dashboard.QhorusDashboardService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class CaseBrowserServiceTest {

    CaseInstanceRepository caseRepo = mock(CaseInstanceRepository.class);
    SessionRegistry sessionRegistry;
    QhorusDashboardService dashboardService = mock(QhorusDashboardService.class);
    TenantContext tenantContext = mock(TenantContext.class);
    CaseBrowserService service;

    @BeforeEach
    void setUp() {
        sessionRegistry = new SessionRegistry(tenantContext);
        when(tenantContext.currentTenantId()).thenReturn("default");
        when(dashboardService.listChannels()).thenReturn(List.of());
        service = new CaseBrowserService(caseRepo, sessionRegistry, dashboardService, tenantContext);
    }

    @Test
    void listCases_empty() {
        when(caseRepo.findAll("default")).thenReturn(List.of());
        var result = service.listCases();
        assertTrue(result.isEmpty());
    }

    @Test
    void listCases_withActiveWorkers() {
        var ci = caseInstance(UUID.randomUUID(), CaseStatus.RUNNING, "pr-review");
        when(caseRepo.findAll("default")).thenReturn(List.of(ci));
        sessionRegistry.register(session("s1", ci.getUuid().toString(), "reviewer"));

        var result = service.listCases();
        assertEquals(1, result.size());
        assertEquals("RUNNING", result.get(0).status());
        assertEquals("pr-review", result.get(0).definitionName());
        assertEquals(1, result.get(0).activeWorkerCount());
    }

    @Test
    void getCaseDetail_notFound() {
        when(caseRepo.findByUuid(any(), eq("default"))).thenReturn(null);
        assertTrue(service.getCaseDetail(UUID.randomUUID()).isEmpty());
    }

    @Test
    void getCaseDetail_withWorkers() {
        var uuid = UUID.randomUUID();
        var ci = caseInstance(uuid, CaseStatus.RUNNING, "investigation");
        when(caseRepo.findByUuid(uuid, "default")).thenReturn(ci);
        sessionRegistry.register(session("s1", uuid.toString(), "analyst"));

        var result = service.getCaseDetail(uuid);
        assertTrue(result.isPresent());
        assertEquals(1, result.get().workers().size());
        assertEquals("analyst", result.get().workers().get(0).roleName());
    }

    private CaseInstance caseInstance(UUID uuid, CaseStatus status, String name) {
        var ci = new CaseInstance();
        ci.setUuid(uuid);
        ci.setState(status);
        var meta = new CaseMetaModel();
        meta.setName(name);
        ci.setCaseMetaModel(meta);
        return ci;
    }

    private Session session(String id, String caseId, String role) {
        return new Session(id, "session-" + id, "/work", "claude", SessionStatus.ACTIVE,
                Instant.now(), Instant.now(), Optional.empty(),
                Optional.of(caseId), Optional.of(role), "default");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=CaseBrowserServiceTest`
Expected: FAIL — CaseBrowserService does not exist

- [ ] **Step 4: Implement CaseBrowserService**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/browser/CaseBrowserService.java
package io.casehub.claudony.casehub.browser;

import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TenantContext;
import io.casehub.claudony.server.model.Session;
import io.casehub.claudony.server.model.SessionStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.qhorus.runtime.dashboard.QhorusDashboardService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

@ApplicationScoped
public class CaseBrowserService {

    private final CaseInstanceRepository caseRepo;
    private final SessionRegistry sessionRegistry;
    private final QhorusDashboardService dashboardService;
    private final TenantContext tenantContext;

    @Inject
    public CaseBrowserService(CaseInstanceRepository caseRepo,
                               SessionRegistry sessionRegistry,
                               QhorusDashboardService dashboardService,
                               TenantContext tenantContext) {
        this.caseRepo = caseRepo;
        this.sessionRegistry = sessionRegistry;
        this.dashboardService = dashboardService;
        this.tenantContext = tenantContext;
    }

    public List<CaseSummary> listCases() {
        String tenancyId = tenantContext.currentTenantId();
        List<CaseInstance> cases = caseRepo.findAll(tenancyId);
        var channelsByCase = channelCountsByCase();

        return cases.stream().map(ci -> {
            String caseId = ci.getUuid().toString();
            List<Session> sessions = sessionRegistry.findByCaseId(caseId);
            long activeCount = sessions.stream()
                    .filter(s -> s.status() == SessionStatus.ACTIVE)
                    .count();
            Instant lastActivity = sessions.stream()
                    .map(Session::lastActive)
                    .max(Instant::compareTo)
                    .orElse(Instant.EPOCH);
            String name = ci.getCaseMetaModel() != null ? ci.getCaseMetaModel().getName() : "unknown";

            return new CaseSummary(
                    ci.getUuid(),
                    ci.getState().name(),
                    name,
                    (int) activeCount,
                    channelsByCase.getOrDefault(caseId, 0),
                    lastActivity
            );
        }).toList();
    }

    public Optional<CaseDetail> getCaseDetail(UUID caseId) {
        String tenancyId = tenantContext.currentTenantId();
        CaseInstance ci = caseRepo.findByUuid(caseId, tenancyId);
        if (ci == null) return Optional.empty();

        List<Session> sessions = sessionRegistry.findByCaseId(caseId.toString());
        List<WorkerInfo> workers = sessions.stream()
                .map(s -> new WorkerInfo(
                        s.id(), s.roleName().orElse("unknown"),
                        s.status().name(), s.lastActive(),
                        s.status() == SessionStatus.ACTIVE))
                .toList();

        String prefix = "case-" + caseId + "/";
        List<String> channels = dashboardService.listChannels().stream()
                .map(ch -> ch.name())
                .filter(n -> n.startsWith(prefix))
                .toList();

        List<Map<String, Object>> timeline = channels.isEmpty()
                ? List.of()
                : dashboardService.getTimeline(channels.get(0), null, 50);

        String name = ci.getCaseMetaModel() != null ? ci.getCaseMetaModel().getName() : "unknown";
        Instant lastActivity = sessions.stream()
                .map(Session::lastActive)
                .max(Instant::compareTo)
                .orElse(Instant.EPOCH);

        return Optional.of(new CaseDetail(
                ci.getUuid(), ci.getState().name(), name,
                workers, channels, timeline, lastActivity));
    }

    private Map<String, Integer> channelCountsByCase() {
        return dashboardService.listChannels().stream()
                .map(ch -> ch.name())
                .filter(n -> n.startsWith("case-"))
                .collect(Collectors.groupingBy(
                        n -> n.substring(5, n.indexOf('/')),
                        Collectors.collectingAndThen(Collectors.counting(), Long::intValue)));
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=CaseBrowserServiceTest`
Expected: PASS (4 tests)

- [ ] **Step 6: Commit**

```
feat(casehub): CaseBrowserService — aggregate case data from engine + sessions + channels (#176)

CaseSummary (list) and CaseDetail (detail) DTOs. Service composes
CaseInstanceRepository, SessionRegistry, and QhorusDashboardService.
4 unit tests.

Refs #176
```

---

### Task 3: CaseBrowserResource

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/CaseBrowserResource.java`
- Test: `app/src/test/java/io/casehub/claudony/server/CaseBrowserResourceTest.java`

**Interfaces:**
- Consumes: `CaseBrowserService.listCases()` → `List<CaseSummary>`, `CaseBrowserService.getCaseDetail(UUID)` → `Optional<CaseDetail>`
- Produces: `GET /api/cases` → JSON `{cases: [...], totalCount: N}`, `GET /api/cases/{id}` → JSON CaseDetail

- [ ] **Step 1: Write failing tests**

```java
// app/src/test/java/io/casehub/claudony/server/CaseBrowserResourceTest.java
package io.casehub.claudony.server;

import io.casehub.claudony.casehub.browser.CaseBrowserService;
import io.casehub.claudony.casehub.browser.CaseDetail;
import io.casehub.claudony.casehub.browser.CaseSummary;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.hamcrest.Matchers.*;
import static org.mockito.Mockito.when;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class CaseBrowserResourceTest {

    @InjectMock
    CaseBrowserService caseBrowserService;

    @Test
    void listCases_empty() {
        when(caseBrowserService.listCases()).thenReturn(List.of());
        RestAssured.given()
                .when().get("/api/cases")
                .then().statusCode(200)
                .body("cases", hasSize(0))
                .body("totalCount", is(0));
    }

    @Test
    void listCases_populated() {
        var summary = new CaseSummary(UUID.randomUUID(), "RUNNING", "pr-review", 2, 3, Instant.now());
        when(caseBrowserService.listCases()).thenReturn(List.of(summary));
        RestAssured.given()
                .when().get("/api/cases")
                .then().statusCode(200)
                .body("cases", hasSize(1))
                .body("cases[0].status", is("RUNNING"))
                .body("cases[0].definitionName", is("pr-review"))
                .body("totalCount", is(1));
    }

    @Test
    void getCaseDetail_notFound() {
        when(caseBrowserService.getCaseDetail(org.mockito.ArgumentMatchers.any()))
                .thenReturn(Optional.empty());
        RestAssured.given()
                .when().get("/api/cases/" + UUID.randomUUID())
                .then().statusCode(404);
    }

    @Test
    void getCaseDetail_found() {
        var uuid = UUID.randomUUID();
        var detail = new CaseDetail(uuid, "RUNNING", "investigation",
                List.of(), List.of("case-" + uuid + "/work"), List.of(), Instant.now());
        when(caseBrowserService.getCaseDetail(uuid)).thenReturn(Optional.of(detail));
        RestAssured.given()
                .when().get("/api/cases/" + uuid)
                .then().statusCode(200)
                .body("id", is(uuid.toString()))
                .body("definitionName", is("investigation"));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CaseBrowserResourceTest`
Expected: FAIL — CaseBrowserResource does not exist

- [ ] **Step 3: Implement CaseBrowserResource**

```java
// app/src/main/java/io/casehub/claudony/server/CaseBrowserResource.java
package io.casehub.claudony.server;

import io.casehub.claudony.casehub.browser.CaseBrowserService;
import io.casehub.claudony.casehub.browser.CaseSummary;
import io.smallrye.common.annotation.Blocking;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.List;
import java.util.Map;
import java.util.UUID;

@Path("/api/cases")
@Produces(MediaType.APPLICATION_JSON)
public class CaseBrowserResource {

    @Inject
    CaseBrowserService caseBrowserService;

    @GET
    @Blocking
    public Map<String, Object> listCases() {
        List<CaseSummary> cases = caseBrowserService.listCases();
        return Map.of("cases", cases, "totalCount", cases.size());
    }

    @GET
    @Path("/{id}")
    @Blocking
    public Response getCaseDetail(@PathParam("id") UUID id) {
        return caseBrowserService.getCaseDetail(id)
                .map(detail -> Response.ok(detail).build())
                .orElse(Response.status(404).build());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CaseBrowserResourceTest`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```
feat(api): GET /api/cases — case browser REST endpoints (#176)

List and detail endpoints backed by CaseBrowserService.
4 QuarkusTest integration tests.

Refs #176
```

---

### Task 4: Case browser frontend component

**Files:**
- Create: `app/src/main/webui/src/components/claudony-case-browser.ts`
- Create: `app/src/main/webui/src/components/claudony-case-browser.test.ts`
- Modify: `app/src/main/webui/src/app.ts` (register panel)

**Interfaces:**
- Consumes: `GET /api/cases` → `{cases: CaseSummary[], totalCount}`, `GET /api/cases/{id}` → CaseDetail, `blocks-case-explorer` component and types from `@casehubio/blocks-ui-case-explorer`
- Produces: `<claudony-case-browser>` custom element registered as `case-browser` panel

- [ ] **Step 1: Write failing test**

```typescript
// app/src/main/webui/src/components/claudony-case-browser.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import './claudony-case-browser';

describe('claudony-case-browser', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('claudony-case-browser');
    document.body.appendChild(el);
  });

  it('renders as a custom element', () => {
    expect(el).toBeDefined();
    expect(el.tagName.toLowerCase()).toBe('claudony-case-browser');
  });

  it('configures case entity type with correct endpoints', async () => {
    await (el as any).updateComplete;
    const explorer = el.shadowRoot?.querySelector('blocks-case-explorer');
    expect(explorer).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm --prefix app/src/main/webui test -- --run claudony-case-browser`
Expected: FAIL — component does not exist

- [ ] **Step 3: Implement claudony-case-browser.ts**

```typescript
// app/src/main/webui/src/components/claudony-case-browser.ts
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import type { EntityTypeRegistration } from '@casehubio/blocks-ui-case-explorer';
import '@casehubio/blocks-ui-case-explorer';
import { authenticatedFetch } from '../util/auth';

const CASE_ENTITY_TYPE: EntityTypeRegistration = {
  type: 'case',
  label: 'Cases',
  listEndpoint: '/api/cases',
  detailEndpoint: (id: string) => `/api/cases/${id}`,
  columnConfig: [
    { id: 'status', name: 'Status', sortable: true, width: '100px' },
    { id: 'definitionName', name: 'Type', sortable: true, width: '1fr' },
    { id: 'activeWorkerCount', name: 'Workers', width: '80px' },
    { id: 'channelCount', name: 'Channels', width: '80px' },
    { id: 'lastActivity', name: 'Last Activity', sortable: true, width: '1fr' },
  ],
  reader: {
    id: (c: any) => c.id,
    summary: (c: any) => c.definitionName,
    status: (c: any) => c.status,
  },
  responseReader: {
    entities: (r: any) => r.cases,
    totalCount: (r: any) => r.totalCount,
  },
  filters: [
    { field: 'status', label: 'Status', type: 'status' as const, options: [
      { value: 'RUNNING', label: 'Active' },
      { value: 'COMPLETED', label: 'Completed' },
      { value: 'FAULTED', label: 'Faulted' },
    ]},
  ],
};

@customElement('claudony-case-browser')
export class ClaudonyCaseBrowser extends LitElement {
  static override styles = css`
    :host { display: block; height: 100%; }
  `;

  override render() {
    return html`
      <blocks-case-explorer
        .entityTypes=${[CASE_ENTITY_TYPE]}
        .fetchFn=${authenticatedFetch}
      ></blocks-case-explorer>
    `;
  }
}
```

- [ ] **Step 4: Wire into app.ts**

Add import and panel registration to `app/src/main/webui/src/app.ts`:

```typescript
import "./components/claudony-case-browser";
// ... after existing registerPanel calls:
registerPanel("case-browser", "claudony-case-browser");
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm --prefix app/src/main/webui test -- --run claudony-case-browser`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(ui): case browser component — blocks-case-explorer composition (#176)

Thin Lit wrapper configuring blocks-case-explorer with case entity type
registration. Wired into fleet home Cases tab.

Refs #176
```

---

### Task 5: ActionItem model, StallTracker, and ActionAggregationService

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionItem.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionDescriptor.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/SourceType.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/Urgency.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/StallTracker.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionAggregationService.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/inbox/StallTrackerTest.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/inbox/ActionAggregationServiceTest.java`

**Interfaces:**
- Consumes: `CommitmentStore.findAllOpen()` → `List<Commitment>`, `SessionRegistry.all()`, `StallTracker.stalledWorkerIds()` → `Set<String>`, `QhorusDashboardService.listChannels()`, `ChannelService.findByName(String)`, `WorkerSessionMapping.findByRole(String)` → `Optional<String>`
- Produces: `ActionAggregationService.listActions()` → `ActionInboxResponse(List<ActionItem>, ActionCounts)`, `StallTracker.markStalled(String)`, `StallTracker.clearStall(String)`, `StallTracker.isStalled(String)` → `boolean`

- [ ] **Step 1: Create enums and DTOs**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/inbox/SourceType.java
package io.casehub.claudony.casehub.inbox;
public enum SourceType { COMMITMENT, STALL, OVERSIGHT, WORKITEM }

// casehub/src/main/java/io/casehub/claudony/casehub/inbox/Urgency.java
package io.casehub.claudony.casehub.inbox;
public enum Urgency { HIGH, MEDIUM, LOW }

// casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionDescriptor.java
package io.casehub.claudony.casehub.inbox;
public record ActionDescriptor(String name, String label, String method, String endpoint) {}

// casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionItem.java
package io.casehub.claudony.casehub.inbox;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record ActionItem(
    String id,
    SourceType sourceType,
    Urgency urgency,
    String title,
    String status,
    boolean actionable,
    UUID caseId,
    String channelName,
    Instant createdAt,
    List<ActionDescriptor> actions
) {}

// casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionCounts.java
package io.casehub.claudony.casehub.inbox;
public record ActionCounts(int high, int medium, int low) {}

// casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionInboxResponse.java
package io.casehub.claudony.casehub.inbox;
import java.util.List;
public record ActionInboxResponse(List<ActionItem> items, ActionCounts counts) {}
```

- [ ] **Step 2: Write StallTracker tests**

```java
// casehub/src/test/java/io/casehub/claudony/casehub/inbox/StallTrackerTest.java
package io.casehub.claudony.casehub.inbox;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class StallTrackerTest {

    StallTracker tracker = new StallTracker();

    @Test
    void initiallyEmpty() {
        assertTrue(tracker.stalledWorkerIds().isEmpty());
    }

    @Test
    void markStalled() {
        tracker.markStalled("worker-1");
        assertTrue(tracker.isStalled("worker-1"));
        assertEquals(1, tracker.stalledWorkerIds().size());
    }

    @Test
    void clearStall() {
        tracker.markStalled("worker-1");
        tracker.clearStall("worker-1");
        assertFalse(tracker.isStalled("worker-1"));
    }

    @Test
    void clearUnknownIsNoOp() {
        tracker.clearStall("unknown");
        assertTrue(tracker.stalledWorkerIds().isEmpty());
    }
}
```

- [ ] **Step 3: Implement StallTracker**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/inbox/StallTracker.java
package io.casehub.claudony.casehub.inbox;

import io.casehub.claudony.casehub.ClaudonyWorkerStatusListener.WorkerStalledEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;

import java.util.Collections;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class StallTracker {

    private final Set<String> stalled = ConcurrentHashMap.newKeySet();

    void onStall(@Observes WorkerStalledEvent event) {
        stalled.add(event.workerId());
    }

    public void markStalled(String workerId) {
        stalled.add(workerId);
    }

    public void clearStall(String workerId) {
        stalled.remove(workerId);
    }

    public boolean isStalled(String workerId) {
        return stalled.contains(workerId);
    }

    public Set<String> stalledWorkerIds() {
        return Collections.unmodifiableSet(stalled);
    }
}
```

- [ ] **Step 4: Run StallTracker tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=StallTrackerTest`
Expected: PASS (4 tests)

- [ ] **Step 5: Write ActionAggregationService tests**

```java
// casehub/src/test/java/io/casehub/claudony/casehub/inbox/ActionAggregationServiceTest.java
package io.casehub.claudony.casehub.inbox;

import io.casehub.claudony.casehub.WorkerSessionMapping;
import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TenantContext;
import io.casehub.claudony.server.model.Session;
import io.casehub.claudony.server.model.SessionStatus;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.CommitmentStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class ActionAggregationServiceTest {

    CommitmentStore commitmentStore = mock(CommitmentStore.class);
    SessionRegistry sessionRegistry;
    StallTracker stallTracker = new StallTracker();
    WorkerSessionMapping sessionMapping = mock(WorkerSessionMapping.class);
    TenantContext tenantContext = mock(TenantContext.class);
    ActionAggregationService service;

    @BeforeEach
    void setUp() {
        when(tenantContext.currentTenantId()).thenReturn("default");
        sessionRegistry = new SessionRegistry(tenantContext);
        when(commitmentStore.findAllOpen()).thenReturn(List.of());
        service = new ActionAggregationService(commitmentStore, sessionRegistry, stallTracker, sessionMapping);
    }

    @Test
    void emptyWhenNoSources() {
        var result = service.listActions();
        assertTrue(result.items().isEmpty());
        assertEquals(0, result.counts().high());
    }

    @Test
    void commitmentMappedToActionItem() {
        var commitment = Commitment.builder()
                .id(UUID.randomUUID())
                .correlationId("corr-1")
                .channelId(UUID.randomUUID())
                .messageType(MessageType.COMMAND)
                .requester("agent-1")
                .obligor("human-1")
                .state(CommitmentState.OPEN)
                .createdAt(Instant.now())
                .build();
        when(commitmentStore.findAllOpen()).thenReturn(List.of(commitment));

        var result = service.listActions();
        assertEquals(1, result.items().size());
        assertEquals(SourceType.COMMITMENT, result.items().get(0).sourceType());
        assertEquals(Urgency.LOW, result.items().get(0).urgency());
        assertTrue(result.items().get(0).actionable());
    }

    @Test
    void overdueCommitmentIsHighUrgency() {
        var commitment = Commitment.builder()
                .id(UUID.randomUUID())
                .correlationId("corr-1")
                .channelId(UUID.randomUUID())
                .messageType(MessageType.COMMAND)
                .requester("agent-1")
                .state(CommitmentState.OPEN)
                .expiresAt(Instant.now().minusSeconds(3600))
                .createdAt(Instant.now().minusSeconds(7200))
                .build();
        when(commitmentStore.findAllOpen()).thenReturn(List.of(commitment));

        var result = service.listActions();
        assertEquals(Urgency.HIGH, result.items().get(0).urgency());
        assertEquals(1, result.counts().high());
    }

    @Test
    void stalledWorkerMappedToHighUrgency() {
        var session = new Session("s1", "worker-stalled", "/work", "claude",
                SessionStatus.ACTIVE, Instant.now(), Instant.now(), Optional.empty(),
                Optional.of(UUID.randomUUID().toString()), Optional.of("reviewer"), "default");
        sessionRegistry.register(session);
        stallTracker.markStalled("reviewer");
        when(sessionMapping.findByRole("reviewer")).thenReturn(Optional.of("s1"));

        var result = service.listActions();
        var stalls = result.items().stream()
                .filter(a -> a.sourceType() == SourceType.STALL)
                .toList();
        assertEquals(1, stalls.size());
        assertEquals(Urgency.HIGH, stalls.get(0).urgency());
    }

    @Test
    void sortedByUrgencyThenCreatedAt() {
        var oldCommitment = Commitment.builder()
                .id(UUID.randomUUID()).correlationId("c1").channelId(UUID.randomUUID())
                .messageType(MessageType.COMMAND).requester("a").state(CommitmentState.OPEN)
                .createdAt(Instant.now().minusSeconds(100)).build();
        var newCommitment = Commitment.builder()
                .id(UUID.randomUUID()).correlationId("c2").channelId(UUID.randomUUID())
                .messageType(MessageType.COMMAND).requester("b").state(CommitmentState.OPEN)
                .createdAt(Instant.now()).build();
        when(commitmentStore.findAllOpen()).thenReturn(List.of(oldCommitment, newCommitment));

        stallTracker.markStalled("stalled-role");
        var stalledSession = new Session("s2", "stalled", "/work", "claude",
                SessionStatus.ACTIVE, Instant.now(), Instant.now(), Optional.empty(),
                Optional.of(UUID.randomUUID().toString()), Optional.of("stalled-role"), "default");
        sessionRegistry.register(stalledSession);
        when(sessionMapping.findByRole("stalled-role")).thenReturn(Optional.of("s2"));

        var result = service.listActions();
        assertEquals(SourceType.STALL, result.items().get(0).sourceType());
    }
}
```

- [ ] **Step 6: Implement ActionAggregationService**

```java
// casehub/src/main/java/io/casehub/claudony/casehub/inbox/ActionAggregationService.java
package io.casehub.claudony.casehub.inbox;

import io.casehub.claudony.casehub.WorkerSessionMapping;
import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.model.Session;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.store.CommitmentStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.stream.Stream;

@ApplicationScoped
public class ActionAggregationService {

    private final CommitmentStore commitmentStore;
    private final SessionRegistry sessionRegistry;
    private final StallTracker stallTracker;
    private final WorkerSessionMapping sessionMapping;

    @Inject
    public ActionAggregationService(CommitmentStore commitmentStore,
                                     SessionRegistry sessionRegistry,
                                     StallTracker stallTracker,
                                     WorkerSessionMapping sessionMapping) {
        this.commitmentStore = commitmentStore;
        this.sessionRegistry = sessionRegistry;
        this.stallTracker = stallTracker;
        this.sessionMapping = sessionMapping;
    }

    public ActionInboxResponse listActions() {
        List<ActionItem> items = new ArrayList<>();
        items.addAll(commitmentItems());
        items.addAll(stallItems());
        items.sort(Comparator.comparing(ActionItem::urgency)
                .thenComparing(ActionItem::createdAt, Comparator.reverseOrder()));

        long high = items.stream().filter(a -> a.urgency() == Urgency.HIGH).count();
        long medium = items.stream().filter(a -> a.urgency() == Urgency.MEDIUM).count();
        long low = items.stream().filter(a -> a.urgency() == Urgency.LOW).count();
        return new ActionInboxResponse(items, new ActionCounts((int) high, (int) medium, (int) low));
    }

    private List<ActionItem> commitmentItems() {
        return commitmentStore.findAllOpen().stream()
                .map(this::toCommitmentAction)
                .toList();
    }

    private ActionItem toCommitmentAction(Commitment c) {
        Urgency urgency = commitmentUrgency(c);
        return new ActionItem(
                "commitment:" + c.id(),
                SourceType.COMMITMENT,
                urgency,
                c.messageType().name() + " from " + c.requester(),
                c.state().name(),
                c.state().isActive(),
                null,
                null,
                c.createdAt() != null ? c.createdAt() : Instant.now(),
                List.of(
                        new ActionDescriptor("accept", "Accept", "POST",
                                "/api/actions/commitment:" + c.id() + "/execute/accept"),
                        new ActionDescriptor("decline", "Decline", "POST",
                                "/api/actions/commitment:" + c.id() + "/execute/decline")
                )
        );
    }

    private Urgency commitmentUrgency(Commitment c) {
        if (c.expiresAt() == null) return Urgency.LOW;
        Instant now = Instant.now();
        if (c.expiresAt().isBefore(now)) return Urgency.HIGH;
        if (Duration.between(now, c.expiresAt()).toMinutes() < 60) return Urgency.MEDIUM;
        return Urgency.LOW;
    }

    private List<ActionItem> stallItems() {
        List<ActionItem> items = new ArrayList<>();
        for (String workerId : stallTracker.stalledWorkerIds()) {
            sessionMapping.findByRole(workerId)
                    .flatMap(sessionRegistry::findUnscoped)
                    .ifPresent(session -> items.add(toStallAction(session, workerId)));
        }
        return items;
    }

    private ActionItem toStallAction(Session session, String workerId) {
        UUID caseId = session.caseId().map(UUID::fromString).orElse(null);
        return new ActionItem(
                "stall:" + session.id(),
                SourceType.STALL,
                Urgency.HIGH,
                "Worker stalled: " + workerId,
                "STALLED",
                true,
                caseId,
                null,
                session.lastActive(),
                List.of(
                        new ActionDescriptor("view", "View Terminal", "GET",
                                "/app/session.html?id=" + session.id()),
                        new ActionDescriptor("interjection", "Send Interjection", "POST",
                                "/api/actions/stall:" + session.id() + "/execute/interjection")
                )
        );
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest="StallTrackerTest,ActionAggregationServiceTest"`
Expected: PASS (4 + 5 = 9 tests)

- [ ] **Step 8: Commit**

```
feat(casehub): ActionItem model + StallTracker + ActionAggregationService (#176)

Unified ActionItem abstraction over Qhorus commitments and stalled
workers. StallTracker observes WorkerStalledEvent CDI events.
ActionAggregationService aggregates and sorts by urgency.
9 unit tests.

Refs #176
```

---

### Task 6: ActionInboxResource

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/ActionInboxResource.java`
- Test: `app/src/test/java/io/casehub/claudony/server/ActionInboxResourceTest.java`

**Interfaces:**
- Consumes: `ActionAggregationService.listActions()` → `ActionInboxResponse`
- Produces: `GET /api/actions` → JSON `{items: [...], counts: {high, medium, low}}`

- [ ] **Step 1: Write failing tests**

```java
// app/src/test/java/io/casehub/claudony/server/ActionInboxResourceTest.java
package io.casehub.claudony.server;

import io.casehub.claudony.casehub.inbox.*;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.hamcrest.Matchers.*;
import static org.mockito.Mockito.when;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class ActionInboxResourceTest {

    @InjectMock
    ActionAggregationService aggregationService;

    @Test
    void listActions_empty() {
        when(aggregationService.listActions())
                .thenReturn(new ActionInboxResponse(List.of(), new ActionCounts(0, 0, 0)));
        RestAssured.given()
                .when().get("/api/actions")
                .then().statusCode(200)
                .body("items", hasSize(0))
                .body("counts.high", is(0));
    }

    @Test
    void listActions_withItems() {
        var item = new ActionItem("commitment:123", SourceType.COMMITMENT,
                Urgency.HIGH, "COMMAND from agent", "OPEN", true,
                null, null, Instant.now(), List.of());
        when(aggregationService.listActions())
                .thenReturn(new ActionInboxResponse(List.of(item), new ActionCounts(1, 0, 0)));
        RestAssured.given()
                .when().get("/api/actions")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].sourceType", is("COMMITMENT"))
                .body("items[0].urgency", is("HIGH"))
                .body("counts.high", is(1));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ActionInboxResourceTest`
Expected: FAIL — ActionInboxResource does not exist

- [ ] **Step 3: Implement ActionInboxResource**

```java
// app/src/main/java/io/casehub/claudony/server/ActionInboxResource.java
package io.casehub.claudony.server;

import io.casehub.claudony.casehub.inbox.ActionAggregationService;
import io.casehub.claudony.casehub.inbox.ActionInboxResponse;
import io.smallrye.common.annotation.Blocking;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;

@Path("/api/actions")
@Produces(MediaType.APPLICATION_JSON)
public class ActionInboxResource {

    @Inject
    ActionAggregationService aggregationService;

    @GET
    @Blocking
    public ActionInboxResponse listActions() {
        return aggregationService.listActions();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ActionInboxResourceTest`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```
feat(api): GET /api/actions — unified action inbox endpoint (#176)

Returns ActionItem list with urgency counts, aggregated from
commitments and stalled workers. 2 QuarkusTest integration tests.

Refs #176
```

---

### Task 7: Action inbox frontend component

**Files:**
- Create: `app/src/main/webui/src/components/claudony-action-inbox.ts`
- Create: `app/src/main/webui/src/components/claudony-action-inbox.test.ts`
- Modify: `app/src/main/webui/src/app.ts` (register panel)

**Interfaces:**
- Consumes: `GET /api/actions` → `ActionInboxResponse`, `pages-table` from `@casehubio/pages-table`
- Produces: `<claudony-action-inbox>` custom element registered as `action-inbox` panel

- [ ] **Step 1: Write failing test**

```typescript
// app/src/main/webui/src/components/claudony-action-inbox.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import './claudony-action-inbox';

describe('claudony-action-inbox', () => {
  let el: any;

  beforeEach(() => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({
        items: [
          { id: 'stall:s1', sourceType: 'STALL', urgency: 'HIGH',
            title: 'Worker stalled: reviewer', status: 'STALLED',
            actionable: true, createdAt: new Date().toISOString(), actions: [] },
          { id: 'commitment:c1', sourceType: 'COMMITMENT', urgency: 'LOW',
            title: 'COMMAND from agent', status: 'OPEN',
            actionable: true, createdAt: new Date().toISOString(), actions: [] },
        ],
        counts: { high: 1, medium: 0, low: 1 },
      }),
    } as any);
    el = document.createElement('claudony-action-inbox');
    document.body.appendChild(el);
  });

  it('renders as a custom element', () => {
    expect(el.tagName.toLowerCase()).toBe('claudony-action-inbox');
  });

  it('fetches actions on connect', async () => {
    await el.updateComplete;
    expect(globalThis.fetch).toHaveBeenCalledWith(
      expect.stringContaining('/api/actions'),
      expect.any(Object),
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm --prefix app/src/main/webui test -- --run claudony-action-inbox`
Expected: FAIL — component does not exist

- [ ] **Step 3: Implement claudony-action-inbox.ts**

```typescript
// app/src/main/webui/src/components/claudony-action-inbox.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { authenticatedFetch } from '../util/auth';
import { relativeTime } from '../util/time';

interface ActionItem {
  id: string;
  sourceType: 'COMMITMENT' | 'STALL' | 'OVERSIGHT' | 'WORKITEM';
  urgency: 'HIGH' | 'MEDIUM' | 'LOW';
  title: string;
  status: string;
  actionable: boolean;
  caseId?: string;
  channelName?: string;
  createdAt: string;
  actions: { name: string; label: string; method: string; endpoint: string }[];
}

interface ActionCounts { high: number; medium: number; low: number; }
interface ActionInboxResponse { items: ActionItem[]; counts: ActionCounts; }

const URGENCY_ICON: Record<string, string> = { HIGH: '🔴', MEDIUM: '🟡', LOW: '⚪' };
const SOURCE_ICON: Record<string, string> = {
  COMMITMENT: '📋', STALL: '⚠️', OVERSIGHT: '👁️', WORKITEM: '📝',
};

@customElement('claudony-action-inbox')
export class ClaudonyActionInbox extends LitElement {
  @state() private items: ActionItem[] = [];
  @state() private counts: ActionCounts = { high: 0, medium: 0, low: 0 };
  @state() private loading = true;
  @state() private error: string | null = null;
  private eventSource: EventSource | null = null;

  static override styles = css`
    :host { display: block; height: 100%; overflow-y: auto; padding: 12px; }
    .summary { display: flex; gap: 16px; padding: 8px 0; font-size: 0.875rem;
               border-bottom: 1px solid var(--pages-neutral-5, #333); margin-bottom: 8px; }
    .summary .count { font-weight: 600; }
    table { width: 100%; border-collapse: collapse; font-size: 0.8125rem; }
    th { text-align: left; padding: 6px 8px; border-bottom: 1px solid var(--pages-neutral-5, #333);
         color: var(--pages-neutral-9, #999); font-weight: 500; }
    td { padding: 8px; border-bottom: 1px solid var(--pages-neutral-4, #222); }
    tr:hover { background: var(--pages-neutral-4, #1a1a1a); }
    .urgency { width: 30px; text-align: center; }
    .source { width: 30px; text-align: center; }
    .actions { display: flex; gap: 4px; }
    .actions button { padding: 2px 8px; border: 1px solid var(--pages-neutral-5, #555);
                      background: var(--pages-neutral-3, #222); color: inherit;
                      border-radius: 3px; cursor: pointer; font-size: 0.75rem; }
    .actions button:hover { background: var(--pages-accent-9, #0066cc); border-color: var(--pages-accent-9); }
    .empty { padding: 32px; text-align: center; color: var(--pages-neutral-9, #666); }
    .error { padding: 12px; color: var(--pages-danger-9, #cc3333); }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.fetchActions();
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this.eventSource?.close();
  }

  private async fetchActions(): Promise<void> {
    this.loading = true;
    try {
      const res = await authenticatedFetch('/api/actions', { headers: { Accept: 'application/json' } });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const data: ActionInboxResponse = await res.json();
      this.items = data.items;
      this.counts = data.counts;
      this.error = null;
    } catch (e: any) {
      this.error = e.message;
    } finally {
      this.loading = false;
    }
  }

  override render() {
    if (this.error) return html`<div class="error">Error: ${this.error}</div>`;
    if (this.loading) return html`<div class="empty">Loading...</div>`;
    return html`
      <div class="summary">
        <span>🔴 <span class="count">${this.counts.high}</span> urgent</span>
        <span>🟡 <span class="count">${this.counts.medium}</span> pending</span>
        <span>${this.items.length} total</span>
      </div>
      ${this.items.length === 0
        ? html`<div class="empty">No actions pending</div>`
        : html`
          <table>
            <thead><tr>
              <th class="urgency"></th><th class="source"></th>
              <th>Title</th><th>Status</th><th>Age</th><th>Actions</th>
            </tr></thead>
            <tbody>
              ${this.items.map(item => html`
                <tr>
                  <td class="urgency">${URGENCY_ICON[item.urgency] ?? ''}</td>
                  <td class="source">${SOURCE_ICON[item.sourceType] ?? ''}</td>
                  <td>${item.title}</td>
                  <td>${item.status}</td>
                  <td>${relativeTime(item.createdAt)}</td>
                  <td class="actions">
                    ${item.actions.map(a => html`
                      <button @click=${() => this.executeAction(a)}>${a.label}</button>
                    `)}
                  </td>
                </tr>
              `)}
            </tbody>
          </table>`}
    `;
  }

  private async executeAction(action: { method: string; endpoint: string }): Promise<void> {
    if (action.method === 'GET') {
      window.location.href = action.endpoint;
      return;
    }
    try {
      await authenticatedFetch(action.endpoint, { method: action.method });
      await this.fetchActions();
    } catch (e) {
      console.error('Action failed', e);
    }
  }
}
```

- [ ] **Step 4: Wire into app.ts**

Add import and panel registration to `app/src/main/webui/src/app.ts`:

```typescript
import "./components/claudony-action-inbox";
// ... after existing registerPanel calls:
registerPanel("action-inbox", "claudony-action-inbox");
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm --prefix app/src/main/webui test -- --run claudony-action-inbox`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(ui): action inbox component — unified view of items needing attention (#176)

Summary bar with urgency counts, tabular list with action buttons.
Fetches from /api/actions, supports execute-action dispatch.

Refs #176
```

---

### Task 8: Integration test and full-stack verification

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/e2e/CaseBrowserE2ETest.java` (optional — requires Playwright + running server)
- Modify: `app/src/test/java/io/casehub/claudony/server/McpServerIntegrationTest.java` (verify new endpoints don't break tool count)

**Interfaces:**
- Consumes: All prior tasks
- Produces: End-to-end validation

- [ ] **Step 1: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All existing tests pass, plus ~15 new tests from Tasks 2-6

- [ ] **Step 2: Run frontend tests**

Run: `npm --prefix app/src/main/webui test`
Expected: All existing vitest tests pass, plus ~4 new tests from Tasks 1, 4, 7

- [ ] **Step 3: Verify McpServerIntegrationTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=McpServerIntegrationTest`
Expected: PASS — `fullHandshakeSequence_asClaudeWouldSendIt` still asserts 8 tools at `/mcp` (case browser REST doesn't add MCP tools)

- [ ] **Step 4: Final commit**

```
test: full-stack verification — case browser + action inbox (#176)

All backend tests pass (CaseBrowserService, CaseBrowserResource,
StallTracker, ActionAggregationService, ActionInboxResource).
All frontend tests pass. MCP tool count unchanged.

Refs #176
```

---

## Phase 3: WorkItem Integration (Follow-On)

Phase 3 adds `casehub-work` as a compile dependency to `app/pom.xml` and extends `ActionAggregationService` with a fourth source:

1. Add `<dependency><groupId>io.casehub</groupId><artifactId>casehub-work</artifactId></dependency>` to `app/pom.xml`
2. Inject `WorkBroker` in `ActionAggregationService` (guarded by `@IfBuildProperty(name = "claudony.casehub.enabled", stringValue = "true")`)
3. Query `WorkBroker.findAssigned(actorId, tenancyId)` for active WorkItems
4. Map to `ActionItem(sourceType=WORKITEM)` with SLA-based urgency
5. Add claim/delegate/complete routing to `ActionInboxResource.execute()`
6. Extend tests

This is a natural follow-on once Phases 1-2 are validated. Track as a child issue of #176 if implementation is deferred.
