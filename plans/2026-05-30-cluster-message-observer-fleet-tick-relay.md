# CLUSTER MessageObserver — Fleet Tick Relay Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `FleetMessageRelayObserver`, a CLUSTER-scoped `MessageObserver` that relays a channel-name tick to all healthy Claudony fleet peers on every message, enabling browsers on any fleet node to receive real-time SSE updates for messages posted on any other node.

**Architecture:** `FleetMessageRelayObserver` (`@ApplicationScoped`, implements `MessageObserver`) observes every Qhorus message dispatch and fires a virtual thread per healthy peer, POSTing `ChannelNotifyRequest{channelName}` to each peer's `POST /api/internal/channels/notify` endpoint. The receiving node calls `ChannelEventBus.emit(channelName)` directly — never re-entering Qhorus dispatch. No relay loop is possible. The relay is tick-only (channelName, no message content); browsers fetch content from their local Qhorus handle which points at shared PostgreSQL.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny (`Multi`, `Cancellable`), MicroProfile REST Client (`RestClientBuilder`), CDI (`@ApplicationScoped`), JUnit 5 + AssertJ + RestAssured, `@QuarkusTest`

**Spec:** `docs/superpowers/specs/2026-05-30-cluster-message-observer-fleet-tick-relay-design.md`

---

## File Map

| Action | File | What it does |
|---|---|---|
| Create | `app/src/main/java/io/casehub/claudony/server/fleet/ChannelNotifyRequest.java` | Record `{String channelName}` — fleet relay payload |
| Create | `app/src/main/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserver.java` | CLUSTER-scoped `MessageObserver`; relays tick to healthy peers |
| Create | `app/src/test/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserverTest.java` | 3 `@QuarkusTest` tests |
| Modify | `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncResource.java` | Add `POST /api/internal/channels/notify`, inject `ChannelEventBus` |
| Modify | `app/src/main/java/io/casehub/claudony/server/fleet/PeerClient.java` | Add `notifyChannel(ChannelNotifyRequest)` method |
| Modify | `app/src/test/java/io/casehub/claudony/server/fleet/ChannelSyncResourceTest.java` | Add 2 tests for the notify endpoint; add bus subscription cleanup |
| Modify | `CLAUDE.md` | Update test baseline count: 520 → 525 |

---

## Task 1: Notify endpoint — write failing tests

**Files:**
- Modify: `app/src/test/java/io/casehub/claudony/server/fleet/ChannelSyncResourceTest.java`

- [ ] **Step 1: Add imports and fields to `ChannelSyncResourceTest`**

  Add after the existing `@Inject` fields and before `@AfterEach`:

  ```java
  @Inject io.casehub.claudony.server.ChannelEventBus eventBus;
  private io.smallrye.mutiny.subscription.Cancellable busSubscription;
  ```

  Add `busSubscription` cancellation to the existing `@AfterEach` cleanup method (add as the last line inside the method body):

  ```java
  if (busSubscription != null) { busSubscription.cancel(); busSubscription = null; }
  ```

- [ ] **Step 2: Add two new tests at the end of `ChannelSyncResourceTest`**

  ```java
  @Test
  void notify_noFleetKey_returns401() {
      given()
          .contentType(JSON)
          .body("{\"channelName\":\"case-test/work\"}")
      .when()
          .post("/api/internal/channels/notify")
      .then()
          .statusCode(401);
  }

  @Test
  void notify_validRequest_returns204_andTicksChannelEventBus() {
      String channelName = "case-notify-test-" + java.util.UUID.randomUUID() + "/work";
      java.util.List<Integer> ticks = new java.util.concurrent.CopyOnWriteArrayList<>();
      busSubscription = eventBus.subscribe(channelName).subscribe().with(ticks::add);

      given()
          .header("X-Api-Key", "test-fleet-key-do-not-use-in-prod")
          .contentType(JSON)
          .body("{\"channelName\":\"" + channelName + "\"}")
      .when()
          .post("/api/internal/channels/notify")
      .then()
          .statusCode(204);

      assertThat(ticks).isNotEmpty();
  }
  ```

  Note: `test-fleet-key-do-not-use-in-prod` is the fleet key set by `%test.claudony.fleet-key` in `app/src/main/resources/application.properties`. `ChannelEventBus.emit()` is synchronous on the request thread — no sleep needed.

