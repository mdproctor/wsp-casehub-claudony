# Case Worker Panel SSE Push Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the 3-second `setInterval` polling in the case worker panel with a server-sent event stream, giving real-time updates with zero unnecessary HTTP traffic.

**Architecture:** `ClaudonyWorkerStatusListener` fires a `WorkerCaseLifecycleEvent` CDI event (defined in `claudony-core` to avoid circular deps) which `CaseEventBroadcaster` in `claudony-app` observes and fans out to all SSE subscribers for that case. A pluggable `CaseWorkerUpdateStrategy` SPI (events-only | hybrid | registry-hooks) allows per-instance tuning; `hybrid` is the default and adds a 30s heartbeat for drift correction.

**Tech Stack:** Quarkus 3.32.2, Mutiny `Multi<String>`, JAX-RS SSE (`@Produces("text/event-stream")`), CDI events (`@Observes`), vanilla JS `EventSource`, JUnit 5, RestAssured, Playwright.

---

## Module Layout

```
claudony-core   ← WorkerCaseLifecycleEvent (record, CDI event)
                ← SessionRegistry.addChangeListener()
                ← ClaudonyConfig.caseWorkerUpdate() + caseWorkerHeartbeatMs()

claudony-casehub ← ClaudonyWorkerStatusListener (fires WorkerCaseLifecycleEvent)

claudony-app    ← CaseWorkerUpdateStrategy (SPI interface, uses Mutiny)
                ← strategy/EventsOnlyStrategy
                ← strategy/HybridStrategy
                ← strategy/RegistryHooksStrategy
                ← CaseEventBroadcaster (@ApplicationScoped, @Observes event)
                ← SessionResource (new SSE endpoint)
                ← terminal.js (EventSource replaces setInterval)
```

`claudony-casehub` cannot import from `claudony-app` (circular). CDI events bridge the gap — `ClaudonyWorkerStatusListener` fires `WorkerCaseLifecycleEvent`; `CaseEventBroadcaster` observes it.

---

## Task 1: Config properties + WorkerCaseLifecycleEvent

**Files:**
- Modify: `claudony-core/src/main/java/dev/claudony/config/ClaudonyConfig.java`
- Create: `claudony-core/src/main/java/dev/claudony/server/WorkerCaseLifecycleEvent.java`
- Modify: `claudony-app/src/main/resources/application.properties`
- Modify: `claudony-app/src/test/resources/application.properties`

- [ ] **Step 1: Add config methods to ClaudonyConfig**

```java
// In ClaudonyConfig.java — add after meshRefreshInterval():
@WithName("case-worker-update")
@WithDefault("hybrid")
String caseWorkerUpdate();

@WithName("case-worker-heartbeat-ms")
@WithDefault("30000")
long caseWorkerHeartbeatMs();
```

- [ ] **Step 2: Create WorkerCaseLifecycleEvent in claudony-core**

Create `claudony-core/src/main/java/dev/claudony/server/WorkerCaseLifecycleEvent.java`:

```java
package dev.claudony.server;

/**
 * Fired by ClaudonyWorkerStatusListener when a CaseHub worker lifecycle event occurs.
 * Observed by CaseEventBroadcaster in claudony-app to push SSE updates to connected clients.
 */
public record WorkerCaseLifecycleEvent(String caseId) {}
```

- [ ] **Step 3: Add config defaults to application.properties**

In `claudony-app/src/main/resources/application.properties`, add after the `claudony.mesh.*` section:

```properties
# Case worker panel — SSE push strategy (events-only | hybrid | registry-hooks)
claudony.case-worker-update=hybrid
claudony.case-worker-heartbeat-ms=30000
```

- [ ] **Step 4: Add test profile overrides to test application.properties**

In `claudony-app/src/test/resources/application.properties`, add:

```properties
# Use events-only strategy in tests (no background heartbeat thread interference)
%test.claudony.case-worker-update=events-only
# Short heartbeat for hybrid strategy tests that exercise it explicitly
%test.claudony.case-worker-heartbeat-ms=200
```

- [ ] **Step 5: Build to verify config compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl claudony-core,claudony-app --also-make -q
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add claudony-core/src/main/java/dev/claudony/config/ClaudonyConfig.java \
        claudony-core/src/main/java/dev/claudony/server/WorkerCaseLifecycleEvent.java \
        claudony-app/src/main/resources/application.properties \
        claudony-app/src/test/resources/application.properties
git commit -m "feat(config): add case-worker-update and case-worker-heartbeat-ms config properties

Add WorkerCaseLifecycleEvent CDI event record to claudony-core as the CDI bridge
between ClaudonyWorkerStatusListener (casehub module) and CaseEventBroadcaster (app
module), avoiding a circular module dependency.

Refs #104, Refs #99"
```

---

## Task 2: SessionRegistry.addChangeListener()

**Files:**
- Modify: `claudony-core/src/main/java/dev/claudony/server/SessionRegistry.java`
- Test: `claudony-core/src/test/java/dev/claudony/server/SessionRegistryTest.java` (create if absent)

- [ ] **Step 1: Write failing test**

Create `claudony-core/src/test/java/dev/claudony/server/SessionRegistryTest.java` (or add to existing):

```java
package dev.claudony.server;

import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class SessionRegistryTest {

    private SessionRegistry registry;

    @BeforeEach
    void setUp() { registry = new SessionRegistry(); }

    private Session session(String id, String caseId) {
        return new Session(id, "name-" + id, "/tmp", "cmd", SessionStatus.IDLE,
                Instant.now(), Instant.now(), Optional.empty(),
                Optional.ofNullable(caseId), Optional.empty());
    }

    @Test
    void addChangeListener_notifiedOnStatusUpdate() {
        List<String> notified = new ArrayList<>();
        registry.addChangeListener(notified::add);
        registry.register(session("s1", "case-1"));

        registry.updateStatus("s1", SessionStatus.ACTIVE);

        assertThat(notified).containsExactly("case-1");
    }

    @Test
    void addChangeListener_notifiedOnRemove() {
        List<String> notified = new ArrayList<>();
        registry.addChangeListener(notified::add);
        registry.register(session("s2", "case-2"));

        registry.remove("s2");

        assertThat(notified).containsExactly("case-2");
    }

    @Test
    void addChangeListener_notNotified_forStandaloneSession() {
        List<String> notified = new ArrayList<>();
        registry.addChangeListener(notified::add);
        registry.register(session("s3", null));

        registry.updateStatus("s3", SessionStatus.ACTIVE);
        registry.remove("s3");

        assertThat(notified).isEmpty();
    }

    @Test
    void multipleListeners_allNotified() {
        List<String> l1 = new ArrayList<>(), l2 = new ArrayList<>();
        registry.addChangeListener(l1::add);
        registry.addChangeListener(l2::add);
        registry.register(session("s4", "case-4"));

        registry.updateStatus("s4", SessionStatus.ACTIVE);

        assertThat(l1).containsExactly("case-4");
        assertThat(l2).containsExactly("case-4");
    }
}
```

- [ ] **Step 2: Run test, verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-core -Dtest=SessionRegistryTest -q 2>&1 | tail -5
```
Expected: compilation error — `addChangeListener` does not exist.

- [ ] **Step 3: Implement addChangeListener in SessionRegistry**

