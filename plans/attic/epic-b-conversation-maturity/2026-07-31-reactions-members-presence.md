# Reactions, Members, and Presence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #178 — feat: reactions and member/presence panels
**Issue group:** #178

**Goal:** Make the member panel, reactions, and presence functional
end-to-end — member join/leave, auto-join on post, reaction SSE push,
presence via subscriber count, mesh panel upgrade to `<channel-feed>`.

**Architecture:** All backend changes are in `MeshResource.java` —
inject `ChannelEventBus`, add member join/leave endpoints, auto-join
on message post, tick `ChannelEventBus` on reaction changes, add
presence endpoint. Frontend: upgrade mesh panel channel view to use
`<channel-feed>` with reactions.

**Tech Stack:** Java 21/26, Quarkus 3.32, Qhorus 0.2-SNAPSHOT
(`ChannelMembershipService`, `ReactionService`, `ChannelEventBus`),
LitElement, blocks-ui (`<channel-feed>`), RestAssured, Mockito

## Global Constraints

- IntelliJ MCP for all `.java`/`.ts` edits
- `ChannelMembershipService.join()` is idempotent — returns existing membership
- `ChannelEventBus.subscriberCount()` must be made public (currently package-private)
- `@TestSecurity(user = "test", roles = "user")` on all QuarkusTest classes
- Qhorus test cleanup: `channelStore.clear()` + `messageStore.clear()` in `@AfterEach`

---

### Task 1: Member join/leave endpoints + auto-join on post

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/MeshResource.java`
  (add `joinChannel`, `leaveChannel` methods; modify `postMessage` for auto-join;
  inject `ChannelEventBus`)
- Modify: `app/src/main/java/io/casehub/claudony/server/ChannelEventBus.java`
  (make `subscriberCount` public)
- Modify: `app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java`
  (add member and auto-join tests)

**Interfaces:**
- Consumes: `ChannelMembershipService.join(UUID channelId, String memberId)` → `ChannelMembership`,
  `ChannelMembershipService.leave(UUID channelId, String memberId)` → `void`,
  `ChannelMembershipService.listMembers(UUID channelId)` → `List<ChannelMembership>`,
  `ChannelService.findByName(String name)` → `Optional<Channel>`,
  `SecurityIdentity.getPrincipal().getName()` → `String`
- Produces: `POST /api/mesh/channels/{name}/members` → 200 + ChannelMembership,
  `DELETE /api/mesh/channels/{name}/members` → 204,
  `postMessage` auto-joins sender as member

- [ ] **Step 1: Write failing tests for join/leave and auto-join**

Add to `MeshResourceTest.java` using `ide_insert_member`:

```java
@Test
void joinChannel_returnsOkWithMembership() {
    channelStore.put(testChannel("team/chat"));

    given()
        .when().post("/api/mesh/channels/team%2Fchat/members")
        .then()
            .statusCode(200)
            .body("memberId", containsString("test"));
}

@Test
void joinChannel_unknownChannel_returns404() {
    given()
        .when().post("/api/mesh/channels/does-not-exist/members")
        .then().statusCode(404);
}

@Test
void joinChannel_idempotent_sameResult() {
    channelStore.put(testChannel("team/idem"));

    given().when().post("/api/mesh/channels/team%2Fidem/members")
        .then().statusCode(200);

    given().when().post("/api/mesh/channels/team%2Fidem/members")
        .then().statusCode(200)
        .body("memberId", containsString("test"));
}

@Test
void leaveChannel_returns204() {
    var ch = testChannel("team/leave");
    channelStore.put(ch);
    membershipService.join(ch.id(), "human:test");

    given()
        .when().delete("/api/mesh/channels/team%2Fleave/members")
        .then().statusCode(204);
}

@Test
void leaveChannel_unknownChannel_returns404() {
    given()
        .when().delete("/api/mesh/channels/does-not-exist/members")
        .then().statusCode(404);
}

