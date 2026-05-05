# Handover — 2026-05-05

**Head commit:** `275a274` — blog entry (SSE panel and two silent failures)  
**Branch:** `main`, all tests passing (474, zero failures)

---

## What happened this session

*Previous session context — `git show HEAD~1:HANDOFF.md` (3 days old)*

**Issues closed this session:** #82 (engine NPE, upstream fix), #93 (concurrent same-role workers — `WorkResult.caseId()` + precise lookup), #103 (user identity in interjections — sender `"human:<principal>"`), #106 (messaging conventions tests — EVENT rendering, COMMAND default, allowedTypes filtering), #107 (InstanceActorIdProvider — `claudony-worker-{uuid}` → `claude:{roleName}@v1`)

**#104 SSE case worker panel — complete, not yet pushed to casehubio/claudony:**

Replaces `setInterval(pollWorkers, 3000)` in `terminal.js` with `EventSource('/api/sessions/{id}/case-events')`.
- `WorkerCaseLifecycleEvent` CDI bridge in `claudony-core` (avoids casehub→app circular dep)
- `CaseWorkerUpdateStrategy` SPI: `EventsOnlyStrategy`, `HybridStrategy` (30s heartbeat, default), `RegistryHooksStrategy`
- `CaseEventBroadcaster` — `@ApplicationScoped`, `@Observes WorkerCaseLifecycleEvent`
- `GET /api/sessions/{id}/case-events` SSE endpoint, 7 Playwright E2E tests
- Config: `claudony.case-worker-update=hybrid`, `claudony.case-worker-heartbeat-ms=30000`
- Test profile forces `events-only` (no background tick noise)

Two bugs found during E2E: Quarkus auto-wraps `Multi<String>` with `data:` SSE prefix (return plain JSON, not pre-formatted frames); `static new ObjectMapper()` silently fails on `Instant` fields (use `@Inject ObjectMapper`).

**Also fixed:** McpServerIntegrationTest tool count 57→59; GitStatusTest uses `endsWith("/claudony")` to accept any fork remote.

**Engine changes landed:** `ProvisionContext` gained `triggerChannelId` + `triggerCorrelationId` (engine#229 — needed for #94); two Claudony test constructors pass `null, null` for new fields.

---

## Test count

**474 passing, 0 failures.** All modules.

---

## Open issues

*Unchanged from prior handover — `git show HEAD~1:HANDOFF.md`*

**New:** `casehubio/claudony#108` — registry-hooks + hybrid-is-default integration test gap (identified in code review of #104)

---

## Immediate next

Push #104 to fork and open PR targeting `casehubio/claudony`:
```bash
git push origin main
gh pr create --repo casehubio/claudony --title "feat: SSE push for case worker panel (#104)"
```

After that: `#86` (agent mesh epic, self-contained) or wait for engine#221 / engine#229 to unblock `#94`.

---

## Key files

- Spec: `docs/superpowers/specs/2026-05-05-case-worker-sse-design.md`
- Plan: `docs/superpowers/plans/2026-05-05-sse-case-worker-panel.md`
- Blog: `docs/blog/2026-05-05-mdp01-sse-two-silent-failures.md`