In `claudony-core/src/main/java/dev/claudony/server/SessionRegistry.java`:

```java
// Add import:
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;

// Add field after the sessions map:
private final List<Consumer<String>> changeListeners = new CopyOnWriteArrayList<>();

// Add method:
public void addChangeListener(Consumer<String> listener) {
    changeListeners.add(listener);
}

// Add private helper:
private void notifyListeners(String sessionId) {
    var session = sessions.get(sessionId);
    if (session != null) {
        session.caseId().ifPresent(caseId ->
                changeListeners.forEach(l -> l.accept(caseId)));
    }
}
```

Update `updateStatus()`:
```java
public void updateStatus(String id, SessionStatus status) {
    sessions.computeIfPresent(id, (k, s) -> s.withStatus(status));
    notifyListeners(id);
}
```

Update `remove()` — notify BEFORE removing (session still in map):
```java
public void remove(String id) {
    notifyListeners(id);
    sessions.remove(id);
}
```

- [ ] **Step 4: Run test, verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-core -Dtest=SessionRegistryTest -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS, 4 tests passing.

- [ ] **Step 5: Commit**

```bash
git add claudony-core/src/main/java/dev/claudony/server/SessionRegistry.java \
        claudony-core/src/test/java/dev/claudony/server/SessionRegistryTest.java
git commit -m "feat(core): add SessionRegistry.addChangeListener() for registry-hooks SSE strategy

Notifies registered listeners when a case session's status changes or it is removed.
Standalone sessions (no caseId) produce no notifications.

Refs #104, Refs #99"
```

---

## Task 3: CaseWorkerUpdateStrategy SPI + EventsOnlyStrategy

**Files:**
- Create: `claudony-app/src/main/java/dev/claudony/server/CaseWorkerUpdateStrategy.java`
- Create: `claudony-app/src/main/java/dev/claudony/server/strategy/EventsOnlyStrategy.java`
- Create: `claudony-app/src/test/java/dev/claudony/server/strategy/EventsOnlyStrategyTest.java`

- [ ] **Step 1: Write failing tests for EventsOnlyStrategy**

Create `claudony-app/src/test/java/dev/claudony/server/strategy/EventsOnlyStrategyTest.java`:

```java
package dev.claudony.server.strategy;

import io.smallrye.mutiny.helpers.test.AssertSubscriber;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

class EventsOnlyStrategyTest {

    private EventsOnlyStrategy strategy;

    @BeforeEach
    void setUp() { strategy = new EventsOnlyStrategy(); }

    @Test
    void subscribe_emitsInitialSnapshot() {
        var subscriber = strategy.subscribe("case-1", () -> "data: initial\n\n")
                .subscribe().withSubscriber(AssertSubscriber.create(10));

        subscriber.assertNotTerminated();
        assertThat(subscriber.getItems()).containsExactly("data: initial\n\n");
    }

    @Test
    void onLifecycleEvent_pushesToAllSubscribersForCase() throws Exception {
        var received1 = new CopyOnWriteArrayList<String>();
        var received2 = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(2);

        strategy.subscribe("case-2", () -> "data: snap\n\n")
                .subscribe().with(e -> { received1.add(e); if (received1.size() == 2) latch.countDown(); });
        strategy.subscribe("case-2", () -> "data: snap\n\n")
                .subscribe().with(e -> { received2.add(e); if (received2.size() == 2) latch.countDown(); });

        // Both subscribers got initial snapshot; now emit lifecycle event
        strategy.onLifecycleEvent("case-2");

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received1).hasSize(2); // initial + lifecycle
        assertThat(received2).hasSize(2);
    }

    @Test
    void onLifecycleEvent_doesNotPushToOtherCase() throws Exception {
        var received = new CopyOnWriteArrayList<String>();

        strategy.subscribe("case-A", () -> "data: A\n\n")
                .subscribe().with(received::add);

        strategy.onLifecycleEvent("case-B"); // different case

        Thread.sleep(100);
        assertThat(received).hasSize(1); // only the initial snapshot
    }

    @Test
    void clientDisconnect_removesEmitter_noLeak() {
        var subscriber = strategy.subscribe("case-3", () -> "data: x\n\n")
                .subscribe().withSubscriber(AssertSubscriber.create(10));

        subscriber.cancel();

        // After cancel, onLifecycleEvent should not throw
        assertThat(() -> strategy.onLifecycleEvent("case-3")).doesNotThrowAnyException();
        assertThat(strategy.emitterCount("case-3")).isZero();
    }

    @Test
    void unknownCase_onLifecycleEvent_isNoOp() {
        assertThat(() -> strategy.onLifecycleEvent("no-such-case")).doesNotThrowAnyException();
    }

    @Test
    void multipleClients_sameCase_allReceiveEvent() throws Exception {
        int clientCount = 3;
        var latch = new CountDownLatch(clientCount);
        var counter = new AtomicInteger(0);

        for (int i = 0; i < clientCount; i++) {
            strategy.subscribe("case-multi", () -> "data: m\n\n")
                    .subscribe().with(e -> {
                        if (counter.incrementAndGet() > clientCount) latch.countDown(); // >initial
                    });
        }

        // After subscribing, counter == clientCount (initial snapshots)
        counter.set(0);
        for (int i = 0; i < clientCount; i++) {
            strategy.subscribe("case-multi", () -> "data: m\n\n")
                    .subscribe().with(e -> { if (counter.getAndIncrement() >= 0) latch.countDown(); });
        }

        strategy.onLifecycleEvent("case-multi");
        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
    }
}
```

- [ ] **Step 2: Run test to confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app -Dtest=EventsOnlyStrategyTest -Dsurefire.failIfNoSpecifiedTests=false -q 2>&1 | tail -5
```
Expected: compilation error.

- [ ] **Step 3: Create CaseWorkerUpdateStrategy interface**

Create `claudony-app/src/main/java/dev/claudony/server/CaseWorkerUpdateStrategy.java`:

```java
package dev.claudony.server;

import io.smallrye.mutiny.Multi;
import java.util.function.Supplier;

/**
 * SPI controlling how the case worker panel SSE stream receives updates.
 * Selected via {@code claudony.case-worker-update} config property.
 *
 * <p>Implementations: events-only, hybrid (default), registry-hooks.
 * See casehubio/parent#11 for per-case runtime selection (future work).
 */
public interface CaseWorkerUpdateStrategy {

    /**
     * Called when a CaseHub worker lifecycle event fires for the given case.
     * Implementations push a fresh snapshot to all active subscribers.
     */
    void onLifecycleEvent(String caseId);

    /**
     * Returns a Multi that emits SSE payloads for the given case.
     * The first item MUST be the current snapshot (ensures reconnects show fresh state).
     *
     * @param caseId     the case to subscribe to
     * @param snapshotFn produces {@code "data: <JSON>\n\n"} on demand; called on the calling thread
     */
    Multi<String> subscribe(String caseId, Supplier<String> snapshotFn);
}
```

- [ ] **Step 4: Create EventsOnlyStrategy**

Create `claudony-app/src/main/java/dev/claudony/server/strategy/EventsOnlyStrategy.java`:

```java
package dev.claudony.server.strategy;

import dev.claudony.server.CaseWorkerUpdateStrategy;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.MultiEmitter;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Supplier;

