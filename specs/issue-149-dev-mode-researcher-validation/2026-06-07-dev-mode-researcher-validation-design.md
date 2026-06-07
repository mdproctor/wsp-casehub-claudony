# Dev Mode Live Validation — ResearcherCase End-to-End

**Issue:** #149  
**Branch:** issue-149-dev-mode-researcher-validation  
**Date:** 2026-06-07  
**Revised:** 2026-06-07 (post code review)

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
This conflates session trigger mechanics with session content. The binding should fire
immediately when the case starts, with no required input.

The engine already handles this cleanly: `CaseDefinitionYamlMapper.convertTrigger()` produces
`ContextChangeTrigger(null)` when no `filter` is present in the YAML.
`DefaultExpressionEngineRegistry.evaluate()` short-circuits to `true` when the evaluator is
null (bytecode confirmed: `ifnonnull` branch, null path returns `iconst_1`). A binding with no
filter therefore fires unconditionally on every `contextChange` event.

`CaseStartedEventHandler` publishes `CaseContextChangedEvent` at case start regardless of
whether any initial context was provided — so calling `startCase()` (no args) triggers the
null-filter binding immediately on case creation.

**Note on re-firing:** A null-filter binding also fires on every subsequent context change
(e.g., when `signal("workers.researcher.exited", true)` is called later). This is not a
problem because `WorkOrchestrator`'s `LoopControl` prevents double-provisioning for a binding
that has already been satisfied. The design relies on this engine guarantee.

Changes:
- Binding name: `start-researcher-on-topic` → `start-session-on-init`
- Remove the `filter:` field entirely (no filter → fires unconditionally on context change)
- Caller change: `startCase()` with no args (no sentinel context needed)

The capability naming (`researcher`) is addressed in #150.

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
unsatisfied at runtime. If `isUnsatisfied()`, return 503 with a clear message.

The endpoint returns `CompletionStage<Response>` — RESTEasy Reactive natively suspends the
HTTP request and resumes on completion without holding a worker thread. No `@Blocking`, no
`.toCompletableFuture().get()`.

```java
@POST
@Path("/cases/researcher")
public CompletionStage<Response> startResearcher() {
    if (researcherCase.isUnsatisfied()) {
        return CompletableFuture.completedFuture(
            Response.status(503).entity(Map.of("error", "CaseHub engine not available")).build());
    }
    return researcherCase.get()
        .startCase()
        .thenApply(caseId -> Response.accepted(new CaseStartedResponse(caseId)).build());
}
```

Class annotations: `@Path("/api/casehub")`, `@Produces(APPLICATION_JSON)`,
`@Consumes(APPLICATION_JSON)`, `@Authenticated`.

Response model: inline record `record CaseStartedResponse(UUID caseId)`.

### 5. Tests — updated and new

**Updated: `ResearcherCaseStartupTest`**

The binding assertion must change. With null filter, `ctx.getFilter()` returns `null`
(not a `JQExpressionEvaluator`), so the existing `instanceof JQExpressionEvaluator` pattern
match fails. Updated assertion:

```java
assertThat(def.getBindings())
    .anyMatch(b ->
        "start-session-on-init".equals(b.getName())
        && b.getOn() instanceof ContextChangeTrigger ctx
        && ctx.getFilter() == null);
```

**Updated: `ResearcherCaseCompletionTest`**

Change `startCase(Map.of("topic", "test-topic"))` → `startCase()`. The null-filter binding
fires on the initial `CaseContextChangedEvent` regardless of context content.

**Updated: `TestResearcherCase`**

Aligns test bean with production YAML. Change:
- Binding name: `start-researcher-on-topic` → `start-session-on-init`
- Remove `new ContextChangeTrigger(".topic != null")` → `new ContextChangeTrigger((ExpressionEvaluator) null)`
- Also update `CaseEngineRoundTripTest` call: `startCase(Map.of("topic", "test-topic"))` → `startCase()`

**New: `CasehubResourceTest`**

`@QuarkusTest @TestSecurity(user="test", roles="user")`:
- `POST /api/casehub/cases/researcher` → 503 (ResearcherCase excluded in default test profile)
- Response body contains `"error"` field

Auth protection (no `@TestSecurity`):
- `POST /api/casehub/cases/researcher` unauthenticated → 401

The engine-enabled path (202 + real case UUID) is validated manually per the acceptance
criteria.

---

## Files Changed

| File | Change |
|------|--------|
| `casehub/src/main/resources/casehub/researcher.yaml` | Rename binding, remove filter |
| `app/src/main/resources/application.properties` | `%dev` exclusion override + casehub config |
| `app/src/main/java/io/casehub/claudony/server/CasehubResource.java` | New REST resource |
| `app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java` | Update binding assertion (null filter, new name) |
| `app/src/test/java/io/casehub/claudony/ResearcherCaseCompletionTest.java` | `startCase()` — no topic arg |
| `app/src/test/java/io/casehub/claudony/TestResearcherCase.java` | Null filter, renamed binding |
| `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java` | `startCase()` — no topic arg |
| `app/src/test/java/io/casehub/claudony/server/CasehubResourceTest.java` | New test class |

---

## Validation

```bash
# Run new and updated tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CasehubResourceTest,ResearcherCaseStartupTest,ResearcherCaseCompletionTest -pl claudony-app

# Full test suite (regression check)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test

# Live validation
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server -pl claudony-app --also-make
# Then: POST http://localhost:7777/api/casehub/cases/researcher
# Expect: tmux session appears, Claude runs, session exits, case COMPLETED in dashboard
```
