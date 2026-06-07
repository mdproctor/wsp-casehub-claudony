# Dev Mode Researcher Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable ResearcherCase to run live in `quarkus:dev` — fix the YAML binding, add a REST trigger endpoint, and configure the dev profile so a real case reaches COMPLETED in the dashboard.

**Architecture:** The YAML binding gains a null filter (fires on case start) and a `when` guard (prevents re-provisioning on exit signal). A new `CasehubResource` triggers it via `POST /api/casehub/cases/researcher` using `Instance<ResearcherCase>` injection (engine-absent safe). Three dev-profile properties wire the missing config.

**Tech Stack:** Java 21, Quarkus 3.32.2, RESTEasy Reactive (`CompletionStage<Response>`), CDI `Instance<T>`, casehub-engine 0.2-SNAPSHOT, JUnit 5 + RestAssured + AssertJ, Maven Surefire

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `casehub/src/main/resources/casehub/researcher.yaml` | Modify | Rename binding, remove filter, add `when` guard |
| `app/src/main/resources/application.properties` | Modify | `%dev` CDI exclusion override + CaseHub config |
| `app/src/main/java/io/casehub/claudony/server/CasehubResource.java` | Create | `POST /api/casehub/cases/researcher` endpoint |
| `app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java` | Modify | Assert null filter + `when` guard + new binding name |
| `app/src/test/java/io/casehub/claudony/ResearcherCaseCompletionTest.java` | Modify | `startCase()` with no args |
| `app/src/test/java/io/casehub/claudony/TestResearcherCase.java` | Modify | Null filter, `when` guard, renamed binding |
| `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java` | Modify | `startCase()` with no args |
| `app/src/test/java/io/casehub/claudony/server/CasehubResourceTest.java` | Create | 503 + 401 coverage for new endpoint |

---

## Task 1: Update ResearcherCaseStartupTest → change YAML binding

The plain-JUnit `ResearcherCaseStartupTest` instantiates `ResearcherCase` directly and asserts the YAML parses correctly. It has no Quarkus context — just instantiate and call `getDefinition()`. We update the assertion first (TDD), run it to confirm it fails, then fix the YAML.

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java`
- Modify: `casehub/src/main/resources/casehub/researcher.yaml`

- [ ] **Step 1.1: Update the binding assertion in ResearcherCaseStartupTest**

Replace the existing binding `assertThat` block (the one currently checking `.topic != null`) with:

```java
assertThat(def.getBindings())
    .anyMatch(b ->
        "start-session-on-init".equals(b.getName())
        && b.getOn() instanceof ContextChangeTrigger ctx
        && ctx.getFilter() == null
        && b.getWhen() instanceof JQExpressionEvaluator jq
        && ".workers.researcher.exited != true".equals(jq.expression()));
```

The `JQExpressionEvaluator` import is already present (used for the goal assertion on line ~32). No new import needed.

- [ ] **Step 1.2: Run the test to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ResearcherCaseStartupTest -pl claudony-app 2>&1 | tail -20
```

Expected: test failure — the YAML still has `start-researcher-on-topic` with `.topic != null` filter.

- [ ] **Step 1.3: Replace the YAML binding**

Full replacement of `casehub/src/main/resources/casehub/researcher.yaml`:

```yaml
dsl: "0.1"
namespace: claudony
name: researcher
version: "1.0.0"
title: "Researcher Case"
spec:
  capabilities:
    - name: researcher
      description: "Claude Code research worker started as a tmux session via Claudony"
      inputSchema: "{}"
      outputSchema: "{}"
  bindings:
    - name: start-session-on-init
      capability: researcher
      on:
        contextChange:
      when: ".workers.researcher.exited != true"
  goals:
    - name: research-complete
      condition: ".workers.researcher.exited == true"
      kind: success
  completion:
    success:
      allOf:
        - research-complete
```

Key changes from the original:
- `start-researcher-on-topic` → `start-session-on-init`
- `filter: ".topic != null"` removed (null filter fires on first `contextChange`)
- `when: ".workers.researcher.exited != true"` added (prevents re-provisioning when exit signal patches context)