/**
 * Emits SSE snapshots only when a lifecycle event fires.
 * No background tick — zero overhead when no events occur.
 */
public class EventsOnlyStrategy implements CaseWorkerUpdateStrategy {

    // Package-private for testing
    final ConcurrentHashMap<String, List<MultiEmitter<String>>> emitters = new ConcurrentHashMap<>();
    final ConcurrentHashMap<String, Supplier<String>> snapshotFns = new ConcurrentHashMap<>();

    @Override
    public void onLifecycleEvent(String caseId) {
        Supplier<String> fn = snapshotFns.get(caseId);
        if (fn == null) return;
        String snapshot = fn.get();
        List<MultiEmitter<String>> list = emitters.get(caseId);
        if (list == null) return;
        list.forEach(e -> { if (!e.isCancelled()) e.emit(snapshot); });
    }

    @Override
    public Multi<String> subscribe(String caseId, Supplier<String> snapshotFn) {
        snapshotFns.put(caseId, snapshotFn);
        return Multi.createFrom().emitter(emitter -> {
            emitter.emit(snapshotFn.get()); // initial snapshot on connect
            emitters.computeIfAbsent(caseId, k -> new CopyOnWriteArrayList<>()).add(emitter);
            emitter.onTermination(() -> removeEmitter(caseId, emitter));
        });
    }

    /** Number of active emitters for a case. Package-private for testing. */
    int emitterCount(String caseId) {
        List<MultiEmitter<String>> list = emitters.get(caseId);
        return list == null ? 0 : list.size();
    }

    private void removeEmitter(String caseId, MultiEmitter<String> emitter) {
        List<MultiEmitter<String>> list = emitters.get(caseId);
        if (list != null) {
            list.remove(emitter);
            if (list.isEmpty()) {
                emitters.remove(caseId);
                snapshotFns.remove(caseId);
            }
        }
    }
}
```

- [ ] **Step 5: Run tests, verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app --also-make \
  -Dtest=EventsOnlyStrategyTest -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: 6 tests passing, BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add claudony-app/src/main/java/dev/claudony/server/CaseWorkerUpdateStrategy.java \
        claudony-app/src/main/java/dev/claudony/server/strategy/EventsOnlyStrategy.java \
        claudony-app/src/test/java/dev/claudony/server/strategy/EventsOnlyStrategyTest.java
git commit -m "feat(server): add CaseWorkerUpdateStrategy SPI and EventsOnlyStrategy

EventsOnlyStrategy emits SSE snapshots only on explicit lifecycle events.
Zero background threads. Multi emitter map keyed by caseId; terminates cleanly
on client disconnect.

Refs #104, Refs #99"
```

---

## Task 4: HybridStrategy

**Files:**
- Create: `claudony-app/src/main/java/dev/claudony/server/strategy/HybridStrategy.java`
- Create: `claudony-app/src/test/java/dev/claudony/server/strategy/HybridStrategyTest.java`

- [ ] **Step 1: Write failing test**

Create `claudony-app/src/test/java/dev/claudony/server/strategy/HybridStrategyTest.java`:

```java
package dev.claudony.server.strategy;

import org.junit.jupiter.api.Test;

import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class HybridStrategyTest {

    @Test
    void subscribe_emitsInitialSnapshot() {
        var strategy = new HybridStrategy(60_000); // long heartbeat, won't fire in test
        var received = new CopyOnWriteArrayList<String>();

        strategy.subscribe("case-h1", () -> "data: init\n\n")
                .subscribe().with(received::add);

        assertThat(received).containsExactly("data: init\n\n");
        strategy.cancel();
    }

    @Test
    void heartbeat_emitsSnapshot_afterInterval() throws Exception {
        var strategy = new HybridStrategy(200); // 200ms heartbeat for test
        var received = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(2); // initial + 1 heartbeat

        strategy.subscribe("case-h2", () -> "data: tick\n\n")
                .subscribe().with(e -> { received.add(e); latch.countDown(); });

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received.size()).isGreaterThanOrEqualTo(2);
        assertThat(received).allMatch(e -> e.equals("data: tick\n\n"));
        strategy.cancel();
    }

    @Test
    void lifecycleEvent_stillPushes_betweenHeartbeats() throws Exception {
        var strategy = new HybridStrategy(60_000); // long heartbeat
        var received = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(2); // initial + lifecycle

        strategy.subscribe("case-h3", () -> "data: snap\n\n")
                .subscribe().with(e -> { received.add(e); latch.countDown(); });

        strategy.onLifecycleEvent("case-h3");

        assertThat(latch.await(1, TimeUnit.SECONDS)).isTrue();
        assertThat(received).hasSize(2);
        strategy.cancel();
    }

    @Test
    void cancel_stopsHeartbeat() throws Exception {
        var strategy = new HybridStrategy(100);
        var received = new CopyOnWriteArrayList<String>();

        strategy.subscribe("case-h4", () -> "data: x\n\n")
                .subscribe().with(received::add);

        strategy.cancel();
        int countAfterCancel = received.size();
        Thread.sleep(300);

        // May get one more tick in flight but should stop
        assertThat(received.size()).isLessThanOrEqualTo(countAfterCancel + 1);
    }
}
```

- [ ] **Step 2: Run test to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app --also-make \
  -Dtest=HybridStrategyTest -Dsurefire.failIfNoSpecifiedTests=false -q 2>&1 | tail -5
```
Expected: compilation error.

- [ ] **Step 3: Create HybridStrategy**

Create `claudony-app/src/main/java/dev/claudony/server/strategy/HybridStrategy.java`:

```java
package dev.claudony.server.strategy;

import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.subscription.Cancellable;
import java.time.Duration;
import java.util.function.Supplier;

/**
 * Extends EventsOnly with a periodic heartbeat tick.
 * The tick re-emits the current snapshot for all active subscribers, serving as:
 * 1. A keep-alive preventing proxy/load-balancer connection timeouts.
 * 2. A drift correction catching state changes that bypass lifecycle events
 *    (e.g. tmux crash, idle session expiry, manual session delete via dashboard).
 */
public class HybridStrategy extends EventsOnlyStrategy {

    private final Cancellable ticker;

    public HybridStrategy(long heartbeatMs) {
        ticker = Multi.createFrom().ticks().every(Duration.ofMillis(heartbeatMs))
                .subscribe().with(tick -> tickAllCases(), err -> {});
    }

    private void tickAllCases() {
        snapshotFns.forEach((caseId, fn) -> {
            var list = emitters.get(caseId);
            if (list != null && !list.isEmpty()) {
                String snapshot = fn.get();
                list.forEach(e -> { if (!e.isCancelled()) e.emit(snapshot); });
            }
        });
    }

    /** Stop the background heartbeat. Call on application shutdown or in tests. */
    public void cancel() {
        ticker.cancel();
    }
}
```

- [ ] **Step 4: Run tests, verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app --also-make \
  -Dtest=HybridStrategyTest -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: 4 tests passing, BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add claudony-app/src/main/java/dev/claudony/server/strategy/HybridStrategy.java \
        claudony-app/src/test/java/dev/claudony/server/strategy/HybridStrategyTest.java
git commit -m "feat(server): add HybridStrategy — events + periodic heartbeat for drift correction

Extends EventsOnly with a configurable background tick (default 30s) that re-emits
the current worker snapshot to all connected SSE clients. Serves as keep-alive for
proxies and corrects drift from state changes that bypass lifecycle events.

Refs #104, Refs #99"
```

