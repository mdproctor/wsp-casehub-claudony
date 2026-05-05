# System Prompt Template Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate a formatted mesh onboarding prompt for each Claude agent at worker-context-build time, stored in `WorkerContext.properties["systemPrompt"]`, so agents know their channels, startup sequence, prior workers, and message discipline without manual prompt engineering.

**Architecture:** A new package-private `MeshSystemPromptTemplate` class (pure Java, no CDI) generates the prompt given workerId, capability, caseId, channel specs, prior workers, and participation level. `ClaudonyWorkerContextProvider` gains a `CaseChannelLayout` dependency (new 4th constructor parameter), calls `layout.channelsFor()` to get channel specs, then calls `MeshSystemPromptTemplate.generate()` and stores the result in `props["systemPrompt"]`. ACTIVE → full template with startup sequence; REACTIVE → reduced template without registration; SILENT → no key added. Early-exit paths (clean-start, missing/malformed caseId) receive no systemPrompt.

**Tech Stack:** Java 21, Quarkus 3.9.5, JUnit 5, AssertJ, Mockito, `@QuarkusTest` + `QuarkusTestProfile`

**GitHub issue:** #89 (part of epic #86). Commits reference both: `Refs #89 #86` / `Closes #89`.

---

## File Map

| Status | File | Role |
|--------|------|------|
| Create | `claudony-casehub/src/main/java/dev/claudony/casehub/MeshSystemPromptTemplate.java` | Pure Java template generator — no CDI |
| Create | `claudony-casehub/src/test/java/dev/claudony/casehub/MeshSystemPromptTemplateTest.java` | 18 unit tests: all paths, robustness, correctness |
| Modify | `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java` | Add `CaseChannelLayout` field + `selectLayout()` + wire template |
| Modify | `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java` | Update 3-arg constructor calls + add 8 systemPrompt tests |
| Create | `claudony-app/src/test/java/dev/claudony/SystemPromptIntegrationTest.java` | 3 Quarkus integration tests: ACTIVE present, SILENT absent, content check |
| Create | `claudony-app/src/test/java/dev/claudony/SystemPromptSilentProfileTest.java` | 1 Quarkus integration test: silent profile → no systemPrompt |
| Modify | `CLAUDE.md` | Test count, add MeshSystemPromptTemplate to component list, update ClaudonyWorkerContextProvider description |
| Modify | `docs/DESIGN.md` | Update Agent Mesh section — systemPrompt now shipped; update outstanding items |

---

## Background: Template Content

**ACTIVE template** (full — registration, channels, prior workers, message discipline):

```
You are a Claudony-managed agent working on case {caseId}.

ROLE: {capability}

MESH CHANNELS:
  work: case-{caseId}/work — {work.description}
  observe: case-{caseId}/observe — {observe.description}
  oversight: case-{caseId}/oversight — {oversight.description}

STARTUP:
  1. register("{workerId}", "Starting {capability}", ["{capability}"])
  2. send_message("case-{caseId}/work", STATUS, "Starting: {capability}")

PRIOR WORKERS:
  - {workerName}: {outputSummary}      ← one line per prior worker
  (none — you are the first worker on this case)   ← if priorWorkers is empty

MESSAGE DISCIPLINE:
  - Post EVENT to observe for every significant tool call (no obligations created)
  - Post STATUS to work when you reach major milestones
  - Use QUERY/RESPONSE for questions with other agents — these create obligations
  - Use HANDOFF to pass work to a named next worker
  - Use DONE only when your task is fully complete
  - If you cannot proceed: DECLINE with a clear reason
  - Check work channel every few steps: check_messages("case-{caseId}/work", afterId=N)
  - Check oversight if expecting human input
```

**REACTIVE template** (reduced — no startup registration, only work channel):

```
You are a Claudony-managed agent working on case {caseId}.

ROLE: {capability}

MESH CHANNELS (respond when directly addressed):
  work: case-{caseId}/work — {work.description}

PRIOR WORKERS:
  ...

MESSAGE DISCIPLINE:
  - Monitor work channel for QUERY or COMMAND addressed to you
  - Use RESPONSE to answer QUERY; DONE when work is complete
  - Post EVENT to observe for diagnostic output
```

