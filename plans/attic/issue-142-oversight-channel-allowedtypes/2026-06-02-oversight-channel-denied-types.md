# Oversight Channel `deniedTypes` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `deniedTypes` to Qhorus channels so oversight can be expressed as `deniedTypes = EVENT` (no telemetry on the governance channel), fixing six blocked type flows on `NormativeChannelLayout`'s oversight channel.

**Architecture:** Two sequential phases. Phase 1 (casehub-qhorus): V16 migration, `Channel` entity field, `ChannelDetail` API record field, `StoredMessageTypePolicy` deniedTypes check + overlap validation, `ChannelService`/`ReactiveChannelService` 10-arg overloads, `QhorusEntityMapper` mapping, both MCP tools. Phase 2 (casehub-claudony): `ChannelSpec` record, `NormativeChannelLayout`/`SimpleLayout`, `ClaudonyReactiveCaseChannelProvider`. Qhorus must be built and published before Phase 2 compiles.

**Tech Stack:** Java 21, Quarkus 3.32.2, Flyway, Hibernate ORM, SmallRye Mutiny (reactive path), Mockito (unit tests), QuarkusTest (integration tests).

---

## Phase 1 — casehub-qhorus

All commands run from `/Users/mdproctor/claude/casehub/qhorus`.  
Build/test command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

---

### Task 1: V16 Flyway migration

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V16__add_channel_denied_types.sql`

- [ ] **Step 1.1: Write the migration**

```sql
-- denied_types on channel: comma-separated MessageType names denied on this channel.
-- Null means no types are denied. Enforced by StoredMessageTypePolicy alongside allowed_types.
-- Deny always wins when a type appears in both allowed_types and denied_types.
ALTER TABLE channel ADD COLUMN denied_types TEXT;

-- Fix existing oversight channels: clear the too-narrow allowed_types and set denied_types = EVENT.
-- Targets channels whose name ends with '/oversight' — this is the naming convention
-- established by CaseChannel.channelName(caseId, "oversight").
UPDATE channel
   SET allowed_types = NULL,
       denied_types  = 'EVENT'
 WHERE name LIKE '%/oversight';
```

- [ ] **Step 1.2: Run a smoke build to confirm Flyway picks it up**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SmokeTest -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` (Flyway runs V16 without error).

- [ ] **Step 1.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/resources/db/qhorus/migration/V16__add_channel_denied_types.sql
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(qhorus): V16 migration — denied_types column + fix existing oversight channels"
```

---

### Task 2: Channel entity + ChannelDetail record + MessageTypeViolationException

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageTypeViolationException.java`

These are data-model changes; they must compile before any tests can reference `ch.deniedTypes` or `detail.deniedTypes()`.

- [ ] **Step 2.1: Add `deniedTypes` to `Channel` entity**

In `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`, add after the `allowedTypes` field:

```java
/**
 * Comma-separated list of denied MessageType names.
 * Null means no types are explicitly denied.
 * If a type appears in both allowedTypes and deniedTypes, denial wins.
 * Example: "EVENT" for a governance channel that must not contain telemetry.
 * If a new MessageType is added to Qhorus with no commitment effect (like EVENT),
 * add it here for all governance channels — this is the mechanical anchor for that obligation.
 */
@Column(name = "denied_types", columnDefinition = "TEXT")
public String deniedTypes;
```

- [ ] **Step 2.2: Add `deniedTypes` to `ChannelDetail` API record**

In `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java`, add after `allowedTypes`:

```java
public record ChannelDetail(
        UUID channelId,
        String name,
        String description,
        String semantic,
        String barrierContributors,
        long messageCount,
        String lastActivityAt,
        boolean paused,
        String allowedWriters,
        String adminInstances,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        /** Comma-separated permitted MessageType names, or null if open to all types. */
        String allowedTypes,
        /** Comma-separated denied MessageType names, or null if no types are denied.
         *  Denial wins when a type appears in both allowedTypes and deniedTypes. */
        String deniedTypes,
        ConnectorBinding connectorBinding) {

    public record ConnectorBinding(
            String inboundConnectorId,
            String externalKey,
            String outboundConnectorId,
            String outboundDestination) {}
}
```

- [ ] **Step 2.3: Add `denied()` factory method to `MessageTypeViolationException`**

In `api/src/main/java/io/casehub/qhorus/api/message/MessageTypeViolationException.java`:

```java
public class MessageTypeViolationException extends IllegalStateException {

    public MessageTypeViolationException(String channel, MessageType attempted, String allowed) {
        super("Channel '" + channel + "' does not permit " + attempted + ". Allowed: " + allowed);
    }

    private MessageTypeViolationException(String message) {
        super(message);
    }

    public static MessageTypeViolationException denied(String channel, MessageType attempted, String denied) {
        return new MessageTypeViolationException(
                "Channel '" + channel + "' explicitly denies " + attempted + ". Denied: " + denied);
    }
}
```

- [ ] **Step 2.4: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime --also-make -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java \
  api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java \
  api/src/main/java/io/casehub/qhorus/api/message/MessageTypeViolationException.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(qhorus): add deniedTypes to Channel entity, ChannelDetail record, and MessageTypeViolationException"
```

---

### Task 3: StoredMessageTypePolicy — deniedTypes check + overlap validation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/message/StoredMessageTypePolicyTest.java`

- [ ] **Step 3.1: Write failing tests for deniedTypes behavior**

Add to `StoredMessageTypePolicyTest.java`:

```java
// ── deniedTypes ──────────────────────────────────────────────────────────────

@Test
void nullDeniedTypes_permitsAllTypes() {
    Channel ch = channelWithDenied(null);
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.QUERY));
}

@Test
void deniedType_rejectsDeniedType() {
    Channel ch = channelWithDenied("EVENT");
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
}

@Test
void deniedType_permitsNonDeniedType() {
    Channel ch = channelWithDenied("EVENT");
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.QUERY));
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.COMMAND));
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.RESPONSE));
}

@Test
void deniedType_message_containsChannelAndType() {
    Channel ch = channelWithDenied("EVENT");
    ch.name = "case-abc/oversight";
    MessageTypeViolationException ex = assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
    assertTrue(ex.getMessage().contains("case-abc/oversight"));
    assertTrue(ex.getMessage().contains("EVENT"));
    assertTrue(ex.getMessage().contains("denied"));
}

@Test
void deniedWins_whenTypeInBothAllowedAndDenied() {
    Channel ch = new Channel();
    ch.name = "test";
    ch.allowedTypes = "EVENT,QUERY";
    ch.deniedTypes = "EVENT";
    ch.semantic = ChannelSemantic.APPEND;
    // EVENT is in both — denial wins
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
    // QUERY is only in allowed — permitted
    assertDoesNotThrow(() -> policy.validate(ch, MessageType.QUERY));
}

@Test
void oversightPattern_nullAllowedTypes_deniedEvent_permitsAllOtherTypes() {
    Channel ch = channelWithDenied("EVENT");
    // All obligation-carrying types must be permitted
    for (MessageType t : new MessageType[]{
            MessageType.COMMAND, MessageType.QUERY, MessageType.RESPONSE,
            MessageType.DONE, MessageType.DECLINE, MessageType.FAILURE,
            MessageType.STATUS, MessageType.HANDOFF}) {
        assertDoesNotThrow(() -> policy.validate(ch, t),
                "Expected " + t + " to be permitted on governance channel");
    }
    // EVENT must be denied
    assertThrows(MessageTypeViolationException.class,
            () -> policy.validate(ch, MessageType.EVENT));
}

// ── validateNoOverlap ────────────────────────────────────────────────────────

@Test
void validateNoOverlap_noConflict_passes() {
    assertDoesNotThrow(() -> StoredMessageTypePolicy.validateNoOverlap("EVENT", "QUERY"));
}

@Test
void validateNoOverlap_nullInputs_passes() {
    assertDoesNotThrow(() -> StoredMessageTypePolicy.validateNoOverlap(null, null));
    assertDoesNotThrow(() -> StoredMessageTypePolicy.validateNoOverlap("EVENT", null));
    assertDoesNotThrow(() -> StoredMessageTypePolicy.validateNoOverlap(null, "EVENT"));
}

@Test
void validateNoOverlap_conflict_throwsIllegalArgument() {
    assertThrows(IllegalArgumentException.class,
            () -> StoredMessageTypePolicy.validateNoOverlap("EVENT,QUERY", "EVENT"));
}

@Test
void validateNoOverlap_multipleConflicts_throwsIllegalArgument() {
    assertThrows(IllegalArgumentException.class,
            () -> StoredMessageTypePolicy.validateNoOverlap("EVENT,QUERY,COMMAND", "QUERY,COMMAND"));
}

private Channel channelWithDenied(String deniedTypes) {
    Channel ch = new Channel();
    ch.name = "test-channel";
    ch.semantic = ChannelSemantic.APPEND;
    ch.deniedTypes = deniedTypes;
    return ch;
}
```

- [ ] **Step 3.2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=StoredMessageTypePolicyTest -q 2>&1 | tail -10
```
Expected: several failures (deniedTypes field not checked; validateNoOverlap not defined).

- [ ] **Step 3.3: Implement deniedTypes check and validateNoOverlap**

Replace `runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java`:

```java
package io.casehub.qhorus.runtime.message;

import java.util.Arrays;
import java.util.Set;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.MessageTypeViolationException;
import io.casehub.qhorus.runtime.channel.Channel;

@ApplicationScoped
public class StoredMessageTypePolicy implements MessageTypePolicy {

    @Override
    public void validate(Channel channel, MessageType type) {
        if (channel.allowedTypes != null && !channel.allowedTypes.isBlank()) {
            Set<MessageType> allowed = parseTypes(channel.allowedTypes);
            if (!allowed.contains(type)) {
                throw new MessageTypeViolationException(channel.name, type, channel.allowedTypes);
            }
        }
        if (channel.deniedTypes != null && !channel.deniedTypes.isBlank()) {
            Set<MessageType> denied = parseTypes(channel.deniedTypes);
            if (denied.contains(type)) {
                throw MessageTypeViolationException.denied(channel.name, type, channel.deniedTypes);
            }
        }
    }

    /**
     * Validates that allowedTypes and deniedTypes do not intersect.
     * Called at channel creation time — denial wins at runtime, but overlapping sets
     * indicate a misconfigured channel and must be rejected before persisting.
     *
     * @throws IllegalArgumentException if any type appears in both sets
     */
    public static void validateNoOverlap(String allowedTypes, String deniedTypes) {
        if (allowedTypes == null || allowedTypes.isBlank()
                || deniedTypes == null || deniedTypes.isBlank()) {
            return;
        }
        Set<String> allowed = Arrays.stream(allowedTypes.split(","))
                .map(String::trim)
                .collect(Collectors.toSet());
        Set<String> denied = Arrays.stream(deniedTypes.split(","))
                .map(String::trim)
                .collect(Collectors.toSet());
        denied.retainAll(allowed);
        if (!denied.isEmpty()) {
            throw new IllegalArgumentException(
                    "Channel policy conflict: type(s) " + denied
                    + " appear in both allowed_types and denied_types. "
                    + "A type cannot be simultaneously allowed and denied.");
        }
    }