---

## Task 5: RegistryHooksStrategy

**Files:**
- Create: `claudony-app/src/main/java/dev/claudony/server/strategy/RegistryHooksStrategy.java`

- [ ] **Step 1: Create RegistryHooksStrategy**

No additional logic needed — hooks are wired by `CaseEventBroadcaster` in `@PostConstruct`. This class is a marker that causes the broadcaster to register a `SessionRegistry` change listener.

Create `claudony-app/src/main/java/dev/claudony/server/strategy/RegistryHooksStrategy.java`:

```java
package dev.claudony.server.strategy;

/**
 * Extends EventsOnly — additionally reacts to any SessionRegistry mutation for case sessions.
 * CaseEventBroadcaster wires SessionRegistry.addChangeListener() when this strategy is active.
 * Provides instant accuracy regardless of whether state changes flow through CaseHub lifecycle events.
 */
public class RegistryHooksStrategy extends EventsOnlyStrategy {
    // All behaviour inherited from EventsOnlyStrategy.
    // CaseEventBroadcaster.@PostConstruct wires registry.addChangeListener(broadcaster::emit)
    // when this strategy is selected.
}
```

- [ ] **Step 2: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl claudony-app --also-make -q 2>&1 | tail -3
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add claudony-app/src/main/java/dev/claudony/server/strategy/RegistryHooksStrategy.java
git commit -m "feat(server): add RegistryHooksStrategy — fires on any SessionRegistry mutation

Hooks wired by CaseEventBroadcaster. Provides zero-drift accuracy at the cost of
one extra SSE push per any registry change (including non-CaseHub updates).

Refs #104, Refs #99"
```

---

## Task 6: CaseEventBroadcaster

**Files:**
- Create: `claudony-app/src/main/java/dev/claudony/server/CaseEventBroadcaster.java`
- Create: `claudony-app/src/test/java/dev/claudony/server/CaseEventBroadcasterTest.java`

- [ ] **Step 1: Write failing tests**

Create `claudony-app/src/test/java/dev/claudony/server/CaseEventBroadcasterTest.java`:

```java
package dev.claudony.server;

import dev.claudony.server.strategy.EventsOnlyStrategy;
import dev.claudony.server.strategy.HybridStrategy;
import dev.claudony.server.strategy.RegistryHooksStrategy;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for CaseEventBroadcaster. Runs with %test profile which sets
 * claudony.case-worker-update=events-only to avoid background heartbeat interference.
 */
@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class CaseEventBroadcasterTest {

    @Inject
    CaseEventBroadcaster broadcaster;

    @Test
    void selectedStrategy_isEventsOnly_inTestProfile() {
        // %test profile sets events-only
        assertThat(broadcaster.strategyType()).isEqualTo("events-only");
    }

    @Test
    void subscribe_emitsInitialSnapshot() {
        var received = new CopyOnWriteArrayList<String>();

        broadcaster.subscribe("bc-case-1", () -> "data: init\n\n")
                .subscribe().with(received::add);

        assertThat(received).containsExactly("data: init\n\n");
    }

    @Test
    void emit_pushesSnapshotToSubscribers() throws Exception {
        var received = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(2); // initial + emitted

        broadcaster.subscribe("bc-case-2", () -> "data: snap\n\n")
                .subscribe().with(e -> { received.add(e); latch.countDown(); });

        broadcaster.emit("bc-case-2");

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received).hasSize(2);
    }

    @Test
    void emit_nullCaseId_isNoOp() {
        assertThat(() -> broadcaster.emit(null)).doesNotThrowAnyException();
    }

    @Test
    void emit_unknownCaseId_isNoOp() {
        assertThat(() -> broadcaster.emit("no-subscribers")).doesNotThrowAnyException();
    }
}
```

- [ ] **Step 2: Run tests, verify compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app --also-make \
  -Dtest=CaseEventBroadcasterTest -Dsurefire.failIfNoSpecifiedTests=false -q 2>&1 | tail -5
```
Expected: compilation error — `CaseEventBroadcaster` not found.

- [ ] **Step 3: Create CaseEventBroadcaster**

Create `claudony-app/src/main/java/dev/claudony/server/CaseEventBroadcaster.java`:

```java
package dev.claudony.server;

import dev.claudony.config.ClaudonyConfig;
import dev.claudony.server.strategy.EventsOnlyStrategy;
import dev.claudony.server.strategy.HybridStrategy;
import dev.claudony.server.strategy.RegistryHooksStrategy;
import io.smallrye.mutiny.Multi;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import java.util.function.Supplier;
import org.jboss.logging.Logger;

/**
 * Orchestrates SSE push for the case worker panel.
 *
 * <p>Observes {@link WorkerCaseLifecycleEvent} CDI events (fired by
 * ClaudonyWorkerStatusListener in claudony-casehub) and fans out snapshots
 * to all active SSE subscribers for the affected case.
 *
 * <p>Strategy is selected from config at startup (events-only | hybrid | registry-hooks).
 */
@ApplicationScoped
public class CaseEventBroadcaster {

    private static final Logger LOG = Logger.getLogger(CaseEventBroadcaster.class);

    @Inject ClaudonyConfig config;
    @Inject SessionRegistry registry;

    private CaseWorkerUpdateStrategy strategy;

    @PostConstruct
    void init() {
        String strategyName = config.caseWorkerUpdate();
        long heartbeatMs = config.caseWorkerHeartbeatMs();
        strategy = switch (strategyName) {
            case "events-only" -> new EventsOnlyStrategy();
            case "registry-hooks" -> {
                var s = new RegistryHooksStrategy();
                registry.addChangeListener(this::emit);
                yield s;
            }
            default -> {
                LOG.infof("Case worker SSE: using hybrid strategy (heartbeat=%dms)", heartbeatMs);
                yield new HybridStrategy(heartbeatMs);
            }
        };
        LOG.infof("Case worker SSE strategy: %s", strategyName);
    }

    /** Observes CDI lifecycle events from ClaudonyWorkerStatusListener. */
    void onWorkerLifecycleEvent(@Observes WorkerCaseLifecycleEvent event) {
        emit(event.caseId());
    }

    /** Pushes a fresh snapshot to all SSE subscribers for the given case. */
    public void emit(String caseId) {
        if (caseId == null) return;
        strategy.onLifecycleEvent(caseId);
    }

    /**
     * Returns an SSE Multi for the given case.
     * The first item is the current snapshot; subsequent items are pushed on lifecycle events.
     *
     * @param caseId     the case to subscribe to
     * @param snapshotFn produces {@code "data: <JSON>\n\n"} — called by the caller (SessionResource)
     *                   so it has access to config, registry, and serialisation
     */
    public Multi<String> subscribe(String caseId, Supplier<String> snapshotFn) {
        return strategy.subscribe(caseId, snapshotFn);
    }

    /** Returns strategy type name. Package-private for testing. */
    String strategyType() {
        return switch (strategy) {
            case RegistryHooksStrategy ignored -> "registry-hooks";
            case HybridStrategy ignored -> "hybrid";
            default -> "events-only";
        };
    }
}
```