**SILENT**: `Optional.empty()` — no `"systemPrompt"` key in `props`.

---

## Task 1: `MeshSystemPromptTemplate` + 18 Unit Tests

**Files:**
- Create: `claudony-casehub/src/main/java/dev/claudony/casehub/MeshSystemPromptTemplate.java`
- Create: `claudony-casehub/src/test/java/dev/claudony/casehub/MeshSystemPromptTemplateTest.java`

### Step 1a: Write failing tests

- [ ] **Step 1: Create the test file**

```java
package dev.claudony.casehub;

import io.casehub.api.model.WorkerSummary;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class MeshSystemPromptTemplateTest {

    private static final UUID CASE_ID = UUID.fromString("00000000-0000-0000-0000-000000000001");
    private static final String WORKER_ID = "worker-abc";
    private static final String CAPABILITY = "researcher";

    private static final List<CaseChannelLayout.ChannelSpec> NORMATIVE_SPECS = List.of(
            new CaseChannelLayout.ChannelSpec("work", ChannelSemantic.APPEND, null,
                    "Primary coordination — all obligation-carrying message types"),
            new CaseChannelLayout.ChannelSpec("observe", ChannelSemantic.APPEND,
                    Set.of(MessageType.EVENT), "Telemetry — EVENT only, no obligations created"),
            new CaseChannelLayout.ChannelSpec("oversight", ChannelSemantic.APPEND,
                    Set.of(MessageType.QUERY, MessageType.COMMAND),
                    "Human governance — agent QUERY and human COMMAND")
    );

    private static final List<CaseChannelLayout.ChannelSpec> SIMPLE_SPECS = List.of(
            new CaseChannelLayout.ChannelSpec("work", ChannelSemantic.APPEND, null,
                    "Primary coordination — all obligation-carrying message types"),
            new CaseChannelLayout.ChannelSpec("observe", ChannelSemantic.APPEND,
                    Set.of(MessageType.EVENT), "Telemetry — EVENT only, no obligations created")
    );

    private static WorkerSummary summary(String name, String output) {
        return new WorkerSummary("id-" + name, name, Instant.now().minusSeconds(60),
                Instant.now(), output, UUID.randomUUID());
    }

    // ── Happy path: SILENT ────────────────────────────────────────────────

    @Test
    void silent_returnsEmpty() {
        Optional<String> result = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.SILENT);

        assertThat(result).isEmpty();
    }

    // ── Happy path: ACTIVE ────────────────────────────────────────────────

    @Test
    void active_returnsPresent() {
        Optional<String> result = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE);

        assertThat(result).isPresent();
    }

    @Test
    void active_containsCaseId() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt).contains(CASE_ID.toString());
    }

    @Test
    void active_containsCapabilityAsRole() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt).contains("ROLE: " + CAPABILITY);
    }

    @Test
    void active_normativeLayout_containsAllThreeChannels() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt)
                .contains("case-" + CASE_ID + "/work")
                .contains("case-" + CASE_ID + "/observe")
                .contains("case-" + CASE_ID + "/oversight");
    }

    @Test
    void active_simpleLayout_containsOnlyWorkAndObserve() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, SIMPLE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt)
                .contains("case-" + CASE_ID + "/work")
                .contains("case-" + CASE_ID + "/observe")
                .doesNotContain("case-" + CASE_ID + "/oversight");
    }

    @Test
    void active_containsStartupSection() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt).contains("STARTUP:");
    }

    @Test
    void active_startupContainsWorkerId() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt).contains("register(\"" + WORKER_ID + "\"");
    }

    @Test
    void active_containsMessageDisciplineSection() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt).contains("MESSAGE DISCIPLINE:");
    }

    @Test
    void active_noPriorWorkers_showsNoneMessage() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt).contains("none — you are the first worker");
    }

    @Test
    void active_withPriorWorkers_listsThem() {
        var workers = List.of(
                summary("alice", "auth-analysis complete"),
                summary("bob", null));

        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                workers, MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt)
                .contains("alice")
                .contains("auth-analysis complete")
                .contains("bob");
    }

    @Test
    void active_priorWorkerWithNullSummary_noNullInOutput() {
        var workers = List.of(summary("alice", null));

        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                workers, MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        assertThat(prompt)
                .contains("alice")
                .doesNotContain("null");
    }

    // ── Happy path: REACTIVE ──────────────────────────────────────────────

    @Test
    void reactive_returnsPresent() {
        Optional<String> result = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.REACTIVE);

        assertThat(result).isPresent();
    }

    @Test
    void reactive_doesNotContainStartupSection() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.REACTIVE).orElseThrow();

        assertThat(prompt)
                .doesNotContain("STARTUP:")
                .doesNotContain("register(\"");
    }

    @Test
    void reactive_containsWorkChannel() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.REACTIVE).orElseThrow();

        assertThat(prompt).contains("case-" + CASE_ID + "/work");
    }

    @Test
    void reactive_containsMessageDiscipline() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.REACTIVE).orElseThrow();

        assertThat(prompt).contains("MESSAGE DISCIPLINE:");
    }

    // ── Correctness ───────────────────────────────────────────────────────

    @Test
    void channelNamesFollowCaseIdPurposePattern() {
        String prompt = MeshSystemPromptTemplate.generate(
                WORKER_ID, CAPABILITY, CASE_ID, NORMATIVE_SPECS,
                List.of(), MeshParticipationStrategy.MeshParticipation.ACTIVE).orElseThrow();

        // Each channel name must follow case-{caseId}/{purpose} — not hardcoded
        assertThat(prompt)
                .contains("case-" + CASE_ID + "/work")
                .contains("case-" + CASE_ID + "/observe")
                .contains("case-" + CASE_ID + "/oversight");
    }
}
```