- [ ] **Step 3: Run the new tests to verify they fail**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelSyncResourceTest -pl claudony-app
  ```

  Expected: `notify_noFleetKey_returns401` → 404 (endpoint missing); `notify_validRequest_returns204_andTicksChannelEventBus` → 404. Both fail.

---

## Task 2: Notify endpoint — implement

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/ChannelNotifyRequest.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncResource.java`

- [ ] **Step 1: Create `ChannelNotifyRequest` record**

  ```java
  package io.casehub.claudony.server.fleet;

  public record ChannelNotifyRequest(String channelName) {}
  ```

- [ ] **Step 2: Add `ChannelEventBus` injection and `notify()` endpoint to `ChannelSyncResource`**

  In `ChannelSyncResource`, add an `@Inject` field after the existing `@Inject ChannelGateway gateway;`:

  ```java
  @Inject io.casehub.claudony.server.ChannelEventBus channelEventBus;
  ```

  Add the new endpoint method after the existing `sync()` method:

  ```java
  @POST
  @Path("/notify")
  @Consumes(MediaType.APPLICATION_JSON)
  @RolesAllowed("fleet")
  public Response notify(ChannelNotifyRequest request) {
      channelEventBus.emit(request.channelName());
      return Response.noContent().build();
  }
  ```

- [ ] **Step 3: Run the tests to verify they pass**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelSyncResourceTest -pl claudony-app
  ```

  Expected: all tests in `ChannelSyncResourceTest` pass (existing 2 + new 2 = 4 total).

- [ ] **Step 4: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/claudony add \
    app/src/main/java/io/casehub/claudony/server/fleet/ChannelNotifyRequest.java \
    app/src/main/java/io/casehub/claudony/server/fleet/ChannelSyncResource.java \
    app/src/test/java/io/casehub/claudony/server/fleet/ChannelSyncResourceTest.java
  git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(fleet): #118 POST /api/internal/channels/notify — tick ChannelEventBus on fleet notify"
  ```

---

