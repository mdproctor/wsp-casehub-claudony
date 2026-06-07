# Dev Mode Live Validation — ResearcherCase End-to-End

**Issue:** #149  
**Branch:** issue-149-dev-mode-researcher-validation  
**Date:** 2026-06-07

---

## Context

ResearcherCase signal chain is proven in tests (#148). Three gaps prevent running it live in
`quarkus:dev`: the production CDI exclusion of `ResearcherCase` is over-broad for dev mode;
no REST endpoint exists to trigger `startCase()`; and the dev profile has no CaseHub config.

---

## Design

### 1. YAML binding change

**File:** `casehub/src/main/resources/casehub/researcher.yaml`

The existing binding fires on `.topic != null`, requiring callers to supply a topic field.
This conflates session trigger mechanics with session content — the engine should provision
the session immediately when `startCase()` is called, not wait for a content-specific field.
Only two trigger types exist in the engine API (`ContextChangeTrigger`, `ScheduleTrigger`);
there is no `StartTrigger`. The pragmatic equivalent is a `contextChange` binding that fires
on a neutral sentinel.

Changes:
- Binding name: `start-researcher-on-topic` → `start-session-on-init`
- Filter: `.topic != null` → `.started != null`

The REST endpoint calls `startCase(Map.of("started", true))`. The sentinel field name will
be revisited in #150 alongside the broader capability/case rename.

### 2. Dev profile CDI exclusion

**File:** `app/src/main/resources/application.properties`

Production excludes `ResearcherCase` from CDI because `casehub-engine` is not on the
production classpath — `CaseHubRuntime` is unavailable, so augmentation would fail.

In dev mode, Quarkus includes test-scope deps, so `casehub-engine` IS on the classpath.
`ResearcherCase` can register. A `%dev` profile override copies the production exclusion
list minus `ResearcherCase`:

```properties
%dev.quarkus.arc.exclude-types=io.casehub.ledger.repository.CaseLedgerEntryRepository,\
  io.casehub.ledger.service.CaseLedgerEventCapture,\
  io.casehub.ledger.service.WorkerDecisionEventCapture
```

The `%dev` value replaces (not appends to) the base `quarkus.arc.exclude-types`.
The three ledger beans remain excluded in dev mode for the same reasons as production.

### 3. Dev profile CaseHub config

**File:** `app/src/main/resources/application.properties`

```properties
%dev.claudony.casehub.enabled=true
%dev.claudony.casehub.workers.commands.researcher=claude
```

`claudony.casehub.enabled` defaults to `false`. Setting it `true` in the dev profile
activates `ServerStartup.bootstrapCasehubWatchers()` and signals intent. The worker
command maps the `researcher` capability to the `claude` CLI.
`claudony.casehub.workers.default-working-dir` defaults to `${user.home}/claudony-workspace`
— no override needed.

### 4. REST endpoint

**New file:** `app/src/main/java/io/casehub/claudony/server/CasehubResource.java`

Path: `POST /api/casehub/cases/researcher`

Injection pattern follows `ServerStartup` exactly:
```java
@Inject Instance<ResearcherCase> researcherCase;
```

`Instance<T>` never fails CDI augmentation for an excluded type — it resolves as
unsatisfied at runtime. If `isUnsatisfied()`, return 503 with a clear message. Otherwise
call `startCase(Map.of("started", true))`, await the `CompletionStage`, return 202 with
the case UUID.

Annotations: `@Path("/api/casehub")`, `@Produces(APPLICATION_JSON)`,
`@Consumes(APPLICATION_JSON)`, `@Authenticated`. The start method is `@POST @Blocking`
(CompletionStage awaited on a worker thread via `.toCompletableFuture().get()`).

Response model: inline record `record CaseStartedResponse(UUID caseId)`.

No request body — the sentinel context is hardcoded inside the method. The session's
content is determined by what the user does once inside Claude, not by the trigger payload.

### 5. Test

**New file:** `app/src/test/java/io/casehub/claudony/server/CasehubResourceTest.java`

`@QuarkusTest @TestSecurity(user="test", roles="user")`:
- `POST /api/casehub/cases/researcher` → 503 (ResearcherCase excluded in default test profile)
- Response body contains `"error"` field

Auth protection (no `@TestSecurity`):
- `POST /api/casehub/cases/researcher` unauthenticated → 401

The engine-enabled path (202 + real case UUID) is validated manually per the acceptance
criteria — the `ResearcherCaseCompletionTest` already covers the full signal chain.

---

## Files Changed

| File | Change |
|------|--------|
| `casehub/src/main/resources/casehub/researcher.yaml` | Rename binding, change filter |
| `app/src/main/resources/application.properties` | `%dev` exclusion override + casehub config |
| `app/src/main/java/io/casehub/claudony/server/CasehubResource.java` | New REST resource |
| `app/src/test/java/io/casehub/claudony/server/CasehubResourceTest.java` | New test class |

---

## Validation

```bash
# Run new tests (should pass)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CasehubResourceTest -pl claudony-app

# Full test suite (regression check)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test

# Live validation
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server -pl claudony-app --also-make
# Then: POST http://localhost:7777/api/casehub/cases/researcher
# Expect: tmux session appears, Claude runs, session exits, case COMPLETED in dashboard
```