- [ ] **Step 2: Run tests — expect compile error**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=MeshSystemPromptTemplateTest -q 2>&1 | tail -5
```

Expected: COMPILE ERROR (class doesn't exist yet)

### Step 1b: Implement `MeshSystemPromptTemplate`

- [ ] **Step 3: Create `MeshSystemPromptTemplate.java`**

```java
package dev.claudony.casehub;

import io.casehub.api.model.WorkerSummary;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

class MeshSystemPromptTemplate {

    static Optional<String> generate(
            String workerId,
            String capability,
            UUID caseId,
            List<CaseChannelLayout.ChannelSpec> channelSpecs,
            List<WorkerSummary> priorWorkers,
            MeshParticipationStrategy.MeshParticipation participation) {
        return switch (participation) {
            case ACTIVE -> Optional.of(buildActive(workerId, capability, caseId, channelSpecs, priorWorkers));
            case REACTIVE -> Optional.of(buildReactive(capability, caseId, channelSpecs, priorWorkers));
            case SILENT -> Optional.empty();
        };
    }

    private static String buildActive(String workerId, String capability, UUID caseId,
                                       List<CaseChannelLayout.ChannelSpec> channelSpecs,
                                       List<WorkerSummary> priorWorkers) {
        return "You are a Claudony-managed agent working on case " + caseId + ".\n\n"
                + "ROLE: " + capability + "\n\n"
                + "MESH CHANNELS:\n" + formatChannels(caseId, channelSpecs) + "\n"
                + "STARTUP:\n"
                + "  1. register(\"" + workerId + "\", \"Starting " + capability + "\", [\"" + capability + "\"])\n"
                + "  2. send_message(\"case-" + caseId + "/work\", STATUS, \"Starting: " + capability + "\")\n\n"
                + "PRIOR WORKERS:\n" + formatPriorWorkers(priorWorkers) + "\n"
                + "MESSAGE DISCIPLINE:\n"
                + "  - Post EVENT to observe for every significant tool call (no obligations created)\n"
                + "  - Post STATUS to work when you reach major milestones\n"
                + "  - Use QUERY/RESPONSE for questions with other agents — these create obligations\n"
                + "  - Use HANDOFF to pass work to a named next worker\n"
                + "  - Use DONE only when your task is fully complete\n"
                + "  - If you cannot proceed: DECLINE with a clear reason\n"
                + "  - Check work channel every few steps: check_messages(\"case-" + caseId + "/work\", afterId=N)\n"
                + "  - Check oversight if expecting human input\n";
    }