## Task 3: `PeerClient` — add `notifyChannel()` method

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/fleet/PeerClient.java`

- [ ] **Step 1: Add `notifyChannel()` to `PeerClient` interface**

  Add after the existing `syncChannel()` method:

  ```java
  @POST
  @Path("/internal/channels/notify")
  @Consumes(MediaType.APPLICATION_JSON)
  Response notifyChannel(ChannelNotifyRequest request);
  ```

  No new test needed — `PeerClient` is an interface used by `FleetMessageRelayObserver`, which is tested end-to-end in Task 4 via the loopback peer.

- [ ] **Step 2: Verify the app still compiles**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl claudony-app
  ```

  Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/claudony add \
    app/src/main/java/io/casehub/claudony/server/fleet/PeerClient.java
  git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(fleet): #118 PeerClient.notifyChannel() — fleet tick relay method"
  ```

---

## Task 4: `FleetMessageRelayObserver` — write failing tests

**Files:**
- Create: `app/src/test/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserverTest.java`

- [ ] **Step 1: Create `FleetMessageRelayObserverTest`**

  ```java
  package io.casehub.claudony.server.fleet;

  import io.casehub.claudony.server.ChannelEventBus;
  import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
  import io.casehub.qhorus.api.message.MessageType;
  import io.casehub.qhorus.testing.InMemoryChannelStore;
  import io.casehub.qhorus.testing.InMemoryMessageStore;
  import io.quarkus.test.junit.QuarkusTest;
  import io.smallrye.mutiny.subscription.Cancellable;
  import jakarta.inject.Inject;
  import org.junit.jupiter.api.AfterEach;
  import org.junit.jupiter.api.Test;

  import java.util.List;
  import java.util.UUID;
  import java.util.concurrent.CopyOnWriteArrayList;

  import static org.assertj.core.api.Assertions.assertThat;
  import static org.assertj.core.api.Assertions.assertThatCode;

  @QuarkusTest
  class FleetMessageRelayObserverTest {

      @Inject FleetMessageRelayObserver observer;
      @Inject PeerRegistry peerRegistry;
      @Inject ChannelEventBus eventBus;
      @Inject InMemoryChannelStore channelStore;
      @Inject InMemoryMessageStore messageStore;

      private Cancellable busSubscription;

      @AfterEach
      void cleanup() {
          messageStore.clear();
          channelStore.clear();
          peerRegistry.getAllPeers().stream()
                  .filter(p -> p.source() == DiscoverySource.MANUAL)
                  .forEach(p -> peerRegistry.removePeer(p.id()));
          if (busSubscription != null) { busSubscription.cancel(); busSubscription = null; }
      }

      @Test
      void onMessage_noPeers_returnsImmediately() {
          assertThatCode(() -> {
              observer.onMessage(new MessageReceivedEvent(
                      "case-no-peers/work", UUID.randomUUID(),
                      MessageType.STATUS, "sender-1", null, "content"));
              Thread.sleep(100);
          }).doesNotThrowAnyException();
      }

      @Test
      void onMessage_healthyPeer_ticksChannelEventBusViaLoopback() throws InterruptedException {
          String channelName = "case-relay-" + UUID.randomUUID() + "/work";
          int testPort = io.restassured.RestAssured.port;

          peerRegistry.addPeer("loopback-peer", "http://localhost:" + testPort,
                  "Loopback Test Peer", DiscoverySource.MANUAL, TerminalMode.DIRECT);

          List<Integer> ticks = new CopyOnWriteArrayList<>();
          busSubscription = eventBus.subscribe(channelName).subscribe().with(ticks::add);

          observer.onMessage(new MessageReceivedEvent(
                  channelName, UUID.randomUUID(),
                  MessageType.STATUS, "sender-1", null, "content"));

          Thread.sleep(500);

          assertThat(ticks).isNotEmpty();
      }

      @Test
      void onMessage_peerDown_recordsFailureWithoutCrashing() throws InterruptedException {
          peerRegistry.addPeer("down-peer", "http://localhost:19999",
                  "Down Peer", DiscoverySource.MANUAL, TerminalMode.DIRECT);

          assertThatCode(() -> {
              observer.onMessage(new MessageReceivedEvent(
                      "case-down-test/work", UUID.randomUUID(),
                      MessageType.STATUS, "sender-1", null, "content"));
              Thread.sleep(300);
          }).doesNotThrowAnyException();

          assertThat(peerRegistry.findById("down-peer")).isPresent();
      }
  }
  ```

  Notes:
  - `onMessage()` is called directly — `FleetMessageRelayObserver` is a plain `MessageObserver` SPI, not a CDI observer. The `@QuarkusTest` test injects the bean and calls the method directly.
  - `MessageType.STATUS` with non-null content is used throughout — `MessageType.EVENT` would require `content == null` (enforced by `MessageReceivedEvent`'s compact constructor).
  - The loopback peer at `http://localhost:{testPort}` causes the observer to call `POST /api/internal/channels/notify` on the test server itself. `FleetKeyClientFilter` sends `X-Api-Key: test-fleet-key-do-not-use-in-prod` (the test fleet key), which `ApiKeyAuthMechanism` accepts as `fleet` role — same mechanism that makes `ChannelFleetBroadcasterTest` work.
  - No `@TestSecurity` annotation — this is a CDI+HTTP integration test, not a pure CDI test, and `@TestSecurity` would interfere with the fleet key auth path.