Why the `when` guard is required: `CaseStartedEventHandler` fires `CONTEXT_CHANGED` on case start, and `signal("workers.researcher.exited", true)` fires another `CONTEXT_CHANGED` while the case is still `RUNNING`. `ChoreographyLoopControl` has no dedup — it passes every eligible binding through. Without the guard, a second researcher session is provisioned on the exit signal before the goal transition fires (which is async, next Vert.x iteration).

The null-filter path through the engine: `CaseDefinitionYamlMapper.convertTrigger()` produces `ContextChangeTrigger(null)` when no `filter:` key is present → `DefaultExpressionEngineRegistry.evaluate(null, ctx)` short-circuits to `true` (bytecode verified: null evaluator path returns `iconst_1`).

- [ ] **Step 1.4: Run the test to confirm it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ResearcherCaseStartupTest -pl claudony-app 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, `Tests run: 1, Failures: 0`.

- [ ] **Step 1.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  casehub/src/main/resources/casehub/researcher.yaml \
  app/src/test/java/io/casehub/claudony/casehub/ResearcherCaseStartupTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
feat(casehub): null-filter binding + when guard in researcher.yaml

Replaces topic-driven contextChange binding with a start-on-init
binding: null filter fires on case start; when guard prevents
re-provisioning when the exit signal patches the context.

Refs #149
EOF
)"
```

---

## Task 2: Align TestResearcherCase and fix two engine round-trip tests

`TestResearcherCase` is the in-memory equivalent used by `CaseEngineRoundTripTest` and `ResearcherCaseCompletionTest`. It must match production semantics or those tests will diverge. We update `TestResearcherCase` first, then fix the two test callers.

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/TestResearcherCase.java`
- Modify: `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java`
- Modify: `app/src/test/java/io/casehub/claudony/ResearcherCaseCompletionTest.java`

- [ ] **Step 2.1: Update TestResearcherCase**

In `TestResearcherCase.java`, replace the `getDefinition()` method body. The full updated method:

```java
@Override
public CaseDefinition getDefinition() {
    Capability cap = Capability.builder()
            .name("researcher")
            .inputSchema("{}")
            .outputSchema("{}")
            .build();

    Binding binding = Binding.builder()
            .name("start-session-on-init")
            .capability(cap)
            .on(new ContextChangeTrigger((ExpressionEvaluator) null))
            .when(".workers.researcher.exited != true")
            .build();

    return CaseDefinition.builder()
            .namespace("io.casehub.claudony.test")
            .name("researcher-round-trip")
            .version("1.0.0")
            .capabilities(cap)
            .bindings(binding)
            .build();
}
```

Add this import to the file's import block:
```java
import io.casehub.api.model.evaluator.ExpressionEvaluator;
```

The `(ExpressionEvaluator) null` cast disambiguates between `ContextChangeTrigger(String)` and `ContextChangeTrigger(ExpressionEvaluator)`. The `Binding.Builder.when(String)` convenience method (confirmed in the API) wraps the string in a `JQExpressionEvaluator` internally.

- [ ] **Step 2.2: Update CaseEngineRoundTripTest — remove topic arg**

Find the `startCase` call in `CaseEngineRoundTripTest.java` (line ~111):
```java
UUID caseId = researcherCase.startCase(Map.of("topic", "test-topic"))
```
Replace with:
```java
UUID caseId = researcherCase.startCase()
```

Remove the `Map.of` import line if `Map` is no longer used elsewhere in the file. (Check first — `Map.of` appears at line 57 for `ctx -> Map.of()` in the `Worker` constructor, so the import stays.)

- [ ] **Step 2.3: Update ResearcherCaseCompletionTest — remove topic arg**

Find the `startCase` call in `ResearcherCaseCompletionTest.java` (line ~113):
```java
UUID caseId = researcherCase.startCase(Map.of("topic", "test-topic"))
```
Replace with:
```java
UUID caseId = researcherCase.startCase()
```

Same `Map` import check applies — if `Map` is used elsewhere, keep the import.

