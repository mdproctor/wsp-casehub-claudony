# postToChannel SPI: correlationId and deadline as first-class params

**Issue:** casehubio/claudony#135
**Date:** 2026-05-23
**Status:** Approved

---

## Problem

`ReactiveCaseChannelProvider.postToChannel()` and its blocking mirror `CaseChannelProvider.postToChannel()` carry only `(CaseChannel, String from, String content, MessageType type)`. The engine's `WorkerScheduleEventHandler` has `correlationId` and `deadline` as typed values at the call site but must serialize them into the `CommandContent` JSON content. Claudony's implementation then parses `correlationId` back out of the JSON to pass it to the Qhorus message service. This is content coupling — the integration layer must know the internal wire format of engine-generated COMMAND messages.

`deadline` is even worse: it's serialized into `CommandContent` but Claudony never extracts it, so `Message.deadline` is always null and Layer 3 temporal obligation tracking (Watchdog, deadline enforcement) never fires.

## Design

### SPI signatures (6-param primary)

Both `CaseChannelProvider` (blocking) and `ReactiveCaseChannelProvider` (reactive):

```java
void postToChannel(CaseChannel channel, String from, String content,
                   MessageType type, String correlationId, String deadline);
```

- `correlationId`: nullable. The engine passes `String.valueOf(eventLogId)` for COMMAND/QUERY.
- `deadline`: nullable ISO-8601 Instant string. The engine passes `PropagationContext.getDeadline().map(Object::toString).orElse(null)`.

Existing 3-param convenience default updated to delegate with three nulls:

```java
default void postToChannel(CaseChannel channel, String from, String content) {
    postToChannel(channel, from, content, null, null, null);
}
```

### Engine caller

`WorkerScheduleEventHandler.dispatchCommand()` passes both values directly:

```java
caseChannelProvider.postToChannel(channel, "casehub-engine:orchestrator",
        serialize(command), MessageType.COMMAND,
        String.valueOf(eventLogId), deadline);
```

`CommandContent` retains both fields in the JSON (the worker agent reads them from content).

### No-op implementations

`NoOpCaseChannelProvider` and `NoOpReactiveCaseChannelProvider`: update method signatures, ignore both new params (no-op body unchanged).

### Claudony implementation

```java
@Override
public Uni<Void> postToChannel(CaseChannel channel, String from, String content,
        MessageType type, String correlationId, String deadline) {
    return messageService.dispatch(MessageDispatch.builder()
                    .channelId(UUID.fromString(channel.id()))
                    .sender(from)
                    .type(type)
                    .content(content)
                    .correlationId(correlationId)
                    .deadline(deadline != null ? Instant.parse(deadline) : null)
                    .actorType(ActorType.AGENT)
                    .build())
            .replaceWithVoid();
}
```

`extractCorrelationId()` private method deleted entirely. No JSON parsing of content.

### Contract tests

`CaseChannelProviderContractTest` and `ReactiveCaseChannelProviderContractTest`: update reflection-based method signature checks (4-param → 6-param) and behavioral test calls.

---

## Changes

### casehub-engine (cross-repo)

1. **`CaseChannelProvider.java`** — primary method: add `String correlationId, String deadline`. Update 3-param default to delegate with `null, null, null`.
2. **`ReactiveCaseChannelProvider.java`** — same changes as above.
3. **`NoOpCaseChannelProvider.java`** — update `postToChannel` signature.
4. **`NoOpReactiveCaseChannelProvider.java`** — update `postToChannel` signature.
5. **`WorkerScheduleEventHandler.dispatchCommand()`** — pass `String.valueOf(eventLogId)` and `deadline` to the new 6-param `postToChannel`.
6. **`CaseChannelProviderContractTest.java`** — update reflection checks and behavioral calls.
7. **`ReactiveCaseChannelProviderContractTest.java`** — update reflection checks and behavioral calls.

### Claudony

8. **`ClaudonyReactiveCaseChannelProvider.java`** — accept `correlationId` and `deadline` as params, remove `extractCorrelationId()`, pass `deadline` to `MessageDispatch.builder().deadline()`.
9. **`ClaudonyReactiveCaseChannelProviderTest.java`** — update mock/stub calls for new 6-param signature. Add test verifying `correlationId` and `deadline` are threaded through to `MessageDispatch`.

---

## Platform coherence

- SPI blocking-reactive parity maintained: both interfaces get the same parameter change.
- The SPI is in `casehub-engine-api` (correct module per module-tier-structure protocol).
- No application-tier callers (devtown, aml, clinical don't call postToChannel).
- `CommandContent` Javadoc updated to note that `correlationId` and `deadline` are now first-class SPI params, not only embedded in content.
