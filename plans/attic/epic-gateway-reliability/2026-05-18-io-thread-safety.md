# IO Thread Safety — Reactive SPI Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `BlockingOperationNotAllowedException` in production by migrating Claudony's blocking SPI implementations (`WorkerContextProvider`, `CaseChannelProvider`, `WorkerProvisioner`) to their reactive counterparts, and updating the engine to use the reactive SPIs.

**Architecture:** The engine's `CaseContextChangedEventHandler.tryProvision()` becomes a `Uni<Void>` integrated into the existing reactive chain, injecting `ReactiveWorkerContextProvider` and `ReactiveWorkerProvisioner`. Claudony implements all three reactive SPIs. `CaseLineageQuery` shifts from `List` to `Uni<List>` with a virtual-thread offload. Qhorus reactive channel services are activated via build-time property. Blocking SPI implementations are deleted.

**Tech Stack:** Java 21, Quarkus 3.32.2, SmallRye Mutiny, Hibernate ORM (blocking, @LedgerPersistenceUnit), Quarkus Reactive H2 Client, `io.casehub.api.spi.ReactiveWorkerContextProvider`, `io.casehub.api.spi.ReactiveWorkerProvisioner`, `io.casehub.api.spi.ReactiveCaseChannelProvider`, `io.casehub.qhorus.runtime.channel.ReactiveChannelService`, `io.casehub.qhorus.runtime.message.ReactiveMessageService`

**Two repos involved:**
- `casehub-engine` at `~/claude/casehub/engine/` — engine handler change (Task 7)
- `claudony` at `~/claude/casehub/claudony/` — all other tasks

**Test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub` (casehub module), `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` (full suite)

---

## File Map

**claudony/casehub module** (`casehub/src/main/java/io/casehub/claudony/casehub/`):
- **Modify:** `CaseLineageQuery.java` — interface method returns `Uni<List<WorkerSummary>>`
- **Modify:** `EmptyCaseLineageQuery.java` — implement reactive interface
- **Modify:** `JpaCaseLineageQuery.java` — virtual-thread offload + self-injection for `@Transactional`
- **Create:** `ClaudonyReactiveCaseChannelProvider.java` — implements `ReactiveCaseChannelProvider`
- **Create:** `ClaudonyReactiveWorkerContextProvider.java` — implements `ReactiveWorkerContextProvider`
- **Create:** `ClaudonyReactiveWorkerProvisioner.java` — implements `ReactiveWorkerProvisioner`
- **Delete:** `ClaudonyCaseChannelProvider.java`, `ClaudonyWorkerContextProvider.java`, `ClaudonyWorkerProvisioner.java`

**claudony/casehub module tests** (`casehub/src/test/java/io/casehub/claudony/casehub/`):
- **Create:** `ClaudonyReactiveCaseChannelProviderTest.java`
- **Create:** `ClaudonyReactiveWorkerContextProviderTest.java`
- **Create:** `ClaudonyReactiveWorkerProvisionerTest.java`
- **Delete:** `ClaudonyCaseChannelProviderTest.java`, `ClaudonyWorkerContextProviderTest.java`, `ClaudonyWorkerProvisionerTest.java`
- **Modify:** `JpaCaseLineageQueryTest.java` — update assertions to unwrap `Uni`
- **Modify:** `WorkerLifecycleSequenceTest.java` — use reactive context provider

**claudony/app module:**
- **Modify:** `app/pom.xml` — add `quarkus-reactive-h2-client` extension
- **Modify:** `app/src/main/resources/application.properties` — add `quarkus.datasource.qhorus.reactive=true` and reactive URL
- **Modify:** `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java` — remove `@InjectMock ClaudonyWorkerContextProvider`, update lineage assertions
- **Modify:** `app/src/test/java/io/casehub/claudony/SystemPromptIntegrationTest.java` — inject reactive provider
- **Modify:** `app/src/test/java/io/casehub/claudony/MeshParticipationIntegrationTest.java` — inject reactive provider
- **Modify:** `app/src/test/java/io/casehub/claudony/SystemPromptSilentProfileTest.java` — inject reactive provider
- **Modify:** `app/src/test/java/io/casehub/claudony/MeshParticipationSilentProfileTest.java` — inject reactive provider

**casehub-engine repo** (`~/claude/casehub/engine/`):
- **Modify:** `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`

---

## Task 1: `CaseLineageQuery` interface and `EmptyCaseLineageQuery` go reactive

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/CaseLineageQuery.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/EmptyCaseLineageQuery.java`

- [ ] **Step 1: Write the failing test for `EmptyCaseLineageQuery`**

Create `casehub/src/test/java/io/casehub/claudony/casehub/EmptyCaseLineageQueryTest.java`:

```java
package io.casehub.claudony.casehub;

import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class EmptyCaseLineageQueryTest {

    @Test
    void findCompletedWorkers_returnsEmptyUni() {
        var query = new EmptyCaseLineageQuery();
        var result = query.findCompletedWorkers(UUID.randomUUID())
                          .await().indefinitely();
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd ~/claude/casehub/claudony
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EmptyCaseLineageQueryTest -q
```
Expected: compile error — `findCompletedWorkers` returns `List`, not `Uni`.

- [ ] **Step 3: Update `CaseLineageQuery` interface**

Replace the entire content of `casehub/src/main/java/io/casehub/claudony/casehub/CaseLineageQuery.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.WorkerSummary;
import io.smallrye.mutiny.Uni;
import java.util.List;
import java.util.UUID;

/** Queries the case ledger for prior worker summaries. */
public interface CaseLineageQuery {
    Uni<List<WorkerSummary>> findCompletedWorkers(UUID caseId);
}
```

- [ ] **Step 4: Update `EmptyCaseLineageQuery`**

Replace the entire content of `casehub/src/main/java/io/casehub/claudony/casehub/EmptyCaseLineageQuery.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.WorkerSummary;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
@DefaultBean
public class EmptyCaseLineageQuery implements CaseLineageQuery {

    @Override
    public Uni<List<WorkerSummary>> findCompletedWorkers(UUID caseId) {
        return Uni.createFrom().item(List.of());
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=EmptyCaseLineageQueryTest -q
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
cd ~/claude/casehub/claudony
git add casehub/src/main/java/io/casehub/claudony/casehub/CaseLineageQuery.java \
        casehub/src/main/java/io/casehub/claudony/casehub/EmptyCaseLineageQuery.java \
        casehub/src/test/java/io/casehub/claudony/casehub/EmptyCaseLineageQueryTest.java
git commit -m "feat(casehub): CaseLineageQuery and EmptyCaseLineageQuery go reactive — Uni<List<WorkerSummary>>

Refs #115"
```

---

## Task 2: `JpaCaseLineageQuery` goes reactive

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/JpaCaseLineageQuery.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/JpaCaseLineageQueryTest.java`

The EntityManager (`@LedgerPersistenceUnit`) is blocking. The reactive method offloads to a virtual-thread worker pool and calls a `@Transactional` public helper via CDI self-injection — this ensures the transaction interceptor fires even from within a lambda.

- [ ] **Step 1: Update `JpaCaseLineageQueryTest` to unwrap `Uni`**

The test currently calls `lineageQuery.findCompletedWorkers(caseId)` and gets a `List` directly. Update every such assertion to unwrap the `Uni`. Find all occurrences in `JpaCaseLineageQueryTest.java` and change:

```java
// Before:
List<WorkerSummary> result = lineageQuery.findCompletedWorkers(caseId);