- [ ] **Step 2.4: Run the three affected tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest=CaseEngineRoundTripTest,ResearcherCaseCompletionTest \
  -pl claudony-app 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`, both tests pass. These are `CasehubEnabledProfile` tests — they take longer (~30s) and require the full engine context.

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/test/java/io/casehub/claudony/TestResearcherCase.java \
  app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java \
  app/src/test/java/io/casehub/claudony/ResearcherCaseCompletionTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
test(casehub): align TestResearcherCase + callers with null-filter YAML

Null filter + when guard in TestResearcherCase matches researcher.yaml.
startCase() needs no topic arg — null-filter binding fires on case start.

Refs #149
EOF
)"
```

---

## Task 3: Write CasehubResourceTest → implement CasehubResource

TDD: write the test first, confirm failure, then create the resource.

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/server/CasehubResourceTest.java`
- Create: `app/src/main/java/io/casehub/claudony/server/CasehubResource.java`

- [ ] **Step 3.1: Write CasehubResourceTest**

Create `app/src/test/java/io/casehub/claudony/server/CasehubResourceTest.java`:

```java
package io.casehub.claudony.server;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.notNullValue;

@QuarkusTest
class CasehubResourceTest {

    @Test
    void startResearcher_unauthenticated_returns401() {
        given()
            .when().post("/api/casehub/cases/researcher")
            .then().statusCode(401);
    }

    @Test
    @TestSecurity(user = "test", roles = "user")
    void startResearcher_engineAbsent_returns503() {
        given()
            .when().post("/api/casehub/cases/researcher")
            .then()
            .statusCode(503)
            .body("error", notNullValue());
    }
}
```

Why 503 in the default test profile: `ResearcherCase` is excluded from CDI via `%test.quarkus.arc.exclude-types` (see `app/src/test/resources/application.properties` line ~56). `Instance<ResearcherCase>.isUnsatisfied()` returns `true`.

Why no `@TestSecurity` on the 401 test: the annotation bypasses auth entirely, so it must be absent to exercise the unauthenticated path.

- [ ] **Step 3.2: Run the test to confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest=CasehubResourceTest -pl claudony-app 2>&1 | tail -15
```

Expected: `BUILD FAILURE` — 404 (no resource registered at `/api/casehub/cases/researcher`).

- [ ] **Step 3.3: Create CasehubResource**

Create `app/src/main/java/io/casehub/claudony/server/CasehubResource.java`:

```java
package io.casehub.claudony.server;

import io.casehub.claudony.casehub.ResearcherCase;
import io.quarkus.security.Authenticated;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

@Path("/api/casehub")
@Produces(MediaType.APPLICATION_JSON)
@Authenticated
public class CasehubResource {

    @Inject
    Instance<ResearcherCase> researcherCase;

    @POST
    @Path("/cases/researcher")
    public CompletionStage<Response> startResearcher() {
        if (researcherCase.isUnsatisfied()) {
            return CompletableFuture.completedFuture(
                Response.status(503)
                    .entity(Map.of("error", "CaseHub engine not available"))
                    .build());
        }
        return researcherCase.get()
            .startCase()
            .thenApply(caseId -> Response.accepted(new CaseStartedResponse(caseId)).build());
    }

    record CaseStartedResponse(UUID caseId) {}
}
```

`CompletionStage<Response>` — RESTEasy Reactive suspends the HTTP request and resumes on completion without holding a worker thread. No `@Blocking` annotation needed.

`Instance<ResearcherCase>` follows the existing pattern in `ServerStartup` (`@Inject Instance<ClaudonyWorkerExecutionManager>`). It never fails CDI augmentation even when `ResearcherCase` is excluded from CDI — it resolves as unsatisfied at runtime.

- [ ] **Step 3.4: Run the tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest=CasehubResourceTest -pl claudony-app 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, `Tests run: 2, Failures: 0`.

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/main/java/io/casehub/claudony/server/CasehubResource.java \
  app/src/test/java/io/casehub/claudony/server/CasehubResourceTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
feat(server): POST /api/casehub/cases/researcher — trigger ResearcherCase

Instance<ResearcherCase> injection safe when engine absent (returns 503).
CompletionStage<Response> — RESTEasy Reactive suspends without blocking.