@Test
void postMessage_autoJoinsMember() {
    var ch = testChannel("team/autojoin");
    channelStore.put(ch);

    given().contentType("application/json")
        .body("{\"content\": \"hello\", \"type\": \"STATUS\"}")
        .when().post("/api/mesh/channels/team%2Fautojoin/messages")
        .then().statusCode(200);

    var members = membershipService.listMembers(ch.id());
    assertThat(members).extracting("memberId").contains("human:test");
}
```

Also add a `testChannel` helper and inject `membershipService` if not
already present:

```java
@Inject ChannelMembershipService membershipService;

private static Channel testChannel(String name) {
    return new Channel(UUID.randomUUID(), name, name, ChannelSemantic.APPEND,
            List.of(), List.of(), List.of(), null, null, null, null,
            false, false, null, List.of(), List.of(), List.of(), null,
            "default", java.time.Instant.now(), null);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -nsu -pl app -Dtest="MeshResourceTest#joinChannel_returnsOkWithMembership" -Dsurefire.failIfNoSpecifiedTests=false -f pom.xml --also-make`
Expected: FAIL — 404 (no endpoint)

- [ ] **Step 3: Make ChannelEventBus.subscriberCount public**

Change visibility from package-private to public in `ChannelEventBus.java`:

```java
public int subscriberCount(String channelName) {
```

- [ ] **Step 4: Inject ChannelEventBus into MeshResource**

Add field after existing `membershipService` injection:

```java
@Inject
ChannelEventBus channelEventBus;
```

- [ ] **Step 5: Implement join and leave endpoints**

Add after the existing `members()` method using `ide_insert_member`:

```java
@POST
@Path("/channels/{name}/members")
public Response joinChannel(@PathParam("name") String name) {
    var channel = channelService.findByName(name);
    if (channel.isEmpty()) { return Response.status(404).build(); }
    String actorId = "human:" + securityIdentity.getPrincipal().getName();
    var membership = membershipService.join(channel.get().id(), actorId);
    return Response.ok(membership).build();
}

@jakarta.ws.rs.DELETE
@Path("/channels/{name}/members")
public Response leaveChannel(@PathParam("name") String name) {
    var channel = channelService.findByName(name);
    if (channel.isEmpty()) { return Response.status(404).build(); }
    String actorId = "human:" + securityIdentity.getPrincipal().getName();
    membershipService.leave(channel.get().id(), actorId);
    return Response.noContent().build();
}
```

- [ ] **Step 6: Add auto-join to postMessage**

In `postMessage()`, after the successful `dashboard.sendHumanMessage()`
call and before the `return Response.ok(result).build()`, add:

```java
var ch = channelService.findByName(name);
if (ch.isPresent()) {
    try { membershipService.join(ch.get().id(), sender); }
    catch (Exception ignored) {}
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -nsu -pl app -Dtest="MeshResourceTest#joinChannel*+MeshResourceTest#leaveChannel*+MeshResourceTest#postMessage_autoJoinsMember" -Dsurefire.failIfNoSpecifiedTests=false -f pom.xml --also-make`
Expected: all pass

- [ ] **Step 8: Commit**

```
git add app/src/main/java/io/casehub/claudony/server/MeshResource.java \
       app/src/main/java/io/casehub/claudony/server/ChannelEventBus.java \
       app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java
git commit -m "feat(#178): member join/leave endpoints and auto-join on post

POST/DELETE /api/mesh/channels/{name}/members for join/leave. postMessage
auto-joins the sender as a channel member. ChannelEventBus.subscriberCount
made public for presence endpoint (next commit).

Refs #178"
```

---

### Task 2: Reaction SSE push + presence endpoint

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/MeshResource.java`
  (add `presence` method; add `channelEventBus.emit()` to reaction endpoints)
- Modify: `app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java`
  (add presence and reaction-tick tests)

**Interfaces:**
- Consumes: `ChannelEventBus.subscriberCount(String)` → `int`,
  `ChannelEventBus.emit(String)` → `void`
- Produces: `GET /api/mesh/channels/{name}/presence` → `{"subscribers": N}`,
  reaction add/remove ticks SSE

- [ ] **Step 1: Write failing tests**

```java
@Test
void presence_returnsSubscriberCount() {
    channelStore.put(testChannel("team/presence"));

    given()
        .when().get("/api/mesh/channels/team%2Fpresence/presence")
        .then()
            .statusCode(200)
            .body("subscribers", equalTo(0));
}

@Test
void presence_unknownChannel_returns404() {
    given()
        .when().get("/api/mesh/channels/does-not-exist/presence")
        .then().statusCode(404);
}

@Test
void addReaction_returnsOk() {
    var ch = testChannel("team/react");
    channelStore.put(ch);
    messageStore.put(testMessage(ch.id(), 1L));

    given().contentType("application/json")
        .body("{\"emoji\": \"👍\"}")
        .when().post("/api/mesh/channels/team%2Freact/messages/1/reactions")
        .then().statusCode(200);
}
```

Also add a `testMessage` helper:

```java
private static Message testMessage(UUID channelId, Long id) {
    return new Message(id, channelId, "agent:test", MessageType.STATUS,
            "test", null, null, null, null, null,
            ActorType.AGENT, java.time.Instant.now(), null, 0);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: `presence` returns 404 (no endpoint); `addReaction` may pass
already (endpoint exists) — the SSE tick is not directly testable via
RestAssured but the presence endpoint is.

- [ ] **Step 3: Implement presence endpoint**

Add after `members()` method:

```java
@GET
@Path("/channels/{name}/presence")
public Response presence(@PathParam("name") String name) {
    var channel = channelService.findByName(name);
    if (channel.isEmpty()) { return Response.status(404).build(); }
    int count = channelEventBus.subscriberCount(name);
    return Response.ok(java.util.Map.of("subscribers", count)).build();
}
```

- [ ] **Step 4: Add SSE tick to reaction endpoints**

In `addReaction()`, after `reactionService.react(...)`:

```java
channelEventBus.emit(name);
```

In `removeReaction()`, after `reactionService.unreact(...)`:

```java
channelEventBus.emit(name);
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -nsu -pl app -Dtest=MeshResourceTest -Dsurefire.failIfNoSpecifiedTests=false -f pom.xml --also-make`
Expected: all tests pass including new ones

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -nsu -f pom.xml`
Expected: all pass (except known GitStatusTest worktree issue)

- [ ] **Step 7: Commit**

```
git add app/src/main/java/io/casehub/claudony/server/MeshResource.java \
       app/src/test/java/io/casehub/claudony/server/MeshResourceTest.java
git commit -m "feat(#178): presence endpoint and reaction SSE push

GET /api/mesh/channels/{name}/presence returns active SSE subscriber
count. Reaction add/remove now ticks ChannelEventBus so SSE subscribers
receive real-time reaction updates.

Refs #178"
```

---

### Task 3: Mesh panel upgrade — channel-feed with reactions

**Files:**
- Modify: `app/src/main/webui/src/components/claudony-mesh-panel.ts`
  (replace raw text rendering with `<channel-feed>`, add reaction fetching,
  add member count display)

**Interfaces:**
- Consumes: `GET /api/mesh/channels/{name}/timeline?limit=50` → timeline entries,
  `POST /api/mesh/channels/{name}/reactions/batch` → reaction groups,
  `GET /api/mesh/channels/{name}/members` → member list,
  `GET /api/mesh/channels/{name}/presence` → `{subscribers: N}`,
  `<channel-feed .messages=${} .reactions=${}>` blocks-ui component
- Produces: Rich channel view in mesh panel with reactions and member count

- [ ] **Step 1: Add state properties for timeline, reactions, members**

Add `@state()` properties to the mesh panel class:

```typescript
@state() private _channelMessages: QhorusMessage[] = [];
@state() private _channelReactions: Reaction[] = [];
@state() private _channelMembers: number = 0;
```

Add imports for `QhorusMessage`, `Reaction` from channel-adapter, and
import `channel-feed` and `channel-input` from blocks-ui.

- [ ] **Step 2: Add fetch methods**

```typescript
private async _fetchChannelTimeline(name: string): Promise<void> {
  try {
    const resp = await fetchWithAuth(`/api/mesh/channels/${encodeURIComponent(name)}/timeline?limit=50`);
    if (!resp.ok) { this._channelMessages = []; return; }
    const data = await resp.json();
    this._channelMessages = data.map((e: any) => toQhorusMessage(e));
  } catch { this._channelMessages = []; }
}

private async _fetchChannelReactions(name: string): Promise<void> {
  if (this._channelMessages.length === 0) { this._channelReactions = []; return; }
  const messageIds = this._channelMessages.map(m => Number(m.id)).filter(id => !isNaN(id));
  if (messageIds.length === 0) { this._channelReactions = []; return; }
  try {
    const resp = await fetchWithAuth(`/api/mesh/channels/${encodeURIComponent(name)}/reactions/batch`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ messageIds }),
    });
    if (!resp.ok) { this._channelReactions = []; return; }
    const data = await resp.json();
    const reactions: Reaction[] = [];
    for (const [msgId, groups] of Object.entries(data)) {
      for (const g of groups as any[]) {
        for (const actorId of g.actorIds) {
          reactions.push({ messageId: msgId, emoji: g.emoji, actorId, createdAt: '' });
        }
      }
    }
    this._channelReactions = reactions;
  } catch { this._channelReactions = []; }
}

private async _fetchChannelPresence(name: string): Promise<void> {
  try {
    const resp = await fetchWithAuth(`/api/mesh/channels/${encodeURIComponent(name)}/presence`);
    if (!resp.ok) { this._channelMembers = 0; return; }
    const data = await resp.json();
    this._channelMembers = data.subscribers ?? 0;
  } catch { this._channelMembers = 0; }
}
```

- [ ] **Step 3: Update _selectChannel to fetch timeline and reactions**

Modify `_selectChannel()` to trigger fetches:

```typescript
private _selectChannel(name: string): void {
  this._selectedChannel = name;
  this._dockChannel = name;
  this._fetchChannelTimeline(name).then(() => this._fetchChannelReactions(name));
  this._fetchChannelPresence(name);
}
```

- [ ] **Step 4: Replace _renderChannel with channel-feed**

Replace the raw text rendering:

```typescript
private _renderChannel() {
  const { channels } = this._data;
  if (!channels.length) return html`<div class="empty">No active channels</div>`;

  const selected = (!this._selectedChannel || !channels.find(c => c.name === this._selectedChannel))
    ? channels[0]!.name : this._selectedChannel;

  if (selected !== this._selectedChannel) {
    this._selectChannel(selected);
  }

  return html`
    <div style="display:flex; align-items:center; gap:8px; margin-bottom:8px">
      <select class="ch-select" style="flex:1"
        @change=${(e: Event) => this._selectChannel((e.target as HTMLSelectElement).value)}>
        ${channels.map(ch => html`<option value=${ch.name} ?selected=${ch.name === selected}>#${ch.name}</option>`)}
      </select>
      ${this._channelMembers > 0 ? html`
        <pages-badge label="${this._channelMembers} watching" variant="info" size="sm"></pages-badge>
      ` : nothing}
    </div>
    <channel-feed .messages=${this._channelMessages}
      .channelId=${selected}
      .reactions=${this._channelReactions}
      .staleCursorMinutes=${0}></channel-feed>
  `;
}
```

- [ ] **Step 5: Build frontend and verify**

Run: `npm --prefix app/src/main/webui run build`
Expected: builds without errors

- [ ] **Step 6: Commit**

```
git add app/src/main/webui/src/components/claudony-mesh-panel.ts
git commit -m "feat(#178): mesh panel channel view uses channel-feed with reactions

Replace raw text rendering with blocks-ui <channel-feed> in the mesh
panel channel view. Fetches timeline, reactions, and presence count.
Shows 'N watching' badge when SSE subscribers are active.

Closes #178"
```