// After:
List<WorkerSummary> result = lineageQuery.findCompletedWorkers(caseId)
        .await().atMost(java.time.Duration.ofSeconds(5));
```

- [ ] **Step 2: Run `JpaCaseLineageQueryTest` to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=JpaCaseLineageQueryTest -q
```
Expected: compile error — return type mismatch (still returning `List`).

- [ ] **Step 3: Rewrite `JpaCaseLineageQuery`**

Replace the entire content of `casehub/src/main/java/io/casehub/claudony/casehub/JpaCaseLineageQuery.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.WorkerSummary;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.smallrye.mutiny.Uni;
import io.vertx.core.impl.cpu.CpuCoreSensor;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
@Alternative
@Priority(1)
public class JpaCaseLineageQuery implements CaseLineageQuery {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Inject
    JpaCaseLineageQuery self;

    @Override
    public Uni<List<WorkerSummary>> findCompletedWorkers(UUID caseId) {
        return Uni.createFrom()
                  .item(() -> self.blocking(caseId))
                  .runSubscriptionOn(io.smallrye.mutiny.infrastructure.Infrastructure.getDefaultWorkerPool());
    }

    @Transactional(TxType.SUPPORTS)
    public List<WorkerSummary> blocking(UUID caseId) {
        List<CaseLedgerEntry> completed = em.createQuery(
                        "SELECT e FROM CaseLedgerEntry e " +
                        "WHERE e.caseId = :caseId AND e.eventType = 'WorkerExecutionCompleted' " +
                        "ORDER BY e.occurredAt ASC",
                        CaseLedgerEntry.class)
                .setParameter("caseId", caseId)
                .getResultList();

        return completed.stream()
                .map(e -> new WorkerSummary(
                        e.actorId,
                        e.actorId,
                        findStartedAt(caseId, e.actorId, e.occurredAt),
                        e.occurredAt,
                        null,
                        e.id))
                .toList();
    }

    private Instant findStartedAt(UUID caseId, String actorId, Instant before) {
        return em.createQuery(
                        "SELECT e.occurredAt FROM CaseLedgerEntry e " +
                        "WHERE e.caseId = :caseId AND e.actorId = :actorId " +
                        "AND e.eventType = 'WorkerExecutionStarted' AND e.occurredAt <= :before " +
                        "ORDER BY e.occurredAt DESC",
                        Instant.class)
                .setParameter("caseId", caseId)
                .setParameter("actorId", actorId)
                .setParameter("before", before)
                .setMaxResults(1)
                .getResultStream()
                .findFirst()
                .orElse(before);
    }
}
```

Note: `self.blocking(caseId)` goes through the CDI proxy, ensuring `@Transactional(TxType.SUPPORTS)` is intercepted. The `Infrastructure.getDefaultWorkerPool()` uses virtual threads on Java 21+.

- [ ] **Step 4: Run `JpaCaseLineageQueryTest` to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=JpaCaseLineageQueryTest -q
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/JpaCaseLineageQuery.java \
        casehub/src/test/java/io/casehub/claudony/casehub/JpaCaseLineageQueryTest.java
git commit -m "feat(casehub): JpaCaseLineageQuery returns Uni via virtual-thread offload

Refs #115"
```

---

## Task 3: Configuration — activate reactive Qhorus stack

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/main/resources/application.properties`

The `ReactiveChannelService` and `ReactiveMessageService` in Qhorus are activated by `@IfBuildProperty(name = "quarkus.datasource.qhorus.reactive", stringValue = "true")`. This is a BUILD-TIME property — it must be in `application.properties`, not a test profile override.

Switching to reactive requires a reactive datasource URL instead of a JDBC URL for the Qhorus H2 database. The `quarkus-reactive-h2-client` extension provides a Vert.x JDBC-bridging reactive H2 driver.

- [ ] **Step 1: Add `quarkus-reactive-h2-client` extension to `app/pom.xml`**

In `app/pom.xml`, inside the `<dependencies>` block, add after the existing Qhorus runtime dependency:

```xml
<dependency>
    <groupId>io.quarkiverse.reactiveh2client</groupId>
    <artifactId>quarkus-reactive-h2-client</artifactId>
    <version>0.7.0</version>
</dependency>
```

Verify the exact group ID and artifact ID from: https://github.com/quarkiverse/quarkus-reactive-h2-client — the pom.xml in that repo is authoritative. The version 0.7.x is compatible with Quarkus 3.28–3.33.

- [ ] **Step 2: Update `app/src/main/resources/application.properties`**

Find the Qhorus datasource block (currently around line 98-99):
```properties
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:~/.claudony/qhorus;DB_CLOSE_ON_EXIT=FALSE;AUTO_SERVER=TRUE
```

Replace with:
```properties
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.reactive=true
quarkus.datasource.qhorus.reactive.url=h2:file:~/.claudony/qhorus
```

Note: The reactive H2 URL format uses `h2:file:` without `jdbc:` prefix. If Flyway fails to run (it uses JDBC internally), also keep the JDBC URL alongside the reactive URL:
```properties
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.reactive=true
quarkus.datasource.qhorus.reactive.url=h2:file:~/.claudony/qhorus
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:~/.claudony/qhorus;DB_CLOSE_ON_EXIT=FALSE;AUTO_SERVER=TRUE
```
Quarkus supports both reactive and JDBC URLs simultaneously. Flyway uses JDBC; Hibernate Reactive uses the reactive pool.

Also add to the existing `quarkus.arc.exclude-types` line in `application.properties` (around line 135) — no additional excludes needed since `ReactiveChannelService` is activated via `@IfBuildProperty`, not `@Alternative @Priority`.

- [ ] **Step 3: Verify the build compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app --also-make -q
```
Expected: BUILD SUCCESS (no compile errors from the new extension).

- [ ] **Step 4: Commit**

```bash
git add app/pom.xml app/src/main/resources/application.properties
git commit -m "feat(app): activate reactive Qhorus stack — quarkus.datasource.qhorus.reactive=true

Enables ReactiveChannelService and ReactiveMessageService via @IfBuildProperty.
Adds quarkus-reactive-h2-client for H2 reactive datasource support in dev/test.

Refs #115"
```

---

## Task 4: `ClaudonyReactiveCaseChannelProvider`

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java`

Implements `ReactiveCaseChannelProvider`. Injects `ReactiveChannelService` and `ReactiveMessageService` from Qhorus (both activated by the build property set in Task 3). Preserves the in-memory channel cache from `ClaudonyCaseChannelProvider`.

- [ ] **Step 1: Write the failing test**