    private static String buildReactive(String capability, UUID caseId,
                                         List<CaseChannelLayout.ChannelSpec> channelSpecs,
                                         List<WorkerSummary> priorWorkers) {
        List<CaseChannelLayout.ChannelSpec> workOnly = channelSpecs.stream()
                .filter(s -> s.purpose().equals("work"))
                .toList();
        List<CaseChannelLayout.ChannelSpec> channelsToShow = workOnly.isEmpty() ? channelSpecs : workOnly;

        return "You are a Claudony-managed agent working on case " + caseId + ".\n\n"
                + "ROLE: " + capability + "\n\n"
                + "MESH CHANNELS (respond when directly addressed):\n"
                + formatChannels(caseId, channelsToShow) + "\n"
                + "PRIOR WORKERS:\n" + formatPriorWorkers(priorWorkers) + "\n"
                + "MESSAGE DISCIPLINE:\n"
                + "  - Monitor work channel for QUERY or COMMAND addressed to you\n"
                + "  - Use RESPONSE to answer QUERY; DONE when work is complete\n"
                + "  - Post EVENT to observe for diagnostic output\n";
    }

    private static String formatChannels(UUID caseId, List<CaseChannelLayout.ChannelSpec> specs) {
        var sb = new StringBuilder();
        for (var spec : specs) {
            sb.append("  ").append(spec.purpose())
              .append(": case-").append(caseId).append("/").append(spec.purpose())
              .append(" — ").append(spec.description())
              .append("\n");
        }
        return sb.toString();
    }

    private static String formatPriorWorkers(List<WorkerSummary> priorWorkers) {
        if (priorWorkers.isEmpty()) {
            return "  (none — you are the first worker on this case)\n";
        }
        var sb = new StringBuilder();
        for (var w : priorWorkers) {
            sb.append("  - ").append(w.workerName());
            if (w.outputSummary() != null && !w.outputSummary().isBlank()) {
                sb.append(": ").append(w.outputSummary());
            }
            sb.append("\n");
        }
        return sb.toString();
    }
}
```

- [ ] **Step 4: Run tests — expect all 18 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=MeshSystemPromptTemplateTest
```

Expected: Tests run: 18, Failures: 0, Errors: 0