Refs #149
EOF
)"
```

---

## Task 4: Add dev profile CDI exclusion and CaseHub config

Config-only task. No TDD — changes verified by running the full test suite (regression check), then manual dev mode validation.

**Files:**
- Modify: `app/src/main/resources/application.properties`

- [ ] **Step 4.1: Add %dev overrides to application.properties**

Open `app/src/main/resources/application.properties`. Add at the end of the file:

```properties
# Dev profile: un-exclude ResearcherCase from CDI.
# casehub-engine is on the test-scope classpath in dev mode, so CaseHubRuntime is available.
# This override replaces (not appends) the base quarkus.arc.exclude-types list.
%dev.quarkus.arc.exclude-types=io.casehub.ledger.repository.CaseLedgerEntryRepository,\
  io.casehub.ledger.service.CaseLedgerEventCapture,\
  io.casehub.ledger.service.WorkerDecisionEventCapture

# Dev profile: enable CaseHub integration with the researcher worker command.
%dev.claudony.casehub.enabled=true
%dev.claudony.casehub.workers.commands.researcher=claude
```

Why this works: The base `quarkus.arc.exclude-types` excludes four beans including `ResearcherCase`. The `%dev` override replaces the whole list with three beans (the three ledger conflicts), leaving `ResearcherCase` out of the exclusion. In dev mode, Quarkus includes test-scope deps, so `casehub-engine` and therefore `CaseHubRuntime` are on the classpath — `ResearcherCase` can register.

The three ledger beans remain excluded in dev mode for the same reason as production: they conflict with Claudony's own ledger layer.

- [ ] **Step 4.2: Run the full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`. Baseline: 4 core + 162 casehub + 407 app = 573+ tests, all passing. Config changes don't affect the `%test` profile — no regressions expected.

- [ ] **Step 4.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  app/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/claudony commit -m "$(cat <<'EOF'
feat(config): dev profile CDI override + CaseHub config for #149

%dev.quarkus.arc.exclude-types omits ResearcherCase (engine on test classpath).
%dev.claudony.casehub.enabled=true + researcher command = claude.

Closes #149
EOF
)"
```

---

## Task 5: Manual dev mode validation

This task validates the full acceptance criteria: ResearcherCase registers in CDI, the REST trigger starts a real session, and the case reaches COMPLETED.

**Prerequisites:** `claude` CLI authenticated, tmux installed, `~/.claudony/qhorus` H2 database writable.

- [ ] **Step 5.1: Start the server in dev mode**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) \
  mvn quarkus:dev -Dclaudony.mode=server -pl claudony-app --also-make
```

Watch for:
- `ResearcherCase` in the CDI bean list (logged at startup, or check `/q/dev-ui`)
- `Listening on: http://localhost:7777`
- No CDI augmentation errors

- [ ] **Step 5.2: Authenticate (dev quick login)**

```bash
curl -s -c /tmp/claudony-cookies.txt \
  -X POST http://localhost:7777/auth/dev-login
```

Expected: HTTP 200 with session cookie set in `/tmp/claudony-cookies.txt`.

- [ ] **Step 5.3: Trigger a researcher case**

```bash
curl -s -b /tmp/claudony-cookies.txt \
  -X POST http://localhost:7777/api/casehub/cases/researcher \
  -H "Content-Type: application/json"
```

Expected:
```json
{"caseId": "<some-uuid>"}
```

HTTP 202 Accepted.

- [ ] **Step 5.4: Observe the tmux session**

```bash
tmux ls
```

Expected: a session named `claudony-worker-<uuid>` appears within a few seconds.

- [ ] **Step 5.5: Exit the session and observe COMPLETED**

Attach to the session or wait for it to exit naturally. After the session exits:

```bash
tmux ls
```

Expected: the `claudony-worker-*` session is gone.

Open the dashboard at `http://localhost:7777/app/` and navigate to the case. Expected: status shows `COMPLETED`.

---

## Targeted Test Command (reference)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest=CasehubResourceTest,ResearcherCaseStartupTest,ResearcherCaseCompletionTest,CaseEngineRoundTripTest \
  -pl claudony-app 2>&1 | tail -15
```