Create `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.CaseChannel;
import io.casehub.qhorus.runtime.channel.ReactiveChannelService;
import io.casehub.qhorus.runtime.message.ReactiveMessageService;
import io.casehub.qhorus.runtime.store.ChannelStore;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ClaudonyReactiveCaseChannelProviderTest {

    private ReactiveChannelService channelService;
    private ReactiveMessageService messageService;
    private ClaudonyReactiveCaseChannelProvider provider;

    @BeforeEach
    void setUp() {
        channelService = mock(ReactiveChannelService.class);
        messageService = mock(ReactiveMessageService.class);
        provider = new ClaudonyReactiveCaseChannelProvider(channelService, messageService, new NormativeChannelLayout());
    }

    @Test
    void openChannel_createsCaseChannel() {
        UUID caseId = UUID.randomUUID();
        var qhorusChannel = buildChannel("case-" + caseId + "/work");
        when(channelService.create(anyString(), anyString(), any(), any(), any(), any(), any()))
                .thenReturn(Uni.createFrom().item(qhorusChannel));

        CaseChannel result = provider.openChannel(caseId, "work")
                .await().indefinitely();

        assertThat(result.id()).isEqualTo(qhorusChannel.id.toString());
        assertThat(result.name()).isEqualTo("case-" + caseId + "/work");
    }

    @Test
    void openChannel_cacheHit_doesNotCallChannelService() {
        UUID caseId = UUID.randomUUID();
        var qhorusChannel = buildChannel("case-" + caseId + "/work");
        when(channelService.create(anyString(), anyString(), any(), any(), any(), any(), any()))
                .thenReturn(Uni.createFrom().item(qhorusChannel));

        provider.openChannel(caseId, "work").await().indefinitely();
        provider.openChannel(caseId, "work").await().indefinitely();

        verify(channelService, times(1)).create(anyString(), anyString(), any(), any(), any(), any(), any());
    }

    @Test
    void listChannels_filtersToCase() {
        UUID caseId = UUID.randomUUID();
        UUID otherId = UUID.randomUUID();
        var ch1 = buildChannel("case-" + caseId + "/work");
        var ch2 = buildChannel("case-" + otherId + "/work");
        when(channelService.listAll()).thenReturn(Uni.createFrom().item(List.of(ch1, ch2)));

        List<CaseChannel> result = provider.listChannels(caseId).await().indefinitely();

        assertThat(result).hasSize(1);
        assertThat(result.get(0).name()).isEqualTo("case-" + caseId + "/work");
    }

    @Test
    void listChannels_noMatch_returnsEmpty() {
        UUID caseId = UUID.randomUUID();
        when(channelService.listAll()).thenReturn(Uni.createFrom().item(List.of()));

        List<CaseChannel> result = provider.listChannels(caseId).await().indefinitely();

        assertThat(result).isEmpty();
    }

    @Test
    void postToChannel_sendsViaMessageService() {
        UUID channelId = UUID.randomUUID();
        var channel = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus", Map.of());
        when(messageService.send(eq(channelId), eq("worker-1"), any(), anyString(), any()))
                .thenReturn(Uni.createFrom().nullItem());

        provider.postToChannel(channel, "worker-1", "hello", null).await().indefinitely();

        verify(messageService).send(eq(channelId), eq("worker-1"), any(), eq("hello"), any());
    }

    @Test
    void closeChannel_isNoOp() {
        var channel = new CaseChannel("id", "name", "work", "qhorus", Map.of());
        // closeChannel must complete without error (Qhorus channels are persistent)
        provider.closeChannel(channel).await().indefinitely();
        verifyNoInteractions(channelService, messageService);
    }

    private io.casehub.qhorus.runtime.store.model.Channel buildChannel(String name) {
        var ch = new io.casehub.qhorus.runtime.store.model.Channel();
        ch.id = UUID.randomUUID();
        ch.name = name;
        ch.description = "";
        ch.semantic = ChannelSemantic.APPEND;
        return ch;
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveCaseChannelProviderTest -q
```
Expected: compile error — `ClaudonyReactiveCaseChannelProvider` does not exist.

- [ ] **Step 3: Create `ClaudonyReactiveCaseChannelProvider`**

Create `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.spi.ReactiveCaseChannelProvider;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ReactiveChannelService;
import io.casehub.qhorus.runtime.message.ReactiveMessageService;
import io.casehub.qhorus.runtime.store.model.Channel;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ClaudonyReactiveCaseChannelProvider implements ReactiveCaseChannelProvider {

    private static final Logger log = Logger.getLogger(ClaudonyReactiveCaseChannelProvider.class);
    private static final String CHANNEL_PREFIX = "case-";
    private static final String QHORUS_NAME_KEY = "qhorus-name";

    private final ReactiveChannelService channelService;
    private final ReactiveMessageService messageService;
    private final CaseChannelLayout layout;
    private final ConcurrentHashMap<UUID, Map<String, CaseChannel>> caseChannels = new ConcurrentHashMap<>();

    @Inject
    public ClaudonyReactiveCaseChannelProvider(ReactiveChannelService channelService,
                                               ReactiveMessageService messageService,
                                               CaseHubConfig config) {
        this.channelService = channelService;
        this.messageService = messageService;
        this.layout = CaseChannelLayout.named(config.channelLayout());
    }

    ClaudonyReactiveCaseChannelProvider(ReactiveChannelService channelService,
                                        ReactiveMessageService messageService,
                                        CaseChannelLayout layout) {
        this.channelService = channelService;
        this.messageService = messageService;
        this.layout = layout;
    }

    @Override
    public Uni<CaseChannel> openChannel(UUID caseId, String purpose) {
        Map<String, CaseChannel> channels = caseChannels.get(caseId);
        if (channels != null) {
            CaseChannel cached = channels.get(purpose);
            if (cached != null) return Uni.createFrom().item(cached);
        }
        return initializeLayout(caseId)
                .map(chans -> chans.get(purpose));
    }

    @Override
    public Uni<Void> postToChannel(CaseChannel channel, String from, String content, MessageType type) {
        UUID channelId = UUID.fromString(channel.id());
        io.casehub.qhorus.api.message.MessageType qhorusType = type != null
                ? io.casehub.qhorus.api.message.MessageType.valueOf(type.name())
                : io.casehub.qhorus.api.message.MessageType.STATUS;
        return messageService.send(channelId, from, qhorusType, content, null)
                .replaceWithVoid();
    }

    @Override
    public Uni<Void> closeChannel(CaseChannel channel) {
        return Uni.createFrom().voidItem();
    }

    @Override
    public Uni<List<CaseChannel>> listChannels(UUID caseId) {
        String prefix = CHANNEL_PREFIX + caseId;
        return channelService.listAll()
                .map(all -> all.stream()
                        .filter(ch -> ch.name.startsWith(prefix))
                        .map(this::toCaseChannel)
                        .toList());
    }

    private Uni<Map<String, CaseChannel>> initializeLayout(UUID caseId) {
        List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(caseId, null);
        List<Uni<CaseChannel>> creates = specs.stream()
                .map(spec -> createChannel(caseId, spec))
                .toList();
        return Uni.combine().all().unis(creates).with(list -> {
            Map<String, CaseChannel> channels = new ConcurrentHashMap<>();
            for (int i = 0; i < specs.size(); i++) {
                channels.put(specs.get(i).purpose(), (CaseChannel) list.get(i));
            }
            caseChannels.put(caseId, channels);
            return channels;
        });
    }

    private Uni<CaseChannel> createChannel(UUID caseId, CaseChannelLayout.ChannelSpec spec) {
        String channelName = CHANNEL_PREFIX + caseId + "/" + spec.purpose();
        String allowedTypes = toAllowedTypesString(spec.allowedTypes());
        ChannelSemantic semantic = ChannelSemantic.valueOf(spec.semantic().name());
        return channelService.create(channelName, spec.purpose(), semantic, null, null, null, allowedTypes)
                .map(ch -> toCaseChannel(ch, spec.purpose()));
    }

    private CaseChannel toCaseChannel(Channel ch) {
        String casePrefix = ch.name.contains("/") ? ch.name.substring(0, ch.name.lastIndexOf('/') + 1) : ch.name;
        String purpose = ch.name.contains("/") ? ch.name.substring(ch.name.lastIndexOf('/') + 1) : ch.name;
        return new CaseChannel(ch.id.toString(), ch.name, purpose, "qhorus",
                Map.of(QHORUS_NAME_KEY, ch.name));
    }

    private CaseChannel toCaseChannel(Channel ch, String purpose) {
        return new CaseChannel(ch.id.toString(), ch.name, purpose, "qhorus",
                Map.of(QHORUS_NAME_KEY, ch.name));
    }

    private static String toAllowedTypesString(Set<MessageType> types) {
        if (types == null || types.isEmpty()) return null;
        return types.stream().map(MessageType::name).sorted().collect(Collectors.joining(","));
    }
}
```

