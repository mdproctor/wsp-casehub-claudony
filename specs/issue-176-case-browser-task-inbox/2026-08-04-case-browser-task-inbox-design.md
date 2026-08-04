# Case Browser and Task Inbox — Design Spec

**Issue:** casehubio/claudony#176
**Branch:** `issue-176-case-browser-task-inbox`
**Date:** 2026-08-04

## Problem

Claudony manages Claude Code sessions and provides a browser-based workspace with terminal, observation, and conversation layers. The session view answers "I have a Claude running" and the channel view answers "I can see what it's doing." Two layers are missing:

- **Context** — "I understand why it's doing it" — seeing cases across the system, not scoped to a single session
- **Action** — "I can intervene, delegate, respond" — a unified inbox of things needing human attention

## Approach

Compose existing platform components rather than build from scratch.

**Case browser:** Use `blocks-case-explorer` (blocks-ui) — a generic, data-driven entity explorer with configurable entity types, list/tree views, split-pane detail, breadcrumb navigation, filters, and relationship drilling. Claudony provides REST endpoints that aggregate case data from multiple sources and configures `EntityTypeRegistration` with the right endpoints, columns, and renderers.

**Action inbox:** Define a new `ActionItem` abstraction — a view-layer unification of Qhorus commitments, stalled workers, oversight messages, and (later) WorkItems. Each source keeps its own lifecycle; the inbox sorts by urgency and routes actions back to the appropriate backend. A lightweight `claudony-action-inbox.ts` component uses `pages-table` for rendering.

**Extraction path:** ActionItem is a platform pattern (every app needs "things needing human attention from multiple sources"). Build in Claudony first to validate, then extract `blocks-action-inbox` component to blocks-ui and `ActionSource` SPI to blocks (Java) as a follow-on issue.

## Phasing

| Phase | Scope | New deps |
|-------|-------|----------|
| 1 | Case browser — blocks-case-explorer + REST endpoints | None |
| 2 | Action inbox — ActionItem abstraction + commitments + stalls + oversight | None |
| 3 | WorkItem integration — add casehub-work, extend inbox | `casehub-work` |

Single branch, phased commits. Each phase is independently testable.

## Dashboard Integration

Fleet home (`app.ts`) gains a tab bar:

```
Sessions | Cases | Inbox | Fleet | Mesh
```

Each tab lazy-loads its panel component. "Cases" loads `claudony-case-browser.ts`, "Inbox" loads `claudony-action-inbox.ts`. The inbox tab carries a badge showing the count of high + medium urgency items.

## Phase 1: Case Browser

### Data Sources

| Source | What it provides | API |
|--------|-----------------|-----|
| `CaseInstanceRepository` | Case state, status, definition name | Engine runtime (on classpath via `claudony-app`) |
| `SessionRegistry` | Active workers (sessions with caseId/roleName) | Claudony core |
| `QhorusDashboardService` | Channel count, message activity per case | Qhorus runtime |
| `CaseLineageQuery` | Completed worker history | Claudony casehub |

### Backend

**`CaseBrowserService`** in `claudony-casehub/` — aggregates case data.

**`CaseSummary`** record:

| Field | Type | Source |
|-------|------|--------|
| `id` | UUID | CaseInstanceRepository |
| `status` | String | CaseInstanceRepository (CaseStatus) |
| `definitionName` | String | CaseInstanceRepository |
| `activeWorkerCount` | int | SessionRegistry.findByCaseId() |
| `channelCount` | int | QhorusDashboardService |
| `lastActivity` | Instant | max(session touch, channel message, lifecycle event) |
| `createdAt` | Instant | CaseInstanceRepository |

**`CaseDetail`** record — extends CaseSummary with:

| Field | Type | Source |
|-------|------|--------|
| `workers` | List<WorkerInfo> | SessionRegistry + CaseLineageQuery |
| `channels` | List<ChannelInfo> | QhorusDashboardService |
| `timeline` | List<TimelineEntry> | CaseLedgerEntry |