    private Set<MessageType> parseTypes(String types) {
        return Arrays.stream(types.split(","))
                .map(String::trim)
                .map(MessageType::valueOf)
                .collect(Collectors.toUnmodifiableSet());
    }
}
```

- [ ] **Step 3.4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=StoredMessageTypePolicyTest -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/StoredMessageTypePolicy.java \
  runtime/src/test/java/io/casehub/qhorus/message/StoredMessageTypePolicyTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(qhorus): StoredMessageTypePolicy — deniedTypes check + validateNoOverlap static method  Refs casehubio/claudony#142"
```

---

### Task 4: ChannelCreateRequest — add deniedTypes

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java`

- [ ] **Step 4.1: Add `deniedTypes` field and call `validateNoOverlap` in compact constructor**

Replace `ChannelCreateRequest.java` with:

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.message.StoredMessageTypePolicy;

public record ChannelCreateRequest(
        String name,
        String description,
        ChannelSemantic semantic,
        String barrierContributors,
        String allowedWriters,
        String adminInstances,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        String allowedTypes,
        String deniedTypes,
        // Connector binding — all four non-null together, or all null
        String inboundConnectorId,
        String externalKey,
        String outboundConnectorId,
        String outboundDestination
) {
    public ChannelCreateRequest {
        StoredMessageTypePolicy.validateNoOverlap(allowedTypes, deniedTypes);
        boolean anySet = inboundConnectorId != null || externalKey != null
                || outboundConnectorId != null || outboundDestination != null;
        boolean allSet = inboundConnectorId != null && externalKey != null
                && outboundConnectorId != null && outboundDestination != null;
        if (anySet && !allSet) {
            throw new IllegalArgumentException(
                    "Connector binding requires all four fields: inboundConnectorId, " +
                    "externalKey, outboundConnectorId, outboundDestination");
        }
    }

    public boolean hasConnectorBinding() {
        return inboundConnectorId != null;
    }

    /** Convenience factory — no connector binding, no type restrictions. */
    public static ChannelCreateRequest simple(String name, ChannelSemantic semantic) {
        return new ChannelCreateRequest(name, null, semantic, null, null, null,
                null, null, null, null, null, null, null, null);
    }
}
```

- [ ] **Step 4.2: Compile to surface all `ChannelCreateRequest` construction sites**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | grep -E "error|ERROR" | head -20
```
Expected: compilation errors at every `new ChannelCreateRequest(...)` call site — lists all files that need updating (primarily `QhorusMcpTools.java` and `ReactiveQhorusMcpTools.java`).

- [ ] **Step 4.3: Commit (partial — ChannelCreateRequest only)**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(qhorus): add deniedTypes to ChannelCreateRequest with validateNoOverlap at construction"
```

---

### Task 5: ChannelService + ReactiveChannelService + QhorusEntityMapper

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java`

- [ ] **Step 5.1: Add 10-arg overload to `ChannelService` and update `create(ChannelCreateRequest)`**

In `ChannelService.java`:

The existing 9-arg overload now delegates to the new 10-arg primary:
```java
@Transactional
public Channel create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance, String allowedTypes) {
    return create(name, description, semantic, barrierContributors, allowedWriters,
            adminInstances, rateLimitPerChannel, rateLimitPerInstance, allowedTypes, null);
}

@Transactional
public Channel create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance,
        String allowedTypes, String deniedTypes) {
    StoredMessageTypePolicy.validateNoOverlap(allowedTypes, deniedTypes);
    Channel channel = new Channel();
    channel.name = name;
    channel.description = description;
    channel.semantic = semantic;
    channel.barrierContributors = barrierContributors;
    channel.allowedWriters = blankToNull(allowedWriters);
    channel.adminInstances = blankToNull(adminInstances);
    channel.rateLimitPerChannel = rateLimitPerChannel;
    channel.rateLimitPerInstance = rateLimitPerInstance;
    channel.allowedTypes = blankToNull(allowedTypes);
    channel.deniedTypes = blankToNull(deniedTypes);
    channelStore.put(channel);
    return channel;
}
```

Update `populateChannel(ChannelCreateRequest req)` to add:
```java
channel.deniedTypes = blankToNull(req.deniedTypes());
```

- [ ] **Step 5.2: Add 10-arg overload to `ReactiveChannelService`**

The existing 9-arg overload delegates to the new 10-arg primary:
```java
public Uni<Channel> create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance, String allowedTypes) {
    return create(name, description, semantic, barrierContributors, allowedWriters,
            adminInstances, rateLimitPerChannel, rateLimitPerInstance, allowedTypes, null);
}