- [ ] **Step 5: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/MeshSystemPromptTemplate.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/MeshSystemPromptTemplateTest.java
git commit -m "feat: add MeshSystemPromptTemplate — ACTIVE/REACTIVE prompt generation, SILENT omitted Refs #89 #86"
```

---

## Task 2: Wire Template into `ClaudonyWorkerContextProvider` + Tests

**Files:**
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java`
- Modify: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java`

### Step 2a: Write new tests first (TDD)

- [ ] **Step 1: Append 8 new tests to `ClaudonyWorkerContextProviderTest`**

Read the test file first. The existing setup uses:
```java
provider = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider);
```
This 2-arg constructor will be updated to default to `NormativeChannelLayout`. Existing tests are unaffected.

New tests use a 4-arg package-private constructor `(CaseLineageQuery, CaseChannelProvider, MeshParticipationStrategy, CaseChannelLayout)`.

Add these imports if not already present:
```java
import dev.claudony.casehub.NormativeChannelLayout;
import dev.claudony.casehub.SimpleLayout;
```
(Same package — no import needed.)

Append these 8 tests inside the test class body:

```java
    // ── systemPrompt: presence and absence ───────────────────────────────

    @Test
    void buildContext_activeStrategy_withCaseId_systemPromptPresent() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).containsKey("systemPrompt");
    }

    @Test
    void buildContext_silentStrategy_withCaseId_systemPromptAbsent() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new SilentParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).doesNotContainKey("systemPrompt");
    }

    @Test
    void buildContext_reactiveStrategy_withCaseId_systemPromptPresent() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ReactiveParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("analyst", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).containsKey("systemPrompt");
    }

    // ── systemPrompt: absent on early-exit paths ──────────────────────────

    @Test
    void buildContext_activeStrategy_cleanStart_systemPromptAbsent() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy(), new NormativeChannelLayout());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("researcher", Map.of("clean-start", true)));

        assertThat(ctx.properties()).doesNotContainKey("systemPrompt");
    }

    @Test
    void buildContext_activeStrategy_missingCaseId_systemPromptAbsent() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy(), new NormativeChannelLayout());

        WorkerContext ctx = p.buildContext("w1", WorkRequest.of("researcher", Map.of()));

        assertThat(ctx.properties()).doesNotContainKey("systemPrompt");
    }

    // ── systemPrompt: content correctness ────────────────────────────────

    @Test
    void buildContext_activeStrategy_systemPromptContainsCaseId() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        String prompt = (String) ctx.properties().get("systemPrompt");
        assertThat(prompt).contains(caseId.toString());
    }

    @Test
    void buildContext_activeStrategy_systemPromptContainsCapability() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("code-reviewer", Map.of("caseId", caseId.toString())));

        String prompt = (String) ctx.properties().get("systemPrompt");
        assertThat(prompt).contains("code-reviewer");
    }

    @Test
    void buildContext_reactiveStrategy_systemPromptLacksStartupSection() {
        var p = new ClaudonyWorkerContextProvider(lineageQuery, channelProvider,
                new ReactiveParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(List.of());
        when(channelProvider.listChannels(caseId)).thenReturn(List.of());

        WorkerContext ctx = p.buildContext("w1",
                WorkRequest.of("analyst", Map.of("caseId", caseId.toString())));

        String prompt = (String) ctx.properties().get("systemPrompt");
        assertThat(prompt)
                .doesNotContain("STARTUP:")
                .doesNotContain("register(\"");
    }
```

- [ ] **Step 2: Run new tests — expect compile errors (4-arg constructor doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerContextProviderTest -q 2>&1 | tail -5
```

Expected: COMPILE ERROR

### Step 2b: Refactor `ClaudonyWorkerContextProvider`

- [ ] **Step 3: Rewrite `ClaudonyWorkerContextProvider.java`**

Key changes from current version:
1. Add `CaseChannelLayout layout` field
2. CDI constructor now reads both `meshParticipation` and `channelLayout` from config
3. Add 4-arg package-private test constructor
4. Update 3-arg constructor to default `NormativeChannelLayout`
5. 2-arg constructor unchanged (delegates to 3-arg with Active)
6. Add `selectLayout()` static method (mirrors `ClaudonyCaseChannelProvider.selectLayout()`)
7. In `buildContext()` full path: call `layout.channelsFor()` + `MeshSystemPromptTemplate.generate()` + store result

```java
package dev.claudony.casehub;

import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.CaseChannel;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerSummary;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.api.spi.WorkerContextProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ClaudonyWorkerContextProvider implements WorkerContextProvider {

    private static final Logger log = Logger.getLogger(ClaudonyWorkerContextProvider.class);

    private final CaseLineageQuery lineageQuery;
    private final CaseChannelProvider channelProvider;
    private final MeshParticipationStrategy strategy;
    private final CaseChannelLayout layout;

    @Inject
    public ClaudonyWorkerContextProvider(CaseLineageQuery lineageQuery,
                                          CaseChannelProvider channelProvider,
                                          CaseHubConfig config) {
        this(lineageQuery, channelProvider,
                selectStrategy(config.meshParticipation()),
                selectLayout(config.channelLayout()));
    }

    ClaudonyWorkerContextProvider(CaseLineageQuery lineageQuery,
                                   CaseChannelProvider channelProvider,
                                   MeshParticipationStrategy strategy,
                                   CaseChannelLayout layout) {
        this.lineageQuery = lineageQuery;
        this.channelProvider = channelProvider;
        this.strategy = strategy;
        this.layout = layout;
    }

    ClaudonyWorkerContextProvider(CaseLineageQuery lineageQuery,
                                   CaseChannelProvider channelProvider,
                                   MeshParticipationStrategy strategy) {
        this(lineageQuery, channelProvider, strategy, new NormativeChannelLayout());
    }

    ClaudonyWorkerContextProvider(CaseLineageQuery lineageQuery,
                                   CaseChannelProvider channelProvider) {
        this(lineageQuery, channelProvider, new ActiveParticipationStrategy());
    }

    @Override
    public WorkerContext buildContext(String workerId, WorkRequest task) {
        MeshParticipationStrategy.MeshParticipation participation =
                strategy.strategyFor(workerId, null);

        var props = new HashMap<String, Object>();
        props.put("meshParticipation", participation.name());

        if (Boolean.TRUE.equals(task.input().get("clean-start"))) {
            props.put("clean-start", true);
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), props);
        }

        String caseIdStr = (String) task.input().get("caseId");
        if (caseIdStr == null || caseIdStr.isBlank()) {
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), props);
        }

        UUID caseId;
        try {
            caseId = UUID.fromString(caseIdStr);
        } catch (IllegalArgumentException e) {
            return new WorkerContext(task.capability(), null, null, List.of(),
                    PropagationContext.createRoot(), props);
        }

        List<WorkerSummary> priorWorkers = lineageQuery.findCompletedWorkers(caseId);

        CaseChannel channel = channelProvider.listChannels(caseId).stream()
                .findFirst()
                .orElse(null);

        List<CaseChannelLayout.ChannelSpec> channelSpecs = layout.channelsFor(caseId, null);
        MeshSystemPromptTemplate.generate(workerId, task.capability(), caseId,
                        channelSpecs, priorWorkers, participation)
                .ifPresent(prompt -> props.put("systemPrompt", prompt));

        return new WorkerContext(task.capability(), caseId, channel, priorWorkers,
                PropagationContext.createRoot(), props);
    }

    private static MeshParticipationStrategy selectStrategy(String name) {
        return switch (name) {
            case "active" -> new ActiveParticipationStrategy();
            case "reactive" -> new ReactiveParticipationStrategy();
            case "silent" -> new SilentParticipationStrategy();
            default -> {
                log.errorf("Unknown mesh-participation '%s' — valid values: active, reactive, silent", name);
                throw new IllegalArgumentException("Unknown mesh participation: " + name);
            }
        };
    }

    private static CaseChannelLayout selectLayout(String name) {
        return switch (name) {
            case "normative" -> new NormativeChannelLayout();
            case "simple" -> new SimpleLayout();
            default -> {
                log.errorf("Unknown channel-layout '%s' — valid values: normative, simple", name);
                throw new IllegalArgumentException("Unknown channel layout: " + name);
            }
        };
    }
}
```

- [ ] **Step 4: Run all casehub tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub
```

Expected: All tests pass. The 16 existing `ClaudonyWorkerContextProviderTest` tests plus 8 new = 24 total in that class.

**If existing 3-arg tests fail:** The 3-arg constructor now delegates to 4-arg with `new NormativeChannelLayout()`. Existing tests using `new ClaudonyWorkerContextProvider(lineageQuery, channelProvider, new SilentParticipationStrategy())` still compile and behave identically (silent → no systemPrompt, NormativeChannelLayout is never consulted).

- [ ] **Step 5: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerContextProvider.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerContextProviderTest.java
git commit -m "feat: wire MeshSystemPromptTemplate into ClaudonyWorkerContextProvider — systemPrompt stamped for ACTIVE/REACTIVE Closes #89 #86"
```

---

## Task 3: Quarkus Integration Tests

**Files:**
- Create: `claudony-app/src/test/java/dev/claudony/SystemPromptIntegrationTest.java`
- Create: `claudony-app/src/test/java/dev/claudony/SystemPromptSilentProfileTest.java`

- [ ] **Step 1: Read existing QuarkusTest files to confirm `@TestSecurity` pattern**

```bash
head -20 claudony-app/src/test/java/dev/claudony/MeshParticipationIntegrationTest.java
```

Confirm `@TestSecurity(user = "test", roles = "user")` is required (it is — all `@QuarkusTest` injection tests in this module use it).

- [ ] **Step 2: Create `SystemPromptIntegrationTest.java`**

```java
package dev.claudony;

import dev.claudony.casehub.ClaudonyWorkerContextProvider;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class SystemPromptIntegrationTest {

    @Inject
    ClaudonyWorkerContextProvider provider;

    @Test
    void defaultConfig_activeStrategy_systemPromptPresent() {
        UUID caseId = UUID.randomUUID();
        WorkerContext ctx = provider.buildContext("integration-worker",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).containsKey("systemPrompt");
    }

    @Test
    void defaultConfig_systemPromptContainsCaseId() {
        UUID caseId = UUID.randomUUID();
        WorkerContext ctx = provider.buildContext("integration-worker",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        String prompt = (String) ctx.properties().get("systemPrompt");
        assertThat(prompt).contains(caseId.toString());
    }

    @Test
    void defaultConfig_systemPromptContainsStartupSection() {
        UUID caseId = UUID.randomUUID();
        WorkerContext ctx = provider.buildContext("integration-worker",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        String prompt = (String) ctx.properties().get("systemPrompt");
        assertThat(prompt).contains("STARTUP:");
    }
}
```

- [ ] **Step 3: Create `SystemPromptSilentProfileTest.java`**

```java
package dev.claudony;

import dev.claudony.casehub.ClaudonyWorkerContextProvider;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestProfile(SystemPromptSilentProfileTest.SilentProfile.class)
@TestSecurity(user = "test", roles = "user")
class SystemPromptSilentProfileTest {

    public static class SilentProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("claudony.casehub.mesh-participation", "silent");
        }
    }

    @Inject
    ClaudonyWorkerContextProvider provider;

    @Test
    void silentConfig_systemPromptAbsent() {
        UUID caseId = UUID.randomUUID();
        WorkerContext ctx = provider.buildContext("integration-worker",
                WorkRequest.of("researcher", Map.of("caseId", caseId.toString())));

        assertThat(ctx.properties()).doesNotContainKey("systemPrompt");
    }
}
```

- [ ] **Step 4: Run targeted integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app \
  -Dtest=SystemPromptIntegrationTest,SystemPromptSilentProfileTest
```

Expected: 4 tests, 0 failures.

- [ ] **Step 5: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add claudony-app/src/test/java/dev/claudony/SystemPromptIntegrationTest.java \
        claudony-app/src/test/java/dev/claudony/SystemPromptSilentProfileTest.java
git commit -m "test: add Quarkus integration tests for system prompt generation Refs #89"
```

---

## Task 4: Documentation Updates

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Run tests to get actual current count**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | \
  grep -E "^\[INFO\] Tests run:" | awk -F'[, ]' '{sum+=$4} END{print sum}'
```

Note the exact total, casehub count, and app count.

- [ ] **Step 2: Update `CLAUDE.md`**

**(A) Test count line** — find and replace the current count with the new actual count:
```
**N tests passing** (as of 2026-04-27, all modules): X in `claudony-casehub` + Y in `claudony-app`. Zero failures, zero errors.
```

**(B) Component list** — find the `claudony-casehub` tree. After `SilentParticipationStrategy.java`, add:
```
└── MeshSystemPromptTemplate.java       — package-private: generates ACTIVE/REACTIVE/SILENT prompt text
```

**(C) `ClaudonyWorkerContextProvider` description** — find the existing entry:
```
├── ClaudonyWorkerContextProvider.java  — WorkerContextProvider SPI: lineage + channel context
```
Update to:
```
├── ClaudonyWorkerContextProvider.java  — WorkerContextProvider SPI: lineage + channel context + systemPrompt
```

**(D) Test class list** — in the `claudony-casehub` tests section, after `WorkerLifecycleSequenceTest`, add:
```
- `MeshSystemPromptTemplateTest` — 18 unit tests: ACTIVE/REACTIVE full templates, SILENT omitted, channel names, prior workers, correctness
```
In the `claudony-app` tests integration section, add:
```
- `SystemPromptIntegrationTest`, `SystemPromptSilentProfileTest` — Quarkus integration: systemPrompt present for ACTIVE, absent for SILENT
```

- [ ] **Step 3: Update `docs/DESIGN.md`**

**(A) Component tree** — after `SilentParticipationStrategy`, add:
```
└── MeshSystemPromptTemplate       — package-private: generates ACTIVE full template, REACTIVE reduced,
                                     SILENT returns empty. Channel names from CaseChannelLayout; prior
                                     workers from JpaCaseLineageQuery. Stored in WorkerContext.properties["systemPrompt"]
```

**(B) `WorkerContextProvider` SPI table row** — current text ends with `mesh-participation=active|reactive|silent`. Extend:
```
Builds worker context: task, lineage, channel, meshParticipation and systemPrompt stamped via MeshSystemPromptTemplate. Config: claudony.casehub.mesh-participation=active|reactive|silent, claudony.casehub.channel-layout=normative|simple
```

**(C) Agent Mesh section** — find:
```
**Outstanding:** System prompt template (#89), `MeshParticipationStrategy.strategyFor()` currently receives `null` for `context` since the context isn't yet built at strategy-call time.
```

Replace with:
```
**Outstanding:** `MeshParticipationStrategy.strategyFor()` currently receives `null` for `context` (context not yet built at call time); shared-data keys from prior workers not yet included in the prompt (requires additional Qhorus integration, tracked as future work under epic #86).
```

**(D) Add systemPrompt to Agent Mesh section** — after the `WorkerContext.properties["meshParticipation"]` sentence, add:

```
`WorkerContext.properties["systemPrompt"]` is present for ACTIVE and REACTIVE workers when `caseId` is valid — the formatted mesh onboarding prompt. SILENT workers and early-exit paths (clean-start, missing/malformed caseId) receive no prompt. The prompt includes: case header, ROLE, MESH CHANNELS (from `CaseChannelLayout`), STARTUP sequence (ACTIVE only), PRIOR WORKERS (from `JpaCaseLineageQuery`), and MESSAGE DISCIPLINE.
```

- [ ] **Step 4: Verify no remaining stale references**

```bash
grep -n "system prompt template\|#89\|Outstanding.*89\|systemPrompt.*not yet" docs/DESIGN.md
```

Ensure no references to #89 as outstanding (it's now closed).

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md docs/DESIGN.md
git commit -m "docs: sync CLAUDE.md and DESIGN.md with #89 — MeshSystemPromptTemplate, systemPrompt shipped, updated test count Refs #89"
```

---

## Self-Review

**Spec coverage:**

| Requirement | Task |
|---|---|
| `WorkerContext` carries formatted system prompt | Task 2 — `props["systemPrompt"]` |
| Channel names from `CaseChannelLayout.channelsFor()` | Task 2 — `layout.channelsFor(caseId, null)` → `MeshSystemPromptTemplate` |
| Prior worker summaries from `List<WorkerSummary>` | Task 1 — `formatPriorWorkers()` in template |
| Template omitted when SILENT | Task 1 — `Optional.empty()` for SILENT |
| ACTIVE → full template with channels, startup, prior workers, message discipline | Task 1 |
| REACTIVE → reduced template (no startup registration) | Task 1 |
| Integration test verifies prompt for known case config | Task 3 |
| All commits reference issue and epic | All tasks — `Refs #89 #86` |
| Documentation updated | Task 4 |

**Gaps noted:**
- Shared artefacts section (from spec) not implemented — `WorkerSummary` has no shared-data keys. Noted in DESIGN.md as future work.
- ROLE and GOAL are both set to `capability` (same value). Will diverge when `CaseDefinition` is integrated. Acceptable for MVP.
- `context` parameter in `strategyFor()` is still `null` — acknowledged in DESIGN.md.

**Placeholder scan:** No TBD, TODO, "implement later". All code blocks are complete.

**Type consistency:**
- `MeshSystemPromptTemplate.generate(String, String, UUID, List<ChannelSpec>, List<WorkerSummary>, MeshParticipation)` defined in Task 1, called identically in Task 2
- 4-arg constructor `(CaseLineageQuery, CaseChannelProvider, MeshParticipationStrategy, CaseChannelLayout)` defined in Task 2 step 3, used in Task 2 step 1 tests
- `props["systemPrompt"]` key consistent across implementation (Task 2) and all tests (Tasks 2 and 3)
- `layout.channelsFor(caseId, null)` — null for CaseDefinition, consistent with #87 and #88 precedent