**`CaseBrowserResource`** in `claudony-app/server/`:

| Method | Path | Returns | Notes |
|--------|------|---------|-------|
| GET | `/api/cases` | `{cases: CaseSummary[], totalCount}` | Filter: `?status=`, sort: `?sort=lastActivity` |
| GET | `/api/cases/{id}` | `CaseDetail` | Full detail for split-pane |
| GET | `/api/cases/{id}/tree` | `EntityTreeNode[]` | Case → Workers + Channels hierarchy |
| SSE | `/api/cases/events` | Case lifecycle events | Reuse CaseEventBroadcaster pattern |

All endpoints are `@Blocking` — CaseInstanceRepository, SessionRegistry, and CaseLineageQuery are blocking APIs.

### Frontend

**`claudony-case-browser.ts`** — thin Lit wrapper configuring `blocks-case-explorer`:

```typescript
const CASE_ENTITY_TYPE: EntityTypeRegistration = {
  type: 'case',
  label: 'Cases',
  listEndpoint: '/api/cases',
  detailEndpoint: (id) => `/api/cases/${id}`,
  treeEndpoint: (id) => `/api/cases/${id}/tree`,
  columnConfig: [
    { id: 'status', name: 'Status', sortable: true, width: '100px' },
    { id: 'definitionName', name: 'Type', sortable: true, width: '1fr' },
    { id: 'activeWorkerCount', name: 'Workers', width: '80px' },
    { id: 'channelCount', name: 'Channels', width: '80px' },
    { id: 'lastActivity', name: 'Last Activity', sortable: true, width: '1fr' },
  ],
  reader: {
    id: (c) => c.id,
    summary: (c) => c.definitionName,
    status: (c) => c.status,
    createdAt: (c) => c.createdAt,
  },
  responseReader: {
    entities: (r) => r.cases,
    totalCount: (r) => r.totalCount,
  },
  relationships: [
    { childType: 'worker', label: 'Workers', endpointTemplate: '/api/cases/{id}/workers' },
    { childType: 'channel', label: 'Channels', endpointTemplate: '/api/cases/{id}/channels' },
  ],
  filters: [
    { field: 'status', label: 'Status', type: 'status', options: [
      { value: 'STARTED', label: 'Active' },
      { value: 'COMPLETED', label: 'Completed' },
      { value: 'FAULTED', label: 'Faulted' },
    ]},
  ],
};
```

Custom column renderers for status badges (colour-coded by CaseStatus) and relative timestamps (via `time.ts` utilities).

Detail pane renders three sections:
1. Case timeline using `blocks-timeline`
2. Worker list (active + completed) with status and role
3. Channel list with links to workbench view

## Phase 2: Action Inbox

### Data Model

**`ActionItem`** — the unified abstraction. Something needing human attention, regardless of source.

```java
public record ActionItem(
    String id,                      // composite: "commitment:{uuid}" or "stall:{sessionId}"
    SourceType sourceType,          // COMMITMENT, STALL, OVERSIGHT, WORKITEM (Phase 3)
    Urgency urgency,                // HIGH, MEDIUM, LOW — derived, not stored
    String title,                   // human-readable summary
    String status,                  // source-native status string
    boolean actionable,             // true if user can act (not yet resolved)
    UUID caseId,                    // nullable — links to case browser
    String channelName,             // nullable — for commitments and oversight
    Instant createdAt,
    List<ActionDescriptor> actions   // what the user can do
) {}

public record ActionDescriptor(
    String name,       // "accept", "interjection", "acknowledge"
    String label,      // "Accept", "Send Interjection"
    String method,     // POST, PUT
    String endpoint    // REST endpoint to invoke
) {}

public enum SourceType { COMMITMENT, STALL, OVERSIGHT, WORKITEM }
public enum Urgency { HIGH, MEDIUM, LOW }
```

**Urgency derivation** (deterministic):