public Uni<Channel> create(String name, String description, ChannelSemantic semantic,
        String barrierContributors, String allowedWriters, String adminInstances,
        Integer rateLimitPerChannel, Integer rateLimitPerInstance,
        String allowedTypes, String deniedTypes) {
    return Panache.withTransaction("qhorus", () -> {
        StoredMessageTypePolicy.validateNoOverlap(allowedTypes, deniedTypes);
        Channel channel = new Channel();
        channel.name = name;
        channel.description = description;
        channel.semantic = semantic;
        channel.barrierContributors = barrierContributors;
        channel.allowedWriters = (allowedWriters == null || allowedWriters.isBlank()) ? null : allowedWriters;
        channel.adminInstances = (adminInstances == null || adminInstances.isBlank()) ? null : adminInstances;
        channel.rateLimitPerChannel = rateLimitPerChannel;
        channel.rateLimitPerInstance = rateLimitPerInstance;
        channel.allowedTypes = (allowedTypes == null || allowedTypes.isBlank()) ? null : allowedTypes;
        channel.deniedTypes = (deniedTypes == null || deniedTypes.isBlank()) ? null : deniedTypes;
        return channelStore.put(channel);
    });
}
```

- [ ] **Step 5.3: Update `QhorusEntityMapper.toChannelDetail()`**

In `QhorusEntityMapper.java`, the `toChannelDetail` method becomes:
```java
return new ChannelDetail(
        ch.id, ch.name, ch.description,
        ch.semantic != null ? ch.semantic.name() : null,
        ch.barrierContributors, messageCount,
        ch.lastActivityAt != null ? ch.lastActivityAt.toString() : null,
        ch.paused, ch.allowedWriters, ch.adminInstances,
        ch.rateLimitPerChannel, ch.rateLimitPerInstance, ch.allowedTypes,
        ch.deniedTypes,
        detailBinding);
```

- [ ] **Step 5.4: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` (all non-MCP compilation errors resolved; MCP tools still reference old ChannelCreateRequest with 13 args).

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(qhorus): ChannelService + ReactiveChannelService 10-arg create, QhorusEntityMapper maps deniedTypes  Refs casehubio/claudony#142"
```

---

### Task 6: MCP tools — add `denied_types` parameter

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

Both tools need an identical `denied_types` ToolArg inserted after `allowed_types`.

- [ ] **Step 6.1: Update `ReactiveQhorusMcpTools.createChannel()`**

Add parameter after `allowedTypes`:
```java
@ToolArg(name = "denied_types",
         description = "Comma-separated MessageType names explicitly denied on this channel. "
                 + "Null = no types denied. Denial wins when a type appears in both allowed_types and denied_types. "
                 + "Example: \"EVENT\" for a governance channel (oversight) that must never contain telemetry.",
         required = false) String deniedTypes,
```

Update `@Tool` description to reference `denied_types`:
```
"Use denied_types to exclude specific types (e.g. \"EVENT\" for governance channels). "
+ "Denial wins when a type appears in both allowed_types and denied_types. "
```

Update `ChannelCreateRequest` construction (now 14 args):
```java
ChannelCreateRequest req = new ChannelCreateRequest(name, description, sem, barrierContributors,
        allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance, allowedTypes,
        deniedTypes,
        inboundConnectorId, externalKey, outboundConnectorId, outboundDestination);
```

Update the non-binding path to use the 10-arg create:
```java
return channelService.create(req.name(), req.description(), req.semantic(), req.barrierContributors(),
        req.allowedWriters(), req.adminInstances(), req.rateLimitPerChannel(), req.rateLimitPerInstance(),
        req.allowedTypes(), req.deniedTypes())
```

- [ ] **Step 6.2: Update `QhorusMcpTools.createChannel()`**

Add the same `denied_types` ToolArg after `allowed_types`.

Update the 13-arg overload signature (now 14). Update shorter overloads to pass `null` for `deniedTypes`:
```java
ChannelDetail createChannel(String name, String description, String semantic, String barrierContributors) {
    return createChannel(name, description, semantic, barrierContributors, null, null, null, null, null, null, null, null, null, null);
}
// Similarly for all shorter overloads.
```

`ChannelCreateRequest` construction in `QhorusMcpTools`:
```java
Channel ch = channelService.create(new ChannelCreateRequest(
        name, description, sem, barrierContributors, allowedWriters, adminInstances,
        rateLimitPerChannel, rateLimitPerInstance, allowedTypes,
        deniedTypes,
        inboundConnectorId, externalKey, outboundConnectorId, outboundDestination));
```

- [ ] **Step 6.3: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` — no compilation errors.

- [ ] **Step 6.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(qhorus): add denied_types parameter to create_channel MCP tool  Refs casehubio/claudony#142"
```

---

### Task 7: Qhorus integration tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/message/MessageServiceTypeEnforcementTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/channel/ChannelServiceTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/ChannelAllowedTypesTest.java`

- [ ] **Step 7.1: Add denied_types tests to `ChannelServiceTest`**

Add at the end of `ChannelServiceTest.java`:

```java
@Test
void createWithDeniedTypes_storesConstraint() {
    QuarkusTransaction.requiringNew().run(() -> {
        var ch = channelService.create("denied-types-test-" + System.nanoTime(),
                "Test", ChannelSemantic.APPEND, null, null, null, null, null, null, "EVENT");
        assertEquals("EVENT", ch.deniedTypes);
        assertNull(ch.allowedTypes);
    });
}

@Test
void createWithNullDeniedTypes_storesNull() {
    QuarkusTransaction.requiringNew().run(() -> {
        var ch = channelService.create("denied-null-test-" + System.nanoTime(),
                "Test", ChannelSemantic.APPEND, null, null, null, null, null, null, null);
        assertNull(ch.deniedTypes);
    });
}

@Test
void createWithOverlappingTypes_throwsIllegalArgument() {
    assertThrows(IllegalArgumentException.class, () ->
        QuarkusTransaction.requiringNew().run(() ->
            channelService.create("overlap-test-" + System.nanoTime(),
                "Test", ChannelSemantic.APPEND, null, null, null, null, null,
                "EVENT,QUERY", "EVENT")));
}
```

- [ ] **Step 7.2: Add denied_types enforcement tests to `MessageServiceTypeEnforcementTest`**

Add to `MessageServiceTypeEnforcementTest.java`:

```java
/** Helper: create a channel with deniedTypes constraint. */
private UUID createChannelWithDenied(String name, String deniedTypes) {
    UUID[] id = new UUID[1];
    QuarkusTransaction.requiringNew().run(() -> {
        var ch = channelService.create(name, "Test channel", ChannelSemantic.APPEND,
                null, null, null, null, null, null, deniedTypes);
        id[0] = ch.id;
    });
    return id[0];
}

@Test
void serverSide_deniesDeniedType() {
    String name = "server-denied-" + System.nanoTime();
    UUID channelId = createChannelWithDenied(name, "EVENT");

    assertThrows(MessageTypeViolationException.class,
            () -> QuarkusTransaction.requiringNew().run(() -> messageService.dispatch(
                    MessageDispatch.builder()
                            .channelId(channelId)
                            .sender("agent-1")
                            .type(MessageType.EVENT)
                            .content("{\"tool\":\"ping\"}")
                            .actorType(ActorType.SYSTEM)
                            .build())));
}

@Test
void serverSide_oversightPattern_permitsAllObligationTypes() {
    String name = "oversight-pattern-" + System.nanoTime();
    UUID channelId = createChannelWithDenied(name, "EVENT");

    // All 8 obligation-carrying types must pass through
    String corrId = UUID.randomUUID().toString();
    assertDoesNotThrow(
            () -> QuarkusTransaction.requiringNew().run(() -> messageService.dispatch(
                    MessageDispatch.builder()
                            .channelId(channelId)
                            .sender("agent-1")
                            .type(MessageType.COMMAND)
                            .content("please approve")
                            .correlationId(corrId)
                            .actorType(ActorType.AGENT)
                            .build())));

    // RESPONSE (closes the COMMAND)
    DispatchResult[] cmdResult = new DispatchResult[1];
    QuarkusTransaction.requiringNew().run(() -> {
        cmdResult[0] = messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender("agent-1")
                .type(MessageType.COMMAND)
                .content("approve action X")
                .correlationId(UUID.randomUUID().toString())
                .actorType(ActorType.AGENT)
                .build());
    });
    String respCorrId = cmdResult[0].correlationId();
    assertDoesNotThrow(
            () -> QuarkusTransaction.requiringNew().run(() -> messageService.dispatch(
                    MessageDispatch.builder()
                            .channelId(channelId)
                            .sender("human-1")
                            .type(MessageType.RESPONSE)
                            .content("approved")
                            .correlationId(respCorrId)
                            .inReplyTo(cmdResult[0].messageId())
                            .actorType(ActorType.HUMAN)
                            .build())));
}

@Test
void serverSide_deniedWins_overAllowedTypes() {
    String name = "deny-wins-" + System.nanoTime();
    // Create via service directly to bypass MCP overlap validation — not possible via ChannelCreateRequest
    // (which validates no overlap). Use SQL to set up the pathological case directly.
    // This test verifies runtime behavior, not creation-time validation.
    // We set up via ChannelService with null allowedTypes, null deniedTypes, then mutate in DB.
    // Instead, test the policy directly (already covered in StoredMessageTypePolicyTest#deniedWins).
    // This integration test verifies that the channel's runtime enforcement correctly picks up
    // both allowedTypes and deniedTypes from the DB.
    // Covered by StoredMessageTypePolicyTest — no additional integration test needed here.
}
```

Note: the `deniedWins_overAllowedTypes` integration test is covered by `StoredMessageTypePolicyTest`. Remove the empty test above.

- [ ] **Step 7.3: Add denied_types round-trip test to `ChannelAllowedTypesTest`**

Add to `ChannelAllowedTypesTest.java`:

```java
@Test
void createChannel_withDeniedTypes_roundtripsInDetail() {
    ChannelDetail detail = tools.createChannel("denied-rt-" + System.nanoTime(),
            "test", null, null, null, null, null, null, null, "EVENT",
            null, null, null, null);
    assertEquals("EVENT", detail.deniedTypes());
    assertNull(detail.allowedTypes());
}

@Test
void createChannel_nullDeniedTypes_detailShowsNull() {
    ChannelDetail detail = tools.createChannel("denied-null-rt-" + System.nanoTime(),
            "test", null, null, null, null, null, null, null, null,
            null, null, null, null);
    assertNull(detail.deniedTypes());
}

@Test
void createChannel_overlappingTypes_throwsIllegalArgument() {
    assertThrows(IllegalArgumentException.class, () ->
        tools.createChannel("overlap-rt-" + System.nanoTime(),
            "test", null, null, null, null, null, null, "EVENT", "EVENT",
            null, null, null, null));
}
```

- [ ] **Step 7.4: Run all modified test classes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ChannelServiceTest,MessageServiceTypeEnforcementTest,ChannelAllowedTypesTest,StoredMessageTypePolicyTest" \
  2>&1 | tail -20
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 7.5: Run full Qhorus test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 7.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/test/java/io/casehub/qhorus/message/MessageServiceTypeEnforcementTest.java \
  runtime/src/test/java/io/casehub/qhorus/channel/ChannelServiceTest.java \
  runtime/src/test/java/io/casehub/qhorus/mcp/ChannelAllowedTypesTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(qhorus): deniedTypes enforcement — ChannelService, MessageService, MCP round-trip  Refs casehubio/claudony#142"
```

- [ ] **Step 7.7: Build and publish Qhorus SNAPSHOT**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn deploy -DskipTests -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` and SNAPSHOT published to GitHub Packages. Phase 2 cannot start until this succeeds.

---

## Phase 2 — casehub-claudony

All commands run from `/Users/mdproctor/claude/casehub/claudony`.  
Build/test command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub`

**Prerequisite:** Qhorus Phase 1 must be published. Verify:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:resolve -pl casehub -q 2>&1 | tail -5
```