- [ ] **Step 2: Run to verify it fails to compile (class not yet defined)**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl claudony-app
  ```

  Expected: `COMPILATION ERROR` — `FleetMessageRelayObserver` cannot be found.

---

## Task 5: `FleetMessageRelayObserver` — implement

**Files:**
- Create: `app/src/main/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserver.java`

- [ ] **Step 1: Create `FleetMessageRelayObserver`**

  ```java
  package io.casehub.claudony.server.fleet;

  import io.casehub.qhorus.api.gateway.MessageObserver;
  import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.inject.Inject;
  import org.eclipse.microprofile.rest.client.RestClientBuilder;
  import org.jboss.logging.Logger;

  import java.net.URI;
  import java.util.List;
  import java.util.concurrent.TimeUnit;

  /**
   * CLUSTER-scoped {@link MessageObserver} that relays a channel-name tick to every
   * healthy fleet peer on every Qhorus message dispatch.
   *
   * <p>Tick-only relay: {@link ChannelNotifyRequest} carries only {@code channelName}.
   * Peers retrieve message content from the shared PostgreSQL Qhorus instance.
   *
   * <p>Fires on both blocking {@code MessageService.dispatch()} (pre-commit on that path)
   * and reactive {@code ReactiveMessageService.dispatch()} (post-commit). Pre-commit race
   * on the blocking path is benign — spurious ticks find nothing on peer fetch.
   * Tracked: qhorus#166 (after-commit dispatch for blocking service).
   *
   * <p>Not called for LAST_WRITE overwrites — {@code MessageService} returns before
   * reaching {@code MessageObserverDispatcher} in that case.
   */
  @ApplicationScoped
  public class FleetMessageRelayObserver implements MessageObserver {

      private static final Logger LOG = Logger.getLogger(FleetMessageRelayObserver.class);

      @Inject PeerRegistry peerRegistry;

      @Override
      public Scope scope() {
          return Scope.CLUSTER;
      }

      @Override
      public void onMessage(MessageReceivedEvent event) {
          if (event.channelName() == null) return;
          List<PeerRecord> peers = peerRegistry.getHealthyPeers();
          if (peers.isEmpty()) return;
          var request = new ChannelNotifyRequest(event.channelName());
          for (PeerRecord peer : peers) {
              Thread.ofVirtual().name("channel-notify-" + peer.id())
                      .start(() -> relayToPeer(peer, request));
          }
      }

      private void relayToPeer(PeerRecord peer, ChannelNotifyRequest request) {
          try {
              PeerClient client = RestClientBuilder.newBuilder()
                      .baseUri(URI.create(peer.url()))
                      .connectTimeout(5, TimeUnit.SECONDS)
                      .readTimeout(5, TimeUnit.SECONDS)
                      .register(FleetKeyClientFilter.class)
                      .build(PeerClient.class);
              var resp = client.notifyChannel(request);
              if (resp.getStatus() >= 200 && resp.getStatus() < 300) {
                  peerRegistry.recordSuccess(peer.id());
              } else {
                  peerRegistry.recordFailure(peer.id());
                  LOG.warnf("Channel notify to peer %s returned %d", peer.url(), resp.getStatus());
              }
          } catch (Exception e) {
              peerRegistry.recordFailure(peer.id());
              LOG.warnf("Channel notify to peer %s failed: %s", peer.url(), e.getMessage());
          }
      }
  }
  ```

- [ ] **Step 2: Run all five new tests**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelSyncResourceTest,FleetMessageRelayObserverTest -pl claudony-app
  ```

  Expected: 5 tests pass (2 `ChannelSyncResourceTest` additions + 3 `FleetMessageRelayObserverTest`).

- [ ] **Step 3: Run the full test suite to check for regressions**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
  ```

  Expected: 525 passing, 0 failures (520 baseline + 5 new).

- [ ] **Step 4: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/claudony add \
    app/src/main/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserver.java \
    app/src/test/java/io/casehub/claudony/server/fleet/FleetMessageRelayObserverTest.java
  git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(fleet): #118 FleetMessageRelayObserver — CLUSTER MessageObserver tick relay to fleet peers"
  ```

---

## Task 6: Update CLAUDE.md test baseline

**Files:**
- Modify: `CLAUDE.md` (project root)

- [ ] **Step 1: Update the test baseline count**

  Find the line in `CLAUDE.md` that reads:
  ```
  **Baseline (as of 2026-05-29, after #102 fleet channel backend):** 4 in `claudony-core` + 137 in `claudony-casehub` + 379 in `claudony-app` = **520 passing, 0 failures**
  ```

  Replace with:
  ```
  **Baseline (as of 2026-05-30, after #118 CLUSTER MessageObserver):** 4 in `claudony-core` + 137 in `claudony-casehub` + 384 in `claudony-app` = **525 passing, 0 failures** (run from user terminal — requires tmux on PATH). Previous baseline: 520 (4 + 137 + 379, 2026-05-29 after #102).
  ```

  Also find and update the test description list in `claudony-app tests` section. After the line mentioning `ChannelFleetBroadcasterTest`, add:
  ```
  `FleetMessageRelayObserverTest` (3 unit tests: no peers early return, loopback tick via notify endpoint, peer-down no crash + registry intact), `ChannelSyncResourceTest` additions (2 tests: notify 401 no key, notify 204 + ChannelEventBus tick)
  ```

- [ ] **Step 2: Commit CLAUDE.md**

  ```bash
  git -C /Users/mdproctor/claude/casehub/claudony add CLAUDE.md
  git -C /Users/mdproctor/claude/casehub/claudony commit -m "docs(claude): #118 update test baseline 520→525, document FleetMessageRelayObserverTest"
  ```
