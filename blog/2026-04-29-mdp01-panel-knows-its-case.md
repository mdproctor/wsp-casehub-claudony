---
layout: post
title: "Phase 14: The Panel Knows Its Case"
date: 2026-04-29
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [casehub, session-model, playwright, e2e, frontend, three-panel]
excerpt: "Adding caseId and roleName to the immutable Session record touches 20+ construction sites, and the left panel auto-expands with clickable worker rows that reconnect the WebSocket without reloading the page."
---

#76 was called "Left panel — CaseHub case graph and worker assignment list." I narrowed it immediately. The full case graph from the ecosystem spec — multiple cases, transition arrows, the orchestration view — is aspirational. What we needed was specific: when a terminal page opens for a CaseHub worker, show the other workers in the same case and let you click between them. Everything else could wait.

The first question was data model. Where does case affiliation live in Claudony's world? The `Session` record is the authoritative identity — a Java record tracking the tmux session from creation. CaseHub knows which case a worker belongs to, but that information is only available at provisioning time, through the `WorkerProvisioner` SPI. The right answer was to stamp it directly on `Session`: two new `Optional<String>` fields, `caseId` and `roleName`, added after `expiryPolicy`.

That sounds clean. Java records are immutable — construction is explicit — and `new Session(...)` appeared in more places than I expected. I brought Claude in to track them all down. We found 20+ sites across three modules: `SessionResource.create()`, `SessionResource.rename()`, `ServerStartup.bootstrapRegistry()`, the provisioner, plus a dozen test files. The rename case is the interesting one — a renamed worker is still the same worker in the same case, so `rename()` propagates rather than blanks the fields:

```java
var renamed = new Session(id, newTmuxName, session.workingDir(),
        session.command(), session.status(), session.createdAt(), Instant.now(),
        session.expiryPolicy(), session.caseId(), session.roleName());
```

The provisioner stamps real values from the `ProvisionContext`. Everyone else gets `Optional.empty()`.

Once `Session` carries `caseId`, the REST layer is direct. `SessionRegistry.findByCaseId()` filters and sorts by `createdAt` — provisioning order doubles as worker sequence. A new `?caseId=` filter on `GET /api/sessions` short-circuits federation (case workers are always local) and returns the sorted list.

The frontend adds a left panel. On load, `terminal.js` fetches the current session, reads its `caseId`, and if present, auto-expands the panel and starts polling every 3 seconds. Workers render as clickable rows — role name, a colour-coded status dot (green for active, grey for idle, red for faulted), time since provisioning. Clicking a worker closes the WebSocket and reconnects to the new session; `history.replaceState` updates the URL.

For sessions without a case — anything created from the dashboard — the panel collapses and shows "No case assigned."

One thing we'd missed: `casePoller`, the `setInterval` for worker polling, was never cleared when the panel closed or the page unloaded. Claude caught it during the code quality pass. The fix was the standard interval lifecycle pattern:

```javascript
function closeCasePanel() {
    clearInterval(casePoller);
    casePoller = null;
    casePanel.classList.add('collapsed');
}
window.addEventListener('beforeunload', function () {
    if (casePoller) clearInterval(casePoller);
});
```

`openCasePanel` restarts polling only when `activeCaseId` is set and `casePoller` is null — no duplicate timers on re-open.

The more interesting investigation was `ChannelPanelE2ETest`. Four tests failing mid-session, all in the channel panel — which we'd just modified the HTML layout around. It had the shape of a regression. We ran `git checkout <handover-commit> -- session.html terminal.js PlaywrightBase.java` and re-ran. Same failures, actually worse: 2 failures and 6 errors against the old files versus 1 failure and 2 errors with the new ones. Not a regression — pre-existing.

The improvement traced back to `PlaywrightBase`. `BASE_URL` had been hardcoded as `"http://localhost:8081"`. With `quarkus.http.test-port=0` assigning a random port each run, that's wrong whenever Quarkus doesn't happen to choose 8081. The fix reads the URL from MicroProfile Config after the server starts:

```java
@BeforeAll
static void launchBrowser() {
    BASE_URL = ConfigProvider.getConfig().getValue("test.url", String.class);
    // ...
}
```

Quarkus sets `test.url` to the actual bound URL in `@QuarkusTest` context — including the random port. It's not in the testing guide; we confirmed it works.

419 tests. #76 closed.