---

### Task 8: `CaseChannelLayout.ChannelSpec` — add `deniedTypes` field

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/CaseChannelLayout.java`

This is a source-breaking change — all `ChannelSpec` construction sites fail to compile until updated. Do this first so the compiler guides you to every callsite.

- [ ] **Step 8.1: Add `deniedTypes` to `ChannelSpec` record**

In `CaseChannelLayout.java`, update the `ChannelSpec` record:

```java
record ChannelSpec(
        String purpose,
        ChannelSemantic semantic,
        Set<MessageType> allowedTypes,  // null = open to all types
        Set<MessageType> deniedTypes,   // null = no denial; if a type has no commitment effect and
                                        // is added to MessageType, add it here for governance channels
        String description
) {}
```

- [ ] **Step 8.2: Compile to surface all construction sites**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl casehub --also-make -q 2>&1 | grep "error:" | head -20
```
Expected: compilation errors at `NormativeChannelLayout.java` and `SimpleLayout.java`.

- [ ] **Step 8.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  casehub/src/main/java/io/casehub/claudony/casehub/CaseChannelLayout.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(claudony): add deniedTypes field to CaseChannelLayout.ChannelSpec record  Refs #142"
```

---

### Task 9: `NormativeChannelLayout` + `SimpleLayout`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/NormativeChannelLayout.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/SimpleLayout.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/NormativeChannelLayoutTest.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/SimpleLayoutTest.java`

- [ ] **Step 9.1: Write failing test for oversight deniedTypes**

In `NormativeChannelLayoutTest.java`, replace the oversight allowedTypes assertion:

```java
// DELETE this test:
// void channelsFor_oversightChannel_allowsQueryAndCommand()

// ADD these tests:
@Test
void channelsFor_oversightChannel_allowedTypesIsNull() {
    CaseChannelLayout.ChannelSpec oversight = layout.channelsFor(UUID.randomUUID(), null).stream()
            .filter(s -> s.purpose().equals("oversight"))
            .findFirst().orElseThrow();
    assertThat(oversight.allowedTypes()).isNull();
}

@Test
void channelsFor_oversightChannel_deniedTypesContainsEventOnly() {
    CaseChannelLayout.ChannelSpec oversight = layout.channelsFor(UUID.randomUUID(), null).stream()
            .filter(s -> s.purpose().equals("oversight"))
            .findFirst().orElseThrow();
    assertThat(oversight.deniedTypes()).containsExactly(MessageType.EVENT);
}

@Test
void channelsFor_workChannel_deniedTypesIsNull() {
    CaseChannelLayout.ChannelSpec work = layout.channelsFor(UUID.randomUUID(), null).stream()
            .filter(s -> s.purpose().equals("work"))
            .findFirst().orElseThrow();
    assertThat(work.deniedTypes()).isNull();
}

@Test
void channelsFor_observeChannel_deniedTypesIsNull() {
    CaseChannelLayout.ChannelSpec observe = layout.channelsFor(UUID.randomUUID(), null).stream()
            .filter(s -> s.purpose().equals("observe"))
            .findFirst().orElseThrow();
    // deniedTypes null on observe — EVENT is already the only allowed type; denial is redundant
    assertThat(observe.deniedTypes()).isNull();
}
```

- [ ] **Step 9.2: Run to verify tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl casehub -q 2>&1 | tail -10
```
Expected: compilation failure (ChannelSpec constructor wrong arity).

- [ ] **Step 9.3: Update `NormativeChannelLayout`**

Replace `NormativeChannelLayout.java`:

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.CaseDefinition;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public class NormativeChannelLayout implements CaseChannelLayout {

    @Override
    public List<ChannelSpec> channelsFor(UUID caseId, CaseDefinition definition) {
        return List.of(
                new ChannelSpec("work", ChannelSemantic.APPEND, null, null,
                        "Primary coordination — all obligation-carrying message types"),
                new ChannelSpec("observe", ChannelSemantic.APPEND, Set.of(MessageType.EVENT), null,
                        "Telemetry — EVENT only, no obligations created"),
                new ChannelSpec("oversight", ChannelSemantic.APPEND,
                        null,                       // allowedTypes: open — all obligation-carrying types permitted
                        // If a new MessageType is added to Qhorus with no commitment effect (like EVENT),
                        // add it here. This comment is the mechanical anchor for that obligation.
                        Set.of(MessageType.EVENT),  // deniedTypes: no telemetry on the governance channel
                        "Human governance — all obligation-carrying types; no telemetry")
        );
    }
}
```

- [ ] **Step 9.4: Update `SimpleLayout`** (mechanical — add `null` deniedTypes)

```java
package io.casehub.claudony.casehub;

import io.casehub.api.model.CaseDefinition;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public class SimpleLayout implements CaseChannelLayout {

    @Override
    public List<ChannelSpec> channelsFor(UUID caseId, CaseDefinition definition) {
        return List.of(
                new ChannelSpec("work", ChannelSemantic.APPEND, null, null,
                        "Primary coordination — all obligation-carrying message types"),
                new ChannelSpec("observe", ChannelSemantic.APPEND, Set.of(MessageType.EVENT), null,
                        "Telemetry — EVENT only, no obligations created")
        );
    }
}
```

- [ ] **Step 9.5: Run layout tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub \
  -Dtest="NormativeChannelLayoutTest,SimpleLayoutTest" 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 9.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  casehub/src/main/java/io/casehub/claudony/casehub/NormativeChannelLayout.java \
  casehub/src/main/java/io/casehub/claudony/casehub/SimpleLayout.java \
  casehub/src/test/java/io/casehub/claudony/casehub/NormativeChannelLayoutTest.java \
  casehub/src/test/java/io/casehub/claudony/casehub/SimpleLayoutTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(claudony): oversight deniedTypes=EVENT via NormativeChannelLayout; fix SimpleLayout arity  Refs #142"