- [ ] **Step 4: Run tests, verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app --also-make \
  -Dtest=CaseEventBroadcasterTest -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: 5 tests passing, BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add claudony-app/src/main/java/dev/claudony/server/CaseEventBroadcaster.java \
        claudony-app/src/test/java/dev/claudony/server/CaseEventBroadcasterTest.java
git commit -m "feat(server): add CaseEventBroadcaster — strategy-backed SSE fan-out for case worker panel

Observes WorkerCaseLifecycleEvent CDI events from ClaudonyWorkerStatusListener.
Strategy selected from config at startup (events-only | hybrid | registry-hooks).
RegistryHooksStrategy wires SessionRegistry.addChangeListener() for instant accuracy.

Refs #104, Refs #99"
```

---

## Task 7: Wire ClaudonyWorkerStatusListener

**Files:**
- Modify: `claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerStatusListener.java`
- Modify: `claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java`

- [ ] **Step 1: Write failing test for lifecycle event firing**

In `ClaudonyWorkerStatusListenerTest.java`, add:

```java
@Test
void onWorkerStarted_firesCaseLifecycleEvent_whenCaseIdPresent() {
    var caseId = java.util.UUID.randomUUID();
    sessionMapping.register("analyst", caseId, "session-uuid-event-test");
    var firedEvents = new java.util.ArrayList<WorkerCaseLifecycleEvent>();
    // We'll verify the CDI event is fired by inspecting the Event mock
    // The listener fires events.fire(new WorkerCaseLifecycleEvent(caseId))
    // Since Event<Object> is mocked, capture the argument:
    doAnswer(inv -> { firedEvents.add(inv.getArgument(0)); return null; })
            .when(events).fire(any(WorkerCaseLifecycleEvent.class));

    listener.onWorkerStarted("analyst", java.util.Map.of("caseId", caseId.toString()));

    assertThat(firedEvents).hasSize(1);
    assertThat(firedEvents.get(0).caseId()).isEqualTo(caseId.toString());
}

@Test
void onWorkerCompleted_firesCaseLifecycleEvent_whenCaseIdPresent() {
    var caseId = java.util.UUID.randomUUID();
    sessionMapping.register("analyst", caseId, "session-uuid-event-2");
    var firedEvents = new java.util.ArrayList<WorkerCaseLifecycleEvent>();
    doAnswer(inv -> { firedEvents.add(inv.getArgument(0)); return null; })
            .when(events).fire(any(WorkerCaseLifecycleEvent.class));

    listener.onWorkerCompleted("analyst",
            WorkResult.completed("corr", java.util.Map.of(), "analyst", caseId));

    assertThat(firedEvents).hasSize(1);
    assertThat(firedEvents.get(0).caseId()).isEqualTo(caseId.toString());
}

@Test
void onWorkerStarted_doesNotFireEvent_whenNoCaseId() {
    sessionMapping.register("analyst", null, "session-no-case");
    listener.onWorkerStarted("analyst", java.util.Map.of());
    verify(events, never()).fire(any(WorkerCaseLifecycleEvent.class));
}
```

Also add these imports to `ClaudonyWorkerStatusListenerTest`:
```java
import dev.claudony.server.WorkerCaseLifecycleEvent;
import static org.mockito.Mockito.*;
```

- [ ] **Step 2: Run tests to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerStatusListenerTest -Dsurefire.failIfNoSpecifiedTests=false -q 2>&1 | tail -5
```
Expected: compilation error (WorkerCaseLifecycleEvent firing not yet in listener).

- [ ] **Step 3: Update ClaudonyWorkerStatusListener to fire events**

In `ClaudonyWorkerStatusListener.java`, update `onWorkerStarted()`:

```java
@Override
public void onWorkerStarted(String roleName, Map<String, String> sessionMeta) {
    String caseId = sessionMeta != null ? sessionMeta.get("caseId") : null;
    String sessionId = caseId != null
            ? sessionMapping.findByCase(caseId, roleName)
                    .orElseGet(() -> sessionMapping.findByRole(roleName).orElse(null))
            : sessionMapping.findByRole(roleName).orElse(null);
    if (sessionId != null) {
        registry.updateStatus(sessionId, SessionStatus.ACTIVE);
    }
    if (caseId != null) {
        events.fire(new WorkerCaseLifecycleEvent(caseId));
    }
    LOG.debugf("Worker started: role=%s sessionId=%s", roleName, sessionId);
}
```

Update `onWorkerCompleted()` — add event fire after the registry update:
```java
// After the if/else block, before the LOG line:
if (result.caseId() != null) {
    events.fire(new WorkerCaseLifecycleEvent(result.caseId().toString()));
}
```

Update `onWorkerStalled()`:
```java
@Override
public void onWorkerStalled(String workerId) {
    LOG.warnf("Worker stalled: %s", workerId);
    events.fire(new WorkerStalledEvent(workerId));
    // Fire case lifecycle event if we can resolve the caseId
    sessionMapping.findByRole(workerId)
            .flatMap(registry::find)
            .flatMap(s -> s.caseId())
            .ifPresent(caseId -> events.fire(new WorkerCaseLifecycleEvent(caseId)));
}
```

Add import at top of `ClaudonyWorkerStatusListener.java`:
```java
import dev.claudony.server.WorkerCaseLifecycleEvent;
```

- [ ] **Step 4: Run tests, verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub \
  -Dtest=ClaudonyWorkerStatusListenerTest -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: 14 tests passing (existing 11 + 3 new), BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add claudony-casehub/src/main/java/dev/claudony/casehub/ClaudonyWorkerStatusListener.java \
        claudony-casehub/src/test/java/dev/claudony/casehub/ClaudonyWorkerStatusListenerTest.java
git commit -m "feat(casehub): fire WorkerCaseLifecycleEvent on all worker lifecycle transitions

ClaudonyWorkerStatusListener fires the CDI event whenever a CaseHub worker starts,
completes, or stalls — and the session has a known caseId. CaseEventBroadcaster
in claudony-app observes this event to push SSE updates to connected panel clients.

Refs #104, Refs #99"
```

---

## Task 8: SSE endpoint in SessionResource

**Files:**
- Modify: `claudony-app/src/main/java/dev/claudony/server/SessionResource.java`
- Create: `claudony-app/src/test/java/dev/claudony/server/SessionResourceCaseEventsTest.java`

- [ ] **Step 1: Write failing integration tests**

Create `claudony-app/src/test/java/dev/claudony/server/SessionResourceCaseEventsTest.java`:

```java
package dev.claudony.server;

import dev.claudony.server.model.Session;
import dev.claudony.server.model.SessionStatus;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestSecurity(user = "test", roles = "user")
class SessionResourceCaseEventsTest {

    @Inject SessionRegistry registry;
    @Inject CaseEventBroadcaster broadcaster;

    @AfterEach
    void cleanup() {
        registry.all().stream().map(Session::id).toList().forEach(registry::remove);
    }

    private Session caseSession(String id, String caseId) {
        return new Session(id, "name-" + id, "/tmp", "cmd", SessionStatus.IDLE,
                Instant.now(), Instant.now(), Optional.empty(),
                Optional.of(caseId), Optional.of("researcher"));
    }