| Source | Condition | Urgency |
|--------|-----------|---------|
| STALL | Always | HIGH |
| COMMITMENT | Deadline past | HIGH |
| COMMITMENT | Deadline within 1 hour | MEDIUM |
| OVERSIGHT | Unacknowledged | MEDIUM |
| COMMITMENT | No deadline or deadline far | LOW |

**Available actions per source type:**

| Source | Actions |
|--------|---------|
| COMMITMENT | Accept, Decline, Fulfill (→ Qhorus speech act dispatch via MessageService) |
| STALL | Interjection (→ channel message), View Terminal (→ navigate) |
| OVERSIGHT | Acknowledge (→ mark read), Open Channel (→ navigate) |
| WORKITEM | Claim, Delegate, Complete (Phase 3, → casehub-work API) |

### Backend

**`ActionAggregationService`** in `claudony-casehub/` — the composition layer:

```
Source 1: QhorusDashboardService → commitments
  filter: state in {OPEN, ACCEPTED} (actionable only)
  map to ActionItem(sourceType=COMMITMENT)

Source 2: SessionRegistry → stalled sessions
  filter: status == FAULTED && caseId != null
  map to ActionItem(sourceType=STALL, urgency=HIGH)

Source 3: QhorusDashboardService → oversight channel messages
  filter: unread messages in channels with purpose "oversight"
  map to ActionItem(sourceType=OVERSIGHT, urgency=MEDIUM)
```

Sort order: HIGH first, then MEDIUM, then LOW. Within same urgency: newest first.

**`ActionInboxResource`** in `claudony-app/server/`:

| Method | Path | Returns | Notes |
|--------|------|---------|-------|
| GET | `/api/actions` | `{items: ActionItem[], counts: {high, medium, low}}` | Filter: `?sourceType=`, `?urgency=` |
| POST | `/api/actions/{id}/execute/{actionName}` | `{success, message}` | Routes to appropriate backend |
| SSE | `/api/actions/events` | Action item state changes | New stalls, commitment changes |

The `execute` endpoint parses the composite ID to determine source type and dispatches:
- `commitment:*` → `MessageService.dispatch()` with the appropriate speech act type
- `stall:*` → interjection via MeshResource or session restart
- `oversight:*` → mark message as read
- `workitem:*` (Phase 3) → casehub-work WorkBroker

### Frontend

**`claudony-action-inbox.ts`** — Lit component using `pages-table`:

- **Summary bar:** urgency counts — `{high} urgent · {medium} pending · {total} total`
- **Table columns:** urgency badge, title, source type icon, case link, age, action buttons
- **Row click:** context navigation (to channel, terminal, or case detail depending on source)
- **Action buttons:** in-row, POST to `/api/actions/{id}/execute/{action}`
- **SSE connection:** `/api/actions/events` for live updates (stalls appear immediately)
- **Polling fallback:** EventSource error detection, same pattern as channel-panel.ts
- **Tab badge:** high + medium count, updated via SSE

## Phase 3: WorkItem Integration

### Dependency

Add `casehub-work` to `claudony-app/pom.xml` as a compile dependency. Claudony already has `casehub-work-core` transitively via engine — Phase 3 adds the full runtime for query and mutation.

### Backend Extension

`ActionAggregationService` gains a fourth source:

```
Source 4: WorkBroker → assigned work items
  filter: status in {CREATED, READY, RESERVED, IN_PROGRESS}
  map to ActionItem(sourceType=WORKITEM)
```

Urgency for WorkItems: SLA deadline past → HIGH, deadline within 1 hour → MEDIUM, no SLA or far → LOW.

`ActionInboxResource.execute` gains `workitem:*` routing → `WorkBroker.claim()`, `delegate()`, `complete()`.

### Guard

Phase 3 is behind `claudony.casehub.enabled=true`. When disabled, the WorkItem source is absent from aggregation. No conditional frontend code — the backend simply doesn't return WorkItem-sourced items.

### Frontend