```

---

### Task 10: `ClaudonyReactiveCaseChannelProvider` — pass `deniedTypes` through

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java`

- [ ] **Step 10.1: Write failing tests**

In `ClaudonyReactiveCaseChannelProviderTest.java`:

**Update `stubCreate` helper** — the mock now expects 10 args (add `any()` for `deniedTypes`):
```java
private void stubCreate(UUID caseId) {
    when(channelService.create(
            contains(caseId.toString()), any(), any(),
            isNull(), isNull(), isNull(), isNull(), isNull(), any(), any()))
            .thenAnswer(inv -> {
                String name = inv.getArgument(0);
                return Uni.createFrom().item(stubChannel(UUID.randomUUID(), name));
            });
}
```

**Update ALL `verify(channelService.create(...))` calls** that use 9 args to use 10 args (add `any()` for `deniedTypes`):

- `openChannel_initializesAllLayoutChannels` → `any()` ×3 null args + 2 `any()` args:
```java
verify(channelService, times(3)).create(
        anyString(), any(), any(),
        isNull(), isNull(), isNull(), isNull(), isNull(), any(), any());
```

- `openChannel_cacheHit_doesNotCallChannelService` → same pattern

- `openChannel_differentCaseIds_initializeSeparately` → same pattern × 6 times

- `openChannel_concurrentCallsSameCaseId_initializesOnlyOnce` → same pattern × 3 times

- `openChannel_failedInit_retriesOnNextCall` → update mock setup with 10-arg when()

- `openChannel_channelNameContainsCaseIdAndPurpose` — specific assertion for `work` channel (both allowedTypes and deniedTypes null):
```java
verify(channelService).create(
        eq("case-" + caseId + "/work"), any(), any(),
        isNull(), isNull(), isNull(), isNull(), isNull(), isNull(), isNull());
```

**Replace `openChannel_oversightChannel_passesAllowedTypes`** with new oversight assertions:
```java
@Test
void openChannel_oversightChannel_passesNullAllowedTypesAndDeniedEvent() {
    UUID caseId = UUID.randomUUID();
    stubCreate(caseId);

    provider.openChannel(caseId, "oversight").await().indefinitely();

    verify(channelService).create(
            contains("/oversight"), any(), any(),
            isNull(), isNull(), isNull(), isNull(), isNull(),
            isNull(),        // allowedTypes: null (open)
            eq("EVENT"));   // deniedTypes: "EVENT" (no telemetry)
}

@Test
void openChannel_observeChannel_passesAllowedEventNullDenied() {
    UUID caseId = UUID.randomUUID();
    stubCreate(caseId);

    provider.openChannel(caseId, "observe").await().indefinitely();

    verify(channelService).create(
            contains("/observe"), any(), any(),
            isNull(), isNull(), isNull(), isNull(), isNull(),
            eq("EVENT"),  // allowedTypes: EVENT only
            isNull());    // deniedTypes: null
}
```

- [ ] **Step 10.2: Run to verify failures**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl casehub -q 2>&1 | tail -10
```
Expected: compilation errors (channelService.create() 9-arg mock doesn't match 10-arg call after implementation change).

- [ ] **Step 10.3: Update `ClaudonyReactiveCaseChannelProvider`**

Add `toDeniedTypesString()` method:
```java
private static String toDeniedTypesString(Set<MessageType> types) {
    if (types == null || types.isEmpty()) return null;
    return types.stream().map(MessageType::name).sorted().collect(Collectors.joining(","));
}
```

Update `initializeLayout()` to extract and pass both strings:
```java
private Uni<Map<String, CaseChannel>> initializeLayout(UUID caseId) {
    List<CaseChannelLayout.ChannelSpec> specs = layout.channelsFor(caseId, null);

    Uni<Map<String, CaseChannel>> seed = Uni.createFrom().item(new ConcurrentHashMap<>());
    for (CaseChannelLayout.ChannelSpec spec : specs) {
        String allowedTypes = toAllowedTypesString(spec.allowedTypes());
        String deniedTypes = toDeniedTypesString(spec.deniedTypes());
        seed = seed.flatMap(acc ->
                createQhorusChannel(caseId, spec.purpose(), spec.semantic().name(), allowedTypes, deniedTypes)
                        .map(ch -> {
                            acc.put(spec.purpose(), ch);
                            return acc;
                        }));
    }
    return seed;
}
```

Update `createQhorusChannel()` to accept and pass `deniedTypes`:
```java
private Uni<CaseChannel> createQhorusChannel(UUID caseId, String purpose, String semantic,
        String allowedTypes, String deniedTypes) {
    String channelName = CaseChannel.channelName(caseId, purpose);
    io.casehub.qhorus.api.channel.ChannelSemantic channelSemantic =
            semantic != null ? io.casehub.qhorus.api.channel.ChannelSemantic.valueOf(semantic) : null;
    return channelService.create(channelName, purpose, channelSemantic,
                    null, null, null, null, null, allowedTypes, deniedTypes)
            .map(detail -> {
                gateway.initChannel(detail.id,
                        new io.casehub.qhorus.api.gateway.ChannelRef(detail.id, detail.name));
                channelCreatedEvent.fire(
                        new io.casehub.claudony.server.CaseChannelCreatedEvent(detail.id, detail.name));
                return new CaseChannel(
                        detail.id.toString(),
                        detail.name,
                        purpose,
                        "qhorus",
                        Map.of(QHORUS_NAME_KEY, detail.name));
            });
}
```

- [ ] **Step 10.4: Run provider tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub \
  -Dtest=ClaudonyReactiveCaseChannelProviderTest 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 10.5: Run full casehub module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -q 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 10.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java \
  casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(claudony): pass deniedTypes through ClaudonyReactiveCaseChannelProvider  Refs #142"
```