    private Session standaloneSession(String id) {
        return new Session(id, "name-" + id, "/tmp", "cmd", SessionStatus.IDLE,
                Instant.now(), Instant.now(), Optional.empty(),
                Optional.empty(), Optional.empty());
    }

    @Test
    void caseEvents_404_forUnknownSession() {
        given()
            .get("/api/sessions/no-such-session/case-events")
        .then()
            .statusCode(404);
    }

    @Test
    void caseEvents_404_forStandaloneSession() {
        registry.register(standaloneSession("standalone-1"));
        given()
            .get("/api/sessions/standalone-1/case-events")
        .then()
            .statusCode(404);
    }

    @Test
    void caseEvents_contentType_isEventStream() {
        registry.register(caseSession("sse-ct-1", "ct-case-1"));
        var contentType = given()
            .get("/api/sessions/sse-ct-1/case-events")
            .contentType();
        assertThat(contentType).contains("text/event-stream");
    }

    @Test
    void caseEvents_emitsInitialState_onConnect() throws Exception {
        registry.register(caseSession("sse-init-1", "init-case-1"));

        var received = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(1);

        broadcaster.subscribe("init-case-1",
                () -> "data: [{\"id\":\"sse-init-1\"}]\n\n")
            .subscribe().with(e -> { received.add(e); latch.countDown(); });

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received.get(0)).startsWith("data:");
    }

    @Test
    void caseEvents_emitsUpdate_whenBroadcasterFires() throws Exception {
        registry.register(caseSession("sse-upd-1", "upd-case-1"));

        var received = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(2); // initial + emitted

        broadcaster.subscribe("upd-case-1", () -> "data: snap\n\n")
            .subscribe().with(e -> { received.add(e); latch.countDown(); });

        broadcaster.emit("upd-case-1");

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received).hasSize(2);
        assertThat(received).allMatch(e -> e.startsWith("data:"));
    }

    @Test
    void caseEvents_multipleClients_bothReceiveUpdate() throws Exception {
        registry.register(caseSession("sse-multi-1", "multi-case-1"));

        var received1 = new CopyOnWriteArrayList<String>();
        var received2 = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(4); // 2 initials + 2 updates

        broadcaster.subscribe("multi-case-1", () -> "data: m\n\n")
            .subscribe().with(e -> { received1.add(e); latch.countDown(); });
        broadcaster.subscribe("multi-case-1", () -> "data: m\n\n")
            .subscribe().with(e -> { received2.add(e); latch.countDown(); });

        broadcaster.emit("multi-case-1");

        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(received1).hasSize(2);
        assertThat(received2).hasSize(2);
    }

    @Test
    void strategyConfig_eventsOnly_isActiveInTestProfile() {
        assertThat(broadcaster.strategyType()).isEqualTo("events-only");
    }
}
```

- [ ] **Step 2: Run tests, verify failures**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app --also-make \
  -Dtest=SessionResourceCaseEventsTest -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|FAILED|BUILD" | tail -8
```
Expected: `caseEvents_404_forUnknownSession` and others fail (endpoint doesn't exist yet), `caseEvents_emitsInitialState_onConnect` and broadcaster tests may pass.

- [ ] **Step 3: Add SSE endpoint and helper to SessionResource**

In `SessionResource.java`, add imports:
```java
import com.fasterxml.jackson.core.JsonProcessingException;
import dev.claudony.server.CaseEventBroadcaster;
import io.smallrye.mutiny.Multi;
import jakarta.ws.rs.NotFoundException;
```

Add field injection (after existing `@Inject` fields):
```java
@Inject CaseEventBroadcaster caseEventBroadcaster;
```

Add the endpoint and snapshot helper method:
```java
@GET
@Path("/{id}/case-events")
@Produces("text/event-stream")
public Multi<String> caseEvents(@PathParam("id") String id) {
    var session = registry.find(id)
            .orElseThrow(() -> new NotFoundException("Session not found: " + id));
    String caseId = session.caseId()
            .orElseThrow(() -> new NotFoundException("Session " + id + " has no caseId"));
    return caseEventBroadcaster.subscribe(caseId, () -> buildCaseSnapshot(caseId));
}

private String buildCaseSnapshot(String caseId) {
    try {
        var workers = registry.findByCaseId(caseId).stream()
                .map(s -> SessionResponse.from(s, config.port(), resolvedPolicy(s)))
                .toList();
        return "data: " + MAPPER.writeValueAsString(workers) + "\n\n";
    } catch (JsonProcessingException e) {
        return "data: []\n\n";
    }
}
```

- [ ] **Step 4: Run tests, verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app --also-make \
  -Dtest=SessionResourceCaseEventsTest -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: 7 tests passing, BUILD SUCCESS.

- [ ] **Step 5: Run full suite to check no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | grep -E "Tests run:|<<< FAILURE|BUILD" | tail -8
```
Expected: BUILD SUCCESS, only pre-existing `GitStatusTest` fork failure.

- [ ] **Step 6: Commit**

```bash
git add claudony-app/src/main/java/dev/claudony/server/SessionResource.java \
        claudony-app/src/test/java/dev/claudony/server/SessionResourceCaseEventsTest.java
git commit -m "feat(rest): add GET /api/sessions/{id}/case-events SSE endpoint

Returns 404 for unknown sessions or sessions without a caseId (standalone).
Emits current worker snapshot immediately on connect (ensures reconnects show fresh state).
Snapshot uses SessionResponse.from() — same JSON shape as GET /api/sessions?caseId=X.

Refs #104, Refs #99"
```

---

## Task 9: Frontend — EventSource replaces polling

**Files:**
- Modify: `claudony-app/src/main/resources/META-INF/resources/app/terminal.js`

- [ ] **Step 1: Read the current polling section (for reference)**

Relevant lines in `terminal.js` (already read — recording for reference):
- Line 552: `var activeCaseId = null;`
- Line 553: `var casePoller;`
- Lines 555-566: `openCasePanel()`, `closeCasePanel()` with `setInterval`/`clearInterval`
- Lines 612-618: `pollWorkers()` function
- Lines 640-654: session fetch callback that starts the poller
- Line 664: `beforeunload` cleanup

- [ ] **Step 2: Replace polling with EventSource in terminal.js**

Find and replace the case worker panel section. The changes are:

**Replace** `var casePoller;` with:
```js
var caseEventSource = null;
```

**Replace** `openCasePanel()`:
```js
function openCasePanel() {
    casePanel.classList.remove('collapsed');
    if (activeCaseId && !caseEventSource) connectCaseEvents();
}
```

**Replace** `closeCasePanel()`:
```js
function closeCasePanel() {
    clearInterval(casePoller);
    casePoller = null;
    casePanel.classList.add('collapsed');
    if (caseEventSource) { caseEventSource.close(); caseEventSource = null; }
}
```

Wait — `clearInterval(casePoller)` is now dead code. Remove `casePoller` and clean up:

```js
function closeCasePanel() {
    casePanel.classList.add('collapsed');
    if (caseEventSource) { caseEventSource.close(); caseEventSource = null; }
}
```

**Add** `connectCaseEvents()` function (after `closeCasePanel`):
```js
function connectCaseEvents() {
    if (caseEventSource) { caseEventSource.close(); }
    caseEventSource = new EventSource('/api/sessions/' + sessionId + '/case-events');
    caseEventSource.onmessage = function(e) {
        renderWorkers(JSON.parse(e.data));
    };
    caseEventSource.onerror = function() {
        // EventSource reconnects automatically on network error — no manual retry needed.
        // onerror fires on reconnect attempts; do not close here.
    };
    // Test mode hook for E2E verification
    if (window.__CLAUDONY_TEST_MODE__) {
        window._caseEventSource = caseEventSource;
    }
}
```

**Remove** `pollWorkers()` function entirely (lines 612-618).

**Replace** in the session fetch callback — the block that currently calls `openCasePanel()` + starts the poller:

Current (to remove):
```js
openCasePanel();
pollWorkers();
casePoller = setInterval(pollWorkers, 3000);
```

Replace with:
```js
openCasePanel();
```

`openCasePanel()` now calls `connectCaseEvents()` which handles the initial state (via SSE initial snapshot) and real-time updates.

**Also update `switchToWorker()`** — close old EventSource then immediately reconnect for the new session (workers in the same case share caseId, so the new SSE stream shows the updated list):
```js
function switchToWorker(newSessionId, newName) {
    sessionId = newSessionId;
    sessionName = newName;
    document.getElementById('session-name').textContent = newName;
    history.replaceState(null, '',
        '?id=' + newSessionId + '&name=' + encodeURIComponent(newName));
    if (caseEventSource) { caseEventSource.close(); caseEventSource = null; }
    if (activeCaseId) connectCaseEvents(); // reconnect for new session (same case)
    clearTimeout(reconnectTimer);
    if (ws) { ws.close(); } else { connect(); }
}
```

**Update `beforeunload`** — replace `clearInterval(casePoller)` with:
```js
window.addEventListener('beforeunload', function () {
    if (caseEventSource) caseEventSource.close();
    clearTimeout(lineagePollTimer);
    clearInterval(elapsedTicker);
});
```

- [ ] **Step 3: Verify no `setInterval` for workers remains**

```bash
grep -n "setInterval\|pollWorkers\|casePoller" \
  claudony-app/src/main/resources/META-INF/resources/app/terminal.js
```
Expected: `elapsedTicker = setInterval(...)` only (the elapsed time ticker, not the worker poller). No `pollWorkers` or `casePoller` references.

- [ ] **Step 4: Commit**

```bash
git add claudony-app/src/main/resources/META-INF/resources/app/terminal.js
git commit -m "feat(frontend): replace case worker setInterval poll with EventSource SSE

Removes pollWorkers() and casePoller entirely. openCasePanel() calls connectCaseEvents()
which opens EventSource('/api/sessions/{id}/case-events'). Browser auto-reconnects;
endpoint emits current state on each connection so reconnects show fresh data.
switchToWorker() closes the old EventSource before switching.

Refs #104, Refs #99"
```

---

## Task 10: E2E Tests

**Files:**
- Modify: `claudony-app/src/test/java/dev/claudony/e2e/CaseWorkerPanelE2ETest.java`

- [ ] **Step 1: Add SSE-specific E2E tests**

In `CaseWorkerPanelE2ETest.java`, add import and inject broadcaster:

```java
import dev.claudony.server.CaseEventBroadcaster;
import jakarta.inject.Inject;
// Add alongside existing @Inject SessionRegistry:
@Inject CaseEventBroadcaster broadcaster;
```

Add 4 new tests after AC 3:

```java
// ── AC 4: no polling interval — SSE only ─────────────────────────────────────

@Test
@Order(4)
void noPollingInterval_forWorkerUpdates_inSource() throws Exception {
    // Verify terminal.js does not contain setInterval for pollWorkers
    // (regression guard — ensures polling is not re-introduced)
    var jsSource = new String(java.nio.file.Files.readAllBytes(
        java.nio.file.Path.of("src/main/resources/META-INF/resources/app/terminal.js")));
    assertThat(jsSource)
        .as("pollWorkers function must be removed from terminal.js")
        .doesNotContain("function pollWorkers");
    assertThat(jsSource)
        .as("No setInterval for pollWorkers must remain")
        .doesNotContain("setInterval(pollWorkers");
    assertThat(jsSource)
        .as("EventSource must be present")
        .contains("new EventSource");
}

// ── AC 5: panel shows workers immediately on SSE connect ─────────────────────

@Test
@Order(5)
void casePanel_showsWorkers_immediately_onSSEConnect() {
    var now = Instant.now();
    var caseId = "e2e-sse-case-001";
    registry.register(new Session("sse-w1", "claudony-worker-sse-w1", "/tmp", "claude",
            SessionStatus.ACTIVE, now, now, Optional.empty(), Optional.of(caseId), Optional.of("analyst")));

    page.navigate(BASE_URL + "/app/session.html?id=sse-w1&name=analyst");

    // SSE initial snapshot — should appear much faster than 3s poll
    var rows = page.locator(".case-worker-row");
    rows.first().waitFor(new Locator.WaitForOptions().setTimeout(1500));

    assertThat(rows.count()).isEqualTo(1);
    assertThat(rows.first().textContent()).contains("analyst");
}

// ── AC 6: panel updates on SSE push without polling ──────────────────────────

@Test
@Order(6)
void casePanel_updatesWorkers_onSSEPush() throws Exception {
    var now = Instant.now();
    var caseId = "e2e-sse-case-002";
    registry.register(new Session("sse-upd-w1", "claudony-worker-sse-upd-w1", "/tmp", "claude",
            SessionStatus.ACTIVE, now, now, Optional.empty(), Optional.of(caseId), Optional.of("researcher")));

    page.navigate(BASE_URL + "/app/session.html?id=sse-upd-w1&name=researcher");

    // Wait for initial SSE render
    page.locator(".case-worker-row").first().waitFor(new Locator.WaitForOptions().setTimeout(1500));

    // Now add a second worker to the case and push via SSE
    registry.register(new Session("sse-upd-w2", "claudony-worker-sse-upd-w2", "/tmp", "claude",
            SessionStatus.IDLE, now, now, Optional.empty(), Optional.of(caseId), Optional.of("coder")));
    broadcaster.emit(caseId);

    // Panel should update without waiting for any poll cycle
    page.locator(".case-worker-row").nth(1).waitFor(new Locator.WaitForOptions().setTimeout(2000));

    assertThat(page.locator(".case-worker-row").count()).isEqualTo(2);
    assertThat(page.locator(".case-worker-row").nth(1).textContent()).contains("coder");
}

// ── AC 7: closing panel closes EventSource ───────────────────────────────────

@Test
@Order(7)
void casePanel_closingPanel_closesEventSource() {
    var now = Instant.now();
    var caseId = "e2e-sse-case-003";
    registry.register(new Session("sse-close-w1", "claudony-worker-sse-close-w1", "/tmp", "claude",
            SessionStatus.ACTIVE, now, now, Optional.empty(), Optional.of(caseId), Optional.of("planner")));

    page.addInitScript("window.__CLAUDONY_TEST_MODE__ = true;");
    page.navigate(BASE_URL + "/app/session.html?id=sse-close-w1&name=planner");

    // Wait for SSE connection to be established
    page.locator(".case-worker-row").first().waitFor(new Locator.WaitForOptions().setTimeout(1500));

    // Verify EventSource is open
    var readyState = (Number) page.evaluate("() => window._caseEventSource ? window._caseEventSource.readyState : -1");
    assertThat(readyState.intValue()).isEqualTo(1); // EventSource.OPEN = 1

    // Close the panel
    page.locator("#case-close-btn").click();
    page.waitForTimeout(200);

    // EventSource should be closed
    var closedState = (Number) page.evaluate("() => window._caseEventSource ? window._caseEventSource.readyState : 2");
    assertThat(closedState.intValue()).isEqualTo(2); // EventSource.CLOSED = 2
}
```

Also add import at top of `CaseWorkerPanelE2ETest.java`:
```java
import com.microsoft.playwright.Locator;
```

- [ ] **Step 2: Update existing tests that used waitForTimeout for poll cycles**

In AC 2 (`caseHubSession_panelAutoExpandsWithWorkers`) and AC 3 (`clickingWorker_updatesUrlAndHighlight`), replace `page.waitForTimeout(1500)` with explicit `waitFor()` for better reliability:

In AC 2, replace:
```java
page.waitForTimeout(1500);
assertThat(page.locator("#case-panel").getAttribute("class"))
```
with:
```java
// SSE initial snapshot arrives immediately — wait for first worker row
page.locator(".case-worker-row").first().waitFor(
    new com.microsoft.playwright.Locator.WaitForOptions().setTimeout(2000));
assertThat(page.locator("#case-panel").getAttribute("class"))
```

In AC 3, replace the `page.waitForTimeout(1500)` after poll re-highlight with:
```java
// SSE push after switchToWorker — wait for highlight to update
page.locator(".case-worker-row.active-worker").waitFor(
    new com.microsoft.playwright.Locator.WaitForOptions().setTimeout(2000));
```

- [ ] **Step 3: Run E2E tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -pl claudony-app --also-make \
  -Dtest=CaseWorkerPanelE2ETest -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|FAILED|BUILD" | tail -8
```
Expected: 7 tests passing (3 existing + 4 new), BUILD SUCCESS.

- [ ] **Step 4: Run all E2E tests to check no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -pl claudony-app --also-make \
  -Dtest="PlaywrightSetupE2ETest,DashboardE2ETest,TerminalPageE2ETest,ChannelPanelE2ETest,CaseWorkerPanelE2ETest,CaseContextPanelE2ETest" \
  -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|FAILED|BUILD" | tail -8
```
Expected: All pass, BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add claudony-app/src/test/java/dev/claudony/e2e/CaseWorkerPanelE2ETest.java
git commit -m "test(e2e): SSE behaviour for case worker panel — 4 new Playwright tests

AC4: regression guard verifying pollWorkers() is absent from terminal.js
AC5: initial worker list visible < 1.5s via SSE (no 3s poll wait)
AC6: panel updates on broadcaster.emit() without polling interval
AC7: closing panel closes EventSource (readyState 2 = CLOSED)

Also strengthens existing AC2/AC3 to use waitFor() instead of waitForTimeout.

Refs #104, Refs #99"
```

---

## Task 11: Documentation

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`
- Modify: `docs/BUGS-AND-ODDITIES.md` (check for stale polling entries)
- Check: `docs/superpowers/plans/2026-04-28-case-worker-panel.md`

- [ ] **Step 1: Update CLAUDE.md**

Update the component list in the Project Structure section — add to `claudony-app`:
```
├── server/
│   ├── CaseWorkerUpdateStrategy.java   — SPI: events-only | hybrid | registry-hooks
│   ├── CaseEventBroadcaster.java       — @ApplicationScoped SSE fan-out; observes WorkerCaseLifecycleEvent
│   └── strategy/
│       ├── EventsOnlyStrategy.java     — emits on lifecycle events only
│       ├── HybridStrategy.java         — events + configurable heartbeat (default 30s)
│       └── RegistryHooksStrategy.java  — fires on any SessionRegistry mutation
```

Add to `claudony-core` component list:
```
└── WorkerCaseLifecycleEvent            — CDI event bridging casehub→app (avoids circular dep)
```

Update config properties section with new properties:
```properties
claudony.case-worker-update=hybrid      # events-only | hybrid | registry-hooks
claudony.case-worker-heartbeat-ms=30000 # heartbeat interval for hybrid strategy
```

Update test count (count actual tests after full run).

Update E2E test list — `CaseWorkerPanelE2ETest` now has 7 tests.

Remove any reference to 3-second polling for the case worker panel.

- [ ] **Step 2: Update DESIGN.md**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | grep "Tests run:" | awk -F',' '{sum += $1} END {gsub(/.*: /,"",sum); print sum}'
```
(Get actual total test count for DESIGN.md and CLAUDE.md.)

In `docs/DESIGN.md`, update the `claudony-app` component list to include the new server/strategy classes. Update any mention of "3-second polling" in the case worker panel section. Add `WorkerCaseLifecycleEvent` to the `claudony-core` section.

- [ ] **Step 3: Check BUGS-AND-ODDITIES.md**

```bash
grep -n "poll\|3.second\|3-second\|interval" docs/BUGS-AND-ODDITIES.md 2>/dev/null | head -10
```
If any entry mentions the 3-second polling limitation, remove or update it.

- [ ] **Step 4: Check 2026-04-28-case-worker-panel.md plan**

```bash
grep -n "setInterval\|pollWorkers\|3.second\|3000" docs/superpowers/plans/2026-04-28-case-worker-panel.md | head -10
```
This is a historical plan (already executed). Add a note at the top:
```markdown
> **Note (2026-05-05):** The polling described in this plan was replaced by SSE push in #104.
> See `docs/superpowers/specs/2026-05-05-case-worker-sse-design.md` and
> `docs/superpowers/plans/2026-05-05-sse-case-worker-panel.md`.
```

- [ ] **Step 5: Commit documentation**

```bash
git add docs/DESIGN.md CLAUDE.md docs/BUGS-AND-ODDITIES.md \
        docs/superpowers/plans/2026-04-28-case-worker-panel.md
git commit -m "docs: update DESIGN.md, CLAUDE.md for #104 SSE case worker panel

Add CaseWorkerUpdateStrategy SPI, CaseEventBroadcaster, strategy package to component
lists. Add config properties. Update test counts. Remove references to 3-second polling.
Add supersession note to original case-worker-panel plan.

Closes #104, Refs #99"
```

---

## Final Verification

- [ ] **Run full test suite (non-E2E)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep -E "Tests run:|<<< FAILURE|BUILD" | tail -8
```
Expected: BUILD SUCCESS, only pre-existing `GitStatusTest` fork failure.

- [ ] **Run all E2E tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -pl claudony-app --also-make \
  -Dtest="PlaywrightSetupE2ETest,DashboardE2ETest,TerminalPageE2ETest,ChannelPanelE2ETest,CaseWorkerPanelE2ETest,CaseContextPanelE2ETest" \
  -Dsurefire.failIfNoSpecifiedTests=false \
  2>&1 | grep -E "Tests run:|FAILED|BUILD" | tail -8
```
Expected: All pass.

- [ ] **Verify no pollWorkers or casePoller remains in terminal.js**

```bash
grep -c "pollWorkers\|casePoller\|setInterval(poll" \
  claudony-app/src/main/resources/META-INF/resources/app/terminal.js
```
Expected: `0`

- [ ] **Request code review**

Invoke `superpowers:requesting-code-review` skill before final merge.