No new component. WorkItems appear as rows with a "WorkItem" type icon and claim/delegate/complete action buttons. Summary bar counts include them.

## Module Placement

| File | Module | Rationale |
|------|--------|-----------|
| `CaseBrowserService.java` | claudony-casehub | Composes CaseHub + Qhorus data |
| `ActionAggregationService.java` | claudony-casehub | Same — crosses CaseHub + Qhorus + Sessions |
| `CaseSummary.java`, `CaseDetail.java` | claudony-casehub | DTOs for the aggregation layer |
| `ActionItem.java`, `ActionDescriptor.java` | claudony-casehub | DTOs for the inbox abstraction |
| `CaseBrowserResource.java` | claudony-app/server | REST surface |
| `ActionInboxResource.java` | claudony-app/server | REST surface |
| `claudony-case-browser.ts` | claudony-app/webui/components | Frontend — configures blocks-case-explorer |
| `claudony-action-inbox.ts` | claudony-app/webui/components | Frontend — action inbox using pages-table |

## Testing

### Phase 1

**Backend:**
- `CaseBrowserServiceTest` — unit tests: aggregation from mocked sources, empty sources, multiple cases
- `CaseBrowserResourceTest` — `@QuarkusTest` + `@TestSecurity`: list (empty, populated), filter by status, detail (happy, 404), tree endpoint
- Mock CaseInstanceRepository and SessionRegistry via `@InjectMock`, use InMemoryChannelStore for Qhorus data

**Frontend (vitest):**
- `case-browser.test.ts` — entity type registration wiring, fetchFn auth headers, column renderer output

**E2E (Playwright):**
- `CaseBrowserE2ETest` — tab visible in fleet home, case list renders, detail split-pane opens, status filter works

### Phase 2

**Backend:**
- `ActionAggregationServiceTest` — unit tests: commitment mapping, stall mapping, oversight mapping, urgency derivation logic, sort order, empty sources, mixed sources
- `ActionInboxResourceTest` — `@QuarkusTest`: list actions, filter by sourceType, filter by urgency, execute routing (commitment → MessageService, stall → interjection, oversight → mark read), SSE events

**Frontend (vitest):**
- `action-inbox.test.ts` — summary bar counts, row rendering per source type, action button dispatch, SSE reconnection

**E2E (Playwright):**
- `ActionInboxE2ETest` — tab with badge count, rows grouped by urgency, action button click dispatches

### Phase 3

- Extend `ActionAggregationServiceTest` — WorkItem source mapping, SLA urgency derivation
- Extend `ActionInboxResourceTest` — execute routes to WorkBroker
- `WorkItemIntegrationTest` — `@QuarkusTest` with casehub-work: claim/complete round-trip

## Extraction Path

ActionItem is a platform pattern — every app needs a unified inbox of things needing human attention from multiple sources. Once validated in Claudony:

1. Extract `blocks-action-inbox` component to blocks-ui (the inbox UI, pages-table rendering, urgency badges, action buttons, SSE)
2. Extract `ActionSource` SPI to blocks (Java) — CDI-discovered action sources with pluggable urgency policy, similar to `ChannelBackend` or `ActorStateContributor`
3. Track as a follow-on issue

## Garden Context

- **GE-20260607-b6478d** — `pendingActionGate` is in-memory only in CaseInstanceRepository. When displaying case status, gate state exists only in `CaseInstanceCache`, not the database. The case browser must either query from cache or accept that gate state may be missing for cases loaded from persistence after restart.
- **GE-20260714-3418d2** — `QhorusDashboardService` injects `@Vetoed` reactive services — CDI deployment fails with `reactive=false`. Verify Claudony's Qhorus reactive configuration before injecting QhorusDashboardService.

## Non-Goals

- Case creation from the UI (cases are created programmatically via the engine)
- Workflow designer / case definition editing
- Cross-fleet case aggregation (single-server scope initially)
- Full notification subscription management (use blocks-notification-inbox directly as a follow-on)