**Note on `ReactiveChannelService.create()` signature:** The exact parameter list depends on the version of `ReactiveChannelService` in the local repo. Run `grep -n "public Uni<Channel> create" ~/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java` to see the full signature and adjust the call accordingly. The key parameters are name, description, semantic, and optionally allowedTypes.

**Note on `ReactiveMessageService.send()` signature:** Similarly check the exact `send()` parameter list.

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveCaseChannelProviderTest -q
```
Expected: BUILD SUCCESS. If signature mismatches occur, fix the method calls to match the actual service signatures.

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java \
        casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java
git commit -m "feat(casehub): ClaudonyReactiveCaseChannelProvider — implements ReactiveCaseChannelProvider

Injects ReactiveChannelService + ReactiveMessageService directly.
Preserves in-memory channel cache from blocking predecessor.

Refs #115"
```

---

## Task 5: `ClaudonyReactiveWorkerContextProvider`

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerContextProvider.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerContextProviderTest.java`

Implements `ReactiveWorkerContextProvider`. Lineage and channel listing run in parallel via `Uni.combine().all()`. All synchronous assembly logic is in a private `assemble()` helper for testability.

- [ ] **Step 1: Write failing tests**

Create `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerContextProviderTest.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerSummary;
import io.casehub.api.spi.ReactiveCaseChannelProvider;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class ClaudonyReactiveWorkerContextProviderTest {

    private CaseLineageQuery lineageQuery;
    private ReactiveCaseChannelProvider channelProvider;
    private ClaudonyReactiveWorkerContextProvider provider;

    @BeforeEach
    void setUp() {
        lineageQuery = mock(CaseLineageQuery.class);
        channelProvider = mock(ReactiveCaseChannelProvider.class);
        provider = new ClaudonyReactiveWorkerContextProvider(lineageQuery, channelProvider);
    }

    @Test
    void buildContext_noPriorWorkers_returnsEmptyPriorWorkers() {
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(Uni.createFrom().item(List.of()));
        when(channelProvider.listChannels(caseId)).thenReturn(Uni.createFrom().item(List.of()));

        WorkerContext ctx = provider.buildContext("worker-1", caseId,
                WorkRequest.of("researcher", Map.of())).await().indefinitely();

        assertThat(ctx.priorWorkers()).isEmpty();
        assertThat(ctx.taskDescription()).isEqualTo("researcher");
    }

    @Test
    void buildContext_withPriorWorkers_includesThem() {
        UUID caseId = UUID.randomUUID();
        UUID entryId = UUID.randomUUID();
        var summary = new WorkerSummary("alice", "AliceRole", null, Instant.now().minusSeconds(60), null, entryId);
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(Uni.createFrom().item(List.of(summary)));
        when(channelProvider.listChannels(caseId)).thenReturn(Uni.createFrom().item(List.of()));

        WorkerContext ctx = provider.buildContext("worker-2", caseId,
                WorkRequest.of("coder", Map.of())).await().indefinitely();

        assertThat(ctx.priorWorkers()).hasSize(1);
        assertThat(ctx.priorWorkers().get(0).workerId()).isEqualTo("alice");
        assertThat(ctx.priorWorkers().get(0).ledgerEntryId()).isEqualTo(entryId);
    }

    @Test
    void buildContext_withChannel_includesChannelInContext() {
        UUID caseId = UUID.randomUUID();
        var channel = new CaseChannel("ch-id", "case-coord", "coordination", "qhorus", Map.of());
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(Uni.createFrom().item(List.of()));
        when(channelProvider.listChannels(caseId)).thenReturn(Uni.createFrom().item(List.of(channel)));

        WorkerContext ctx = provider.buildContext("worker-1", caseId,
                WorkRequest.of("task", Map.of())).await().indefinitely();

        assertThat(ctx.channels()).isNotEmpty();
        assertThat(ctx.channels().getFirst().name()).isEqualTo("case-coord");
    }

    @Test
    void buildContext_cleanStart_returnsEmptyContextWithNoInteractions() {
        WorkerContext ctx = provider.buildContext("worker-new", null,
                WorkRequest.of("task", Map.of("clean-start", true))).await().indefinitely();

        assertThat(ctx.priorWorkers()).isEmpty();
        verifyNoInteractions(lineageQuery, channelProvider);
    }

    @Test
    void buildContext_nullCaseId_returnsEmptyContextWithNoInteractions() {
        WorkerContext ctx = provider.buildContext("worker-1", null,
                WorkRequest.of("task", Map.of())).await().indefinitely();

        assertThat(ctx.priorWorkers()).isEmpty();
        assertThat(ctx.caseId()).isNull();
        verifyNoInteractions(lineageQuery, channelProvider);
    }

    @Test
    void buildContext_lineageAndChannelRunInParallel() {
        UUID caseId = UUID.randomUUID();
        // Both stubs resolve immediately — we just verify both are called
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(Uni.createFrom().item(List.of()));
        when(channelProvider.listChannels(caseId)).thenReturn(Uni.createFrom().item(List.of()));

        provider.buildContext("w1", caseId, WorkRequest.of("task", Map.of())).await().indefinitely();

        verify(lineageQuery).findCompletedWorkers(caseId);
        verify(channelProvider).listChannels(caseId);
    }

    @Test
    void buildContext_activeStrategy_stampsMeshParticipationActive() {
        var p = new ClaudonyReactiveWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(Uni.createFrom().item(List.of()));
        when(channelProvider.listChannels(caseId)).thenReturn(Uni.createFrom().item(List.of()));

        WorkerContext ctx = p.buildContext("w1", caseId,
                WorkRequest.of("task", Map.of())).await().indefinitely();

        assertThat(ctx.properties()).containsEntry("meshParticipation", "ACTIVE");
    }

    @Test
    void buildContext_silentStrategy_systemPromptAbsent() {
        var p = new ClaudonyReactiveWorkerContextProvider(lineageQuery, channelProvider,
                new SilentParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(Uni.createFrom().item(List.of()));
        when(channelProvider.listChannels(caseId)).thenReturn(Uni.createFrom().item(List.of()));

        WorkerContext ctx = p.buildContext("w1", caseId,
                WorkRequest.of("researcher", Map.of())).await().indefinitely();

        assertThat(ctx.properties()).doesNotContainKey("systemPrompt");
    }

    @Test
    void buildContext_activeStrategy_withCaseId_systemPromptPresent() {
        var p = new ClaudonyReactiveWorkerContextProvider(lineageQuery, channelProvider,
                new ActiveParticipationStrategy(), new NormativeChannelLayout());
        UUID caseId = UUID.randomUUID();
        when(lineageQuery.findCompletedWorkers(caseId)).thenReturn(Uni.createFrom().item(List.of()));
        when(channelProvider.listChannels(caseId)).thenReturn(Uni.createFrom().item(List.of()));

        WorkerContext ctx = p.buildContext("w1", caseId,
                WorkRequest.of("researcher", Map.of())).await().indefinitely();

        assertThat(ctx.properties()).containsKey("systemPrompt");
    }

    @Test
    void buildContext_propagationContextIsAlwaysSet() {
        WorkerContext ctx = provider.buildContext("w1", null,
                WorkRequest.of("task", Map.of())).await().indefinitely();

        assertThat(ctx.propagationContext()).isNotNull();
    }
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveWorkerContextProviderTest -q
```
Expected: compile error — `ClaudonyReactiveWorkerContextProvider` does not exist.

- [ ] **Step 3: Create `ClaudonyReactiveWorkerContextProvider`**

Create `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerContextProvider.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.CaseChannel;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerSummary;
import io.casehub.api.spi.ReactiveCaseChannelProvider;
import io.casehub.api.spi.ReactiveWorkerContextProvider;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ClaudonyReactiveWorkerContextProvider implements ReactiveWorkerContextProvider {

    private static final Logger log = Logger.getLogger(ClaudonyReactiveWorkerContextProvider.class);

    private final CaseLineageQuery lineageQuery;
    private final ReactiveCaseChannelProvider channelProvider;
    private final MeshParticipationStrategy strategy;
    private final CaseChannelLayout layout;

    @Inject
    public ClaudonyReactiveWorkerContextProvider(CaseLineageQuery lineageQuery,
                                                  ReactiveCaseChannelProvider channelProvider,
                                                  CaseHubConfig config) {
        this(lineageQuery, channelProvider,
                selectStrategy(config.meshParticipation()),
                CaseChannelLayout.named(config.channelLayout()));
    }

    ClaudonyReactiveWorkerContextProvider(CaseLineageQuery lineageQuery,
                                           ReactiveCaseChannelProvider channelProvider,
                                           MeshParticipationStrategy strategy,
                                           CaseChannelLayout layout) {
        this.lineageQuery = lineageQuery;
        this.channelProvider = channelProvider;
        this.strategy = strategy;
        this.layout = layout;
    }

    ClaudonyReactiveWorkerContextProvider(CaseLineageQuery lineageQuery,
                                           ReactiveCaseChannelProvider channelProvider,
                                           MeshParticipationStrategy strategy) {
        this(lineageQuery, channelProvider, strategy, new NormativeChannelLayout());
    }

    ClaudonyReactiveWorkerContextProvider(CaseLineageQuery lineageQuery,
                                           ReactiveCaseChannelProvider channelProvider) {
        this(lineageQuery, channelProvider, new ActiveParticipationStrategy());
    }

    @Override
    public Uni<WorkerContext> buildContext(String workerId, UUID caseId, WorkRequest task) {
        MeshParticipationStrategy.MeshParticipation participation = strategy.strategyFor(workerId, null);
        var props = new HashMap<String, Object>();
        props.put("meshParticipation", participation.name());

        if (Boolean.TRUE.equals(task.input().get("clean-start"))) {
            props.put("clean-start", true);
            return Uni.createFrom().item(new WorkerContext(task.capability(), null, List.of(),
                    List.of(), PropagationContext.createRoot(), props));
        }

        if (caseId == null) {
            return Uni.createFrom().item(new WorkerContext(task.capability(), null, List.of(),
                    List.of(), PropagationContext.createRoot(), props));
        }

        final UUID finalCaseId = caseId;
        return Uni.combine().all()
                .unis(lineageQuery.findCompletedWorkers(finalCaseId),
                      channelProvider.listChannels(finalCaseId))
                .asTuple()
                .map(tuple -> assemble(workerId, finalCaseId, task,
                        tuple.getItem1(), tuple.getItem2(), participation, props, layout));
    }

    private static WorkerContext assemble(String workerId, UUID caseId, WorkRequest task,
                                          List<WorkerSummary> priorWorkers,
                                          List<CaseChannel> channels,
                                          MeshParticipationStrategy.MeshParticipation participation,
                                          Map<String, Object> props,
                                          CaseChannelLayout layout) {
        List<CaseChannelLayout.ChannelSpec> channelSpecs = layout.channelsFor(caseId, null);
        MeshSystemPromptTemplate.generate(workerId, task.capability(), caseId,
                        channelSpecs, priorWorkers, participation)
                .ifPresent(prompt -> props.put("systemPrompt", prompt));

        CaseChannel firstChannel = channels.stream().findFirst().orElse(null);
        return new WorkerContext(task.capability(), caseId,
                firstChannel != null ? List.of(firstChannel) : List.of(),
                priorWorkers, PropagationContext.createRoot(), props);
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
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveWorkerContextProviderTest -q
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerContextProvider.java \
        casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerContextProviderTest.java
git commit -m "feat(casehub): ClaudonyReactiveWorkerContextProvider — implements ReactiveWorkerContextProvider

Runs lineage and channel queries in parallel via Uni.combine().all().
Synchronous assembly extracted to testable private helper.

Refs #115"
```

---

## Task 6: `ClaudonyReactiveWorkerProvisioner`

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`
- Create: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java`

Implements `ReactiveWorkerProvisioner`. Tmux `ProcessBuilder` is blocking — offloaded to `Infrastructure.getDefaultWorkerPool()`. The dead `contextProvider` injection is removed.

- [ ] **Step 1: Write failing tests**

Create `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TmuxService;
import io.casehub.claudony.server.model.Session;
import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.Worker;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.spi.ProvisioningException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ClaudonyReactiveWorkerProvisionerTest {

    private TmuxService tmux;
    private SessionRegistry registry;
    private WorkerCommandResolver resolver;
    private WorkerSessionMapping sessionMapping;
    private ClaudonyReactiveWorkerProvisioner provisioner;

    @BeforeEach
    void setUp() {
        tmux = mock(TmuxService.class);
        registry = mock(SessionRegistry.class);
        sessionMapping = new WorkerSessionMapping();
        resolver = new WorkerCommandResolver(Map.of("code-reviewer", "claude", "default", "claude"));
        provisioner = new ClaudonyReactiveWorkerProvisioner(true, tmux, registry, resolver, sessionMapping, "/tmp/workers");
    }

    @Test
    void provision_createsSessionAndRegistersWorker() throws Exception {
        var caseId = UUID.randomUUID();
        Worker worker = provisioner.provision(Set.of("code-reviewer"), provisionContext(caseId))
                .await().indefinitely();

        assertThat(worker.getName()).isEqualTo("code-reviewer");
        verify(tmux).createSession(contains(ClaudonyReactiveWorkerProvisioner.SESSION_PREFIX), eq("/tmp/workers"), eq("claude"));
        verify(registry).register(any(Session.class));
    }

    @Test
    void provision_returnsWorkerWithRequestedCapabilities() throws Exception {
        var caseId = UUID.randomUUID();
        Worker worker = provisioner.provision(Set.of("code-reviewer"), provisionContext(caseId))
                .await().indefinitely();

        assertThat(worker.getCapabilities()).extracting(c -> c.getName())
                .containsExactlyInAnyOrder("code-reviewer");
    }

    @Test
    void provision_registersRoleToSessionMapping() throws Exception {
        var caseId = UUID.randomUUID();
        provisioner.provision(Set.of("code-reviewer"), provisionContext(caseId))
                .await().indefinitely();

        assertThat(sessionMapping.findByRole("code-reviewer")).isPresent();
        assertThat(sessionMapping.findByCase(caseId.toString(), "code-reviewer")).isPresent();
    }

    @Test
    void provision_disabled_failsWithProvisioningException() {
        var disabledProvisioner = new ClaudonyReactiveWorkerProvisioner(
                false, tmux, registry, resolver, sessionMapping, "/tmp");

        assertThatThrownBy(() ->
                disabledProvisioner.provision(Set.of("code-reviewer"), provisionContext(UUID.randomUUID()))
                        .await().indefinitely())
                .isInstanceOf(ProvisioningException.class)
                .hasMessageContaining("disabled");
    }

    @Test
    void getCapabilities_returnsResolverCapabilities() {
        Set<String> caps = provisioner.getCapabilities().await().indefinitely();
        assertThat(caps).contains("code-reviewer");
    }

    @Test
    void terminate_killsSession() throws Exception {
        provisioner.terminate("some-worker").await().indefinitely();
        verify(tmux).killSession(contains(ClaudonyReactiveWorkerProvisioner.SESSION_PREFIX));
    }

    private ProvisionContext provisionContext(UUID caseId) {
        var workerContext = new WorkerContext("task", caseId, null, List.of(),
                PropagationContext.createRoot(), Map.of());
        return new ProvisionContext(caseId, "code-reviewer", workerContext,
                PropagationContext.createRoot(), null, null);
    }
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveWorkerProvisionerTest -q
```
Expected: compile error — `ClaudonyReactiveWorkerProvisioner` does not exist.

- [ ] **Step 3: Create `ClaudonyReactiveWorkerProvisioner`**

Create `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.claudony.server.SessionRegistry;
import io.casehub.claudony.server.TmuxService;
import io.casehub.claudony.server.model.Session;
import io.casehub.claudony.server.model.SessionStatus;
import io.casehub.api.model.Capability;
import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.Worker;
import io.casehub.api.spi.ProvisioningException;
import io.casehub.api.spi.ReactiveWorkerProvisioner;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.io.IOException;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class ClaudonyReactiveWorkerProvisioner implements ReactiveWorkerProvisioner {

    static final String SESSION_PREFIX = "claudony-worker-";

    private final boolean enabled;
    private final TmuxService tmux;
    private final SessionRegistry registry;
    private final WorkerCommandResolver resolver;
    private final WorkerSessionMapping sessionMapping;
    private final String defaultWorkingDir;

    @Inject
    public ClaudonyReactiveWorkerProvisioner(CaseHubConfig config,
                                              TmuxService tmux,
                                              SessionRegistry registry,
                                              WorkerCommandResolver resolver,
                                              WorkerSessionMapping sessionMapping) {
        this(config.enabled(), tmux, registry, resolver, sessionMapping,
                config.workers().defaultWorkingDir());
    }

    ClaudonyReactiveWorkerProvisioner(boolean enabled, TmuxService tmux, SessionRegistry registry,
                                       WorkerCommandResolver resolver,
                                       WorkerSessionMapping sessionMapping,
                                       String defaultWorkingDir) {
        this.enabled = enabled;
        this.tmux = tmux;
        this.registry = registry;
        this.resolver = resolver;
        this.sessionMapping = sessionMapping;
        this.defaultWorkingDir = defaultWorkingDir;
    }

    @Override
    public Uni<Worker> provision(Set<String> capabilities, ProvisionContext context) {
        return Uni.createFrom()
                  .item(() -> doProvision(capabilities, context))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public Uni<Void> terminate(String workerId) {
        return Uni.createFrom()
                  .<Void>item(() -> {
                      try {
                          tmux.killSession(SESSION_PREFIX + workerId);
                      } catch (IOException | InterruptedException e) {
                          // Session may already be gone — no-op
                      }
                      registry.remove(workerId);
                      return null;
                  })
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public Uni<Set<String>> getCapabilities() {
        return Uni.createFrom().item(resolver.getAvailableCapabilities());
    }

    private Worker doProvision(Set<String> capabilities, ProvisionContext context) {
        if (!enabled) {
            throw new ProvisioningException(
                    "CaseHub integration is disabled — set claudony.casehub.enabled=true");
        }
        String sessionId = UUID.randomUUID().toString();
        String roleName = context.taskType() != null
                ? context.taskType()
                : capabilities.stream().findFirst().orElse("worker");
        String command = resolver.resolve(capabilities);
        String sessionName = SESSION_PREFIX + sessionId;

        try {
            tmux.createSession(sessionName, defaultWorkingDir, command);
        } catch (IOException | InterruptedException e) {
            throw new ProvisioningException("Failed to create tmux session for worker " + sessionId, e);
        }

        var session = new Session(sessionId, sessionName, defaultWorkingDir, command,
                SessionStatus.IDLE, Instant.now(), Instant.now(), Optional.empty(),
                Optional.ofNullable(context.caseId()).map(UUID::toString),
                Optional.of(roleName));
        registry.register(session);
        sessionMapping.register(roleName, context.caseId(), sessionId);

        List<Capability> capList = capabilities.stream()
                .map(cap -> new Capability(cap, null, null))
                .toList();
        return new Worker(roleName, capList, ctx -> Map.of());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=ClaudonyReactiveWorkerProvisionerTest -q
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java \
        casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisionerTest.java
git commit -m "feat(casehub): ClaudonyReactiveWorkerProvisioner — implements ReactiveWorkerProvisioner

Offloads blocking tmux ProcessBuilder to virtual-thread worker pool.
Removes dead ClaudonyWorkerContextProvider injection from predecessor.

Refs #115"
```

---

## Task 7: Engine — `tryProvision()` becomes `Uni<Void>`

**Repo:** `~/claude/casehub/engine/`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`

Changes `tryProvision()` from a void fire-and-forget to a `Uni<Void>` integrated into the reactive pipeline. Drops `@Inject WorkerContextProvider` and `@Inject WorkerProvisioner`. Adds `@Inject ReactiveWorkerContextProvider` and `@Inject ReactiveWorkerProvisioner`.

- [ ] **Step 1: Verify engine builds cleanly before changes**

```bash
cd ~/claude/casehub/engine
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -q
```
Expected: BUILD SUCCESS

- [ ] **Step 2: Modify `CaseContextChangedEventHandler`**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`:

**Remove these two `@Inject` fields:**
```java
@Inject WorkerProvisioner workerProvisioner;
@Inject WorkerContextProvider workerContextProvider;
```

**Add these two `@Inject` fields:**
```java
@Inject ReactiveWorkerContextProvider reactiveWorkerContextProvider;
@Inject ReactiveWorkerProvisioner reactiveWorkerProvisioner;
```

**Add the required imports** (remove unused ones for `WorkerProvisioner` and `WorkerContextProvider`):
```java
import io.casehub.api.spi.ReactiveWorkerContextProvider;
import io.casehub.api.spi.ReactiveWorkerProvisioner;
```

**Replace the `tryProvision` method** (currently at the bottom of the class):

```java
private Uni<Void> tryProvision(CaseInstance caseInstance, Capability capability) {
    return reactiveWorkerProvisioner.getCapabilities()
            .flatMap(caps -> {
                if (!caps.contains(capability.getName())) {
                    return Uni.createFrom().voidItem();
                }
                Map<String, Object> inputData =
                        caseInstance.getCaseContext().evalObjectTemplate(capability.getInputSchema());
                WorkRequest workRequest = WorkRequest.of(capability.getName(), inputData);
                return reactiveWorkerContextProvider
                        .buildContext(null, caseInstance.getUuid(), workRequest)
                        .flatMap(workerContext -> {
                            ProvisionContext provisionContext = new ProvisionContext(
                                    caseInstance.getUuid(),
                                    capability.getName(),
                                    workerContext,
                                    PropagationContext.createRoot(),
                                    null,
                                    null);
                            return reactiveWorkerProvisioner.provision(caps, provisionContext)
                                    .replaceWithVoid();
                        });
            })
            .onFailure(ProvisioningException.class).invoke(e ->
                    LOG.warnf(e,
                            "WorkerProvisioner failed for capability '%s' on case %s — binding remains eligible",
                            capability.getName(),
                            caseInstance.getUuid()));
}
```

**Update the two call sites** in `publishWorkerSchedule()` that previously called `tryProvision(...)` as a void fire-and-forget. Both look like:

```java
// Before (fire-and-forget):
tryProvision(caseInstance, capability);
return Uni.createFrom().voidItem();

// After (integrated into chain):
return tryProvision(caseInstance, capability);
```

There are two such occurrences in `publishWorkerSchedule()` — one when `workers == null || workers.isEmpty()` and one when `candidates.isEmpty()`. Change both.

- [ ] **Step 3: Compile engine to verify**

```bash
cd ~/claude/casehub/engine
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -q
```
Expected: BUILD SUCCESS. Fix any import/signature errors.

- [ ] **Step 4: Run engine tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q
```
Expected: BUILD SUCCESS. The engine's own tests use `EmptyReactiveWorkerContextProvider` and `NoOpReactiveWorkerProvisioner` (`@DefaultBean`), which satisfy the new injections.

- [ ] **Step 5: Build and install engine locally**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q
```
This installs `casehub-engine:0.2-SNAPSHOT` to the local Maven repository so Claudony picks it up.

- [ ] **Step 6: Commit engine changes**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java
git commit -m "feat(engine): tryProvision() returns Uni<Void> — uses ReactiveWorkerContextProvider + ReactiveWorkerProvisioner

Integrates provisioning into the reactive pipeline.
Provisioning errors now propagate to the handler's onFailure() logging chain.

Refs casehubio/claudony#115"
```

---

## Task 8: Delete blocking SPI implementations, update remaining tests

**Files:**
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyCaseChannelProvider.java`
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerContextProvider.java`
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisioner.java`
- Delete: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyCaseChannelProviderTest.java`
- Delete: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerContextProviderTest.java`
- Delete: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/WorkerLifecycleSequenceTest.java`
- Modify: `app/src/test/java/io/casehub/claudony/MeshParticipationIntegrationTest.java`
- Modify: `app/src/test/java/io/casehub/claudony/SystemPromptIntegrationTest.java`
- Modify: `app/src/test/java/io/casehub/claudony/MeshParticipationSilentProfileTest.java`
- Modify: `app/src/test/java/io/casehub/claudony/SystemPromptSilentProfileTest.java`

- [ ] **Step 1: Delete blocking production classes**

```bash
cd ~/claude/casehub/claudony
rm casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyCaseChannelProvider.java
rm casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerContextProvider.java
rm casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisioner.java
```

- [ ] **Step 2: Delete old test files**

```bash
rm casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyCaseChannelProviderTest.java
rm casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerContextProviderTest.java
rm casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyWorkerProvisionerTest.java
```

- [ ] **Step 3: Update `WorkerLifecycleSequenceTest`**

In `casehub/src/test/java/io/casehub/claudony/casehub/WorkerLifecycleSequenceTest.java`:

Replace all imports and usages of `ClaudonyWorkerContextProvider` with `ClaudonyReactiveWorkerContextProvider`, `CaseChannelProvider` with `ReactiveCaseChannelProvider`.

Change mock construction (line ~55):
```java
// Before:
var contextProvider = mock(ClaudonyWorkerContextProvider.class);
// After:
var contextProvider = mock(ClaudonyReactiveWorkerContextProvider.class);
```

Change the direct construction (lines ~147-148):
```java
// Before:
var contextProvider = new ClaudonyWorkerContextProvider(
        mock(CaseLineageQuery.class), mock(CaseChannelProvider.class));
WorkerContext ctx = contextProvider.buildContext("worker-1", null, ...);
// After:
var contextProvider = new ClaudonyReactiveWorkerContextProvider(
        mock(CaseLineageQuery.class), mock(ReactiveCaseChannelProvider.class));
WorkerContext ctx = contextProvider.buildContext("worker-1", null, ...)
        .await().indefinitely();
```

Also update imports: add `io.casehub.api.spi.ReactiveCaseChannelProvider`, `io.smallrye.mutiny.Uni`; remove `io.casehub.api.spi.CaseChannelProvider`.

- [ ] **Step 4: Update `SystemPromptIntegrationTest`**

In `app/src/test/java/io/casehub/claudony/SystemPromptIntegrationTest.java`:

Change import:
```java
// Before:
import io.casehub.claudony.casehub.ClaudonyWorkerContextProvider;
// After:
import io.casehub.claudony.casehub.ClaudonyReactiveWorkerContextProvider;
```

Change field:
```java
// Before:
@Inject ClaudonyWorkerContextProvider provider;
// After:
@Inject ClaudonyReactiveWorkerContextProvider provider;
```

Change each `buildContext()` call to unwrap the `Uni`:
```java
// Before:
WorkerContext ctx = provider.buildContext("integration-worker", caseId, WorkRequest.of(...));
// After:
WorkerContext ctx = provider.buildContext("integration-worker", caseId, WorkRequest.of(...))
        .await().atMost(java.time.Duration.ofSeconds(5));
```

Apply the same pattern to all three test methods in the file.

- [ ] **Step 5: Update `MeshParticipationIntegrationTest`**

Apply the same changes as Step 4 to `MeshParticipationIntegrationTest.java` — same import, field, and `buildContext()` call changes.

- [ ] **Step 6: Update `SystemPromptSilentProfileTest` and `MeshParticipationSilentProfileTest`**

Apply the same changes to both silent-profile test files.

- [ ] **Step 7: Compile casehub module to verify no broken references**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test-compile -pl casehub -q
```
Expected: BUILD SUCCESS. If there are still references to deleted classes, fix them.

- [ ] **Step 8: Compile app module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test-compile -pl app --also-make -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 9: Run casehub module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -q
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(casehub): delete blocking SPI implementations, update tests to reactive

Removes ClaudonyWorkerContextProvider, ClaudonyCaseChannelProvider, ClaudonyWorkerProvisioner.
Updates WorkerLifecycleSequenceTest and app integration tests to use reactive provider.

Refs #115"
```

---

## Task 9: `CaseEngineRoundTripTest` — remove `@InjectMock`, update assertions

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java`

The `@InjectMock ClaudonyWorkerContextProvider workerContextProvider` stub and the `UserTransaction` wrappers around `findCompletedWorkers()` are removed. The real `ClaudonyReactiveWorkerContextProvider` is exercised through the engine pipeline.

- [ ] **Step 1: Remove the `@InjectMock` field and its stub**

In `CaseEngineRoundTripTest.java`:

Remove the field:
```java
@InjectMock ClaudonyWorkerContextProvider workerContextProvider;
```

Remove the stub at the top of the test method:
```java
when(workerContextProvider.buildContext(any(), any(), any()))
        .thenReturn(new WorkerContext("researcher", null, List.of(), List.of(),
                PropagationContext.createRoot(), Map.of()));
```

Remove unused imports:
```java
import io.casehub.claudony.casehub.ClaudonyWorkerContextProvider;
import io.casehub.api.model.WorkerContext;
// (keep PropagationContext if still needed elsewhere; remove if not)
```

Also remove `@Inject UserTransaction tx;` if it is no longer used (see next step).

- [ ] **Step 2: Update the lineage assertions**

The two `UserTransaction` blocks around `findCompletedWorkers()` are replaced with direct `Uni` unwrapping. The `@Transactional` on `JpaCaseLineageQuery.blocking()` now manages transactions internally.

Replace:
```java
Awaitility.await()
    .atMost(Duration.ofSeconds(20))
    .pollInterval(Duration.ofMillis(200))
    .untilAsserted(() -> {
        tx.begin();
        try {
            assertThat(lineageQuery.findCompletedWorkers(caseId))
                    .as("lineage must contain the completed worker")
                    .hasSize(1);
        } finally {
            tx.rollback();
        }
    });

tx.begin();
WorkerSummary summary;
try {
    summary = lineageQuery.findCompletedWorkers(caseId).get(0);
} finally {
    tx.rollback();
}
```

With:
```java
Awaitility.await()
    .atMost(Duration.ofSeconds(20))
    .pollInterval(Duration.ofMillis(200))
    .untilAsserted(() ->
            assertThat(lineageQuery.findCompletedWorkers(caseId)
                    .await().atMost(Duration.ofSeconds(5)))
                    .as("lineage must contain the completed worker")
                    .hasSize(1));

WorkerSummary summary = lineageQuery.findCompletedWorkers(caseId)
        .await().atMost(Duration.ofSeconds(5)).get(0);
```

Also remove `import jakarta.transaction.UserTransaction;` if `UserTransaction tx` is fully removed.

- [ ] **Step 3: Run `CaseEngineRoundTripTest` in isolation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CaseEngineRoundTripTest -q
```
Expected: BUILD SUCCESS, test passes. The real `ClaudonyReactiveWorkerContextProvider` now runs through the full provisioning path without `BlockingOperationNotAllowedException`.

If the test fails with a `BlockingOperationNotAllowedException`, this means the reactive Qhorus stack did not activate (check `application.properties` has `quarkus.datasource.qhorus.reactive=true`) or the `@IfBuildProperty` annotation is not being evaluated at build time.

If `ReactiveChannelService` or `ReactiveMessageService` is not found as a CDI bean, verify `quarkus.datasource.qhorus.reactive=true` is in the main `application.properties` (not a test profile override).

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/claudony/CaseEngineRoundTripTest.java
git commit -m "test(app): CaseEngineRoundTripTest removes @InjectMock — real reactive context provider exercised

The real ClaudonyReactiveWorkerContextProvider now runs through CaseContextChangedEventHandler.tryProvision().
No BlockingOperationNotAllowedException. No more mock bypass.

Closes #115"
```

---

## Task 10: Full test suite verification

- [ ] **Step 1: Run all three modules**

```bash
cd ~/claude/casehub/claudony
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```
Expected: all tests pass. The count should be ≥ 479 (Task deletions removed ~30 old tests; new tests added ~30; net roughly the same or slightly higher).

- [ ] **Step 2: Verify `McpServerIntegrationTest` tool count is unchanged**

The switch from `QhorusMcpTools` (blocking, 52 tools) to `ReactiveQhorusMcpTools` (reactive, same 52 tools) via `quarkus.datasource.qhorus.reactive=true` should not change the tool count. If `McpServerIntegrationTest.toolsList_includesQhorusTools` fails with a different count, check whether `ReactiveQhorusMcpTools` exposes the same 52 tools. Update the assertion and the CLAUDE.md count comment if the number differs.

- [ ] **Step 3: Update CLAUDE.md test count**

Update the line in `CLAUDE.md` that reads:
```
**479 tests passing** (as of 2026-05-15, all modules): 4 in `claudony-core` + 134 in `claudony-casehub` + 341 in `claudony-app`.
```

Change to reflect the new count and date. Run `mvn test | grep "Tests run"` to get the per-module counts.

- [ ] **Step 4: Final commit**

```bash
git add CLAUDE.md
git commit -m "docs: update test count after reactive SPI migration

Refs #115"
```

---

## Checklist: Spec Coverage

| Spec section | Task |
|---|---|
| Engine: `tryProvision()` → `Uni<Void>` | Task 7 |
| `CaseLineageQuery` reactive interface | Task 1 |
| `EmptyCaseLineageQuery` reactive | Task 1 |
| `JpaCaseLineageQuery` reactive (virtual-thread offload) | Task 2 |
| `ClaudonyReactiveCaseChannelProvider` | Task 4 |
| `ClaudonyReactiveWorkerContextProvider` | Task 5 |
| `ClaudonyReactiveWorkerProvisioner` | Task 6 |
| Delete blocking impls | Task 8 |
| `quarkus.datasource.qhorus.reactive=true` config | Task 3 |
| `CaseEngineRoundTripTest` drops `@InjectMock` | Task 9 |
| All tests pass | Task 10 |