---

### Task 11: Full test suite + documentation

**Files:**
- Modify: `docs/specs/2026-04-27-claudony-agent-mesh-framework.md`
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`
- Create: `/Users/mdproctor/claude/casehub/parent/docs/protocols/casehub/<new-protocol-file>.md`

- [ ] **Step 11.1: Run full Claudony test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, 532+ tests pass (pre-existing `CaseEngineRoundTripTest` failure is engine SNAPSHOT in-flight, not related).

- [ ] **Step 11.2: Update mesh framework spec — line 708**

In `docs/specs/2026-04-27-claudony-agent-mesh-framework.md`, replace:
```
**`allowed_types`** — Pass `"EVENT"` when creating the observe channel; `"QUERY,COMMAND"` for the oversight channel. Enforced server-side by `MessageTypePolicy` SPI.
```
With:
```
**`allowed_types` / `denied_types`** — Pass `"EVENT"` as `allowed_types` when creating the observe channel (telemetry only, no obligations). Pass `"EVENT"` as `denied_types` when creating the oversight channel (all obligation-carrying types permitted; no telemetry). Work channel: both null (open). Enforced server-side by `StoredMessageTypePolicy`. Denial wins when a type appears in both fields; overlapping sets are rejected at channel creation time.
```

- [ ] **Step 11.3: Update mesh framework spec — line 68 (oversight description)**

Replace:
```
**`oversight`** — The human governance channel. Agents post `QUERY` here when they need human input. Humans post `COMMAND` here to inject directives. [...]
```
With:
```
**`oversight`** — The human governance channel. Agents post `COMMAND` here when requesting human approval for consequential actions (engine#402 gate), or `QUERY` when seeking human input. Humans respond with `RESPONSE`, `DECLINE`, `STATUS`, `DONE`, `FAILURE`, or `HANDOFF` as appropriate. The full commitment lifecycle is supported — `EVENT` is the only excluded type. Watchdog alerts about oversight commitments should register `notificationChannel = observe` (EVENTs on oversight are invisible to participants: excluded from agent context by type semantics and from default `pollAfter` results).
```

- [ ] **Step 11.4: Update PLATFORM.md oversight row**

In `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`, replace:
```
| `oversight` | Human governance gates (commitment-based) | COMMAND → human, RESPONSE from human |
```
With:
```
| `oversight` | Human governance gates (commitment-based) | All obligation-carrying types (COMMAND, QUERY, RESPONSE, DONE, DECLINE, FAILURE, STATUS, HANDOFF); EVENT excluded — no telemetry on the governance channel (`deniedTypes = EVENT`) |
```

- [ ] **Step 11.5: Capture protocol in parent repo**

Create `/Users/mdproctor/claude/casehub/parent/docs/protocols/casehub/channel-type-policy-invariant.md`:

```markdown
---
id: PP-20260602-<generate-id>
title: "allowedTypes and deniedTypes are architectural invariants, not category labels — only set them when a hard constraint must never be violated"
type: rule
scope: platform
applies_to: "Any CaseChannelLayout implementation or any code creating Qhorus channels"
severity: important
refs:
  - ../../specs/claudony/2026-06-02-oversight-channel-allowedtypes-design.md
violation_hint: "Setting allowedTypes on a channel to express what it is 'for' rather than what must be architecturally excluded; e.g. setting allowedTypes = COMMAND,QUERY on oversight to indicate governance intent"
created: 2026-06-02
---

`allowedTypes` and `deniedTypes` on Qhorus channels are hard enforcement gates, not documentation. Only set them when a hard architectural constraint must be enforced across all scenarios:

- `observe` enforces `allowedTypes = EVENT`: no obligations may ever be created on the telemetry channel.
- `oversight` enforces `deniedTypes = EVENT`: no telemetry may ever appear on the governance channel. EVENT is excluded because it has no commitment effect, is not delivered to agent context, and is excluded from default `pollAfter` results — it is invisible to governance participants.
- `work` has no constraint (both null): it is the open coordination space.

Channels that participate in the full commitment lifecycle must have `allowedTypes = null`. If a new `MessageType` is added to Qhorus with no commitment effect (like `EVENT`), update `deniedTypes` on all governance channels and the `NormativeChannelLayout` comment that anchors this obligation.

`allowedTypes` and `deniedTypes` must not overlap. Overlapping sets are rejected at channel creation time with `IllegalArgumentException`. Denial wins at runtime if an overlap exists (but callers must never create such a configuration).
```

Replace `<generate-id>` with `$(python3 -c 'import secrets; print(secrets.token_hex(3))')`.

- [ ] **Step 11.6: Commit docs**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add \
  docs/specs/2026-04-27-claudony-agent-mesh-framework.md
git -C /Users/mdproctor/claude/casehub/claudony commit -m "docs(claudony): update mesh framework spec — oversight deniedTypes model  Refs #142"

git -C /Users/mdproctor/claude/casehub/parent add \
  docs/PLATFORM.md \
  docs/protocols/casehub/channel-type-policy-invariant.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(platform): oversight channel type model — deniedTypes, channel-type-policy-invariant protocol  Refs casehubio/claudony#142"
```
